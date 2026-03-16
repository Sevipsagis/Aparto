# Requirements Specification

## 1. Project Overview
**Aparto** is a comprehensive, serverless rental management ecosystem integrated with LINE Official Account (LINE OA). It streamlines communication, billing, utility tracking, and payment verification between landlords and tenants while maintaining near-zero infrastructure costs.

## 2. User Roles
* **Tenant:** Residents using LINE OA to register, receive automated bills, submit payments, request maintenance, and manage receipts.
* **Landlord (Admin):** Property managers using a secure web dashboard/LIFF to configure rooms, input utility readings, verify slips, and monitor financials.

---

## 3. Functional Requirements

### 3.1 Tenant Module (LINE OA Interface)
| ID | Requirement | Description |
| :--- | :--- | :--- |
| **FR-T1** | **Tenant Onboarding** | Register via a LINE LIFF form mapping `line_user_id` to a specific room. |
| **FR-T2** | **Room Verification** | Validate registration against the landlord’s master list before granting access. |
| **FR-T3** | **Billing Notification** | Receive automated Push Messages containing itemized invoices (rent + utilities) and due dates. |
| **FR-T4** | **PromptPay QR** | Generate dynamic, ISO-20022 compliant PromptPay QR codes for exact invoice amounts. |
| **FR-T5** | **Payment Submission** | Upload payment slip images directly through the LINE chat interface. |
| **FR-T6** | **Receipt Access** | Download auto-generated PDF receipts via secure links upon payment verification. |
| **FR-T7** | **Payment History** | View historical invoices and payment statuses via a LIFF menu. |
| **FR-T8** | **Slip Rejection Recovery** | Receive instant notification if a slip is rejected (e.g., blurred, incorrect amount) with reasons and a prompt to re-upload. |
| **FR-T9** | **Maintenance Requests** | Submit repair requests (text/images) via LINE and track resolution status. |
| **FR-T10** | **Prorated Rent Calculation** | Support automated pro-rata rent calculation for mid-month move-ins or move-outs. |

### 3.2 Landlord Module (Admin Dashboard)
| ID | Requirement | Description |
| :--- | :--- | :--- |
| **FR-L1** | **Room Configuration** | Manage room statuses (vacant, occupied, maintenance) and base rent prices. |
| **FR-L2** | **Utility Rate Management** | Set and update global or room-specific rates for water and electricity. |
| **FR-L3** | **Meter Reading Input** | Input monthly utility readings. |
| **FR-L4** | **Billing Engine** | Automatically calculate utility costs: `(Current - Previous) * Rate` and append to base rent. |
| **FR-L5** | **Batch Invoicing** | Trigger bulk or individual invoice generation and dispatch LINE push notifications. |
| **FR-L6** | **Payment Verification** | Review uploaded slips, approve/reject transactions, and update invoice statuses. |
| **FR-L7** | **Auto-Receipt Generation** | Automatically generate and store serialized PDF receipts upon slip approval. |
| **FR-L8** | **Financial Dashboard** | View real-time metrics: collected revenue, outstanding balances, and occupancy rates. |
| **FR-L9** | **Automated Late Fees** | Configure and apply daily/monthly late fees automatically, triggering reminder notifications. |
| **FR-L10** | **Deposit Management** | Track security deposits and handle deductions upon tenant move-out. |
| **FR-L11** | **Targeted Broadcasts** | Send operational announcements (e.g., power outages) to specific rooms or all tenants without consuming standard LINE broadcast quota. |

---

## 4. Non-Functional Requirements (NFRs)

### 4.1 Security & Data Privacy
* **NFR-S1: Zero-Trust Data Isolation:** Implement PostgreSQL Row-Level Security (RLS) to guarantee tenants can only access data tied to their authenticated `tenant_id`.
* **NFR-S2: Webhook Integrity:** Enforce strict HMAC-SHA256 signature validation on all incoming LINE webhooks.
* **NFR-S3: Secret Management:** Store API keys and database credentials strictly in encrypted environment variables (e.g., Cloudflare Secrets).

### 4.2 Resilience & Reliability
* **NFR-R1: Idempotency:** Webhook processors and billing engines must be idempotent to prevent duplicate invoice generation or duplicate payment approvals during network retries.
* **NFR-R2: Storage Rate Limiting:** Restrict the number of image uploads per tenant per day to prevent storage exhaustion attacks (DDoS) on Cloudflare R2/Supabase Storage.
* **NFR-R3: Graceful Degradation:** Log failed push messages to a dead-letter queue for automated or manual replay.

### 4.3 Performance
* **NFR-P1: Edge Execution:** Core API logic must execute on Edge infrastructure (Cloudflare Workers) to ensure < 50ms cold starts and low latency.
* **NFR-P2: Asset Optimization:** Payment slips must be compressed before storage to minimize cloud storage consumption.