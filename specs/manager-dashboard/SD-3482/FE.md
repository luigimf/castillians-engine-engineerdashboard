# FE — Subscriptions Page: Manage Subscription & New Bench Request

Angular handoff for **SD-3482**. The two actions on the Monthly Capacity Plans card (SD-3481). Page access and brand scope: **SD-3480**.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header is a navigation device only and **must not be built**.

---

## 1. Nothing here writes to a subscription

Both actions raise **requests** into the Castillians queue (SD-3465). Confirmations say what happens next and who does it. No figure on any surface moves until the team acts.

## 2. Which Admins can act on a bench

> **Changed 23 Sep.** Previously: "Admin only" — one Admin.

- **Every Admin who can see a bench can act on it**: the bench's own brand Admin and every Admin above that brand. The root Admin can act on every bench in the channel.
- **None outranks another.** No approval step between Admins. Every request goes straight to the Castillians queue.
- **One open capacity request per bench across all of them.** If one Admin has a request open, every other Admin sees it as pending, with the name of whoever raised it.

## 3. Manage Subscription

- 35px outline button per bench row, shown to every Admin who can see that bench.
- Opens the **same Change capacity plan modal** as the bench page (SD-3471). One flow, two entry points.
- Current plan and current period first; new hours, from when, optional note.
- Cost implication at the **client-facing** rate, in the bench's own currency, always visible.
- Pending state: what was asked, when, **by whom**.

## 4. Subscribe to a new Virtual Bench

- Divider + button beneath the last brand group, Admin only.
- Opens the **Request a new Virtual Bench** modal from SD-3470 — same form.
- Its **brand field lists only the brands in the Admin's own subtree**.

## Reference

```
BE.md                          request endpoints, subtree guard
../SD-3480/                    page access and the subtree rule
../SD-3471/                    the Change capacity plan modal
../SD-3470/                    the new-bench request flow
../../../prototype/index.html  → Manager → Subscriptions; Viewing as → Admin / Sub-brand Admin
```
