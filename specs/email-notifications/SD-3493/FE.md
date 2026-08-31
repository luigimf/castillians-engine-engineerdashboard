# FE — The email shell and the rules every template follows

Build spec for **SD-3493**, under epic **SD-3492**. The shared shell and the rules the other six stories inherit. **Build this first.**

Nothing here is user-visible on its own. It exists so the six template stories are about **content**, not about re-solving Outlook.

---

## The shell

- **600px table**, centred on `#F4F4F4`; white card, `1px solid #E5E5E5`, `8px` radius.
- Header: **logo linking to `https://castillians.com`**, then a **40×3px red rule** (`#D01329`), then the `h1`.
- **Arial/Helvetica throughout**, `mso-line-height-rule:exactly` on every text cell. **Not** Bricolage or Montserrat — webfonts are unreliable in email and Outlook substitutes unpredictably. This is the one place the design system's type is deliberately not used.
- `h1` 30px/36px bold, `-0.5px` tracking, `#141313`. Intro 16px/26px `#787878`, key nouns bolded to `#141313`.
- **Inline styles only.** No external stylesheet, no `<link>`, no class-based layout beyond the responsive block.
- One responsive block below **620px**: table `100%`, gutters `24px`, `h1` 26px/32px.
- Footer: hairline rule, then 12px `#9A9A9A` naming who it was sent to and why, then the company line.

## The preheader

- **Every template**, as the **first element in the body**.
- Says something the subject does not — the figure, the name, the consequence.
- Hidden with the full set (`display:none`, `visibility:hidden`, `opacity:0`, zero size, `mso-hide:all`); no single property works everywhere.
- **A missing preheader is a defect** — the client scrapes the first visible text, which is the logo's alt text.

## Label/value rows

Label 13px `#787878` at `width="200"`; value 15px bold `#141313`, **right-aligned**; `1px solid #E5E5E5` beneath. Fixed label column so figures line up down the table.

## Panels — three, with fixed meanings

**Semantic, not decorative.** A reader who gets a dozen of these a week learns the colours; using them loosely destroys that.

| Panel | Fill / border / text | Means |
|---|---|---|
| **Amber** | `#FEF6E7` / `#F0D9A8` / `#7A5A12` | **Something needs doing** — an action, a deadline, a consequence |
| **White** | `#FFFFFF` / `#E5E5E5` / `#787878` | Context or reassurance — visible, deliberate, not urgent |
| **Blue** | `#F2F9FF` / `#BFE2FF` / `#0A4E7A` | A fact about the recipient's **own account** |

**At most one amber panel per email.** Two competing urgencies means neither is read.

## CTAs — BE-30

One primary CTA: `#141313` fill, 4px radius, 15px/28px padding, white bold 16px.

- **Deep-links to the page the email is about** — the bench, the entry, the filtered queue, the invoice. **Never a dashboard home.**
- Signed out, the reader lands on **their own** dashboard's sign-in page and is redirected to the target on success:

| Recipient | Sign-in page |
|---|---|
| Internal teams | `https://castillians.com/internal-dashboard` |
| Engineer | `https://castillians.com/login` |
| Manager / Admin / Viewer | `https://castillians.com/manager-login` |

- **The deep link must survive the round trip.** Landing on the dashboard home after signing in is the failure this rule exists to prevent, and the one most often shipped.
- The email carries the **full deep link**; the sign-in hop is the app's doing, not a second `href`.
- **Exceptions**, specified where they occur: invitation links carry a single-use token to sign-up (SD-3499), and the satisfaction rating CTAs record a response without a session (SD-3497).

## Money

Currency **codes**, never symbols — `EUR 1,234.56`, exact cents. **Never converted**; an email summing across currencies shows **one figure per currency** (BE-09, BE-27).

## What no template has

- **No unsubscribe and no notification-settings link.** All transactional; offering an opt-out implies there is one.
- **No emoji**, per the brand.
- **No tracking pixel** unless Legal signs it off separately.
- **No CMS hook.** Transactional copy is not editable outside a deploy.

## Testing

- **Outlook (Windows), Apple Mail, Gmail web, Gmail iOS.** A template that works only in Gmail is not done.
- Dark mode checked; `color-scheme` and `supported-color-schemes` metas set.
- **Every template's recipient list asserted in a test.** A wrong recipient is invisible in production until someone complains.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| The shell | This story | All 39 templates |
| CTA targets and sign-in pages | **BE-30** | Every template with a CTA |
| Money formatting | **BE-09** | Every dashboard and report |
| Recipient addresses | **BE-20** | Every template |

**Acceptance criteria**

- **The shell is one artefact**, not copied per template. A footer or header change applies to all 39 without 39 edits.
- Every figure in an email **matches the surface it came from**. An email that disagrees with the dashboard is a defect in the email.
- **A send failure is logged and alerted on**, never swallowed.

---

## Reference

```
../SD-3494/ … ../SD-3499/     the six template stories
../../ENGINEERING-BRIEF.md    BE-09 money, BE-20 triggers and recipients, BE-30 CTA rules
../../../emails/              all 39 templates
```

**Preview:** the `Castillians Emails` canvas. `08-capacity-120-on.html` shows the amber panel, `06-satisfaction-weekly.html` the white one, `11-bench-invite.html` the blue one.
