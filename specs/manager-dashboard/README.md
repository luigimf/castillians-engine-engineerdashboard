# Manager Dashboard — specs

Epic: [SD-3469](https://castille-labs.atlassian.net/browse/SD-3469)

Replaces the existing pages at `castillians.com/v-benches` and `castillians.com/v-benches/{id}`.

| Folder | Jira | Covers |
|---|---|---|
| `SD-3470/` | [SD-3470](https://castille-labs.atlassian.net/browse/SD-3470) | Virtual Benches page — list, filter, Create a new bench |
| `SD-3471/` | [SD-3471](https://castille-labs.atlassian.net/browse/SD-3471) | V Bench page: Bench Setup & Sharing — the page shell, naming, Need help?, capacity plan, members |
| `SD-3472/` | [SD-3472](https://castille-labs.atlassian.net/browse/SD-3472) | V Bench page: Skills Mix, Skills Matrix & Monthly Engineering Hours |
| `SD-3473/` | [SD-3473](https://castille-labs.atlassian.net/browse/SD-3473) | V Bench page: Engineer Work Logs |
| `SD-3474/` | [SD-3474](https://castille-labs.atlassian.net/browse/SD-3474) | V Bench page: Engineers |

This round covers the **Virtual Benches module**. Organisation and Subscriptions join this same epic as they are specified.

---

## Four rules worth reading before writing code

- **Nothing on this dashboard writes to a subscription or a roster.** Capacity changes, overage grants and engineer add/remove are all **requests** actioned on Internal (SD-3465). A manager pressing a button must never silently change what a client is billed, or who is on their bench.
- **Viewer controls are absent, not disabled.** A greyed-out button invites a support ticket asking why.
- **The client sees the blended rate; the engineer-facing rate never appears.** No mark-up, no retained margin, no per-engineer earnings (BE-04, BE-08).
- **Entries declined by a person, and all edit history, are filtered server-side** for Manager — never sent and hidden client-side (SD-3467, SD-3473). **Auto-declined entries are shown**: terminal, in the client's favour, and counted towards nothing.
- **Bench membership confers Performance Log eligibility.** A Manager with access to a bench may act as the reviewing manager for its engineers (SD-3471).

## Two exceptions to "the prototype is authoritative"

- The **engineer profile cards** in SD-3474 must be the **production** cards, not the prototype's simplified version.
- The **Engineer / Internal / Manager** header toggle is a prototype navigation device and **must not be built**.

## Build order

1. **SD-3470** — the list page; the entry point to everything else.
2. **SD-3471** — the bench page shell. The other three stories are sections **inside** it.
3. **SD-3472** — skills and hours, which own the capacity figures the rest of the page reads.
4. **SD-3473** and **SD-3474** in either order.

**SD-3459** (per-bench Start and Auto-Renew dates) blocks the lot: every period label, days-remaining chip and capacity figure resolves from it.

Spec folders are named by **Jira key only**, so renaming a ticket never invalidates a path.
