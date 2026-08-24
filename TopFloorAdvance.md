# What's Built and Working

List of what is **finished** in the Top Floor Advance CRM — no dev jargon,
written for Yosef, Ofir and Zack rather than for a developer.

---

## Logging in and accounts

- Everyone has their own login and their own password.
- Five wrong password attempts locks that account for 15 minutes, whether or not the next
  attempt is correct.
- You can change your own password from inside the system, without asking anyone.
- Four kinds of account exist — Yosef, Ofir, Zack, and brokers — and each one sees a
  different version of the system.
- Brokers are asked to sign the team agreement before they can use the system, and their
  typed name, the date, and where they signed from are all recorded.

## Who can see what

- **Brokers only ever see their own leads, pipeline, funded deals and commissions.** Never
  another broker's. This is enforced on the server, on every page and every action, so it
  cannot be worked around by anyone poking at the system.
- Admins see everything.
- The rules live in one single place rather than being repeated screen by screen, and there
  is an automated check that refuses to let new code ship if someone tries to write their
  own version of a permission rule. This is what stops one broker's commissions leaking to
  another as the system grows.
- Social Security Numbers show as the last 4 digits only. Seeing the full number takes a
  deliberate click, and every one of those clicks is logged with who did it.

## Leads

- Create a lead by hand, or import a whole spreadsheet of them at once.
- Full lead profile page with the business details, contacts, notes, documents, status
  history and submission history all in one place.
- A lead that is missing its business name or a phone number goes to **Missing Info** and is
  not handed to a broker until it is complete. Fill in what is missing and it joins the
  rotation automatically.
- Every status change is recorded with the date, so any lead's full history is visible.
- Mark a lead Do Not Call.
- Flag a lead as a duplicate, with a reason.
- Multiple phone numbers and multiple email addresses per contact, for both the CEO and the
  CFO — not one of each.
- Correcting or removing a phone number or email does not destroy anything; the old detail
  stops being used but the record survives.

## Round robin — automatic lead assignment

- New leads are handed out to brokers automatically, in order, working properly through the
  full list rather than piling up on one person.
- **Brokers cannot assign a lead to themselves.** Only an admin can override who gets a lead.
- A broker marked **Away** is skipped but keeps their place in the order.
- Imported leads go through the same rotation as leads entered by hand.
- Two leads created at the same instant cannot accidentally land on the same broker.

## Pipeline

- Two ways to look at the pipeline: a **list view** and a **kanban board**.
- Search, filter by status, and sort in both.
- Change a lead's status straight from either view without opening the lead.
- The three admin-only statuses — Shopped Out, Approved, and Funded — can only be set by an
  admin.

## Lenders

- Add, edit and deactivate lenders, with their tiers, position and notes.
- A deactivated lender drops out of the lists without being deleted, so old deals that used
  them still make sense.
- Duplicate lender names are refused cleanly.
- Every change is logged with what it was before and what it became.

## Funding a deal

- The **Funded popup** captures the lender used, funding amount, factor rate, term, payment
  frequency and amount, the commission, and the PSF.
- Funding a deal automatically works out and records everyone's commission — nobody
  calculates anything by hand.
- The deal lands on the **Master Funded Sheet** for admins, and on the broker's own Funded
  page for the broker.
- A funded deal can be **reversed**. Everything unwinds together in one go — commission
  records, the funded sheet entry, the renewal timer, and the PSF — and the deal is marked
  reversed rather than deleted, so the history stays.

## Commission

- The full commission calculation is built and tested: the house take (Yosef 10%, Zack 10%,
  and Ofir taking what is left of the remaining 80% after the broker's own split).
- **Every split works out to the exact cent.** The individual shares always add back up to
  the total exactly — never a penny missing, never a penny extra. Any leftover cent goes to
  the deal's broker.
- The Commission page shows each person what they are owed, broken out by deal.
- Admins mark a commission line as **Paid**, and can unmark it if something goes wrong.
  Both directions are logged.
- Brokers see only their own commission lines, never anyone else's, and never the
  company-wide totals.
- The money math alone has **53 automated tests** behind it, written from the worked
  examples in your specification.

## Split deals — two or more brokers on one deal

- The original broker invites other brokers onto a deal. Each one accepts or declines.
- **No limit on how many brokers** can be on a deal, and only the original broker can invite.
- The inviting broker's own split divides equally among everyone on the deal — 50% between
  two is 25% each, between three is 16.67% each. The house side never changes no matter how
  many brokers are involved.
- Once a broker accepts, they can see and work that lead like their own.
- Funding a split deal pays each broker their share automatically.
- Pending invites are visible on the pipeline so nothing sits forgotten.

## PSF

- Every funded deal carrying a PSF is marked **Pulled**, **Cleared**, or **Blocked by
  Merchant**, and all three admins can move it between those at any time, in either
  direction.
- The status is set **once on the deal**, not person by person — so "has this PSF cleared?"
  always has a straight answer, even on a deal shared between brokers.
- The PSF follows the identical split as the commission, Ofir's share included.
- Brokers see their own share of the PSF from the moment the deal funds, not only once it
  clears — but never the total on the deal.
- When a PSF clears, everyone owed a piece of it is notified, and so are all three admins.

## Notifications

- A notification bell with an unread count that updates on its own, without refreshing the
  page.
- **14 different events** raise a notification, including: a deal funded, killed, declined
  or approved; a split invite sent or answered; a broker removed from a split; commission
  marked or unmarked as paid; and a PSF cleared.
- Notifications can be marked as read or dismissed.
- Admins get their own admin-level notifications separately from the brokers'.

## Dashboard

- Submitted this month, funded this month, and a trend chart over time.
- Broker-by-broker performance for admins; brokers see their own numbers only.
- The monthly figures are worked out from when a deal actually changed status, not from a
  snapshot of where it sits today — so past months do not quietly rewrite themselves.

## Notes

- Notes on any lead, stamped with who wrote them and when.
- **Notes are permanent — nobody can delete one, ever, including an admin.**
- The author can correct their own note within 15 minutes of writing it. After that it is
  fixed. There is no admin override at any point.

## Documents

- Upload documents against a lead, rename them, and delete them.
- Only people who can see the lead can touch its documents.
- Every deletion is logged with who did it.

## The audit log

- Every action that matters records **who did it, what they did, and when**.
- The record and the change itself save together or not at all — they can never fall out of
  step, so there is no such thing as a change with no log entry, or a log entry for a change
  that did not happen.
- Status changes, edits, funding, reversals, commission payments, PSF changes, deletions,
  lender changes, and full-SSN views are all covered.
- There is an automated check that refuses to let new code ship if someone writes to the
  audit log any other way.
- A change to a lead's **source** is always logged, because it feeds who gets paid.

## Under the hood — things you would only notice if they were wrong

- **Money is stored as exact whole cents**, never as decimals. This removes an entire class
  of rounding drift that would otherwise creep into commission math over months.
- Percentages are stored the same exact way, so a 17.5% split stays 17.5% forever.
- All times are stored in one universal format and shown in Eastern Time, with daylight
  saving handled properly rather than assumed.
- **Over 400 automated checks** run against the system, concentrated on the highest-stakes
  areas: money, permissions, and the audit trail.
- Three automated guards refuse to let new code ship if it bypasses the audit log, the
  permission rules, or the notification system.
- The groundwork for scheduled background tasks is in place and tested, including the case
  where two copies of the server are running at once and could otherwise both fire the same
  reminder twice.
- Only approved websites can talk to the system, and the database schema is kept in step
  across the live system and the test copy, verified by direct query rather than by
  trusting a tool's "done" message.

---

## Finished, but needs a small adjustment

These are built and working, but a few of the answers that came back on **20 August** landed
after the code had already shipped, so they need a change to match. None of them is broken
in a way that is costing anything today.

- **The rotation timer.** The 7-day clock and the +3-day bonus days are recorded, but the
  bonus is currently added on every forward move rather than once per stage, moving into
  Submitted does not reset the clock to a fresh 7 days, and a lead held by Yosef, Ofir or
  Zack still gets a timer when it should not. Worth knowing: **nothing acts on these
  deadlines yet** — no lead is being taken off anybody — so the wrong dates are sitting
  unused rather than causing a problem.
- **The Funded popup's prefills.** "Lender used" starts blank instead of filling in from the
  approved offer, and the phone and email fill in from the first contact added rather than
  the one marked Primary.
- **Contacts.** The system currently allows one CEO and one CFO per lead; you have since
  told us a lead can have two of either.
- **Lender tiers.** They work, but renaming or retiring a tier is not yet something an admin
  can do from the screen.
- **The audit log.** All three admins can currently read it; you have asked for it to be
  Yosef only.
- **The team agreement.** The signing flow works end to end, but the agreement text in it is
  placeholder wording — we are still waiting on the real legal text before anyone signs it
  for real.
