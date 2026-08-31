# FE — Work log approvals & timesheet reminders

Build spec for **SD-3494**, under epic **SD-3492**. **8 templates.**

| Template | To | Fires |
|---|---|---|
| `01-worklog-daily-summary.html` | HC + SS | Daily 08:00 CET, previous day's entries |
| `02-approval-required.html` | HC + SS | An entry crosses 120% of the capacity plan |
| `02a-approval-reminder-3-days.html` | HC + SS | 3 days before cut-off, **only if** entries pending |
| `02b-approval-reminder-cutoff.html` | HC + SS | On the cut-off date, **only if** entries still pending |
| `03-entry-approved.html` | Engineer | An entry that needed review is approved |
| `04-entry-declined.html` | Engineer | An entry is declined |
| `07-timesheet-weekly.html` | Engineer | Weekly, naming unlogged days |
| `08-timesheet-cutoff.html` | Engineer | 3 days before the timesheet cut-off |

---

## Shared rules

Everything in **SD-3493** applies: the 600px Arial shell, the hidden preheader, label/value rows, the three panel meanings, BE-30 CTA and sign-in behaviour, money formatting, no unsubscribe, and the four-client test pass. This document covers only what is specific to these templates.

## Daily summary — one email, not one per entry

- **Once daily at 08:00 CET**, covering everything submitted the previous day. One row per entry: engineer, bench, client, hours. Entry and hours totals at the foot.
- **Never one email per entry.** A busy day would produce thirty emails and be ignored wholesale.
- **A day with no submissions sends nothing.** An empty summary trains people to skim.

## Approval required — a distinct template

The reviewer must be able to **decide from the email**. That is its whole design brief (BE-20).

- Carries **hours requested**; the engineer's **assigned hours** this period, **hours already logged**, **hours needing approval**; and the client position — **capacity plan**, **overages agreed**, **overages used**, **overage rate**, bench capacity used.
- **"Overages agreed" resolves per BE-13** — the authorised total where one is set, otherwise the standard 20% tolerance. Not a fixed figure.
- A **separate template** from the daily summary, not a variant of it.

## The two reminders

- Both fire **only when entries are actually awaiting a decision**. An empty queue sends nothing — this is what stops them becoming background noise.
- Both list every pending entry with engineer, bench, date and hours.
- The cut-off one uses the **red treatment** and states the **consequence rather than implying loss**: after 23:59 the entries bill in the following month. They are not destroyed, and the copy must not suggest they are.

## Approved and declined

- **Approved** confirms the entry will be billed this period.
- **Declined** carries the reviewer's message **verbatim** — never trimmed, never summarised. A decline message is mandatory (BE-13), so there is always one.
- The declined entry stays visible to the engineer and Internal, and **hidden from the Manager** (SD-3467, SD-3473). The email must not imply the client saw it.
- Both CTAs deep-link to `app.castillians.com/engineer/work-log`.

## Timesheet reminders

- **Weekly** to every activated engineer, naming **their own** benches and unlogged days.
- **Three days before cut-off**, flagged as closing soon: anything unlogged will not be invoiced.
- An engineer with **nothing outstanding is not emailed**.

## What must not fire

- **A same-or-fewer-hours edit fires nothing** (SD-3456). A description fix is not a new claim: no status change, no email. **The single most likely accidental over-notification in the epic.**
- Increasing hours past capacity **does** re-open review, and does notify.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| Entry status and hours | Approved work logs (SD-3456, SD-3467) | Engineer Work Log, Internal Work Logs, Manager bench page |
| Approval threshold | BE-13 — authorised total, else the 20% tolerance | Internal bench entry, the 120% notices |
| Cut-off date | Derived from the period rules (BE-02) | Every period label on all three dashboards |
| Decline message | The reviewer's own words | The entry on Engineer and Internal — **never** Manager |

**Acceptance criteria**

- Every figure **matches the dashboard** for the same entry and period.
- **Pending-approval hours are excluded** from any capacity figure quoted.
- Reminders read the **live pending queue at send time**, not a snapshot taken earlier in the day.

---

## Reference

```
../SD-3493/                   the shell and shared rules
../../ENGINEERING-BRIEF.md    BE-13 thresholds, BE-20 triggers, BE-21 reminders, BE-30 CTAs
```

> **`BE-nn` refers to numbered requirement sections inside `specs/ENGINEERING-BRIEF.md`** — e.g. **BE-30** is *"Email CTAs: deep link, sign-in hop, redirect back"*. They are **not** the `BE.md` files in spec folders, which are backend specs for a single story.
