# Execution details

Companion to the [design](DESIGN.md). This document specifies implementation rules and validation criteria, not observed behavior of a running service. Quotas and request-level notices apply only when those product extensions are enabled.

## 1. Product policies

### Matching

Normalize with Unicode NFC, a defined case-folding implementation, trimming, and whitespace collapsing. Match phrases at Unicode letter/digit boundaries. The matcher version includes its examples and normalization rules. Languages that need different tokenization require an explicit supported mode.

Process new user comments only. Ignore own replies, reflected outbound events, service events, and edits. Resolve overlapping automation definitions by priority and stable ID, recording suppressed matches. This differs from separate requests, which may enter the conversation queue.

### Requests and input

An equivalent unfinished request has the same conversation, automation, and material version. Search all nonterminal runs, including queued ones, before creating another. Preserve each comment and its association with the chosen run. A status reply has its own action key and a conversation-level rate limit.

Configuration includes total `request_ttl`, `response_timeout`, `max_invalid_prompts`, and a minimum status-message interval. Validate these at publication. The response deadline starts on confirmed acceptance of the email prompt and is bounded by the total lifetime and known platform window. Numerical values need product input.

A message with no email, multiple emails, or unsupported attachments receives bounded clarification. Consume the message and update counters atomically. When an email is accepted, stop scheduling clarifications and cancel pending clarification actions in `ready`, `retry_wait`, or `blocked` in the same transaction that closes the wait. Preserve `in_flight` and `unknown` outcomes for normal finalization or reconciliation; already-started calls cannot be retracted. Internationalized addresses are supported only within the chosen validator’s actual capabilities.

Corrections are accepted while the wait remains open, until the first syntactically valid email is consumed. Updating an already accepted address is outside scope, including after run completion. A separate interaction policy would need to distinguish corrections from input to another request. This does not prevent receiving the material, which is sent by DM.

For queued requests, explain the material and expiry time with a timezone. On expiry, record the reason and, where permitted, send one notice inviting a fresh request. Show the automation owner when the notice itself cannot be sent.

### Material versions and quotas

Quota scope is `(tenant, account, participant, material_version)`. Automations sharing a material version share its quota. Reserve atomically when preparing the delivery action, not while merely queueing a request.

A reservation moves from `reserved` to `accepted` on integration acceptance, or to `released` on proven non-send. Cancelling an action with no in-flight or unknown attempt is proven non-send: cancel it, release its reservation, and decrement reserved usage in one transaction. An unknown send retains the reservation. Reducing a limit does not erase history or retract an in-flight call. New versions get separate accounting without removing opt-outs or blocks.

If `accepted_count + reserved_count >= limit` when valid input is processed, atomically claim the message, store the accepted email, close the wait, cancel pending clarifications, and stop the run with `quota_unavailable`. Create neither a link action nor a reservation. Record the reason and apply normal safe conversation release. This conservative policy applies even when capacity is held by unresolved reservations: it does not automatically retry the input or wait for quota. A later fresh request can be evaluated against then-current capacity.

A pinned material version requires a versioned URL or controlled storage if content must remain reproducible. A mutable third-party URL cannot provide that guarantee.

For a late permanent delivery failure, the proposed policy restores one delivery entitlement exactly once against the original action and opens an issue, without automatically resending. Duplicate or reordered status events must not repeat this adjustment. This policy requires product confirmation. Previously collected email is not reused automatically.

## 2. Records and constraints

The [data model](DATA-MODEL.md) defines all records, relationships, and access paths. The protocols below rely on these constraints:

- Customer-owned records include `tenant_id`; composite foreign keys and account/conversation checks prevent invalid relationships. Timestamps are UTC, but explicit sequences and revisions determine order.
- Event uniqueness uses the source's actual ID scope. `message_claim` separately enforces uniqueness by canonical message ID, including invalid replies, so another event envelope cannot repeat input processing.
- A `request` preserves each matched comment and is unique by canonical comment ID within tenant/account scope. A run's initial request establishes its queue position; merged repeats refer to that same run without another queue slot.
- A partial unique index prevents multiple nonterminal `(conversation, automation, material_version)` runs. The conversation row serializes activation and may remain owned by a terminal run with unresolved sends.
- Each action has exactly one run or request owner. Separate owner/purpose/ordinal unique indexes prevent duplicate logical effects without relying on nullable-key behavior. Retries preserve action ID, recipient, and rendered content.
- Jobs have a checked typed target and unique kind/target/generation identity. They wake state-aware processing rather than defining the scenario or replacing outbound actions.
- Quota reservations are unique by action and update atomically with counters and action outcomes when that extension is enabled.

Do not equate comment-author and DM-sender IDs without verified mapping; block a run whose recipient cannot be established. Keep raw bodies and history out of hot polling queries, clean up in bounded batches, and monitor autovacuum and growth. Partition only when measurements justify it.

## 3. Transactions, ordering, and deadlines

### Ingestion and activation

The source must retry unacknowledged events or provide replay. Without this, loss before our commit cannot be ruled out. Authenticate the source, validate identities, structure, and size, then persist the event and job in one transaction before acknowledging.

When selecting an automation, check its current published revision, lock the conversation, and search all nonterminal runs for an equivalent request. Merge into an active or queued run if found. Otherwise create a new request/run. Commit the request disposition, event-processing status, and scheduled work together.

Use one lock order: automation definition when required, conversation, run/wait, quota/reservation, action. A worker starting with an action ID reads its relationships, acquires locks in that order, and rechecks revisions. A serialization failure before any external call permits retry of the whole transaction.

Claim jobs with a lease and token in a short transaction. Recheck generation and state; commit job completion with the state change and next work. Stale wakeups are no-ops. An in-memory buffer may accelerate execution but cannot be its source of truth.

### Input consumption

Create an `arming` wait before sending the email prompt. The first send attempt establishes the candidate boundary. Confirmed acceptance activates the wait and schedules a stored-DM scan. Do not move the boundary on retry: that could exclude a fast reply.

A proven non-send does not activate the wait. An ambiguous prompt send does not authorize delivering material in response to an arbitrary following DM. Reliable reply-to metadata or platform ordering takes precedence; otherwise correlation remains the documented heuristic.

Under conversation and wait locks, one shared algorithm selects the next unprocessed candidate by ingress sequence. Message jobs, wait activation, and timers all call it. Invalid input is marked processed. When delivery is permitted, valid input is claimed atomically with storing the email, closing the wait, cancelling pending clarifications, reserving quota if enabled, and creating the link action. If quota is unavailable, use the terminal branch in the product policies instead. A consumed message is not handed to the next queued request.

Scan large candidate sets in bounded batches with a durable cursor and continuation job. A timeout must not close a wait before all eligible candidates have been considered.

### Deadline boundary

After locking the conversation row, ingestion assigns its sequence and `ingress_mark_at` using PostgreSQL `clock_timestamp()`. An event qualifies when `ingress_mark_at <= deadline` and its transaction eventually commits. This is deliberately not a commit-before-deadline promise: the mark can precede the deadline while commit follows it.

The timer takes the same lock and sees either the committed event or its absence after rollback. Network arrival time and sender clocks do not define eligibility. Keep ingestion transactions short, time-bounded, and free of external calls. Timer generations prevent an old job from closing a newer wait.

### Releasing conversation ownership

Cancellation, expiry, and stopping cancel pending scenario actions with no in-flight or unknown attempt. Release any reserved quota for those cancelled actions and update counters in the same transaction. An `in_flight` or `unknown` DM retains ownership and relevant reservations even if the run is terminal. Otherwise an old prompt could appear during the next scenario.

When the last unresolved send is finalized, one transaction checks remaining actions, releases ownership, and activates the first eligible queued request. Integration acceptance does not establish actual platform delivery order.

Expiry follows this order: cancel unstarted scenario actions, record the terminal outcome, create at most one optional `terminal_notice`, then evaluate queue advancement. The notice is request-owned with its own short deadline and unique `(request, terminal_notice)` key. It neither resumes the run nor reserves material. Cleanup using the expired run’s `request_ttl` must not cancel or recreate it. Opt-outs, holds, rate limits, eligibility, and ambiguous-send handling still apply. Explicit cancellation or opt-out does not create an expiry notice.

## 4. Sending and recovery

Before a call, recheck deadline, restrictions, eligibility, shared account limits, and reservation. For a clarification, also verify under the normal lock order that its input wait is still open, its generation matches, and no email has been accepted. This check and the transition to `in_flight` are atomic with respect to input consumption; a stale clarification is cancelled without a call. Some platforms require a comment ID for the first private reply rather than a normal recipient ID; the adapter must expose that requirement. Commit an attempt and `in_flight`, call the integration, then commit the outcome, accounting, and continuation. If cancellation happened meanwhile, keep the result but do not create the next scenario step.

A crash after recording `in_flight` but before the call is indistinguishable from a crash after platform acceptance. Lease expiry alone does not permit another send. A worker that lost its claim must not start new attempts, although a call already in progress may finish. A stale token cannot overwrite a newer decision; preserve late evidence for reconciliation.

Retries reuse the idempotency key within its actual scope and retention window. Expired keys do not establish retry safety. Searching history for similar message text is not reliable reconciliation. Without a trustworthy result lookup or idempotency, retain `unknown`.

The normal action path is `ready -> in_flight -> succeeded`. Proven transient rejection moves to `retry_wait`, a repairable non-effect failure to `blocked`, a permanent rejection to `failed`, and ambiguity to `unknown`. Unstarted actions can become `cancelled`. Every retry revalidates conditions.

Retries are bounded, use exponential backoff with jitter, honor `Retry-After`, and respect deadlines. Issues expose the step, cause, attempt count, and next action. Group account-wide access failures rather than alerting for every affected run. Unknown sends remain visible as unresolved work.

Reconciliation finds unprocessed events without jobs, stranded runs, expired claims, unknown sends, overdue waits, and idle conversations with queued requests. It restores jobs through normal processing paths rather than bypassing the send protocol. Monitor the reconciler itself.

Manual recovery is authorized, idempotent, and audited. For `blocked`, verify the repair. For `unknown`, require evidence of send, evidence of non-send, or an explicit exception accepting duplicate risk. Such an exception does not restore a no-duplicates guarantee. Monitor database availability independently through health checks and external signals.

## 5. Scaling and deployment

API, transition, send, and reconciliation roles run from one image. Background roles can share a process with bounded concurrency initially. Scaling does not require sticky conversation ownership on a particular instance.

Database-backed account limits coordinate all workers. Bound connections, claim batches, buffers, and concurrent network calls. Allocate capacity fairly across tenants and accounts. When ingestion cannot accept work, return a proper failure so the source can retry. Keep backlog on disk rather than in an unbounded RAM queue.

Track queue age, time to first action, pending/unknown sends, platform failures and throttling, issue age, expired requests, lock waits, and database resources. Tune indexes, batch sizes, and transactions before adding a broker or sharding. A transport change must preserve the business invariants.

Deployment rules:

- Use a multi-stage OCI build and lockfile. Exclude build tools and caches from the runtime image.
- Use a maintained minimal base pinned by digest and update it regularly. Do not choose Alpine automatically if dependency compatibility suffers.
- Run without root, forward signals correctly, and install no dependencies at startup.
- Separate build, release, and run. Inject environment configuration and secrets at deployment; store customer definitions in the database.
- Readiness checks include dependencies and schema compatibility. Liveness should not cause restart storms during a short database outage.
- On SIGTERM, remove readiness, stop claiming work, and allow a bounded drain. Interrupted sends recover through the same ambiguity protocol.

Run migrations as a separate, locked administrative task. Expand the schema compatibly before removing old fields. Application rollback does not make destructive schema rollback safe.

A run pins `execution_version`. A release retains its handlers or explicitly migrates its state first. Old workers cannot claim jobs for unsupported versions. Readiness, job routing, and queue monitoring must reveal stranded versions.

Disabling an automation stops new requests; cancelling accepted runs is separate. An account emergency hold applies to existing runs too, but cannot retract in-flight calls. Image size, startup time, and throughput have not been measured.

## 6. Database failure and backup restoration

Process restart, primary failure with intact storage, and storage loss need different procedures. One VPS is not high availability. WAL, fsync, and commit policy must support the promised durability. Asynchronous replication does not establish zero loss of acknowledged work on failover.

If acknowledged work must survive primary loss, production may require managed PostgreSQL with synchronous replication, backups, PITR, and tested restore procedures. This costs more than the minimum deployment. RPO, RTO, workload, and budget must be agreed rather than invented.

Do not start workers immediately after restoring an older backup. External sends, opt-outs, holds, or deletions may have happened after the snapshot. Establish a recovery hold outside the rolled-back database through deployment controls, covering both sends and access to restored personal data.

Lifting the hold requires two checks:

1. Reconcile external actions using replay, reliable platform evidence, and still-valid idempotency. Preserve unresolved outcomes as `unknown`.
2. Recover current opt-outs, blocks, emergency holds, and deletions from a non-rolled-back record. Apply deletions before ordinary processing or API access.

The assignment does not establish that such a record exists. If it is absent or incomplete, keep the affected scope closed pending verification and a newly authorized decision supported by sufficient evidence. Absence of a restriction in an old backup is not permission. This is a recovery limit, not an implicit proposal to add another service.

## 7. Security and retention

External JSON does not supply a trusted tenant/account identity. Authenticate through the existing integration or a verified delivery channel. Check ownership and roles for definition changes, history, recovery, and email export. Row-level security may supplement authorization and composite foreign keys, not replace them.

Keep secrets out of payloads and logs. Encrypt transport and restrict database/backup access appropriately. Ordinary logs contain correlation IDs, not email addresses or message bodies.

Bound event sizes, strings, and matching rules. Never execute message content. Sending a material link does not require arbitrary server-side URL fetching. Recheck opt-outs, holds, and platform eligibility before every send. A full operator chat product is out of scope.

Retain raw messages and emails only for a defined purpose and period. Minimal deduplication keys and quota counters also identify people and need justified retention. Deletion covers source data, run variables, history, diagnostic evidence, and the backup lifecycle. After irreversible deletion, indefinite deduplication cannot be promised. No jurisdiction-specific compliance is claimed without its requirements.

## 8. Validation plan

These are acceptance criteria for implementation. No executable tests have been run.

| Scenario | Expected result |
| --- | --- |
| Happy path | Public reply, email prompt, accepted email, link; history matches actions |
| Duplicate source-event delivery and competing workers | One accepted source event and one logical action per effect |
| One comment wrapped in different event IDs | One request by canonical comment ID; no repeated execution |
| A active, B queued, another comment for B | Merge into B without another run or uniqueness-retry loop |
| One DM wrapped in different event IDs | One claim by canonical message ID |
| M1 persisted first, M2 worker runs first | Shared consumer considers M1 first |
| Invalid/valid reply races | Bounded clarification, one link action, outcomes retained for in-flight sends |
| Valid email accepted while a clarification is ready or waiting to retry | Wait closes and unstarted clarifications are cancelled atomically; the sender cannot start a stale clarification |
| Another email after acceptance | No change to the accepted address and no second link action for that run |
| Reply precedes prompt-success commit | Activation scan finds the stored reply |
| Reply races timeout | One outcome under the shared lock and ingress boundary; stale timer cannot close a newer wait |
| Ingress mark before deadline, commit after | Eligible after commit; absent if rolled back |
| Different materials or a new version | Durable queue, explicit context, one active scenario |
| Expiry and `terminal_notice` | Old actions cancelled; notice has its own deadline and respects restrictions |
| Fresh request after completion | New run if permitted; no replay of old actions |
| Competing for last quota slot | One reservation; the denied run consumes its input and stops with `quota_unavailable`, without a link action or retry loop |
| Cancel or expire a reserved link action before sending | Action cancellation, reservation release, and counter update commit together; in-flight/unknown reservations remain held |
| Crashes around every commit and network call | No lost acknowledged work or blind retries of unknown sends within the stated durability model |
| Lost lease and late result | Stale token cannot overwrite decisions; evidence remains available |
| Cancel A during prompt send while B waits | B cannot start its DM exchange until A’s send is resolved |
| Repair access | Resume at the blocked action |
| Database outage, restart, restore | Retry/reconciliation follows the specific failure model; no unsupported RPO=0 claim |
| Backup, then opt-out/hold/delete, then restore | Hold until current restrictions and deletions are recovered |
| Failed configuration edit or publication | Previous published revision remains current; unpublished settings never match |
| Publish a new revision while a matching run exists | New run uses the new revision; merged repeat retains the existing run's settings |
| Rolling release | Every run has a supported execution version |
| Foreign tenant ID, own messages, oversized payload | No cross-tenant access or self-trigger loops; bounded resource use |
| One tenant’s traffic spike | Other tenants retain capacity; memory and concurrency stay bounded |

Unit tests cover transitions, matching, and email parsing. Integration tests need real PostgreSQL and multiple workers. Fault injection exercises commit/send boundaries. Run the full flow through a controllable adapter, then verify the real integration contracts. Compilation and an always-successful mock cannot establish these properties.
