# Castillians Platform — Engineering Brief

Scope of work represented by the interactive prototype (`Castillians Platform.dc.html`).
Estimates assume 1 backend + 1 frontend engineer working in parallel, Angular 11 portal + existing API, Zoho as source of truth for client and currency data.

**How to read this document.** Section A is the backend specification: every rule carries a stable ID (`BE-xx`), the integration it touches, and testable acceptance criteria. These are written to be converted into Jira items one-to-one. Sections B–E describe the UI work per dashboard. Section F covers effort and risk.

---

# A. Backend specification

## A1. Integration points

These are the existing systems the prototype's data must read from and write back to. Nothing below should be re-entered or hardcoded in the new modules.

| Ref | System | Field / surface | Direction |
|---|---|---|---|
| INT-1 | Zoho CRM | Client record → `Parent Brand` | Read → builds the channel hierarchy |
| INT-2 | Zoho CRM | Client record → currency field | Read → drives all money formatting |
| INT-3 | Zoho CRM | Client record → client type (SI, VAD/VAR, Enterprise) | Read → tags on the Channel page |
| INT-3b | Zoho CRM | Client record → finance block: Client Code, VBench Code, Entity Code, SFM A/C, Nominal Account (subscription + overage), VAT Code | Read → month-end finance export (BE-23) |
| INT-3c | SFM | Supplier record → Company Name, SFM A/C, Nominal Account, VAT Code, Payment Type, IBAN, BIC | Read → month-end finance export (BE-23) |
| INT-4 | Zoho CRM | Contact / account records | Write → org member deletion must delete in Zoho |
| INT-5 | Internal Dashboard (live) | Manage Subscription modal → `Included Hours` | Read → capacity plan for every dashboard |
| INT-6 | Internal Dashboard (live) | Manage Subscription modal → `Purchased Hours` | Write ← derived overage writes back here |
| INT-7 | Internal Dashboard (live) | Manage Subscription modal → `Configured Blended Hourly Rate` | Read → rate shown to Internal + Manager |
| INT-8 | Internal Dashboard (live) | Manage Subscription modal → Core / Additional Skills | Read → Skills Mix (read-only downstream) |
| INT-9 | Internal Dashboard (live) | Mark-Ups & Discounts tab → `Percentage Mark-up` | Read → derives the engineer-facing rate |
| INT-10 | Internal Dashboard (live) | Manage Client modal → email domains | Read/Write, bidirectional |

---

## A2. Subscription & billing engine

> This is the critical path. Every dashboard reads from it. Build and test it first.

### BE-01 — Subscription record
A Virtual Bench subscription holds: `includedHours`, `purchasedHours`, `startDate`, `configuredBlendedRate`, `currency`, `autoRenewDate`, `benchType`, `timeZoneGroup`, `coreSkills[]`, `additionalSkills[]`.

**Acceptance criteria**
- Every field is sourced from the Manage Subscription modal (INT-5 → INT-8); none are duplicated or independently editable in the new modules.
- Currency resolves from the Client record in Zoho (INT-2). No default, no hardcoding.

### BE-02 — Billing cycle
Subscriptions bill on **calendar months**, with one exception for the first period.

- **First period** = `startDate` → last day of the **following** calendar month.
- **Every period after** = a whole calendar month.

**Acceptance criteria**
- A subscription starting 13 Jun has a first period of 13 Jun → 31 Jul; its second period is 1 Aug → 31 Aug.
- A subscription starting on the 1st of a month still gets the two-month first period.
- The period boundaries returned by the API are the same ones the Engineer date picker uses (BE-10).

### BE-03 — Pro-rated first-period capacity
`firstPeriodCapacity = round(includedHours × (daysRemainingInStartMonth ÷ daysInStartMonth)) + includedHours`

**Acceptance criteria**
- 160 h/month starting 13 Jun (18 of 30 days remaining) → `96 + 160 = 256` h for the first period.
- Subsequent periods return exactly `includedHours`.
- The pro-rated figure is what capacity bars, remaining-hours and approval thresholds all use during the first period.

### BE-04 — Billing floor (internal margin)
The client is billed for the **subscribed capacity regardless of hours logged**. Under-use is never refunded or pro-rated down.

`periodBill = capacity × configuredBlendedRate + overageHours × overageRate`

**Acceptance criteria**
- A bench with 160 h capacity and 78 h logged still bills `160 × rate`.
- The retained difference (capacity − logged) is **never** exposed in any API response consumed by the Engineer or Manager dashboards, and never rendered in the UI.
- Only hours above capacity increase the bill.

### BE-05 — Overage → Purchased Hours
Hours logged above the period capacity become purchased hours.

**Acceptance criteria**
- `purchasedHours = max(0, totalApprovedHours − capacity)`, computed per period.
- The derived value **writes back to the `Purchased Hours` field** in the Manage Subscription modal (INT-6). That field is the billing record — it is not separately keyed in.
- A bench at or under capacity writes `0`.

### BE-06 — First-3-months overage multiplier
For the first three months of a subscription, overage hours bill at **1.25 ×** the configured blended rate.

**Acceptance criteria**
- The multiplier and the three-month window are **configurable constants**, not literals in code.
- Month 4 onwards, overage bills at 1.0 ×.
- Any surface labelled **"Overage rate"** shows the *multiplied* figure (rate × 1.25 while in the promo window), not the raw configured blended rate. The raw rate is only shown where the label says "Blended Hourly Rate".
- Finance signs off the rule before release (see Risks).

### BE-07 — Auto-renew date
`autoRenewDate` = the 1st day of the 6th month after the subscription's start month.

**Acceptance criteria**
- Start 13 Jun 2025 → auto-renew 1 Dec 2025.
- Derived, never manually entered.

### BE-08 — Two hourly rates
| Audience | Rate shown |
|---|---|
| Internal + Manager dashboards | `configuredBlendedRate` (INT-7) |
| Engineer dashboard | `configuredBlendedRate ÷ (1 + markup)` where markup comes from INT-9 |

**Acceptance criteria**
- At a 60% mark-up, a configured rate of 95.00 shows engineers 59.38.
- The mark-up is read from the Mark-Ups & Discounts tab at request time; changing it there changes the engineer-facing rate with no deploy.
- The configured rate is never returned to an engineer-scoped API call.

### BE-09 — Money formatting
**Acceptance criteria**
- Currency **codes**, never symbols: `EUR 1,234.56`.
- Exact cents, never rounded to whole units.
- The code appears in the field label where the pattern allows: `BILLING (EUR)` with a bare value.

---

## A3. Work logs

### BE-10 — Loggable date range
An engineer may log hours only within the bench's **current** subscription period.

**Acceptance criteria**
- Future dates rejected.
- Dates in a prior period rejected, with a message naming the current period.
- The calendar view may still **display** prior periods read-only.

### BE-11 — Automatic approval routing
Entries that take a bench over its period capacity are **automatically** sent for internal approval. There is no manual "submit for approval" step.

**Acceptance criteria**
- Entry within capacity → status `auto-approved`, immediately visible on all dashboards.
- Entry that crosses capacity → status `approval required`, appears in the Internal review queue, and an approval-request email fires (BE-20).
- The status transition is recorded in the entry's history with actor `Castillians System`.

### BE-11 — Overage behaviour toggle (per bench)
Each Virtual Bench carries an **Allow Overages** switch that governs what happens when logged hours pass the capacity plan.

**Acceptance criteria**
- **OFF** — logging beyond the capacity plan is refused (hard cap at 100%). No approval flow runs and no overage emails are sent for that bench.
- **ON** — overage tracking, the approval flow and overage emails all become active.
- The switch is set from the bench row on the Internal → Engagements → Virtual Benches tab, with helper text stating what each state does.
- Toggling it re-derives every dependent figure live (engineer ceilings, thresholds, tooltips) with no page reload.

### BE-12 — Engineer allocation shares
When a bench has **more than one engineer**, each engineer's share of the bench is fixed as a **percentage of bench capacity**.

**Acceptance criteria**
- Allocation is **stored and configured as a percentage**, never as raw hours. Rationale: the 20% tolerance is itself a percentage, so a single unit avoids mixed-unit conversion, and shares auto-scale when capacity changes without manual re-entry.
- Shares across a bench **validate to ≤ 100%**; the UI shows a running total and flags any total above 100%.
- Engineers are **never shown the percentage**. Every engineer-facing surface renders the resolved figure in hours — e.g. "you can log up to 44h this period."
- Resolved ceiling = `allocation% × (capacity × 1.20)` when the tolerance applies (Allow Overages ON), and `allocation% × capacity` when it does not.
- Resolved hours update live when bench capacity changes or the overage state flips.

### BE-13 — Approval thresholds (only when Allow Overages is ON)
Approval applies **only** to hours exceeding the 100% capacity plan.

**Acceptance criteria**
- **0 → 100%** of capacity: accepted, auto-approved.
- **100% → 120%** (within the 20% tolerance): accepted, **no approval required**.
- **Above 120%**: **manual approval required** before those hours are accepted.
- At **90% of capacity**, an automated email fires to **both the client and Castillians**, warning that overages are about to trigger. Sent once per bench per period.
- With the toggle OFF none of the above runs — the entry is simply refused at 100%.
- Status indicators on Internal work-log entries, and the engineer's threshold tooltip and in-form note, all reflect these bands.

**Threshold notification matrix.** Which thresholds notify whom depends entirely on the bench's overage mode. Each fires at most **once per bench per period**.

| Overage mode | Threshold | Internal | Manager | Why |
|---|---|---|---|---|
| **Off** | 90% of plan | ✓ | ✓ | Logging will be blocked at the plan — both sides need warning |
| **Off** | 100% of plan | ✓ | ✓ | Work is now blocked; the client must act to continue |
| **On** | 100% of plan | ✓ | — | Hours keep flowing inside the tolerance, so this is operational only. The client is **not** alerted |
| **On** | 120% of plan | ✓ | ✓ | Entries now queue for approval — this affects the client commercially |
| **Unlimited** | any | — | — | **No notifications.** The client has pre-authorised everything; there is no threshold to announce |

**Acceptance criteria**
- Switching a bench between modes changes which emails are armed, with no other configuration.
- Turning **Unlimited** on suppresses every capacity notification for that bench, including any already-armed 90% or 100% notice not yet sent this period.
- A bench that crosses two thresholds in one logging action sends only the higher one.

### BE-22 — Month-end finance export (Payroll checklist + SFM batch)

On the **last day of the month**, once all engineer invoices are submitted, an automated email goes to **Shared Services** with a generated `.xlsx` populating the finance master file. It has two sheets.

**Sheet 1 — Payroll checklist (one row per engineer × Virtual Bench × month)**

| Group | Columns |
|---|---|
| Client | Client Name, Client Code, VBench Code |
| Supplier (engineer) | Supplier, Supplier Company Name, SFM A/C, NAC, Payment Type, VAT Code, Currency |
| Rates | Supplier Rate, Subscription Rate, Overage Rate (1st month) |
| Hours | Normal Hours, Overage Hours |
| Amounts | Invoice Contractor, Client Overage |
| Process tracking | T/Sheet Recvd, T/Sheet Apprvd, INV Calc Posted, Invoice Recvd, Invoice Number, Ready for SFM, Copied to File, SFM Posted, Payment Processed, Payment Checked |

**Sheet 2 — SFM supplier bulk upload (22 fixed columns, one row per supplier invoice)**

`SYSTEMBOOK · BOOK · BATCH · ENTITYCODE · NACCODE · TRANDATE · DUEDATE · TRANTYPE · PAYTYPE · INTREF · INV NO · PAYEE · DETAILS · ANALYSIS · DISCTYPE · DBCR · VATCODE · CURRENCY · AMOUNT · VATAMOUNT · FAMOUNT · FVATAMOUNT`

Observed constants from the sample: `SYSTEMBOOK=P`, `BOOK=P`, `TRANTYPE=IN`, `INTREF=Auto`, `DBCR=C`, `DISCTYPE` blank, `VATAMOUNT/FAMOUNT/FVATAMOUNT=0`. `TRANDATE` = period end date, `INV NO` = `MM/YYYY`, `DETAILS` = `MMM YY`.

**Column sourcing** — `ENTITYCODE` and `NACCODE` vary per client and come from that client's finance record (BE-23); everything above is a fixed constant. `BATCH` is generated per export run. `PAYEE` and `VATCODE` come from the supplier record, `CURRENCY` and `AMOUNT` from the invoice line.

**Timing** — the export fires on the **last calendar day of the month regardless of pending approvals**. Entries still awaiting approval at that moment are excluded from the run and carry into the next month's export; the email body lists them so Shared Services can see what was held back.

**Acceptance criteria**
- One checklist row per engineer per bench per month — an engineer on three benches produces three rows, each with its own client, rate and currency. Never aggregated (resolves the "different VBs and different clients" concern).
- Hours split into **Normal** (within the capacity plan) and **Overage** (everything beyond 100% of it, including hours inside the 20% tolerance) per BE-13, taken from approved work logs only.
- Supplier amounts use the **engineer-facing rate** (post-mark-up-removal, BE-08); client amounts use the **configured blended rate**. Never mixed.
- Currency per row comes from the relevant party's record — a supplier paid in USD against a client billed in EUR produces two rows in two currencies. **No cross-currency totalling.**
- Process-tracking columns are emitted blank for Shared Services to complete, except those the portal genuinely knows (T/Sheet Recvd, T/Sheet Apprvd).
- The file also serves as the CSV attachment described in BE-20's month-end payroll export.

### BE-23 — Finance reference data
The export needs static finance fields the portal does not currently hold: SFM Account Number, SFM Nominal Account (separate codes for subscription and overage on the client side), VAT Code, Payment Type, Bank IBAN, Bank BIC, Company Name, Client Code, VBench Code, Entity Code.

**Acceptance criteria**
- Each field has exactly one system of record. The portal **reads** these values at export time and never stores or re-keys them.
- A missing mandatory field blocks that row from the SFM sheet and is reported in the email body rather than silently exporting a blank.

**System of record — agreed.** Client-side fields come from **Zoho** (it already owns the client hierarchy and currency); supplier-side payment fields come from **SFM** (it owns the payment rails).

| Field | Applies to | Source |
|---|---|---|
| Client Name, Client Code | Client | **Zoho** |
| VBench Code | Client | **Zoho** |
| Entity Code (`ENTITYCODE`) | Client | **Zoho** |
| Client SFM Account Number | Client | **Zoho** |
| SFM Nominal Account — Subscription (`NACCODE`) | Client | **Zoho** |
| SFM Nominal Account — Overage | Client | **Zoho** |
| Client VAT Code | Client | **Zoho** |
| Currency | Client | **Zoho** (already the source per BE-01) |
| Monthly Subscription Rate, Overage Rate, Hours Included | Client | Internal Dashboard → Manage Subscription (INT-5 → INT-7) |
| Supplier Company Name | Supplier | **SFM** |
| Supplier SFM Account Number | Supplier | **SFM** |
| Supplier SFM Nominal Account | Supplier | **SFM** |
| Supplier VAT Code | Supplier | **SFM** |
| Payment Type | Supplier | **SFM** |
| Bank IBAN, Bank BIC | Supplier | **SFM** |
| Engineer Name, Email | Supplier | Portal (engineer profile) |
| Rate per hour (engineer-facing) | Supplier | Derived — configured rate ÷ (1 + mark-up), BE-08 |

**Engineering note.** Add a Zoho read for the client finance block (Client Code, VBench Code, Entity Code, SFM A/C, both Nominal Accounts, VAT Code) alongside the existing hierarchy and currency reads — same integration, extra fields. The SFM supplier lookup is new work: keyed on the engineer's supplier record, needed only at export time, so a batch fetch at month-end is sufficient rather than a live per-request call.

### BE-24 — Automatic engineer invoice submission
Finance asked whether the engineer can submit an invoice directly from the portal, since the portal already captures the hours. **The system auto-submits it on their behalf** at the end of the last day of the month — the engineer never files an invoice manually.

**Acceptance criteria**
- On the last day of the month the portal generates each engineer's invoice from their approved work logs and submits it automatically, in the same run as the finance export (BE-22).
- One invoice per engineer covering all their benches, itemised per bench (client, hours, engineer-facing rate, earnings) — matching the Invoices card on the Engineer dashboard.
- Submission stamps Invoice Number and Invoice Recvd on the checklist row.
- The engineer is emailed a copy of what was submitted.
- Hour verification is already covered: approval flow (BE-13) plus immutable edit history (BE-14) resolve Finance's "verification of engineer hours worked" concern.

### BE-25 — Ad-hoc Excel export
Finance asked for the flexibility to generate any report to Excel.

**Acceptance criteria**
- Every list surface that already filters (Engagements work logs, channel billing, subscriptions breakdown) exposes a "Download as CSV" action returning the current filtered view.

### BE-14 — Edit history
Every create, edit, approval and decline is recorded immutably.

**Acceptance criteria**
- Each edit stores per-field **before → after** values for hours, date and description.
- History is visible on the Engineer and Internal dashboards; **hidden on the Manager dashboard**.
- Editing an entry with the **same or fewer hours** (e.g. a description fix) does **not** re-open review or fire a notification.
- Increasing hours past capacity does re-open review.

### BE-13 — Decline flow
**Acceptance criteria**
- A decline message is mandatory; the API rejects an empty message.
- Declined entries remain visible on the Internal and Engineer dashboards with a `Declined` label.
- Declined entries are **not** returned to the Manager dashboard.

### BE-14 — Shared bench capacity
Multiple engineers draw from one bench pool.

**Acceptance criteria**
- Any engineer's logged hours immediately update the bench totals seen by every other engineer on that bench.
- Capacity checks are performed **server-side**; two simultaneous submissions cannot both pass a check that only one should.

---

## A4. Organisation, roles & access

### BE-15 — Role model
Three roles per client organisation: **Admin**, **Manager**, **Viewer**.

**Acceptance criteria**
- Exactly **one Admin per client** at any time.
- Admin: full channel access, invites Managers and Viewers, owns the subscription, creates benches.
- Manager: assigned benches only, invites Viewers, cannot create benches.
- Viewer: read-only on assigned benches; no work-log detail.
- Default role for a cobranded sign-up is **Manager**.
- Enforced in the API, not by hiding UI.

### BE-16 — Channel-wide visibility
The Admin of a Root client sees members of every client beneath it in the channel hierarchy (INT-1).

**Acceptance criteria**
- Members are grouped by client in the response.
- A child-client Admin sees only its own subtree.

### BE-17 — Transfer of Admin ownership
**Acceptance criteria** — the API rejects a transfer unless all hold:
1. The nominee already has access to a client within this account.
2. The nominee is not already the Admin of another account.
3. The nominee's email uses one of that client's approved domains (BE-18).

On success: the subscription moves to the new Admin, and the previous Admin is demoted to Manager in the same transaction.

**Notifications** — three emails fire on a successful transfer (BE-20): the **incoming Admin** (what they now own), the **outgoing Admin** (that it moved, to whom, and that they are now a Manager), and the **Castillians team** (audit record — the subscription's billing contact has changed). All three send only after the transaction commits; a rejected transfer sends nothing.

### BE-18 — Email domains
Per-client allow-list of one or more domains. Only these addresses may sign up or be invited.

**Acceptance criteria**
- Bare domain only — no scheme, no `@`, no path. Reject anything else.
- No duplicates within a client.
- A client can never be left with **zero** domains; the delete action is blocked when one remains.
- Deleting a domain: existing accounts **keep** access; pending invites on that domain are **revoked**; new sign-ups and invites on it are refused.
- **Bidirectional sync with the Manage Client modal** on the Internal Dashboard (INT-10).

### BE-19 — Member removal: two distinct actions
| Surface | Effect |
|---|---|
| Virtual Bench page → revoke | Strips **that one bench** from the member's access. Account stays active; other benches unaffected. |
| Organisation page → remove | **Deletes the account entirely.** Must also delete the corresponding record in Zoho (INT-4). |

**Acceptance criteria**
- Both actions require confirmation.
- The bench-level revoke never deletes an account.
- The org-level removal is reflected in Zoho within the same operation.

---

## A5. Notifications

### BE-20 — Transactional emails
| Trigger | Recipient | Contents |
|---|---|---|
| Daily work log summary | Castillians team | **Once daily at 08:00 CET**, covering everything submitted the previous day — one row per entry: engineer full name, email, bench name, client name, hours logged, plus an entry and hours total. **Not** one email per entry |
| Entry exceeds capacity → approval required | Castillians team | **Distinct template** from the above. Must carry enough for a reviewer to decide without opening the dashboard: **hours requested**; the engineer's **assigned hours** this period, **hours already logged**, and **hours needing approval**; and the client position — **capacity plan**, **overages agreed**, **overages used**, **overage rate**, and bench capacity used. "Overages agreed" resolves per BE-13: the authorised total where one is set, otherwise the standard 20% tolerance |
| Entry approved | Engineer | Confirmation |
| Entry declined | Engineer | Carries the decline message verbatim |
| Bench invite sent | Invitee | Link to the bench |
| Admin ownership transferred — to the **new Admin** | Incoming Admin | Confirms they now hold the account: subscription ownership, billing responsibility, full channel access, ability to invite Managers and Viewers |
| Admin ownership transferred — to the **outgoing Admin** | Previous Admin | Confirms the transfer, names who it moved to, and states they are now a Manager |
| Admin ownership transferred — notice | Castillians team | Audit record: client, previous Admin, new Admin, timestamp — the subscription's billing contact has changed |
| Weekly satisfaction report (per bench) | Manager | Traffic-light colour **plus** hours used vs total for the bench — e.g. "68 / 80h — 85%". CTA + link to the work log; submitting notifies the team |
| 90% capacity, overages OFF | Internal + Manager | Logging will be blocked at the plan (BE-13) |
| 100% capacity, overages OFF | Internal + Manager | Work now blocked — switch overages on or raise the plan |
| 100% capacity, overages ON | Internal only | Overage has started; client not alerted |
| 120% capacity, overages ON | Internal + Manager | Entries now queue for approval |
| Unlimited overage | — | **No capacity notifications at all** |
| Month-end payroll export | **Shared Services** | Sent on the **last day of the month**. Carries the per-engineer earnings breakdown (bench, client, hours logged, hourly rate, earnings) **plus a CSV attachment covering every engineer across all benches**. Engineer-facing rates are post-mark-up-removal per BE-08. Body lists any entries held back for pending approval (BE-22) |
| Auto-submitted invoice copy | Engineer | Sent when the system files their invoice on their behalf (BE-24) |
| Weekly timesheet reminder | Engineer | Their bench(es) + outstanding unlogged hours (BE-21) |
| Timesheet cut-off reminder (3 days before) | Engineer | Same scope, flagged as closing soon (BE-21) |

### BE-21 — Timesheet reminders
Automated reminders prompting engineers to keep their work log current.

**Acceptance criteria**
- A **weekly** reminder is sent to every activated engineer to log their hours.
- An **additional** reminder is sent **3 days before the timesheet cut-off date** for the current subscription period.
- Both emails are scoped to the recipient's own Virtual Bench(es) — an engineer on three benches sees all three, and never another engineer's.
- Each bench line states its **outstanding / unlogged hours** for the current period, derived from the same figures the Engineer dashboard shows.
- An engineer with no benches, or whose engagements have all ended, receives no reminder.
- The cut-off date is derived from the period rules in BE-02, not stored separately.

### BE-22 — Bench invite journey
**Acceptance criteria**
- No account → the invitee must register with the **exact** invited email address; any other address is rejected.
- Existing account → log in.
- Either path redirects to the invited bench on success.

---

# B. Engineer Dashboard — Work Log

- Tab per assigned Virtual Bench
- Log hours + description against a date; picker constrained per BE-10
- Auto-routing to approval per BE-11 — no manual submit
- Edit entries; expandable history with before → after diffs (BE-12)
- List and calendar views; calendar browsable across past periods, read-only
- Capacity bar with segment hovers: total logged, your share (darker band), remaining
- Overview stats: period hours, hours logged (bench-wide), your hours logged, your rate (BE-08), your earnings, manager

**Effort: 2–2.5 weeks** FE, ~1 week BE.

# C. Internal Dashboard

**Channel page** — root-client cards (benches, engineers, skills, capacity, own billing + channel billing); interactive family-tree accordion with generation and client-type tags; bench drilldown with rate, capacity, purchased hours, skills matrix, engineer rosters, auto-renew and a link to Manage Subscription.

**Engagements page** — *Virtual Benches* tab: capacity table with search plus channel/client filters, expandable rows holding an internal notes field and that bench's work logs with inline approve/decline. *Work Logs* tab: chronological inbox, engineer search, period filter, ongoing/past scope, approval-only toggle, pagination; per-engineer work-log page with per-bench tabs.

**Effort: 3–3.5 weeks** FE, ~1.5 weeks BE.

# D. Manager Dashboard

Virtual Benches grid with capacity bars and a create card gated on the subscription allowance; bench detail with members-with-access, Skills Mix (read-only — set only via INT-8), Skills Matrix, hours, rate and per-engineer usage; Organisation page (Admin only) with channel-wide members, role changes, revoke, Admin transfer and email domains; Subscriptions page with channel totals, per-client plan breakdown, per-bench Manage Subscription, a Request-a-Bench flow and a period-navigable Billing card.

**Effort: 2 weeks**, mostly authorisation.

# E. Design system

Two additions recommended:

1. **Tertiary button variant** — currently a prop-level override (`#F8F8F8` fill, 2px `#E5E5E5` border, `#4d4d4d` label, hover `#F2F2F2`). Smallest change: add `color="tertiary"` to `Button`'s fill map.
2. **Capacity progress bar** — colour-banded (green ≤59%, amber 60–89%, red ≥90%) with tinted track, used on every dashboard. Should become a real component rather than repeated inline markup.

Everything else (Card, Input, Select, Button, StatusBadge, Avatar, SkillTag, Toast) is reused as-is.

**Effort: 3–4 days.**

---

# F. Timeframe & risk

| Phase | Duration |
|---|---|
| Data model + subscription/billing engine (A2) | 3–4 weeks |
| Engineer Work Log (A3, B) | 2.5 weeks |
| Internal Channel + Engagements (C) | 3.5 weeks |
| Roles, permissions, Manager dashboard (A4, D) | 2 weeks |
| Notifications (A5) | 1.5 weeks |
| Design system additions (E) | 0.5 week |
| QA, UAT, hardening | 2 weeks |

**Sequential total ~15 weeks. Two engineers in parallel after the data model lands: 9–11 weeks.**

## Risks

- **Zoho hierarchy sync** (BE / INT-1) is the least specified area — infinite nesting and parent reassignment need their own spike.
- **Pro-rating and the ×1.25 multiplier** (BE-03, BE-06) directly affect invoices. Finance must sign off the rules before build.
- **Shared-bench concurrency** (BE-14) needs server-side capacity checks; a client-side guard will let double-submissions through.
- **The billing floor** (BE-04) is commercially sensitive. Confirm no API response leaks the capacity-vs-logged difference to client-facing dashboards.
