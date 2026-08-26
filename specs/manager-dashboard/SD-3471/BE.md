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
- Add existing → Admin and Manager. **Invite new → Admin only.** Remove → **Admin only.** Enforced `403` server-side, not by the absent button.
- An invitation's email domain is checked against the organisation's allow-list (§A4, INT-10) **server-side** → `422` on mismatch.
- **Removing the last Admin or Manager with access is refused** `409`. A bench nobody can act on is a dead end.
- Granting access makes the bench visible on that member's list immediately, and **subscribes them to its notifications**. Removing it does both in reverse — one membership, read by both.
- Removing bench access never removes the member from the organisation.
- **Bench membership is what makes a Manager eligible to review that bench's engineers** on the Performance Log. One membership record, read for both purposes — there is no separate reviewer assignment to keep in step.
- A **Viewer** is never an eligible reviewer, on any bench.
- Because at least one Admin or Manager must retain access, **a bench always retains at least one eligible reviewer**. Assert both readings of that rule in one test.

---

## Integration & sync

| Value | Source | Also appears on |
|---|---|---|
| Bench name, type | The bench record — **renamed here** | List page, Internal Engagements, Channel page, every report |
| Capacity plan | `Included Hours` (SD-3459) — **set by the Castillians team** | Hours card, list page, Internal, Channel page |
| Period | Per-bench Start + Auto-Renew (SD-3459) | Every period label |
| Client-facing rate | Configured blended rate (BE-08) | Cost implication here, Channel page billing |
| Membership | Organisation + bench membership (§A4) | List visibility, notification recipients |
| Domain allow-list | Manage Client modal (§A4, INT-10) | Every invitation on the platform |
| Capacity requests | Written here, actioned on Internal (SD-3465) | Internal bench entry's order form history |

**Acceptance criteria**
- **Nothing here writes to a subscription.** A capacity request changes the plan only when the Castillians team actions it — at which point the new figure appears on this page, the list page, Internal and the Channel page **together**.
- A rename propagates everywhere on the next read and touches nothing else.
- **`timeZone` is gone from the record**; no response carries it after this ships.
