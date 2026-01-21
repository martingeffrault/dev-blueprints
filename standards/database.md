# Database Conventions (2025)

> **Last updated**: January 2025
> **Scope**: Tables, columns, indexes, migrations

---

## Philosophy

Database schema is the foundation. Design for data integrity first, query performance second. The schema should be self-documenting.

---

## Table Naming

| Rule | Example |
|------|---------|
| Plural, snake_case | `users`, `order_items` |
| Descriptive | `user_preferences` not `up` |
| Join tables | `user_roles`, `order_products` |

---

## Column Naming

| Type | Convention | Example |
|------|------------|---------|
| Primary key | `id` | `id` |
| Foreign key | `<table>_id` | `user_id` |
| Boolean | `is_` prefix | `is_active`, `is_verified` |
| Timestamp | `_at` suffix | `created_at`, `updated_at`, `deleted_at` |
| Date | `_on` or `_date` suffix | `published_on`, `birth_date` |
| Count | `_count` suffix | `view_count`, `order_count` |

---

## Required Columns

Every table should have:

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  -- ... your columns ...
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Soft Deletes (when needed)

```sql
deleted_at TIMESTAMPTZ
```

---

## Data Types

| Use | Type | Notes |
|-----|------|-------|
| IDs | `UUID` | Prefer over auto-increment |
| Money | `DECIMAL(19,4)` | Never use FLOAT |
| Timestamps | `TIMESTAMPTZ` | Always with timezone |
| Text | `TEXT` | Prefer over VARCHAR |
| JSON | `JSONB` | For semi-structured data |
| Boolean | `BOOLEAN` | Never 0/1 |
| Enum | `TEXT` + CHECK | Or dedicated enum type |

---

## Indexes

### When to Index

- Foreign keys
- Columns in WHERE clauses
- Columns in ORDER BY
- Columns in JOIN conditions

### Naming

```sql
idx_<table>_<column(s)>

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_id_created_at ON orders(user_id, created_at);
```

---

## Migrations

### Naming

```
<timestamp>_<description>.sql

20250120143022_create_users_table.sql
20250120150000_add_email_to_users.sql
```

### Rules

- One change per migration
- Always include DOWN migration
- Test rollback before deploying
- Never modify deployed migrations

---

## Quick Reference

| What | Convention |
|------|------------|
| Table | `snake_case`, plural |
| Column | `snake_case` |
| Primary key | `id` |
| Foreign key | `<table>_id` |
| Boolean | `is_<something>` |
| Timestamp | `<action>_at` |
| Index | `idx_<table>_<columns>` |
| Migration | `<timestamp>_<description>` |
