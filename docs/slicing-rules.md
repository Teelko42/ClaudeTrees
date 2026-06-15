# Slicing Rules

How ClaudeTrees turns one raw idea into independent feature lanes that separate
background workers can build in parallel without colliding. The
`claudetrees-idea-splitter` agent applies these rules; this doc is the human-
readable reference behind them.

## The goal of a good split

A lane plan is good when each lane can be handed to its own Claude Code worker
and built **without waiting on, or writing to, any other lane's files**. The
two properties that make this possible:

1. **Disjoint file ownership** — every file belongs to exactly one lane.
2. **Vertical slices** — each lane is a feature with a visible outcome, not a
   horizontal layer ("backend", "UI") that has to touch everything.

Aim for **2-5 lanes**. Fewer than 2 is not a split. More than 5 fragments a
single run into coordination overhead.

## Good vs bad lane boundaries

### Good boundaries

- **Vertical, feature-shaped.** "Idea Splitter" owns its agent, its template,
  and its doc end to end. A worker can finish it and you can verify it alone.
- **Exclusive file ownership.** Lane A owns `docs/a.md`; lane B owns
  `docs/b.md`. No file appears in two lanes.
- **Name-only cross references.** Lane B's prose may reference an agent name
  that lane A produces (e.g. "calls `foo-splitter`"). The name is fixed in
  FEATURES up front, so B never has to wait for A or edit A's files.
- **Checkable Done when.** "The agent file exists, parses, and its example
  matches the template" — you can confirm it without judgment calls.

### Bad boundaries

- **Horizontal layers.** "Backend" + "Frontend" + "Glue" — every lane touches
  shared files, nothing is independently useful, and Done is unverifiable.
- **Shared central file.** Two lanes both edit `README.md` or one config file.
  This is a guaranteed collision. Fix it: move that file to a shared `F00`
  setup lane, or let the conductor own it at integration.
- **Hidden ordering.** Lane B silently needs lane A's output file to exist. If
  a real ordering dependency exists, it must be declared, not implied.
- **Vague Goal / User value.** "Improve architecture" gives a worker nothing to
  build and gives you nothing to verify.

When a candidate split has a collision, resolve it in this priority order:

1. Reassign the file so one lane owns it exclusively, and have the other lane
   reference it by name only.
2. Pull the shared file into a `F00` shared-setup lane that runs first.
3. Merge the two lanes into one.
4. Leave it to the conductor at integration (use for top-level wiring like the
   root `README.md`).

## FNN numbering scheme

Lanes are numbered `FNN`, zero-padded, in declaration order:

- `F00` — reserved for an optional **shared-setup** lane that other lanes
  depend on (scaffolding, shared config). Use it only when something genuinely
  must exist before the parallel lanes start.
- `F01`, `F02`, `F03`, ... — the independent parallel lanes.

Each lane's folder downstream is `features/FNN-<kebab-name>/` inside the run
dir. The number is stable for the whole run; never renumber a lane mid-run,
because worker prompts, status files, and manual-task IDs (`MAN-FNN-NNN`) all
key off it.

## How dependencies are expressed

Most lanes should be `Dependencies: none`. There are three kinds of cross-lane
relationship; the last two block:

- **Name-only reference (non-blocking).** Lane B mentions a fixed name that
  lane A owns — an agent name, a template path. Because the name is written
  into FEATURES before any worker starts, B can be built in parallel. Express
  it as: `Dependencies: references <name> from FNN`. This does NOT make B wait.
- **Shared data contract (blocking seam).** Lane A produces a data structure
  (a record, a JSON shape, a return value) that lane B reads and *interprets*.
  This is a real interface, not a name — see the next section. Express it as:
  `Dependencies: shares the <Contract> contract with FNN (see CONTRACTS.md)`.
  It blocks: the seam is pinned in `CONTRACTS.md` and tested against one shared
  fixture before integration.
- **Ordering dependency (blocking).** Lane B cannot start until lane A's output
  physically exists. Avoid these where possible; when unavoidable, state it in
  the FEATURES intro AND in B's field: `Dependencies: must follow FNN`. The
  conductor schedules these sequentially.

`Collision risk` is separate from dependencies: it describes shared-file
danger, and should read `none — exclusive ownership` for a clean split.

## Name-only reference vs shared data contract

This distinction is load-bearing — collapsing the two is what shipped this
project's one real integration bug.

- A **name-only reference** is safe to leave non-blocking: lane B calls
  `fetch_weather` by name, or points at `docs/a.md`. The *name* is fixed in
  FEATURES up front, nothing about its meaning is ambiguous, and B never waits.
- A **shared data contract** is NOT name-only. When lane A produces a value that
  lane B must interpret, agreeing on the field *name* does not make them agree on
  its *meaning* — especially its edge and sentinel values. That agreement is a
  real interface and must be pinned and tested, not assumed.

**The cautionary tale (INV-8).** Two lanes shared a `supersedes` field. The
producer set `supersedes = segment_id` (a self-reference) to mean "this final
closes its own partial." The consumer read `supersedes !== null` as "this
corrects a prior segment" and routed it away, so **no output was ever produced**.
Both honored the shared *name*; the build still broke. Worse, the consumer's unit
tests **passed**, because its fixture defaulted `supersedes: null` and never
exercised the self-reference — a false green.

So when a split has a shared data contract, the splitter must:

1. **Flag it as a seam, not a name-only reference** — list every producer and
   consumer lane, and mark the dependency blocking.
2. **Hand it to `CONTRACTS.md`** — every shared field's meaning *and* its
   sentinel / edge-case values (what `null`, empty, or a self-reference means).
3. **Require one shared fixture** — including the adversarial edge case — that
   both lanes test against, so a private fixture cannot dodge the disagreement.
   The conductor runs it as a **seam test** before integrating the seam.

A clean split has no shared data contracts at all — prefer redesigning a seam
into disjoint ownership with a name-only reference where you can. Only pin a
contract when a data structure genuinely must cross the seam.

## Worked example

**Sample idea:** "A small CLI that fetches the weather for a city and can also
save favorite cities to a local file, with docs."

Scanning a fresh repo, the splitter produces three independent lanes — disjoint
files, vertical slices, one name-only reference. Using the exact field set from
`FEATURES.template.md`:

---

Three independent lanes. File ownership is namespaced so workers do not
collide. F03 references the fetch function name from F01 by name only — a
documented dependency, not a shared-file edit. Top-level wiring (root
`README.md`, CLI entry assembly) is handled by the conductor at integration.

## F01: Weather Fetch Core

- Goal: Build the module that calls the weather API for a given city and returns
  a normalized result object.
- User value: A reusable function that turns a city name into current weather —
  the engine the rest of the CLI is built on.
- Likely files:
  - `src/weather/fetch.py`
  - `tests/test_fetch.py`
- Dependencies: none.
- Collision risk: none — exclusive ownership of the files above.
- Manual user tasks likely: supply a weather API key as an env var (the fetch
  module reads `WEATHER_API_KEY`); confirm the API endpoint is reachable.
- Out of scope: the favorites store (F02), the docs (F03).
- Done when: `fetch_weather(city)` returns a normalized result for a valid city
  and raises a clear error for an unknown city, covered by passing tests.

## F02: Favorites Store

- Goal: Build the local store that adds, lists, and removes favorite cities in a
  plain JSON file under the user's config dir.
- User value: The user can save and reuse favorite cities instead of retyping
  them every run.
- Likely files:
  - `src/weather/favorites.py`
  - `tests/test_favorites.py`
- Dependencies: none.
- Collision risk: none — exclusive ownership of the files above.
- Manual user tasks likely: none expected.
- Out of scope: fetching weather (F01), the docs (F03).
- Done when: add/list/remove operations round-trip through the JSON file and
  survive a process restart, covered by passing tests.

## F03: Usage Docs

- Goal: Write the user-facing usage guide covering install, the fetch command,
  and the favorites commands.
- User value: The user can learn the whole CLI from one page without reading
  source.
- Likely files:
  - `docs/usage.md`
- Dependencies: references the `fetch_weather` function name from F01 — name
  only, non-blocking.
- Collision risk: none — exclusive ownership of the file above.
- Manual user tasks likely: none expected.
- Out of scope: implementing fetch (F01) or favorites (F02); the root
  `README.md` (conductor at integration).
- Done when: `docs/usage.md` documents install plus every command with a copy-
  pasteable example, and every referenced command name matches the code F01/F02
  ship.

---

Note how the example holds the line: three vertical lanes, each file owned once,
collision risk `none` across the board, the only cross-lane link expressed as a
name-only reference, and the root `README.md` deliberately left to integration.
