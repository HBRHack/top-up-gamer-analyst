# Requirements Traceability Matrix — Fofa Shop

Traceability dari requirements → use cases → business rules → status.

---

## UC Numbering Mapping

Old → New mapping setelah reorganisasi ke chapter structure:

| Old UC | New UC | Chapter | Nama |
|--------|--------|---------|------|
| UC-001 | UC-001 | BAB 1 | Browse Kategori & Produk (merged with old UC-002) |
| UC-002 | UC-001 | BAB 1 | (merged into UC-001) |
| UC-003 | UC-002 | BAB 1 | Detail Produk + Harga Per Role |
| UC-004 | UC-003 | BAB 1 | Search Produk |
| UC-005 | UC-004 | BAB 1 | Verifikasi ID Game |
| UC-006 | UC-005 | BAB 1 | Checkout & Voucher |
| UC-007 | UC-006 | BAB 1 | View Invoice |
| UC-008 | UC-007 | BAB 1 | Riwayat Transaksi & Refund |
| UC-009 | UC-008 | BAB 2 | Payment Callback |
| UC-010 | UC-017 | BAB 4 | List Produk API |
| UC-011 | UC-018 | BAB 4 | Top-up API |
| UC-012 | UC-019 | BAB 4 | Cek Status API |
| UC-013 | UC-009 + UC-020 | BAB 2 + BAB 4 | Cek Saldo (split) |
| UC-014 | UC-012 | BAB 3 | Dashboard Admin |
| UC-015 | UC-013 + UC-014 | BAB 3 | CRUD + Retry (split) |
| UC-016 | UC-015 | BAB 3 | Manual Fulfill Minecraft |
| UC-017 | UC-016 | BAB 3 | Kelola Refund |
| UC-018 | UC-010 | BAB 2 | Kelola Deposit Wallet |
| UC-019 | UC-011 + UC-023 | BAB 2 + BAB 5 | Laporan + Toggle Payment (split) |
| UC-020 | UC-021 | BAB 5 | Kelola Admin |
| — | UC-022 | BAB 5 | Broadcast Pengumuman (new) |

> **Catatan (OD-09):** `UC-023` sengaja tidak punya baris `Old: —` sendiri karena ia **bukan UC baru** —
> ia pecahan dari old `UC-019` ("Laporan + Toggle Payment") dan sudah tercakup di baris
> `UC-019 | UC-011 + UC-023`. Kolom *Old UC* dipakai sebagai key unik (setiap old id muncul tepat
> sekali), jadi baris `—` hanya untuk UC yang benar-benar tidak punya old id, yaitu `UC-022`.
> Sumber: `NUMBERING-LOG.md` baris `UC-023 … pecahan old UC-019` + `docs/analyst/SRS.md:140`
> (`#### FR-019: Toggle Payment Method`) dan user story spec `#76/#77` (toggle payment methods);
> chapter `BAB-5-System-Administration.md:135` = `## UC-023: Toggle Payment Method`.

---

## Requirements by Priority

### Must Have

| ID | Requirement | Use Case | Business Rules | Status |
|----|-------------|----------|----------------|--------|
| FR-001 | Browse kategori game | UC-001 (BAB-1) | BR-PROD-001, BR-PROD-002 | Confirmed |
| FR-002 | Search produk | UC-003 (BAB-1) | — | Confirmed |
| FR-003 | Cek nickname/ID game | UC-004 (BAB-1) | BR-GAME-001, BR-GAME-002 | Confirmed |
| FR-004 | Detail produk + harga per role | UC-002 (BAB-1) | BR-PRICE-001 | Confirmed |
| FR-005 | Checkout + validasi | UC-005 (BAB-1) | BR-WF-001, BR-CHECK-001 | Confirmed |
| FR-006 | Invoice page | UC-006 (BAB-1) | BR-WF-002, BR-WF-003 | Confirmed |
| FR-008 | Riwayat transaksi | UC-007 (BAB-1) | — | Confirmed |
| FR-009 | List produk (API) | UC-017 (BAB-4) | BR-PRICE-001 | Confirmed |
| FR-010 | Top-up via API | UC-018 (BAB-4) | BR-WAL-001, BR-FUL-003 | Confirmed |
| FR-011 | Cek status (API) | UC-019 (BAB-4) | BR-FUL-004 | Confirmed |
| FR-012 | Cek saldo (API) | UC-020 (BAB-4) | — | Confirmed |
| FR-014 | Dashboard monitoring | UC-012 (BAB-3) | BR-AUTH-001, BR-AUTH-003 | Confirmed |
| FR-015 | CRUD produk/kategori | UC-013 (BAB-3) | — | Confirmed |
| FR-016 | Retry transaksi gagal | UC-014 (BAB-3) | BR-FUL-002 | Confirmed |
| FR-017 | Manual fulfill Minecraft | UC-015 (BAB-3) | BR-FUL-001, BR-MC-001 | Confirmed |
| FR-018 | Kelola wallet reseller | UC-010 (BAB-2) | BR-WAL-001, BR-WAL-002 | Confirmed |
| FR-024 | View omset | UC-012 (BAB-3) | BR-AUTH-003 | Confirmed |
| FR-025 | Laporan keuangan | UC-011 (BAB-2) | BR-AUTH-003 | Dropped |
| FR-028 | Kelola admin | UC-021 (BAB-5) | BR-AUTH-002, BR-AUTH-004 | Confirmed |
| FR-029 | Hapus produk (unrestricted) | UC-013 (BAB-3) | BR-DEL-001, BR-DEL-002, BR-DEL-003 | Confirmed |
| FR-031 | Email notifikasi | UC-022 (BAB-5) | BR-ANN-002 | Confirmed |
| FR-033 | MC credential reveal | UC-015 (BAB-3) | BR-MC-001 | Confirmed |
| FR-034 | Transaction chart | UC-011 (BAB-2) | — | Confirmed |
| FR-035 | Financial data access control | UC-011 (BAB-2) | BR-AUTH-003 | Confirmed |
| FR-037 | Guest email saat checkout | UC-005 (BAB-1) | — | Confirmed |

### Should Have

| ID | Requirement | Use Case | Business Rules | Status |
|----|-------------|----------|----------------|--------|
| FR-007 | Refund request | UC-007 (BAB-1) | BR-REF-001, BR-REF-002, BR-WAL-003, BR-WAL-004 | Confirmed |
| FR-013 | Webhook callback | UC-008 (BAB-2) | BR-PAY-001, BR-WF-002, BR-WF-004 | Confirmed |
| FR-019 | Toggle payment method | UC-023 (BAB-5) | BR-PAY-003 | Confirmed |
| FR-020 | Broadcast pengumuman | UC-022 (BAB-5) | BR-ANN-001 | Confirmed |
| FR-021 | Audit log | UC-012 (BAB-3) | — | Confirmed |
| FR-022 | Kelola voucher | UC-005 (BAB-1) | BR-VOUCH-001, BR-VOUCH-002, BR-VOUCH-003 | Confirmed |
| FR-023 | Manage refund requests | UC-016 (BAB-3) | BR-REF-001, BR-REF-003 | Confirmed |
| FR-026 | Export PDF | UC-011 (BAB-2) | BR-AUTH-003 | Confirmed |
| FR-030 | Voucher diskon | UC-005 (BAB-1) | BR-PRICE-002, BR-VOUCH-001 | Confirmed |
| FR-036 | Product image upload | UC-013 (BAB-3) | — | Confirmed |

### Could Have

| ID | Requirement | Use Case | Business Rules | Status |
|----|-------------|----------|----------------|--------|
| FR-027 | Export pajak | UC-011 (BAB-2) | — | Dropped |
| FR-032 | Notification preference | UC-022 (BAB-5) | BR-ANN-002 | Confirmed |
| FR-038 | Theme toggle (terang/gelap) | — | — | Confirmed |

---

## Full Traceability Matrix

| ID | Requirement | Use Case | Chapter | Business Rules | Non-Functional | Status |
|----|-------------|----------|---------|----------------|----------------|--------|
| FR-001 | Browse kategori | UC-001 | BAB-1 | BR-PROD-001, BR-PROD-002 | NFR-PERF-003, NFR-SCP-004 | Confirmed |
| FR-002 | Search produk | UC-003 | BAB-1 | — | NFR-PERF-003, NFR-SEC-003, NFR-SCP-004 | Confirmed |
| FR-003 | Cek game ID | UC-004 | BAB-1 | BR-GAME-001, BR-GAME-002 | NFR-PERF-001 | Confirmed |
| FR-004 | Detail produk | UC-002 | BAB-1 | BR-PRICE-001 | NFR-PERF-003 | Confirmed |
| FR-005 | Checkout | UC-005 | BAB-1 | BR-WF-001, BR-CHECK-001 | NFR-SEC-001, NFR-SEC-003 | Confirmed |
| FR-006 | Invoice | UC-006 | BAB-1 | BR-WF-002, BR-WF-003 | — | Confirmed |
| FR-007 | Refund request | UC-007 | BAB-1 | BR-REF-001, BR-REF-002, BR-WAL-003, BR-WAL-004 | — | Confirmed |
| FR-008 | Riwayat transaksi | UC-007 | BAB-1 | — | — | Confirmed |
| FR-009 | List produk API | UC-017 | BAB-4 | BR-PRICE-001 | — | Confirmed |
| FR-010 | Top-up API | UC-018 | BAB-4 | BR-WAL-001, BR-FUL-003 | NFR-SEC-002, NFR-AVL-001, NFR-AVL-003, NFR-SCP-002 | Confirmed |
| FR-011 | Cek status API | UC-019 | BAB-4 | BR-FUL-004 | NFR-AVL-004 | Confirmed |
| FR-012 | Cek saldo API | UC-020 | BAB-4 | — | — | Confirmed |
| FR-013 | Webhook callback | UC-008 | BAB-2 | BR-PAY-001, BR-WF-002, BR-WF-004 | NFR-AVL-002, NFR-SCP-002 | Confirmed |
| FR-014 | Dashboard admin | UC-012 | BAB-3 | BR-AUTH-001, BR-AUTH-003 | — | Confirmed |
| FR-015 | CRUD produk | UC-013 | BAB-3 | — | — | Confirmed |
| FR-016 | Retry transaksi | UC-014 | BAB-3 | BR-FUL-002 | NFR-AVL-001, NFR-AVL-004 | Confirmed |
| FR-017 | Manual fulfill MC | UC-015 | BAB-3 | BR-FUL-001, BR-MC-001 | — | Confirmed |
| FR-018 | Wallet reseller | UC-010 | BAB-2 | BR-WAL-001, BR-WAL-002 | NFR-AVL-003 | Confirmed |
| FR-019 | Toggle payment | UC-023 | BAB-5 | BR-PAY-003 | — | Confirmed |
| FR-020 | Broadcast | UC-022 | BAB-5 | BR-ANN-001 | — | Confirmed |
| FR-021 | Audit log | UC-012 | BAB-3 | — | — | Confirmed |
| FR-022 | Kelola voucher | UC-005 | BAB-1 | BR-VOUCH-001, BR-VOUCH-002, BR-VOUCH-003 | — | Confirmed |
| FR-023 | Manage refund | UC-016 | BAB-3 | BR-REF-001, BR-REF-003 | — | Confirmed |
| FR-024 | View omset | UC-012 | BAB-3 | BR-AUTH-003 | — | Confirmed |
| FR-025 | Laporan keuangan | UC-011 | BAB-2 | BR-AUTH-003 | — | Dropped |
| FR-026 | Export PDF | UC-011 | BAB-2 | BR-AUTH-003 | — | Confirmed |
| FR-027 | Export pajak | UC-011 | BAB-2 | — | — | Dropped |
| FR-028 | Kelola admin | UC-021 | BAB-5 | BR-AUTH-002, BR-AUTH-004 | — | Confirmed |
| FR-029 | Hapus produk | UC-013 | BAB-3 | BR-DEL-001, BR-DEL-002, BR-DEL-003 | — | Confirmed |
| FR-030 | Voucher diskon | UC-005 | BAB-1 | BR-PRICE-002, BR-VOUCH-001 | — | Confirmed |
| FR-031 | Email notifikasi | UC-022 | BAB-5 | BR-ANN-002 | — | Confirmed |
| FR-032 | Notif preference | UC-022 | BAB-5 | BR-ANN-002 | — | Confirmed |
| FR-033 | MC credential | UC-015 | BAB-3 | BR-MC-001 | — | Confirmed |
| FR-034 | Transaction chart | UC-011 | BAB-2 | — | — | Confirmed |
| FR-035 | Financial data access control | UC-011 | BAB-2 | BR-AUTH-003 | — | Confirmed |
| FR-036 | Product image upload | UC-013 | BAB-3 | — | — | Confirmed |
| FR-037 | Guest email checkout | UC-005 | BAB-1 | — | — | Confirmed |
| FR-038 | Theme toggle (terang/gelap) | — | — | — | — | Confirmed |

---

## Business Rules Coverage

Semua 38 `### BR-` (37 aktif + 1 superseded) di `docs/analyst/business-rules.md` dipetakan ke baris matrix.
Kolom **FR (matrix)** = baris yang mencantumkannya di kolom *Business Rules*; kolom **UC (chapter)** = chapter use case yang mengutip id BR itu (bukan UC baris FR — lihat Full Traceability Matrix untuk UC per FR).

| BR ID | Rule (ringkas) | FR (matrix) | UC (chapter) | Status |
|-------|----------------|-------------|--------------|--------|
| BR-PROD-001 | Kategori coming soon tampil, beli disabled | FR-001 | UC-001 | Traced |
| BR-PROD-002 | Hanya produk `is_active` di storefront | FR-001 | UC-001 | Traced |
| BR-PRICE-001 | Harga per role | FR-004, FR-009 | UC-002, UC-017 | Traced |
| BR-PRICE-002 | Voucher discount cap (minimal bayar Rp1) | FR-030 | — | Traced (FR only) |
| BR-WAL-001 | Operasi wallet atomik (`lockForUpdate`) | FR-010, FR-018 | UC-010, UC-018 | Traced |
| BR-WAL-002 | Wallet auto-create balance 0 | FR-018 | UC-009, UC-010 | Traced |
| BR-WAL-003 | Unik `(reference_id, type)` anti double-refund | FR-007 | UC-009 | Traced |
| BR-WAL-004 | Reseller auto-refund ke wallet saat gagal | FR-007 | — | Traced (FR only) |
| BR-VOUCH-001 | Voucher validation chain (5 check) | FR-022, FR-030 | UC-005 | Traced |
| BR-VOUCH-002 | Voucher case-insensitive (`LOWER(code)`) | FR-022 | — | Traced (FR only) |
| BR-VOUCH-003 | Voucher atomic apply (`lockForUpdate`) | FR-022 | — | Traced (FR only) |
| BR-WF-001 | State machine transaksi | FR-005 | UC-005 | Traced |
| BR-WF-002 | Payment callback idempotent (`ref_id` unique) | FR-006, FR-013 | UC-008 | Traced |
| BR-WF-003 | Expire cron transaksi pending (1 menit) | FR-006 | — | Traced (FR only) |
| BR-WF-004 | Reconciliation late payment (expired → paid) | FR-013 | UC-008 | Traced |
| BR-FUL-001 | Minecraft manual fulfillment | FR-017 | UC-015 | Traced |
| BR-FUL-002 | Fulfillment retry 3x backoff | FR-016 | UC-014 | Traced |
| BR-FUL-003 | Vendor mapping hilang → failed langsung | FR-010 | UC-018 | Traced |
| BR-FUL-004 | Sync cron `waiting_fulfillment` 5 menit (last 24h, limit 100) | FR-011 | — | Traced (FR only) |
| BR-PAY-001 | Signature HMAC-SHA256 timing-safe | FR-013 | UC-008 | Traced |
| BR-PAY-002 | Checkout rate limit 3 dimensi (duplikat BR-CHECK-001) | FR-005 | — | Superseded |
| BR-PAY-003 | Toggle metode pembayaran | FR-019 | UC-023 | Traced |
| BR-AUTH-001 | RoleMiddleware 403 + blokir user inactive | FR-014 | — | Traced (FR only) |
| BR-AUTH-002 | Hanya owner buat/toggle admin | FR-028 | UC-021 | Traced |
| BR-AUTH-003 | Financial data access control (column exclusion) | FR-014, FR-024, FR-025 *(Dropped)*, FR-026, FR-035 | UC-011, UC-012 | Traced |
| BR-AUTH-004 | Self-protection (tidak bisa toggle diri sendiri) | FR-028 | UC-021 | Traced |
| BR-DEL-001 | Produk pernah dibeli → selalu soft delete | FR-029 | UC-013 | Traced |
| BR-DEL-002 | Hard delete ≤24h + owner + tanpa transaksi | FR-029 | UC-013 | Traced |
| BR-DEL-003 | Setiap delete → ProductDeletionLog + snapshot | FR-029 | UC-013 | Traced |
| BR-MC-001 | Password Minecraft ter-encrypt | FR-017, FR-033 | UC-015 | Traced |
| BR-REF-001 | Reseller vs customer refund | FR-007, FR-023 | UC-007 | Traced |
| BR-REF-002 | Reseller auto-refund tanpa manual request | FR-007 | UC-007 | Traced |
| BR-REF-003 | Refund selesai wajib sudah ada request | FR-023 | UC-016 | Traced |
| BR-GAME-001 | Verifikasi game ID: API → fallback regex | FR-003 | UC-004 | Traced |
| BR-GAME-002 | Validasi regex lokal per edisi | FR-003 | UC-004 | Traced |
| BR-ANN-001 | Audience pengumuman per `target_audience` | FR-020 | UC-022 | Traced |
| BR-ANN-002 | Aturan email transaksi vs announcement | FR-031, FR-032 | UC-022 | Traced |
| BR-CHECK-001 | Checkout rate limit 3 dimensi (per IP 5/mnt, per user 10/jam, per product+target 3/10mnt) | FR-005 | UC-005 | Traced |

**Rekap:** 38/38 BR (37 aktif + 1 superseded) punya kolom FR di tabel ini — **0 Untraced** (baris FR tidak ada yang `—`; catatan: sel FR-005 di § Full Traceability Matrix hanya memuat `BR-WF-001, BR-CHECK-001` karena `BR-PAY-002` sudah superseded dan memang tidak dicantumkan di sana). 30 BR aktif dikutip di chapter use case (`grep` kutipan BR di `use-cases/chapters/` → 30 id unik); 7 BR aktif hanya lewat baris FR (BR-PRICE-002, BR-WAL-004, BR-VOUCH-002, BR-VOUCH-003, BR-WF-003, BR-FUL-004, BR-AUTH-001) sehingga kolom UC-nya `—`; ditambah BR-PAY-002 (superseded) yang juga kolom UC-nya `—` (8 baris `—` di kolom UC).

---

## NFR Traceability

14 NFR didefinisikan di `docs/analyst/nfr.md`. 11 punya pemilik baris FR (kolom *Non-Functional* di Full Traceability Matrix); 3 system-wide sengaja tidak dipaksakan ke baris FR tertentu.

| NFR ID | Target singkat | FR pemilik (matrix) | Status |
|--------|----------------|---------------------|--------|
| NFR-PERF-001 | Response time < 200ms (cached) | FR-003 | Referenced |
| NFR-PERF-002 | Throughput ~60 req/s | — (system-wide) | System-wide |
| NFR-PERF-003 | Cache hit rate > 80% product listings | FR-001, FR-002, FR-004 | Referenced |
| NFR-SEC-001 | Keamanan jalur checkout | FR-005 | Referenced |
| NFR-SEC-002 | Autentikasi & otorisasi API reseller | FR-010 | Referenced |
| NFR-SEC-003 | Pencegahan SQL injection & XSS | FR-002, FR-005 | Referenced |
| NFR-AVL-001 | Queue retry 3 attempts per job | FR-010, FR-016 | Referenced |
| NFR-AVL-002 | Payment idempotency (ref_id + lock) | FR-013 | Referenced |
| NFR-AVL-003 | Wallet atomicity | FR-010, FR-018 | Referenced |
| NFR-AVL-004 | Scheduled tasks (expire 1 mnt, sync 5 mnt) | FR-011, FR-016 | Referenced |
| NFR-SCP-001 | Single VPS (2 vCPU, 2GB RAM) | — (system-wide) | System-wide |
| NFR-SCP-002 | Redis: cache + queue + session | FR-010, FR-013 | Referenced |
| NFR-SCP-003 | PHP-FPM 25 workers | — (system-wide) | System-wide |
| NFR-SCP-004 | Nginx static cache + gzip | FR-001, FR-002 | Referenced |

Pemetaan FR untuk NFR-PERF-003, SEC-003, AVL-001…004, SCP-002, SCP-004 mengikuti daftar kandidat di `docs/analyst/nfr.md` § Traceability — tidak ada link FR↔NFR yang ditambahkan di luar usulan dokumen itu.

---

## Process Flow Traceability

11 process flow didefinisikan di `docs/analyst/process-flows.md` (`grep -cE "^## PF-" docs/analyst/process-flows.md` → **11**).
Kolom **BR (implementasi)** = id BR yang mekanismenya muncul eksplisit di langkah / decision point / bagian `### Business Rules` flow itu.
`process-flows.md` tidak pernah menyitir FR (`grep -nE "FR-[0-9]{3}" docs/analyst/process-flows.md` → **0 hit**), jadi kolom **FR (matrix)** diturunkan dari baris § Business Rules Coverage yang memuat BR tersebut — bukan klaim langsung PF → FR.

| PF | Proses | FR (matrix, via BR) | BR (implementasi di flow) |
|----|--------|---------------------|---------------------------|
| PF-001 | Customer Checkout | FR-003, FR-005 | BR-WF-001, BR-CHECK-001, BR-GAME-001, BR-GAME-002 |
| PF-002 | Payment Callback | FR-006, FR-013 | BR-PAY-001, BR-WF-002, BR-WF-004 |
| PF-003 | Transaction Fulfillment | FR-010, FR-016, FR-017 | BR-FUL-001, BR-FUL-002, BR-FUL-003 |
| PF-004 | Minecraft Manual Fulfillment | FR-017, FR-033 | BR-FUL-001, BR-MC-001 |
| PF-005 | Reseller Auto-Refund | FR-007, FR-010, FR-018 | BR-REF-002, BR-WAL-004, BR-WAL-001 |
| PF-006 | Voucher Apply | FR-022, FR-030 | BR-VOUCH-001, BR-VOUCH-002, BR-VOUCH-003, BR-PRICE-002 |
| PF-007 | Reseller API Top-up | FR-010, FR-018 | BR-WAL-001 |
| PF-008 | Expire Pending Transactions (Scheduled) | FR-006 | BR-WF-003 |
| PF-009 | Sync Waiting Fulfillment (Scheduled) | FR-011 | BR-FUL-004 |
| PF-010 | Announcement Broadcast | FR-020, FR-031, FR-032 | BR-ANN-001, BR-ANN-002 |
| PF-011 | Product Deletion | FR-029 | BR-DEL-001, BR-DEL-002, BR-DEL-003 |

- FR turunan mengikuti sel FR di § Business Rules Coverage; mis. `BR-WF-002` tercatat di FR-006 **dan** FR-013 sehingga PF-002 memunculkan keduanya — bukan berarti callback = halaman invoice. Untuk dua cron (PF-008, PF-009) FR turunan = FR-006 / FR-011 hanya karena di baris itu BR-nya tercatat; kronya sendiri tidak dimiliki requirement mana pun.
- Judul PF memakai `process-flows.md` sebagai acuan. Header `Process Flow Related:` di chapter memakai nomor PF yang tidak selalu sinkron (mis. BAB 4 menulis `PF-009 (Reseller API Top-up)` padahal `PF-009` = Sync Waiting Fulfillment dan `PF-007` = Reseller API Top-up) — karena itu rujukan PF dari chapter tidak dipakai untuk menurunkan FR di atas.
- `process-flows.md` tidak memuat field status (0 hit `Confirmed`/`Dropped`/`Pending`) — lihat baris **Process Flows** di § Coverage Summary.

---

## Coverage Summary

| Area | Total | Confirmed | Dropped | Pending | N/A |
|------|-------|-----------|---------|---------|-----|
| Functional Requirements | 38 | 36 | 2 | 0 | 0 |
| Business Rules | 38 | 37 | 0 | 0 | 1 |
| Use Cases | 23 | 23 | 0 | 0 | 0 |
| Non-Functional | 14 | 11 | 0 | 0 | 3 |
| Process Flows | 11 | 11 | 0 | 0 | 0 |
| ADR | 19 | 16 | 3 | 0 | 0 |

- **FR 38** = 33 FR lama + FR-034…FR-038. **Dropped 2** = FR-025, FR-027 (spec v1.3 scope-cut) — baris tetap ada di SRS dan matrix, hanya statusnya berubah; **Confirmed 36** = 38 − 2.
- **BR 38** = 38 blok `### BR-` di `business-rules.md`: **Confirmed 37** (aktif, ber-status `[x] Confirmed`) + **N/A 1** = `BR-PAY-002` ber-status `Superseded by BR-CHECK-001` (duplikat rate limit; id dipertahankan agar tidak dipakai ulang). Semuanya dipetakan di § Business Rules Coverage.
- **NFR**: 14 id di `nfr.md`; **Confirmed 11** = punya pemilik baris FR; **N/A 3** = system-wide tanpa FR owner (NFR-PERF-002, NFR-SCP-001, NFR-SCP-003).
- **PF 11** = 11 blok `## PF-` di `docs/analyst/process-flows.md` (`grep -cE "^## PF-" docs/analyst/process-flows.md` → 11). File itu tidak memuat field status (0 hit `Confirmed`/`Dropped`/`Pending`), jadi **Confirmed 11** = semua blok PF punya `### Flow` + `### Step Descriptions` (masing-masing `grep -c` → 11) dan tidak ada penanda Dropped/Pending; **Dropped 0 / Pending 0** karena memang tidak ada di file. Pemetaan PF → FR/BR ada di § Process Flow Traceability.
- **ADR 19** = 19 file ADR (direktori ADR di repo aplikasi, tidak disertakan di repo ini); **Dropped 3** = ADR-0009, ADR-0010, ADR-0012 (Status `Cancelled` di file ADR).

---

## Gaps and Risks

| Gap | Impact | Mitigation |
|-----|--------|------------|
| Backup distributor belum ada | Jika Digiflazz down, tidak ada failover | Multi-vendor strategy (ADR-0006) bisa diaktifkan |
| Monitoring/alerting belum didefinikan | Tidak ada notifikasi jika sistem down | Perlu setup UptimeRobot / Sentry |
| Backup database belum didefinisikan | Data loss risk | Perlu cron backup + offsite storage |
| Email templates tanpa shared layout | Konsistensi visual kurang | Bisa ditambahkan email layout component |
| Tidak ada spec OpenAPI/Swagger — `docs/analyst/api-contract.md` §Contract Source: grep repo untuk `openapi` / `swagger` = 0 hit, tanpa tooling di `composer.json`; kontrak hanya prosa `PRD.md` §6 yang sudah drift (`PRD.md:209-211` → endpoint tidak ada di `routes/api.php:8-13`) | Kontrak reseller/H2H tidak machine-readable → drift terus & integrator menebak | Buat `openapi.yaml` (OpenAPI 3.1) dari snapshot api-contract.md + CI check route ↔ spec (rekomendasi R1 di dokumen tersebut) |
| Rate-limit config vs dokumentasi beda — `routes/web.php:39` menulis "per user 10/jam, per product+target 3/10m", default `config/fofa.php` = `checkout_user` 20/60 menit dan `checkout_target` 6/10 menit | Baseline kuota checkout salah baca saat review/test (nfr.md GAP-08) | Ratifikasi satu angka: ubah komentar ke default config, atau ubah default env ke angka spec |
| Deviasi ASM-001 / ASM-002 / ASM-004 (`docs/analyst/assumptions-constraints.md` § Deviasi) — endpoint Digiflazz default `/v1` + `DISTRIBUTOR_FAKE=true` (klaim "v2" tak terbukti); hanya 1 endpoint webhook generik untuk 2 gateway di SRS §5; default `MAIL_MAILER=log` bukan smtp | Asumsi integrasi tak terverifikasi sampai konfigurasi produksi | Putuskan saat setup produksi: `DISTRIBUTOR_FAKE=false`, rancang multi-gateway atau revisi SRS §5, set `MAIL_MAILER=smtp` |
| Id BR bentrok antar dokumen — **RESOLVED**: `BR-CHECK-001` kini kanonik = Checkout Rate Limiting (cocok dengan tabel kategori `CHECK / Validation / Rate limiting` dan kutipan BAB-1 UC-005 baris 334); aturan cron sync pindah ke id baru `BR-FUL-004` (kategori Workflow, sumber `SyncWaitingFulfillmentJob`, mengacu PF-009 di process-flows.md); `BR-PAY-002` ditandai `Superseded by BR-CHECK-001` dan tidak dipakai ulang | Penelusuran BR dari chapter kini konsisten dengan `business-rules.md`, `erd.md`, dan matrix ini | Selesai — konflik ditutup; tidak ada tindak lanjut selain menjaga konsistensi bila menambah id BR baru |

---

## ADR Traceability

| ADR | Decision | Status | Implementation |
|-----|----------|--------|----------------|
| ADR-0001 | Single Transaction entity | Confirmed | `app/Models/Transaction.php` |
| ADR-0002 | Payments 1:N | Confirmed | `app/Models/Payment.php` |
| ADR-0003 | Transaction status flow | Confirmed | `app/Enums/TransactionStatus.php` |
| ADR-0004 | Dynamic categories | Confirmed | `app/Models/Category.php` |
| ADR-0005 | Wallet system | Confirmed | `app/Models/Wallet.php` |
| ADR-0006 | Multi-vendor failover | Confirmed | `app/Services/DistributorService.php` |
| ADR-0007 | Owner separate role | Confirmed | `app/Enums/UserRole.php` |
| ADR-0008 | Voucher/promo system | Confirmed | `app/Services/VoucherService.php` |
| ADR-0009 | UTM click tracking | **CANCELLED** | Not needed for community-scale |
| ADR-0010 | White-label store | **CANCELLED** | Not needed for community-scale |
| ADR-0011 | Admin audit log | Confirmed | `app/Services/AuditLogService.php` |
| ADR-0012 | Live chat Reverb | **CANCELLED** | Not needed for community-scale |
| ADR-0013 | Product deletion rules | Confirmed | `app/Services/ProductDeletionService.php` |
| ADR-0014 | Redis queue/cache/session | Confirmed | Config files |
| ADR-0015 | Financial access control | Confirmed | `Transaction::scopeForAdmin()` |
| ADR-0016 | Minecraft edition | Confirmed | `app/Enums/MinecraftEdition.php` |
| ADR-0017 | Product image upload | Confirmed | Livewire product form |
| ADR-0018 | Tailwind CSS v4 | Confirmed | `resources/css/app.css` |
| ADR-0019 | Role hierarchy (4 role: customer/reseller/admin/owner) | Confirmed | ADR-0019 (role-hierarchy) + docs/roles.md |
