# ClaudeTrees

A Claude Code skill that takes a project idea, splits it into **independent
feature lanes**, launches multiple **background Claude Code workers** to build
them in parallel, coordinates everything through **markdown "bus" files**, and
keeps a clear **manual-task ledger** for anything that needs a human.

ClaudeTrees is a self-contained sibling to the `feature-forge` skill. Same proven
shape — one conductor skill, three subagents, markdown templates, and docs — with
its own `claudetrees-*` namespace so the two coexist without conflict.

## How it works

```
idea ──▶ /claudetrees ──▶ inspect repo
                          │
                          ├─▶ claudetrees-idea-splitter ──▶ 2–5 disjoint lanes (FEATURES.md)
                          │
                          ├─▶ plan each lane (FEATURE.md + PLAN.md)
                          │
                          ├─▶ claudetrees-worker  ×N  (background, one lane each)
                          │        └─ coordinate via the markdown bus
                          │
                          ├─▶ claudetrees-scribe ──▶ consolidates NEEDS_USER.md
                          │
                          └─▶ integrate under review (never auto-merge)
```

The conductor never lets two workers edit the same file unless a lane explicitly
depends on another, and it never hides manual work — credentials, accounts, DNS,
OAuth, billing, product decisions, and permission prompts all surface in
`NEEDS_USER.md`.

## Usage

From Claude Code in this repo:

```
/claudetrees Build me <your project idea in a sentence or two>
```

If the idea is too vague to split, the conductor asks up to five short clarifying
questions first. Then it creates a run workspace under `.claudetrees/runs/<id>/`,
slices the idea, launches workers, and reports how to monitor them.

## What's in here

```
.claude/
  skills/claudetrees/
    SKILL.md                       # the /claudetrees conductor
    templates/                     # files the conductor instantiates per run
      IDEA.template.md
      FEATURES.template.md
      DISPATCH.template.md
      STATUS.template.md
      NEEDS_USER.template.md
      MANUAL.template.md
  agents/
    claudetrees-idea-splitter.md   # idea  -> independent lanes
    claudetrees-worker.md          # implements exactly one lane
    claudetrees-scribe.md          # maintains the manual-task ledger
docs/
  slicing-rules.md                 # good vs bad lane boundaries, FNN scheme
  markdown-bus-protocol.md         # every bus file, who reads/writes it
  manual-task-rules.md             # manual-task taxonomy + MAN-FNN-NNN ids
```

## The markdown bus

Each run gets a workspace whose files are the shared communication layer between
the conductor and the workers:

| File | Purpose |
|---|---|
| `IDEA.md` | raw idea, assumptions, constraints, non-goals |
| `FEATURES.md` | the lane split (one section per lane) |
| `DISPATCH.md` | every launched worker session |
| `STATUS.md` | overall state + timeline |
| `INTEGRATION.md` | the reviewed merge checklist |
| `NEEDS_USER.md` | consolidated human-action ledger |
| `DECISIONS.md` | conductor defaults (override any) |
| `BLOCKERS.md` | cross-lane blockers / races |
| `features/FNN-slug/` | per-lane `FEATURE`, `PLAN`, `STATUS`, `NOTES`, `MANUAL`, `RESULT` |

See `docs/markdown-bus-protocol.md` for the full reader/writer contract.

## Background launch note

Background workers are launched with the `claude` CLI using absolute paths and
`--add-dir "$RUN_DIR"` so isolated sessions can still reach the shared bus.
Some `claude` CLI versions have **no `--bg` flag** and instead manage background
agents through the interactive `claude agents` view; a foreground-subagent
fallback also works. The conductor and `docs/markdown-bus-protocol.md` document
all three paths.

---

Built with ClaudeTrees' own sibling, `feature-forge` — three lanes (idea
splitter, worker dispatcher, manual-task tracker) implemented in parallel by
background workers.
