# Audit Log — Dokumentasi & Cara Penggunaan

> Issue #15 — `audit_logs` table + `AuditLogService` + Observers
> Status: `done` • Tidak ada UI viewer, akses via Tinker/DB query (sesuai spec v1.3 simplified)

---

## 1. Apa itu Audit Log?

Mencatat semua aksi CRUD oleh **admin/owner** ke tabel terpusat `audit_logs`:

| Kolom | Tipe | Keterangan |
|-------|------|------------|
| `id` | bigint | PK |
| `user_id` | FK → users (nullable) | Siapa aktor. `null` = system / tanpa auth |
| `action` | string | `created` / `updated` / `deleted` / custom |
| `auditable_type` | string | FQCN model, mis. `App\Models\Product` |
| `auditable_id` | bigint | ID model |
| `old_values` | json (nullable) | Snapshot sebelum berubah |
| `new_values` | json (nullable) | Snapshot sesudah berubah |
| `ip_address` | string(45) | `request()->ip()` |
| `user_agent` | string | `request()->userAgent()` |
| `created_at` / `updated_at` | timestamp | Kapan log dibuat |

Index: `(auditable_type, auditable_id)` untuk query cepat per model.

**Model yang di-observe otomatis:**
- `Product` — `created`, `updated`, `deleted` selalu di-log
- `Category` — sama
- `User` — hanya `updated` saat `role` berubah, dan `created` saat `role != customer` (admin/owner via seeder/command)
- `Voucher` — `created`, `updated`, `deleted` (via `VoucherObserver`, issue #18/#30)
- `Announcement` — `created`, `updated`, `deleted` (via `AnnouncementObserver`, issue #30)

---

## 2. Cara Kerja Otomatis (Observer)

Observer sudah diregistrasi di `app/Providers/AppServiceProvider.php:36-42`:

```php
Product::observe(ProductObserver::class);
Category::observe(CategoryObserver::class);
User::observe(UserObserver::class);
Voucher::observe(VoucherObserver::class);
Announcement::observe(AnnouncementObserver::class);
// + Event::listen(TransactionFailed::class, RefundResellerListener::class) fallback jika auto-discovery mati (issue #29)
```

Setiap `Product::create()` / `$product->save()` / `$product->delete()` yang dilakukan **dalam konteks HTTP request yang sudah `Auth::user()` = admin/owner** akan otomatis bikin 1 row di `audit_logs`.

**Contoh:** admin bikin kategori via Livewire
```php
// Tidak perlu panggil service manual — observer yang handle
$category = Category::create([
    'name' => 'Mobile Legends',
    'slug' => 'mobile-legends',
    'display_order' => 1,
]);
 // → otomatis 1 row: action=created, auditable_type=Category, new_values={...}
```

Jika **tidak ada user login** (mis. factory di tinker, seeder, queue), `user_id` akan `null` — tetap ke-log tapi bisa di-filter.

---

## 3. Penggunaan Manual via `AuditLogService`

Untuk kasus custom (bukan observer), inject atau resolve service:

```php
use App\Services\AuditLogService;
use Illuminate\Database\Eloquent\Model;

class ExampleController extends Controller
{
    public function __construct(protected AuditLogService $audit) {}

    public function manualLog(Model $model)
    {
        $this->audit->log(
            user: auth()->user(),          // null = system
            action: 'custom_action',       // bebas string
            model: $model,                 // model Eloquent apapun
            oldValues: ['status' => 'pending'],
            newValues: ['status' => 'success'],
        );
        // ip & user_agent otomatis dari request()
    }
}
```

### Tanpa DI (di Tinker / Job)

```php
app(\App\Services\AuditLogService::class)->log(
    user: \App\Models\User::where('email','admin@fofashop.test')->first(),
    action: 'updated',
    model: $product,
    oldValues: ['price' => '50000'],
    newValues: ['price' => '55000'],
);
```

Signature lengkap (`app/Services/AuditLogService.php:20`):

```php
public function log(?User $user, string $action, Model $model, ?array $oldValues = null, ?array $newValues = null): AuditLog
```

---

## 4. Cara Query / Dokumentasi Penggunaan

### Via Tinker (paling sering)

```bash
php artisan tinker
```

```php
use App\Models\AuditLog;
use App\Models\Product;

// 1. Lihat semua log terbaru
AuditLog::latest()->limit(10)->get()->toArray();

// 2. Filter per model
AuditLog::where('auditable_type', Product::class)
    ->where('auditable_id', 5)
    ->latest()->get();

// 3. Filter per aktor
AuditLog::where('user_id', 1)->latest()->get();

// 4. Filter per action
AuditLog::where('action', 'updated')->latest()->get();

// 5. Lihat perubahan role user
AuditLog::where('auditable_type', \App\Models\User::class)
    ->where('action', 'updated')
    ->latest()->get();

// 6. Pakai morph relation
$log = AuditLog::latest()->first();
$log->auditable; // → Product / Category / User instance
$log->user;      // → User aktor
$log->old_values; // array
$log->new_values; // array

// 7. Rentang waktu
AuditLog::whereBetween('created_at', [now()->subDays(7), now()])->get();
```

### Via SQL Langsung

```sql
-- Semua update produk minggu ini
SELECT * FROM audit_logs
WHERE auditable_type = 'App\\Models\\Product'
  AND action = 'updated'
  AND created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
ORDER BY created_at DESC;

-- Audit trail 1 produk spesifik
SELECT user_id, action, old_values, new_values, ip_address, created_at
FROM audit_logs
WHERE auditable_type = 'App\\Models\\Product' AND auditable_id = 12
ORDER BY id DESC;

-- Siapa yang ganti role?
SELECT * FROM audit_logs
WHERE auditable_type = 'App\\Models\\User'
  AND action = 'updated'
ORDER BY created_at DESC;
```

### Via Eloquent (di kode)

```php
// Ambil audit trail untuk 1 produk
$product = Product::find(1);
$logs = AuditLog::where('auditable_type', Product::class)
    ->where('auditable_id', $product->id)
    ->orderByDesc('id')
    ->get();

// Atau tambah relation di model jika mau (opsional):
// di Product.php: public function auditLogs() { return $this->morphMany(AuditLog::class, 'auditable'); }
// lalu: $product->auditLogs()->latest()->get();
```

---

## 5. Contoh Output

```php
AuditLog::latest()->first()->toArray();
/*
[
  "id" => 42,
  "user_id" => 3, // admin@fofashop.test
  "action" => "updated",
  "auditable_type" => "App\\Models\\Product",
  "auditable_id" => 7,
  "old_values" => ["name" => "86 Diamonds", "price" => "25000.00", ...],
  "new_values" => ["name" => "100 Diamonds", "price" => "30000.00", ...],
  "ip_address" => "192.168.1.10",
  "user_agent" => "Mozilla/5.0 ...",
  "created_at" => "2026-08-28T12:06:00.000000Z",
]
*/
```

Untuk `User` role change:
```php
[
  "action" => "updated",
  "auditable_type" => "App\\Models\\User",
  "old_values" => ["role" => "customer"],
  "new_values" => ["role" => "admin"],
]
```

---

## 6. Testing

Test ada di `tests/Feature/AuditLogTest.php` (25 test).

Jalankan:
```bash
php artisan test tests/Feature/AuditLogTest.php
php artisan test --filter AuditLog
```

Cakupan:
- `created` / `updated` / `deleted` Product & Category
- `created` / `updated` / `deleted` Voucher & Announcement (issue #30)
- `User` role change only (non-role update tidak log)
- `User` customer create tidak log, admin/owner create log
- Service direct call & null user
- Morph relation
- Livewire integration (Product + Voucher)

---

## 7. FAQ

**Q: Kenapa `user_id` null di beberapa log?**
A: Dibuat tanpa `Auth::user()` — mis. seeder, factory di test tanpa `actingAs()`, atau Tinker tanpa login. Di production, admin/owner selalu login jadi terisi.

**Q: Apakah customer/reseller checkout trigger audit?**
A: Tidak. Hanya `Product`, `Category`, `User(role)` yang di-observe. `Transaction` punya log sendiri (`distributor_logs`, `payments`), bukan `audit_logs`.

**Q: Mau tambah model baru ke audit?**
A: Buat observer mirip `ProductObserver` pakai `LogsAudit` trait, lalu daftarkan di `AppServiceProvider::boot()`:
```php
Voucher::observe(VoucherObserver::class);
Announcement::observe(AnnouncementObserver::class);
```
Sudah ada: `VoucherObserver` (`app/Observers/VoucherObserver.php:1`) & `AnnouncementObserver` (`app/Observers/AnnouncementObserver.php:1`) terdaftar (issue #30).

**Q: Data audit bisa dihapus?**
A: Tidak ada UI delete. Hapus manual jika perlu:
```php
AuditLog::where('created_at', '<', now()->subYear())->delete();
```
Tapi sebaiknya jangan — untuk akuntabilitas.

---

## 8. File Terkait

- Migration: `database/migrations/2026_08_28_000002_create_audit_logs_table.php:1`
- Model: `app/Models/AuditLog.php:1`
- Service: `app/Services/AuditLogService.php:1`
- Observers: `app/Observers/ProductObserver.php:1`, `CategoryObserver.php:1`, `UserObserver.php:1`, `VoucherObserver.php:1`, `AnnouncementObserver.php:1`
- Provider: `app/Providers/AppServiceProvider.php:36`
- Factory: `database/factories/AuditLogFactory.php:1`
- ADR: `ADR-0011 (admin-audit-log):1`
- CONTEXT: `CONTEXT.md:139` (Audit Log glossary)
