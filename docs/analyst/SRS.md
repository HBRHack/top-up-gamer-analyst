# SRS: Fofa Shop

Website top-up game berbasis reseller H2H — beli produk digital dari distributor via API, jual ke end user dengan markup.

## 1. System Overview

### 1.1 Purpose

Fofa Shop menyelesaikan masalah pembelian voucher/diamond/game currency secara instan dengan model reseller H2H. Sistem mengintegrasikan distributor (Digiflazz), payment gateway (Tripay/Duitku), dan menyediakan automasi fulfillment.

### 1.2 Scope

**Included:**
- Storefront (customer checkout, invoice, riwayat)
- Reseller API (H2H top-up via token)
- Admin panel (CRUD, transaksi, wallet, voucher)
- Owner panel (laporan keuangan, manajemen role)
- Email notifikasi (9 template)
- Audit logging

**Excluded:**
- Mobile apps
- Multi-currency
- Marketplace / multi-vendor store
- Live chat (cancelled)
- Fraud detection (cancelled)
- Click tracking analytics (cancelled)

### 1.3 Actors

| Actor | Description | Primary Interface |
|-------|-------------|-------------------|
| Customer | Beli produk via storefront | Web (session) |
| Reseller | Beli via API dengan harga khusus | REST API (Sanctum token) |
| Admin | Kelola sistem, operasional harian | Web (session) |
| Owner | Pemilik, akses penuh + finansial | Web (session) |
| Payment Gateway | Kirim callback pembayaran | Webhook |
| Digiflazz API | Distributor produk digital | HTTP API |

---

## 2. Functional Requirements

### 2.1 Storefront

#### FR-001: Browse Kategori Game
- **Description:** Homepage menampilkan grid kategori game dengan icon
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-002: Search Produk
- **Description:** Full-text search berdasarkan nama produk dan kategori
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-003: Cek Nickname/ID Game
- **Description:** Verifikasi game ID via Digiflazz API dengan fallback regex
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-004: Detail Produk + Harga Per Role
- **Description:** Tampilkan produk dengan harga berdasarkan role (customer/reseller/admin)
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-005: Checkout + Validasi
- **Description:** Form checkout dengan validasi game ID, payment method, voucher
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-006: Invoice Page
- **Description:** Halaman invoice dengan status, countdown, payment details
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-007: Refund Request
- **Description:** Form submit refund untuk transaksi gagal (customer manual)
- **Priority:** Should Have
- **Status:** [x] Confirmed

#### FR-008: Riwayat Transaksi
- **Description:** Daftar riwayat transaksi user + re-check status
- **Priority:** Must Have
- **Status:** [x] Confirmed

### 2.2 Reseller API

#### FR-009: List Produk (API)
- **Description:** Daftar produk dengan harga reseller
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-010: Top-up via API
- **Description:** Buat transaksi top-up via API, deduct wallet atomik
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-011: Cek Status (API)
- **Description:** Query status transaksi
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-012: Cek Saldo (API)
- **Description:** Query saldo wallet
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-013: Webhook Callback
- **Description:** POST callback ke reseller URL saat status berubah
- **Priority:** Should Have
- **Status:** [x] Confirmed

### 2.3 Admin Panel

#### FR-014: Dashboard Monitoring
- **Description:** Statistik transaksi count-based (tanpa finansial untuk admin)
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-015: CRUD Produk/Kategori/Subkategori
- **Description:** Kelola produk, kategori, subkategori
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-016: Retry Transaksi Gagal
- **Description:** Re-dispatch FulfillTransactionJob
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-017: Manual Fulfill Minecraft
- **Description:** Simpan credentials + kirim email (waiting_fulfillment → success)
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-018: Kelola Wallet Reseller
- **Description:** Deposit saldo ke reseller
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-019: Toggle Payment Method
- **Description:** Aktif/nonaktifkan metode pembayaran
- **Priority:** Should Have
- **Status:** [x] Confirmed

#### FR-020: Broadcast Pengumuman
- **Description:** In-app notification + email blast ke target audience
- **Priority:** Should Have
- **Status:** [x] Confirmed

#### FR-021: Audit Log
- **Description:** Log semua aksi CRUD admin/owner
- **Priority:** Should Have
- **Status:** [x] Confirmed

#### FR-022: Kelola Voucher
- **Description:** CRUD voucher (percentage/fixed)
- **Priority:** Should Have
- **Status:** [x] Confirmed

#### FR-023: Manage Refund Requests
- **Description:** Daftar dan proses refund
- **Priority:** Should Have
- **Status:** [x] Confirmed

#### FR-034: Transaction Chart
- **Description:** Line chart transaksi per hari/minggu/bulanan di halaman admin transactions (Chart.js CDN, data count-based, filter periode mempengaruhi chart DAN tabel)
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-036: Product Image Upload
- **Description:** Upload gambar produk (jpg/jpeg/png/webp, max 2048 KB) + hapus/replace gambar via form Livewire admin
- **Priority:** Should Have
- **Status:** [x] Confirmed

### 2.4 Owner Panel

#### FR-024: View Omset
- **Description:** Lihat total omset di dashboard (owner only)
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-025: Laporan Keuangan
- **Description:** Omset, cost, margin per periode — **tidak lagi jadi requirement**: spec v1.3 Out of Scope membuang standalone finance reports dashboard (`/owner/reports/*`); ringkasan omset tetap ada di FR-024
- **Priority:** Must Have
- **Status:** [ ] Dropped — spec v1.3 scope-cut (laporan keuangan standalone dihapus)

#### FR-026: Export PDF
- **Description:** Export transaksi sebagai PDF (A4 landscape)
- **Priority:** Should Have
- **Status:** [x] Confirmed

#### FR-027: Export Pajak
- **Description:** Export data pajak (CSV/Excel + PDF) — **tidak lagi jadi requirement**: spec v1.3 Out of Scope membuang tax calculation (PPN 11%) dan CSV/Excel export (`maatwebsite/excel`); export PDF ada di FR-026
- **Priority:** Could Have
- **Status:** [ ] Dropped — spec v1.3 scope-cut (tax calculation & CSV/Excel export dihapus)

#### FR-028: Kelola Admin
- **Description:** Buat/toggle akun admin lain
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-029: Hapus Produk (Unrestricted)
- **Description:** Hapus produk kapan saja tanpa batasan 24 jam
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-035: Financial Data Access Control
- **Description:** Pembatasan akses data finansial di level query (column exclusion): admin tidak memilih kolom `amount`/omset, owner akses penuh — bukan sekadar UI hide
- **Priority:** Must Have
- **Status:** [x] Confirmed

### 2.5 Customer Features

#### FR-030: Voucher Diskon
- **Description:** Input kode voucher di checkout, diskon percentage/fixed
- **Priority:** Should Have
- **Status:** [x] Confirmed

#### FR-031: Email Notifikasi
- **Description:** Email untuk: paid, success, failed, expired, reconciliation
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-032: Notification Preference
- **Description:** Toggle opt-out untuk email pengumuman
- **Priority:** Could Have
- **Status:** [x] Confirmed

#### FR-033: Minecraft Credential Reveal
- **Description:** On-demand password reveal untuk akun Minecraft yang dikirim
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-037: Guest Email saat Checkout
- **Description:** Guest wajib isi email saat checkout (`required|email`) — dipakai untuk notifikasi status transaksi; user login memakai `users.email`
- **Priority:** Must Have
- **Status:** [x] Confirmed

#### FR-038: Theme Toggle (Terang/Gelap)
- **Description:** Toggle tema light/dark, persist ke localStorage (`fofa-theme`), ikut system default (`prefers-color-scheme`) — pure presentational
- **Priority:** Could Have
- **Status:** [x] Confirmed

---

## 3. Non-Functional Requirements

Ringkasan 14 NFR ber-ID. Skenario QA 6 bagian (Sumber Stimulus · Stimulus · Environment · Artifact · Respons · Ukuran Respons), gap kejelasan (GAP-01…GAP-12), dan kutipan verbatim §3 ada di **`docs/analyst/nfr.md`** — dokumen itu satu-satunya sumber lengkapnya.

| ID | Target singkat | Detail |
|----|----------------|--------|
| NFR-PERF-001 | Response time < 200ms (cached) | `nfr.md` §1 — cek game ID (cache 300s) & `/search` |
| NFR-PERF-002 | Throughput ~60 req/s (2 vCPU, 2GB) | `nfr.md` §1 — `deploy/OPTIMIZATION.md`; system-wide |
| NFR-PERF-003 | Cache hit rate > 80% product listings | `nfr.md` §1 — `config/cache.php` Redis; metrik (GAP-05) |
| NFR-SEC-001 | Jalur checkout aman | `nfr.md` §2 — CSRF, rate limit, HMAC signature, bcrypt, encrypted cast, RoleMiddleware |
| NFR-SEC-002 | Auth & otorisasi API reseller | `nfr.md` §2 — Sanctum + `role:reseller` (`routes/api.php`) |
| NFR-SEC-003 | Anti SQL injection & XSS | `nfr.md` §2 — Eloquent ORM + escaping Blade `{{ }}` |
| NFR-AVL-001 | Queue retry 3 attempts | `nfr.md` §3 — `FulfillTransactionJob::tries()=3`, backoff 30/120/300 |
| NFR-AVL-002 | Payment idempotency | `nfr.md` §3 — `ref_id` UNIQUE + `lockForUpdate` |
| NFR-AVL-003 | Wallet atomicity | `nfr.md` §3 — `DB::transaction` + `lockForUpdate` |
| NFR-AVL-004 | Scheduled tasks 1 mnt / 5 mnt | `nfr.md` §3 — `routes/console.php` everyMinute / everyFiveMinutes |
| NFR-SCP-001 | Single VPS (2 vCPU, 2GB RAM) | `nfr.md` §4 — `docs/deployment.md`; system-wide |
| NFR-SCP-002 | Redis: cache + queue + session | `nfr.md` §4 — `config/cache.php`, `config/queue.php`, `config/session.php` |
| NFR-SCP-003 | PHP-FPM 25 workers | `nfr.md` §4 — `pm.max_children = 25`; system-wide |
| NFR-SCP-004 | Nginx static cache + gzip | `nfr.md` §4 — `deploy/nginx-site.conf` (gzip, expires 30d) |

Angka target verbatim per kategori (§3.1 Performance, §3.2 Security, §3.3 Availability, §3.4 Scalability) tersimpan di Appendix `docs/analyst/nfr.md`. Sub-seksi §3.1–§3.4 tidak lagi dipecah di SRS — kategori kini diwakili prefiks ID NFR (`PERF`/`SEC`/`AVL`/`SCP`). Pemetaan NFR → baris FR ada di `requirements-matrix.md` § NFR Traceability.

---

## 4. Data Requirements

26 database tables, 18 Eloquent models, 10 enums.
Full details: `data-dictionary.md`
ERD: `erd.md`

---

## 5. Integration Points

| System | Protocol | Purpose | Direction |
|--------|----------|---------|-----------|
| Digiflazz v2 | HTTP POST, JSON | Distributor (top-up, check-username, status) | Bidirectional |
| Tripay | HTTP POST webhook | Payment gateway | Inbound |
| Duitku | HTTP POST webhook | Payment gateway (backup) | Inbound |
| Reseller | HTTP POST | Callback URL | Outbound |
| SMTP | SMTP | Email delivery | Outbound |

---

## 6. Constraints

Detail, impact, dan validasi per file repo: **`docs/analyst/assumptions-constraints.md`** (§ Constraints) — SRS hanya menyimpan daftar id di bawah.

- **CON-001 … CON-005** — hosting VPS berbayar, domain + SSL, Redis wajib, root access deploy, fulfillment Minecraft manual

---

## 7. Assumptions

Detail, impact jika salah, dan reality-check: **`docs/analyst/assumptions-constraints.md`** (§ Assumptions).

- **ASM-001 … ASM-004** — ketersediaan Digiflazz, webhook gateway < detik, single-server, SMTP — deviasi tercatat pada ASM-001, ASM-002, ASM-004

---

## 8. Open Questions

Detail, impact, dan status verifikasi: **`docs/analyst/assumptions-constraints.md`** (§ Open Questions).

- **OQ-001 … OQ-003** — backup distributor, monitoring/alerting, backup database
