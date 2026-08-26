# FE — V Bench page: Engineers

Angular handoff for **SD-3474**. The Engineers section inside the SD-3471 page shell. Replaces the existing section at `castillians.com/v-benches/{id}`.

> ### ⚠ Use the engineer profile cards already in production
>
> The cards in this section must be the ones **currently implemented in the platform** — not the simplified cards in the prototype. The prototype's version exists only to show the section's structure, spacing and grouping. **Do not rebuild the cards from it.**
>
> Confirm the component path with Samuel before starting. Preserve the card's own responsive behaviour rather than overriding it.
>
> Everything else in the prototype — the grouping, the request flows, the copy, the states — is authoritative.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle exists only so one file can demonstrate all three dashboards. **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System, plus the **existing production engineer card**. No new one-offs.

---

## Three groups

| Group | Who | Action (Admin + Manager) |
|---|---|---|
| **Activated** | On the bench now | **Request removal** |
| **Available** | Engaged with this client, not on this bench | **Request to add** |
| **Recommended** | Proposed by Castillians, matched on the bench's skills mix | **Request to add** |

- Each group heading states its count.
- **Activated is ordered by name**, so the list is stable between visits rather than reordering as hours change.
- The **Available / Recommended distinction must be legible**: available engineers are people the client **already knows**; recommended ones are new to them.
- **Recommended** carries a short line explaining *why* the group exists — matched to this bench's skills — so it does not read as an advert.
- **An empty group is omitted, not rendered empty.** An empty "Recommended" heading suggests we have nothing to offer.

---

## Nothing here changes the bench

This is the core of the story, and the change from today's behaviour.

- **Requesting removal removes nobody.** The engineer stays on the bench, keeps logging hours, and keeps appearing in every figure until the Castillians team actions it.
- **Requesting an addition adds nobody.** No activation, no allocation change, nothing billed.
- Both are **requests into the Castillians team's queue**, each carrying the manager's optional note.
- Every confirmation says **what happens next and who does it**. It must not imply the change has been made.
- **A pending request renders on the engineer's own card**, stating what was asked and when — so a manager does not raise it twice. While pending, that engineer's action reads as pending rather than inviting a duplicate.

### Request removal

- Confirmation names the **engineer** and the **bench** — never a bare "Are you sure?".
- States plainly that the engineer continues until the team actions it, and that their logged hours are unaffected.
- Optional reason. **(optional)** in the **same ink** as the field title, never a lighter grey.
- Removing the **last activated engineer** is allowed but warns clearly: the bench will have nobody on it and no hours will be logged.

### Request to add

- Confirmation names the **engineer** and the **bench**.
- States that the team will confirm availability **and any effect on the capacity plan** — a new engineer may need more hours, and that is a conversation, not an automatic change.
- Optional note: when they are needed, and what for.

Both: submit disables on submit; double-submit is guarded **server-side** too.

---

## Confidentiality

- An engineer's **rate, earnings and personal contact details never appear** in this section (BE-04, BE-08).
- **Allocation percentages never appear.** The Castillians team maintains them; a manager sees hours only.
- Available and Recommended are scoped to **this client**. A manager never sees an engineer's engagement with another client, and never sees an engineer not offered to them.

---

## States (§G2)

| State | Behaviour |
|---|---|
| Loading | Skeleton cards at the production card's natural height. No spinner that collapses the section |
| Empty — activated | `No engineers are on this bench yet.` plus, for Admin and Manager, a pointer to the groups beneath |
| Empty — available / recommended | The group is **omitted** |
| Error | Inline with a retry **inside** the section |

Empty copy `var(--gray-700)`, 13px, body font.

## Roles — the platform rule

**A Viewer's controls are absent, not disabled.** A Viewer sees the same three groups with no request affordance anywhere. Role is resolved **server-side**; the client renders what it is given.

---

## Reference

```
BE.md                          endpoints, the requests, scoping
../SD-3471/                    the page shell this mounts into
../SD-3472/                    the skills mix that drives recommendations
../../internal-dashboard/SD-3465-bench-entry/   where requests are actioned
../../ENGINEERING-BRIEF.md     §G2 states, BE-04/08 confidentiality
../../../prototype/index.html  → Manager → Virtual Benches → open a bench → Engineers
```
