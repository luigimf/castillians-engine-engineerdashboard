# Castillians Platform — design specs & prototype

Everything an engineer needs to build the Engineer Dashboard.

```
prototype/index.html                  the prototype — open in any browser, no build step
specs/ENGINEERING-BRIEF.md            whole-platform spec: 25 numbered BE rules, integrations
specs/engineer-dashboard/SD-3453-tabs-overview/FE.md     Angular code
specs/engineer-dashboard/SD-3453-tabs-overview/BE.md     endpoints, payloads, test cases
specs/engineer-dashboard/SD-3455-log-hours/
specs/engineer-dashboard/SD-3456-entries/
specs/engineer-dashboard/SD-3458-invoices/
```

Every path above is **stable**. Files are updated in place, never renamed or moved, so a Jira
ticket that points at `specs/engineer-dashboard/SD-3455-log-hours/FE.md` stays correct however many times the
content is revised.

---

## The prototype

`prototype/index.html` is self-contained — fonts, styles and assets are inlined, so it runs
offline with nothing to install.

**GitHub will not run it.** Click the file, then **Download raw file**, and open the saved copy
in a browser. Use the header toggle to switch between the **Engineer**, **Internal** and
**Manager** dashboards.

### What it is, and what to take from it

The prototype is a single HTML file on a small template runtime. Control flow is `<sc-for>` /
`<sc-if>` rather than `*ngFor` / `*ngIf`, styles are inline attributes, and state lives in one
logic class. It is **not** Angular and does not paste across.

What **is** authoritative in it:

- **Every value** — hex codes, font sizes and weights, padding, radii, the
  `300ms cubic-bezier(0.35,0,0.25,1)` easing, the capacity colour bands
- **All copy** — labels, placeholders, tooltips, toasts, validation messages, in final wording
- **The behaviour** — what happens at each threshold, which fields reset, what expands on click
- **The business rules** — pro-rated first period, the authorised-block-replaces-tolerance
  ceiling, the same-or-fewer-hours edit that must fire nothing

Write from the **FE doc**; keep the prototype open beside it.

---

## Stories

| Spec folder | Jira | Covers |
|---|---|---|
| `SD-3453-tabs-overview/` | [SD-3453](https://castille-labs.atlassian.net/browse/SD-3453) | Bench tabs & Overview — the `/work-log` page frame |
| `SD-3455-log-hours/` | [SD-3455](https://castille-labs.atlassian.net/browse/SD-3455) | Log Hours — the form |
| `SD-3456-entries/` | [SD-3456](https://castille-labs.atlassian.net/browse/SD-3456) | Entries — list, calendar, edit history |
| `SD-3458-invoices/` | [SD-3458](https://castille-labs.atlassian.net/browse/SD-3458) | Invoices — the `/invoices` page |

All under Epic [SD-3452](https://castille-labs.atlassian.net/browse/SD-3452).

Each folder holds `FE.md` (Angular components, service, models, SCSS) and `BE.md` (endpoints,
payloads, validation order, data model, test cases). The Jira ticket carries the **FE UX
requirements**; backend detail lives only in `BE.md`.

---

## Email notifications

Flow stories name the emails that fire within them, for awareness only — recipients and timing, not copy. Templates are specified separately in the **Email Notifications** epic.

## Build order

1. **SD-3453 backend first.** The subscription-period engine and capacity logic are shared
   infrastructure — the Internal and Manager dashboards read the same data. Critical path.
2. **SD-3453 frontend** — the page frame the others mount into.
3. **SD-3455, SD-3456, SD-3458** in any order.

Three rules are worth reading before writing code, because each is easy to get subtly wrong:

- **The pro-rated first period.** A subscription starting mid-month runs to the end of the
  *following* month at pro-rated capacity. Starting on the 1st still gets the two-month first
  period — it is not special-cased. (`SD-3453-tabs-overview/BE.md`, §2)
- **The authorised block replaces the tolerance.** A 160h plan with a 40h authorised overage
  accepts 200h, not 232h. (`SD-3455-log-hours/BE.md`, §3)
- **A same-or-fewer-hours edit fires nothing.** A description fix is not a new claim: no status
  change, no email. (`SD-3456-entries/BE.md`, §2)

---

## Angular conventions

This repo was empty when the FE docs were written, so their Angular follows standard conventions
rather than ours, and is written version-agnostically — NgModule and standalone registration are
both shown.

Once real source lands here, the FE docs should be re-pointed at our actual module layout,
naming and styling setup. Nothing else needs to move.

---

## Repo

https://github.com/luigimf/castillians-engine-engineerdashboard

Every Jira ticket links here and cites its spec folder by path.

## Updating this

When a spec or the prototype changes, the file is replaced at the same path. Jira tickets need no
edit — they point at folders, not at versions.
