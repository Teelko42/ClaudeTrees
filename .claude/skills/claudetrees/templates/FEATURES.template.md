# Features

<!--
  This file is the lane plan for one ClaudeTrees run. The
  claudetrees-idea-splitter agent fills it in from a raw idea; the conductor
  reads it to dispatch one background worker per lane.

  Replace this intro with 2-4 sentences that state:
    - how many lanes there are (2-5),
    - that the lanes are independent and own disjoint files,
    - how file ownership is namespaced so workers do not collide,
    - any name-only cross-lane references (a lane mentioning another lane's
      fixed agent/file name is allowed and is NOT a blocking dependency).

  Then add one "## FNN: <name>" block per lane, using the placeholder below as
  the exact field set. Keep the fields in the order shown. Delete every comment
  before this file is treated as final.
-->

Three independent lanes (example count). File ownership is namespaced so workers
do not collide. Any cross-lane reference is by name only — a documented
dependency, not a shared-file edit. Top-level wiring is handled by the conductor
at integration, not by any worker.

## FNN: <name>

<!--
  FNN = a zero-padded lane number, assigned in declaration order:
  F01, F02, F03, ... Use F00 only for a shared-setup lane that other lanes
  depend on. <name> is a short Title Case feature name.
-->

- Goal: <what this lane builds, in one or two sentences>
- User value: <the concrete outcome the user gets — not "a backend", a result>
- Likely files:
  - <exact/path/owned/exclusively/by/this/lane>
  - <another/exact/path>
- Dependencies: <none | "references <name> from FNN" | "must follow FNN">
- Collision risk: <none — exclusive ownership | name the shared file + mitigation>
- Manual user tasks likely: <none expected | install X / supply key / confirm flag>
- Out of scope: <what this lane does NOT do, naming the lanes that own it>
- Done when: <a checkable definition of done — the verifiable end state>
