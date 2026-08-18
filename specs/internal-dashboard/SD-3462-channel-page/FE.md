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
- Differing currencies → a total **per currency**. Never combined, never converted.
- Currency codes in labels — `BILLING (EUR)` — bare values, exact cents.
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
