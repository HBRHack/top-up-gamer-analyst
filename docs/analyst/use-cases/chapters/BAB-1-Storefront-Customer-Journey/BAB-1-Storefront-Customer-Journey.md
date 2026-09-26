# BAB 1: Storefront & Customer Journey

**Fokus:** Pengalaman pembeli umum (guest/customer) dari buka web sampai dapat nota/invoice.

**Boundary:** Public Access, Unauthenticated / Light Auth.

**Process Flow Related:** PF-001 (Customer Checkout), PF-006 (Voucher Apply).

---

## Use Cases

| UC | Nama | Actors | Trigger | Status |
|----|------|--------|---------|--------|
| UC-001 | Browse Kategori & Produk | Guest / Auth User | Akses homepage | [x] Confirmed |
| UC-002 | Detail Produk + Harga Per Role | Guest / Auth User | Klik produk | [x] Confirmed |
| UC-003 | Search Produk | Guest / Auth User | Input search query | [x] Confirmed |
| UC-004 | Verifikasi ID / Nickname Game | Guest / Auth User | Input game ID | [x] Confirmed |
| UC-005 | Checkout & Penerapan Voucher | Guest / Auth User | Submit checkout | [x] Confirmed |
| UC-006 | View Invoice Status | Guest / Auth User | Akses invoice | [x] Confirmed |
| UC-007 | Riwayat Transaksi & Form Refund | Customer | Akses riwayat | [x] Confirmed |

---

## UC-001: Browse Kategori & Produk

Menampilkan halaman utama dengan daftar kategori game dan produk dalam kategori.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Guest / Auth User | Pengunjung storefront | Melihat kategori game yang tersedia |

### Preconditions

- Tidak ada (halaman publik)

### Postconditions

- Kategori game ditampilkan dengan icon dan nama
- Produk dalam kategori ditampilkan dalam kartu (paginated)

### Main Flow

1. **User** mengakses halaman utama (`GET /`)
2. **System** mengambil semua kategori dengan `is_active = true`
3. **System** mengurutkan berdasarkan `display_order`
4. **System** menampilkan grid kategori dengan icon
5. **User** melihat kategori (termasuk "Segera Hadir" badge jika `status = coming_soon`)
6. **User** mengklik kategori (`GET /products/{category:slug}`)
7. **System** memuat kategori + produk aktif
8. **System** menampilkan breadcrumb, header kategori, grid produk

### Alternative Flows

#### AF-1: Kategori Coming Soon
- **Trigger:** Kategori memiliki `status = coming_soon`
- **Step 4a:** System menampilkan badge "Segera Hadir"
- **Step 4b:** Tombol beli di-disable

#### AF-2: Produk Coming Soon
- **Trigger:** Produk dalam kategori `status = coming_soon`
- **Step 8a:** Produk ditampilkan tapi `cursor: not-allowed`

### Exception Flows

#### EF-1: Tidak Ada Kategori Aktif
- **Trigger:** Semua kategori `is_active = false`
- **Step 2a:** System menampilkan halaman kosong

#### EF-2: Kategori Tidak Ditemukan
- **Trigger:** Slug tidak valid
- **Step 6a:** System return 404

### Business Rules

- **BR-PROD-001:** Kategori `coming_soon` tampil tapi beli disabled
- **BR-PROD-002:** Hanya produk `is_active = true` yang ditampilkan

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| categories | Read | Filter is_active=true, order by display_order |
| products | Read | Filter category_id + is_active |

### UI Requirements

- Grid layout: 4→3→2→1 responsive
- Floating particles + mascot (Minecraft theme)
- Hero section dengan headline

### Notes

- Implementasi: `StorefrontController::index`, `StorefrontController::category`
- File: `app/Http/Controllers/StorefrontController.php`

---

## UC-002: Detail Produk + Harga Per Role

Menampilkan detail produk, harga berdasarkan role user, dan form checkout.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Guest / Auth User | Calon pembeli | Melihat detail produk dan melakukan checkout |

### Preconditions

- Produk exists, `is_active = true`, bukan coming_soon

### Postconditions

- Detail produk + form checkout ditampilkan

### Main Flow

1. **User** mengklik produk (`GET /products/{product}`)
2. **System** memuat produk + kategori + subcategory
3. **System** menghitung harga berdasarkan role (`CalculatePriceForUser`)
4. **System** menampilkan: gambar, nama, harga, deskripsi, form checkout
5. **User** melihat harga (customer: `price`, reseller: `cost_price + markup`, admin: `cost_price`)

### Alternative Flows

#### AF-1: Reseller Pricing
- **Trigger:** User login + punya `reseller_profiles`
- **Step 3a:** Harga = `cost_price + (cost_price × markup_percentage / 100)`

#### AF-2: Admin Pricing
- **Trigger:** User role admin/owner
- **Step 3a:** Harga = `cost_price` langsung

### Exception Flows

#### EF-1: Produk Tidak Aktif
- **Trigger:** Produk `is_active = false` atau coming_soon
- **Step 1a:** System return 404

### Business Rules

- **BR-PRICE-001:** Harga dihitung berdasarkan role user

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| products | Read | By ID + relations |
| CalculatePriceForUser | Execute | Role-based pricing |

### UI Requirements

- 2-column grid (1.6fr detail + 0.9fr checkout)
- Mobile: stacked
- GameIdCheck Livewire component
- Payment method selector
- Voucher code input

### Notes

- Implementasi: `StorefrontController::show`

---

## UC-003: Search Produk

Mencari produk berdasarkan nama atau kategori.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Guest / Auth User | Pengunjung | Mencari produk spesifik |

### Preconditions

- Tidak ada

### Postconditions

- Hasil pencarian ditampilkan

### Main Flow

1. **User** memasukkan query di search bar (`GET /search?q={query}`)
2. **System** melakukan full-text search pada `products.name` + `categories.name`
3. **System** menampilkan hasil dalam grid (paginated)
4. **User** melihat jumlah hasil dan kartu produk

### Alternative Flows

#### AF-1: Query Kosong
- **Trigger:** `q` parameter kosong
- **Step 2a:** System menampilkan semua produk

### Business Rules

- Tidak ada rules spesifik

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| products | Search | Full-text on name |
| categories | Join | Full-text on name |

### Notes

- Implementasi: `SearchController::index`

---

## UC-004: Verifikasi ID / Nickname Game

Memverifikasi ID game sebelum checkout menggunakan GameIdVerifierChain.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Guest / Auth User | Calon pembeli | Memastikan ID game valid |

### Preconditions

- Produk dipilih

### Postconditions

- ID game divalidasi (valid/invalid + nickname)

### Main Flow

1. **User** memasukkan game ID di form checkout
2. **System** mengirim AJAX ke `GET /cek-nickname`
3. **System** menjalankan `GameIdVerifierChain::check()`
4. **System** mencoba Digiflazz API → fallback ke local regex
5. **System** return `GameIdResult` (valid + nickname, atau invalid + message)

### Alternative Flows

#### AF-1: Digiflazz Unsupported
- **Trigger:** Digiflazz return `GameNotSupportedException`
- **Step 4a:** Fallback ke `FallbackGameIdVerifier`
- **Step 4b:** Validasi regex (Java: `^[a-zA-Z0-9_]{3,16}$`, Bedrock: `^[a-zA-Z0-9_ ]{3,12}$`)

#### AF-2: Digiflazz Down
- **Trigger:** HTTP error / connection error
- **Step 4a:** Fallback ke local regex

### Exception Flows

#### EF-1: ID Tidak Valid
- **Trigger:** Regex tidak match
- **Step 5a:** Return `valid: false, message: "Format ID tidak valid"`

### Business Rules

- **BR-GAME-001:** Game ID divalidasi via chain: Digiflazz → local regex
- **BR-GAME-002:** Hasil di-cache 5 menit

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| GameIdVerifierChain | Execute | Chain of responsibility |
| Cache | Read/Write | 5-minute TTL |

### Notes

- Implementasi: `GameIdCheckController::__invoke` + `GameIdVerifierChain`

---

## UC-005: Checkout & Penerapan Voucher Diskon

Membuat transaksi baru dari form checkout dengan validasi voucher.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Guest / Auth User | Pembeli | Membeli produk game |
| System | Backend | Memproses checkout |

### Preconditions

- Produk aktif, game ID valid, payment method aktif

### Postconditions

- Transaksi baru dibuat (status: pending)
- Invoice page ditampilkan

### Main Flow

1. **User** submit form checkout (`POST /checkout`)
2. **System** validasi: product, game ID, payment method, voucher
3. **System** generate `ref_id` (FOFA-YYYYMMDD-XXXXXXXX)
4. **System** buat Transaction (status: pending) + Payment
5. **System** apply voucher jika ada (`VoucherService::applyAndRecord`)
6. **System** return halaman invoice atau JSON

### Alternative Flows

#### AF-1: Guest Checkout
- **Trigger:** User tidak login
- **Step 2a:** `guest_email` wajib diisi
- **Step 4a:** `user_id` = null

#### AF-2: Voucher Applied
- **Trigger:** User memasukkan kode voucher
- **Step 5a:** Hitung diskon, record usage, increment used_count

### Exception Flows

#### EF-1: Rate Limit Exceeded
- **Trigger:** IP/user/product+target melebihi limit
- **Step 1a:** Return 429 Too Many Requests

#### EF-2: Validation Error
- **Trigger:** Input tidak valid
- **Step 2a:** Return 422 dengan error messages

#### EF-3: Voucher Invalid
- **Trigger:** Voucher expired/used/inactive
- **Step 5a:** Return 422

### Business Rules

- **BR-WF-001:** Transaksi dimulai dengan status `pending`
- **BR-CHECK-001:** Rate limit: 5/min per IP, 10/hour per user, 3/10min per product+target
- **BR-VOUCH-001:** Voucher divalidasi sebelum apply

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Create | status: pending |
| payments | Create | status: pending |
| voucher_usages | Create | Jika voucher applied |
| RefIdGenerator | Execute | Generate unique ref_id |

### UI Requirements

- Redirect ke `/invoice/{ref_id}` (web) atau JSON 201 (API)

### Notes

- Implementasi: `CheckoutController::store`
- Middleware: `checkout.ratelimit`

---

## UC-006: View Invoice Status

Menampilkan halaman invoice dengan status pembayaran.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Guest / Auth User | Pembeli | Melihat status dan detail transaksi |

### Preconditions

- Transaksi exists dengan ref_id

### Postconditions

- Invoice ditampilkan dengan status, detail, dan aksi

### Main Flow

1. **User** mengakses `/invoice/{ref_id}`
2. **System** memuat transaksi + payment + voucher usage
3. **System** menampilkan: ref_id, status badge, produk, jumlah, metode bayar
4. **User** melihat countdown timer (jika pending + expired_at)

### Alternative Flows

#### AF-1: Pending + Countdown
- **Trigger:** Status = pending + expired_at set
- **Step 4a:** Tampilkan countdown timer JavaScript

#### AF-2: Minecraft Credentials Delivered
- **Trigger:** `delivered_minecraft_username` terisi
- **Step 3a:** Tampilkan box credentials (via `MinecraftCredentialReveal`)

#### AF-3: Failed + Refund Form
- **Trigger:** Status = failed
- **Step 3a:** Tampilkan form refund manual (customer) atau info auto-refund (reseller)

### Exception Flows

#### EF-1: Transaksi Expired
- **Trigger:** Status = expired
- **Step 3a:** Tampilkan "Buat transaksi baru" CTA

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Read | By ref_id |
| payments | Read | By transaction_id |
| voucher_usages | Read | By transaction_id |

### UI Requirements

- Status badge (colored)
- Countdown timer (pending)
- Payment details (QRIS QR / VA number / e-wallet deeplink)
- Refund form (if failed)

### Notes

- Implementasi: `InvoiceController::show`

---

## UC-007: Riwayat Transaksi & Form Pengajuan Refund

Customer melihat riwayat transaksi dan mengajukan refund untuk transaksi gagal.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Customer | Pemilik transaksi | Melihat riwayat + mengajukan refund |

### Preconditions

- User login, role: customer

### Postconditions

- Riwayat transaksi ditampilkan
- Data refund request tersimpan di transaksi

### Main Flow

1. **Customer** akses halaman riwayat
2. **System** tampilkan daftar transaksi user
3. **Customer** klik transaksi gagal
4. **Customer** isi form refund (bank/e-wallet, no rekening, nama)
5. **System** validasi: status = failed, user is owner
6. **System** simpan: `refund_bank`, `refund_account_number`, `refund_account_name`, `refund_requested = true`

### Alternative Flows

#### AF-1: Reseller Auto-Refund
- **Trigger:** User adalah reseller
- **Step 6a:** Tidak perlu manual request — refund otomatis ke wallet

### Exception Flows

#### EF-1: Bukan Pemilik Transaksi
- **Trigger:** User lain mencoba akses
- **Step 5a:** Return 403 Forbidden

#### EF-2: Status Bukan Failed
- **Trigger:** Transaksi belum gagal
- **Step 5a:** Return 422

### Business Rules

- **BR-REF-001:** Hanya pemilik transaksi yang bisa submit refund
- **BR-REF-002:** Reseller auto-refund tanpa manual request

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| transactions | Read/Update | Read riwayat, Update refund fields |

### Notes

- Implementasi: `RefundRequestController::store`
