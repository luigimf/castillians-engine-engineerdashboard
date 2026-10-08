# SD-3567: Weekly inactive engineers report

Jira **[SD-3567](https://castille-labs.atlassian.net/browse/SD-3567)** (Enhancement, epic SD-3531). Follows the shell and rules in **SD-3493**.

Template: `repo/emails/17-inactive-engineers-weekly.html`. Email prototype: `repo/prototype/emails/index.html` → **Timesheet reminders** → **Weekly inactive engineers report**.

## Send rule

| | |
|---|---|
| Recipient | `customerexperience@castillians.com` (config, not hardcoded) |
| When | Every **Monday 08:00 Europe/Paris** (CET/CEST, follows daylight saving) |
| Window | Previous **Monday 00:00 – Sunday 23:59**, same time zone |
| Frequency | Once per week; idempotent on retry |

## Who is listed

- One row per **engineer per Virtual Bench**: an activated engineer on a bench with an active subscription who logged **0 hours** on that bench in the window.
- An engineer who logged on one bench but not another is listed for the bench they missed.
- **Excluded:** engineers activated on the bench partway through the window; benches whose subscription started or ended partway through; engineers removed from the bench.
- Entries count whatever their status (auto-approved, approval required, approved). **Declined entries don't count.**

## Content

- **Subject:** `Engineers with no hours logged — week of <Mon date>` (e.g. "week of 28 Sep 2026").
- **Preheader:** `<n> engineers logged no hours between <Mon> and <Sun>.`
- **Heading:** `<n> engineers logged no hours last week`, with the date range in the intro.
- **Table**, three columns:
  - **Engineer**: name, with email beneath;
  - **Virtual Bench · Client**: bench name, with client beneath;
  - **Last entry**: date of the engineer's last entry on that bench, or "No entries yet".
- **Sort:** last entry, oldest first; "No entries yet" rows go at the bottom.
- **CTA:** **Open Engagements** → `/internal-dashboard/engagements`. Signed out, the reader goes through internal sign-in and is redirected there; the deep link must survive the round trip (BE-30).
- **Note under CTA:** "One row per engineer per Virtual Bench. An engineer who logged on one bench but not another is listed for the bench they missed."
- **Footer:** "Sent to Customer Experience every Monday at 08:00 CET."

## Empty state

If nobody is inactive, the email **still sends**, heading "Every activated engineer logged hours last week." and **no table**, so CX knows the job ran.

## Data sources

| Data | Source of truth |
|---|---|
| Engineer activation per bench | Platform (bench capacity allocation) |
| Bench subscription status | Platform subscriptions (synced from Zoho) |
| Client name | Zoho Parent Brand lineage |
| Work log entries | Platform work logs |

## Not this

Separate from the engineer-facing **Weekly timesheet reminder** (SD-3494). This one goes to Customer Experience only and is never sent to engineers or clients.
