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

### The three criteria — *added 9 Sep*

Above the CTAs, a **grey panel headed "What to consider before you rate"** lists the three criteria the rating covers, numbered, each with its question title and a plain-language line beneath, separated by hairlines:

| # | Question | Criteria shown to the Manager |
|---|---|---|
| 1 | **Is progress satisfactory?** | Are the assigned engineers making satisfactory progress against the objectives agreed with your team? |
| 2 | **Is productivity acceptable?** | Are the assigned engineers delivering the level of productivity expected by your team? |
| 3 | **Is responsiveness acceptable?** | Are the assigned engineers and Castillians team responding appropriately to requests, priorities and communication needs? |

- **The three criteria are guidance, not three questions.** There is exactly **one submission** — the single row of three traffic-light CTAs beneath them, and the sub-line **"One tap, taking all three together."**
- **The CTA hrefs are unchanged** (`?r=green|amber|red`), so the response notice, the stored rating and the alerting all behave exactly as before. **Nothing per-criterion is captured, stored or emailed.**
- Copy reads **"Castillians team"**, not "Castille" — the legacy name never appears in client-facing email.
- The panel is grey `#F8F8F8` with a `#E5E5E5` hairline border. It is **not** one of the three panel meanings from SD-3493 — it carries no status, it is reading material before a tap.

**Rejected alternative, for the record:** three separate CTA rows, one per question. It was built and withdrawn — three rows means three submissions, which would change the response notice, the stored shape of a rating and every downstream alert. The criteria earn their place as context for a single tap.
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
- **The rating is one value, not three.** The criteria panel does not decompose it — there is no per-criterion field on the record, and a red does not say which of the three drove it. That detail arrives through the escalation route, by email, from a person.
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
