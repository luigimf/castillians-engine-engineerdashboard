# BE — Channel Page: Breakdown & Billing

Backend specification for **SD-3462**.

---

## Endpoints

### `GET /api/internal/channels/{rootClientId}`

Returns the channel's root, its summary aggregates, and the full descendant tree.

```json
{
  "rootClient": { "id": "c-1", "name": "Northmill Bank", "clientType": "VAD", "currency": "EUR" },
  "summary": { "clients": 4, "benches": 3, "engineers": 6, "capacityHours": 477, "billing": 45658.00 },
  "tree": [
    {
      "clientId": "c-1", "name": "Northmill Bank", "clientType": "VAD",
      "depth": 0, "benchCount": 2, "capacityHours": 381, "billing": 36858.00,
      "channelBilling": 45658.00,
      "children": [ { "clientId": "c-2", "depth": 1, "children": [ … ] } ]
    }
  ]
}
```

**Acceptance criteria**
- `depth` is returned; the client derives the generation **label** from it. Do not send the label.
- `channelBilling` is present **only** on depth 0.
- `children` is ordered **alphabetically by name** — deterministic across loads.
- Recursion is depth-unbounded.
- A cycle is detected and returns a **422 naming the clients involved**, rather than recursing.

### `GET /api/internal/channels/{rootClientId}/billing?period={YYYY-MM}`

Per-client bench rows for one period, plus totals and the available period range.

**Acceptance criteria**
- Totals are keyed **by currency**; a channel spanning currencies returns one entry each.
- The **current, open** period is flagged so the client can label it as to-date.
- `earliestPeriod` / `latestPeriod` bound the client's navigation.

### `GET /api/internal/channels/{rootClientId}/billing/export?period={YYYY-MM}`

Returns **.xlsx**, not CSV. One row per bench with its client, plus per-client totals and the grand total. Scoped to this channel only.

---

## Aggregation

- Every figure sums the **selected client and all descendants** — never ancestors, never a sibling's subtree.
- Bench counts, engineer counts and skills are **distinct** across the subtree: an engineer on two benches counts once.
- Capacity comes from each bench's own subscription (per-bench, SD-3459).
- Currency per client from Zoho. **No conversion anywhere.**

---

## Test cases

1. The three fixtures in §H of the brief render with exactly the depths shown.
2. Two PARENT-level brands under one root both return `depth: 1`.
3. Two siblings at depth 3 both return `depth: 3`.
4. Re-parenting a brand changes only its `depth`; no stored label to migrate.
5. A cycle `A→B→A` returns 422 and does not hang.
6. A `Parent Brand` pointing at an archived record → that client is treated as a Root and flagged.
7. An engineer on two benches in the same channel counts **once** in `summary.engineers`.
8. A channel spanning EUR and GBP returns two total entries; no combined figure appears.
9. Sibling order is identical across two consecutive calls.


---

## Client hierarchy — SD-3416 / SD-3417

This story **renders** the hierarchy; it does not build it. Structure, depth and parentage all come from those items.

Full logic, the generation-label table and three worked test fixtures: **§H of `../../ENGINEERING-BRIEF.md`**.

### What this surface must honour

- Built from Zoho's **`Parent Brand`** field, derived on read — nothing about the tree is stored.
- **Depth-unbounded.** No hardcoded generation limit.
- **Siblings supported at every depth** — any number of brands may share a parent, and a parent may itself be a child, so branching occurs at any level.
- Generation labels (`ROOT`, `PARENT`, `CHILD GENERATION 1…n`) are computed from **depth on render**. Re-parenting a brand changes its label with no migration.
- Siblings at equal depth carry the **same** label.
- Sibling order is **deterministic** — alphabetical by client name, so the tree does not reshuffle between loads.
- Aggregates roll up the **selected client and all descendants** — never ancestors, never a sibling's subtree.
- A **cycle** in `Parent Brand` is detected and reported, not followed.
- A `Parent Brand` naming a **missing or archived** record is treated as a Root and flagged.
