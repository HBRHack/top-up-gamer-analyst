# Non-Functional Requirements (NFR) — Fofa Shop

Dokumen ini menstandarkan seluruh Non-Functional Requirement proyek Fofa Shop menjadi ber-ID
sehingga bisa di-trace dari `docs/analyst/requirements-matrix.md` dan diuji per skenario.

- **Sumber utama:** `docs/analyst/SRS.md` §3 *Non-Functional Requirements* — kategori §3.1
  Performance, §3.2 Security, §3.3 Availability, §3.4 Scalability (nama sub-seksi versi lama:
  setelah §3 dijadikan pointer ke dokumen ini, SRS §3 tidak lagi memecah §3.1–§3.4 — kategori
  kini diwakili prefiks ID NFR). Semua angka target di bawah ini dibawa **verbatim** dari `SRS.md`
  §3 versi sebelum §3 dijadikan pointer ke dokumen ini
  (lihat [Appendix](#appendix--kutipan-verbatim-srsmd-section-3)).
- **Sumber pendukung:** kode dan dokumen implementasi di repo ini (path disebut eksplisit per NFR).
- **Konvensi ID:** `NFR-<KATEGORI>-<NNN>` dengan KATEGORI = `PERF` (Performance), `SEC` (Security),
  `AVL` (Availability), `SCP` (Scalability).
- **Format skenario:** setiap NFR memiliki minimal satu tabel skenario QA 6 bagian:
  **Sumber Stimulus · Stimulus · Environment · Artifact · Respons · Ukuran Respons**.
  Setiap baris wajib terukur; target yang tidak punya ukuran yang jujur ditulis sebagai gap di
  [`Known Clarity Gaps`](#known-clarity-gaps) — tidak diisi pernyataan kabur.
- **Nilai implementasi** (angka dari config/kode) ditandai sebagai implementasi dan tidak
  menggantikan target SRS.

---

## Index NFR

| ID           | Kategori     | Inti Requirement                                   | Sumber SRS | FR Anchor (matrix)     |
|--------------|--------------|----------------------------------------------------|------------|------------------------|
| NFR-PERF-001 | Performance  | Response time < 200ms (cached)                     | SRS §3.1   | FR-003                 |
| NFR-PERF-002 | Performance  | Throughput ~60 req/s (2 vCPU, 2GB RAM)             | SRS §3.1   | — (system-wide)        |
| NFR-PERF-003 | Performance  | Cache hit rate > 80% untuk product listings        | SRS §3.1   | FR-001, FR-002, FR-004 |
| NFR-SEC-001  | Security     | Keamanan jalur checkout (FR-005)                   | SRS §3.2   | FR-005                 |
| NFR-SEC-002  | Security     | Autentikasi & otorisasi API reseller               | SRS §3.2   | FR-010                 |
| NFR-SEC-003  | Security     | Pencegahan SQL injection & XSS                     | SRS §3.2   | FR-002, FR-005         |
| NFR-AVL-001  | Availability | Queue retry: 3 attempts per job                    | SRS §3.3   | FR-010, FR-016         |
| NFR-AVL-002  | Availability | Payment idempotency: ref_id unique + lockForUpdate | SRS §3.3   | FR-013                 |
| NFR-AVL-003  | Availability | Wallet atomicity: DB::transaction + lockForUpdate  | SRS §3.3   | FR-010, FR-018         |
| NFR-AVL-004  | Availability | Scheduled tasks: expire (1 menit) + sync (5 menit) | SRS §3.3   | FR-011, FR-016         |
| NFR-SCP-001  | Scalability  | Single VPS deployment (2 vCPU, 2GB RAM)            | SRS §3.4   | — (system-wide)        |
| NFR-SCP-002  | Scalability  | Redis untuk cache + queue + session                | SRS §3.4   | FR-010, FR-013         |
| NFR-SCP-003  | Scalability  | PHP-FPM 25 workers                                 | SRS §3.4   | — (system-wide)        |
| NFR-SCP-004  | Scalability  | Nginx static cache + gzip                          | SRS §3.4   | FR-001, FR-002         |

---

## 1. Performance (SRS §3.1)

### NFR-PERF-001 — Latency respons cached: cek game ID & pencarian storefront

- **Sumber:** SRS §3 (kategori §3.1) — *Response time < 200ms (cached)*
- **Anchored FR:** FR-003 *Cek Nickname/ID Game* (matrix § Full Traceability Matrix → FR-003);
  jalur pembanding FR-002 *Search Produk*.
- **Requirement (verbatim):** Response time < 200ms (cached).
- **Artifact implementasi:** route `GET /cek-nickname` → `GameIdCheckController`;
  `GameIdVerifierChain::check()` (`Cache::remember($key, 300, …)`); route `GET /search` →
  `SearchController@index` (tanpa cache aplikasi — lihat GAP-03).

| Sumber Stimulus             | Stimulus                                       | Environment                          | Artifact                                          | Respons                        | Ukuran Respons                              |
|-----------------------------|------------------------------------------------|--------------------------------------|---------------------------------------------------|--------------------------------|---------------------------------------------|
| Browser (GET /cek-nickname) | Request identik ke-2 dalam TTL cache 300 detik | Staging; Redis aktif; VPS 2 vCPU/2GB | GameIdVerifierChain::check (Cache::remember 300s) | HTTP 200 JSON {valid, message} | TTFB < 200 ms pada request ke-2 (cache hit) |
| Browser (GET /search?q=…)   | Kueri sama diulang tanpa perubahan data        | Staging; Nginx + PHP-FPM aktif       | SearchController@index (tanpa cache aplikasi)     | HTTP 200 halaman HTML          | TTFB < 200 ms; syarat cached lihat GAP-03   |

### NFR-PERF-002 — Throughput ~60 req/s

- **Sumber:** SRS §3 (kategori §3.1) — *Throughput ~60 req/s (2 vCPU, 2GB RAM)*
- **Anchored FR:** system-wide (mendukung semua jalur publik FR-001…FR-006); di matrix § NFR
  Traceability berstatus *System-wide* (N/A pada Coverage Summary) — sengaja tidak dipetakan ke
  satu FR.
- **Requirement (verbatim):** Throughput ~60 req/s (2 vCPU, 2GB RAM).
- **Artifact implementasi:** `deploy/OPTIMIZATION.md` (Target: 60 req/s sustained),
  `deploy/nginx-site.conf`, `docs/deployment.md`.

| Sumber Stimulus        | Stimulus                                                                | Environment                                         | Artifact                                | Respons                          | Ukuran Respons                                   |
|------------------------|-------------------------------------------------------------------------|-----------------------------------------------------|-----------------------------------------|----------------------------------|--------------------------------------------------|
| Load generator (k6/ab) | Campuran GET /, /products/{id}, /search, POST /checkout selama 60 detik | VPS 2 vCPU + 2 GB RAM; PHP-FPM 25 workers; Redis on | deploy/OPTIMIZATION.md; log akses Nginx | HTTP 2xx/3xx untuk request valid | >= 60 req/s sustained 60 detik; error rate <= 1% |

### NFR-PERF-003 — Cache hit rate > 80% (product listings)

- **Sumber:** SRS §3 (kategori §3.1) — *Cache hit rate > 80% untuk product listings*
- **Anchored FR:** FR-001/FR-002/FR-004 — direferensikan di matrix (kolom Non-Functional,
  § NFR Traceability).
- **Requirement (verbatim):** Cache hit rate > 80% untuk product listings.
- **Artifact implementasi:** `config/cache.php` (`CACHE_STORE=redis`), Redis DB 1
  (`docs/redis-setup.md`); instrumen metrik cache belum ada (GAP-05).

| Sumber Stimulus    | Stimulus                                                       | Environment                                      | Artifact                                  | Respons                  | Ukuran Respons                                    |
|--------------------|----------------------------------------------------------------|--------------------------------------------------|-------------------------------------------|--------------------------|---------------------------------------------------|
| Redis (INFO stats) | Baca keyspace_hits dan keyspace_misses di akhir window traffic | CACHE_STORE=redis (config/cache.php); Redis DB 1 | docs/redis-setup.md; cache store aplikasi | Metrik hit rate tercatat | hits/(hits+misses) > 80% selama window pengukuran |

---

## 2. Security (SRS §3.2)

### NFR-SEC-001 — Keamanan jalur checkout (FR-005)

- **Sumber:** SRS §3.2 — CSRF protection (Laravel tokens); Rate limiting (CheckoutRateLimit:
  IP/user/product+target); Payment signature: HMAC-SHA256 + timing-safe comparison; Password
  hashing: bcrypt (Hash::make); Minecraft password: encrypted cast; Role-based access
  (RoleMiddleware + scopeForAdmin()).
- **Anchored FR:** FR-005 *Checkout + validasi* (matrix § Full Traceability Matrix → FR-005).
- **Requirement:** Seluruh permintaan pada jalur checkout dan pembayaran harus melewati lapisan
  keamanan di atas; tidak ada satu pun jalur yang boleh melewati token CSRF, rate limit,
  verifikasi signature, hash bcrypt, enkripsi kredensial Minecraft, maupun pemeriksaan role.
- **Artifact implementasi:** `bootstrap/app.php` (`validateCsrfTokens` dengan pengecualian
  `webhook/*`), `app/Http/Middleware/CheckoutRateLimit.php`, `config/fofa.php`
  (`checkout_ip` default 5/1 menit, `checkout_user` 20/60 menit, `checkout_target` 6/10 menit),
  `app/Services/PaymentSignatureVerifier.php`, `app/Services/CreatesUserWithRole.php`,
  `app/Services/AdminService.php`, `app/Models/Transaction.php` (cast `encrypted`),
  `app/Http/Middleware/RoleMiddleware.php`.

| Sumber Stimulus                       | Stimulus                                  | Environment                                        | Artifact                                                      | Respons                                          | Ukuran Respons                                                  |
|---------------------------------------|-------------------------------------------|----------------------------------------------------|---------------------------------------------------------------|--------------------------------------------------|-----------------------------------------------------------------|
| Browser, POST /checkout               | POST tanpa/menggunakan _token tidak valid | Sesi web Laravel aktif                             | bootstrap/app.php validateCsrfTokens (webhook/* dikecualikan) | HTTP 419; transaksi tidak dibuat                 | 100% request tanpa token valid ditolak 419                      |
| Klien dari 1 IP                       | 6x POST /checkout dalam 1 menit           | Limit IP default checkout_ip.max=5                 | app/Http/Middleware/CheckoutRateLimit.php                     | HTTP 429 pada percobaan ke-6                     | 0 transaksi dibuat dari request yang dibatasi                   |
| Klien, POST /webhook/payment-callback | Signature HMAC salah atau beda 1 byte     | PAYMENT_GATEWAY_SECRET ter-set; webhook tanpa auth | PaymentSignatureVerifier (hash_hmac sha256 + hash_equals)     | HTTP 200 generik; status transaksi tidak berubah | 0 transaksi berubah status; signature beda 1 byte tetap ditolak |
| Aksi admin, buat user/admin baru      | Simpan password baru                      | App Laravel (facade Hash)                          | CreatesUserWithRole & AdminService -> Hash::make              | Kolom users.password berisi hash                 | 100% nilai diawali $2y$ (bcrypt); 0 plaintext                   |
| Checkout produk Minecraft             | Simpan delivered_minecraft_password       | MySQL; APP_KEY terisi                              | app/Models/Transaction.php cast encrypted                     | Nilai kolom tersimpan terenkripsi                | 0 nilai plaintext pada 100% record                              |
| User tanpa role admin/owner           | GET /admin/dashboard dan GET/owner/admins | Sesi login customer/reseller                       | RoleMiddleware (alias role) + scopeForAdmin()                 | HTTP 403 atau redirect                           | 100% akses non-admin ditolak 403                                |

### NFR-SEC-002 — Autentikasi & otorisasi API reseller (FR-010)

- **Sumber:** SRS §3.2 (Role-based access) diperinci untuk API reseller pada
  `routes/api.php` prefix `v1` dengan middleware `auth:sanctum` dan `role:reseller`.
- **Anchored FR:** FR-010 *Top-up via API* (matrix § Full Traceability Matrix → FR-010) —
  endpoint `POST /api/v1/topup` yang juga memicu deduct wallet atomik (BR-WAL-001).
- **Requirement:** Setiap endpoint `/api/v1/*` (products, topup, topup status, balance) wajib
  autentikasi Sanctum **dan** role reseller; token yang tidak ada, salah role, atau sudah
  dicabut harus ditolak sebelum menyentuh bisnis logika.
- **Artifact implementasi:** `routes/api.php` baris 8 (`Route::prefix('v1')->middleware(['auth:sanctum', 'role:reseller'])`),
  `app/Http/Controllers/Admin/ResellerTokenController.php` (provisioning/revocation token oleh
  admin), tabel `personal_access_tokens`. Catatan: kolom `reseller_profiles.api_key` /
  `api_secret` ada di skema tetapi tidak dipakai jalur auth (GAP-09).

| Sumber Stimulus           | Stimulus                                        | Environment                            | Artifact                                                       | Respons                     | Ukuran Respons                                   |
|---------------------------|-------------------------------------------------|----------------------------------------|----------------------------------------------------------------|-----------------------------|--------------------------------------------------|
| Klien API, tanpa token    | GET /api/v1/products tanpa header Authorization | routes/api.php prefix v1               | Middleware auth:sanctum                                        | HTTP 401 Unauthorized       | 100% request tanpa token ditolak 401             |
| Klien API, token customer | POST /api/v1/topup memakai token role customer  | Sesi/token Sanctum valid               | Middleware role:reseller (RoleMiddleware)                      | HTTP 403 Forbidden          | 0 transaksi dibuat dari token non-reseller       |
| Klien API, token reseller | POST /api/v1/topup dengan saldo wallet cukup    | MySQL InnoDB; wallet tersedia          | Api/V1/TopupController::store (wallet atomik)                  | HTTP 200 + ref_id transaksi | 1 transaksi tercatat; selisih saldo vs harga = 0 |
| Klien API, token dicabut  | Request ulang dengan token setelah admin revoke | Admin revoke via panel reseller tokens | ResellerTokenController::destroy; tabel personal_access_tokens | HTTP 401 Unauthorized       | 100% token ter-revoke ditolak 401                |

### NFR-SEC-003 — Pencegahan SQL injection & XSS

- **Sumber:** SRS §3 (kategori §3.2) — *SQL injection prevention (Eloquent ORM)*,
  *XSS prevention (Blade `{{ }}` escaping)*.
- **Anchored FR:** FR-002 (search) dan FR-005 (checkout) di matrix (kolom Non-Functional,
  § NFR Traceability); juga mendukung FR-003 (cek ID) yang tidak dipetakan di matrix.
- **Requirement:** Semua input pengguna diproses lewat Eloquent ORM (tanpa SQL string mentah)
  dan semua output HTML di-escape Blade `{{ }}`.
- **Artifact implementasi:** `SearchController` (escaping LIKE `\\`, `%`, `_`),
  `GameIdCheckController` (validasi input), seluruh view Blade.

| Sumber Stimulus              | Stimulus                                            | Environment            | Artifact                                        | Respons                                | Ukuran Respons                                                        |
|------------------------------|-----------------------------------------------------|------------------------|-------------------------------------------------|----------------------------------------|-----------------------------------------------------------------------|
| Klien, GET /search           | q=%27%20OR%20%271%27%3D%271 (payload SQL injection) | Staging; MySQL aktif   | Eloquent ORM + escaping LIKE (SearchController) | HTTP 200 tanpa error SQL               | 0 pesan error SQL di response/log; hanya produk is_active=true tampil |
| Klien, form/kueri storefront | Isi field dengan <script>alert(1)</script>          | Sesi web Laravel aktif | Blade escaping {{ }} pada semua view            | Payload di-escape menjadi teks literal | 0 tag <script> mentah pada response (raw tag count = 0)               |

---

## 3. Availability (SRS §3.3)

### NFR-AVL-001 — Queue retry 3 attempts per job

- **Sumber:** SRS §3 (kategori §3.3) — *Queue retry: 3 attempts per job*
- **Anchored FR:** FR-010/FR-016 (fulfillment & retry transaksi) — keduanya kini dirujuk di
  matrix (§ NFR Traceability).
- **Requirement (verbatim):** Queue retry: 3 attempts per job.
- **Artifact implementasi:** `FulfillTransactionJob::tries(): int { return 3; }` dengan
  `backoff() = [30, 120, 300]`; `SendResellerCallbackJob::tries(): int { return 3; }`;
  worker `fofo-queue-worker_00` via `deploy/supervisor-worker.conf`.

| Sumber Stimulus    | Stimulus                                             | Environment                                          | Artifact                                                                                 | Respons                            | Ukuran Respons                               |
|--------------------|------------------------------------------------------|------------------------------------------------------|------------------------------------------------------------------------------------------|------------------------------------|----------------------------------------------|
| Queue worker Redis | Job fulfill yang selalu gagal dikirim ke distributor | Supervisor fofo-queue-worker_00 (docs/deployment.md) | FulfillTransactionJob::tries()=3, backoff 30/120/300; SendResellerCallbackJob::tries()=3 | Job berpindah ke tabel failed_jobs | Tepat 3 attempt tercatat, tanpa attempt ke-4 |

### NFR-AVL-002 — Payment idempotency: ref_id unique + lockForUpdate

- **Sumber:** SRS §3 (kategori §3.3) — *Payment idempotency: ref_id unique + lockForUpdate*
- **Anchored FR:** FR-013 *Webhook callback* — kini dirujuk di matrix (§ NFR Traceability).
- **Requirement (verbatim):** Payment idempotency: ref_id unique + lockForUpdate.
- **Artifact implementasi:** `transactions.ref_id` UNIQUE
  (`database/migrations/2026_08_26_000007_create_transactions_table.php` baris 13);
  `PaymentWebhookController` (lookup `lockForUpdate()` di dalam `DB::transaction`);
  `wallet_transactions` UNIQUE (`reference_id`, `type`); endpoint
  `POST /webhook/payment-callback` (CSRF dikecualikan — wajar untuk webhook eksternal;
  rate limit infra tidak diterapkan di endpoint ini, detail konfigurasi disembunyikan).

| Sumber Stimulus               | Stimulus                                       | Environment                                          | Artifact                                                                                             | Respons                               | Ukuran Respons                                         |
|-------------------------------|------------------------------------------------|------------------------------------------------------|------------------------------------------------------------------------------------------------------|---------------------------------------|--------------------------------------------------------|
| Payment gateway, POST webhook | Kirim callback ref_id sama 2x (replay identik) | Webhook bebas CSRF; tanpa rate limit di lapisan nginx (kebijakan: andalan idempotency) | transactions.ref_id UNIQUE (migration 2026_08_26_000007) + lockForUpdate di PaymentWebhookController | HTTP 200; status transaksi tetap PAID | 1 baris payment dan delta saldo wallet = 0 pada replay |

### NFR-AVL-003 — Wallet atomicity: DB::transaction + lockForUpdate

- **Sumber:** SRS §3 (kategori §3.3) — *Wallet atomicity: DB::transaction + lockForUpdate*
- **Anchored FR:** FR-010 *Top-up API* (deduct wallet atomik) & FR-018 kelola wallet reseller.
- **Requirement (verbatim):** Wallet atomicity: DB::transaction + lockForUpdate.
- **Artifact implementasi:** `app/Services/WalletService.php` (baris 46/54/82/89/111/118:
  `DB::transaction` + `Wallet::where(...)->lockForUpdate()`), `app/Services/WalletRefundService.php`.

| Sumber Stimulus       | Stimulus                                           | Environment                          | Artifact                                                        | Respons                                         | Ukuran Respons                                        |
|-----------------------|----------------------------------------------------|--------------------------------------|-----------------------------------------------------------------|-------------------------------------------------|-------------------------------------------------------|
| 2 klien API bersamaan | Deduct wallet serentak sampai melewati batas saldo | MySQL InnoDB; QUEUE_CONNECTION=redis | app/Services/WalletService.php: DB::transaction + lockForUpdate | Hanya permintaan dengan saldo cukup yang sukses | Saldo akhir >= 0; 0 partial write; selisih mutasi = 0 |

### NFR-AVL-004 — Scheduled tasks: expire pending (1 min) & sync fulfillment (5 min)

- **Sumber:** SRS §3 (kategori §3.3) — *Scheduled tasks: expire pending (1 min), sync fulfillment (5 min)*
- **Anchored FR:** FR-016 *Retry transaksi gagal* / FR-011 cek status — keduanya kini dirujuk
  di matrix (§ NFR Traceability).
- **Requirement (verbatim):** Scheduled tasks: expire pending (1 min), sync fulfillment (5 min).
- **Artifact implementasi:** `routes/console.php` baris 40
  `->everyMinute()->name('expire-pending-transactions')` dan baris 42
  `Schedule::job(new SyncWaitingFulfillmentJob)->everyFiveMinutes()`;
  scheduler berjalan sebagai `fofo-scheduler` di supervisor (`docs/deployment.md` baris 139).

| Sumber Stimulus                       | Stimulus                                       | Environment                     | Artifact                                                                        | Respons                                           | Ukuran Respons                                                 |
|---------------------------------------|------------------------------------------------|---------------------------------|---------------------------------------------------------------------------------|---------------------------------------------------|----------------------------------------------------------------|
| Scheduler (supervisor fofo-scheduler) | Transaksi pending melewati batas waktu invoice | Produksi; cron/supervisor aktif | routes/console.php ->everyMinute() name expire-pending-transactions             | Status transaksi berubah expired + email terkirim | Eksekusi tiap <= 60 detik; expired <= 1 menit setelah batas    |
| Scheduler (supervisor fofo-scheduler) | Transaksi waiting fulfillment menumpuk         | Produksi; queue worker aktif    | routes/console.php Schedule::job(SyncWaitingFulfillmentJob)->everyFiveMinutes() | Job sync fulfillment berjalan                     | Interval <= 5 menit; tiap transaksi waiting di-sync <= 5 menit |

---

## 4. Scalability (SRS §3.4)

### NFR-SCP-001 — Single VPS deployment (2 vCPU, 2GB RAM)

- **Sumber:** SRS §3 (kategori §3.4) — *Single VPS deployment (2 vCPU, 2GB RAM)*
- **Anchored FR:** system-wide.
- **Requirement (verbatim):** Single VPS deployment (2 vCPU, 2GB RAM).
- **Artifact implementasi:** `docs/deployment.md` baris 7 (*VPS 2 VCPU + 2 GB RAM + 2 GB SWAP
  (minimum)*), `deploy/OPTIMIZATION.md` (rincian 1250 MB PHP-FPM, 50 MB Redis, 20 MB Nginx).

| Sumber Stimulus             | Stimulus                             | Environment                                            | Artifact                                                             | Respons                  | Ukuran Respons                                           |
|-----------------------------|--------------------------------------|--------------------------------------------------------|----------------------------------------------------------------------|--------------------------|----------------------------------------------------------|
| Monitor host (free -h, top) | Beban puncak 60 req/s selama 5 menit | VPS 2 vCPU + 2 GB RAM + 2 GB SWAP (docs/deployment.md) | docs/deployment.md; deploy/OPTIMIZATION.md (rincian 1250 MB PHP-FPM) | Konsumsi RAM/CPU terukur | RAM total <= 2 GB (SWAP <= 2 GB); 0 proses kena OOM-kill |

### NFR-SCP-002 — Redis untuk cache + queue + session

- **Sumber:** SRS §3 (kategori §3.4) — *Redis untuk cache + queue + session*
- **Anchored FR:** FR-010 (Top-up API) & FR-013 (Webhook callback) — dirujuk di matrix
  (§ NFR Traceability; kolom Non-Functional).
- **Requirement (verbatim):** Redis untuk cache + queue + session.
- **Artifact implementasi:** default `config/cache.php` = `redis`,
  `config/queue.php` = `redis`, `config/session.php` = `redis`; `.env.example` baris 30/40/42;
  panduan `docs/redis-setup.md`.

| Sumber Stimulus       | Stimulus                                          | Environment                                       | Artifact                                                                    | Respons                                | Ukuran Respons                                            |
|-----------------------|---------------------------------------------------|---------------------------------------------------|-----------------------------------------------------------------------------|----------------------------------------|-----------------------------------------------------------|
| Health check aplikasi | Cek store cache/queue/session lalu redis-cli ping | CACHE_STORE/QUEUE_CONNECTION/SESSION_DRIVER=redis | config/cache.php, config/queue.php, config/session.php; docs/redis-setup.md | 3 store memakai Redis; koneksi terbuka | redis-cli ping = PONG; 0 fallback ke driver file/database |

### NFR-SCP-003 — PHP-FPM 25 workers

- **Sumber:** SRS §3 (kategori §3.4) — *PHP-FPM 25 workers*
- **Anchored FR:** system-wide.
- **Requirement (verbatim):** PHP-FPM 25 workers.
- **Artifact implementasi:** `docs/deployment.md` §5 — `pm.max_children = 25`
  (25 workers × ~50MB = 1.25GB RAM), `pm.max_requests = 500`; `deploy/php-fpm-pool.conf`.

| Sumber Stimulus          | Stimulus                                    | Environment                            | Artifact                                                | Respons                          | Ukuran Respons                                  |
|--------------------------|---------------------------------------------|----------------------------------------|---------------------------------------------------------|----------------------------------|-------------------------------------------------|
| Inspeksi runtime PHP-FPM | Baca pm.max_children dan hitung proses pool | VPS produksi; deploy/php-fpm-pool.conf | docs/deployment.md §5 (25 workers x ~50MB = 1.25GB RAM) | Pool berjalan sesuai konfigurasi | pm.max_children = 25; jumlah proses aktif <= 25 |

### NFR-SCP-004 — Nginx static cache + gzip

- **Sumber:** SRS §3 (kategori §3.4) — *Nginx static cache + gzip*
- **Anchored FR:** FR-001 (Browse kategori) & FR-002 (Search produk) — dirujuk di matrix
  (§ NFR Traceability; kolom Non-Functional).
- **Requirement (verbatim):** Nginx static cache + gzip.
- **Artifact implementasi:** `deploy/nginx-site.conf` — `gzip on`, `gzip_comp_level 5`,
  `gzip_min_length 256`, `gzip_types …`; lokasi aset statis `expires 30d` +
  `Cache-Control "public, immutable"`.

| Sumber Stimulus                | Stimulus                                                       | Environment                                                  | Artifact                               | Respons                                                     | Ukuran Respons                                    |
|--------------------------------|----------------------------------------------------------------|--------------------------------------------------------------|----------------------------------------|-------------------------------------------------------------|---------------------------------------------------|
| Klien GET aset statis          | GET /build/assets/*.css atau *.js dengan Accept-Encoding: gzip | Nginx (gzip on, gzip_min_length 256, deploy/nginx-site.conf) | Header response Nginx                  | Content-Encoding: gzip terkirim                             | 100% aset teks >= 256 byte mengembalikan gzip     |
| Klien GET aset statis berulang | Permintaan ulang gambar/css/js yang sama                       | deploy/nginx-site.conf location aset (expires 30d)           | Header Cache-Control public, immutable | Cache-Control max-age 30 hari terkirim; PHP tidak dipanggil | 100% aset membawa max-age 30 hari (2592000 detik) |

---

## Traceability

Pemetaan NFR → `docs/analyst/requirements-matrix.md`, dirujuk lewat anchor stabil (nama seksi
dan id FR, bukan nomor baris yang bisa bergeser). Per 2026-09-26 matrix memuat seksi **§ NFR
Traceability** yang mencakup seluruh 14 id: 11 ber-status *Referenced* (punya pemilik baris FR
pada kolom **Non-Functional** di § Full Traceability Matrix) dan 3 ber-status *System-wide*
(NFR-PERF-002, NFR-SCP-001, NFR-SCP-003 — sengaja tidak dipetakan ke FR tertentu dan dihitung
N/A pada § Coverage Summary). Kolom FR pemilik di bawah mengikuti § NFR Traceability matrix.

10 baris FR di § Full Traceability Matrix menyebut id NFR pada kolom **Non-Functional**
(FR-001, FR-002, FR-003, FR-004, FR-005, FR-010, FR-011, FR-013, FR-016, FR-018).

| NFR ID       | Anchor di requirements-matrix.md | FR pemilik (matrix)    | Kolom matrix   | Status                                                            |
|--------------|----------------------------------|------------------------|----------------|-------------------------------------------------------------------|
| NFR-PERF-001 | § NFR Traceability               | FR-003                 | Non-Functional | Referenced                                                        |
| NFR-PERF-002 | § NFR Traceability               | — (system-wide)        | —              | System-wide (N/A)                                                 |
| NFR-PERF-003 | § NFR Traceability               | FR-001, FR-002, FR-004 | Non-Functional | Referenced                                                        |
| NFR-SEC-001  | § NFR Traceability               | FR-005                 | Non-Functional | Referenced                                                        |
| NFR-SEC-002  | § NFR Traceability               | FR-010                 | Non-Functional | Referenced                                                        |
| NFR-SEC-003  | § NFR Traceability               | FR-002, FR-005         | Non-Functional | Referenced                                                        |
| NFR-AVL-001  | § NFR Traceability               | FR-010, FR-016         | Non-Functional | Referenced                                                        |
| NFR-AVL-002  | § NFR Traceability               | FR-013                 | Non-Functional | Referenced                                                        |
| NFR-AVL-003  | § NFR Traceability               | FR-010, FR-018         | Non-Functional | Referenced                                                        |
| NFR-AVL-004  | § NFR Traceability               | FR-011, FR-016         | Non-Functional | Referenced                                                        |
| NFR-SCP-001  | § NFR Traceability               | — (system-wide)        | —              | System-wide (N/A)                                                 |
| NFR-SCP-002  | § NFR Traceability               | FR-010, FR-013         | Non-Functional | Referenced                                                        |
| NFR-SCP-003  | § NFR Traceability               | — (system-wide)        | —              | System-wide (N/A)                                                 |
| NFR-SCP-004  | § NFR Traceability               | FR-001, FR-002         | Non-Functional | Referenced                                                        |

Catatan: kolom FR pemilik awalnya ditulis sebagai kandidat/usulan dari dokumen ini; matrix kini
mengadopsinya apa adanya (§ NFR Traceability matrix: "tidak ada link FR↔NFR yang ditambahkan
di luar usulan dokumen itu"). Status *System-wide* untuk NFR-PERF-002, NFR-SCP-001, dan
NFR-SCP-003 ditetapkan matrix (N/A pada Coverage Summary) dan sudah diselaraskan di dokumen ini.
Penambahan/pengubahan baris di `requirements-matrix.md` bukan wewenang dokumen ini (file
tersebut dimiliki agen lain).

---

## Known Clarity Gaps

Daftar bagian dari SRS §3 yang **tidak** bisa diberi ukuran yang jujur tanpa keputusan tambahan.
Ini defect dokumen, bukan defect implementasi — kecuali yang ditandai *Implementation mismatch*.

| ID     | NFR                      | Jenis                   | Gap (apa yang hilang)                                                                                                                                                                                                                                       | Dampak / tindakan                                                                                                                                                                      |
|--------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| GAP-01 | NFR-PERF-001             | Missing target          | SRS hanya menulis "Response time < 200ms (cached)" tanpa statistik (mean/median/p95/p99) dan tanpa window pengukuran                                                                                                                                        | Ukuran Respons memakai TTFB per request; perlu keputusan apakah p95 atau rata-rata                                                                                                     |
| GAP-02 | NFR-PERF-001             | Missing target          | Tidak ada target latency untuk kondisi cache miss maupun panggilan eksternal Digiflazz (timeout implementasi DISTRIBUTOR_TIMEOUT_MS=5000 bukan target SRS)                                                                                                  | Batas waktu cek game ID saat provider lambat belum terukur                                                                                                                             |
| GAP-03 | NFR-PERF-001             | Implementation mismatch | Route /search, /, dan /products/{id} tidak punya cache aplikasi (verifikasi: tidak ada pemanggilan Cache:: pada controller storefront); yang di-cache hanya hasil cek game ID (300 detik)                                                                   | Syarat "(cached)" pada jalur pencarian tidak terpenuhi implementasi; baris ke-2 skenario hanya bisa diuji apa adanya                                                                   |
| GAP-04 | NFR-PERF-002             | Missing target          | SRS tidak menetapkan durasi sustain, mix request, maupun batas error rate untuk "~60 req/s"                                                                                                                                                                 | Skenario memakai asumsi 60 detik + error <= 1%; perlu ratifikasi                                                                                                                       |
| GAP-05 | NFR-PERF-003             | Missing target          | SRS tidak menetapkan window pengukuran maupun definisi "product listings" untuk cache hit > 80%; belum ada instrumentasi metrik cache                                                                                                                       | Hit rate tidak bisa dipantau sampai ada logging metrik cache                                                                                                                           |
| GAP-06 | NFR-AVL-001..004         | Missing target          | SRS §3.3 tidak memuat target uptime %, RPO/RTO, SLA penyelesaian job, maupun ukuran antrean maksimum; monitoring/alerting masih Open Question                                                                                                               | Availability hanya terukur lewat interval scheduler dan jumlah attempt                                                                                                                 |
| GAP-07 | NFR-SCP-001..004         | Missing target          | SRS §3.4 tidak memuat target pertumbuhan disk/data maupun threshold kapan harus scale up                                                                                                                                                                    | Kapasitas hanya dibatasi CPU/RAM/workers                                                                                                                                               |
| GAP-08 | NFR-SEC-001, NFR-SEC-002 | Missing target          | SRS tidak menetapkan nilai rate limit (nilai ada di config/fofa.php, env-driven), tidak ada kuota/rate limit untuk API reseller (grup route hanya auth:sanctum + role:reseller), dan tidak ada kebijakan masa berlaku/rotasi token                          | Baseline angka sebelum diuji harus diratifikasi; route routes/web.php baris 39 menulis "per user 10/jam, per product+target 3/10m" sedangkan default config 20/60 menit dan 6/10 menit |
| GAP-09 | NFR-SEC-002              | Implementation mismatch | Kolom reseller_profiles.api_key (unique + nullable) dan api_secret (nullable) ada di skema tetapi tidak pernah dibaca jalur autentikasi mana pun; autentikasi nyata memakai Sanctum personal access token                                                   | Perlu keputusan: pakai api_key/api_secret untuk signing H2H atau hapus dari skema                                                                                                      |
| GAP-10 | NFR-SEC-001              | Missing target          | SRS §3.2 tidak memuat target patching/dependency scan, HTTPS wajib di semua rute, maupun enkripsi at-rest di luar kredensial Minecraft                                                                                                                      | Perlindungan lain (security header Nginx, IP whitelist webhook) ada di kode tapi bukan requirement terukur                                                                             |
| GAP-11 | Semua NFR                | Traceability mismatch   | Dokumen ini mencatat mismatch lama (Coverage Summary Non-Functional = 4 dengan hanya 3 id dirujuk); setelah matrix ditambahi § NFR Traceability, hitungan kini selaras: Total 14, Confirmed 11 (Referenced), N/A 3 (NFR-PERF-002, NFR-SCP-001, NFR-SCP-003) | Selesai — matrix § NFR Traceability kini mencakup seluruh 14 id; baris dipertahankan sebagai catatan riwayat reconcile, tidak ada tindakan tersisa                                     |
| GAP-12 | NFR-SCP-004              | Missing target          | SRS tidak menetapkan ukuran pengurangan bandwidth yang diharapkan dari gzip maupun rasio offload PHP ke Nginx                                                                                                                                               | Pengukuran hanya memverifikasi konfigurasi aktif, bukan dampaknya                                                                                                                      |

---

## Appendix — Kutipan verbatim SRS.md §3

Disalin tanpa diubah dari `docs/analyst/SRS.md` §3 *Non-Functional Requirements* (§3.1–§3.4)
— **versi sebelum §3 dijadikan pointer ke dokumen ini**; kini SRS §3 hanya berisi tabel pointer
14 id dan menunjuk ke bagian Appendix ini, sehingga baris aslinya tidak ada lagi di SRS.
Kutipan dipertahankan agar seluruh angka target ada di satu berkas:

```
## 3. Non-Functional Requirements

### 3.1 Performance
- Response time < 200ms (cached)
- Throughput ~60 req/s (2 vCPU, 2GB RAM)
- Cache hit rate > 80% untuk product listings

### 3.2 Security
- CSRF protection (Laravel tokens)
- SQL injection prevention (Eloquent ORM)
- XSS prevention (Blade `{{ }}` escaping)
- Rate limiting (CheckoutRateLimit: IP/user/product+target)
- Payment signature: HMAC-SHA256 + timing-safe comparison
- Password hashing: bcrypt (Hash::make)
- Minecraft password: encrypted cast
- Role-based access: RoleMiddleware + scopeForAdmin()

### 3.3 Availability
- Queue retry: 3 attempts per job
- Payment idempotency: ref_id unique + lockForUpdate
- Wallet atomicity: DB::transaction + lockForUpdate
- Scheduled tasks: expire pending (1 min), sync fulfillment (5 min)

### 3.4 Scalability
- Single VPS deployment (2 vCPU, 2GB RAM)
- Redis untuk cache + queue + session
- PHP-FPM 25 workers
- Nginx static cache + gzip
```
