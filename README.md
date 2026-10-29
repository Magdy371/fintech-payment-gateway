# FinTech Payment Orchestration & Sub-Merchant Gateway

**Production-Grade Modular Monolith Payment Gateway built with [NestJS](https://docs.nestjs.com/) (Express), [Prisma ORM](https://www.prisma.io/docs/orm), [Stripe Sandbox](https://docs.stripe.com/testing), [Neon Serverless PostgreSQL](https://selfhost.dev/blog/neon-pricing-cost-of-serverless-postgres/), and [Upstash Serverless Redis](https://www.13labs.au/compare/upstash-vs-redis-cloud).**

## Overview

This project implements a **Payment Orchestration & Sub-Merchant Aggregator Gateway** (modeled after Stripe Connect and Paymob) engineered to run **100% on free-tier cloud resources** while enforcing strict financial engineering invariants:

* **Double-Entry Immutable Accounting Ledger:** Every payment capture, platform fee split, partial/full refund, and merchant payout writes balanced `DEBIT` and `CREDIT` journal lines (`SUM(DEBIT) == SUM(CREDIT)`) inside [Prisma interactive ACID transactions](https://www.prisma.io/docs/orm/fundamentals/transactions).

* **Zero Double-Charging (Strict Idempotency):** Combines a fast distributed lock in [Upstash Redis](https://www.13labs.au/compare/upstash-vs-redis-cloud) (`SET NX EX 60`) with a durable `IdempotencyRecord` table in PostgreSQL and SHA-256 request payload fingerprinting.

* **Deadlock-Free Pessimistic Concurrency Control:** Acquires PostgreSQL `SELECT ... FOR UPDATE` locks on `PaymentIntent` and `LedgerAccount` rows in deterministic lexicographical UUID order (`ORDER BY id ASC`) to prevent race conditions during concurrent webhooks or partial refunds.

* **Zero Floating-Point Drift:** Stores all monetary balances and journal entries as `BigInt` in **minor currency units** (e.g., cents/piastres: `$100.00 = 10000`, `100.00 EGP = 10000`).

* **PCI-DSS SAQ-A Compliance:** Raw card numbers (PAN) never touch the backend. Frontend clients confirm payments directly against [Stripe Elements](https://docs.stripe.com/payments/payment-intents) using the returned `clientSecret`.

* **Two-Way Cryptographic Webhook Engine:** Verifies inbound `Stripe-Signature` headers against raw Express request buffers (`req.rawBody`) and dispatches signed outbound merchant webhooks (`X-Gateway-Signature`) via [NestJS BullMQ](https://docs.nestjs.com/application/queues) with 5x exponential backoff and dead-letter queueing.

## 100% Free-Tier Technology Stack

| 

| **Layer** | **Technology / Service** | **Free-Tier Allowance** | **Architectural Responsibility** | 
| **HTTP & Core Framework** | [NestJS](https://docs.nestjs.com/) (`@nestjs/platform-express`) | Open-Source (MIT) | Modular monolith DI container, guards, interceptors, and [ValidationPipe](https://docs.nestjs.com/pipes) | 
| **Upstream Processor** | [Stripe Test Mode / Sandbox](https://docs.stripe.com/testing) | Unlimited test charges & [Webhooks](https://docs.stripe.com/webhooks) | Card tokenization, 3D Secure 2 authentication, and upstream settlement simulation | 
| **Database & ORM** | [Prisma ORM](https://www.prisma.io/docs/orm) + [Neon Postgres](https://selfhost.dev/blog/neon-pricing-cost-of-serverless-postgres/) | 0.5 GB storage (\~140k+ txns), 100 CU-hrs/mo | Relational storage, interactive ACID transactions, and row-level locking | 
| **Locks, Rate Limits & Queues** | [Upstash Serverless Redis](https://www.13labs.au/compare/upstash-vs-redis-cloud) + [BullMQ](https://docs.bullmq.io/) | 256 MB RAM, 500,000 commands/mo | Idempotency locks, sliding-window rate limiting, and outbound webhook workers | 
| **Cloud Compute** | [Render Free Web Service](https://render.com/articles/best-netlify-alternatives-in-2026) | 512 MB RAM | Runs the NestJS API server and embedded BullMQ webhook processor | 

## System Architecture

```
+-----------------------------------------------------------------------------------+
|                                CLIENT & MERCHANT LAYER                            |
|                                                                                   |
|  +------------------------+                         +--------------------------+  |
|  | Customer Checkout UI   |                         |   Merchant Backend       |  |
|  | (Stripe Elements SDK)  |                         | (Initiates Intents &     |  |
|  +-----------+------------+                         |  Listens to Webhooks)    |  |
+--------------|--------------------------------------+------------+-------------+--+
               | 3. Confirm Card (PAN)                             | 1. POST /v1/payment-intents
               |    via clientSecret                               |    (Idempotency-Key + API Key)
               v                                                   v
+--------------+-------------+                        +------------+----------------------------------+
|   STRIPE SANDBOX (FREE)    |                        |      NESTJS PAYMENT GATEWAY (EXPRESS)         |
|                            | 2. Create Stripe PI    |                                               |
|  - Tokenizes PAN (SAQ-A)   |<-----------------------+  +-----------------------------------------+  |
|  - Runs 3DS2 Auth          |                        |  | API Security & Reliability Layer        |  |
|  - Emits Signed Webhooks   +----------------------->+  | - ApiKeyGuard (SHA-256 lookup)          |  |
+----------------------------+ 4. POST /webhooks/stripe  | - RateLimitGuard (Upstash Sliding Win)  |  |
                                  (HMAC-SHA256 Raw)   |  | - IdempotencyInterceptor (Redis SET NX) |  |
                                                      |  +--------------------+--------------------+  |
                                                      |                       |                       |
                                                      |  +--------------------v--------------------+  |
                                                      |  | Core FinTech Domain Modules             |  |
                                                      |  | - PaymentsModule  - LedgerModule        |  |
                                                      |  | - RefundsModule   - PayoutsModule       |  |
                                                      |  | - ProviderAdapter (Stripe Strategy)     |  |
                                                      |  +---------+---------------------+---------+  |
                                                      |            |                     |            |
                                                      |  +---------v---------+ +---------v---------+  |
                                                      |  | PrismaService     | | BullMQ Worker     |  |
                                                      |  | ($transaction +   | | - Outbound HMAC   |  |
                                                      |  |  FOR UPDATE lock) | | - 5x Exp Backoff  |  |
                                                      |  +---------+---------+ +---------+---------+  |
                                                      +------------|---------------------|------------+
                                                                   |                     | 6. Signed Webhook
                                                                   v                     |    Delivery
                                                      +------------+---------+           |
                                                      | NEON SERVERLESS PG   |           |
                                                      | (0.5 GB Free Tier)   |           |
                                                      | - LedgerEntry (ACID) |           |
                                                      | - PaymentIntent      |           |
                                                      +----------------------+           |
                                                                                         |
                                                      +----------------------+           |
                                                      | UPSTASH REDIS (FREE) |           |
                                                      | (500k Cmds/Mo)       |<----------+
                                                      | - Idempotency Locks  |  5. Enqueue Job
                                                      | - BullMQ Job State   |
                                                      +----------------------+

```

## Project Directory Structure

```
fintech-payment-gateway/
├── prisma/
│   ├── schema.prisma                            # 10 FinTech models, enums, and composite indexes
│   └── seed.ts                                  # Seeds system accounts, demo merchant & sk_test_ API key
├── src/
│   ├── main.ts                                  # Express bootstrap with rawBody: true, Helmet, & Swagger
│   ├── app.module.ts                            # Root module wiring Config, Prisma, BullMQ, and Domain Modules
│   ├── common/
│   │   ├── constants/
│   │   │   └── injection-tokens.ts              # DI tokens: PAYMENT_PROVIDER_ADAPTER, REDIS_CLIENT
│   │   ├── decorators/
│   │   │   ├── current-merchant.decorator.ts    # @CurrentMerchant() param decorator
│   │   │   └── idempotent.decorator.ts          # @Idempotent() endpoint decorator
│   │   ├── guards/
│   │   │   ├── api-key.guard.ts                 # Validates Bearer sk_test_... via SHA-256 hash lookup
│   │   │   ├── rate-limit.guard.ts              # Upstash Redis sliding-window rate limiter
│   │   │   └── stripe-signature.guard.ts        # Verifies Stripe-Signature against req.rawBody
│   │   ├── interceptors/
│   │   │   ├── bigint-serializer.interceptor.ts # Safely serializes Prisma BigInt fields to JSON
│   │   │   └── idempotency.interceptor.ts       # Redis SET NX lock + Postgres IdempotencyRecord cache
│   │   └── filters/
│   │       └── all-exceptions.filter.ts         # Standardized RFC 7807 error formatting
│   ├── infra/
│   │   ├── prisma/
│   │   │   ├── prisma.module.ts                 # @Global() Prisma database module
│   │   │   └── prisma.service.ts                # PrismaClient lifecycle & transaction wrapper
│   │   └── redis/
│   │       └── redis.module.ts                  # @Global() IORedis / Upstash connection provider
│   └── modules/
│       ├── merchants/                           # Merchant onboarding & API key rotation
│       ├── providers/                           # Strategy Pattern: IPaymentProviderAdapter & StripeAdapter
│       ├── ledger/                              # Double-entry bookkeeper (DEBIT == CREDIT invariant)
│       ├── payments/                            # PaymentIntent state machine & fee calculation
│       ├── refunds/                             # Pessimistic-locked full & partial refund engine
│       ├── payouts/                             # Merchant balance settlement simulation
│       └── webhooks/                            # Inbound Stripe handler & BullMQ outbound dispatcher
├── .env.example
├── package.json
└── README.md

```

## Getting Started

### 1. Prerequisites

* **Node.js** `>= 20.x` and **npm** `>= 10.x`

* A free [**Neon Serverless PostgreSQL**](https://selfhost.dev/blog/neon-pricing-cost-of-serverless-postgres/) database (`DATABASE_URL` and `DIRECT_DATABASE_URL`)

* A free [**Upstash Serverless Redis**](https://www.13labs.au/compare/upstash-vs-redis-cloud) instance (`REDIS_URL`)

* A free [**Stripe Developer Sandbox**](https://docs.stripe.com/testing) account (`STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET`)

### 2. Installation & Scaffolding

```
# Clone the repository (or scaffold with NestJS CLI)
git clone <your-repo-url> fintech-payment-gateway
cd fintech-payment-gateway

# Install dependencies
npm install

```

### 3. Environment Configuration (`.env`)

Create a `.env` file in the root directory:

```
PORT=3000
NODE_ENV=development

# Neon Serverless PostgreSQL (Free Tier: 0.5 GB)
DATABASE_URL="postgresql://user:password@ep-xyz-pooler.us-east-2.aws.neon.tech/neondb?sslmode=require&pgbouncer=true"
DIRECT_DATABASE_URL="postgresql://user:password@ep-xyz.us-east-2.aws.neon.tech/neondb?sslmode=require"

# Upstash Serverless Redis (Free Tier: 500,000 commands/month)
REDIS_URL="rediss://default:your_upstash_password@your-endpoint.upstash.io:6379"

# Stripe Sandbox Keys (100% Free Test Mode)
STRIPE_SECRET_KEY="sk_test_51..."
STRIPE_WEBHOOK_SECRET="whsec_..."

```

### 4. Database Migration & Seeding

Run the Prisma migration and seed script to provision the system-wide `PROVIDER_CLEARING` and `PLATFORM_REVENUE` accounts (for `USD` and `EGP`), a demo merchant, and a test `sk_test_...` API key:

```
# Generate TypeScript Prisma Client
npm run prisma:generate

# Apply database schema migrations to Neon Postgres
npm run prisma:migrate:dev -- --name init_payment_gateway

# Seed system ledger accounts, demo merchant, and test API key
npm run prisma:seed

```

> **Important:** Copy the `sk_test_...` Secret API Key printed in your terminal after running `npm run prisma:seed`. Plaintext keys are never stored in the database (only their SHA-256 hashes).

### 5. Run the Server

```
# Development mode with hot-reload
npm run start:dev

# Production build
npm run build
npm run start:prod

```

* **API Base URL:** `http://localhost:3000`

* **Interactive Swagger Documentation:** `http://localhost:3000/docs`

## Core Accounting & Double-Entry Ledger Rules

When a `$100.00` (`10000` minor units) payment succeeds for a merchant on the default `2.90% + $0.30` (`320` minor units) fee tier, `LedgerService.postJournalEntries()` writes three immutable lines inside a single `transactionGroup`:

| **Ledger Account** | **Account Classification** | **Entry Direction** | **Amount (minor)** | **Balance Impact** | 
| `PROVIDER_CLEARING` | Asset (Receivable) | `DEBIT` | `10000` (\$100.00) | Increases clearing asset (`+10000`) | 
| `MERCHANT_AVAILABLE` | Liability (Payable) | `CREDIT` | `9680` (\$96.80) | Increases merchant balance (`+9680`) | 
| `PLATFORM_REVENUE` | Revenue (Fee Income) | `CREDIT` | `320` (\$3.20) | Increases gateway revenue (`+320`) | 

**Accounting Invariant:** `Total Debits (10000) === Total Credits (9680 + 320 = 10000)`.

## REST API Reference & cURL Examples

### Endpoint Summary

| **Method** | **Endpoint** | **Auth Guard** | **Idempotent** | **Description** | 
| `POST` | `/v1/auth/merchants` | Public | No | Register a merchant and provision `USD`/`EGP` ledger accounts | 
| `POST` | `/v1/api-keys` | Merchant JWT | Yes | Generate a new `sk_test_...` secret API key | 
| `POST` | `/v1/payment-intents` | `ApiKeyGuard` | **Yes** | Create an internal and upstream Stripe `PaymentIntent` | 
| `GET` | `/v1/payment-intents/:id` | `ApiKeyGuard` | No | Inspect payment intent status and fee breakdown | 
| `POST` | `/v1/payment-intents/:id/confirm` | `ApiKeyGuard` | **Yes** | Server-side sandbox confirmation using test tokens (`pm_card_visa`) | 
| `POST` | `/v1/refunds` | `ApiKeyGuard` | **Yes** | Execute a full or partial refund with `FOR UPDATE` row locking | 
| `GET` | `/v1/balances` | `ApiKeyGuard` | No | Query merchant `MERCHANT_AVAILABLE` and `MERCHANT_PENDING` balances | 
| `POST` | `/v1/payouts` | `ApiKeyGuard` | **Yes** | Simulate a bank settlement payout from `MERCHANT_AVAILABLE` | 
| `POST` | `/v1/webhooks/stripe` | `StripeSigGuard` | **Yes** (`event.id`) | Ingest signed Stripe webhook events using `req.rawBody` | 

### 1. Create a Payment Intent

```
curl -X POST http://localhost:3000/v1/payment-intents \
  -H "Authorization: Bearer sk_test_YOUR_SEEDED_KEY" \
  -H "Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000" \
  -H "Content-Type: application/json" \
  -d '{
    "amountMinor": 10000,
    "currency": "USD",
    "customerEmail": "customer@example.com",
    "description": "Order #ORD-9921",
    "metadata": { "orderId": "ORD-9921" }
  }'

```

### 2. Confirm Payment in Sandbox (Using Official [Stripe Test Tokens](https://docs.stripe.com/testing))

* `pm_card_visa` -> Succeeds immediately

* `pm_card_chargeDeclined` -> Simulates issuer decline

* `pm_card_authenticationRequired` -> Simulates 3D Secure 2 challenge

```
curl -X POST http://localhost:3000/v1/payment-intents/<PAYMENT_INTENT_ID>/confirm \
  -H "Authorization: Bearer sk_test_YOUR_SEEDED_KEY" \
  -H "Idempotency-Key: 661f9511-f30c-52e5-b827-557766551111" \
  -H "Content-Type: application/json" \
  -d '{
    "paymentMethodToken": "pm_card_visa"
  }'

```

### 3. Issue a Partial Refund

```
curl -X POST http://localhost:3000/v1/refunds \
  -H "Authorization: Bearer sk_test_YOUR_SEEDED_KEY" \
  -H "Idempotency-Key: 772a0622-a41d-63f6-c938-668877662222" \
  -H "Content-Type: application/json" \
  -d '{
    "paymentIntentId": "<PAYMENT_INTENT_ID>",
    "amountMinor": 2500,
    "reason": "requested_by_customer"
  }'

```

## Local Webhook Testing with Stripe CLI

To test end-to-end settlement and ledger posting locally without deploying to a public URL:

1. Install the official [Stripe CLI](https://www.google.com/search?q=https://docs.stripe.com/webhooks%23test-webhook) and authenticate with your sandbox:

   ```
   stripe login
   
   ```

2. Forward Stripe events to your local NestJS Express server:

   ```
   stripe listen --forward-to localhost:3000/v1/webhooks/stripe
   
   ```

3. Copy the `whsec_...` signing secret output by `stripe listen` into your `.env` as `STRIPE_WEBHOOK_SECRET`.

4. Trigger a simulated payment event in another terminal:

   ```
   stripe trigger payment_intent.succeeded
   
   ```

## Verifying Merchant Outbound Webhooks

When your gateway dispatches an outbound webhook to a merchant's `webhookUrl`, it signs `${timestamp}.${rawBody}` using the merchant's `webhookSecret`. Merchants can verify the `X-Gateway-Signature` header in Node.js with:

```
import { createHmac, timingSafeEqual } from 'crypto';

export function verifyGatewayWebhook(
  rawBody: string,
  signatureHeader: string,
  webhookSecret: string,
): boolean {
  const parts = Object.fromEntries(
    signatureHeader.split(',').map((part) => part.split('=')),
  );
  const timestamp = parts['t'];
  const receivedSig = parts['v1'];
  if (!timestamp || !receivedSig) return false;

  const expectedSig = createHmac('sha256', webhookSecret)
    .update(`${timestamp}.${rawBody}`)
    .digest('hex');

  return timingSafeEqual(Buffer.from(receivedSig), Buffer.from(expectedSig));
}

```

## License

Released under the **MIT License**.
