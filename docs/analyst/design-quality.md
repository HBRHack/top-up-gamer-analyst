# Design Quality Scorecard — Fofa Shop

> Phase 4 dari `analyst-design-reviewer`. **Tanpa Mermaid** — diagram memakai ASCII/Unicode box art.

---

## 1. Architecture Quality

| Metric | Score (1–5) | Evidence |
|--------|:-----------:|----------|
| Separation of Concerns | **4** | Tiga layer jelas: routes/middleware → controllers + Livewire → services (`GameIdVerifierChain`, `PdfExportService`, `MinecraftCheckoutResolver`, `PaymentSignatureVerifier`) → models. Buktikan: tidak ditemukan akses DB langsung dari view |
| Single Responsibility | **4** | Satu service satu tugas — `RefIdGenerator` hanya menghasilkan `ref_id`; `SignatureVerifier` hanya memverifikasi. Kontra: `Admin\TransactionController` menangani list, retry, refund, **dan** export PDF sekaligus |
| Loose Coupling | **3** | Integrasi eksternal terisolasi di belakang interface (`app/Contracts/GameIdVerifier.php` + chain). Kontra: **failover antar-vendor masih manual** (`ADR-0006`) → modul fulfillment tetap rapat ke satu vendor |
| High Cohesion | **4** | Entity sesuai glossary 1-1: `transactions` memegang seluruh siklus, `payments` terpisah justru untuk audit trail multiple attempt |
| DRY Principle | **4** | Middleware `role:` dipakai ulang di web & api; `BR-CHECK-001` rate limit terpusat. Kontra: validasi checkout punya jalur khusus guest vs user yang sebagian duplikatif |
| SOLID Compliance | **3** | Dependency inversion jalan pada verifier chain. Kontra: controller gendut (lihat Single Responsibility) — SRP dan OCP paling lemah |

**Sub-total arsitektur: 21/30**

---

## 2. Documentation Quality

| Metric | Score (1–5) | Evidence |
|--------|:-----------:|----------|
| Component documentation | **5** | 23 UC dengan Preconditions/Postconditions/Main Flow/AF/BR/Data Entities; 11 PF dengan Trigger/Actors/Steps |
| API documentation | **2** | `api-contract.md` (559 baris, 6 endpoint) ada, **tapi tanpa OpenAPI/Swagger** → temuan High. Kontrak implisit pasti drift |
| Data dictionary coverage | **4** | 20 tabel, kolom Validation di 227 baris. Kontra: 3 baris sempat punya `Required=no` vs `Validation=required` (diperbaiki); notasi masih campur prosa & `required_if:` |
| Decision records (ADR) | **5** | 19 ADR, **19/19 lengkap 8 field wajib**; 16 Accepted + 3 Cancelled (0009/0010/0012) — status tak pernah ambigu |
| Traceability matrix | **5** | 1 hub FR↔UC↔BR↔NFR↔ADR; **0 untraced** pada semua family |
| Numbering governance | **4** | `NUMBERING-LOG.md` 605 baris, 10 keluarga, 0 id menggantung/yatim. Kontra: masih memuat jargon internal yang tak lazim dibaca publik |
| Glossary discipline | **4** | `CONTEXT.md` punya `_Avoid_` per canonical term; 3 violation ditemukan lalu diperbaiki |
| Assumptions & constraints | **5** | `assumptions-constraints.md`: 5 CON + 4 ASM + 3 OQ, tiap baris berisi deviasi nyata vs dokumen |

**Sub-total dokumentasi: 34/40**

---

## 3. Testing & Verification Quality

| Metric | Score (1–5) | Evidence |
|--------|:-----------:|----------|
| Test coverage terdokumentasi | **4** | 6 file `tests/Feature/` terdeteksi (OwnerAdminManagement, AdminPdfExport, FinancialDataAccess, ImageUpload, MinecraftEditionScenario, TransactionChart) |
| NFR dapat diuji | **5** | 14 NFR, QA-scenario **6 bagian** (sumber stimulus, stimulus, environment, artifact, respons, ukuran respons) — tanpa response measure tidak lolos |
| Business rules teruji | **4** | 37 BR aktif tiap punya `Rule/When/Then/Else` → dapat diturunkan langsung jadi test case |
| Kontrak API teruji | **2** | Tidak ada contract test; endpoint eksternal tanpa spec → regresi tak terdeteksi |

**Sub-total testing: 15/20**

---

## 4. Overall Score

```
┌──────────────────────────────────────────────────────────────────┐
│                     DESIGN QUALITY SCORECARD                    │
├──────────────────────────────┬───────────────────────────────────┤
│  Architecture (÷30)          │  21 / 30   ████████████░░░░  70%  │
│  Documentation  (÷40)        │  34 / 40   ███████████████░  85%  │
│  Testing        (÷20)        │  15 / 20   ███████████████░  75%  │
├──────────────────────────────┼───────────────────────────────────┤
│  TOTAL            (÷90)      │  70 / 90   ███████████████░  78%  │
│  NORMALISASI      (÷100)     │  78 / 100                     │
├──────────────────────────────┴───────────────────────────────────┤
│  KATEGORI: 3-4 → "Good — ada area yang perlu diperbaiki"         │
└──────────────────────────────────────────────────────────────────┘
```

**Overall Score: 78/100 — "Good"**

- 4–5: Excellent — siap implementasi
- **3–4: Good — ada area perlu diperbaiki sebelum dev lanjut** ← posisi saat ini
- 2–3: Fair — perlu redesign di area kritis
- 1–2: Poor — arsitektur perlu dibuat ulang

> **Pembeda dua skor:** `design-quality.md` menilai **kualitas desain & dokumentasi** (78),
> sedangkan **Quality Gate** di `architecture-review.md` menilai **kelayakan lanjut fase** (85).
> Keduanya punya rubrik berbeda — Gate lebih longgar karena mengukur kesiapan, scorecard lebih
> ketat karena menghukum ketiadaan bukti (mis. skor 2 untuk API & contract test karena
> "tidak ada" memang tidak bisa diberi angka tinggi).

---

## 5. Prioritas Perbaikan

| # | Area | Skor sekarang | Tindakan | Efek ke skor |
|---|------|:---:|---|---|
| 1 | API documentation | 2 | Generate OpenAPI dari kode; jadikan CI check | +2 → Documentation 36 |
| 2 | Kontrak teruji | 2 | Contract test untuk webhook + `/api/v1/*` | +2 → Testing 17 |
| 3 | Loose Coupling / SOLID | 3 | Ekstraksi `TransactionExportService` dari controller; runbook failover vendor | +1/+1 → Architecture 23 |
| 4 | Numbering governance | 4 | Redaksikan jargon internal `NUMBERING-LOG.md` | +1 → Documentation 35 |

Jika keempatnya dikerjakan: **78 → 85/100**.

---

## 6. Catatan Ketidaksempurnaan

Skor ini **bukan angka hiburan** — setiap potongan punya sebab yang bisa ditelusuri:

1. **API documentation = 2** — `api-contract.md` memang ada, tapi ketiadaan OpenAPI membuatnya
   tidak bisa dijadikan kontrak mesin. Ini temuan **High** yang tidak bisa dipoles dengan
   menulis lebih banyak Markdown.
2. **Contract test = 2** — tidak ada file test untuk webhook maupun endpoint reseller.
   Angka 2 mencerminkan "ada dokumentasi, tidak ada verifikasi otomatis".
3. **Loose Coupling / SOLID = 3** — terperinci di bukti: controller gendut + failover manual.
4. **Baseline kapasitas kosong** (`cross-module-review.md` §3) — tidak ada persentil p95/p99
   yang tercatat, jadi klaim performa apa pun tak bisa dibuktikan.
5. **Tidak ada Critical Flaw Veto** pada Gate, tapi scorecard tetap menahan di 78 karena
   kategori "Excellent" mensyaratkan ketiadaan skor 2 — dan ada dua.

**Kesimpulan:** desain dan dokumentasinya **Good**, belum Excellent. Yang menghalangi bukan
struktur, melainkan **tiga hal spesifik**: OpenAPI, contract test, dan baseline performa.
