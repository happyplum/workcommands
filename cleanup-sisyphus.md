---
description: Audit and clean the .sisyphus workspace while preserving durable knowledge.
---

You are executing the `/cleanup-sisyphus` command.

Treat `$ARGUMENTS` as the cleanup scope, target subdirectory, or any extra retention constraints.

Operate as a command-first cleanup workflow. Do not stop at “invoke a cleanup skill”; perform the cleanup protocol directly and prove the result.

Required behavior:

1. Resolve the concrete cleanup target. When `$ARGUMENTS` is empty, default to the workspace `.sisyphus` directory.
2. Work only on `.sisyphus` or the execution-artifact area explicitly named by the user.
3. Run the full verified cleanup flow:
   - inventory
   - completion/state verification
   - durable-knowledge deduplication
   - deletion of verified temporary artifacts
   - final proof of the remaining directory state or confirmed removal
4. Preserve only durable guardrails, topology, reference, and contract knowledge. Do not mirror transient task logs into long-term memory.
5. If cleanup exposes reusable knowledge that must be restructured, perform that restructuring as part of this command path instead of leaving an implicit follow-up.
6. If cleanup changes long-term truth, sync the relevant reusable memory and README carriers before finishing.
7. End with a concrete cleanup report: what was inventoried, what was deleted, what was preserved, and what durable knowledge carriers were updated.

Use the existing cleanup and memory-governance skills as reference material when helpful, but the command itself owns the operational flow and success criteria.