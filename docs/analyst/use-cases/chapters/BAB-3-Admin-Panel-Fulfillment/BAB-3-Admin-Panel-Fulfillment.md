# BAB 3: Admin Panel & Fulfillment

**Fokus:** Operasional admin harian — monitoring, CRUD produk, retry transaksi, fulfill Minecraft, kelola voucher & refund.

**Boundary:** Authenticated (admin/owner), Role-based access.

**Process Flow Related:** PF-003 (Transaction Fulfillment), PF-004 (Minecraft Manual Fulfillment), PF-011 (Product Deletion).

---

## Use Cases

| UC | Nama | Actors | Trigger | Status |
|----|------|--------|---------|--------|
| UC-012 | Dashboard Admin & Monitoring | Admin / Owner | Akses dashboard | [x] Confirmed |
| UC-013 | CRUD Produk, Kategori & Subkategori | Admin | Kelola produk | [x] Confirmed |
| UC-014 | Retry Transaksi Gagal | Admin | Klik retry | [x] Confirmed |
| UC-015 | Manual Fulfill Minecraft | Admin | Proses MC manual | [x] Confirmed |
| UC-016 | Kelola Refund | Admin | Proses refund | [x] Confirmed |

---

## UC-012: Dashboard Admin & Monitoring

Admin melihat dashboard monitoring. Owner melihat tambahan omset.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | Operator sistem | Memantau status transaksi |
| Owner | Pemilik | Memantau + melihat omset |

### Preconditions

- Authenticated, role: admin atau owner

### Postconditions

- Dashboard ditampilkan dengan statistik

### Main Flow

1. **Admin** akses `GET /admin/dashboard`
2. **System** hitung transaksi per status (count-based)
3. **System** tampilkan 8 stat cards
4. **Owner** melihat omset tambahan

### Alternative Flows

#### AF-1: Owner Omset
- **Trigger:** User role owner
- **Step 3a:** Tambah card omset (hanya owner yang lihat)

### Business Rules

- **BR-AUTH-003:** Admin tidak bisa lihat omset (scopeForAdmin — column exclusion di query level)

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Count | By status |

### Notes

- Implementasi: `Admin\DashboardController::__invoke`

---

## UC-013: CRUD Produk, Kategori & Subkategori

Admin mengelola produk, kategori, dan subkategori.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | Operator | Kelola produk dan kategori |

### Preconditions

- Authenticated, role: admin atau owner

### Postconditions

- Produk/kategori/subkategori di-create, update, atau delete

### Main Flow — Create Produk

1. **Admin** isi form produk (nama, harga, kategori, subkategori, gambar)
2. **System** validasi input
3. **System** simpan produk (`is_active = true`)
4. **System** upload gambar jika ada

### Main Flow — Delete Produk

1. **Admin** klik hapus produk
2. **System** cek apakah produk punya transaksi
3. **System** proses sesuai BR-DEL-001, BR-DEL-002

### Business Rules

- **BR-DEL-001:** Produk yang pernah dibeli → selalu soft delete
- **BR-DEL-002:** Hard delete hanya ≤24h + owner + tanpa transaksi
- **BR-DEL-003:** Setiap delete → buat ProductDeletionLog dengan JSON snapshot

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| products | Create/Update/Delete | CRUD |
| categories | Create/Update/Delete | CRUD |
| subcategories | Create/Update/Delete | CRUD |
| product_vendor_mappings | Read | Vendor mapping |

### Notes

- Implementasi: Livewire product form
- Gambar di-upload via Livewire

---

## UC-014: Retry Transaksi Gagal

Admin me-retry transaksi yang gagal.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | Operator | Me-retry transaksi gagal |

### Preconditions

- Transaksi status = `failed`

### Postconditions

- Transaksi di-retry ke distributor

### Main Flow

1. **Admin** klik retry di detail transaksi
2. **System** validasi: status = failed
3. **System** set status: processing
4. **System** dispatch `FulfillTransactionJob`

### Business Rules

- **BR-FUL-002:** Retry max 3 attempts dengan backoff 30s, 120s, 300s

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Update | status: processing |
| FulfillTransactionJob | Dispatch | Async |

### Notes

- Implementasi: `Admin\TransactionController::retry`

---

## UC-015: Manual Fulfill Minecraft

Admin memproses transaksi Minecraft secara manual (kirim akun ke buyer).

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | Operator | Mengirim akun Minecraft ke buyer |

### Preconditions

- Transaksi status = `waiting_fulfillment`
- Produk di kategori `minecraft`

### Postconditions

- Transaksi selesai (status: success)
- Akun dikirim via email

### Main Flow

1. **Admin** isi form: username, password, delivery note
2. **System** validasi: status + kategori
3. **System** simpan credentials (password di-encrypt via `encrypted` cast)
4. **System** set status: success + completed_at
5. **System** kirim `MinecraftAccountDeliveredMail` (queued)
6. **System** buat audit log

### Business Rules

- **BR-FUL-001:** Produk Minecraft → status `waiting_fulfillment` sampai admin eksekusi
- **BR-MC-001:** Password Minecraft di-encrypt sebelum disimpan

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Update | status, credentials |
| AuditLogService | Log | manual_success |

### Notes

- Implementasi: `Admin\TransactionController::markManualSuccess`
- Minecraft credential reveal via `MinecraftCredentialReveal` Livewire component

---

## UC-016: Kelola Refund

Admin memproses dan menandai refund selesai.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | Operator | Menyelesaikan proses refund |

### Preconditions

- Transaksi status = `failed`
- `refund_requested = true`

### Postconditions

- Refund ditandai selesai

### Main Flow

1. **Admin** lihat daftar refund (filter: `refund_requested = true, refund_completed_at IS NULL`)
2. **Admin** klik "Mark Refund Completed"
3. **System** validasi: failed + refund_requested
4. **System** set: `refund_completed_at`, `refund_completed_by`
5. **System** kirim `RefundCompletedMail`

### Business Rules

- **BR-REF-003:** Refund hanya bisa diselesaikan jika sudah requested

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Read/Update | refund fields |

### Notes

- Implementasi: `Admin\TransactionController::completeRefund`
