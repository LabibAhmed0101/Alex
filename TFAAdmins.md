# The Three Admins — What Each One Can Do

How Yosef, Ofir and Zack differ from each other. Based on spec §2.1–§2.3 plus the answers
we got on the our open questions.

The important thing to read first is the last section: **most of what separates the three
admins is not built yet.**

---

## At a glance

| | Yosef Levy | Ofir Roth | Zack Deutch |
|---|---|---|---|
| Role in the system | `admin1` | `admin2` | `admin3` |
| Title | Owner / Boss | Partner + Manager | Partner |
| Display name | Yosef | **Evan** | Zack |
| Login | `yosef@topflooradvance.com` | `evan@topflooradvance.com` | `zack@topflooradvance.com` |
| House take | 10% | what is left of the 80% | 10% |
| Also works his own deals | No | **Yes** | No |

---

## What only Yosef can do

These five are his alone. Everything else on this page, the other two share.

- **Change passwords** — anyone else's, that is. All three can change their own.
- **Modify the house split structure** — the 10% / 10% arrangement itself.
- **Reassign a user account to a new hire**, so the new person inherits the leads, notes,
  calls, documents and submission history of whoever left.
- **Delete data**, other than duplicates.
- **Read and export the audit log.** Added 20 August; it was previously all three.

He also sees **every** login in the Logins section, including the other two admins'. Ofir
and Zack see only the brokers' logins, never each other's or Yosef's.

## What is specific to Ofir

Ofir is the only admin with a second job, and it changes how the system treats him.

- **He is also an active broker**, working his own book alongside everyone else's. He
  appears to the team as **Evan**.
- **He runs the sales floor** — submits every broker's deal to the lenders (marks them
  Shopped Out), sells the brokers' deals, holds the lender relationships, and updates
  Reverse Consolidation tranches weekly.
- **His commission is a remainder, not a fixed percentage.** Yosef and Zack each take a
  flat 10%. Ofir takes whatever is left of the remaining 80% once the broker on the deal
  has taken their split — so his share moves with the broker's tier, and is 80% on a deal
  he worked himself.
- **Split deal Rule 2 is specific to him:** when Ofir invites a broker onto a deal, the
  deal takes on that broker's structure.
- **Anyone can book a call on his calendar.** Not true of the other two.

## What is specific to Zack

Zack has the narrowest role of the three: full view and edit across the whole system, a
flat 10% of the house take, and none of Yosef's owner powers. In everyday use he and Ofir
have the same permissions — what separates them is that Ofir also carries a broker's
workload and a manager's responsibilities, not that either can reach something the other
cannot.

## What all three share

- Full view and edit access across every lead, broker and deal in the system.
- Mark and unmark commission, PSF and syndication as **Paid** — every change logged.
- Mark deals **Funded** and **Approved**, and reverse a Funded deal.
- Trash duplicates. This is the one deletion that is not Yosef's alone.
- Toggle a broker's **Away** status and reorder the round-robin sequence.
- Exempt from the first-login Team Agreement — that is a broker requirement only.
- Payout every Friday.

---

## What is actually built today

This is the part worth knowing before handing out logins.

**On permissions, the three admins are currently identical.** Every permission check in the
system runs through one list:

```ts
const ADMIN_ROLES = ["admin1", "admin2", "admin3"];   // lib/policy.ts:17
```

Nothing anywhere distinguishes Yosef from Ofir from Zack when deciding whether an action is
allowed. In practice:

| Power | Status today |
|---|---|
| Trash duplicates | ✅ All three — matches the spec |
| Mark Paid (commission / PSF / syndication) | ✅ All three — matches the spec |
| Mark Funded / Approved, reverse a Funded deal | ✅ All three — matches the spec |
| Read the audit log | ✅ **All three can. ** |
| Change another user's password | ⚪ Not built — nobody can, Yosef included |
| Modify the house split | ⚪ Not built — nobody can, Yosef included |
| Reassign an account to a new hire | ⚪ Not built — nobody can, Yosef included |
| Delete data beyond duplicates | ⚪ Nothing else in the system is deletable, so this has never come up |
| Logins section | ⚪ Does not exist in any form |

So the three "only Yosef" restrictions on passwords, the house split and account
reassignment are satisfied today only because **the features themselves do not exist**.
Ofir and Zack cannot use them — but neither can Yosef. The audit log is the one that is
actively wrong rather than merely absent.

**The single practical consequence:** an admin login handed to Ofir or Zack today carries
exactly the same power inside the app as Yosef's. What is meant to separate them is the
four owner-only powers, and none of the four is implemented.

Closing that gap is one work order. It was blocked on deciding how the system should
recognise "the owner" in the first place, since matching on somebody's name is not
acceptable; that was settled on 24 August as an `is_owner` flag on the user record, so the
work is now unblocked.

## Where the three genuinely do differ today — money

The commission engine does tell them apart, and does it correctly:

```ts
ADMIN1_BPS = 1_000                      // Yosef — flat 10%
ADMIN3_BPS = 1_000                      // Zack  — flat 10%
admin2Bps  = POOL_BPS - ownerTierBps    // Ofir  — the remainder of the 80%
```

One consequence worth knowing before either of them takes a lead directly: the engine
treats **a lead held by Ofir** as his own deal with no broker participant, which is a case
the commission rules cover. **A lead held by Yosef or Zack is rejected outright** — the
commission rules define no split for that situation, so rather than guess at one, the
system refuses to fund it.

Admin login alerts also go to Yosef alone, not to all three.
