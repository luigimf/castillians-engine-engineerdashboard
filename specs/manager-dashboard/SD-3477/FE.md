# FE — Organisation Page: Access

Angular handoff for **SD-3477**. The `/organisation` route, its menubar entry, and who may reach it. The functions inside the page are **SD-3478** and **SD-3479**.

Replaces `castillians.com/brand-profile`, which is **unpublished** as part of this work — not left running alongside.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is** — this story adds one entry to it, nothing more.

**Design system:** V2 Castillians Design System — compose from its components; no new one-offs.

**Design:** the menubar follows [Figma node 11613-8085](https://www.figma.com/design/A4ZWwuOmoeUr7p72TYIiLV/castillians.com?node-id=11613-8085&t=LFNSp1kI4NUE0PqG-11). Selected and unselected treatments match the existing sub-tab row **exactly** — this is not a new navigation pattern.

---

## Components

| Component | Responsibility |
|---|---|
| `OrganisationPageComponent` | The route, the title row, and the single-column card frame the functions mount into |
| Menubar (existing) | Gains one **Organisation** entry, rendered only when the server says so |

The cards beneath — Brands & Members, Email Domains (SD-3478) and the role/removal controls (SD-3479) — mount into this frame.

---

## The route

- New route **`/organisation`** on the Manager dashboard. The existing shell, header and navigation are otherwise unchanged.
- **`/brand-profile` is unpublished** — the route is removed, not left pointing at a dead end.
- An old `/brand-profile` link resolves by role: to **`/organisation`** for someone who may see it, and to their **Virtual Benches** for someone who may not. Never a blank page, and never a page that renders then empties.

---

## Who may reach it — Admin only

|  | Admin | Manager | Viewer |
|---|---|---|---|
| Sees the **Organisation** entry in the menubar | **Yes** | **No — absent** | **No — absent** |
| Can open `/organisation` | **Yes** | No | No |

- For a Manager or Viewer the menubar entry is **absent, not disabled**. A greyed-out tab invites a support ticket asking why.
- Navigating to `/organisation` directly as a Manager or Viewer is **refused server-side**, and they are returned to their Virtual Benches with a plain explanation.
- **There is one Admin per account.** The page is therefore visible to exactly one person — which is why every function on it is consequential (SD-3478, SD-3479).
- **The page's existence is not announced** to Managers or Viewers: no entry, no route, no mention in copy anywhere else on the dashboard.

---

## Page frame

- Page title **Organisation** as `h1`, Bricolage **42px weight 700** — matching the Virtual Benches page.
- **Change Account Admin** sits top-right, aligned with the title, as a 45px outline button. Its behaviour is **SD-3478**.
- The page is a **single column of cards**, `24px` gap. Each function in SD-3478 is a card within it.
- **No pagination anywhere on this page.** Every brand, every member and every email domain renders in full — an Admin managing access needs the whole picture in one view, and the lists are bounded by the size of the client's own organisation.
- Every transition uses `300ms cubic-bezier(0.35,0,0.25,1)`.
- Responsive across desktop, tablet and mobile; Chrome, Firefox and Safari.

---

## Roles — the platform rule

**Admin is the only account-wide role.** Manager and Viewer are held **per bench**: the same person can be a Manager of one bench and a Viewer of another. That model drives the cards inside the page (SD-3478, SD-3479); for this story it matters only that **neither a Manager nor a Viewer reaches the page at all**, on any bench.

Role is resolved **server-side** on every response. The client renders what it is given; it never decides who may see this page.

---

## States (§G2)

| State | Behaviour |
|---|---|
| Loading | Skeleton cards at natural height. No spinner that collapses the page |
| Error | Inline, with a retry **inside** the page body — never a toast alone |
| Empty | **Cannot occur.** An account always has at least its Admin and at least one brand |

Empty and helper copy `var(--gray-700)`, 13px, body font.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| Who the Admin is | The account record (§A4) | This page's visibility, Change Account Admin (SD-3478), every ownership notification |
| Menubar entries | The signed-in member's role, resolved server-side | Virtual Benches, Subscriptions |
| Brands in the channel | Zoho `Parent Brand` lineage (SD-3416, SD-3417) | Internal Channel page (SD-3462), the brands list in SD-3478 |

**Acceptance criteria**

- Role is resolved **server-side** on every response; the client never decides who may see this page.
- **Transferring the Admin role moves the page with it** — the outgoing Admin loses the menubar entry and the route on their next read, and the incoming one gains both. One account record, read by both.
- **Unpublishing `/brand-profile` removes nothing else.** Brands, members, domains and benches are untouched; only the route is withdrawn.

---

## Reference

```
BE.md                          the route guard, the redirect, the role resolution
../SD-3478/                    the functions on this page
../SD-3479/                    member roles and removal
../SD-3471/                    bench membership, and the invitation journey the invites use
../../ENGINEERING-BRIEF.md     §A4 roles, §G interaction patterns and §G2 states
../../../prototype/index.html  → Manager → Organisation
```

**Figma:** [menubar, node 11613-8085](https://www.figma.com/design/A4ZWwuOmoeUr7p72TYIiLV/castillians.com?node-id=11613-8085&t=LFNSp1kI4NUE0PqG-11).

**Prototype:** open `prototype/index.html` → **Manager** → **Organisation**. Use the role switcher: the entry is present for **Admin** and absent for **Manager** and **Viewer**.
