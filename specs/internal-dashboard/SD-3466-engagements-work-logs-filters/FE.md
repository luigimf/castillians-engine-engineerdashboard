# FE — Engagements: Work Logs tab, search & filters

Angular handoff for **SD-3466**. The filter row, result set and batching on Internal → Engagements → **Work Logs**.

> ### ⚠ The prototype's menubar is not part of this work
>
> The dark **Engineer / Internal / Manager** toggle in the prototype's header exists **only** so one file can demonstrate all three dashboards. It is a prototype navigation device and **must not be built**.
>
> **Keep the platform's existing menubar as it is.**

**Design system:** V2 Castillians Design System — `Card`, `Input`, `Select`, `Button`, `StatusBadge`.
The entry itself, its history accordion and the per-engineer page are **SD-3467**.

---

## Components

| Component | Responsibility |
|---|---|
| `WorkLogsTabComponent` | Owns filter state, calls the service, holds the reveal count |
| `WorkLogFiltersComponent` | The four controls; emits a single `WorkLogFilters` object |
| `ApprovalsToggleComponent` | The approvals chip — dot, label, count |
| `WorkLogListComponent` | Rows, "N entries" summary, See more, the four states |

Entry rendering is `WorkLogEntryComponent` (SD-3467) — this story renders the list around it.

---

## Filter row

One row, four controls, left to right, all **50px** fixed height. Search takes the remaining width; the other three size to content and do not wrap above 1024px.

| # | Control | Behaviour |
|---|---|---|
| 1 | Search | Placeholder _"Search engineer by name or email"_. Matches engineer **name and email only** |
| 2 | Approval required only (n) | Toggle chip, not a select |
| 3 | Period | Today · Last 3 days · This week · This month · All time. Default **This month** |
| 4 | Engagement scope | Ongoing · Past · All engagements. Default **Ongoing** |

- Search debounced at **300ms**; a superseded request is cancelled (`switchMap`, not `mergeMap`).
- The four filters **compose** — AND, always.
- Any change resets the reveal to the **first batch of 10**.
- Filter state is client-side, passed as query parameters. It is **not** persisted across navigation — returning restores the defaults.

### Approvals toggle

| State | Fill | Border | Label | Dot |
|---|---|---|---|---|
| Off | white | `2px solid var(--border)` | weight 500 | `--gray-300` |
| On | `var(--warning-50)` | `2px solid var(--warning-border)` | weight 600 | `--warning-border` |

Dot 8px, circular, leading the label with an 8px gap. Label body font 15px. Padding `0 14px`, radius `var(--radius-md)`, height **50px**.

- Distinguished by **weight and fill**, never colour alone (§G6).
- **The chip must not change width between states** — a reflowing filter row is the defect to avoid. Reserve the bold width, or set `min-width` from the on-state.

---

## Counts in brackets

Every count is a **grand total for its own scope**, computed server-side and returned with the response. Never derive a count from the rows on screen — a number that moves as the operator types is unreadable.

| Count | Basis |
|---|---|
| Period options | Every entry in that period. Ignores search, approvals, scope |
| Scope options | Every entry on an engagement in that state. Ignores search, approvals, period |
| Approval required only (n) | Awaiting a decision **within the selected period**. Ignores search and scope |
| Work Logs (n) tab counter | Distinct engineer × bench engagements, network-wide. Unaffected by this row (SD-3464) |

The approvals count **excludes entries on ended engagements** — they are never in the queue.

---

## Ongoing and past

- An **engagement** is one engineer on one Virtual Bench. It is **past** when that bench's subscription has expired.
- Derived at read time from the engagement record — **never stored on the entry**.
- Past never appears under Ongoing, and vice versa. All engagements shows both.
- **Ongoing is the default**; past engagements are looked up deliberately.
- The per-entry **"Engagement ended 15 Jul"** label is SD-3467.

---

## Batching

- Reveal **10**, then a **See more (n remaining)** button loading 10 more.
- `WORK_LOGS_BATCH = 10` — a **named constant** on client and server, never a literal in a slice expression. It will be tuned.
- The button **states the remaining count, not the total**, and is absent when nothing remains or the total is ≤ 10.
- Reset to the first 10 on any filter change, any search change, and when a **new entry is created anywhere on the platform**.
- A **"N entries"** summary reflects the **filtered** total.
- Sort **newest first**, tie-broken on entry id, **stable across batches** — two entries on one date must never duplicate or drop at a boundary.
- §G1 of the brief and the prototype both carry this behaviour; the earlier 8-per-page design is superseded everywhere.

---

## States (§G2)

| State | Behaviour |
|---|---|
| Loading | Skeleton rows at the card's natural height. No spinner that collapses the card |
| Empty — search | `No work logs match "<term>".` — the operator's own term, quoted |
| Empty — past scope | `No past engagements in this period.` |
| Empty — otherwise | `No work logs in this period.` |
| Error | Inline, with a retry **inside** the card — never a toast alone |

Empty copy `var(--gray-700)`, 13px, body font, `28px 30px` padding. The card holds a minimum height so switching states never reflows the page. **The filter row stays usable in every state.** An empty result is not an error.

---

## General

- Below 1024px the controls stack rather than overflow, keeping their 50px height.
- Chrome, Firefox, Safari.
- Existing design-system components; no new one-offs.
- Every transition `300ms cubic-bezier(0.35,0,0.25,1)`.

---

## Reference

```
BE.md                          endpoints, filter semantics, counts, batching
../SD-3467-engagements-work-log-entry/   the entry, history and engineer page
../../ENGINEERING-BRIEF.md     §G1 pagination, §G2 states, §G6 filters and search
../../../prototype/index.html  → Internal → Engagements → Work Logs
```
