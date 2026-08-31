# FE — Access & ownership

Build spec for **SD-3499**, under epic **SD-3492**. **6 templates.**

| Template | To | Fires |
|---|---|---|
| `11-bench-invite.html` | Invitee — **no account** | Invited to a bench |
| `14-org-invite.html` | Invitee — **no account** | Invited to a brand from the Organisation page |
| `11-bench-access-granted.html` | **Existing member** | Given access to a bench |
| `12-admin-transfer-new.html` | Incoming Admin | Ownership transferred |
| `13-admin-transfer-previous.html` | Outgoing Admin | Ownership transferred |
| `14-admin-transfer-audit.html` | **CX + Shared Services** | Ownership transferred |

---

## Shared rules

Everything in **SD-3493** applies: the 600px Arial shell, the hidden preheader, label/value rows, the three panel meanings, BE-30 CTA and sign-in behaviour, money formatting, no unsubscribe, and the four-client test pass. This document covers only what is specific to these templates.

## Two paths, two templates — never one

Adding somebody to a bench has **two outcomes**, and conflating them is the defect this split exists to prevent.

| The person | What happens | What they get |
|---|---|---|
| **No account** | An invitation is created; nothing exists until they onboard | The invite — sign-up link and token |
| **Already a member** | **Access is granted immediately.** No invitation, no token, nothing to accept | The access-granted email — links straight to the bench |

- **Never send a sign-up link to someone who already has an account.** It points them at a page they cannot use, and reads as though we have lost their account.
- For an existing member the **grant is live before the email lands** — it announces access rather than offering it.

## The invitations

- CTA lands on **`https://castillians.com/manager-sign-up?invite=TOKEN`** — never a login. The invitee has no account to log into, so a login screen is a dead end.
- **The Work Email field is pre-populated with the invited address and locked.** Read-only and visibly fixed, with a line saying why.
- Resolved **server-side from the token**, never from an editable query parameter. A tampered address is refused, not silently corrected.
- The email says the account is **tied to that address and filled in for them** — it does not warn them off typing a different one, because they cannot.
- **The token carries the grant**: role and bench access are applied on completion. They never choose either.
- Single-use and expiring; a spent or expired token gives a state the page can explain, not a generic error.
- **Never `admin`** — ownership moves only through a transfer.

## Access granted to an existing member

- Names the role held **on that bench**, and states their role on every other bench is **unchanged**.
- **Manager and Viewer are per bench** (SD-3479): someone made a Viewer here may still be a Manager elsewhere, and a mail implying otherwise reads as a demotion.
- CTA links straight to the bench.

## Ownership transfer — three emails, one event

- **Incoming Admin**: confirms they now hold the account — subscription ownership, billing responsibility, full channel access, and the ability to invite Managers and Viewers.
- **Outgoing Admin**: confirms the transfer, names who it moved to, and states they are **now a Manager** — they keep their bench access, and it must not read as removal.
- **Audit notice** to **CX and Shared Services**: client, previous Admin, new Admin, timestamp. **Both teams** — a transfer changes who CX deals with on the account, and it changes the billing contact Shared Services holds.
- All three fire from **one event, in one run**. Two arriving without the third means the transfer is half-announced.

## Role and account facts

- Each states a fact about the recipient's own account, so the **blue panel** is the right treatment.
- **There is exactly one Admin per account.** No email may imply two people hold it, even transiently during a transfer.

---

## Integration & sync

| Value | Source of truth | Also appears on |
|---|---|---|
| Bench access and role per bench | Bench membership (SD-3471, SD-3479) | The bench's Members with access, the member's bench list |
| Who the Admin is | The account record (§A4) | Organisation page visibility, subscription ownership |
| Pending invitations | Invitation record + single-use token (SD-3471) | The **Invited** state on the Organisation page |
| Email domains | Zoho client record, **per brand** (§A4, INT-10) | Which addresses may be invited at all |

**Acceptance criteria**

- **An existing member never receives a token**, and a person with no account never receives a bench deep link. The branch is decided on the invitee's address.
- The role named is **the role on that bench** — the same value the bench page serves.
- An invitation is only sent to an address on **that brand's own** domain allow-list; domains are never pooled across the channel.
- **A transfer's three emails are one atomic set.** A partial send is worth alerting on — the outgoing Admin learning by losing access is not acceptable.

---

## Reference

```
../SD-3493/                        the shell and shared rules
../../manager-dashboard/SD-3471/   the two paths, in full
../../manager-dashboard/SD-3479/   per-bench roles
../../ENGINEERING-BRIEF.md         §A4 roles, BE-20 triggers, BE-22 invite journey, BE-30 CTAs
```
