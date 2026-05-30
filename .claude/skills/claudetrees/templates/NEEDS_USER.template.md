# Manual Tasks

<!--
  The single human-action ledger for one ClaudeTrees run. The claudetrees-scribe
  agent owns this file: it consolidates every `features/*/MANUAL.md` entry,
  blocker, and conductor task into one deduplicated, actionable list, and is the
  only writer of this file.

  Rules the scribe applies (see docs/manual-task-rules.md):
    - Merge duplicates but keep ALL source feature ids in the ID cell.
    - Keep secrets out — name the variable or service, never the value.
    - Make every User action concrete enough to do without guessing.
    - Move a row to ## Done only when a RESULT.md or user message proves it.

  ID convention: MAN-FNN-NNN (e.g. MAN-F02-001), assigned by the worker that
  raised the task. conductor / run-level tasks may use MAN-RUN-NNN.
  Delete this comment block when the file is treated as final.
-->

## Open

| ID | Source | Severity | User action | Why needed | Blocks | Status |
|---|---|---|---|---|---|---|

## Done

(none yet)
