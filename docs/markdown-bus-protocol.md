# ClaudeTrees Markdown Bus Protocol

ClaudeTrees coordinates a conductor and several background workers using nothing
but markdown files in a shared **run directory**. There is no database and no
server — the filesystem is the message bus. This document defines every bus
file, who reads and writes it, and the rules background workers must follow to
reach the bus from inside an isolated worktree.

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

## Absolute Paths and `--add-dir` (the most important rule)

Background workers run with **`isolation: worktree`** — Claude checks each
worker out into its own git worktree before it edits code. That worktree does
**not** contain the git-ignored `.claudetrees/` workspace, so a worker cannot
reach the bus with a relative path. Two rules make the bus reachable:

1. **Always address bus files by absolute path** in the worker prompt
   (`$RUN_DIR/IDEA.md`, never `./IDEA.md`).
2. **Always pass `--add-dir "$RUN_DIR"`** when launching a background session so
   the run directory is granted to the session regardless of its worktree.

The conductor resolves `$RUN_DIR` to an absolute path once and reuses it in
every prompt.

## Bus Files

### Global bus files (run-directory root)

| File | Writer(s) | Reader(s) | Purpose |
|---|---|---|---|
| `IDEA.md` | conductor (seeds from `$ARGUMENTS`) | all workers, scribe | Raw idea, product, audience, assumptions, constraints, non-goals. |
| `FEATURES.md` | `claudetrees-idea-splitter` (or conductor) | all workers, scribe | One section per lane: goal, value, files, dependencies, out-of-scope, done-when. |
| `DISPATCH.md` | conductor | conductor, user | One row per background session: Feature, Worker name, Session id, Status, Worktree/branch, Last update. |
| `STATUS.md` | conductor | conductor, user | Overall run state + dated timeline. Workers do **not** edit this. |
| `INTEGRATION.md` | conductor | conductor, user | What merged, what was skipped, conflicts resolved, build/test results. |
| `NEEDS_USER.md` | `claudetrees-scribe` (consolidated); workers append rows | user, conductor | The single deduplicated manual-task ledger. |
| `DECISIONS.md` | conductor | all workers | Default decisions the conductor made; overridable by the user. Read-only for workers. |
| `BLOCKERS.md` | conductor | conductor | Cross-lane blockers. Workers surface blockers in their own files; the conductor folds them in to avoid races. |

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

- **Conductor** writes all global bus files except `NEEDS_USER.md` (shared with
  the scribe) and seeds `IDEA.md`/`FEATURES.md`.
- **Workers** write only their own `features/FNN-slug/` files and their lane's
  product files. They **append** rows to `NEEDS_USER.md` but never edit the
  other global bus files.
- **Scribe** owns the consolidated body of `NEEDS_USER.md` (dedup, source links,
  keeping open tasks actionable).

## Manual-Task IDs

Manual tasks use stable IDs `MAN-FNN-NNN` (lane number, then a zero-padded
sequence within that lane, e.g. `MAN-F02-001`). A worker appends the row to both
its local `MANUAL.md` and the global `NEEDS_USER.md`; the scribe deduplicates.
Row format:

```markdown
| MAN-F02-001 | F02 | low | Confirm local `claude` CLI background-launch syntax | Background dispatch command must be verified on this machine | Does not block (foreground fallback exists) | open |
```

## Launching Background Workers

The conductor launches one background session per lane with the
`claudetrees-worker` agent, plus one for `claudetrees-scribe`.

### Windows PowerShell

```powershell
$RUN_DIR = Join-Path (Get-Location) ".claudetrees\runs\<run-id>"

claude --bg `
  --agent claudetrees-worker `
  --name "ct-F02-worker-dispatcher" `
  --add-dir "$RUN_DIR" `
  "You are implementing feature F02. Shared run directory: $RUN_DIR. Read $RUN_DIR\IDEA.md, $RUN_DIR\FEATURES.md, $RUN_DIR\DECISIONS.md, and $RUN_DIR\features\F02-worker-dispatcher\PLAN.md. Implement only your lane. Update your features\F02-worker-dispatcher\STATUS.md, MANUAL.md, and RESULT.md. Do not edit global bus files."
```

### bash fallback

```bash
RUN_DIR="$(pwd)/.claudetrees/runs/<run-id>"

claude --bg \
  --agent claudetrees-worker \
  --name "ct-F02-worker-dispatcher" \
  --add-dir "$RUN_DIR" \
  "You are implementing feature F02. Shared run directory: $RUN_DIR. Read $RUN_DIR/IDEA.md, $RUN_DIR/FEATURES.md, $RUN_DIR/DECISIONS.md, and $RUN_DIR/features/F02-worker-dispatcher/PLAN.md. Implement only your lane. Update your features/F02-worker-dispatcher/STATUS.md, MANUAL.md, and RESULT.md. Do not edit global bus files."
```

### Launching the scribe

```bash
claude --bg \
  --agent claudetrees-scribe \
  --name "ct-scribe" \
  --add-dir "$RUN_DIR" \
  "Shared run directory: $RUN_DIR. Monitor every features/*/MANUAL.md, STATUS.md, and RESULT.md until all workers are complete, blocked, or failed. After each pass, update $RUN_DIR/NEEDS_USER.md: consolidate duplicates, preserve source links, keep open tasks actionable. Do not implement code."
```

## CLI Caveat — there may be no `--bg` flag

**Real-world note:** some `claude` CLI versions have **no `--bg` flag**. On those
versions, `claude agents` is an *interactive* view for managing background agents
— it is not a scriptable one-shot launcher, so the `claude --bg ...` commands
above will not work verbatim. The exact background-launch syntax must be verified
on the local machine (track this as a manual task, e.g. `MAN-F02-001`).

When background launch is unavailable, use the **foreground-subagent fallback**:
the conductor invokes `claudetrees-worker` (and later `claudetrees-scribe`) as
normal subagents through the `Agent`/`Task` tool, one lane at a time or as the
harness permits. Workers still read and write the same absolute bus files, so the
protocol is unchanged — only the launch mechanism differs. In the harness-native
background path, the existing worktree-isolation mechanism provides the
concurrency that `--bg` would otherwise provide.

## Monitoring

```bash
claude agents              # interactive background-agent view
claude attach <session-id>
claude logs <session-id>
claude stop <session-id>
```

The bus itself is also the dashboard: tail the global `STATUS.md`, `DISPATCH.md`,
and each `features/FNN-slug/STATUS.md`.

## Integration

The conductor never auto-merges blindly. At integration it reads every
`RESULT.md`, reviews `NEEDS_USER.md`, inspects each worker's branch/worktree,
resolves conflicts one lane at a time, runs tests/build, writes the top-level
`README.md` and final wiring, and records the outcome in `INTEGRATION.md`.
