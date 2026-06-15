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
  - Workflow
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
- **Pin shared data contracts by semantics, not just names.** A shared agent
  name or file path is a safe name-only reference. A shared data structure (a
  record both a producer and a consumer lane interpret) is a real interface:
  its field meanings *and edge/sentinel values* go in `CONTRACTS.md`, and the
  seam is tested against one shared fixture before integration.
- **Workers run in the shared tree by default.** Because lanes own disjoint
  files, a worker can write its product files and reach the bus by plain
  relative path with zero collision risk. Use worktree isolation (`isolation:
  worktree` on the worker, plus **absolute paths** + `--add-dir "$RUN_DIR"`)
  **only** when a lane genuinely risks editing files another lane owns — it is
  opt-in, not the default.

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

Workers run in the shared tree by default and reach the bus by relative path.
**Only if you opt a lane into worktree isolation** does the bus become
unreachable by relative path — an isolated worktree does not contain the
git-ignored `.claudetrees/` workspace. In that case (and only then) address the
bus by absolute path and pass `--add-dir "$RUN_DIR"`. Keep `$RUN_DIR` resolved
to an absolute path regardless, so it is available the moment a lane needs it.

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
CONTRACTS.md      # only when a shared data contract crosses a lane seam
```

Instantiate them from the templates under
`.claude/skills/claudetrees/templates/`:

- `IDEA.template.md`        -> `IDEA.md`        (this lane owns this template)
- `FEATURES.template.md`    -> `FEATURES.md`    (owned by F01 idea-splitter)
- `DISPATCH.template.md`    -> `DISPATCH.md`    (this lane owns this template)
- `STATUS.template.md`      -> `STATUS.md`      (this lane owns this template)
- `NEEDS_USER.template.md`  -> `NEEDS_USER.md`  (owned by F03 scribe)
- `MANUAL.template.md`      -> per-lane `MANUAL.md` (owned by F03 scribe)
- `CONTRACTS.template.md`   -> `CONTRACTS.md`   (conductor-owned; create only
  when a data structure crosses a lane seam)

`INTEGRATION.md`, `DECISIONS.md`, and `BLOCKERS.md` have no template — create
them as simple, dated logs. `CONTRACTS.md` is created **only when** the
idea-splitter flags a data structure that one lane produces and another
consumes (see "Pin Shared Contracts" below); a pure name-only-reference run has
no `CONTRACTS.md`.

Ownership and read/write rules for every bus file are documented in
`docs/markdown-bus-protocol.md`. The short version:

- **Conductor** owns and writes the global bus files
  (`STATUS.md`, `DISPATCH.md`, `DECISIONS.md`, `BLOCKERS.md`, `INTEGRATION.md`,
  `CONTRACTS.md`) and seeds `IDEA.md`, `FEATURES.md`, `NEEDS_USER.md`.
- **Workers** write only their own `features/FNN-slug/` files plus their product
  files; they append manual tasks to their local `MANUAL.md` only (the scribe
  folds those into the global `NEEDS_USER.md`). Workers read `CONTRACTS.md` but
  never write it.
- **Scribe** owns the consolidated `NEEDS_USER.md` — it is the sole writer.

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

Invoke the **`claudetrees-idea-splitter`** subagent (via the `Agent` tool) and
ask it to produce 2-5 independent, parallelizable lanes. Pass it the absolute
run directory and the inspected-repo summary. It writes `FEATURES.md` from
`FEATURES.template.md`, and — crucially — flags any **shared data contract**
(a data structure one lane produces and another consumes) as a blocking seam in
a `## Shared Contracts` section, distinct from harmless name-only references.

If `claudetrees-idea-splitter` is unavailable, do the split yourself using the
same contract.

Good lane boundaries: independent user-visible features, low file overlap, clear
done criteria, dependencies stated up front, explicit integration surface.

Bad lane boundaries: "frontend/backend/tests" splits of one feature, the same
core file edited by every worker, vague chores with no user value, or a worker
that only waits on another worker.

After `FEATURES.md` exists, create each `features/FNN-slug/` directory.

### 2b. Pin Shared Contracts (only if a data structure crosses a seam)

A shared **name** (an agent name, a file path) is safe to reference by name
only. A shared **data contract** — a record or value one lane produces and
another interprets — is a real interface, and pinning only its field *names*
(not their *meanings*) is exactly how this project's one integration bug shipped:
one lane used a field's self-reference as a sentinel, the consumer read any
non-null value as a correction, and **both unit suites passed** because each
lane's private fixture defaulted the edge case away.

So, for every shared data contract the splitter flagged:

1. Write it into a conductor-owned **`CONTRACTS.md`** (from
   `CONTRACTS.template.md`). For each field pin its **meaning *and* its
   sentinel / edge-case values** — e.g. *"`supersedes`: id of a **prior,
   distinct** segment this corrects; `null` **or** a value equal to this
   segment's own id means *no correction*."*
2. List **every** producer and consumer lane for the contract, and treat the
   seam as a **blocking dependency** in each involved lane's `FEATURE.md` /
   `PLAN.md` — not a name-only reference.
3. Create **one shared fixture** for the seam (a checked-in file under the run
   dir or repo) that includes the adversarial edge case, and point both the
   producer and consumer lanes at it. A lane may not substitute its own fixture
   for the seam.

If no data structure crosses a seam, skip this step — there is no `CONTRACTS.md`.

### 3. Plan Each Lane

For each lane write `FEATURE.md` and `PLAN.md`. Each `PLAN.md` must include:

- objective
- files to read first (include `CONTRACTS.md` if the lane is on a shared seam)
- tasks
- verification (for a seam lane: run the **shared** fixture from `CONTRACTS.md`,
  adversarial edge case included — not a private fixture)
- manual-task rules (stable IDs like `MAN-F01-001`, appended to the lane's local
  `MANUAL.md` **only**; the scribe consolidates them into `NEEDS_USER.md`)
- result-reporting contract (the `RESULT.md` fields)
- boundaries (which files the lane must NOT touch)

### 4. Dispatch Workers

Launch one `claudetrees-worker` per lane. **Prefer the harness-native paths** —
they work here today; the `claude --bg` CLI launcher (footnote below) often does
not.

**Primary — `Agent` tool subagents.** For each lane, invoke a `claudetrees-worker`
subagent through the `Agent` tool. To run lanes in parallel, send several `Agent`
calls in a **single message**. Pass each worker its lane id and the run directory:

```text
You are implementing feature F01. Run directory: <RUN_DIR>.
Read <RUN_DIR>/IDEA.md, FEATURES.md, DECISIONS.md, CONTRACTS.md (if present),
and features/F01-feature-slug/PLAN.md. Implement only your lane. Update your
features/F01-feature-slug/STATUS.md, MANUAL.md, and RESULT.md. Do not edit the
global bus files.
```

Workers run in the shared tree, so a plain relative path to the bus is fine. Only
if you opted a lane into `isolation: worktree` must you address the bus by
**absolute path** and grant it with `--add-dir "$RUN_DIR"` (see Run Directory).

**Alternative — the `Workflow` tool (deterministic fan-out).** When lanes have an
ordering dependency (e.g. seam consumers wait on producers, `A·B·C·D → E`) or you
want structured returns instead of free-text, orchestrate the workers with the
`Workflow` tool: each `agent()` call runs one `claudetrees-worker` lane, and the
script encodes the dependency graph (`parallel()` for independent lanes,
`pipeline()` for a producer→consumer seam). The structured return of each lane
can stand in for that lane's `RESULT.md`. This is a blessed execution mode — see
"Deterministic execution via the Workflow tool" below.

After each launch (either path), record a row in `DISPATCH.md` (see
`templates/DISPATCH.template.md`). Capture the session id if one is available;
otherwise record `pending lookup`.

Do **not** pass `--dangerously-skip-permissions` unless the user explicitly asks.
Let the project default permission mode govern workers.

> **Footnote — `claude --bg` CLI launch (only if your CLI supports it).** Some
> `claude` CLI versions expose a scriptable `claude --bg --agent claudetrees-worker
> --add-dir "$RUN_DIR" "<prompt>"` launcher; many do not (they manage background
> agents only through the interactive `claude agents` view). Both prior runs found
> `--bg` unavailable and used the harness-native path above. Treat `--bg` as an
> optional convenience, not the primary mechanism. The full syntax matrix is in
> `docs/markdown-bus-protocol.md`.

### 5. Run the Scribe

Run the **`claudetrees-scribe`** subagent through the `Agent` tool to consolidate
every lane's `MANUAL.md` into the single `NEEDS_USER.md`. The simplest reliable
shape is a scribe pass **after** the workers finish (or after each batch); if your
CLI supports a long-lived `claude --bg` agent you may instead keep one running to
re-consolidate continuously. Prompt:

```text
Run directory: <RUN_DIR>. Read every features/*/MANUAL.md, STATUS.md, and
RESULT.md. Update <RUN_DIR>/NEEDS_USER.md: consolidate duplicates (keep all
source ids, comma-separated), preserve source links, keep open tasks actionable.
You are the only writer of NEEDS_USER.md. Do not implement code.
```

The scribe is the **sole writer** of `NEEDS_USER.md`; workers only append to their
own `MANUAL.md`, which removes the concurrent-append race entirely.

### 6. Monitor

The bus **is** the dashboard and is always available: point the user at the
global `STATUS.md` and `DISPATCH.md`, and each `features/FNN-slug/STATUS.md`. If
you dispatched via the `Workflow` tool, its progress view tracks each lane.

If this CLI exposes background-agent commands, they help too (skip any that error
— they depend on CLI support):

```bash
claude agents              # interactive background-agent view
claude attach <session-id>
claude logs <session-id>
claude stop <session-id>
```

### 7. Integrate (under review)

Do **not** auto-merge worker output blindly. When workers report complete:

1. Read every `features/FNN-slug/RESULT.md`.
2. Review `NEEDS_USER.md` and resolve or surface manual tasks.
3. Inspect each worker's branch/worktree and changed files.
4. **Run the seam test (if `CONTRACTS.md` exists).** Before merging any lane on a
   shared seam, run the **shared** fixture from `CONTRACTS.md` — including its
   adversarial edge case — against *both* the producer and consumer lane's code,
   so a passing private fixture cannot hide a semantic disagreement. A seam that
   only passes each lane's own fixture is **not** verified. Block integration of
   the seam until the shared fixture passes end-to-end.
5. Resolve conflicts one lane at a time.
6. Run tests/build.
7. Write the top-level `README.md` and any final wiring (conductor-only).
8. Update `INTEGRATION.md` with what merged, what was skipped, and why (record
   the seam-test result).

## Deterministic Execution via the Workflow Tool

The step-by-step `Agent`-tool dispatch above is the default. For runs with lane
ordering (a producer→consumer seam) or where you want structured, validated
returns, the **`Workflow` tool** is a blessed alternative — it gives
deterministic fan-out (`parallel()` for independent lanes, `pipeline()` for a
seam) and returns each lane's result as data instead of free text.

This mode trades some bus files for structured returns; keep the contract honest:

- Each `agent()` call runs one `claudetrees-worker` lane (`agentType:
  'claudetrees-worker'`), still keyed off its **`FNN`** id so `MAN-FNN-NNN` ids
  stay coherent. Do **not** rename lanes to letters — the manual-task ids key off
  `FNN`.
- A lane's **structured return may stand in for its `RESULT.md`**; you do not have
  to also emit a per-lane `RESULT.md` file when the Workflow return captures the
  same fields (Feature, State, Summary, Files changed, Verification, Risks,
  Manual tasks, Integration notes). The per-lane `MANUAL.md` is still written by
  the worker so the scribe can consolidate `NEEDS_USER.md`.
- `CONTRACTS.md` and the **seam test still apply**: model the seam as a
  `pipeline()` so the consumer lane runs after the producer, and run the shared
  fixture (edge case included) as a verification stage before the result is
  accepted.
- Still record dispatched lanes in `DISPATCH.md` and integrate under review — the
  Workflow tool changes the *launch* mechanism, not the no-auto-merge rule.

Use whichever fits the run; just keep `FNN` numbering, `CONTRACTS.md`, the seam
test, and reviewed integration intact in both modes.

## Final Response

End your conductor turn with:

- the run directory (absolute path)
- the launched feature workers and their lane IDs
- the scribe status
- how to monitor
- the first thing the user should check
