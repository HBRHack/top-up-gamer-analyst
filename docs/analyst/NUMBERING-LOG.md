# Numbering Log — Fofa Shop

**Tujuan:** inventaris otoritatif untuk seluruh keluarga id yang dipakai lintas dokumen analisis
(`FR`, `UC`, `BR`, `NFR`, `PF`, `ADR`, `GAP`, `CON`, `ASM`, `OQ`) supaya pemeriksaan *Numbering*
pada design review bisa cross-check setiap id ke file yang mendefinisikannya, barisnya, dan
dokumen mana yang menyitanya.

**Sumber kebenaran = file yang dirujuk, bukan ingatan.** Setiap baris di bawah ini dihasilkan dari
`grep`/`sed`/`python` yang dijalankan terhadap repo pada saat penulisan — bukan dari resume percakapan.
Baris (*line number*) dicantutkan agar reviewer dapat memverifikasi ulang dengan perintah yang sama.

**Tanggal:** 2026-09-26
**Snapshot bertahan (jam lokal):** 09:45 – 09:51 WIB. Bagian **Business Rules** diambil paling akhir
(pukul 09:49–09:51) karena `docs/analyst/business-rules.md`, `docs/analyst/erd.md` dan
`docs/analyst/requirements-matrix.md` sedang diedit secara konkuren saat dokumen ini disusun.

**Perintah dasar untuk verifikasi ulang:**

```sh
grep -nE "^#### FR-"              docs/analyst/SRS.md
grep -nE "^## UC-"                docs/analyst/use-cases/chapters/*/*.md
grep -cE "^### BR-"               docs/analyst/business-rules.md
grep -noE "NFR-[A-Z]+-[0-9]+"     docs/analyst/nfr.md | sort -u -t: -k2
grep -nE "^## PF-"                docs/analyst/process-flows.md
(ADR dikelola di repo aplikasi — tidak disertakan di repo ini)
grep -nE "\| GAP-[0-9]+ \|"       docs/analyst/nfr.md
grep -nE "\| (CON|ASM|OQ)-[0-9]+ \|" docs/analyst/assumptions-constraints.md
```

---

## Reserved Ranges

| Prefix      | Range dipakai (terverifikasi)                                     | Range reserved (belum dipakai)                                                | Pemilik file (titik definisi)                          |
|-------------|-------------------------------------------------------------------|-------------------------------------------------------------------------------|--------------------------------------------------------|
| `FR-`       | `FR-001`–`FR-038` — 38 id, **tanpa celah**                        | `FR-039`+                                                                     | `docs/analyst/SRS.md` (heading `#### FR-…`)            |
| `UC-`       | `UC-001`–`UC-023` — 23 id, **tanpa celah**                        | `UC-024`+                                                                     | `docs/analyst/use-cases/chapters/*/*.md` (heading `## UC-…`) |
| `BR-`       | 38 id di 14 kategori (lihat §BR di bawah): PROD 001–002, PRICE 001–002, WAL 001–004, VOUCH 001–003, WF 001–004, FUL 001–004, PAY 001–003, AUTH 001–004, DEL 001–003, MC 001, REF 001–003, GAME 001–002, ANN 001–002, CHECK 001 | nomor berikutnya per kategori (`CHECK-002`, `MC-002`, …); **`BR-PAY-002` ditahan, jangan dipakai ulang** (status `Superseded`) | `docs/analyst/business-rules.md` (heading `### BR-…`) |
| `NFR-`      | PERF 001–003, SEC 001–003, AVL 001–004, SCP 001–004 — 14 id       | `PERF-004`+, `SEC-004`+, `AVL-005`+, `SCP-005`+; kategori baru bebas            | `docs/analyst/nfr.md` (heading `### NFR-…`)            |
| `PF-`       | `PF-001`–`PF-011` — 11 id, **tanpa celah**                        | `PF-012`+                                                                     | `docs/analyst/process-flows.md` (heading `## PF-…`)    |
| `ADR-`      | `ADR-0001`–`ADR-0019` — 19 file, **tanpa celah**                  | `ADR-0020`+; nomor `0014` dan `0019` tidak boleh dipakai ulang untuk topik lain | 19 file ADR (heading `# ADR-…`) — tidak disertakan di repo ini                    |
| `GAP-`      | `GAP-01`–`GAP-12` — 12 id                                         | `GAP-13`+                                                                     | `docs/analyst/nfr.md` (tabel §Gap)                     |
| `CON-`      | `CON-001`–`CON-005` — 5 id                                        | `CON-006`+                                                                    | `docs/analyst/assumptions-constraints.md` (§Constraints) |
| `ASM-`      | `ASM-001`–`ASM-004` — 4 id                                        | `ASM-005`+                                                                    | `docs/analyst/assumptions-constraints.md` (§Assumptions) |
| `OQ-`       | `OQ-001`–`OQ-003` — 3 id                                          | `OQ-004`+                                                                     | `docs/analyst/assumptions-constraints.md` (§Open Questions) |

Aturan penomoran yang terbaca dari pola di atas: **semua id padat 3 digit (`001`)** kecuali `ADR-`
(4 digit, mengikuti konvensi ADR `NNNN`) dan `GAP-` (2 digit). Nomor **tidak pernah dipakai ulang**
— id lama dipensiunkan (`Superseded`) alih-alih di-recycle (lihat `BR-PAY-002`).

---

## FR — Functional Requirements

`grep -nE "^#### FR-" docs/analyst/SRS.md` → **38 baris / 38 id unik / 0 celah**.
Urutan baris di SRS **tidak monoton** (lihat OD-10), tetapi himpunan id lengkap `001`–`038`.

| ID     | Judul                                    | File sumber (baris)          | Status   | Catatan                                  |
|--------|------------------------------------------|------------------------------|----------|------------------------------------------|
| FR-001 | Browse Kategori Game                     | `docs/analyst/SRS.md:46`     | Confirmed | → UC-001                                 |
| FR-002 | Search Produk                            | `docs/analyst/SRS.md:51`     | Confirmed | → UC-003                                 |
| FR-003 | Cek Nickname/ID Game                     | `docs/analyst/SRS.md:56`     | Confirmed | → UC-004                                 |
| FR-004 | Detail Produk + Harga Per Role           | `docs/analyst/SRS.md:61`     | Confirmed | → UC-002                                 |
| FR-005 | Checkout + Validasi                      | `docs/analyst/SRS.md:66`     | Confirmed | → UC-005; `BR-WF-001`, `BR-CHECK-001`    |
| FR-006 | Invoice Page                             | `docs/analyst/SRS.md:71`     | Confirmed | → UC-006                                 |
| FR-007 | Refund Request                           | `docs/analyst/SRS.md:76`     | Confirmed | → UC-007                                 |
| FR-008 | Riwayat Transaksi                        | `docs/analyst/SRS.md:81`     | Confirmed | → UC-007                                 |
| FR-009 | List Produk (API)                        | `docs/analyst/SRS.md:88`     | Confirmed | → UC-017                                 |
| FR-010 | Top-up via API                           | `docs/analyst/SRS.md:93`     | Confirmed | → UC-018                                 |
| FR-011 | Cek Status (API)                         | `docs/analyst/SRS.md:98`     | Confirmed | → UC-019; pemilik `BR-FUL-004`           |
| FR-012 | Cek Saldo (API)                          | `docs/analyst/SRS.md:103`    | Confirmed | → UC-020                                 |
| FR-013 | Webhook Callback                         | `docs/analyst/SRS.md:108`    | Confirmed | → UC-008                                 |
| FR-014 | Dashboard Monitoring                     | `docs/analyst/SRS.md:115`    | Confirmed | → UC-012                                 |
| FR-015 | CRUD Produk/Kategori/Subkategori         | `docs/analyst/SRS.md:120`    | Confirmed | → UC-013                                 |
| FR-016 | Retry Transaksi Gagal                    | `docs/analyst/SRS.md:125`    | Confirmed | → UC-014                                 |
| FR-017 | Manual Fulfill Minecraft                 | `docs/analyst/SRS.md:130`    | Confirmed | → UC-015                                 |
| FR-018 | Kelola Wallet Reseller                   | `docs/analyst/SRS.md:135`    | Confirmed | → UC-010                                 |
| FR-019 | Toggle Payment Method                    | `docs/analyst/SRS.md:140`    | Confirmed | → UC-023                                 |
| FR-020 | Broadcast Pengumuman                     | `docs/analyst/SRS.md:145`    | Confirmed | → UC-022                                 |
| FR-021 | Audit Log                                | `docs/analyst/SRS.md:150`    | Confirmed | → UC-012                                 |
| FR-022 | Kelola Voucher                           | `docs/analyst/SRS.md:155`    | Confirmed | → UC-005                                 |
| FR-023 | Manage Refund Requests                   | `docs/analyst/SRS.md:160`    | Confirmed | → UC-016                                 |
| FR-034 | Transaction Chart                        | `docs/analyst/SRS.md:165`    | Confirmed | disisipkan di sela FR-023/FR-024 (urutan) |
| FR-036 | Product Image Upload                     | `docs/analyst/SRS.md:170`    | Confirmed | disisipkan di sela FR-034/FR-024 (urutan) |
| FR-024 | View Omset                               | `docs/analyst/SRS.md:177`    | Confirmed | → UC-012                                 |
| FR-025 | Laporan Keuangan                         | `docs/analyst/SRS.md:182`    | Dropped  | scope-cut v1.3; baris tetap ada di matrix |
| FR-026 | Export PDF                               | `docs/analyst/SRS.md:187`    | Confirmed | → UC-011                                 |
| FR-027 | Export Pajak                             | `docs/analyst/SRS.md:192`    | Dropped  | scope-cut v1.3; baris tetap ada di matrix |
| FR-028 | Kelola Admin                             | `docs/analyst/SRS.md:197`    | Confirmed | → UC-021                                 |
| FR-029 | Hapus Produk (Unrestricted)              | `docs/analyst/SRS.md:202`    | Confirmed | → UC-013                                 |
| FR-035 | Financial Data Access Control            | `docs/analyst/SRS.md:207`    | Confirmed | disisipkan di sela FR-029/FR-030 (urutan) |
| FR-030 | Voucher Diskon                           | `docs/analyst/SRS.md:214`    | Confirmed | → UC-005                                 |
| FR-031 | Email Notifikasi                         | `docs/analyst/SRS.md:219`    | Confirmed | → UC-022                                 |
| FR-032 | Notification Preference                  | `docs/analyst/SRS.md:224`    | Confirmed | → UC-022                                 |
| FR-033 | Minecraft Credential Reveal              | `docs/analyst/SRS.md:229`    | Confirmed | → UC-015                                 |
| FR-037 | Guest Email saat Checkout                | `docs/analyst/SRS.md:234`    | Confirmed | → UC-005                                 |
| FR-038 | Theme Toggle (Terang/Gelap)              | `docs/analyst/SRS.md:239`    | Confirmed | tanpa UC (`—` di matrix)                 |

Status diambil dari kolom **Status** di `docs/analyst/requirements-matrix.md:43-90` (tabel prioritas,
header `## Requirements by Priority` baris 37) dan :98-135 (Full Traceability Matrix, header baris 94)
— **kedua tabel memuat 38 id yang sama** (diverifikasi:
`sed -n '43,90p'` dan `sed -n '95,150p'` masing-masing menghasilkan 38 `FR-*` unik).
Coverage Summary `requirements-matrix.md:218`: `FR 38`, Confirmed 36, Dropped 2 (`FR-025`, `FR-027`).

---

## UC — Use Cases

`grep -nE "^## UC-" docs/analyst/use-cases/chapters/*/*.md` → **23 baris / 23 id unik / 0 celah**.

| ID     | Judul                                       | File sumber (baris)                                                            | Status   | Catatan                       |
|--------|---------------------------------------------|--------------------------------------------------------------------------------|----------|-------------------------------|
| UC-001 | Browse Kategori & Produk                    | `…/BAB-1-Storefront-Customer-Journey.md:25`                                    | Confirmed | gabungan old UC-001 + UC-002  |
| UC-002 | Detail Produk + Harga Per Role              | `…/BAB-1-Storefront-Customer-Journey.md:101`                                   | Confirmed | old UC-003                    |
| UC-003 | Search Produk                               | `…/BAB-1-Storefront-Customer-Journey.md:168`                                   | Confirmed | old UC-004                    |
| UC-004 | Verifikasi ID / Nickname Game               | `…/BAB-1-Storefront-Customer-Journey.md:216`                                   | Confirmed | old UC-005                    |
| UC-005 | Checkout & Penerapan Voucher Diskon         | `…/BAB-1-Storefront-Customer-Journey.md:277`                                   | Confirmed | old UC-006                    |
| UC-006 | View Invoice Status                         | `…/BAB-1-Storefront-Customer-Journey.md:357`                                   | Confirmed | old UC-007                    |
| UC-007 | Riwayat Transaksi & Form Pengajuan Refund   | `…/BAB-1-Storefront-Customer-Journey.md:423`                                   | Confirmed | old UC-008                    |
| UC-008 | Payment Gateway Callback                    | `…/BAB-2-Payment-GW-Financial.md:22`                                           | Confirmed | old UC-009                    |
| UC-009 | Cek Saldo & Mutasi Wallet                   | `…/BAB-2-Payment-GW-Financial.md:95`                                           | Confirmed | pecahan old UC-013            |
| UC-010 | Kelola Deposit Wallet Reseller              | `…/BAB-2-Payment-GW-Financial.md:144`                                          | Confirmed | old UC-018                    |
| UC-011 | Laporan Keuangan, Omset & Export PDF/Pajak  | `…/BAB-2-Payment-GW-Financial.md:188`                                          | Confirmed | pecahan old UC-019            |
| UC-012 | Dashboard Admin & Monitoring                | `…/BAB-3-Admin-Panel-Fulfillment.md:23`                                        | Confirmed | old UC-014                    |
| UC-013 | CRUD Produk, Kategori & Subkategori         | `…/BAB-3-Admin-Panel-Fulfillment.md:71`                                        | Confirmed | pecahan old UC-015            |
| UC-014 | Retry Transaksi Gagal                       | `…/BAB-3-Admin-Panel-Fulfillment.md:124`                                       | Confirmed | pecahan old UC-015            |
| UC-015 | Manual Fulfill Minecraft                    | `…/BAB-3-Admin-Panel-Fulfillment.md:166`                                       | Confirmed | old UC-016                    |
| UC-016 | Kelola Refund                               | `…/BAB-3-Admin-Panel-Fulfillment.md:214`                                       | Confirmed | old UC-017                    |
| UC-017 | List Produk via API                         | `…/BAB-4-Reseller-API.md:22`                                                  | Confirmed | old UC-010                    |
| UC-018 | Top-up via API                              | `…/BAB-4-Reseller-API.md:65`                                                  | Confirmed | old UC-011                    |
| UC-019 | Cek Status via API                          | `…/BAB-4-Reseller-API.md:135`                                                 | Confirmed | old UC-012                    |
| UC-020 | Cek Saldo via API                           | `…/BAB-4-Reseller-API.md:172`                                                 | Confirmed | pecahan old UC-013            |
| UC-021 | Kelola Akun Admin                           | `…/BAB-5-System-Administration.md:21`                                          | Confirmed | old UC-020                    |
| UC-022 | Broadcast Pengumuman                        | `…/BAB-5-System-Administration.md:80`                                          | Confirmed | UC baru (old: `—`)            |
| UC-023 | Toggle Payment Method                       | `…/BAB-5-System-Administration.md:135`                                         | Confirmed | pecahan old UC-019            |

### UC Numbering Mapping (old → new)

Sumber: `docs/analyst/requirements-matrix.md` § `## UC Numbering Mapping` (header baris **7**,
baris tabel **11–33**; data 21 baris di baris 13–33). Diverifikasi: 20 baris `Old UC` (`UC-001`…`UC-020`,
**tanpa celah**) dan **23 id baru semuanya tercakup** di kolom `New UC`.

| Old UC  | New UC            | Keterangan                        |
|---------|-------------------|-----------------------------------|
| UC-001  | UC-001            | merge (old UC-002 dilebur)        |
| UC-002  | UC-001            | dilebur                           |
| UC-003  | UC-002            |                                   |
| UC-004  | UC-003            |                                   |
| UC-005  | UC-004            |                                   |
| UC-006  | UC-005            |                                   |
| UC-007  | UC-006            |                                   |
| UC-008  | UC-007            |                                   |
| UC-009  | UC-008            |                                   |
| UC-010  | UC-017            | pindah BAB                        |
| UC-011  | UC-018            | pindah BAB                        |
| UC-012  | UC-019            | pindah BAB                        |
| UC-013  | UC-009 + UC-020   | split                             |
| UC-014  | UC-012            |                                   |
| UC-015  | UC-013 + UC-014   | split                             |
| UC-016  | UC-015            |                                   |
| UC-017  | UC-016            |                                   |
| UC-018  | UC-010            |                                   |
| UC-019  | UC-011 + UC-023   | split                             |
| UC-020  | UC-021            |                                   |
| —       | UC-022            | UC baru                           |

> Catatan: `UC-023` tidak punya baris `Old: —` sendiri — ia hanya muncul di dalam baris split
> `UC-019`. Lihat OD-09.

---

## BR — Business Rules

`grep -cE "^### BR-" docs/analyst/business-rules.md` → **38**; id unik juga **38** (0 duplikat).
Snapshot terakhir: **2026-09-26 09:51** (bagian ini memang ditulis paling akhir).

Distribusi per kategori (dihitung dari heading):

| Kategori | Id dipakai                | Jumlah |
|----------|---------------------------|--------|
| PROD     | 001–002                   | 2      |
| PRICE    | 001–002                   | 2      |
| WAL      | 001–004                   | 4      |
| VOUCH    | 001–003                   | 3      |
| WF       | 001–004                   | 4      |
| FUL      | 001–004                   | 4      |
| PAY      | 001–003 (002 Superseded)  | 3      |
| AUTH     | 001–004                   | 4      |
| DEL      | 001–003                   | 3      |
| MC       | 001                       | 1      |
| REF      | 001–003                   | 3      |
| GAME     | 001–002                   | 2      |
| ANN      | 001–002                   | 2      |
| CHECK    | 001                       | 1      |
| **Total**| **14 kategori**           | **38** |

| ID           | Judul                                    | File sumber (baris)                     | Status                    | Catatan (`Use Case:` di blok)         |
|--------------|------------------------------------------|-----------------------------------------|---------------------------|---------------------------------------|
| BR-PROD-001  | Kategori Coming Soon Tampil, Beli Disabled | `docs/analyst/business-rules.md:28`   | [x] Confirmed             | UC-001                                |
| BR-PROD-002  | Hanya Produk Aktif di Storefront         | `docs/analyst/business-rules.md:60`      | [x] Confirmed             | UC-001                                |
| BR-PRICE-001 | Harga Per Role                           | `docs/analyst/business-rules.md:90`      | [x] Confirmed             | UC-002, UC-017                        |
| BR-PRICE-002 | Voucher Discount Cap                     | `docs/analyst/business-rules.md:121`     | [x] Confirmed             | UC-005 — **tak dikutip chapter** (OD-06) |
| BR-WAL-001   | Wallet Atomic Operations                 | `docs/analyst/business-rules.md:151`     | [x] Confirmed             | UC-010, UC-018                        |
| BR-WAL-002   | Wallet Auto-Create                       | `docs/analyst/business-rules.md:175`     | [x] Confirmed             | UC-009, UC-010                        |
| BR-WAL-003   | Wallet Transaction Uniqueness            | `docs/analyst/business-rules.md:199`     | [x] Confirmed             | UC-009                                |
| BR-WAL-004   | Reseller Auto-Refund                     | `docs/analyst/business-rules.md:223`     | [x] Confirmed             | UC-007 — **tak dikutip chapter** (OD-06) |
| BR-VOUCH-001 | Voucher Validation Chain                 | `docs/analyst/business-rules.md:247`     | [x] Confirmed             | UC-005                                |
| BR-VOUCH-002 | Voucher Case-Insensitive                 | `docs/analyst/business-rules.md:271`     | [x] Confirmed             | `—` (belum dimiliki UC)               |
| BR-VOUCH-003 | Voucher Atomic Apply                     | `docs/analyst/business-rules.md:295`     | [x] Confirmed             | `—` (belum dimiliki UC)               |
| BR-WF-001    | Transaction State Machine                | `docs/analyst/business-rules.md:319`     | [x] Confirmed             | UC-005, UC-008, UC-014, UC-015 (lebih luas dari matrix, OD-06) |
| BR-WF-002    | Transaction Idempotency                  | `docs/analyst/business-rules.md:343`     | [x] Confirmed             | UC-008                                |
| BR-WF-003    | Pending Expire Cron                      | `docs/analyst/business-rules.md:367`     | [x] Confirmed             | UC-008 — **bertentangan dengan FR→UC** (OD-07) |
| BR-WF-004    | Reconciliation                           | `docs/analyst/business-rules.md:391`     | [x] Confirmed             | UC-008                                |
| BR-FUL-001   | Minecraft Manual Fulfillment             | `docs/analyst/business-rules.md:415`     | [x] Confirmed             | UC-015                                |
| BR-FUL-002   | Fulfillment Retry                        | `docs/analyst/business-rules.md:439`     | [x] Confirmed             | UC-014                                |
| BR-FUL-003   | Missing Vendor Mapping                   | `docs/analyst/business-rules.md:463`     | [x] Confirmed             | UC-018                                |
| BR-FUL-004   | Sync Waiting Fulfillment                 | `docs/analyst/business-rules.md:487`     | [x] Confirmed             | `—` (milik PF-009); **id baru**       |
| BR-PAY-001   | Payment Signature                        | `docs/analyst/business-rules.md:511`     | [x] Confirmed             | UC-008                                |
| BR-PAY-002   | Checkout Rate Limiting                   | `docs/analyst/business-rules.md:534`     | **Superseded by BR-CHECK-001** | UC-005; id ditahan agar tak dipakai ulang |
| BR-PAY-003   | Payment Method Toggle                    | `docs/analyst/business-rules.md:559`     | [x] Confirmed             | UC-023                                |
| BR-AUTH-001  | Role Middleware                          | `docs/analyst/business-rules.md:583`     | [x] Confirmed             | `—` (belum dimiliki UC)               |
| BR-AUTH-002  | Owner Admin Management                   | `docs/analyst/business-rules.md:606`     | [x] Confirmed             | UC-021                                |
| BR-AUTH-003  | Financial Data Access Control            | `docs/analyst/business-rules.md:629`     | [x] Confirmed             | UC-011, UC-012                        |
| BR-AUTH-004  | Self-Protection                          | `docs/analyst/business-rules.md:653`     | [x] Confirmed             | UC-021                                |
| BR-DEL-001   | Product Deletion — Has Transactions      | `docs/analyst/business-rules.md:676`     | [x] Confirmed             | UC-013 (dahulu salah `UC-029`)        |
| BR-DEL-002   | Product Deletion — Timing & Role         | `docs/analyst/business-rules.md:700`     | [x] Confirmed             | UC-013 (dahulu salah `UC-029`)        |
| BR-DEL-003   | Product Deletion Audit                   | `docs/analyst/business-rules.md:725`     | [x] Confirmed             | UC-013                                |
| BR-MC-001    | Minecraft Credential Encryption          | `docs/analyst/business-rules.md:749`     | [x] Confirmed             | UC-015                                |
| BR-REF-001   | Reseller vs Customer Refund              | `docs/analyst/business-rules.md:772`     | [x] Confirmed             | UC-007                                |
| BR-REF-002   | Reseller Auto-Refund Tanpa Manual Request| `docs/analyst/business-rules.md:798`     | [x] Confirmed             | UC-007                                |
| BR-REF-003   | Refund Selesai Wajib Sudah Ada Request   | `docs/analyst/business-rules.md:829`     | [x] Confirmed             | UC-016                                |
| BR-GAME-001  | Game ID Verification Chain               | `docs/analyst/business-rules.md:857`     | [x] Confirmed             | UC-004                                |
| BR-GAME-002  | Local Game ID Validation                 | `docs/analyst/business-rules.md:880`     | [x] Confirmed             | UC-004                                |
| BR-ANN-001   | Announcement Audience                    | `docs/analyst/business-rules.md:903`     | [x] Confirmed             | UC-022                                |
| BR-ANN-002   | Email Notification Rules                 | `docs/analyst/business-rules.md:931`     | [x] Confirmed             | UC-022                                |
| BR-CHECK-001 | Checkout Rate Limiting                   | `docs/analyst/business-rules.md:957`     | [x] Confirmed             | UC-005; kini kanonik untuk rate limit  |

Ringkasan status: **37 `[x] Confirmed` + 1 `Superseded by BR-CHECK-001`** (`BR-PAY-002`).
38 dari 38 blok punya baris `- Use Case:` (sebelumnya hanya 29).

---

## NFR — Non-Functional Requirements

`grep -oE "NFR-[A-Z]+-[0-9]+" docs/analyst/nfr.md | sort -u` → **14 id unik**
(PERF 3, SEC 3, AVL 4, SCP 4 — tanpa celah tiap kategori).
Definisi di heading `### NFR-…`; ringkasan tabel di `nfr.md:25-38`.

| ID          | Judul (ringkas)                                        | File sumber (baris)             | Status       | Catatan                            |
|-------------|--------------------------------------------------------|---------------------------------|--------------|------------------------------------|
| NFR-PERF-001| Latency respons cached: cek game ID & pencarian        | `docs/analyst/nfr.md:44` (tabel :25) | Referenced | pemilik FR-003                     |
| NFR-PERF-002| Throughput ~60 req/s                                    | `docs/analyst/nfr.md:59` (tabel :26) | System-wide | tanpa FR owner                     |
| NFR-PERF-003| Cache hit rate > 80% (product listings)                | `docs/analyst/nfr.md:71` (tabel :27) | Referenced | FR-001, FR-002, FR-004             |
| NFR-SEC-001 | Keamanan jalur checkout (FR-005)                       | `docs/analyst/nfr.md:87` (tabel :28) | Referenced | pemilik FR-005                     |
| NFR-SEC-002 | Autentikasi & otorisasi API reseller                   | `docs/analyst/nfr.md:113` (tabel :29) | Referenced | pemilik FR-010                     |
| NFR-SEC-003 | Pencegahan SQL injection & XSS                         | `docs/analyst/nfr.md:134` (tabel :30) | Referenced | FR-002, FR-005                     |
| NFR-AVL-001 | Queue retry 3 attempts per job                         | `docs/analyst/nfr.md:153` (tabel :31) | Referenced | FR-010, FR-016                     |
| NFR-AVL-002 | Payment idempotency: ref_id unique + lockForUpdate     | `docs/analyst/nfr.md:166` (tabel :32) | Referenced | pemilik FR-013                     |
| NFR-AVL-003 | Wallet atomicity: DB::transaction + lockForUpdate      | `docs/analyst/nfr.md:181` (tabel :33) | Referenced | FR-010, FR-018                     |
| NFR-AVL-004 | Scheduled tasks: expire (1 menit) + sync (5 menit)     | `docs/analyst/nfr.md:193` (tabel :34) | Referenced | FR-011, FR-016                     |
| NFR-SCP-001 | Single VPS deployment (2 vCPU, 2GB RAM)                | `docs/analyst/nfr.md:212` (tabel :35) | System-wide | tanpa FR owner                     |
| NFR-SCP-002 | Redis untuk cache + queue + session                    | `docs/analyst/nfr.md:224` (tabel :36) | Referenced | FR-010, FR-013                     |
| NFR-SCP-003 | PHP-FPM 25 workers                                     | `docs/analyst/nfr.md:237` (tabel :37) | System-wide | tanpa FR owner                     |
| NFR-SCP-004 | Nginx static cache + gzip                              | `docs/analyst/nfr.md:249` (tabel :38) | Referenced | FR-001, FR-002                     |

Status & pemilik FR dari `requirements-matrix.md:195-208` (§ NFR Traceability, header baris 189);
Coverage Summary `requirements-matrix.md:221` + bullet `:226`
(`Non-Functional 14`, Confirmed 11, N/A 3).

---

## PF — Process Flows

`grep -nE "^## PF-" docs/analyst/process-flows.md` → **11 baris / 11 id unik / 0 celah**.

| ID     | Judul                                       | File sumber (baris)           | Status                  | Catatan                                  |
|--------|---------------------------------------------|-------------------------------|-------------------------|------------------------------------------|
| PF-001 | Customer Checkout                           | `docs/analyst/process-flows.md:7`   | Tidak dimuat di matrix | dirujuk UC-001/002/005/006                |
| PF-002 | Payment Callback                            | `docs/analyst/process-flows.md:82`  | Tidak dimuat di matrix | dirujuk UC-008                            |
| PF-003 | Transaction Fulfillment                     | `docs/analyst/process-flows.md:152` | Tidak dimuat di matrix | dirujuk UC-014                            |
| PF-004 | Minecraft Manual Fulfillment                | `docs/analyst/process-flows.md:233` | Tidak dimuat di matrix | dirujuk UC-015                            |
| PF-005 | Reseller Auto-Refund                        | `docs/analyst/process-flows.md:290` | Tidak dimuat di matrix | dirujuk UC-009                            |
| PF-006 | Voucher Apply                               | `docs/analyst/process-flows.md:358` | Tidak dimuat di matrix | dirujuk UC-005                            |
| PF-007 | Reseller API Top-up                         | `docs/analyst/process-flows.md:431` | Tidak dimuat di matrix | dirujuk UC-016                            |
| PF-008 | Expire Pending Transactions (Scheduled)     | `docs/analyst/process-flows.md:488` | Tidak dimuat di matrix | **tidak dirujuk INDEX** (OD-03)          |
| PF-009 | Sync Waiting Fulfillment (Scheduled)        | `docs/analyst/process-flows.md:536` | Tidak dimuat di matrix | pemilik `BR-FUL-004`                      |
| PF-010 | Announcement Broadcast                      | `docs/analyst/process-flows.md:588` | Tidak dimuat di matrix | dirujuk UC-022                            |
| PF-011 | Product Deletion                            | `docs/analyst/process-flows.md:652` | Tidak dimuat di matrix | **tidak dirujuk INDEX** (OD-03)          |

Verifikasi baris: `grep -nE "^## PF-" docs/analyst/process-flows.md` →
`7, 82, 152, 233, 290, 358, 431, 488, 536, 588, 652`.

---

## ADR — Architecture Decision Records

**19 file ADR** (`ADR-0001`…`ADR-0019`, tanpa celah) — file fisik ADR tidak disertakan di repo dokumentasi ini.

| ID       | Judul                                                     | File sumber (baris)                                | Status     | Catatan                       |
|----------|-----------------------------------------------------------|-----------------------------------------------------|------------|-------------------------------|
| ADR-0001 | Single Transaction Entity (No Separate Invoice)           | `ADR-0001 (single-transaction-entity):1`      | Confirmed  |                               |
| ADR-0002 | Payments Table 1-N Relationship                           | `ADR-0002 (payments-1-n-relationship):1`      | Confirmed  |                               |
| ADR-0003 | Transaction Status Flow                                   | `ADR-0003 (transaction-status-flow):1`        | Confirmed  |                               |
| ADR-0004 | Dynamic Categories Table                                  | `ADR-0004 (dynamic-categories-table):1`       | Confirmed  |                               |
| ADR-0005 | Wallet System for Reseller Refunds                        | `ADR-0005 (wallet-system-for-reseller-refunds):1` | Confirmed |                          |
| ADR-0006 | Multi-Vendor Failover Manual for MVP                      | `ADR-0006 (multi-vendor-failover-manual):1`   | Confirmed  |                               |
| ADR-0007 | Owner as Separate Role                                    | `ADR-0007 (owner-separate-role):1`            | Confirmed  |                               |
| ADR-0008 | Voucher/Promo System Design                               | `ADR-0008 (voucher-promo-system):1`           | Confirmed  |                               |
| ADR-0009 | UTM-Based Click Tracking for Reseller                     | `ADR-0009 (utm-click-tracking):1`             | **Cancelled** | Coverage Summary "Dropped" |
| ADR-0010 | White-Label Store via Custom Path                         | `ADR-0010 (white-label-store-path):1`         | **Cancelled** | Coverage Summary "Dropped" |
| ADR-0011 | Admin Audit Log                                           | `ADR-0011 (admin-audit-log):1`                | Confirmed  |                               |
| ADR-0012 | Live Chat via Laravel Reverb                              | `ADR-0012 (live-chat-reverb):1`               | **Cancelled** | Coverage Summary "Dropped" |
| ADR-0013 | Product Deletion Rules (3-Tier)                           | `ADR-0013 (product-deletion-rules):1`         | Confirmed  |                               |
| ADR-0014 | Redis for Cache, Queue, and Session                       | `ADR-0014 (redis-queue-cache-session):1`      | Confirmed  | pemenang nomor ganda lama      |
| ADR-0015 | Financial Data Access Control (Admin vs Owner)            | `ADR-0015 (financial-data-access-control):1`  | Confirmed  |                               |
| ADR-0016 | Minecraft Edition Scenario + Akun Delivery via Email      | `ADR-0016 (minecraft-edition-scenario):1`     | Confirmed  |                               |
| ADR-0017 | Upload Foto Produk & Subkategori Minecraft                | `ADR-0017 (product-image-upload):1`           | Confirmed  |                               |
| ADR-0018 | Tailwind CSS v4 Upgrade                                   | `ADR-0018 (tailwindcss-v4-upgrade):1`         | Confirmed  |                               |
| ADR-0019 | Role Hierarchy — Guest, Customer, Admin, Owner            | `ADR-0019 (role-hierarchy):1`                 | Confirmed  | **renumber dari `0014`** (lihat §Resolved) |

Status dari `requirements-matrix.md` § ADR Traceability (header baris **246**, tabel baris **250–268**):
19 baris `ADR-0001`…`ADR-0019` — konfirmasi 19 id terpetakan, 3 Cancelled.

---

## GAP / CON / ASM / OQ — Gap, Constraint, Assumption, Open Question

`grep -nE "\| GAP-[0-9]+ \|" docs/analyst/nfr.md` → **12 id** (`GAP-01`…`GAP-12`, tanpa celah).

| ID      | Judul (ringkas)                                             | File sumber (baris) | Status | Catatan                                |
|---------|-------------------------------------------------------------|---------------------|--------|----------------------------------------|
| GAP-01  | NFR-PERF-001: statistik & window latency tak didefinisikan  | `docs/analyst/nfr.md:300` | Open | Missing target                         |
| GAP-02  | NFR-PERF-001: tanpa target cache miss / provider lambat     | `docs/analyst/nfr.md:301` | Open | Missing target                         |
| GAP-03  | NFR-PERF-001: route search/list/detail tanpa cache aplikasi | `docs/analyst/nfr.md:302` | Open | Implementation mismatch                |
| GAP-04  | NFR-PERF-002: durasi sustain, mix, error rate tak ada       | `docs/analyst/nfr.md:303` | Open | Missing target                         |
| GAP-05  | NFR-PERF-003: window & definisi "product listings"          | `docs/analyst/nfr.md:304` | Open | Missing target                         |
| GAP-06  | NFR-AVL-001..004: uptime %, RPO/RTO, SLA antrian            | `docs/analyst/nfr.md:305` | Open | Missing target                         |
| GAP-07  | NFR-SCP-001..004: pertumbuhan disk & threshold scale-up     | `docs/analyst/nfr.md:306` | Open | Missing target                         |
| GAP-08  | NFR-SEC-001/002: angka rate limit, rotasi token             | `docs/analyst/nfr.md:307` | Open | Missing target; juga disitir `requirements-matrix.md:240` |
| GAP-09  | NFR-SEC-002: `api_key`/`api_secret` tak pernah dibaca       | `docs/analyst/nfr.md:308` | Open | Implementation mismatch; juga disitir `assumptions-constraints.md:79` |
| GAP-10  | NFR-SEC-001: patching/HTTPS wajib/enkripsi at-rest           | `docs/analyst/nfr.md:309` | Open | Missing target                         |
| GAP-11  | Traceability NFR: klaim matrix "Non-Functional = 4"         | `docs/analyst/nfr.md:310` | Open | **Pernyataan basi — lihat OD-01**      |
| GAP-12  | NFR-SCP-004: ukuran dampak gzip/offload                     | `docs/analyst/nfr.md:311` | Open | Missing target                         |

| ID      | Judul                                        | File sumber (baris)                        | Status | Catatan                        |
|---------|----------------------------------------------|--------------------------------------------|--------|--------------------------------|
| CON-001 | VPS hosting berbayar (bukan shared hosting)  | `docs/analyst/assumptions-constraints.md:23` | Closed | Terverifikasi deployment docs  |
| CON-002 | Domain custom + SSL                          | `docs/analyst/assumptions-constraints.md:24` | Closed | Terverifikasi certbot          |
| CON-003 | Redis wajib (cache + queue + session)        | `docs/analyst/assumptions-constraints.md:25` | Closed | Terverifikasi config defaults  |
| CON-004 | Root access untuk deployment                 | `docs/analyst/assumptions-constraints.md:26` | Closed | Terverifikasi deployment docs  |
| CON-005 | Minecraft produk butuh manual fulfillment    | `docs/analyst/assumptions-constraints.md:27` | Closed | Terverifikasi job + FR-017     |
| ASM-001 | Digiflazz API v2 available dan responsive    | `docs/analyst/assumptions-constraints.md:35` | Partial | Deviasi tercatat (klaim "v2")  |
| ASM-002 | Payment gateway webhook dalam hitungan detik | `docs/analyst/assumptions-constraints.md:36` | Partial | Deviasi tercatat               |
| ASM-003 | Single-server deployment                     | `docs/analyst/assumptions-constraints.md:37` | Closed | Terverifikasi                  |
| ASM-004 | Email delivery via SMTP provider             | `docs/analyst/assumptions-constraints.md:38` | Partial | Default `MAIL_MAILER=log`      |
| OQ-001  | Backup distributor selain Digiflazz?         | `docs/analyst/assumptions-constraints.md:46` | Open   | ADR-0006 siap diaktifkan       |
| OQ-002  | Monitoring/alerting strategy?                | `docs/analyst/assumptions-constraints.md:47` | Open   | Belum ada Sentry/UptimeRobot   |
| OQ-003  | Backup strategy untuk database?              | `docs/analyst/assumptions-constraints.md:48` | Open   | Belum ada cron backup          |

---

## Cross-Document Id Usage

Semua angka = **jumlah id unik** di file itu, dihitung dengan
`grep -ohE '<pola>' <file> | sort -u | wc -l`; kolom *Baris pertama* = `grep -nE '<pola>' <file> | head -1`.

### FR (38 id)

| Dokumen yang menyitir                              | Id unik | Baris pertama | Catatan                              |
|----------------------------------------------------|---------|---------------|--------------------------------------|
| `docs/analyst/SRS.md`                              | 38      | 46            | pemilik definisi                     |
| `docs/analyst/requirements-matrix.md`              | 38      | 43            | tabel prioritas + full traceability  |
| `docs/analyst/nfr.md`                              | 11      | 25            | kolom "FR terkait" per NFR           |
| `docs/analyst/assumptions-constraints.md`          | 1       | 27            | `FR-017` (CON-005)                   |
| chapter use case / `process-flows.md` / `PRD.md`   | 0       | —             | FR↔UC hidup hanya di matrix (OD-05)  |

### UC (23 id)

| Dokumen yang menyitir                                            | Id unik | Baris pertama | Catatan                                   |
|-----------------------------------------------------------------|---------|---------------|-------------------------------------------|
| `docs/analyst/use-cases/chapters/BAB-1-…Customer-Journey.md`     | 7       | 25            | UC-001…UC-007                             |
| `…/BAB-2-Payment-GW-Financial.md`                               | 4       | 22            | UC-008…UC-011                             |
| `…/BAB-3-Admin-Panel-Fulfillment.md`                            | 5       | 23            | UC-012…UC-016                             |
| `…/BAB-4-Reseller-API.md`                                       | 4       | 22            | UC-017…UC-020                             |
| `…/BAB-5-System-Administration.md`                              | 3       | 21            | UC-021…UC-023                             |
| `docs/analyst/use-cases/INDEX-Use-Cases-Fofa-Shop.md`           | 23      | 28            | tabel per BAB + Coverage (total 23)       |
| `docs/analyst/requirements-matrix.md`                           | 23      | 13            | § UC Numbering Mapping + kolom UC         |
| `docs/analyst/business-rules.md`                                | 19      | 55            | baris `- Use Case:` di 38 blok BR         |
| `docs/analyst/extraction-report.md`                             | 1       | 84            | jalur lama `use-cases/UC-001-…` (OD-08)   |
| `docs/analyst/SRS.md`                                           | 0       | —             | SRS tidak menyitir UC (OD-05)            |

### BR (38 id)

| Dokumen yang menyitir                                            | Id unik | Baris pertama | Catatan                                     |
|-----------------------------------------------------------------|---------|---------------|---------------------------------------------|
| `docs/analyst/business-rules.md`                                | 38      | 3             | pemilik definisi                            |
| `docs/analyst/requirements-matrix.md`                           | 38      | 43            | kolom Business Rules + § BR Coverage (38 baris) |
| `docs/analyst/use-cases/chapters/BAB-1-…Customer-Journey.md`     | 10      | 78            |                                             |
| `…/BAB-3-Admin-Panel-Fulfillment.md`                            | 8       | 57            |                                             |
| `…/BAB-2-Payment-GW-Financial.md`                               | 7       | 75            |                                             |
| `…/BAB-5-System-Administration.md`                              | 5       | 64            |                                             |
| `…/BAB-4-Reseller-API.md`                                       | 3       | 50            |                                             |
| **Total 5 chapter**                                              | **30**  | —             | 8 id BR tidak dikutip chapter (lihat OD-06) |
| `docs/analyst/erd.md`                                           | 27      | 120           | 11 id BR tidak disitir ERD (OD-08b)         |
| `docs/analyst/process-flows.md`                                 | 3       | 711           |                                             |
| `docs/analyst/nfr.md`                                           | 1       | 118           |                                             |

### NFR (14 id)

| Dokumen yang menyitir                       | Id unik | Baris pertama | Catatan                        |
|---------------------------------------------|---------|---------------|--------------------------------|
| `docs/analyst/nfr.md`                       | 14      | 25            | pemilik definisi               |
| `docs/analyst/requirements-matrix.md`       | 14      | 98            | kolom NFR + § NFR Traceability |
| `docs/analyst/SRS.md`                       | 14      | 252           | tabel ringkasan §NFR           |

### PF (11 id)

| Dokumen yang menyitir                                            | Id unik | Baris pertama | Catatan                        |
|-----------------------------------------------------------------|---------|---------------|--------------------------------|
| `docs/analyst/process-flows.md`                                 | 11      | 7             | pemilik definisi               |
| `docs/analyst/use-cases/INDEX-Use-Cases-Fofa-Shop.md`           | 9       | 28            | **minus PF-008 & PF-011** (OD-03) |
| `…/BAB-2-Payment-GW-Financial.md`                               | 3       | 7             |                                |
| `…/BAB-3-Admin-Panel-Fulfillment.md`                            | 3       | 7             |                                |
| `…/BAB-4-Reseller-API.md`                                       | 3       | 7             |                                |
| `…/BAB-1-…Customer-Journey.md`                                  | 2       | 7             |                                |
| `…/BAB-5-System-Administration.md`                              | 2       | 7             |                                |
| `docs/analyst/business-rules.md`                                | 1       | 506           | `BR-FUL-004` → PF-009          |
| `docs/analyst/requirements-matrix.md`                           | 1       | 242           | hanya di teks risk (OD-04)      |
| `docs/analyst/SRS.md`                                           | 0       | —             | (OD-04)                        |

### ADR (19 id)

| Dokumen yang menyitir                       | Id unik | Baris pertama | Catatan                              |
|---------------------------------------------|---------|---------------|--------------------------------------|
| 19 file ADR (heading `# ADR-NNNN:`)     | 19      | 1             | pemilik definisi (`# ADR-NNNN:`)     |
| `docs/analyst/requirements-matrix.md`       | 19      | 227           | pertama di bullet Coverage `:227`; § ADR Traceability `:246`, baris `:250-268` |
| issue tracker internal `01-…md` (tidak dipublikasikan) | 5       | 26            | ADR-0001…ADR-0005                    |
| `docs/analyst/extraction-report.md`         | 2       | 134           | ADR-0013, ADR-0015                   |
| issue tracker internal `02-…md` (tidak dipublikasikan) | 1       | 19            | ADR-0001                             |
| spec internal (tidak dipublikasikan)        | 1       | 851           | ADR-0006                             |
| `docs/analyst/assumptions-constraints.md`   | 1       | 46            | ADR-0006 (OQ-001)                    |
| `ADR-0019 (role-hierarchy)`           | 3       | 1             | diri-sendiri + cross-ref 0014/0015   |
| `docs/agents/domain.md`                     | 1       | 51            | **contoh boilerplate, bukan rujukan nyata** (OD-11) |

### GAP / CON / ASM / OQ

| Id    | Definisi di                                | Disitir oleh                                            |
|-------|--------------------------------------------|---------------------------------------------------------|
| GAP   | `nfr.md` (12 id)                           | `SRS.md:248,254` (3 id, salah satunya rentang `GAP-01…GAP-12`), `requirements-matrix.md:240` (`GAP-08`), `assumptions-constraints.md:79` (`GAP-09`) |
| CON   | `assumptions-constraints.md:23-27` (5 id)  | `SRS.md:295` (rentang `CON-001 … CON-005`), `requirements-matrix.md:241` (beberapa) |
| ASM   | `assumptions-constraints.md:35-38` (4 id)  | `SRS.md:303` (rentang `ASM-001 … ASM-004`), `requirements-matrix.md:241`     |
| OQ    | `assumptions-constraints.md:46-48` (3 id)  | `SRS.md:311` (rentang `OQ-001 … OQ-003`), `requirements-matrix.md:241`       |

---

## Resolved Numbering Defects (history)

Enam defect berikut **sudah diperbaiki**; masing-masing diverifikasi terhadap file saat ini
(bukan dari ingatan) dengan bukti yang dicantumkan.

1. **Duplikat nomor ADR — dua file `0014`.**
   *Sebelum:* `ADR-0014 (redis-queue-cache-session)` **dan** `ADR-0014 (role-hierarchy)`
   keduanya bernomor `0014`.
   *Sesudah:* file role-hierarchy direnumber menjadi `ADR-0019 (role-hierarchy)`.
   **Bukti:** `git status --porcelain` di direktori ADR →
   `RM 0014-role-hierarchy.md -> 0019-role-hierarchy.md`;
   `git log --all --name-status` memuat `A 0014-role-hierarchy.md` dan
   `A 0014-redis-queue-cache-session.md`.
   Rujukan basi sudah dipindahkan: `docs/roles.md:3,336`, `ADR-0007 (owner-separate-role):160`,
   `ADR-0015 (financial-data-access-control):203`,
   issue tracker internal `25-admin-pdf-export.md:66` (tidak dipublikasikan) → semuanya menyebut `0019`.
   `grep -rn "0014-role-hierarchy" . --include=*.md` → **0 hit** (bersih).

2. **Duplikat subbab PRD — dua `### 7.9`.**
   *Sebelum:* `PRD.md` memuat dua heading `### 7.9`.
   *Sesudah:* §7 direnumber `7.1`–`7.12`.
   **Bukti:** `grep -nE "^### 7\.[0-9]+" PRD.md` → 12 heading, urutan 7.1, 7.2, … 7.12;
   `grep -cE "^### 7\.9" PRD.md` → **1**.

3. **NFR menggantung (dangling) — dikutip, tak didefinisikan.**
   *Sebelum:* `requirements-matrix.md` menyitir `NFR-PERF-001`, `NFR-SEC-001`, `NFR-SEC-002`
   yang belum punya definisi di dokumen mana pun.
   *Sesudah:* `docs/analyst/nfr.md` dibuat dan mendefinisikan 14 id.
   **Bukti:** `nfr.md:44` (`### NFR-PERF-001`), `nfr.md:87` (`### NFR-SEC-001`),
   `nfr.md:113` (`### NFR-SEC-002`); cek silang repo-wide menghasilkan
   `cited-but-NOT-defined=[]` untuk seluruh keluarga `NFR`.

4. **Id BR dikutip use case tapi tak pernah didefinisikan.**
   *Sebelum:* `BR-PROD-001`, `BR-REF-002`, `BR-REF-003` dirujuk tapi tidak ada blok `### BR-` nya;
   satu id (`BR-PROD-001`) dipakai untuk dua aturan berbeda sehingga dipecah.
   *Sesudah:* keduanya ada — `BR-PROD-001` (baris 28) + `BR-PROD-002` (baris 60) untuk dua aturan
   storefront; `BR-REF-002` (baris 798), `BR-REF-003` (baris 829).
   **Bukti:** `grep -nE "^### BR-(PROD|REF)-" docs/analyst/business-rules.md` → 5 heading;
   cek silang `BR cited but NOT defined = []`.

5. **Id UC tidak ada — `UC-029`.**
   *Sebelum:* `BR-DEL-001`/`BR-DEL-002` menyitir `UC-029`, padahal maksimum `UC-023`.
   *Sesudah:* keduanya menyitir `UC-013` (CRUD Produk, Kategori & Subkategori).
   **Bukti:** `grep -nE "^### BR-DEL-00[12]" …` → baris 676 & 700, masing-masing `- Use Case: UC-013`;
   `grep -rn "UC-029" . --include=*.md` → **0 hit** di seluruh repo.

6. **Baris `Use Case:` basi di `business-rules.md` (25 dari 29 salah tunju).**
   *Sebelum:* 25 dari 29 baris `- Use Case:` menunjuk UC yang salah setelah reorganisasi chapter.
   *Sesudah:* dikoreksi; jumlah baris `Use Case:` kini **38** (satu per blok BR).
   **Verifikasi batas:** klaim historis "25 dari 29" **tidak bisa dibuktikan lewat git** —
   `git ls-files docs/analyst` kosong (seluruh `docs/analyst/` berstatus untracked), jadi tidak ada
   revisi yang bisa dibedah. Yang **bisa** diverifikasi adalah keadaan sekarang: membandingkan
   baris `Use Case:` dengan kolom **UC (chapter)** di § Business Rules Coverage matrix
   → **re-verifikasi 2026-09-26 (tahap akhir): 30 baris `Use Case:` tervalidasi penuh** (setiap UC
   yang disebut benar-benar mencantumkan BR itu di bagian `### Business Rules` chapter-nya),
   **7 baris berupa gap jujur `—`** (BR-PRICE-002, BR-WAL-004, BR-WF-003, BR-VOUCH-002,
   BR-VOUCH-003, BR-AUTH-001, BR-FUL-004 — tak dikutip chapter mana pun, dinyatakan terus terang
   alih-alih mengarang rujukan), dan **1 baris `BR-PAY-002` Superseded** yang tak lagi mengklaim UC.
   Total 38. Klaim lama "33 dari 38 cocok persis" sudah basi dan diganti angka di atas.

---

## Known Numbering Defects (open)

Dicek dengan cek silang otomatis (definisi vs sitasi repo-wide) pada 2026-09-26 09:50:

```text
FR : defined=38 cited=38 cited-but-NOT-defined=[]  orphans=[]
UC : defined=23 cited=23 cited-but-NOTdefined=[]   orphans=[]
PF : defined=11 cited=11 cited-but-NOT-defined=[]  orphans=[]
NFR: defined=14 cited=14 cited-but-NOT_defined=[]  orphans=[]
ADR: defined=19 cited=19 cited-but-NOT_DEFINED=[]  orphans=[]
BR : defined=38 cited=38 cited-but-NOT_DEFINED=[]  orphans=[]
GAP: defined=12 cited=12 cited-but-NOT_defined=[]  orphans=[]
CON: defined=5  cited=5  cited-but-NOT_defined=[]  orphans=[]
ASM: defined=4  cited=4  cited-but-NOT_defined=[]  orphans=[]
OQ : defined=3  cited=3  cited-but-NOT_defined=[]  orphans=[]
```

Artinya: **tidak ada id yang dirujuk tapi tak didefinisikan, tidak ada id yatim (duplikat pun nihil).**
Defect yang tersisa bersifat *konsistensi konten antar-dokumen*, bukan celah penomoran:

| #    | Defect                                                        | Lokasi (bukti grep)                                                                 | Dampak                                                | Severity |
|------|---------------------------------------------------------------|--------------------------------------------------------------------------------------|--------------------------------------------------------|----------|
| OD-01 | `GAP-11` memuat klaim basi: "matrix menghitung Non-Functional = 4, hanya 3 id dirujuk" — kini matrix punya **14 baris NFR** (`requirements-matrix.md:195-208`) dan Coverage `Non-Functional 14` (`:221`, bullet `:226`) | `docs/analyst/nfr.md:310`                                                            | Catatan audit salah; pembaca dikira traceability belum beres | Medium   |
| OD-02 | Tabel §Traceability di `nfr.md` menandai 11 NFR `"Belum dirujuk … belum ditambahkan pemilik matrix"` padahal matrix sudah punya pemilik FR untuk 11 id itu | `docs/analyst/nfr.md:273-286` vs `requirements-matrix.md:195-208`                     | Dua sumber kebenaran bertentangan tentang status rujukan | Medium   |
| OD-03 | `INDEX-Use-Cases-Fofa-Shop.md` tidak menyitir **`PF-008`** dan **`PF-011`** — grep daftar PF di INDEX hanya menghasilkan 9 id (`PF-001`…`PF-007`, `PF-009`, `PF-010`) | `docs/analyst/use-cases/INDEX-Use-Cases-Fofa-Shop.md:28-78`; `grep -oE "\bPF-[0-9]{3}"` → 9 unik | 2 dari 11 process flow tak punya jalur masuk dari index | Medium   |
| OD-04 | Keluarga `PF` tidak direpresentasikan di `requirements-matrix.md` (1 id, itu pun hanya di teks risk) dan **0 id di `SRS.md`** — tidak ada baris PF di Coverage Summary | `grep -ohE '\bPF-[0-9]{3}\b' docs/analyst/requirements-matrix.md \| sort -u \| wc -l` → 1 ; di `SRS.md` → 0 | PF tak ter-trace dari sisi requirement                | Medium   |
| OD-05 | `SRS.md` **0 id UC** dan **0 id BR**; chapter use case **0 id FR** — traceability FR↔UC↔BR hanya hidup di `requirements-matrix.md` | per-hitungan pada § Cross-Document Id Usage                     | Jika matrix hilang/drift, tak ada dokumen kedua yang menjembatani | Low      |
| OD-06 | 5 baris `Use Case:` di `business-rules.md` tidak didukung kutipan chapter mana pun (kolom UC (chapter) matrix = `—`): `BR-PRICE-002`(UC-005), `BR-WAL-004`(UC-007), `BR-WF-003`(UC-008), `BR-WF-001`(UC-008/014/015 di luar UC-005), `BR-PAY-002`(UC-005, sudah Superseded) | `business-rules.md:121,223,367,319,534` vs `requirements-matrix.md` § BR Coverage     | Klaim rujukan UC tak bisa diverifikasi dari chapter    | Low      |
| OD-07 | Ketidakkonsistenan id-level: `BR-WF-003` (Pending Expire Cron) menyebut `Use Case: UC-008` (Payment Gateway Callback) sedangkan pemetaan matrix `FR-006 → UC-006` (View Invoice Status) | `business-rules.md:367` (baris `Use Case: UC-008` di baris ~386); `requirements-matrix.md` baris `FR-006` | Rujukan UC salah sasaran untuk aturan expire cron      | Low      |
| OD-08  | `erd.md` hanya menyitir **27 dari 38** BR — 11 tidak muncul: `BR-AUTH-002`, `BR-AUTH-004`, `BR-GAME-001`, `BR-GAME-002`, `BR-MC-001`, `BR-PAY-002`, `BR-PRICE-002`, `BR-REF-001`, `BR-REF-002`, `BR-REF-003`, `BR-WF-004` | `grep -ohE '\bBR-[A-Z]+-[0-9]+\b' docs/analyst/erd.md \| sort -u \| wc -l` → 27       | Sebagian aturan (mis. refund, game-id) tak terlihat dari ERD | Low      |
| OD-09  | Tabel `## UC Numbering Mapping` tidak punya baris `Old: —` tersendiri untuk `UC-023` (hanya muncul di dalam split `UC-019`), padahal `UC-022` punya baris sendiri | `requirements-matrix.md:7` baris 11–33 (data 13–33)                                 | Inkonsistensi format tabel mapping                     | Low      |
| OD-10  | Urutan heading FR di `SRS.md` tidak monoton: `FR-034` (baris 165) & `FR-036` (baris 170) muncul sebelum `FR-024` (baris 177); `FR-035` (baris 207) di antara `FR-029` dan `FR-030` — **hanya kosmetik, himpunan id lengkap 001–038** | `grep -nE "^#### FR-" docs/analyst/SRS.md`                                          | Reviewer bisa mengira ada celah saat membaca cepat     | Info     |
| OD-11  | `docs/agents/domain.md:51` memakai contoh "`ADR-0007 (event-sourced orders)`" padahal `ADR-0007` repo = *Owner as Separate Role* | `grep -n "ADR-0007" docs/agents/domain.md`                                          | Bukan rujukan nyata (boilerplate agent), tapi bisa menipu pemeriksaan otomatis | Info     |
| OD-12  | `docs/analyst/extraction-report.md:84` masih menyebut jalur lama `use-cases/UC-001-browse-homepage.md` dan "20 files", sedangkan struktur kini 5 file chapter berisi 23 UC | `grep -n "UC-001-browse-homepage" docs/analyst/extraction-report.md`                | Dokumen historis menunjuk artefak yang tak ada          | Info     |
| OD-13  | Seluruh `docs/analyst/` **untracked di git** (`git status --porcelain docs/analyst` → `?? docs/analyst/`) sehingga history perbaikan penomoran (klaim "25 dari 29", dsb.) tak dapat diaudit ulang | `git ls-files docs/analyst` → kosong                                                 | Audit trail defect hilang; risiko regresi tak terdeteksi | Medium   |

> Tabel di atas adalah **snapshot awal (09:50)** dan dipertahankan apa adanya sebagai jejak temuan.
> Status terkini per baris ada di § Rekonsiliasi Status di bawah — sebagian sudah diperbaiki,
> sebagian ternyata **false flag** (hasil pembacaan snapshot lama saat dokumen sedang diedit).

### Rekonsiliasi Status (re-verifikasi 2026-09-26, tahap akhir)

| OD    | Status kini     | Bukti verifikasi ulang (grep pada file saat ini) |
|-------|-----------------|--------------------------------------------------|
| OD-01 | **FALSE FLAG**  | `nfr.md:331` (GAP-11) kini bertuliskan *"Selesai — matrix § NFR Traceability kini mencakup seluruh 14 id"*. Klaim "4 vs 3" sudah diganti; pembacaan awal mengambil snapshot basi |
| OD-02 | **FALSE FLAG**  | `grep -c "Belum dirujuk" docs/analyst/nfr.md` → **0** |
| OD-03 | **Selesai**      | `grep -oE 'PF-[0-9]{3}' INDEX… \| sort -u \| wc -l` → **11/11**; section `## Process Flow Coverage` ditambahkan di INDEX |
| OD-04 | **Parsial**      | matrix kini punya baris `requirements-matrix.md:256` → `| Process Flows | 11 | 11 | 0 | 0 | 0 |` + section `## Process Flow Traceability`. **`SRS.md` masih 0 id PF** — PF sengaja ditelusuri lewat matrix, bukan SRS |
| OD-05 | **Dibiarkan (desain)** | FR↔UC↔BR sengaja dijadikan satu hub di `requirements-matrix.md`; menduplikasi ke SRS/chapter justru memunculkan risiko drift (Check 2 redundansi). Dicatat sebagai keputusan, bukan defect |
| OD-06 | **Selesai**      | 5 baris diserahkan pada gap jujur: `BR-WF-003` → `— (… dimiliki PF-008)`; `BR-PRICE-002` → `— (… diimplementasi PF-006)`; `BR-WAL-004` → `— (… diimplementasi PF-005)`; `BR-WF-001` dipangkas jadi `UC-005` saja (UC-008/014/015 dibuang karena tak mencantumkannya); `BR-PAY-002` → `— (Superseded…)` |
| OD-07 | **Selesai**      | `BR-WF-003` tak lagi menunjuk UC-008; pemiliknya dinyatakan `PF-008 (Expire Pending Transactions — Scheduled)` sesuai `process-flows.md:488` |
| OD-08 | **Selesai**      | `grep -ohE '\bBR-[A-Z]+-[0-9]+\b' erd.md \| sort -u \| wc -l` → **37/38**; satu-satunya yang absen = `BR-PAY-002` (Superseded, sengaja tidak dipetakan ke entity `payments`). `BR-GAME-001/002` kini punya baris `Data Entity: transactions` dan tercatat di `erd.md` entri transactions (`target_game_id`, erd:46/290) |
| OD-09 | **Selesai**      | Catatan eksplisit `requirements-matrix.md:35` "Catatan (OD-09): `UC-023` sengaja tidak punya baris `Old: —` sendiri karena ia bukan UC baru" |
| OD-10 | **Masih terbuka (Info)** | Urutan heading FR di `SRS.md` tetap non-monoton; himpunan id lengkap `001`–`038`. Hanya kosmetik — tidak diperbaiki agar tak menggeser ratusan rujukan baris |
| OD-11 | **Selesai**      | `docs/agents/domain.md:51` kini memakai `ADR-0003 (transaction status flow)` — nyata, sesuai `ADR-0003 (transaction-status-flow)` |
| OD-12 | **Selesai**      | `grep -c "UC-001-browse-homepage" extraction-report.md` → **0**; inventaris disesuaikan ke 5 file chapter / 23 UC, klaim `42 migrations` → **43** (`ls database/migrations \| wc -l`) |
| OD-13 | **Masih terbuka (Medium)** | `git ls-files docs/analyst` → 0. Sesuai kebijakan repo, perubahan sengaja **tidak di-commit** kecuali diminta eksplisit — jadi audit trail tetap hilang sampai user memerintahkan commit |

**Defect ikutan yang ditemukan saat rekonsiliasi (belum ada di tabel awal):**

| ID     | Temuan | Status |
|--------|--------|--------|
| OD-14  | Label `**Process Flow Related:**` di header chapter salah: BAB-3 menulis `PF-007 (Reseller Auto-Refund by Admin)` (PF-007 = Reseller API Top-up), BAB-4 menulis `PF-009 (Reseller API Top-up)` (PF-009 = Sync) & `PF-011 (Scheduled Sync)` (PF-011 = Product Deletion), BAB-5 menulis `PF-003 (Admin Actions)` | **Selesai** — label disamakan persis dengan judul `process-flows.md`; pemetaan chapter diturunkan dari kepemilikan UC (PF-011→UC-013/BAB-3, PF-007→UC-018/BAB-4, PF-010→UC-022/BAB-5). Tersisa 0 mismatch |
| OD-15  | `extraction-report.md` menyebut `42 migrations` padahal `ls database/migrations \| wc -l` → 43 | **Selesai** |

**Tidak ditemukan (terbukti grep):** celah sekuens di `FR`/`UC`/`PF`/`ADR`/`NFR`/`GAP`/`CON`/`ASM`/`OQ`,
duplikat id di file mana pun, id yang dirujuk tapi tak dedefinisikan, dan id yang didefinisikan tapi
tak pernah disitir di luar file pemiliknya.

### Catatan status konflik BR (untuk reviewer)

Konflik `BR-CHECK-001` / `BR-PAY-002` / `BR-FUL-004` yang sedang diselesaikan **sudah
tertutup pada snapshot akhir** — diverifikasi dengan grep:

- `business-rules.md:961` → `### BR-CHECK-001: Checkout Rate Limiting` (kanonik, `[x] Confirmed`).
- `business-rules.md:487` → `### BR-FUL-004: Sync Waiting Fulfillment` (**id baru**, cron 5 menit,
  `Use Case: — (… dimiliki PF-009 di process-flows.md)`).
- `business-rules.md:534` → `### BR-PAY-002` ber-status `Superseded by BR-CHECK-001`, id ditahan
  agar tidak dipakai ulang.
- `requirements-matrix.md:149` → "Semua 38 `### BR-` (37 aktif + 1 superseded)"; `:193` → "38/38";
  `:260` → "BR 38 … Confirmed 37 + N/A 1"; baris coverage `BR-CHECK-001` = *Checkout rate limit
  3 dimensi*; baris `BR-FUL-004` → `FR-011`.
- `requirements-matrix.md:278` → entri risk ditandai **RESOLVED**.
- `erd.md:313` → memuat `BR-FUL-004` dan `BR-CHECK-001`.
- `use-cases/chapters/BAB-1-…md:334` → `BR-CHECK-001` = rate limit (kini cocok dengan definisi).

**Penutupan sisa implikasi (re-verifikasi tahap akhir):** kedua butir yang dulu tercatat sebagai
**OD-06** dan **OD-08** kini **tertutup** —

- `BR-PAY-002` **tidak lagi memuat baris `- Use Case: UC-005`**; kini berbunyi
  `- Use Case: — (Superseded by BR-CHECK-001; UC-005 dimiliki BR-CHECK-001)`, jadi tidak ada lagi
  dua BR yang sama-sama mengklaim UC-005.
- `BR-PAY-002` **memang sengaja tetap absen dari `erd.md`** (entity `payments` tidak menampung
  aturan rate limit checkout — itu milik `transactions`), sehingga ERD kini menyitir **37/38 BR**
  dan satu-satunya yang tak disitir adalah BR superseded itu sendiri.

Keseluruhan penomoran ditutup pada: **FR 38 · BR 38 (37 aktif + 1 superseded) · UC 23 · PF 11 ·
NFR 14 · ADR 19** — 0 id yatim, 0 id menggantung, 0 tabel markdown tidak sejajar, 0 mismatch
label PF antara header chapter dan `process-flows.md`.
