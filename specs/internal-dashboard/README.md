# Internal Dashboard — specs

Epic: [SD-3460](https://castille-labs.atlassian.net/browse/SD-3460)

| Folder | Jira | Covers |
|---|---|---|
| `SD-3461-channel-billing-cards/` | [SD-3461](https://castille-labs.atlassian.net/browse/SD-3461) | Channel & Billing — root client cards & report downloads |
| `SD-3462-channel-page/` | [SD-3462](https://castille-labs.atlassian.net/browse/SD-3462) | Channel page — breakdown tree & billing |
| `SD-3464-engagements-benches/` | [SD-3464](https://castille-labs.atlassian.net/browse/SD-3464) | Engagements → Virtual Benches tab — search, filters, results |
| `SD-3465-bench-entry/` | [SD-3465](https://castille-labs.atlassian.net/browse/SD-3465) | Expanded bench entry — overage, allocation, notes, order forms |
| `SD-3466-engagements-work-logs-filters/` | [SD-3466](https://castille-labs.atlassian.net/browse/SD-3466) | Work Logs tab — search, filters & batching (10 + See more) |
| `SD-3467-engagements-work-log-entry/` | [SD-3467](https://castille-labs.atlassian.net/browse/SD-3467) | Work log entry, edit history, View Work Log & the per-engineer page |

**SD-3459** — per-bench Start and Auto-Renew dates — is specified as **BE-07** in `../ENGINEERING-BRIEF.md` rather than in its own folder. The change is to the Internal dashboard's Manage Subscription modal, but it **blocks the Engineer Dashboard epic**: every date-derived figure resolves from those two fields.

---

## Two rules worth reading before writing code

- **The authorised block replaces the 20% tolerance.** 160h plan + 40h block → **200h**, not 232h. (`SD-3465-bench-entry/BE.md`)
- **Late approvals carry over.** An entry from a previous period, approved late on an ongoing engagement, bills in the **next** period and must show as `Period Earned` ≠ `Period Billed` on the payroll checklist and the engineer invoicing export. (`SD-3467-engagements-work-log-entry/BE.md`, BE-22)

## Build order

1. **SD-3464** — the Engagements page frame and its two tab counters. Everything else mounts into it.
2. **SD-3465** — the bench entry, which owns the overage state the approval logic reads.
3. **SD-3466** — the Work Logs list, filters and batching.
4. **SD-3467** — the entry, history, and the per-engineer page. Ship the BE-22 report columns with it.

Scope still to spec, per `../ENGINEERING-BRIEF.md`: the month-end reports as a story, and the SFM integration.
