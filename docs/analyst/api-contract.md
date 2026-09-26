# API Contract — Fofa Shop

Hand-written contract snapshot of every HTTP interface this application owns, extracted from
the code as it exists today. This is the input for the **API Contract Review** step of the
design review process.

| Field | Value |
|---|---|
| Verified against code on | 2026-09-26 |
| Sources read | `routes/api.php`, `routes/web.php`, `bootstrap/app.php`, `app/Http/Controllers/Api/V1/*`, `app/Http/Requests/Api/TopupRequest.php`, `app/Http/Controllers/PaymentWebhookController.php`, `app/Services/PaymentSignatureVerifier.php`, `app/Jobs/SendResellerCallbackJob.php`, `app/Observers/TransactionObserver.php`, `app/Providers/AppServiceProvider.php`, `app/Http/Middleware/*`, `config/fofa.php`, `deploy/nginx-site.conf`, `PRD.md` §6 |
| Status | Snapshot (no machine-readable spec exists to compare against) |

> Scope: only contract-relevant endpoints are documented — the 4 reseller/H2H API routes,
> the inbound payment webhook, and the outbound reseller callback. Storefront / admin / owner
> routes are session-based HTML and are described but not contracted field-by-field.

---

## Contract Source

**There is no OpenAPI/Swagger specification anywhere in this repository.** No spec is generated,
one-way or otherwise. This document is a hand-written contract snapshot plus a remediation
recommendation.

| Question | Answer | Evidence |
|---|---|---|
| OpenAPI/Swagger spec exists? | ❌ No | Repo-wide grep `openapi\|swagger` over `*.php,*.json,*.yaml,*.yml,*.md` → 0 hits |
| Spec-generation tooling installed? | ❌ No | `composer.json` / `composer.lock`: no `darkaonline/l5-swagger`, `knuckleswtf/scribe`, `justinrainbow` or similar package |
| Spec generated one-way from code? | ❌ No | No generator, no attributes/annotations on controllers (`app/Http/Controllers/Api/V1/*` carry none) |
| Machine-readable contract of any kind? | ❌ No | No `openapi.yaml`, no `swagger.json`, no `.postman.json` in repo |
| Human-authored contract? | ⚠️ Prose only | `PRD.md:168-253` (`## 6. API Endpoint List`) — endpoint list, no schemas/status codes |
| PRD ↔ code agreement | ⚠️ Drifted | `PRD.md:209-211` lists `GET /api/v1/analytics/clicks`, `GET /api/v1/store`, `PUT /api/v1/store`; `routes/api.php:8-13` implements none of them |
| Promised "Dokumentasi API" feature | ❌ Not implemented | `PRD.md:277` (feature 7.2 #4); no docs route exists in `routes/api.php` or `routes/web.php` |
| What this document is | Hand-written snapshot | Written by reading the files listed above; nothing was inferred from a spec |
| Remediation | Generate + enforce | See [Recommendations](#recommendations) R1 |

Because the only contract artifact is prose in `PRD.md` that has already drifted from
`routes/api.php`, "kontrak implisit pasti drift" is confirmed, not hypothetical.

---

## Endpoints

Inventory of everything this app exposes or calls:

| # | Method + Path | Direction | Auth | Route evidence |
|---|---|---|---|---|
| E1 | `GET /api/v1/products` | Inbound (reseller) | `auth:sanctum` + `role:reseller` | `routes/api.php:9` |
| E2 | `POST /api/v1/topup` | Inbound (reseller) | `auth:sanctum` + `role:reseller` | `routes/api.php:10` |
| E3 | `GET /api/v1/topup/{ref_id}/status` | Inbound (reseller) | `auth:sanctum` + `role:reseller` | `routes/api.php:11` |
| E4 | `GET /api/v1/balance` | Inbound (reseller) | `auth:sanctum` + `role:reseller` | `routes/api.php:12` |
| E5 | `POST /webhook/payment-callback` | Inbound (payment gateway) | HMAC signature (+ optional IP whitelist) | `routes/web.php:46` |
| E6 | `POST {reseller_profiles.callback_url}` | Outbound (system → reseller) | **None** | `app/Observers/TransactionObserver.php:17,35` → `app/Jobs/SendResellerCallbackJob.php:59` |
| — | Storefront / customer / admin / owner routes | Inbound (browser) | Session cookie + role (HTML, CSRF-protected) | `routes/web.php:31-129` |

Group-level middleware for E1–E4: `Route::prefix('v1')->middleware(['auth:sanctum','role:reseller'])`
(`routes/api.php:8`). Bearer tokens are Sanctum personal access tokens provisioned manually by
admin (`app/Http/Controllers/Admin/ResellerTokenController.php:45`), no expiration configured
(`config/sanctum.php:53` → `'expiration' => null`).

---

### E1 — `GET /api/v1/products`

- **Controller:** `app/Http/Controllers/Api/V1/ProductController.php:13` (`index`)
- **Auth:** Bearer Sanctum token, role must be `reseller` and account active
  (`app/Http/Middleware/RoleMiddleware.php:39-46`). Missing/invalid token → 401;
  wrong role or inactive account → 403.
- **Request schema:** none. No query parameters are read — the handler only uses
  `$request->user()` (`ProductController.php:15`). No pagination, filter, or sort parameters.

**Response — 200 OK**

| Field | Type | Notes |
|---|---|---|
| `data` | array | All active products, ordered by `id` (`ProductController.php:22-23`) |
| `data[].id` | integer | |
| `data[].category_id` | integer | |
| `data[].category` | object\|null | `{id, name, slug}` (`ProductController.php:31-35`) |
| `data[].name` | string | |
| `data[].description` | string\|null | |
| `data[].price` | string | Decimal string, 2 dp — `number_format(..., 2, '.', '')` (`ProductController.php:38`) |
| `data[].reseller_price` | string | Decimal string, 2 dp from `CalculatePriceForUser::calculate()` (`ProductController.php:26`, `app/Services/CalculatePriceForUser.php:18-44`) |
| `data[].is_active` | boolean | Always `true` in practice (query filters on it) |

```json
{
  "data": [
    {
      "id": 1,
      "category_id": 2,
      "category": { "id": 2, "name": "Minecraft", "slug": "minecraft" },
      "name": "Minecoin 660",
      "description": "Koin in-game Minecraft",
      "price": "50000.00",
      "reseller_price": "47500.00",
      "is_active": true
    }
  ]
}
```

**Errors**

| Status | Body shape | Evidence |
|---|---|---|
| 401 | `{"message":"Unauthenticated."}` | Laravel default; rendered via `ExceptionHandler` (verified) |
| 403 (wrong role / inactive) | `{"message":""}` | `RoleMiddleware.php:40` → `abort(403)` with no message; render verified with `APP_DEBUG=false` |
| 429 | `{"message":"Too Many Requests."}` **only from nginx** (429, empty body) | App layer has no limit — see Design Checklist |

---

### E2 — `POST /api/v1/topup`

- **Controller:** `app/Http/Controllers/Api/V1/TopupController.php:24` (`store`)
- **FormRequest:** `app/Http/Requests/Api/TopupRequest.php` (rules at `:17-60`, messages at `:62-71`)
- **Auth:** same as E1. `authorize()` always returns `true` (`TopupRequest.php:12-15`) —
  authorization is done entirely by route middleware.

**Request schema**

| Field | Type | Required | Validation | Evidence |
|---|---|---|---|---|
| `product_id` | integer | yes | `required`, `integer`, `exists:products,id` | `TopupRequest.php:20` |
| `target_game_id` | string | conditional | see conditional rules below | `TopupRequest.php:34-57` |
| `target_server_id` | string | no | `nullable`, `string`, `max:255` | `TopupRequest.php:21` |
| `minecraft_edition` | string | no | `nullable`, `string`, `Rule::in(MinecraftEdition::values())` | `TopupRequest.php:22` |

Conditional `target_game_id` rules (resolved per product category, `TopupRequest.php:25-57`):

| Product / edition | Rules | Evidence |
|---|---|---|
| Non-Minecraft | `required`, `string`, `max:255` | `TopupRequest.php:54-57` |
| Minecraft, edition `manual` | `nullable`, `string`, `max:255` | `TopupRequest.php:37-38` |
| Minecraft, edition `java` (or defaulted) | `required`, `string`, `max:255`, `regex:/^[a-zA-Z0-9_]{3,16}$/` | `TopupRequest.php:39-40` |
| Minecraft, edition `bedrock` | `required`, `string`, `min:3`, `max:12`, custom rule: `^[a-zA-Z0-9_ ]{3,12}$`, no leading/trailing space | `TopupRequest.php:41-50` |
| Other edition value | `required`, `string`, `max:255` | `TopupRequest.php:51-53` |

**Response — 201 Created** (`TopupController.php:134-138`)

| Field | Type | Notes |
|---|---|---|
| `ref_id` | string | `FOFA-YYYYMMDD-XXXXXXXX`, server-generated (`app/Services/RefIdGenerator.php:15-31`) |
| `status` | string | Always `"paid"` — reseller topup is wallet-paid and created as `Paid` (`TopupController.php:102`) |
| `amount` | string | Decimal string, 2 dp |

```json
{ "ref_id": "FOFA-20260926-A1B2C3D4", "status": "paid", "amount": "50000.00" }
```

**Errors** — all JSON, Laravel shape `{message}` or `{message, errors}`:

| Status | Body | Evidence |
|---|---|---|
| 422 (validation) | `{"message":"product_id wajib diisi. (and 1 more error)","errors":{"product_id":["..."],...}}` | Framework render of `TopupRequest::messages()` (`TopupRequest.php:64-70`) — verified via `ExceptionHandler` |
| 422 (inactive product) | `{"message":"Produk tidak tersedia.","errors":{"product_id":["Produk tidak tersedia."]}}` | `TopupController.php:31-36` |
| 422 (game id rejected) | `{"message":"<verifier message>","errors":{"target_game_id":["<msg>"]}}` | `TopupController.php:57-63` |
| 422 (wallet) | `{"message":"Insufficient balance","errors":{"balance":["Saldo tidak cukup."]}}` | `TopupController.php:139-145` |
| 401 / 403 | see E1 | `routes/api.php:8`, `RoleMiddleware.php:40` |
| 404 | `{"message":"No query results for model [App\\Models\\Product] <id>"}` | `TopupController.php:29` `findOrFail` (only reachable if row deleted between validation and read) |

---

### E3 — `GET /api/v1/topup/{ref_id}/status`

- **Controller:** `app/Http/Controllers/Api/V1/TopupController.php:150` (`status`)
- **Auth:** same as E1. Result is scoped to `$request->user()->id` (`TopupController.php:156`),
  so another reseller's `ref_id` returns 404 (no existence leak).
- **Request schema:** path parameter `{ref_id}` — plain `string`, **no validation rule**
  (no FormRequest is bound; `TopupController.php:150`). No query parameters.

**Response — 200 OK** (`TopupController.php:163-173`)

| Field | Type | Notes |
|---|---|---|
| `ref_id` | string | |
| `status` | string | Lowercase enum value: `pending`, `paid`, `processing`, `waiting_fulfillment`, `success`, `failed`, `expired` (`app/Enums/TransactionStatus.php:3-12`) |
| `amount` | string | Decimal string, 2 dp |
| `target_game_id` | string | |
| `target_server_id` | string\|null | |
| `product` | object\|null | `{id, name}` |

```json
{
  "ref_id": "FOFA-20260926-A1B2C3D4",
  "status": "success",
  "amount": "50000.00",
  "target_game_id": "Steve",
  "target_server_id": null,
  "product": { "id": 1, "name": "Minecoin 660" }
}
```

**Errors**

| Status | Body | Evidence |
|---|---|---|
| 404 | `{"message":"Transaction not found"}` | `TopupController.php:159-161` |
| 401 / 403 | see E1 | |

No timestamps (`created_at`, `paid_at`, `expired_at`) are exposed on this endpoint.

---

### E4 — `GET /api/v1/balance`

- **Controller:** `app/Http/Controllers/Api/V1/BalanceController.php:12` (`show`)
- **Auth:** same as E1.
- **Request schema:** none (no path or query parameters).

**Response — 200 OK** (`BalanceController.php:17-19`)

| Field | Type | Notes |
|---|---|---|
| `balance` | string | Decimal string, 2 dp — `WalletService::getBalance(): string` (`app/Services/WalletService.php:14`); test expects `"123456.78"` (`tests/Feature/ResellerApiTest.php:131`) |

```json
{ "balance": "123456.78" }
```

**Errors:** 401 / 403 only, as in E1. No other failure mode exists in code.

---

### E5 — `POST /webhook/payment-callback` (inbound, payment gateway)

- **Route:** `routes/web.php:46`; CSRF-exempt via `bootstrap/app.php:24-26` (`'webhook/*'`).
  No auth middleware, **no app-level rate limit** — stated in `routes/web.php:44-45` and
  asserted by `tests/Feature/RateLimitingTest.php:239`.
- **Controller:** `app/Http/Controllers/PaymentWebhookController.php:20` (`handle`)

**Defense order** (`PaymentWebhookController.php:22-48`): IP whitelist → payload validation →
signature verification → atomic state transition.

**Request schema** (inline validator, `PaymentWebhookController.php:157-163`)

| Field | Type | Required | Validation / semantics | Evidence |
|---|---|---|---|---|
| `reference` | string | `required_without:merchant_ref` | Used as transaction lookup key | `:158`, `:201-203` |
| `merchant_ref` | string | `required_without:reference` | Preferred lookup key | `:159`, `:200` |
| `status` | string | yes | Only `PAID` (trimmed, case-insensitive) is acted on; anything else → ignored | `:160`, `:217-219` |
| `amount` | *any* | yes | **No numeric/type rule**; normalized with `number_format((float)$amount, 2, '.', '')` and compared to transaction amount | `:161`, `:210-214` |
| `signature` | string | yes | HMAC-SHA256 hex of `refId . normalizedAmount` under `fofa.payment.secret`; verified against `merchant_ref`, then `reference` | `:162`, `:177-188`; `app/Services/PaymentSignatureVerifier.php:12-30` |

**Response — always HTTP 200**, body `{"success": true}` with an optional `message`
(and `errors` for validation failure):

| Case | Body | Evidence |
|---|---|---|
| Processed | `{"success":true}` | `:132` |
| Invalid signature | `{"success":true,"message":"Invalid signature"}` | `:44` |
| Not whitelisted IP | `{"success":true,"message":"IP not whitelisted"}` | `:149` (only when `fofa.payment.whitelist_ips` non-empty — `config/fofa.php:15`, default empty) |
| Validation failed | `{"success":true,"message":"Validation failed","errors":{...}}` | `:171` |
| Transaction not found | `{"success":true,"message":"Transaction not found"}` | `:56` |
| Amount mismatch | `{"success":true,"message":"Amount mismatch"}` | `:66` |
| Non-PAID status | `{"success":true,"message":"Status not PAID, ignored"}` | `:75` |
| Late payment after failure | `{"success":true,"message":"Transaction already failed, late payment rejected"}` | `:85` |
| Duplicate | `{"success":true,"message":"Already processed"}` | `:94` |
| Invalid current status | `{"success":true,"message":"Invalid transaction status"}` | `:103` |
| Concurrent duplicate | `{"success":true,"message":"Already processed (race)"}` | `:111` |

There is **no non-2xx response path at all** — status codes carry no information on this
endpoint (behavior pinned by `tests/Feature/PaymentWebhookTest.php:111`).

---

### E6 — `POST {reseller_profiles.callback_url}` (outbound, system → reseller H2H)

- **Trigger:** transaction status changed (`app/Observers/TransactionObserver.php:11-20`) or a
  reseller transaction created already `paid` (`:22-37`), dispatched only when the owning user
  has a `callback_url` (`app/Models/Transaction.php:142`).
- **Sender:** `app/Jobs/SendResellerCallbackJob.php:59` — `Http::timeout(5)->post(...)`,
  JSON body by default (`vendor/laravel/framework/src/Illuminate/Http/Client/PendingRequest.php:266`).
- **Auth / signature:** **none.** No HMAC, no shared secret, no timestamp, no event id.

**Request payload** (`SendResellerCallbackJob.php:52-56`)

| Field | Type | Notes |
|---|---|---|
| `ref_id` | string | Transaction reference |
| `status` | string | Lowercase `TransactionStatus` value |
| `amount` | string | Decimal string, 2 dp |

```json
{ "ref_id": "FOFA-20260926-A1B2C3D4", "status": "success", "amount": "50000.00" }
```

**Response handling:** the HTTP response is only logged (`:61-67`); status code and body are
otherwise ignored — the contract has no acknowledgement semantics.

**Retries:** `tries(): 3`, `backoff(): [10, 60, 300]` seconds (`:81-89`). Retries happen **only
when an exception is thrown** (`:68-78`), i.e. connection failures/timeouts. A 4xx/5xx HTTP
response does not throw (no `->throw()`/`->retry()` on the call at `:59`), so it is logged once
and dropped. No callback delivery ledger exists (grep `callback` in `database/migrations/` →
only `2026_08_26_000005_create_reseller_profiles_table.php:17` defining the URL column).

---

### Storefront / customer / admin routes — session-based HTML, not API

`routes/web.php:31-66` (storefront, checkout, invoice, history) and `:68-129` (admin, owner)
are Blade/Livewire routes behind session auth + CSRF (`bootstrap/app.php:24-26`, no `except`
besides `webhook/*`). They are intentionally excluded from this contract: they return HTML
unless the request `expectsJson()`, in which case the Laravel exception shape is used
(`bootstrap/app.php:29-31`). Note `POST /checkout` does carry a documented multi-dimensional
rate limit (`routes/web.php:39-40`, `app/Http/Middleware/CheckoutRateLimit.php:12-27`,
`config/fofa.php:35-46`: 5/min per IP, 20/hour per user, 6/10 min per product+target by default).

---

## Design Checklist

Legend: ✅ satisfied · ⚠️ partially satisfied · ❌ not satisfied · N/A not applicable (read-only endpoint).

### Summary matrix

| Endpoint | Versioning eksplisit | Auth per endpoint | Schema lengkap | Error format konsisten | Pagination/filter/sort | Idempotency | Rate limiting | Breaking-change handling |
|---|---|---|---|---|---|---|---|---|
| E1 `GET /api/v1/products` | ✅ | ✅ | ⚠️ | ⚠️ | ❌ | N/A | ❌ | ❌ |
| E2 `POST /api/v1/topup` | ✅ | ✅ | ✅ | ⚠️ | N/A | ❌ | ❌ | ❌ |
| E3 `GET /api/v1/topup/{ref_id}/status` | ✅ | ✅ | ⚠️ | ⚠️ | N/A | N/A | ❌ | ❌ |
| E4 `GET /api/v1/balance` | ✅ | ✅ | ⚠️ | ⚠️ | N/A | N/A | ❌ | ❌ |
| E5 `POST /webhook/payment-callback` | ❌ | ⚠️ | ⚠️ | ❌ | N/A | ✅ | ⚠️ | ❌ |
| E6 `POST {callback_url}` (outbound) | ❌ | ❌ | ❌ | ❌ | N/A | ⚠️ | N/A | ❌ |

### E1 — `GET /api/v1/products`

| Kriteria | Status | Evidence |
|---|---|---|
| Versioning eksplisit | ✅ | Path prefix `/v1` (`routes/api.php:8`) |
| Auth per endpoint | ✅ | `auth:sanctum` + `role:reseller` on the group (`routes/api.php:8`) |
| Request/response schema lengkap | ⚠️ | Response fields are explicit in code (`ProductController.php:28-41`) but exist only in prose here — no OpenAPI/JSON Schema |
| Error format konsisten | ⚠️ | Laravel default `{message}` / `{message,errors}`; but 403 body is `{"message":""}` (`RoleMiddleware.php:40`, render verified) |
| Pagination/filter/sort | ❌ | `Product::...->orderBy('id')->get()` — unbounded, no params read (`ProductController.php:17-23`) |
| Idempotency | N/A | Read-only |
| Rate limiting | ❌ | Tidak ada limiter di lapisan aplikasi untuk route API (infra nginx menyediakan 30 r/s per IP, burst 20, 429 — `deploy/nginx-site.conf:32,50`) |
| Breaking vs non-breaking change handling | ❌ | No deprecation/versioning policy found (repo grep `breaking change\|deprecat` → only `ADR-0018 (tailwindcss-v4-upgrade):85`, unrelated) |

### E2 — `POST /api/v1/topup`

| Kriteria | Status | Evidence |
|---|---|---|
| Versioning eksplisit | ✅ | `/v1` prefix (`routes/api.php:8`) |
| Auth per endpoint | ✅ | `auth:sanctum` + `role:reseller` (`routes/api.php:8`) |
| Request/response schema lengkap | ✅ | Full rule set in `TopupRequest.php:17-60` incl. conditional game-id rules; response shape fixed at `TopupController.php:134-138` |
| Error format konsisten | ⚠️ | Manual errors mirror the framework shape (`TopupController.php:32-35,59-63,141-145`) but messages are a mix of English and Indonesian (`"Insufficient balance"` vs `"Saldo tidak cukup."`) |
| Pagination/filter/sort | N/A | Single-resource POST |
| Idempotency | ❌ | `ref_id` di-generate server (`RefIdGenerator.php:21`) dan request **tidak menerima idempotency key dari klien**. Unique constraint `ref_id` + `lockForUpdate` di `WalletService::deduct` menjaga **integritas internal** (tanpa dup ref, tanpa race double-spend) — tetapi tidak mencegah klien yang retry setelah timeout membuat order & potongan saldo kedua |
| Rate limiting | ❌ | Same evidence as E1 — no app-level limit on a fund-moving endpoint |
| Breaking vs non-breaking change handling | ❌ | No policy; changing `amount`/`status` from decimal-string/enum would silently break resellers |

### E3 — `GET /api/v1/topup/{ref_id}/status`

| Kriteria | Status | Evidence |
|---|---|---|
| Versioning eksplisit | ✅ | `/v1` prefix (`routes/api.php:8`) |
| Auth per endpoint | ✅ | `auth:sanctum` + `role:reseller`, plus per-user scoping (`TopupController.php:156`) |
| Request/response schema lengkap | ⚠️ | Response explicit (`TopupController.php:163-173`); `{ref_id}` path param has no validation/format rule (`TopupController.php:150`) |
| Error format konsisten | ⚠️ | `{"message":"Transaction not found"}` (`TopupController.php:160`) — consistent with Laravel, but 401/403 use different message conventions |
| Pagination/filter/sort | N/A | Single resource |
| Idempotency | N/A | Read-only |
| Rate limiting | ❌ | Same evidence as E1 |
| Breaking vs non-breaking change handling | ❌ | No policy |

### E4 — `GET /api/v1/balance`

| Kriteria | Status | Evidence |
|---|---|---|
| Versioning eksplisit | ✅ | `/v1` prefix (`routes/api.php:8`) |
| Auth per endpoint | ✅ | `auth:sanctum` + `role:reseller` (`routes/api.php:8`) |
| Request/response schema lengkap | ⚠️ | One field, explicit (`BalanceController.php:17-19`); no schema artifact |
| Error format konsisten | ⚠️ | Framework default only |
| Pagination/filter/sort | N/A | Single value |
| Idempotency | N/A | Read-only |
| Rate limiting | ❌ | Same evidence as E1 (balance enumeration is cheap without a per-token limit) |
| Breaking vs non-breaking change handling | ❌ | No policy |

### E5 — `POST /webhook/payment-callback`

| Kriteria | Status | Evidence |
|---|---|---|
| Versioning eksplisit | ❌ | No version in path (`routes/web.php:46`) and none in payload (`PaymentWebhookController.php:157-163`); future field changes are undetectable by the gateway |
| Auth per endpoint | ⚠️ | HMAC-SHA256 (`PaymentSignatureVerifier.php:12-30`) + optional IP whitelist (`config/fofa.php:15` — empty by default). **Catatan keamanan (detail disembunyikan untuk publikasi):** konfigurasi secret punya jalur fallback yang tidak gagal-fast saat env kosong — wajib diperbaiki sebelum production |
| Request/response schema lengkap | ⚠️ | Inline validator exists (`PaymentWebhookController.php:157-163`) but `amount` is untyped and no schema artifact exists |
| Error format konsisten | ❌ | Every outcome — including invalid signature — returns HTTP 200 `{"success":true,...}` (`:44,56,66,75,85,94,103,111,132,149,171`), a different envelope than the API endpoints |
| Pagination/filter/sort | N/A | Single event |
| Idempotency | ✅ | `ref_id` unique (`...create_transactions_table.php:13`) + `lockForUpdate` inside one DB transaction (`PaymentWebhookController.php:198-207`) + status guard `already_processed`/`race` (`:227-246`) → duplicate delivery is a no-op; pinned by `tests/Feature/PaymentWebhookTest.php:299-305` |
| Rate limiting | ⚠️ | Tidak ada rate limit eksplisit di lapisan aplikasi maupun nginx untuk endpoint ini (keputusan desain: andalan berupa signature + IP whitelist opsional). **Detail konfigurasi disembunyikan untuk publikasi** |
| Breaking vs non-breaking change handling | ❌ | No version field, no change log, no gateway contract doc in repo |

### E6 — `POST {reseller_profiles.callback_url}` (outbound)

| Kriteria | Status | Evidence |
|---|---|---|
| Versioning eksplisit | ❌ | Payload has no version/event field (`SendResellerCallbackJob.php:52-56`) |
| Auth per endpoint | ❌ | Plain POST with no signature, secret, or timestamp (`SendResellerCallbackJob.php:59`) — the reseller cannot authenticate the sender |
| Request/response schema lengkap | ❌ | Three hard-coded fields; no schema artifact, no documented response expectation (`:52-67`) |
| Error format konsisten | ❌ | No error contract: response body/status ignored (`:61-67`); failures are only logged (`:69-75`) |
| Pagination/filter/sort | N/A | Single event |
| Idempotency | ⚠️ | Payload carries `ref_id`+`status` so a receiver *can* dedupe, but there is no event id/sequence/timestamp, and a callback fires on **every** status change (`TransactionObserver.php:13-19`) → retries with backoff can be delivered out of order with no way to order them |
| Rate limiting | N/A | Outbound job; retry policy is `tries=3`, `backoff=[10,60,300]` (`SendResellerCallbackJob.php:81-89`) but only on thrown (connection) exceptions |
| Breaking vs non-breaking change handling | ❌ | No version field, no deprecation path for payload changes |

---

## Design Principles

| Principle | Status | Evidence |
|---|---|---|
| Least surprise | ⚠️ | The webhook answers `200 {"success":true}` even when the signature is invalid (`app/Http/Controllers/PaymentWebhookController.php:44`), and a role denial returns `{"message":""}` — both surprising for an integrator, both pinned by tests (`tests/Feature/PaymentWebhookTest.php:111`; `tests/Feature/ResellerApiTest.php:76` asserts status only) |
| Small interfaces | ✅ | The whole machine interface is 4 verbs + 1 webhook — `routes/api.php:8-13` (13 lines) and `routes/web.php:46`; no speculative endpoints, no unused verbs |
| Uniform access | ⚠️ | Money is uniformly a 2-dp decimal string (`ProductController.php:38`, `TopupController.php:137,166`, `WalletService.php:14`), but envelopes are not uniform: `{data:[...]}` (`ProductController.php:44-46`) vs `{balance}` (`BalanceController.php:17-19`) vs `{ref_id,status,amount}` (`TopupController.php:134-138`) vs `{success,message}` (`PaymentWebhookController.php:132`) |
| DRY primitives | ✅ | One signature primitive reused for both lookup keys (`PaymentSignatureVerifier.php:25-30`, tried against `merchant_ref` then `reference` at `PaymentWebhookController.php:177-188`), one pricing primitive shared with storefront (`CalculatePriceForUser.php:18-45` used at `ProductController.php:26` and `TopupController.php:74`), one `ref_id` primitive shared with checkout (`RefIdGenerator.php:15`, used at `TopupController.php:88`) |
| Multi-interface per actor | ⚠️ | Reseller has 3 interfaces (REST API `routes/api.php:8-13`, admin-provisioned token UI `routes/web.php:101-103`, async callback `TransactionObserver.php:17`), while customers have only session HTML (`routes/web.php:31-66`) — appropriate per `PRD.md:193-201`, but the reseller's 3 interfaces are not described in one place (hence this doc) |

---

## Interface Scope & Agreement

**Scope (in).** Machine-to-machine surface only:

- E1–E4: reseller/H2H REST resources under `/api/v1`.
- E5: inbound payment-gateway webhook.
- E6: outbound reseller status callback.

**Scope (out).** Storefront, checkout, invoice, history, admin and owner panels
(`routes/web.php:31-129`): session cookie + CSRF + Blade/Livewire, i.e. a browser interface,
not an API contract. JSON is only produced there for exception rendering
(`bootstrap/app.php:29-31`).

**Interaction style.**

| Style | Used by | Notes |
|---|---|---|
| REST over JSON, stateless, Bearer token | E1–E4 | `Route::prefix('v1')` + `auth:sanctum` (`routes/api.php:8`) |
| Inbound webhook (fire-and-forget POST) | E5 | CSRF-exempt (`bootstrap/app.php:24-26`), receiver responds 200 unconditionally |
| Outbound asynchronous webhook (queue job) | E6 | `SendResellerCallbackJob` on status change, 5 s timeout, 3 attempts with backoff (`SendResellerCallbackJob.php:59,81-89`) |

**Data representation concerns.**

- Money is a **string**, 2 dp, no thousands separator: `"50000.00"` (`TopupController.php:137`,
  `ProductController.php:38`, `SendResellerCallbackJob.php:55`) — consistently applied, but
  undocumented anywhere in-repo, and `"amount"` in E5 is untyped input.
- Transaction status vocabulary is **two-sided**: lowercase app enum
  (`app/Enums/TransactionStatus.php:3-12`) in E3/E6, uppercase gateway literal `PAID` in E5
  (`PaymentWebhookController.php:217`). A consumer touching both must map them.
- No timestamps in any API response (`TopupController.php:134-138,163-173`,
  `BalanceController.php:17-19`, `ProductController.php:44-46`) → integrators cannot reconcile
  by time, only by polling `ref_id`.
- `ref_id` is the sole correlation key across E2/E3/E5/E6
  (`RefIdGenerator.php:21`; lookup at `PaymentWebhookController.php:200-203`).

**Error handling channel (verified, and inconsistent — a finding).**

| Interface | Channel | Status codes | Evidence |
|---|---|---|---|
| E1–E4 | Laravel default JSON: `{"message"}` or `{"message","errors"}` | Real HTTP codes (401/403/404/422) | No custom shape anywhere; `bootstrap/app.php:28-31` only chooses JSON vs HTML. Rendered shapes verified with `APP_DEBUG=false`: 401 `{"message":"Unauthenticated."}`, 403 `{"message":""}`, 422 `{"message":"<first error> (and N more errors)","errors":{...}}` |
| E5 | Custom envelope `{"success":true, "message"?, "errors"?}` | Always 200, including signature failure | `PaymentWebhookController.php:44,132,149,171` |
| E6 | No channel at all — sender ignores response | n/a | `SendResellerCallbackJob.php:61-67` |
| nginx layer | Plain-text `429` body from `limit_req` | 429 | `deploy/nginx-site.conf:33,50` |

Two different error envelopes plus a code-free webhook makes a single client error handler
impossible; this is the concrete form of "error format tidak konsisten".

---

## Cross-Module Contract Map

| Provider module | Consumer module | Endpoint/Event | Version | Breaking Risk |
|---|---|---|---|---|
| Payment gateway (external) | Payment module | `POST /webhook/payment-callback` | **None** — unversioned path and payload | **High** — tanpa version field, semua respons 200; perubahan nama field tidak terdeteksi penerima |
| Transaction/fulfillment module | Reseller H2H system (external) | `POST {reseller_profiles.callback_url}` on status change | **None** — payload `{ref_id,status,amount}` only | **High** — unsigned, unversioned, unordered; penambahan/penghapusan field merusak penerima tanpa pemberitahuan |
| Reseller API | Reseller H2H client (external) | `GET /api/v1/products`, `POST /api/v1/topup`, `GET /api/v1/topup/{ref_id}/status`, `GET /api/v1/balance` | `v1` path prefix | **Medium** — path berversi, tapi tidak ada artefak kontrak untuk deteksi drift, tidak ada kebijakan `/v2`/deprecation |
| Admin panel (session HTML) | Reseller API | Provisioning/revoke token | n/a (session UI) | Low — provisioning manual, token tidak pernah kadaluarsa |
| Distributor module | Digiflazz (external) | Outbound `POST {DISTRIBUTOR_ENDPOINT}` | External API (config-driven) | Medium — kontrak eksternal milik vendor, tidak ada di repo ini; stub mode default aktif |
| Storefront/admin (browser) | Web layer | Session HTML routes | n/a | Low — bukan machine contract |

---

## Findings

> **Catatan publikasi (versi publik):** tabel di bawah diringkas menjadi **kategori + severity**.
> Bukti berupa lokasi kode persis (`file.php:baris`) dan uraian akar masalah keamanan yang
> spesifik sengaja dihilangkan agar tidak menjadi peta serangan. Versi lengkap berisi bukti
> file-per-baris disimpan di repo privat.

| # | Kategori | Severity |
|---|---|---|
| 1 | Tidak ada artefak kontrak mesin (OpenAPI/Swagger) untuk endpoint mana pun; kontrak hanya prosa di PRD yang sudah drift dari route aktual | **High** |
| 2 | Tidak ada rate limiting di lapisan aplikasi untuk `E1–E4`, termasuk endpoint yang memindahkan dana wallet | **High** |
| 3 | Tidak ada idempotency key dari klien pada `E2` — retry setelah timeout bisa membuat order & potongan saldo kedua | **High** |
| 4 | Secret HMAC webhook bisa terdegradasi diam-diam menjadi nilai kosong saat env tidak di-set → verifikasi signature melemah | **High** |
| 5 | Signature webhook tanpa timestamp/nonce → request yang terekam bisa di-replay | Medium |
| 6 | Konvensi error tidak seragam: API memakai status code asli, webhook selalu 200, callback outbound tanpa kontrak error | Medium |
| 7 | Callback outbound (E6) unsigned & unversioned, tanpa event id — autentikasi pengirim tak bisa diverifikasi penerima, urutan tak terdeteksi | Medium |
| 8 | Gap retry pada E6: backoff yang terdokumentasi hanya berlaku untuk exception koneksi, bukan response 4xx/5xx | Medium |
| 9 | Tidak ada pagination/filter/sort pada `E1`; rate limit dan kebijakan tidak terdokumentasi di kontrak | Medium |
| 10 | Hygiene materi autentikasi: token reseller tidak pernah kadaluarsa dan tanpa kebijakan rotasi | Medium |
| 11 | Drift PRD ↔ implementasi route API (endpoint yang dijanjikan PRD tidak ada) | Medium |
| 12 | Body 403 kosong dari middleware role; `callback_url` tanpa validasi skema/host (SSRF-shaped); secret duplikat mati di config; tanpa header `RateLimit` pada respons API | Low |

---

## Recommendations

Ordered by severity (High → Low). Each item is self-contained for the next agent.
*Lokasi kode persis per butir sengaja dihilangkan pada versi publik.*

1. **R1 — High — Publish a machine-readable contract.** Buat `openapi.yaml` (OpenAPI 3.1) yang
   menutup E1–E6 memakai skema di [Endpoints](#endpoints); tambah CI step yang gagal saat daftar
   route dan daftar path di spec tidak sinkron. Rekonsiliasi §6 PRD dengan realita di perubahan
   yang sama (hapus/ticket endpoint yang tidak ada, dan putuskan nasib fitur "Dokumentasi API").
2. **R2 — High — Rate-limit `/api/v1/*` di lapisan aplikasi.** Daftarkan `RateLimiter::for('api', …)`,
   panggil `throttleApi()` dari bootstrap aplikasi, dan dokumentasikan nilainya. Pertahankan nginx
   `limit_req` sebagai defense in depth, dan dokumentasikan bahwa itu per-IP dengan 429 tanpa body.
3. **R3 — High — Buat `POST /api/v1/topup` idempotent bagi klien.** Terima idempotency key dari
   klien, simpan dengan unique index, dan saat tabrakan kembalikan body `201` semula alih-alih
   membuat transaksi & potongan wallet kedua. (Integritas internal via `ref_id` unique +
   `lockForUpdate` tetap seperti adanya.)
4. **R4 — High — Jangan pernah menjalankan webhook dengan secret HMAC kosong.** Tambahkan asersi
   saat boot bahwa secret non-empty (gagal di production), hapus jalur fallback `?? ''` pada
   verifier, dan buang secret duplikat default yang tidak pernah dibaca kode.
5. **R5 — Medium — Contract-kan callback reseller keluar.** Tambah `event_id`, `occurred_at`, dan
   HMAC signature pada payload; throw pada HTTP `>=400` supaya `tries`/`backoff` benar-benar
   berlaku; simpan baris delivery (jumlah attempt, status terakhir) agar callback terlewat bisa
   direkonsiliasi. Dokumentasikan payload di spec dengan field versi `v1`.
6. **R6 — Medium — Unifikasi error envelope.** Satu bentuk untuk `/api/v1` dan webhook
   (mis. RFC 9457 `application/problem+json`), berhenti membalas HTTP 200 untuk penolakan webhook,
   dan berikan pesan eksplisit pada `abort(403)` di kedua cabang middleware role.
7. **R7 — Medium — Version & dokumentasikan E5.** Tambah `api_version` (atau pindah ke
   `/webhook/v1/payment-callback`) dan catat setiap outcome diterima/ditolak beserta status HTTP-nya
   di spec, agar gateway dan sistem sepakat arti "accepted".
8. **R8 — Medium — Publikasikan rate limit.** Satu bagian spec per endpoint: limit, window, key
   (IP vs token), response code, dan perilaku `Retry-After`; selaraskan default rate limit aplikasi
   dengan angka yang terdokumentasi.
9. **R9 — Medium — Token lifecycle.** Setel TTL token, tentukan kebijakan rotasi/revocation, dan
   catat event issuance/revoke ke audit log yang sudah ada.
10. **R10 — Medium — Paginasi `GET /api/v1/products`.** Param page/per_page + filter `category`/`q`
    dengan default page size terdokumentasi; pertahankan `orderBy('id')` sebagai urutan stabil.
11. **R11 — Low — Perkeras `callback_url`.** Validasi skema/host saat write dan blokir address range
    privat; dokumentasikan siapa yang boleh mengaturnya.
12. **R12 — Low — Polesan kontrak kecil.** Echo `ref_id` di setiap respons webhook, tambah timestamp
    pada respons API yang harus direkonsiliasi integrator, dan dokumentasikan pemetaan kosakata
    status lowercase-aplikasi vs uppercase-gateway.

---

*Prepared for the API Contract Review. All claims above were verified by reading the cited
files; items that could not be verified are explicitly marked `Unverified` — currently: only
the deployment state of `deploy/nginx-site.conf` (template file, actual `/etc/nginx` state not
inspected) and the payment gateway's own external contract (no gateway spec exists in this
repository).*
