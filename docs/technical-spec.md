# Technical Specification Document

## 1. Executive Summary
Aparto is a serverless, event-driven rental management ecosystem integrated with LINE Official Account (OA). It automates billing, utility calculation, and payment verification, operating with near-zero idle cloud costs. The architecture prioritizes security, idempotency, and high performance via edge computing.

## 2. Functional Requirements

### 2.1 Tenant Module (LINE OA & LIFF)
* **Onboarding**: Tenants register via a LIFF form, mapping their `line_user_id` to a specific room.
* **Billing Notification**: Automated push messages containing itemized invoices (base rent + utilities) and due dates.
* **Payment**: Dynamic PromptPay QR (ISO-20022 compliant) generation. Slip images are uploaded directly via LINE chat.
* **Receipt Access**: System provides secure download links for auto-generated PDF receipts upon payment verification.

### 2.2 Landlord Module (Web Dashboard)
* **Property Management**: Manage room configurations, base rent, and occupancy statuses.
* **Utility Configuration**: Set and update global rates for water and electricity.
* **Meter Reading**: Input monthly readings. The system automatically calculates utility costs: $Cost = (Current - Previous) \times Rate$.
* **Payment Verification**: Dashboard to approve/reject uploaded slips, triggering automatic receipt generation.

---

## 3. System Architecture

### 3.1 Architectural Principles
* **Cost Optimization**: 100% Serverless architecture.
* **Zero-Trust Security**: Strict validation, stateless authentication, and database-level Row-Level Security (RLS).
* **Resilience**: Webhook handlers are idempotent to prevent duplicate billing or state mutations.

### 3.2 Technology Stack
* **Compute/API**: Cloudflare Workers (V8 Edge Isolates), Hono.js.
* **Database**: Supabase (PostgreSQL).
* **Storage**: Cloudflare R2 (S3-compatible, zero egress fees).
* **Frontend**: React, Next.js, Tailwind CSS (hosted on Cloudflare Pages).
* **Messaging**: LINE Messaging API.

### 3.3 Core Workflows
1.  **Edge Routing**: Cloudflare Workers intercept LINE webhooks, verify `x-line-signature`, and route to specific Hono controllers.
2.  **Billing Cycle**: Scheduled workers aggregate meter readings, generate invoices, and dispatch LINE Flex Messages to tenants.
3.  **Payment Verification**: Slips are routed to R2. The database updates payment status to `pending`. Upon admin approval, a headless process generates a PDF to R2 and pushes the URL to the tenant.

---

## 4. Database Schema

### 4.1 Enums
```sql
CREATE TYPE user_role AS ENUM ('admin', 'tenant');
CREATE TYPE room_status AS ENUM ('vacant', 'occupied', 'maintenance');
CREATE TYPE invoice_status AS ENUM ('unpaid', 'pending_verification', 'paid', 'overdue');
CREATE TYPE payment_status AS ENUM ('pending', 'approved', 'rejected');

```

### 4.2 Tables

#### `users`

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Internal identifier. |
| `line_user_id` | VARCHAR(255) | UNIQUE, NOT NULL | LINE Platform User ID. |
| `role` | user_role | NOT NULL, DEFAULT 'tenant' | System access level. |
| `display_name` | VARCHAR(255) | NOT NULL | LINE Display Name. |
| `phone_number` | VARCHAR(20) |  | Contact number. |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

#### `rooms`

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `room_number` | VARCHAR(10) | UNIQUE, NOT NULL | e.g., "101", "A205". |
| `base_rent` | NUMERIC(10,2) | NOT NULL, CHECK (base_rent >= 0) | Standard monthly rent. |
| `status` | room_status | NOT NULL, DEFAULT 'vacant' |  |
| `tenant_id` | UUID | REFERENCES users(id) ON DELETE SET NULL | Current occupant. |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

#### `utility_rates`

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `water_rate` | NUMERIC(10,2) | NOT NULL, CHECK (water_rate >= 0) | Cost per unit. |
| `electric_rate` | NUMERIC(10,2) | NOT NULL, CHECK (electric_rate >= 0) | Cost per unit. |
| `effective_from` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Rate active date. |

#### `meter_readings`

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `room_id` | UUID | REFERENCES rooms(id) ON DELETE CASCADE |  |
| `billing_cycle` | DATE | NOT NULL | Format: YYYY-MM-01. |
| `water_prev` | NUMERIC(10,2) | NOT NULL, CHECK (water_prev >= 0) |  |
| `water_curr` | NUMERIC(10,2) | NOT NULL, CHECK (water_curr >= water_prev) |  |
| `elec_prev` | NUMERIC(10,2) | NOT NULL, CHECK (elec_prev >= 0) |  |
| `elec_curr` | NUMERIC(10,2) | NOT NULL, CHECK (elec_curr >= elec_prev) |  |
| `recorded_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

#### `invoices`

| Column | Type | Constraints | Description |
| --- | --- | --- | --- |
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() |  |
| `room_id` | UUID | REFERENCES rooms(id) |  |
| `tenant_id` | UUID | REFERENCES users(id) | Snapshotted tenant. |
| `billing_cycle` | DATE | NOT NULL | Format: YYYY-MM-01. |
| `base_rent` | NUMERIC(10,2) | NOT NULL, CHECK (base_rent >= 0) |  |
| `water_amount` | NUMERIC(10,2) | NOT NULL, CHECK (water_amount >= 0) | Calculated cost. |
| `elec_amount` | NUMERIC(10,2) | NOT NULL, CHECK (elec_amount >= 0) | Calculated cost. |
| `total_amount` | NUMERIC(10,2) | NOT NULL, CHECK (total_amount >= 0) | Total to pay. |
| `status` | invoice_status | NOT NULL, DEFAULT 'unpaid' |  |
| `due_date` | DATE | NOT NULL |  |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

#### `payments`

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

---

## 5. Security & Performance Configurations

* **Indexes**: Applied to `users(line_user_id)`, `rooms(tenant_id)`, `invoices(tenant_id, status)`, and `payments(invoice_id)` for high-speed queries.
* **Data Isolation**: Row-Level Security (RLS) policies enforce that tenants can only execute `SELECT` on records where `tenant_id = auth.uid()`. Admins maintain unrestricted access.
* **Environment Secrets**: Credentials (`LINE_CHANNEL_SECRET`, database connection strings) are strictly managed via Cloudflare Encrypted Secrets.
