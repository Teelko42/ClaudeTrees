# Shared Contracts

<!--
  Conductor-owned. Create this file ONLY when a data structure crosses a lane
  seam — i.e. one lane PRODUCES a record/value that another lane CONSUMES and
  interprets. A shared *name* (an agent name, a file path) does NOT need a
  contract; a shared *meaning* does.

  This file exists because pinning a field's NAME is not enough: two lanes can
  agree on `supersedes` as a field and still disagree on what its edge values
  mean, and both unit suites pass while the integrated build breaks. Pin the
  MEANING and the SENTINEL / EDGE-CASE values here, list every producer and
  consumer lane, and point both at ONE shared fixture (including the adversarial
  edge case). The seam is verified at integration against that shared fixture —
  never against a lane's private fixture.

  One "## Contract:" block per shared data structure. Delete this comment block
  when the file is treated as final.
-->

## Contract: <ContractName>

- **Produced by:** <FNN, ...>   (every lane that creates this structure)
- **Consumed by:** <FNN, ...>   (every lane that reads/interprets it)
- **Authoritative shape owned by:** <FNN>   (the lane that owns the schema/type)
- **Shared fixture:** <relative/path/to/shared/fixture>   (both lanes test
  against THIS file — including its adversarial edge case — not a private one)
- **Seam dependency:** blocking — consumers integrate only after the shared
  fixture passes end-to-end (see SKILL.md step 7, "Run the seam test").

### Fields

| Field | Type | Meaning | Sentinel / edge-case values (the part that bites) |
|---|---|---|---|
| `<field>` | `<type>` | <what it means in one line> | <what `null` / empty / a self-reference / out-of-range means — spell out every value a consumer must special-case> |

### Adversarial cases the shared fixture MUST include

- <the edge case a private fixture would default away — e.g. "a record whose
  `supersedes` equals its own id (self-finalize), which must NOT be routed as a
  correction">

<!--
  Worked example — the real INV-8 defect this file prevents:

  ## Contract: SttSegment

  - Produced by: F03 (capture/STT)
  - Consumed by: F04 (concept-card builder)
  - Authoritative shape owned by: F03
  - Shared fixture: fixtures/stt-segments.shared.json
  - Seam dependency: blocking

  | Field | Type | Meaning | Sentinel / edge-case values |
  |---|---|---|---|
  | `segment_id` | string | stable id of this segment | never null |
  | `supersedes` | string \| null | id of a PRIOR, DISTINCT segment this final corrects | `null` means "no correction"; a value EQUAL TO this segment's own `segment_id` ALSO means "no correction" (it is a self-finalize, not a correction) — only a value naming a *different* prior segment routes to propagateSupersede |

  ### Adversarial cases the shared fixture MUST include
  - a segment with `supersedes === segment_id` (self-finalize) → a ConceptCard
    IS produced; it is NOT routed as a correction.

  Without the bolded sentinel rule, the producer set `supersedes = segment_id`
  to self-finalize, the consumer read `supersedes !== null` as "correction" and
  produced no ConceptCard, and BOTH lanes' unit tests passed because each
  fixture defaulted `supersedes: null`.
-->
