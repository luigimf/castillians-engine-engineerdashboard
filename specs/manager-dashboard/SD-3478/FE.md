# FE — Organisation Page: Functions

Angular handoff for **SD-3478**. The functions on the `/organisation` page. **SD-3477** establishes the page and who may reach it — **Admin only**. Changing a member's role and removing a member are **SD-3479**.

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
| `ChangeAccountAdminModalComponent` | The ownership transfer, listing the account's Managers |
| `BrandsMembersCardComponent` | Brands from Zoho, their members, and per-brand invite |
| `InviteMemberModalComponent` | Email, and a role **per bench** |
| `AddBrandModalComponent` | The new-brand request |
| `EmailDomainsCardComponent` | One column per brand |

---

## Change Account Admin

- Top-right of the page, aligned with the title (SD-3477).
- Opens a modal listing the organisation's **Managers**. A Viewer cannot become Admin without first becoming a Manager, and that is a role change (SD-3479), not an ownership transfer.
- States plainly what transfers: **subscription ownership, billing, and this page**.
- The outgoing Admin becomes a **Manager**, keeping their bench access.
- **There is exactly one Admin.** The transfer is atomic — never two, never none, even if two are submitted at once.
- Confirmation names **both people**. On success both are notified.
- **The outgoing Admin loses this page immediately** — on their next read the menubar entry and the route are gone (SD-3477).

---

## Brands & Members

One card, titled **Brands & Members**, listing every brand in the channel with its members beneath it.

- **Brands come from Zoho** — the `Parent Brand` lineage (SD-3416, SD-3417). The page renders what Zoho says; it never holds its own list of brands.
- Members are the **manager profiles** associated with each brand in Zoho, so the platform and Zoho cannot disagree about who belongs to whom.
- Brands are ordered as the channel is: **the root first, then its children**.
- Each member row: avatar, name, email, and their **bench access**.

### Bench access carries the role — because the role is per bench

- **Manager and Viewer are held per bench.** A member row shows **one chip per bench, with the role on that bench** — `Core Platform · Manager`, `Payments Squad · Viewer`. A single role tag on the row would be a lie.
- **The chips are labels, not links.** They are not interactive: no hover treatment, no pointer cursor, no click target, and nothing navigates. A chip states the bench and the role held on it — that is its whole job. A link invites an Admin to leave a page they are part-way through administering, and a chip that looks clickable but leads somewhere unexpected is worse than one that plainly does not.
- The **Admin's row** shows a single **`Admin · All Virtual Benches`** chip instead. Admin is the one account-wide role; it is not held per bench and carries no per-bench chips.
- A member with no benches yet renders **`No bench access yet`** as a single chip — never an empty row.
- **Bench access is synced with the bench pages** (SD-3471): granting or removing access there changes these chips on the next read, and vice versa. One membership record, two surfaces.

### Empty brands

- A brand with **no members yet** renders **`No members added yet.`** plus its own **Invite members** action — **never omitted**. An absent brand looks like it was never set up.
- Empty copy `var(--gray-700)`, 13px, body font.

---

## Inviting members

- **Invite members** per brand, so an invitation is always scoped to the brand it belongs to.
- Modal title **Invite a member**; the body names the brand and states they will be asked to create an account with the invited address.
- Collects the **email**, and **bench access with a role chosen per bench** — the same model the member rows display. Toggle a bench on, then pick **Manager** or **Viewer** for it.
- **Admin is never invitable.** Ownership moves through Change Account Admin.
- The address must be on **that brand's own** email domains. A mismatch is refused with a plain explanation, **server-side**.
- An invitee appears in the list as **Invited** until they complete onboarding, so it is clear they cannot see anything yet.
- The invitation email's CTA lands on **`https://castillians.com/manager-sign-up?invite=TOKEN`** and carries the per-bench grants — the full journey is specified in **SD-3471**.
- **The Work Email field there is pre-populated with the invited address and locked**, resolved server-side from the token. The address is the identity the invitation was issued against; it is never editable, and a mismatched submission is refused server-side.

---

## Add another brand

- A **divider and a primary button** beneath the last brand.
- The modal collects the **brand name**, **whether it is a sub-brand**, and an **optional message**.
- The sub-brand question is asked in plain words — *"Is this a sub-brand of one of your brands?"* — with answers naming each existing brand, or **"No — it sits above all my other brands"**. A client should not need to know what a parent record is to answer.
- **A brand always belongs to the channel.** There is no "stands on its own" option — an unparented brand would be a separate account.
- **This is a request, not a provisioning action.** Our team creates the Zoho record and sets its Parent field; nothing exists until they do.
- A **pending request stays on the page** as an **Awaiting setup** row stating what was asked and when, so an Admin does not ask twice. It **cannot be opened** — there is no brand behind it.

---

## Email domains, per brand

- One card, **Email Domains**, explaining that only addresses on these domains can sign up.
- **One column per brand**, derived from the **Zoho client records** — not a configured list. A brand parented into the lineage in Zoho arrives with its own column, with nothing to set up.
- A brand with no domains yet renders an **empty input**, ready to fill — not omitted.
- Domains are added and removed per brand, one `@`-prefixed input per domain.
- **Domains are never pooled across the channel.** An address on one brand's domain does not admit anyone to another brand's benches.
- Removing the **last** domain for a brand is **refused** — nobody could then be invited to it. The delete control states why it cannot be used.

---

## States (§G2)

| State | Behaviour |
|---|---|
| Loading | Skeleton rows at each card's natural height |
| Empty — brands | **Cannot occur**; an account always has at least one brand |
| Empty — members of a brand | The sentence plus **Invite members** |
| Empty — domains of a brand | An empty input, ready to fill |
| Error | Inline, with a retry **inside** the affected card |

**No pagination on this page** (SD-3477). Every brand, member and domain renders.

---

## Roles — the platform rule

**Admin is account-wide; Manager and Viewer are per bench.** The same person can manage one bench and only view another, and the page must show that rather than flatten it.

Every action here is **Admin only**, and refused **server-side** for anyone else — not merely hidden. Role is resolved server-side on every response.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| Brands in the channel | **Zoho** `Parent Brand` lineage (SD-3416, SD-3417) | Internal Channel page (SD-3462), Email Domains columns |
| Member ↔ brand association | **Zoho** manager profiles on the client record | This page; written at onboarding (SD-3471) |
| Bench access + role per bench | Bench membership (SD-3471) | The bench's Members with access card, the member's own bench list (SD-3470) |
| Who the Admin is | The account record (§A4) | This page's visibility (SD-3477), subscription ownership |
| Email domains | Zoho client record, **per brand** (§A4, INT-10, BE-23) | Every invitation on the platform |
| A new-brand request | Written here, actioned in **Zoho** by our team | The brand's own row once linked |

**Acceptance criteria**

- **Zoho is the source of truth for brands and for who belongs to which brand.** The page holds no parallel list, so re-parenting a brand or moving a contact in Zoho is reflected on the next read with nothing to migrate.
- **Bench access shown here and on the bench page are one record.** Granting access on a bench page adds the chip here; removing it here removes it there. They can never disagree.
- **The role on a chip is the role on that bench** — the same value the bench page's Members with access card shows, read once and rendered twice.
- Transferring the Admin role moves subscription ownership **and** this page together, in one write.
- A brand's email domains govern **only that brand's** invitations.
- **Adding a brand provisions nothing.** No client record, no billing, no benches — until our team creates it in Zoho and parents it.

---

## Email notifications in this flow

For awareness only — templates and copy are specified in the **Email Notifications** epic.

| Trigger | Recipient |
|---|---|
| Account Admin transferred | Both the outgoing and incoming Admin |
| Member invited to a brand | The invitee — links to Manager sign-up |
| New brand requested | `sharedservices@` + `humancapital@` + `customerexperience@`, with the Zoho actions |
| New brand request acknowledged | The requesting Admin |
| Email domain added or removed | — **None.** It is the Admin's own setting |

---

## Reference

```
BE.md                          endpoints, the transfer, invitations, domains, the brand request
../SD-3477/                    the page and its Admin-only access
../SD-3479/                    member roles and removal
../SD-3471/                    bench membership, and the invitation journey these invites use
../SD-3470/                    the invitee's own bench list on completion
../../ENGINEERING-BRIEF.md     §A4 roles + INT-10 domains, BE-23 Zoho fields, §G patterns
../../../prototype/index.html  → Manager → Organisation
```

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.

**Prototype:** open `prototype/index.html` → **Manager** → **Organisation**. **Northmill Markets** is a brand with no members yet, so the empty state and its Invite members action are both visible. **Claire Bonnici** holds two benches at different roles — the case a single role tag would misrepresent.
