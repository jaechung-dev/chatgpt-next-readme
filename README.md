# AI-powered eCommerce customer service system
## - supports multiple merchants

### A Next.js 15 + React 19 application for running multi-tenant group-buy and preorder campaigns. Includes a Cart/Order drawer UX, MongoDB backend with transaction-safe stock reservation, and an LLM (ChatGPT)-enhanced Admin Panel for customer service and operations.

⸻

### Highlights
	•	Multi-tenant routing: per-company and per-form isolation using path params
	•	URLs: https://chatgptform.com/forms/order/:companyName/:formId (customer UI)
	•	Data partitioning via companyName and preorderFormId on all product/order docs
	•	Asset paths: images/<companyName>/<preorderFormId>/<slug>.png
	•	LLM Admin Panel (/admin)
	•	ChatGPT-assisted customer support: order lookup, stock checks, status updates, canned responses
	•	MCP-style tool calls (serverless) to read/write domain data (orders, stock, pickups)
	•	Channel intents: Instagram Messenger, KakaoTalk, SMS (extensible)
	•	Guardrails and summarization for agent replies
	•	Drawer UX: Cart, Product Detail, and Order flows using a bounded drawer pattern
	•	Stock signals: Live stock badge overlay; transactional reservation on checkout
	•	Serverless: API routes under /api, deployable to AWS Lambda + CloudFront + S3
	•	Type-safe: Shared types.ts, utility helpers for pricing and cart math

⸻

## Tech Stack

### Frontend
	•	Next.js 15 (App Router, React Server Components)
	•	React 19 (e.g., useOptimistic, useFormState)
	•	Tailwind CSS
	•	Zustand stores: useCartStore, useDrawerStore, useProductsStore, useStockStore
	•	clsx for conditional classes; minimal headless UI components
	•	i18n via t() helper

### Backend
	•	Next.js API Routes following a Model–Controller–Service (MCS) pattern  
    •	Node.js Express server for WebSocket communication
    •   MongoDB Atlas + Mongoose, Redis
	•	Transactional controllers: orderController, stockController
	•	Singleton DB connector: dbConnect() to prevent multi-client/session errors

### Infra / Integrations
	•	AWS Lambda, CloudFront, S3, Fargate
	•	GitHub Actions (build/lint/deploy)
	•	ChatGPT integration (server-side) for admin panel
	•	MCP-style tool layer for LLM actions

⸻

## Project Structure

```plaintext
chatgpt-next/
├── .github/
│   └── workflows/              # CI/CD pipelines
├── public/
│   └── images/                 # static assets
│       └── <company>/<formId>/<product>.png
├── src/
│   ├── app/                    # Next.js App Router (RSC + routes)
│   │   ├── api/
│   │   │   └── orders/[id]/    # API routes
│   │   │       └── route.ts
│   │   └── forms/
│   │       └── order/[companyName]/[formId]/
│   │           └── page.tsx    # entry for customer preorder form
│   ├── components/
│   │   ├── forms/
│   │   │   ├── parts/          # ProductDetails, CartContent, etc.
│   │   │   ├── shared/         # ProductImage, StockBadgeOverlay
│   │   │   └── utils/          # cartUtils.ts, etc.
│   │   └── shared/             # Drawer, modal, generic UI
│   ├── lib/                    # shared libs
│   │   ├── dbConnect.ts        # MongoDB singleton
│   │   └── i18n.ts             # translations
│   ├── models/                 # Mongoose models
│   │   ├── Order.ts
│   │   └── Stock.ts
│   ├── controllers/            # business logic (MVC style)
│   │   ├── orderController.ts
│   │   └── stockController.ts
│   ├── stores/                 # Zustand stores
│   │   ├── useCartStore.ts
│   │   ├── useDrawerStore.ts
│   │   ├── useProductsStore.ts
│   │   └── useStockStore.ts
│   └── types.ts                # Product, Order, shared interfaces
├── .eslintrc.js / eslint.config.js
├── package.json
├── tsconfig.json
├── Dockerfile
└── README.md
```
⸻

## Environment Variables(.env.local, .env.production)

```json
# Server
NEXT_PUBLIC_BASE_URL=
NEXT_PUBLIC_IMAGE_BASE_URL=

# Socket
NEXT_PUBLIC_SOCKET_URL=

# DB
MONGODB_URI=
REDIS_URL=

# AWS
AWS_REGION=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
NEXT_PUBLIC_AWS_LAMBDA_END_POINT=
S3_BASE_URL=

# Email
LOCAL_EMAIL=
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASS=
MAIL_FROM=
MAIL_TO_ADMIN=

# 3rd party API
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY=
SNS_API(1-n) .... = 
```

⸻

## Multi-Tenant Model

### Routing
	•	Customer UI: https://chatgptform.com/forms/order/:companyname/:formId
	•	Example: https://chatgptform.com/forms/order/acme/68909461e93e8196b0fd577a

### Data Partitioning
	•	Product includes companyName and preorderFormId
	•	Order includes companyName and preorderFormId
	•	All queries must filter by both to enforce isolation

### Assets
	•	images/<companyName>/<preorderFormId>/<product-slug>-<option>.png
	•	Alt attributes derived from product slug + option

⸻

## LLM supported Admin Panel

### Route: /admin

### Auth (Cognito, JWT)

### Capabilities
	•	Chat console for operators with:
	•	Order lookup by email/mobile/token
	•	Stock checks and adjustments (guarded)
	•	Create/modify order notes, pickup/delivery details
	•	Generate polite, translated customer replies (Korean/English)
	•	Auto-summaries of long customer threads
	•	MCP-style Tools (serverless):
	•	orders.findOne, orders.update, stock.getById, stock.reserve, pickupDelivery.create …
	•	Tools enforce company + form scoping and validation

## Safety/Guardrails
	•	Tool results are summarized; PII is masked in logs
	•	All write operations require explicit operator confirmation in UI

⸻

## DB Connection Pattern
	•	Use a single dbConnect() (singleton) everywhere
	•	Define models with mongoose.model on the default connection
	•	Transactions:
	•	const session = await mongoose.startSession()
	•	await session.withTransaction(async () => { ... })
	•	Pass { session } into all queries within the txn

## WebSocket
	•	IO path: /socket
	•	Broadcast stock updates to subscribed product rooms

⸻

@@ API Reference (excerpt)

POST /api/orders/:formId

Create an order and reserve stock in a transaction.


Request (JSON)
```json
{
  "companyName": "acme",
  "items": [
    {
      "id": "solefrutta-jam",
      "name": "Solefrutta Jam",
      "priceOptions": [{"quantity":1,"price":12}],
      "selectedOptions": {"Flavor": "Grape"},
      "quantity": 2,
      "subtotal": 24
    }
  ],
  "total": 24,
  "customer": {
    "name": "Jane Doe",
    "email": "jane@example.com",
    "mobile": "+61..."
  },
  "preorderFormId": "68909461e93e8196b0fd577a"
}
```

Response
```json
{ "ok": true, "id": "6648f..." }
```

⸻

## Testing Notes
	•	Ensure .env.local is set before calling APIs
	•	If you see ClientSession must be from the same MongoClient:
	•	You have multiple Mongo clients. Use only the singleton dbConnect() and the default mongoose connection for all models and sessions.

⸻

## Roadmap
	•	Payment integration – connect to banking APIs, support automatic reconciliation via webhook callbacks when payer info is received
	•	Text message phone verification – OTP-based verification using Redis as temporary store
	•	Messaging & channel connectors – integrate rich messaging channels (Instagram, KakaoTalk, SMS, WhatsApp Business API) into the Admin UI
	•	AWS service integrations
	-   AWS S3 – multi-tenant product image & asset storage
	-	AWS SES – transactional emails (order confirmations, stock alerts)
	-	AWS SNS – push notifications & SMS fallback
	-	AWS Lambda / API Gateway – serverless extensions & event-driven workflows
	•	Advanced operator management – fine-grained roles, audit logs, and impersonation
	•	Multi-tenant customization – theming, branding, and feature flags configurable per tenant
	•	Progressive Web App (PWA) – offline cart support and installable app experience


## Deployment Artifacts (ECS Task Definitions)

When deploying to ECS Fargate, you may generate task definition files (e.g., td.json, td-current.json, td-preorder.json, td-register.json).

### Recommendation

Do not commit environment-specific task definition files to git (they include ARNs, image URIs, etc.).

Instead, commit a template with placeholders and have CI/CD materialize the real file per environment.

```json
{
  "family": "preorder-taskdef",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "${ECS_EXECUTION_ROLE_ARN}",
  "containerDefinitions": [
    {
      "name": "preorder",
      "image": "${ECR_IMAGE_URI}:${GIT_SHA}",
      "portMappings": [{ "containerPort": 3000, "protocol": "tcp" }],
      "essential": true,
      "environment": []
    }
  ]
}
```

```bash
sed \
  -e "s|\${ECS_EXECUTION_ROLE_ARN}|$ECS_EXECUTION_ROLE_ARN|g" \
  -e "s|\${ECR_IMAGE_URI}|$ECR_IMAGE_URI|g" \
  -e "s|\${GIT_SHA}|$GIT_SHA|g" \
  infra/ecs-taskdef-template.json > dist/td.json

aws ecs register-task-definition --cli-input-json file://dist/td.json
```

```bash
# Generated task definitions
/td*.json
/dist/
```

## Deployment Artifacts

### Option 1 — Container-centric 
1) GitHub Actions + Docker + ECS Fargate + S3 + CloudFront + MongoDB Atlas + Redis + Cognito + API Gateway

| Capability / Integration          | Trigger Style                         | Runs Where                               | Notes (incl. Auth/JWT) |
|----------------------------------|---------------------------------------|------------------------------------------|-------------------------|
| Next.js app (SSR/API)            | HTTPS                                 | ECS Fargate (service)                     | Verify Cognito **JWT** in app middleware or via ALB → App; attach `req.user`. |
| Stock control (reserve/decrement)| In-app call                           | Fargate (service)                         | Use Mongo **transactions**; same service for lowest latency. |
| Banking API — **webhook**        | Webhook → API Gateway                 | API GW → Fargate (or tiny Lambda forwarder)| Prefer API GW + Cognito Authorizer; forward to app. |
| Banking API — **polling/recon**  | Scheduled                             | Fargate scheduled task (EventBridge)      | Use when provider lacks webhooks / needs periodic recon. |
| SMS phone verification (OTP)     | Outbound; receipt webhooks (optional) | Fargate calls SNS/Twilio; OTP in **Redis**| Store OTP with TTL; verify server-side; optional receipt via API GW → Fargate. |
| Open API (3rd-party)             | Webhook or on-demand                  | Fargate                                   | Keep external API calls in app/worker container. |
| Authentication (Cognito)         | Login/refresh                         | Cognito User Pool + app middleware        | JWT in httpOnly cookies; multi-tenant claim check. |



### Option 2 — Hybrid (Fargate for web, Lambda for async)
2) GitHub Actions + Docker + ECS Fargate + AWS Lambda (workers) + API Gateway + EventBridge + SQS

| Capability / Integration           | Trigger Style                         | Runs Where                                         | Notes (incl. Auth/JWT) |
|-----------------------------------|---------------------------------------|----------------------------------------------------|-------------------------|
| Next.js app (SSR/API)             | HTTPS                                 | ECS Fargate (service)                              | Verify Cognito **JWT** in app middleware. |
| Stock control (reserve/decrement) | In-app call                           | Fargate (service)                                  | Keep transactional writes with web tier. |
| Banking API — **webhook**         | Webhook                               | API GW → Lambda (short) → SQS → Fargate worker     | Lambda validates JWT (Authorizer), ack fast; worker does heavy work. |
| Banking API — **polling/recon**   | Scheduled                             | EventBridge → Lambda (≤15m) or Fargate cron        | Use Lambda only if quick; otherwise Fargate. |
| SMS phone verification (OTP)      | Outbound; receipt webhooks (optional) | Lambda (short) + **Redis**                         | Send via SNS/Twilio; OTP TTL in Redis; receipts via API GW → Lambda. |
| Open API (3rd-party)              | Webhook or on-demand                  | Lambda (short) or Fargate worker                   | Choose by duration/CPU. |
| Authentication (Cognito)          | Login/refresh                         | Cognito User Pool + API GW Authorizer / middleware | JWT verified at API GW or app. |


### Option 3 — Serverless-first (Lambda@Edge + API Gateway)
3) GitHub Actions + SST (or CDK) + All-Serverless (Lambda@Edge) + CloudFront + S3 + API Gateway + Cognito

| Capability / Integration           | Trigger Style                         | Runs Where                                   | Notes (incl. Auth/JWT) |
|-----------------------------------|---------------------------------------|----------------------------------------------|-------------------------|
| Next.js app (SSR/ISR)             | HTTPS (global)                        | Lambda@Edge (SST/CDK `NextjsSite`)           | JWT verified in Edge/Lambda handlers; cookies httpOnly. |
| Stock control (reserve/decrement) | API call                              | Lambda (short)                               | Keep < 15m; reuse Mongo client across invocations. |
| Banking API — **webhook**         | Webhook                               | API GW → Lambda (short) → SQS                | Cognito Authorizer on API GW; async consumers drain SQS. |
| Banking API — **polling/recon**   | Scheduled                             | EventBridge → Lambda (short) or Fargate cron | Pick by runtime length. |
| SMS phone verification (OTP)      | Outbound; receipt webhooks (optional) | Lambda (short) + **Redis**                   | OTP TTL; receipt webhook via API GW → Lambda. |
| Open API (3rd-party)              | Webhook or on-demand                  | Lambda (short); spill to Fargate if heavy    | Keep serverless unless duration/CPU requires containers. |
| Authentication (Cognito)          | Login/refresh                         | Cognito User Pool + API GW/Lambda auth       | Use Cognito Authorizer or verify JWKS in code. |


### Deployment Options Comparison

| Capability / Integration           | Option 1 — Container-centric           | Option 2 — Hybrid (Fargate + Lambda)                    | Option 3 — Serverless-first (Lambda@Edge)            |
|-----------------------------------|----------------------------------------|---------------------------------------------------------|------------------------------------------------------|
| **Next.js app (SSR/API)**         | ECS Fargate service                    | ECS Fargate service                                     | Lambda@Edge (SST/CDK `NextjsSite`)                   |
| **Stock control (reserve/decrement)** | Fargate service (in-app call)          | Fargate service (keep transactional writes local)       | Lambda (short) — < 15m txn limit; reuse Mongo client |
| **Banking API — webhook**         | API GW → Fargate (or tiny Lambda forwarder) | API GW → Lambda (short) → SQS → Fargate worker          | API GW → Lambda (short) → SQS                        |
| **Banking API — polling/recon**   | Fargate scheduled task (EventBridge)   | EventBridge → Lambda (≤15m) or Fargate cron             | EventBridge → Lambda (short) or Fargate cron         |
| **SMS phone verification (OTP)**  | Fargate → SNS/Twilio; OTP in Redis     | Lambda (short) + Redis; receipts via API GW → Lambda    | Lambda (short) + Redis; receipts via API GW → Lambda |
| **Open API (3rd-party)**          | Fargate container/worker               | Lambda (short) or Fargate worker (choose by duration)   | Lambda (short); spill to Fargate if heavy workloads  |
| **Authentication (Cognito)**      | Cognito User Pool + app middleware     | Cognito User Pool + API GW Authorizer / middleware      | Cognito User Pool + API GW/Lambda Authorizer         |
| **JWT handling**                  | Verify in app middleware (or ALB)      | Verified at API GW (Authorizer) or app                  | Verified at API GW or Lambda@Edge with JWKS          |
| **Best fit**                      | Long-running workloads, predictable traffic, tight DB coupling | Mixed workloads (sync on Fargate, async/offload via Lambda) | Global edge delivery, bursty traffic, low infra mgmt |


#### Add-on notes (applies to all options)
	•	Secrets: AWS Secrets Manager / SSM Parameter Store.
	•	Observability: CloudWatch Logs, X-Ray; consider OpenTelemetry to Datadog/New Relic.
	•	Images: ECR for containers; S3 for product images; optional CloudFront image optimization or Lambda@Edge transforms.
	•	Phone verification: SMS via SNS or a provider (Twilio), OTP stored in Redis with TTL.
