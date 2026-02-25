# Resident Directory App — Database Schema (PostgreSQL)

This document describes the **current PostgreSQL schema** created in the `database` container for the Resident Directory App.

## Connection

Connection helper (created by container startup):

- `database/db_connection.txt` contains a `psql postgresql://...` command.

Example:

- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Extensions

The schema enables the following extensions:

- `pgcrypto` (UUID generation via `gen_random_uuid()`)
- `citext` (case-insensitive email columns)
- `pg_trgm` (trigram indexes for fast partial name search)

## Enums

### `app_role`
Roles for application users:

- `admin`
- `staff`
- `resident`

### `announcement_audience`
Announcement targeting:

- `all`
- `staff`
- `residents`

### `contact_request_status`
Workflow for optional contact requests:

- `pending`
- `approved`
- `declined`
- `cancelled`

## Tables

### `auth_users`
Application identity table (for auth + roles).

Columns (key):
- `id` (uuid, PK)
- `email` (citext, unique)
- `password_hash` (text, nullable; backend may use external auth later)
- `role` (app_role)
- `is_active` (bool)
- `email_verified_at`, `last_login_at`
- `created_at`, `updated_at`

Indexes:
- `idx_auth_users_role` on `role`

Triggers:
- `updated_at` maintained by `trg_auth_users_updated_at`

---

### `units`
Represents a unit in a property (supports resident/unit management).

Columns (key):
- `id` (uuid, PK)
- `building` (text, nullable)
- `unit_number` (text, required)
- `floor` (text, nullable)
- `notes` (text, nullable)
- `created_at`, `updated_at`

Constraints:
- `UNIQUE (building, unit_number)` (allows building nulls; intended for simple properties)

Triggers:
- `updated_at` maintained by `trg_units_updated_at`

---

### `resident_profiles`
Resident/staff profile tied to an `auth_users` row.

Columns (key):
- `id` (uuid, PK)
- `user_id` (uuid, unique FK -> auth_users.id ON DELETE CASCADE)
- `unit_id` (uuid, nullable FK -> units.id ON DELETE SET NULL)

Profile fields:
- `first_name`, `last_name` (required)
- `preferred_name` (nullable)
- `phone` (nullable)
- `email_secondary` (citext, nullable)
- `bio` (nullable)

Photo metadata (optional):
- `photo_url` (text)
- `photo_storage_key` (text) — for object storage integration
- `photo_content_type` (text)
- `photo_file_size_bytes` (int, >= 0)
- `photo_width`, `photo_height` (int, > 0)

Resident lifecycle / approval:
- `move_in_date`, `move_out_date` (date)
- `is_resident` (bool)
- `is_profile_approved` (bool)
- `approved_at` (timestamptz)
- `approved_by` (uuid FK -> auth_users.id ON DELETE SET NULL)

Directory search helper:
- `search_name` (text) — denormalized and maintained by trigger:
  - `trg_resident_profiles_set_search_name`

Timestamps:
- `created_at`, `updated_at` (updated by trigger)

Indexes:
- `idx_resident_profiles_unit_id` on `unit_id`
- `idx_resident_profiles_approved` on `(is_profile_approved, is_resident)`
- `idx_resident_profiles_search_name_trgm` gin trigram index on `search_name`

Notes:
- `search_name` is used for fast name search (partial matching).
- This design supports privacy via the separate `profile_privacy_settings` table.

---

### `profile_privacy_settings`
Privacy/visibility controls per profile (1:1 with `resident_profiles`).

Columns:
- `profile_id` (uuid, PK/FK -> resident_profiles.id ON DELETE CASCADE)
- `show_in_directory` (bool)
- `show_unit` (bool)
- `show_phone` (bool)
- `show_email` (bool)
- `show_photo` (bool)
- `allow_contact_requests` (bool)
- `created_at`, `updated_at`

Triggers:
- `updated_at` maintained by `trg_profile_privacy_settings_updated_at`

---

### `announcements`
Announcements created by admin/staff and optionally published.

Columns:
- `id` (uuid, PK)
- `title` (text)
- `body` (text)
- `audience` (announcement_audience)
- `is_published` (bool)
- `published_at` (timestamptz)
- `created_by` (uuid FK -> auth_users.id ON DELETE SET NULL)
- `created_at`, `updated_at`

Indexes:
- `idx_announcements_published_at` on `(is_published, published_at DESC)`

Triggers:
- `updated_at` maintained by `trg_announcements_updated_at`

---

### `contact_requests` (optional messaging/contact workflow)
Allows a resident to request contact with another resident without exposing private info.

Columns:
- `id` (uuid, PK)
- `requester_profile_id` (uuid FK -> resident_profiles.id ON DELETE CASCADE)
- `target_profile_id` (uuid FK -> resident_profiles.id ON DELETE CASCADE)
- `message` (text, nullable)
- `status` (contact_request_status)
- `decided_at` (timestamptz, nullable)
- `decided_by` (uuid FK -> auth_users.id ON DELETE SET NULL)
- `created_at`, `updated_at`

Constraints:
- `contact_requests_not_self` ensures requester != target

Indexes:
- `uniq_contact_requests_active` unique partial index on `(requester_profile_id, target_profile_id)` where status = 'pending'
- `idx_contact_requests_target_status` on `(target_profile_id, status, created_at DESC)`

Triggers:
- `updated_at` maintained by `trg_contact_requests_updated_at`

---

### `audit_log`
Security/audit trail for important actions.

Columns:
- `id` (uuid, PK)
- `actor_user_id` (uuid FK -> auth_users.id ON DELETE SET NULL)
- `action` (text)
- `entity_type` (text, nullable)
- `entity_id` (uuid, nullable)
- `metadata` (jsonb)
- `created_at` (timestamptz)

Indexes:
- `idx_audit_log_actor_created` on `(actor_user_id, created_at DESC)`

## Common trigger function

### `set_updated_at()`
A shared trigger function that sets `NEW.updated_at = now()`.

Used by:
- `auth_users`
- `units`
- `resident_profiles`
- `profile_privacy_settings`
- `announcements`
- `contact_requests`

## Seed data (minimal)

A single sample unit is inserted for development/demo:

- building `A`, unit_number `101`

This is inserted with `ON CONFLICT DO NOTHING` to remain idempotent.
