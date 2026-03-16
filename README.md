# 🏢 Aparto: Serverless Rental Management Ecosystem

**Aparto** is a next-generation property management solution designed to bridge the gap between landlords and tenants through the **LINE Messaging API**. Built on a high-concurrency, zero-idle cost serverless architecture.

### 🚀 Key Capabilities
- **Automated Billing Engine**: Monthly rent and utility calculation ($(Current - Previous) \times Rate$) with automated Flex Message dispatching.
- **Dynamic PromptPay Integration**: Real-time generation of ISO-20022 compliant QR codes for seamless payments.
- **Automated Verification**: Streamlined slip upload and landlord approval workflow with PDF receipt generation.
- **Landlord Dashboard (LIFF)**: Mobile-first interface for meter reading entries and financial monitoring.

### 🛠 Tech Stack
- **Runtime**: Cloudflare Workers (Edge Computing)
- **Framework**: Hono.js (Web Standards-based API)
- **Database**: Supabase (Postgres with RLS)
- **Frontend**: React + Vite + Tailwind CSS (LINE LIFF)
- **Infrastructure**: Event-driven, Stateless, and Serverless.
