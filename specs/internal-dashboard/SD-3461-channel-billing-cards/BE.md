# BE — Channel & Billing: root client cards & report downloads

Backend specification for **SD-3461**.

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** this file: `BE.md` is the backend spec for one story, `BE-nn` is a numbered rule in the brief.

---

## Endpoints

### `GET /api/internal/channels/summary`

One entry per **root client**, each aggregating its whole subtree.

```json
{ "channels": [ {
  "rootClientId": "c-northmill", "rootClientName": "Northmill Group",
  "clientCount": 5, "benchCount": 6, "engineerCount": 14, "skillCount": 23,
  "monthlyCapacityHours": 620,
  "monthlyBilling": [ { "currency": "EUR", "amount": 74250.00 },
                      { "currency": "GBP", "amount": 18600.00 } ]
} ] }
```

**Acceptance criteria**
- Aggregates roll up the root **and all descendants** — never ancestors, never a sibling's subtree.
- `engineerCount` and `skillCount` are **distinct** across the channel.
- `monthlyCapacityHours` sums each bench **against its own period** (BE-02, BE-03), pro-rated first period included.
- `monthlyBilling` is an array **per currency**. Never summed across currencies, never converted.
- Hours pending approval are excluded from every figure.
- Nothing here is stored — the whole payload is derived on read.

### `GET /api/internal/reports/{report}?period={YYYY-MM}`

`report` ∈ `payroll-checklist` | `sfm-supplier-upload` | `client-billing`. Returns **.xlsx**.

**Acceptance criteria**
- **Same generator as the month-end attachment** — one code path, so the on-demand file and the emailed file can never diverge.
- Re-downloading a **closed** period returns a byte-identical file (§G5).
- Any prior period is requestable; the current period is served with a to-date flag.
- **No email is sent.**
- Filenames `castillians-{report}-{YYYY-MM}.xlsx`.
- No report carries a period-split or carry-over column: a period's file holds that period's hours only, and client hours equal supplier hours exactly (BE-29).
- The SFM upload's 22 columns and header row are **untouched** by that change.
- Finance fields are read from **Zoho at generation time** (BE-23) — never copied into the portal.
- A record missing a **mandatory** field **blocks that row** and is named in the response; it never exports as a blank cell that fails on SFM import.

---

## Integration & sync

Every figure is an aggregation of rows that appear elsewhere.

| Value | Source | Also appears on |
|---|---|---|
| Channel membership | Zoho `Parent Brand` lineage (SD-3416, SD-3417) | Channel drilldown (SD-3462), Engagements |
| Capacity | `Included Hours` per bench (SD-3459) | Drilldown, Virtual Benches tab, Manager bench page |
| Hours and overage | Approved work logs only | Payroll Checklist, Client Billing, Internal Work Logs |
| Client amounts | Configured blended rate | Client Billing, invoice PDF |
| Supplier amounts | Configured rate ÷ (1 + mark-up) (BE-08) | Payroll Checklist, SFM upload |
| Currency, finance refs | Zoho (SD-3463, BE-23) | Every money figure, every report |

**Acceptance criteria**
- A card total and the sum of its rows can never disagree — benches, capacity and billing reconcile with SD-3462, SD-3464 and the Manager bench page.
- Supplier amounts use the **engineer-facing** rate and client amounts the **blended** rate; the two are never mixed.
- The SFM upload sets exactly one of `AMOUNT` (EUR) or `FAMOUNT` (other) per row — nothing is converted.
- Hours unresolved at the close are **auto-declined** (BE-29) and enter no report on either side; nothing is billed or paid in a later period.
- A blocked row never silently shrinks a card total.
