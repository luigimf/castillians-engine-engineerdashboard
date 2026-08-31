# FE — Client requests & their acknowledgements

Build spec for **SD-3496**, under epic **SD-3492**. **8 templates.**

Four flows, each firing **two emails in one run** — the internal one and a **copy to the requester**:

| Flow | Internal | Copy to requester |
|---|---|---|
| **Overage hours requested** | `12-overage-request.html` → CX | `12-overage-request-manager.html` |
| **New brand requested** | `13-new-brand-request.html` → SS + HC + CX | `13-new-brand-request-manager.html` |
| **New Virtual Bench requested** | `11-bench-request.html` → HC | `11-bench-request-manager.html` |
| **Capacity plan change requested** | `11-plan-change-request.html` → CX + HC + SS | — |

Plus `14-help-request.html` → CX, which has no copy: the client already knows what they typed.

---

## Shared rules

Everything in **SD-3493** applies: the 600px Arial shell, the hidden preheader, label/value rows, the three panel meanings, BE-30 CTA and sign-in behaviour, money formatting, no unsubscribe, and the four-client test pass. This document covers only what is specific to these templates.

## A request changes nothing — and both emails say so

The substance of this story. Every one of these creates a **request**, never the thing requested.

- The internal email states plainly that **nothing is provisioned, priced or billed**.
- **The client's copy is more explicit still**: nothing is set up, nothing is charged, and it exists only once they have agreed the plan and rate with us. A client who reads a confirmation and assumes they now have a bench will plan work around it.
- Neither email implies the change has been made.

## The internal emails carry every field verbatim

- Every field **word for word** — never trimmed, summarised or normalised. Free-text like *"not sure yet"* is carried as written; it tells us something.
- The client's **message renders in a tinted panel**, quoted.
- Each names **who asked**, **which client and brand**, and **when**.
- **New Virtual Bench** carries brand, **monthly engineering hours** and **billing currency** — currency is per subscription (BE-27), so a client may ask for a bench billed differently from their others.
- **New brand** carries the name, its relationship to existing brands, the email domains, and the two Zoho actions.
- **Capacity plan change** carries current plan and period, requested plan, and commencement date.

## The copies to the requester

- **Same submitted values**, **minus the internal mechanics** — no overage rate, no cost estimate, no Zoho actions, no internal timestamp.
- The overage copy restates that **only hours actually used are billed** — the authorisation is a ceiling, not a purchase.
- Each invites a **reply if a value is wrong**. That is the correction path: a client cannot edit a submitted request.

## Capacity plan change — the timing rule

- **Upgrades may commence any time. Downgrades only from the auto-renew date**, since the current period is committed and billed.
- An **amber note above the CTA** states the plan **does not change itself**: apply the new hours at the **start of the month** the change commences, via **Manage Subscription → Monthly Engineering Hours**.
- It names the consequence: the bench runs the whole period on its old capacity and bills at it.
- **Above the CTA, not below** — it must be read before the button.

## Pending state

- Each leaves a **pending request on the client's own dashboard**, so nobody submits twice wondering whether the first arrived. The emails and that state describe **one record**.
- A **new-bench request from Subscriptions is the same record** as one from Virtual Benches (SD-3470, SD-3482) — one request, two entry points, one pair of emails.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| The submitted request | The form that raised it | The internal queue, the client's pending state, both emails |
| Capacity plan and period | `includedHours` + per-bench period (SD-3459) | Bench page, Subscriptions, Internal, Channel page |
| Subscription currency | Per bench (BE-27) | Every rate and total on the platform |
| Client-facing blended rate | The configured blended rate (BE-08) | The cost implication on the request forms |

**Acceptance criteria**

- **Both emails render the same submitted values from one payload.** They can never disagree.
- **The engineer-facing rate, mark-up and retained margin never appear in a client copy** (BE-04, BE-08).
- **A request writes no subscription, capacity, bench or billing row.** The only row written is the request itself.

---

## Reference

```
../SD-3493/                   the shell and shared rules
../../manager-dashboard/SD-3470/   the new-bench request flow
../../manager-dashboard/SD-3471/   the capacity plan change request
../../ENGINEERING-BRIEF.md    BE-04/08 confidentiality, BE-20 triggers, BE-27 currency
```

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.
