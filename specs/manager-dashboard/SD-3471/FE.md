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
| Invite a new person | **Admin and Manager** |
| Remove access | **Admin only**, with a confirmation naming the person and the bench |

- **A Manager may invite a colleague as a Manager or a Viewer** — the two roles that can be granted on a bench. Neither an Admin nor a Manager can create another **Admin** this way: account ownership moves through Change Account Admin, not an invitation.
- **Removing access stays Admin-only.** Granting a colleague sight of a bench is routine; taking it away affects someone else's work, and one role should own that.
- An invite checks the organisation's **email domain allow-list**; a mismatch is refused with a plain explanation.
- An invited person renders as **pending** until they accept, so it is clear they cannot see the bench yet.
- Removing bench access does **not** remove them from the organisation.

### Two paths: a new person, or someone already here

Adding somebody to a bench has **two outcomes**, and they must not share a template.

| The person | What happens | What they are sent |
|---|---|---|
| **Has no account** | An invitation is created; nothing exists until they onboard | The bench invite — sign-up link, token, Work Email locked |
| **Already a member of the organisation** | **Access is granted immediately.** No invitation, no token, nothing to accept | `emails/11-bench-access-granted.html` — links straight to the bench |

- **An existing member is never sent to sign-up.** They have an account; a sign-up page is a dead end that reads as though we have lost them.
- The grant is **live before the email lands** — the email announces it rather than asking for anything.
- The email names the role they hold **on that bench**, and says their role on every other bench is unchanged. **Manager and Viewer are per bench** (SD-3479), so a person made a Viewer here may still be a Manager elsewhere, and a mail that implies otherwise will be read as a demotion.
- The CTA links to the bench; signed out, they hit `castillians.com/manager-login` and are redirected to it (BE-30).

### What an invitation actually does

An invite is the **start of an account**, not a permission grant on an account that already exists. The journey is:

1. The invitee receives an email whose CTA lands on **`https://castillians.com/manager-sign-up?invite=TOKEN`** — the existing Manager sign-up page, pre-bound to this organisation and to the role and bench access the Admin gave them.
   - **The Work Email field is pre-populated with the invited address and locked.** Read-only and visibly fixed, with a line saying why. The address is the identity the invitation was issued against, so it must not be typed over.
   - The value is resolved **server-side from the token**, never from an editable query parameter, and a submission whose email does not match the token's is refused even if the client were bypassed.
2. They complete **Manager onboarding** — the platform's existing sign-up and onboarding flow, unchanged by this story. Both roles go through the same onboarding; the role decides what they can do afterwards, not which flow they take.
3. On finishing, they land on **their own Manager dashboard**, showing **exactly the benches the Admin gave them access to** — no more.

**Acceptance criteria**
- The invite email's call to action goes to the **Manager sign-up page**, never to a bare login. An invitee has no account yet, and a login screen is a dead end.
- The link **carries the invitation**, so the role and bench access the inviter chose are applied on completion. The invitee never chooses their own role or picks benches.
- An invitation can grant **Manager** or **Viewer**. **Admin is never invitable.**
- They must register with the **exact address the invitation was sent to** — the email says so plainly (BE-21). Signing up with another address is a different person.
- Until onboarding completes they remain **pending** in the members list, and they can see nothing.
- On completion their dashboard shows **only their granted benches** (SD-3470) — every bench an **Admin or a Manager** has given them access to, in one list, however many people did the granting. A Viewer lands read-only, with no request controls anywhere; a Manager lands with the full request set.
- Access granted or removed **after** they accept takes effect on their next read — the same membership record, no re-invitation, whoever grants it.
- An expired or already-used link explains itself and offers a route to ask again — never a generic error.
- **On completing onboarding, a manager profile is created on the Zoho client record** — the same **brand** the bench belongs to, as named on this page. A manager of a Northmill Insurance bench is a contact on **Northmill Insurance**, not on the root client.

### A new brand brings its own Email Domains column

When our team sets a brand up in Zoho and parents it into this lineage, the **Email Domains** section gains a **column of its own for that brand** — the section renders one column per brand in the channel, derived from the client records, not from a fixed list.

**Acceptance criteria**
- The column appears as soon as the Zoho record exists and its Parent places it in the channel. No separate configuration step, and nothing to add by hand.
- A brand with no domains yet renders its column with an **empty input**, ready to fill — not omitted. An absent column looks like the brand was not set up.
- Domains are held **per brand**, never pooled across the channel: an address on one brand's domain does not grant access to another brand's benches.
- Removing a brand from the lineage in Zoho removes its column on the next read, and its domains stop admitting anyone.

### Bench membership also confers Performance Log eligibility

**A Manager with access to a bench is eligible to act as the reviewing manager for the engineers on it** — approving and generating their reviews on the Performance Log.

> ### A bench has exactly one Admin
>
> **There is one Admin per client**, and **a bench is never shared across clients** — it belongs to exactly one. Therefore **a bench has exactly one Admin: its own client's.**
>
> Another client's Admin in the same channel has **no access to this bench** and **must not appear** in its members list. Two ADMIN rows on one bench is a defect, not a valid state — assert it.
>
> The members list is scoped to the bench's **own client** before any access filter is applied. A channel-wide roster is the wrong starting point.

- **The Admin is eligible on every bench belonging to their client, and appears in the reviewer list alongside the Managers.** They hold no per-bench role (SD-3479) — their eligibility comes from the account role, so it needs no grant and cannot be removed by revoking bench access. On this card the Admin's row shows an **ADMIN** tag rather than a role control, and they are still an eligible reviewer for that bench's engineers.
- Because a bench has exactly one Admin, **it always has exactly one always-eligible reviewer** — no more, and never none.
- That is what keeps the last-reviewer rule satisfiable: a bench whose only Manager is removed still has the Admin, so **no engineer is ever left without an eligible reviewer**.
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
