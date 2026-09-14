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

- **Weekly, one per bench**, to the organisation's **Admin** and every **Manager assigned to that bench**.

### Recipients — *updated 14 Sep*

| Recipient | Included |
|---|---|
| The organisation's **Admin** | **Always**, for every bench in the organisation |
| **Managers assigned to that bench** | Yes — bench membership, not organisation membership |
| Managers **not** on that bench | **No** |
| **Viewers** | **No** |

_Outdated on 14 Sep. Previously: "Weekly, one per bench, to the bench's Managers." — the Admin was not a recipient._

- **The Admin receives the check-in for every bench**, because they carry the commercial relationship across the organisation and are the person CX escalates to when a bench goes amber or red. They must not learn about a problem from the follow-up.
- **Managers are resolved by bench membership**, not organisation membership. A Manager on three of five benches receives three check-ins, not five — asking someone about a bench they cannot see produces a meaningless rating.
- **One email per recipient per bench**, so an Admin with five benches receives five separate check-ins, each with its own bench name, hours context and one-tap links. Do not batch them into a digest: the rating is stored per bench, so one tap must map to exactly one bench.
- **An Admin who is also assigned to a bench receives one email for it, not two.** De-duplicate by address.
- **Viewers are never recipients.** A Viewer cannot act on a bench, so a satisfaction rating from one would carry no accountability.
- **Every recipient's tap is its own response.** Responses from different recipients **do not replace each other** — a bench with an Admin and three Managers can produce four responses in one week, and **each one fires its own notice to CX**. Only a repeat tap by the **same** recipient replaces their own earlier answer.
- **Three equal-width traffic-light CTAs** — going well / some concerns / needs attention.
- **One tap records the response.** No sign-in, no form, no landing page asking them to choose again.
- Carries **hours context** alongside the ask — used vs plan, e.g. `144 / 160h — 90%`. A rating without context invites a rating about the wrong thing.

### Subject, heading and framing — *updated 14 Sep*

| Element | Copy |
|---|---|
| Subject | **How is your {bench} Virtual Bench performing?** |
| Heading | **Your weekly satisfaction check-in** |
| Sub-line | A quick check-in on how your Virtual Bench, **{bench}**, is performing. One tap is all we need — your feedback helps us understand how things are going and where we may need to provide additional support. |

_Outdated on 14 Sep. Previously: subject "How is {bench} going?", heading "Your weekly check-in", and the sub-line "A quick read on how your Virtual Bench, {bench}, is performing. One tap is all we need — it takes a second and it shapes how we support the bench."_

- **The subject says "Virtual Bench" explicitly.** A Manager on several benches reads the bench name in an inbox list; "How is Core Platform going?" could be a project, a team or a person. Name the product.
- **"Satisfaction" belongs in the heading**, because that is what the record is called everywhere else — the response notice, the bench's satisfaction history and the internal alerting all use the word. A Manager who taps here should recognise the language when CX follows up.
- The sub-line's second half states **what the tap is for** — how we support the bench — rather than how little it costs them. "It takes a second" sells the effort; "where we may need to provide additional support" explains the purpose.
- **The rating question reads "How satisfied are you with your Virtual Bench?"** — the full product name, not "your bench".

### The three criteria — *added 9 Sep, wording updated 14 Sep*

Above the CTAs, a **grey panel headed "What to consider before you rate"** lists the three criteria the rating covers, numbered, each with its question title and a plain-language line beneath, separated by hairlines:

| # | Question | Criteria shown to the Manager |
|---|---|---|
| 1 | **Is progress satisfactory?** | Are the assigned engineers making satisfactory progress against the objectives agreed with your team? |
| 2 | **Is productivity satisfactory?** | Are the assigned engineers delivering the level of productivity expected by your team? |
| 3 | **Is responsiveness satisfactory?** | Are the assigned engineers and Castillians team responding appropriately to requests, priorities and communication needs? |

_Outdated on 14 Sep. Previously: questions 2 and 3 read "Is productivity acceptable?" and "Is responsiveness acceptable?"._

- **All three questions use "satisfactory"**, matching question 1 and the word the whole flow is named after. "Acceptable" set a lower bar than "satisfactory" and asked two questions on a different scale from the first.

- **The three criteria are guidance, not three questions.** There is exactly **one submission** — the single row of three traffic-light CTAs beneath them, and the sub-line **"One tap, taking all three together."**
- **The CTA hrefs are unchanged** (`?r=green|amber|red`), so the response notice, the stored rating and the alerting all behave exactly as before. **Nothing per-criterion is captured, stored or emailed.**
- Copy reads **"Castillians team"**, not "Castille" — the legacy name never appears in client-facing email.
- The panel is grey `#F8F8F8` with a `#E5E5E5` hairline border. It is **not** one of the three panel meanings from SD-3493 — it carries no status, it is reading material before a tap.

**Rejected alternative, for the record:** three separate CTA rows, one per question. It was built and withdrawn — three rows means three submissions, which would change the response notice, the stored shape of a rating and every downstream alert. The criteria earn their place as context for a single tap.
- **Silence is taken as "going well".** A **white panel** says so plainly. It is stated in the client's words, **not** as "we will take it as a green" — the traffic light is our internal vocabulary, not theirs.
- Beneath it, an **amber panel** with an escalation route to `customerexperience@castillians.com`. The three taps are deliberately coarse, so anything needing a person rather than a colour needs a named way out — otherwise the amber tap becomes the only channel for real problems.

### The escalation panel — *updated 14 Sep*

Two paragraphs, in this order:

> Have a concern or something you'd like to discuss? Email us at **customerexperience@castillians.com**, and we'll follow up directly.
>
> Your response helps us understand the overall health of the engagement, while any additional detail helps us determine how best to support you.

_Outdated on 14 Sep. Previously one sentence: "**Something to raise?** Email us at customerexperience@castillians.com and we will pick it up directly — these answers tell us the temperature, but not the detail."_

- **It opens with the reader's situation, not a headline.** "Something to raise?" in bold read as a section label; "Have a concern or something you'd like to discuss?" is the question a Manager is already asking themselves.
- The second paragraph states **what each part is for** — the tap gives us engagement health, the email gives us the detail we need to act. The old "temperature, but not the detail" metaphor said the same thing but left the reader to work out the implication.
- The two paragraphs are separated by a **blank line inside the panel**, not split into two panels.
- Panel treatment is unchanged: amber `#FEF6E7` with a `#F0D9A8` hairline, ink `#7A5A12`, the address a bold mailto in the same ink.

## The response notice

- Fires **the moment** a rating is tapped — not batched, not daily. **One notice per submission**, so several recipients rating the same bench in the same week produce several notices.
- States the **response**, the **bench**, the **client**, **who responded** and **when**.

### Heading and row labels — *updated 14 Sep*

Heading: **Weekly Satisfaction Check-in Received**. Row labels, in order:

| Label | Value |
|---|---|
| Virtual Bench | The bench name |
| Client | The client name |
| **Respondent** | Name and role, e.g. `James Whitmore (Admin)` |
| **Response received** | Date and time with timezone |
| **Reporting period** | The week as a date range |
| **Capacity vs utilised hours at time of rating** | `144 / 160h — 90%` |
| Previous rating | **This respondent's** prior response and its date — not the bench's, which with several raters would say nothing about whether this person's view has changed |

_Outdated on 14 Sep. Previously the heading read "Satisfaction response received", and four labels read "Responded by", "Received", "Week rated" and "Hours at time of rating"._

- **The heading names the notification, in title case**, so it reads as a distinct email type in a CX inbox rather than as a sentence fragment.
- **"Respondent"** over "Responded by" — a noun for a label whose value is a person, consistent with every other label on the table.
- **"Response received"** and **"Reporting period"** are unambiguous. "Received" alone invited "received what?", and "Week rated" read as a verb phrase.
- **"Capacity vs utilised hours at time of rating"** says which two numbers the pair is. "Hours at time of rating" gave no clue that `144 / 160h` was utilised against capacity.
- **Values, sources and derivations are unchanged** — this is label wording only, and nothing recomputes.
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
