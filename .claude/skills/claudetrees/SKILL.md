---
description: Split a project idea into independent feature lanes, launch background Claude Code workers, and coordinate everything through markdown bus files.
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

# ClaudeTrees

You are the **ClaudeTrees conductor**. The user gives you a project idea in
`$ARGUMENTS`. You turn it into independent feature lanes, create a markdown
coordination workspace (the "bus"), launch multiple background Claude Code
workers, run a scribe to keep a manual-task ledger, and integrate the results
under review.

ClaudeTrees is a sibling to the `feature-forge` skill. It reuses the same proven
shape: one conductor skill, three subagents, markdown templates, and docs.

## Non-Negotiables

- Markdown files are the **only** coordination layer. No database, no server.
- Keep feature lanes independent enough for parallel work. Disjoint file
  ownership per lane is the goal.
- Do not let two workers edit the same file unless a lane explicitly depends on
  another lane (state the dependency up front in `FEATURES.md`).
- Every worker writes status, blockers, manual tasks, and a final result.
- Do not hide manual work. Anything requiring the user — credentials, accounts,
  product decisions, payment details, DNS, OAuth, app-store setup, external
  approvals, secrets, or package-legitimacy checks — goes into `NEEDS_USER.md`
  (via the scribe and the workers' `MANUAL.md` files).
- **Never auto-merge worker output blindly.** Integration is a reviewed step.
- Prefer two or three workers for a first run. Add more only if lane boundaries
  are very clean.
- Use **absolute paths** in every worker prompt and pass the run directory with
  `--add-dir` so background sessions (which may run in isolated worktrees) can
  still read and write the shared bus.

## Input

Project idea:

```text
$ARGUMENTS
```

If the idea is missing or too vague to split, ask up to five short clarifying
questions before creating any files. Otherwise continue.

## Run Directory

Create a run directory at:

```text
.claudetrees/runs/<YYYYMMDD-HHMMSS>-<short-slug>/
```

This is the ClaudeTrees convention (the `.claudetrees/` workspace is the sibling
of `feature-forge`'s `.feature-forge/`). Keep it consistent for the whole run.
The directory is orchestration scaffolding and should be git-ignored; the
*product* you build lives under the repo's normal source tree and is committed
at integration.

Resolve the run directory to an **absolute path** and reuse it in every worker
prompt:

```powershell
# Windows PowerShell
$RUN_DIR = Join-Path (Get-Location) ".claudetrees\runs\<run-id>"
```

```bash
# bash fallback
RUN_DIR="$(pwd)/.claudetrees/runs/<run-id>"
```

Background Claude sessions may move into isolated worktrees before editing code,
so the shared markdown bus must be addressed by absolute path and passed with
`--add-dir "$RUN_DIR"`.

## File Protocol

Create these **global bus files** at the run-directory root first:

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

Instantiate them from the templates under
`.claude/skills/claudetrees/templates/`:

- `IDEA.template.md`        -> `IDEA.md`        (this lane owns this template)
- `FEATURES.template.md`    -> `FEATURES.md`    (owned by F01 idea-splitter)
- `DISPATCH.template.md`    -> `DISPATCH.md`    (this lane owns this template)
- `STATUS.template.md`      -> `STATUS.md`      (this lane owns this template)
- `NEEDS_USER.template.md`  -> `NEEDS_USER.md`  (owned by F03 scribe)
- `MANUAL.template.md`      -> per-lane `MANUAL.md` (owned by F03 scribe)

`INTEGRATION.md`, `DECISIONS.md`, and `BLOCKERS.md` have no template — create
them as simple, dated logs.

Ownership and read/write rules for every bus file are documented in
`docs/markdown-bus-protocol.md`. The short version:

- **Conductor** owns and writes the global bus files
  (`STATUS.md`, `DISPATCH.md`, `DECISIONS.md`, `BLOCKERS.md`, `INTEGRATION.md`)
  and seeds `IDEA.md`, `FEATURES.md`, `NEEDS_USER.md`.
- **Workers** write only their own `features/FNN-slug/` files plus their product
  files; they append manual tasks to their local `MANUAL.md` (the scribe folds
  those into the global `NEEDS_USER.md`).
- **Scribe** owns the consolidated `NEEDS_USER.md`.

### Feature Directories

For each lane create:

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

- stack and language
- app entry points
- tests and how they run
- package manager
- architecture boundaries
- uncommitted user changes (never overwrite user work)

### 2. Slice the Idea into Lanes

Invoke the **`claudetrees-idea-splitter`** subagent (via the `Agent`/`Task`
tool) and ask it to produce 2-5 independent, parallelizable lanes. Pass it the
absolute run directory and the inspected-repo summary. It writes `FEATURES.md`
from `FEATURES.template.md`.

If `claudetrees-idea-splitter` is unavailable, do the split yourself using the
same contract.

Good lane boundaries: independent user-visible features, low file overlap, clear
done criteria, dependencies stated up front, explicit integration surface.

Bad lane boundaries: "frontend/backend/tests" splits of one feature, the same
core file edited by every worker, vague chores with no user value, or a worker
that only waits on another worker.

After `FEATURES.md` exists, create each `features/FNN-slug/` directory.

### 3. Plan Each Lane

For each lane write `FEATURE.md` and `PLAN.md`. Each `PLAN.md` must include:

- objective
- files to read first
- tasks
- verification
- manual-task rules (stable IDs like `MAN-F01-001`, appended to both the local
  `MANUAL.md` and global `NEEDS_USER.md`)
- result-reporting contract (the `RESULT.md` fields)
- boundaries (which files the lane must NOT touch)

### 4. Dispatch Workers in the Background

For each lane, launch a background Claude Code session whose main agent is
**`claudetrees-worker`**. Use absolute paths and `--add-dir "$RUN_DIR"`.

```powershell
# Windows PowerShell
claude --bg `
  --agent claudetrees-worker `
  --name "ct-F01-feature-slug" `
  --add-dir "$RUN_DIR" `
  "You are implementing feature F01. Shared run directory: $RUN_DIR. Read $RUN_DIR\IDEA.md, $RUN_DIR\FEATURES.md, $RUN_DIR\DECISIONS.md, and $RUN_DIR\features\F01-feature-slug\PLAN.md. Implement only your lane. Update your features\F01-feature-slug\STATUS.md, MANUAL.md, and RESULT.md as instructed. Do not edit global bus files."
```

```bash
# bash fallback
claude --bg \
  --agent claudetrees-worker \
  --name "ct-F01-feature-slug" \
  --add-dir "$RUN_DIR" \
  "You are implementing feature F01. Shared run directory: $RUN_DIR. Read $RUN_DIR/IDEA.md, $RUN_DIR/FEATURES.md, $RUN_DIR/DECISIONS.md, and $RUN_DIR/features/F01-feature-slug/PLAN.md. Implement only your lane. Update your features/F01-feature-slug/STATUS.md, MANUAL.md, and RESULT.md as instructed. Do not edit global bus files."
```

After each launch, record a row in `DISPATCH.md` (see
`templates/DISPATCH.template.md`). Capture the session id if the command prints
one; otherwise record `pending lookup`.

Do **not** pass `--dangerously-skip-permissions` unless the user explicitly asks.
Let the project default permission mode govern background workers.

**Caveat — no `--bg` flag:** some `claude` CLI versions have no `--bg` flag and
manage background agents only through the interactive `claude agents` view, not
a scriptable one-shot launcher. If `claude --bg ...` is not supported on this
machine, fall back to launching each worker as a **foreground subagent** via the
`Agent`/`Task` tool (`claudetrees-worker`), one lane at a time or as the harness
allows, and note the limitation as a manual task. The full launch-syntax matrix
and this caveat are documented in `docs/markdown-bus-protocol.md`.

### 5. Launch the Scribe

Launch the **`claudetrees-scribe`** subagent to consolidate manual tasks into
`NEEDS_USER.md`:

```bash
claude --bg \
  --agent claudetrees-scribe \
  --name "ct-scribe" \
  --add-dir "$RUN_DIR" \
  "Shared run directory: $RUN_DIR. Monitor every features/*/MANUAL.md, STATUS.md, and RESULT.md until all workers are complete, blocked, or failed. After each pass, update $RUN_DIR/NEEDS_USER.md: consolidate duplicates, preserve source links, keep open tasks actionable. Do not implement code."
```

If background launch is unavailable, run `claudetrees-scribe` as a normal
foreground subagent after the workers finish.

### 6. Monitor

Tell the user how to watch progress:

```bash
claude agents              # interactive background-agent view
claude attach <session-id>
claude logs <session-id>
claude stop <session-id>
```

Also point them at the bus: tail `STATUS.md`, `DISPATCH.md`, and each
`features/FNN-slug/STATUS.md`.

### 7. Integrate (under review)

Do **not** auto-merge worker output blindly. When workers report complete:

1. Read every `features/FNN-slug/RESULT.md`.
2. Review `NEEDS_USER.md` and resolve or surface manual tasks.
3. Inspect each worker's branch/worktree and changed files.
4. Resolve conflicts one lane at a time.
5. Run tests/build.
6. Write the top-level `README.md` and any final wiring (conductor-only).
7. Update `INTEGRATION.md` with what merged, what was skipped, and why.

## Final Response

End your conductor turn with:

- the run directory (absolute path)
- the launched feature workers and their lane IDs
- the scribe status
- how to monitor
- the first thing the user should check
