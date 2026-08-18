# FE — Engagements: Virtual Benches tab

Angular handoff for **SD-3464**. The Engagements page frame and its Virtual Benches tab.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — `Card`, `Input`, `Select`, `Button`.
The expanded bench entry is **SD-3465**; the Work Logs tab is **SD-3466** / **SD-3467**.

---

## Page frame

- Two tabs: **Virtual Benches (n)** and **Work Logs (n)**.
- Counts are **grand totals** — they do not react to search or filters. A filtered view showing 2 benches still reads "Virtual Benches (6)".
- Selected tab: Bricolage 24px/700, ink fill, white label. Unselected: 400, `#828282`, hover `#141313` **without** a weight change.

## Download Engineer Invoicing

- **Medium Tertiary Button** top-right, aligned with the page title.
- Modal with a **billing period** selector, defaulting to the most recently closed period; any prior period selectable.
- Downloads **.xlsx** — one row per engineer per bench: period, engineer, email, client, bench, manager(s), currency, hours, rate, earnings, totals **per currency**.
- The rate is **engineer-facing**; the configured blended rate must never appear.
- **No email is sent** — the file arrives in the browser.

## Search and filters

One row, all controls **50px** fixed height: **Search** (client and bench names, live), **All channels**, **All clients**. They compose with AND; changing any resets the list to the top. An empty result shows a plain sentence, not a blank card.

## Results table

- Headers: **VIRTUAL BENCH**, **CAPACITY**, **REMAINING**.
- One row per active bench, ordered by capacity used **descending**.
- Row: chevron · bench name (Bricolage 22px/700) · client beneath · capacity bar with percentage · `used / capacity` · remaining hours.
- **Remaining shows 0h** when over plan — never a negative figure.
- Bands ≤59% green `#10b77f`, 60–89% amber `#f59f0a`, ≥90% red `#ef4343`, each on its tinted track.
- Bar geometry identical across rows, so the 100% mark sits at the same width whatever the capacity.
- The whole row header is the accordion toggle; the chevron rotates 180° over 300ms.

---

## General

- Responsive; Chrome, Firefox, Safari; existing design-system components only.
- Every transition `300ms cubic-bezier(0.35,0,0.25,1)`.

---

## Reference

```
BE.md                          endpoints, totals, invoicing export
../SD-3465-bench-entry/        the expanded bench
../SD-3466-engagements-work-logs-filters/   the other tab
../../ENGINEERING-BRIEF.md     §G interaction patterns
../../../prototype/index.html  → Internal → Engagements → Virtual Benches
```
