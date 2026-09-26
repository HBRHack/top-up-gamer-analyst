# Extraction Report — Fofa Shop

**Analysis Date:** 2026-09-03
**Depth:** Deep (all code artifacts extracted)
**Framework Detected:** Laravel 13 + Livewire 3 + Volt + Tailwind CSS v4
**Project Type:** Fullstack (Backend + Frontend)
**Security Assessment:** SKIPPED (per user request)
**Bug Analysis:** SKIPPED (per user request)

---

## Summary

| Metric | Count |
|--------|-------|
| Entities (Models) | 18 |
| Database Tables | 26 |
| API Endpoints | 4 (Reseller) |
| Web Routes | 56 |
| Auth Routes | 7 |
| Scheduled Tasks | 3 |
| Roles | 4 (customer, reseller, admin, owner) |
| Services | 17 |
| Jobs | 5 |
| Mail Classes | 10 |
| Events | 1 |
| Listeners | 1 |
| Observers | 6 + 1 trait |
| Enums | 10 |
| Contracts | 2 |
| DTOs | 2 |
| Middleware | 3 |
| Blade Views | 80 |
| Livewire Components | 23 |
| Business Rules | 38 |
| External Integrations | 4 |

---

## Confidence Levels

### High Confidence (Directly from code)

- Entity definitions and relationships (from 18 models + 43 migrations)
- API route definitions and middleware (from routes/*.php)
- Business logic in services (from 17 service classes)
- State machine transitions (from TransactionStatus enum + FulfillTransactionJob)
- Auth/role enforcement (from RoleMiddleware + route constraints)
- Voucher validation rules (from VoucherService)
- Product deletion rules (from ProductDeletionService)
- Wallet operations (from WalletService + WalletRefundService)
- Game ID verification chain (from GameIdVerifierChain)
- Payment signature verification (from PaymentSignatureVerifier)
- Rate limiting rules (from CheckoutRateLimit middleware)
- Scheduled tasks (from routes/console.php)

### Medium Confidence (Inferred from code patterns)

- Payment gateway specifics (configurable, code supports multiple)
- Digiflazz API endpoint structure (inferred from payload building)
- Email template content (view files referenced but not read in detail)
- Chart.js integration (referenced in Livewire components)
- Font loading strategy (Chakra Petch + Plus Jakarta Sans + VT323)

### Low Confidence (Needs verification)

- Actual Digiflazz API response format (code handles multiple status keywords)
- Payment gateway IP whitelist configuration (configurable, not read)
- Redis connection details (configurable via .env)
- SMTP provider configuration (configurable via .env)

---

## Output Files Generated

| File | Purpose | Lines | Format |
|------|---------|-------|--------|
| `SRS.md` | Software Requirements Specification (FR-{NNN} IDs) | 311 | Markdown + ASCII tables |
| `data-dictionary.md` | Data dictionary (26 tables, Validation + BR per entity) | 420 | Markdown + ASCII tables |
| `erd.md` | Entity Relationship Diagram (ASCII box art) | 615 | ASCII Unicode box art |
| `business-rules.md` | 38 business rules (BR-{Category}-{NNN}, When/Then/Else) | 977 | Markdown + structured blocks |
| `process-flows.md` | 11 process flows (ASCII flow diagrams) | 730 | ASCII Unicode flow diagrams |
| `requirements-matrix.md` | Traceability matrix (priority groups + coverage) | 268 | Markdown + ASCII tables |
| `use-cases/chapters/BAB-1..BAB-5/*.md` | 5 chapter files, 23 UCs total (7/4/5/4/3) | 1354 total | Markdown per chapter format |
| `use-cases/INDEX-Use-Cases-Fofa-Shop.md` | Master index (structure, per-UC tables, PF coverage) | 124 | Markdown index |
| `nfr.md` | Non-functional requirements (NFR-{Category}-{NNN} IDs) | 372 | Markdown |
| `assumptions-constraints.md` | Constraints / assumptions / open questions (CON-NNN, ASM-NNN, OQ-NNN), split out of SRS §6/§7/§8 | 82 | Markdown tables |
| `api-contract.md` | HTTP interface contract snapshot (verified against code) | 559 | Markdown tables |
| `NUMBERING-LOG.md` | Authoritative id inventory (FR/UC/BR/NFR/PF/ADR/GAP/CON/ASM/OQ) | 571 | Markdown tables |
| `extraction-report.md` | This file | 145 | — |

**Total documentation:** ~6,528 lines of SA documentation

---

## Items Needing Confirmation — RESOLVED

1. **Payment Gateway Details** — **RESOLVED:** Dual-vendor: Tripay + Duitku. Both use HMAC-SHA256 signature verification, configurable via `fofa.payment.secret`.

2. **Digiflazz API Version** — **RESOLVED:** API v2.

3. **Email Templates** — **RESOLVED:** 9 email templates analyzed. See "Email Template Analysis" section below.

4. **Frontend Animations** — CONTEXT.md references complex Minecraft animations (mascot, particles, scroll fling). Not part of SA extraction but documented in DESIGN-SYSTEM.md.

5. **Dropped Features** — **RESOLVED:** Click tracking, reseller store, live chat, fraud detection are **CANCELLED** (not needed for community-scale shop). Marked as such in requirements-matrix.md.

---

## Email Template Analysis

| Template | Subject | Variables | Styling | Links |
|----------|---------|-----------|---------|-------|
| `transaction_success` | Transaksi berhasil - {ref_id} | ref_id, product, target, voucher, amount | None (plain HTML) | None |
| `transaction_paid` | Pembayaran diterima - {ref_id} | ref_id, product, voucher, amount | None | None |
| `transaction_failed` | Transaksi gagal - {ref_id} | Full transaction | None | Invoice link |
| `transaction_expired` | Transaksi kedaluwarsa - {ref_id} | Full transaction | None | Storefront link |
| `transaction_reconciliation` | Pembayaran diterima, transaksi diproses ulang - {ref_id} | Product, voucher, amount | None | None |
| `reseller_refunded` | Refund otomatis ke wallet - {ref_id} | User name, ref_id, product, target, amount | None | Hardcoded URL (not `route()`) |
| `refund_completed` | Refund diproses — {ref_id} | ref_id, product, amount, bank details | None | Invoice link |
| `minecraft_account_delivered` | Akun Minecraft Anda telah dikirim - {ref_id} | ref_id, product, edition, credentials | Inline CSS (credentials box) | None |
| `announcement` | [Fofa Shop] {title} | title, type, content | Inline CSS (type badge) | None |

### Email Template Findings

- **No shared layout** — 9 standalone raw HTML fragments, no `@extends`
- **No Tailwind CSS** — 7 of 9 templates are pure unstyled HTML
- **Inconsistent links** — `reseller_refunded` uses hardcoded URL instead of `route()`
- **Duplicated voucher logic** — 6 templates share identical voucher loading `@php` block
- **No email-safe CSS** — 2 templates with inline CSS lack responsive/email-client patterns
- **All Indonesian** — Consistent language throughout

---

## Recommendations for Portfolio

1. **ASCII diagrams** — The ERD and process flows use Unicode box art for maximum compatibility (no rendering tool needed)

2. **Highlight key decisions** — ADR-0013 (3-tier product deletion) and ADR-0015 (financial access control) are strong portfolio pieces

3. **Show the chain pattern** — `GameIdVerifierChain` (chain of responsibility) demonstrates design pattern knowledge

4. **Emphasize data integrity** — `lockForUpdate()`, atomic wallet operations, idempotent payment callbacks show production-quality thinking

5. **Document the "why"** — The PRD already has rationale for tech decisions; pair with ADRs for complete story
