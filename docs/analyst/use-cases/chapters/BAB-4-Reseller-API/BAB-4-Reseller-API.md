# BAB 4: Reseller API (H2H Integration)

**Fokus:** REST API untuk reseller H2H — list produk, top-up, cek status, cek saldo. Semua via Sanctum token.

**Boundary:** Authenticated (Sanctum token), role: reseller only.

**Process Flow Related:** PF-007 (Reseller API Top-up), PF-009 (Sync Waiting Fulfillment (Scheduled)).

---

## Use Cases

| UC | Nama | Actors | Trigger | Status |
|----|------|--------|---------|--------|
| UC-017 | List Produk via API | Reseller | GET products | [x] Confirmed |
| UC-018 | Top-up via API | Reseller | POST topup | [x] Confirmed |
| UC-019 | Cek Status via API | Reseller | GET status | [x] Confirmed |
| UC-020 | Cek Saldo via API | Reseller | GET balance | [x] Confirmed |

---

## UC-017: List Produk via API

Reseller mengambil daftar produk + harga via REST API.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Reseller | API consumer | Mendapatkan daftar produk + harga |

### Preconditions

- Valid Sanctum token, role: reseller

### Postconditions

- Daftar produk dengan harga reseller dikembalikan

### Main Flow

1. **Reseller** kirim `GET /api/v1/products`
2. **System** auth: Sanctum token + role:reseller
3. **System** ambil produk aktif + vendor mappings
4. **System** hitung harga: `cost_price + (cost_price × markup_percentage / 100)`
5. **System** return JSON

### Business Rules

- **BR-PRICE-001:** Harga reseller = `cost_price + (cost_price × markup / 100)`

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| products | Read | Active + vendor mappings |
| reseller_profiles | Read | markup_percentage |

### Notes

- Implementasi: `Api\V1\ProductController::index`

---

## UC-018: Top-up via API

Reseller membuat transaksi top-up via API. Wallet di-deduct atomik.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Reseller | API consumer | Membeli produk untuk customer |
| System | Backend | Memproses dan fulfill |

### Preconditions

- Valid Sanctum token, role: reseller
- Wallet balance mencukupi

### Postconditions

- Transaksi dibuat (status: paid), fulfillment dimulai
- ref_id dikembalikan

### Main Flow

1. **Reseller** kirim `POST /api/v1/topup`
2. **System** validasi: product, game ID, wallet balance
3. **System** deduct wallet (atomic: lockForUpdate)
4. **System** buat Transaction + Payment (status: paid)
5. **System** dispatch `FulfillTransactionJob`
6. **System** return 201 dengan `ref_id`

### Alternative Flows

#### AF-1: Insufficient Balance
- **Trigger:** Wallet balance < price
- **Step 3a:** Return 422 insufficient balance

#### AF-2: Missing Vendor Mapping
- **Trigger:** Produk tanpa vendor mapping
- **Step 5a:** Transaksi langsung Failed

### Exception Flows

#### EF-1: Product Not Found
- **Trigger:** product_id tidak valid
- **Step 2a:** Return 422

#### EF-2: Game ID Invalid
- **Trigger:** Game ID gagal validasi
- **Step 2a:** Return 422

### Business Rules

- **BR-WAL-001:** Deduct menggunakan lockForUpdate
- **BR-FUL-003:** Missing vendor mapping → immediate fail

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| wallets | Read/Update | Deduct balance |
| transactions | Create | status: paid |
| payments | Create | status: paid |
| FulfillTransactionJob | Dispatch | Async |

### Notes

- Implementasi: `Api\V1\TopupController::store`

---

## UC-019: Cek Status via API

Reseller mengecek status transaksi via API.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Reseller | API consumer | Mengetahui status transaksi |

### Preconditions

- Valid Sanctum token, role: reseller
- Transaksi milik reseller

### Postconditions

- Status transaksi dikembalikan

### Main Flow

1. **Reseller** kirim `GET /api/v1/topup/{ref_id}/status`
2. **System** cari transaksi (scoped ke user)
3. **System** return JSON `{ref_id, status}`

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Read | By ref_id + user scope |

### Notes

- Implementasi: `Api\V1\TopupController::status`

---

## UC-020: Cek Saldo via API

Reseller mengecek saldo wallet via API.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Reseller | API consumer | Mengetahui saldo |

### Preconditions

- Valid Sanctum token, role: reseller

### Postconditions

- Saldo wallet dikembalikan

### Main Flow

1. **Reseller** kirim `GET /api/v1/balance`
2. **System** ambil/buat wallet
3. **System** return JSON `{balance}`

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| wallets | Read | By user_id |

### Notes

- Implementasi: `Api\V1\BalanceController::show`
