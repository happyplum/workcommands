---
description: Repair, normalize, and finalize a reviewed or imported execution plan before downstream execution.
---

You are executing the `/repair-plan` command.

Treat `$ARGUMENTS` as the target plan path, plan identifier, or the user's focused repair request.

Operate as a command-first governance workflow. Do not reduce this command to “just invoke a skill and stop”. Produce the repair outcome directly.

Required behavior:

1. Resolve the concrete target plan from `$ARGUMENTS`, the current conversation, or the current workspace. Ask one focused question only if no recoverable target exists.
2. Audit the plan against local execution governance, including routing shape, task granularity, verification closure, execution-skill requirements, and imported upstream runtime labels.
3. Classify the plan as exactly one of:
   - `direct-execute`
   - `normalize-before-execute`
   - `repair-before-execute`
4. If the plan only needs local-shape alignment, normalize it in place. If it fails structural gates, repair it deterministically instead of handing off a vague diagnosis.
5. When upstream runtime labels, weak routing shape, missing verification closure, or missing local constraints appear, convert them into local execution-ready form before any downstream execution handoff.
6. If files, README truth, plan docs, or durable memory must change to keep the repaired plan authoritative, update them in the same command run.
7. End with a concrete verdict: final classification, repaired sections, remaining blockers if any, and whether the plan is now execution-ready.

Use the existing governance skills and memories as repo knowledge sources when useful, but the command itself owns the operational flow and completion standard.