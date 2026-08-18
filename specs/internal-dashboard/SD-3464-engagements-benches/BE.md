# BE — Engagements: Virtual Benches tab

Backend specification for **SD-3464**.

---

## Endpoints

### `GET /api/internal/benches?channel=&client=&q=`

Active benches across all channels, filtered.

```json
{
  "totals": { "benches": 6, "workLogs": 27 },
  "benches": [
    {
      "benchId": "vb-ins", "name": "Insurance Web", "clientName": "Northmill Insurance",
      "capacityHours": 80, "hoursUsed": 96, "percentUsed": 120, "remainingHours": 0,
      "currency": "EUR"
    }
  ]
}
```

**Acceptance criteria**
- `totals` are **grand totals**, unaffected by the filters — the tab counters must not move as the operator searches.
- `remainingHours` is `max(0, capacity − used)`; never negative.
- Ordered by `percentUsed` **descending**.
- `q` matches client **and** bench name, case-insensitive.
- `channel`, `client` and `q` compose with AND.
- `hoursUsed` counts approved entries only.

### `GET /api/internal/reports/engineer-invoicing?period={YYYY-MM}`

Returns **.xlsx**. One row per engineer per bench: period, engineer, email, client, bench, manager(s), currency, hours, rate, earnings; totals **per currency**.

**Acceptance criteria**
- Rates are engineer-facing (post mark-up removal). `configuredBlendedRate` never appears.
- Any prior period is requestable.
- **No email is sent** — on-demand downloads are silent.
- Same generator as the month-end attachment, so the two can never diverge.


---

## Integration & sync

Nothing on this page owns its own copy of a shared figure.

| Value | Source | Also appears on |
|---|---|---|
| Capacity plan | Bench subscription, per-bench period (SD-3459) | Manager bench page, Engineer overview |
| Hours used | Approved work logs only | Engineer Work Log, Manager bench page |
| Allocation % → hours | `bench_allocation` | Engineer's own ceiling and threshold |
| Overage state | `bench_overage_setting` | Engineer's Log Hours validation copy |
| Client, currency | **Zoho** (SD-3463) | Every money figure, all dashboards |

**Acceptance criteria**
- An entry logged on the Engineer dashboard changes these figures on the next read — no cache, no reconciliation job.
- Capacity and percentage shown here are **identical** to the Manager dashboard for the same bench and period.
- Pending-approval hours are excluded from "hours used" on every surface, consistently.
- Declined entries are withheld from Manager responses; edit history likewise.
- No currency is ever converted.


---

## Client hierarchy — SD-3463

Channel structure is derived from Zoho's **`Parent Brand`** field. Nothing about the tree is stored on the platform.

- **Depth-unbounded**, with siblings at every level.
- Generation labels (`ROOT`, `PARENT`, `CHILD GENERATION 1…n`) are computed from depth **on render** — re-parenting changes a label with no migration.
- Sibling order is deterministic: alphabetical by client name.
- Aggregates roll up the selected client **and all descendants** — never ancestors, never a sibling's subtree.
- A cycle is detected and reported, not followed.

Full logic and worked fixtures: **§H of `../../ENGINEERING-BRIEF.md`**.
