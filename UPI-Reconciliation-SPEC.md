Below is a full, build-ready specification you can paste into Codex to generate a fully functional MVP in .NET 8 + SQL Server for a single-feature product: UPI Payment Reconciliation.

You can copy-paste this whole spec as-is.

⸻

SPEC: UPI Payment Reconciliation MVP (Single Feature)

0) Product definition

Build a SaaS-style backend + minimal UI that does exactly one thing:

Reconcile incoming UPI payment credits against merchant invoices/orders and output Matched / Unmatched / Partial / Duplicate results, with CSV export.

No other features. Avoid extra modules like accounting, dashboards, analytics, reminders, WhatsApp, KYC, etc.

⸻

1) Tech stack constraints
•.NET 8
•ASP.NET Core Web API
•Entity Framework Core
•SQL Server as primary database
•Background Worker: .NET Worker Service (or hosted BackgroundService inside API)
•Queue: implement an abstraction IQueue with two implementations:
•InMemoryQueue (default for local/dev)
•AzureServiceBusQueue (production-ready, optional)
•Auth: simple JWT auth for merchants (MVP)
•Logging: Serilog + structured logs
•API docs: Swagger
•UI: minimal Razor Pages OR minimal React (keep simple). If uncertain, use Razor Pages.

⸻

2) System overview

Services (projects)

Create a solution with 4 projects:
1.Recon.Api (ASP.NET Core Web API + Swagger + minimal UI endpoints)
2.Recon.Worker (.NET Worker Service consuming queue and performing matching)
3.Recon.Core (domain models, matching engine, interfaces, DTOs)
4.Recon.Infrastructure (EF Core DbContext, repositories, queue implementations, provider adapters)

⸻

3) Core workflows

3.1 Invoice ingestion (input)

Merchant can upload invoices/orders (CSV) OR create via API.

Each invoice has:
•InvoiceNo (string, unique per merchant)
•ExpectedAmount (decimal)
•InvoiceDate (datetime)
•optional CustomerRef (string)

3.2 Payment ingestion (input)

System receives payment credit events from providers (webhook) OR via a simple manual API (for testing).

Payment event fields normalized:
•Provider name (e.g., Razorpay/Cashfree/Manual)
•ProviderTxnId (string) OR UTR (string)
•Amount
•PayeeVpa
•PayerVpa (optional)
•TxnTime
•Remarks / reference (optional)
•RawJson stored for audit/debug

3.3 Processing (matching engine)

Worker:
•Deduplicates by unique constraint
•Normalizes & tokenizes remarks
•Finds invoice candidates
•Creates match row + allocation rows

3.4 Output (single feature)

Merchant can:
•View reconciliation results for a date range
•Export CSV for that range

⸻

4) Database model (SQL Server)

Tables

4.1 Merchants
•MerchantId UNIQUEIDENTIFIER PK
•Name NVARCHAR(128)
•PrimaryVpa NVARCHAR(128) NULL
•TimeZone NVARCHAR(64) NOT NULL DEFAULT ‘Asia/Kolkata’
•CreatedAt DATETIME2 NOT NULL
•IsActive BIT NOT NULL DEFAULT 1

4.2 MerchantUsers (MVP auth)
•UserId UNIQUEIDENTIFIER PK
•MerchantId FK
•Email NVARCHAR(256) UNIQUE
•PasswordHash NVARCHAR(256)
•CreatedAt DATETIME2
•IsActive BIT

Use bcrypt/ASP.NET Identity PasswordHasher (but keep simple).

4.3 Invoices
•InvoiceId UNIQUEIDENTIFIER PK
•MerchantId FK
•InvoiceNo NVARCHAR(64) NOT NULL
•ExpectedAmount DECIMAL(18,2) NOT NULL
•InvoiceDate DATETIME2 NOT NULL
•CustomerRef NVARCHAR(64) NULL
•Status NVARCHAR(32) NOT NULL DEFAULT ‘Open’  // Open, PartiallyPaid, Paid, Overpaid
•PaidAmount DECIMAL(18,2) NOT NULL DEFAULT 0
•CreatedAt DATETIME2 NOT NULL

Indexes
•UNIQUE: (MerchantId, InvoiceNo)
•INDEX: (MerchantId, InvoiceDate)
•INDEX: (MerchantId, ExpectedAmount)

4.4 PaymentEvents
•PaymentEventId UNIQUEIDENTIFIER PK
•MerchantId FK
•Provider NVARCHAR(32) NOT NULL
•ProviderTxnId NVARCHAR(64) NOT NULL
•Utr NVARCHAR(32) NULL
•Amount DECIMAL(18,2) NOT NULL
•PayeeVpa NVARCHAR(128) NOT NULL
•PayerVpa NVARCHAR(128) NULL
•TxnTime DATETIME2 NOT NULL
•Remarks NVARCHAR(256) NULL
•RawJson NVARCHAR(MAX) NOT NULL
•CreatedAt DATETIME2 NOT NULL

Indexes
•UNIQUE: (MerchantId, Provider, ProviderTxnId)
•INDEX: (MerchantId, TxnTime)
•INDEX: (MerchantId, Amount)

4.5 Matches
•MatchId UNIQUEIDENTIFIER PK
•MerchantId FK
•PaymentEventId FK UNIQUE
•InvoiceId FK NULL
•MatchStatus NVARCHAR(32) NOT NULL // Matched, Partial, Duplicate, Overpay, Unidentified, Ambiguous
•MatchType NVARCHAR(32) NOT NULL // Exact, Contains, AmountTime, Fuzzy, DuplicateRule, None
•Confidence INT NOT NULL // 0..100
•Reason NVARCHAR(256) NOT NULL
•MatchedAt DATETIME2 NOT NULL

Indexes
•INDEX: (MerchantId, MatchStatus, MatchedAt)

4.6 InvoicePayments (allocation mapping)
•InvoicePaymentId UNIQUEIDENTIFIER PK
•InvoiceId FK
•PaymentEventId FK
•AllocatedAmount DECIMAL(18,2) NOT NULL
•AllocatedAt DATETIME2 NOT NULL

Indexes
•UNIQUE: (InvoiceId, PaymentEventId) (prevent double allocation)

⸻

5) Matching engine specification (MVP rules)

5.1 Normalization
•Trim and uppercase InvoiceNo and extracted tokens
•Normalize amount to 2 decimals
•Convert all times to UTC internally; store TxnTime as UTC

5.2 Token extraction from Remarks

Extract candidate invoice references using:
•Merchant-configured prefixes (MVP: default common prefixes): INV, BK, ORD, REC
•Regex examples:
•\bINV[-\s]?\d{1,20}\b
•\bBK[-\s]?\d{1,20}\b
•also detect exact invoice numbers if merchant uses alphanumeric: \b[A-Z]{2,6}[-]?\d{2,20}\b
Store extracted tokens in-memory for matching (no new DB table needed in MVP).

5.3 Candidate invoice selection

For a payment event, find candidate invoices for the merchant:
•Prefer invoices in last 90 days with status not Paid (Open/PartiallyPaid/Overpaid allowed)
•Filter by same ExpectedAmount when using amount-based rules

5.4 Rules (in priority order)
1.Exact match
•If token equals InvoiceNo AND Amount == ExpectedAmount - PaidAmount
•Status: Matched (or Partial if invoice still has remaining)
•Confidence: 98
•Reason: Exact token+amount match
2.Contains match
•If Remarks contains InvoiceNo (case-insensitive) AND amount matches remaining due
•Confidence: 92
•Reason: Remarks contains invoice number
3.Partial payment
•If token matches invoice AND amount < remaining due
•Allocate amount
•Set invoice PaidAmount += amount, status PartiallyPaid
•Create match status Partial
•Confidence: 95
•Reason: Partial payment against referenced invoice
4.Overpay
•If token matches invoice AND amount > remaining due
•Allocate only remaining due to invoice
•Mark match as Overpay, store Reason including leftover amount
•Confidence: 90
•Reason: Overpayment; leftover unallocated
5.Duplicate
•If there exists prior PaymentEvent within 10 minutes with same Amount and same PayerVpa and same PayeeVpa
•Mark as Duplicate
•Confidence: 97
•Reason: Duplicate rule triggered
6.Amount + time heuristic (weak)
•If no reference token:
•Find open invoices with same remaining amount
•If exactly 1 invoice candidate created within last 7 days → mark Matched with Confidence 70
•If multiple candidates → mark Ambiguous (do NOT auto allocate)
•Reason accordingly
7.Else Unidentified
•Confidence 0
•Reason: No reliable match

5.5 Allocation rules
•Only allocate when match is Matched/Partial/Overpay and not Ambiguous/Duplicate/Unidentified
•Update invoice PaidAmount and Status:
•PaidAmount == ExpectedAmount → Paid
•PaidAmount between 0 and ExpectedAmount → PartiallyPaid
•PaidAmount > ExpectedAmount → Overpaid (should not happen if you cap allocation; treat remainder as unallocated)

5.6 Idempotency

If same webhook is received multiple times:
•API should accept and return 200
•DB unique constraint prevents duplicate PaymentEvent
•Worker should detect existing Match for that PaymentEvent and skip

⸻

6) API specification (REST)

6.1 Auth

POST /api/auth/register
•body: { "merchantName": "...", "email": "...", "password": "...", "primaryVpa": "..." }
•creates Merchant + MerchantUser
•returns JWT

POST /api/auth/login
•body: { "email": "...", "password": "..." }
•returns JWT

All other endpoints require Authorization: Bearer <token> and infer MerchantId from user.

⸻

6.2 Invoices

POST /api/invoices
Create invoice(s)
•body: single or array:

[
  {"invoiceNo":"INV-1001","expectedAmount":2500,"invoiceDate":"2025-12-01T00:00:00Z","customerRef":"CUST1"}
]

POST /api/invoices/import-csv
•multipart file upload
•CSV columns: invoice_no,expected_amount,invoice_date,customer_ref
•validate and insert; reject duplicates unless ?mode=upsert

GET /api/invoices?from=...&to=...&status=...
returns paginated list

⸻

6.3 Payment ingestion

POST /api/webhooks/manual
For MVP testing without real provider.
•body:

{
  "providerTxnId":"TXN123",
  "utr":"312345678901",
  "amount":2500,
  "payeeVpa":"merchant@upi",
  "payerVpa":"user@upi",
  "txnTime":"2025-12-16T10:00:00Z",
  "remarks":"INV-1001"
}

•store raw json
•enqueue message {paymentEventId}

POST /api/webhooks/{provider}
•Accept provider payload as raw JSON
•Validate signature headers if available (implement for Manual; for others keep placeholder with clear TODO)
•Map to normalized PaymentEvent (ProviderTxnId must be extracted)
•enqueue message

Important: respond within 200ms if possible; do not run matching inline.

⸻

6.4 Reconciliation output

GET /api/recon?from=...&to=...&status=Matched|Unidentified|Partial|Duplicate|Ambiguous
Return list of recon results:
•PaymentEvent info + match info + invoice info (if any)
•Include confidence, reason

GET /api/recon/summary?from=...&to=...
Return counts:
•total payments
•matched
•partial
•duplicates
•ambiguous
•unidentified
•total amount matched
•total amount unidentified

GET /api/recon/export?from=...&to=...
Return CSV with columns:
•txn_time, provider, provider_txn_id, utr, amount, payer_vpa, payee_vpa, remarks, match_status, invoice_no, allocated_amount, confidence, reason

⸻

7) Worker processing spec

7.1 Queue message contract

Message JSON:

{ "paymentEventId":"GUID" }

7.2 Worker behavior
•Poll queue
•Fetch PaymentEvent by ID
•If Match exists for PaymentEvent → ack and skip
•Run matching engine
•Write Matches row
•If allocated → create InvoicePayments row and update invoice PaidAmount/Status in a transaction

7.3 Transaction boundary

Matching + allocation must be wrapped in SQL transaction to avoid race conditions.

7.4 Concurrency
•Process messages in parallel with max degree (configurable, default 4)
•Protect invoice updates with optimistic concurrency (RowVersion) or transaction isolation

⸻

8) Minimal UI (optional but requested to be functional)

Implement Razor Pages minimal UI with:
•Login/Register page
•Upload invoices CSV page
•Manual webhook test page (create payment)
•Recon list with filters + export button

Keep UI extremely basic; this is not a product dashboard.

⸻

9) Security & compliance (MVP)
•Store secrets in config/env vars
•HTTPS only
•Validate payload sizes
•Merchant isolation: every query must filter by MerchantId
•Do not log full raw payload in error logs (store RawJson in DB; log truncated)

⸻

10) Non-functional requirements
•Handle 50 webhook requests/sec bursts (queue absorbs)
•Idempotent ingestion
•CSV export for up to 100k rows (stream response)
•99% correctness on deterministic rules; ambiguous cases should not auto-match

⸻

11) Testing requirements (must implement)

Use xUnit.

11.1 Unit tests (Core)
•Token extraction tests
•Each match rule tests:
•Exact
•Contains
•Partial
•Overpay
•Duplicate
•Amount-time single candidate
•Ambiguous multiple candidates
•Unidentified

11.2 Integration tests (Infrastructure)
•Unique constraints prevent duplicates
•Worker creates exactly one Match per PaymentEvent
•Allocation updates invoice correctly

11.3 E2E test (happy path)
•Import invoice
•Send manual webhook
•Worker processes
•Recon endpoint shows Matched
•Export returns correct row

Target: meaningful coverage (at least 80% on Core matcher).

⸻

12) Configuration

Use appsettings.json + env overrides.

Required settings:
•ConnectionStrings: SqlServer
•Jwt: Issuer, Audience, Key
•Queue: Provider = InMemory|AzureServiceBus
•AzureServiceBus: ConnectionString, QueueName

⸻

13) Deliverables (what code must include)
•Full runnable solution with docker-compose for local:
•SQL Server container
•API container
•Worker container
•EF Core migrations
•Seed script for demo merchant
•Swagger documented endpoints
•README with:
•Local run steps
•How to import invoices
•How to post manual webhook
•How to view recon and export

⸻

14) Hard constraints (do not violate)
•No extra product features
•No payment initiation / no UPI switching
•Only reconciliation and export
•Provider integrations should be adapter-based; Manual must work fully

⸻

OUTPUT FORMAT REQUEST TO CODEX

Generate:
1.Full source code for all projects
2.SQL migrations
3.Docker compose
4.Tests
5.README

⸻
⸻
