# FE — Organisation Page: Member Roles & Removal

Angular handoff for **SD-3479**. Two functions on the `/organisation` page, split out from **SD-3478** because they share one guard, one confirmation pattern and one write-back to Zoho. Both are **Admin only** — SD-3477 makes the page Admin-only in the first place.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — compose from its components; no new one-offs.

---

> ### ⚠ Roles are per bench — read this before the criteria
>
> **Admin is the only account-wide role.** Manager and Viewer are held **per bench**: the same person can be a **Manager of bench A and a Viewer of bench B**, and both are true at once.
>
> A role control therefore belongs **on a bench**, not on a member. There is no single "this person's role" to set.
>
> This matches SD-3478's member rows, the prototype's invite modal, and the bench pages' Members with access card (SD-3471). **SD-3479's Jira description was corrected on 26 Aug** — the earlier account-wide wording ("Role is the organisation's role. It is not set per bench") is retained there under an *Outdated on 26 Aug* note, with this model beneath it.

---

## Components

| Component | Responsibility |
|---|---|
| `MemberBenchRoleControlComponent` | The role control on a single bench chip — Manager or Viewer |
| `RemoveMemberModalComponent` | The two removal scopes and their confirmations |

---

## Changing a member's role on a bench

- **Each bench chip on a member's row carries its own role control** — Manager or Viewer for that bench.
- Changing it is **immediate**, with a success toast naming the person, **the bench**, and their new role. No approval step: it is the Admin's own organisation.
- **Admin is not an option in any control.** Ownership moves through **Change Account Admin** (SD-3478), never a role control — otherwise an account could end with two Admins or none.
- The **Admin's own row shows `Admin · All Virtual Benches` as read-only text**, not a disabled control. A disabled dropdown looks broken.
- **Demoting the last Manager on a bench is refused**, with a message that **names the bench**: a bench must keep at least one person who can act on it.
- A member **promoted to Manager on a bench** gains the request controls **on that bench** — no re-invitation, no new grant, and no change to their role on any other bench.
- A member **demoted to Viewer on a bench** loses that bench's request controls on their next read, and stops being eligible to review **that bench's** engineers on the Performance Log (SD-3471). Their other benches are untouched.

---

## Removing a member

Removal has **two scopes**, and the difference matters enough to ask:

| Scope | What happens |
|---|---|
| **From a bench** | They lose that bench and its notifications. They keep their account, their **other benches at their existing roles**, and their place in the organisation |
| **From the organisation** | They lose the account entirely — every bench, and their ability to sign in |

- The confirmation **names the person and states which scope is being applied**, in words — never a bare "Are you sure?".
- Removing from a bench **names the bench**, and says which benches they keep.
- **Removing from the organisation states its consequences plainly**: every bench, and their sign-in. It is the more serious act and **must not be one click away from the lesser one**.
- **The Admin cannot remove themselves.** They must transfer ownership first — the control is **absent** on their own row, and the endpoint refuses it.
- **A bench must keep at least one Admin or Manager with access.** Removing the last one is refused, naming the bench (SD-3471).
- Removal is **immediate**. There is no pending state, because access is either granted or it is not.
- **A removed person's work is untouched.** Bench notes, requests they raised and history they appear in all stay. Removing a person removes access, not their record.

---

## Invited members

- Someone invited but not yet onboarded shows as **Invited**, and their row makes clear they cannot see anything yet.
- **Their per-bench roles can be changed while pending** — it changes what the invitation will grant when they complete onboarding.
- **Removing a pending invitee cancels the invitation**, and the confirmation says so. Their sign-up link stops working.
- An invitee who never accepts is **not** silently dropped; they stay visible as **Invited** until an Admin removes them.

---

## States (§G2)

| State | Behaviour |
|---|---|
| Error | Inline, **on the affected row** — never a toast alone for a failed removal |
| Refusal | The reason is stated, **naming the bench or the constraint**. Never a generic "cannot do that" |

Error copy `var(--danger-500)`, 12px, body font.

---

## Roles — the platform rule

Every control here is **Admin only** and refused **server-side** for anyone else. **A Viewer's controls are absent, not disabled** — and on this page a Viewer has no page at all (SD-3477).

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| Role **on a bench** | Bench membership (SD-3471) | Role on the chip here, the bench's Members with access card, what that bench offers that member |
| Bench access | Bench membership (SD-3471) | The member's own bench list (SD-3470), notification recipients |
| Member ↔ brand association | **Zoho** manager profile on the client record | This page; written at onboarding (SD-3471) |
| Pending invitations | The invitation record + its token (SD-3471) | The **Invited** state here |
| Performance Log eligibility | Derived from **the role on that bench** + access (SD-3471) | Who may review that bench's engineers |

**Acceptance criteria**

- **Removing a member from the organisation writes back to Zoho** (INT-4) — their manager profile on the client record is updated, so Zoho and the platform do not disagree about who works with us. If the Zoho write fails, **the platform removal still stands** and the failure is **reported, not swallowed**.
- **Removing from a bench does not touch Zoho.** They are still a contact for that brand; they simply cannot see one bench.
- A role change or removal takes effect on **every Manager surface on the next read** — the bench pages' Members with access, the member's own bench list, and the notification recipients. **Recomputed, never patched client-side.**
- **A role change is scoped to one bench.** Assert that changing a member's role on bench A leaves their role on bench B byte-identical.
- **Their logged work and requests are never deleted.** A removed Manager's capacity requests stay in the Internal team's queue and in the bench's order form history (SD-3465).
- **Every guard is enforced server-side** — last Manager on a bench, last Admin, self-removal. The absent control is never the only protection.
- Two Admins acting at once cannot leave a bench with nobody who can act on it, nor the account without an Admin.

---

## Email notifications in this flow

For awareness only — templates and copy are specified in the **Email Notifications** epic.

| Trigger | Recipient | Note |
|---|---|---|
| Role changed on a bench | The member | States what they can now do, and **on which bench** |
| Removed from a bench | The member | Names the bench; makes clear they keep their account and their other benches |
| Removed from the organisation | The member | States that their access has ended |
| Pending invitation cancelled | The invitee | Their sign-up link no longer works |
| Role changed while pending | — | **None.** They have not seen the first invitation's terms yet, so there is nothing to correct |

---

## Reference

```
BE.md                          endpoints, the guards, the Zoho write-back
../SD-3477/                    the page and its Admin-only access
../SD-3478/                    Change Account Admin, the member rows these controls sit on
../SD-3471/                    bench membership, the invitation record, Performance Log eligibility
../SD-3470/                    the member's own bench list, recomputed after a change
../../internal-dashboard/SD-3465-bench-entry/   where a removed Manager's requests remain
../../ENGINEERING-BRIEF.md     §A4 roles, INT-4 Zoho write-back, §G patterns
../../../prototype/index.html  → Manager → Organisation
```

**Prototype:** open `prototype/index.html` → **Manager** → **Organisation**. **Claire Bonnici** is a Manager on Core Platform and a Viewer on Payments Squad — the case a single role control cannot express. The Admin's own row has no role control and no remove control, and the removal confirmation asks which scope applies.
