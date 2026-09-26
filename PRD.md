# Rancangan Teknis — Fofa Shop (Website Top Up Game)

Catatan ini berisi rancangan awal arsitektur untuk proyek Fofa Shop, direkam sebagai bagian dari dokumentasi analisis sistem.

---

## 1. Konteks Proyek

- Fofa Shop adalah website top-up game (voucher/diamond/UC dll).
- Model bisnis: reseller H2H — beli produk digital dari distributor via API, jual ke end user dengan markup.

---

## 2. Keputusan Tech Stack (Final)

| Komponen | Pilihan | Alasan |
|---|---|---|
| Backend & frontend | **Laravel** (monolith, Blade + Livewire bila perlu interaktif) | Ekosistem top-up Indonesia mayoritas pakai Laravel, built-in queue/scheduler/auth, cocok untuk tim kecil ship cepat |
| CSS framework | **Tailwind CSS v4** (`@tailwindcss/vite` plugin, CSS-first config) | CSS cascade layers untuk dark mode, zero-config Vite integration, tree-shaking lebih agresif dari v3 |
| Database | MySQL/PostgreSQL | Standar untuk data transaksi, produk, user |
| Auth admin & storefront | **Session-based** (Laravel Breeze/Fortify) | Bukan API stateless, jadi session lebih simpel & aman dibanding JWT |
| Auth API reseller | **Laravel Sanctum** (token/API key), bukan JWT | Lebih ringan, native Laravel, cukup untuk kebutuhan reseller H2H |
| Distributor produk (H2H) | Digiflazz (utama), TokoVoucher/VIPayment/Apigames (alternatif/backup) | Marketplace produk digital terpopuler, dokumentasi API jelas |
| Payment gateway | Tripay / Duitku / Tokopay / Pakasir | Cocok skala kecil-menengah, setup cepat, biaya lebih murah dibanding Xendit/Midtrans |
| Cache | Redis | Cache daftar produk & harga biar tidak bolak-balik hit API distributor |
| Queue | Laravel Queue (driver Redis) | Proses async hit API distributor & kirim notifikasi |
| Scheduler | Laravel Scheduler (built-in) | Cron cek transaksi pending & sync harga distributor |
| Mail | Laravel Mail (built-in) + provider SMTP/API (Mailgun/Resend) | Notifikasi invoice & status transaksi |
| Event/listener | Laravel Event & Listener (built-in) | Broadcast event "transaksi lunas" ke beberapa handler (proses topup, kirim email, update dashboard) |
| WebSocket | **Laravel Reverb** | Live chat/CS widget — real-time messaging tanpa dependency third-party |
| Export | **Maatwebsite/Excel** + **DomPDF** | Export CSV/Excel untuk import ke software accounting, PDF untuk print/presentasi |
| Hosting | VPS berbayar (bukan hosting gratisan) | Wajib untuk production — lihat bagian 7 |

---

## 3. User Roles

| Role | Keterangan | Akses |
|------|-----------|-------|
| **customer** | User biasa, beli produk via storefront | Browse produk, checkout, cek invoice, riwayat transaksi, request refund |
| **reseller** | H2H API, beli produk via API dengan harga khusus | Semua akses customer + wallet deposit, API access, white-label store, click analytics |
| **admin** | Mengelola sistem | CRUD produk/kategori, kelola transaksi (retry/refund), approve reseller, dashboard monitoring, broadcast |
| **owner** | Pemilik sistem | Semua akses admin + hapus produk kapan saja, manage voucher/promo, kelola role admin, export pajak, fraud alerts |

---

## 4. Alur Transaksi (Happy Path)

1. **User checkout** — pilih produk & isi ID game
2. **Buat invoice** — status `pending`, simpan `ref_id` (harus unique untuk idempotency)
3. **User bayar** — via QRIS/e-wallet/VA lewat payment gateway
4. **Callback gateway** — verifikasi signature, update status pembayaran
5. **Cek status lunas** — jika lunas lanjut fulfillment; jika expired, transaksi berhenti di status `expired`
6. **Hit API distributor** — kirim topup ke provider (Digiflazz dll)
7. **Update status final** — sukses/gagal + kirim notifikasi ke user
   - Jika distributor gagal/pending → refund saldo (jika prabayar) atau retry otomatis via cron sebelum di-mark `failed`

Status enum: `pending → paid → processing → waiting_fulfillment → success/failed/expired`

---

## 5. Skema Database

### Tabel Existing

**users**
- id (PK, uuid), name, email, role (enum: customer/reseller/admin/owner), email_notification_enabled (bool, default true), created_at

**categories**
- id (PK), name, slug (unique), icon_url, display_order, is_active

**products**
- id (PK, uuid), category_id (FK), subcategory_id (FK, nullable), name, description, image_path (nullable, upload foto produk game lain — JPG/PNG/WebP max 2MB, storage public/products), price, cost_price, is_active, deleted_at (nullable, soft delete)

**product_vendor_mappings**
- id (PK), product_id (FK), vendor, vendor_product_code, vendor_price

**reseller_profiles**
- id (PK), user_id (FK, unique), api_key (unique), api_secret, markup_percentage, callback_url, status (enum: active/inactive/suspended)

**wallets**
- id (PK), user_id (FK, unique), balance

**wallet_transactions**
- id (PK), wallet_id (FK), type (enum: deposit/deduction/refund), amount, reference_id (FK, nullable), notes

**transactions**
- id (PK, uuid), ref_id (unique), user_id (FK, nullable), guest_email (wajib untuk guest, email notifikasi status transaksi), product_id (FK), target_game_id (nullable untuk manual), target_server_id (nullable), minecraft_edition (nullable, enum java/bedrock/manual), delivered_minecraft_username/password (nullable, akun dikirim via email), delivery_note (nullable), amount, status, expired_at, created_at

**payments**
- id (PK, uuid), transaction_id (FK), method, gateway_ref, status, signature_verified, paid_at

**distributor_logs**
- id (PK, uuid), transaction_id (FK), provider, attempt_number, request_payload (json), response_payload (json), http_status_code, duration_ms, status, error_message

### Tabel Baru

**vouchers**
- id (PK), code (unique), type (enum: percentage/fixed), value (decimal), min_order_amount (decimal, nullable), max_uses (int, nullable), used_count (int, default 0), starts_at (timestamp, nullable), expires_at (timestamp, nullable), is_active (bool, default true), created_by (FK users)

**voucher_usages**
- id (PK), voucher_id (FK), user_id (FK), transaction_id (FK), discount_amount (decimal), created_at

**click_logs**
- id (PK), referrer_api_key (string), product_id (FK, nullable), category_id (FK, nullable), ip_address (string), user_agent (text), converted (bool, default false), transaction_id (FK, nullable), created_at

**reseller_stores**
- id (PK), user_id (FK, unique), slug (unique), store_name, logo_url (nullable), theme_color (default '#6366f1'), is_active (bool, default true), created_at

**audit_logs**
- id (PK), user_id (FK), action (string), auditable_type, auditable_id, old_values (json, nullable), new_values (json, nullable), ip_address, user_agent, created_at

**announcements**
- id (PK), title, content (text), type (enum: info/maintenance/warning), target_audience (enum: all/customer/reseller/admin), is_active (bool, default true), published_at (timestamp, nullable), created_by (FK users), created_at

**notifications**
- id (PK), user_id (FK), title, message (text), read_at (timestamp, nullable), created_at

**chat_rooms**
- id (PK), user_id (FK), admin_id (FK, nullable), status (enum: open/waiting/closed), created_at, closed_at (timestamp, nullable)

**chat_messages**
- id (PK), room_id (FK), sender_id (FK users), message (text), created_at

**fraud_alerts**
- id (PK), user_id (FK, nullable), transaction_id (FK, nullable), rule_type (string), severity (enum: low/medium/high), details (json), resolved_at (timestamp, nullable), resolved_by (FK users, nullable), created_at

**fraud_rules**
- id (PK), rule_type (unique), label, description, threshold (int), window_minutes (int), is_active (bool, default true), config (json, nullable), created_at

**subcategories**
- id (PK), category_id (FK), name, slug, image_path (nullable, foto subkategori Minecraft — upload JPG/PNG/WebP max 2MB, storage public/subcategories), display_order, is_active, created_at

**admin_payment_methods**
- id (PK), method_code (unique), display_name, icon (nullable), is_active (bool, default true), sort_order (int, default 0), created_at

**users (alter)**
- email_notification_enabled (bool, default true) — preferensi email pengumuman, toggle di Profile

**notifications** (sudah ada, dipakai untuk broadcast pengumuman)
- id (PK), user_id (FK), title, message (text), read_at (timestamp, nullable), created_at — dibuat per user saat announcement publish

**product_deletion_logs**
- id (PK), product_id (FK), product_snapshot (json), deleted_by (FK users), reason (text, nullable), deleted_at

**reseller_monthly_reports**
- id (PK), user_id (FK), month (int), year (int), total_transactions (int), total_revenue (decimal), total_commission (decimal), generated_at, created_at

### Relasi

- users 1—N transactions
- users 1—1 reseller_profiles
- users 1—1 wallets
- users 1—N audit_logs
- users 1—N notifications
- products 1—N transactions
- products 1—N product_vendor_mappings
- products 1—N click_logs
- transactions 1—N payments
- transactions 1—N distributor_logs
- transactions 1—1 voucher_usages (nullable)
- wallets 1—N wallet_transactions
- vouchers 1—N voucher_usages
- chat_rooms 1—N chat_messages

---

## 6. API Endpoint List

### Storefront (public)

- `GET /` — list kategori game
- `GET /products/{category}` — list produk per game
- `GET /products/{id}` — detail produk
- `POST /checkout` — buat invoice baru
- `GET /invoice/{ref_id}` — cek status transaksi
- `POST /invoice/{ref_id}/refund-request` — submit refund
- `GET /search` — search produk (full-text)
- `GET /cek-nickname` — cek nickname/ID game via distributor API (dengan fallback)
- `GET /history` — riwayat transaksi user (login required)
- `GET /toko/{slug}` — halaman toko pribadi reseller

### Live Chat (session-based)

- `GET /chat` — buka/lanjutkan chat room
- `POST /chat/messages` — kirim pesan (Livewire + Reverb real-time)
- `POST /chat/close` — tutup chat room

### Payment webhook (CSRF exempt)

- `POST /webhook/payment-callback` — terima notifikasi dari payment gateway (wajib verifikasi signature)

### Auth (session-based)

- `GET/POST /login`, `POST /logout`, `GET/POST /register`

### Customer panel (session + auth)

- `GET /dashboard` — dashboard user
- `GET /history` — riwayat transaksi
- `GET /history/{ref_id}` — detail transaksi + re-check status

### Reseller/H2H API (Sanctum token)

- `GET /api/v1/products` — list produk dengan harga reseller
- `POST /api/v1/topup` — buat transaksi
- `GET /api/v1/topup/{ref_id}/status` — cek status
- `GET /api/v1/balance` — cek saldo wallet
- `GET /api/v1/analytics/clicks` — statistik klik & konversi
- `GET /api/v1/store` — info toko pribadi
- `PUT /api/v1/store` — update toko pribadi

### Admin panel (session + middleware role:admin)

- `GET /admin/dashboard` — monitoring dashboard (status count-based, **tanpa data finansial untuk admin**)
- `GET/POST /admin/products`, `PUT/DELETE /admin/products/{id}` — CRUD produk (termasuk price & cost_price)
- `POST /admin/products/sync` — sync harga distributor
- `GET /admin/categories`, `GET/POST /admin/categories`, `PUT/DELETE /admin/categories/{id}` — CRUD kategori
- `GET /admin/transactions` — lihat transaksi (grafik + tabel, **tanpa kolom amount untuk admin**)
- `GET /admin/transactions/{id}` — detail transaksi (termasuk log distributor, **tanpa amount untuk admin**)
- `POST /admin/transactions/{id}/retry` — retry manual
- `POST /admin/transactions/{id}/refund-complete` — selesaikan refund
- `POST /admin/transactions/{id}/mark-manual-success` — Minecraft manual fulfillment
- `POST /admin/transactions/export-pdf` — export PDF (**owner-only**, admin → 403)
- `GET /admin/refunds` — daftar refund request (**tanpa amount untuk admin**)
- `GET /admin/users` — daftar user
- `GET /admin/wallets` — daftar wallet reseller (**tanpa saldo untuk admin**)
- `POST /admin/wallets/{user}/deposit` — deposit saldo
- `GET /admin/vouchers` — daftar voucher (**tanpa value untuk admin**)
- `POST /admin/vouchers` — buat voucher
- `GET /admin/payment-methods` — daftar metode bayar
- `PUT /admin/payment-methods/{id}/toggle` — aktif/nonaktifkan metode bayar
- `GET /admin/announcements` — daftar pengumuman
- `POST /admin/announcements` — buat pengumuman
- `GET /admin/audit-logs` — log aktivitas admin

### Owner panel (session + middleware role:owner)

- `GET /owner/vouchers` — daftar voucher
- `POST /owner/vouchers` — buat voucher
- `PUT /owner/vouchers/{id}` — edit voucher
- `DELETE /owner/vouchers/{id}` — hapus voucher
- `GET /owner/reports/finance` — laporan keuangan (omset/cost/margin)
- `GET /owner/reports/products` — analitik produk terlaris
- `GET /owner/reports/vendors` — performa vendor
- `GET /owner/reports/tax` — data pajak
- `POST /owner/reports/export` — export CSV/Excel/PDF
- `GET /owner/admins` — kelola role admin lain
- `PUT /owner/admins/{user}/role` — ubah role admin
- `DELETE /owner/products/{id}` — hapus produk (kapan saja)
- `GET /owner/settings` — pengaturan global bisnis
- `PUT /owner/settings` — update pengaturan

---

## 7. Fitur MVP (Lengkap)

### 7.1 Customer Features

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Katalog kategori game | Homepage tampilkan list kategori dengan icon |
| 2 | Search produk | Full-text search di nama produk |
| 3 | Cek nickname/ID game | Auto-check via distributor API (Digiflazz check username), fallback validasi format jika game tidak didukung |
| 4 | Riwayat transaksi | Halaman riwayat transaksi user + re-check status via API |
| 5 | Multi metode pembayaran | QRIS, e-wallet, VA (BCA/BNI/BRI/Mandiri), Alfamart/Indomaret |
| 6 | Notifikasi email | Email untuk status: paid, success, failed, expired, refund completed |
| 7 | Voucher/promo diskon | Input kode voucher di checkout, diskon percentage atau fixed amount |

### 7.2 Reseller Features

> **Status: BAGIAN DROPPED in spec v1.3 (scope-cut komunitas) — SEBAGIAN.** Alasan: reseller channel dormant untuk launch skala komunitas. Baris yang di-drop: **4 Dokumentasi API** (reseller API docs — dormant), **5 Laporan transaksi & komisi** (reseller monthly reports — dormant), **7 Toko pribadi /toko/{slug}** (reseller store — no white-label store). Keputusan: scope-cut v1.3 (lihat `docs/analyst/spec.md` §Out of Scope). Baris tersebut TIDAK jadi requirement downstream — jangan di-trace ke SRS/use case; baris lain tetap berlaku.

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Dashboard saldo wallet | Tampilan saldo + grafik transaksi (Livewire + Chart.js) |
| 2 | Deposit saldo | Manual transfer + approval admin |
| 3 | Harga khusus reseller | cost_price + markup_percentage (dari reseller_profiles) |
| 4 | Dokumentasi API | Halaman docs endpoint + request/response examples |
| 5 | Laporan transaksi & komisi | Laporan bulanan: total transaksi, revenue, komisi |
| 6 | Webhook callback | POST ke callback_url saat status transaksi berubah |
| 7 | Toko pribadi /toko/{slug} | Halaman toko custom: store_name, logo, theme_color |

### 7.3 Reseller Analytics

> **Status: DROPPED in spec v1.3 (scope-cut komunitas).** Alasan: click tracking & analytics tidak kritis untuk launch skala komunitas (reseller channel dormant). Keputusan: scope-cut v1.3 (lihat `docs/analyst/spec.md` §Out of Scope). Fitur ini TIDAK jadi requirement downstream — jangan di-trace ke SRS/use case.

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Click tracking | UTM parameter `?ref={api_key}` — log setiap klik |
| 2 | Conversion rate | klik vs pembelian — hitung efektivitas promosi |
| 3 | Grafik analytics | Line chart klik per hari, bar chart konversi per produk |

### 7.4 Admin Features

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | CRUD produk/kategori/harga | Hapus dibatasi: produk bertransaksi = locked, ≤24h = owner-only, >24h = soft-delete |
| 2 | Kelola transaksi | Retry manual, refund manual, mark manual success (Minecraft) |
| 3 | Kelola reseller | Approve, atur markup |
| 4 | Dashboard monitoring | Status transaksi (count-based), alert distributor down. **Admin TIDAK bisa lihat omset/amount** (financial data access control) |
| 5 | Grafik transaksi | Line chart per hari/minggu/bulanan (count-based, tanpa data finansial) |
| 6 | Broadcast pengumuman | In-app notification + email blast (email respect `email_notification_enabled`, transaction proof tetap WAJIB, target hanya customer/reseller, admin/owner tidak dapat announcement) |
| 7 | Audit log | Log semua aksi admin/owner (old/new values) |
| 8 | Aktif/nonaktifkan metode bayar | Toggle metode pembayaran aktif |

### 7.5 Live Chat / CS Widget

> **Status: DROPPED in spec v1.3 (scope-cut komunitas).** Alasan: live chat (Reverb) tidak diperlukan untuk launch skala komunitas — `laravel/reverb` dihapus total. Keputusan: scope-cut v1.3 (lihat `docs/analyst/spec.md` §Out of Scope). Fitur ini TIDAK jadi requirement downstream — jangan di-trace ke SRS/use case.

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Chat room | User buka chat → status `waiting` → admin assign → `open` → selesai → `closed` |
| 2 | Real-time messaging | Laravel Reverb WebSocket |
| 3 | Admin panel | Daftar chat rooms, assign admin, lihat history |
| 4 | Chat widget | Floating button di storefront |

### 7.6 Owner Features

> **Status: BAGIAN DROPPED in spec v1.3 (scope-cut komunitas) — SEBAGIAN.** Alasan: baris **10 Export data pajak** (CSV/Excel di-drop, `maatwebsite/excel` dihapus — PDF tetap ada dari halaman transaksi; PPN 11% di-drop) dan baris **11 Pengaturan global** (global settings UI `/owner/settings` di-drop → pakai `.env`). Keputusan: scope-cut v1.3 (lihat `docs/analyst/spec.md` §Out of Scope). Baris tersebut TIDAK jadi requirement downstream — jangan di-trace ke SRS/use case; baris lain tetap berlaku.

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Semua akses admin | Owner punya semua hak admin **termasuk akses data finansial** |
| 2 | Hapus produk kapan saja | Tidak ada batasan 24 jam untuk owner |
| 3 | Manage voucher/promo | CRUD voucher (percentage/fixed) |
| 4 | Laporan keuangan | Omset, cost, margin per periode |
| 5 | Grafik transaksi + omset | Line chart + total omset di dashboard |
| 6 | Export PDF | Laporan transaksi PDF dari halaman transaksi (mengikuti filter periode aktif) |
| 7 | Analitik produk terlaris | Ranking produk berdasarkan volume transaksi |
| 8 | Performa vendor | Statistik success rate per vendor distributor |
| 9 | Kelola role admin | Assign/revoke role admin lain |
| 10 | Export data pajak | CSV/Excel + PDF — data transaksi untuk akuntansi |
| 11 | Pengaturan global | Konfigurasi bisnis (markup default, metode bayar, dsb) |

### 7.7 Fraud Detection

> **Status: DROPPED in spec v1.3 (scope-cut komunitas).** Alasan: fraud detection (rule-based) di-drop — manual monitoring cukup untuk launch. Keputusan: scope-cut v1.3 (lihat `docs/analyst/spec.md` §Out of Scope). Fitur ini TIDAK jadi requirement downstream — jangan di-trace ke SRS/use case.

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Rate limit fraud | Banyak transaksi dari IP/user yang sama dalam waktu singkat |
| 2 | Anomali harga | Reseller beli tapi tidak markup (harga jual = cost) |
| 3 | Transaksi jam 2-4am | Transaksi di luar jam normal |
| 4 | Target ID berubah-ubah | User yang beli untuk banyak ID game berbeda |
| 5 | Alert notification | Notifikasi ke admin saat fraud terdeteksi |
| 6 | Resolve alert | Admin/owner bisa resolve fraud alert |

### 7.8 Reports & Export

> **Status: BAGIAN DROPPED in spec v1.3 (scope-cut komunitas) — SEBAGIAN, tidak seluruh subsection.** Alasan: standalone finance reports dashboard (`/owner/reports/*`) di-drop → PDF export pindah ke halaman transaksi admin (dipertahankan, see spec v1.3.1); CSV/Excel export (`maatwebsite/excel`) dan kalkulasi pajak (PPN 11%) di-drop. Keputusan: scope-cut v1.3 (lihat `docs/analyst/spec.md` §Out of Scope). Baris **1, 2, 3, 5** di bawah TIDAK jadi requirement downstream — jangan di-trace ke SRS/use case. Baris **4 (Export PDF)** TETAP berlaku (dari halaman transaksi, ikut filter periode aktif).

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Laporan keuangan owner | Omset, cost, margin per hari/bulan/tahun |
| 2 | Laporan reseller | Komisi bulanan per reseller |
| 3 | Export CSV/Excel | Import ke software accounting |
| 4 | Export PDF | Print/presentasi — dari halaman transaksi, mengikuti filter periode aktif |
| 5 | Pajak (PPN 11%) | Hitung otomatis PPN untuk export pajak |

### 7.9 Financial Data Access Control (v1.3.1)

| Aspek | Admin | Owner |
|-------|-------|-------|
| Grafik transaksi (count) | ✅ | ✅ |
| Nama produk, game ID, status | ✅ | ✅ |
| Log distributor | ✅ | ✅ |
| **Nominal per transaksi** | ❌ | ✅ |
| **Total omset** | ❌ | ✅ |
| **Export PDF** | ❌ | ✅ |

Implementasi: query-level column exclusion via `scopeForAdmin()` — kolom `amount` tidak di-select dari database untuk admin. Bukan cuma UI hide.

### 7.10 Transaction Chart (v1.3.1)

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Line chart | Jumlah transaksi per hari (30 hari terakhir) |
| 2 | Filter harian | Tampilkan transaksi per hari |
| 3 | Filter mingguan | Tampilkan transaksi per minggu |
| 4 | Filter bulanan | Tampilkan transaksi per bulan |
| 5 | Sync dengan tabel | Filter periode mempengaruhi chart DAN tabel transaksi |
| 6 | Sync dengan PDF | Export PDF mengikuti filter periode yang sedang aktif |

### 7.11 Notifikasi & Preferensi Email

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Guest email wajib | Checkout guest **wajib** isi email (`transactions.guest_email`). Email digunakan untuk notifikasi status transaksi (paid/success/failed/expired). Tidak ada opt-in — email selalu disimpan untuk guest. |
| 2 | User notification preference | Field `users.email_notification_enabled` (bool, default true). Toggle di halaman Profile (`livewire:profile.notification-preferences`). Jika dimatikan, tidak dapat email pengumuman, tapi tetap dapat email bukti transaksi (paid/success/failed/expired/reconciliation = WAJIB). |
| 3 | Aturan email | Transaksi = WAJIB kirim (jika ada alamat email). Pengumuman = respect `email_notification_enabled`. Guest tanpa email = tidak ada email (tapi guest wajib isi email, jadi selalu ada). |
| 4 | Broadcast target | `all` = customer+reseller, `customer` = hanya customer, `reseller` = hanya reseller. Admin/owner TIDAK menerima announcement (hanya notifikasi penting). |

### 7.12 UI/UX Polish — Page Title & Favicon (v1.5)

| # | Fitur | Keterangan |
|---|-------|-----------|
| 1 | Dynamic page title | `<title>` tag berubah per halaman. Format: `[Page Name] - Fofa Shop`. Auth pages: `Masuk - Fofa Shop`, `Daftar - Fofa Shop`. Dashboard: `Dashboard — [Nama User] - Fofa Shop`. Storefront: `Fofa Shop`. |
| 2 | Favicon | `public/logo.png` (Fofa Shop red circle FS monogram) + generated `favicon.ico`, `favicon-32x32.png`, `favicon-16x16.png`. `<link rel="icon">` di semua layout files. |
| 3 | APP_NAME | `.env` `APP_NAME=Fofa Shop` — menggantikan default `Laravel`. |
| 4 | Title pattern | Layout files pakai `$title ?? config('app.name', 'Fofa Shop')` sebagai fallback. Livewire Volt pages pakai `View::share('title', ...)` di component PHP. Regular Blade components pakai `<x-slot name="title">`. |

---

## 8. Hosting & Deployment

**Wajib pakai hosting berbayar, tidak bisa mengandalkan yang gratisan.** Alasannya:

- Website ini menangani **transaksi uang beneran** (payment gateway callback, API distributor) — butuh **uptime stabil** dan **IP tetap** (beberapa payment gateway/distributor mewajibkan whitelist IP server, hosting gratisan biasanya IP dinamis/shared yang tidak bisa di-whitelist)
- Hosting gratisan umumnya punya **limit resource ketat** (CPU/RAM kecil, sleep/idle setelah tidak diakses) — kalau website "tidur" pas ada callback masuk dari payment gateway, transaksi bisa gagal diproses
- Butuh akses penuh untuk setup **queue worker** (proses background Laravel Queue) dan **cron job** (scheduler) yang jalan terus-menerus — ini biasanya tidak didukung di hosting gratisan/shared hosting murah
- Domain custom & SSL wajib (bukan subdomain gratisan) untuk kredibilitas dan supaya payment gateway/distributor mau approve integrasi produksi
- **Laravel Reverb** butuh akses WebSocket (port 443/80) — tidak didukung di shared hosting

**Rekomendasi opsi VPS berbayar:**
- **DigitalOcean / Vultr / Linode** — VPS internasional, mulai ~$5-6/bulan, banyak tutorial deploy Laravel
- **Biznet Gio / IDCloudHost / Niagahoster Cloud VPS** — provider lokal, latency lebih rendah untuk user Indonesia, harga mulai ~Rp50-100rb/bulan
- Minimal spek buat mulai: 2 vCPU, 2GB RAM — cukup untuk MVP + Reverb, bisa upgrade seiring traffic naik

---

## 9. Dependencies Baru

| Package | Purpose | Phase |
|---------|---------|-------|
| `laravel/reverb` | WebSocket server untuk live chat | 7 |
| `maatwebsite/excel` | Export CSV/Excel | 9 |
| `barryvdh/laravel-dompdf` | Export PDF | 9 |

---

## 10. Implementation Phases

### Phase 1: Foundation
Owner role + Audit log + Product deletion rules

### Phase 2: Voucher System
Voucher CRUD + checkout integration

### Phase 3: Reseller Features
Click tracking + White-label store + Grafik analytics

### Phase 4: Search + Nickname Check
Full-text search + Game ID verification

### Phase 5: Customer Features
Riwayat transaksi + re-check status

### Phase 6: Admin Enhancements
Payment method toggle + Broadcast pengumuman

### Phase 7: Live Chat
Laravel Reverb WebSocket

### Phase 8: Fraud Detection
Rule engine + alert system

### Phase 9: Reports & Export
Laporan keuangan + CSV/Excel/PDF export

### Phase 10: API Documentation
Halaman dokumentasi API reseller

Status: Reseller API docs di-scope-cut (lihat `docs/analyst/spec.md` v1.3); kontrak API terdokumentasi di `docs/analyst/api-contract.md`.
