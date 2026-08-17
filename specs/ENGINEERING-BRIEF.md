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
| **INT-11** | **SFM (Shireburn)** | **Supplier invoice batch upload + supplier/nominal reference data** | **Write + Read → the whole month-end export (BE-22). Replaces the sync our current timesheet system already performs. See §A6.** |
| INT-4 | Zoho CRM | Contact / account records | Write → org member deletion must delete in Zoho |
| INT-5 | Internal Dashboard (live) | Manage Subscription modal → `Included Hours` | Read → capacity plan for every dashboard |
| INT-5b | Internal Dashboard (live) | Manage Subscription modal → per-bench Start Date + Auto-Renew Date, in the `AUTO-RENEW DATE` accordion (BE-07) | Read → billing cycle, capacity and loggable dates, resolved per bench |
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

### BE-07 — Auto-renew date, **per Virtual Bench**

Each Virtual Bench carries **its own** subscription period. The account-level Start/Expiration pair is removed — one client's benches may run on entirely different cycles, so their teams can be scaled and renewed independently.

**Current behaviour being replaced.** The Manage Subscription modal (Internal → V Benches → Subscriptions) holds **Start Date** and **Expiration Date** in its *Details* section, applying to every bench on the account. Expiration defaults to 180 days (6 months) from the Start Date, with a **Set as indefinite** checkbox.

**Target behaviour**

| Change | Detail |
|---|---|
| **Remove** from *Details* | The account-level Start Date and Expiration Date fields, their helper text, and the **Set as indefinite** checkbox |
| **Add** per bench | A new **AUTO-RENEW DATE** accordion inside each bench's dropdown in *Activate V Benches*, holding that bench's Start Date and Auto-Renew Date |
| **Rename** | **Expiration Date → Auto-Renew Date**, everywhere. One field, one name, matching `autoRenewDate` in the API |
| **Accordion order** | `AUTO-RENEW DATE` → `V BENCH PRICE` → `BLENDED HOURLY RATE` → `MONTHLY ENGINEERING HOURS` → *(remaining accordions unchanged)*. The new one is **first** |

#### Auto-renew date pre-fill

The pre-filled value is the **last day of the sixth full month** after the Start Date. A partial start month does not count as one of the six.

| Start Date | Sixth full month | Pre-filled auto-renew |
|---|---|---|
| 13 January | July | **31 July** |
| 1 March | August | **31 August** |
| 28 February | August | **31 August** |
| 15 September | March (next year) | **31 March** |

This aligns renewal with the month-end billing cycle (BE-02), so a period never closes mid-month and no partial month is billed at renewal.

**Acceptance criteria**
- Every bench stores `startDate` and `autoRenewDate` on its own record; nothing reads an account-level pair.
- The pre-fill is the **last day of the sixth full month** after `startDate`. A start on the 1st still yields six full months — the start month counts only when the subscription begins on its first day.
- The pre-fill is a **suggestion, not a constraint**. An internal user may change it to **any future date**, month-end or not, and the saved value is used verbatim. The backend validates only that it is in the future and after `startDate` — it never recalculates or corrects a manually set date.
- Changing `startDate` re-runs the pre-fill **only while the auto-renew date is untouched**. Once edited manually it is sticky, and a later Start Date change must not silently overwrite it.
- The **Set as indefinite** option moves with the field, per bench. When set, `autoRenewDate` is null and no renewal is scheduled.
- The existing three-select control (day / month / year) is preserved as-is; only its location and label change.
- Two benches on one account can hold different periods with no interaction between them.
- Changing one bench's dates **must not** alter any other bench's period.
- Everything derived from the subscription period is re-derived **per bench** — the billing cycle and pro-rated first period (BE-02, BE-03), capacity (INT-5), and the engineer's loggable date range (BE-10). A bench's engineers see that bench's period, never the account's.
- Both dashboards read the per-bench value: the Internal Channel page's Auto-renew row, and the Manager Subscriptions page.

**Migration.** Existing accounts hold one pair covering all their benches. On release, **copy the account-level Start and Expiration onto every bench verbatim** — each bench inherits the same values it was already effectively running on, so no period changes on the day — then drop the account-level columns. Migrated dates are treated as **manually set**, so a later Start Date edit will not re-run the pre-fill over them. A bench left without dates would silently break its billing cycle, capacity resolution and work-log date bounds.

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

Observed from the supplied template (`SFM Supplier Invoices Upload.xlsx`), single sample row:
`SYSTEMBOOK=P`, `BOOK=P`, `BATCH=26`, `NACCODE=501006`, `TRANDATE=29/05/2026`, `DUEDATE=21`, `PAYTYPE=24`, `INV NO=05/2026`, `PAYEE=29`, `ANALYSIS=25`, `VATCODE=60`, `CURRENCY=EUR`, `AMOUNT=2646`, `VATAMOUNT/FAMOUNT/FVATAMOUNT=0`. `ENTITYCODE`, `TRANTYPE`, `INTREF`, `DETAILS`, `DISCTYPE` and `DBCR` were **blank**.

> ⚠ **Do not treat the sample as a complete contract.** An earlier version of this template carried values for `TRANTYPE`, `INTREF` and `DBCR` that the current one leaves blank, and four cells hold bare integers whose meaning is not self-evident — `DUEDATE=21`, `PAYTYPE=24`, `PAYEE=29`, `ANALYSIS=25`. These look like SFM internal reference codes rather than dates or names. **Confirm every column against a real SFM import before building the generator** (see INT-11).

**Column sourcing** — `ENTITYCODE` and `NACCODE` vary per client and come from that client's finance record (BE-23). `BATCH` is generated per export run. `PAYEE` and `VATCODE` come from the supplier record, `CURRENCY` and `AMOUNT` from the invoice line.

**Timing** — the export fires at **23:59 on the last calendar day of the month**, which is also the work-log cut-off. It runs **regardless of pending approvals**: entries still awaiting approval at 23:59 are excluded from the run and carry into the next month's export; the email body lists them so Shared Services can see what was held back.

**All four month-end emails are sent by the same 23:59 job**, in this order, so the numbers in each reconcile against a single snapshot:

| # | Email | Recipient | Attachment |
|---|---|---|---|
| 1 | Payroll checklist | Shared Services | `payroll-checklist-YYYY-MM.csv` |
| 2 | SFM supplier upload | Shared Services | `sfm-supplier-upload-YYYY-MM.csv` |
| 3 | Engineer invoicing | Shared Services | `engineer-invoicing-YYYY-MM.csv` |
| 4 | Client billing | Shared Services | `client-billing-YYYY-MM.csv` |

Plus one per engineer: the auto-submitted invoice copy (BE-24).

**Acceptance criteria for the job**
- Scheduled at **23:59 local (CET)** on the last calendar day of the month — the same instant the work-log cut-off closes. It must not run before the cut-off, or late entries logged that evening are silently dropped from the period they belong to.
- All four reports are generated from **one snapshot** taken at 23:59. Generating them sequentially against live state risks an entry landing mid-run and the totals disagreeing between attachments.
- The job is **idempotent**: re-running it for a closed period reproduces byte-identical files and does not re-send.
- A failure on any one report does not suppress the others; each is sent independently and failures are alerted.
- Every report is **also downloadable on demand** for the current or any prior period (BE-25) — Finance is never blocked waiting for the scheduled run. On-demand delivers the CSV directly in the browser; **no email is sent for an on-demand download.**

**Acceptance criteria**
- One checklist row per engineer per bench per month — an engineer on three benches produces three rows, each with its own client, rate and currency. Never aggregated (resolves the "different VBs and different clients" concern).
- Hours split into **Normal** (within the capacity plan) and **Overage** (everything beyond 100% of it, including hours inside the 20% tolerance) per BE-13, taken from approved work logs only.
- Supplier amounts use the **engineer-facing rate** (post-mark-up-removal, BE-08); client amounts use the **configured blended rate**. Never mixed.
- Currency per row comes from the relevant party's record — a supplier paid in USD against a client billed in EUR produces two rows in two currencies. **No cross-currency totalling.**
- Process-tracking columns are emitted blank for Shared Services to complete, except those the portal genuinely knows (T/Sheet Recvd, T/Sheet Apprvd).
- The file also serves as the CSV attachment described in BE-20's month-end payroll export.

### BE-23 — Finance reference data

> **Terminology.** In both report templates and in SFM, **"supplier"** and **"contractor"** both mean **the engineer**. The three terms are interchangeable; the reports use supplier/contractor, the platform UI uses engineer.

The export needs static finance fields the portal does not currently hold. These are added as **custom fields in Zoho** under a new **Finance** section on each record type, and **read at export time** — never copied into the portal.

#### Engineer (supplier) — Zoho Finance section

| Field | Feeds |
|---|---|
| Engineer's Name | Checklist `Supplier` |
| Company Name | Checklist `Supplier Company Name` |
| Email Address | Invoice copy recipient (BE-24) |
| SFM Account Number | Checklist `SFM AC` |
| SFM Nominal Account | Checklist `NAC`, SFM `NACCODE` |
| VAT Code | Checklist `VAT Code`, SFM `VATCODE` |
| Currency | Checklist `Currency`, SFM `CURRENCY` |
| Payment Type | Checklist `Payment Type`, SFM `PAYTYPE` |
| Bank IBAN Number | SFM payment rails |
| Bank BIC Code | SFM payment rails |

#### Client — Zoho Finance section

| Field | Feeds |
|---|---|
| Client Name | Checklist `Client Name` |
| Email Address | Billing correspondence |
| SFM Account Number | Client-side SFM posting |
| SFM Nominal Account **Subscription** | Subscription revenue line |
| SFM Nominal Account **Overage** | Overage revenue line — a **separate** code from subscription |
| VAT Code | Client-side VAT |
| Currency | All client-facing money (already INT-2) |
| Payment Type | Client-side posting |
| Bank IBAN Number | Client-side posting |
| Bank BIC Code | Client-side posting |

Some of these already exist on the Zoho records — **audit before creating**, so the same value does not end up in two fields.

#### What comes from the platform, not Zoho

Zoho supplies the *static* finance attributes above. Everything period-specific is computed by the platform and joined at export time:

| From the platform | Source |
|---|---|
| **VBench Code** | The bench's **V Bench Index Number** — the identifier that already exists on the live platform. Not a new sequence |
| **VBench Name** | **New column**, added beside VBench Code, carrying the bench's display name |
| bench type | Bench record |
| Subscription Rate, capacity plan | Manage Subscription (INT-5) |
| Supplier Rate (engineer-facing) | Configured rate ÷ (1 + mark-up), BE-08 |
| Overage Rate, 1st-month multiplier | BE-05, BE-06 |
| Normal Hours, Overage Hours | Approved work logs, split at 100% of plan |
| Invoice Contractor, Client Subscription, Client Overage | Computed amounts |
| Period start and end | Per-bench cycle (BE-02, BE-07) |

#### Row structure

The checklist alternates **one client-level row** carrying the subscription rate and client totals, then **one row per engineer** beneath it with that engineer's supplier detail.

**Acceptance criteria**
- A bench with several engineers produces **one engineer row each**, all beneath the same client row.
- **VBench Code and VBench Name repeat on every engineer row**, not only on the client row — each row must be readable in isolation, since Finance sorts and filters the sheet.
- VBench Code is the **existing V Bench Index Number** from the live platform. Generating a fresh code would break reconciliation against records already keyed on it.
- An engineer on several benches appears once per bench, each occurrence carrying that bench's code, name, rate and hours.

**Acceptance criteria**
- Each field has exactly one system of record. The portal **reads** these values at export time and never stores or re-keys them.
- A missing mandatory field blocks that row from the SFM sheet and is reported in the email body rather than silently exporting a blank.
- An engineer or client with an incomplete Finance section is surfaced **before** the month-end run, not at 23:59 — a validation view listing incomplete records is worth building alongside.
- Both reports are generated from the same Zoho read, so a supplier's currency cannot differ between the checklist and the SFM upload.

#### Open questions on the checklist template

None outstanding.

**Resolved:**
- `Q` is **not populated by the system** — emitted blank.
- `R` is **Client Subscription**, `S` is **Client Overage** — both platform-computed.
- Columns **`T` onwards are filled by hand** by the Castillians team. The platform emits them **blank** and never writes to them: `T/Sheet Recvd.`, `T/Sheet Apprvd.`, `INV Calc Posted`, `Invoice Recvd.`, `Invoice Number`, `Ready for SFM`, `Copied to File`, `SFM posted`, `Payment Process.`, `Payment Checked`, the reviewer column, and `Changes after checking`.
- The client-side payment fields (`Payment Type`, `Bank IBAN`, `Bank BIC`) are **synced per client from the Zoho client profile** — they are read like every other Finance field.

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

### BE-27 — Per-engineer invoice PDFs, zipped

The Engineer invoicing email carries **two** attachments: the spreadsheet breakdown, and a zip holding **one PDF invoice per engineer**.

**Acceptance criteria**
- One PDF per engineer, not per bench — an engineer across three benches gets a single invoice itemised by bench, matching BE-24.
- Filenames sort predictably inside the zip: `surname-forename-YYYY-MM.pdf`. Finance opens these in a file browser, so alphabetical order by surname matters.
- Zip name: `engineer-invoices-YYYY-MM.zip`.
- An engineer who **uploaded their own** invoice (BE-24) has **their PDF** in the zip, not a generated one — the zip is the complete set for the period either way, never a mix of both for the same person.
- Engineers with zero approved hours are **omitted**, not included as a zero invoice.
- Generated from the same 23:59 snapshot as every other month-end report, so the zip and the spreadsheet always reconcile.

#### Invoice PDF template

A simple, standard invoice document, issued **from the engineer to Castille Resources Ltd.** — the engineer is the supplier, we are the customer.

**From (supplier) — read from the engineer's Zoho Finance section (BE-23):**

| Field |
|---|
| Engineer's Name |
| Company Name |
| Email Address |
| SFM Account Number |
| SFM Nominal Account |
| VAT Code |
| Currency |
| Payment Type |
| Bank IBAN Number |
| Bank BIC Code |

**To (customer) — fixed on every invoice:**

> **Castille Resources Ltd.**
> Glandore, City Quarter, Lapp's Quay, Cork City, Ireland
> VAT: MT1765-0137

**One template, three surfaces.** The **same** PDF template renders for:
1. **Download auto-generated invoice** on the Engineer → Invoices page;
2. the **Engineer invoicing** month-end email's zipped per-engineer PDFs;
3. the **blank template** an engineer downloads from the upload modal (BE-28) — identical layout, empty rows.

They must never diverge. Build one renderer and call it from all three.

**Header — auto-generated**

| Field | Rule |
|---|---|
| Invoice date | The period end date |
| Invoice number | `INV00001`, `INV00002`, … — a **single global sequence**, always counting upward |
| Name | The engineer's **full name** |
| Address | Read from Zoho. **Left blank when null** — never a placeholder or "N/A" |

**Line items — one row per Virtual Bench**

| Column | Value |
|---|---|
| Description | `{Bench Name} - {Client Name}` — e.g. **Core Platform - Northmill Bank** |
| Hours | Hours the engineer logged on that bench in the period |
| Rate | That **bench's** engineer-facing rate |
| Amount | Hours × Rate |

**Acceptance criteria**
- The document reads as issued **by the engineer**, not by Castillians on their behalf — supplier details lead, our address sits in the To block.
- The customer block is a **configurable constant**, not hardcoded per invoice. It will change if the billing entity does.
- Description is exactly `{Bench} - {Client}`. Both names, one row per bench.
- The total sums the line amounts and must reconcile with the Invoices page for that period, to the cent.

**Multiple currencies → multiple tables in one PDF**

Currency follows the **client engagement** — bill a client in GBP and that bench's rate is GBP.

- An engineer on benches billed in **different currencies** gets **one table per currency**, each with its own subtotal, **all within the same PDF**.
- Group benches by currency; a currency with one bench still gets its own table.
- **Never** combine currencies into one total, and **never** convert between them.
- Table order is alphabetical by currency code, so successive invoices read consistently.

**Invoice numbering**
- A **single global sequence** across all engineers and all periods — `INV00001` upward. Not per engineer, not per period.
- **Numbers are never reused.** Allocate from a sequence the database owns, not by counting existing invoices, or two concurrent month-end runs will collide.
- Zero-padded to five digits, prefixed `INV`.
- The number on the PDF matches the payroll checklist row for that engineer, so the two reconcile.

**Sourcing**
- Name, address, company name, VAT code, payment type, **IBAN and BIC all read from Zoho** (BE-23) at render time — never copied into the portal.
- Bank details print **in full**; a truncated IBAN makes the invoice unusable.
- A missing mandatory supplier field **blocks that PDF** and is reported in the email body, consistent with BE-23. Never print a blank where an account number belongs. The **address is the exception** — blank is valid.

**Confirmed with Finance:**
- **One invoice number per PDF**, however many currency tables it contains. The subtotals sit under a single number — do **not** allocate one per table.
- **An uploaded invoice does not consume a number.** Where an engineer uploads their own (BE-24), no PDF is generated and the sequence **skips them entirely** — it is not reserved, and the next generated invoice takes the next number in order. The sequence therefore has no gaps.

**Size ceiling.** At current volumes (14 engineers) the zip is a few hundred KB and attaches without issue. Most mail servers reject above ~25 MB, so:
- If the zip exceeds a **configurable threshold**, attach the spreadsheet only and replace the zip with a **secure, expiring download link** in the email body.
- The threshold is a config value, not a literal — it will need tuning as the engineer count grows.

### Pagination — specified per surface

Every list that paginates or truncates must state **its own** page size or reveal threshold. These are product decisions, not defaults for a developer to pick.

| Surface | Behaviour | Size |
|---|---|---|
| Engineer → Work Log → **Entries** (list view) | Paginated, prev/next with "Page N of M" | **5 per page** |
| Internal → Engagements → **Work Logs** | Paginated | **8 per page** |
| Manager → Virtual Bench → **Engineer Work Logs** | **First 3 shown**, then a "See more (N)" button expands the list; once expanded it paginates | **3 collapsed / 20 per page expanded** |
| Manager → Virtual Bench → **Skills Matrix** | First 12 skills, then "See more" | **12** |

**Acceptance criteria**
- Pagination controls render **only** when the list exceeds one page — never a lone "Page 1 of 1".
- A **newly created entry resets pagination to page 1** on every list showing it, so it cannot land on an unseen page.
- Changing a filter, period or tab resets to page 1.
- Page size is a **named constant** per surface, not a literal buried in a slice expression — these values will be tuned.
- Where a list both truncates and paginates (the Manager bench logs), expanding via "See more" jumps to page 1 of the paginated view; it does not append.
- "See more (N)" states the **remaining** count, not the total.

### Upload UI — platform default

**Every file-upload surface on the platform uses this treatment.** Established on the engineer invoice upload; reuse it rather than designing a new one.

| Property | Value |
|---|---|
| Panel | `--gray-50` fill (`--drop-bg` variable), **8px** radius, **2px solid `--border`** |
| Padding | 26px 24px, contents centred with a 20px gap |
| Badge | 45×45 circular asset (`assets/icons/upload-badge.svg`), rendered as `<img>` — **not** a CSS mask, which would flatten its two-tone fill |
| Label | Bricolage 16px. Lead **weight 600 in `--ink-900`** ("Click to upload"), remainder `#4d4d4d` ("or drag and drop") |
| Hint | Montserrat 12px `#4D4D4D`, stating format and size limit — e.g. "Upload in PDF, under 10MB" |
| Hover | Panel to `#F2F2F2` over 300ms `cubic-bezier(0.35,0,0.25,1)` |
| Chosen state | The lead swaps to the filename; the hint stays |
| Input | Native `<input type="file">` hidden inside the `<label>`, with an explicit `accept` |

**Acceptance criteria**
- The whole panel is the click target, and it is a `<label>` — so keyboard focus and screen readers reach the input without extra ARIA.
- `accept` is always set to the formats named in the hint; the two must not disagree.
- The size limit in the hint is enforced **server-side** as well, never by the hint alone.
- Worth extracting as a shared component the second time it is needed.

### BE-28 — Blank invoice template download

Engineers without an invoice of their own can download a **blank template** from the upload modal.

**Acceptance criteria**
- A **Download template** action sits beneath the drop zone, prefaced "No invoice of your own?".
- Serves a static PDF — the same document for every engineer, with no personal or period data merged in.
- Addressed to **Castille Resources Ltd.**, matching the generated invoices (BE-27).
- Stored as a versioned asset so Finance can replace it without a deploy.

### BE-24 — Automatic engineer invoice submissionFinance asked whether the engineer can submit an invoice directly from the portal, since the portal already captures the hours. **The system auto-submits it on their behalf** at the end of the last day of the month — the engineer never files an invoice manually.

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

---

## A6. SFM integration (Shireburn)

> **This is the dependency that gates the whole of A5.** Neither month-end report can be produced without it.

**SFM** is the accounting system supplied by **Shireburn**, and it is the system of record for supplier payments. The platform we are building must integrate with it the same way **the timesheet system it replaces already does** — that existing sync is the closest available reference implementation, and the integration should be scoped against it rather than designed from scratch.

### INT-11 — What the integration must do

| Direction | Payload | Used by |
|---|---|---|
| **Read** | Supplier reference data — SFM account number, nominal account, VAT code, payment type, IBAN, BIC, company name | Populates supplier columns in both reports (BE-23) |
| **Read** | Nominal account codes per client — entity code, subscription NAC, overage NAC | Populates client columns (BE-23) |
| **Write** | Supplier invoice batch — the 22-column upload defined in BE-22 | The month-end run |

### BE-26 — Integration requirements

**Acceptance criteria**
- **Audit the existing timesheet-system sync first.** Document how it authenticates, what transport it uses (file drop, API, direct DB), how batches are numbered, and how failures are surfaced. Build to the same contract unless there is reason not to — Finance's operational process is already built around it.
- Reference data is **read at export time, never copied into the portal**. A stale local cache of nominal codes will produce a silently wrong upload.
- The batch write is **idempotent**. Re-running a month's export must not double-post. `BATCH` numbering has to be reconciled with whatever SFM already expects — the sample shows `BATCH=26`, which suggests a running sequence owned by SFM rather than by us.
- A missing mandatory reference field **blocks that row** and is named in the month-end email body, rather than exporting a blank cell that fails on import.
- The generated file validates against a **real SFM import in a non-production company** before the first live run.
- Both reports are also downloadable on demand from Internal → Channel & Billing, so Finance is never blocked waiting for the scheduled run.

### Open questions for Finance — resolve before build

These four cells in the supplied template hold bare integers whose meaning is not self-evident. Each is likely an SFM internal reference code:

| Column | Sample value | Question |
|---|---|---|
| `DUEDATE` | `21` | Payment-terms days, or a code? Not a date in the sample, unlike `TRANDATE` |
| `PAYTYPE` | `24` | SFM payment-method code — which value maps to SEPA, Priority, USD? |
| `PAYEE` | `29` | An SFM payee ID rather than a name — where is it looked up? |
| `ANALYSIS` | `25` | What dimension does SFM expect here? |

Also confirm: `ENTITYCODE`, `TRANTYPE`, `INTREF`, `DETAILS`, `DISCTYPE` and `DBCR` were **blank** in this template but populated in an earlier version. Which are genuinely optional, and which were simply not filled in the sample?

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
| Daily work log summary | `humancapital@` + `sharedservices@` | **Once daily at 08:00 CET**, covering everything submitted the previous day — one row per entry: engineer full name, email, bench name, client name, hours logged, plus an entry and hours total. **Not** one email per entry |
| Entry exceeds capacity → approval required | `humancapital@` + `sharedservices@` | **Distinct template** from the above. Must carry enough for a reviewer to decide without opening the dashboard: **hours requested**; the engineer's **assigned hours** this period, **hours already logged**, and **hours needing approval**; and the client position — **capacity plan**, **overages agreed**, **overages used**, **overage rate**, and bench capacity used. "Overages agreed" resolves per BE-13: the authorised total where one is set, otherwise the standard 20% tolerance |
| Entry approved | Engineer | Confirmation |
| Entry declined | Engineer | Carries the decline message verbatim |
| Bench invite sent | Invitee | Link to the bench |
| Admin ownership transferred — to the **new Admin** | Incoming Admin | Confirms they now hold the account: subscription ownership, billing responsibility, full channel access, ability to invite Managers and Viewers |
| Admin ownership transferred — to the **outgoing Admin** | Previous Admin | Confirms the transfer, names who it moved to, and states they are now a Manager |
| Admin ownership transferred — notice | `sharedservices@castillians.com` | Audit record: client, previous Admin, new Admin, timestamp — the subscription's billing contact has changed |
| Weekly satisfaction report (per bench) | Manager | Sent **weekly**. Traffic-light colour **plus** hours used vs total for the bench — e.g. "68 / 80h — 85%". Three equal-width rating CTAs; no reply is treated as a green |
| Satisfaction response received | **Human Capital team** (`humancapital@castillians.com`) | Fires the moment a Manager taps a rating. States the client's response, the bench, the client, who responded, and the hours context at the time of the rating |
| 90% capacity, overages OFF | Internal + Manager | Logging will be blocked at the plan (BE-13). **Two separate templates** — the client's carries no overage mechanics and invites a capacity request; Castillians' keeps the hard-cap detail |
| 100% capacity, overages OFF | Internal + Manager | Work now blocked — switch overages on or raise the plan |
| 100% capacity, overages ON | Internal only | Overage has started; client not alerted |
| 120% capacity, overages ON | Internal + Manager | **Only when no Total Authorised Overage is set** — the 20% tolerance is then the ceiling. Entries now queue for approval. **Two separate templates** — the client's omits the auto-accept ceiling and internal mechanics, and invites authorising more overage |
| Total authorised overage used up | Internal + Manager | **Only when a Total Authorised Overage is set** — a named block supersedes the tolerance, so the ceiling is plan + authorised and the 120% notice does not fire. **Two separate templates**, same split as above |
| Unlimited overage | — | **No capacity notifications at all** |
| Month-end engineer invoicing | **Shared Services** | Sent on the **last day of the month**, once every engineer invoice has been auto-submitted (BE-24). Attaches `engineer-invoicing-YYYY-MM.xlsx` — one row per **engineer × Virtual Bench**, carrying engineer name and email, bench name, client name, bench manager(s), hours logged, engineer-facing rate, and earnings, plus a per-engineer total across their benches. Excluded entries (still awaiting approval at cut-off) are listed in the body and carry to the next run |
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
