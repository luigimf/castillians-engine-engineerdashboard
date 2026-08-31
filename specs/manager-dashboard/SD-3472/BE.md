# BE — V Bench page: Skills Mix, Skills Matrix & Monthly Engineering Hours

Backend specification for **SD-3472**.

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** this file: `BE.md` is the backend spec for one story, `BE-nn` is a numbered rule in the brief.

---

## Endpoints

### `GET /api/manager/benches/{benchId}/capacity`

```json
{
  "periodStart": "2026-08-01", "periodEnd": "2026-08-31", "daysRemaining": 6,
  "capacityPlanHours": 160, "capacityUsedHours": 168, "percentUsed": 105,
  "overagesAgreedHours": 40, "overagesUsedHours": 8, "pendingHours": 8,
  "overageRate": { "amount": 118.75, "currency": "EUR" },
  "unlimitedOverage": false,
  "perEngineer": [ { "engineerId": "eng-7", "name": "Maria Alves", "hours": 62 } ],
  "openCapacityRequest": null,
  "canRequestCapacity": true
}
```

**Acceptance criteria**
- `capacityUsedHours` counts **approved** work logs only, against **this bench's own** period (BE-02, BE-03) — pro-rated first period included. Never the calendar month by default.
- **`perEngineer` hours sum exactly to `capacityUsedHours`.** A response where they differ is a defect; assert it in a test.
- Every **activated** engineer appears in `perEngineer`, including at `hours: 0`. Omitting them would read as having left the bench.
- `pendingHours` is reported so the client can name what is awaiting a decision, and is **excluded from `capacityUsedHours`** and from `perEngineer` hours. Excluded from the arithmetic, disclosed in words.
- `perEngineer` carries **no rate and no earnings** — assert that neither reaches this response (BE-04, BE-08).
- `overageRate` is the **client-facing blended rate**, at **1.25× for the first three months** of the subscription (BE-06). `configured ÷ (1 + mark-up)` never appears.
- `overagesAgreedHours` renders under the label **OVERAGES AGREED**; it is the **Authorised Total Overage** granted on Internal (SD-3465). It **replaces the 20% tolerance rather than stacking on it** — a 160h plan with a 40h block accepts **200h**, not 232h.
- The grant **holds for the whole final day** of the period and lapses after it, so end-of-period billing settles against the hours actually authorised.
- `unlimitedOverage: true` → the client renders the overage figures as uncapped; no ceiling is implied.
- `percentUsed` may exceed 100 and is reported true; the client caps the bar's fill, not the figure.

### `GET /api/manager/benches/{benchId}/skills`

Core skills, additional skills, and the matrix of activated engineers against them.

**Acceptance criteria**
- The skills **matrix** is the **aggregated union of the activated engineers' own skills**, each with a coverage band — Popular (3+ engineers), Common (2), Unique (1) — ordered by coverage descending. It is **not** the bench's agreed mix, and it legitimately contains skills outside it.
- **There is no write endpoint for the skills mix on this dashboard.** No client role can change it; it moves only when the Castillians team updates the subscription (INT-5), and then it changes here, on the Channel page and on Internal together.
- The matrix is derived from the engineers' **own vetted profiles**; it is not authored against the bench, and it carries no uncovered-skill state.

### `POST /api/manager/benches/{benchId}/capacity-requests/period`

**Acceptance criteria**
- **Refused for a Viewer** (`403`).
- Scoped to the **current period only**. A request never rolls forward into the next.
- Creates a request record and **nothing else** — `purchasedHours` is unchanged in the same read. Assert that no overage grant is written.
- `openCapacityRequest` is returned on the capacity read until actioned; a second identical request while one is open is refused `409`.
- Granting it on Internal moves `overagesAgreedHours` and the ceiling on the next read.

---

## Integration & sync

| Value | Source | Also appears on |
|---|---|---|
| Skills mix | The bench's subscription (INT-5) | Channel page bench detail, Internal bench entry |
| Skills matrix | Activated engineers' vetted profiles | Internal bench detail, Channel page |
| Capacity plan | `Included Hours` (SD-3459) | List page, Internal Engagements, Channel page |
| Period, days remaining | Per-bench Start + Auto-Renew (SD-3459) | Every period label |
| Capacity used, per-engineer | Approved work logs | Engineer overview, Internal Work Logs, Manager work logs (SD-3473) |
| Overages Agreed | Authorised Total Overage (SD-3465) | Internal bench entry, order form history |
| Overage rate | Configured blended rate (BE-06, BE-08) | Channel page billing, client billing report |

**Acceptance criteria**
- **Capacity Used equals the sum of the per-engineer rows, and equals** the same figure on the list page, the Internal Engagements row and the Channel page for the same bench and period.
- **Pending-approval hours are excluded everywhere.** An approval on Internal moves all of them on the next read — **recomputed, never patched client-side**.
- **Declined** and **auto-declined** entries count towards nothing here and are never listed.
- **Requesting capacity never changes a figure.** `overagesAgreedHours` moves only when the team actions the request.
- Each bench resolves its **own** period; no figure assumes a shared calendar month.
