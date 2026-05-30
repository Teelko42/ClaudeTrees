# Dispatch

> Conductor-owned. One row per background session (worker and scribe). Update the
> `Status` and `Last update` columns as sessions progress. Lives at the
> run-directory root: `.claudetrees/runs/<run-id>/DISPATCH.md`.
>
> Status values: `launching` | `running` | `complete` | `blocked` | `failed`.

| Feature | Worker name | Session id | Status | Worktree/branch | Last update |
|---|---|---|---|---|---|
| F01 | ct-F01-slug | <session-id or pending lookup> | launching | <worktree/branch or main> | <YYYY-MM-DD HH:MM> |
| —   | ct-scribe   | <session-id or pending lookup> | launching | —               | <YYYY-MM-DD HH:MM> |
