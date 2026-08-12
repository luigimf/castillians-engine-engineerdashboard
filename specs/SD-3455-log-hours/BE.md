# BE — Work Log: Log Hours

Backend specification for **SD-3455**. Covers creating a work-log entry: validation, routing
and the notifications that follow.

Depends on the period and allocation endpoints from **SD-3453**.

---

## 1. Endpoint

### `POST /api/engineer/work-logs`

```json
{
  "benchId": "vb-core",
  "date": "2026-08-08",
  "hours": 6,
  "description": "Refactored the payment reconciliation job."
}
```

Response `201`:

```json
{
  "entryId": "wl-8842",
  "status": "auto_approved",
  "message": null
}
```

There is **no** `POST /work-log/{id}/submit`. Routing is automatic — do not build a
client-initiated submit action.

---

## 2. Validation, in order

Return the **first** failure; do not aggregate.

| # | Condition | Status | Message |
|---|---|---|---|
| 1 | `date` after `maxLoggableDate` | `422` | "You can't log hours for a future date." |
| 2 | `date` before `periodStart` | `422` | Names the current period, e.g. "This date falls outside the current period (1 – 31 Aug 2026)." |
| 3 | `hours` ≤ 0 | `422` | "Enter hours between 0.5 and 16." |
| 4 | `hours` > 16 | `422` | Same |
| 5 | `description` empty after trim | `422` | "Describe the work you did." |
| 6 | overage mode `off` **and** projected total > capacity | `422` | States hours remaining, e.g. "Overages are switched off for Core Platform. You can log up to 2h more this period." |

`maxLoggableDate` and `periodStart` come from the period engine — do not recompute them here,
call the same resolver so the client and server never disagree.

Validation is the **gate**. The client performs the same checks for immediacy, but a request
that bypasses the UI must still be refused.

---

## 3. Status assignment

### The auto-accept ceiling

One function owns this. Given `cap = capacityForPeriod` and `block = authorisedTotalOverage`:

```
unlimited        → no ceiling
block > 0        → cap + block          ← the 20% tolerance NO LONGER APPLIES
block = 0        → cap × 1.20           ← tolerance only
overages off     → cap
```

**The block replaces the tolerance, it does not stack on it.** Once the client names an
authorised total, that total governs. A 160 h plan with a 40 h authorised block accepts up to
**200 h**, not 232 h. This is the single rule most likely to be implemented wrongly.

### Status assignment

Let `projected = usedForEngineerShare + hours` and `ceiling = autoCeiling(bench)`.

| Overage mode | Condition | Status | Email |
|---|---|---|---|
| `unlimited` | any | `auto_approved` | none |
| `on` | `projected > ceiling` | `approval_required` | approval request → Castillians team |
| `on` | `cap < projected ≤ ceiling` | `auto_approved` | **none** — accepted silently |
| `on` | `projected ≤ cap` | `auto_approved` | none |
| `off` | `projected ≤ cap` | `auto_approved` | none |
| `off` | `projected > cap` | *rejected at validation step 6* | — |

`1.20` is a **configurable constant**, not a literal. So is the 16-hour daily ceiling.

### Thresholds are per engineer, not per bench

Every comparison above is against the **engineer's own share**, not the bench total:

```
myPlan    = round(cap × allocationPercent)
myCeiling = round(ceiling × allocationPercent)
```

An engineer on 35% of a 160 h plan is measured against 56 h, not 160 h.

### The overage grant is per period

`Allow Overages` is granted for **one period only**. It stays in force for the whole of the
final day — so end-of-period billing closes against the hours that were actually authorised —
then lapses at 00:00 on the first day of the next period.

Store the period it was granted in and compare on read; no scheduled job is needed. When the
grant lapses, `Unlimited Overage` and the authorised block lapse with it.

**Where the overage mode comes from.** The **Allow Overages** and **Unlimited Overage** switches,
and the pre-authorised block beyond 120%, are set by Castillians staff on the **Internal
dashboard** (Engagements > Virtual Benches > bench row). Those controls are specified in **their
own story** and are not part of this one — here the mode is simply read and applied. Until that
tool ships, treat `mode: "on"` with no authorised block as the default.

---

## 4. Side effects on success

1. Append a `Created` record to the entry's history — actor = the engineer, action `Created`,
   no diffs.
2. When routed for approval, append a second record — actor **Castillians System**, action
   `Sent for approval`, detail "monthly hours exceeded".
3. When routed for approval, fire the approval-request email to the Castillians team. This is a
   **distinct template** from the routine work-log-submitted notification.
4. Recompute bench totals. The new entry immediately changes the figures returned to **every**
   engineer on that bench.

---

## 5. Concurrency

Multiple engineers draw from one bench pool, so the capacity check and the insert must be
atomic. Two engineers submitting 3 h each against 4 remaining hours must not both succeed.

A read-then-write without a lock or a conditional insert will let both through under load.
Cover it with a concurrent test, not just a sequential one.

---

## 6. Test cases worth writing first

1. Date one day after `maxLoggableDate` → `422`, future-date message.
2. Date one day before `periodStart` → `422`, message contains the period range.
3. `hours: -19` submitted directly to the API → `422`. (The UI cannot produce this, but the
   endpoint must still refuse it.)
4. `hours: 17` → `422`.
5. `description: "   "` → `422`.
6. Overages `off`, capacity 80, used 78, submit 5 → `422` stating 2 h remain.
7. Overages `off`, capacity 80, used 78, submit 2 → `201`, `auto_approved`.
8. Overages `on`, capacity 160, block 0, used 150, submit 5 → `auto_approved`, **zero emails sent**.
9. Overages `on`, capacity 160, block 0, used 190, submit 5 → `approval_required`, one email sent.
9b. Overages `on`, capacity 160, **block 40**, used 195, submit 4 → `auto_approved` (ceiling 200).
9c. Same bench, used 199, submit 4 → `approval_required` (203 > 200). Assert the ceiling is
    **not** 232 — the tolerance must not stack on the block.
9d. Grant made in period N, read in period N+1 → `overagesOn` returns false with no job having run.
10. Overages `unlimited`, capacity 160, used 400, submit 20 → `auto_approved`, no email.
11. Two concurrent submissions of 3 h against 4 remaining hours → exactly one `201`.
12. Successful create → bench totals visible to a second engineer on the bench change immediately.
13. Entry routed for approval → history contains both `Created` and `Sent for approval`, the
    second with actor `Castillians System`.
