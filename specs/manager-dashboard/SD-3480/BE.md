# BE — Subscriptions Page: Access

Backend specification for **SD-3480**.

> **`BE-nn` refers to numbered sections in `specs/ENGINEERING-BRIEF.md`**, not this file.

---

## 1. Guard

- `GET /subscriptions` → `403` for any member whose account role is not `admin`. Resolved server-side on every request.
- `canViewSubscriptions` is derived from the account record on every read and feeds **both** the menubar entry and the route guard — one derivation, two consumers.
- Transferring the Admin role (SD-3478) moves access on the next read.

## 2. Scope — the subtree rule

> **Changed 23 Sep.** Previously: one Admin per client, so this page was visible to exactly one person.

```
scope(admin) = [admin.brand] + descendants(admin.brand)   // via Zoho Parent, read at request time
```

- Resolve `descendants` from the **Zoho Parent lineage at read time**. No portal-side copy, no cache that outlives the request.
- Every response on this page returns **only** brands and benches in `scope`. Out-of-scope data is **absent**, not flagged.
- The same `scope()` is used by Virtual Benches (SD-3470) and Organisation (SD-3477). **One function, three consumers.**
- A brand with no Admin of its own is still covered: it is in the scope of every Admin above it.
- Re-parenting a brand in Zoho changes scope on the next read for every affected Admin.

## 3. Response

```json
{
  "brands": [
    { "clientId": "c-ins", "name": "Northmill Insurance", "benches": [ /* SD-3481 */ ],
      "totals": [ { "currency": "EUR", "amount": "6000.00" } ] }
  ]
}
```

- `totals` is **per currency** (BE-27). Never combined, never converted.
- Never carries the engineer-facing rate, mark-up or margin (BE-04, BE-08).

## 4. Tests

- Root Admin: every brand in the channel.
- Insurance Admin: Insurance, Capital, Wealth — **assert Bank is absent**.
- Wealth Admin: Wealth only.
- Manager / Viewer: `403`.
- Re-parent Capital under Bank in Zoho → Insurance Admin loses Capital and Wealth on the next read.
