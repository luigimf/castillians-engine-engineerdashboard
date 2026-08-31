# BE — Organisation Page: Functions

Backend specification for **SD-3478**. The ownership transfer, the brands-and-members read, invitations, the new-brand request, and per-brand email domains.

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** this file: `BE.md` is the backend spec for one story, `BE-nn` is a numbered rule in the brief.

Every endpoint here is **Admin only** and refused `403` for anyone else — no combination of per-bench Manager or Viewer roles grants any of it.

---

## Endpoints

### `GET /api/manager/organisation/brands`

```json
{
  "brands": [
    { "brandId": "cl-1", "name": "Northmill Bank", "isRoot": true,
      "members": [
        { "memberId": "mem-1", "name": "James Whitmore", "email": "james@northmill.com",
          "accountRole": "admin", "benchAccess": [], "status": "active" },
        { "memberId": "mem-2", "name": "Claire Bonnici", "email": "claire@northmill.com",
          "accountRole": null, "status": "active",
          "benchAccess": [ { "benchId": "vb-core", "name": "Core Platform", "role": "manager" },
                           { "benchId": "vb-pay",  "name": "Payments Squad", "role": "viewer" } ] } ] },
    { "brandId": "cl-3", "name": "Northmill Markets", "isRoot": false, "members": [] }
  ],
  "pendingBrands": [ { "requestId": "br-1", "name": "Northmill Wealth", "requestedAt": "2026-08-20T09:12:00Z" } ]
}
```

**Acceptance criteria**
- `brands` is the **Zoho `Parent Brand` lineage** (SD-3416, SD-3417), root first then children. The platform holds **no parallel list of brands** — assert that re-parenting in Zoho changes this response with no migration.
- `members` are the **Zoho manager profiles** on that brand's client record.
- **`benchAccess` carries a role per bench.** Manager and Viewer are per-bench roles; there is no single `role` field for them, because a member can be a Manager of one bench and a Viewer of another. Returning one flattened role is the bug this shape exists to prevent.
- `accountRole` is `"admin"` or `null`. **Admin is the only account-wide role**, and an Admin's `benchAccess` is empty — they reach every bench in the channel by virtue of the account role, not by grants.
- **Exactly one member across all brands has `accountRole: "admin"`.**
- A brand with no members returns **`members: []`** — never omitted from `brands`.
- `pendingBrands` are requests our team has not yet created in Zoho. They carry **no `brandId`**, so there is nothing for the client to open.
- **No pagination.** The full lineage and full membership are returned in one read.

### `POST /api/manager/organisation/admin-transfer`

`{ "toMemberId": "mem-2" }`

**Acceptance criteria**
- Candidates are members who **hold at least one bench as Manager**. A member who is only ever a Viewer is **not** a valid target `422` — they must be made a Manager on a bench first (SD-3479).
- **Atomic.** The account has exactly one Admin at every observable moment. Two concurrent transfers: one succeeds, the other `409`. Never two Admins, never none — assert under concurrency.
- The outgoing Admin becomes a **Manager**, keeping every bench they had access to. Their existing per-bench Viewer roles are unchanged.
- Moves **subscription ownership, billing and this page** in **one write** — never three that can partially fail.
- On the outgoing Admin's next read: `403` on this page and no menubar entry (SD-3477). On the incoming Admin's: both.
- Both parties are notified.

### `POST /api/manager/organisation/brands/{brandId}/invitations`

`{ "email": "…", "benchAccess": [ { "benchId": "vb-core", "role": "manager" } ] }`

**Acceptance criteria**
- `benchAccess` carries **a role per bench**. An invitation with a bare account-level role is rejected `422` — the grant is per bench.
- Roles may be **`manager` or `viewer` only**. `admin` is refused `422`; ownership moves through the transfer endpoint.
- The email domain is checked against **that brand's own** allow-list (§A4, INT-10) **server-side** → `422` on mismatch. **Domains are never pooled across the channel.**
- Every `benchId` must belong to **that brand** → `422` otherwise. An invitation scoped to one brand cannot grant a sibling brand's bench.
- **Creates no user.** It creates a pending invitation holding the brand, the address, and the per-bench grants, and issues a **single-use expiring token** (SD-3471).
- Until onboarding completes the invitee reads as **`status: "invited"`** and **no bench response includes them**.
- Onboarding applies the recorded per-bench grants exactly; the invitee cannot alter them, and must register with the **exact invited address** (BE-21).
- On completion a **manager profile is written to the Zoho client record for that brand** — not the root client (SD-3471).

### `POST /api/manager/organisation/brand-requests`

`{ "name": "…", "parentBrandId": "cl-1" | null, "message": "…" }`

**Acceptance criteria**
- **Provisions nothing.** No client record, no billing, no benches, no domains. Assert the only row written is the request itself.
- `parentBrandId: null` means "above all my other brands" — still **inside the channel**. There is no unparented option; an unparented brand would be a separate account.
- The request is returned in `pendingBrands` until our team creates the Zoho record and parents it, at which point it appears as a **real brand** and leaves `pendingBrands` on the next read.
- A duplicate open request for the same name is refused `409` — the pending row is there so nobody asks twice.

### `GET` / `PUT /api/manager/organisation/brands/{brandId}/domains`

**Acceptance criteria**
- Domains are held **per brand**, keyed to that brand's **Zoho client record** (BE-23) — not a configured platform-side list. A brand newly parented in Zoho returns an **empty list**, never a 404.
- **Removing the last domain is refused `422`** — nobody could then be invited to that brand.
- A domain governs **only that brand's** invitations. Assert that an address on brand A's domain is refused for brand B.

---

## Integration & sync

| Value | Source | Also appears on |
|---|---|---|
| Brands in the channel | **Zoho** `Parent Brand` lineage (SD-3416, SD-3417) | Internal Channel page (SD-3462), the Email Domains columns |
| Member ↔ brand association | **Zoho** manager profile on the client record | This page; written at onboarding (SD-3471) |
| Bench access + role per bench | Bench membership (SD-3471) | The bench's Members with access card, the member's bench list (SD-3470) |
| Who the Admin is | The account record (§A4) | This page's visibility (SD-3477), subscription ownership |
| Email domains | **Zoho** client record, per brand (§A4, INT-10, BE-23) | Every invitation on the platform |
| New-brand request | Written here, actioned in **Zoho** | `pendingBrands`, then the brand itself |

**Acceptance criteria**
- **One membership record, two surfaces.** Granting access on a bench page (SD-3471) adds a `benchAccess` entry here; removing it here removes it there. There is no second store to reconcile.
- **The role on a `benchAccess` entry is the same value the bench page serves** for that member on that bench. Read once, rendered twice.
- Zoho is authoritative for brands and for who belongs to which brand; the platform is authoritative for **bench access**.
- The Admin transfer moves ownership and page access **together**.

---

## Test cases

| # | Case | Expected |
|---|---|---|
| 1 | Member who manages bench A and views bench B | Two `benchAccess` entries, roles `manager` and `viewer` |
| 2 | Admin's own row | `accountRole: "admin"`, `benchAccess: []` |
| 3 | Brand with no members | Present in `brands` with `members: []` |
| 4 | Transfer to a Viewer-only member | `422` |
| 5 | Two transfers at once | One `200`, one `409`; exactly one Admin after |
| 6 | Invite with `role: "admin"` | `422` |
| 7 | Invite with an address on a sibling brand's domain | `422` |
| 8 | Invite naming a bench of another brand | `422` |
| 9 | Pending invitee | `status: "invited"`; absent from every bench response |
| 10 | Remove a brand's only domain | `422`, domain retained |
| 11 | New-brand request | Request row only; no client, billing, bench or domain rows |
| 12 | Duplicate open brand request | `409` |
| 13 | Manager or Viewer calls any endpoint here | `403` |
