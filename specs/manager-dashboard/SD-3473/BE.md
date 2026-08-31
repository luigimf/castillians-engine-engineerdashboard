# BE — V Bench page: Engineer Work Logs

Backend specification for **SD-3473**. A read-only surface.

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** this file: `BE.md` is the backend spec for one story, `BE-nn` is a numbered rule in the brief.

---

## Endpoint

### `GET /api/manager/benches/{benchId}/work-logs?scope=period|all&limit=&offset=`

```json
{
  "scopeTotal": 34,
  "entries": [ {
    "entryId": "wl-1042",
    "engineerId": "eng-7", "engineerName": "Maria Alves",
    "date": "2026-08-07", "hours": 7, "description": "…",
    "status": "auto_approved"
  } ]
}
```

**Acceptance criteria**
- **Settled entries only** — `approved`, `auto_approved` and `auto_declined`. Entries still `submitted`, and entries `declined` by a person, are excluded **server-side**.
- **`auto_declined` is included deliberately.** It is terminal and in the client's favour: the hours were logged, nobody reviewed them before the close (BE-29), and the client is not billed. Omitting it would leave the manager unable to account for hours their engineer reported.
- **`declined` is excluded** because it carries a reviewer's message written for internal use.
- `status` is returned, and is only ever `approved`, `auto_approved` or `auto_declined` — the manager sees **whether a person signed the entry off**, which is what they ask about when querying an invoice. It is never `submitted` or `declined`.
- **No `history`, no `declineMsg`, no rate and no amount in the payload.** Assert in a test that the Manager response carries none of them — hiding them client-side would leak them to anyone reading the response.
- `scope=period` resolves **this bench's own** period (SD-3459), never the calendar month. `scope=all` spans the whole engagement.
- `scopeTotal` is the scope's grand total, independent of `limit` and `offset`.
- `limit` defaults to `MANAGER_WORK_LOGS_BATCH = 10`; the constant lives server-side and the client reads it from the response rather than duplicating it.
- Sort is `date DESC, entryId DESC` — **stable**, so no row duplicates or drops across a reveal boundary.
- **There is no write endpoint.** No approve, no decline, no edit — for any client role, including Admin.
- No session state: the same query string always returns the same page.

---

## Integration & sync

| Value | Source | Also appears on |
|---|---|---|
| The entries | Work log records (SD-3455) | Engineer Entries, Internal Work Logs (SD-3466), Internal bench accordion (SD-3465) |
| Which are visible | `approved`, `auto_approved` and `auto_declined`. `submitted` and `declined` are excluded server-side | The Manager-visible subset of SD-3467 |
| Engineer name, avatar | Engineer profile | Every dashboard |
| Period boundaries | Per-bench Start + Auto-Renew (SD-3459) | Hours card, list page, Engineer Work Log page |

**Acceptance criteria**
- An entry **approved** on Internal appears here on the next read, and the bench's Capacity Used and per-engineer usage move with it — one approval, every surface.
- An entry **declined** on Internal disappears here on the next read, and its hours leave every figure.
- An entry **auto-declined at the period close** (BE-29) appears here with its badge on the next read, and its hours leave every figure at the same moment.
- **The sum of the `approved` and `auto_approved` hours under `scope=period` equals `capacityUsedHours`** on the capacity endpoint for the same bench and period. If the log and the figure can disagree, one of them is wrong.
- **`auto_declined` hours are excluded from that sum**, and from every figure on the hours card. They are shown, not counted.
- An entry **edited** on the Engineer dashboard returns its new hours, date and description immediately — with **no history record and no indication it was edited**. A manager sees the current truth, not the audit trail.
- Each bench resolves its own period, so `scope=period` means different date ranges on two benches in one organisation.
