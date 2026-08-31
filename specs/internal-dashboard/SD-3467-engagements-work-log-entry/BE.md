# BE — Engagements: work log entry, edit history & the per-engineer Work Log page

Backend specification for **SD-3467**. Carries a change to two month-end reports — see **Carry-over**.

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** this file: `BE.md` is the backend spec for one story, `BE-nn` is a numbered rule in the brief.

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

## The period close, and automatic decline — BE-29

```
cutOff    = 23:59 on the period's last day        // logging stops
close     = the 3rd of the following month        // reports generate (SD-3468)
```

**Approval required applies only to ongoing engagements**, but on an ongoing engagement an entry from a **previous billing period stays approvable or declinable right up to the close**. It is not aged out and not silently approved.

**Acceptance criteria**
- An approval taken **between the cut-off and the close** is filed against the period the hours were **worked** in, so the client charge and the engineer's payment both land in that month. Nothing carries over.
- At the close, every entry still `submitted` becomes **`auto_declined`** — a distinct status from a human `declined`, because an auditor asks which it was.
- `auto_declined` hours count towards **nothing**: not capacity, not remaining hours, not earnings, not billing, not any report.
- The entry carries **no decline message and no reviewer** — the endpoints never set one. It gains an append-only `Castillians System` history record, _"Automatically declined — not reviewed before the period close"_, timestamped at the close.
- Both approve and decline are refused on an `auto_declined` entry — `409`, current status returned. Reinstatement is a deliberate internal action, not the ordinary approve path.
- The close is derived from the bench's own period (SD-3459); two benches on one account may close on different dates.
- Auto-decline is a **fallback**. The two reminder emails (three days before the cut-off, and on the cut-off date) fire only when the queue is non-empty, so the close should never have to do this.

## No carry-over — extends BE-22

**Acceptance criteria**
- An approved entry's hours are invoiced in the period they were **worked** in. There is no state in which hours are paid or billed in a later period.
- `Period Earned`, `Period Billed`, `Carried Over` and `payableNextPeriod` **do not exist** — not on the entry payload, not in any report, not on any surface.
- **Client hours = supplier hours, every period.** The same approval bills the client and pays the engineer, and it can only fall inside one period.
- An **auto-decline is final**. There is no reinstatement endpoint: moving hours into a later period is exactly the carry-over Finance ruled out. A genuine error is corrected by Finance **outside the platform**, so the platform's record stays a truthful account of what was approved in time.
- Approving late **never rewrites a closed period** — the endpoints refuse it (`409`).

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

**Acceptance criteria**
- Approving makes the entry count towards billing, surfaces it on Manager, and emails the engineer. Hours used, capacity bar and remaining hours all move on the next read — **recomputed, never patched client-side**.
- Declining keeps it on Internal and Engineer with its message, hides it from Manager, and removes its hours from every figure.
- Either action **removes the entry from the Approval required only filter and lowers that count in the same read**.
- An entry edited on the Engineer dashboard appears here with new values and a new history record immediately. An edit with the **same or fewer hours** changes no status and notifies nobody.
- Ending a bench's subscription closes out its entries — awaiting reads as approved, actions disappear, the capacity bar is withdrawn — with **no write to the entries**.
- Pending-approval hours are excluded from every "hours logged" figure on all three dashboards.
- No money figure is converted between currencies.
