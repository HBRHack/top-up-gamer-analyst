# TOP-UP-GAMER-ANALYST

**Analysis & System Design Portfolio — Simulasi Operasional Toko Top-Up Game**

| | |
|---|---|
| **Status proyek** | Analysis-first / design portfolio |
| **Bentuk** | Dokumentasi analisis, requirement, data model, business rules, process flow, API contract, dan review arsitektur |
| **Implementasi aplikasi** | Bukan fokus utama repository ini — tersedia sebagai jalur implementasi yang direkomendasikan |

---

## 1. Apa sebenarnya proyek ini?

**TOP-UP-GAMER-ANALYST** adalah simulasi analisis sistem untuk toko top-up game (voucher, diamond, game currency) dengan model penjualan H2H ke reseller, menggunakan Minecraft sebagai kategori launch pertama.

Repository ini sengaja dibuat **analysis-first**.

Saya tidak menjadikan pembuatan aplikasi sebagai tujuan utama. Yang ingin ditunjukkan adalah bagaimana sebuah masalah bisnis diterjemahkan menjadi:

- kebutuhan bisnis dan requirement;
- aktor, hak akses, dan tanggung jawab;
- master data dan relasi data;
- business rules;
- process flow dan state machine;
- hubungan antar-modul;
- kebutuhan non-fungsional;
- asumsi dan constraint;
- kontrak antar-sistem (API contract);
- serta rekomendasi teknologi untuk implementasi.

Dengan pendekatan ini, interviewer dapat membaca cara saya berpikir sebagai analis/system designer, tanpa harus terlebih dahulu menjalankan program.

---

## 2. Repository ini ditujukan kepada siapa?

**Untuk interviewer / hiring team**

- IT Business Analyst
- System Analyst
- Solution Architect
- System / Software Architecture track
- posisi IT yang berhubungan dengan transformasi proses bisnis

**Untuk technical interviewer**

Bagian yang paling relevan adalah model data, state machine transaksi, business rules pricing/pembayaran, integrasi payment gateway–distributor–wallet, kontrak API reseller, serta alasan pemilihan teknologi.

**Untuk pembaca non-teknis**

Tidak perlu memahami Laravel untuk mengikuti proyek ini. Mulai dari bagian **Project Overview**, **Operational Flow**, dan **Business Rules**.

> **Catatan penting:** ini adalah proyek latihan dan portofolio, bukan representasi sistem produksi
> dan bukan klaim bahwa seluruh prosedur bisnis telah dimodelkan secara lengkap.
> Seluruh kredensial, domain, dan detail sensitif dalam dokumen telah disamarkan untuk publikasi.

---

## 3. Kenapa tidak langsung membuat program?

Karena pada proyek ini program bukan titik utama yang ingin saya demonstrasikan.

Sebuah aplikasi dapat dibuat dari requirement yang sudah tersedia, tetapi nilai analitisnya justru terlihat **sebelum** kode ditulis: apa masalahnya, siapa yang bertanggung jawab, data apa yang diperlukan, kapan status berubah, apa yang tidak boleh terjadi, dan bagaimana modul saling memengaruhi.

Jadi repository ini diposisikan sebagai:

> "Bukti bahwa saya mampu memetakan masalah bisnis menjadi rancangan sistem yang dapat diimplementasikan."

Implementasi tetap dipikirkan sejak awal, tetapi diperlakukan sebagai **jalur implementasi yang direkomendasikan**, bukan sebagai pusat portofolio.

---

## 4. Project Overview

### Domain

Simulasi operasional toko top-up game, dengan enam area utama:

| Area | Cakupan |
|---|---|
| **Storefront & Checkout** | katalog game, pencarian, cart-less checkout, invoice via `ref_id`, refund request |
| **Payment Gateway** | integrasi dual-vendor (Tripay / Duitku), webhook ber-signature, late-payment reconciliation |
| **Distributor Fulfillment** | integrasi Digiflazz via queue dengan retry, fulfillment manual untuk Minecraft |
| **Admin Panel** | produk, transaksi, pengguna, reseller, voucher, audit log, export PDF |
| **Reseller API** | endpoint H2H ber-auth Sanctum, pricing markup, callback outbound |
| **Wallet & Keuangan** | saldo reseller, atomic deduction, akses data finansial terbatas (admin vs owner) |

### Tujuan

Proyek memiliki dua tujuan yang berjalan bersama:

- **Latihan** — memahami desain relasi data, state machine, business rules, kontrak API, dan integrasi antarmodul.
- **Portofolio** — menyediakan artefak analisis yang dapat dibedah saat interview.

### Skala dokumen

| Keluarga id | Jumlah | Sumber |
|---|---|---|
| FR (Functional Requirements) | 38 | `docs/analyst/SRS.md` |
| UC (Use Case) | 23 | `docs/analyst/use-cases/` |
| BR (Business Rules) | 38 (14 kategori) | `docs/analyst/business-rules.md` |
| NFR | 14 | `docs/analyst/nfr.md` |
| PF (Process Flow) | 11 | `docs/analyst/process-flows.md` |
| GAP / CON / ASM / OQ | 12 / 5 / 4 / 3 | `nfr.md`, `assumptions-constraints.md` |
| Endpoint (E1–E6) | 6 | `docs/analyst/api-contract.md` |

---

## 5. Operational Flow / State Machine

### State machine transaksi

```
                        ── telat / tidak bayar ──► expired
                       │
pending ─── (webhook PAID)
                       │
                       └──► paid ──► processing ──► waiting_fulfillment ──┬──► success
                                                                        │
                                                                        └──► failed
                                                                         (distributor gagal
                                                                          setelah 3x retry)
```

Representasi ringkas

```
pending
  ├── expired                      (telat / tidak bayar)
  └── paid
       └── processing
            └── waiting_fulfillment
                 ├── success
                 └── failed        (retry distributor habis)
```

### Titik percabangan

1. `pending` → `paid` (bayar tepat waktu) **atau** `expired` (telat/tidak bayar)
2. `waiting_fulfillment` → `success` **atau** `failed` (distributor gagal setelah 3x retry)

### Prinsip state machine

Status transaksi **tidak diperlakukan sebagai label bebas** yang dapat diganti sembarang role. Perubahan status mengikuti event dan business rule:

- Webhook diterima → verifikasi signature dulu, baru status boleh bergerak.
- Webhook masuk saat status sudah `paid`/`processing`/`success` → **idempotent**, tidak diproses ulang.
- Webhook `PAID` masuk saat status sudah `expired` → **late-payment reconciliation**, bukan ditolak buta.
- Fulfillment Minecraft → berhenti di `waiting_fulfillment`, admin menyelesaikan manual lalu menandai `success`.
- Setelah `success`, tidak ada transisi balik.

Aturan ini menjadi salah satu inti desain: **status harus konsisten dengan pembayaran dan kondisi fulfillment**, bukan dengan keinginan role tertentu.

---

## 6. Business Rules Utama

### Pricing

| Role | Harga yang dibayar |
|---|---|
| Guest / Customer | `products.price` |
| Reseller | `cost_price + (cost_price × markup_percentage)` |
| Admin / Owner | `cost_price` langsung (tanpa markup) |

Pricing dihitung melalui satu primitive bersama (`CalculatePriceForUser`) sehingga storefront dan API tidak punya dua logika harga yang bisa berbeda.

### Identitas transaksi

- Format `ref_id`: `FOFA-YYYYMMDD-XXXXXXXX` (entropy 8 karakter, panjang bisa di-set via env).
- `ref_id` dibuat **di server**, unique, dan menjadi kunci korelasi tunggal di seluruh endpoint.

### Pembayaran

- Verifikasi signature webhook memakai HMAC-SHA256 + perbandingan timing-safe.
- Proses webhook berada dalam satu `DB::transaction` dengan `lockForUpdate` → mencegah double-processing.
- Webhook yang sama yang datang dua kali tidak mengubah apa pun (idempotency).

### Checkout & Rate Limit

Rate limit tiga dimensi:

| Dimensi | Default |
|---|---|
| Per IP | 5 / 1 menit |
| Per user | 20 / 1 jam |
| Per produk + target | 6 / 10 menit |

### Wallet

- Deduction wallet atomik (`lockForUpdate`) — tidak ada race double-spend.
- Saldo tidak boleh minus.

### Fulfillment

- Produk non-Minecraft → otomatis lewat distributor (Digiflazz) dengan retry 3x exponential backoff.
- Produk Minecraft → **manual**: admin melihat status `waiting_fulfillment`, mengeksekusi in-game, lalu menandai sukses.
- Minecraft punya 3 skenario edisi: **Java** (username 3–16), **Bedrock** (Gamertag 3–12), **Manual** (tanpa game ID, email wajib).
- Akun Minecraft pengganti dapat dikirim ke pembeli via email.

### Keuangan & akses

Pembatasan data finansial dibedakan antara admin dan owner pada titik-titik tertentu:

| Titik akses | Admin | Owner |
|---|---|---|
| Dashboard | count-only | + total omset |
| Refund request | tanpa kolom amount | + amount |
| Wallet reseller | tanpa kolom saldo | + saldo |
| Export PDF transaksi | 403 | boleh |

Pembatasan dilakukan di level query (column exclusion), bukan hanya di level tampilan.
Refund diproses manual oleh admin berdasarkan form rekening yang diisi pembeli di halaman invoice.

---

## 7. Roles & Responsibilities

Empat role pada simulasi. Kanal reseller (API + wallet) tetap ada di kode tetapi **dormant** untuk launch skala komunitas — tidak ada UI registrasi, admin memprovision token secara manual:

| Role | Fokus utama | Login | Harga |
|---|---|---|---|
| **Guest** | browse, checkout tanpa akun, akses invoice via `ref_id` | tidak | `products.price` |
| **Customer** | checkout terlogin, riwayat, voucher, refund | session | `products.price` |
| **Admin** | operasional: produk, transaksi, fulfillment, pengguna | session | `cost_price` (tanpa markup) |
| **Owner** | semua hak admin + omset, laporan keuangan, export PDF/CSV, kelola voucher & role admin | session | `cost_price` (tanpa markup) |

Login memakai email + password. Role-based access (`RoleMiddleware`, route admin memakai `role:admin,owner`) adalah kontrol dasar. Owner didefinisikan sebagai role terpisah di enum (bukan admin dengan flag), dibuat via command artisan `app:create-owner`.

---

## 8. Data yang Dimodelkan

### Contoh data dummy untuk simulasi

- kategori game (Minecraft aktif, lainnya "Coming Soon")
- produk per kategori + subkategori
- user dengan 4 role
- transaksi + pembayaran
- wallet + mutasi
- voucher
- audit log

### Entitas inti

```
users ─┬─< transactions >─┬─ payments
       │                  ├─ wallet_transactions >─ wallets
       └─ reseller_profiles
       
products >─ categories >─ subcategories
transactions >─ distributor_logs
transactions >─ audit_logs (admin actions)
```

### Contoh format data

| Jenis | Format / nilai |
|---|---|
| `ref_id` | `FOFA-20260926-A1B2C3D4` |
| Status transaksi | `pending`, `paid`, `processing`, `waiting_fulfillment`, `success`, `failed`, `expired` |
| Edisi Minecraft | `java`, `bedrock`, `manual` |
| Uang | string 2 desimal tanpa pemisah ribuan (`"50000.00"`) |

Pembahasan lengkap ada di `docs/analyst/data-dictionary.md` dan `docs/analyst/erd.md`.

---

## 9. Modul dan Hubungannya

```
                 ┌───────────────────────┐
                 │      Storefront       │
                 │  katalog + checkout   │
                 └───────────┬───────────┘
                             │ transaksi dibuat (pending)
                             ▼
                 ┌───────────────────────┐
                 │   Payment Gateway     │
                 │  webhook + signature  │
                 └───────────┬───────────┘
                             │ status → paid
                             ▼
                 ┌───────────────────────┐
                 │  Distributor / Admin  │
                 │  fulfillment + retry  │
                 └───────────┬───────────┘
                             │ success / failed
                             ▼
                 ┌───────────────────────┐
                 │   Wallet & Keuangan   │
                 │  saldo + audit log    │
                 └───────────────────────┘

   Reseller API ──(Sanctum)──► ikut jalur transaksi yang sama
   Callback outbound ────────► notify sistem reseller eksternal
```

### Tiga integrasi inti

1. **Payment → Transaksi** — webhook ber-signature menggerakkan state, dalam satu transaksi atomik.
2. **Transaksi → Distributor** — fulfillment lewat queue dengan retry; kegagalan berujung `failed`.
3. **Transaksi → Reseller** — callback outbound saat status berubah, supaya sistem eksternal tetap sinkron.

Modul detail dan temuan kontrak antar-modul dibahas di `docs/analyst/api-contract.md` dan `docs/analyst/cross-module-review.md`.

---

## 10. Scope dan Non-Scope

### In Scope

- master data kategori, subkategori, produk
- user dan role (4 role)
- checkout guest & terlogin, invoice via `ref_id`
- payment gateway webhook
- fulfillment distributor + fulfillment manual Minecraft
- API reseller H2H
- wallet dan mutasi
- voucher
- admin panel + audit log + export PDF
- refund request
- business rules dan state machine
- kontrak API, NFR, dan review arsitektur

### Out of Scope

Untuk menjaga fokus analisis, simulasi ini sengaja **memotong** beberapa fitur (terdokumentasi sebagai scope-cut di `docs/analyst/spec.md`):

- click tracking & analytics reseller
- toko pribadi reseller white-label (`/toko/{slug}`)
- laporan komisi bulanan reseller
- live chat / broadcast (Reverb)
- deteksi fraud otomatis
- dashboard laporan keuangan terpisah
- RCON / otomasi fulfillment Minecraft
- integrasi eksternal: payment gateway lain, WhatsApp/SMS notifikasi, procurement, accounting

---

## 11. Struktur Dokumentasi Repository

Struktur utama memisahkan analisis global, analisis per domain/modul, dan dokumen pendukung.

```
docs/
├── agents/                          # konvensi kerja agent (issue tracker, label, domain)
│
├── analyst/                         # ARTIFAK ANALISIS INTI
│   ├── SRS.md                       # 38 FR + non-functional summary
│   ├── spec.md                      # spesifikasi evolusi per versi (v1.0 → v1.3.5)
│   ├── erd.md                       # entity relationship diagram + relasi
│   ├── data-dictionary.md           # definisi kolom lengkap + validasi
│   ├── business-rules.md            # 38 BR dalam 14 kategori
│   ├── process-flows.md             # 11 PF dengan step descriptions
│   ├── nfr.md                       # 14 NFR + QA scenario + gap (GAP 12)
│   ├── requirements-matrix.md       # traceability FR ↔ UC ↔ BR ↔ PF ↔ NFR
│   ├── assumptions-constraints.md   # ASM 4, CON 5, OQ 3
│   ├── api-contract.md              # kontrak E1–E6 + findings + rekomendasi
│   ├── cross-module-review.md       # review integrasi antar-modul
│   ├── architecture-review.md       # review arsitektur + quality gate
│   ├── consistency-audit.md         # audit konsistensi lintas dokumen
│   ├── design-quality.md            # penilaian kualitas desain
│   ├── extraction-report.md         # laporan ekstraksi dari kode
│   ├── NUMBERING-LOG.md             # inventaris otoritatif seluruh keluarga id
│   │
│   └── use-cases/                   # 23 UC dalam 5 bab
│       ├── INDEX-Use-Cases-Fofa-Shop.md
│       └── chapters/
│           ├── BAB-1-Storefront-Customer-Journey/
│           ├── BAB-2-Payment-GW-Financial/
│           ├── BAB-3-Admin-Panel-Fulfillment/
│           ├── BAB-4-Reseller-API/
│           └── BAB-5-System-Administration/
│
├── roles.md                         # role, wewenang, hak akses
├── owner-role.md                    # role owner + CLI provisioning
├── audit-log.md                     # desain audit log
├── deployment.md                    # panduan deploy VPS
├── redis-setup.md                   # setup Redis (cache/queue/session)
│
PRD.md                               # product requirements
CONTEXT.md                           # glossary & keputusan desain
AGENTS.md                            # konvensi repo
```

> **Catatan:** Architecture Decision Records (ADR) berjumlah 19 file dan dipantau
> lewat `NUMBERING-LOG.md`, tetapi file fisiknya tidak disertakan di repository ini.

---

## 12. Recommended Technology Stack

Rekomendasi implementasi yang dipakai sebagai baseline analisis arsitektur:

| Layer | Recommendation | Catatan |
|---|---|---|
| Backend | **PHP + Laravel 13** | framework utama untuk implementasi |
| UI | **Blade + Livewire/Volt** | form, CRUD, dan interaksi reaktif terbatas |
| Auth | **Laravel Breeze** + guard Sanctum | login + role dasar + token API |
| Database | **MySQL** | MariaDB kompatibel untuk development |
| Cache / Queue / Session | **Redis** | satu infra untuk tiga peran |
| Styling | **Tailwind CSS** | token OKLCH dual light/dark |
| Testing | **PHPUnit / Feature Test** | fokus state machine + idempotency |
| Local Environment | Laragon / XAMPP | development lokal |
| Deployment demo | VPS murah / shared hosting | hanya bila diperlukan untuk demo |
| Git | Git + GitHub | commit per tahap desain/implementasi |

### Prinsip pemilihan stack

Stack dipilih untuk menjaga implementasi tetap **ringan, mudah dijelaskan, dan tidak mengaburkan inti analisis**.

Karena repository ini analysis-first:

- framework berat untuk admin tidak dijadikan baseline;
- CRUD manual berbasis Blade tetap menjadi opsi utama;
 Livewire dipakai hanya pada interaksi yang memang butuh reaktifitas (dropdown edisi, upload, reactive validation).

Detail keputusan teknologi sebaiknya dibaca di `docs/analyst/architecture-review.md` dan `docs/analyst/nfr.md`.

---

## 13. What I Am Demonstrating

Repository ini tidak dimaksudkan untuk membuktikan bahwa saya bisa membuat sebanyak mungkin halaman atau fitur.

Yang ingin saya tunjukkan adalah kemampuan untuk:

```
Problem → Requirement → Data → Rule → Process → State → Integration → Architecture
```

Contoh konkret — satu transaksi top-up dari ujung ke ujung:

```
Customer beli produk
   ↓
Checkout → transaksi pending + ref_id dibuat server
   ↓
Payment gateway kirim webhook
   ↓
Signature diverifikasi (HMAC)
   ↓
Status → paid (atomik, idempotent)
   ↓
Queue fulfillment dilepas
   ↓
Distributor diproses (retry 3x) — ATAU admin fulfill manual (Minecraft)
   ↓
Status → success / failed
   ↓
Callback outbound ke reseller (bila ada)
   ↓
Audit log tercatat
```

Pada titik ini, interviewer dapat menguji saya dengan pertanyaan lanjutan seperti:

- Apa yang terjadi jika webhook datang dua kali?
- Apa yang terjadi jika pembayaran masuk setelah transaksi expired?
- Kapan saldo wallet berkurang, dan bagaimana mencegah double-spend?
- Bagaimana harga reseller dihitung, dan kenapa admin tidak kena markup?
- Siapa yang boleh melihat angka rupiah?
- Kenapa `ref_id` dibuat di server, bukan dikirim klien?
- Bagaimana mencegah pembelian berulang dari retry jaringan?
- Entitas mana yang source of truth untuk status tertentu?
- Apa konsekuensi kalau state machine diubah?

Jawaban terhadap pertanyaan tersebut diturunkan dari artefak yang ada di `docs/analyst/`.

---

## 14. Why This Repository Is Interview-Friendly

Repository ini dirancang supaya interviewer dapat membaca artefaknya terlebih dahulu, lalu melakukan deep-dive pada reasoning saya.

### Urutan baca yang disarankan

```
README.md
   ↓
PRD.md + docs/analyst/spec.md
   ↓
docs/analyst/business-rules.md
   ↓
docs/analyst/erd.md + data-dictionary.md
   ↓
docs/analyst/process-flows.md + use-cases/
   ↓
docs/analyst/nfr.md + requirements-matrix.md
   ↓
docs/analyst/api-contract.md
   ↓
docs/analyst/architecture-review.md
```

Dengan urutan itu, pembaca bisa melihat hubungan antara keputusan bisnis dan keputusan teknis, bukan hanya melihat diagram yang berdiri sendiri.

---

## 15. Development Path (Bila Nanti Diimplementasikan)

Apabila simulasi ini dilanjutkan menjadi aplikasi, urutan implementasi yang disarankan:

```
Database Schema
      ↓
Seeder / Dummy Data
      ↓
CRUD Modul Inti (produk, transaksi, pengguna)
      ↓
State Machine Transaksi
      ↓
Integrasi Payment Webhook + Distributor
      ↓
Wallet & Rate Limit
      ↓
Validation & Feature Tests
      ↓
Admin Panel + Export
      ↓
Optional Deployment Demo
```

Fokus testing pertama adalah dua area yang paling mudah rusak secara logika:

1. **Webhook diterima dua kali → status dan saldo tidak berubah dua kali.**
2. **Perubahan status transaksi → selalu mengikuti business rule, bukan input bebas.**

---

## 16. Limitations

Model ini sengaja dibatasi agar dapat dikerjakan sebagai solo-project dan tetap terbaca dalam sesi interview.

Beberapa hal yang belum dimodelkan secara penuh antara lain:

- detail dispatching & fleet management
- telemetry / IoT hour meter
- geospatial tracking
- procurement lifecycle dan anggaran
- vendor / supplier management
- deteksi fraud otomatis
- integrasi enterprise (accounting, HR)

Dokumen dalam repository ini juga telah **diredaksi untuk publikasi**: kredensial contoh disamarkan, domain disamarkan, dan detail teknis keamanan yang bersifat sensitif diringkas (versi lengkapnya disimpan terpisah).

Karena itu, pembaca sebaiknya melihat proyek ini sebagai **analisis dan system design simulation**, bukan blueprint produksi untuk perusahaan sungguhan.

---

## 17. Closing

Saya tidak sedang menunjukkan "aplikasi toko top-up yang sudah jadi".

Saya sedang menunjukkan bagaimana saya membedah sebuah domain bisnis, menemukan aturan penting, menghubungkan data dan proses, lalu menurunkannya menjadi rancangan sistem yang siap diimplementasikan.

Repository ini adalah **analysis portfolio dengan implementasi sebagai langkah lanjutan**, bukan syarat utama untuk memahami hasil analisis.

---

### Source of Decisions

Keputusan fungsional dan teknis pada README ini mengacu pada dokumen berikut:

| Dokumen | Yang diputuskan |
|---|---|
| `PRD.md` | lingkup produk, role, daftar fitur |
| `docs/analyst/spec.md` | state machine, pricing, scope-cut, baseline tech stack, evolusi v1.0 → v1.3.5 |
| `docs/analyst/business-rules.md` | 38 business rules dalam 14 kategori |
| `docs/analyst/api-contract.md` | kontrak endpoint E1–E6 dan temuannya |
| `docs/analyst/NUMBERING-LOG.md` | inventaris otoritatif seluruh keluarga id |
