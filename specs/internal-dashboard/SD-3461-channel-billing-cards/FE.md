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
- Stats in order: Virtual Benches · Engineers · Total Skills · Monthly Capacity (**hrs**, not h) · Monthly Billing.
- Labels Bricolage 12px/500 uppercase, 0.7px tracking, `var(--gray-700)`. Values Bricolage 22px/700.
- **View Channel** — Medium Secondary, 45px, pinned to the **bottom** of the card (`margin-top: auto`) so the CTAs line up across a row.
- Responsive grid, wrapping rather than scrolling, built with `repeat(auto-fill, minmax(330px, 1fr))`.

_Outdated on 16 Sep. Previously: "Responsive grid, wrapping rather than scrolling." with no track rule, and the CTA sat directly under the last stat._

**`auto-fill`, not `auto-fit`.** With `auto-fit` a filtered result collapsed the empty tracks and stretched the surviving card across the full row — searching one parent client produced a single full-width card whose stats sat metres apart. `auto-fill` keeps the empty tracks, so a filtered card holds the same width it had in the unfiltered grid.

### Cards in a row share the tallest card's height

- Every card in a row is the height of the **tallest card in that row** — a channel with two billing currencies makes its row taller and its neighbours match it, rather than each card ending under its own last line.
- Implementation: the grid row stretches (`align-items: stretch`, the default) **and `height: 100%` is set on the `Card` element itself**, passed as the component's own `style` prop.
- **Setting the height on a wrapper around the card does not work.** A wrapper stretches to the row height while the card inside it keeps its content height, so the border still ends early and the fix looks applied but is not. The height has to reach the element that draws the border.
- The card is a **flex column** so `margin-top: auto` on **View Channel** can hold the CTA at the bottom of the taller box.

**Counts are distinct across the channel** — an engineer activated on two benches in one channel counts once, and their skills once.

## Currency

- From the **billing client's Zoho record**. No default, no hardcoding.
- The label reads **MONTHLY BILLING**, with **no currency in it**, and **each figure carries its own code** — `EUR 74,250.00` — exact cents.
- Mixed-currency channel → one figure **per currency**, stacked, one line each. Never combined, never converted.

_Outdated on 26 Aug. Previously: "Currency codes in the label — `MONTHLY BILLING (EUR)` — with a bare value, exact cents."_

A code in the label cannot hold two currencies, so a mixed channel rendered one label and either a wrong combined figure or an ambiguous one. The code moves onto the value, and the cell renders one line per currency. **Currency is a property of the subscription, not of the channel or the client** (BE-27) — a single brand can hold benches billed in different currencies, so this is the normal case, not an edge case.

---

## Search — added 16 Sep

A single full-width search field sits between the page intro and the card grid.

- Placeholder **"Search by parent client"**. It matches the **root client name only** — the string on the card title — case-insensitive, matching on substring, live as the operator types.
- Child client names are **not** searched here: the cards are one per root client, so a child match would have no card to return. The channel page (SD-3462) is where child clients are searchable.
- Searching **resets the grid to page 1**.
- No results → a plain sentence, *"No channels match that parent client."*, never a blank grid. The pager stays visible and reads `Showing 0–0 of 0 channels`.

## Pagination — added 16 Sep

The card grid pages at **9 cards per page** — three full rows at desktop width.

- The pager sits **below the grid**: the range label `Showing 1–9 of 24 channels` on the left, and `Page 1 of 3` between prev/next buttons on the right.
- **The pager is always rendered**, including when everything fits on one page (`Page 1 of 1`). An operator should be able to see that the list is paged without having to have enough data to trigger it.
- Prev/next are 35px square, 4px radius, `#F8F8F8` fill on a 2px `#E5E5E5` border, hover `#F2F2F2` — the same pager control as Engagements (SD-3464) and the bench entry lists (SD-3465). One pager control, every list.
- Page is clamped to the available range: prev on page 1 and next on the last page are no-ops, never a page 0 or an empty page.
- The count in the label is the **filtered** count, not the grand total — it describes what the pager is paging.

---

## Download a Report

- **Download a Report** button top-right, aligned with the page title. Opens a modal — never a direct download, since the user must choose what and for when.

| Report | Contents |
|---|---|
| Payroll Checklist | One row per engineer per bench — codes, rates, normal and overage hours, amounts, plus the hand-filled tracking columns left blank |
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

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.
