# Assumptions & Constraints — Fofa Shop

Dokumen kerja (extract, bukan rewrite) dari `docs/analyst/SRS.md`:

- **§6 Constraints** → ID `CON-001`…
- **§7 Assumptions** → ID `ASM-001`…
- **§8 Open Questions** → ID `OQ-001`…

Isi kolom *Constraint / Assumption / Open Question* disalin **verbatim** dari SRS §6/§7/§8,
**versi sebelum ketiga seksi dijadikan pointer ke dokumen ini** — kini SRS hanya menyimpan
daftar id (`CON-001…`, `ASM-001…`, `OQ-001…`) dan menunjuk ke sini, jadi teks verbatim tidak
ada lagi di SRS (tidak diparafrase). Kolom `Sumber` menunjuk seksi asal; kolom `Impact` dan
`Validation` ditambahkan karena dibutuhkan review process.

**Cara verifikasi:** setiap baris dicek langsung ke file di repo (kode, config, migration,
dokumen deploy) pada 2026-09-26 — bukan dari ingatan. Status ditulis apa adanya:
`Terverifikasi`, `Sebagian` (ada deviasi dari isi dokumen), atau `Masih terbuka`.

---

## Constraints

| ID      | Constraint (verbatim SRS §6)                       | Sumber | Impact                                       | Validation (cek repo)                                                                                                                 |
|---------|----------------------------------------------------|--------|----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| CON-001 | VPS hosting berbayar (wajib, bukan shared hosting) | SRS §6 | Biaya operasional tetap; batas spek hardware | Terverifikasi: docs/deployment.md baris 7 (VPS 2 VCPU + 2 GB RAM + 2 GB SWAP minimum)                                                 |
| CON-002 | Domain custom + SSL                                | SRS §6 | Perlu domain berbayar + sertifikat           | Terverifikasi: docs/deployment.md baris 9 dan 126-130 (certbot --nginx)                                                               |
| CON-003 | Redis wajib (cache + queue + session)              | SRS §6 | Redis mati = cache/queue/session gagal       | Terverifikasi: config/cache.php, config/queue.php, config/session.php default redis; .env.example baris 30/40/42; docs/redis-setup.md |
| CON-004 | Root access untuk deployment                       | SRS §6 | Proses deploy butuh sudo                     | Terverifikasi: docs/deployment.md baris 20-23 (apt/systemctl), 130 (certbot), 159 (ufw)                                               |
| CON-005 | Minecraft produk butuh manual fulfillment          | SRS §6 | Tidak ada automasi penuh; beban admin        | Terverifikasi: app/Jobs/FulfillTransactionJob.php baris 55-60 skip distributor untuk kategori Minecraft; SRS FR-017                   |

---

## Assumptions

| ID      | Assumption (verbatim SRS §7)                        | Sumber | Impact jika salah                 | Validation (cek repo)                                                                                                                                                                                                                                                                               |
|---------|-----------------------------------------------------|--------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ASM-001 | Digiflazz API v2 available dan responsive           | SRS §7 | Cek ID dan fulfillment terhambat  | Sebagian: app/Services/DigiflazzGameIdVerifier.php, DistributorService.php, GameIdVerifierChain + FallbackGameIdVerifier ada. Catatan: .env.example DISTRIBUTOR_FAKE=true (stub tanpa HTTP) dan endpoint default https://api.digiflazz.com/v1/transaction; klaim "v2" belum terbukti dari config    |
| ASM-002 | Payment gateway webhook delivered within seconds    | SRS §7 | Invoice kadaluarsa / status telat | Sebagian: routes/web.php baris 46 POST /webhook/payment-callback (CSRF dikecualikan bootstrap/app.php), PaymentSignatureVerifier HMAC-SHA256 + hash_equals, IP whitelist opsional config/fofa.php. Tidak ada kode khusus Tripay/Duitku (dual gateway hanya di dokumen) dan tanpa target waktu kirim |
| ASM-003 | Single-server deployment (tidak horizontal scaling) | SRS §7 | Skala terbatas bila trafik naik   | Terverifikasi: docs/deployment.md satu VPS; deploy/supervisor-worker.conf satu worker (fofo-queue-worker_00); tidak ada artefak multi-node                                                                                                                                                          |
| ASM-004 | Email delivery via SMTP provider                    | SRS §7 | Notifikasi transaksi tidak sampai | Sebagian: config/mail.php dan .env.example memakai MAIL_MAILER=log sebagai default; SMTP didukung tapi harus di-set manual di produksi                                                                                                                                                              |

---

## Open Questions

| ID     | Open Question (verbatim SRS §8)      | Sumber | Impact                                | Status / Validation                                                                                                                                                                                                                        |
|--------|--------------------------------------|--------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| OQ-001 | Backup distributor selain Digiflazz? | SRS §8 | Digiflazz down = fulfillment berhenti | Masih terbuka: DISTRIBUTOR_PROVIDER default digiflazz (config/fofa.php); DistributorService + ADR-0006 siap untuk multi-vendor; requirements-matrix.md § Gaps and Risks → baris "Backup distributor belum ada"                             |
| OQ-002 | Monitoring/alerting strategy?        | SRS §8 | Sistem down tanpa notifikasi          | Masih terbuka: health endpoint /up ada (bootstrap/app.php baris 17); belum ada konfigurasi Sentry/UptimeRobot di repo; requirements-matrix.md § Gaps and Risks → baris "Monitoring/alerting belum didefinikan"                             |
| OQ-003 | Backup strategy untuk database?      | SRS §8 | Kehilangan data transaksi/keuangan    | Masih terbuka: tidak ada cron backup MySQL di docs/deployment.md; docs/redis-setup.md baris 195-198 hanya membahas Redis (tidak perlu backup rutin); requirements-matrix.md § Gaps and Risks → baris "Backup database belum didefinisikan" |

---

## Ringkasan Reality Check

| Status                 | Jumlah | ID                                                   |
|------------------------|--------|------------------------------------------------------|
| Terverifikasi          | 6      | CON-001, CON-002, CON-003, CON-004, CON-005, ASM-003 |
| Sebagian (ada deviasi) | 3      | ASM-001, ASM-002, ASM-004                            |
| Masih terbuka          | 3      | OQ-001, OQ-002, OQ-003                               |

**Total 12 entri: 5 Constraints + 4 Assumptions + 3 Open Questions.**

### Deviasi yang perlu diputuskan pemilik SRS

1. **ASM-001 vs config** — SRS menyebut "Digiflazz API v2", tetapi `DISTRIBUTOR_ENDPOINT`
   default mengarah ke `https://api.digiflazz.com/v1/transaction` dan `.env.example` menyetel
   `DISTRIBUTOR_FAKE=true` (stub, tidak melakukan HTTP). Integrasi
   `DigiflazzGameIdVerifier` + `FallbackGameIdVerifier` (chain) **ada**, tetapi asumsi
   ketersediaan/responsivitas hanya bisa divalidasi di produksi dengan `DISTRIBUTOR_FAKE=false`.
2. **ASM-002 vs kode** — SRS §5 memisahkan Tripay (inbound) dan Duitku (backup, inbound);
   di kode hanya ada **satu** endpoint generik `POST /webhook/payment-callback` dengan satu
   secret HMAC (`PAYMENT_GATEWAY_SECRET`) dan IP whitelist opsional. Nama gateway tidak muncul
   di file PHP mana pun (hanya di dokumen). Tidak ada target "within seconds" yang terukur.
3. **ASM-004 vs default** — `MAIL_MAILER` default = `log` (bukan `smtp`) di `config/mail.php`
   dan `.env.example`; asumsi SMTP hanya berlaku bila dikonfigurasi eksplisit di produksi.

Catatan tambahan terkait asumsi reseller API: `reseller_profiles.api_key` (unique + nullable)
dan `api_secret` (nullable) di migration `2026_08_26_000005` tidak pernah dibaca oleh jalur
autentikasi mana pun; autentikasi nyata memakai Sanctum personal access token yang
di-provision/revoke admin (`ResellerTokenController`). Detailnya tercatat sebagai GAP-09 di
`docs/analyst/nfr.md`.
