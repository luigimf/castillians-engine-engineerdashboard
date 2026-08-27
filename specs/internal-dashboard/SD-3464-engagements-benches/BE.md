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

---

## Overage status and the resolved allowance — added 27 Aug

The bench row response carries **both** the plan and the allowance, plus the overage state as an enum. The table's column order is **CAPACITY PLAN → OVERAGE → REMAINING** (SD-3464 FE), and the payload is ordered to match.

```json
{
  "benchId": "vb-ins", "name": "Insurance Web", "client": "Northmill Insurance",
  "capacityPlanHours": 80, "hoursLogged": 102,
  "overageState": "off" | "tolerance" | "authorised" | "unlimited",
  "authorisedOverageHours": 0,
  "allowanceHours": 80,
  "remainingHours": 0
}
```

**Acceptance criteria**
- `overageState` is an **enum, not a rendered string.** The client maps it to copy, so the label can be reworded without a backend change. `authorisedOverageHours` supplies the X in "On (X hours authorised)".
- **The four states are mutually exclusive and exhaustive.** Assert that a bench always returns exactly one, including where a stale authorised block sits behind a since-disabled Allow Overages — the state is `off` and the block is not reported.
- `allowanceHours` is the **resolved ceiling**: plan for `off`, 120% of plan for `tolerance`, plan + block for `authorised`, and **null** for `unlimited`. Resolved **server-side** — the client never recomputes a ceiling from a percentage.
- **An authorised block replaces the tolerance rather than stacking on it** (BE-13): an 80h plan with a 40h block gives `allowanceHours: 120`, not 136.
- `remainingHours` is `allowanceHours − hoursLogged`, and **null** for `unlimited`. **It may be negative**, and the client renders a negative value as `Nh over` rather than `0h`.
- **Do not clamp `remainingHours` at zero.** `allowanceHours` is the **auto-approve threshold**, not a hard cap (BE-13): hours beyond it are accepted and queued for approval, so `hoursLogged` legitimately exceeds it. Clamping discards the overshoot, and the client cannot distinguish "exactly at the ceiling" from "5h past it".
- Worked example — an 80h plan on the plain tolerance with **101h approved**: `allowanceHours: 96`, `remainingHours: -5`. The client shows `5h over` against `of 96h auto-approved`. All 101h are approved and billable; 5h of them required a signature.
- `capacityPlanHours` is the plan **before** overage, resolved against the bench's own period including a pro-rated first one (SD-3459). It is the denominator of the CAPACITY column, so it must not silently become the allowance.
- **The same overage state drives the engineer's logging validation** (SD-3455) and the bench entry's controls (SD-3465) — one derivation, three consumers, so a bench can never read "On (Unlimited)" here while refusing an engineer's hours.
- Hours logged counts **approved entries only**, as everywhere else.

| # | Case | Expected |
|---|---|---|
| 1 | Overages off, 102h on an 80h plan | `off`, allowance 80, remaining **−22** |
| 2 | Tolerance, 101h on an 80h plan | `tolerance`, allowance 96, remaining **−5** — rendered `5h over` |
| 3 | 40h authorised, 102h on an 80h plan | `authorised`, allowance **120**, remaining **18** |
| 4 | Unlimited, any hours | `unlimited`, allowance **null**, remaining **null** |
| 5 | Block set, then Allow Overages switched off | `off`; the block is not reported |
| 6 | Bench in its pro-rated first period | Plan is the pro-rated figure; allowance derives from it |
