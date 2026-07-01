# RALPH 

---

## The short version

1. **It can't fully do its main job yet** — it can't take online payments, can't email clients, and can't tell you whether an event made money.
2. **A few safety gaps** could let the wrong person see or change records, or expose the system to online attacks. None are disasters, but they should be closed before it handles real client and payment data.

---

## 1. What's missing (things that were planned but aren't there)

- **No online payments.** The system can only log cash and check payments by hand. Credit-card/online payment (Stripe) is started but not actually working — clients can't pay through the system.
- **No emails or texts to clients or staff.** Every reminder and notification stays *inside* the app. If a designer misses a proposal deadline, the "escalation" never actually reaches anyone by email or text. This undercuts the whole point of the automatic reminder system.
- **Clients can't actually sign proposals.** Clients view a proposal through a public link, but the "sign" button only works for logged-in staff. So in practice, a client can look but not sign.
- **A built-in input-validation feature isn't turned on.** The system already generated tools to double-check incoming data, but they aren't being used. The safety mechanism exists in the box — it just isn't plugged in.
- **Basic web protections aren't installed**).
- **A settings/notes placeholder was left blank** — minor, but a sign of unfinished setup.

---

## 2. What needs fixing (things that work, but not reliably)

- **Bad web addresses can confuse the system.** If someone types or lands on a malformed record link, the system doesn't cleanly reject it — it can pass along "not a number" into a database lookup instead of simply saying "not found." This happens in several places.
- **Deleting or editing something that doesn't exist looks like success.** In some cases, if you try to change a record that isn't there, the system reports "OK" instead of "that doesn't exist." That can mask mistakes.
- **The daily reminder job has no backup plan.** The reminder engine runs once every morning at 9am. If the system is down at that moment (e.g., during an update), that day's reminders are simply skipped. If it accidentally runs twice, people might get duplicate notifications. This needs a guardrail so reminders neither get missed nor doubled.

---

## 3. Security issues

_None are catastrophic, but the first two should be addressed before the system handles real client or payment data._

- **HIGH — Anyone logged in can access almost any record.** Most parts of the system only check *whether* you're logged in, not *whether you're allowed*. That means a receptionist could, in principle, view, edit, or **delete** proposals, payments, or client records they shouldn't touch. Access should be limited by role and ownership, not just "is this person signed in."

- **HIGH — The system is too trusting of outside websites.** Its current settings let other websites make requests using a logged-in user's session, and there's no protection against a common trick where a malicious site acts on a user's behalf without their knowledge. The fix is to restrict access to the app's own official web addresses and add anti-forgery protection for anything that changes data.

- **MEDIUM — No limits on repeated attempts, and missing standard web safeguards.** There's no "slow down" mechanism to stop someone from hammering the system with automated guesses, and some standard protective settings aren't installed. The public proposal links use very strong, hard-to-guess codes (good), but adding request limits is still basic hygiene.

- **LOW — Login errors are slightly too revealing.** The sign-in process gives away a small hint about whether an account exists. Minor, and acceptable for an internal tool, but worth noting.

---

## 4. Business feature gaps

_The technical foundation is solid; these are missing **business capabilities** a florist / event-design company actually needs day to day._

### Money & profitability — the biggest gaps
- **You can't tell if an event made money.** The system tracks what you *charge* for flowers, but not what they *cost you* (wholesale price) or the labor involved. So it can't answer "was this event profitable?"
- **No real invoicing.** A proposal isn't an invoice. There are no invoice numbers, client receipts, or sales-tax reporting. Deposits and balances are loosely tracked.
- **Payment handling is shallow.** There's no structured deposit → 2nd payment → final balance plan, no automatic payment-due reminders, and no refund/cancellation-fee handling. Clients still can't pay online.
- **No accounting export** (e.g., to QuickBooks) and no clear view of deposits owed vs. revenue earned.

### Client communication
- **The system can't email clients at all.** "Send proposal" creates a link but never actually emails it. No reminders, thank-you notes, or receipts.
- **Client records are thin** — just name, email, and phone. No address, company, tags, communication history, preferences, or important dates. No repeat-client or referral tracking.
- **Contracts aren't truly binding.** "Signing" only records a timestamp — no captured signature, audit trail, cancellation policy, or signed PDF.

### Operations / production (florist-specific)
- **No flower-sourcing workflow.** The product list is a price sheet, not real inventory. No vendor management, no purchase orders to suppliers, and no per-event flower "pull sheet" — the operational heart of a florist.
- **No production or crew planning.** No design briefs, delivery run sheets, or staff scheduling. The calendar shows events but doesn't assign people to them.
- **No task lists or checklists** beyond the proposal-deadline reminder (e.g., "order flowers, confirm delivery, final walkthrough").
- **No file attachments per deal** — no place for mood boards, inspiration photos, venue floor plans, or contracts.

### Growth / getting new business
- **No online inquiry form.** Every lead is entered by hand; there's no public form feeding new leads in.
- **No reusable proposal templates** — every proposal is built from scratch.
- **No after-the-event follow-up** — no review/testimonial requests, portfolio capture, or referral tracking (important for a referral-driven luxury florist).
- **No planner referral/commission tracking** — planners are just a contact list, even though they often earn referral fees.

### The 5 most valuable things to build next (business priority)
1. **Cost & profit tracking** per product and per event.
2. **Real online payments** with structured installments and automatic payment reminders.
3. **Client email** — send proposals, reminders, and receipts.
4. **Flower sourcing** — purchase orders and per-event pull sheets.
5. **Online lead-intake form** that feeds new inquiries into the pipeline.

---
