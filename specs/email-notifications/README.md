# Email Notifications — spec folders

Epic **[SD-3492](https://castille-labs.atlassian.net/browse/SD-3492)**. **39 transactional email templates** across seven stories.

> **Reading `BE-nn`.** Stories and specs cite platform rules as **BE-01 … BE-30** — these are **numbered sections in `specs/ENGINEERING-BRIEF.md`**, not the `BE.md` files in spec folders. A `BE.md` is the backend spec for one story; a `BE-nn` is a platform-wide rule in the brief. Same two letters, two different things.

Flow stories in the other epics name the emails that fire within them, **for awareness only** — recipient and timing, not copy. These folders own the templates.

| Folder | Jira | Templates |
|---|---|---|
| `SD-3493/` | [SD-3493](https://castille-labs.atlassian.net/browse/SD-3493) | The shell and the rules every template follows — **build first** |
| `SD-3494/` | [SD-3494](https://castille-labs.atlassian.net/browse/SD-3494) | Work log approvals & timesheet reminders — 8 |
| `SD-3495/` | [SD-3495](https://castille-labs.atlassian.net/browse/SD-3495) | Capacity & overage notifications — 9 |
| `SD-3496/` | [SD-3496](https://castille-labs.atlassian.net/browse/SD-3496) | Client requests & their acknowledgements — 8 |
| `SD-3497/` | [SD-3497](https://castille-labs.atlassian.net/browse/SD-3497) | Client feedback — 2 |
| `SD-3498/` | [SD-3498](https://castille-labs.atlassian.net/browse/SD-3498) | Month-end finance emails — 6 |
| `SD-3499/` | [SD-3499](https://castille-labs.atlassian.net/browse/SD-3499) | Access & ownership — 6 |

Each folder holds **FE.md**. There is no BE.md: these are templates and send rules, and the data each carries is owned by the flow story that fires it.

## Rules worth reading before writing code

- **SD-3493 first.** The shell is one artefact, not copied per template — a footer change must apply to all 39 without 39 edits.
- **CTAs deep-link to the page the email is about**, never a dashboard home. Signed out, the reader hits **their own** dashboard's sign-in and is redirected to the target (BE-30). **The deep link must survive the round trip** — this is the rule most often shipped broken.
- **Three sign-in pages, chosen by recipient**: internal `/internal-dashboard`, engineer `/login`, manager `/manager-login`.
- **Client-facing emails point at `customerexperience@`.** Human Capital owns the engineer-facing and recruitment flows. Getting it backwards sends a client to the wrong team, and they will not know.
- **Panels are semantic**: amber = something needs doing; white = context or reassurance; blue = a fact about the recipient's own account. At most one amber per email.
- **One event can need two templates.** Most capacity thresholds fire a client version and an internal version, and they say genuinely different things — never one template with a conditional block, which is how internal figures reach a client.
- **A missing preheader is a defect.** Without it the client scrapes the logo's alt text.
- **No unsubscribe, no notification-settings link, no emoji, no CMS hook.** All transactional.
- **Money carries its currency code and is never converted.** Mixed currencies mean one figure per currency (BE-09, BE-27).

## Build order

1. **SD-3493** — the shell. Everything else depends on it.
2. **SD-3494**, **SD-3495**, **SD-3499** — independent of each other.
3. **SD-3496**, **SD-3497** — need their flows' request and rating records.
4. **SD-3498** — last: it cannot be finished before **SD-3468** produces the attachments.

## Not here

The **Request a Capacity Quote** emails belong to the website epic **SD-3487**, in its own repository (`castillians-enquire-page`).
