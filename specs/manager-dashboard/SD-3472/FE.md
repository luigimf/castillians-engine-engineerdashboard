# FE — V Bench page: Skills Mix, Skills Matrix & Monthly Engineering Hours

Angular handoff for **SD-3472**. Three sections inside the SD-3471 page shell. Replaces the existing sections at `castillians.com/v-benches/{id}`.

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
| `SkillsMixCardComponent` | Core and additional skill chips. **Read-only for every role** |
| `SkillsMatrixCardComponent` | Engineer × skill, with the see more / see less reveal |
| `MonthlyHoursCardComponent` | Period tag, days chip, the five figures, the bar, per-engineer usage |
| `CapacityRequestModalComponent` | Hours needed, always-visible authorised cost |

The capacity bar is the shared `CapacityBarComponent` from SD-3470 — same band logic, same geometry.

---

## Skills Mix

- **Core skills** and **Additional skills** as two labelled chip rows. Labels Bricolage **12px/700**, uppercase, 0.05em tracking, ink.
- Chips: white, `1px var(--gray-150)`, `var(--radius-md)`, 10px padding, body 11px.
- **No edit affordance at all, for any role.** Not a disabled button, not a pencil icon: nothing. The skills a bench covers are what the client is charged for, so they change through the subscription (INT-5) — a short line says so and points at **Need help?**.
- Core and Additional stay **visually distinct**: a bench sold on three core skills and two nice-to-haves is not the same as one sold on five.
- No additional skills → **omit the row**, don't render it empty.

---

## Skills Matrix

**Read the prototype before building this card.** It is *not* an engineer × skill grid — it is an **aggregated skills cloud**, subtitled _"Aggregated skills across engineers in this Virtual Bench."_

- It shows the **union of the skills the bench's activated engineers actually hold** — which legitimately includes skills that are **not** in the bench's Core/Additional mix. An engineer who also knows Go contributes Go.
- Each skill is tagged by **coverage strength**, colour-coded, from most to least widely held across the bench:

| Tag | Meaning | Foreground / fill / border |
|---|---|---|
| **Popular** | Held by 3 or more engineers | `#CB4641` / `#FFF1F1` / `#FFD0CF` |
| **Common** | Held by 2 | `#DD6B1D` / `#FFF7EC` / `#FFD6A3` |
| **Unique** | Held by 1 | `#C5A007` / `#FFFCEE` / `#FFE57B` |

- Ordered by **coverage descending**, so the bench's deepest strengths read first and single-point dependencies fall to the end. That ordering is the card's real value: a **Unique** skill is a person-shaped risk.
- An **info tooltip** explains the colour coding. It is a button, reachable on touch.
- **Read-only for every role** — derived from the engineers' own vetted profiles, not authored against the bench.
- **See more / see less:**
  - A first tranche on load; the rest on demand, so a bench with many skills does not push the hours card off screen.
  - The control **states how many more there are** — not a bare "See more".
  - **See less** returns to the first tranche and the card to its original height, **without scrolling the page** or shifting the sections beneath it.
  - Reveal via `grid-template-rows` (`0fr` → `1fr`) over 300ms so the card grows rather than clipping; chevron rotates 180°.
  - **Absent when everything already fits** — an affordance that reveals nothing must not be shown.

**Do not add an uncovered-skill state to this card.** A gap between the agreed mix and what the bench can actually do is a real question, but it belongs to the Skills Mix card or to a conversation with the Castillians team — not invented here.

## Monthly Engineering Hours

**Header of the card**

- **Current period** tag: white fill, `1px var(--gray-150)`, uppercase label paired with the period as a **date range** — never a bare month name.
- **Days remaining** chip beside it: 16+ green (`#E7F7F1` / `#B9E6D4` / `#0A5C43`), 6–15 amber (`#FEF6E7` / `#F0D9A8` / `#7A5A12`), 0–5 red (`#FDECEC` / `#F3C9C9` / `#8C1F1F`).
- Both resolve from **this bench's own** period — two benches in one organisation may show different periods on the same screen.

**The five figures** — label Bricolage 12px/700 uppercase, value Bricolage **34px/700**:

| Label, as it renders | Meaning |
|---|---|
| `CAPACITY USED` | Approved hours logged this period |
| `CAPACITY PLAN` | The subscribed monthly hours |
| `OVERAGES AGREED` | Additional hours the client has authorised this period |
| `OVERAGES USED` | Hours logged beyond the plan |
| `OVERAGE RATE (CUR)` | The **client-facing** rate those hours bill at |

**Use these labels verbatim.** The currency **code sits in the Overage Rate label** — `OVERAGE RATE (EUR)` — with a bare value beneath, per the platform's money rule.

- Hour values are **bare integers** — no `h` suffix.
- `OVERAGES AGREED`, `OVERAGES USED` and `OVERAGE RATE` each carry an **info tooltip**, one sentence each. The tooltip is a **button**, not hover-only — it must be reachable on touch.
- **Overage Rate is the blended client rate**, at **1.25× for the first three months** (BE-06). The engineer-facing rate and mark-up never appear on this page.
- Bar beneath: 233px wide, 8px track, 4px radius, percentage at Bricolage 22px/700 beside it, and a **plain-English sentence** under it describing where the bench stands.
- **Pending-approval hours are excluded from CAPACITY USED**, the bar and the percentage — a manager must never plan against hours that may yet be declined.
- They are, however, **named**: where a bench or an engineer has hours awaiting a decision, a note states how many — _"8h awaiting approval from the Castillians team"_. Excluding them from the figures while saying nothing would leave a manager wondering why the numbers do not match what their engineers told them. **Excluded from the arithmetic, disclosed in words.**
- **Unlimited overage** → the overage figures read as uncapped rather than showing a misleading ceiling.

---

## Request more capacity this month

Button beside the bar. **Absent for a Viewer.**

- The modal states the bench's position — plan, used, remaining — **before** asking for anything.
- Hours needed, plus an optional reason. **(optional)** in the **same ink** as the field title.
- **Total authorised cost** beside the hours field: requested hours × this bench's **Overage Rate**.
  - **Same outline treatment** as the field beside it, and the same text properties as a standard field input — body 15px/500 — so the pair reads as one row.
  - **Always visible**, holding a muted placeholder before any figure is entered. A field that appears and disappears makes the row jump.
- **Current period only** — that is what the button says and what the modal must honour. A request does not roll forward.
- **A request, not a purchase.** Nothing is authorised or billed until the team actions it; the success state does not show the hours as available.
- A **pending request stays visible on the card**, stating what was asked and when — so a manager under pressure does not ask three times.
- Once granted, the hours appear in **OVERAGES AGREED** and the ceiling moves. **The grant expires at period end.**

---

## Bench usage per engineer

- One row per **activated** engineer: avatar, name, hours logged against this bench **this period**.
- **The Total row must equal Capacity Used exactly.** If the rows and the total can disagree, the card is wrong.
- Ordered by hours **descending** — the heaviest contributor reads first.
- **An engineer with no hours is still listed, reading zero.** Their absence would look like they had left the bench.
- Approved hours only.
- **No rates, no earnings, per engineer or in total.** A manager sees hours; what an individual is paid never appears.

---

## States (§G2)

| State | Behaviour |
|---|---|
| Loading | Skeleton rows at each card's natural height. No spinner that collapses a card |
| Empty — no hours | Figures read zero; the usage list says nothing has been logged this period. The card keeps its height |
| Empty — no engineers | The usage list says the bench has no activated engineers yet |
| Error | Inline with a retry **inside** the affected card |

## Roles — the platform rule

**A Viewer's controls are absent, not disabled.** A greyed-out button invites a support ticket asking why it is greyed out. Render the control or do not render it.

Role is resolved **server-side** on every response. The client renders what it is given; it never decides what a role may do.

---

## Reference

```
BE.md                          endpoints, figures, the capacity request
../SD-3471/                    the page shell these sections mount into
../SD-3473/                    Engineer Work Logs — must reconcile with Capacity Used
../../ENGINEERING-BRIEF.md     BE-03 pro-rating, BE-05/06 overage + rate, BE-08 mark-up, BE-11 to BE-13 thresholds
../../../prototype/index.html  → Manager → Virtual Benches → open a bench
```

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.
