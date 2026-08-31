# FE — Capacity & overage notifications

Build spec for **SD-3495**, under epic **SD-3492**. **9 templates.**

Most thresholds fire **two emails for one event** — client-facing and internal. **Separate templates, never one with a flag.**

| Threshold | Client (Manager) | Internal (CX) |
|---|---|---|
| **90%**, overages OFF | `05-capacity-90-manager.html` | `05-capacity-90-off.html` |
| **100%**, overages OFF | `06-capacity-100-off-manager.html` | `06-capacity-100-off.html` |
| **100%**, overages ON | — none | `07-capacity-100-on.html` |
| **Approval ceiling reached** | `08-capacity-120-on-manager.html` | `08-capacity-120-on.html` |
| **Authorised overage used up** | `08-authorised-overage-used-manager.html` | `08-authorised-overage-used.html` |

---

## Shared rules

Everything in **SD-3493** applies: the 600px Arial shell, the hidden preheader, label/value rows, the three panel meanings, BE-30 CTA and sign-in behaviour, money formatting, no unsubscribe, and the four-client test pass. This document covers only what is specific to these templates.

## Two audiences, two templates

- **The client's version carries no overage mechanics** — no internal rate, no margin, no tolerance arithmetic (BE-04, BE-08). It says what it means for them and what they can do.
- The internal version carries the full position: plan, tolerance, authorised total, used, rate, capacity consumed.
- **Never one template with a conditional block.** The two say genuinely different things, and a shared template is how internal figures reach a client.

## Overages OFF — 90% and 100%

- **90%** warns logging will be **blocked at the plan**; the client's version invites a capacity request.
- **100%** states engineers **can no longer log hours** against the bench.
- Both fire **once per bench per period**.
- The client's 100% notice carries an **amber panel** with a route to more hours — the one client-facing capacity email where the bench is **hard-blocked**, so it needs it most.

## Overages ON — 100%

**Internal only.** The client is not alerted at 100% when overages are on: hours keep flowing inside the tolerance and there is nothing for them to do. They are notified once the **ceiling** is reached.

## The approval ceiling — either of two rules

A bench reaches the point where further entries need approval **one of two ways**, and the email must say **which applied**:

1. It passes **120% of the capacity plan** where **no** Total Authorised Overage is set, **or**
2. It **exhausts the authorised block** that was set.

- The **Approval ceiling row names the rule** — e.g. `192h — 120% tolerance, no authorised block set`.
- Fires **once per bench per ceiling, not per period**: authorising a further block **raises the ceiling and arms the notice again**.
- Where a bench has an Authorised Total Overage, `08-authorised-overage-used` **replaces** the 120% notice — a named block supersedes the tolerance, so the two never both fire for one bench.

## The client's route to more hours

Every client-facing capacity email carries an **amber panel**: raise it from the Virtual Bench page, or email **`customerexperience@castillians.com`**.

**CX, not Human Capital.** HC owns the engineer-facing flows; a client sent to HC has been sent to the wrong team and will not know it.

## Unlimited overage

A bench with **unlimited overage sends nothing** at any ceiling. There is no threshold to announce, and a notice implying one would be wrong.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| Capacity plan and period | `includedHours` + per-bench period (SD-3459) | Bench page, benches list, Internal, Channel page |
| Hours used | **Approved** work logs only | Every capacity figure on every dashboard |
| Overage state and authorised total | Bench overage settings (SD-3465, BE-11) | Internal bench entry, Manager bench page |
| Overage rate | Blended rate, ×1.25 in the first 3 months (BE-06) | Internal bench entry, billing history |
| "Overages agreed" | Authorised total where set, else the 20% tolerance (BE-13) | Approval-required email, Internal bench entry |

**Acceptance criteria**

- **Pending-approval hours are excluded** from every figure quoted.
- Figures **reconcile to the bench page** for the same bench and period, to the hour.
- **Once per bench per period** (or per ceiling). Re-crossing after an authorisation is a **new** ceiling, not a repeat.
- A bench crossing two thresholds in one logging action sends **only the higher**.
- **The engineer-facing rate and retained margin never appear**, internal or client (BE-04, BE-08).

---

## Reference

```
../SD-3493/                   the shell and shared rules
../../ENGINEERING-BRIEF.md    BE-04/05/06/08 billing and confidentiality, BE-11/13 thresholds
```

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.
