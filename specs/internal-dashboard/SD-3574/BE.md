# BE — Engagements: "Managed by" in the expanded bench entry

Backend notes for **SD-3574** (Enhancement, epic SD-3531). Extends **SD-3465**.

## Data

The expanded bench entry read (SD-3465) gains a `manager` object:

```json
"manager": {
  "name": "Claire Bonnici",
  "email": "claire.bonnici@northmill.com",
  "phone": "+44 7700 900654",
  "role": "Manager"
}
```

| Field | Source of truth |
|---|---|
| `name`, `email`, `phone` | Manager's platform account (synced from the Zoho contact) |
| `role` | The manager's role **on this bench**: `Admin` / `Manager` / `Viewer` |
| Which manager | Platform bench assignment, the **same source** Channel Breakdown uses (SD-3462) |

## Rules

- Returned in the **same response** as the rest of the bench entry. No extra request per accordion open.
- `phone` may be `null`. Never send a placeholder string.
- A bench without a manager is a data error. Log it and return `manager: null` rather than failing the whole read.

## Acceptance criteria

- For any bench, `manager` equals the manager shown on Channel Breakdown for that bench.
- Changing the bench's manager is reflected on both pages on the next read.
- No N+1: opening ten benches makes no extra manager lookups.
