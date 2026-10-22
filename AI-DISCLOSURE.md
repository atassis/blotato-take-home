# AI disclosure

## My contribution

I set the engineering priorities: explicit states and invariants, maintainable scenario code, and operational simplicity. I chose explicit trigger phrases and code-based behavior rather than a configurable workflow language. I also proposed repeat-request handling, per-material-version quotas with preserved history, sequential conversations, and informational activity signals. These product extensions are separated from the required flow.

## AI assistance

I used AI for research, exploring alternatives, and developing the written design. It proposed the concrete stack and transaction protocols and drafted the architecture, data model, and documentation.

I also requested a separate AI answer using only the original assignment, without our discussion or design choices, as a baseline. Comparing it with this proposal helped clarify the trade-offs: scenario code versus configurable workflows, and preserving distinct requests in a queue rather than suppressing them. The value was in making those choices explicit, not treating another generated answer as validation.

Additional AI review passes examined concurrency, user experience, operations, security, and scope. They surfaced issues in queued-request merging, reply ordering, and recovery that were addressed in the design.
