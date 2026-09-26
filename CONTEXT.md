# Fofa Shop — Toko Top-Up Skala Komunitas Minecraft

Website top-up game (voucher/diamond/UC) dengan model bisnis reseller H2H — beli produk digital dari distributor via API, jual ke end user dengan markup. Minecraft sebagai kategori pertama yang aktif, game lain "Coming Soon".

## Infrastructure

**VPS Spec:** 2 VCPU + 2 GB RAM + 2 GB SWAP

**Stack:**
- Nginx (reverse proxy + static cache + gzip)
- PHP-FPM 8.3 (25 workers, OPcache JIT)
- MySQL 8 (256MB innodb buffer pool)
- Redis (cache + queue + session)
- Supervisor (queue worker + scheduler)

**Redis Usage:**
- `CACHE_STORE=redis` — rate limiter, query cache, view cache
- `SESSION_DRIVER=redis` — session storage (faster than DB)
- `QUEUE_CONNECTION=redis` — FulfillTransactionJob, SyncProductPricesJob, SendResellerCallbackJob, mail queue
- Mail queued via `Mail::to()->queue()` (non-blocking, async delivery)

**Target Throughput:** ~60 req/s (40-45 clean, 60 with light swap)

**Deployment configs:** file konfigurasi php-fpm, nginx, mysql, dan supervisor — tidak disertakan di repository ini (panduan langkah demi langkah di `docs/deployment.md`)

**Documentation:**
- `docs/deployment.md` — Full VPS deployment guide (step-by-step)
- `docs/redis-setup.md` — Redis install, config, monitoring, troubleshooting
- `ADR-0014 (redis-queue-cache-session)` — ADR for Redis decision
- Dokumen optimasi deploy (quick deploy commands + memory budget) — tidak disertakan di repository ini

## Design System — Hallmark & Theme (Implemented)

**Hallmark Design System Overhaul — Catalogue macrostructure, Minecraft white-red blocky dual + green creature + grid BG:**
- **Macrostructure Catalogue** — F6 grid 4→3→2→1 (`catalogue-grid`), density 60:30:15, minmax(0,1fr), responsive clamp.
- **Nav N7 Brutal slab** — sticky 2px ink border, slab pill, Brutal slab typography, `nav-slab__link` + `nav-slab__cta` (accent 58%↔62% warm).
- **Footer Ft8 Marquee** — scroll dual, @keyframes marquee 32s linear infinite, reduced-motion `animation: none`.
- **Brand Lockup** — `components/brand-lockup.blade.php` pill MINECRAFT + tagline "Berbagai Kebutuhan Minecraft", `components/application-logo.blade.php` slab rect rx 8 FS monogram (shopping-bag handle DNA), voxel highlight 0.14, 2 tiny pixels bottom-right Minecraft DNA.
- **Tokens** — `tokens.css` OKLCH dual custom-dual: light paper 98.2% 0.008 35 / accent 58% 0.22 24, dark paper 16% 0.02 35 / accent 62% 0.24 24 warm. Semantic `--color-paper` / `--color-ink` (17.2:1 light / 14.8:1 dark PASS 4.5), muted 5.26:1 / 7.1:1, badge text WCAG 4.5:1 dual, shadow `0.07`↔`0.45`.
- **Typography 2+1** — Chakra Petch (display) + Plus Jakarta Sans (body) + VT323 outlier (mono), `--font-display` / `--font-body` / `--font-mono`, tracking `-0.02em`, line-height tight 1.02 / body 1.55.
- **Radii subtle blocky** — `--radius-card 8px` (Minecraft A, not pill), not full voxel per legibility.
- **Pixel-cube CSS Tier-A** — `pixel-cube` w-7 h-7 box-shadow 2px 2px 0 ink, subtle blocky, dark shadow adaptif, Tier-A enrichment per log Hallmark internal (tidak dipublikasikan).
- **Badge Tokens** — 6 state (success/danger/warning/info/processing/neutral), `--color-*-soft` + `--color-*-text` dual, gate 41 accent-ink 4.54:1 light / 7.3:1 dark PASS, `transaction-status-badge` component.
- **Card Animations** — `fofa-card` hover lift `-1px` + shadow `0 4px 16px`, press `translateY(0)` / `scale(0.97)`, `card-enter` stagger 300ms ease-out + 40ms per index, modal/dropdown enter ease-out dur 300/200, toast 800ms, prefers-reduced-motion nuclear + targeted guard, theme-toggle crossfade opacity 150ms.
- **Tailwind CSS v4** — CSS-first config via `@import "tailwindcss"` + `@custom-variant dark (&:where(.dark, .dark *))` di `resources/css/app.css`. Tidak ada `tailwind.config.js` — theme config di CSS. Dark mode via `.dark` class toggle di `<html>`. Vite plugin `@tailwindcss/vite` (bukan PostCSS). Unlayered CSS menang atas layered utilities (important untuk custom CSS override).
- **Layout System** — 3 layout types: (1) `layouts/app.blade.php` — Livewire default (customer), padding-top 64px untuk navbar; (2) `layouts/admin.blade.php` — sidebar-based (admin/owner), NO top navbar, desktop sidebar fixed 240px + mobile bottom sheet; (3) `components/layouts/storefront.blade.php` — storefront hero, sticky header with search.
- **Admin Sidebar** — `components/admin-sidebar.blade.php`: desktop fixed left (240px), mobile bottom sheet (Alpine.js), user footer (name + role badge), dark mode support via CSS custom properties.
- **Minecraft Animations (v1.3.5):**
  - **Mascot creature** — Abstract block creature (9 CSS blocks: head green, eyes, mouth, body, 2 arms accent, 2 feet dark green) di hero section sebagai dekorasi di atas headline. `resources/css/animations.css` class `.mc-mascot` + `.mc-mascot__block--*`.
  - **Idle bounce** — `@keyframes mascot-bounce` 2.4s ease-in-out infinite, translateY 0↔-6px. `will-change: transform`. Reduced-motion: `animation: none`.
  - **Scroll fling** — `resources/js/mascot-scroll.js`. Scroll event dengan `requestAnimationFrame` throttle (`ticking` flag), NO unthrottled scroll listener. Mascot ter-'lempar' ke atas (translateY -delta*0.8) + slight rotate (delta*0.15°, max 6°). 300ms debounce return-to-idle via CSS transition `transform 0.3s var(--ease-out)`. Passive scroll listener. Livewire `livewire:navigate` cleanup. Reduced-motion: module bails entirely.
  - **Floating particles** — 6 block squares di hero section only (`.mc-particles` container `position: absolute; inset: 0; pointer-events: none`). 4 green (`var(--mc-green)`) + 2 accent (`var(--color-accent)`) dengan opacity rendah (0.2-0.35). Stagger via `animation-delay: calc(var(--i) * N)`. `@keyframes float-particle` translateY -12px + rotate 3deg. Reduced-motion: `animation: none; opacity: 0.3`.
  - **Blocky button** — `.blocky-btn` class: border 2px + `box-shadow: 3px 3px 0 var(--color-ink)`. `:active` → shadow shrinks to `1px 1px 0` + `translateY(1px)` — NOT `scale()`. Gate 10 pass: transition specifies exact properties (`box-shadow, transform, background-color`). Gate 11 pass: no `hover:scale-105`.
  - **Green tokens** — `--mc-green: oklch(62% 0.17 145)`, `--mc-green-dark: oklch(55% 0.15 145)`, `--mc-green-soft`, `--mc-green-soft-dark`. **SCOPE: mascot + particles ONLY — NOT as widespread secondary accent.** Dark mode overrides in tokens.css.
  - **Grid pattern BG** — `body` background-image: repeating 48px grid lines via `linear-gradient(var(--grid-pattern-color) 1px, transparent 1px)` × 2 (horizontal + vertical). Opacity 0.06 light, dark uses `--ref-dark-border`. Tokens: `--grid-pattern-size: 48px`, `--grid-pattern-color`, `--grid-pattern-opacity`.
  - **Theme toggle in hero** — `<x-theme-toggle id="theme-toggle-hero" />` ditempatkan di hero section (kanan, sebelah pixel-cube), square blocky style konsisten dengan estetika.

**Dark Mode / Theme Toggle System — 2 modes LIVE:**
- **Tokens Dual** — `tokens.css` dual block `html[data-theme="dark"]` + `html.dark` (Tailwind `@custom-variant dark`) + `@media (prefers-color-scheme: dark)` fallback when JS hasn't set data-theme. Opsi A single system dual tokens (ikut system default, accent dark 62%).
- **Theme Toggle** — `components/theme-toggle.blade.php` 8-state (default/hover/focus/active/disabled/loading/error/success), sun↔moon crossfade opacity 150ms (not display:none), aria-pressed, outside nav placement (desktop+mobile) + hero placement, contrast light 4.54:1 / dark 7.3:1 PASS.
- **Controller** — `resources/js/app.js` localStorage `fofa-theme` > `prefers-color-scheme` system listener + MutationObserver untuk Livewire, FOUC guard inline script (localStorage > system), `app.css` `color-scheme: light dark` + body transition `background-color / color 240ms ease-out`.
- **Tailwind v4** — CSS-first configuration (`@import "tailwindcss"` + `@custom-variant dark`), `@tailwindcss/vite` plugin (no PostCSS). Storefront header/footer use CSS custom properties (`var(--color-*)`) for reliable theme switching; admin/customer views use Tailwind `dark:` utilities which work via cascade layers. No `tailwind.config.js` — theme config in CSS.
- **Page Title System** — Dynamic per-page `<title>` via `$title` variable. Format: `[Page Name] - Fofa Shop`. Auth pages (Livewire Volt) pakai `View::share('title', ...)` di component PHP. Admin/customer dashboard pakai `<x-slot name="title">` yang diakses di layout. Layout fallback: `$title ?? config('app.name', 'Fofa Shop')`.
- **Favicon** — `public/logo.png` (Fofa Shop red circle FS monogram) + generated `favicon.ico`, `favicon-32x32.png`, `favicon-16x16.png`. `<link rel="icon">` di semua 6 layout files.

**Behaviour:** Pure presentational — tidak ada perubahan business logic (checkout/invoice/distributor/wallet). Implementasi actual lebih kaya dari deskripsi awal "Minecraft subtle blocky" → sekarang "Minecraft white-red blocky dual + green creature + grid BG (OKLCH 98.2%↔16% + accent 58%↔62% warm + mc-green 62% 0.17 145 mascot-only, slab rx8, pixel-cube Tier-A, mascot creature + 6 particles + blocky button + scroll fling, Catalogue, N7+Ft8)" per log Hallmark internal (tidak dipublikasikan).

**Quick Reference:** dokumen design system terpisah (tidak disertakan di repository ini) — color palette, typography, graphics tech stack, motion tokens, common bug patterns. Load this FIRST when debugging design/theme issues.

## Language

### Identity & Role

**User**:
Entitas identitas tunggal di sistem. Punya role enum: `customer`, `reseller`, `admin`, `owner`. Satu jalur auth — session untuk storefront/admin, Sanctum token untuk reseller (token nempel ke `user_id` yang sama). Guest checkout tidak membuat user — `user_id` nullable di `transactions`, `guest_email` opsional (checkbox "Saya ingin menerima notifikasi email" default UNCENTANG, centang = simpan email & terima notifikasi). Punya field `email_notification_enabled` (bool, default true) untuk preferensi pengumuman.
_Avoid_: Account, member

**Reseller Profile**:
Data spesifik reseller yang terpisah dari `User`. Relasi 1-1 ke `users`. Berisi `api_key`, `api_secret`, `markup_percentage`, `callback_url`, `status`. Tidak punya saldo sendiri — saldo dikelola di tabel `wallets`.
_Avoid_: Seller, vendor, partner

**Admin**:
User dengan role `admin`. Mengelola produk, transaksi, dan melakukan refund manual untuk customer biasa. Akses panel admin via session + middleware role.
_Avoid_: operator, staff

**Owner**:
User dengan role `owner`. Punya semua akses admin plus hak istimewa: hapus produk kapan saja, kelola role admin lain, manage voucher/promo, export data pajak, alert fraud. Bukan "superadmin" — role terpisah di enum.
_Avoid_: superadmin, boss

### Product & Category

**Category**:
Kategori game (Minecraft, Mobile Legends, Free Fire, dll). Tabel terpisah, dinamis — admin bisa CRUD kapan aja tanpa deploy ulang. Punya `name`, `slug`, `icon_url`, `display_order`, `is_active`, `status` (enum: `active` / `coming_soon`). Kategori `active` berfungsi penuh. Kategori `coming_soon` tampil di storefront dengan badge "Segera Hadir" tapi tombol beli disabled. Kategori `is_active = false` tersembunyi total.
_Avoid_: genre, group

**Product**:
Item yang dijual ke user (diamond, UC, voucher). Punya `provider_code` (kode di distributor), `price` (harga jual ke customer), `cost_price` (harga beli dari distributor), `image_path` (nullable, foto produk untuk game lain & marketplace — upload JPG/PNG/WebP max 2MB via `public` disk, accessor `image_url`), `is_active`, `deleted_at` (nullable, soft delete). Relasi FK ke `categories` + optional `subcategories`.
_Avoid_: item, sku, listing

**Subcategory**:
Subkategori Minecraft (Vanilla, Skyblock, Mini Games, dll). Tabel `subcategories` dengan `category_id`, `name`, `slug`, `image_path` (nullable, foto subkategori Minecraft — upload sama), `display_order`, `is_active`. Relasi ke `products` (optional) — tampil di form produk & katalog (badge/filter). Upload via Livewire `WithFileUploads` ke `storage/app/public/subcategories`.
_Avoid_: sub-category, child-category

### Transaction & Payment

**Transaction**:
Entitas inti yang merekam seluruh proses — dari checkout sampai fulfillment. Punya `ref_id` (unique, untuk idempotency), `user_id` (nullable untuk guest), `guest_email` (opsional untuk guest — hanya diisi jika checkbox "Saya ingin menerima notifikasi email" dicentang; nullable di DB karena user login pakai `users.email`), `product_id`, `target_game_id`, `amount`, `status`. Satu source of truth; "invoice" cuma nama endpoint/view. Untuk kategori Minecraft, transaksi berhenti di `waiting_fulfillment` sampai admin eksekusi manual.
_Avoid_: order, invoice

**Payment**:
Data pembayaran yang terpisah dari transaksi. Relasi 1-N ke `transactions` — satu transaksi bisa punya multiple payment attempts (audit trail + retry). Punya `method`, `gateway_ref`, `status`, `signature_verified`, `paid_at`.
_Avoid__: billing, charge

**Payment Method Toggle**:
Admin/owner bisa aktif/nonaktifkan metode pembayaran. Metode yang nonaktif tidak tampil di halaman checkout. Disimpan di tabel `admin_payment_methods`.
_Avoid_: payment toggle, method switch

### Fulfillment

**Distributor Log**:
Log terstruktur setiap percobaan hit API distributor. Kolom terstruktur (`http_status_code`, `duration_ms`, `status`, `error_message`) untuk query & monitoring, plus JSON payload untuk debugging detail. `attempt_number` untuk tracking percobaan keberapa.
_Avoid__: provider log, api log

**Waiting Fulfillment**:
Status transaksi ketika sudah dikirim ke distributor tapi belum ada hasil akhir. Berbeda dari `processing` (status transisi sebelum dikirim). Untuk kategori Minecraft, status ini berarti transaksi menunggu aksi manual admin (eksekusi item di server).
_Avoid__: pending_fulfillment, processing_fulfillment

**Manual Fulfillment**:
Proses fulfillment tanpa integrasi API otomatis. Digunakan untuk kategori Minecraft — admin melihat transaksi di `waiting_fulfillment`, mengeksekusi item secara manual di server game, lalu mengklik "Mark as Success" di admin panel. Tidak ada retry otomatis karena tidak ada API yang dihubungi.
_Avoid__: manual processing, hand fulfillment

### Category Status

**Category Status (enum: active / coming_soon)**:
Status kategori game. `active` = kategori berfungsi penuh, produk bisa dibeli. `coming_soon` = kategori tampil di storefront dengan badge "Segera Hadir", tombol beli disabled, produk tidak bisa dibeli. `is_active = false` = tersembunyi total. Dual mechanism: `is_active` untuk visibility, `status` untuk buy capability.
_Avoid_: category_state, availability

### Wallet & Refund

**Wallet**:
Saldo reseller. Tabel terpisah dari `reseller_profiles` — `reseller_profiles.balance` tidak dipakai, semua transaksi saldo lewat `wallets` + `wallet_transactions` untuk audit trail.
_Avoid__: balance, credit

**Wallet Transaction**:
Catatan setiap pergerakan saldo reseller. Type enum: `deposit`, `deduction`, `refund`. `reference_id` nullable — refund punya reference ke transaction, deposit tidak.
_Avoid__: wallet_log, balance_history

**Refund (Customer)**:
Proses manual oleh admin. Transfer ke rekening/e-wallet user berdasarkan data yang disubmit. Form isi rekening tujuan otomatis muncul di halaman invoice saat status `failed`.
_Avoid__: return, reversal

**Refund (Reseller)**:
Proses otomatis & instan. Begitu status transaksi `failed`, sistem langsung tambah `wallets.balance` dan catat di `wallet_transactions` sebagai type `refund`.
_Avoid__: return, reversal

### Pricing

**Cost Price**:
Harga beli produk dari distributor. Basis untuk menghitung harga reseller. Sinkron otomatis via sync harga distributor.
_Avoid__: wholesale price, base price

**Markup Percentage**:
Fee yang platform ambil dari reseller. Disimpan di `reseller_profiles`. Harga reseller dihitung: `cost_price + (cost_price × markup_percentage)`. Bukan margin reseller — margin reseller ditentukan sendiri di luar sistem kita.
_Avoid__: margin, commission

**Calculate Price For User**:
Method/service yang menentukan harga berdasarkan role user. Customer → `products.price`. Reseller → `cost_price + (cost_price × markup_percentage)`. Admin → `cost_price` langsung.
_Avoid__: get_price, price_resolver

### Voucher & Promo

**Voucher**:
Kode diskon yang dibuat oleh owner. Tipe: `percentage` (persen) atau `fixed` (nominal). Scope global — berlaku untuk semua produk. Punya batasan: `max_uses`, periode waktu (`starts_at`, `expires_at`), minimum order amount.
_Avoid__: promo, coupon, discount code

**Voucher Usage**:
Catatan pemakaian voucher. Relasi ke `voucher`, `user`, dan `transaction`. Menyimpan `discount_amount` (nilai diskon yang diterapkan).
_Avoid__: voucher redemption, promo usage

### Click Tracking & Reseller Store

**Click Log**:
_Dropped in v1.3 — click tracking tidak diperlukan untuk toko skala komunitas. Kalau dibutuhkan nanti, implementasi ulang._

**Reseller Store**:
_Dropped in v1.3 — reseller store tidak diperlukan untuk toko skala komunitas. Kalau dibutuhkan nanti, implementasi ulang._

**Conversion Rate**:
_Dropped in v1.3 — conversion tracking tidak diperlukan untuk toko skala komunitas._

### Communication

**Announcement**:
Pengumuman yang dibroadcast ke user. Tipe: `info`, `maintenance`, `warning`. Target: `all` (customer+reseller), `customer`, `reseller`, `admin`. Disimpan di tabel `announcements`. Hanya `admin` & `owner` yang bisa membuat. Broadcast via `SendAnnouncementJob` (queue): buat `notifications` per user + kirim `AnnouncementMail` hanya ke yang `email_notification_enabled = true`. Admin/owner tidak menerima announcement (hanya notifikasi penting).
_Avoid_: broadcast, notification message

**Notification (In-app)**:
Pesan in-app yang muncul di header/nav user. Belum dibaca (`read_at` nullable). Dibuat otomatis saat announcement publish untuk setiap target user (customer/reseller sesuai `target_audience`). Berbeda dari email — ini notifikasi dalam aplikasi, tidak respect opt-out. Email pengumuman respect `email_notification_enabled`.
_Avoid_: alert, popup, toast

**Notification Preference**:
Preferensi email untuk pengumuman & promosi. Field `users.email_notification_enabled` (bool, default true). Toggle di halaman Profile (`livewire:profile.notification-preferences`). Jika dimatikan, user tetap dapat email bukti transaksi (paid/success/failed/expired/reconciliation = WAJIB), tapi tidak dapat email pengumuman. Guest `guest_email` null = tidak dapat email apapun.
_Avoid_: email setting, notification toggle

**Chat Room**:
_Dropped in v1.3 — live chat tidak diperlukan untuk toko skala komunitas. Kalau dibutuhkan nanti, implementasi ulang dengan Reverb._

**Chat Message**:
_Dropped in v1.3 — live chat tidak diperlukan untuk toko skala komunitas._

### Security & Audit

**Audit Log**:
Catatan setiap aksi CRUD oleh admin/owner. Menyimpan `action`, `auditable_type`, `auditable_id`, `old_values` (json), `new_values` (json) untuk investigasi. Tidak ada UI viewer — akses via Tinker/query manual.
_Avoid__: activity log, action log, system log

**Fraud Alert**:
_Dropped in v1.3 — fraud detection tidak diperlukan untuk toko skala komunitas. Monitoring manual via admin dashboard._

**Fraud Rule**:
_Dropped in v1.3 — fraud detection tidak diperlukan untuk toko skala komunitas._

**Product Deletion Log**:
Audit saat produk dihapus. Menyimpan snapshot produk (json), siapa yang hapus, kapan, dan alasan. Produk yang sudah pernah dibeli tidak bisa dihapus — hanya bisa di-archive via log ini.
_Avoid__: product archive, delete history

### Financial Data Access Control

**Financial Data Access Control**:
Pembatasan akses data finansial sensitif berdasarkan role user. Admin hanya bisa melihat data operasional (status transaksi, produk, game ID, log distributor) tanpa angka rupiah. Owner memiliki akses penuh ke data finansial (nominal per transaksi, total omset, export PDF). Implementasi di level query (column exclusion via `scopeForAdmin()`), bukan cuma UI hide — data benar-benar tidak di-select dari database untuk admin.
_Avoid__: financial visibility, amount access, revenue access

**Rationale — Admin vs Owner Financial Access**:
Admin bertanggung jawab atas operasional harian (retry transaksi gagal, fulfillment manual Minecraft, monitoring status) sehingga butuh visibilitas penuh atas detail barang/produk dan status transaksi. Namun data finansial sensitif (nominal uang, omset, laporan keuangan) dibatasi hanya untuk Owner sebagai pemegang keputusan bisnis tertinggi. Product price (`products.price`, `products.cost_price`) tetap terlihat admin karena dibutuhkan untuk operasional manage produk — yang dibatasi adalah transaction amount dan kalkulasi omset.
_Avoid__: financial rationale, access control reason

### Minecraft Edition Scenario

**Minecraft Edition (enum: java / bedrock / manual)**:
Skenario checkout khusus kategori Minecraft. Dipilih via dropdown yang hanya muncul jika `product.category.slug === 'minecraft'`. Nilai disimpan di `transactions.minecraft_edition`. Mengontrol tampil/sembunyi + wajib/tidak-nya field `target_game_id` secara reaktif via Livewire `$edition` (`wire:model.live` + `x-show`), validasi kondisional di `CheckoutRequest`, dan `try/catch` sebelum membeli agar pembeli tidak 500 saat field beda format.
- `java` — Java Edition (username) — field ID WAJIB, format `^[a-zA-Z0-9_]{3,16}$`, verifier nickname.
- `bedrock` — Bedrock Edition (Gamertag) — field ID WAJIB, format `^[a-zA-Z0-9_ ]{3,12}$` (boleh spasi tengah, tidak boleh awali/akhiri spasi).
- `manual` — Tanpa ID (ambil manual via Discord/WA) — field ID DISEMBUNYIKAN & tidak wajib (`target_game_id` nullable), `guest_email` WAJIB untuk guest (kode/akun dikirim via email), transaksi tetap `waiting_fulfillment` sampai admin kirim.
_Avoid__: minecraft scenario, edition dropdown, game mode

**Minecraft Account Delivery**:
Admin mengirim akun Minecraft baru via email saat fulfillment manual. Field di `transactions`: `delivered_minecraft_username`, `delivered_minecraft_password`, `delivery_note` (nullable). Input di `admin/transactions/show` (form mark-manual-success dengan 3 field), disimpan lalu dikirim via `MinecraftAccountDeliveredMail` (queue) ke `guest_email ?? user.email`. Buyer juga bisa lihat akun di `invoice` & `history/show` (box hijau) selain email. Status tetap `waiting_fulfillment → success` + `completed_at` + audit log `manual_success`.
_Avoid__: minecraft delivery, account sending, manual fulfill email
