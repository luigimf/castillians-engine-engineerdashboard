# Castillians Platform — prototype & handoff

Everything an engineer needs to build the Engineer Dashboard, without needing access to the
design tool.

---

## The prototype

**`prototype-standalone.html`** — one self-contained file, 2.1 MB. Fonts, styles, design-system
bundle and all assets are inlined, so it runs offline with no build step and no server.

Open it in any browser. Use the header toggle to switch between the **Engineer**, **Internal**
and **Manager** dashboards.

Ways to make it available to the team:

| Route | How |
|---|---|
| **Commit it** | Drop it in the repo — e.g. `/design/engineer-dashboard/` — and it is versioned alongside the code, readable by Claude Code in place |
| **Share the file** | Email or Slack it; it works on open, nothing to install |
| **Host it** | Any static host, or GitHub Pages straight from the repo |

### Reading the source

The prototype is a Design Component — a single HTML file on a small template runtime. Control
flow is `<sc-for>` / `<sc-if>` rather than `*ngFor` / `*ngIf`, styles are inline attributes, and
state lives in one logic class. It is **not** Angular and does not paste across.

What **does** transfer, and is authoritative:

- **Every value** — hex codes, font sizes and weights, padding, radii, the
  `300ms cubic-bezier(0.35,0,0.25,1)` easing, the capacity colour bands
- **All copy** — labels, placeholders, tooltips, toasts, validation messages, in final wording
- **The behaviour** — what happens at each threshold, which fields reset, what expands on click
- **The business rules** — pro-rated first period, the authorised-block-replaces-tolerance
  ceiling, the same-or-fewer-hours edit that must fire nothing

Use the **FE doc** as the code to write, and the prototype as the reference beside it.

---

## Story bundles

| Bundle | Story | Covers |
|---|---|---|
| `SD-3453/` | [SD-3453](https://castille-labs.atlassian.net/browse/SD-3453) | Virtual Bench tabs & Overview — the page frame |
| `SD-3455/` | [SD-3455](https://castille-labs.atlassian.net/browse/SD-3455) | Log Hours — the form |
| `SD-3456/` | [SD-3456](https://castille-labs.atlassian.net/browse/SD-3456) | Entries — list, calendar, edit history |

All three sit under Epic
[SD-3452](https://castille-labs.atlassian.net/browse/SD-3452).

Each bundle contains:

```
FE-*.md                    Angular components, service, models, SCSS
BE-*.md                    endpoints, payloads, validation order, data model, test cases
prototype-standalone.html  the same offline prototype
prototype/                 the uncompiled source, if you want to read it
README.md                  where to start on that story
```

---

## Repository

```
repo:   luigimf/castillians-engine
branch: main
```

Empty at the time of writing, so the Angular in the FE docs follows standard conventions rather
than yours. Once source lands, the docs should be re-pointed at your real module layout, naming
and styling setup.

---

## Build order

1. **SD-3453 backend first.** The subscription-period engine and capacity logic are shared
   infrastructure — the Internal and Manager dashboards read the same data. Critical path.
2. **SD-3453 frontend** — the page frame the other two mount into.
3. **SD-3455 and SD-3456** in either order.

Three rules are worth reading before writing any code, because each is easy to get subtly wrong:

- **The pro-rated first period.** A subscription starting mid-month runs to the end of the
  *following* month, at pro-rated capacity. A subscription starting on the 1st still gets the
  two-month first period — it is not special-cased. (SD-3453 BE doc, §2)
- **The authorised block replaces the tolerance.** A 160 h plan with a 40 h authorised overage
  accepts 200 h, not 232 h. (SD-3455 BE doc, §3)
- **A same-or-fewer-hours edit fires nothing.** A description fix is not a new claim: no status
  change, no email. (SD-3456 BE doc, §2)

---

## Full specification

`ENGINEERING-BRIEF.md` in the project root carries the whole platform — 25 numbered backend
rules (`BE-xx`), integration points (`INT-x`) naming the exact Zoho and Internal Dashboard
fields each value is read from, all 17 email notifications, and the month-end finance export.
The story bundles are extracts from it.
