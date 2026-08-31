# Engineer Dashboard — specs

Epic: [SD-3452](https://castille-labs.atlassian.net/browse/SD-3452)

> **Reading `BE-nn`.** Stories and specs cite platform rules as **BE-01 … BE-30** — these are **numbered sections in `specs/ENGINEERING-BRIEF.md`**, not the `BE.md` files in spec folders. A `BE.md` is the backend spec for one story; a `BE-nn` is a platform-wide rule in the brief. Same two letters, two different things.

| Folder | Story |
|---|---|
| `SD-3453-tabs-overview` | Bench tabs & overview — the `/work-log` page frame |
| `SD-3455-log-hours` | Log Hours — the form |
| `SD-3456-entries` | Entries — list, calendar, edit history |
| `SD-3458-invoices` | Invoices — the `/invoices` page |

**Build order:** SD-3459 (Internal, per-bench dates) → SD-3453 backend → SD-3453 frontend → the rest in any order.
