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

- Headers: **VIRTUAL BENCH**, **CAPACITY PLAN**, **OVERAGE**, **REMAINING**.
- One row per active bench, ordered by capacity used **descending**.
- Row: chevron · bench name (Bricolage 22px/700) · client beneath · capacity bar with percentage · `Xh of Yh used` · **overage status** · remaining hours.
- **Column order is CAPACITY PLAN → OVERAGE → REMAINING**, left to right, and it is the reading order of the row: what was subscribed, what is allowed on top of it, what is left. OVERAGE sits between the two figures it reconciles.
- **The header and the row are ONE CSS grid**, not two lists of fixed widths kept in sync:

  ```css
  grid-template-columns: minmax(220px, 1fr) minmax(0, 260px) minmax(0, 190px) 140px;
  gap: 40px;
  ```

  Alignment is then structural, and the columns narrow **together** as the row does. Applied identically to the header `div` and the row `button`.
- **REMAINING is 140px, sized by its longest string** — `of 192h auto-approved` is ~137px, so a narrower track leaves the qualifier wider than the figure above it and spilling into the OVERAGE gap.

_Outdated on 27 Aug. Previously: "The header cells and the row cells share one set of fixed widths — 260px · 190px · 110px, 40px gaps, in a 640px group", and the row was laid out with `justify-content: space-between` and a `flex: 0 1 auto` figure group._

Both earlier attempts failed the same way, and the grid exists to stop a third: **fixed widths inside a flex row relocate an overflow rather than resolving it.** With `flex: none` children totalling 640px, shrinking the group's box to 544px left its contents at 664px — they spilled, pushing REMAINING 66px outside the card and making the page scroll horizontally. A shrinkable **box** is not a shrinkable **track list**.
- **VIRTUAL BENCH holds the `minmax(220px, 1fr)` track** — it takes the slack when the row is wide and stops at 220px when it is narrow. The three figure tracks carry `minmax(0, …)` so they shrink rather than overflow.
- **A too-long bench name truncates with an ellipsis** (`white-space: nowrap; overflow: hidden; text-overflow: ellipsis`) on both the name and the client line. Without it, `min-width: 0` plus visible overflow lets the text escape its box and run into CAPACITY PLAN rather than being clipped — silently, and only on the longest names.
- Do not lay the row out with `justify-content: space-between` against differently-sized headers: the cells drift out of line as content changes width, and the OVERAGE label is the first to show it.

_Outdated on 27 Aug. Previously: "Headers: VIRTUAL BENCH, CAPACITY, REMAINING", "Row: … capacity bar with percentage · `used / capacity` · remaining hours", and "Remaining shows 0h when over plan — never a negative figure."_

### OVERAGE — new column

States the bench's overage setting in words. A bench is always in **exactly one** of four states, and they are the states the Internal controls can produce (SD-3465):

| State | Label |
|---|---|
| Allow Overages off | **Off** |
| On, no authorised block | **On (up to 120%)** |
| On, block authorised | **On (X hours authorised)** |
| Unlimited Overage on | **On (Unlimited)** |

- The percentage in the second label is the platform tolerance, read from the same constant the logging validation uses — never a literal in the cell.
- **Two weights in one label:** the state — **On** / **Off** — at **weight 600**, and the qualifier in brackets beside it at **weight 500**. The state is what an operator scans for; the bracket is the detail behind it.
- Bricolage **14px**, ink, left-aligned in a 190px column so the labels form a readable column rather than ragged text. The two runs sit on a baseline-aligned flex row with a 4px gap and wrap rather than truncating.
- Read-only here. The controls that set it live in the expanded bench entry (SD-3465).

### CAPACITY PLAN — renamed and reformatted

_Outdated on 27 Aug. Previously headed **CAPACITY**._ Renamed to **CAPACITY PLAN**, which is what the denominator actually is — hours before overage. With an OVERAGE column beside it, a bare "CAPACITY" invited the reading that the figure already included the allowance.

- Reads **`Xh of Yh used`** — X is hours logged, Y is the **capacity plan**, i.e. hours **before** any overage.
- The plan, not the overage allowance, stays the denominator: it is what the client subscribed to, and it keeps the bar and the percentage meaningful once a bench goes past it. A bench over its plan therefore still reads above 100%.

### REMAINING — follows the overage allowance

- Shows the hours **actually still loggable**, which is the overage allowance minus hours logged — **not** the plan minus hours logged.
- Resolves per state: **Off** → plan − used · **On (up to 120%)** → 120% of plan − used · **On (X hours authorised)** → plan + X − used · **On (Unlimited)** → **No cap**.
- **Never a negative figure** — floored at `0h`.
- Why: a bench past its plan with authorised overage still has hours left. Showing `0h` there sends a manager to raise a request they do not need, and tells an operator the bench is blocked when it is not.
- **The figure carries a qualifier line beneath it** naming what it is measured against — `of 192h auto-approved`. Without it the number cannot be derived from the two figures in CAPACITY PLAN beside it: 92h remaining on a 160h plan with 100h used reads as an arithmetic error, when it is 192h − 100h used. Bricolage 13px `#787878`, right-aligned, no wrap.
- **The word is "auto-approved", not "allowed".** The ceiling is the point past which hours need a person's signature (BE-13) — **not** a hard cap. Hours beyond it are still accepted, they simply queue for approval, so a bench genuinely can sit above the figure. "Allowed" claimed a limit the data can exceed.
- **Past the ceiling the column reports the excess, not `0h`** — `5h over`, in `var(--danger-500)`. Flooring at zero hid the overshoot and read as "blocked", which is what **Off** means, not this. An engineer can still log on a bench reading `5h over`; the hours will wait for approval.

### Worked example — Insurance Web, and why the row is coherent

The case most likely to be read as a bug, so it is worth stating in full:

| Cell | Value | Where it comes from |
|---|---|---|
| CAPACITY PLAN | `101h of 80h used` — **126%** | 80h plan; 101h approved; 101 ÷ 80 |
| OVERAGE | **On (up to 120%)** | Overages on, no authorised block → tolerance applies |
| REMAINING | **5h over** · _of 96h auto-approved_ | Ceiling 80 × 1.2 = 96h; 96 − 101 = −5 |

**101h logged against a 96h ceiling is not a defect.** The tolerance is the auto-accept threshold: the first 96h clear automatically, and the 5h beyond it were accepted and manually approved (BE-13, SD-3467). The row therefore says three true things at once — the client is 21h past what they subscribed to, 5h past what clears without review, and every one of those hours has been approved and is billable.
- **The line appears only where the allowance and the plan genuinely differ** — that is, on **On (up to 120%)** and **On (X hours authorised)**. It is **omitted on Off**, where the allowance *is* the plan, so the line would restate the denominator immediately to its left and "allowed" would imply headroom that does not exist. It is **omitted on On (Unlimited)**, where "No cap" already says it.
- **CAPACITY PLAN and REMAINING answer different questions and will disagree** — 127% used with 20h remaining is correct and expected on a bench with an authorised block. That is the point of having both.
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

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.
