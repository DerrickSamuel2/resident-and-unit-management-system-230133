# Schema Applied

The database schema for the Resident Directory App has been applied to the running PostgreSQL instance in this repository.

## Applied objects

- Extensions: `pgcrypto`, `citext`, `pg_trgm`
- Types: `app_role`, `announcement_audience`, `contact_request_status`
- Tables:
  - `auth_users`
  - `units`
  - `resident_profiles`
  - `profile_privacy_settings`
  - `announcements`
  - `contact_requests`
  - `audit_log`
- Indexes: documented in `schema.md`
- Triggers:
  - `set_updated_at()`-based updated_at triggers on all mutable tables
  - `resident_profiles.search_name` maintenance trigger

## How it was applied

Statements were executed via `psql` using the connection command in:

- `database/db_connection.txt`

Per container guidance, statements were executed one-at-a-time using `psql ... -c "SQL"`.

## Notes

- `resident_profiles.search_name` is a denormalized column maintained by trigger (generated column expression was rejected as non-immutable).
- A sample unit (`A-101`) was seeded with `ON CONFLICT DO NOTHING`.
