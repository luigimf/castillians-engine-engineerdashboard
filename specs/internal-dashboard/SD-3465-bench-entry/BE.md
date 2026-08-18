# BE — Engagements: expanded bench entry

Backend specification for **SD-3465**.

---

## Overage state machine

A bench is in exactly one state. Resolve it in this order:

```
if (unlimited)            → ceiling = ∞
else if (!allowOverages)  → ceiling = capacity
else if (block > 0)       → ceiling = capacity + block     // REPLACES the tolerance
else                      → ceiling = capacity × 1.20
```

**Acceptance criteria**
- An authorised block **replaces** the 20% tolerance. 160h plan + 40h block → **200h**, not 232h. The single most likely misreading.
- `1.20` is a **named constant**, not a literal.
- With `unlimited`, `block` reads `0` — never a stale value.
- The grant is scoped to the current period and lapses at **00:00 on day one of the next**, so end-of-period billing closes against what was authorised.
- Flipping any of these recomputes every engineer's ceiling on next read.

## Allocation

`bench_allocation` — `bench_id`, `engineer_id`, `allocation_percent`.

**Acceptance criteria**
- Shares validate to **exactly 100%** on save; anything else is rejected.
- A single-engineer bench is implicitly 100% and immutable.
- Engineer-scoped responses return **hours only** — the percentage never reaches them.
- `engineerCeiling = allocationPercent × ceiling`.

## Work logs

- Batched at **10**, newest first, with a remaining count for the See more control.
- Decline requires a non-empty message; rejected otherwise.
- Approve → entry counts towards billing, appears on Manager, engineer emailed.
- Decline → stays on Internal and Engineer with the message, **excluded from Manager**.

## Notes & order forms

- Notes are internal-only; never returned to Manager or Engineer.
- Order forms are created by the Manager overage-request flow and read here. Period is stored and displayed as a **full date range**, never a bare month.


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
