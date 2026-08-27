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
| `SD-3477/` | [SD-3477](https://castille-labs.atlassian.net/browse/SD-3477) | Organisation page: Access — the route, the menubar entry, Admin-only |
| `SD-3478/` | [SD-3478](https://castille-labs.atlassian.net/browse/SD-3478) | Organisation page: Functions — Change Account Admin, Brands & Members, invites, Add another brand, Email Domains |
| `SD-3479/` | [SD-3479](https://castille-labs.atlassian.net/browse/SD-3479) | Organisation page: Member Roles & Removal |

This round covers the **Virtual Benches module** and the **Organisation module**.

The **Subscriptions module** is filed into this same epic as **SD-3480** (page access + summary totals), **SD-3481** (Monthly Capacity Plans), **SD-3482** (Manage Subscription + Subscribe to a new Virtual Bench) and **SD-3483** (Monthly Billing History). Spec folders for those four follow; the Jira descriptions are authoritative until they land.

---

## Four rules worth reading before writing code

- **Nothing on this dashboard writes to a subscription or a roster.** Capacity changes, overage grants and engineer add/remove are all **requests** actioned on Internal (SD-3465). A manager pressing a button must never silently change what a client is billed, or who is on their bench.
- **Viewer controls are absent, not disabled.** A greyed-out button invites a support ticket asking why.
- **The client sees the blended rate; the engineer-facing rate never appears.** No mark-up, no retained margin, no per-engineer earnings (BE-04, BE-08).
- **Entries declined by a person, and all edit history, are filtered server-side** for Manager — never sent and hidden client-side (SD-3467, SD-3473). **Auto-declined entries are shown**: terminal, in the client's favour, and counted towards nothing.
- **An invitation starts an account.** The email links to the **Manager sign-up page** carrying the invitation; on completing onboarding the invitee lands on their own Manager dashboard showing exactly the benches they were granted (SD-3471, SD-3470). **Admins and Managers may both invite**, as Manager or Viewer — **Admin is never invitable**. Removing access stays Admin-only. On completion a **manager profile is written to the Zoho client record for that brand** — the brand the bench belongs to, not the root client.
- **Email domains are per brand.** A brand parented into the lineage in Zoho arrives with its own Email Domains column, derived from the client records — nothing to configure, and domains are never pooled across the channel (SD-3471).
- **Bench membership confers Performance Log eligibility.** A Manager with access to a bench may act as the reviewing manager for its engineers (SD-3471).
- **Admin is the only account-wide role. Manager and Viewer are per bench.** The same person can be a **Manager of one bench and a Viewer of another**, so a role control belongs on a bench, not on a member — and a single role tag on a member row would be a lie (SD-3478, SD-3479). SD-3479's Jira description was corrected on 26 Aug — the earlier account-wide wording is retained there under an *Outdated on 26 Aug* note, with the per-bench model beneath it.
- **Subscriptions is Admin-only too**, on the same rule and the same Figma node as Organisation (SD-3480). A Manager could previously open it read-only with a note saying their Admin manages it; that access and that note are both gone.
- **Currency is a property of the subscription, not of the brand or the channel** (BE-27). A single brand can hold benches billed in different currencies, so **every total that sums benches is one figure per currency** — never combined, never converted, no FX anywhere. The new-bench request therefore collects a currency (SD-3470).
- **The Organisation page is Admin-only, and absent for everyone else.** No menubar entry, no route, no mention (SD-3477). It **replaces** `castillians.com/brand-profile`, which is unpublished as part of that work.

## Two exceptions to "the prototype is authoritative"

- The **engineer profile cards** in SD-3474 must be the **production** cards, not the prototype's simplified version.
- The **Engineer / Internal / Manager** header toggle is a prototype navigation device and **must not be built**.

## Build order

1. **SD-3470** — the list page; the entry point to everything else.
2. **SD-3471** — the bench page shell. The other three stories are sections **inside** it.
3. **SD-3472** — skills and hours, which own the capacity figures the rest of the page reads.
4. **SD-3473** and **SD-3474** in either order.

Then the Organisation module, which is independent of the bench pages:

5. **SD-3477** — the route and its Admin-only guard; the frame the rest mounts into.
6. **SD-3478** — the functions, which need SD-3471's bench membership to render access chips.
7. **SD-3479** — roles and removal, which share SD-3478's member rows.

**SD-3459** (per-bench Start and Auto-Renew dates) blocks the lot: every period label, days-remaining chip and capacity figure resolves from it.

Spec folders are named by **Jira key only**, so renaming a ticket never invalidates a path.
