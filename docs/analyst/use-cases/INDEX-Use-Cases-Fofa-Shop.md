# Use Cases — Fofa Shop

23 use cases, dikelompokkan ke dalam 5 bab berdasarkan boundary dan fokus.

---

## Structure

```
use-cases/
├── INDEX-Use-Cases-Fofa-Shop.md                                          ← Master index (this file)
└── chapters/
    ├── BAB-1-Storefront-Customer-Journey/BAB-1-Storefront-Customer-Journey.md  ← 7 UC
    ├── BAB-2-Payment-GW-Financial/BAB-2-Payment-GW-Financial.md                ← 4 UC
    ├── BAB-3-Admin-Panel-Fulfillment/BAB-3-Admin-Panel-Fulfillment.md          ← 5 UC
    ├── BAB-4-Reseller-API/BAB-4-Reseller-API.md                                ← 4 UC
    └── BAB-5-System-Administration/BAB-5-System-Administration.md              ← 3 UC
```

---

## BAB 1: Storefront & Customer Journey

> Public Access, Unauthenticated / Light Auth

| UC | Nama | Actors | Process Flow |
|----|------|--------|-------------|
| UC-001 | Browse Kategori & Produk | Guest / Auth User | PF-001 |
| UC-002 | Detail Produk + Harga Per Role | Guest / Auth User | PF-001 |
| UC-003 | Search Produk | Guest / Auth User | — |
| UC-004 | Verifikasi ID / Nickname Game | Guest / Auth User | — |
| UC-005 | Checkout & Penerapan Voucher | Guest / Auth User | PF-001, PF-006 |
| UC-006 | View Invoice Status | Guest / Auth User | PF-001 |
| UC-007 | Riwayat Transaksi & Form Refund | Customer | — |

---

## BAB 2: Payment Gateway & Financial Operations

> High Security, Atomic Transaction, Idempotency Locked

| UC | Nama | Actors | Process Flow |
|----|------|--------|-------------|
| UC-008 | Payment Gateway Callback | Payment Gateway | PF-002 |
| UC-009 | Cek Saldo & Mutasi Wallet | Reseller | PF-005 |
| UC-010 | Kelola Deposit Wallet Reseller | Admin | — |
| UC-011 | Laporan Keuangan, Omset & Export | Owner | — |

---

## BAB 3: Admin Panel & Fulfillment

> Authenticated (admin/owner), Role-based access

| UC | Nama | Actors | Process Flow |
|----|------|--------|-------------|
| UC-012 | Dashboard Admin & Monitoring | Admin / Owner | — |
| UC-013 | CRUD Produk, Kategori & Subkategori | Admin | — |
| UC-014 | Retry Transaksi Gagal | Admin | PF-003 |
| UC-015 | Manual Fulfill Minecraft | Admin | PF-004 |
| UC-016 | Kelola Refund | Admin | PF-007 |

---

## BAB 4: Reseller API (H2H Integration)

> Authenticated (Sanctum token), role: reseller only

| UC | Nama | Actors | Process Flow |
|----|------|--------|-------------|
| UC-017 | List Produk via API | Reseller | — |
| UC-018 | Top-up via API | Reseller | PF-009 |
| UC-019 | Cek Status via API | Reseller | — |
| UC-020 | Cek Saldo via API | Reseller | — |

---

## BAB 5: System Administration & Configuration

> Owner-only untuk admin management

| UC | Nama | Actors | Process Flow |
|----|------|--------|-------------|
| UC-021 | Kelola Akun Admin | Owner | — |
| UC-022 | Broadcast Pengumuman | Admin | PF-010 |
| UC-023 | Toggle Payment Method | Admin | — |

---

## Coverage

| Chapter | Boundary | UC Count |
|---------|----------|----------|
| BAB 1 | Public / Light Auth | 7 |
| BAB 2 | High Security / Atomic | 4 |
| BAB 3 | Admin / Owner Auth | 5 |
| BAB 4 | Sanctum Token (Reseller) | 4 |
| BAB 5 | Owner-only | 3 |
| **Total** | | **23** |

---

## Process Flow Coverage

Judul diambil dari `docs/analyst/process-flows.md` (`grep -nE "^## PF-"`, 11 id, 0 celah);
kolom chapter dari baris `**Process Flow Related:**` di header tiap file chapter.

| PF     | Title                                   | Chapter(s) that cite it |
|--------|-----------------------------------------|-------------------------|
| PF-001 | Customer Checkout                       | BAB 1                   |
| PF-002 | Payment Callback                        | BAB 2                   |
| PF-003 | Transaction Fulfillment                 | BAB 3                   |
| PF-004 | Minecraft Manual Fulfillment            | BAB 3                   |
| PF-005 | Reseller Auto-Refund                    | BAB 2                   |
| PF-006 | Voucher Apply                           | BAB 1                   |
| PF-007 | Reseller API Top-up                     | BAB 4                   |
| PF-008 | Expire Pending Transactions (Scheduled) | BAB 2                   |
| PF-009 | Sync Waiting Fulfillment (Scheduled)    | BAB 4                   |
| PF-010 | Announcement Broadcast                  | BAB 5                   |
| PF-011 | Product Deletion                        | BAB 3                   |

> Catatan: label `**Process Flow Related:**` di header chapter pernah salah
> (BAB 3 menulis `PF-007 (Reseller Auto-Refund by Admin)` padahal PF-007 = Reseller
> API Top-up; BAB 4 menulis `PF-009 (Reseller API Top-up)` dan `PF-011 (Scheduled Sync)`
> padahal PF-009 = Sync dan PF-011 = Product Deletion; BAB 5 menulis `PF-003 (Admin Actions)`).
> Label telah dikoreksi agar sama persis dengan judul di `process-flows.md`, dan pemetaan
> chapter diturunkan dari kepemilikan UC di chapter tersebut (PF-011 → UC-013 di BAB 3,
> PF-007 → UC-018 di BAB 4, PF-010 → UC-022 di BAB 5).
> Tabel ini memakai judul `process-flows.md` sebagai acuan tunggal.
