# Missing Features — Gap Backlog (mapped to handoffs, by priority)

**Source handoffs:** `HANDOFF.md` (invoicing) · `CRM-HANDOFF.md` (CRM/ERP v2) · `CRM-HANDOFF-Pickup.md` (pickup flow)
**Scope:** only what is **not done or partial** in the current project. (For what's already built, see `IMPLEMENTATION-STATUS.md`.)
**Date:** 2026-07-01

Priority key: **P0** = foundational/unblocks others · **P1** = core correctness · **P2** = depth & compliance · **P3** = pickup flow · **P4** = large/enhancement.
Handoff shorthand: **INV** = `HANDOFF.md` · **CRM** = `CRM-HANDOFF.md` · **PU** = `CRM-HANDOFF-Pickup.md`.

---

## Master table

| # | Missing feature | Handoff → section | Priority | Depends on |
|---|---|---|---|---|
| 1 | **Payments table** + derived status (partial / paid / open / overdue + `overdue_override`) | CRM §6 | **P0** | — |
| 2 | **Order auto-gen routing**: split lines local vs. overseas by `made_to_order`; partial→On Hold, full→Awaiting Pick; create overseas order too; make it a DB trigger on payment insert | CRM §7.1 | **P0** | 1 |
| 3 | **Storage layer** (Supabase Storage or S3/Azure Blob) — for signatures, proof photos, delivery slips, plan PDFs | INV §5a · CRM §7.4 · PU §4 | **P0** | — |
| 4 | **Realtime layer** (Supabase Realtime or WebSocket/SSE) — live pipeline, order status, notifications, pick tracking | CRM §4, §8 · PU §5 | **P0** | — |
| 5 | **Atomic stock RPC** (`adjust_stock`) + **`scan_log`** table (no client read-modify-write) | CRM §5 | **P1** | — |
| 6 | **Scan-to-pack hardening**: `pack_local_item` atomic RPC, **decrement inventory** on pack, append system note, auto-flip to Packing on first scan, notify | CRM §7.3 | **P1** | 4, 5 |
| 7 | **Notification generation** — wire the event triggers (invoice paid → Office/Warehouse/Invoice Issuer; first scan → packing; ready; pickup-needed; closed) | CRM §8 | **P1** | 4 |
| 8 | **Notification roles** (Office Manager / Warehouse Manager / Invoice Issuer streams; admin can view any) | CRM §2, §8 | **P1** | 7 |
| 9 | **`fulfillment` field** (pickup \| delivery) on local orders | CRM §7.2 | **P1** | — |
| 10 | **Order closing**: signature capture (pickup) + delivery-slip photo upload (delivery) → Closed | CRM §7.4 | **P1** | 3 |
| 11 | **Tier lock** — trigger/RPC rejecting employee changes to `clients.tier` | INV §2 | **P2** | — |
| 12 | **Full snapshotting** — freeze tier discount % **and** tax rate onto the invoice at save (immutable history) | INV §3 | **P2** | — |
| 13 | **Overseas depth**: full 10-stage `ship_stage` enum, per-order `unit_cost` (admin-only), incoterm/freight ref, **auto-gen from made-to-order lines**, **Delivered → receive into inventory** | CRM §7.5 | **P2** | 2, 5 |
| 14 | **`bank_account` table** + cleared / pending / **projected** balance model | CRM §9 | **P2** | — |
| 15 | **Pickup token** — signed HMAC/JWT, **3-day TTL**, revocable, invalid once `released` | PU §3, §6 | **P3** | — |
| 16 | **Customer notification** — email **and** SMS every time, with server-generated QR image (Resend/Postmark + Twilio, Edge Function on status change) | PU §2 | **P3** | 3 |
| 17 | **Picker slip** route `/pickup/[orderNo]` — scan-to-pack **with quantity**, quick chips (1/5/10/all), fast manual number entry, click-to-pack-all circle, print stylesheet | PU §4 | **P3** | 4 |
| 18 | **Handheld scanner sync** — `order:{orderNo}:scans` channel + `POST /api/pickup/{orderNo}/scan`, **SKU auto-match**, +1 / scan-then-count | PU §5a | **P3** | 4, 17 |
| 19 | **Per-line proof photos** + **documentation gate** (pickup line not done until packed **and** ≥1 photo) | PU §4 | **P3** | 3, 17 |
| 20 | **Dual signatures** — packer "Released by" + customer "Received by", both PNGs at release | PU §4, §8 | **P3** | 3, 17 |
| 21 | **Live pick tracking** — Instacart-style back-end view over `order:{orderNo}` (progress bar, per-line checklist, activity feed, presence) + persist to `pickup_event` | PU §5 | **P3** | 4, 17 |
| 22 | **Inventory decrement on completion** — same-txn as release, `inventory_adjustment` audit rows, projected-negative flag (soft) | PU §5b | **P3** | 3, 17 |
| 23 | **Pickup data model** — `pickup_token`, `pickup_line`, `pickup_photo`, `pickup_event`, `pickup_release`, `product` (stock_on_hand), `inventory_adjustment` | PU §6 | **P3** | — |
| 24 | **Pickup API surface** — `/ready`, `/pack`, `/scan`, `/photo`, `/release` (all server-side, atomic) | PU §7 | **P3** | 15–23 |
| 25 | **Per-item spec fields** on invoice lines: `glass_color`, `frame_color`, `exterior_trim`, `location`, `arch`, `opening`, `active_leaf` (mandatory on double-leaf) | INV §3 | **P4** | — |
| 26 | **Rich thumbnail drawings** — per-panel grids, half-radius arch tops, hinge-swing (solid/dashed OUT/IN), active-leaf "A", handles, tilt-&-turn, frame clipping; store as `thumbnail_svg`; shared builder/PDF module | INV §4 | **P4** | 25 |
| 27 | **Server-side PDF** (`@react-pdf/renderer` or Puppeteer) — two-page, **admin-gated internal cost/profit/margin page**, keep on-screen margin-blur privacy | INV §5 | **P4** | 26 |
| 28 | **CSV client import** — server-side parse, validate, dedupe by email/phone, map tier | INV §6 | **P4** | — |
| 29 | **AI Takeoff** — PDF upload → Claude-API schedule-table extraction → match to systems → review UI → convert to invoice | INV §7 | **P4** | 3 |
| 30 | **RLS + cost-omitting views** (`inventory_public`, `overseas_orders_public`; gate ledger/bank/cost) | CRM §10 | **P4** | — |
| 31 | **ZXing camera scanning** (`@zxing/browser`) for phone/tablet | CRM §5 | **P4** | — |
| 32 | **Unknown-code → create item** prefilled with scanned barcode | CRM §5 | **P4** | — |
| 33 | **Dashboard deep-linking** to bookmarkable routes (`/invoices/…`, `/orders/…`, `/pipeline/{id}`, low-stock item) | CRM §12 | **P4** | — |
| 34 | **`client_purchases`** table (client profile purchase history) | CRM §3 | **P4** | — |

---
