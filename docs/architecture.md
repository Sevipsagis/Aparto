# System Architecture Document

## 1. Architectural Principles
* **Cost Optimization:** 100% Serverless architecture. Zero idle costs.
* **Rapid Development:** End-to-end TypeScript ecosystem for code sharing and type safety between frontend, backend, and database clients.
* **Zero-Trust Security:** Assume all inputs are malicious. Strict validation, stateless authentication, and Row-Level Security (RLS).
* **Resilience & Idempotency:** Webhook handlers must be idempotent to prevent duplicate billing or payment verifications.

## 2. Technology Stack Selection

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Frontend (LIFF/Admin)** | React, Next.js, Tailwind CSS | High productivity, rich ecosystem. Deployed on Cloudflare Pages (Free, infinite scale). |
| **API / Compute** | Hono.js on Cloudflare Workers | V8 isolates. 0ms cold starts. 100k free requests/day. Native TypeScript support. |
| **Database** | Supabase (PostgreSQL) | Built-in Auth, instant REST/GraphQL APIs, RLS for data isolation. Generous free tier. |
| **Storage** | Cloudflare R2 | S3-compatible, zero egress fees. Ideal for storing payment slips and PDF receipts. |
| **Messaging** | LINE Messaging API | Core UI for tenants. Push/Reply messages and Rich Menus. |
| **QR Generation** | `promptpay-qr` (npm) | Lightweight library, runs natively on Edge workers. |

## 3. System Architecture Diagram



## 4. Component Deep Dive

### 4.1. Edge Compute Layer (Cloudflare Workers)
Acts as the central orchestrator and API gateway.
* **LINE Webhook Handler:** Verifies `x-line-signature`. Parses events (Follow, Message, Postback) and routes to specific controllers.
* **Admin API:** Handles requests from the Landlord dashboard (manage rooms, input meters). Protected by JWT.
* **Tenant API (LIFF):** Handles requests from Tenant LIFF apps (slip upload, profile setup).
* **CRON Triggers:** Scheduled workers (e.g., 1st of every month) to aggregate meter readings and dispatch billing notifications via LINE Push API.

### 4.2. Data & Persistence Layer (Supabase & Cloudflare R2)
* **PostgreSQL (Supabase):** Relational data model.
    * Tables: `users`, `rooms`, `utility_rates`, `meter_logs`, `invoices`, `payments`.
    * *Security:* Row-Level Security (RLS) enforces that tenants can only query `invoices` where `tenant_id` matches their authenticated `user_id`.
* **Cloudflare R2:** Object storage.
    * Buckets configured with strict CORS and signed-URL access only.
    * Structure: `aparto-slips/{year-month}/{invoice_id}.jpg`, `aparto-receipts/{year-month}/{invoice_id}.pdf`.

### 4.3. Client Layer
* **LINE OA Chat:** Primary interface for notifications, PromptPay QR display, and slip image submission.
* **LIFF App (Tenant):** Web-view inside LINE for complex inputs (registration forms, viewing detailed invoice breakdowns).
* **Web Dashboard (Admin):** Next.js SPA for landlords to manage properties, approve payments, and view analytics.

## 5. Security & Resilience Configurations

### 5.1. Threat Prevention
* **Signature Verification:** Reject any LINE webhook lacking a valid HMAC-SHA256 signature.
* **Environment Secrets:** `LINE_CHANNEL_SECRET`, `SUPABASE_SERVICE_KEY`, and database URLs must be injected via Cloudflare Encrypted Secrets. Never commit `.env` files.
* **Rate Limiting:** Implement Cloudflare rate limiting on the Webhook endpoint to mitigate DDoS attacks.

### 5.2. Idempotency & Error Handling
* **Webhook Duplication:** LINE may send duplicate webhooks. Use `x-line-delivery-context-id` to verify if an event was already processed (cache status in Cloudflare KV or DB).
* **Transactional Integrity:** Database updates (e.g., marking invoice as paid and updating room status) must execute within PostgreSQL transactions.
* **Failsafe Logging:** Unhandled exceptions at the Edge trigger alerts (e.g., via Sentry) and log failure payloads for manual replay.