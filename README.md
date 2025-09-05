# AI-powered eCommerce customer service system - supports multiple merchants

### A Next.js 15 + React 19 application for running multi-tenant group-buy form-based order and campaigns. Includes a Cart/Order drawer UX, MongoDB backend with transaction-safe real-time stock handling, and an LLM (ChatGPT)-enhanced Admin Panel for customer service and operations.

⸻

### Highlights
	•	Multi-tenant routing: per-company and per-form isolation
	•	URLs: https://chatgptform.com/forms/order/:companyName/:formId (customer UI)
	•	Data partitioning via companyName and preorderFormId on all product/order docs
	•	Asset location: S3
	•	LLM Admin Panel (/admin)
	•	ChatGPT-assisted customer support: order lookup, stock checks, status updates, canned responses
	•	MCP-style tool calls (serverless) to read/write domain data (orders, stock, pickups)
	•	Channel intents: Instagram Messenger, KakaoTalk, SMS (extensible)
	•	Drawer UX: Cart page, Product Detail page, and Order flows using a bounded drawer pattern
	•	Realtime Stock signals: Websocket
	•	Serverless: API routes under /api, deployable to ECS on Fargate + CloudFront + S3
	•	Type-safe: Shared types.ts, utility helpers for pricing and cart utility

⸻

## Tech Stack

### Frontend
	•	Next.js 15
	•	React 19
	•	Tailwind CSS
	•	Zustand stores
	•	i18n via t() helper

### Backend
	•	Next.js API Routes following a Model–Controller–Service pattern  
    •	Node.js Express server for WebSocket communication
    •   MongoDB Atlas + Mongoose, Redis
	•	Transactional controllers

### Infra / Integrations
	•	CloudFront, S3, Fargate
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
│   ├── app/                    # Next.js App Router 
│   │   ├── api/
│   │   │   └── orders/[id]/    # API routes
│   │   │       └── route.ts
│   │   └── forms/
│   │       └── order/[companyName]/[formId]/
│   │           └── page.tsx    # entry for customer order form
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
├── package.json
├── tsconfig.json
├── Dockerfile
└── README.md
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
	•	Auto-summaries of long customer threads
	•	MCP-style Tools (serverless):
	•	orders.findOne, orders.update, stock.getById, stock.reserve, pickupDelivery.create …
	•	Tools enforce company + form scoping and validation

### Safety/Guardrails
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

## API Reference 

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



## Deployment Artifacts

### Container-centric 
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

