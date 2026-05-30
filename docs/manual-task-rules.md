# Manual-Task Rules

This doc defines what counts as a **manual task** in ClaudeTrees, how those
tasks are identified (`MAN-FNN-NNN`), and how they flow from a worker into the
single user-facing ledger. It is the contract the `claudetrees-worker` writes
against and the `claudetrees-scribe` reads.

A manual task is anything a background worker **cannot or must not do for the
user** — because it needs a human's credentials, account, judgment, money,
legal authority, or explicit permission. Workers do not guess at these; they
log them so nothing the user owes the build gets lost.

## Where manual tasks live

- A worker **appends** each task it hits to its own
  `features/FNN-<lane>/MANUAL.md` (the `MANUAL.template.md` shape). It does
  **not** edit the global ledger.
- The `claudetrees-scribe` reads every `features/*/MANUAL.md`, plus
  `STATUS.md`, `RESULT.md`, and `BLOCKERS.md`, and consolidates them into the
  one global `NEEDS_USER.md`, merging duplicates and keeping all source ids.
- `NEEDS_USER.md` is the only file the user has to read to know what is on them.

## Taxonomy — what to log as a manual task

Log a task when it falls into any of these categories:

- **Credentials and secrets** — API keys, tokens, passwords, signing keys,
  service-account JSON. Name the variable/service, **never** the value
  (e.g. "set `OPENAI_API_KEY`"), and never paste a secret into any file.
- **Account setup** — creating an account or project with a third-party
  provider (cloud console, SaaS dashboard, registrar) the worker has no login
  for.
- **External integration setup** — anything configured outside the repo:
  - **DNS** — adding/changing records, verifying domain ownership.
  - **OAuth** — registering an OAuth client, setting redirect URIs, consent
    screen.
  - **Webhooks** — registering an endpoint with a provider, copying a signing
    secret.
  - **App store** — store listings, bundle ids, review submission.
  - **Billing** — enabling billing, choosing a plan, adding a payment method.
  - **Legal / privacy** — terms of service, privacy policy, data-processing
    agreements, consent flows that need human/legal sign-off.
  - **Email provider** — verifying a sending domain, SPF/DKIM/DMARC, getting
    out of a sandbox/allowlist.
- **Human product decisions** — choices the worker should not make alone:
  naming, pricing, copy, which option to ship, scope trade-offs.
- **Package legitimacy confirmation** — when a worker wants to add a dependency
  it cannot verify is the genuine, non-typosquatted, maintained package; the
  user confirms before install.
- **Production DB migrations** — running irreversible or production-affecting
  schema/data changes; the worker prepares, the user approves and runs.
- **Payment / vendor approvals** — purchases, paid-tier upgrades, contracts, or
  vendor sign-offs that commit money.
- **Permission prompts a worker cannot safely answer** — any harness or tool
  permission prompt whose consequences the worker cannot evaluate (destructive
  commands, granting access, sending real messages). Surface it instead of
  guessing.

If a task does not fit a category but still needs a human, log it anyway and
describe it concretely — the taxonomy is a floor, not a fence.

## ID scheme — `MAN-FNN-NNN`

Each manual task gets a stable id so it stays traceable through merges:

- `MAN` — fixed prefix.
- `FNN` — the **zero-padded feature/lane number** that raised it (e.g. `F02`).
  Conductor / run-level tasks not owned by a lane use `RUN` (e.g.
  `MAN-RUN-001`).
- `NNN` — a **zero-padded counter unique within that lane**, assigned in append
  order: `001`, `002`, `003`, ...

Example: the first manual task raised by lane F02 is `MAN-F02-001`; its second
is `MAN-F02-002`.

When the scribe merges duplicate tasks from several lanes into one ledger row,
it **keeps every source id** in the `ID` cell (e.g. `MAN-F01-002, MAN-F04-001`)
so each lane's reference remains intact.

## Severity and status

- **Severity** — `high` (a lane is blocked until it is done), `med` (needed
  before release but not blocking now), `low` (nice-to-have / informational).
- **Status** — `open` (not done), `blocked` (waiting on another task), `done`
  (proven complete by a `RESULT.md` or a user message — never assumed).

## Worked example

A worker on lane F02 needs a Stripe key to wire up checkout. It cannot create
the account or read the user's secret, so it appends this row to
`features/F02-.../MANUAL.md`, and the scribe folds it into `NEEDS_USER.md`.
The columns match the NEEDS_USER ledger exactly:

| ID | Source | Severity | User action | Why needed | Blocks | Status |
|---|---|---|---|---|---|---|
| MAN-F02-001 | F02 | high | Create a Stripe account, then set `STRIPE_SECRET_KEY` in `.env` (paste the secret key, do not commit it) | Checkout calls the Stripe API and fails without a live key | F02 payments lane | open |
