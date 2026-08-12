# BE — Invoices (`/invoices`)

Backend specification for **SD-3458**. Reading an engineer's earnings per billable period, and
the automatic invoice submitted on their behalf.

Depends on the period engine from **SD-3453** and the approved-hours store from **SD-3456**.

---

## 1. Integration points

| System | Field | Use |
|---|---|---|
| Internal Dashboard → Manage Subscription | `Configured Blended Hourly Rate` | Basis for the engineer rate |
| Internal Dashboard → Mark-Ups & Discounts | `Percentage Mark-up` | Divisor for the engineer rate |
| SFM | Supplier record → currency | The engineer's single currency |

---

## 2. Endpoint

### `GET /api/engineer/invoices?period={YYYY-MM}`

Omit `period` to get the **most recent closed period**, or the current one if none has closed.

```json
{
  "periodKey": "2026-08",
  "label": "August 2026",
  "isOngoing": true,
  "currency": "EUR",
  "rows": [
    { "benchName": "Core Platform",  "clientName": "Northmill Bank",      "hours": 25, "ratePerHour": 59.38, "earnings": 1484.38 },
    { "benchName": "Insurance Web",  "clientName": "Northmill Insurance", "hours": 37, "ratePerHour": 55.00, "earnings": 2035.00 },
    { "benchName": "Data Squad",     "clientName": "Vertex Health",       "hours": 37, "ratePerHour": 57.50, "earnings": 2127.50 }
  ],
  "total": 5646.88,
  "invoiceNumber": null,
  "earliestPeriod": "2026-05",
  "latestPeriod": "2026-08"
}
```

**Rules**

- One row per Virtual Bench the engineer logged **approved** hours against in that period.
  Pending and declined hours are excluded entirely.
- `ratePerHour` is the **engineer-facing** rate: `configuredBlendedRate ÷ (1 + markup)`. The
  configured rate must never appear in this response.
- `earnings = hours × ratePerHour`, and `total` is the sum of the rows.
- `isOngoing` is true while the period has not closed. `invoiceNumber` is `null` until it has.
- `earliestPeriod` / `latestPeriod` bound the client's navigator — it must not have to probe.
- **Single currency.** The engineer is a supplier with one currency on their SFM record; every
  row and the total use it, whatever the clients' own currencies are. Never mix, never total
  across currencies.
- Money is exact to the cent, never rounded to whole units. Currency **codes**, not symbols.

---

## 3. Automatic invoice submission

On the **last day of the month** the portal generates each engineer's invoice from their approved
work logs and submits it **on their behalf**. The engineer never files one manually.

**Acceptance criteria**

- One invoice per engineer covering all their benches, itemised per bench — the same breakdown
  this endpoint returns.
- Runs in the same job as the month-end finance export, so the two can never disagree.
- Stamps `invoiceNumber` and flips `isOngoing` to false for that period.
- Writes Invoice Number and Invoice Recvd onto the payroll checklist row.
- Emails the engineer a copy of what was submitted.
- Entries still awaiting approval at the cut-off are **excluded** and carry into the next
  period's invoice.

Hour verification needs no extra work here: the approval flow and the append-only edit history
already establish it.

---

## 4. Test cases worth writing first

1. Engineer with hours on three benches in one period → three rows, one currency, total is their sum.
2. Engineer with 8h awaiting approval → those hours absent from rows and total.
3. Declined hours → absent from rows and total.
4. Open period → `isOngoing` true, `invoiceNumber` null.
5. Closed period → `isOngoing` false, `invoiceNumber` populated.
6. Request with no `period` → returns the most recent **closed** period.
7. Request with no `period` and nothing closed yet → returns the current period, `isOngoing` true.
8. Period with no approved hours → empty `rows`, `total` 0, still a valid response.
9. Engineer paid in EUR working a USD client's bench → row and total both in EUR.
10. Grep the response for `configuredBlendedRate` → absent.
11. Configured 95.00 at a 60% mark-up → `ratePerHour` 59.38. Change the mark-up to 50% → 63.33,
    no deploy.
12. Month-end job runs → invoice submitted, number stamped, engineer emailed, checklist row updated.
