# Process Flows — Fofa Shop

Sistem Fofa Shop memiliki 11 proses bisnis utama yang mencakup siklus transaksi dari checkout hingga fulfillment, operasi wallet, manajemen voucher, dan scheduled tasks.

---

## PF-001: Customer Checkout

Membuat transaksi baru dari storefront.

**Trigger:** Customer mengklik "Bayar" di halaman produk.
**Goal:** Transaksi baru dibuat dengan status `pending`, menunggu pembayaran.
**Actors:**
- Customer: Mengisi form checkout
- System: Memvalidasi dan membuat transaksi

### Flow

```
START
  |
  v
[1] Customer mengisi form checkout
  |   (game_id, payment_method, voucher, guest_email)
  v
[2] System memvalidasi input
  |
  +--- VALID ------> [3] System memvalidasi game ID
  |                    |   (GameIdVerifierChain)
  |                    |
  |                    +--- VALID --> [4] System generate ref_id
  |                    |                  |
  |                    |                  v
  |                    |               [5] System buat Transaction + Payment
  |                    |                  (status: pending)
  |                    |                  |
  |                    |                  v
  |                    |               [6] System apply voucher (jika ada)
  |                    |                  |
  |                    |                  v
  |                    |               END (redirect ke invoice)
  |                    |
  |                    +--- INVALID --> [7] System return error 422
  |                                       |
  |                                       v
  +--- INVALID ----> [8] System return error 422    END
                       |
                       v
                    END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Customer | Mengisi form checkout | Menampilkan form |
| 2 | System | Validasi input | Cek product active, payment method active |
| 3 | System | Validasi game ID | GameIdVerifierChain → Digiflazz API → fallback regex |
| 4 | System | Generate ref_id | Format FOFA-YYYYMMDD-XXXXXXXX, cek uniqueness |
| 5 | System | Buat transaksi | Create Transaction + Payment (status: pending) |
| 6 | System | Apply voucher | VoucherService::applyAndRecord (atomic lock) |
| 7 | System | Return error | 422 validation error |
| 8 | System | Rate limit | 429 Too Many Requests |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | Input valid? | Step 3 | Step 8 |
| 3 | Game ID valid? | Step 4 | Step 7 |
| 6 | Voucher provided? | Step 6a (apply) | END |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 3 | Digiflazz down | Fallback to local regex | Still allows checkout |
| 6 | Voucher expired | Reject with 422 | Customer removes voucher |

---

## PF-002: Payment Callback

Payment gateway mengirim notifikasi pembayaran.

**Trigger:** Payment gateway mengirim webhook ke `/webhook/payment-callback`.
**Goal:** Verifikasi pembayaran dan memulai fulfillment.
**Actors:**
- Payment Gateway: Mengirim callback
- System: Memverifikasi dan memproses

### Flow

```
START
  |
  v
[1] Gateway kirim webhook
  |   (ref_id, amount, signature, status)
  v
[2] System verifikasi signature
  |   (PaymentSignatureVerifier::verify)
  |
  +--- VALID ------> [3] System find Transaction (lockForUpdate)
  |                    |
  |                    +--- FOUND --> [4] Cek idempotency
  |                    |                  |
  |                    |                  +--- ALREADY PAID --> skip
  |                    |                  |
  |                    |                  +--- NOT YET ----> [5] Buat Payment record
  |                    |                                      |
  |                    |                                      v
  |                    |                                   [6] Dispatch FulfillTransactionJob
  |                    |                                      |
  |                    |                                      v
  |                    |                                   END (return 200)
  |                    |
  |                    +--- NOT FOUND --> return 404
  |
  +--- INVALID ----> return 401 (signature mismatch)
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Gateway | POST webhook | Menerima data |
| 2 | System | Verifikasi signature | hash_hmac('sha256', ref_id + amount, secret) |
| 3 | System | Cari transaksi | lockForUpdate() untuk cegah race condition |
| 4 | System | Cek idempotency | Skip jika sudah paid/expired |
| 5 | System | Buat payment | Create Payment record dengan gateway_ref |
| 6 | System | Dispatch job | FulfillTransactionJob ke queue |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | Signature valid? | Step 3 | Return 401 |
| 3 | Transaksi ditemukan? | Step 4 | Return 404 |
| 4 | Sudah diproses? | Skip | Step 5 |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 2 | Signature mismatch | Return 401 | Investigate |
| 3 | Race condition | lockForUpdate waits | Automatic |
| 4 | Late payment (expired) | Reconcile: expired→paid→processing | Auto re-fulfill |

---

## PF-003: Transaction Fulfillment

Memproses top-up ke distributor.

**Trigger:** FulfillTransactionJob di-dispatch dari payment callback.
**Goal:** Top-up berhasil dikirim ke distributor.
**Actors:**
- System: Memproses fulfillment
- Digiflazz API: Menerima/memproses top-up

### Flow

```
START
  |
  v
[1] Job mulai handle
  |
  v
[2] Cek status transaksi
  |
  +--- PAID/PROCESSING --> [3] Cek kategori produk
  |                            |
  |                            +--- MINECRAFT --> [4] Set waiting_fulfillment
  |                            |                      |
  |                            |                      v
  |                            |                   END (manual fulfill)
  |                            |
  |                            +--- OTHER ------> [5] Set processing
  |                                                 |
  |                                                 v
  |                                              [6] Hit DistributorService::topUp()
  |                                                 |
  |                                                 +--- SUCCESS --> [7] Set success + kirim email
  |                                                 |
  |                                                 +--- PENDING --> [8] Set waiting_fulfillment
  |                                                 |
  |                                                 +--- FAILED ---> [9] Retry atau terminal fail
  |
  +--- OTHER STATUS ----> skip (idempotency)
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Queue | Mulai job | Ambil transaction ID |
| 2 | System | Cek status | Hanya proses paid/processing |
| 3 | System | Cek kategori | Minecraft → manual, lainnya → API |
| 4 | System | Set waiting_fulfillment | Menunggu admin manual fulfill |
| 5 | System | Set processing | Tandai sedang diproses |
| 6 | System | Hit distributor | DistributorService::topUp() |
| 7 | System | Handle success | Set success, completed_at, kirim email |
| 8 | System | Handle pending | Set waiting_fulfillment, sync via cron |
| 9 | System | Handle failed | Retry (30s, 120s, 300s) atau terminal fail |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | Status paid/processing? | Step 3 | Skip |
| 3 | Kategori Minecraft? | Step 4 | Step 5 |
| 6 | Has vendor mapping? | Hit API | Immediate fail (422) |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 6 | Missing vendor mapping | Set Failed (terminal) | Admin check product config |
| 6 | HTTP timeout | Retry with backoff | Auto (3 attempts) |
| 6 | HTTP 5xx | Retry with backoff | Auto (3 attempts) |
| 9 | Max retries reached | Set Failed (terminal) | Admin manual retry |

### Metrics

- **Average Duration:** 5-15 seconds (API call + processing)
- **Max Retries:** 3
- **Backoff:** 30s, 120s, 300s

---

## PF-004: Minecraft Manual Fulfillment

Admin memproses transaksi Minecraft secara manual.

**Trigger:** Admin mengklik "Mark as Success" di transaction detail.
**Goal:** Transaksi selesai, akun Minecraft dikirim ke buyer.
**Actors:**
- Admin: Mengisi credentials
- System: Menyimpan dan mengirim email

### Flow

```
START
  |
  v
[1] Admin isi form:
  |   username, password, delivery_note
  v
[2] System validasi:
  |   status = waiting_fulfillment
  |   kategori = minecraft
  v
[3] System simpan credentials
  |   (password di-encrypt)
  v
[4] System set status: success
  |   + completed_at
  v
[5] System kirim email
  |   (MinecraftAccountDeliveredMail, queued)
  v
[6] System buat audit log
  |
  v
END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Admin | Isi form credentials | Menampilkan form |
| 2 | System | Validasi | Cek status + kategori |
| 3 | System | Simpan | Encrypt password, simpan ke DB |
| 4 | System | Update status | Set success + completed_at |
| 5 | System | Kirim email | Queue MinecraftAccountDeliveredMail |
| 6 | System | Audit log | Catat aksi admin |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | Status waiting_fulfillment + MC? | Step 3 | Reject (422) |

---

## PF-005: Reseller Auto-Refund

Refund otomatis ke wallet reseller saat transaksi gagal.

**Trigger:** TransactionFailed event di-dispatch.
**Goal:** Saldo wallet reseller dikembalikan.
**Actors:**
- System: Memproses refund
- Reseller: Menerima refund

### Flow

```
START
  |
  v
[1] TransactionFailed event dispatched
  |
  v
[2] RefundResellerListener::handle()
  |   (runs synchronously)
  v
[3] WalletRefundService::refundIfReseller()
  |
  +--- IS RESELLER --> [4] Cek idempotency
  |                       |   (existing refund for this txn?)
  |                       |
  |                       +--- EXISTS --> skip (already refunded)
  |                       |
  |                       +--- NEW ----> [5] DB::transaction + lockForUpdate
  |                                        |
  |                                        v
  |                                     [6] Buat/update wallet
  |                                        |
  |                                        v
  |                                     [7] Tambah wallet_transaction (type: refund)
  |                                        |
  |                                        v
  |                                     [8] Kirim ResellerAutoRefundedMail
  |                                        |
  |                                        v
  |                                      END
  |
  +--- NOT RESELLER --> skip
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | System | Event dispatched | TransactionFailed |
| 2 | Listener | Handle event | Synchronous (no queue) |
| 3 | Service | Check role | Hanya reseller |
| 4 | Service | Check idempotency | Cek existing refund |
| 5 | Service | Lock rows | DB::transaction + lockForUpdate |
| 6 | Service | Ensure wallet | Get or create wallet |
| 7 | Service | Record refund | Create wallet_transaction |
| 8 | Service | Send email | Queue mail (failure logged) |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 3 | User is reseller? | Step 4 | Skip |
| 4 | Refund exists? | Skip | Step 5 |

---

## PF-006: Voucher Apply

Menerapkan voucher diskon saat checkout.

**Trigger:** Customer memasukkan kode voucher di checkout.
**Goal:** Diskon diterapkan ke transaksi.
**Actors:**
- Customer: Memasukkan kode voucher
- System: Memvalidasi dan menghitung diskon

### Flow

```
START
  |
  v
[1] Customer masukkan kode voucher
  |
  v
[2] System cari voucher (case-insensitive)
  |   lockForUpdate()
  |
  +--- FOUND ----> [3] Validasi:
  |                    |
  |                    +--- is_active? ---- NO --> reject
  |                    +--- within period? NO --> reject
  |                    +--- uses < max?   NO --> reject
  |                    +--- amount >= min? NO --> reject
  |                    +--- one-use-per-user? YES --> reject
  |                    |
  |                    +--- ALL PASS --> [4] Hitung diskon
  |                                        |
  |                                        +--- percentage --> amount × (value/100)
  |                                        +--- fixed --> value
  |                                        |
  |                                        +--- Cap: min(diskon, amount - 1)
  |                                        |
  |                                        v
  |                                     [5] Record usage + increment used_count
  |                                        |
  |                                        v
  |                                      END (return discount_amount)
  |
  +--- NOT FOUND --> reject (kode tidak valid)
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Customer | Masukkan kode | Input voucher code |
| 2 | System | Cari voucher | LOWER(code), lockForUpdate |
| 3 | System | Validasi 5 checks | Semua harus pass |
| 4 | System | Hitung diskon | Percentage atau fixed, cap at amount-1 |
| 5 | System | Record | VoucherUsage + increment used_count |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | Voucher ditemukan? | Step 3 | Reject |
| 3 | Semua valid? | Step 4 | Reject per check |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 2 | Voucher not found | ValidationException | Customer coba kode lain |
| 3 | Expired | ValidationException | — |
| 3 | Max uses reached | ValidationException | — |

---

## PF-007: Reseller API Top-up

Reseller membuat transaksi via API.

**Trigger:** Reseller mengirim POST /api/v1/topup.
**Goal:** Transaksi dibuat dan fulfillment dimulai.
**Actors:**
- Reseller: Mengirim API request
- System: Memproses dan fulfill

### Flow

```
START
  |
  v
[1] Reseller kirim POST /api/v1/topup
  |   (auth: Sanctum token + role:reseller)
  v
[2] System validasi:
  |   product exists, game ID valid, wallet balance >= price
  v
[3] System deduct wallet (atomic)
  |   WalletService::deduct (lockForUpdate)
  v
[4] System buat Transaction + Payment
  |   (status: paid — langsung paid karena wallet)
  v
[5] System dispatch FulfillTransactionJob
  |
  v
[6] Return 201 with ref_id
  |
  v
END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Reseller | POST /api/v1/topup | Auth + data |
| 2 | System | Validasi | Product, game ID, balance |
| 3 | System | Deduct wallet | lockForUpdate, cek balance |
| 4 | System | Buat transaksi | Transaction + Payment (paid) |
| 5 | System | Dispatch job | FulfillTransactionJob |
| 6 | System | Return | 201 Created |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | Semua valid? | Step 3 | Return 422 |
| 3 | Balance cukup? | Step 4 | Return 422 (insufficient) |

---

## PF-008: Expire Pending Transactions (Scheduled)

Cron job expire transaksi yang belum dibayar.

**Trigger:** Laravel Scheduler setiap menit.
**Goal:** Transaksi kedaluwarsa di-expire.
**Actors:**
- System: Scheduler + Closure

### Flow

```
START (every minute)
  |
  v
[1] Query: Transaction WHERE status = 'pending'
  |         AND expired_at < NOW()
  v
[2] For each expired transaction:
  |
  +---> [3] Set status: expired
  |           |
  |           v
  |        [4] Queue TransactionExpiredMail
  |           |
  |           v
  |        END (next transaction)
  |
  v
END (all processed)
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | System | Query | Find pending + expired |
| 2 | System | Loop | Process each |
| 3 | System | Update status | Set expired |
| 4 | System | Send email | Queue mail |

### Metrics

- **Schedule:** Every minute
- **Batch size:** Unlimited (all matching)

---

## PF-009: Sync Waiting Fulfillment (Scheduled)

Cron job sync status transaksi menunggu fulfillment.

**Trigger:** Laravel Scheduler setiap 5 menit.
**Goal:** Update status transaksi dari distributor.
**Actors:**
- System: Scheduler + Job
- Digiflazz API: Status check

### Flow

```
START (every 5 minutes)
  |
  v
[1] Query: Transaction WHERE status = 'waiting_fulfillment'
  |         AND updated_at >= NOW() - 24h
  |         LIMIT 100
  v
[2] For each transaction:
  |
  +---> [3] DistributorService::checkStatus()
  |           |
  |           +--- SUCCESS --> [4] Set success + kirim email
  |           |
  |           +--- FAILED ---> [5] Set failed + dispatch TransactionFailed
  |           |
  |           +--- PENDING --> skip (re-check next cycle)
  |
  v
END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | System | Query | Find waiting_fulfillment, last 24h, limit 100 |
| 2 | System | Loop | Process each |
| 3 | System | Check status | Hit distributor API |
| 4 | System | Handle success | Set success, send mail |
| 5 | System | Handle failed | Set failed, trigger refund |

### Metrics

- **Schedule:** Every 5 minutes
- **Batch size:** 100 max
- **Window:** Last 24 hours

---

## PF-010: Announcement Broadcast

Mengirim pengumuman ke target users.

**Trigger:** Admin/Owner membuat announcement dan publish.
**Goal:** Notif in-app + email terkirim ke target audience.
**Actors:**
- Admin/Owner: Membuat announcement
- System: Broadcast

### Flow

```
START
  |
  v
[1] Admin buat announcement
  |   (title, content, type, target_audience)
  v
[2] AnnouncementObserver::created()
  |   + Log audit
  |   + If isPublished → dispatch SendAnnouncementJob
  v
[3] Job resolve target users:
  |   all → customer + reseller
  |   customer → customer only
  |   reseller → reseller only
  |   admin → admin + owner
  v
[4] For each target user:
  |
  +---> [5] Buat Notification record (in-app)
  |           |
  |           v
  |        [6] Jika email_notification_enabled = true:
  |               Queue AnnouncementMail
  |           |
  |           v
  |        END (next user)
  |
  v
END
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Admin | Buat announcement | Fill form |
| 2 | Observer | Created event | Audit log + dispatch job |
| 3 | Job | Resolve audience | Filter by target_audience |
| 4 | Job | Loop users | Process each |
| 5 | Job | Create notification | In-app notification |
| 6 | Job | Send email | Queue if enabled |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | isPublished? | Dispatch job | Skip |
| 6 | email enabled? | Queue mail | Skip |

---

## PF-011: Product Deletion

Hapus produk dengan aturan 3-tier.

**Trigger:** Admin/Owner menghapus produk.
**Goal:** Produk dihapus sesuai aturan bisnis.
**Actors:**
- Admin/Owner: Request hapus
- System: Validasi dan eksekusi

### Flow

```
START
  |
  v
[1] Admin/Owner request hapus produk
  |
  v
[2] ProductDeletionService::delete()
  |
  v
[3] Validasi: sudah soft-deleted?
  |   +--- YES --> reject (RuntimeException)
  |
  +--- NO --> [4] Cek: punya transaksi?
               |
               +--- YES --> [5] Soft delete + snapshot
               |               (selalu, bahkan owner dalam 24h)
               |
               +--- NO --> [6] Cek timing & role
                             |
                             +--- ≤24h + Owner --> [7] Hard delete + snapshot
                             |
                             +--- >24h / Admin --> [5] Soft delete + snapshot
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Admin/Owner | Request hapus | — |
| 2 | Service | Mulai proses | — |
| 3 | Service | Cek soft-deleted | Reject if already deleted |
| 4 | Service | Cek transaksi | Query transactions |
| 5 | Service | Soft delete | Set deleted_at + log |
| 6 | Service | Cek timing & role | Compare timestamps + role |
| 7 | Service | Hard delete | Force delete + log |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 3 | Sudah soft-deleted? | Reject | Step 4 |
| 4 | Punya transaksi? | Step 5 (always) | Step 6 |
| 6 | ≤24h + Owner? | Step 7 (hard) | Step 5 (soft) |

### Business Rules

- **BR-DEL-001:** Produk dengan transaksi → selalu soft delete
- **BR-DEL-002:** ≤24h + owner + tanpa transaksi → hard delete
- **BR-DEL-003:** Setiap hapus → buat ProductDeletionLog dengan snapshot

---

## Process Relationship Map

```
PF-001 (Checkout) --triggers--> PF-002 (Payment Callback)
PF-002 (Payment Callback) --triggers--> PF-003 (Fulfillment)
PF-003 (Fulfillment) --on-failed--> PF-005 (Auto-Refund)
PF-003 (Fulfillment) --minecraft--> PF-004 (Manual Fulfill)
PF-008 (Expire Cron) --runs-every--> 1 minute
PF-009 (Sync Cron) --runs-every--> 5 minutes
PF-010 (Announcement) --triggered-by--> Admin action
PF-011 (Product Delete) --triggered-by--> Admin action
PF-006 (Voucher) --called-by--> PF-001 (Checkout)
PF-007 (API Top-up) --triggers--> PF-003 (Fulfillment)
```
