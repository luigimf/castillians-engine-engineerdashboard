# BE — Work Log: Virtual Bench Tabs & Overview

Backend specification for **SD-3453**. Everything here feeds the page frame: which benches
the engineer sees, which period is current, how many hours they personally hold, and what
they earn.

> The period engine and capacity logic below are **shared infrastructure** — the Internal
> dashboard (Engagements) and Manager dashboard (bench detail) read the same data. Build and
> test them before any UI.

---

## 1. Integration points

| System | Field | Use |
|---|---|---|
| Zoho CRM | Client record → currency | Money formatting |
| Internal Dashboard → Manage Subscription | `Included Hours` | Bench capacity plan |
| Internal Dashboard → Manage Subscription | `Configured Blended Hourly Rate` | Basis for the engineer rate |
| Internal Dashboard → Mark-Ups & Discounts | `Percentage Mark-up` | Divisor for the engineer rate (currently 60%) |

None of these are re-keyed or stored by this module — they are read at request time.

---

## 2. Endpoints

### `GET /api/engineer/benches`

Virtual Benches the authenticated engineer is **activated** on. Shortlisted-only benches
are excluded — this drives the tab row.

```json
[
  {
    "benchId": "vb-core",
    "benchName": "Core Platform",
    "clientName": "Northmill Bank",
    "benchType": "Work From Anywhere",
    "managers": [
      { "name": "James Whitmore", "email": "james.whitmore@northmill.com" },
      { "name": "Claire Bonnici", "email": "claire.bonnici@northmill.com" }
    ]
  }
]
```

A bench **always** has at least one manager — it drives the Performance Log. Treat an empty
`managers` array as a data error, not a valid state.

---

### `GET /api/engineer/benches/{benchId}/period`

Resolves the bench's **current** subscription period.

```json
{
  "periodStart": "2026-08-01",
  "periodEnd": "2026-08-31",
  "isFirstPeriod": false,
  "daysRemaining": 21,
  "capacityHours": 160,
  "minLoggableDate": "2026-08-01",
  "maxLoggableDate": "2026-08-10"
}
```

**Rules**

- Periods are **calendar months**. The **first** period runs `startDate` → last day of the
  **following** calendar month.
- First-period capacity is pro-rated:
  `round(includedHours × (daysRemainingInStartMonth ÷ daysInStartMonth)) + includedHours`
- `daysRemainingInStartMonth` is **inclusive** of the start date.
- `maxLoggableDate` is the earlier of today and `periodEnd`. The client must not compute this.

**Worked examples**

| Plan | Start | First period | First capacity | Then |
|---|---|---|---|---|
| 160 h/month | 13 Jun 2026 | 13 Jun – 31 Jul | `96 + 160 = 256` | 1 Aug – 31 Aug @ 160 |
| 60 h/month | 15 Feb 2026 | 15 Feb – 31 Mar | `30 + 60 = 90` | 1 Apr – 30 Apr @ 60 |
| 120 h/month | 1 Sep 2026 | 1 Sep – 31 Oct | `120 + 120 = 240` | 1 Nov – 30 Nov @ 120 |

The third row is the case people miss: a subscription starting on the 1st **still** gets the
two-month first period. It is not special-cased.

---

### `GET /api/engineer/benches/{benchId}/allocation`

The engineer's **own** ceiling and usage. Drives four of the overview tiles and the capacity bar.

```json
{
  "allocatedHours": 96,
  "loggedHours": 62,
  "remainingHours": 34,
  "percentUsed": 65,
  "pendingApprovalHours": 8,
  "overageMode": "on",
  "unlimited": false
}
```

**Rules**

- Allocation is stored server-side as a **percentage** of bench capacity. This endpoint
  returns **hours only** — the percentage must never reach the engineer client.
- `allocatedHours = allocationPercent × capacity × 1.20` when `overageMode = "on"`,
  else `× capacity`.
- `loggedHours` **excludes** hours awaiting approval. Those are reported separately in
  `pendingApprovalHours` so the UI can note them without counting them.
- `remainingHours = max(0, allocatedHours − loggedHours)`.
- When `unlimited` is true, return `allocatedHours: null` and `remainingHours: null`. The UI
  **omits those tiles entirely** rather than printing a placeholder — an unbounded ceiling shown
  to an engineer reads as licence. Do **not** return `Infinity`; it does not survive JSON.
- Shares per bench validate to **exactly 100%**. A single-engineer bench is implicitly 100%
  and cannot be set to anything else.
- Recomputed on every read, so changing bench capacity or flipping the overage mode is
  reflected with no manual re-entry.

**Why percentages.** The 20% tolerance is itself a percentage, so storing shares the same way
avoids mixed-unit conversion, and shares auto-scale when capacity changes.

**Where allocation is set.** Only relevant when a bench carries **more than one engineer** — a
single-engineer bench is implicitly 100%. The tool for setting those shares lives on the
**Internal dashboard** (Engagements > Virtual Benches > bench row) and is specified in its own
story. This story only **consumes** the resulting figure: it reads the engineer's share and
returns it as hours. Until that tool ships, an even split across the bench's engineers is a
sound default.

---

### `GET /api/engineer/benches/{benchId}/rate`

```json
{
  "currency": "EUR",
  "ratePerHour": 59.38,
  "earningsThisPeriod": 3681.56
}
```

**Rules**

- `ratePerHour = configuredBlendedRate ÷ (1 + markup)`. Configured 95.00 at a 60% mark-up
  → **59.38**.
- `markup` is read at request time, so changing it on the Mark-Ups & Discounts tab changes
  this response with **no deploy**.
- `earningsThisPeriod = loggedHours × ratePerHour`, using the same `loggedHours` as
  `/allocation` — pending hours are excluded from earnings too.
- **`configuredBlendedRate` must never appear in an engineer-scoped response.**
- An engineer is a **supplier with one currency** on their record. Their rate and earnings
  are always in that currency regardless of which clients' benches they work on — there is
  never more than one total to reconcile.

---

## 3. Data model additions

**`bench_allocation`** — `bench_id`, `engineer_id`, `allocation_percent`

Shares per bench must total exactly 100. Enforce on write, not on read.

**`bench_overage_setting`** — `bench_id`, `mode` (`off` | `on` | `unlimited`), `authorised_block_hours`

`authorised_block_hours` is the pre-authorised overage beyond 120%, split by the same
allocation shares. It is ignored entirely when `mode = "unlimited"` — return 0, do not carry
a stale value.

**Where the overage mode is set.** Like allocation, the controls for **Allow Overages**,
**Unlimited Overage** and the authorised block live on the **Internal dashboard**
(Engagements > Virtual Benches > bench row) and are specified in **their own story**. This
story only reads the resulting mode to decide whether a ceiling applies and what to return as
`allocatedHours`. Until that tool ships, default new benches to `mode: "on"` with no
authorised block.

---

## 4. Test cases worth writing first

1. 160 h/month starting 13 Jun 2026 → first period 13 Jun – 31 Jul, capacity 256 h.
2. Same bench, one period later → 1 Aug – 31 Aug, capacity exactly 160 h.
3. 60 h/month starting 15 Feb 2026 → 15 Feb – 31 Mar, capacity 90 h.
4. 120 h/month starting 1 Sep 2026 → 1 Sep – 31 Oct, capacity 240 h (start-of-month is not special-cased).
5. `maxLoggableDate` on a bench whose period ends in the future → returns today, not the period end.
6. `maxLoggableDate` on a bench whose period has ended → returns the period end.
7. Two engineers at 65% / 35% on a 96 h bench with overages on → 74 h and 40 h respectively.
8. Single-engineer bench → allocation reads 100%, and an attempt to set 80% is rejected.
9. Bench flipped to `unlimited` → `allocatedHours` and `remainingHours` are both `null`, no `Infinity` anywhere in the payload, and no string fallback such as "No cap" is emitted by the API.
10. Engineer with 8 h awaiting approval → `loggedHours` excludes them, `pendingApprovalHours` is 8, earnings exclude them.
11. Configured rate 95.00 at 60% mark-up → 59.38. Change mark-up to 50% → 63.33, no deploy.
12. Grep every engineer-scoped response for `configuredBlendedRate` → absent.


---

## Interaction patterns

Platform-wide rules for pagination contracts, empty/error states, validation order, optimistic updates, concurrency and money are specified once in **§G of `ENGINEERING-BRIEF.md`**. Read it before implementing — it covers the FE/BE interaction cases this file does not repeat.

### For this story

- No paginated endpoint here — all four reads return a single object.
- Every read is **recomputed per request**. Do not cache allocation or rate responses: shared-pool figures change when any engineer on the bench logs hours (§G4).
- `allocatedHours: null` for unlimited overage — **never** `Infinity`, which does not survive JSON.
- An empty result is not an error: a bench with no logged hours returns zeros, not a 404.
