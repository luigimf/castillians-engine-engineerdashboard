# BE — V Bench page: Bench Setup & Sharing

Backend specification for **SD-3471**.

---

## Endpoints

### `GET /api/manager/benches/{benchId}`

Returns the bench, its plan and period, its members, and any pending requests. **No `timeZone` field** — it is removed from the record, not merely hidden.

**Acceptance criteria**
- Refused `403` when the caller is not a member (Manager, Viewer) or not in the organisation (Admin).
- Carries `canRename`, `canRequestCapacity`, `canAddMember`, `canInvite`, `canRemoveMember` — resolved server-side from the caller's role.
- `capacityPlanHours` is returned **with its period** (`periodStart`, `periodEnd`), so the client never infers which period a figure belongs to.
- The **client-facing blended rate** is returned for the cost implication. `configured ÷ (1 + mark-up)` and the retained margin are **never** in this response (BE-04, BE-08) — assert it in a test.

### `PATCH /api/manager/benches/{benchId}` — rename

**Acceptance criteria**
- Accepts `name` only. **Refused for a Viewer** (`403`).
- Stored **verbatim**; leading/trailing whitespace trimmed. Empty or all-whitespace → `422`, previous name kept.
- **Unique within the organisation** → `409` naming the existing bench.
- **Immediate — no approval step.**
- Writes the name and nothing else. Work logs, roster, capacity, periods and history are untouched; assert that a rename produces no other row change.
- The new name is served by every other surface on the next read — Internal Engagements, Channel page, and every subsequent report. **One name, held once.**

### `POST /api/manager/benches/{benchId}/help`

**Acceptance criteria**
- Available to **all three roles**.
- `message` mandatory, non-whitespace → `422` otherwise.
- The server attaches bench, client and caller identity; the client never sends them, and cannot spoof them.

### `POST /api/manager/benches/{benchId}/capacity-requests`

**Acceptance criteria**
- **Refused for a Viewer** (`403`).
- Creates a request record. **No write to the subscription** — assert it.
- An open request is returned on the bench read until actioned, so the client can render it as pending.
- A second identical request while one is open is refused `409` — the pending state is not a suggestion.

### Membership — `POST` / `DELETE /api/manager/benches/{benchId}/members`

**Acceptance criteria**
- Add existing → Admin and Manager. **Invite new → Admin and Manager.** Remove → **Admin only.** Enforced `403` server-side, not by the absent button.
- An invitation may grant **`manager` or `viewer` only**. A request to invite an **Admin** is refused `422` — account ownership moves through Change Account Admin, never an invitation.
- An invitation's email domain is checked against **that brand's own** allow-list (§A4, INT-10) **server-side** → `422` on mismatch. Domains are never pooled across the channel.
- The allow-list is keyed **per client record**, so a brand newly parented into the lineage in Zoho arrives with its **own Email Domains column** — derived from the client records, with no configuration step. A brand with no domains yet returns an empty list rather than being omitted.
- An invitation **creates no user**. It creates a pending invitation record holding the organisation, the invitee's address, their role, and the benches the Admin granted — and issues a **single-use token**.
- The email links to the **Manager sign-up page** with that token, never to login. The invitee has no account to log into.
- **The token carries the grant.** On completing Manager onboarding the platform creates the user, attaches them to the organisation with the invited **role**, and grants exactly the **bench access** recorded on the invitation. The invitee cannot alter either.
- The invitation records **who invited them**, so a grant made by a Manager is auditable, but the resulting access is indistinguishable from one an Admin made — one membership model, no second class of grant.
- Sign-up is refused unless the address matches the invitation **exactly** (BE-21) → `422`.
- The token is **single-use and expiring**; a spent or expired token returns a state the UI can explain, not a generic error.
- Until onboarding completes the invitee reads as **pending** in the members list and **no bench response includes them** — they can see nothing.
- Once complete, their Virtual Benches list (SD-3470) returns **only their granted benches**, filtered server-side — the union of every grant made by an **Admin or a Manager**, in one list.
- Changing their access afterwards is a **membership** change, not a new invitation — one record, read by both the list and the notifications.
- **Onboarding also writes a manager profile onto the Zoho client record for the brand the invitation was scoped to** — the brand named on the bench, not the root client. It is written **once, at completion**; a pending invitation writes nothing to Zoho.
- The Zoho contact and the platform user are **one identity**: the address on the profile is the invited address, so the two never drift.
- If the Zoho write fails, the account is still created and the failure is **reported rather than swallowed** — the person can work, and our record is visibly incomplete rather than silently wrong.
- **Removing the last Admin or Manager with access is refused** `409`. A bench nobody can act on is a dead end.
- Granting access makes the bench visible on that member's list immediately, and **subscribes them to its notifications**. Removing it does both in reverse — one membership, read by both.
- Removing bench access never removes the member from the organisation.
- **A bench has exactly one Admin.** There is one Admin per client and a bench belongs to exactly one client, so the members response for a bench carries **exactly one member with `accountRole: "admin"`** — that client's. Another client's Admin in the same channel is **absent** from the response, not merely hidden. Scope the roster to the bench's own `clientId` before applying any access filter; a channel-wide roster is the wrong starting point.
- **The account Admin is an eligible reviewer on every bench belonging to their client**, returned in the reviewer list beside the Managers. Eligibility derives from the **account role**, not from a bench membership row, so there is nothing to grant and revoking bench access cannot remove it. Assert that an Admin appears as eligible on a bench they hold no membership row for.
- **Bench membership is what makes a Manager eligible to review that bench's engineers** on the Performance Log. One membership record, read for both purposes — there is no separate reviewer assignment to keep in step.
- A **Viewer** is never an eligible reviewer, on any bench.
- Because a bench has exactly one Admin and that Admin is always eligible, **a bench always retains exactly one always-eligible reviewer** — even with every Manager removed. Assert both readings of that rule in one test, plus that two `admin` rows can never appear for one bench.

---

## Integration & sync

| Value | Source | Also appears on |
|---|---|---|
| Bench name, type | The bench record — **renamed here** | List page, Internal Engagements, Channel page, every report |
| Capacity plan | `Included Hours` (SD-3459) — **set by the Castillians team** | Hours card, list page, Internal, Channel page |
| Period | Per-bench Start + Auto-Renew (SD-3459) | Every period label |
| Client-facing rate | Configured blended rate (BE-08) | Cost implication here, Channel page billing |
| Membership | Organisation + bench membership (§A4) | List visibility, notification recipients |
| Domain allow-list | Manage Client modal (§A4, INT-10), held **per brand** | Every invitation on the platform; one Email Domains column per brand |
| Invitation → account | Pending invitation + single-use token; resolved by Manager onboarding | The invitee's own Manager dashboard and bench list (SD-3470) |
| Manager profile on the client record | Written to **Zoho** at onboarding completion, against the **brand** the bench belongs to (SD-3463) | Zoho client contacts; every report that names a client contact |
| Capacity requests | Written here, actioned on Internal (SD-3465) | Internal bench entry's order form history |

**Acceptance criteria**
- **Nothing here writes to a subscription.** A capacity request changes the plan only when the Castillians team actions it — at which point the new figure appears on this page, the list page, Internal and the Channel page **together**.
- A rename propagates everywhere on the next read and touches nothing else.
- **`timeZone` is gone from the record**; no response carries it after this ships.
