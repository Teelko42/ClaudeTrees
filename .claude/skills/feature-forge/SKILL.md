---
description: Split a project idea into feature lanes, launch background Claude Code workers, and coordinate via markdown mailboxes.
disable-model-invocation: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Agent
  - Task
---

# Feature Forge

You are the Feature Forge conductor. The user gives you a project idea in
`$ARGUMENTS`. You turn it into feature lanes, create a markdown coordination
workspace, launch multiple background Claude Code sessions, and keep a manual
tasks ledger for the user.

## Non-Negotiables

- Use markdown files as the coordination layer.
- Keep feature lanes independent enough for parallel work.
- Do not let workers edit the same files unless the lane explicitly depends on
  another lane.
- Every worker must write status, blockers, manual tasks, and final results.
- Do not hide manual work. Anything requiring the user, credentials, accounts,
  product decisions, payment details, DNS, app-store setup, external approvals,
  secrets, or package legitimacy checks goes into `NEEDS_USER.md`.
- Prefer two or three workers for a first run. More workers only if feature
  boundaries are very clean.

## Input

Project idea:

```text
$ARGUMENTS
```

If the idea is missing or too vague to split, ask up to five short clarifying
questions before creating files. Otherwise continue.

## Run Directory

Create a run directory at:

```text
.feature-forge/runs/<YYYYMMDD-HHMMSS>-<short-slug>/
```

Use an absolute path for worker prompts:

```bash
RUN_DIR="$(pwd)/.feature-forge/runs/<run-id>"
```

Background Claude sessions may move into isolated worktrees before editing code,
so the shared markdown bus must be addressed by absolute path and passed with
`--add-dir "$RUN_DIR"`.

## File Protocol

Create these files first:

```text
IDEA.md
FEATURES.md
DISPATCH.md
STATUS.md
INTEGRATION.md
NEEDS_USER.md
DECISIONS.md
BLOCKERS.md
```

Use this structure:

### IDEA.md

- Raw user idea
- Assumptions
- Constraints
- Non-goals

### FEATURES.md

One section per feature:

```markdown
## F01: Feature Name

- Goal:
- User value:
- Files likely touched:
- Dependencies:
- Out of scope:
- Done when:
```

### DISPATCH.md

Track every background session:

```markdown
| Feature | Worker name | Session id | Status | Worktree/branch | Last update |
|---|---|---|---|---|---|
```

### STATUS.md

Short overall status:

```markdown
## Overall

- State: planning | dispatched | integrating | blocked | complete
- Active workers:
- Completed workers:
- Needs user:

## Timeline
```

### NEEDS_USER.md

The manual-task ledger. Keep this file readable and current:

```markdown
# Manual Tasks

## Open

| ID | Source | Severity | User action | Why needed | Blocks | Status |
|---|---|---|---|---|---|---|

## Done
```

### Feature Directories

For each feature lane, create:

```text
features/FNN-slug/
  FEATURE.md
  PLAN.md
  STATUS.md
  NOTES.md
  MANUAL.md
  RESULT.md
```

## Process

### 1. Inspect Repo

Use `Read`, `Glob`, `Grep`, and `git status --short` to understand:

- stack
- app entry points
- tests
- package manager
- obvious architecture boundaries
- uncommitted user changes

Do not overwrite user work.

### 2. Split Idea

Use the `feature-slicer` subagent if available. Ask it to create 2-5 feature
lanes that can be worked independently. If the subagent is unavailable, do the
split yourself.

Good lane boundaries:

- independent user-visible features
- low file overlap
- clear done criteria
- dependencies stated up front
- integration surface explicit

Bad lane boundaries:

- "frontend", "backend", "tests" as separate lanes for one feature
- same core file edited by every worker
- vague platform chores with no user-facing value
- a worker that only waits on another worker

Write the split to `FEATURES.md`, then create each feature directory.

### 3. Plan Each Feature

For each feature, write `FEATURE.md` and `PLAN.md`. Plans must include:

- objective
- files to read first
- tasks
- verification
- manual-task rules
- result-reporting contract

Each worker must append any user-required action to both:

- its local `features/FNN-slug/MANUAL.md`
- the global `NEEDS_USER.md`

Workers must use stable IDs like `MAN-F01-001`.

### 4. Launch Background Workers

For each feature lane, launch a background Claude Code session.

Use the project subagent as the main agent:

```bash
claude --bg \
  --agent feature-worker \
  --name "forge-F01-feature-slug" \
  --add-dir "$RUN_DIR" \
  "You are implementing feature F01. Shared run directory: $RUN_DIR. Read $RUN_DIR/IDEA.md, $RUN_DIR/FEATURES.md, and $RUN_DIR/features/F01-feature-slug/PLAN.md. Implement only your lane. Update STATUS.md, MANUAL.md, RESULT.md, and NEEDS_USER.md as instructed."
```

After each launch, record the command output in `DISPATCH.md`. If the command
prints a session id, capture it. If not, record `pending lookup`.

Do not pass `--dangerously-skip-permissions` unless the user explicitly asks.
Let the project default permission mode govern background workers.

### 5. Launch Manual Task Scribe

Launch a separate background session to consolidate user tasks:

```bash
claude --bg \
  --agent manual-task-scribe \
  --name "forge-manual-tasks" \
  --add-dir "$RUN_DIR" \
  "Shared run directory: $RUN_DIR. Monitor the feature MANUAL.md, STATUS.md, and RESULT.md files until all workers are complete, blocked, or failed. Update $RUN_DIR/NEEDS_USER.md after each pass. Poll at a reasonable interval; do not implement code. Consolidate duplicates, preserve source links, and keep open tasks actionable."
```

If background launch is unavailable, run `manual-task-scribe` as a normal
subagent after workers finish.

### 6. Monitor

Tell the user how to monitor:

```bash
claude agents --cwd .
```

Also mention:

```bash
claude attach <session-id>
claude logs <session-id>
claude stop <session-id>
```

### 7. Integrate

Do not auto-merge all worker outputs blindly. When workers complete:

1. Read every `RESULT.md`.
2. Check `NEEDS_USER.md`.
3. Inspect worker branches/worktrees.
4. Resolve conflicts one feature at a time.
5. Run tests/build.
6. Update `INTEGRATION.md`.

## Final Response

End with:

- run directory
- launched feature workers
- manual-task scribe status
- how to monitor
- first thing the user should check
