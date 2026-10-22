# Comment-triggered automations

## 1. Scope and design choice

The service orchestrates the existing capabilities for receiving comments and DMs, replying to comments, and sending DMs. It does not replace social integrations or implement OAuth. An email address is collected as input; the requested link is sent by DM, not by email.

I propose a TypeScript application using a supported Node.js LTS release, Fastify, and PostgreSQL. Scenario behavior is ordinary code with explicit states. Customers configure supported conditions, messages, materials, and deadlines. New behavior goes through development and release rather than a workflow DSL or runtime code generation.

This trades unrestricted customer-defined workflows for a smaller execution model with understandable failure recovery. Common mechanisms handle persistence, waits, actions, retries, and diagnostics; scenario code decides what happens next.

The required flow is the core of the proposal. Queued requests and versioned delivery quotas are extensions described separately. An administrative UI and a working implementation are outside this submission.

## 2. Customer-visible behavior

### Triggers and configuration

An automation belongs to a connected account and selects posts and explicit trigger phrases. A new rule does not match comments until it has a published revision. The customer prepares supported parameters, previews matches, and submits them for validation: account/post ownership, trigger rules, supported scenario versions, templates, material reference, and timeouts.

Publication locks the automation and atomically inserts an immutable revision and switches `current_revision_id`. Failed validation or a rolled-back publication leaves the previous revision in service; unpublished edits never match comments. Editing repeats this process. Newly created runs pin the selected revision and its execution-code version; a repeat request merged into an existing run does not change that run's settings. Disabling stops new admissions, while cancelling accepted runs is a separate operation.

The initial matcher normalizes case and whitespace, then matches a complete phrase at Unicode letter/digit boundaries. For `pricing`, both `PRICING!` and `pricing please` match; `repricing` does not. Synonyms, translations, and misspellings must be listed explicitly. `not pricing` still matches: this is phrase matching, not intent classification. A preview should expose those distinctions.

I would not add semantic matching initially. Its benefit must justify false positives, latency, and maintenance cost. Languages without suitable word boundaries need a separately supported matching mode.

If multiple automations match one comment, the proposed default selects one by priority, then stable ID, and records the other matches. This is a product policy to confirm, not a reliability requirement. Only new user comments trigger runs; echoed outbound events, own replies, and comment edits do not.

### Conversation and input

One scenario owns a conversation at a time. Prompts name the requested material and, where available, its source post:

> You requested the price list. What email address should we use? I’ll send the link here in this conversation.

A reply must contain exactly one syntactically valid email address. Validation does not prove ownership or deliverability. Invalid replies receive a bounded number of clarification prompts and can be corrected while the scenario is waiting for an email. Accepting a valid address cancels any clarification actions that have not started; calls already in flight cannot be retracted.

The scenario accepts the first syntactically valid email address. Correcting an accepted address is outside this proposal's scope; it requires a separate interaction policy when multiple requests share a conversation. Later messages neither replace the accepted address nor trigger another delivery for that run. The link is sent by DM, so an email typo does not prevent access to the material.

The total request lifetime includes queueing and recovery. A separate response timeout begins when the integration confirms acceptance of the email prompt. Both respect known platform messaging restrictions. A new run asks for an address again; reusing previously collected personal data would be a separate policy.

The example’s “Sent you a DM!” precedes the actual DM and can become misleading if that send fails. The execution order is preserved. If copy is configurable, I would recommend “Let’s continue in DMs” instead.

## 3. Architecture

```text
Existing social integrations
             | inbound events
             v
            API ----------> PostgreSQL
                                | events, runs, jobs, actions
                                v
                         Transition workers
                                | persisted outbound action
                                v
                           Send workers ----> adapter ----> platform
                                ^
                         Timers and reconciliation
```

These are process roles, not separate microservices. A small deployment needs an API process, one background process, and PostgreSQL. The roles can scale independently later. No process holds irreplaceable execution state.

| Option | When it fits | Main cost |
| --- | --- | --- |
| PostgreSQL jobs | A bounded scenario without an established queue | Job claiming, cleanup, and reconciliation must be maintained |
| PostgreSQL plus a broker | A suitable queue already exists, or job transport needs independent capacity | Another component and a transactional outbox to avoid lost scheduling |
| Durable workflow engine | Already adopted, or scenario complexity has grown substantially | Another execution and upgrade model; external sends still need idempotency |

I choose PostgreSQL jobs for the initial design because state changes and scheduling can commit together. If the product already has a suitable queue, reusing it may be cheaper than introducing a new mechanism. Neither Redis nor Kafka is necessary just to wait durably for a DM. The HTTP framework is secondary to the transaction model.

## 4. Data model

Every customer-owned record carries `tenant_id`. External IDs are scoped by platform and connected account. Mapping a comment author to a DM recipient is an integration contract, not an assumption that their IDs are interchangeable.

An `automation` is a customer's configured rule, not an executable workflow definition. Its revision supplies triggers, copy, material, and timeouts to a supported scenario implemented in code. A `run` is one execution with those settings pinned. Configuration revisions, material versions, and execution-code versions solve different problems; they are not interchangeable.

| Entity | Purpose and key constraint |
| --- | --- |
| `automation` | Stable customer rule: account, enabled flag, priority, current published revision |
| `automation_revision` | Immutable scenario parameters; runs keep their selected revision after edits |
| `inbox_event` | Durable incoming event, source deduplication key, canonical message ID, conversation sequence |
| `conversation`, `request` | Active run, ordered pending requests, source comments, and send restrictions |
| `run`, `input_wait` | Execution state, deadlines, accepted input, and wait generation |
| `message_claim` | Prevents one platform message from satisfying multiple waits |
| `action`, `attempt` | One logical outbound effect and its individual execution attempts |
| `job` | Durable work with a due time, lease, and generation |
| `material_version`, `delivery_quota`, `quota_reservation` | Optional versioned material limits and atomic reservations |
| `issue`, `run_event` | Current problems and an audit trail with minimal personal data |

A logical action has a unique key such as `(run, purpose, ordinal)`. Request-level notices use the request as owner. Retries preserve the action ID, recipient, and content but create new attempt records.

A partial unique constraint prevents two nonterminal runs for the same conversation, automation, and material version. The conversation row serializes activation. Message consumption is unique by canonical message ID, not merely webhook event ID.

A `request` records a matched comment; several repeat requests may join one run. An `action` is one intended external effect; its `attempt` records are the individual tries. A `job` is only a durable wakeup to process those records.

Current records drive execution. History explains outcomes; full event sourcing is unnecessary. The [data model](DATA-MODEL.md) includes Mermaid relationship diagrams and describes every record's purpose, fields, lifecycle, and constraints. [Execution details](EXECUTION.md#2-records-and-constraints) summarize the invariants used by the transaction protocols.

## 5. Execution model

The normal run path is:

```text
queued -> comment_reply_pending -> email_prompt_pending
       -> awaiting_email -> material_pending -> completed
```

Other terminal outcomes are `expired`, `cancelled`, and `stopped`, each with a reason. A transient error does not reset the run.

Actions have separate states: `ready`, `in_flight`, `retry_wait`, `blocked`, `unknown`, `succeeded`, `failed`, and `cancelled`. An `unknown` action may already have affected the outside world. It is not an ordinary retryable failure.

### Receive and transition

The API persists an event and its processing job in one transaction, then acknowledges receipt. A database failure must not produce a successful acknowledgment. The source must retry or support replay to cover failures before persistence.

A transition worker locks the conversation and run, checks their state and version, and atomically marks the event processed, updates state, and creates the next action or job. Repeated processing cannot create another logical effect because both state checks and uniqueness constraints apply.

Jobs are claimed with short `FOR UPDATE SKIP LOCKED` transactions. Leases expose abandoned work; attempt tokens stop stale workers from overwriting newer decisions. Neither mechanism revokes an external request already in progress.

### Send

Before sending, a worker checks deadlines, opt-outs, holds, platform eligibility, shared account rate limits, and any quota reservation. It records an attempt and `in_flight`, commits, then calls the integration outside the transaction. A new transaction records the result and continuation. No database transaction stays open across the network call.

An accepted action is not resent during ordinary recovery. A proven temporary rejection may be retried with bounded backoff. An ambiguous result requires safe integration-level retry or reconciliation.

### Wait for input

The input wait is created before sending the prompt and activated after confirmed acceptance. Incoming DMs are persisted regardless of wait state. Activation schedules a scan of already stored messages, covering a reply that arrives before the send result is recorded.

Every consumer uses the same conversation-ingress ordering. A job wakes the consumer; it does not let its associated message jump ahead of earlier messages. When delivery is permitted, claiming an email, closing the wait, and scheduling the link are atomic. Quota denial follows the terminal branch described below, not a technical retry.

Reliable reply-to metadata or platform ordering improves correlation. Without it, the fallback associates the first eligible reply after the prompt attempt began with the active scenario. This is a heuristic: a delayed reply to an earlier interaction can be misattributed. Strict causality requires a stronger integration contract.

### Finish

`completed` means the integration accepted the final DM, not that the person received or read it. Cancellation prevents future steps but cannot retract an in-flight send. A terminal run retains conversation ownership while an old DM is `in_flight` or `unknown`; otherwise a late email prompt could overlap the next scenario. Releasing ownership and advancing the queue are atomic.

## 6. End-to-end example

On the successful path, comment `C1` contains `pricing`, the selected automation points to material version `V1`, and any configured quota has capacity.

| Step | Records committed together |
| --- | --- |
| Receive `C1` | Inbox event and processing job |
| Match the comment | Request, run `R1`, pinned configuration, and public reply `A1` |
| Integration accepts `A1` | Action success, input wait, and email prompt `A2` |
| Integration accepts `A2` | Active wait, response deadline, and stored-DM scan job |
| Accept email from `M1` | Message claim, email, closed wait, quota reservation if enabled, and link action `A3` |
| Integration accepts `A3` | Action success, quota accounting if enabled, completed run, and safe queue advancement |

**`M1` arrives before `A2`’s result is committed.** The message is already durable. Once the wait activates, the shared consumer finds it. Competing workers cannot claim it twice.

**The platform accepts `A3`, but the response is lost.** Our database cannot establish whether the send happened. If idempotency is supported, retry with the same key within its validity window. If a reliable result lookup exists, reconcile first. Otherwise retain `unknown`, the reservation, and conversation ownership. Show the automation owner an unresolved issue rather than reporting success or hiding uncertainty with another send.

## 7. Reliability and recovery

The implementation must preserve these invariants:

1. Acknowledged input is persisted within the database’s stated durability guarantees.
2. Reprocessing does not create a second logical action.
3. One DM satisfies at most one wait; link delivery requires an accepted email.
4. Concurrent deliveries cannot exceed an enabled quota because they reserve capacity atomically.
5. Recovery does not repeat successful actions or treat unknown sends as unsent.
6. A new scenario does not start a DM exchange over an unresolved send from its predecessor.

| Failure | Response |
| --- | --- |
| Transaction rolls back | Retry without a partial state transition |
| Proven transient rejection without an effect | Bounded retry with backoff and deadline checks |
| Repairable access or configuration issue | Block the action, expose the reason, resume after repair |
| Permanent rejection | Stop with a reason instead of retrying indefinitely |
| Ambiguous send | Reconcile or require an audited decision; do not retry blindly |

Issues become visible when recorded. Alerts group by cause and scope so an account outage does not generate an alert per run. Resuming after repair rechecks eligibility and continues from the failed action, not from the beginning.

A reconciler finds missing jobs, expired leases, and overdue waits. It schedules the same idempotent processing paths rather than sending directly. Database availability needs independent monitoring: an issue row cannot be the only signal that the database is down.

Without external idempotency or reliable result lookup, the system cannot promise both eventual sending and no duplicates after an ambiguous outcome. Platform API acceptance is also not proof of delivery to a person.

## 8. Optional product extensions

These policies are not required by the original example and can be omitted from a minimal release:

- **Repeat requests:** merge a new comment into an existing nonterminal run for the same automation and material version, including queued runs. Keep each source comment in history. After completion, a new comment can start a fresh run.
- **Different requests:** queue distinct materials or versions in durable order rather than silently discarding them. Prompts identify their material; queue acknowledgments explain expiry. Status replies are rate-limited and do not start another DM chain.
- **Versioned quotas:** count by tenant, account, recipient, and material version. Reserve only when preparing the link action, using `accepted_count + reserved_count < limit`. Success consumes the reservation; proven non-send, including cancellation with no in-flight or unknown attempt, releases it; an unknown result retains it. If capacity is unavailable, consume the valid input, close the wait, and stop with `quota_unavailable` without creating a link action or reservation. This is a visible business outcome, not an automatic retry or wait for quota.
- **Activity signals:** show unusual repeat activity to the administrator without automatically declaring spam, blocking delivery, or assuming multiple accounts belong to one person.

A new material version gets separate accounting without erasing history or removing opt-outs. The accepted request pins its version. An external URL with mutable content is not itself an immutable material version. Exact timeouts, quota values, notices, and late delivery-failure handling remain product decisions; [execution details](EXECUTION.md#1-product-policies) state the proposed policies without inventing numerical requirements.

## 9. Operations, security, and validation

API and worker roles are stateless. Connections, batches, and network concurrency are bounded; account rate limits are shared across instances. Scheduling must prevent one tenant from monopolizing capacity. More workers do not remove database limits.

Release one immutable OCI image, with a minimal runtime and no dependency installation at startup. Run without root, handle shutdown signals, and execute migrations separately. Rolling deployment must preserve handlers for pinned execution versions or explicitly migrate those runs.

One VPS is not high availability. Replication, backups, and recovery procedures must match agreed RPO, RTO, and budget. After restoring an old backup, hold sends and access to restored personal data until external effects and newer opt-outs, blocks, and deletions are reconciled. Without reliable evidence of newer restrictions, automatic resumption is unsafe.

Tenant authorization and composite relationships protect ownership. Secrets, emails, and message bodies stay out of ordinary logs. Retention and deletion cover derived records and backup recovery. Deduplication has a defined retention horizon, not an indefinite guarantee.

Before implementation, confirm recipient identity mapping, event replay, messaging eligibility, idempotency scope and lifetime, and delivery-status semantics. No workload or SLO was supplied, so no measured capacity is claimed.

The [validation matrix](EXECUTION.md#8-validation-plan) covers the full flow, competing events, fast replies, timeout races, ambiguous sends, quota contention, cancellation, and backup restoration. These are implementation acceptance criteria, not tests already run. The repository contains no executable service or runtime validation.
