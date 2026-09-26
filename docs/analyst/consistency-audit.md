# Consistency Audit — Fofa Shop

> Audit konsistensi lintas dokumen + skoring. Rekan dari `architecture-review.md` (Section 0).
> **Tanpa Mermaid** — diagram memakai ASCII/Unicode box art.

---

## 1. Inventaris Dokumen (yang diaudit)

| Dokumen | Path | Ukuran | Catatan |
|---------|------|--------|---------|
| PRD | `PRD.md` | §7.1–7.12 direnumber | 6 subbab dianotasi DROPPED |
| SRS | `docs/analyst/SRS.md` | 38 FR | 36 Confirmed / 2 Dropped |
| Requirements Matrix | `docs/analyst/requirements-matrix.md` | — | 1 hub traceability FR↔UC↔BR↔NFR↔ADR |
| Business Rules | `docs/analyst/business-rules.md` | 979 baris | 38 blok = 37 aktif + 1 superseded |
| Data Dictionary | `docs/analyst/data-dictionary.md` | 420 baris | 20 tabel, kolom Validation |
| ERD | `docs/analyst/erd.md` | 617 baris | 19 entity + Value Lists |
| Process Flows | `docs/analyst/process-flows.md` | 730 baris | 11 PF |
| Use Cases | `docs/analyst/use-cases/` | 5 chapter + INDEX | 23 UC |
| NFR | `docs/analyst/nfr.md` | 372 baris | 14 NFR, QA-scenario 6 bagian |
| Assumptions & Constraints | `docs/analyst/assumptions-constraints.md` | 82 baris | 5 CON + 4 ASM + 3 OQ |
| API Contract | `docs/analyst/api-contract.md` | 559 baris | 6 endpoint, 20 findings |
| Numbering Log | `docs/analyst/NUMBERING-LOG.md` | 605 baris | 10 keluarga id |
| ADR | 19 file ADR | 19 file | 16 Accepted / 3 Cancelled |
| Domain Glossary | `CONTEXT.md` | — | berisi `_Avoid_` untuk Check 9 |

**Struktur:** flat (tidak ada `00-Global/` / `BAB-*`), jadi skill memakai jalur fallback.

---

## 2. Traceability Matrix (Ringkas)

### 2.1 Coverage Summary

| Family | Total | Confirmed | N/A or Dropped | Untraced |
|--------|-------|-----------|----------------|----------|
| Functional Requirements | 38 | 36 | 2 (FR-025, FR-027) | 0 |
| Business Rules | 38 | 37 | 1 (BR-PAY-002 Superseded) | 0 |
| Use Cases | 23 | 23 | 0 | 0 |
| Process Flows | 11 | 11 | 0 | 0 |
| Non-Functional | 14 | 11 | 3 (NFR-PERF-002, NFR-SCP-001, NFR-SCP-003) | 0 |
| ADR | 19 | 16 | 3 (Cancelled: 0009, 0010, 0012) | 0 |

### 2.2 Bukti Grep (bisa dijalankan ulang)

```sh
grep -cE '^#### FR-'  docs/analyst/SRS.md                    # 38
grep -cE '^### BR-'   docs/analyst/business-rules.md         # 38
grep -c '| \[x\] Confirmed |' docs/analyst/business-rules.md # 37
grep -hcE '^## UC-'   docs/analyst/use-cases/chapters/*/*.md  # 23
grep -cE '^## PF-'    docs/analyst/process-flows.md           # 11
ls ADR/*.md | wc -l                                        # 19 (direktori ADR — di repo aplikasi)
grep -rhoE 'NFR-[A-Z]+-[0-9]+' docs/analyst/nfr.md | sort -u | wc -l  # 14
```

### 2.3 Rujukan Menggantung (dangling) — SEMUA KOSONG

| Pemeriksaan | Hasil |
|---|---|
| `BR-*` disitir chapter/matrix/erd tapi tak didefinisikan | **0** |
| `UC-*` disitir tapi tak ada di chapter | **0** |
| `NFR-*` disitir tapi tak didefinisikan | **0** |
| `FR-*` disitir matrix tapi tak ada di SRS | **0** |
| Tautan relatif antar file Markdown yang targetnya tak ada | **0** |
| `TODO` / `TBD` / `FIXME` | **0** |
| Duplikat nomor ADR | **0** |
| Tabel Markdown tidak sejajar | **0** (74 "hit" awal = ASCII art di `erd.md`/`process-flows.md`, bukan tabel) |

> **Catatan `ADR-0020`:** muncul di grep sebagai "id disitir tanpa file" — **bukan** dangling.
> Itu baris `NUMBERING-LOG.md:41` yang menyatakan `0020+` **dilarang dipakai ulang**.

---

## 3. Temuan yang Ditemukan Lalu Diperbaiki

Empat temuan Phase 0 ditemukan dari disk, diperbaiki, dan di sini bukti sebelum-sesudahnya:

| # | Sebelum | Sesudah | Lokasi |
|---|---------|---------|--------|
| 1 | `\| refund_bank \| … \| no \| null \| required, string, max:50 \|` — kolom `Required=no` tapi `Validation` menulis `required` | `required_if:refund_requested,true, string, max:50` | `data-dictionary.md:205-207` |
| 2 | `UC-011` memuat *Main Flow — Laporan* dan *Main Flow — Export Pajak* tanpa penanda bahwa FR-025/FR-027 sudah Dropped | Blok scope-cut ditambahkan di judul; kedua alur ditandai `~~(DROPPED — FR-025)~~` / `~~(DROPPED — FR-027)~~` | `use-cases/chapters/BAB-2-…:188,206,219` |
| 3 | `\| BR-AUTH-003 \| … \| FR-014, FR-024, FR-025, FR-026, FR-035 \|` — FR-025 bersanding tanpa penanda | `FR-025 *(Dropped)*` | `requirements-matrix.md:178` |
| 4 | `_Avoid_: sku` dipakai 2× ("SKU at vendor"); `_Avoid_: seller` dipakai 1× | "Kode produk pada vendor" (2×) dan "multi-vendor store" (1×) | `data-dictionary.md:114`, `erd.md:207`, `SRS.md:24` |

### Perbaikan dari gelombang sebelumnya (dicatat agar tidak hilang)

- `NUMBERING-LOG.md` OD-01 & OD-02 ternyata **FALSE FLAG** — snapshot basi; keduanya sudah beres sebelum dibaca.
- `OD-03/06/07/08/09/11/12/14/15` → **Selesai** (rincian di `NUMBERING-LOG.md` § Rekonsiliasi Status).
- `BR-GAME-001`/`BR-GAME-002` sempat tak punya baris `Data Entity:` → kini `transactions` dan tercatat di `erd.md` baris transactions.
- Label `Process Flow Related:` di header BAB-3/BAB-4/BAB-5 salah menempatkan PF-007/009/011 → disamakan persis dengan judul `process-flows.md`.

---

## 4. Scope Creep Findings

| Pemeriksaan | Hasil | Bukti |
|---|---|---|
| Fitur di dokumen tanpa dasar bisnis (Downstream→Upstream) | **0 orphaned item** | Fitur scope-cut v1.3 justru **dihapus**, bukan ditambah |
| Click tracking | Dibatalkan, ditutup bersih | `ADR-0009 (utm-click-tracking)` → **Cancelled** |
| White-label store path | Dibatalkan, ditutup bersih | `ADR-0010 (white-label-store-path)` → **Cancelled** |
| Live chat (Reverb) | Dibatalkan, ditutup bersih | `ADR-0012 (live-chat-reverb)` → **Cancelled** |
| Standalone reports / tax calculation | `Dropped` di SRS + matrix | `SRS.md:182` (FR-025), `SRS.md:192` (FR-027) |
| Requirement upstream tanpa design/task (Upstream→Downstream) | **0 MISSING COVERAGE** | Setiap FR Confirmed punya baris di matrix + entri UC |

**Kesimpulan:** arahnya justru sebaliknya — proyek ini **memangkas scope**, bukan menambah.
Tidak ditemukan over-engineering (contoh pola buruk: Kubernetes/Redis Cluster tanpa dasar NFR)
karena spec v1.3 menegaskan skala komunitas.

---

## 5. Scope-Cut Rekonsiliasi (spec v1.3 → dokumen)

| Fitur | Status di spec | Di PRD | Di SRS | Di matrix | Di ADR |
|---|---|---|---|---|---|
| Click tracking | dibuang | § anotasi DROPPED | Excluded | — | 0009 Cancelled |
| Reseller store / white-label | dibuang | § anotasi DROPPED | Excluded | — | 0010 Cancelled |
| Live chat | dibuang | § anotasi DROPPED | Excluded | — | 0012 Cancelled |
| Fraud detection | dibuang | § anotasi DROPPED | Excluded | — | — |
| Standalone reports | dibuang | § anotasi DROPPED | FR-025 Dropped | Dropped | — |
| Tax calculation / CSV-Excel export | dibuang | §7.8 parsial | FR-027 Dropped | Dropped | — |
| PDF export | **tetap** | hidup | FR-026 Confirmed | Confirmed | — |

**Konsistensi: 7/7 baris selaras.** Tidak ada fitur yang dihapus di satu dokumen tapi masih
dianggap hidup di dokumen lain — itulah inti Check 7.

---

## 6. Quality Gate Scoring

| Dimensi | Bobot | Skor | Catatan |
|---------|-------|------|---------|
| Completeness | 40 | **34** | Tanpa OpenAPI; SRS tanpa id PF; 3 kontradiksi DD (sudah diperbaiki) |
| Clarity | 30 | **25** | Jargon internal di `NUMBERING-LOG.md`; campur id/en; notasi Validation campur |
| Alignment | 30 | **26** | 0 dangling; rujukan `.scratch/` **sudah dinetralkan** untuk publikasi; `docs/analyst/` kini ter-track |
| **Total** | 100 | **85** | **≥ 80 → LOLOS** |

**Critical Flaw Veto: TIDAK AKTIF.**

---

## 7. Catatan Ketidaksempurnaan (jujur — ini yang TIDAK beres)

Daftar ini dipertahankan apa adanya. **Tidak ada yang disembunyikan.**

| ID | Ketidaksempurnaan | Kenapa tidak diperbaiki |
|----|-------------------|--------------------------|
| **OD-04** | `SRS.md` **0 id PF** | Desain: PF ditelusuri lewat matrix. Menambah 11 rujukan PF ke SRS berisiko drift dua arah |
| **OD-05** | `SRS.md` 0 id UC & 0 id BR; chapter 0 id FR | Desain: traceability sengaja satu hub (`requirements-matrix.md`). Menduplikasi = redundansi, justru dilawan Check 2 |
| **OD-10** | Urutan heading FR di SRS non-monoton (`FR-034` sebelum `FR-024`) | Kosmetik; himpunan id lengkap `001`–`038`. Merapikan urutan = menggeser ratusan rujukan baris |
| **OD-13** | `docs/analyst/` **untracked di git** | **Keputusan pemilik.** Repo menyimpan 165 perubahan belum di-commit; urusan Git diserahkan sepenuhnya ke pemilik |
| **REF-01** | Rujukan `.scratch/...` di docs | **Dinetralkan untuk publikasi (keputusan pemilik 2026-09-26).** Path ke tracker internal diganti frasa netral ("issue tracker internal (tidak dipublikasikan)") agar tidak jadi tautan mati maupun membocorkan struktur kerja privat |
| **JRG-01** | `NUMBERING-LOG.md` memuat jargon internal ("post Wave-4", "snapshot 09:51", "false flag") | **Sebagian dinetralkan saat publikasi 2026-09-26** — sisa jargon dipertahankan karena menyangkut jejak audit yang tidak boleh dihapus |
| **API-01** | Tanpa OpenAPI/Swagger | **High** — menambahkan spec = pekerjaan tersendiri, di luar scope audit dokumen |
| **API-02** | HMAC secret fallback `''` | **High** — temuan ke **kode**, bukan dokumen. Perbaikannya menyentuh `app/`, bukan `docs/` |
| **API-03** | Tanpa rate limit aplikasi di endpoint API | **High** — sama: temuan kode |
| **ATB-01** | Semua 19 ADR memuat `Deciders: … + opencode (AI agent — satu-satunya author git repo)` | **Keputusan pemilik — "pertahankan"** |
| **LAN-01** | Campur Indonesia/Inggris (202 marker `yang/tidak/dengan` vs 4 marker `shall/must`) | Bahasa pemilik; bukan defect |

---

## 8. Verifikasi Keamanan untuk Publikasi

Dipindai pada dokumen yang akan dipublikasikan:

| Pemeriksaan | Hasil |
|---|---|
| API key / token / password asli tertanam | **0** |
| `password: 'password'` | 2× — keduanya **contoh seeder**, kini diberi komentar `// KREDENSIAL CONTOH …` (`docs/owner-role.md:206`, ADR owner-role bagian seeder) |
| Placeholder `role:xxx` | **0** — diganti `role:admin,owner` (`business-rules.md:595`), sesuai `routes/web.php:68` |
| Domain keras | 2× `yourdomain.com` (domain pemilik, disamarkan), 2× `api.digiflazz.com` (vendor publik) |
| Kredensial/kata sandi contoh | Disamarkan (`<REDACTED>`) sebelum publikasi |
| Rujukan ke dokumen proyek lain (repo privat) | **Dinetralkan** — path `.scratch/`, `docs/adr/`, `.hallmark/` diganti frasa netral |
| `.scratch/` bocor ke Git | **Tidak** — folder itu tidak ikut repo ini, dan rujukannya sudah dinetralkan |
| Rahasia di `.env` | Tidak ikut — `.env` di-`.gitignore` (lokal + global) |
