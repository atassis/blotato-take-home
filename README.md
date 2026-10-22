# Blotato Take Home

A backend design for automations that start with a social media comment and continue in direct messages.

## Summary

For the example automation, a comment containing `pricing` on any post of the connected Instagram account starts this sequence:

1. Reply to the comment with “Sent you a DM!”
2. Send a DM asking for an email address.
3. Persist the wait for a reply.
4. Accept a valid email address and send the link in another DM.

I propose a TypeScript application backed by PostgreSQL. The existing social integrations receive events and send messages. This service owns automation definitions, durable execution, message correlation, and recovery.

Scenario behavior lives in ordinary, versioned code. Customers configure supported triggers, messages, materials, and timeouts, not arbitrary workflows. This keeps the first version small enough to understand and operate while allowing new scenarios to reuse the execution mechanisms.

PostgreSQL stores incoming events, scenario state, jobs, and outbound actions. Each state transition commits with its next piece of work. API and worker processes can scale independently without keeping essential state in memory. A separate broker is not required initially.

The key reliability boundary is external sending. A stable action ID prevents duplicate logical work inside the service. Avoiding duplicate platform messages after an ambiguous response also requires integration-level idempotency or reliable reconciliation. Without either, the action remains unresolved rather than being retried blindly.

## Read the design

- **[Design](docs/DESIGN.md):** scope, architecture, data model, execution, an end-to-end example, and trade-offs.
- **[Data model](docs/DATA-MODEL.md):** Mermaid relationship diagrams and each record's purpose, fields, lifecycle, and constraints.
- **[Execution details](docs/EXECUTION.md):** transaction boundaries, ordering, recovery, deployment, security, and validation scenarios.
- **[AI disclosure](AI-DISCLOSURE.md):** my contribution and the assistance used to develop this proposal.

The main design covers the requested flow. Material-version quotas, repeat-request policies, and administrative activity signals are explicitly separated as product extensions, not prerequisites for that flow.

## Status

This is a design submission, not a running application. There are no setup commands, benchmarks, or executable tests. The proposed guarantees depend on implementation and verification of the integration contracts identified in the design.

## Easter Egg =)

I used to write functional Typesctipt with Ramda
