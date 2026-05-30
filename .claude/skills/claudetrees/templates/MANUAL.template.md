# Manual Tasks — FNN-<lane-name>

<!--
  Per-lane manual-task log. The worker building this lane APPENDS a row here
  whenever it hits something only the user can do: a credential, an account, an
  external setup (DNS/OAuth/webhook/app-store/billing/legal/privacy/email
  provider), a human product decision, package-legitimacy confirmation, a
  production DB migration, a payment/vendor approval, or any permission prompt a
  worker cannot safely answer. See docs/manual-task-rules.md for the taxonomy.

  ID convention: MAN-FNN-NNN
    - FNN  = this lane's zero-padded feature number (e.g. F02).
    - NNN  = a zero-padded counter, unique within the lane, in append order
             (001, 002, ...).
    Example: the first manual task in lane F02 is MAN-F02-001.

  Keep secrets OUT of this file: name the variable or service, never the value.
  The claudetrees-scribe reads every features/*/MANUAL.md and folds these rows
  into the global NEEDS_USER.md (merging duplicates across lanes). Do NOT edit
  the global NEEDS_USER.md yourself.

  Columns below match the NEEDS_USER ledger exactly. Delete this comment block
  if you keep the file as a finished artifact.
-->

| ID | Source | Severity | User action | Why needed | Blocks | Status |
|---|---|---|---|---|---|---|
