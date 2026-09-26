# Fofa Shop — Unified Spec

Status: ready-for-agent

Tagline: **Toko Top-Up Skala Komunitas Minecraft**

---

## Version History

| Version | Date | Scope | Issues |
|---------|------|-------|--------|
| **v1.0** | 2026-08-26 | Core MVP — storefront, checkout, payment webhook, distributor fulfillment, admin panel, reseller API, wallet system | #01-#12 |
| **v1.1** | 2026-08-27 | Hardening — pricing integrity, guest/reseller/admin actor clarification, race-safe checkout, ref_id entropy, code quality | #01-#05 (repair) |
| **v1.2** | 2026-08-27 | MVP Expansion — owner role, vouchers, click tracking, reseller store, search, nickname check, customer history, payment toggle, broadcast, live chat, fraud detection, reports/export, API docs | #14-#33 |
| **v1.3** | 2026-08-28 | Scope-cut (community-scale) + Minecraft-first launch — drop click tracking, reseller store, live chat, fraud detection, standalone reports; simplify audit log, owner settings, PDF export; Minecraft category with manual fulfillment; guest email mandatory; other games "Coming Soon" | #15-simplified, #25-simplified, #26-simplified, #27-new |
| **v1.3.1** | 2026-08-29 | Transaction chart + PDF export + Financial data access control — grafik transaksi interaktif (harian/mingguan/bulanan), export PDF mengikuti filter aktif, query-level column exclusion: admin tanpa data finansial, owner akses penuh | #25-expanded |
| **v1.3.2** | 2026-08-30 | Minecraft edition scenario + account delivery via email — dropdown edisi (Java/Bedrock/Manual) khusus Minecraft dengan Livewire reactive, validasi & try/catch per-edisi, serta admin fulfillment kirim akun Minecraft baru via email | #27-expanded |
| **v1.3.3** | 2026-08-30 | Upload foto produk & subcategory — admin bisa upload foto produk game lain (public/products) & foto subkategori Minecraft (public/subcategories) via Livewire WithFileUploads, tampil di katalog & detail | #27-expanded |
| **v1.3.4** | 2026-08-30 | Design System & Theme — dokumentasi Hallmark overhaul (brand lockup, pixel-cube, badge tokens, card animations) + dual light/dark theme (tokens.css OKLCH dual, theme-toggle 8-state, FOUC guard, app.js controller) — doc-only, tanpa perubahan kode | docs |
| **v1.3.5** | 2026-08-30 | Minecraft Animations — mascot creature (idle/bounce + scroll fling), 6 floating particles (hero only), blocky button press, subtle grid BG pattern, green mc-green tokens (mascot/particles only), theme toggle in hero | — |

---

## Problem Statement

### v1.0

Fofa Shop butuh website top-up game yang bisa dijual ke klien. Saat ini project baru fresh Laravel 13 + Breeze + Livewire/Volt, belum ada domain logic sama sekali. Yang dibutuhkan: sistem transaksi end-to-end (customer beli → bayar → distributor proses → sukses/gagal), admin panel untuk manage produk & transaksi, API untuk reseller H2H, dan integrasi payment gateway + distributor (Digiflazz).

### v1.1

Setelah implementasi awal issues 01-05 (database, auth, admin CRUD, storefront, checkout/invoice), review dua-axis menemukan gap finansial & keamanan yang harus ditutup sebelum lanjut ke 06-12: `User` mass-assignment `role`, `ref_id` race pada double-click/concurrent, entropy invoice rendah (4-char brute-force), pricing `CalculatePriceForUser` belum memperlakukan Admin sebagai Reseller + BONUS, dan belum ada pemisahan konsep `reseller_profiles` vs Admin.

### v1.2

Fofa Shop currently has a working core (storefront, checkout, payment webhook, distributor fulfillment, admin panel, reseller API, wallet system). However, the platform lacks critical features needed for production use: owner-level controls, promotional tools (vouchers), reseller analytics, customer self-service, real-time support, fraud detection, and financial reporting. Without these, the business cannot scale beyond basic top-up transactions.

### v1.3

Two major scope changes for launch readiness:

**A. Scope-cut (community-scale shop):** Simplify to 3 user roles — guest, admin, owner. Reseller channel (API + wallet) exists in code but dormant (no registration UI, admin manually provisions). Drop features that add complexity without immediate value: click tracking, reseller store, click analytics, reseller monthly reports, live chat/Reverb, fraud detection, standalone finance reports, reseller API docs. Simplify: audit log (#15 — no UI viewer), owner settings (#26 — use .env, keep admin management UI), PDF export (#25 — from admin transactions page, not standalone dashboard).

**B. Minecraft-first launch:** Minecraft opens as the first active category. Other game categories (Mobile Legends, Free Fire, etc.) stay in database but display as "Coming Soon" — visible in storefront with disabled buy button and "Segera Hadir" badge. Minecraft fulfillment is manual: admin sees `waiting_fulfillment` transaction, executes item in-game, clicks "Mark as Success". No RCON integration. Checkout wajib isi email untuk guest users (notifikasi status transaksi), logged-in users use account email.

### v1.3.1

**Financial data access control + transaction chart + PDF export:**

**A. Financial data access control:** Admin dan Owner punya akses yang sama ke fitur operasional, tapi data finansial sensitif (nominal per transaksi, total omset, export PDF) dibatasi hanya untuk Owner. Implementasi di level query via `scopeForAdmin()` yang exclude kolom `amount` dari SELECT — data benar-benar tidak sampai ke browser admin. Ini BUKAN cuma UI hide.

**B. Transaction chart:** Halaman `/admin/transactions` mendapat line chart interaktif (Chart.js via CDN) yang menampilkan jumlah transaksi per hari/minggu/bulanan. Chart count-based (aman untuk admin karena tidak ada data finansial). Filter periode (harian/mingguan/bulanan) mempengaruhi tampilan chart DAN filter data di tabel.

**C. PDF export (owner-only):** Tombol "Export PDF" hanya muncul untuk Owner. PDF mengikuti filter periode yang sedang aktif di grafik — jika grafik di-set bulanan, PDF yang di-generate adalah laporan untuk bulan tersebut. Route: `POST /admin/transactions/export-pdf` dengan guard `abort_unless(isOwner, 403)`.

### v1.3.2

**Minecraft edition scenario + account delivery via email:**

**A. Edition scenario:** Checkout Minecraft punya dropdown edisi yang hanya muncul jika `product.category.slug === 'minecraft'` (cek `isMinecraft()`). 3 opsi: `java` (username 3-16, `^[a-zA-Z0-9_]{3,16}$`), `bedrock` (Gamertag 3-12, `^[a-zA-Z0-9_ ]{3,12}$`), `manual` (tanpa ID, `target_game_id` nullable, `guest_email` wajib). Livewire reactive `$edition` (`wire:model.live` + `x-show`) mengontrol tampil/sembunyi & required, validasi kondisional di `CheckoutRequest`, `try/catch` sebelum membeli agar format beda tidak 500. Non-Minecraft tetap `target_game_id` required tanpa dropdown.

**B. Account delivery:** Admin di `admin/transactions/show` untuk `waiting_fulfillment` Minecraft bisa isi `minecraft_account_username`/`password`/`delivery_note` lalu `Mark as Success & Kirim via Email` — simpan ke `transactions` + audit log + queue `MinecraftAccountDeliveredMail` ke `guest_email ?? user.email`. Buyer lihat akun di `invoice` & `history/show` (box hijau) selain email.

### v1.3.3

**Upload foto produk & subcategory:**
Admin butuh upload foto produk game lain (MLBB, FF, dll) dan foto subkategori Minecraft (Vanilla, Skyblock) agar katalog tidak polos teks. Sebelumnya produk hanya punya `name`/`price` tanpa visual, subkategori juga tanpa foto.
- Produk `image_path` (nullable, `storage/app/public/products` — JPG/PNG/WebP max 2MB) via Livewire `WithFileUploads`, preview + hapus, tampil di `storefront/category` (card top) & `storefront/show` (hero).
- Subkategori `image_path` (nullable, `storage/app/public/subcategories`) via form yang sama, tampil di `category` card (small badge) & `show` detail. Admin `products/index` & `subcategories/index` tampil thumbnail.
- Validasi `image:nullable|image|mimes:jpg,jpeg,png,webp|max:2048`, old image di-delete saat replace/remove, accessor `image_url`.

### v1.3.4 — Design System & Theme (Hallmark + Dark Mode) — Doc-Only

**Konteks:** Review v1.3 menemukan deskripsi "Minecraft subtle blocky" di spec/CONTEXT belum mencerminkan implementasi aktual yang lebih kaya: Hallmark overhaul (Catalogue macrostructure, brand lockup, pixel-cube, badge tokens, card animations, OKLCH dual tokens, typography 2+1) + dual light/dark theme system. Perintah 2 meminta sinkronisasi dokumentasi tanpa perubahan kode.

**Hallmark Design System Overhaul:**
- **Macrostructure Catalogue** — F6 grid 4→3→2→1 (`catalogue-grid`), nav N7 Brutal slab (2px ink border, sticky), footer Ft8 Marquee scroll dual, density responsive (60:30:15 rule).
- **Brand Lockup** — `components/application-logo.blade.php` slab rect rx 8 + FS monogram (shopping-bag handle DNA dari logo.png/tagline.png) + `brand-lockup.blade.php` pill MINECRAFT + tagline "Berbagai Kebutuhan Minecraft". Logo accent via `var(--color-accent)`, paper via `var(--color-paper)`, voxel highlight `opacity 0.14`.
- **Pixel-cube CSS** — Tier-A CSS pixel cube subtle blocky (bukan voxel penuh, 8px radius), shadow adaptif light/dark (`--color-shadow` / `0.07` light vs `0.45` dark), 2 tiny pixels bottom-right sebagai Minecraft DNA (bukan dekorasi-for-its-own).
- **Badge Tokens** — WCAG badge text vs paper 4.5:1, dual light/dark (`--ref-green-text` 36% vs `84%` dark, dst), `transaction-status-badge` component 6 state (success/danger/warning/info/processing/neutral), background soft vs text contrast pass Gate 41 (accent-ink 4.54:1 light, 7.3:1 dark).
- **Card Animations** — `card-enter` keyframe stagger (opacity 0→1 + translateY 8px→0, 300ms ease-out, 40ms stagger per card), hover lift `-1px` + shadow `0 4px 16px`, press feedback `translateY(0)` / `scale(0.97)`, `prefers-reduced-motion` nuclear block + targeted Alpine transition guard.

**Dark Mode / Theme Toggle System:**
- **Tokens** — `tokens.css` OKLCH dual custom-dual: light paper `oklch(98.2% 0.008 35)` + accent `58% 0.22 24`, dark paper `oklch(16% 0.02 35)` + accent `62% 0.24 24` warm. Dual via `html[data-theme="dark"]` + `html.dark` (Tailwind `darkMode: class`) + `@media (prefers-color-scheme)` fallback. Contrast pass semua gate (paper vs ink 17.2:1 light / 14.8:1 dark, muted 5.26:1 / 7.1:1, focus 3:1).
- **Theme Toggle** — `components/theme-toggle.blade.php` 8-state (default/hover/focus/active/disabled/loading/error/success) + sun/moon SVG crossfade `opacity 150ms`, `aria-pressed`, outside nav placement. FOUC guard inline script (localStorage `fofa-theme` > system), `resources/js/app.js` controller (localStorage + system listener + MutationObserver untuk Livewire), body transition `background-color / color 240ms ease-out`.
- **Implementation** — `tailwind.config.js` `darkMode: class` + colors `var(--color-*)`, `app.css` `color-scheme: light dark` + dark overrides (nav/marquee/selection/pixel-cube shadow + body transition), `storefront.blade.php` + `app.blade.php` FOUC guard + `<x-theme-toggle>` desktop+mobile. Tidak ada perubahan behavior checkout/transaksi — pure presentational.

**Sync:** Deskripsi "Minecraft subtle blocky" di-update menjadi "Minecraft white-red blocky dual (OKLCH 98.2%↔16% + accent 58%↔62% warm, slab rect rx 8 subtle blocky, pixel-cube Tier-A, Catalogue, N7+Ft8)" di CONTEXT.md dan spec.

### v1.3.5 — Minecraft Animations + Grid BG + Green Tokens + Hero Toggle

**Konteks:** Storefront perlu terasa hidup dan playful sesuai tema Minecraft (mirip pengalaman undangan digital) tanpa mengorbankan performa dan aksesibilitas. Semua animasi harus lolos Hallmark preflight gates (10: no transition-all, 11: no hover:scale-105, 27: reduced-motion, 48: token-only).

**Mascot creature:**
- Abstract block creature dari 9 CSS blocks (head, 2 eyes, mouth, body, 2 arms accent, 2 feet) — `resources/css/animations.css` class `.mc-mascot` + `.mc-mascot__block--*`.
- Idle bounce: `@keyframes mascot-bounce` 2.4s ease-in-out infinite, translateY 0↔-6px.
- Scroll fling: `resources/js/mascot-scroll.js` — scroll event dengan `requestAnimationFrame` throttle (`ticking` flag), NO unthrottled scroll listener. Mascot ter-'lempar' ke atas (translateY) + slight rotate, 300ms debounce return-to-idle via CSS transition. Passive listener. Livewire cleanup via `livewire:navigate`. Reduced-motion: module bails entirely.
- Posisi: hero section, di atas headline sebagai dekorasi.

**Floating particles:**
- 6 block squares di hero section only (`.mc-particles` container `position: absolute; inset: 0; pointer-events: none`).
- 4 green (`var(--mc-green)`) + 2 accent (`var(--color-accent)`) dengan opacity rendah (0.2-0.35).
- Stagger via `animation-delay`, `@keyframes float-particle` translateY -12px + rotate 3deg.

**Blocky button:**
- `.blocky-btn` class: border 2px + `box-shadow: 3px 3px 0 var(--color-ink)`.
- `:active` → shadow shrinks to `1px 1px 0` + `translateY(1px)` — NOT `scale()`.
- Gate 10: transition specifies exact properties (box-shadow, transform, background-color).
- Gate 11: no `hover:scale-105`.

**Green tokens (scope-limited):**
- `--mc-green: oklch(62% 0.17 145)`, `--mc-green-dark: oklch(55% 0.15 145)`, `--mc-green-soft`, `--mc-green-soft-dark`.
- **SCOPE: mascot + particles ONLY — NOT as widespread secondary accent.**
- Dark mode overrides in tokens.css.

**Grid pattern BG:**
- `body` background-image: repeating 48px grid lines via `linear-gradient(var(--grid-pattern-color) 1px, transparent 1px)` × 2 (horizontal + vertical).
- Tokens: `--grid-pattern-size: 48px`, `--grid-pattern-color`, `--grid-pattern-opacity: 0.06`.
- Dark mode: uses `--ref-dark-border`.

**Theme toggle in hero:**
- `<x-theme-toggle id="theme-toggle-hero" />` ditempatkan di hero section (kanan, sebelah pixel-cube), square blocky style konsisten.

**Files:**
- Created: `resources/css/animations.css`, `resources/js/mascot-scroll.js`
- Modified: `tokens.css` (+green +grid tokens +dark overrides), `app.css` (+import animations, +grid BG, +dark overrides), `storefront/index.blade.php` (+mascot +particles +hero toggle), `layouts/storefront.blade.php` (+vite import mascot-scroll.js), `vite.config.js` (+input entry)

---

## Solution

### v1.0

Bangun full-stack monolith Laravel dengan:
- Storefront (Blade/Livewire) untuk customer beli produk game
- Admin panel untuk manage produk, kategori, transaksi, refund manual
- Reseller API (Sanctum) untuk transaksi H2H via API
- Integrasi payment gateway (Tripay/Duitku) via webhook
- Integrasi distributor (Digiflazz) via queue jobs dengan auto-retry 3x exponential backoff
- Wallet system untuk reseller (deposit, saldo, refund otomatis)
- Email notification di setiap transisi status transaksi

### v1.1

Hardening slice 01-05 tanpa mengubah skema besar:
- Tutup mass-assignment `role` dan rapikan `AdminSeeder`/`CreateAdminUser` agar `role` hanya set via assignment langsung.
- Buat `POST /checkout` race-safe dengan retry `QueryException` 23000 (duplicate `ref_id`) dan naikkan entropy `ref_id` dari 4 → 8 char (env `FOFA_REF_ID_LENGTH`).
- Perjelas `CalculatePriceForUser`: Guest/Customer → `products.price`; Sales/Reseller (WAJIB login, punya row `reseller_profiles`) → `cost_price + markup%`; Admin → **langsung `cost_price` tanpa cek `reseller_profiles`** (opsi 2, tidak numpang tabel reseller, perubahan markup reseller tidak mempengaruhi bonus admin).
- Dokumentasikan 3 aktor di `CONTEXT.md`/`spec` agar semua pricing, auth, dan test konsisten.

### v1.2

Expand the existing Laravel monolith with 17 new features across 10 implementation phases, adding 14 new database tables, 7 new ADRs, a new `owner` user role, and 3 new Composer dependencies. The solution preserves all existing architecture decisions (single Transaction entity, 1-N payments, dynamic categories, wallet system, manual vendor failover) while layering new capabilities on top.

### v1.3

Toko top-up skala komunitas: guest + admin + owner saja, reseller channel dormant. Minecraft-first launch — kategori Minecraft aktif dengan fulfillment manual admin, kategori game lain (MLBB, FF, dll) tampil di storefront sebagai "Coming Soon". Checkout wajib isi email untuk guest. Export PDF transaksi dari admin panel. Scope-cut menghapus click tracking, reseller store, live chat, fraud detection, standalone reports — semua fitur yang tidak kritis untuk launch.

### v1.3.1

Halaman `/admin/transactions` ditingkatkan dengan grafik transaksi interaktif (line chart per hari/minggu/bulanan), export PDF yang mengikuti filter periode aktif, dan pembatasan akses data finansial di level query. Admin melihat semua data operasional (produk, status, log distributor) tanpa angka rupiah. Owner memiliki akses penuh termasuk nominal, omset, dan export PDF.

### v1.3.2

Minecraft checkout dengan 3 skenario edisi (Java/Bedrock/Manual) — dropdown Livewire hanya untuk Minecraft (`product.category.isMinecraft()`), validasi per-edisi + `try/catch` sebelum membeli, serta admin fulfillment kirim akun Minecraft baru via email. Enum `MinecraftEdition`, `minecraft_edition` di `transactions`, `target_game_id` nullable untuk manual, `delivered_minecraft_username/password` + `delivery_note` untuk akun, `GameIdCheck` Livewire reactive (`$edition` + `x-show` + `minecraft-edition-changed` event), `CheckoutRequest` kondisional, `FallbackGameIdVerifier` union Java/Bedrock, `MinecraftAccountDeliveredMail` via `queue()`.

### v1.3.3

Upload foto produk & subkategori — `products.image_path` & `subcategories.image_path` nullable di `public` disk (`products/`, `subcategories/`), Livewire `WithFileUploads` 2MB JPG/PNG/WebP, preview + hapus checkbox, admin `products/index` & `subcategories/index` thumbnail, storefront `category` card & `show` hero menampilkan `Storage::url()`, accessor `image_url`.

### v1.3.5

Minecraft animations — mascot creature (abstract 9-block CSS art, idle bounce 2.4s, scroll fling dengan rAF throttle + ticking flag + 300ms return-to-idle), 6 floating particles (hero only, green + accent, staggered), blocky button (border 2px + box-shadow offset shrink on :active, NOT scale), subtle grid pattern BG (48px repeating, 0.06 opacity), green mc-green tokens (mascot/particles ONLY, NOT widespread), theme toggle in hero section. All animations respect `prefers-reduced-motion: reduce`. Files: `resources/css/animations.css`, `resources/js/mascot-scroll.js`, `tokens.css` (green + grid tokens), `app.css` (import + grid BG), `index.blade.php` (mascot + particles + toggle), `layouts/storefront.blade.php` (+vite import), `vite.config.js` (+input).

---

## User Stories

### Customer (v1.0)

1. As a customer, I want to browse game categories on the storefront, so that I can find the game I want to top up
2. As a customer, I want to view products within a selected game category, so that I can choose the specific item (diamond, UC, voucher) I need
3. As a customer, I want to see product details including price, so that I can make an informed purchase decision
4. As a customer, I want to checkout by selecting a product, entering my game ID, and choosing a payment method, so that I can start the top-up process
5. As a customer, I want to receive an invoice with a unique ref_id after checkout, so that I can track my transaction
6. As a customer, I want to pay via QRIS, e-wallet, or virtual account through the payment gateway, so that I have multiple payment options
7. As a customer, I want to check my transaction status using the ref_id, so that I know whether my top-up is processing, successful, or failed
8. As a customer, I want to receive email notifications when my transaction status changes (paid, success, failed, expired), so that I don't need to constantly check manually
9. As a customer, I want to see a refund form on the invoice page when my transaction fails, so that I can submit my bank/e-wallet details for manual refund without contacting admin
10. As a customer, I want my invoice to expire quickly if I don't pay, so that pending transactions don't accumulate and waste resources
11. As a customer, I want to register an account and login, so that my transactions are linked to my identity

### Customer (v1.1 — Guest/Pelanggan Biasa)

12. As a guest, I want to browse categories and products without login, so that I can shop with minimal friction
13. As a guest, I want to checkout by entering `target_game_id` + `payment_method` without creating an account, so that I can buy quickly
14. As a guest, I want my checkout to use `products.price` (harga customer), so that I am not over/under-charged
15. As a guest, I want my invoice accessible via `ref_id` (public token) even though I have no `user_id`, so that I can track payment
16. As a guest, I want double-clicking checkout not to create duplicate transactions or crash, so that rapid clicks are safe

### Customer (v1.2)

17. As a customer, I want to search for products by name, so that I can quickly find what I need without browsing categories
18. As a customer, I want to have my game ID automatically verified before payment, so that I don't top-up the wrong account
19. As a customer, I want to see my confirmed nickname after entering my game ID, so that I can verify it's correct
20. As a customer, I want the system to gracefully skip verification if the game doesn't support it, so that I can still purchase products for unsupported games
21. As a customer, I want to apply a voucher code at checkout for a discount, so that I save money on my purchase
22. As a customer, I want to see the discount amount and final price after applying a voucher, so that I know exactly what I'm paying
23. As a customer, I want to view my transaction history, so that I can track all my past purchases
24. As a customer, I want to filter my transaction history by status, so that I can find specific transactions quickly
25. As a customer, I want to re-check the status of a pending transaction, so that I know if it's been fulfilled
26. As a customer, I want to see in-app notifications for announcements, so that I'm aware of maintenance or important updates
30. As a customer, I want to see only active payment methods at checkout, so that I don't try to use a disabled method

### Reseller (v1.0)

31. As a reseller, I want to authenticate via API token (Sanctum), so that I can integrate programmatically
32. As a reseller, I want to view available products via API with my calculated price (cost_price + markup_percentage), so that I know what to offer my customers
33. As a reseller, I want to top up a customer's game account via API (POST /api/v1/topup), so that I can fulfill orders from my own platform
34. As a reseller, I want to check the status of my top-up request via API, so that I can report back to my customer
35. As a reseller, I want to check my wallet balance via API, so that I know how much deposit I have left
36. As a reseller, I want my wallet balance to be automatically deducted when I make a top-up, so that I don't need manual settlement
37. As a reseller, I want failed top-ups to be automatically refunded to my wallet, so that my balance is always accurate
38. As a reseller, I want to receive webhook callbacks to my registered URL when transaction status changes, so that my system stays in sync

### Reseller (v1.1)

39. As a reseller, I want to be required to login before getting reseller price, so that the discount is not abused by guests
40. As a reseller, I want my price calculated as `cost_price + (cost_price × markup_percentage)` from my `reseller_profiles` row, so that my margin is correct
41. As a reseller without a profile row, I want to fallback to `products.price` (not crash), so that misconfiguration is visible
42. As a reseller, I want my `markup_percentage` changes (e.g., promo 0%) to affect only resellers, not admin bonus, so that business rules stay separate
43. As a reseller, I want my identity (`user_id`) stored on `transactions` when I checkout logged-in, so that audit is correct

### Reseller (v1.2)

_44-53: Dropped in v1.3 — click tracking, reseller store, analytics, monthly reports not needed for community-scale launch. Reseller channel dormant._

### Admin (v1.0)

54. As an admin, I want to manage (CRUD) game categories with name, slug, icon, and display order, so that the storefront stays organized
55. As an admin, I want to add new game categories dynamically without deploying code, so that new games can be added quickly
56. As an admin, I want to manage (CRUD) products with name, price, cost_price, and is_active status, so that I can control what's sold and at what price
57. As an admin, I want to assign vendor product codes for multiple distributors per product (vendor_mappings), so that the system has fallback options if primary distributor fails
58. As an admin, I want to sync product prices from the distributor automatically, so that prices stay current without manual updates
59. As an admin, I want to view all transactions with their current status, so that I can monitor the health of the system
60. As an admin, I want to view detailed transaction information including payment and distributor logs, so that I can debug issues
61. As an admin, I want to manually retry a failed transaction, so that I can recover from distributor failures
62. As an admin, I want to process manual refunds for customers (transfer to their bank/e-wallet), so that failed transactions are resolved
63. As an admin, I want to view all registered users and their roles, so that I can manage access
64. As an admin, I want to be protected by stricter rate limiting on login (max 5 failed attempts per 15 min per IP), so that brute-force attacks are prevented

### Admin (v1.1)

65. As an admin, I want to be required to login via `auth` + `role:admin` for all `/admin/*`, so that admin panel is protected
66. As an admin, I want to checkout via the same `/checkout` if I want to buy, and be charged `cost_price` directly (BONUS, no markup), so that I get the cheapest price
67. As an admin, I want my bonus price to NOT depend on a row in `reseller_profiles` (no profile needed, not polluting that table), so that admin concept stays clean
68. As an admin, even if a `reseller_profiles` row somehow exists for my user_id, I want it ignored and still get `cost_price`, so that admin bonus is deterministic
69. As an admin, I want my purchase to have `user_id` filled (not null) for audit, so that guest vs admin is distinguishable

### Admin (v1.2)

70. As an admin, I want to see a monitoring dashboard with real-time omset and distributor status alerts, so that I can manage operations
71. As an admin, I want to retry failed transactions manually, so that I can resolve fulfillment issues
72. As an admin, I want to approve reseller applications, so that I control who becomes a reseller
73. As an admin, I want to adjust reseller markup percentages, so that I can manage reseller pricing
74. As an admin, I want to manage reseller API tokens (create/revoke), so that I control API access
75. As an admin, I want to deposit wallet balance to reseller accounts, so that I can process manual transfers
76. As an admin, I want to toggle payment methods on/off, so that I can disable problematic methods without code changes
77. As an admin, I want to see a list of all payment methods with toggle switches, so that I can manage them easily
78. As an admin, I want to create and broadcast announcements (info/maintenance/warning), so that users are informed
79. As an admin, I want announcements to be delivered as in-app notifications and email blasts, so that maximum reach is achieved
80. As an admin, I want to see a notification badge with unread count, so that I know when new notifications arrive
81. As an admin, I want to see all admin/owner activity in an audit log, so that I can review who did what (data stored in table, no UI viewer — access via Tinker/DB query)
88. As an admin, I want products with existing transactions to be locked (cannot delete), so that historical data is preserved
89. As an admin, I want to soft-delete products older than 24 hours that have no transactions, so that I can clean up the catalog
83-87: _Dropped in v1.3 — chat admin, fraud alerts not needed for community-scale launch._

### Owner (v1.2)

90. As an owner, I want all admin access plus additional privileges, so that I have full control of the system
91. As an owner, I want to hard-delete products within 24 hours of creation (even without transactions), so that I can fix mistakes quickly
92. As an owner, I want to create, edit, and delete vouchers (percentage or fixed amount), so that I can run promotions
93. As an owner, I want vouchers to support max usage limits and expiry dates, so that I can control promotion scope
94. As an owner, I want to see financial reports (omset/cost/margin) with filters by date/category/product, so that I can monitor business performance
95. As an owner, I want to see product analytics ranked by sales volume, so that I know which products are most popular
96. As an owner, I want to see vendor performance metrics (success rate per distributor), so that I can evaluate vendor reliability
97. As an owner, I want to manage admin accounts (add new admin, toggle active/inactive), so that I can control who has admin access (UI sederhana, seeder untuk akun awal)
101. As an owner, I want to configure global business settings (default markup, maintenance mode) via .env, so that I can manage the platform
103. As an owner, I want all my actions to be logged in the audit log, so that there's accountability
98-100, 102: _Dropped in v1.3 — standalone CSV/Excel export, PDF dashboard, tax calculation, fraud detection not needed for community-scale launch. PDF export available from admin transactions page instead._

### Customer (v1.3 — Minecraft-first + Checkout Email)

120. As a customer, I want to see Minecraft category as the first active category on the storefront, so that I know it's available for purchase
121. As a customer, I want to see other game categories (MLBB, FF, etc.) with a "Coming Soon" badge, so that I know they will be available later
122. As a customer, I want the buy button disabled on "Coming Soon" categories, so that I cannot purchase products that aren't fulfilled yet
123. As a guest, I want to provide my email at checkout (mandatory), so that I receive transaction status notifications
124. As a customer, I want my email from my account used automatically when logged in, so that I don't need to enter it again

### Admin (v1.3 — Minecraft Manual Fulfillment + PDF Export)

125. As an admin, I want to see Minecraft transactions stuck at `waiting_fulfillment` in my transaction list, so that I can fulfill them manually in-game
126. As an admin, I want to click "Mark as Success" on a Minecraft transaction after manual fulfillment, so that the status updates to `success` and the user gets notified
127. As an admin, I want to export filtered transactions as a categorized PDF (per day/week/month), so that I can do bookkeeping

### Admin & Owner (v1.3.1 — Transaction Chart + Financial Access Control)

128. As an admin, I want to see a line chart of transaction counts per day/week/month on the transactions page, so that I can monitor transaction volume trends
129. As an admin, I want to filter the transaction list and chart by date range and period (harian/mingguan/bulanan), so that I can focus on specific timeframes
130. As an admin, I want my transaction queries to exclude financial data (amount) at the query level, so that sensitive financial information is not exposed to my session
131. As an owner, I want to see the same transaction chart plus financial data (amount per transaction, total omset), so that I can monitor both operations and business performance
132. As an owner, I want to export a PDF report that matches my currently active period filter, so that I get consistent data between screen and export
133. As an owner, I want to see total omset on the admin dashboard, so that I can get a quick financial overview

### Customer (v1.3.2 — Minecraft Edition Scenario)

134. As a customer buying Minecraft, I want to see a dropdown for Java / Bedrock / Manual only when product is Minecraft, so that I choose correct ID type
135. As a customer choosing Java Edition, I want Game ID required with username validation 3-16, so that my Java purchase is valid
136. As a customer choosing Bedrock Edition, I want Gamertag required 3-12 with spaces allowed, so that my Bedrock purchase is valid
137. As a customer choosing Manual (Tanpa ID), I want Game ID hidden and not required, but email required, so that I can buy voucher to be sent manually via Discord/WA

### Admin (v1.3.2 — Minecraft Account Delivery via Email)

138. As an admin fulfilling Minecraft `waiting_fulfillment`, I want to fill new Minecraft account username/password/note and `Mark as Success & Kirim via Email`, so that buyer receives account via email
139. As an admin, I want delivery note/code to be stored and visible in transaction detail, invoice and history, so that buyer can see account even without email

### Admin (v1.3.3 — Upload Foto)

140. As an admin, I want to upload foto produk game lain (MLBB, FF, dll) via admin product form (JPG/PNG/WebP max 2MB), so that katalog tampil menarik
141. As an admin, I want to upload foto subkategori Minecraft (Vanilla, Skyblock) via admin subcategory form, so that subkategori tampil dengan visual
142. As a customer browsing, I want to see product foto di katalog kategori & detail, and subcategory foto badge, so that produk lebih jelas

### System (v1.0)

104. As the system, I need to verify payment gateway webhook signatures (HMAC), so that fraudulent callbacks are rejected
105. As the system, I need to process payment webhooks idempotently (check if already processed), so that duplicate callbacks don't cause double processing
106. As the system, I need to process late payment webhooks even when the local transaction status is already expired, so that users who paid just before/after the expiry deadline are not denied fulfillment
107. As the system, I need to auto-retry failed distributor calls up to 3 times with exponential backoff (30s, 2m, 5m), so that transient failures are recovered automatically
108. As the system, I need to log each distributor attempt with structured fields (http_status_code, duration_ms, status, error_message), so that monitoring and debugging are possible
109. As the system, I need to send email notifications on every transaction status transition, so that users stay informed
110. As the system, I need to rate-limit checkout requests (per IP, per user, per product+target_game_id combo), so that abuse and bots are prevented
111. As the system, I need to whitelist payment gateway IPs at the firewall level, so that webhook endpoints are not exposed to the public internet
112. As the system, I need to calculate different prices based on user role (customer vs reseller), so that each segment pays the correct amount

### System (v1.1)

113. As the system, I want `User` `#[Fillable]` to exclude `role`, so that no one can self-assign `role=admin` via mass assignment
114. As the system, I want `AdminSeeder` and `app:create-admin` to set `role` via direct assignment, not mass-assign, so that seeding works despite guarded `role`
115. As the system, I want `ref_id` generation to be race-safe (catch `23000` duplicate and retry up to 3 `DB::transaction` attempts), so that concurrent checkouts do not 500
116. As the system, I want `ref_id` entropy 8 chars (`FOFA-YYYYMMDD-XXXXXXXX`, env `FOFA_REF_ID_LENGTH`), so that invoice public token cannot be brute-forced (2e11 combos/day vs 456k)
117. As the system, I want `GET /invoice/{ref_id}` to stay public for guests but be protected by the higher entropy, so that UX stays frictionless without IDOR
118. As the system, I want `transactions.expired_at` + scheduler `everyMinute` to expire `pending` past deadline, so that unpaid invoices do not pile up
119. As the system, I want `SyncProductPricesService` deduplicated (no duplicate `sync`/`syncProduct` logic), so that code smell `Middle Man` is reduced

---

## Implementation Decisions

### v1.0 — Database Schema

10 tables (+ hardening MVP):

- `users` — extend existing: add `role` enum (customer/reseller/admin)
- `reseller_profiles` — 1-1 to users: api_key, api_secret, markup_percentage, callback_url, status
- `categories` — dynamic: name, slug, icon_url, display_order, is_active
- `products` — FK to categories: name, price, cost_price, is_active, description (nullable)
- `product_vendor_mappings` — FK to products: vendor, vendor_product_code, vendor_price
- `transactions` — ref_id (unique), user_id (nullable for Guest), product_id, target_game_id, target_server_id (nullable), amount, status, expired_at, is_reconciliation, reconciled_at, reconciliation_note, refund fields
- `payments` — 1-N to transactions: method, gateway_ref, status, signature_verified, paid_at
- `distributor_logs` — FK to transactions: provider, attempt_number, request/response payload, http_status_code, duration_ms, status, error_message
- `wallets` — FK to users (reseller only): balance, updated_at
- `wallet_transactions` — FK to wallets: type (deposit/deduction/refund), amount, reference_id (nullable), notes (nullable)

### v1.0 — Transaction Status Flow

```
pending ─→ expired (telat/tidak bayar)
  │
  └─→ paid → processing → waiting_fulfillment ─→ success
                                                    │
                                                    └─→ failed
```

Dua titik percabangan:
1. `pending` → `paid` (bayar tepat waktu) ATAU `expired` (telat/tidak bayar)
2. `waiting_fulfillment` → `success` ATAU `failed` (distributor gagal setelah 3x retry)

### v1.0 — Pricing Logic

- Customer: `products.price`
- Reseller: `cost_price + (cost_price × markup_percentage)`
- Implementation: `CalculatePriceForUser` service/method

### v1.0 — Payment Webhook Handler

1. Verify signature (HMAC) — reject & log if invalid
2. Find transaction by ref_id inside single `DB::transaction` with `lockForUpdate`
   - 2a. Jika status `pending` → proses normal (update payments.status ke `paid`, trigger distributor queue job)
   - 2b. Jika status `expired` tapi gateway konfirmasi `PAID` → late-payment reconciliation
   - 2c. Jika status `paid`/`processing`/`waiting_fulfillment`/`success` → idempotent, return success tanpa proses ulang
   - 2d. Jika status `failed` → tolak, log sebagai late-payment-after-failure
3. Return `{ "success": true }` ke gateway

### v1.0 — Distributor Retry Strategy

- 3x auto-retry with exponential backoff (30s → 2m → 5m)
- Use Laravel Queue `backoff()` method
- After 3 failures → status `failed`, fallback to manual admin action

### v1.0 — Rate Limiting

- Checkout: per IP (5/menit) + per user (10/jam) + per product+target_game_id (max 3 per 10 min for same target) — via custom `CheckoutRateLimit` middleware, hit hanya setelah validasi lolos
- Webhook: no rate limit (signature + idempotency + IP whitelist)
- Admin general: 60 req/menit per user via `RateLimiter::for('admin')`
- Admin login: max 5 failed per 15 min per IP
- Headers: 429 dengan `Retry-After` + `X-RateLimit-Limit`/`X-RateLimit-Remaining`

### v1.0 — Refund

- Customer: manual admin transfer, proactive refund form on invoice page when status = failed
- Reseller: auto-instant refund to `wallets.balance`, logged in `wallet_transactions`

### v1.0 — Reseller API

- Auth: Sanctum `personal_access_tokens`, guard `sanctum`, middleware `auth:sanctum + role:reseller` untuk `prefix('v1')`
- Endpoints: `GET /api/v1/products`, `POST /api/v1/topup`, `GET /api/v1/topup/{ref_id}/status`, `GET /api/v1/balance`
- Callback: `TransactionObserver` dispatch `SendResellerCallbackJob` pada `created` dan `updated` (`wasChanged('status')`), payload `{ref_id,status,amount}`, 3x retry
- Token Management: admin-only via `Admin\ResellerTokenController` — **admin manual provisioning via UI `admin/resellers/{user}/tokens` (create/revoke, `admin/resellers/tokens.blade.php`), BUKAN reseller self-service.** Reseller channel dormant artinya reseller tidak bisa self-register/self-token — admin yang create user + token manual via Tinker/UI. UI ini tidak bertentangan dengan v1.3 scope-cut "no registration UI" (yang dimaksud adalah no reseller self-registration di storefront), dan di-keep karena tanpa UI admin harus via `php artisan tinker` yang error-prone untuk revoke.

### v1.0 — Notification

- Email via Laravel Mail on every status transition: paid, success, failed, expired, reconciliation
- Special notification for late-payment reconciliation cases

### v1.1 — Actors & Auth

- 3 actors canonical: `Guest` (no `user_id`, nullable FK `transactions.user_id`), `Sales/Reseller` (`role=reseller`, WAJIB login, 1-1 `reseller_profiles`), `Admin` (`role=admin`, WAJIB login, `auth+role:admin` middleware, treated as buyer at `cost_price`).
- `RoleMiddleware` tetap variadic `...$roles` + `BackedEnum` + comma-split; routes `admin/*` tetap guarded.

### v1.1 — Pricing Logic (`CalculatePriceForUser`)

- `Guest`/`Customer`/`Reseller` without profile → `products.price` (fallback).
- `Reseller` with profile → `cost_price + cost_price * markup_percentage / 100` (existing logic, WAJIB login).
- `Admin` → **langsung `cost_price`** (`number_format((float)product.cost_price,2)`) tanpa query `reseller_profiles` sama sekali. Opsi 2 dipilih karena: jika promo reseller ubah `markup_percentage` ke 0, admin tidak ikut kena; tabel `reseller_profiles` tetap murni untuk reseller saja.

### v1.1 — Security & Schema

- `users.role` — string column + `UserRole` enum cast; DB enum constraint not added (keep `string` to avoid migration churn on sqlite). Fillable now `['name','email','password']` only; `role` set via `$user->role = UserRole::Admin` in seeder/command.
- `transactions.ref_id` — `unique` index stays; generation `FOFA-YYYYMMDD-{random}` now `Str::random(length)` where `length = config('fofa.checkout.ref_id_length',8)` (env `FOFA_REF_ID_LENGTH`). `TransactionFactory` aligned to same length.

### v1.1 — Checkout & Invoice

- `CheckoutController@store`: validate `product.is_active` + `category.is_active`; `CalculatePriceForUser` per actor; `expired_at = now()->addMinutes(config FOFA_INVOICE_EXPIRY_MINUTES)`; `DB::transaction` wrapped in retry loop catching `QueryException` with `23000` + `ref_id` in message, max 3 attempts before throw.
- `InvoiceController@show`: public via `ref_id` unchanged, security now via entropy.

### v1.1 — Config

- `config/fofa.php`: `checkout.expiry_minutes` + `checkout.ref_id_length` (int, env `FOFA_REF_ID_LENGTH`, default 8) + `payment_methods` 6 opsi. `.env.example` updated.

### v1.1 — Code Smells Cleaned

- `SyncProductPricesService`: keep `sync()` canonical, `syncProduct()` as alias with comment (not duplicate logic). Pint fixes applied.
- `PaymentWebhookController::handle()` refactored from ~220 line monolith to ~80 line orchestrator + 8 private methods. No behavior change.
- `TransactionMail` abstract base class created; `TransactionPaidMail` & `TransactionReconciliationMail` extend it (removed duplication).
- CSRF except centralized in `bootstrap/app.php` only (removed duplicate in routes).

### v1.2 — Database Changes

**14 new tables:**
- `vouchers` — code (unique), type (percentage/fixed), value, min_order_amount, max_uses, used_count, starts_at, expires_at, is_active, created_by
- `voucher_usages` — voucher_id, user_id, transaction_id, discount_amount
- `click_logs` — referrer_api_key, product_id, category_id, ip_address, user_agent, converted, transaction_id
- `reseller_stores` — user_id (unique), slug (unique), store_name, logo_url, theme_color, is_active
- `audit_logs` — user_id, action, auditable_type, auditable_id, old_values (json), new_values (json), ip_address, user_agent
- `announcements` — title, content, type (info/maintenance/warning), target_audience (all/customer/reseller/admin), is_active, published_at, created_by
- `notifications` — user_id, title, message, read_at
- `chat_rooms` — user_id, admin_id, status (open/waiting/closed), closed_at
- `chat_messages` — room_id, sender_id, message
- `fraud_alerts` — user_id, transaction_id, rule_type, severity (low/medium/high), details (json), resolved_at, resolved_by
- `fraud_rules` — rule_type (unique), label, threshold, window_minutes, is_active, config (json)
- `admin_payment_methods` — method_code (unique), display_name, is_active, sort_order
- `product_deletion_logs` — product_id, product_snapshot (json), deleted_by, reason, deleted_at
- `reseller_monthly_reports` — user_id, month, year, total_transactions, total_revenue, total_commission

**2 alter tables:**
- `users` — add `owner` to role enum
- `products` — add `deleted_at` (nullable, soft delete)

### v1.2 — Enums

- `UserRole` — add `Owner` value
- `VoucherType` — new enum: `Percentage`, `Fixed`
- `AnnouncementType` — new enum: `Info`, `Maintenance`, `Warning`
- `AnnouncementAudience` — new enum: `All`, `Customer`, `Reseller`, `Admin`
- `ChatRoomStatus` — new enum: `Open`, `Waiting`, `Closed`
- `FraudSeverity` — new enum: `Low`, `Medium`, `High`

### v1.2 — Services

- `VoucherService` — apply voucher code, validate constraints, calculate discount, atomically update used_count
- `GameIdVerifier` — contract with Digiflazz implementation + fallback validator; chain pattern: try distributor first, fallback to format validation
- `ClickTrackingService` — log UTM clicks, attach conversion on checkout success
- `AuditLogService` — record admin/owner actions with old/new values
- `FraudDetectionService` — run rule checks on each transaction, generate alerts
- `BroadcastService` — create in-app notifications + email blast for target audience
- `ChatService` — manage chat rooms, send messages, real-time broadcast via Reverb

### v1.2 — Jobs

- `GenerateMonthlyReportsJob` — scheduled 1st of each month, compute reseller commission reports
- `FraudCheckJob` — async fraud rule evaluation per transaction
- `BroadcastEmailJob` — queue email blast for announcements
- `SendChatMessageJob` — broadcast chat message via Reverb WebSocket

### v1.2 — Middleware

- `TrackReferrer` — capture `?ref={api_key}` on storefront routes, log to click_logs, share via session
- Existing `RoleMiddleware` — updated to support `owner` role

### v1.2 — Routes

**New public routes:** `GET /search`, `GET /cek-nickname`, `GET /toko/{slug}`, `GET /chat`

**New auth routes:** `GET /history`, `GET /history/{ref_id}`, `POST /chat/messages`, `POST /chat/close`

**New admin routes:** `GET /admin/payment-methods`, `PUT /admin/payment-methods/{id}/toggle`, `GET/POST /admin/announcements`, `GET /admin/audit-logs`, `GET /admin/fraud-alerts`, `POST /admin/fraud-alerts/{id}/resolve`, `GET /admin/chat`, `POST /admin/chat/{room}/assign`

**New owner routes:** `GET/POST/PUT/DELETE /owner/vouchers`, `GET /owner/reports/finance`, `GET /owner/reports/products`, `GET /owner/reports/vendors`, `GET /owner/reports/tax`, `POST /owner/reports/export`, `GET /owner/admins`, `PUT /owner/admins/{user}/role`, `DELETE /owner/products/{id}`, `GET/PUT /owner/settings`

**New API routes:** `GET /api/v1/analytics/clicks`, `GET /api/v1/store`, `PUT /api/v1/store`

### v1.2 — Key Behavioral Decisions

**Product deletion rules (3-tier):**
1. Has transactions → locked (cannot delete, only soft-delete + snapshot to deletion_logs)
2. No transactions + ≤24h → owner-only hard delete
3. No transactions + >24h → soft delete only

**Voucher system:**
- Scope: global (all products)
- Types: percentage (0-100%) and fixed (nominal amount)
- Owner-only CRUD
- One voucher per transaction, one use per user per voucher
- Discount cannot make amount negative (minimum Rp1)

**Click tracking:**
- UTM parameter `?ref={api_key}` on any storefront URL
- Logged: IP, user_agent, product_id, category_id
- Conversion tracked at checkout success when `?ref` is present in session
- Conversion rate = (conversions / clicks) × 100

**Live chat flow:**
- User opens chat → `chat_rooms` created with status `waiting`
- Admin assigns self → status `open`
- Real-time messaging via Laravel Reverb channel `chat.{room_id}`
- Either party closes → status `closed`

**Fraud rules (medium complexity):**
- Rate limit: >5 transactions from same IP/user in 10 minutes
- Price anomaly: reseller transaction with no markup
- Late night: transaction between 2am-4am
- Changing game IDs: >3 different target_game_ids in 1 hour
- All checks run async via FraudCheckJob

### v1.2 — New Dependencies

- `laravel/reverb` — WebSocket server for live chat
- `maatwebsite/excel` — Export CSV/Excel
- `barryvdh/laravel-dompdf` — Export PDF

### v1.3 — Scope-cut Rationale

Community-scale top-up shop — fitur yang tidak kritis untuk launch di-drop atau dormant:

**Dropped issues (8):**
- Click Logs/UTM — no referral tracking needed
- Reseller Store — no white-label store
- Click Analytics — no conversion tracking
- Reseller Monthly Reports — dormant
- Live Chat/Reverb — no real-time chat
- Fraud Detection — no fraud rules
- Finance Reports/Export (standalone) — simplified to PDF from admin transactions page (#25)
- Reseller API Docs — dormant

**Simplified issues (3):**
- #15 Audit Log — table + service + observer stay, no Livewire viewer UI (data queryable via Tinker/DB)
- #25 PDF Export — from `/admin/transactions` page, not standalone `/owner/reports/*` dashboard. **Expanded in v1.3.1:** now includes transaction chart, period-based filtering, and query-level financial data access control (ADR-015).
- #26 Owner Settings — drop `/owner/settings` (use .env), keep `/owner/admins` as simple Livewire UI (add admin + toggle active/inactive)

**Dropped dependencies (2):**
- `laravel/reverb` — removed entirely (live chat dropped)
- `maatwebsite/excel` — removed entirely (CSV/Excel export dropped)

**Kept dependency (1):**
- `barryvdh/laravel-dompdf` — for PDF export from admin transactions page

### v1.3 — Category Status Enum

- New `CategoryStatus` enum: `Active`, `ComingSoon`
- `categories` table: add `status` string column (default `coming_soon`), NOT native DB enum (same pattern as `users.role`)
- `is_active` boolean stays — dual mechanism:
  - `is_active = false` → hidden everywhere (admin can fully disable)
  - `is_active = true + status = 'coming_soon'` → visible in storefront, buy disabled, "Segera Hadir" badge
  - `is_active = true + status = 'active'` → fully functional
- `Category` model: add `status` to fillable + casts

### v1.3 — Storefront Behavior (Coming Soon)

- `StorefrontController::index()` — show ALL categories (active + coming_soon), ordered by `display_order`
- `StorefrontController::category()` — show category page; if `status === 'coming_soon'`: show products but disable buy button + "Segera Hadir" badge
- `StorefrontController::show()` — 404 if category is `coming_soon` (can't view individual product details)
- Checkout guard — reject if product category is `coming_soon`

### v1.3 — Minecraft Manual Fulfillment

- `FulfillTransactionJob` — new branching: if `product.category.slug === 'minecraft'`, skip distributor API entirely, leave status at `waiting_fulfillment`
- No retry logic for Minecraft (no API to call, no exponential backoff)
- Admin endpoint: `POST /admin/transactions/{transaction}/mark-manual-success`
  - Guard: transaction must be `waiting_fulfillment` AND product category must be `minecraft`
  - Update status to `success`, set `completed_at`, dispatch email notification
  - Add audit log entry (`action: 'manual_success'`)
- Vendor mapping: `minecraft_server` vendor type in `product_vendor_mappings` — metadata only, no API call triggered
- `vendor_product_code` = server-specific item code (admin fills manually)
- `vendor_price` = cost to admin (if any)

### v1.3 — Checkout Guest Email

- Guest checkout: `guest_email` validated `required|email` on checkout form
- Logged-in user: `guest_email` not shown, `users.email` used for notifications
- `Transaction` model: `guest_email` nullable (DB level), required at form validation level for guest path
- Email notification: use `guest_email` if set, else fall back to `users.email`

### v1.3.1 — Financial Data Access Control

**Rationale (ADR-015):**
Admin bertanggung jawab atas operasional harian (retry, fulfillment manual, monitoring status) — butuh visibilitas penuh atas produk dan status transaksi. Data finansial sensitif (nominal, omset, laporan) dibatasi hanya untuk Owner sebagai pemegang keputusan bisnis tertinggi.

**Query-level column exclusion:**
- Transaction model: `scopeForAdmin()` exclude kolom `amount` dari SELECT query
- Transaction model: `scopeForOwner()` no-op (default behavior, semua kolom)
- Livewire components dan controllers memanggil scope berdasarkan `auth()->user()->isAdmin()` / `isOwner()`
- Blade views menerima flag `$isOwner` untuk conditional rendering

**Ini BUKAN cuma UI hide:**
- Kolom `amount` tidak di-select dari database untuk admin
- Data tidak ada di response Livewire, network tab, atau inspect element
- Product price (`products.price`, `products.cost_price`) TIDAK dibatasi — admin butuh untuk operasional manage produk

**Access matrix:**

| Data | Admin | Owner |
|------|-------|-------|
| Grafik transaksi (count) | ✅ | ✅ |
| Nama produk, kategori, game ID, status | ✅ | ✅ |
| Log distributor | ✅ | ✅ |
| Transaction amount | ❌ | ✅ |
| Total omset | ❌ | ✅ |
| Export PDF | ❌ | ✅ |

### v1.3.1 — Transaction Chart

- Line chart via Chart.js (CDN, bukan npm) — project ini pakai Vite vanilla
- 3 mode periode: Harian (per hari), Mingguan (per minggu), Bulanan (per bulan)
- Data: `selectRaw('DATE(created_at) as date, count(*) as total')` grouped by period
- Chart count-based (aman untuk admin — tidak ada data finansial)
- Filter periode mempengaruhi chart DAN tabel transaksi
- `<canvas id="txChart">` + inline Chart.js script di Livewire Blade

### v1.3.1 — PDF Export (Owner-Only)

- Route: `POST /admin/transactions/export-pdf` (owner-only guard di controller)
- Tombol "Export PDF" hanya muncul untuk owner (`@if($isOwner)`)
- PDF mengikuti filter periode yang sedang aktif di grafik
- Service: `PdfExportService` — generate PDF dari collection transactions via DomPDF
- Template: header (Fofa Shop, periode), tabel transaksi per grup, summary (total omset, total transaksi)
- Dependencies: `barryvdh/laravel-dompdf`

### v1.3.1 — Routes Changed

**Added in v1.3.1:**
- `POST /admin/transactions/export-pdf` — PDF export (owner-only, di dalam admin route group tapi dengan guard `abort_unless(isOwner, 403)`)

### v1.3.4 — Design System & Theme (Doc-Only, Implementation Already Exists)

**Tidak ada perubahan kode** — section ini hanya mendokumentasikan Hallmark overhaul dan dark mode yang sudah di-implementasi di log Hallmark internal (tidak dipublikasikan), tokens.css, resources/css/app.css, dan theme-toggle. Detail:

**Hallmark:**
- `tokens.css` — OKLCH dual custom-dual (Chakra Petch + Plus Jakarta Sans + VT323), accent <5% vibrant, subtle blocky 8px, shadow light/dark, badge tokens WCAG 4.5:1, contrast gate 40-41 pass.
- `app.css` — Catalogue macrostructure, N7 Brutal slab + Ft8 Marquee, F6 grid 4→1, fofa-card hover lift + press feedback, card-enter stagger 40ms, modal/dropdown easing, reduced-motion guard, overflow-x:clip, focus instant 3:1.
- Components — `brand-lockup` (pill MINECRAFT + tagline), `application-logo` (slab rect rx8 FS monogram handle DNA), `theme-toggle` 8-state, `transaction-status-badge` 6-state, `modal` + `dropdown` + `action-message` microinteractions.
- Tokens — dual light (paper 98.2% / accent 58%) + dark (paper 16% / accent 62% warm), semantic `--color-*`, motion `--ease-out 0.25,1,0.5,1` + `--dur-*`, radii `--radius-card 8px`, spacing `--space-*`.

**Dark Mode:**
- `tokens.css` dual block `html[data-theme="dark"]` + `html.dark` + `@media (prefers-color-scheme)` fallback, 2 modes LIVE ikut system (localStorage `fofa-theme` > `prefers-color-scheme`), contrast 14.8:1 / 7.1:1 / 7.3:1 pass.
- `tailwind.config.js` `darkMode: class` + `colors: var(--color-*)`, `app.js` controller (localStorage + system listener + MutationObserver Livewire), FOUC guard inline script, body transition 240ms.

**Behaviour:** Pure presentational — tidak ada perubahan business logic checkout/invoice/distributor. Deskripsi "Minecraft subtle blocky" di-update menjadi "Minecraft white-red blocky dual (OKLCH light 98.2%+58% / dark 16%+62% warm, slab rx8, pixel-cube Tier-A subtle, Catalogue, N7+Ft8)".

### v1.3 — Dropped Features (from v1.2)

**Click tracking:** No `click_logs` table, no `TrackReferrer` middleware, no `ClickTrackingService`, no analytics dashboard.

**Reseller store:** No `reseller_stores` table, no `/toko/{slug}` route, no reseller store Livewire components.

**Live chat:** No `chat_rooms`/`chat_messages` tables, no `ChatService`, no `SendChatMessageJob`, no `laravel/reverb`.

**Fraud detection:** No `fraud_alerts`/`fraud_rules` tables, no `FraudDetectionService`, no `FraudCheckJob`.

**Finance reports (standalone):** No `/owner/reports/*` routes, no report Livewire components. PDF export from admin transactions page (#25) instead.

**Reseller monthly reports:** No `reseller_monthly_reports` table, no `GenerateMonthlyReportsJob`.

**Reseller API docs:** No documentation page, dormant.

### v1.3 — Routes Changed

**Removed from v1.2:**
- `GET /toko/{slug}` — reseller store
- `GET /chat`, `POST /chat/messages`, `POST /chat/close` — live chat
- `GET /admin/audit-logs` — audit log viewer
- `GET /admin/fraud-alerts`, `POST /admin/fraud-alerts/{id}/resolve` — fraud
- `GET/POST /admin/chat`, `POST /admin/chat/{room}/assign` — chat admin
- `GET /owner/reports/finance`, `GET /owner/reports/products`, `GET /owner/reports/vendors`, `GET /owner/reports/tax`, `POST /owner/reports/export` — standalone reports
- `GET /owner/settings`, `PUT /owner/settings` — global settings
- `GET /api/v1/analytics/clicks`, `GET /api/v1/store`, `PUT /api/v1/store` — reseller analytics/store

**Added in v1.3:**
- `POST /admin/transactions/{transaction}/mark-manual-success` — Minecraft manual fulfillment

---

## Testing Decisions

### v1.0

- Test external behavior only (HTTP responses, database state), not implementation details
- Key seams to test:
  - Auth role-based access (customer can't access admin, reseller can't access storefront admin)
  - Transaction status transitions (each valid and invalid transition)
  - Payment webhook signature verification (valid, invalid, duplicate)
  - Late-payment reconciliation (expired transaction receiving late payment)
  - Distributor retry logic (success on attempt 1, success on attempt 3, all fail)
  - Price calculation per role (customer price vs reseller price)
  - Rate limiting behavior (checkout abuse, login brute-force)
- Prior art: existing `tests/Feature/Auth/` and `ProfileTest.php` patterns

### v1.1

- Only test external behavior (HTTP, DB state), not implementation details.
- Prior art: `tests/Feature/CheckoutInvoiceTest.php` (21 tests), `RoleMiddlewareTest.php` (10 tests), `CategoryProductCrudTest.php`.
- New/Updated seams:
  - `User` mass-assignment: `User::create(['role'=>admin])` must not elevate
  - `CalculatePriceForUser` per actor: guest 50000.00, customer 50000.00, reseller 10% 44000.00, reseller no profile 50000.00, admin 40000.00 (cost), adminWithProfile 5% still 40000.00 (ignore profile)
  - `Checkout` race: ref_id unique + format assertion
  - `Invoice` entropy: `FOFA-YYYYMMDD-{8}` regex
- All after hardening: `php artisan test` 98 passed (333 assertions), `vendor/bin/pint --test` passed.

### v1.2

**What makes a good test:** Test external behavior (HTTP responses, database state changes, job dispatches) — not internal implementation details.

**Modules to test:**
- VoucherService: apply valid/expired/used-up vouchers, discount calculation
- GameIdVerifier chain: successful check, unsupported game fallback
- ClickTrackingService: log click, attach conversion
- AuditLogService: log create/update/delete with old/new values
- FraudDetectionService: each rule triggers correctly, alert generation
- Chat room lifecycle: create → assign → message → close
- Product deletion rules: 3-tier logic
- Payment method toggle: checkout filters active methods only

**New test files needed:**
- `tests/Feature/VoucherTest.php`
- `tests/Feature/ClickTrackingTest.php`
- `tests/Feature/GameIdVerificationTest.php`
- `tests/Feature/AuditLogTest.php`
- `tests/Feature/FraudDetectionTest.php`
- `tests/Feature/ChatTest.php`
- `tests/Feature/ProductDeletionTest.php`
- `tests/Feature/PaymentMethodToggleTest.php`
- `tests/Feature/BroadcastTest.php`
- `tests/Feature/OwnerReportsTest.php`
- `tests/Feature/ResellerStoreTest.php`
- `tests/Feature/CustomerHistoryTest.php`

### v1.3

**Dropped test files (features removed in v1.3):**
- `tests/Feature/ClickTrackingTest.php` — click tracking dropped
- `tests/Feature/ChatTest.php` — live chat dropped
- `tests/Feature/FraudDetectionTest.php` — fraud detection dropped
- `tests/Feature/ResellerStoreTest.php` — reseller store dropped
- `tests/Feature/OwnerReportsTest.php` — standalone reports dropped

**Kept test files (features still in scope):**
- `tests/Feature/VoucherTest.php` — voucher system unchanged
- `tests/Feature/GameIdVerificationTest.php` — game ID verification unchanged
- `tests/Feature/AuditLogTest.php` — audit log table + service stay (test DB writes, not UI)
- `tests/Feature/ProductDeletionTest.php` — product deletion rules unchanged
- `tests/Feature/PaymentMethodToggleTest.php` — payment toggle unchanged
- `tests/Feature/BroadcastTest.php` — broadcast announcements unchanged
- `tests/Feature/CustomerHistoryTest.php` — customer history unchanged

**New test files for v1.3:**
- `tests/Feature/CategoryStatusTest.php` — coming_soon display, buy disabled, 404 on product detail, storefront shows all categories
- `tests/Feature/MinecraftManualFulfillmentTest.php` — FulfillTransactionJob skips API for minecraft category, admin mark-manual-success endpoint, email notification on manual success
- `tests/Feature/AdminPdfExportTest.php` — PDF generation from filtered transactions, categorization per day/week/month
- `tests/Feature/GuestEmailCheckoutTest.php` — guest checkout requires email, logged-in user uses account email

### v1.3.1

**Updated test files:**
- `tests/Feature/AdminPdfExportTest.php` — expand: PDF only accessible by owner (admin gets 403), PDF follows active period filter, PDF contains amount data

**New test files:**
- `tests/Feature/FinancialDataAccessTest.php` — admin transaction queries exclude `amount` column, owner queries include `amount`, chart data is count-only (no financial data), export PDF returns 403 for admin
- `tests/Feature/TransactionChartTest.php` — chart data endpoint returns correct groupings per period (daily/weekly/monthly), date range filtering works

---

## Out of Scope

### v1.0

- Multi-vendor automatic failover (manual for MVP)
- Mobile app (headless API architecture for future)
- Reseller registration flow (admin creates reseller accounts)
- Reseller token self-service API (admin-only via `Admin\ResellerTokenController`)
- Payment gateway refund API integration (manual for MVP)
- Real-time notifications (email only for MVP)
- Advanced analytics/reporting dashboard

### v1.1

- Payment webhook (06), distributor fulfillment retry (07), admin transaction management (08), refund (09), reseller API Sanctum (10), wallet deduction/refund (11), rate limiting (12) — tetap di ticket masing-masing.
- `GET /invoice` IP whitelist / signed URL — not needed for MVP, entropy 8-char deemed sufficient.
- Admin wallet/balance — admin bonus is price discount, not wallet.
- Changing `users.role` column to native DB `enum` type — deferred (needs doctrine/dbal, sqlite vs mysql).

### v1.2

- WhatsApp notifications (email only for MVP, WhatsApp later)
- Subdomain/custom domain white-label (custom path `/toko/{slug}` only)
- Real-time fraud ML models (rule-based only)
- OpenAPI/Swagger documentation (static Blade docs only)
- Mobile app (web only)
- Multi-language/i18n
- Third-party live chat (custom Reverb build)
- Per-category/per-product voucher scope (global only for MVP)
- Automatic vendor failover (manual retry only, per ADR-0006)

### v1.3

- Click tracking, reseller store, live chat (Reverb) — dropped, not needed for community-scale launch
- Fraud detection (rule-based) — dropped, manual monitoring sufficient
- Standalone finance reports dashboard (`/owner/reports/*`) — dropped, PDF export from admin transactions page
- CSV/Excel export (`maatwebsite/excel`) — dropped
- Tax calculation (PPN 11%) — dropped
- Global settings UI (`/owner/settings`) — use .env instead
- RCON/automated Minecraft fulfillment — manual only for MVP
- `laravel/reverb` — removed entirely
- `maatwebsite/excel` — removed entirely

---

## Further Notes

- PRD original ada di `PRD.md` — spec ini adalah evolusi dari diskusi grilling session
- Semua keputusan arsitektur terdokumentasi di ADR-0001 sampai ADR-0015
- Domain glossary ada di `CONTEXT.md` — gunakan istilah dari glossary saat implementasi
- v1.0 issues #01-#12: all done
- v1.1 repair: all done (98 tests, pint passed)
- v1.2 issues #14-#33: all `ready-for-agent` di issue tracker internal (tidak dipublikasikan)
- v1.3 scope-cut: 8 issues dropped (click tracking, reseller store, click analytics, reseller monthly reports, live chat, fraud detection, standalone reports, reseller API docs), 3 simplified (#15, #25, #26), Minecraft-first added (#27)
- v1.3 renumbered issues: #20-#27 (sequential after drop)
- v1.3 tagline: **Toko Top-Up Skala Komunitas Minecraft**
- v1.3.1: #25 expanded — transaction chart + PDF export + financial data access control (ADR-015). Query-level column exclusion ensures admin cannot access financial data even via inspect element.
- v1.3.4: doc-only sync — Hallmark overhaul + dark mode documentation. No code changes.
- v1.3.5: Minecraft animations — mascot creature + scroll fling, 6 floating particles, blocky button, subtle grid BG, green mc-green tokens (mascot/particles only), theme toggle in hero. All gates pass (10, 11, 27, 48).
- v1.1 keputusan `reseller_profiles` vs `admin` (opsi 2) logged di issue tracker internal (komentar FINAL 2026-08-26, tidak dipublikasikan) dan di `app/Services/CalculatePriceForUser.php:11` docblock
