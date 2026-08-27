# FE — Channel Page: Breakdown & Billing

Angular handoff for **SD-3462**. The channel drilldown on Internal → Channel & Billing.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — `Card`, `Button`, `Avatar`, `StatusBadge`, `SkillTag`.

---

## Sections

1. **Page header** — back link, root client name, summary tags.
2. **Channel Breakdown** — the recursive client tree.
3. **Billing** — per-client rows with period navigation and Excel export.

---

## Generation tag palette

Colour by depth, Bricolage, uppercase, 11px weight 800, 0.6px tracking:

| Depth | Label | Fill | Text |
|---|---|---|---|
| 0 | `ROOT` | `#EDE9FE` | `#6D28D9` |
| 1 | `PARENT` | `#DBEAFE` | `#1D4ED8` |
| 2 | `CHILD GENERATION 1` | `#DCFCE7` | `#15803D` |
| 3 | `CHILD GENERATION 2` | `#FEF3C7` | `#B45309` |
| 4+ | `CHILD GENERATION n-1` | cycle the palette | — |

Beyond depth 4 the palette repeats rather than running out — indentation already conveys depth.

---

## Tree row

- One row per client, indented by depth, each an accordion.
- Row carries: chevron · client name · **Client Type** tag (from Zoho) · **Generation** tag · then Virtual Benches, Capacity (hrs) and Billing right-aligned.
- The **root row alone** adds **Channel Billing** — the sum across every descendant.
- Chevron rotates 180° over 300ms; reveal uses a `grid-template-rows` transition (`0fr` → `1fr`).
- A client with **no benches** renders its chevron in `#E5E5E5` and does not expand.
- Vertical connectors beside nested rows are **rounded**, identical stroke weight at every depth.

---

## Billing section

- Grouped by client, one row per bench: name, type, engineers activated, Capacity Plan, Hours, Overage Rate, Billing.
- **Total** row per client group; **grand total** closes the section.
- Period navigation bounded at the earliest period with data and the current one.
- Differing currencies → a total **per currency**, stacked, one line each. Never combined, never converted.
- The labels read **BILLING** and **CHANNEL BILLING**, with **no currency in them**, and **each figure carries its own code** — `EUR 24,660.00` — exact cents.

_Outdated on 26 Aug. Previously: "Currency codes in labels — `BILLING (EUR)` — bare values, exact cents."_

**The page header's summary tags are per currency too, not only the tree rows.** A channel spanning currencies renders **one MONTHLY BILLING tag per currency**, each carrying its own code — `MONTHLY BILLING (EUR)`, `MONTHLY BILLING (USD)` — and the currency appears in the label only when there is more than one to tell apart. **Never one tag summing across currencies**: adding USD to EUR and labelling the result EUR is a wrong number wearing a right label, and it is the defect this rule exists to prevent. The header must agree with the root row's CHANNEL BILLING cell beneath it, figure for figure.

**Every figure cell on a tree row aligns to the TOP of the row, not its centre.** A billing cell can hold one line per currency, so a centred row drops VIRTUAL BENCHES and CAPACITY out of line with BILLING and CHANNEL BILLING beside them. Set the row's figure group to `align-items: flex-start` so all four labels sit on one line whatever the cells beneath them do.

**A client with no benches renders an em dash**, matching the dash the same row already shows under Capacity — never a bare label with nothing beneath it, which reads as a figure that failed to load. There is no currency to key a figure to, so there is nothing to render per currency; one dash is the whole cell.

A code in the label cannot hold two currencies. The code moves onto the value, and each cell renders one line per currency. **Currency is a property of the subscription, not of the channel or the client** (BE-27): a single client can hold benches billed in different currencies, so a client row and a channel row both need this. The Client Billing and Grand Total rows beneath already work this way — this brings the tree rows in line with them.
- **Download as Excel** closes the section, exporting the shown period for **this channel only**.


---

## Client hierarchy — SD-3416 / SD-3417

This story **renders** the hierarchy; it does not build it. Structure, depth and parentage all come from those items.

Full logic, the generation-label table and three worked test fixtures: **§H of `../../ENGINEERING-BRIEF.md`**.

### What this surface must honour

- Built from Zoho's **`Parent Brand`** field, derived on read — nothing about the tree is stored.
- **Depth-unbounded.** No hardcoded generation limit.
- **Siblings supported at every depth** — any number of brands may share a parent, and a parent may itself be a child, so branching occurs at any level.
- Generation labels (`ROOT`, `PARENT`, `CHILD GENERATION 1…n`) are computed from **depth on render**. Re-parenting a brand changes its label with no migration.
- Siblings at equal depth carry the **same** label.
- Sibling order is **deterministic** — alphabetical by client name, so the tree does not reshuffle between loads.
- Aggregates roll up the **selected client and all descendants** — never ancestors, never a sibling's subtree.
- A **cycle** in `Parent Brand` is detected and reported, not followed.
- A `Parent Brand` naming a **missing or archived** record is treated as a Root and flagged.
