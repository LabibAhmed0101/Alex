# QuickBooks Integration — Test Progress

---

## Finished

### Phase 0 — Foundations

| Test | Goal |
|------|------|
| **T0-1** Database tables created | Confirm the three QB tables exist in the database |
| **T0-2** Base URL switches on `QBO_ENVIRONMENT` | Confirm the server points at the sandbox API, not production |
| **T0-3** Token refresh single-flight guard | Two simultaneous expired-token requests must trigger exactly one refresh call |

### Phase 1 — OAuth Connect

| Test | Goal |
|------|------|
| **T1-1** Connect flow — happy path | Complete the full OAuth authorization code flow and store tokens |
| **T1-2** Verify tokens persisted | Tokens survive the callback and are stored server-side only |
| **T1-3** Tokens persist across server restart | Reconnecting on restart is not required |
| **T1-4** Non-admin blocked from `/connect` | The connect route is admin-gated |
| **T1-5** Unauthenticated request blocked | No session = 401 |
| **T1-6** CSRF protection — invalid state rejected | A callback with a fabricated or expired state is rejected |

### Phase 2 — Customer + Invoice Push

| Test | Goal |
|------|------|
| **T2-1** Create "Window & Door — Custom" service item in QBO sandbox | The service item the mapper references must exist before any invoice sync |
| **T2-2** Manual invoice sync — new invoice creates customer + invoice in QBO | A Formhaus Atelier invoice is pushed to QBO; a new customer is created in QBO |
| **T2-3** Re-push updates, does not duplicate | Pushing the same invoice twice updates, never creates a second QB invoice |
| **T2-4** Invoice with no line items is rejected | Syncing an invoice that has no line items must fail with a clear error |
| **T2-5** Invoice with no client is rejected | Cannot sync an invoice with no associated client |
| **T2-6** Client with duplicate DisplayName in QBO (6240 recovery) | If a customer with the same name already exists in QBO independently, the sync adopts it instead of failing |
| **T2-7** Invoice with discount = 0 — no discount line in QBO | Zero discount does not produce a `DiscountLineDetail` line |
| **T2-8** Invoice with a discount — discount line appears in QBO | A non-zero discount is pushed as a `DiscountLineDetail` |
| **T2-9** Line descriptions contain no internal-only fields | QB sees dollars and human-readable text, never internal dimension/tier data |

### Phase 3 — Tax Mapping

| Test | Goal |
|------|------|
| **T3-1** List sandbox tax codes | Discover what tax codes are available in the sandbox company |
| **T3-2** Set `QBO_DEFAULT_TAX_CODE` and confirm invoices use it | After setting the env var, synced invoices carry the correct tax code |
| **T3-3** `QBO_DEFAULT_TAX_CODE` not set — fallback to `'TAX'` | When env var is absent, code falls back to Automated Sales Tax |
| **T3-4** Tax reconciliation warning on mismatch | When QB's computed tax diverges from Formhaus Atelier's by > 1%, a warning log appears |

### Phase 4 — Auto-Sync on Status Change

| Test | Goal |
|------|------|
| **T4-1** `draft → sent` auto-creates QB invoice | When an invoice's status changes to `sent`, the sync worker automatically creates the QB invoice |
| **T4-2** `sent → accepted` updates existing QB invoice (no duplicate) | Moving to `accepted` updates the QB invoice in place |
| **T4-3** `sent → paid` creates QB invoice + payment; A/R clears | Marking an invoice paid auto-syncs both the invoice and a QB payment, clearing A/R |
| **T4-4** `sent → void` voids the QB invoice | Voiding in Formhaus Atelier propagates a void to QB |
| **T4-5** `draft` status — nothing enqueued to QB | Drafts never go to QB |
| **T4-6** Idempotency — two rapid `paid` PATCHes produce one QB payment | Double-trigger doesn't create duplicate payments |
| **T4-7** QB failure → `error` state → retry → `synced` | Worker retries a failed job with backoff and eventually succeeds |
| **T4-8** Restart recovery — pending jobs not lost | A `pending` row surviving a server restart is picked up on boot |

### Phase 5 — Inbound Webhook (QB → Formhaus Atelier)

| Test | Goal |
|------|------|
| **T5-1** Valid webhook signature accepted | A correctly signed webhook payload is processed |
| **T5-2** Invalid/missing signature rejected | Unsigned webhooks are rejected with 401 |
| **T5-3** Wrong signature rejected | A webhook signed with the wrong key is rejected |
| **T5-6** Rate limiter — 61+ requests from same IP throttled | The in-process rate limiter caps webhook delivery at 60/min per IP |

### Phase 6 — Overseas Expense Sync

| Test | Goal |
|------|------|
| **T6-1** Admin push creates a QB Purchase | An overseas order is pushed to QBO as a Purchase (expense) |
| **T6-2** Re-push updates the existing QB Purchase — no duplicate | Idempotency: pushing the same overseas order twice updates the QB Purchase |
| **T6-3** Non-admin blocked from expense sync | Cost data is admin-gated |
| **T6-4** Overseas order with no `totalCost` is rejected | A meaningful error is returned for orders with no cost |
| **T6-5** Non-existent overseas order returns 404 | Syncing an ID that doesn't exist returns 404, not a server crash |
| **T6-6** Expense sync without QB connection returns 503 | Trying to sync with no active QB connection fails gracefully |

### Security Checks

| Test | Result | Notes |
|------|--------|-------|
| **TS-1** Tokens never appear in any API response | PASS | `/status` uses explicit `select({})` with 5 safe fields; grep confirms no token fields in route file; `exchangeCode` returns only `realmId` + `expiresAt` |
| **TS-2** `QBO_*` env vars never reach the client bundle | PASS (static) | No `define` block in `vite.config.ts`; zero `QBO_*` references in `artifacts/formhaus/`; no `VITE_QBO*` anywhere; all `QBO_*` access confined to `api-server/src/`. Live build blocked by missing Rollup native bindings on Windows — static analysis is conclusive |
| **TS-3** Webhook raw body available for HMAC | PASS | `app.ts` lines 39–44: `express.json({ verify })` stashes `req.rawBody`; webhook route guards against missing `rawBody` with `500 "Server configuration error"`; `verifyWebhookSignature` uses `timingSafeEqual` |
| **TS-4** Sync and status routes require admin | PASS | All 7 listed routes have `requireAdmin`; middleware returns 401 (no session) or 403 (non-admin); fast-path JWT check + slow-path Clerk API lookup prevents stale-token bypass |

---

## Deferred — Requires Real QB Payment

These tests require a Payment to be received in sandbox QBO and delivered via webhook. They will be run once a real payment event is available.

| Test | Goal | Blocked on |
|------|------|-----------|
| **T5-4** QB-side payment flips invoice to `paid` without duplicate ledger | A Payment created in sandbox QBO marks the Formhaus Atelier invoice `paid` and creates exactly one local order + ledger entry | Real QB payment in sandbox + publicly reachable webhook URL (ngrok or deployed URL) |
| **T5-5** Idempotent — webhook received twice does not duplicate | If Intuit retries the same webhook, the second delivery is a no-op | Depends on T5-4 completing first |
