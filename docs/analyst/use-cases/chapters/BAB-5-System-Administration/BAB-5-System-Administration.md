# BAB 5: System Administration & Configuration

**Fokus:** Manajemen akun admin, pengumuman broadcast, dan konfigurasi payment method.

**Boundary:** Authenticated, Owner-only untuk admin management.

**Process Flow Related:** PF-010 (Announcement Broadcast).

---

## Use Cases

| UC | Nama | Actors | Trigger | Status |
|----|------|--------|---------|--------|
| UC-021 | Kelola Akun Admin | Owner | Buat/toggle admin | [x] Confirmed |
| UC-022 | Broadcast Pengumuman | Admin | Publish pengumuman | [x] Confirmed |
| UC-023 | Toggle Payment Method | Admin | Aktif/nonaktifkan metode | [x] Confirmed |

---

## UC-021: Kelola Akun Admin

Owner membuat dan mengelola akun admin lain.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Owner | Pemilik | Membuat/mengelola admin |

### Preconditions

- Role: owner

### Postconditions

- Akun admin dibuat atau status diubah

### Main Flow — Create Admin

1. **Owner** akses `/owner/admins`
2. **Owner** buat admin baru (name, email, password)
3. **System** `AdminService::createAdmin` (hashed, verified, active)
4. **System** buat audit log

### Main Flow — Toggle Admin

1. **Owner** klik toggle active/inactive
2. **System** `AdminService::toggleAdmin`
3. **System** flip `is_active`

### Alternative Flows

#### AF-1: Self-Toggle Attempt
- **Trigger:** Owner coba toggle diri sendiri
- **Step 2a:** Reject (DomainException)

#### AF-2: Non-Admin Toggle
- **Trigger:** Target bukan admin
- **Step 2a:** Reject (DomainException)

### Business Rules

- **BR-AUTH-002:** Hanya owner yang bisa buat/toggle admin
- **BR-AUTH-004:** Tidak bisa toggle diri sendiri

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| users | Create/Update | role: admin |
| AuditLogService | Log | admin_created/admin_toggled |

### Notes

- Implementasi: `Owner\AdminController::store`, `toggle`

---

## UC-022: Broadcast Pengumuman

Admin mempublish pengumuman in-app + email blast ke target audience.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | Operator | Mengirim pengumuman ke user |

### Preconditions

- Authenticated, role: admin atau owner

### Postconditions

- Pengumuman terkirim ke target audience

### Main Flow

1. **Admin** isi form pengumuman (title, type, content, target_audience)
2. **System** simpan announcement (status: published)
3. **System** dispatch `SendAnnouncementJob`
4. **Job** filter user berdasarkan target_audience
5. **Job** buat InAppNotification per user
6. **Job** kirim email (jika `email_notification_enabled = true`)

### Audience Rules

| target_audience | Recipients |
|-----------------|------------|
| all | customer + reseller |
| customer | customer only |
| reseller | reseller only |
| admin | admin + owner (notif only, no email) |

### Business Rules

- **BR-ANN-001:** Admin/owner tidak dapat announcement
- **BR-ANN-002:** Email hanya dikirim jika `email_notification_enabled = true`

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| announcements | Create | status: published |
| in_app_notifications | Create | Per recipient |
| SendAnnouncementJob | Dispatch | Async queue |

### Notes

- Implementasi: Livewire announcement form + `SendAnnouncementJob`

---

## UC-023: Toggle Payment Method

Admin mengaktifkan atau menonaktifkan metode pembayaran.

### Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | Operator | Kelola metode pembayaran |

### Preconditions

- Authenticated, role: admin atau owner

### Postconditions

- Metode payment aktif/nonaktif berubah

### Main Flow

1. **Admin** akses halaman payment methods
2. **Admin** klik toggle active/inactive
3. **System** flip `is_active`
4. **Metode nonaktif tidak tampil di checkout**

### Business Rules

- **BR-PAY-003:** Metode nonaktif tidak disertakan di `activeMethodsOrdered()`

### Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| admin_payment_methods | Update | is_active toggle |

### Notes

- Implementasi: `AdminPaymentMethod::toggleActive`
- Data: Tripay, Duitku (2 gateway aktif)
