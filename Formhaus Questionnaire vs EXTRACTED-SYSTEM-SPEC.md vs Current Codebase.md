# Formhaus Questionnaire vs. EXTRACTED-SYSTEM-SPEC.md vs. Current Codebase

**Sources compared:**
1. `Formhaus Questionnaire.docx` — the client's answers to the intake questionnaire (referred to below as "the Questionnaire").
2. `AppSeed/EXTRACTED-SYSTEM-SPEC.md` — the ground-truth spec reverse-extracted from the AppSeed prototype handoffs (referred to below as "the Spec").
3. The current live codebase: `lib/db/src` (schema), `artifacts/api-server/src` (routes/business logic), `artifacts/formhaus/src` (UI), `lib/integrations-anthropic-ai`, `docs/*.md`.

**Method:** Every claim below is backed by a file path, line number, table/column name, or an explicit negative-search statement ("grepped X, zero hits"). Nothing here is inferred from memory of what the app "probably" does — everything was verified against the current repository state. As of this report, the codebase is on branch `main` at commit `8c73914`.

**Structure:** The Questionnaire covers 13 distinct topics. For each topic, this report addresses the four requested dimensions — **Implemented**, **Missing**, **Spec Contradictions**, **Code Contradictions** — but only includes the subsections that actually have content for that topic (an empty "Code Contradictions" subsection is omitted rather than padded).

---

## Quick-scan summary

| # | Topic | Status |
|---|---|---|
| 1 | Team, Roles & Devices | ❌ 3-tier permission model (full / view-only / warehouse-only) does not exist — codebase is binary admin/employee |
| 2 | Current Data & Import | ⚠️ Client CSV import works; QuickBooks/Drive/Sheets client **pull** and historical invoice import do not exist |
| 3 | Pipeline / Workflow Stages | ⚠️ 12 deal stages exist but drawings, contract signatures, and phase-duration tracking are shallow or absent |
| 4 | Dashboard Analytics | ⚠️ Several requested widgets (today's tasks, daily pickups/deliveries, full open-invoice list, overseas timeline) missing |
| 5 | Notifications | ⚠️ Invoice-paid and warehouse events fire; invoice-overdue and team-reminder events do not exist anywhere |
| 6 | Calendar | ❌ Entirely missing — no table, route, or Google Calendar integration of any kind |
| 7 | Client Page & Contact Associations | ⚠️ Most fields exist; multiple job addresses per client and any contact-association model are both missing |
| 8 | Supplier Page | ⚠️ All requested data fields exist, but the page has **zero** access restriction — every authenticated user, including warehouse, currently sees supplier bank info and costs |
| 9 | QuickBooks Integration | ⚠️ Correctly targets QuickBooks Online; sync is push-only (Formhaus→QB); no client/invoice pull |
| 10 | Invoice Structure — 6 Brands | ❌ Only ACERO exists, and only 10 of its 16 required fields; TERA/CIEL/LUME/ETZ/PYRA are 0% implemented behind a cosmetic-only dropdown |
| 11 | Dropdowns & Variables | ⚠️ Top-level product catalog is admin-editable; the actual option sub-lists (openings, trims, glass types) are hardcoded in code/enum, not data |
| 12 | Sending Invoices | 🔴 **Direct contradiction** — the app already emails invoices directly from Formhaus's mailbox in production; the client explicitly chose PDF-generate-and-sync-to-QuickBooks instead |
| 13 | AI Takeoff | ⚠️ A full working PDF→invoice AI pipeline is already built, despite the client's answer being "to be discussed during a call" (i.e., not yet scoped/approved) |

---

## 1. Team, Roles & Devices

**Questionnaire ask:** 4 named users — Tal (full access), Taylor (full access), Eli (restricted: view-only on pipeline/pricing/ETA, no editing permissions), Cesar (warehouse access only). Devices: desktop/laptop/phone for Tal/Eli/Taylor, iPad or desktop for Cesar.

### Implemented
- Binary access-level model: `lib/db/src/schema/users.ts:5` — `roleEnum = ["admin", "employee"]`. Tal and Taylor can both be seeded as `admin`, satisfying "full access" for those two specifically.
- A separate **notification-role** axis exists (`office_manager`, `warehouse_manager`, `invoice_issuer`, `admin` — `users.ts:10-15`), which is independent of edit permissions.
- Responsive/cross-device UI: `artifacts/formhaus/src/hooks/use-mobile.tsx` (breakpoint hook), `artifacts/formhaus/public/manifest.json` (PWA manifest), Tailwind responsive classes throughout — adequate for the desktop/laptop/phone/iPad usage the client described. No further action needed here.

### Missing
- **No third/fourth access tier.** The schema has exactly two access levels. There is no way to express Eli's "view only, no editing permissions" (a restricted-but-broader-than-warehouse tier) or Cesar's "warehouse access only" (a tier narrower than general employee access) as anything other than the same undifferentiated `employee` role. Both would currently get identical permissions.
- `artifacts/api-server/src/lib/requireAdmin.ts` only exposes a single `isAdminRequest` check — confirms there is no finer-grained RBAC anywhere in the request-authorization middleware to build this on top of without schema changes.

### Spec Contradictions
- The Spec's access-control matrix (§2.2) is explicitly built as a **two-row** model (admin vs. employee) and treats "employee" as a single undifferentiated tier that can build invoices, edit clients, and run takeoffs. The Questionnaire requires **three** effectively distinct tiers under that same "employee" umbrella (full-access Taylor if not admin / view-only Eli / warehouse-only Cesar). The Spec's role model cannot represent this — Eli and Cesar would both be "employee" under the Spec's design, yet need opposite supplier-page and edit permissions (see §8 below). This is a structural gap in the Spec's access model, not just an implementation gap.

---

## 2. Current Data & Import

**Questionnaire ask:** ~40 open clients currently live in QuickBooks Online + Google Drive + a Google Sheet. Client wants **both** existing clients **and** historical invoices imported into the new system — not a clean slate.

### Implemented
- Client CSV import: `POST /clients/import` (`artifacts/api-server/src/routes/clients.ts:197`) + `artifacts/api-server/src/lib/takeoff/csvImport.ts` — fuzzy-header parser recognizing name/company/address/phone/email/tier, tier-name resolution, per-row error reporting. This is a workable path for the ~40-client Google Sheet import.

### Missing
- **QuickBooks → Formhaus pull:** does not exist. `docs/QUICKBOOKS-INTEGRATION.md` is explicitly one-directional ("Sync Formhaus Atelier → QuickBooks Online"); `routes/quickbooks.ts` only exposes connect/callback/status/tax-codes/bank-accounts/sync/expense-sync/webhook — all push or inbound-payment-webhook, never a customer or invoice import from QB.
- **Google Drive / Google Sheets import:** no integration code found for either — only the generic CSV importer above, which the client would need to manually export a sheet into.
- **Historical invoice import (backfill):** no route, script, or doc exists anywhere. The QuickBooks sync only pushes invoices created going forward inside the app; there is no bulk-backfill mechanism for the client's existing historical invoices sitting in QuickBooks today. This directly fails the Questionnaire's explicit requirement ("Existing clients and historical invoices should be imported into the new system," line 37).

---

## 3. Pipeline / Workflow Stages

**Questionnaire ask:** a specific 15-step real-world workflow — consultation → quote/invoice/contract build → send invoice+contract for **both to be signed** → deposit received → drawings received → drawings revised if needed → final drawing approval **+ signature** → order placed overseas → production ~8wk → ocean transit ~4wk → request remaining balance before shipment leaves port (sometimes delayed until LA arrival instead) → shipment arrives, held 2–3 days for customs → delivered to Formhaus → warehouse unloads/preps → delivered to job site same/next day.

### Implemented
- `lib/db/src/schema/deals.ts:5-19` (`dealStageEnum`) — 12 stages: `new_lead, consultation, working_on_invoice, invoice_sent, deposit_paid, drawings_received, drawings_approved, ordered_overseas, in_transit, balance_requested, ready_delivery, closed_won, closed_lost`. Consultation, quote/invoice build, invoice sent, deposit paid, order placed overseas, in-transit, and ready/delivery are all implemented as distinct trackable stages.
- Overseas sub-stage tracking: `lib/db/src/schema/overseas.ts:5-16` (`overseasStageEnum`), 10 values from `order_placed` through `delivered`, including `in_production`, `at_sea`, and `customs`.
- Invoice e-signature capture is real: `invoiceSignaturesTable` + `POST` capture endpoint at `artifacts/api-server/src/routes/publicInvoices.ts:110-151` (in-app canvas signature, one per invoice, audit-logged).
- Downstream warehouse unload/prep/delivery is solidly covered by the `local_order_status` lifecycle (`On Hold → Awaiting Pick → Packing → Ready → Out for Delivery → Closed`), implemented in `lib/db/src/schema/orders.ts`.

### Missing
- **Drawings as an artifact, not just a label.** `drawings_received` and `drawings_approved` are two coarse enum values with no drawings file/attachment entity behind them anywhere in the schema (confirmed by grep) — there is no way to actually store the drawings, track a revision round, or record who requested/made a revision.
- **Contract signature.** Only invoice signatures exist. There is no separate "contract" entity or signature capture anywhere in the schema — the client explicitly said **both** invoice and contract require signatures (line 44); only half of that is built.
- **Phase-duration tracking.** Production (~8wk) and ocean transit (~4wk) are not tracked as durations anywhere — only current-state (`overseasStageEnum`) plus a single `estimatedArrival` date and `leadTimeDays`. There's no expected-vs-actual tracking per phase, and no distinction between the two balance-timing policies the client described (request before shipment leaves port vs. delay until LA arrival) — `balance_requested` is one generic stage with no field for which policy applies to a given deal.
- **Customs hold duration.** `customs` is a single stage value with no hold-start/hold-length field, so the "2–3 days" fact from the client can't be tracked or alerted on.

### Spec Contradictions
- The client's answer, "Deals are not relevant for Formhaus" (Questionnaire line 100), directly conflicts with the Spec's core value proposition ("A Pipeline with role-based handoffs lets any employee pick up any client's deal," §1.3) and the entire `deals`-table-centric architecture (§3.B.3, §4.3). Read alongside the surrounding context — the client describing clients with **multiple job addresses, each its own project** — the likely resolution is that the pipeline's unit of work should be scoped to a **job/address**, not a **client**, and the client is rejecting the word "deal," not the concept of tracked work-in-progress. As currently built, `deals.client_id` scopes one deal per client relationship rather than per job/address, meaning a client with 4 simultaneous jobs (per the Questionnaire's own John Smith example, lines 103) cannot have 4 independently-tracked pipeline items today. This is a genuine architectural conflict between the Spec's client-scoped deal model and the client's job-scoped mental model, not just terminology.

---

## 4. Dashboard Analytics

**Questionnaire ask:** open invoices, tasks for today/reminders/follow-ups, current orders in play, item tracking, overseas orders timeline ("see where they are at"), daily pickups and deliveries, and an embedded calendar.

### Implemented
- Receivables/open-balance stat card (`artifacts/formhaus/src/pages/dashboard.tsx:74-85`, backed by `artifacts/api-server/src/routes/dashboard.ts:26-41`).
- Open Deals stat card (`dashboard.tsx:89-100`) and Packing Queue card (`dashboard.tsx:141-154`) — partial coverage of "current orders in play."
- Low Stock count/list (`dashboard.ts:52-65`, `dashboard.tsx:102-113`) — reorder-alert style item tracking (not general inventory tracking).
- "Arriving from Overseas" panel (`dashboard.tsx:222-253`, `dashboard.ts:67-81`) — shows ETA countdown per overseas order.

### Missing
- **Open invoices list.** Only a single aggregate $ stat card exists — no dashboard panel listing individual open invoices (that view only exists on the dedicated Invoices page).
- **Tasks for today / reminders / follow-ups.** Not present at all — `dashboard.ts` only computes an overdue-deal *count*, with no "due today" or task/reminder panel anywhere in the dashboard component.
- **Overseas order timeline/stage detail on the dashboard.** The "Arriving from Overseas" panel shows only an ETA countdown, not the `overseasStageEnum` position — a user can't see "which of the 10 stages is this order in" from the dashboard itself (that detail only lives on the dedicated Overseas Orders page).
- **Daily pickups and deliveries.** No widget and no backing query for today's pickups/deliveries anywhere in `dashboard.ts`.
- **Calendar.** Missing entirely — see §6.

---

## 5. Notifications

**Questionnaire ask:** "Internal Office" needs shipment updates (tracking/delays), invoice paid, invoice overdue, team reminders. "Warehouse" needs deliveries/pickups and past-due + today's follow-ups.

### Implemented
- Role model maps reasonably: `office_manager`/`warehouse_manager` (`notificationRoleEnum`, `users.ts`) roughly correspond to the Questionnaire's "Internal Office" and "Warehouse," gated through a single chokepoint (`artifacts/api-server/src/lib/notify.ts:24-29`).
- Invoice paid: fires (`invoice_paid` in `lib/orderAutoGen.ts:211-217`, `payment_recorded` in `lib/payments.ts:156-162`).
- Warehouse deliveries/pickups: fully implemented — `order_ready`, `pickup_needed`, `order_out_for_delivery`, `order_released`, `order_packing`, `order_closed` all target `warehouse_manager` (`routes/localOrders.ts:301-346`, `lib/localOrders.ts:93-541`, `lib/pickup/service.ts:243-674`).
- Partial shipment-update coverage: `overseas_arrived` and `overseas_delivered` fire (`lib/overseas.ts:94-99, 115-120`).

### Missing
- **Shipment delays / intermediate stage changes.** Only arrival and final delivery notify — no notification fires for intermediate ship-stage transitions (`at_sea`, `customs`, `booked_freight`, etc.), and there's no delay-detection logic of any kind.
- **Invoice overdue.** `overdue` exists only as an on-demand derived boolean (`deriveOverdueFlag`, `lib/orderAutoGen.ts:19-51`) — it is never pushed as a notification event, and no scheduled job checks for newly-overdue invoices (the only cron-style job in the codebase, `lowStockDigest.ts`, is inventory-only and off by default).
- **Team reminders.** No "reminder" event exists anywhere in the notify call sites, and no scheduled job watches deal due-dates.
- **Warehouse "past and follow-ups for today."** Same root cause as team reminders — no due-date-driven notification exists for any role today.

### Spec Contradictions
- The Spec's notification-role axis introduces an "Invoice Issuer" role the Questionnaire never names, while the Questionnaire's two named groups ("Internal Office," "Warehouse") don't cleanly separate into the Spec's three-role structure — a minor naming/mapping mismatch worth resolving explicitly with the client rather than assuming Office Manager = "Internal Office."

---

## 6. Calendar

**Questionnaire ask:** Yes — specifically "linked to calendar with all bank holidays" (implying an external calendar integration, not just an internal events table). Must include sales appointments, team schedules, out-of-office days, other events, client meetings.

### Missing
- **Confirmed entirely absent by exhaustive search.** Grepped `calendar|holiday|gcal` (case-insensitive) across `lib/` — zero hits — and across `artifacts/` — 6 hits, all false positives (the lucide `Calendar` *icon* used for ETA badges, and the generic shadcn/`react-day-picker` date-picker widget `components/ui/calendar.tsx` used for ordinary date-input fields, not scheduling). There is no calendar entity, no Google Calendar integration, no bank-holiday data source, and no appointments/team-schedule/out-of-office tracking anywhere in the codebase.

### Spec Contradictions
- The Spec never mentions a calendar feature anywhere in its functional modules (§3), integration surface (§5), or recommended stack (§1.5). This isn't a contradiction so much as confirmation that the Questionnaire introduces a wholly new requirement the Spec never anticipated — worth calling out because it means calendar/holiday work needs its own scoping pass, it can't be assembled from anything the Spec already described.

---

## 7. Client Page & Contact Associations

**Questionnaire ask:** Name, Phone, Email, **multiple job addresses per client**, open quotes, invoices, tax-exempt flag, front+back ID upload, credit card on file, DOB. Deals are "not relevant." Contact associations wanted at two levels: (a) simple secondary contacts (e.g., a client's son), and (b) **per-job contact assignment** — one client with 4 properties, each with its own distinct contact person. Company clients with multiple employees: explicitly **not** needed.

### Implemented
- `lib/db/src/schema/clients.ts:7-28`: `name`, `phone`, `email`, single `address` (text), `taxExempt` (boolean, line 20), `dob` (text, line 22), `idFrontUrl`/`idBackUrl` (line 23-24), `creditCardOnFile` (boolean flag only — no actual card data stored, line 25, which is the correct PCI-safe approach). Open quotes and invoices are covered functionally via the existing `invoices` table's client linkage, even though they're not literal columns on `clients`.
- Multi-employee-per-company support: correctly absent, matching the client's "No" answer — no action needed.

### Missing
- **Multiple job addresses per client.** Only a single free-text `address` column exists — confirmed both in the schema and in the UI (`artifacts/formhaus/src/components/client-detail.tsx:131, 195`, single address field). No `client_addresses`/job-site table of any kind.
- **Contact associations — both levels.** No `contacts`, `client_contacts`, or job/property-level contact table exists anywhere in `lib/db/src/schema` (zero table hits on a grep for "contact" across the schema directory). Neither the simple client-level association (Dave/Brian) nor the per-job contact assignment (John Smith's 4 properties, each with a different contact person) has any backing schema — this is a two-level gap, both levels fully unimplemented.

### Spec Contradictions
- See §3 above — "Deals are not relevant for Formhaus" directly conflicts with the Spec's deals-centric Pipeline architecture, and is closely tied to this section's missing multi-job-address model: the natural fix for both gaps is likely the same schema change (a job/property entity under each client), so these should be scoped together, not independently.

---

## 8. Supplier Page

**Questionnaire ask:** ~10 suppliers. Fields: contact info, bank information, hours of operation, leadtime, container size, shipping cost per container, location. **Visibility restricted to Eli, Tal, and Taylor only** — explicitly excluding Cesar (warehouse).

### Implemented
- `lib/db/src/schema/overseas.ts:18-38` (`suppliersTable`) has essentially every requested field: `contactName`/`email`/`phone` (contact info, line ~30-31), `bankInfo` (line 32), `hoursOfOperation` (line 33), `leadTimeDays` (line 27), `containerSize` (line 28), `shippingCostPerContainer` (line 29), `country` (partial location — no street/city columns beyond country).

### Missing / Code Contradictions
- **The access restriction does not exist, and current behavior actively violates it.** `artifacts/api-server/src/routes/overseas.ts:313-337` — the `/suppliers` GET/POST/PATCH/DELETE routes only check that a user is authenticated at all (`if (!userId) return res.status(401)...`); there is no role check, not even the existing admin/employee gate, let alone the Questionnaire's specific named-user restriction. Confirmed on the frontend too: `artifacts/formhaus/src/components/layout.tsx:42` renders the Suppliers nav link unconditionally, while the Catalog (line 43) and Financials (line 44) links are correctly gated behind `isAdmin` — Suppliers is not, and `artifacts/formhaus/src/pages/suppliers.tsx` has no role check of its own either.
- **Practical impact:** this is not a passive gap, it's active over-exposure — Cesar (warehouse) currently has full visibility into supplier bank information and container/shipping costs today, which the client explicitly said should be hidden from him. This should be treated as a priority fix, not a backlog item, since it's live financial/banking data currently reachable by an unintended audience.

### Spec Contradictions
- The Spec's access-control matrix (§2.2) never mentions a Supplier page restriction at all — it has no row for it. Since the Spec's role model is binary (admin/employee) anyway, it structurally cannot express "Eli (employee) sees it, Cesar (employee) doesn't" — both are the same access level under the Spec's design. This is the same structural gap flagged in §1.

---

## 9. QuickBooks Integration

**Questionnaire ask:** Client uses QuickBooks Online (web version); Taylor occasionally uses the QBO iPhone app.

### Implemented
- Correctly targets QuickBooks **Online**, not Desktop: `docs/QUICKBOOKS-INTEGRATION.md` — OAuth via `intuit-oauth`, sandbox/production base URLs `sandbox-quickbooks.api.intuit.com` / `quickbooks.api.intuit.com`, realm-based auth. No mention of QuickBooks Desktop anywhere in the doc or `lib/qbo/*`. This matches the client's stated tooling exactly — no action needed here.

### Missing
- See §2 above — sync is push-only (Formhaus → QuickBooks). There is no pull path for customers or historical invoices already in QuickBooks.

### Spec Contradictions
- **QuickBooks is not mentioned anywhere in EXTRACTED-SYSTEM-SPEC.md** — not in the Recommended Stack table (§1.5), not in Integration & API Specifications (§5), nowhere. Despite QuickBooks being one of the client's three current data stores and, per the Questionnaire, the client's chosen invoice-delivery mechanism (§12 below), the Spec — which is meant to be the ground-truth extraction of the system's requirements — has a complete blind spot on this integration. This confirms the Spec document predates or otherwise excludes the QuickBooks integration entirely, even though it is now a core, already-partially-built piece of the product. The Spec should be treated as stale/incomplete on this point, not authoritative.

---

## 10. Invoice Structure — Brands (ACERO / TERA / CIEL / LUME / ETZ / PYRA)

**Questionnaire ask:** the system must support six distinct Formhaus product-line brands, each with its own line-item field set. The client explicitly flagged ACERO as the already-well-specified baseline ("Everything needed for ACERO is in the Claude," line 130); the other five are new asks.

### Ground truth on "brands" in the current codebase
The codebase has one generic product-type table, `system_types` (`lib/db/src/schema/catalog.ts:5-66`), plus a free-text `brand` column (line 14 — used for manufacturer sub-brands like "Milgard," not Formhaus product lines) and a `category` column (line 9, default `"windows_doors"`). The admin Catalog UI (`artifacts/formhaus/src/pages/catalog.tsx`) **has already wired a category dropdown listing all six Formhaus brand names** (lines 212-217, 368-378: `acero`, `tera`, `ciel`, `lume`, `etz`, `pyra` with their correct display names).

**This dropdown is cosmetic only.** The form's pricing/constraints/flags sections (`catalog.tsx:249-364`) are identical regardless of which category is selected — every system type gets the same windows/doors-specific fields (cost/sf, sale/sf, TB cost/sf, glass adders, screen adder, arch adder, frame colors, openings) no matter which brand is chosen, and there is no image-upload widget (`photoUrl` is a raw URL text `<input>`, line 237, not a file upload).

### ACERO — steel windows & doors (10/16 fields fully implemented)

| # | Required field | Status | Evidence |
|---|---|---|---|
| 1 | Product Type | ✅ Implemented | `system_types.name`/`code` (`catalog.ts:6-8`) |
| 2 | Overall Size (W×H) | ✅ Implemented | `invoice_line_items.widthIn`/`heightIn` (`invoices.ts:66-67`) |
| 3 | **Frame Size** (distinct ROUGH/NET FRAME dims) | ❌ Missing | No `frame_width`/`rough_opening`/`net_frame` column anywhere (negative grep); only overall width/height exists |
| 4 | Color / Finish | ✅ Implemented | `frameColor` (`invoices.ts:81`) + `availableFrameColors` catalog field |
| 5 | Handing | ⚠️ Partial | `opening` text field exists, but `OPENING_OPTIONS` is hardcoded to 5 values (`invoice-new.tsx:36`) with **no "Pivot" option**, and no per-system allowed-openings list |
| 6 | Opening Configuration (Pivot / Center Open / X-X / X-O / Tilt Down-Out) | ❌ Missing | Same hardcoded list has none of these |
| 7 | Panel Configuration (2/3/4-panel slider, double casement) | ❌ Missing | Only `doubleLeaf` boolean + `activeLeaf` exist — no slider panel count |
| 8 | Grid Pattern (vertical / horizontal / none) | ⚠️ Partial | `hasGrids` is boolean only (`invoices.ts:72`) — yes/no, not a pattern choice |
| 9 | Bottom Steel Panel | ❌ Missing | No dedicated field; not present in `EXTERIOR_TRIM_OPTIONS` |
| 10 | Stucco Bead | ✅ Implemented | `EXTERIOR_TRIM_OPTIONS` includes `1" Stucco Bead`/`2" Stucco Bead` (`invoice-new.tsx:38`) |
| 11 | Screens | ✅ Implemented | `hasScreen` + per-system `screenCostAdder`/`screenSaleAdder` |
| 12 | Additional Options (e.g. Track Height +1.75") | ❌ Missing | No generic additional-options/free-form addon field on the line item |
| 13 | Quantity | ✅ Implemented | `quantity` (`invoices.ts:73`) |
| 14 | Unit Price | ✅ Implemented | derived from `salePerSf` × sf |
| 15 | Tax | ✅ Implemented | invoice-level `taxRate`/`taxAmount` (`invoices.ts:32-33`) |
| 16 | Extended Amount | ✅ Implemented | `lineTotal` (`invoices.ts:76`) |

### TERA — tile & marble — Missing
No SF + quantity + rate + description + photo line-item model exists. A TERA product could only be forced through either (a) the windows/doors `system_types` form — which would still show glass type, opening, screen, arch fields, all nonsensical for tile — or (b) a generic `inventory` stock item (`lib/db/src/schema/inventory.ts:5-19`), which only supports flat name/SKU/cost/price/quantity: **no SF field, no "rate," no per-line description, and no photo/image column** at all. Even `system_types.photoUrl` (the only image-adjacent field in the whole schema) is a plain URL text box, not an actual photo upload — so the "uploaded photo of the actual tile, not a generated thumbnail" requirement fails on two independent counts.

### CIEL — wood interior doors — Missing
Zero matches anywhere in `lib/db/src` or `artifacts/formhaus/src` for any of: No./Position, Door Type, Lock Type, Privacy/Passage, Door Profile, Hidden Jamb, Jamb Color, Jamb Thickness, Rough Opening W/H, Opening Way (LHI/LHR/RHI/RHR), Door Finish, Hinge Type, Hinge Color, Handles. No door-schedule field set exists beyond ACERO's window/door fields, which don't cover jamb/hinge/lock attributes at all.

### LUME — lighting — Missing
No lighting SKU catalog exists. Could only be entered as a flat-price `inventory` stock item (name/SKU/price/qty) — none of Model/Power/CCT/CRI/Size/Material/Color/Beam-Angle spec fields exist anywhere.

### ETZ — wood siding & ceiling beams — Missing
No siding/beam catalog or fields (color options, length options, per-SF vs. per-beam unit toggle). Same generic `inventory` stock-item fallback only.

### PYRA — fireplaces — Missing
No fireplace catalog. No accessories sub-structure (Remote/Filling Hose/Plug/AC Adapter, each with its own quantity), no fuel-type/material fields. Same generic `inventory` stock-item fallback only, which also has no room for an accessory breakout.

### Code Contradictions
- **The Catalog admin UI actively misleads on this.** Since selecting "TERA," "CIEL," "LUME," "ETZ," or "PYRA" from the category dropdown produces the exact same windows/doors-specific form (glass type, opening, screen, arch fields) as ACERO, an admin who picks one of those categories expecting a sensible brand-appropriate form instead gets a nonsensical one. The dropdown implies support that does not exist — this is worse than the feature simply being absent, because the UI actively signals it's available.

### Spec Contradictions
- The Spec's entire invoicing data model (§3.A, §4.2) is built exclusively around the single Acero window/door product line and states outright that "brand identity must be configurable per invoice/product-line (future brands under Formhaus Atelier)" (§1.2) as a forward-looking statement — but defines no actual mechanism for a second brand's field set. The Questionnaire's five additional brands are not an edge case the Spec merely under-specified; the Spec's schema (`invoice_lines` with `width_in`, `height_in`, `opening`, `glass_type`, etc. as NOT NULL/typed columns, §4.2) is structurally single-product-type and would need real re-architecture (e.g. a JSON attributes column, or per-brand line-item tables) to hold TERA/CIEL/LUME/ETZ/PYRA data at all.

---

## 11. Dropdowns & Variables

**Questionnaire ask:** "Yes. All applicable categories and variables should be configurable as dropdown selections" — i.e., admin-editable without a code deploy.

### Implemented
- The top-level product/system catalog itself is admin-editable: add/edit/delete a `system_type` row with its own pricing, min/max size, and lead time via `catalog.tsx` CRUD (`useCreateSystem`/`useUpdateSystem`/`useDeleteSystem`). Per-tier discounts (`client_tiers` table, `settings.ts:38-46`) and company/tax settings (`business_settings`, `settings.ts:10-25`) are likewise admin-editable database rows, not hardcoded.

### Code Contradictions
- **The sub-option lists the client actually cares about are hardcoded, not data.** `OPENING_OPTIONS` and `EXTERIOR_TRIM_OPTIONS` are JS array literals in `invoice-new.tsx:36, 38`. `glass_type` is a **Postgres enum** (`glassTypeEnum`, `invoices.ts:7` — `standard|low_e|low_e3|low_e_366`), which requires a schema migration to add a value, not an admin UI action. This directly contradicts the client's stated requirement's *mechanism*, not just its scope: it's not that these dropdowns are missing, it's that they exist and are implemented in a way that requires a code deploy to change, exactly what "all applicable categories and variables should be configurable" was meant to avoid. It also explains why TERA/CIEL/LUME/ETZ/PYRA can't simply be "added via the dropdown admin" today — the category selector is data-driven, but the field sets those categories would need are compiled into the windows/doors-specific form and schema, not expressed as configurable metadata.

---

## 12. Sending Invoices

**Questionnaire ask (explicit binary choice, lines 444-449):** the client was asked to pick between (a) sending invoices directly from the app via email, or (b) generating a PDF and syncing it to QuickBooks for QuickBooks to send. **The client chose (b): "Generate PDF and sync to QuickBooks for sending."**

### Code Contradictions — highest-priority finding in this report
The app does the opposite of what was chosen, **and it's already live in production**, not just built-but-unshipped:
- `docs/GMAIL-INTEGRATION.md` Phase 5 is marked complete, including a production smoke test: *"a real invoice sent draft → sent landed in the recipient's inbox... appeared in the shared mailbox's Sent folder... Enabled broadly — no further gating; the worker picks up every future status change."*
- Flow: `PATCH /invoices/:id/status` → `sent` (`artifacts/api-server/src/routes/invoices.ts:222`) fires `enqueueEmail("invoice_sent", invoice.id)` alongside the QuickBooks sync call (~line 232) → `lib/gmail/send.ts` sends via the Gmail API (`users.messages.send`) from the shared Workspace mailbox, with the invoice PDF **attached** (per `GMAIL-INTEGRATION.md` §12: `sendInvoiceEmail` calls `renderInvoicePdf(...)` and attaches it), plus a tokenized read-only invoice link.
- On the QuickBooks side: `docs/QUICKBOOKS-INTEGRATION.md` §1 states plainly that "QB never learns about windows... it sees dollars, a customer, a date, and a tax code" — QuickBooks receives a synced Invoice object for bookkeeping purposes only. There is **no code path where a PDF is transferred to QuickBooks for QuickBooks to email** — the entire delivery mechanism the client asked for does not exist.

**This is not a gap, it's a shipped feature running opposite to the client's recorded decision.** It should be treated as the top item to resolve or explicitly re-confirm with the client, since real client-facing emails are going out today through the rejected channel.

### Spec Contradictions
- The Spec never actually settles this question — §3.A.7/§5.3 describe PDF generation mechanics but not the delivery channel, and the Spec's own Gap Analysis (§7.3, item 10) lists "notification delivery channels (in-app only vs. email/SMS) + provider" as an explicit **open question**. The Questionnaire's answer should have resolved that open question — instead, the Gmail-based direct-send path was already built and shipped before the client's answer was captured, which is itself a process issue (building ahead of an open decision) as much as a code issue.
- Minor stack deviation: the Spec's Recommended Stack table (§1.5) suggests Resend or Postmark for email; the actual implementation uses the Gmail API. Secondary to the channel question above, but worth noting if the direct-send approach is ever revisited/kept.

---

## 13. AI Takeoff

**Questionnaire ask:** "Status: To be discussed during a call" — i.e., the client has not yet approved or scoped this feature; it is explicitly still open on the client's side.

### Implemented (built ahead of client sign-off)
A full working pipeline already exists in `artifacts/api-server/src/routes/takeoffs.ts`:
- `POST /takeoffs/demo` — canned 5-row sample flow.
- `POST /takeoffs/upload` — real pipeline: PDF from object storage → base64 → Claude (`claude-sonnet-4-6`, via `lib/integrations-anthropic-ai`) with a prompt extracting `{mark, type, width_in, height_in, qty, screen, grids, notes}` → fuzzy match against system types (`lib/takeoff/match.ts`) → stored `takeoffLinesTable` rows with confidence scores.
- `PATCH /takeoffs/:id/lines/:lineId` — manual override/correction.
- `POST /takeoffs/:id/convert` — converts a reviewed takeoff into a real, priced draft invoice.
- `docs/AI-TAKEOFF-CLIENT-REQUIREMENTS.md` — a client-facing requirements doc (digital-PDF quality, W1/D1 labeling conventions, schedule-table format) written as though the feature is an already-settled going concern, including an "accuracy improves over time" section implying a learning/feedback loop that has **no corresponding implementation** anywhere in the code (no evidence of a feedback-storage or model-retraining mechanism) — that section of the doc appears aspirational rather than descriptive of current behavior.

### Spec Contradictions
- The Spec (§3.A.9) presents AI Takeoff as an already-designed, near-production feature with a fully worked-out matching/confidence/human-review pipeline. The Questionnaire's own answer is that scope is still open pending a client call. Nothing about what's built technically conflicts with the Spec's design — the conflict is one of **process/authority**: a client-facing requirements doc and a working end-to-end pipeline were both produced before the client's own explicitly-flagged decision gate was resolved. Recommend confirming with the client whether the already-built pipeline matches what they'll actually approve before investing further, rather than treating the Spec's description as pre-approved scope.

---

## Priorities, in order

1. **Sending Invoices (§12)** — resolve immediately; the app is actively emailing clients through a channel the client explicitly rejected.
2. **Supplier page access (§8)** — active data over-exposure (bank info + costs visible to warehouse); trivial fix (add a role check), high impact.
3. **Role model (§1)** — the missing third access tier blocks correctly implementing both #2 and Eli's restrictions; worth doing once, not per-page.
4. **Deals vs. jobs / multi-address clients (§3, §7)** — same underlying architecture question, should be scoped together with the client before either is built.
5. **TERA/CIEL/LUME/ETZ/PYRA (§10)** — largest build item; needs a scoping/architecture decision (attribute-per-brand JSON vs. per-brand tables) before implementation starts, given the current schema is structurally single-product-type.
6. **QuickBooks/historical data pull (§2, §9), Calendar (§6), Notifications gaps (§5), Dashboard gaps (§4)** — sizeable but additive; no architecture blockers, can be scheduled independently.
7. **AI Takeoff (§13)** — confirm scope with the client on the already-scheduled call before extending the existing build further.
