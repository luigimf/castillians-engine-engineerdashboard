# FE — Client feedback: weekly check-in & response

Build spec for **SD-3497**, under epic **SD-3492**. **2 templates.**

| Template | To | Fires |
|---|---|---|
| `06-satisfaction-weekly.html` | Manager | Weekly, one per bench |
| `07-satisfaction-response.html` | Customer Experience | The moment a rating is tapped |

---

## Shared rules

Everything in **SD-3493** applies: the 600px Arial shell, the hidden preheader, label/value rows, the three panel meanings, BE-30 CTA and sign-in behaviour, money formatting, no unsubscribe, and the four-client test pass. This document covers only what is specific to these templates.

## The weekly check-in

- **Weekly, one per bench**, to the bench's Managers.
- **Three equal-width traffic-light CTAs** — going well / some concerns / needs attention.
- **One tap records the response.** No sign-in, no form, no landing page asking them to choose again.
- Carries **hours context** alongside the ask — used vs plan, e.g. `144 / 160h — 90%`. A rating without context invites a rating about the wrong thing.
- **Silence is taken as "going well".** A **white panel** says so plainly. It is stated in the client's words, **not** as "we will take it as a green" — the traffic light is our internal vocabulary, not theirs.
- Beneath it, an **amber panel** with an escalation route to `customerexperience@castillians.com`. The three taps are deliberately coarse, so anything needing a person rather than a colour needs a named way out — otherwise the amber tap becomes the only channel for real problems.

## The response notice

- Fires **the moment** a Manager taps a rating — not batched, not daily.
- States the **response**, the **bench**, the **client**, **who responded** and **when**.
- The **subject names the rating**, so an amber or red is visible in the inbox list without opening it.
- **CX, not Human Capital.** This is a client-relationship signal.

## Rating semantics

- Stored against **the bench and the week**, so history builds per bench rather than per client.
- A second tap in the same week **replaces** the first — someone re-reading and tapping again is correcting themselves, not rating twice.
- **A non-response is not a green in the data.** It is recorded as no response, and treated as "going well" for alerting only. Storing silence as an explicit green would inflate the satisfaction record with answers nobody gave.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| Hours used vs plan | Approved work logs against the bench period (SD-3459) | Bench page, benches list, Internal, Channel page |
| The rating | Written by the one-tap link | The internal notice, the bench's satisfaction history |
| Bench Managers | Bench membership (SD-3471) | Who receives the check-in |

**Acceptance criteria**

- The hours figure **matches the bench page** for that bench and period, to the hour.
- Recipients are **the bench's own Managers** — a Manager with no access to a bench is never asked about it.
- The one-tap link **records a response without a session**; the deliberate exception to BE-30.
- An already-used link is **idempotent within the week** — it updates rather than duplicating.

---

## Reference

```
../SD-3493/                   the shell and shared rules
../../ENGINEERING-BRIEF.md    BE-20 triggers and recipients, BE-30 CTAs and this exception
```

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.
