# FE — Virtual Benches page & Create a new bench

Angular handoff for **SD-3470**. Replaces the existing page at `castillians.com/v-benches`.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — compose from its components; no new one-offs.

---

## Components

| Component | Responsibility |
|---|---|
| `VBenchesPageComponent` | Owns the client filter and the request-modal state |
| `VBenchCardComponent` | One bench: type tag, name, engineer count, capacity bar, period |
| `CapacityBarComponent` | Shared — the 8px track, bands and percentage. Also used on the bench page |
| `NewBenchRequestModalComponent` | The request form |

`CapacityBarComponent` is worth extracting rather than inlining: it appears on this page, the bench page's hours card, and the Internal and Engineer dashboards. Its band logic must be identical in all four.

---

## The grid

- `repeat(auto-fit, minmax(330px, 1fr))`, 24px gap. **Wraps, never scrolls horizontally.**
- Card: white, `2px solid #E5E5E5`, 8px radius, 30px padding; hover lift `0 8px 20px rgba(0,0,0,.1)`. The whole card is the click target.
- **Sorted by capacity used, descending** — the bench nearest its ceiling reads first. Ties break on name.

| Element | Treatment |
|---|---|
| Type tag | Permanent / Project, with its icon, in the type's own colour treatment |
| Bench name | Bricolage 22px/700 |
| Client | Body 14px `var(--gray-700)` — only when the organisation has more than one client entity |
| Engineer count | Activated engineers only |
| Capacity bar | 8px track, 4px radius, percentage beside it, `used / plan` beneath |
| Period | The bench's **own** period as a date range, with days remaining |
| Over-capacity tag | Only when past plan. Fill + weight, never colour alone (§G6) |

**Bands:** ≤59% green `#10b77f`, 60–89% amber `#f59f0a`, ≥90% red `#ef4343`, each on its **tinted** track — never grey. Fill animates from 0 over 300ms `cubic-bezier(0.35,0,0.25,1)`.

**Bar geometry is identical on every card** — the 100% mark sits at the same width whatever the plan, so two cards can be compared at a glance. **Remaining reads `0h` when over plan**, never a negative figure.

---

## Client filter

- One control, 50px height, default **All Virtual Benches**.
- **Rendered only when the organisation has more than one client entity.** A control with one meaningful option is noise.
- Options derive from the benches this member can see, so a Manager never sees a client entity they have no bench on.
- Changing it re-renders with no reload; state is **not** persisted across navigation.

---

## Create a new bench

Button top-right, aligned with the page title. **Absent for a Viewer.**

Fields: bench name (mandatory) · type (mandatory) · client entity (only when >1) · skills needed · engineers needed · monthly hours needed · when needed · anything else (optional).

- Labels Bricolage **16px/400** — the platform standard, **not bold**. Typed input weight 500.
- Mandatory marked with a **black** asterisk, not red.
- Fields 45px; textarea resizes vertically, 96px minimum.
- The **(optional)** qualifier renders in the **same ink** as the field title — never a lighter grey.
- Validation messages sit **beneath the field they concern** — never a toast alone, never one summary at the top.
- Submit disables on submit; double-submit is guarded server-side too.

### It is a request

- Submitting creates a **request**, not a bench. Nothing is provisioned, priced or billed.
- The success copy says the team will come back with a proposed plan and rate. **Do not imply the bench exists.**
- A **pending request renders as a card** in a distinct pending treatment, with **no capacity bar** — there is no capacity yet — and is **not clickable**.

---

## States (§G2)

| State | Behaviour |
|---|---|
| Loading | Skeleton cards at the natural card height. No spinner that collapses the layout |
| Empty — none | A plain sentence, plus the create action for Admin and Manager. For a Viewer, the sentence alone |
| Empty — filtered | `No Virtual Benches for this client.` |
| Error | Inline with a retry **inside** the page body |

Empty copy `var(--gray-700)`, 13px, body font. **The filter stays usable in every state** — an empty result is not an error.

## Roles — the platform rule

**A Viewer's controls are absent, not disabled.** A greyed-out button invites a support ticket asking why it is greyed out. Render the control or do not render it.

Role is resolved **server-side** on every response. The client renders what it is given; it never decides what a role may do.

---

## Reference

```
BE.md                          endpoints, visibility, the request
../SD-3471/                    the bench page a card opens
../../ENGINEERING-BRIEF.md     §G interaction patterns, §A4 roles, BE-02/03 periods, BE-05 overage
../../../prototype/index.html  → Manager → Virtual Benches
```
