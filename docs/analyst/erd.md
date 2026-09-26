# Entity Relationship Diagram — Fofa Shop

Data model untuk sistem top-up game Fofa Shop dengan 18 entitas bisnis + 8 framework tables.

## Entity Relationship Diagram

```
+------------------+       +------------------+
|     users        |       |   categories     |
+------------------+       +------------------+
| PK | id          |       | PK | id          |
|    | name        |       |    | name        |
|    | email       |       |    | slug        |
|    | role        |       |    | icon_url    |
|    | is_active   |       |    | image_path  |
|    | email_notif |       |    | display_order|
|    | password    |       |    | is_active   |
+--------+---------+       |    | status      |
         |                 +--------+----------+
         |                          |
         |                 +--------+----------+
         |                 |                   |
+--------+---------+ +-----+--------+  +------+-------+
| reseller_profiles| |   wallets    |  |  products    |
+------------------+ +--------------+  +--------------+
| PK | id           | | PK | id     |  | PK | id      |
| FK | user_id  1:1 | | FK | user_id|  | FK | category_id|
|    | api_key      | |    | balance|  | FK | subcategory_id|
|    | api_secret   | +----+--------+  |    | name     |
|    | markup_pct   |       |         |    | description|
|    | callback_url |       |         |    | price     |
|    | status       |       |         |    | cost_price |
+--------+----------+       |         |    | is_active |
         |                  |         |    | deleted_at|
         |                  |         +----+----+-----+
         |                  |              |     |
         |                  |              |     |
+--------+---------+  +-----+-----------+  |  +--+------------------+
|    transactions   |  | wallet_txn      |  |  | product_vendor_map|
+------------------+  +-----------------+  |  +------------------+
| PK | id           |  | PK | id        |  |  | PK | id          |
|    | ref_id       |  | FK | wallet_id |  |  | FK | product_id  |
| FK | user_id      |  |    | type      |  |  |    | vendor      |
|    | guest_email  |  |    | amount    |  |  |    | vendor_code |
| FK | product_id   |  | FK | ref_id    |  |  |    | vendor_price|
|    | target_game_id| +----+-----------+  |  +----+-------------+
|    | target_server|                      |
|    | mc_edition   |  +------------------+  +------------------+
|    | mc_username  |  |    payments      |  | distributor_logs |
|    | mc_password  |  +------------------+  +------------------+
|    | delivery_note|  | PK | id          |  | PK | id          |
|    | amount       |  | FK | transaction_id| | FK | transaction_id|
|    | status       |  |    | method      |  |    | provider    |
|    | expired_at   |  |    | gateway_ref |  |    | attempt_num |
|    | completed_at |  |    | status      |  |    | req_payload |
|    | refund_*     |  |    | sig_verified|  |    | res_payload |
+--------+----------+  |    | paid_at     |  |    | http_status |
         |              +----+-------------+  |    | duration_ms |
         |                                   |    | status      |
+--------+---------+  +------------------+  |    | error_msg   |
|      vouchers    |  | voucher_usages   |  +----+-------------+
+------------------+  +------------------+
| PK | id           |  | PK | id        |
|    | code         |  | FK | voucher_id|
|    | type         |  | FK | user_id   |
|    | value        |  | FK | txn_id    |
|    | min_amount   |  |    | discount  |
|    | max_uses     |  +------------------+
|    | used_count   |
|    | starts_at    |  +------------------+  +------------------+
|    | expires_at   |  |  announcements   |  | notifications    |
|    | is_active    |  +------------------+  +------------------+
| FK | created_by   |  | PK | id          |  | PK | id          |
+------------------+  |    | title        |  | FK | user_id     |
                      |    | content      |  |    | title       |
+------------------+  |    | type         |  |    | message     |
|    audit_logs    |  |    | target_aud   |  |    | read_at     |
+------------------+  |    | is_active    |  +------------------+
| PK | id           |  |    | published_at |
| FK | user_id      |  | FK | created_by   |
|    | action       |  +------------------+  +------------------+
|    | auditable_typ|
|    | auditable_id |  +------------------+  +------------------+
|    | old_values   |  | admin_pay_method |  | prod_deletion_log|
|    | new_values   |  +------------------+  +------------------+
|    | ip_address   |  | PK | id          |  | PK | id          |
|    | user_agent   |  |    | method_code |  |    | product_id  |
+------------------+  |    | display_name |  |    | snapshot    |
                      |    | is_active    |  | FK | deleted_by  |
+------------------+  |    | sort_order   |  |    | reason      |
|  subcategories   |  +------------------+  |    | deleted_at  |
+------------------+                        +------------------+
| PK | id          |
| FK | category_id |  LEGEND:
|    | name        |    PK = Primary Key
|    | slug        |    FK = Foreign Key
|    | image_path  |    1:1 = One-to-One
|    | display_order|   1:N = One-to-Many
|    | is_active   |    M:N = Many-to-Many
+------------------+
```

## Entity Details

### users

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| name | string | Yes | — | — | User's full name |
| email | string | Yes | — | unique, email | Email address |
| role | enum | Yes | `customer` | in:customer,reseller,admin,owner | User role |
| is_active | boolean | Yes | `true` | — | Account active status |
| email_notification_enabled | boolean | Yes | `true` | — | Email opt-in/out |
| email_verified_at | timestamp | No | null | — | Email verification |
| password | string | Yes | — | hashed | Hashed password |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-AUTH-001, BR-AUTH-002, BR-AUTH-003, BR-AUTH-004

**Relationships:**
- users --1:1-- reseller_profiles
- users --1:1-- wallets
- users --1:N-- transactions
- users --1:N-- notifications

---

### categories

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| name | string | Yes | — | — | Category display name |
| slug | string | Yes | — | unique | URL slug |
| image_path | string | No | null | — | Uploaded image |
| icon_url | string | No | null | — | External icon URL |
| display_order | integer | Yes | `0` | — | Sort order |
| is_active | boolean | Yes | `true` | — | Visibility toggle |
| status | enum | Yes | `coming_soon` | in:active,coming_soon | Buy capability |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-PROD-001

**Relationships:**
- categories --1:N-- products
- categories --1:N-- subcategories

---

### products

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| category_id | bigint (FK) | Yes | — | exists:categories | Parent category |
| subcategory_id | bigint (FK) | No | null | exists:subcategories | Subcategory |
| name | string | Yes | — | — | Product name |
| description | text | No | null | — | Product description |
| image_path | string | No | null | — | Product image |
| price | decimal(12,2) | Yes | — | min:0 | Customer selling price |
| cost_price | decimal(12,2) | Yes | — | min:0 | Distributor buy price |
| is_active | boolean | Yes | `true` | — | Active status |
| deleted_at | timestamp | No | null | — | Soft delete |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-PROD-002, BR-DEL-001, BR-DEL-002

**Relationships:**
- products --N:1-- categories
- products --N:1-- subcategories
- products --1:N-- product_vendor_mappings
- products --1:N-- transactions

---

### subcategories

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| category_id | bigint (FK) | Yes | — | exists:categories, CASCADE | Parent category |
| name | string | Yes | — | — | Subcategory name |
| slug | string | Yes | — | unique within category | URL slug |
| image_path | string | No | null | — | Subcategory image |
| display_order | integer | Yes | `0` | — | Sort order |
| is_active | boolean | Yes | `true` | — | Visibility toggle |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Relationships:**
- subcategories --N:1-- categories
- subcategories --1:N-- products

---

### product_vendor_mappings

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| product_id | bigint (FK) | Yes | — | exists:products, CASCADE | Parent product |
| vendor | string | Yes | — | unique per product | Vendor name |
| vendor_product_code | string | Yes | — | — | Kode produk pada vendor |
| vendor_price | decimal(12,2) | Yes | — | min:0 | Price at vendor |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-FUL-003

**Relationships:**
- product_vendor_mappings --N:1-- products

---

### reseller_profiles

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| user_id | bigint (FK) | Yes | — | unique, exists:users, CASCADE | Reseller user |
| api_key | string | No | null | unique | API authentication key |
| api_secret | string | No | null | — | API secret |
| markup_percentage | decimal(5,2) | Yes | `0` | min:0, max:100 | Platform markup % |
| callback_url | string | No | null | url | Webhook URL |
| status | enum | Yes | `active` | in:active,inactive,suspended | Account status |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-PRICE-001

**Relationships:**
- reseller_profiles --1:1-- users

---

### wallets

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| user_id | bigint (FK) | Yes | — | unique, exists:users, CASCADE | Wallet owner |
| balance | decimal(12,2) | Yes | `0` | min:0 | Current balance |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-WAL-001, BR-WAL-002, BR-REF-002

**Relationships:**
- wallets --1:1-- users
- wallets --1:N-- wallet_transactions

---

### wallet_transactions

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| wallet_id | bigint (FK) | Yes | — | exists:wallets, CASCADE | Parent wallet |
| type | enum | Yes | — | in:deposit,deduction,refund | Transaction type |
| amount | decimal(12,2) | Yes | — | min:0.01 | Amount |
| reference_id | bigint (FK) | No | null | unique per type | Related transaction |
| notes | string(500) | No | null | — | Optional notes |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Constraints:** UNIQUE(reference_id, type) — prevents double-refund

**Business Rules:** BR-WAL-003, BR-WAL-004, BR-REF-001, BR-REF-002

**Relationships:**
- wallet_transactions --N:1-- wallets
- wallet_transactions --N:1-- transactions

---

### transactions

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| ref_id | string | Yes | — | unique | FOFA-YYYYMMDD-XXXXXXXX |
| user_id | bigint (FK) | No | null | exists:users | Nullable for guest |
| guest_email | string | No | null | email | Guest notification |
| product_id | bigint (FK) | Yes | — | exists:products, RESTRICT | Product purchased |
| target_game_id | string | No | null | — | Game account ID |
| target_server_id | string | No | null | — | Game server ID |
| minecraft_edition | enum | No | null | in:java,bedrock,manual | MC edition |
| delivered_minecraft_username | string | No | null | — | Delivered MC username |
| delivered_minecraft_password | string | No | null | encrypted | Delivered MC password |
| delivery_note | text | No | null | — | Admin note |
| amount | decimal(12,2) | Yes | — | min:0 | Transaction amount |
| status | enum | Yes | `pending` | in:7 states | Current status |
| expired_at | timestamp | No | null | — | Payment deadline |
| completed_at | timestamp | No | null | — | Completion time |
| is_reconciliation | boolean | Yes | `false` | — | Reconciliation flag |
| reconciled_at | timestamp | No | null | — | Reconciliation time |
| reconciliation_note | string | No | null | — | Reconciliation note |
| refund_requested | boolean | Yes | `false` | — | Refund request flag |
| refund_bank | string | No | null | — | Refund bank |
| refund_account_number | string | No | null | — | Refund account |
| refund_account_name | string | No | null | — | Refund name |
| refund_requested_at | timestamp | No | null | — | Request time |
| refund_completed_at | timestamp | No | null | — | Completion time |
| refund_completed_by | bigint (FK) | No | null | exists:users | Who completed |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-WF-001, BR-WF-002, BR-WF-003, BR-WF-004, BR-FUL-001, BR-FUL-002, BR-FUL-004, BR-REF-001, BR-REF-002, BR-REF-003, BR-MC-001, BR-CHECK-001, BR-GAME-001, BR-GAME-002

**Relationships:**
- transactions --N:1-- users
- transactions --N:1-- products
- transactions --1:N-- payments
- transactions --1:N-- distributor_logs
- transactions --1:N-- wallet_transactions
- transactions --1:1-- voucher_usages

---

### payments

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| transaction_id | bigint (FK) | Yes | — | exists:transactions, CASCADE | Parent transaction |
| method | string | Yes | — | — | Payment method |
| gateway_ref | string | No | null | — | Gateway reference |
| status | string | Yes | `pending` | — | Payment status |
| signature_verified | boolean | Yes | `false` | — | Signature check |
| paid_at | timestamp | No | null | — | Payment time |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-PAY-001

**Relationships:**
- payments --N:1-- transactions

---

### distributor_logs

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| transaction_id | bigint (FK) | Yes | — | exists:transactions, CASCADE | Parent transaction |
| provider | string | Yes | — | — | Provider name |
| attempt_number | integer | Yes | `1` | min:1 | Attempt sequence |
| request_payload | json | No | null | — | Outbound request |
| response_payload | json | No | null | — | Inbound response |
| http_status_code | integer | No | null | — | HTTP status |
| duration_ms | integer | No | null | — | Request duration |
| status | string | Yes | — | — | success/pending/failed |
| error_message | text | No | null | — | Error details |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Relationships:**
- distributor_logs --N:1-- transactions

---

### vouchers

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| code | string | Yes | — | unique | Voucher code |
| type | enum | Yes | `percentage` | in:percentage,fixed | Discount type |
| value | decimal(12,2) | Yes | — | min:0 | Discount value |
| min_order_amount | decimal(12,2) | No | null | min:0 | Minimum order |
| max_uses | unsignedInteger | No | null | min:1 | Max total uses |
| used_count | unsignedInteger | Yes | `0` | — | Current uses |
| starts_at | timestamp | No | null | — | Start period |
| expires_at | timestamp | No | null | — | End period |
| is_active | boolean | Yes | `true` | — | Active status |
| created_by | bigint (FK) | No | null | exists:users | Creator |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-PRICE-002, BR-VOUCH-001, BR-VOUCH-002, BR-VOUCH-003

**Relationships:**
- vouchers --1:N-- voucher_usages
- vouchers --N:1-- users

---

### voucher_usages

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| voucher_id | bigint (FK) | Yes | — | exists:vouchers, CASCADE | Parent voucher |
| user_id | bigint (FK) | No | null | exists:users | User who used |
| transaction_id | bigint (FK) | No | null | exists:transactions | Related transaction |
| discount_amount | decimal(12,2) | Yes | — | min:0.01 | Discount applied |
| created_at | timestamp | Yes | now() | — | Record creation |

**Business Rules:** BR-PRICE-002, BR-VOUCH-001, BR-VOUCH-003

**Relationships:**
- voucher_usages --N:1-- vouchers
- voucher_usages --N:1-- users
- voucher_usages --N:1-- transactions

---

### announcements

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| title | string | Yes | — | — | Announcement title |
| content | text | Yes | — | — | Announcement body |
| type | enum | Yes | `info` | in:info,maintenance,warning | Type |
| target_audience | enum | Yes | `all` | in:all,customer,reseller,admin | Target |
| is_active | boolean | Yes | `true` | — | Active status |
| published_at | timestamp | No | null | — | Publish time |
| created_by | bigint (FK) | No | null | exists:users | Creator |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-ANN-001, BR-ANN-002

**Relationships:**
- announcements --N:1-- users

---

### admin_payment_methods

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| method_code | string | Yes | — | unique | Method code |
| display_name | string | Yes | — | — | Display name |
| is_active | boolean | Yes | `true` | — | Active status |
| sort_order | integer | Yes | `0` | — | Display order |
| created_at | timestamp | Yes | now() | — | Record creation |
| updated_at | timestamp | Yes | now() | — | Last update |

**Business Rules:** BR-PAY-003

---

### notifications

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| user_id | bigint (FK) | Yes | — | exists:users, CASCADE | Target user |
| title | string | Yes | — | — | Notification title |
| message | text | Yes | — | — | Notification body |
| read_at | timestamp | No | null | — | Read timestamp |
| created_at | timestamp | Yes | now() | — | Record creation |

**Relationships:**
- notifications --N:1-- users

---

### audit_logs

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| user_id | bigint (FK) | No | null | exists:users | Actor |
| action | string | Yes | — | — | Action type |
| auditable_type | string | Yes | — | — | Polymorphic type |
| auditable_id | unsignedBigInteger | Yes | — | — | Polymorphic ID |
| old_values | json | No | null | — | Previous state |
| new_values | json | No | null | — | New state |
| ip_address | string(45) | No | null | — | Actor IP |
| user_agent | string | No | null | — | Actor UA |
| created_at | timestamp | Yes | now() | — | Record creation |

**Relationships:**
- audit_logs --N:1-- users
- audit_logs --MorphTo-- any model

---

### product_deletion_logs

| Attribute | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| id | bigint | Yes | auto | unique | Primary key |
| product_id | unsignedBigInteger | Yes | — | — | Deleted product ID |
| product_snapshot | json | Yes | — | — | Product state at deletion |
| deleted_by | bigint (FK) | No | null | exists:users | Who deleted |
| reason | text | No | null | — | Deletion reason |
| deleted_at | timestamp | Yes | — | — | Deletion time |
| created_at | timestamp | Yes | now() | — | Record creation |

**Business Rules:** BR-DEL-003

**Relationships:**
- product_deletion_logs --N:1-- users

---

## Relationship Summary

| From | Cardinality | To | FK Location | Description |
|------|-------------|-----|-------------|-------------|
| users | 1:N | transactions | transactions.user_id | User places transactions |
| users | 1:1 | reseller_profiles | reseller_profiles.user_id | Reseller data |
| users | 1:1 | wallets | wallets.user_id | Reseller wallet |
| users | 1:N | notifications | notifications.user_id | In-app notifications |
| users | 1:N | audit_logs | audit_logs.user_id | Audit trail |
| users | 1:N | announcements | announcements.created_by | Announcement creator |
| users | 1:N | voucher_usages | voucher_usages.user_id | Voucher usage |
| users | 1:N | product_deletion_logs | product_deletion_logs.deleted_by | Deletion audit |
| categories | 1:N | products | products.category_id | Product categorization |
| categories | 1:N | subcategories | subcategories.category_id | Sub-categorization |
| subcategories | 1:N | products | products.subcategory_id | Product grouping |
| products | 1:N | transactions | transactions.product_id | Product purchases |
| products | 1:N | product_vendor_mappings | product_vendor_mappings.product_id | Vendor SKUs |
| transactions | 1:N | payments | payments.transaction_id | Payment attempts |
| transactions | 1:N | distributor_logs | distributor_logs.transaction_id | Fulfillment logs |
| transactions | 1:1 | voucher_usages | voucher_usages.transaction_id | Applied voucher |
| transactions | 1:N | wallet_transactions | wallet_transactions.reference_id | Wallet mutations |
| wallets | 1:N | wallet_transactions | wallet_transactions.wallet_id | Balance history |
| vouchers | 1:N | voucher_usages | voucher_usages.voucher_id | Usage tracking |

---

## Value Lists

### UserRole

| Code | Label | Description |
|------|-------|-------------|
| customer | Customer | Beli produk via storefront |
| reseller | Reseller | H2H API, harga khusus |
| admin | Admin | Kelola sistem |
| owner | Owner | Pemilik, akses penuh |

### TransactionStatus

| Code | Label | Description |
|------|-------|-------------|
| pending | Pending | Menunggu pembayaran |
| paid | Paid | Pembayaran diterima |
| processing | Processing | Sedang diproses |
| waiting_fulfillment | Waiting Fulfillment | Menunggu fulfillment |
| success | Success | Transaksi berhasil |
| failed | Failed | Transaksi gagal |
| expired | Expired | Pembayaran kedaluwarsa |

### WalletTransactionType

| Code | Label | Description |
|------|-------|-------------|
| deposit | Deposit | Saldo masuk |
| deduction | Deduction | Saldo keluar |
| refund | Refund | Pengembalian dana |

### MinecraftEdition

| Code | Label | Description |
|------|-------|-------------|
| java | Java Edition | Username, 3-16 char |
| bedrock | Bedrock Edition | Gamertag, 3-12 char |
| manual | Manual | Tanpa ID, ambil manual |

### VoucherType

| Code | Label | Description |
|------|-------|-------------|
| percentage | Percentage | Diskon persen |
| fixed | Fixed | Diskon nominal |

### CategoryStatus

| Code | Label | Description |
|------|-------|-------------|
| active | Active | Berfungsi penuh |
| coming_soon | Coming Soon | Segera hadir |

### AnnouncementType

| Code | Label | Description |
|------|-------|-------------|
| info | Info | Informasi umum |
| maintenance | Maintenance | Jadwal maintenance |
| warning | Warning | Peringatan |

### AnnouncementAudience

| Code | Label | Description |
|------|-------|-------------|
| all | All | Customer + Reseller |
| customer | Customer | Hanya customer |
| reseller | Reseller | Hanya reseller |
| admin | Admin | Admin + Owner |

### ResellerStatus

| Code | Label | Description |
|------|-------|-------------|
| active | Active | Aktif |
| inactive | inactive | Nonaktif |
| suspended | Suspended | Ditangguhkan |

### ProductDeletionResult

| Code | Label | Description |
|------|-------|-------------|
| hard_deleted | Hard Deleted | Dihapus permanen |
| soft_deleted | Soft Deleted | Dihapus lunak |
