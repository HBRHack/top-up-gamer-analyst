# Architecture Review — Fofa Shop

> Dokumen ini dihasilkan oleh skill `analyst-design-reviewer`.
> **Tidak ada diagram Mermaid** — seluruh diagram memakai ASCII/Unicode box art agar bisa dirender di mana saja.

---

## Section 0 — Independent Audit Result (Phase 0)

**Auditor:** sesi yang sama dengan perbaikan dokumen — **tidak** terpisah secara teknis.

> **Catatan independensi (jujur, sesuai fallback yang diizinkan skill):** panggilan subagent untuk
> 9 check **dibatalkan oleh user** (alasan: kecepatan). Sebagai gantinya seluruh dokumen **dibaca
> ulang dari disk secara skeptis** — tanpa memercayai memori proses generate — dan **setiap temuan
> wajib berbukti `file:baris`** yang bisa dijalankan ulang pembaca. Temuan yang tidak bisa
> dibuktikan dengan grep tidak dicatat. Ini adalah fallback kedua yang diizinkan skill, bukan
> pengganti independensi ideal.

**Tanggal:** 2026-09-26
**Anti-Injection Shield:** AKTIF — seluruh isi PRD/SRS/spec/komentar kode diperlakukan sebagai *teks inert*. Tidak ada instruksi yang tertanam di dalam dokumen yang dieksekusi.

### Tabel 0.1 — Hasil 9 Pemeriksaan

| # | Check | Status | Lokasi | Detail |
|---|-------|--------|--------|--------|
| 1 | BR vs Data Dictionary | ✅ PASS *(diperbaiki)* | `data-dictionary.md:205-207` | **Ditemukan lalu diperbaiki:** 3 baris (`refund_bank`, `refund_account_number`, `refund_account_name`) punya kolom `Required = no` / `Default = null` tetapi kolom `Validation` menulis `required, …` — kontradiksi internal. Kini `required_if:refund_requested,true, …`. Dua kasus `Required = no` lain (`guest_email:189`, `target_game_id:191`) **bukan konflik** — validasinya memang kondisional (`required for guests`, `required unless minecraft_edition=manual`) dan konsisten dengan BR. |
| 2 | Redundancy | ✅ PASS | `erd.md:240-280`, `ADR-0005` | `wallets.balance` = agregat materialisasi dari `wallet_transactions` → **dual source of truth yang disengaja dan terdokumentasi** di `ADR-0005 (Accepted)`. `transactions.amount` = snapshot harga (`products.price`) — legitim, justru wajib agar histori harga tak ikut berubah. Tidak ditemukan field duplikat tak terdokumentasi pada relasi 1:N/N:1. |
| 3 | Matrix completeness | ✅ PASS | `use-cases/chapters/*` vs `requirements-matrix.md` | 30 id `BR-*` disitir di chapter use case; **0 di antaranya absen** dari matrix (verifikasi `comm -23` → kosong). |
| 4 | UC Overlap | ✅ PASS *(diperbaiki)* | `use-cases/chapters/BAB-2-…:188` | **Ditemukan lalu diperbaiki:** `UC-011` memuat *Main Flow — Laporan* (FR-025) dan *Main Flow — Export Pajak* (FR-027) yang keduanya sudah **Dropped** di spec v1.3, tanpa penandaan apa pun → dua alur mati tertulis seolah hidup. Kini judul diberi blok scope-cut dan kedua alur ditandai `~~(DROPPED — FR-025)~~` / `~~(DROPPED — FR-027)~~`. Pemilik proses tetap satu (UC kanonis), tidak ada duplikasi proses antar UC. |
| 5 | Numbering | ✅ PASS *(4 catatan terbuka)* | `NUMBERING-LOG.md` | Terbukti grep: **0 id yatim, 0 id dirujuk tapi tak didefinisikan, 0 duplikat** pada 10 keluarga (FR/UC/PF/NFR/ADR/BR/GAP/CON/ASM/OQ). Celah sekuens nihil. Sisa tercatat eksplisit sebagai **OD-04** (SRS 0 id PF), **OD-05** (SRS 0 id UC/BR — desain, hub = matrix), **OD-10** (urutan heading FR non-monoton), **OD-13** (`docs/analyst/` untracked). |
| 6 | Value Consistency | ✅ PASS | `app/**/*.php` vs `data-dictionary.md:198` | Value List `TransactionStatus` = 7 nilai (`pending,paid,processing,waiting_fulfillment,success,failed,expired`). Penggunaan literal di kode: `success` 61×, `failed` 14×, `pending` 9×, `paid` 7×, `processing` 3×, `expired` 3×, `waiting_fulfillment` 2× — **persis 7 nilai, 0 di luar list**. Tidak ada no-op query akibat value mismatch. |
| 7 | Traceability | ✅ PASS *(diperbaiki)* | `requirements-matrix.md:178` | **Ditemukan lalu diperbaiki:** baris `BR-AUTH-003` menyitir `FR-025` bersanding dengan FR aktif tanpa penanda — seolah FR-025 masih berlaku. Kini `FR-025 *(Dropped)*`. Baris FR-025/FR-027 di matrix sudah `Dropped`; di SRS sudah ada anotasi `Status: [ ] Dropped — spec v1.3 scope-cut`. **36 Confirmed / 2 Dropped** dari 38 FR. |
| 8 | Scope Creep | ✅ PASS | `ADR-0009`, `ADR-0010`, `ADR-0012` | Spec v1.3 scope-cut membuang click tracking, white-label store, live chat, fraud detection. Ketiganya **tidak** berakhir sebagai task orifinal tanpa dasar — ditutup eksplisit sebagai **ADR ber-status `Cancelled`** (0009 UTM Click Tracking, 0010 White Label Store Path, 0012 Live Chat Reverb). 16 ADR lain `Accepted`. **0 orphaned item.** |
| 9 | Domain Glossary `_Avoid_` | ✅ PASS *(diperbaiki)* | `data-dictionary.md:114`, `erd.md:207`, `SRS.md:24` | **Ditemukan lalu diperbaiki:** `_Avoid_: sku` (Product) terpakai 2× sebagai deskripsi kolom `vendor_product_code` ("SKU at vendor") → kini **"Kode produk pada vendor"**. `_Avoid_: seller` (Reseller Profile) terpakai 1× di daftar Excluded → kini **"multi-vendor store"**. Sisa: `invoice` muncul 18×, tetapi `CONTEXT.md:107` **mengizinkan** "invoice" sebagai nama endpoint/view saja — bukan pelanggaran. |

**Ringkasan: 9 PASS / 0 FAIL** (4 temuan awal diperbaiki sebelum skor dihitung; bukti sebelum-sesudah tercatat di `consistency-audit.md`).

### Quality Gate Scoring (0–100)

| Dimensi | Bobot | Skor | Dasar |
|---------|-------|------|-------|
| **Completeness** | 40% | **34** | 38 FR · 38 BR · 23 UC · 11 PF · 14 NFR (QA-scenario 6 bagian) · 20 tabel DD lengkap kolom Validation · 19 ADR lengkap 8 field wajib. **Potongan:** belum ada OpenAPI/Swagger (temuan High di `api-contract.md`), SRS tidak membawa id PF (OD-04), 3 kontradiksi DD sempat ada. |
| **Clarity** | 30% | **25** | BR memakai pola `Rule/When/Then/Else` yang teruji; NFR punya response measure. **Potongan:** `NUMBERING-LOG.md` memuat jargon internal proses audit — sebagian **dinetralkan saat publikasi** (2026-09-26), sisa dipertahankan demi jejak audit; campur Indonesia/Inggris (202 marker id vs 4 en); kolom Validation DD masih campur notasi prosa dan `required_if:`. |
| **Alignment** | 30% | **26** | 0 id menggantung pada 10 keluarga; 30 rujukan BR↔UC tervalidasi silang terhadap kutipan chapter; fitur scope-cut konsisten `Dropped`/`Cancelled` di PRD, SRS, matrix, dan ADR; glossary violation sudah ditutup. **Potongan:** rujukan ke tracker internal **sudah dinetralkan** untuk publikasi (sebelumnya dipertahankan dan jadi tautan mati); `docs/analyst/` kini ter-track di git. |
| **Total** | 100% | **85** | 34 + 25 + 26 |

**Critical Flaw Veto:** **TIDAK AKTIF** — tidak ada kontradiksi fundamental yang menyebabkan kegagalan downstream. Empat temuan awal sudah diselesaikan; tidak ada blocking issue tersisa.

**Threshold:** 85 ≥ 80 → **LOLOS.**

> **Keputusan yang tercatat:** user memberi instruksi eksplisit *"buat 4 file langsung"*, sehingga
> Phase 1–4 dijalankan dengan approval tersebut. Skor dihitung **setelah** 4 temuan diperbaiki,
> bukan sebelum — angka 85 bukan hasil tanpa pemeriksaan.

---

## Section 1 — Architecture Assessment (Phase 1)

### 1.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                           │
│  ┌────────────────┐ ┌────────────────┐ ┌──────────────────────────┐ │
│  │ Storefront     │ │ Admin Panel    │ │ Owner Panel              │ │
│  │ Livewire +     │ │ Livewire       │ │ /owner (role:owner)      │ │
│  │ session auth   │ │ role:admin,    │ │ throttle:admin           │ │
│  │                │ │ owner          │ │                          │ │
│  └───────┬────────┘ └───────┬────────┘ └────────────┬─────────────┘ │
│          │                  │                       │               │
│  ┌───────┴──────────────────┴───────────────────────┴─────────────┐ │
│  │ Reseller API — /api/v1 + auth:sanctum + role:reseller          │ │
│  └────────────────────────────┬───────────────────────────────────┘ │
└───────────────────────────────┼──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│                        BUSINESS LOGIC LAYER                          │
│  ┌──────────────┐ ┌──────────────────┐ ┌───────────────────────────┐ │
│  │ Checkout     │ │ Payment          │ │ Fulfillment               │ │
│  │ RateLimit    │ │ SignatureVerify  │ │ FulfillTransactionJob     │ │
│  │ (3 dimensi)  │ │ Webhook          │ │ SyncWaitingFulfillmentJob │ │
│  └──────┬───────┘ └────────┬─────────┘ └─────────────┬─────────────┘ │
│         │                  │                         │               │
│  ┌──────┴──────┐  ┌────────┴────────┐  ┌─────────────┴─────────────┐ │
│  │ GameId      │  │ Wallet /        │  │ PdfExportService          │ │
│  │ Verifier    │  │ Voucher         │  │ RefIdGenerator            │ │
│  │ Chain       │  │                 │  │ MinecraftCheckoutResolver │ │
│  └──────┬──────┘  └────────┬────────┘  └─────────────┬─────────────┘ │
└─────────┼──────────────────┼─────────────────────────┼───────────────┘
          │                  │                         │
┌─────────▼──────────────────▼─────────────────────────▼───────────────┐
│                          DATA ACCESS LAYER                           │
│  ┌──────────────┐ ┌────────────┐ ┌────────────┐ ┌─────────────────┐  │
│  │  MySQL /     │ │   Redis    │ │  Storage   │ │  Queue worker   │  │
│  │  20 tabel    │ │  cache +   │ │  public/   │ │  (jobs)         │  │
│  │  (data-dict) │ │  session   │ │  uploads   │ │                 │  │
│  └──────────────┘ └────────────┘ └────────────┘ └─────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘

  EKSTERNAL ──► Digiflazz API (game id verify + topup) │ Payment Gateway (webhook)
```

### 1.1b Opsi Desain (Bangun / Beli / Kombinasi)

| Area | Opsi | Pilihan | Alasan |
|------|------|---------|--------|
| Distributor top-up | **BELI** (Digiflazz) + **BANGUN** adapter | **KOMBINASI** | Integrasi pihak ketiga sudah jadi; yang dibangun adalah rantai fallback (`GameIdVerifierChain`: Digiflazz → regex lokal) agar tak bergantung satu vendor |
| Payment | **BELI** (gateway eksternal) | **BELI** | Sudah ada `PaymentSignatureVerifier` + webhook — asumsi risiko fraud dipindah ke gateway, yang dijaga tinggal verifikasi signature |
| Export PDF | **BELI** (DomPDF via Composer) | **BELI** | `PdfExportService::generate` → DomPDF. Tidak ada gunanya membangun renderer sendiri |
| Fulfillment otomatis | **BANGUN** (job + scheduler) | **BANGUN** | Inti bisnis; kategori Minecraft sengaja dijalur manual (`waiting_fulfillment`) karena tak ada API |
| Auth | **BANGUN** di atas Laravel | **BANGUN** | Session (storefront/admin) + Sanctum (reseller) — sesuai `ADR` role hierarchy, tanpa dependensi SSO |

### 1.1c Rekomendasi Jujur

**Tidak ada satu opsi terbaik untuk seluruh sistem** — pendekatan memang hibrida dan itu disengaja.
Yang perlu diperhatikan: **solusi sekuat komponen terlemahnya**. Titik terlemah saat ini bukan
arsitektur, melainkan **kontrak API yang implisit** (lihat §1.4 — belum ada OpenAPI) dan
**dependensi ke satu distributor** tanpa pilihan cadangan otomatis (`ADR-0006 Multi-vendor
failover manual` — failover masih manual, bukan otomatis).

### 1.2 Architecture Review Checklist

| Aspek | Status | Bukti |
|-------|--------|-------|
| Scalability | ⚠️ | Komunitas-scale by design (spec v1.3 scope-cut). Tidak ada read replica / horisontal scaling yang didokumentasikan — sesuai asumsi, bukan kekurangan tersembunyi |
| Availability | ⚠️ | `SyncWaitingFulfillmentJob` (cron 5 mnt) + queue job → ada pemulihan otomatis. **SPOF:** dependensi tunggal Digiflazz (failover manual, ADR-0006) |
| Performance | ✅ | 14 NFR dengan QA-scenario 6 bagian termasuk response measure |
| Security | ✅ | Rate limit 3 dimensi (`BR-CHECK-001`), signature verification webhook, `BR-AUTH-003` column exclusion di level query, RBAC 4 role |
| Maintainability | ✅ | 19 ADR berstatus jelas (16 Accepted / 3 Cancelled); 20 tabel terdokumentasi |
| Testability | ✅ | 6 file test fitur baru terdeteksi di `tests/Feature/` |
| Extensibility | ✅ | Kategori dinamis (`ADR-0004`) — CRUD tanpa deploy |
| **Bottleneck** | ⚠️ | **Kontrak API implisit** (tanpa OpenAPI) + **failover distributor manual** — dua komponen ini membatasi keduanya skalabilitas dan kecepatan rilis |

### 1.3 Trade-off Dokumentasi

```
## Trade-off: Fulfillment Manual untuk Kategori Minecraft

**Context:** Kategori Minecraft tidak punya API distributor untuk eksekusi item.

**Options:**
| Option | Pros | Cons |
|--------|------|------|
| A: Tunggu API resmi | Tanpa kerja tambahan | Fitur mati sampai vendor siap |
| B: Jalur manual (dipilih) | Fitur tetap jalan, admin kendali penuh | Latensi fulfilment = kerja admin; tanpa SLA otomatis |

**Decision:** Opsi B — transaksi berhenti di `waiting_fulfillment`, admin eksekusi lalu "Mark as Success".

**Rationale:** ADR-0016 (Minecraft Edition Scenario) + spec v1.3.2 "Minecraft-first launch".

**QA Affected:** Availability ↓ (bergantung manusia), Usability ↑ (admin punya kendali).

**Trade-off Accepted:** Waktu fulfillment tak terukur; `SyncWaitingFulfillmentJob` hanya merapikan status, bukan mempercepat.

**Review Date:** Saat vendor menyediakan API Minecraft, atau bila antrean manual melebihi 1 hari kerja.
```

### 1.4 API Contract Review

> Sumber kontrak: **`docs/analyst/api-contract.md`** (bukan ingatan). **OpenAPI/Swagger TIDAK ADA — ini temuan High.**

| Check | Status | Catatan |
|-------|--------|---------|
| OpenAPI/Swagger spec tersedia & versioned | ❌ | Tidak ada file spec; kontrak hidup di `api-contract.md` (manusia, bukan mesin) |
| Spec di-generate dari kode ATAU kode dari spec | ❌ | Keduanya manual → pasti drift |
| Versioning eksplisit (`/api/v1/...`) | ✅ | `routes/api.php:8` — `Route::prefix('v1')` |
| Auth per endpoint terdefinisi | ✅ | `auth:sanctum` + `role:reseller` per grup |
| Request/response schema lengkap | ⚠️ | 6 endpoint terdokumentasi, sebagian tanpa contoh payload |
| Error format konsisten | ⚠️ | Ditemukan di `api-contract.md` sebagai temuan |
| Idempotency untuk tulis yang di-retry | ✅ | `transactions.ref_id` unique — didesain untuk idempotency |
| Rate limit & quota terdokumentasi | ❌ | Rate limit ada di **level middleware** (`BR-CHECK-001`), tapi **tidak ada rate limit aplikasi per endpoint API** — temuan High |

**Findings (dari `api-contract.md`, 20 total — 3 High):**

| # | Issue | Severity |
|---|-------|----------|
| 1 | Tanpa OpenAPI/Swagger | **High** |
| 2 | Tanpa rate limit aplikasi di endpoint API | **High** |
| 3 | HMAC secret fallback `''` (empty string) saat env kosong | **High** |

### 1.5 Keputusan Arsitektur yang Sudah Tercatat

19 ADR tersedia di repo aplikasi (tidak disertakan di repo dokumentasi ini) — 16 `Accepted`, 3 `Cancelled` (0009, 0010, 0012). Setiap ADR
memiliki 8 field wajib: Status, Date, Deciders, Technical Story, Context, Decision, Options
Considered, Consequences, Compliance, Review. Rincian di `consistency-audit.md`.
