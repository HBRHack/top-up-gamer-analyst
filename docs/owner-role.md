# Owner Role — Dokumentasi & CLI Commands

> Issue #14 — Add Owner to UserRole Enum
> Status: `done`

---

## 1. Apa itu Owner Role?

Role terpisah di enum `UserRole` (bukan admin dengan flag). Owner punya semua hak admin plus hak istimewa: hapus produk kapan saja, manage voucher/promo, kelola role admin lain, export pajak, fraud alerts.

| Role | Keterangan | Contoh User |
|------|-----------|-------------|
| `customer` | User biasa, beli produk via storefront | Pelanggan |
| `reseller` | H2H API, harga khusus | Youtuber/streamer |
| `admin` | Mengelola sistem | Staff operator |
| `owner` | Pemilik sistem, full control | Owner Fofa Shop |

---

## 2. File Terkait

| File | Keterangan |
|------|-----------|
| `app/Enums/UserRole.php:10` | Enum value `Owner = 'owner'` |
| `app/Http/Middleware/RoleMiddleware.php:17` | Middleware `role:owner` (variadic, comma-split) |
| `app/Services/CreatesUserWithRole.php:15` | Service idempotent buat user + set role |
| `app/Console/Commands/CreateOwnerUser.php:11` | Artisan command `app:create-owner` |
| `database/seeders/OwnerSeeder.php:9` | Seeder default owner |
| `database/seeders/DatabaseSeeder.php:25` | Panggil `OwnerSeeder` |
| `routes/web.php:41` | Admin routes pakai `role:admin,owner` |
| `ADR-0007 (owner-separate-role):1` | ADR keputusan owner role |

---

## 3. CLI Commands

### 3.1 Buat Owner User via Artisan

```bash
# Default (email: owner@fofashop.test, password: password)
php artisan app:create-owner

# Custom email
php artisan app:create-owner owner@custom.com

# Custom semua
php artisan app:create-owner owner@custom.com --name="Boss" --password=rahasia123
```

**Output:**
```
Owner user created: owner@fofashop.test
```

Kalau sudah ada:
```
Owner user already exists: owner@fofashop.test (ensured role=owner)
```

**Idempotent** — aman dijalankan berulang kali. Kalau user sudah ada, cuma pastikan role-nya `owner`.

Contoh pembuatan owner khusus (kredensial disamarkan untuk publikasi):

```bash
php artisan app:create-owner owner@example.com --name="Boss" --password=<REDACTED>
```

### 3.2 Buat Admin User via Artisan (Perbandingan)

```bash
# Default
php artisan app:create-admin

# Custom
php artisan app:create-admin admin@custom.com --name="Admin Utama" --password=rahasia
```

### 3.3 Seed Database (Termasuk Owner)

```bash
# Full seed: admin + owner + kategori + produk
php artisan db:seed

# Seed spesifik
php artisan db:seed --class=OwnerSeeder
php artisan db:seed --class=AdminSeeder
```

### 3.4 Fresh Migration + Seed (Development)

```bash
# Reset semua data, migrate ulang, seed fresh
php artisan migrate:fresh --seed
```

**Output seed:**
```
Database seeding completed successfully.
```

Default accounts yang dibuat:
- `test@example.com` — customer (default)
- `admin@fofashop.test` — admin, password: `password`
- `owner@fofashop.test` — owner, password: `password`

### 3.5 Cek Role User via Tinker

```bash
php artisan tinker
```

```php
// Cari user
$user = App\Models\User::where('email', 'owner@fofashop.test')->first();
$user->role;          // → App\Enums\UserRole::Owner
$user->role->value;   // → "owner"

// Cek semua user + role
App\Models\User::select('email', 'role')->get()->toArray();
// → [
//   ['email' => 'test@example.com', 'role' => 'customer'],
//   ['email' => 'admin@fofashop.test', 'role' => 'admin'],
//   ['email' => 'owner@fofashop.test', 'role' => 'owner'],
// ]

// Ubah role manual
$user->role = App\Enums\UserRole::Owner;
$user->save();
```

### 3.6 Test Middleware Role

```bash
# Jalankan test role middleware
php artisan test --filter RoleMiddleware
```

---

## 4. Route Access Control

### Owner bisa akses semua admin routes:

```php
// routes/web.php:41
Route::middleware(['auth', 'role:admin,owner', 'throttle:admin'])
    ->prefix('admin')->name('admin.')->group(function () {
        // ... semua admin routes
    });
```

### Owner-only routes (nanti di Phase 2+):

```php
// Contoh route yang akan ditambah:
Route::middleware(['auth', 'role:owner'])
    ->prefix('owner')->name('owner.')->group(function () {
        // /owner/vouchers — CRUD voucher
        // /owner/reports/* — laporan keuangan
        // /owner/admins — kelola admin
    });
```

### Cek access di browser:

| URL | Customer | Reseller | Admin | Owner |
|-----|----------|----------|-------|-------|
| `/` | ✅ | ✅ | ✅ | ✅ |
| `/admin/dashboard` | ❌ 403 | ❌ 403 | ✅ | ✅ |
| `/owner/vouchers` (nanti) | ❌ 403 | ❌ 403 | ❌ 403 | ✅ |

---

## 5. Cara Kerja Internals

### Enum Cast

```php
// app/Enums/UserRole.php
enum UserRole: string
{
    case Customer = 'customer';
    case Reseller = 'reseller';
    case Admin = 'admin';
    case Owner = 'owner';  // ← baru ditambah
}
```

Database column `role` tetap `string` (bukan DB enum), tapi di-cast ke `UserRole` enum via Eloquent.

### Middleware `role:owner`

```php
// Support comma-separated
Route::middleware('role:admin,owner');  // admin ATAU owner boleh akses

// Support single role
Route::middleware('role:owner');  // hanya owner
```

### Service `CreatesUserWithRole`

Idempotent — create user jika belum ada, update role jika berbeda:

```php
app(CreatesUserWithRole::class)->handle(
    email: 'owner@fofashop.test',
    name: 'Owner',
    password: 'password', // KREDENSIAL CONTOH untuk seeder — ganti sebelum dipakai di luar lokal
    role: UserRole::Owner,
);
```

---

## 6. FAQ

**Q: Kenapa owner role terpisah, bukan admin dengan flag `is_owner`?**
A: Lebih clean di query (`where role = 'owner'`) dan authorization (`role:owner`). Flag `is_owner` menambah kompleksitas logic di tiap controller.

**Q: Apakah owner bisa login ke admin panel?**
A: Ya. Route admin pakai `role:admin,owner`, jadi owner bisa akses semua admin routes.

**Q: Bagaimana kalau mau tambah owner baru?**
A: Jalankan `php artisan app:create-owner email@baru.com --name="Nama" --password=rahasia`

**Q: Apakah ada route khusus owner yang admin tidak bisa akses?**
A: Belum di MVP ini. Owner-only routes (vouchers, reports, settings) akan ditambah di Phase 2+. Sekarang owner = admin + extras di masa depan.

**Q: Bagaimana cara ubah user existing jadi owner?**
A: Via Tinker:
```bash
php artisan tinker
```
```php
$user = App\Models\User::where('email', 'target@email.com')->first();
$user->role = App\Enums\UserRole::Owner;
$user->save();
```

---

## 7. Testing

```bash
# Test role middleware
php artisan test --filter RoleMiddleware

# Test semua
php artisan test
```

Cakupan:
- `RoleMiddleware` support comma-separated roles
- `RoleMiddleware` support BackedEnum
- Default role `customer` saat user baru dibuat
- Owner bisa akses admin routes
- Customer/reseller tidak bisa akses admin routes
