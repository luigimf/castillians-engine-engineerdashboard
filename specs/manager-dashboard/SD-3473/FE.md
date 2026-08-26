# FE — V Bench page: Engineer Work Logs

Angular handoff for **SD-3473**. A **new** section inside the SD-3471 page shell.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — compose from its components; no new one-offs.

---

## What a manager sees, and never sees

| Never shown | Why |
|---|---|
| Entries **pending approval** | They may yet be decided either way, and an entry that later changes is worse than one that arrives late. The **count** is disclosed on the hours card — _"8h awaiting approval"_ (SD-3472) — so the omission is never silent |
| **Declined by a person** | The reviewer's message is internal and not meant for the client. Filtered **server-side** — the response never carries the row |
| **Edit history** | Withheld from Manager entirely, again server-side |
| **Approve / decline actions** | No client role approves work logs — that is the Castillians team's job (SD-3467) |
| **Rates, amounts, earnings** | A manager sees hours and what was done (BE-04, BE-08) |

All three roles read the same thing here. **A Viewer is not restricted on this section.**

---

## The entry row

Rows sit flush in a **zero-padding Card**, separated by `1px solid var(--gray-75)`, no rule after the last; first and last carry the card's 8px radius.

| Element | Treatment |
|---|---|
| Hours | Bricolage **22px/700**, as `7h`, on a 60px fixed left rail |
| Date | Beneath, body 13px `var(--gray-700)`, as `7 Aug` — no year |
| Engineer | Avatar 40px with verified badge, name Bricolage 22px/700 |
| Description | Body 14px, 140% line-height, `#787878` |
| Status badge | **Auto-approved** / **Manually approved** (success), or **Auto-declined** (danger) — the platform's `StatusBadge` |

**The status badge stays.** *How* an entry was settled is not noise: **Auto-approved** means it fell inside the agreed hours and passed automatically, **Manually approved** means a person signed it off. Use those labels verbatim — they are the same strings the Engineer and Internal dashboards render. That is the difference a manager asks about when they query an invoice, and it is the same distinction the Engineer and Internal dashboards preserve.

### Auto-declined entries are shown

An entry nobody reviewed before its period's **close on the 3rd** is auto-declined (**BE-29**). **It appears in this list**, with the **Auto-declined** badge in the platform's danger treatment.

- It is shown because the state is **terminal and in the client's favour**: the engineer logged the hours, nobody reviewed them in time, and **the client is not billed for them**. Hiding it would leave the manager's own totals unexplained — an engineer will tell them they logged 8 hours that appear nowhere.
- **Its hours count towards nothing** — not `CAPACITY USED`, not the bar, not billing. The badge is the explanation.
- **No message, no reviewer, no history** — there was no decision to attribute, and there is nothing to quote.
- **Declined-by-a-person stays hidden.** That entry carries a reviewer's message written for internal use, and it is not the client's business why we declined an engineer's claim.
- So the two are not symmetrical, deliberately: an auto-decline is *our* process failing to act, and the client can see that. A human decline is a judgement about an engineer, and it stays internal.

**Newest first**, tie-broken on entry id so the order is **stable across reveals** — an entry must never duplicate or drop at a boundary.

The engineer's name and avatar are shown because a manager reads this per person. There is **no per-engineer filter** — the usage list on the hours card already answers "how much did each of them do".

---

## This period / all time

- Two options; **This period** is the default — it answers the question a manager has most often.
- **This period** means this bench's **own** subscription period (SD-3459), not the calendar month.
- **All time** spans the whole engagement, including closed periods.
- Selected state distinguished by **weight and fill**, never colour alone (§G6), and the control **must not change width between states** — reserve the bold width or set `min-width` from the selected state.
- Changing scope **resets the reveal to the first batch**.
- The section states the **scope's own total**, not the number currently revealed.

---

## Batching

- Reveal **10**, then **See more (n remaining)** loading 10 more.
- `MANAGER_WORK_LOGS_BATCH = 10` — a **named constant** on client and server, never a literal in a slice expression. It will be tuned.
- The button **states the remaining count, not the total**, and is **absent** when nothing remains or the scope holds ≤10.
- Resets to the first 10 on scope change, and when a **new entry is approved** — the newest belongs in the first batch, not an unrevealed one.
- **No collapse control.** Collapsing would discard what the manager was reading.

---

## States (§G2)

| State | Behaviour |
|---|---|
| Loading | Skeleton rows at the natural row height. No spinner that collapses the card |
| Empty — this period | `No hours logged against this bench in this period.` |
| Empty — all time | `No hours have been logged against this bench yet.` |
| Error | Inline with a retry **inside** the card |

Empty copy `var(--gray-700)`, 13px, body font, `28px 30px` padding. The card holds a **minimum height** so switching scope never reflows the page. **The sorter stays usable in every state** — an empty period is not an error, and switching to All time is exactly what a manager does next.

---

## Reference

```
BE.md                          endpoint, filtering, batching
../SD-3471/                    the page shell this mounts into
../SD-3472/                    the hours card this must reconcile with
../../internal-dashboard/SD-3467-engagements-work-log-entry/   the full entry component
../../ENGINEERING-BRIEF.md     §G1 pagination, §G2 states, §G6 filters, BE-29 the period close
../../../prototype/index.html  → Manager → Virtual Benches → open a bench → Engineer Work Logs
```
