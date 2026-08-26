# BE — Organisation Page: Access

Backend specification for **SD-3477**. The route guard, the withdrawn `/brand-profile` route, and the role resolution the menubar reads.

---

## Endpoints

### `GET /api/manager/organisation`

The page's own read. Returns the account, its Admin, and the capability flags the client renders from. The cards' payloads belong to SD-3478 and SD-3479.

```json
{
  "accountId": "acct-1",
  "admin": { "memberId": "mem-1", "name": "James Whitmore", "email": "james@northmill.com" },
  "canViewOrganisation": true,
  "canChangeAccountAdmin": true
}
```

**Acceptance criteria**
- **Refused `403` unless the caller is the account Admin.** Manager and Viewer are refused whatever bench roles they hold — Manager and Viewer are per-bench roles, and no combination of them grants this page.
- `canViewOrganisation` is resolved **server-side** from the account record. The client never derives it.
- **Exactly one `admin` is returned, always.** There is one Admin per account; a response with none or two is a bug worth asserting against.
- The response carries **no data a Manager or Viewer could otherwise not see** — it is refused outright rather than trimmed, so there is no partial payload to leak.

### Menubar capabilities

**Acceptance criteria**
- The signed-in member's menubar entries are returned by the **existing** session/capability response, with `organisation` present only for the Admin.
- The entry is **omitted, not returned disabled.** There is no `enabled: false` shape for it — a flag the client could render greyed is how a greyed tab gets built.
- Adding the entry changes no other entry's presence or order.

### `GET /brand-profile` — withdrawn

**Acceptance criteria**
- The route is **removed**, not aliased indefinitely. A request from an Admin resolves to `/organisation`; from a Manager or Viewer, to their Virtual Benches — a redirect, not an error page.
- Unpublishing it **deletes no data**: brands, members, domains and bench membership are untouched. Assert that the only change is route availability.

---

## Integration & sync

| Value | Source | Also appears on |
|---|---|---|
| Who the Admin is | The account record (§A4) | This page's visibility, Change Account Admin (SD-3478), subscription ownership |
| `canViewOrganisation` | Derived from the account record on every read | The menubar entry and the route guard — **one derivation, two consumers** |
| Brands in the channel | Zoho `Parent Brand` lineage (SD-3416, SD-3417) | Internal Channel page (SD-3462), SD-3478's brands list |

**Acceptance criteria**
- **The menubar entry and the route guard read the same value.** They can never disagree — a visible entry that 403s, or a reachable route with no entry, are both failures of this story.
- **An Admin transfer moves the page atomically** (SD-3478): on the next read the outgoing Admin has neither entry nor route, and the incoming one has both. Never both people, never neither.
- Role is recomputed on every response, **never cached client-side** across a role change.

---

## Test cases

| # | Case | Expected |
|---|---|---|
| 1 | Admin `GET /api/manager/organisation` | `200`, one `admin`, `canViewOrganisation: true` |
| 2 | Manager (Manager on two benches) same call | `403` |
| 3 | Viewer same call | `403` |
| 4 | Manager on bench A, Viewer on bench B | `403` — no combination of per-bench roles grants the page |
| 5 | Session response for a Manager | No `organisation` entry in the menubar payload, in any form |
| 6 | `/brand-profile` as Admin | Redirect to `/organisation` |
| 7 | `/brand-profile` as Viewer | Redirect to Virtual Benches |
| 8 | Admin transferred, then both parties re-read | Outgoing: `403` + no entry. Incoming: `200` + entry |
| 9 | Two transfers submitted at once | One succeeds; the account still has exactly one Admin |
