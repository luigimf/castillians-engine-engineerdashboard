# FE — Engagements: expanded bench entry

Angular handoff for **SD-3465**. Five sections inside the bench accordion: overage controls, capacity allocation, notes, order forms, work logs.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — `Card`, `Input`, `Button`, `Avatar`, `StatusBadge`, `Toast`.
Depends on **SD-3464** for the page frame and collapsed row.

---

## Overage controls

Both toggles ~39×23px, ink when on.

| State | Set by | Behaviour |
|---|---|---|
| Off | Allow Overages off | Logging refused at 100% of plan. No approvals, no overage emails |
| Default 120% | Allow Overages on, Authorised Total `0` | To 120% automatic; beyond that, manual approval |
| Authorised amount | Allow Overages on, Authorised Total `> 0` | The authorised figure **replaces** the 20% tolerance |
| Unlimited | Unlimited Overage on | Nothing capped, nothing queued, no notifications |

- Helper text states what the current state does and rewrites when toggled. **Toggling must not shift the switch vertically** as helper text of differing length swaps in.
- Authorised Total Overage — numeric only, hours beyond 100% of plan, visible only when Allow Overages is on. **160h plan + 40h block accepts 200h, not 232h.**
- Unlimited suppresses the authorised block and reads `0` — a stale value must never linger.
- The grant expires at period end: "Automatically switched off for the following month, after this period has been billed." It holds for the whole final day.

## Capacity allocation

- Renders whenever the bench has one or more engineers; a solo engineer shows a fixed, disabled **100%**.
- Row per engineer: avatar, name, email, numeric `%` field, and the resolved figure — "Can log up to 63 hours this period".
- **Engineers see hours, never percentages.**
- Total must be **exactly 100%**: chip green at 100, amber below with "n% left", red above with "n% over".
- Two engineers → changing one auto-adjusts the other. Three or more → each set independently and validated.
- Where an authorised block exists, each row also shows their share of it.
- **Save Changes** disabled until exactly 100%; success toast on save.

## Notes

Free text beneath the allocation controls, above the divider. Internal only — never shown to Manager or Engineer. Placeholder: "Leave any notes about the allocation of hours, or other details for this Virtual Bench".

## Order form history

One card per overage request, newest first: **period as a full date range** (never a bare month), who raised it and when, hours requested, status tag (Approved / Pending / Declined), and the request message quoted where given. Populated by the Manager _Request overage hours_ flow.

## Work log entries

- Newest first, batched at **10** with **See more (n remaining)** loading 10 more.
- Each entry: hours, date, engineer, description, status tag, edit-history accordion.
- Entries needing review carry **Approve** and **Decline** at Small Tertiary size, grouped with an "Approval required" tag.
- **Decline requires a message** — the modal cannot be submitted empty.
- The reveal count resets when the accordion is closed and reopened.
- An entry **manually approved here after the end date of its own period** carries the **"Payable the following period"** label beside its status tag, from `payableNextPeriod` (SD-3467). Approving an in-period entry never produces it.
- Entry layout, history rendering and the ended-engagement rules are specified once in **SD-3467** — this surface reuses that component rather than restating it.

---

## General

- Responsive; Chrome, Firefox, Safari; existing design-system components only.
- Every transition `300ms cubic-bezier(0.35,0,0.25,1)`.

---

## Reference

```
BE.md                          overage state machine, allocation, work logs
../SD-3467-engagements-work-log-entry/   the shared entry component
../../ENGINEERING-BRIEF.md     BE-11 to BE-14 overage and approval rules
../../../prototype/index.html  → Internal → Engagements → Virtual Benches → expand a bench
```

**Prototype:** _Core Platform_ has an authorised block, _Payments Squad_ and _Mobile App_ have overages off, and _Insurance Web_ has a two-engineer allocation split.
