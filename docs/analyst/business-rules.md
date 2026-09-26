# Business Rules — Fofa Shop

38 blok rules bisnis yang diekstrak dari kode — **37 aktif** (`[x] Confirmed`) + **1 superseded** (`BR-PAY-002`, duplikat yang dipertahankan agar id tidak dipakai ulang). Dikelompokkan per kategori.

---

## Rule Categories

| Code | Category | Description |
|------|----------|-------------|
| PROD | Validation | Visibility produk & kategori di storefront |
| PRICE | Calculation | Harga dan diskon |
| WAL | Calculation | Operasi wallet |
| VOUCH | Validation | Voucher |
| WF | Workflow | State machine transaksi |
| FUL | Workflow | Fulfillment |
| PAY | Validation | Pembayaran |
| AUTH | Authorization | Akses dan role |
| DEL | Workflow | Hapus produk |
| MC | Workflow | Minecraft manual fulfill |
| REF | Workflow | Refund |
| GAME | Validation | Verifikasi game ID |
| ANN | Workflow | Pengumuman |
| CHECK | Validation | Rate limiting |

---

### BR-PROD-001: Kategori Coming Soon Tampil, Beli Disabled

| Field | Value |
|-------|-------|
| **ID** | BR-PROD-001 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | CategoryStatus enum + StorefrontController + CheckoutController |

**Rule:** Kategori `status = coming_soon` tetap tampil di storefront, tetapi produk di dalamnya tidak bisa dibeli (tombol beli disabled).

**When:** User membuka halaman storefront atau halaman kategori.

**Then:**
- Kategori `is_active = true` tetap ditampilkan (badge "Segera Hadir", CTA "Segera Hadir" bukan "Lihat Produk →")
- Halaman kategori menampilkan kartu produk dalam keadaan disabled
- `StorefrontController::show` → 404 untuk produk milik kategori coming_soon
- `CheckoutController` → reject checkout produk kategori coming_soon

**Else:** Kategori `status = active` → CTA normal dan checkout diizinkan.

**Examples:**
- Kategori `coming_soon` → tile tampil dengan badge "Segera Hadir", beli disabled
- Kategori `active` → tile "Lihat Produk →" dan bisa checkout

**Related:**
- Use Case: UC-001
- Data Entity: categories, products

---

### BR-PROD-002: Hanya Produk Aktif di Storefront

| Field | Value |
|-------|-------|
| **ID** | BR-PROD-002 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | StorefrontController::category + StorefrontController::show |

**Rule:** Hanya produk `is_active = true` yang ditampilkan di storefront.

**When:** User membuka halaman kategori atau halaman detail produk.

**Then:**
- `StorefrontController::category` → filter `products.is_active = true`
- `StorefrontController::show` → 404 jika `product.is_active = false` atau kategorinya nonaktif

**Else:** Produk `is_active = false` → tidak muncul di list, 404 saat diakses langsung.

**Examples:**
- Produk nonaktif di kategori aktif → hilang dari grid
- Produk aktif di kategori nonaktif → halaman detail 404

**Related:**
- Use Case: UC-001
- Data Entity: products, categories

---

### BR-PRICE-001: Harga Per Role

| Field | Value |
|-------|-------|
| **ID** | BR-PRICE-001 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | PRD §3, CalculatePriceForUser |

**Rule:** Harga produk dihitung berdasarkan role user.

**When:** User melihat produk atau melakukan checkout.

**Then:**
- Customer/Guest → `products.price`
- Reseller (punya reseller_profiles) → `cost_price + (cost_price × markup_percentage / 100)`
- Admin/Owner → `cost_price` langsung

**Else:** Jika reseller belum punya profile, fallback ke `products.price`.

**Examples:**
- Produk price=50000, cost_price=40000, markup=10% → Reseller bayar 44000
- Admin bayar 40000 (cost_price langsung)

**Related:**
- Use Case: UC-002, UC-017
- Data Entity: products, reseller_profiles

---

### BR-PRICE-002: Voucher Discount Cap

| Field | Value |
|-------|-------|
| **ID** | BR-PRICE-002 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | VoucherService::calculateDiscount |

**Rule:** Diskon voucher tidak boleh melebihi `amount - 1` (minimal bayar Rp1).

**When:** Voucher dihitung.

**Then:**
- Percentage: `amount × (value / 100)`, cap di `amount - 1`
- Fixed: `value` langsung, cap di `amount - 1`

**Else:** Jika discount > amount, set ke `amount - 1`.

**Examples:**
- Amount=10000, percentage=50% → discount=5000
- Amount=10000, fixed=15000 → discount=9999 (cap)

**Related:**
- Use Case: — (belum dikutip chapter manapun; diimplementasi PF-006 di process-flows.md)
- Data Entity: vouchers, voucher_usages

---

### BR-WAL-001: Wallet Atomic Operations

| Field | Value |
|-------|-------|
| **ID** | BR-WAL-001 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | WalletService |

**Rule:** Semua operasi wallet (deduct, deposit, refund) menggunakan `DB::transaction` + `lockForUpdate()`.

**When:** Operasi saldo wallet dilakukan.

**Then:** Lock row wallet, update balance, buat wallet_transaction.

**Else:** Jika balance < amount saat deduct, throw RuntimeException.

**Related:**
- Use Case: UC-010, UC-018
- Data Entity: wallets, wallet_transactions

---

### BR-WAL-002: Wallet Auto-Create

| Field | Value |
|-------|-------|
| **ID** | BR-WAL-002 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | WalletService::ensureWallet |

**Rule:** Wallet auto-create dengan balance 0 jika belum ada.

**When:** Operasi wallet dilakukan untuk user tanpa wallet.

**Then:** Buat wallet baru, lalu lock dan operasi.

**Else:** —

**Related:**
- Use Case: UC-009, UC-010
- Data Entity: wallets

---

### BR-WAL-003: Wallet Transaction Uniqueness

| Field | Value |
|-------|-------|
| **ID** | BR-WAL-003 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Migration wallet_transactions |

**Rule:** Unique constraint `(reference_id, type)` mencegah double-refund per transaksi.

**When:** Membuat wallet_transaction.

**Then:** Database reject jika combination sudah ada.

**Else:** QueryException code 23000 ditangani sebagai idempotent success.

**Related:**
- Use Case: UC-009
- Data Entity: wallet_transactions

---

### BR-WAL-004: Reseller Auto-Refund

| Field | Value |
|-------|-------|
| **ID** | BR-WAL-004 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | WalletRefundService |

**Rule:** Refund otomatis ke wallet reseller saat transaksi gagal (synchronous, bukan queue).

**When:** TransactionFailed event di-dispatch dan user adalah reseller.

**Then:** Tambah balance wallet + buat wallet_transaction (type: refund).

**Else:** Jika bukan reseller, skip. Jika refund sudah ada, skip (idempotent).

**Related:**
- Use Case: — (belum dikutip chapter manapun; diimplementasi PF-005 di process-flows.md)
- Data Entity: wallets, wallet_transactions

---

### BR-VOUCH-001: Voucher Validation Chain

| Field | Value |
|-------|-------|
| **ID** | BR-VOUCH-001 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | VoucherService::validate |

**Rule:** 5 check harus pass sebelum voucher bisa digunakan.

**When:** Voucher di-apply saat checkout.

**Then:** Validasi: (1) is_active, (2) within period, (3) uses < max, (4) amount >= min, (5) one-use-per-user.

**Else:** Salah satu gagal → reject dengan error message.

**Related:**
- Use Case: UC-005
- Data Entity: vouchers, voucher_usages

---

### BR-VOUCH-002: Voucher Case-Insensitive

| Field | Value |
|-------|-------|
| **ID** | BR-VOUCH-002 |
| **Category** | Validation |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | VoucherService::voucherQuery |

**Rule:** Pencarian voucher case-insensitive (`LOWER(code)`).

**When:** User memasukkan kode voucher.

**Then:** Cari dengan `LOWER(code) = LOWER(input)`.

**Else:** —

**Related:**
- Use Case: — (belum dimiliki UC — tidak dikutip di `### Business Rules` chapter manapun)
- Data Entity: vouchers

---

### BR-VOUCH-003: Voucher Atomic Apply

| Field | Value |
|-------|-------|
| **ID** | BR-VOUCH-003 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | VoucherService::applyAndRecord |

**Rule:** Apply voucher menggunakan `lockForUpdate()` + atomic increment `used_count`.

**When:** Voucher digunakan saat checkout.

**Then:** Lock voucher row, validasi, hitung diskon, record usage, increment used_count.

**Else:** Race condition dicegah oleh row lock.

**Related:**
- Use Case: — (belum dimiliki UC — tidak dikutip di `### Business Rules` chapter manapun)
- Data Entity: vouchers, voucher_usages

---

### BR-WF-001: Transaction State Machine

| Field | Value |
|-------|-------|
| **ID** | BR-WF-001 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | TransactionStatus enum + FulfillTransactionJob |

**Rule:** Transaksi mengikuti state machine: pending → paid → processing → waiting_fulfillment → success/failed. Atau pending → expired.

**When:** Status transaksi berubah.

**Then:** Hanya transisi valid yang diizinkan.

**Else:** Transisi invalid ditolak.

**Related:**
- Use Case: UC-005
- Data Entity: transactions

---

### BR-WF-002: Transaction Idempotency

| Field | Value |
|-------|-------|
| **ID** | BR-WF-002 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | PaymentWebhookController |

**Rule:** Payment callback diproses sekali saja (idempotent). `ref_id` unique + `lockForUpdate()`.

**When:** Payment gateway callback diterima.

**Then:** Lock transaksi, cek status, skip jika sudah paid/expired.

**Else:** Double processing dicegah oleh lock + status check.

**Related:**
- Use Case: UC-008
- Data Entity: transactions

---

### BR-WF-003: Pending Expire Cron

| Field | Value |
|-------|-------|
| **ID** | BR-WF-003 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | routes/console.php |

**Rule:** Transaksi `pending` yang melewati `expired_at` di-expire setiap menit.

**When:** Cron job berjalan setiap menit.

**Then:** Find pending + expired_at < now → set expired + kirim email.

**Else:** —

**Related:**
- Use Case: — (tidak dimiliki UC manapun; dimiliki PF-008 di process-flows.md)
- Data Entity: transactions

---

### BR-WF-004: Reconciliation

| Field | Value |
|-------|-------|
| **ID** | BR-WF-004 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | PaymentWebhookController |

**Rule:** Late payment pada transaksi expired → reconcile: expired → paid → processing.

**When:** Callback diterima untuk transaksi expired.

**Then:** Set status paid, buat payment, dispatch fulfillment, kirim reconciliation email.

**Else:** —

**Related:**
- Use Case: UC-008
- Data Entity: transactions

---

### BR-FUL-001: Minecraft Manual Fulfillment

| Field | Value |
|-------|-------|
| **ID** | BR-FUL-001 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | FulfillTransactionJob + Admin\TransactionController |

**Rule:** Produk Minecraft → status `waiting_fulfillment` sampai admin eksekusi manual.

**When:** Fulfillment job menangani transaksi dengan kategori Minecraft.

**Then:** Set waiting_fulfillment, tidak hit API distributor.

**Else:** —

**Related:**
- Use Case: UC-015
- Data Entity: transactions, categories

---

### BR-FUL-002: Fulfillment Retry

| Field | Value |
|-------|-------|
| **ID** | BR-FUL-002 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | FulfillTransactionJob |

**Rule:** Fulfillment retry max 3 attempts dengan backoff 30s, 120s, 300s.

**When:** Fulfillment gagal.

**Then:** Retry dengan backoff. Setelah 3x, set Failed + dispatch TransactionFailed event.

**Else:** —

**Related:**
- Use Case: UC-014
- Data Entity: transactions

---

### BR-FUL-003: Missing Vendor Mapping

| Field | Value |
|-------|-------|
| **ID** | BR-FUL-003 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | DistributorService::topUp |

**Rule:** Produk tanpa vendor mapping → immediate Failed (tidak retryable).

**When:** Fulfillment mencoba hit distributor.

**Then:** Cek vendor_mapping. Jika tidak ada → return failed(status: 422).

**Else:** —

**Related:**
- Use Case: UC-018
- Data Entity: product_vendor_mappings

---

### BR-FUL-004: Sync Waiting Fulfillment

| Field | Value |
|-------|-------|
| **ID** | BR-FUL-004 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | SyncWaitingFulfillmentJob |

**Rule:** Cron setiap 5 menit sync status `waiting_fulfillment` (last 24h, limit 100).

**When:** Scheduler trigger.

**Then:** Hit distributor checkStatus per transaksi. Success → set success. Failed → set failed + refund. Pending → skip.

**Else:** —

**Related:**
- Use Case: — (tidak dimiliki UC manapun; dimiliki PF-009 di process-flows.md)
- Data Entity: transactions

---

### BR-PAY-001: Payment Signature

| Field | Value |
|-------|-------|
| **ID** | BR-PAY-001 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | PaymentSignatureVerifier |

**Rule:** Signature diverifikasi dengan HMAC-SHA256 + timing-safe comparison.

**When:** Payment callback diterima.

**Then:** `hash_hmac('sha256', ref_id + normalizedAmount, secret)` vs signature.

**Else:** Signature mismatch → return 401.

**Related:**
- Use Case: UC-008

---

### BR-PAY-002: Checkout Rate Limiting

_Superseded — duplikat dari BR-CHECK-001 (Checkout Rate Limiting). Id dipertahankan agar tidak dipakai ulang._

| Field | Value |
|-------|-------|
| **ID** | BR-PAY-002 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | Superseded by BR-CHECK-001 |
| **Source** | CheckoutRateLimit middleware |

**Rule:** Rate limit 3-dimensi: per IP (5/min), per user (10/hour), per product+target (3/10min).

**When:** POST /checkout dipanggil.

**Then:** Cek semua dimensi. Jika melebihi → 429 Too Many Requests.

**Else:** Quota hanya dikonsumsi setelah validasi pass (422 exempt).

**Related:**
- Use Case: — (Superseded by BR-CHECK-001; UC-005 dimiliki BR-CHECK-001)

---

### BR-PAY-003: Payment Method Toggle

| Field | Value |
|-------|-------|
| **ID** | BR-PAY-003 |
| **Category** | Validation |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | AdminPaymentMethod::toggleActive |

**Rule:** Admin/owner bisa aktif/nonaktifkan metode pembayaran. Nonaktif = tidak tampil di checkout.

**When:** Toggle payment method.

**Then:** Flip is_active. Metode nonaktif tidak disertakan di `activeMethodsOrdered()`.

**Else:** —

**Related:**
- Use Case: UC-023
- Data Entity: admin_payment_methods

---

### BR-AUTH-001: Role Middleware

| Field | Value |
|-------|-------|
| **ID** | BR-AUTH-001 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | RoleMiddleware |

**Rule:** `RoleMiddleware` menerima comma-separated roles, abort 403 jika tidak match. Block inactive users.

**When:** Route dengan middleware `role:admin,owner` diakses.

**Then:** Cek role user + is_active. Jika tidak match → 403.

**Else:** —
**Related:**
- Use Case: — (belum dimiliki UC)
- Data Entity: users

---

### BR-AUTH-002: Owner Admin Management

| Field | Value |
|-------|-------|
| **ID** | BR-AUTH-002 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | AdminService::ensureOwner |

**Rule:** Hanya owner yang bisa buat/toggle admin.

**When:** Buat atau toggle admin.

**Then:** Cek `isOwner()`. Jika bukan → 403.

**Else:** —

**Related:**
- Use Case: UC-021
- Data Entity: users

---

### BR-AUTH-003: Financial Data Access Control

| Field | Value |
|-------|-------|
| **ID** | BR-AUTH-003 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Transaction::scopeForAdmin |

**Rule:** Admin tidak bisa lihat amount/omset. Owner bisa. Implementasi di query level (column exclusion).

**When:** Admin query transaksi.

**Then:** `scopeForAdmin()` → exclude kolom `amount` dari select.

**Else:** Bukan cuma UI hide — data benar-benar tidak di-select.

**Related:**
- Use Case: UC-011, UC-012
- Data Entity: transactions

---

### BR-AUTH-004: Self-Protection

| Field | Value |
|-------|-------|
| **ID** | BR-AUTH-004 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | AdminService::toggleAdmin |

**Rule:** Admin/owner tidak bisa toggle status diri sendiri.

**When:** Toggle admin status.

**Then:** Cek `target.id !== actor.id`. Jika sama → throw DomainException.

**Else:** —

**Related:**
- Use Case: UC-021
- Data Entity: users

---

### BR-DEL-001: Product Deletion — Has Transactions

| Field | Value |
|-------|-------|
| **ID** | BR-DEL-001 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | ProductDeletionService::delete |

**Rule:** Produk yang pernah dibeli → selalu soft delete (tidak pernah hard delete).

**When:** Hapus produk yang punya transaksi.

**Then:** Soft delete + snapshot + ProductDeletionLog.

**Else:** —

**Related:**
- Use Case: UC-013
- Data Entity: products, product_deletion_logs

---

### BR-DEL-002: Product Deletion — Timing & Role

| Field | Value |
|-------|-------|
| **ID** | BR-DEL-002 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | ProductDeletionService::delete |

**Rule:** Hard delete hanya untuk: ≤24h + owner + tanpa transaksi.

**When:** Hapus produk tanpa transaksi.

**Then:**
- ≤24h + owner → hard delete + snapshot
- >24h atau admin → soft delete + snapshot

**Else:** —

**Related:**
- Use Case: UC-013

---

### BR-DEL-003: Product Deletion Audit

| Field | Value |
|-------|-------|
| **ID** | BR-DEL-003 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | ProductDeletionService::createDeletionLog |

**Rule:** Setiap produk dihapus → buat `ProductDeletionLog` dengan JSON snapshot.

**When:** Produk dihapus (soft atau hard).

**Then:** Simpan snapshot (product + category + vendor_mappings), deleted_by, reason.

**Else:** —

**Related:**
- Use Case: UC-013
- Data Entity: product_deletion_logs

---

### BR-MC-001: Minecraft Credential Encryption

| Field | Value |
|-------|-------|
| **ID** | BR-MC-001 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Transaction model `$casts` |

**Rule:** Password Minecraft di-encrypt sebelum disimpan (`encrypted` cast).

**When:** Admin menyimpan credentials manual fulfill.

**Then:** Password di-encrypt otomatis oleh Eloquent cast.

**Else:** —
**Related:**
- Use Case: UC-015
- Data Entity: transactions

---

### BR-REF-001: Reseller vs Customer Refund

| Field | Value |
|-------|-------|
| **ID** | BR-REF-001 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | RefundRequestController + WalletRefundService |

**Rule:** Customer → manual refund request. Reseller → auto-refund ke wallet.

**When:** Transaksi gagal.

**Then:**
- Reseller → otomatis refund ke wallet (synchronous)
- Customer → submit form refund manual ke admin

**Else:** —

**Related:**
- Use Case: UC-007
- Data Entity: transactions, wallet_transactions

---

### BR-REF-002: Reseller Auto-Refund Tanpa Manual Request

| Field | Value |
|-------|-------|
| **ID** | BR-REF-002 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | WalletRefundService + RefundResellerListener + RefundRequestController |

**Rule:** Reseller otomatis di-refund ke wallet tanpa perlu mengajukan refund request manual.

**When:** `TransactionFailed` event di-dispatch dan pemilik transaksi adalah reseller.

**Then:**
- `RefundResellerListener` (sync, tanpa queue) → `WalletRefundService::refundIfReseller`
- Tambah balance wallet + buat `wallet_transaction` (type: refund), kirim `ResellerAutoRefundedMail`
- `RefundRequestController::store` menolak form refund manual untuk transaksi reseller (422)

**Else:** Jika bukan reseller → customer submit form refund manual. Jika refund sudah ada → skip (idempotent, lock + unique constraint 23000).

**Examples:**
- Transaksi reseller gagal → saldo wallet bertambah otomatis, tanpa form
- Transaksi customer gagal → form refund manual ke admin

**Related:**
- Use Case: UC-007
- Data Entity: transactions, wallets, wallet_transactions

---

### BR-REF-003: Refund Selesai Wajib Sudah Ada Request

| Field | Value |
|-------|-------|
| **ID** | BR-REF-003 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Admin\TransactionController::completeRefund |

**Rule:** Refund hanya bisa ditandai selesai jika transaksi sudah punya refund request (`refund_requested = true`).

**When:** Admin menekan "Mark Refund Completed" pada transaksi.

**Then:** Validasi berurutan: status = failed → `refund_requested = true` → belum `refund_completed_at` → bukan transaksi reseller. Semua pass → set `refund_completed_at` + `refund_completed_by`, kirim `RefundCompletedMail`.

**Else:** Tanpa refund request → 422 "Transaksi belum ada request refund." Sudah diproses → 422. Bukan status failed → 422.

**Examples:**
- Transaksi failed tanpa request → 422, tidak bisa di-complete
- Transaksi failed sudah request → `refund_completed_at` terisi + email notifikasi

**Related:**
- Use Case: UC-016
- Data Entity: transactions

---

### BR-GAME-001: Game ID Verification Chain

| Field | Value |
|-------|-------|
| **ID** | BR-GAME-001 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | GameIdVerifierChain |

**Rule:** Game ID diverifikasi via chain: Digiflazz API → local regex fallback.

**When:** User memasukkan game ID.

**Then:** Coba Digiflazz → jika unsupported/error → fallback regex. Hasil di-cache 5 menit.

**Else:** —

**Related:**
- Use Case: UC-004
- Data Entity: transactions

---

### BR-GAME-002: Local Game ID Validation

| Field | Value |
|-------|-------|
| **ID** | BR-GAME-002 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | FallbackGameIdVerifier |

**Rule:** Regex validation: Java `^[a-zA-Z0-9_]{3,16}$`, Bedrock `^[a-zA-Z0-9_ ]{3,12}$`, Non-MC: digits 4-20.

**When:** Fallback verifier digunakan.

**Then:** Validasi berdasarkan edition. Server ID: alphanumeric + `-` + `_`.

**Else:** Return invalid.

**Related:**
- Use Case: UC-004
- Data Entity: transactions

---

### BR-ANN-001: Announcement Audience

| Field | Value |
|-------|-------|
| **ID** | BR-ANN-001 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | SendAnnouncementJob |

**Rule:** Announcement dikirim berdasarkan target_audience. Admin/owner tidak dapat announcement.

**When:** Announcement dipublish.

**Then:**
- all → customer + reseller
- customer → customer only
- reseller → reseller only
- admin → admin + owner (notif only, no email)

**Else:** —

**Related:**
- Use Case: UC-022
- Data Entity: announcements

---

### BR-ANN-002: Email Notification Rules

| Field | Value |
|-------|-------|
| **ID** | BR-ANN-002 |
| **Category** | Validation |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | SendAnnouncementJob + mail classes |

**Rule:** Transaction emails = always sent. Announcement emails = respect `email_notification_enabled`.

**When:** Email dikirim.

**Then:**
- Transaction (paid/success/failed/expired/reconciliation) → selalu kirim
- Announcement → kirim hanya jika `email_notification_enabled = true`

**Else:** Guest dengan `guest_email = null` → tidak ada email.

**Related:**
- Use Case: UC-022
- Data Entity: users, transactions

---

### BR-CHECK-001: Checkout Rate Limiting

| Field | Value |
|-------|-------|
| **ID** | BR-CHECK-001 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | CheckoutRateLimit middleware |

**Rule:** Rate limit 3-dimensi: per IP (5/min), per user (10/hour), per product+target (3/10min).

**When:** POST /checkout dipanggil.

**Then:** Cek semua dimensi. Jika melebihi → 429 Too Many Requests.

**Else:** Quota hanya dikonsumsi setelah validasi pass (422 exempt).

**Related:**
- Use Case: UC-005
- Data Entity: transactions
