# BE — Organisation Page: Member Roles & Removal

Backend specification for **SD-3479**. Per-bench role changes, the two removal scopes, and the Zoho write-back on organisation-level removal.

Every endpoint here is **Admin only** — `403` for anyone else, whatever per-bench roles they hold.

> **Roles are per bench.** Admin is the only account-wide role. Manager and Viewer are attributes of a **bench membership**, so a role change targets a `(memberId, benchId)` pair — never a member alone. SD-3479's Jira description still carries the earlier account-wide wording; this document is the model to build.

---

## Endpoints

### `PATCH /api/manager/organisation/members/{memberId}/benches/{benchId}`

`{ "role": "manager" | "viewer" }`

**Acceptance criteria**
- **The route is per bench.** There is no `PATCH /members/{memberId}` with a single role — an account-level role change for Manager/Viewer is not a thing the model can express.
- `role: "admin"` is refused `422`. Ownership moves through the transfer endpoint (SD-3478).
- **Refused `422` on the Admin's own membership.** The Admin holds no per-bench roles; their access comes from the account role.
- **Immediate — no approval step.**
- **Demoting the last `manager` on a bench is refused `409`**, and the error **names the bench**. A bench must keep at least one Admin or Manager who can act on it (SD-3471).
- **Scoped write.** Assert that the member's role on **every other bench** is unchanged, and that their brand association, account and logged work are untouched.
- Promotion to `manager` makes the member eligible to **review that bench's engineers** on the Performance Log; demotion to `viewer` removes that eligibility **for that bench only** (SD-3471).
- A member with **`status: "invited"`** may have their per-bench roles changed; the change updates **what the invitation will grant** and creates no user.
- Idempotent: setting the role a member already holds returns `200` and writes nothing.

### `DELETE /api/manager/organisation/members/{memberId}/benches/{benchId}` — remove from a bench

**Acceptance criteria**
- Removes **one** bench membership. The member keeps their account, their brand association, and **their other benches at their existing roles** — assert the latter explicitly.
- **Removing the last Admin or Manager with access is refused `409`**, naming the bench.
- **Does not touch Zoho.** They remain a contact on that brand's client record. Assert no Zoho call is made.
- Unsubscribes them from that bench's notifications, and removes it from their bench list (SD-3470), on the next read.
- **Immediate.** No pending state.

### `DELETE /api/manager/organisation/members/{memberId}` — remove from the organisation

**Acceptance criteria**
- Removes the member from the account: **every bench**, and their ability to sign in.
- **Refused `422` when the caller is the target.** The Admin cannot remove themselves; they must transfer ownership first (SD-3478). The absent control is not the protection.
- Refused `409` where it would leave a bench with **no Admin or Manager** — the error names the bench.
- **Writes back to Zoho** (INT-4): the manager profile on the brand's client record is updated. **If the Zoho write fails the platform removal still stands**, and the failure is surfaced to the Internal team — reported, never swallowed.
- For a member with `status: "invited"`, this **cancels the invitation**: the token is invalidated and the sign-up link stops working. No user was ever created.
- **Deletes none of their work.** Bench notes, capacity and engineer requests they raised, work-log history they appear in, and the bench's order form history (SD-3465) are all retained. Assert that a removal writes no delete to any of them.

---

## Concurrency

**Acceptance criteria**
- **Two Admins cannot exist**, and the account cannot end with none — enforced at the account record, not in application code paths (SD-3478).
- **A bench cannot be left with nobody who can act on it.** Two concurrent operations that would each individually be legal — demoting the second-to-last Manager and removing the last one — must not both succeed. Assert with a concurrent test, not a sequential one.
- Because at least one Admin or Manager must retain access, **a bench always retains at least one eligible Performance Log reviewer**. Assert both readings of the rule in one test (SD-3471).

---

## Integration & sync

| Value | Source | Also appears on |
|---|---|---|
| Role on a bench | **Bench membership** — written here | The chip here, the bench's Members with access card (SD-3471), what that bench offers that member |
| Bench access | Bench membership (SD-3471) | The member's bench list (SD-3470), notification recipients |
| Member ↔ brand association | **Zoho** manager profile (INT-4) | This page; updated on organisation-level removal only |
| Pending invitations | Invitation record + single-use token (SD-3471) | The `invited` status here |
| Performance Log eligibility | Derived from the **role on that bench** + access | Who may review that bench's engineers |

**Acceptance criteria**
- **One membership record.** The role served here and the role served by the bench page are the same field — there is no second store, and no client-side reconciliation.
- Every dependent surface is **recomputed on the next read**, never patched client-side: the bench pages, the member's bench list, notification recipients, and reviewer eligibility.
- **Bench-scope removal never writes to Zoho; organisation-scope removal always does.** Assert both directions.

---

## Test cases

| # | Case | Expected |
|---|---|---|
| 1 | Demote member on bench A; they are Manager on bench B | `200`; bench B role unchanged |
| 2 | Demote the last Manager on a bench | `409` naming the bench; role unchanged |
| 3 | `PATCH` with `role: "admin"` | `422` |
| 4 | `PATCH` on the Admin's own membership | `422` |
| 5 | Promote a Viewer to Manager on one bench | Request controls + reviewer eligibility on that bench only |
| 6 | Remove from a bench | That bench gone; other benches and roles intact; **no Zoho call** |
| 7 | Remove the last Manager with access from a bench | `409` naming the bench |
| 8 | Admin removes themselves | `422` |
| 9 | Remove from the organisation | Account gone; **Zoho profile updated** |
| 10 | Same, with Zoho unavailable | Removal stands; failure reported, not swallowed |
| 11 | Remove a pending invitee | Invitation cancelled; token invalid; no user was created |
| 12 | Change a pending invitee's role on a bench | `200`; the invitation's grant updated; still no user |
| 13 | After any removal | Their requests, notes and work-log history all still present |
| 14 | Concurrent demote + remove leaving a bench empty of Managers | One fails; the bench keeps an Admin or Manager |
| 15 | Manager or Viewer calls any endpoint here | `403` |
