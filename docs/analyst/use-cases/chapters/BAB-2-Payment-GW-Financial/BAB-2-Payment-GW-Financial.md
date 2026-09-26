# BAB 2: Payment Gateway & Financial Operations

**Fokus:** Pengolahan uang masuk, mutasi saldo, alur webhook, dan pencatatan keuangan.

**Boundary:** High Security, Atomic Transaction, Idempotency Locked.

**Process Flow Related:** PF-002 (Payment Callback), PF-005 (Reseller Auto-Refund), PF-008 (Expire Pending Transactions (Scheduled)).

---

## Use Cases

| UC | Nama | Actors | Trigger | Status |
|----|------|--------|---------|--------|
| UC-008 | Payment Gateway Callback | Payment Gateway | Webhook POST | [x] Confirmed |
| UC-009 | Cek Saldo & Mutasi Wallet | Reseller | Request saldo | [x] Confirmed |
| UC-010 | Kelola Deposit Wallet Reseller | Admin | Deposit saldo | [x] Confirmed |
| UC-011 | Laporan Keuangan, Omset & Export | Owner | Akses laporan | [x] Confirmed |

---

## UC-008: Payment Gateway Callback

Payment gateway (Tripay/Duitku) mengirim notifikasi pembayaran via webhook.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Payment Gateway | Sistem eksternal | Memberitahu pembayaran diterima |

### Preconditions

- Transaksi pending ada di sistem

### Postconditions

- Pembayaran terverifikasi, fulfillment dimulai

### Main Flow

1. **Gateway** kirim POST `/webhook/payment-callback`
2. **System** verifikasi IP whitelist (jika dikonfigurasi)
3. **System** verifikasi signature (HMAC-SHA256)
4. **System** cari transaksi by `ref_id` (lockForUpdate)
5. **System** cek idempotency (skip jika sudah paid/expired)
6. **System** buat Payment record
7. **System** dispatch `FulfillTransactionJob`
8. **System** return 200 OK

### Alternative Flows

#### AF-1: Late Payment (Reconciliation)
- **Trigger:** Transaksi expired tapi pembayaran diterima
- **Step 5a:** Set status: expired → paid → processing
- **Step 7a:** Dispatch FulfillTransactionJob
- **Step 8a:** Kirim `TransactionReconciliationMail`

### Exception Flows

#### EF-1: Signature Invalid
- **Trigger:** Signature tidak match
- **Step 3a:** Return 401

#### EF-2: Transaksi Tidak Ditemukan
- **Trigger:** ref_id tidak ada
- **Step 4a:** Return 404

#### EF-3: Race Condition
- **Trigger:** Dua callback bersamaan
- **Step 4a:** lockForUpdate menunggu, lalu idempotency check

### Business Rules

- **BR-PAY-001:** Signature diverifikasi sebelum diproses
- **BR-WF-002:** Idempotency: tidak proses dua kali
- **BR-WF-004:** Reconciliation: expired → paid → processing

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Read/Update | By ref_id, lockForUpdate |
| payments | Create | method, gateway_ref, status |
| FulfillTransactionJob | Dispatch | Async queue |

### Notes

- Implementasi: `PaymentWebhookController::handle`
- CSRF exempted untuk endpoint ini
- Mendukung Tripay dan Duitku (dual-vendor, configurable)

---

## UC-009: Cek Saldo & Mutasi Wallet

Reseller mengecek saldo wallet dan melihat riwayat mutasi.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Reseller | API consumer / Web user | Mengetahui saldo dan riwayat transaksi wallet |

### Preconditions

- Valid auth (Sanctum token atau session)

### Postconditions

- Saldo wallet dan mutasi ditampilkan

### Main Flow (API)

1. **Reseller** kirim `GET /api/v1/balance`
2. **System** ambil/buat wallet
3. **System** return JSON `{balance}`

### Main Flow (Web)

1. **Reseller** akses halaman wallet
2. **System** ambil wallet + wallet_transactions (paginated)
3. **System** tampilkan saldo dan daftar mutasi

### Business Rules

- **BR-WAL-002:** Wallet auto-create dengan balance 0 jika belum ada
- **BR-WAL-003:** Unique constraint `(reference_id, type)` mencegah double-refund

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| wallets | Read | By user_id |
| wallet_transactions | Read | By wallet_id, order by created_at desc |

### Notes

- API: `Api\V1\BalanceController::show`
- Web: Livewire wallet component

---

## UC-010: Kelola Deposit Wallet Reseller

Admin menambah saldo wallet reseller.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | Operator | Deposit saldo ke reseller |

### Preconditions

- Target user adalah reseller
- Amount > 0

### Postconditions

- Saldo wallet bertambah

### Main Flow

1. **Admin** isi form deposit (amount + notes)
2. **System** validasi: amount > 0, ≤ max config
3. **System** `WalletService::deposit` (atomic: lockForUpdate)
4. **System** return new balance

### Business Rules

- **BR-WAL-001:** Deposit menggunakan lockForUpdate
- **BR-WAL-002:** Wallet auto-create jika belum ada

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| wallets | Read/Update | Add balance |
| wallet_transactions | Create | type: deposit |

### Notes

- Implementasi: `Admin\WalletController::deposit`

---

## UC-011: Laporan Keuangan, Omset & Export PDF/Pajak

> **Scope per spec v1.3 (scope-cut) — baca sebelum pakai UC ini:**
> Dari tiga alur di UC ini, hanya **Export PDF** (FR-024 / FR-026) yang masih hidup.
> **Laporan Keuangan** (FR-025) dan **Export Pajak** (FR-027) ber-status **Dropped** —
> keduanya dipertahankan di bawah sebagai catatan riwayat dan ditandai ~~DROPPED~~.
> Ringkasan omset tetap tersedia lewat FR-024. Judul sengaja tidak diubah agar rujukan
> `UC-011` di `requirements-matrix.md` tetap valid.

Owner melihat laporan keuangan (omset, cost, margin) dan mengekspor sebagai PDF atau data pajak.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Owner | Pemilik | Mendapatkan laporan keuangan |

### Preconditions

- Role: owner

### Postconditions

- Laporan ditampilkan / file diunduh

### Main Flow — Laporan ~~(DROPPED — FR-025)~~

1. **Owner** akses halaman laporan
2. **System** ambil data: omset, cost, margin per periode
3. **System** tampilkan chart + tabel

### Main Flow — Export PDF

1. **Owner** klik export PDF
2. **System** terapkan filter aktif (period, status, date range)
3. **System** `PdfExportService::generate` → DomPDF
4. **System** return PDF download (A4 landscape)

### Main Flow — Export Pajak ~~(DROPPED — FR-027)~~

1. **Owner** klik export pajak
2. **System** generate data pajak (CSV/Excel + PDF)

### Alternative Flows

#### AF-1: Admin Coba Akses
- **Trigger:** Role admin
- **Step 1a:** Return 403 Forbidden

### Business Rules

- **BR-AUTH-003:** Admin tidak bisa lihat amount/omset (column exclusion di query level)

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Read | With filters, omset calculation |
| PdfExportService | Execute | Generate PDF |

### Notes

- Implementasi: `Admin\TransactionController::exportPdf`, laporan page
- Admin lihat data transaksi tapi kolom amount tidak di-select (bukan cuma UI hide)
