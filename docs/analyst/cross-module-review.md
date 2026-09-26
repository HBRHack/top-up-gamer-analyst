# Cross-Module Technical Review — Fofa Shop

> Phase 3 dari `analyst-design-reviewer`. **Tanpa Mermaid** — diagram memakai ASCII/Unicode box art.

---

## 1. Module Dependency Map

```
                    ┌──────────────────────────────────┐
                    │           WEB ROUTES             │
                    │   routes/web.php · routes/api.php│
                    └───────────────┬──────────────────┘
                                    │ middleware: auth · role · throttle
        ┌───────────────┬───────────┼───────────────┬─────────────────┐
        ▼               ▼           ▼               ▼                 ▼
 ┌─────────────┐ ┌─────────────┐ ┌──────────┐ ┌────────────┐ ┌──────────────┐
 │ Storefront  │ │ Admin       │ │ Owner    │ │ Auth       │ │ Api\V1       │
 │ Controller  │ │ Livewire/*  │ │ Livewire │ │ (session)  │ │ (sanctum)    │
 └──────┬──────┘ └──────┬──────┘ └────┬─────┘ └─────┬──────┘ └──────┬───────┘
        │               │             │             │               │
        └───────────────┴──────┬──────┴─────────────┴───────────────┘
                               ▼
        ┌──────────────────────────────────────────────────────────┐
        │                      SERVICE LAYER                       │
        │  CheckoutController ─── CheckoutRequest ─── RateLimit     │
        │  GameIdCheck ─── GameIdVerifierChain ──┬── Digiflazz      │
        │                                       └── FallbackRegex   │
        │  PaymentWebhook ── PaymentSignatureVerifier               │
        │  RefundRequest ── Wallet / WalletTransaction              │
        │  Admin\Transaction ── PdfExportService                    │
        │  Checkout ── MinecraftCheckoutResolver ── Mail            │
        └───────────────────────────┬──────────────────────────────┘
                                    ▼
        ┌──────────────────────────────────────────────────────────┐
        │                       MODEL LAYER                        │
        │  User ─┬─ ResellerProfile (1-1) ─┬─ Wallet (1-1)          │
        │        └─ Transaction (1-N)      └─ WalletTransaction     │
        │  Category ─┬─ Product ─┬─ ProductVendorMapping            │
        │            └─ Subcategory                                 │
        │  Transaction ─┬─ Payment (1-N) ─┬─ DistributorLog (1-N)   │
        │               └─ VoucherUsage                                │
        │  Voucher · Announcement · AdminPaymentMethod              │
        │  AuditLog · ProductDeletionLog · Notification             │
        └──────────────────────────────────────────────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
       ┌────────────┐       ┌────────────┐        ┌──────────────┐
       │   MySQL    │       │   Redis    │        │ Storage      │
       │  20 tabel  │       │ cache/queue│        │ public/…     │
       └────────────┘       └────────────┘        └──────────────┘
```

### Dependency Map (tabel)

| Module A | Module B | Tipe | Risk |
|----------|----------|------|------|
| Storefront | Product, Category, Subcategory | Data (read) | Low |
| Storefront (Checkout) | Transaction, Payment, Voucher, Wallet | Sync + Data | **Med** — jalur uang |
| Api\V1 | Product, Transaction, Wallet | Data + Function | **Med** — kontrak publik |
| PaymentWebhook | Transaction, Payment | Sync + Data | **High** — entry point eksternal |
| GameIdVerifierChain | Digiflazz (eksternal) | Sync eksternal | **Med** — bergantung vendor |
| Admin Livewire | Semua entity | Data (CRUD) | Low–Med |
| Owner Livewire | Transaction, AuditLog, ProductDeletionLog | Data | **Med** — hak istimewa |
| Fulfillment jobs | Transaction, DistributorLog | Async + Data | **Med** |

**Jumlah modul > 3** → cross-module review diwajibkan; terpenuhi.

---

## 2. Interface Contracts

| Provider | Consumer | Interface | Version | Status |
|----------|----------|-----------|---------|--------|
| `Api\V1\ProductController` | Reseller external | `GET /api/v1/…` | v1 | Stable |
| `Api\V1\TopupController` | Reseller external | `POST /api/v1/…` | v1 | Stable |
| Payment Gateway | `PaymentWebhookController` | HTTPS webhook + signature | — | Stable |
| Digiflazz | `GameIdVerifierChain`, fulfillment job | REST external | — | **Coupled** (failover manual, ADR-0006) |
| Storefront | Livewire components | internal | — | Changing |

### Breaking-Change Risk

| Endpoint | Risk | Alasan |
|----------|------|--------|
| Semua `/api/v1/*` | **High** | Consumer eksternal (reseller) sudah bergantung; **belum ada OpenAPI** sehingga perubahan tak terdeteksi sampai rusak di sisi consumer (Hyrum's law) |
| Webhook | **High** | Format ditentukan gateway; perlu contract test |
| Livewire internal | Low | Consumer internal saja |

---

## 3. Scalability Assessment

| Metric | Current | Target | Gap |
|--------|---------|--------|-----|
| Concurrent users | belum diukur | komunitas-scale (spec v1.3) | **Tidak didokumentasikan** |
| Data volume | belum diukur | — | Tidak didokumentasikan |
| Request/response | belum diukur (persentil) | — | Tidak didokumentasikan |

**Temuan (Medium):** tidak ada baseline kapasitas maupun target persentil (p95/p99) yang
tercatat. NFR punya response measure, tetapi **angka kapasitas awal** tidak ada — sehingga
"apakah arsitektur sanggup 10×" tidak bisa dijawab dengan data.

### Scaling Strategy (belum terdefinisi)

| Component | Strategy | Trigger | Action |
|-----------|----------|---------|--------|
| — | — | — | **Belum ada** — konsisten dengan asumsi skala komunitas, tapi wajib dicatat sebagai keterbatasan |

### Bottleneck Analysis

| Komponen | Beban | Breaking Point | Mitigasi |
|----------|-------|----------------|----------|
| Digiflazz API | semua top-up + game id check | rate limit vendor | Fallback regex (`GameIdVerifierChain`); **failover antar-vendor masih manual** (ADR-0006) |
| Admin manual fulfillment | semua transaksi Minecraft | kapasitas admin | Tidak ada otomasi — disengaja (ADR-0016) |
| Kontrak API | semua consumer reseller | drift tak terdeteksi | **Belum dimitigasi** — tanpa OpenAPI |

---

## 4. Security Architecture Review

### 4.1 Authentication Flow

| Step | Component | Security Check | Status |
|------|-----------|----------------|--------|
| 1 | Login (session) | Credential + `email_verified_at` | ✅ |
| 2 | `RoleMiddleware` | Cek role + `is_active`; gagal → 403 | ✅ `business-rules.md:955` |
| 3 | Inactive user | Diblokir di middleware | ✅ |
| 4 | Reseller API | Sanctum token → `users.id` | ✅ |
| 5 | Webhook | `PaymentSignatureVerifier` | ✅ |
| 6 | HMAC secret | **fallback `''` bila env kosong** | ❌ High — `api-contract.md` |

### 4.2 Authorization Matrix

| Role | Storefront | Admin | Owner | API Reseller |
|------|-----------|-------|-------|--------------|
| customer | ✅ | ❌ 403 | ❌ 403 | ❌ |
| reseller | ✅ | ❌ 403 | ❌ 403 | ✅ (`role:reseller`) |
| admin | ✅ | ✅ | ❌ 403 | ❌ |
| owner | ✅ | ✅ | ✅ | ❌ |

**Pembatasan finansial (`BR-AUTH-003`):** admin **tidak bisa** membaca kolom `amount`/`omset` —
dikecualikan **di level query**, bukan sekadar disembunyikan di UI. Owner akses penuh.
Rujukan: `ADR-0015 Financial Data Access Control` (Accepted).

### 4.3 Data Flow Security

| Data Type | Source | Destination | Encryption | Access Control |
|-----------|--------|-------------|------------|----------------|
| Kredensial login | Form | MySQL `users` | hash (bcrypt) | session |
| Token reseller | Admin panel | `reseller_profiles.api_key/api_secret` | — | role:owner |
| Data finansial | MySQL | Admin/Owner view | — | **column exclusion query** (`BR-AUTH-003`) |
| Webhook payload | Gateway | `PaymentWebhookController` | signature HMAC | verifier |
| Data game id | User | Digiflazz (eksternal) | HTTPS | — |
| Upload produk | Admin | `public/products` | — | role:admin,owner; JPG/PNG/WebP ≤2MB |

### 4.4 Threat Assessment

> **Catatan publikasi (versi publik):** kolom "Mitigation" diringkas. Detail akar masalah dan
> lokasi kode untuk temuan berstatus ❌ sengaja tidak dipublikasikan — tersimpan di repo privat.

| Threat | Likelihood | Impact | Mitigation | Status |
|--------|-----------|--------|------------|--------|
| Brute-force checkout | Med | Med | Rate limit 3 dimensi (`BR-CHECK-001`) | ✅ |
| Webhook palsu | Low | **High** | Signature verification | ✅ |
| Konfigurasi secret signature tidak fail-fast | Low | **High** | Menunggu perbaikan — **High finding** | ❌ **High finding** |
| Eskalasi role | Low | High | `RoleMiddleware` + 403 | ✅ |
| Bocor data finansial ke admin | Med | High | Column exclusion di query | ✅ |
| Tanpa rate limit aplikasi di API reseller | **Med** | Med | **Belum diterapkan** | ❌ **High finding** |
| Data race saat idempotency | Low | High | `ref_id` unique | ✅ |

---

## 5. Potential Issues

| Issue | Modules Affected | Severity | Recommendation |
|-------|------------------|----------|----------------|
| Kontrak API implisit (tanpa OpenAPI) | `Api\V1`, consumer reseller | **High** | Generate spec dari kode; jadikan CI check |
| Konfigurasi secret signature tanpa fail-fast | Payment module | **High** | Exception bila env kosong — jangan pernah verifikasi dengan secret kosong |
| Tanpa rate limit aplikasi di API | `Api\V1` | **High** | Terapkan `throttle:` per endpoint |
| Failover distributor manual | Fulfillment, GameId | **Med** | Sudah diputuskan di ADR-0006; pastikan runbook eksplisit |
| Tautan ke tracker internal (`.scratch/`) — diringkas pada versi publik | seluruh `docs/analyst/` | **Med** | **Keputusan pemilik: dinetralkan saat publikasi** |
| Kapasitas & persentil tak terukur | seluruh sistem | **Med** | Catat baseline p50/p95 sebelum skala naik |

---

## 6. Review Per-Endpoint API

| Endpoint | Versioning | Auth | Schema | Error | Pagination | Idempotency | Rate Limit |
|----------|-----------|------|--------|-------|------------|-------------|-----------|
| `GET /api/v1/products` | ✅ v1 | ✅ sanctum+role | ⚠️ | ⚠️ | ⚠️ | n/a | ❌ |
| `POST /api/v1/topup` | ✅ v1 | ✅ sanctum+role | ⚠️ | ⚠️ | n/a | ✅ `ref_id` | ❌ |
| `GET /api/v1/status/{ref}` | ✅ v1 | ✅ sanctum+role | ⚠️ | ⚠️ | n/a | n/a | ❌ |
| `GET /api/v1/balance` | ✅ v1 | ✅ sanctum+role | ⚠️ | ⚠️ | n/a | n/a | ❌ |

Kolom ⚠️ = terdokumentasi di `api-contract.md` tapi tidak dalam format schema resmi.

---

## 7. Catatan Ketidaksempurnaan (Phase 3)

- **Baseline kapasitas kosong** — tidak ada angka concurrent user / volume / persentil yang tercatat, sehingga §3 sebagian besar berupa keterbatasan, bukan analisis berbasis data.
- **Skalabilitas tak didefinisikan** — konsisten dengan asumsi komunitas-scale, tapi berarti pertanyaan "sanggup 10×?" **belum bisa dijawab**.
- **3 temuan High bersifat kode**, bukan dokumen — perbaikannya menyentuh `app/`, di luar scope audit dokumentasi.
- **Tidak ada contract test** terdeteksi untuk webhook maupun endpoint reseller.
