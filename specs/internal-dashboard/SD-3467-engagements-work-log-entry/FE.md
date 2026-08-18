# FE — Engagements: work log entry, edit history & the per-engineer Work Log page

Angular handoff for **SD-3467**. The entry row, its history accordion, **View Work Log**, and the per-engineer page.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — `Card`, `Avatar`, `StatusBadge`, `Button`, `ScoreRing`, `Toast`.
The list this entry sits in — search, filters, batching — is **SD-3466**.

---

## Components

| Component | Responsibility |
|---|---|
| `WorkLogEntryComponent` | The row: rails, identity, bench line, capacity bar, status, actions |
| `WorkLogHistoryComponent` | The accordion and its record cards |
| `DeclineEntryModalComponent` | Mandatory message, names what is being declined |
| `EngineerWorkLogPageComponent` | The per-engineer page: header, bench tabs, entries |

---

## Entry row

Rows sit flush in a **zero-padding Card**, separated by `1px solid var(--gray-75)`, no rule after the last; first and last carry the card's 8px radius. Each row `41px 30px`.

**Left rail — 60px fixed**
- Hours: Bricolage **22px/700**, as `7h`.
- Date beneath: body 13px `var(--gray-700)`, as `7 Aug` — no year.

**Centre**

| Element | Detail |
|---|---|
| Avatar | 40px, verified badge pinned bottom-right at 17px |
| Engineer name | Bricolage **22px/700** |
| Vetting score | Five bars 18×11px, 2px radius, `var(--green-500)` on `var(--gray-100)`, then the score in Bricolage **18px/700** |
| Engagement ended tag | Only when ended. White fill, `1px solid var(--gray-150)`, `var(--radius-md)`, `6px 10px`, body **10px/500**, `var(--gray-700)`. Reads **"Engagement ended 15 Jul"** |
| Description | 14px, 140% line-height, `#787878`, 12px below the identity row |
| Decline message | Quoted, `var(--danger-500)` on `var(--danger-50)`, 6px radius, `8px 12px`, 12px/500 |
| Bench line | `Client · Bench Name`, body **14px/600** `var(--ink-900)` |
| Capacity bar | 8px track, 4px radius, percentage and `used h / capacity h` beside it. ≤59% green `#10b77f`, 60–89% amber `#f59f0a`, ≥90% red `#ef4343`, each on its tinted track |

- **The capacity bar is hidden on an ended engagement** — a live reading against a bench that no longer runs is misinformation. The ended tag replaces it.
- The bar reads the **bench's** position, not the engineer's: an entry is a claim against a shared pool, and that is what decides whether approval is needed.

**Right rail** — a status tag **or** the review group. Never both.

### Status tags

| Status | Label | Variant |
|---|---|---|
| Auto-approved | Auto-approved | neutral |
| Awaiting the engineer's submission | Requires approval | warning |
| Awaiting a decision | Approval required | warning |
| Signed off by a person | Manually approved | success |
| Declined | Declined | danger |

**Auto-approved and Manually approved stay distinct** — collapsing them loses the difference between a machine pass and a human signature, which is what an auditor asks about. Tag padding even on all sides.

### Approve / Decline

- Shown **only** when awaiting a decision **and the engagement is ongoing**.
- Grouped in a bordered review group led by an **"Approval required"** label, so the two read as one decision.
- **Approve** — Medium Primary 45px. **Decline** — Medium Secondary 45px.
- Approving appends an `Approved` history row authored by **Castillians Team** — never the individual reviewer — and fires a success toast: _"Approval email sent to the engineer. The entry now appears across the Engineer, Internal and Manager dashboards."_
- **Declining opens a modal and requires a message**; the modal names engineer · bench · date · hours. Empty or whitespace-only is refused server-side, not only in the UI.
- Buttons disable on submit — double-submit is also guarded server-side.

---

## Edit-history accordion

- Bottom-left of the entry beside **View Work Log**, body 13px/500 `var(--gray-700)`, hover `var(--ink-900)`, 13×10px chevron.
- **Hidden entirely** when there is nothing beyond creation — the affordance must not lie (`hasHistory`).
- Chevron rotates 180° over 300ms; reveal uses a `grid-template-rows` transition (`0fr` → `1fr`) so the row grows rather than clipping. **Not a modal.**
- Indented to clear the left rail, stopping short of the action column.
- One card per record — `var(--gray-50)`, `2px solid var(--border)`, 8px radius, `12px 14px`:
  - Actor Bricolage **18px/700**; timestamp opposite, 14px `var(--gray-700)`.
  - Action as a white tag with a `var(--gray-150)` border, body 11px/500; detail beside it at 14px.
  - `Created` shows the original description on its own line.
  - Field changes as `FIELD  old → new` — label Bricolage 11px uppercase 0.5px tracking, old value `#9a9a9a` struck through, new value `var(--ink-900)` weight 500.
- **Newest first.** Actor names come from the API (`Castillians Team`, `Castillians System`) — never constructed client-side.
- Shown on Internal and Engineer; **withheld from Manager**.

---

## View Work Log

**Small Tertiary Button**, 140×35px, on every entry. Navigation only — changes nothing, sends nothing.

---

## Per-engineer Work Log page

**Header**
- Back link with left arrow, body 16px/500: **"Back to Engagements"**, or **"Back to Channel"** when reached from Channel & Billing — the label names where the operator came from.
- Avatar 56px, verified badge 24px. Name as `h1`, Bricolage **32px/700**. Beneath: five vetting bars, score Bricolage 14px/700, email body 13px `var(--gray-700)`.

**Bench tabs**
- One tab per Virtual Bench this engineer is on — `Core Platform (14)`, the count being their **grand total** on that bench across every period.
- Selected: Bricolage 700, ink fill, white label. Unselected: 400, `#828282`, hover `#141313` **without** a weight change — the row must not reflow on hover.
- A tab for an ended engagement is labelled as ended, same _"Ended 15 Jul"_ wording as the entry tag.
- Default is the engineer's **first** bench; arriving from a bench context opens **that** bench's tab.
- Tabs are the only filter here — no search, no period, no scope.

**Entries**
- One card, zero padding, one row per entry, `20px 30px`, `1px solid var(--gray-75)` between.
- Same 60px hours/date rail; **bench name** body 14px/600 with the **client** as a white chip beside it; description; decline message; history accordion, un-indented here.
- Right rail: same status tag, same review group under the same conditions — a decision can be taken here.
- **No avatar, no vetting bars, no capacity bar, no View Work Log.**
- Newest first, across all periods, **not batched** — the tab already narrows to one engineer on one bench.
- Empty: **"No work logs recorded for this engineer yet."** Loading and error per §G2.

---

## General

- Responsive — the entry's three columns stack and the review group stays intact rather than splitting across lines.
- Chrome, Firefox, Safari. Existing design-system components; no new one-offs.
- Every transition `300ms cubic-bezier(0.35,0,0.25,1)`.

---

## Reference

```
BE.md                          approval, history, carry-over columns
../SD-3466-engagements-work-logs-filters/   the list this entry sits in
../../ENGINEERING-BRIEF.md     BE-11 to BE-14 approval and history, BE-22 month-end exports, §G states
../../../prototype/index.html  → Internal → Engagements → Work Logs → an entry, then View Work Log
```
