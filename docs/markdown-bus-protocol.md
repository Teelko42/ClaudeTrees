# ClaudeTrees Markdown Bus Protocol

ClaudeTrees coordinates a conductor and several background workers using nothing
but markdown files in a shared **run directory**. There is no database and no
server — the filesystem is the message bus. This document defines every bus
file, who reads and writes it, and how workers reach the bus — by relative path
in the shared tree by default, or by absolute path when a lane opts into an
isolated worktree.

## Run Directory

Each run gets its own directory:

```text
.claudetrees/runs/<YYYYMMDD-HHMMSS>-<short-slug>/
```

It holds the **global bus files** at its root and one subdirectory per feature
lane under `features/`. The directory is orchestration scaffolding and is
git-ignored; the *product* the workers build lives in the repo's normal source
tree and is committed by the conductor at integration.

```text
.claudetrees/runs/<run-id>/
  IDEA.md
  FEATURES.md
  DISPATCH.md
  STATUS.md
  INTEGRATION.md
  NEEDS_USER.md
  DECISIONS.md
  BLOCKERS.md
  features/
    F01-slug/
      FEATURE.md  PLAN.md  STATUS.md  NOTES.md  MANUAL.md  RESULT.md
    F02-slug/
      ...
```

## Reaching the bus: shared tree by default, worktree is opt-in

Workers run in the **shared working tree by default** (the `claudetrees-worker`
agent does *not* set `isolation: worktree`). Because lanes own disjoint files,
a worker writes its product files and reaches the bus by a plain **relative
path** with zero collision risk. This is the blessed practice — both prior runs
disabled worktree isolation precisely because the lanes own disjoint directories.

**Opt a lane into worktree isolation only** when it genuinely risks editing files
another lane owns. A worktree does **not** contain the git-ignored `.claudetrees/`
workspace, so the moment you opt in, the bus is no longer reachable by relative
path. Two rules then apply (and only then):

1. **Address bus files by absolute path** in the worker prompt
   (`$RUN_DIR/IDEA.md`, never `./IDEA.md`).
2. **Pass `--add-dir "$RUN_DIR"`** when launching the session so the run
   directory is granted regardless of the worktree.

The conductor resolves `$RUN_DIR` to an absolute path once either way, so it is
ready the moment a lane needs it.

## Bus Files

### Global bus files (run-directory root)

| File | Writer(s) | Reader(s) | Purpose |
|---|---|---|---|
| `IDEA.md` | conductor (seeds from `$ARGUMENTS`) | all workers, scribe | Raw idea, product, audience, assumptions, constraints, non-goals. |
| `FEATURES.md` | `claudetrees-idea-splitter` (or conductor) | all workers, scribe | One section per lane: goal, value, files, dependencies, out-of-scope, done-when. |
| `DISPATCH.md` | conductor | conductor, user | One row per background session: Feature, Worker name, Session id, Status, Worktree/branch, Last update. |
| `STATUS.md` | conductor | conductor, user | Overall run state + dated timeline. Workers do **not** edit this. |
| `INTEGRATION.md` | conductor | conductor, user | What merged, what was skipped, conflicts resolved, build/test + seam-test results. |
| `NEEDS_USER.md` | `claudetrees-scribe` (**sole** writer) | user, conductor | The single deduplicated manual-task ledger. Workers do **not** write it — they append to their own `MANUAL.md`. |
| `DECISIONS.md` | conductor | all workers | Default decisions the conductor made; overridable by the user. Read-only for workers. |
| `BLOCKERS.md` | conductor | conductor | Cross-lane blockers. Workers surface blockers in their own files; the conductor folds them in to avoid races. |
| `CONTRACTS.md` | conductor | producer + consumer workers | Per-field meaning **and sentinel/edge-case values** of any data structure that crosses a lane seam. Created only when such a seam exists. Read-only for workers. |

### Per-feature files (`features/FNN-slug/`)

| File | Writer | Reader(s) | Purpose |
|---|---|---|---|
| `FEATURE.md` | conductor | the lane's worker | Lane scope: goal, owned files, dependencies, out-of-scope, done-when. |
| `PLAN.md` | conductor | the lane's worker | Objective, files to read first, tasks, verification, manual-task rules, result contract, boundaries. |
| `STATUS.md` | the lane's worker | conductor, scribe | Per-lane state: started/implementing/verifying/blocked/complete/failed. |
| `NOTES.md` | the lane's worker | conductor | Scratchpad: decisions, shared-file justifications, cross-lane dependencies discovered. |
| `MANUAL.md` | the lane's worker | scribe | Manual tasks raised by this lane (`MAN-FNN-NNN`); the scribe folds these into `NEEDS_USER.md`. |
| `RESULT.md` | the lane's worker | conductor | Final report: Feature, State, Summary, Files changed, Verification run, Commits, Remaining risks, Manual tasks, Integration notes. |

### Write-ownership summary (avoids races)

- **Conductor** writes all global bus files except `NEEDS_USER.md`, and seeds
  `IDEA.md`/`FEATURES.md`. It also owns `CONTRACTS.md` when a seam needs one.
- **Workers** write only their own `features/FNN-slug/` files and their lane's
  product files. They append manual tasks to their **own** `MANUAL.md` only, and
  never write any global bus file (no global `NEEDS_USER.md` append — that was a
  concurrent-write race; one private writer per file is the whole point).
- **Scribe** is the **sole** writer of `NEEDS_USER.md` — it consolidates every
  lane's `MANUAL.md` into it (dedup, source links, keeping open tasks
  actionable). Because only the scribe writes that file, there is no append race.

## Manual-Task IDs

Manual tasks use stable IDs `MAN-FNN-NNN` (lane number, then a zero-padded
sequence within that lane, e.g. `MAN-F02-001`). A worker appends the row to its
local `MANUAL.md` **only**; the scribe reads every `MANUAL.md` and consolidates
into `NEEDS_USER.md`, keeping all merged source ids comma-separated. Row format:

```markdown
| MAN-F02-001 | F02 | low | Confirm local `claude` CLI background-launch syntax | Background dispatch command must be verified on this machine | Does not block (foreground fallback exists) | open |
```

## Shared Data Contracts and Seam Tests

The bus pins shared **names** for free (an agent name or file path written into
`FEATURES.md` before fan-out is a non-blocking reference). It does **not**
automatically pin shared **semantics**, and that gap is the root cause of this
project's one real integration defect: two lanes agreed on a field's *name*
(`supersedes`) but not its *meaning*. One lane used a self-reference
(`supersedes === segment_id`) as a "self-finalize" sentinel; the other read any
non-null value as "this corrects a prior segment" and produced no output. Both
lanes' unit suites passed because each private fixture defaulted the edge case
away — a false green that only surfaced at integration.

A **shared data contract** — a record or value one lane produces and another
interprets — is therefore a real interface, not a name-only reference, and gets
three things:

1. **`CONTRACTS.md`** (conductor-owned, from `CONTRACTS.template.md`): for every
   shared field, its meaning **and its sentinel / edge-case values** (what
   `null`, empty, a self-reference, or an out-of-range value means). This is the
   single source of truth both lanes implement against.
2. **A blocking seam, not a name-only reference.** Every producer and consumer
   lane is listed, and the dependency is declared in each lane's `FEATURE.md` /
   `PLAN.md` as blocking (the consumer integrates after the producer).
3. **One shared fixture, including the adversarial edge case.** Both lanes test
   against the *same* fixture so neither can dodge the disagreement with a
   private default. At integration the conductor runs this fixture against both
   the producer's and the consumer's code (the **seam test**); a seam that only
   passes each lane's own fixture is not verified.

If no data structure crosses a seam, there is no `CONTRACTS.md` — name-only
references remain non-blocking and need none of this.

## Launching Workers

The conductor launches one `claudetrees-worker` per lane, plus one
`claudetrees-scribe`. **Lead with the harness-native paths** — they work here
today. The `claude --bg` CLI launcher is an optional convenience that many CLI
versions do not support (see the footnote), so it is *not* the primary path.

### Primary — the `Agent` tool

Invoke each `claudetrees-worker` as a subagent through the **`Agent`** tool. To
run independent lanes in parallel, send several `Agent` calls in one message.
Workers run in the shared tree, so the bus is reachable by relative path:

```text
You are implementing feature F02. Run directory: <RUN_DIR>.
Read IDEA.md, FEATURES.md, DECISIONS.md, CONTRACTS.md (if present), and
features/F02-worker-dispatcher/PLAN.md. Implement only your lane. Update your
features/F02-worker-dispatcher/STATUS.md, MANUAL.md, and RESULT.md. Do not edit
the global bus files.
```

Run `claudetrees-scribe` the same way (after the workers, or after each batch):

```text
Run directory: <RUN_DIR>. Read every features/*/MANUAL.md, STATUS.md, and
RESULT.md. Update NEEDS_USER.md: consolidate duplicates (keep all source ids,
comma-separated), preserve source links, keep open tasks actionable. You are the
only writer of NEEDS_USER.md. Do not implement code.
```

(If you opted a lane into `isolation: worktree`, address the bus by absolute
path — `$RUN_DIR/IDEA.md` — and add `--add-dir "$RUN_DIR"`; see the opt-in rule
above.)

### Alternative — the `Workflow` tool (deterministic fan-out)

For lane ordering (a producer→consumer seam) or structured returns, drive the
workers with the **`Workflow`** tool: one `agent()` call per lane with
`agentType: 'claudetrees-worker'`, `parallel()` for independent lanes and
`pipeline()` for a seam. Each lane's structured return can stand in for its
`RESULT.md`; the seam fixture runs as a verification stage. `FNN` numbering,
`CONTRACTS.md`, the seam test, and reviewed integration all still apply.

## Footnote — `claude --bg` CLI launch (only if your CLI supports it)

Some `claude` CLI versions expose a scriptable background launcher:

```bash
RUN_DIR="$(pwd)/.claudetrees/runs/<run-id>"
claude --bg --agent claudetrees-worker --name "ct-F02-worker-dispatcher" \
  --add-dir "$RUN_DIR" "<the worker prompt above, with absolute $RUN_DIR paths>"
```

Many versions do **not** — on those, `claude agents` is only an *interactive*
view, not a one-shot launcher, so the command above will not work verbatim. Both
prior runs found `--bg` unavailable and used the `Agent`-tool path. If you want
`--bg` on this machine, verify the exact syntax first and track it as a manual
task (e.g. `MAN-F02-001`); otherwise just use the harness-native paths above —
the bus protocol is identical, only the launch mechanism differs.

## Monitoring

The bus **is** the dashboard and is always available: tail the global
`STATUS.md` and `DISPATCH.md`, and each `features/FNN-slug/STATUS.md`. A
`Workflow`-tool run also exposes its own per-lane progress view.

If this CLI exposes background-agent commands they help too — skip any that error
(they depend on CLI support):

```bash
claude agents              # interactive background-agent view
claude attach <session-id>
claude logs <session-id>
claude stop <session-id>
```

## Integration

The conductor never auto-merges blindly. At integration it reads every
`RESULT.md`, reviews `NEEDS_USER.md`, inspects each worker's branch/worktree,
**runs the seam test** for any `CONTRACTS.md` contract (the shared fixture,
adversarial edge case included, against both the producer's and consumer's code)
before merging that seam, resolves conflicts one lane at a time, runs
tests/build, writes the top-level `README.md` and final wiring, and records the
outcome — seam-test result included — in `INTEGRATION.md`.
