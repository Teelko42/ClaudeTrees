# ClaudeTrees — Review & Improvement Report

**Scope:** the ClaudeTrees orchestration system as it exists for this repo — the
global skill (`~/.claude/skills/claudetrees/`), the three agents
(`~/.claude/agents/claudetrees-*.md`), and the two run artifacts committed under
`T:\ConversationAgent\.claudetrees\runs\`.
**Date:** 2026-06-15
**Method:** read every skill/agent/doc/template + both runs' bus files, cross-checked
the documented protocol against what the runs actually produced, and traced the one
known production bug (INV-8 / "H-7") back to its root cause in the protocol.

---

## TL;DR

ClaudeTrees is a sound, well-documented "filesystem-as-message-bus" orchestration
pattern, and it demonstrably worked (two runs, a design corpus, a green Phase-0
spine). The problems are **not** in the idea — they're in **consistency and
enforcement**:

1. Two agents still point at the *old* `.feature-forge/` path (copy-paste leftover).
2. The protocol pins shared **names** but not shared **semantics** — which is exactly
   how the one real integration bug got through.
3. The worker agent contradicts itself about whether it may write `NEEDS_USER.md`,
   and that write invites a concurrent-append race.
4. Worker `isolation: worktree` is the default, yet both runs (correctly) disabled it,
   and the docs then spend a whole section working around the problem it creates.
5. Round 2 silently diverged from the documented file protocol — the canonical docs
   now describe a process the most recent run didn't follow.

Severity table:

| # | Finding | Severity | Effort |
|---|---|---|---|
| 1 | Agents reference stale `.feature-forge/runs/` path | **High** | trivial |
| 2 | Worker self-contradiction on `NEEDS_USER.md` + append race | **High** | small |
| 3 | DECISIONS pins names, not field semantics (root cause of INV-8) | **High** | medium |
| 4 | `isolation: worktree` default fights the actual/blessed practice | Medium | small |
| 5 | Round-2 run drifted from the documented bus protocol | Medium | medium |
| 6 | Whole `.claudetrees/` is git-ignored → committed docs cite missing files | Medium | small |
| 7 | Orphan `worktree-phase1-foundation` branch + dir never cleaned up | Low | trivial |
| 8 | `claude --bg` documented first but never works here | Low | small |
| 9 | "Task tool" terminology + `MAN-id` separator inconsistency | Low | trivial |

---

## 1. Stale `.feature-forge/` path in two agents — **High**

ClaudeTrees was forked from a `feature-forge` skill, and two agents never had their
run-directory path updated:

- `~/.claude/agents/claudetrees-idea-splitter.md:27` — *"the shared run directory
  under `.feature-forge/runs/<run-id>/`. This is where your output is written"*
- `~/.claude/agents/claudetrees-scribe.md:20` — *"the absolute path of the ClaudeTrees
  run directory under `.feature-forge/runs/<run-id>/`."*

Meanwhile the skill (`SKILL.md:60`) and every real run use `.claudetrees/runs/`.

**Why it matters:** the conductor passes an absolute path, so a run *usually* survives.
But an agent that reasons from its own system prompt (e.g. to validate or re-derive a
path, or when the conductor's prompt is terse) is told the wrong workspace name. It's
a latent foot-gun and it makes the agents internally inconsistent with the skill they
serve.

**Fix:** replace `.feature-forge/runs/` → `.claudetrees/runs/` in both agents. The
*intentional* `feature-forge` mentions in `SKILL.md:23,65` (describing the sibling
lineage) are fine and should stay.

---

## 2. Worker contradicts itself on `NEEDS_USER.md`, and the write is a race — **High**

`~/.claude/agents/claudetrees-worker.md` says both:

- Line 51 — *"**Do not edit the global bus files** (`STATUS.md`, `DISPATCH.md`,
  **`NEEDS_USER.md`**, `BLOCKERS.md`, `DECISIONS.md`, `INTEGRATION.md`)"*
- Lines 58–60 — *"append it **immediately** to … the global `NEEDS_USER.md`"*

These are directly opposed. The bus-protocol doc tries to thread the needle
("workers *append*, never *edit*"), but "append immediately to NEEDS_USER.md" from N
parallel workers is precisely the concurrent-write hazard the design elsewhere avoids
by giving each writer a private file. The scribe *owns* `NEEDS_USER.md` for exactly
this reason.

Run 1 actually did the safe thing: workers wrote only their `features/*/MANUAL.md`,
and the scribe produced `NEEDS_USER.md` — i.e. the *spec's own instruction was not
followed*, and that's why it worked.

**Fix:** make the worker spec single-sourced — workers write **only** their local
`MANUAL.md`; the scribe is the sole writer of `NEEDS_USER.md`. Delete the
"append … to the global NEEDS_USER.md" instruction (lines 58–60) and keep
`NEEDS_USER.md` in the do-not-touch list. Update `markdown-bus-protocol.md` and
`manual-task-rules.md` to match (both currently say "worker appends to both").

---

## 3. The protocol pins shared *names* but not shared *semantics* — **High**

This is the systemic root cause behind the project's one real integration defect
(INV-8, a.k.a. the "H-7 hidden disagreement"), documented in
`.claudetrees/runs/20260601-phase0-spine/INTEGRATION-NOTES.md`.

`DECISIONS.md` D06 (run 1) reads: *"Canonical data contracts (**names are fixed**;
F01 owns the authoritative schema definition, **others reference by name only**)."*
It then lists each contract at the *prose* level only. Nothing pins the meaning of the
`supersedes` field — specifically the edge case of a **self-reference**
(`supersedes === segment_id`):

- **Lane C** set `supersedes = segment_id` to mean *"this final closes its own partial"*
  (self-finalize).
- **Lane D** read `supersedes !== null` as *"this is a correction of a prior segment"*
  and routed to `propagateSupersede`, so **no ConceptCard was ever produced**.

Both lanes honored D06 (same field name) and the build still broke. Worse, **Lane D's
own unit tests passed** because the fixture defaulted `supersedes: null` — the
"all green" was false comfort, later patched with a `FinalizingSttProvider` shim and
only fixed properly afterward.

The slicing rules explicitly *endorse* "name-only cross references" as non-blocking
(`slicing-rules.md`), which is true for an *agent name* or a *file path* — but a
**shared data contract is not name-only**; its field semantics are a real interface,
and the protocol has no place to pin them or to test them across the seam before
fan-out.

**Fix (highest-leverage change in this report):**

1. Add a conductor-owned **`CONTRACTS.md`** (or a checked-in schema, e.g. zod/JSON
   Schema) to the bus that pins, per shared field, its meaning *and its sentinel/
   edge-case values* — e.g. *"`supersedes`: id of a **prior, distinct** segment this
   corrects; `null` or `=== segment_id` means no correction."*
2. Make `claudetrees-idea-splitter` treat a shared data contract as a **blocking
   seam**, not a name-only reference — every lane that produces or consumes it must
   be listed, and the seam gets a single **shared fixture**.
3. Add a "**seam test**" step: before integration, a test that both the producer and
   consumer lane run against the *same* fixtures (including the adversarial edge case),
   so a fixture can't dodge the disagreement. The post-mortem in `INTEGRATION-NOTES.md`
   already wrote the correct 3-line fix; the goal is for the *process* to force that
   discovery, not luck.

---

## 4. `isolation: worktree` default fights the blessed practice — **Medium**

`claudetrees-worker.md:6` sets `isolation: worktree`. Yet:

- Run 1 `DISPATCH.md`: *"Worktree isolation skipped intentionally — lanes own disjoint
  directories, so they write directly to the shared tree with zero collision risk."*
- The "How It Was Built" doc calls worktrees *"the trick that wasn't needed."*
- `markdown-bus-protocol.md` then dedicates its single most-emphasised section
  ("the most important rule") to the fact that a worktree-isolated worker **can't see
  the git-ignored `.claudetrees/` bus**, requiring `--add-dir` + absolute paths.

So the default isolation *creates* the bus-unreachability problem the protocol then
works hard to mitigate — and the runs disabled it anyway.

**Fix:** flip the default. Worktree isolation should be **opt-in**, used only when a
lane genuinely risks editing files another lane owns. When lanes own disjoint
directories (the stated goal of the whole slicing design), workers should run in the
shared tree and reach the bus by plain relative path. Keep the `--add-dir` guidance
as the documented escape hatch for the opt-in worktree case.

---

## 5. Round 2 silently diverged from the documented protocol — **Medium**

The canonical docs describe a process the most recent run didn't follow. Concretely,
`20260601-phase0-spine` deviates from `SKILL.md` / `markdown-bus-protocol.md`:

| Protocol says | Round 2 did |
|---|---|
| Lanes numbered `FNN` (`slicing-rules.md`: "never renumber, MAN-FNN-NNN keys off it") | Lanes named `A`–`E`; yet manual-task IDs still read `MAN-F01-*` → letters and IDs don't correspond |
| Global `INTEGRATION.md` | `INTEGRATION-NOTES.md` |
| `BLOCKERS.md` is a global bus file | absent |
| Per-lane `STATUS/RESULT/MANUAL/NOTES.md` (the worker's core contracts) | only `FEATURE.md` + `PLAN.md` exist per lane |
| `DISPATCH.md` = Feature/Worker/Session-id/Worktree | columns are Lane/Package/Depends-on/Notes |
| (no such file) | added `PHASES.md` |
| Dispatch background `claudetrees-worker` subagents | executed via the **Workflow tool** (deterministic A·B·C·D→E fan-out) |

Using the Workflow tool was arguably an *upgrade* (deterministic, structured returns),
but it means the worker agent and most of the bus protocol **were never exercised** in
the latest run — the structured returns replaced the per-lane `RESULT.md`/`STATUS.md`
files the spec calls for.

**Fix:** pick one and make the docs honest:
- **(a)** Bless the Workflow-tool path: add a "Deterministic execution via Workflow"
  section to `SKILL.md`, define the lighter file set it implies, and either adopt
  `FNN` in that mode or drop the `MAN-FNN` coupling; **or**
- **(b)** Tighten so future runs conform (keep `FNN`, emit per-lane RESULT/STATUS even
  under Workflow).

Either is fine; the current state — docs describing process A, newest run using
process B — is the trap for the next user.

---

## 6. The whole `.claudetrees/` is git-ignored → committed docs cite missing files — **Medium**

`.gitignore:1-2` ignores `.claudetrees/` and `.claude/`. That's defensible for
*scaffolding*. But the runs contain genuinely valuable, referenced history:

- `README.md:165-166` points the reader at
  `.claudetrees/runs/<run>/NEEDS_USER.md` ("**27 tasks, 10 High**").
- `packages/capture/src/index.ts:5` cites
  `.claudetrees/runs/20260601-phase0-spine/features/B-capture/PLAN.md`.
- The Obsidian vault links the run structure throughout.

On a fresh clone, **none of those targets exist** — the committed code and docs point
into an ignored void. There's already good precedent for fixing this: commit `334477e`
*promoted* the decisions ledger in-tree (`docs/architecture/00-integration/DECISIONS.md`).

**Fix:** do the same for the artifacts that committed files reference — promote the
final `NEEDS_USER.md` ledger (and ideally the INV-8 `INTEGRATION-NOTES.md` post-mortem)
into `docs/`, and repoint `README.md` / the source comment at the in-tree copies.
Leave the rest of `.claudetrees/` ignored. Don't let committed artifacts cite ignored
paths.

---

## 7. Orphan `worktree-phase1-foundation` branch + directory — **Low**

`git branch` still lists `worktree-phase1-foundation`, and
`.claude/worktrees/phase1-foundation/` is still on disk (a full package tree with its
own `node_modules`, `pnpm-lock.yaml`, etc.) — but it is **not** a registered worktree
(`git worktree list` shows only the main checkout). It's a never-merged Phase-1 attempt
pinned to an old commit. The "How It Was Built" doc already flagged this with the exact
cleanup commands, but they were never run.

**Fix:** confirm nothing of value is unmerged, then:
```powershell
git branch -D worktree-phase1-foundation
Remove-Item -Recurse -Force .claude\worktrees\phase1-foundation
```
The directory is local-only clutter (`.claude/` is ignored); the dangling **branch**
is the real leftover.

---

## 8. `claude --bg` is documented first but has never worked here — **Low**

`SKILL.md` (step 4/5) and `markdown-bus-protocol.md` lead with `claude --bg …` launch
syntax, then add a caveat that the flag "may not exist." Both runs confirm it doesn't:
*"local `claude --bg` flag unavailable; dispatched as harness-native background
subagents."* So the primary, most-prominent instruction is the one that fails, and the
working path (Agent/Task subagent, or the Workflow tool) is presented as a fallback.

**Fix:** invert the emphasis — lead with the harness-native subagent / Workflow path
as the primary mechanism, and demote `claude --bg` to an "if your CLI supports it"
footnote.

---

## 9. Small consistency nits — **Low**

- **"Task tool" terminology.** `SKILL.md` and the worker doc say to launch via the
  "`Agent`/`Task` tool." In this harness, subagents launch via **Agent**; `Task*` is a
  separate (deferred) task-tracker family. Drop the `Task` alias to avoid sending a
  future conductor to the wrong tool.
- **`MAN-` separator drift.** `manual-task-rules.md` merges source IDs with commas
  (`MAN-F01-002, MAN-F04-001`); the real run-1 `NEEDS_USER.md` uses slashes
  (`MAN-F02-005 / MAN-F03-003 / …`). Pick one separator so the ledger is greppable.
- **Scribe is `haiku`.** Fine for bookkeeping, but run 1's consolidation was real
  judgment work (31→27 rows with cross-lane semantic dedup, severity assignment,
  vague-task rewrites). If ledger quality is load-bearing, consider `sonnet` for the
  consolidation pass — observational, not a defect.

---

## What's genuinely good (keep)

- **Disjoint file ownership per lane** is the right primitive and it held — zero
  file-collision incidents across both runs.
- **The manual-task ledger** (`NEEDS_USER.md` + `MAN-FNN-NNN`) is excellent: nothing
  a human owes the build got lost, and secrets-by-name-never-value is correctly
  enforced.
- **Reviewed integration** (never auto-merge) caught the INV-8 issue and documented
  the root cause honestly rather than burying it.
- **Templates + docs are thorough** and make the pattern reproducible.

## Recommended order of work

1. **#1** (path) and **#2** (worker contradiction) — trivial/small, pure correctness.
2. **#3** (contract semantics + seam test) — the one change that prevents the class of
   bug that actually bit this project.
3. **#5** + **#6** — make the docs match reality and stop citing ignored files.
4. **#4**, **#7**, **#8**, **#9** — cleanup/polish.

> Note: items #1–#4, #8, #9 are edits to **global** files under
> `~/.claude/` (the skill/agents), not this repo. #5–#7 touch this repo
> (`.gitignore`, `docs/`, `README.md`, the orphan branch).
