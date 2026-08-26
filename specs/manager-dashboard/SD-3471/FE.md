# FE — V Bench page: Bench Setup & Sharing

Angular handoff for **SD-3471**. The bench page shell, plus setup, Need help?, change capacity plan, and members with access. Replaces `castillians.com/v-benches/{id}`.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — compose from its components; no new one-offs.

---

## Components

| Component | Responsibility |
|---|---|
| `VBenchPageComponent` | The shell: header, actions, and the section frame the other stories mount into |
| `BenchSetupCardComponent` | Name (editable in place), type, capacity plan with its period |
| `NeedHelpModalComponent` | One message to a human, with bench context carried invisibly |
| `CapacityPlanRequestModalComponent` | Current plan, requested hours, cost implication |
| `BenchMembersCardComponent` | Members with access; add existing, invite new, remove |

The three sections beneath — Skills & Hours (SD-3472), Engineer Work Logs (SD-3473), Engineers (SD-3474) — mount into this shell.

---

## Header

- Back link with left arrow, body 16px/500: **Back to Virtual Benches**.
- Bench name as `h1`, Bricolage **42px/700**.
- Beneath: client name and activated engineer count, body 14px `var(--gray-700)`.
- Top-right on the title's line: over-capacity tag where it applies, **Need help?**, **Change capacity plan**. Both 35px. The capacity action is **absent for a Viewer**.
- Below 1024px the actions wrap **beneath** the title rather than overflowing.

---

## Bench setup

- Name, type, and the current capacity plan **with its period as a date range** — so it is unambiguous which period the figure applies to.
- **There is no time zone field.** Removed from the page, the response and the record.
- Fields a manager cannot change render as **read-only text, not disabled inputs**. A disabled input looks like something is broken.

### Naming UX

- **The manager's own words, saved verbatim** — never re-cased, rewritten, or suffixed with the client name.
- Edit in place: click, type, save. **Escape cancels** and restores the previous value.
- Mandatory; empty or all-whitespace is refused and the previous name kept.
- Unique within the organisation; a duplicate names the existing bench.
- **Immediate, no approval** — it is the client's own label for their own team.
- The new name appears on the list page, Internal and the Channel page on the next read. **Work logs, engineers, capacity and history are untouched.**

---

## Need help?

- Available to **all three roles** — a Viewer may still have a question.
- Subject/topic, and a **mandatory** message.
- **Carries the bench context invisibly** — bench, client, who is asking — so nobody explains which bench they mean.
- Empty message → inline error beneath the field.
- Success: modal closes, toast names who will reply.
- **Not a ticketing system.** No categories, no priority picker, no SLA promises.

---

## Change capacity plan

**Absent for a Viewer.**

- States the **current plan** and **current period** first — the manager is changing something they can see.
- Collects new monthly hours, from when, and an optional note. **(optional)** in the **same ink** as the field title.
- **Cost implication** beside the hours field at the bench's **client-facing** rate, **always visible** — a muted placeholder before any figure is entered, so the row never jumps.
- **It is a request.** The copy says the team will confirm the plan and when it takes effect. The success state does **not** show the new plan as live.
- A **pending request stays visible on the page**, stating what was asked and when; the action reads as pending rather than inviting a duplicate.

---

## Members with access

- Avatar, name, email, **role tag** (Admin / Manager / Viewer) per row.
- Roles are the organisation's, shown here for context. **Changing a role is an organisation-level act** — not on this page.
- **A bench always keeps at least one Admin or Manager.** Removing the last is refused with an explanation — a bench nobody can act on is a dead end.

| Action | Who |
|---|---|
| Add an existing member | Admin, Manager |
| Invite a new person | **Admin only** |
| Remove access | **Admin only**, with a confirmation naming the person and the bench |

- **Only an Admin brings someone new into the organisation.** A Manager can grant access to an existing colleague, but creating an account has billing and confidentiality consequences.
- An invite checks the organisation's **email domain allow-list**; a mismatch is refused with a plain explanation.
- An invited person renders as **pending** until they accept, so it is clear they cannot see the bench yet.
- Removing bench access does **not** remove them from the organisation.

### Bench membership also confers Performance Log eligibility

**A Manager with access to a bench is eligible to act as the reviewing manager for the engineers on it** — approving and generating their reviews on the Performance Log.

- Eligibility follows **bench membership**, so granting or removing access here changes who can review that bench's engineers. Say so on this card: it is a consequence a manager should not discover later.
- A **Viewer** is never eligible, however many benches they can see.
- **A bench must always retain at least one eligible reviewer** — the same rule that keeps at least one Admin or Manager with access, read for a second purpose. An engineer with nobody able to review them has no route to a performance record.
- The Performance Log itself is **out of scope for this story**; this establishes only who is eligible.

## Roles — the platform rule

**A Viewer's controls are absent, not disabled.** A greyed-out button invites a support ticket asking why it is greyed out. Render the control or do not render it.

Role is resolved **server-side** on every response. The client renders what it is given; it never decides what a role may do.

---

## Reference

```
BE.md                          endpoints, rename, requests, membership
../SD-3470/                    the list page this is reached from
../SD-3472/ ../SD-3473/ ../SD-3474/   the sections inside this shell
../../ENGINEERING-BRIEF.md     §G patterns, §A4 roles + INT-10 domains, BE-02/03 periods, BE-08 rates
../../../prototype/index.html  → Manager → Virtual Benches → open a bench
```
