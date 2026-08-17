# BE — Work Log: Entries

Backend specification for **SD-3456**. Reading entries, editing them, and the append-only
history that records every change.

Depends on the period and allocation endpoints from **SD-3453**.

---

## 1. Endpoints

### `GET /api/engineer/work-logs?benchId={id}&period={YYYY-MM}&page={n}`

Paginated entries for one bench and period, **most recent first**. Prior periods are readable;
`canEdit` is `false` on them.

```json
{
  "page": 1,
  "pageSize": 10,
  "total": 14,
  "entries": [
    {
      "entryId": "wl-8842",
      "date": "2026-08-08",
      "hours": 6,
      "description": "Refactored the payment reconciliation job.",
      "status": "auto_approved",
      "canEdit": true,
      "declineMessage": null,
      "history": [
        {
          "timestamp": "2026-08-08T17:42:00Z",
          "actorName": "Maria Borg",
          "actorType": "engineer",
          "action": "Created",
          "detail": "6h logged",
          "diffs": []
        }
      ]
    }
  ]
}
```

`status` ∈ `auto_approved` | `approval_required` | `approved` | `declined`.

`auto_approved` and `approved` are **distinct** — the engineer needs to tell an entry that
passed automatically from one a human signed off.

---

### `GET /api/engineer/work-logs/calendar?benchId={id}&month={YYYY-MM}`

Day-keyed totals for the calendar view. **Any** month is readable, including months in prior
periods — the engineer can look back even though those entries are not editable.

```json
{
  "2026-08-04": { "hours": 5, "entries": 1 },
  "2026-08-07": { "hours": 6, "entries": 1 },
  "2026-08-08": { "hours": 6, "entries": 1 }
}
```

Also return the bounds of the available range so the UI can disable its month arrows:

```json
{ "earliestMonth": "2026-06", "latestMonth": "2026-08", "days": { … } }
```

---

### `PATCH /api/engineer/work-logs/{entryId}`

```json
{ "date": "2026-08-08", "hours": 8, "description": "…" }
```

Same validation as create (see the SD-3455 spec), plus:

- `403` when the entry falls in a **prior** subscription period.
- A **manually approved** entry in the **current** period **is** editable.
- Only `hours`, `date` and `description` are mutable.

---

## 2. Status transitions on edit

This is the section most likely to be implemented wrongly.

| Change | Previous status | New status | Notification |
|---|---|---|---|
| Description only | any | **unchanged** | **none** |
| Hours **decreased** | any | **unchanged** | **none** |
| Hours decreased below capacity | `approval_required` | `auto_approved` | none |
| Hours **increased**, still within threshold | any | `auto_approved` | none |
| Hours **increased** past `capacity × 1.20` | `auto_approved` | `approval_required` | approval email |
| Hours increased past threshold | `approval_required` | stays `approval_required` | none — already queued |

The rule in one line: **only an increase in hours that crosses the threshold re-opens review.**
A description fix is not a new claim and must be silent.

---

## 3. History

**Table: `work_log_history`** — `id`, `entry_id`, `timestamp`, `actor_name`, `actor_type`,
`action`, `detail`, `diffs` (JSON)

**Append-only. No updates, no deletes.** It is the audit trail Finance relies on to verify
hours worked, so it must be immutable at the storage layer, not merely by convention.

### Actions

| Action | Actor type | Detail | Diffs |
|---|---|---|---|
| `Created` | `engineer` | `"6h logged"` | none — but the UI shows the original description |
| `Edited` | `engineer` | `"hours changed"` / `"date, description changed"` | one per changed field |
| `Sent for approval` | `system` | `"monthly hours exceeded"` | none |
| `Approved` | `internal` | — | none |
| `Declined` | `internal` | the decline message, quoted | none |

### Diff shape

```json
{ "field": "Hours", "from": "3h", "to": "4h" }
```

Only `Hours`, `Date` and `Description` are tracked. Format values for display server-side so
the client renders them verbatim.

### Actor naming

| `actorType` | Display name |
|---|---|
| `engineer` | the engineer's own name |
| `internal` | **Castillians Team** — never the individual reviewer's name |
| `system` | **Castillians System** |

### Visibility

| Dashboard | History returned |
|---|---|
| Engineer | **yes** |
| Internal | **yes** |
| Manager | **no** — withheld entirely |

---

## 4. Declined entries

- A decline message is **mandatory**; an empty or whitespace-only message is rejected.
- Declined entries stay readable on the **Engineer** and **Internal** dashboards with the
  message attached.
- Declined entries are **excluded** from every Manager dashboard response — the client should
  never receive them and filter client-side.
- A declined entry's hours do **not** count toward `loggedHours` or earnings.

---

## 5. Entries on ended engagements

An engagement that has ended cannot be re-scoped, so its entries are settled:

- `canEdit` is `false`.
- An entry left in `approval_required` when the engagement ended reads as `approved` — never
  left showing as awaiting a decision that will never come.
- No approve/decline affordance is offered on them anywhere.

---

## 6. Test cases worth writing first

1. Edit description only → status unchanged, **zero** emails, history gains one `Edited` row
   with a single Description diff.
2. Edit hours 6 → 4 → status unchanged, no email.
3. Edit hours 4 → 9 crossing 120% → status `approval_required`, one email, history shows both
   the `Edited` row and a `Sent for approval` row.
4. Edit hours 9 → 4 on an `approval_required` entry, bringing it under capacity → status
   `auto_approved`.
5. `PATCH` an entry dated in the previous period → `403`.
6. `PATCH` a manually approved entry in the current period → succeeds.
7. Decline with an empty message → rejected.
8. Declined entry → present in Engineer and Internal responses, **absent** from Manager.
9. Declined entry's hours → excluded from `loggedHours` and earnings.
10. Attempt to `UPDATE` or `DELETE` a history row → refused at the storage layer.
11. Entry on an ended engagement previously `approval_required` → reads `approved`, `canEdit` false.
12. Calendar request for a month two periods back → returns data, all entries `canEdit: false`.
13. `auto_approved` and `approved` are distinguishable in the payload — not collapsed to one value.


---

## Pagination — 5 per page

`GET /api/engineer/work-logs` returns **5 entries per page** for the Entries list.

**Acceptance criteria**
- `pageSize` is **5**, held as a named constant — not a literal in a slice expression.
- The response carries `page`, `pageSize`, `total` and `entries`, so the client can render "Page N of M" without a second call.
- Requesting a page beyond the last returns the **last** page rather than an empty list — a stale page number in state must not blank the card.
- Sorted **most recent first**, consistently across pages.
- **A newly created entry resets the client to page 1.** It is the newest, so it belongs on the first page; leaving the user on page 3 hides what they just logged.
- Changing bench tab or period resets to page 1.
- The calendar view is **not** paginated — it returns the whole month.
