# FE — Client requests & their acknowledgements

Build spec for **SD-3496**, under epic **SD-3492**. **9 templates.**

Four flows, each firing **two emails in one run** — the internal one and a **copy to the requester**:

| Flow | Internal | Copy to requester |
|---|---|---|
| **Overage hours requested** | `12-overage-request.html` → CX | `12-overage-request-manager.html` |
| **New brand requested** | `13-new-brand-request.html` → SS + HC + CX | `13-new-brand-request-manager.html` |
| **New Virtual Bench requested** | `11-bench-request.html` → HC | `11-bench-request-manager.html` |
| **Capacity plan change requested** | `11-plan-change-request.html` → CX + HC + SS | — |

Plus two internal-only emails, neither of which has a copy to the requester:

| Flow | Internal | Why no copy |
|---|---|---|
| **Help requested** | `14-help-request.html` → CX | The client already knows what they typed |
| **Engineer activation requested** — added 16 Sep | `15-engineer-activation-request.html` → HC | The requester's confirmation is the in-app pending state on the engineer's own card (SD-3474) |

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

## Engineer activation requested — added 16 Sep

`15-engineer-activation-request.html` → **humancapital@castillians.com**. Fires when an Admin or Manager submits **Request to activate engineer** from the Engineers section of a bench (SD-3474).

- **The email exists to answer one question: which manager wants which engineer on which bench.** Those three, plus the client, are the subject line and the first sentence — not buried in the label rows.
- Subject: `Engineer activation requested — <engineer>, <bench>`. Preheader: `<manager> has asked us to activate <engineer> on <bench>.`
- Label rows, in order: **Engineer** (name · **email address** · vetting score), **Virtual Bench** (name — bench type), **Client**, **Requested by** (name, role · email address), **Requested on** (timestamp).
- The engineer's **email address is carried** so Human Capital can reach them without looking the record up; their **rate and earnings are not** (BE-04, BE-08) — this is an internal email, but the rate has no business in it.
- The manager's optional message renders **quoted in a tinted panel**, verbatim.
- **Human Capital, not CX.** Engineer-facing and recruitment flows are Human Capital's; a client-facing request would go to CX. This one is about a person we engage.
- **No engagement dates.** The form collects none — a request is a queue item, not a contract — so the email carries none. _Outdated on 16 Sep. Previously the modal collected a start date, an end date and an “Open-ended engagement” flag; if any template or payload still carries `startDate` / `endDate` for this flow, drop them._
- **CTA: “Open the bench” →** `castillians.com/internal-dashboard/v-benches/manage` — the Manage Subscription modal, where the activation is actually performed. Signed out, the reader hits the **internal** sign-in and is redirected there (BE-30).
- Beneath the CTA, two plain notes: check the bench's **capacity plan** first, since another engineer on the same plan adds no hours; and **nothing has changed** — no roster row, no allocation, nothing billed — until the team actions it.
- **A removal request sends no email.** It raises the same kind of request record (SD-3474) but is handled in the internal queue; only activation notifies.

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
