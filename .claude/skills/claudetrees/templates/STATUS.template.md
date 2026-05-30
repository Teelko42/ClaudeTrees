# Status

> Conductor-owned global status for the whole run. Lives at the run-directory
> root: `.claudetrees/runs/<run-id>/STATUS.md`. (Each lane keeps its own
> `features/FNN-slug/STATUS.md` separately.)

## Overall

- State: planning | dispatched | integrating | blocked | complete
- Active workers:
- Completed workers:
- Blocked/failed workers:
- Needs user: <count of open NEEDS_USER.md rows>

## Timeline

<!-- Append one dated line per event, newest at the bottom. -->

- <YYYY-MM-DD HH:MM> — run created; idea sliced into N lanes.
- <YYYY-MM-DD HH:MM> — dispatched workers: F01, F02, ...
- <YYYY-MM-DD HH:MM> — scribe launched.
- <YYYY-MM-DD HH:MM> — FNN complete.
- <YYYY-MM-DD HH:MM> — integration started.
