# Data Dictionary — Fofa Shop

**Extracted from:** 18 Eloquent models + 42 migrations
**Confidence:** High (directly from code)
**Validation column:** sourced from Form Requests (`app/Http/Requests`), Livewire form `rules()`/`validate()` arrays, model casts, and migration constraints (`nullable`, `unique`, FK `exists`). `—` = no application- or DB-level rule (framework-managed column).

---

## 1. users

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| name | string | yes | — | required, string, max:255 | User's full name |
| email | string | yes | — | required, email, max:255, unique(users.email) | Unique email address |
| role | enum(UserRole) | yes | `customer` | required, in:customer,reseller,admin,owner | Customer/Reseller/Admin/Owner |
| is_active | boolean | yes | `true` | required, boolean | Account active status |
| email_notification_enabled | boolean | yes | `true` | required, boolean | Opt-in/out for announcement emails |
| email_verified_at | timestamp | no | null | — | Email verification timestamp |
| password | string | yes | — | required, string, min:8, confirmed | Hashed password |
| remember_token | string | no | null | — | Remember me token |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `wallet()`: HasOne → `Wallet`
- `resellerProfile()`: HasOne → `ResellerProfile`
- `transactions()`: HasMany → `Transaction`
- `notifications()`: HasMany → `Notification`

**Methods:** `isReseller()`, `isAdmin()`, `isOwner()`, `isActive()`, `isInactive()`, `resellerCallbackUrl()`

---

## 2. categories

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| name | string | yes | — | required, string, max:255 | Category display name |
| slug | string | yes | — | required, unique(categories.slug) | Unique URL slug |
| image_path | string | no | null | nullable, image, mimes:jpg,jpeg,png,webp, max:2048 | Uploaded image (public disk) |
| icon_url | string | no | null | nullable | External icon URL |
| display_order | integer | yes | `0` | required | Sort order |
| is_active | boolean | yes | `true` | required, boolean | Visibility toggle |
| status | enum(CategoryStatus) | yes | `coming_soon` | required, in:active,coming_soon | active/coming_soon |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `products()`: HasMany → `Product`
- `subcategories()`: HasMany → `Subcategory`

**Methods:** `getImageUrlAttribute()`, `isComingSoon()`, `isMinecraft()`

---

## 3. products

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| category_id | bigint (FK→categories) | yes | — | required, exists:categories,id | Parent category |
| subcategory_id | bigint (FK→subcategories) | no | null | nullable, exists:subcategories,id (must belong to category_id) | Subcategory (Minecraft) |
| name | string | yes | — | required, string, max:255 | Product display name |
| description | text | no | null | nullable, string, max:5000 | Product description |
| image_path | string | no | null | nullable, image, mimes:jpg,jpeg,png,webp, max:2048 | Product image (public disk) |
| price | decimal(12,2) | yes | — | required, numeric, min:0.01, decimal:12,2 | Customer selling price |
| cost_price | decimal(12,2) | yes | — | required, numeric, min:0.01, decimal:12,2 | Distributor buy price |
| is_active | boolean | yes | `true` | required, boolean | Product active status |
| deleted_at | timestamp | no | null | — | Soft delete timestamp |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `category()`: BelongsTo → `Category`
- `subcategory()`: BelongsTo → `Subcategory`
- `vendorMappings()`: HasMany → `ProductVendorMapping`
- `transactions()`: HasMany → `Transaction`

**Traits:** SoftDeletes

---

## 4. subcategories

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| category_id | bigint (FK→categories) | yes | — | required, exists:categories,id | Parent category (CASCADE) |
| name | string | yes | — | required, string, max:255 | Subcategory name |
| slug | string | yes | — | required, unique(category_id,slug) | Unique within category |
| image_path | string | no | null | nullable, image, mimes:jpg,jpeg,png,webp, max:2048 | Subcategory image |
| display_order | integer | yes | `0` | required | Sort order |
| is_active | boolean | yes | `true` | required, boolean | Visibility toggle |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Unique constraint:** `(category_id, slug)`

**Relationships:**
- `category()`: BelongsTo → `Category`
- `products()`: HasMany → `Product`

---

## 5. product_vendor_mappings

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| product_id | bigint (FK→products) | yes | — | required, exists:products,id | Parent product (CASCADE) |
| vendor | string | yes | — | required, in:digiflazz,tokovoucher,vipayment,apigames,minecraft_server, unique(product_id,vendor) | Vendor name (e.g., digiflazz) |
| vendor_product_code | string | yes | — | required, string, max:255 | Kode produk pada vendor |
| vendor_price | decimal(12,2) | yes | — | required, numeric, min:0, decimal:12,2 | Price at vendor |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Unique constraint:** `(product_id, vendor)`

**Relationships:**
- `product()`: BelongsTo → `Product`

---

## 6. reseller_profiles

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| user_id | bigint (FK→users) | yes | — | required, unique(user_id), exists:users,id | Unique, CASCADE |
| api_key | string | no | null | nullable, unique(api_key) | Unique API key for auth |
| api_secret | string | no | null | nullable | API secret |
| markup_percentage | decimal(5,2) | yes | `0` | required, decimal:5,2 | Platform markup % |
| callback_url | string | no | null | nullable | Webhook URL for status changes |
| status | enum(ResellerStatus) | yes | `active` | required, in:active,inactive,suspended | active/inactive/suspended |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `user()`: BelongsTo → `User`

---

## 7. wallets

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| user_id | bigint (FK→users) | yes | — | required, unique(user_id), exists:users,id | Unique, CASCADE |
| balance | decimal(12,2) | yes | `0` | required, decimal:12,2 | Current balance |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `user()`: BelongsTo → `User`
- `walletTransactions()`: HasMany → `WalletTransaction`

---

## 8. wallet_transactions

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| wallet_id | bigint (FK→wallets) | yes | — | required, exists:wallets,id | CASCADE |
| type | enum(WalletTransactionType) | yes | — | required, in:deposit,deduction,refund | deposit/deduction/refund |
| amount | decimal(12,2) | yes | — | required, numeric, gt:0, decimal:12,2 | Transaction amount |
| reference_id | bigint (FK→transactions) | no | null | nullable, exists:transactions,id, unique(reference_id,type) | Related transaction |
| notes | string(500) | no | null | nullable, string, max:500 | Optional notes |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Unique constraint:** `(reference_id, type)`

**Relationships:**
- `wallet()`: BelongsTo → `Wallet`
- `transaction()`: BelongsTo → `Transaction`

---

## 9. transactions

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| ref_id | string | yes | — | required, unique(transactions.ref_id) | Unique reference ID (FOFA-YYYYMMDD-XXXXXXXX) |
| user_id | bigint (FK→users) | no | null | nullable, exists:users,id | Nullable for guest checkout |
| guest_email | string | no | null | nullable, email, max:255 (required for guests) | Guest notification email |
| product_id | bigint (FK→products) | yes | — | required, exists:products,id | RESTRICT |
| target_game_id | string | no | null | required unless minecraft_edition=manual, string, max:255, java: regex ^[a-zA-Z0-9_]{3,16}$, bedrock: 3-12 chars | Game account ID |
| target_server_id | string | no | null | nullable, string, max:255 | Game server ID |
| minecraft_edition | enum(MinecraftEdition) | no | null | nullable, in:java,bedrock,manual | java/bedrock/manual |
| delivered_minecraft_username | string | no | null | nullable, string, max:255 | Delivered MC username |
| delivered_minecraft_password | string | no | null | nullable, string, max:255, encrypted | Encrypted MC password |
| delivery_note | text | no | null | nullable, string, max:1000 | Admin delivery note |
| amount | decimal(12,2) | yes | — | required, decimal:12,2 | Transaction amount |
| status | enum(TransactionStatus) | yes | `pending` | required, in:pending,paid,processing,waiting_fulfillment,success,failed,expired | Current status |
| expired_at | timestamp | no | null | nullable | Payment deadline |
| completed_at | timestamp | no | null | nullable | Completion timestamp |
| is_reconciliation | boolean | yes | `false` | required, boolean | Reconciliation flag |
| reconciled_at | timestamp | no | null | nullable | Reconciliation timestamp |
| reconciliation_note | string | no | null | nullable | Reconciliation note |
| refund_requested | boolean | yes | `false` | required, boolean | Refund request flag |
| refund_bank | string | no | null | required_if:refund_requested,true, string, max:50 | Refund destination bank |
| refund_account_number | string | no | null | required_if:refund_requested,true, string, max:50 | Refund account number |
| refund_account_name | string | no | null | required_if:refund_requested,true, string, max:100 | Refund account name |
| refund_requested_at | timestamp | no | null | nullable | When refund was requested |
| refund_completed_at | timestamp | no | null | nullable | When refund was completed |
| refund_completed_by | bigint (FK→users) | no | null | nullable, exists:users,id | Who completed refund |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `user()`: BelongsTo → `User`
- `product()`: BelongsTo → `Product`
- `payments()`: HasMany → `Payment`
- `distributorLogs()`: HasMany → `DistributorLog`
- `walletTransactions()`: HasMany → `WalletTransaction`
- `voucherUsage()`: HasOne → `VoucherUsage`

**Scopes:** `withoutFinancials()`, `forAdmin()`, `forOwner()`

---

## 10. payments

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| transaction_id | bigint (FK→transactions) | yes | — | required, exists:transactions,id | CASCADE |
| method | string | yes | — | required, string, in:active method codes | Payment method (qris/ewallet/va_*) |
| gateway_ref | string | no | null | nullable, string | Gateway reference ID |
| status | string | yes | `pending` | required | Payment status |
| signature_verified | boolean | yes | `false` | required, boolean | Signature check result |
| paid_at | timestamp | no | null | nullable | Payment confirmation time |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `transaction()`: BelongsTo → `Transaction`

---

## 11. distributor_logs

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| transaction_id | bigint (FK→transactions) | yes | — | required, exists:transactions,id | CASCADE |
| provider | string | yes | — | required | Provider name (digiflazz) |
| attempt_number | integer | yes | `1` | required, integer | Attempt sequence |
| request_payload | json | no | null | nullable, array | Outbound request |
| response_payload | json | no | null | nullable, array | Inbound response |
| http_status_code | integer | no | null | nullable, integer | HTTP response code |
| duration_ms | integer | no | null | nullable, integer | Request duration |
| status | string | yes | — | required | success/pending/failed |
| error_message | text | no | null | nullable | Error details |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `transaction()`: BelongsTo → `Transaction`

---

## 12. vouchers

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| code | string | yes | — | required, string, max:255, unique(vouchers.code) | Unique voucher code |
| type | enum(VoucherType) | yes | `percentage` | required, in:percentage,fixed | percentage/fixed |
| value | decimal(12,2) | yes | — | required, numeric, decimal:12,2, percentage: 0-100, fixed: min:1 | Discount value |
| min_order_amount | decimal(12,2) | no | null | nullable, numeric, min:0, decimal:12,2 | Minimum order |
| max_uses | unsignedInteger | no | null | nullable, integer, min:1 | Max total uses |
| used_count | unsignedInteger | yes | `0` | required, unsigned | Current usage count |
| starts_at | timestamp | no | null | nullable, date | Start period |
| expires_at | timestamp | no | null | nullable, date, after_or_equal:starts_at | End period |
| is_active | boolean | yes | `true` | required, boolean | Active status |
| created_by | bigint (FK→users) | no | null | nullable, exists:users,id | Creator |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `usages()`: HasMany → `VoucherUsage`
- `creator()`: BelongsTo → `User`

---

## 13. voucher_usages

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| voucher_id | bigint (FK→vouchers) | yes | — | required, exists:vouchers,id | CASCADE |
| user_id | bigint (FK→users) | no | null | nullable, exists:users,id | User who used it |
| transaction_id | bigint (FK→transactions) | no | null | nullable, exists:transactions,id | Related transaction |
| discount_amount | decimal(12,2) | yes | — | required, decimal:12,2 | Discount applied |
| created_at | timestamp | no | null | — | Record creation |

**Relationships:**
- `voucher()`: BelongsTo → `Voucher`
- `user()`: BelongsTo → `User`
- `transaction()`: BelongsTo → `Transaction`

---

## 14. announcements

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| title | string | yes | — | required, string, max:255 | Announcement title |
| content | text | yes | — | required, string, max:5000 | Announcement body |
| type | enum(AnnouncementType) | yes | `info` | required, in:info,maintenance,warning | info/maintenance/warning |
| target_audience | enum(AnnouncementAudience) | yes | `all` | required, in:all,customer,reseller,admin | Target audience |
| is_active | boolean | yes | `true` | required, boolean | Active status |
| published_at | timestamp | no | null | nullable, date | Publish timestamp |
| created_by | bigint (FK→users) | no | null | nullable, exists:users,id | Creator |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

**Relationships:**
- `creator()`: BelongsTo → `User`

---

## 15. admin_payment_methods

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| method_code | string | yes | — | required, unique(method_code) | Unique code (qris/ewallet/va_*) |
| display_name | string | yes | — | required | Display name |
| is_active | boolean | yes | `true` | required, boolean | Active status |
| sort_order | integer | yes | `0` | required | Display order |
| created_at | timestamp | no | null | — | Record creation |
| updated_at | timestamp | no | null | — | Record update |

---

## 16. notifications

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| user_id | bigint (FK→users) | yes | — | required, exists:users,id | CASCADE |
| title | string | yes | — | required | Notification title |
| message | text | yes | — | required | Notification body |
| read_at | timestamp | no | null | nullable | Read timestamp |
| created_at | timestamp | no | null | — | Record creation |

**Relationships:**
- `user()`: BelongsTo → `User`

---

## 17. audit_logs

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| user_id | bigint (FK→users) | no | null | nullable, exists:users,id | Actor (null for system) |
| action | string | yes | — | required | Action type |
| auditable_type | string | yes | — | required | Polymorphic type |
| auditable_id | unsignedBigInteger | yes | — | required, unsigned | Polymorphic ID |
| old_values | json | no | null | nullable, array | Previous state |
| new_values | json | no | null | nullable, array | New state |
| ip_address | string(45) | no | null | nullable, max:45 | Actor IP |
| user_agent | string | no | null | nullable | Actor UA |
| created_at | timestamp | no | null | — | Record creation |

**Relationships:**
- `user()`: BelongsTo → `User`
- `auditable()`: MorphTo

---

## 18. product_deletion_logs

| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | bigint (PK) | yes | auto-increment | — | Primary key |
| product_id | unsignedBigInteger | yes | — | required, unsigned | Deleted product ID |
| product_snapshot | json | yes | — | required, array | Full product state at deletion |
| deleted_by | bigint (FK→users) | no | null | nullable, exists:users,id | Who deleted |
| reason | text | no | null | nullable | Deletion reason |
| deleted_at | timestamp | yes | — | required | Deletion timestamp |
| created_at | timestamp | no | null | — | Record creation |

**Relationships:**
- `deleter()`: BelongsTo → `User`

---

## 19. Framework Tables

### sessions (Laravel built-in)
| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| id | string (PK) | yes | — | — | Session ID |
| user_id | bigint (FK→users) | no | null | — | Nullable |
| ip_address | string(45) | no | null | — | Client IP |
| user_agent | text | no | null | — | Client UA |
| payload | longText | yes | — | required | Session data |
| last_activity | integer | yes | — | required | Last activity timestamp |

### cache / cache_locks (Redis-backed)
| Column | Type | Required | Default | Validation | Description |
|--------|------|----------|---------|------------|-------------|
| key | string (PK) | yes | — | — | Cache key |
| value | mediumText | yes | — | required | Cached value |
| expiration | bigInteger | yes | — | required | TTL |

### jobs / job_batches / failed_jobs (Queue tables)
Standard Laravel queue infrastructure.

### personal_access_tokens (Sanctum)
Standard Sanctum token storage for reseller API auth.
