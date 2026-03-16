# Database Schema Document

## 1. Overview
This schema is designed for PostgreSQL (Supabase). It enforces data integrity via strict constraints, utilizes UUIDs to prevent enumeration attacks, and implements Row-Level Security (RLS) for zero-trust data access.

## 2. Enums
```sql
CREATE TYPE user_role AS ENUM ('admin', 'tenant');
CREATE TYPE room_status AS ENUM ('vacant', 'occupied', 'maintenance');
CREATE TYPE invoice_status AS ENUM ('unpaid', 'pending_verification', 'paid', 'overdue');
CREATE TYPE payment_status AS ENUM ('pending', 'approved', 'rejected');

```

## 3. Tables

### 3.1. `users`

Stores all system identities mapped to LINE OA.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Internal identifier. |
| `line_user_id` | VARCHAR(255) | UNIQUE, NOT NULL | LINE Platform User ID. |
| `role` | user_role | NOT NULL, DEFAULT 'tenant' | System access level. |
| `display_name` | VARCHAR(255) | NOT NULL | LINE Display Name. |
| `phone_number` | VARCHAR(20) |  | Contact number. |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

### 3.2. `rooms`

Physical properties and their current occupancy state.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `room_number` | VARCHAR(10) | UNIQUE, NOT NULL | e.g., "101", "A205". |
| `base_rent` | NUMERIC(10,2) | NOT NULL, CHECK (base_rent >= 0) | Standard monthly rent. |
| `status` | room_status | NOT NULL, DEFAULT 'vacant' |  |
| `tenant_id` | UUID | REFERENCES users(id) ON DELETE SET NULL | Current occupant. NULL if vacant. |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

### 3.3. `utility_rates`

Global configurations for utility billing. Designed as an append-only log to maintain historical calculation accuracy.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `water_rate` | NUMERIC(10,2) | NOT NULL, CHECK (water_rate >= 0) | Cost per unit. |
| `electric_rate` | NUMERIC(10,2) | NOT NULL, CHECK (electric_rate >= 0) | Cost per unit. |
| `effective_from` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Rate active date. |

### 3.4. `meter_readings`

Monthly snapshots of utility usage.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `room_id` | UUID | REFERENCES rooms(id) ON DELETE CASCADE |  |
| `billing_cycle` | DATE | NOT NULL | Format: YYYY-MM-01. |
| `water_prev` | NUMERIC(10,2) | NOT NULL, CHECK (water_prev >= 0) |  |
| `water_curr` | NUMERIC(10,2) | NOT NULL, CHECK (water_curr >= water_prev) | Ensures valid increment. |
| `elec_prev` | NUMERIC(10,2) | NOT NULL, CHECK (elec_prev >= 0) |  |
| `elec_curr` | NUMERIC(10,2) | NOT NULL, CHECK (elec_curr >= elec_prev) | Ensures valid increment. |
| `recorded_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

*Constraint:* `UNIQUE (room_id, billing_cycle)` prevents duplicate readings per month.

### 3.5. `invoices`

Immutable records of monthly billing.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `room_id` | UUID | REFERENCES rooms(id) |  |
| `tenant_id` | UUID | REFERENCES users(id) | Snapshotted tenant at billing time. |
| `billing_cycle` | DATE | NOT NULL | Format: YYYY-MM-01. |
| `base_rent` | NUMERIC(10,2) | NOT NULL, CHECK (base_rent >= 0) |  |
| `water_amount` | NUMERIC(10,2) | NOT NULL, CHECK (water_amount >= 0) | Calculated cost. |
| `elec_amount` | NUMERIC(10,2) | NOT NULL, CHECK (elec_amount >= 0) | Calculated cost. |
| `total_amount` | NUMERIC(10,2) | NOT NULL, CHECK (total_amount >= 0) | Total to pay. |
| `status` | invoice_status | NOT NULL, DEFAULT 'unpaid' |  |
| `due_date` | DATE | NOT NULL |  |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

### 3.6. `payments`

Tracks slip uploads and verification statuses.

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `invoice_id` | UUID | REFERENCES invoices(id) ON DELETE CASCADE |  |
| `amount_paid` | NUMERIC(10,2) | NOT NULL, CHECK (amount_paid > 0) |  |
| `slip_key` | TEXT | NOT NULL | Cloudflare R2 object key. |
| `receipt_key` | TEXT |  | Cloudflare R2 object key (PDF). |
| `status` | payment_status | NOT NULL, DEFAULT 'pending' |  |
| `verified_by` | UUID | REFERENCES users(id) | Admin who approved. |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

## 4. Indexes (Performance Optimization)

```sql
CREATE INDEX idx_users_line_id ON users(line_user_id);
CREATE INDEX idx_rooms_tenant ON rooms(tenant_id);
CREATE INDEX idx_invoices_tenant_status ON invoices(tenant_id, status);
CREATE INDEX idx_payments_invoice ON payments(invoice_id);

```

## 5. Security: Row-Level Security (RLS)

PostgreSQL RLS must be enabled on all tables to ensure tenant data isolation.

**Example: `invoices` Table Policies**

```sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

-- Admins can read all invoices
CREATE POLICY "Admins can view all invoices" 
ON invoices FOR SELECT 
USING (EXISTS (SELECT 1 FROM users WHERE users.id = auth.uid() AND users.role = 'admin'));

-- Tenants can only read their own invoices
CREATE POLICY "Tenants can view own invoices" 
ON invoices FOR SELECT 
USING (tenant_id = auth.uid());

```

*(Note: `auth.uid()` assumes mapping via Supabase Auth or custom JWT claim injection inside the Cloudflare Worker prior to querying).*
