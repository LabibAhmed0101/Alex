# Feature Audit — Everything That Isn't Cleanly Done

Code-verified against `main` @ `a58d56f` on 2026-08-18. The 30 items that matched their
description exactly are omitted; this file is only the 25 that need action or a correction.

| Category | Count | Meaning |
|---|---|---|
| [Done, but shows as not done](#1-done-but-the-list-shows-it-as-not-done) | 6 | Shipped and working — the status list is wrong |
| [Overstated](#2-overstated--built-but-less-than-described) | 7 | Built, but the description claims more than exists |
| [Partial](#3-partial) | 7 | Half the feature is real |
| [Not built](#4-not-built) | 5 | No code exists |

---

## 1. Done, but the list shows it as not done

Nothing in this section is a stub. Each has a working backend route, a wired-up UI, and in
two cases a running background worker. **No work required — only the status needs correcting.**

### #11 — "Send Contract" standalone action
*Listed as: not built*

`POST /contracts/:id/send` mints a share token, stamps the contract sent, enqueues the
signature-request email through the durable email worker, and advances the job stage. The UI
button reads "Send for signature" (or "Resend") on every non-signed contract.

- `artifacts/api-server/src/routes/contracts.ts:104-140`
- `artifacts/formhaus/src/components/contracts-panel.tsx:114-116`

### #12 — Automated payment / signature reminder emails
*Listed as: not built*

`invoiceReminderWorker` starts at boot and sweeps every 24h, enqueuing `invoice_reminder_d3`,
`_d7` and `_d14`. The sender skips anything paid, void, or missing a client email. Per-invoice
skip/reset routes exist.

Two caveats worth knowing:
- The offsets are **hardcoded**, not configurable.
- They run from the **issue date**, not the proposed "3 days before due / 1 day overdue /
  7 days overdue" schedule.

- `artifacts/api-server/src/lib/invoiceReminderWorker.ts`
- `artifacts/api-server/src/lib/gmail/send.ts:581`
- `artifacts/api-server/src/index.ts:127`

### #26 — Invoice version history with restore
*Listed as: not built*

Fully built, **including restore**. Every edit writes a complete snapshot (not a diff), so old
versions stay readable across schema changes. `GET /:id/versions` and
`POST /:id/versions/:versionId/restore` are both admin-only. The UI ships an admin "History"
button opening a panel with per-version Restore behind a confirm step.

- `artifacts/api-server/src/lib/invoiceVersions.ts`
- `artifacts/api-server/src/routes/invoices.ts:1559, 1577`
- `artifacts/formhaus/src/pages/invoice-detail.tsx:2590, 2847-2900`

### #37 — Inventory item detail modal, tabs, sort-by-stock
*Listed as: partial*

All three are built: tabs for All / Low Stock / Ordered / New Arrivals (the last two backed by
their own server queries), a sort dropdown with A–Z and stock ascending/descending, and a
click-through detail modal showing photo, stock status, brand, category, location, lead time,
item type and barcode.

Two small gaps against the original wording: there is no "Category" *sort* (it exists as a
filter instead), and the modal has no description field.

- `artifacts/formhaus/src/pages/inventory.tsx:27-29, 94-97, 194-225, 354-420`

### #43 — Dashboard redesign (Pickups, Due Today, Quick Actions, Low Stock)
*Listed as: partial*

All four named panels are built, each backed by its own server-side computation — plus the
Messages preview the list also mentions. Beyond those: Arriving from Overseas, Recent
Notifications, a six-month revenue chart, salesperson and lead-source breakdowns, and pipeline
health.

- `artifacts/formhaus/src/pages/dashboard.tsx:426-596`
- `artifacts/api-server/src/routes/dashboard.ts:57, 146-208`

### #45 — Internal messaging
*Listed as: not built (stretch goal)*

Built end to end. Three tables (`staff_rooms`, `staff_room_members`, `staff_messages`)
supporting DMs and group rooms with read tracking and alert flags, a 344-line API route, a
435-line Messages page, a sidebar nav entry carrying an unread badge, and a preview panel on
the dashboard.

- `lib/db/src/schema/staffMessages.ts`
- `artifacts/api-server/src/routes/staffMessages.ts`
- `artifacts/formhaus/src/pages/messages.tsx`
- `artifacts/formhaus/src/components/layout.tsx:89`

---

## 2. Overstated — built, but less than described

The feature exists and works. The description overshoots what's actually there.

### #1 — Client → Job → Invoice hierarchy
The hierarchy itself is solid: picking a client filters the job dropdown to that client's jobs,
picking a job back-fills the client, and the server auto-creates both a job and a paired draft
estimate when an invoice arrives without one.

**Not true:** you cannot create a new job from the invoice form. Only the new-*client* dialog is
wired in there; its `onCreateJob` hook is never passed. Jobs are created from Pipeline or the
Clients page.

- `artifacts/formhaus/src/pages/invoice-new.tsx:1794-1806, 1757`
- `artifacts/api-server/src/routes/invoices.ts:549-560`

### #4 — Invoice status: Draft → Sent → Paid → Closed
The stored status enum is `draft / sent / accepted / void`. There is **no "paid" and no
"closed" status**. "Paid" is derived from payments; "Closed" is
`paymentStatus === "paid" && signed`, computed in the page. The CLOSED badge and follow-up
invoice CTA both work.

**Not true:**
- Edit controls are *not* hidden on closed invoices — that was deliberately reverted, and
  editability now comes from the server edit policy.
- The manual status override explicitly **refuses** "paid" (*"Record a payment to mark an
  invoice paid"*).

- `lib/db/src/schema/invoices.ts:8`
- `artifacts/formhaus/src/pages/invoice-detail.tsx:1587-1606`
- `artifacts/api-server/src/routes/invoices.ts:1365`

### #8 — Rendered invoice view (styled PDF)
The PDF is real and good: line items, discount line, totals, Total / Paid / Balance Due, and an
admin margin block.

**Three details are wrong:**
1. The logo is one global business logo, **not department-specific**.
2. There is **no shipping breakdown line** — shipping is folded into per-line prices upstream,
   so it never appears on its own.
3. What shows is a paid **summary**, not a payment history list.

- `artifacts/api-server/src/lib/invoicePdf.ts:204, 360-385`
- `artifacts/api-server/src/lib/invoiceCalc.ts:19`

### #14 — Invoice templates, Save and Load
Save and Load both exist and are admin-only. Line items *are* written into the saved template.

**But loading never restores them.** Only notes, description, product label and shipping cost
are applied — the toast says so out loud: *"Line items are not auto-applied — add them
manually."* The most useful half of a template is stored and then discarded.

- `artifacts/api-server/src/routes/invoiceTemplates.ts`
- `artifacts/formhaus/src/pages/invoice-new.tsx:1722-1733`

### #15 — Configurable discount thresholds
**Nothing is configurable, and there is only one threshold.**
`DISCOUNT_APPROVAL_THRESHOLD = 15` is hardcoded twice in the invoices route. There is **no 30%
tier anywhere** — the PIN fires above 15% for admins, not above 30%. This shouldn't be built,
formhaus only suggested it without any confirmation that we should do it, if i remmber correctly.

### #31 — Catalog items linked to default supplier
`defaultSupplierId` exists on both system types and catalog products, with a supplier picker in
the catalog form.

**But it does not auto-populate overseas orders.** The overseas route never reads it. Its one
real consumer is supplier-scoped email, where it decides which invoice lines belong to which
supplier.

- `lib/db/src/schema/catalog.ts:80, 121`
- `artifacts/api-server/src/lib/gmail/send.ts:979`

### #42 — Dashboard calendar + recent activity
The recent-activity feed is live and correct.

**The calendar is not monthly and has no markers.** The dashboard card is "Calendar — Today",
showing only today's events. The monthly grid lives on the separate Calendar page, and it
renders only the six manual event types — no job, payment, or overseas-arrival markers anywhere.

- `artifacts/formhaus/src/pages/dashboard.tsx:598-620`
- `artifacts/formhaus/src/pages/calendar.tsx:44-51`

---

## 3. Partial

### #20 — Invoice list filters + sort options
The invoice list has exactly two controls: a status dropdown (all / draft / sent / accepted /
paid / void) and a date-range preset.

**Missing:** the Overdue status option, the client filter, the salesperson filter, the
lead-source filter — and there is **no sort control at all**. Not by date, number, client, or
total.

- `artifacts/formhaus/src/pages/invoices.tsx:61-72, 106-127`

### #23 — Salesperson + Lead Source on invoices
Both fields exist on the schema, on the new-invoice form, on the edit form, and on the detail
view.

**But both are plain free-text inputs.** There is no admin-managed salesperson picklist and no
Walk-in / Phone / Referral / Online / Other picker — that picker exists only on the Job dialog
in Pipeline. Neither field appears in the invoice list filters.

- `artifacts/formhaus/src/pages/invoice-new.tsx:1980-1986`
- `artifacts/formhaus/src/pages/invoice-detail.tsx:2366-2371`

### #24 — Activity logs per client, job, invoice
Three of the four surfaces are real: the global dashboard feed, the per-client "Journey" tab,
and the per-job timeline in the drawer.

**There is no per-invoice activity timeline.** The invoice page has an admin-only Version
History panel instead. The `invoiceActivity` schema module is exported but never queried by the
API — it is dead code.

- `artifacts/formhaus/src/pages/client-detail.tsx:735-745`
- `artifacts/formhaus/src/components/job-detail-drawer.tsx:33-52`

### #32 — Overseas Order record type
A lot is there: supplier assignment, a stage enum with manual admin override, estimated arrival,
admin-only total and unit cost, quantity, incoterm, freight reference, notes, a shipping stage
bar, QuickBooks expense sync, delay detection, and links back to the originating invoice, job
and client.

**Five claimed pieces are absent:**
1. Vessel name
2. Container number
3. An itemized line list — the order carries one description and one quantity
4. Payments-made-to-supplier tracking
5. The "Show Invoice" link to the supplier's invoice

An itemized list *does* exist — but on the separate `purchase_orders` / `purchase_order_items`
tables, which are a different module.

- `lib/db/src/schema/overseas.ts:57-88`
- `artifacts/formhaus/src/pages/overseas-order-detail.tsx:220-247`

### #41 — Marketing campaign scheduling
The picker, the `scheduled_at` column, the PATCH route and the amber "Scheduled for…" badge with
clear/change all exist.

**Nothing ever sends it.** There is no scheduler among the nine background workers, and the
source comment states it plainly: the send route ignores `scheduledAt` and always sends
immediately when called. A scheduled campaign sits untouched until someone clicks Send by hand.

- `artifacts/formhaus/src/pages/marketing-campaign.tsx:35-38`
- `artifacts/api-server/src/routes/marketing.ts:596`
- `artifacts/api-server/src/index.ts:118-127`

### #44 — Salesperson + Source on leads / pipeline
**This one is backwards from the description.** The source picker *does* exist — a full Walk-in /
By phone / Referral / Online / Other dropdown, with a conditional "Referred by" field and a
salesperson input, on the job create *and* edit dialog.

What's missing is the **display**. The kanban card shows title, client, tier, address, due date
and value — neither salesperson nor lead source appears on the card, and neither is a column in
list view.

- `artifacts/formhaus/src/pages/pipeline.tsx:261-278` (picker)
- `artifacts/formhaus/src/pages/pipeline.tsx:297-430` (card)

### #54 — Auto-send Factory Spec Sheet on approval
Accurate as originally written. The manual path is complete — generate, download, and email to
supplier, installer or a custom address with supplier auto-suggestion and activity logging. The
automatic trigger on a job or order reaching approved status does not exist.

- `artifacts/api-server/src/routes/invoices.ts:885-960`

---

## 4. Not built

### #10 — Auto-attach contract + warranty on invoice approval
The word "warranty" does not appear anywhere in the codebase, and there is no attach-on-approval
trigger. Contract upload and storage exist; the automation does not.

### #19 — Invoice naming: collection prefix + job/street identifier
**The described convention does not exist.** Every invoice is `FA-<year>-<4-digit-seq>` from a
single generator with one hardcoded "FA" prefix. No `AC-`, no `CI-`, no derivation from
department, no admin override.

There *is* a separate human-readable Invoice Name — `{Number} | {Client} | {Job Title}`,
auto-generated and editable. Useful, but a different feature from the one on the list.

- `artifacts/api-server/src/lib/invoiceNumber.ts:23`
- `lib/db/src/sql/invoiceNamesBackfill.sql`

### #39 — Overseas arrival dates synced to calendar
The calendar reads only the `calendar_events` table, whose six types are all manually created.
An `estimatedArrival` never becomes an event.

Arrivals do surface elsewhere — as an "Arriving from Overseas" card on the dashboard — just not
on any calendar.

- `artifacts/api-server/src/routes/calendar.ts:45-75`
- `artifacts/formhaus/src/pages/calendar.tsx:44-51, 261-266`

### #52 — Contracts auto-attach on invoice approval
Same finding as #10. Storage and manual send exist; there is no approval trigger.
