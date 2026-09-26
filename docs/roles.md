# Role System — Pembagian Role & Wewenang

> ADR: `ADR-0019 (role-hierarchy)`
> Status: 4 role aktif (reseller dinonaktifkan sementara)

---

## 1. Ringkasan Role

| Role | Keterangan | Login | Harga |
|------|-----------|-------|-------|
| **Guest** | Pengunjung tanpa akun | Tidak | `products.price` |
| **Customer** | User yang sudah daftar/login | Session | `products.price` |
| **Admin** | Staff/operator sistem | Session | `cost_price` (BONUS) |
| **Owner** | Pemilik sistem, full control | Session | `cost_price` (BONUS) |

---

## 2. Guest (Tanpa Login)

### Yang Bisa Dilakukan
- Browse kategori game
- Lihat daftar produk per kategori
- Lihat detail produk + harga
- Checkout (guest checkout, `user_id` nullable)
- Lihat invoice via `ref_id`
- Submit refund request (isi form rekening)
- Search produk (`GET /search`)
- Akses toko reseller (`/toko/{slug}`)

### Yang Tidak Bisa Dilakukan
- Login ke admin panel
- Lihat riwayat transaksi (tidak ada `user_id`)
- Pakai voucher (butuh login)
- Buka live chat (butuh login)
- Akses dashboard

### Cara Kerja
- Tidak punya `user_id` di tabel `users`
- `transactions.user_id` = `null` (guest checkout)
- Invoice diakses via `ref_id` (public token, 8 char)

---

## 3. Customer (Login, role `customer`)

### Yang Bisa Dilakukan
- Semua yang guest bisa
- Dashboard user (`/dashboard`)
- Profile (`/profile`) — update nama, email, password
- Riwayat transaksi (`/history`)
- Re-check status transaksi (hit distributor API)
- Buka live chat (`/chat`)
- Pakai voucher di checkout
- Lihat in-app notification

### Yang Tidak Bisa Dilakukan
- Akses admin panel (`/admin/*`) → 403
- CRUD produk/kategori
- Kelola transaksi user lain
- Lihat data user lain

### Pendaftaran
```bash
# Via browser: GET /register
# Atau via Tinker:
php artisan tinker
```
```php
App\Models\User::create([
    'name' => 'Budi',
    'email' => 'budi@email.com',
    'password' => Hash::make('password123'),
    'role' => 'customer',
]);
```

---

## 4. Admin (Login, role `admin`)

### Yang Bisa Dilakukan
- Semua yang customer bisa
- Admin dashboard (`/admin/dashboard`) — monitoring status transaksi (count-based, tanpa data finansial)
- CRUD kategori (`/admin/categories`)
- CRUD produk (`/admin/products`) — termasuk lihat price & cost_price
- Lihat subcategories (`/admin/subcategories`)
- Lihat daftar transaksi (`/admin/transactions`) — tanpa kolom amount
- Lihat grafik transaksi (harian/mingguan/bulanan) — count-based
- Lihat detail transaksi — termasuk log distributor, tanpa amount
- Retry transaksi gagal
- Complete refund
- Mark manual success
- Lihat refund requests (`/admin/refunds`) — tanpa kolom amount
- Lihat daftar users (`/admin/users`)
- Manage reseller tokens
- Lihat wallets reseller — tanpa kolom saldo
- Deposit wallet reseller
- CRUD vouchers (`/admin/vouchers`) — tanpa kolom value
- Aktif/nonaktifkan metode pembayaran
- Lihat audit logs

### Yang Tidak Bisa Dilakukan
- Lihat nominal/harga per transaksi (`amount`) — **query-level exclusion** via `scopeForAdmin()`
- Lihat total omset/akumulasi uang
- Export/cetak laporan PDF (owner-only)
- Hapus produk kapan saja (hanya soft delete >24h)
- Hard delete produk (hanya owner)
- Kelola role admin lain
- Akses owner-only routes (`/owner/*`)

### Layout & Navigation
- **Layout**: `<x-admin-layout>` — sidebar-based, NO top navbar
- **Sidebar**: `components/admin-sidebar.blade.php` — desktop fixed left (240px), mobile bottom sheet (Alpine.js)
- **Menu items**: Dashboard, Transaksi, Users, Wallets, Vouchers, Kategori, Produk, Subcategories, Pembayaran, Pengumuman, Audit Log
- **User footer**: Nama + role badge (Admin) + link ke Profile, Theme Toggle, Logout
- **Role-based routing**: Login/register/verify/confirm-password otomatis redirect admin/owner ke `/admin/dashboard` via `RedirectAdminToDashboard` middleware
- **Dashboard route**: `/dashboard` dengan `redirect.admin` middleware — admin/owner auto-redirect ke `/admin/dashboard`
- **BERANDA button**: Link ke storefront dari admin dashboard (`components/beranda-button.blade.php`)

### Harga Khusus
Admin diperlakukan seperti Reseller + BONUS:
```
Harga Admin = cost_price (langsung, tanpa markup)
```
Admin tidak perlu punya `reseller_profiles`. Perubahan markup reseller tidak mempengaruhi harga admin.

### CLI Commands
```bash
# Buat admin user
php artisan app:create-admin
php artisan app:create-admin admin@custom.com --name="Admin Baru" --password=rahasia

# Seed admin
php artisan db:seed --class=AdminSeeder
```

---

## 5. Owner (Login, role `owner`)

### Yang Bisa Dilakukan
- Semua yang admin bisa (termasuk semua data finansial)
- Lihat nominal/harga per transaksi (`amount`)
- Lihat total omset/akumulasi uang di dashboard
- Export/cetak laporan PDF dari halaman transaksi
- Lihat grafik transaksi (harian/mingguan/bulanan)
- Hard delete produk ≤24 jam tanpa transaksi
- Kelola role admin lain (`/owner/admins`)
- Laporan keuangan (`/owner/reports/*`)
- Export CSV/Excel/PDF
- Pengaturan global (`/owner/settings`)
- Fraud alerts

### Yang Tidak Bisa Dilakukan (Keterbatasan)
- Hard delete produk yang sudah ada transaksi → tidak bisa (locked)
- Hard delete produk >24 jam → tidak bisa (hanya soft delete)

### Harga Khusus
Sama seperti admin:
```
Harga Owner = cost_price (langsung)
```

### CLI Commands
```bash
# Buat owner user
php artisan app:create-owner
php artisan app:create-owner owner@custom.com --name="Boss" --password=rahasia

# Seed owner
php artisan db:seed --class=OwnerSeeder
```

---

## 6. Pricing Logic

```php
// app/Services/CalculatePriceForUser.php

Guest / Customer  →  products.price          // harga normal
Admin / Owner     →  products.cost_price     // harga beli (BONUS)
```

Admin/owner tidak query `reseller_profiles`. Markup reseller tidak mempengaruhi harga admin/owner.

### Contoh

```
Produk: 86 Diamonds ML
  price:     Rp 25.000
  cost_price: Rp 20.000

Guest/Customer bayar: Rp 25.000
Admin/Owner bayar:    Rp 20.000 (hemat Rp 5.000)
```

---

## 7. Product Deletion Rules (Owner-only Hard Delete)

| Kondisi | Admin | Owner |
|---------|-------|-------|
| Produk ada transaksi | Tidak bisa hapus | Tidak bisa hapus (locked) |
| Produk ≤24h, tanpa transaksi | Soft delete | **Hard delete** |
| Produk >24h, tanpa transaksi | Soft delete | Soft delete |

Hard delete = data benar-benar hilang dari database. Soft delete = `deleted_at` terisi, data masih bisa di-recover.

---

## 8. Route Access Matrix

### Public (tanpa auth)

| Route | Guest | Customer | Admin | Owner |
|-------|-------|----------|-------|-------|
| `GET /` | ✅ | ✅ | ✅ | ✅ |
| `GET /products/{category}` | ✅ | ✅ | ✅ | ✅ |
| `GET /products/{product}` | ✅ | ✅ | ✅ | ✅ |
| `POST /checkout` | ✅ | ✅ | ✅ | ✅ |
| `GET /invoice/{ref_id}` | ✅ | ✅ | ✅ | ✅ |
| `GET /search` | ✅ | ✅ | ✅ | ✅ |
| `GET /toko/{slug}` | ✅ | ✅ | ✅ | ✅ |

### Auth (login required)

| Route | Guest | Customer | Admin | Owner |
|-------|-------|----------|-------|-------|
| `GET /dashboard` | ❌ | ✅ | ✅ | ✅ |
| `GET /profile` | ❌ | ✅ | ✅ | ✅ |
| `GET /history` | ❌ | ✅ | ✅ | ✅ |
| `GET /chat` | ❌ | ✅ | ✅ | ✅ |
| `POST /invoice/{id}/refund-request` | ❌ | ✅ | ✅ | ✅ |

### Admin Panel (`role:admin,owner`)

| Route | Guest | Customer | Admin | Owner | Catatan |
|-------|-------|----------|-------|-------|---------|
| `GET /admin/dashboard` | ❌ | ❌ | ✅ | ✅ | Admin: count-only; Owner: +omset |
| `GET /admin/categories` | ❌ | ❌ | ✅ | ✅ | |
| `POST /admin/categories` | ❌ | ❌ | ✅ | ✅ | |
| `GET /admin/products` | ❌ | ❌ | ✅ | ✅ | Termasuk price & cost_price |
| `POST /admin/products` | ❌ | ❌ | ✅ | ✅ | |
| `DELETE /admin/products/{id}` | ❌ | ❌ | ✅ (soft) | ✅ (soft) | |
| `GET /admin/transactions` | ❌ | ❌ | ✅ | ✅ | Admin: tanpa amount; Owner: +amount |
| `POST /admin/transactions/export-pdf` | ❌ | ❌ | ❌ | ✅ | **Owner-only** — 403 untuk admin |
| `POST /admin/transactions/{id}/retry` | ❌ | ❌ | ✅ | ✅ | |
| `GET /admin/refunds` | ❌ | ❌ | ✅ | ✅ | Admin: tanpa amount; Owner: +amount |
| `GET /admin/users` | ❌ | ❌ | ✅ | ✅ | |
| `GET /admin/wallets` | ❌ | ❌ | ✅ | ✅ | Admin: tanpa saldo; Owner: +saldo |
| `POST /admin/wallets/{id}/deposit` | ❌ | ❌ | ✅ | ✅ | |
| `GET /admin/vouchers` | ❌ | ❌ | ✅ | ✅ | Admin: tanpa value; Owner: +value |
| `POST /admin/vouchers` | ❌ | ❌ | ✅ | ✅ | |
| `GET /admin/audit-logs` | ❌ | ❌ | ✅ | ✅ | |

### Owner-only (`role:owner`)

| Route | Guest | Customer | Admin | Owner |
|-------|-------|----------|-------|-------|
| `DELETE /owner/products/{id}` (hard) | ❌ | ❌ | ❌ | ✅ |
| `GET /owner/admins` | ❌ | ❌ | ❌ | ✅ |
| `PUT /owner/admins/{id}/role` | ❌ | ❌ | ❌ | ✅ |
| `GET /owner/reports/*` | ❌ | ❌ | ❌ | ✅ |
| `POST /owner/reports/export` | ❌ | ❌ | ❌ | ✅ |
| `GET /owner/settings` | ❌ | ❌ | ❌ | ✅ |

---

## 9. Middleware

```php
// Guest: tanpa middleware
Route::get('/', ...);

// Customer: auth required
Route::middleware(['auth'])->group(function () { ... });

// Admin + Owner: auth + role + throttle
Route::middleware(['auth', 'role:admin,owner', 'throttle:admin'])
    ->prefix('admin')->group(function () { ... });

// Owner-only: auth + role:owner
Route::middleware(['auth', 'role:owner'])
    ->prefix('owner')->group(function () { ... });
```

`RoleMiddleware` support comma-separated: `role:admin,owner` = admin ATAU owner.

---

## 10. FAQ

**Q: Kenapa admin/owner dapat harga cost_price?**
A: BONUS. Admin dianggap "internal" jadi tidak perlu markup. Harga admin tidak terpengaruh oleh markup reseller.

**Q: Apakah admin bisa jadi owner?**
A: Bisa, tapi harus ubah role via Tinker atau owner yang ubah:
```php
php artisan tinker
$user = User::where('email', 'admin@email.com')->first();
$user->role = UserRole::Owner;
$user->save();
```

**Q: Apakah guest bisa daftar jadi customer?**
A: Ya, via `/register`. Setelah daftar, role default = `customer`.

**Q: Apakah ada role lain selain 4 ini?**
A: Reseller ada di enum tapi dinonaktifkan sementara. Bisa diaktifkan lagi nanti.

**Q: Bagaimana cara cek role user?**
A: Via Tinker:
```php
$user = User::first();
$user->role;          // → UserRole::Admin
$user->role->value;   // → "admin"
$user->isAdmin();     // → true
$user->isOwner();     // → false
```

---

## 11. File Terkait

- `app/Enums/UserRole.php` — 4 enum values
- `app/Models/User.php:53-72` — `isReseller()`, `isAdmin()`, `isOwner()`
- `app/Http/Middleware/RoleMiddleware.php` — authorization middleware
- `app/Services/CalculatePriceForUser.php` — pricing per role
- `app/Services/ProductDeletionService.php` — owner-only hard delete
- `routes/web.php:41` — admin/owner routes
- `routes/api.php:8` — reseller API (disabled)
- `docs/owner-role.md` — CLI commands owner
- `ADR-0007 (owner-separate-role)` — ADR owner role
- `ADR-0019 (role-hierarchy)` — ADR role hierarchy
