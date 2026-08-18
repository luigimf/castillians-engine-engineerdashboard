# BE — Engagements: work log entry, edit history & the per-engineer Work Log page

Backend specification for **SD-3467**. Carries a change to two month-end reports — see **Carry-over**.

---

## Endpoints

### `GET /api/internal/work-logs/{entryId}/history`

```json
{ "records": [
  { "recordId": "h-9", "actor": "Castillians Team", "actorType": "internal",
    "at": "2026-08-08T09:14:00Z", "action": "Approved", "detail": null, "changes": [] },
  { "recordId": "h-8", "actor": "Maria Alves", "actorType": "engineer",
    "at": "2026-08-07T17:02:00Z", "action": "Edited", "detail": null,
    "changes": [ { "field": "hours", "from": "6", "to": "7" } ] },
  { "recordId": "h-1", "actor": "Maria Alves", "actorType": "engineer",
    "at": "2026-08-07T16:40:00Z", "action": "Created", "detail": "Original description…" }
] }
```

**Acceptance criteria**
- **Append-only, enforced at the storage layer** — no update, no delete paths exist. It is the audit trail Finance relies on.
- Newest first. Only `hours`, `date` and `description` are tracked as diffs.
- Actor strings are resolved server-side: the engineer's own name, **Castillians Team** for internal actions, **Castillians System** for automatic ones.
- **Withheld from Manager responses entirely** — filtered server-side, never sent and hidden client-side.
- An entry with only a `Created` record returns `hasHistory: false` on the list endpoint so the accordion is not rendered.

### `POST /api/internal/work-logs/{entryId}/approve`
### `POST /api/internal/work-logs/{entryId}/decline`  `{ "message": "…" }`

**Acceptance criteria**
- Both are refused when the entry's engagement has **ended**, and when the entry is not awaiting a decision — `409`, with the current status returned.
- Decline requires a non-empty, non-whitespace `message`; rejected `422` otherwise.
- **Double-submit is guarded server-side** — two Approve calls produce one approval, one history record and one email (idempotency on entry + target status).
- Approve → the entry counts towards billing, becomes visible to Manager, and the engineer is emailed.
- Decline → stays on Internal and Engineer with the message; **excluded from Manager**; its hours count towards **nothing** — not capacity, not billing, not earnings.
- Both append a history record; approval is authored by **Castillians Team**, never the reviewer's own name.
- The same entry is actionable from three surfaces — the Work Logs list, the bench accordion (SD-3465) and the per-engineer page. **One record, one guard**: none may offer an action the others would refuse.

### `GET /api/internal/engineers/{engineerId}/work-logs`

```json
{
  "engineer": { "name": "…", "email": "…", "vettingScore": 4.6 },
  "benches": [ { "benchId": "vb-core", "name": "Core Platform", "entryCount": 14,
                 "engagementEnded": false, "engagementEndDate": null } ],
  "entries": { "vb-core": [ /* entries, newest first, unbatched */ ] }
}
```

**Acceptance criteria**
- `entryCount` is that engineer's **grand total** on that bench, across every period.
- Not batched — the bench tab already narrows the set.
- `engagementEnded` derives from the bench subscription (SD-3459), never stored on the entry.

---

## Entries on ended engagements

- An entry left awaiting a decision when its engagement ended **resolves as approved**. It is never left showing a decision that will never come.
- No approve or decline is offered on it, on any surface, and the endpoints refuse it.
- Its history remains readable in full.
- None of this writes to the entries — ending the subscription is the only change.

## Late approval and carry-over — extends BE-22

**Approval required applies only to ongoing engagements**, but on an ongoing engagement an entry from a **previous billing period stays approvable or declinable**. It is not aged out and not silently approved.

```
periodEarned = the bench period containing entry.date      // SD-3459, pro-rated first period included
periodBilled = the first period still open at approval time
carriedOver  = periodEarned != periodBilled                 // derived, never stored
```

**Acceptance criteria**
- Approving late adds the hours to the engineer's **next** period's invoicing.
- `periodEarned` resolves against **that bench's own** subscription period — never the calendar month by default.
- **Approving late never rewrites a closed period.** Re-running a closed month reproduces byte-identical output (§G5).

### Report columns — payroll checklist and engineer invoicing export

| Column | Meaning |
|---|---|
| **Period Earned** | The billing period the entry's date falls in, against the bench's own period |
| **Period Billed** | The period the hours are invoiced in — the open period at approval |
| **Carried Over** | Yes/No, **derived** from the two. No new stored state; gives Finance something to filter on |

- Equal values for an entry approved inside its own period; where they differ the row is a **carry-over**.
- Rows are keyed **engineer × bench × Period Earned**, so one export can carry two or more rows for the same engineer and bench.
- **Carried hours are never merged into the normal row.** 150 hours earned this period plus 12 approved late must not appear as a single 162-hour row — Finance has to see the carry-over rather than absorb it into the wrong month.
- Every existing column keeps its position and name; the new ones are **appended to the Hours group**.
- The **SFM supplier upload's 22 fixed columns are untouched** — an SFM row follows the invoice, which is issued in the Period Billed.
- The month-end email body already lists entries held back at the 23:59 cut-off; the export must now show **where those hours landed** when they were eventually approved.

---

## Integration & sync

Every action writes to shared state; every figure is read from one source.

| Value | Source | Also appears on |
|---|---|---|
| Hours, date, description | The work log record, written on the Engineer dashboard (SD-3455, SD-3456) | Engineer Entries, Internal bench accordion (SD-3465), Manager bench page |
| Status | Bench overage state and thresholds (SD-3465, BE-11 to BE-13) | Engineer status tag, Work Logs list, approvals count |
| Edit history | `work_log_history`, append-only | Engineer entry accordion. **Withheld from Manager** |
| Capacity, hours used | Approved logs against the bench's own period (SD-3459) | Virtual Benches tab, Manager bench page, Engineer overview |
| Engagement ended date | Per-bench subscription (SD-3459) | Ongoing/Past filter (SD-3466), engineer page bench tabs |
| Engineer name, email, vetting score | Engineer profile | Every dashboard, payroll checklist, invoice PDF |
| Period Earned / Period Billed | Derived — entry date against the bench period, and the open period at approval | Payroll checklist, engineer invoicing export |

**Acceptance criteria**
- Approving makes the entry count towards billing, surfaces it on Manager, and emails the engineer. Hours used, capacity bar and remaining hours all move on the next read — **recomputed, never patched client-side**.
- Declining keeps it on Internal and Engineer with its message, hides it from Manager, and removes its hours from every figure.
- Either action **removes the entry from the Approval required only filter and lowers that count in the same read**.
- An entry edited on the Engineer dashboard appears here with new values and a new history record immediately. An edit with the **same or fewer hours** changes no status and notifies nobody.
- Ending a bench's subscription closes out its entries — awaiting reads as approved, actions disappear, the capacity bar is withdrawn — with **no write to the entries**.
- Pending-approval hours are excluded from every "hours logged" figure on all three dashboards.
- No money figure is converted between currencies.
