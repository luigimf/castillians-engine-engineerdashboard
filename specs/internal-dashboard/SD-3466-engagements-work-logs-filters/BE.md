# BE — Engagements: Work Logs tab, search & filters

Backend specification for **SD-3466**.

---

## Endpoints

### `GET /api/internal/work-logs?q=&approvalsOnly=&period=&scope=&limit=&offset=`

```json
{
  "totals": {
    "filtered": 34,
    "approvalsInPeriod": 6,
    "byPeriod":  { "today": 2, "last3": 5, "thisWeek": 9, "thisMonth": 24, "allTime": 118 },
    "byScope":   { "ongoing": 31, "past": 12, "all": 43 }
  },
  "entries": [
    {
      "entryId": "wl-1042", "engineerId": "eng-7", "engineerName": "Maria Alves",
      "engineerEmail": "maria@…", "vettingScore": 4.6,
      "benchId": "vb-core", "benchName": "Core Platform", "clientName": "Northmill Bank",
      "date": "2026-08-07", "hours": 7, "description": "…",
      "status": "approval_required",
      "engagementEnded": false, "engagementEndDate": null,
      "benchCapacityHours": 160, "benchHoursUsed": 96, "benchPercentUsed": 60,
      "hasHistory": true
    }
  ]
}
```

**Acceptance criteria**
- `byPeriod` and `byScope` are **grand totals for their own dimension**, ignoring the other filters. `approvalsInPeriod` follows the **period** only.
- `filtered` is the total for the full filter set — the "N entries" summary. `entries` is one batch of it.
- `q` matches engineer **name and email**, case-insensitive. Not description, not client.
- All four filters compose with **AND**.
- `limit` defaults to `WORK_LOGS_BATCH = 10`; the constant lives server-side and the client reads it from the response rather than duplicating it.
- Sort is `date DESC, entryId DESC` — **stable**, so no row duplicates or drops across a batch boundary.
- `engagementEnded` is **derived at read time** from the bench subscription (SD-3459). It is never a column on the entry.
- `approvalsInPeriod` **excludes entries on ended engagements**.
- `benchPercentUsed` counts **approved** entries only, against the bench's own period (BE-02, BE-03).
- No session state: the same query string always returns the same page.

---

## Ongoing / past resolution

```
engagementEnded = bench.autoRenewDate != null && bench.autoRenewDate < today
engagementEndDate = engagementEnded ? bench.autoRenewDate : null
```

**Acceptance criteria**
- Ending a bench's subscription moves every one of its entries from Ongoing to Past on the next read, with **no write to the entries**.
- An indefinite bench (`autoRenewDate` null) is never past.
- `scope=ongoing` and `scope=past` are strictly disjoint; `all` is their union.

## Approval queue

- **Approval required applies to ongoing engagements only.** An entry left awaiting a decision when its engagement ended resolves as **approved** and is returned with no actions.
- On an ongoing engagement an entry from a **previous billing period stays actionable** — never aged out, never silently approved.
- Approving late bills the hours in the **next open period**; the two report columns carry it (**BE-22**, SD-3467).

---

## Integration & sync

The filters read shared state; they own none of it.

| Value | Source | Also appears on |
|---|---|---|
| The entries | Work log records, written from the Engineer dashboard (SD-3455) | Engineer Entries, Internal bench accordion (SD-3465), Manager bench page |
| Approval state | Bench overage state and thresholds (SD-3465, BE-13) | Engineer status tag, Internal bench accordion |
| Ongoing / past | Per-bench Start + Auto-Renew dates (SD-3459) | The entry's ended label, engineer page bench tabs |
| Engineer name, email | Engineer profile | Every dashboard, payroll checklist, invoice PDF |
| Bench capacity, hours used | Approved logs against the bench's own period | Virtual Benches tab, Manager bench page |

**Acceptance criteria**
- An entry logged on the Engineer dashboard appears here immediately under the default filters — no cache, no reconciliation job.
- A new entry **resets the list to its first batch**, so it can never land in an unrevealed batch.
- An entry crossing its approval threshold appears under **Approval required only** and raises that count **in the same read** — queue and count come from one query.
- Approving or declining anywhere — here or in the bench accordion (SD-3465) — moves this list's status tag and the approvals count on the next read. The two surfaces must never disagree.
- Pending-approval hours are excluded from every "hours logged" figure on all three dashboards. This filter shows them; capacity figures do not count them.
- The **Work Logs (n)** tab counter is a grand total and reacts to nothing on this row.
- No currency is ever converted.
