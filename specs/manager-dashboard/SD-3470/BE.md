# BE — Virtual Benches page & Create a new bench

Backend specification for **SD-3470**.

---

## Endpoints

### `GET /api/manager/benches`

```json
{
  "clientEntities": [ { "id": "c-northmill", "name": "Northmill Bank" } ],
  "benches": [ {
    "benchId": "vb-core", "name": "Core Platform", "type": "permanent",
    "clientId": "c-northmill", "clientName": "Northmill Bank",
    "engineerCount": 4,
    "capacityPlanHours": 160, "hoursUsed": 96, "percentUsed": 60, "hoursRemaining": 64,
    "periodStart": "2026-08-01", "periodEnd": "2026-08-31", "daysRemaining": 6,
    "overCapacity": false,
    "canRequestBench": true
  } ],
  "pendingRequests": [ { "requestId": "r-12", "name": "Data Science Pod", "requestedAt": "2026-08-20" } ]
}
```

**Acceptance criteria**
- **Visibility is enforced server-side.** An Admin receives every bench in the organisation; a Manager or Viewer receives **only benches they are a member of**. The response never carries a bench the caller cannot open.
- `hoursUsed` counts **approved** work logs only, against **that bench's own** period (BE-02, BE-03) — pro-rated first period included.
- `hoursRemaining` is floored at **0**. A negative remainder is never returned.
- `percentUsed` may exceed 100; the client renders the bar capped while the figure reads true.
- `clientEntities` lists only entities the caller can see; a single-entity organisation returns one, and the client renders no filter.
- Benches on an **expired** subscription are excluded from the working list.
- Sort is `percentUsed DESC, name ASC` — **stable**, so the grid does not reshuffle between reads.
- No role flags beyond `canRequestBench`: the server decides, the client renders.

### `POST /api/manager/bench-requests`

**Acceptance criteria**
- **Refused for a Viewer** — `403`. The absence of the button is not the control.
- Creates a request record only. **No subscription row, no capacity, no rate, no billing.** Assert in a test that no subscription is written.
- The bench **name is stored verbatim** — never re-cased, trimmed of internal spacing, or suffixed with the client name. Leading and trailing whitespace is trimmed; an all-whitespace name is empty and refused `422`.
- Name must be **unique within the organisation**; a duplicate is refused with the existing bench named in the message.
- The request collects a **billing currency** alongside the brand and the monthly engineering hours, and carries it to the Internal queue. **Currency is a property of the subscription, not of the brand or the channel** (BE-27) — a client can hold benches billed in different currencies, so it is asked for per request rather than inherited. Mandatory; a request without one is refused `422`.
- Every field the manager entered is carried to the Internal queue **verbatim**, including free-text answers like "not sure yet".
- **Double-submit produces one request** — idempotency on caller + name + open state.
- The request appears in `pendingRequests` on the next read until actioned.

---

## Integration & sync

This page aggregates; it owns nothing but the request.

| Value | Source | Also appears on |
|---|---|---|
| Bench visibility | Organisation + bench membership (§A4) | Bench page members list (SD-3471) |
| Capacity plan | `Included Hours` (SD-3459) | Bench page, Internal Engagements, Channel page |
| Period, days remaining | Per-bench Start + Auto-Renew (SD-3459) | Every period label, all three dashboards |
| Hours used | Approved work logs | Bench page hours card, Internal Work Logs, Engineer overview |
| Over-capacity state | Bench overage state (SD-3465, BE-05) | Bench page, Internal bench entry |
| Engineer count | Activated roster | Bench page Engineers section (SD-3474) |

**Acceptance criteria**
- A card's figures **reconcile to the hour** with the bench page, the Internal Engagements row and the Channel page for the same bench and period — one source, four surfaces.
- **Pending-approval hours are excluded** from every figure; an approval on Internal moves them on the next read, **recomputed, never patched**.
- A capacity plan change by the Castillians team changes the denominator here immediately — no client-side cache of the plan.
- Two benches in one organisation may return **different periods** and different `daysRemaining` in the same response.
- Granting a member bench access makes that bench appear here on their next read; removing it removes the bench **and** stops its notifications reaching them.
- A member who arrives via an **invitation** sees this page for the first time at the end of Manager onboarding, and it shows **exactly the benches they were granted** — by an Admin or by a Manager, both of whom may invite — the invitation's own grant, applied on completion (SD-3471). Nothing else is visible, and they were shown nothing at all while pending.
- **A bench request provisions nothing.** It becomes a bench only when the Castillians team actions it.
