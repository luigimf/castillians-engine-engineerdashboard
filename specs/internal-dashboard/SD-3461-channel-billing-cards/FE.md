# FE — Channel & Billing: root client cards & Download a Report

Angular handoff for **SD-3461**. The landing view of Internal → Channel & Billing.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — `Card`, `Button`, `Select`, `Toast`.
The channel drilldown this page opens is **SD-3462**.

---

## Root client cards

One card per **root client** — the top-most parent in its channel, resolved from Zoho's `Parent Brand`. Child clients get no card of their own.

- White, `1px var(--border)`, 8px radius, 30px padding. **No hover lift** — informational, not the click target.
- Eyebrow `ROOT CLIENT`; title the client name; subtitle **"X Clients"**, counting the root itself.
- Stats in order: Virtual Benches · Engineers · Total Skills · Monthly Capacity (**hrs**, not h) · Monthly Billing (CUR).
- Labels Bricolage 12px/500 uppercase, 0.7px tracking, `var(--gray-700)`. Values Bricolage 22px/700.
- **View Channel** — Medium Secondary, 45px.
- Responsive grid, wrapping rather than scrolling.

**Counts are distinct across the channel** — an engineer activated on two benches in one channel counts once, and their skills once.

## Currency

- From the **billing client's Zoho record**. No default, no hardcoding.
- Currency **codes** in the label — `MONTHLY BILLING (EUR)` — with a bare value, exact cents.
- Mixed-currency channel → one figure **per currency**. Never combined, never converted.

---

## Download a Report

- **Download a Report** button top-right, aligned with the page title. Opens a modal — never a direct download, since the user must choose what and for when.

| Report | Contents |
|---|---|
| Payroll Checklist | One row per engineer per bench — codes, rates, normal and overage hours, amounts, **Period Earned / Period Billed / Carried Over** (BE-22, SD-3467), plus the hand-filled tracking columns left blank |
| SFM Supplier Upload | SFM's fixed 22-column format, one row per supplier invoice |
| Client Billing | One row per Virtual Bench — subscription and overage separated |

- Period selector: the current period and every prior one with data, most recent first.
- Selecting a report updates a short description beneath it — the user should know what they are about to download.
- **.xlsx**, not CSV. Filenames `castillians-{report}-{YYYY-MM}.xlsx`.
- Current open period → a note that figures are to date.
- Success closes the modal and fires a toast naming the report and period. **No email** — on-demand downloads are silent.

Each report is **byte-identical in format** to the month-end attachment for the same period; one generator serves both.

---

## States

- Loading: skeleton cards at natural height, no collapse.
- Empty: a plain sentence, never a blank grid.
- Error: inline with a retry inside the card.
- A blocked report row (missing Zoho finance field) is **named in the response**; card totals still render.

---

## General

- Cards wrap and stats reflow rather than overflow.
- Chrome, Firefox, Safari. Existing design-system components; no new one-offs.
- Every transition `300ms cubic-bezier(0.35,0,0.25,1)`.

---

## Reference

```
BE.md                          aggregation rules, report endpoints
../SD-3462-channel-page/       the drilldown View Channel opens
../../ENGINEERING-BRIEF.md     BE-22, BE-23, §A6 (SFM), §G (interaction patterns)
../../../prototype/index.html  → Internal → Channel & Billing
```
