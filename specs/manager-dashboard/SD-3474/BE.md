# BE — V Bench page: Engineers

Backend specification for **SD-3474**. The section reads the roster and writes only requests.

---

## Endpoints

### `GET /api/manager/benches/{benchId}/engineers`

```json
{
  "activated":   [ { "engineerId": "eng-7", "profile": { /* production card payload */ },
                     "openRequest": null, "canRequestRemoval": true } ],
  "available":   [ { "engineerId": "eng-9", "profile": { }, "openRequest": null, "canRequestAdd": true } ],
  "recommended": [ { "engineerId": "eng-3", "profile": { }, "openRequest": null, "canRequestAdd": true } ]
}
```

**Acceptance criteria**
- The `profile` payload is whatever the **existing production engineer card** consumes. Do not invent a new shape — reuse the one already serving that component.
- **No rate, no earnings, no personal contact details** in any profile here — assert it in a test (BE-04, BE-08).
- **No allocation percentage.** The manager never sets one and never sees one; allocations are maintained on Internal (SD-3465).
- `available` is scoped **server-side** to engineers engaged with **this client** and not on this bench. `recommended` is matched against the bench's skills mix. The response never carries an engineer the caller may not see.
- An engineer's engagements with **other clients** are never returned.
- `activated` is sorted by name — stable between reads.
- `canRequestRemoval` / `canRequestAdd` are `false` for a Viewer; the server refuses regardless.

### `POST /api/manager/benches/{benchId}/engineer-requests`

`{ "engineerId": "eng-7", "kind": "add" | "remove", "note": "…" }`

**Acceptance criteria**
- **Refused for a Viewer** (`403`). The absent button is not the control.
- Creates a request record and **nothing else**. Assert that **no roster row, no allocation, no capacity and no billing** changes — the engineer's membership is byte-identical before and after.
- `note` is optional; `kind` and `engineerId` are mandatory.
- A `remove` is valid only for an activated engineer; an `add` only for one in `available` or `recommended` — `409` otherwise.
- A second identical open request on the same engineer is refused `409`; `openRequest` is returned on the read so the client renders it as pending.
- **Double-submit produces one request** — idempotency on bench + engineer + kind + open state.
- Requesting removal of the **last activated engineer** is **allowed** — the client warns, the server does not block. A client may legitimately be winding a bench down.

---

## When a request is actioned on Internal

**Acceptance criteria**
- **Removal:** the engineer leaves the roster, and the activated count drops **everywhere on the next read** — this section, the Virtual Benches list, the hours card's usage list, Internal and the Channel page.
- **Their logged hours stay.** Removing an engineer never deletes or reassigns work already logged; the hours remain in every figure and every report for the periods they worked. Assert this — it is the rule most likely to be got wrong.
- **Addition:** the engineer appears in `activated`, in the skills matrix, and in the hours card's usage list **at zero hours** — one activation, every surface.
- Either action changes the **allocation shares** the Castillians team maintains (SD-3465). The manager never sets a percentage.
- A removed engineer moves back into `available` if they remain engaged with the client, so they can be requested again.

---

## Integration & sync

| Value | Source | Also appears on |
|---|---|---|
| Activated roster | The bench record | List page count, hours card usage list, Internal bench entry, Channel page |
| Available | The client's engaged engineers not on this bench | Internal bench entry |
| Recommended | Matched against the bench's skills mix (SD-3472) | Internal bench entry |
| Engineer profile | The engineer's own vetted profile | Every dashboard, the skills matrix |
| Allocation share | Internal → Engagements (SD-3465) | The engineer's own ceiling — never shown here |
| Add / remove requests | Written here, actioned on Internal (SD-3465) | Internal team's queue |

**Acceptance criteria**
- **A request changes nothing.** Every figure on every surface reads identically before and after it is raised.
- The activated count here equals the count on the list page card and the row count in the hours card's usage list, always.
- Scoping is **server-side** — never a client-side filter over a wider set.
