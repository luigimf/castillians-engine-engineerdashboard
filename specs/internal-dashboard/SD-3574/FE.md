# FE — Engagements: "Managed by" in the expanded bench entry

Angular handoff for **SD-3574** (Enhancement, epic SD-3531). Extends **SD-3465**. The expanded bench entry gains a new **first** section.

**Design system:** V2 Castillians Design System — `Avatar`. No new components.

---

## Placement

- Internal → Engagements → **Virtual Benches** tab → open a bench row.
- **First** block inside the accordion body, above the Allow Overages / Capacity Allocation container.
- Everything else in the body (overage controls, capacity allocation, notes, order forms, work logs) is unchanged and moves down one place.

## Container

| Property | Value |
|---|---|
| Background | `#FFFFFF` |
| Border | 1px `var(--border)` |
| Radius | 8px |
| Padding | `18px 22px` |
| Gap below | 18px (`margin-bottom`) |

Same white container treatment as the Allow Overages / Capacity Allocation block beneath it.

## Content

The content is identical to the **Managed by** line in the Channel & Billing → Channel Breakdown bench accordion (SD-3462).

- **Label:** `MANAGED BY`: Bricolage Grotesque 12px / 700, uppercase, 0.05em tracking, `#141313`, 10px margin below.
- **Row:** flex, align centre, **14px** gap, wraps.
  1. `Avatar`, 28px, initials from the manager's name.
  2. **Name**: 12px / 600, `var(--ink-900)`.
  3. **Email**: 12px / 400, `var(--gray-700)`.
  4. **Phone**: 12px / 400, `var(--gray-700)`. Left out entirely if missing; no dash or placeholder.
  5. **Role chip**: 1px `var(--gray-150)` border, `var(--radius-md)`, padding 10px, Montserrat 11px / 400, line-height 100%, `var(--gray-50)` fill. Text is the role on that bench: Admin / Manager / Viewer.

## Rules

- A bench always has at least one manager, so the block always renders.
- The same bench shows the same manager here as on Channel Breakdown. A mismatch is a defect.
- No interaction: the name and email aren't links in this release.
- Reflows at narrow widths. Items wrap and never overlap or truncate.

## Prototype

`repo/prototype/index.html` → **Internal** → **Engagements** → **Virtual Benches** → open any bench.
