# Meridian Engineering Wiki

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Service Map](#service-map)
3. [Database Architecture](#database-architecture)
4. [Deployment & Infrastructure](#deployment--infrastructure)
5. [Development Setup](#development-setup)
6. [Error Taxonomy](#error-taxonomy)
7. [Known Sharp Edges](#known-sharp-edges)

---

## Architecture Overview

Meridian is a B2B payments infrastructure platform. We process payments on behalf of enterprise customers who embed our API into their own products. Our platform handles payment initiation, routing, reconciliation, and reporting.

**Core architectural principle:** We separate the *transaction path* (high availability, low latency, write-heavy) from the *reporting path* (read-heavy, analytics, can tolerate delay). Every architectural decision should be evaluated against this separation.

**Current scale:** ~120k transactions/day, 340 enterprise customers, operating in 18 countries.

---

## Service Map

```
┌─────────────────────────────────────────────────────┐
│                   API Gateway                        │
│              (Kong, /api/v2/*)                       │
└──────────────────────┬──────────────────────────────┘
                       │
         ┌─────────────┼──────────────┐
         ▼             ▼              ▼
   ┌──────────┐  ┌──────────┐  ┌──────────────┐
   │ Payments │  │  Auth    │  │   Customer   │
   │ Service  │  │ Service  │  │    Portal    │
   └────┬─────┘  └──────────┘  └──────────────┘
        │
        │ MQTT (QoS Level 1)
        ▼
   ┌──────────┐     ┌──────────────┐
   │  Ledger  │────▶│   Billing    │
   │ Service  │     │   Service    │
   └──────────┘     └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Reconciliation│
                    │   Service    │
                    └──────────────┘
```

**Important:** Services communicate via MQTT pub/sub, NOT REST polling. This was changed in February 2021. Any documentation referencing REST between internal services is outdated. See ADR-007 for the full context.

---

## Database Architecture

We operate **two separate Postgres clusters**. This is intentional and must not be collapsed into one.

### meridian-primary
- **Purpose:** All transactional writes and customer-facing reads
- **Connection:** `/meridian/prod/db/primary` in AWS Secrets Manager
- **Rules:** Never run analytical queries here. Table locks during peak hours caused a 4-minute outage in September 2020 (see postmortem in #incidents).

### meridian-analytics
- **Purpose:** Reporting, dashboards, reconciliation reports
- **Connection:** `/meridian/prod/db/analytics` in AWS Secrets Manager
- **Lag:** 15-minute replication lag from primary. Do not use for real-time data.
- **Rule:** If you are writing any query with GROUP BY, window functions, or aggregations over large datasets — it goes here, not primary.

**How to decide which DB to use:**
- Is this query customer-facing or part of a payment flow? → primary
- Is this a report, dashboard, or batch job? → analytics

---

## Deployment & Infrastructure

### AWS Account Structure

We have **three AWS accounts.** This is confusing. Here is what they are:

| Account Name | Purpose |
|---|---|
| `meridian-prod` | Production environment |
| `meridian-staging` | Staging / QA environment |
| `meridian-dev-old` | **Legacy account, NOT a dev environment** |

> ⚠️ **`meridian-dev-old` is not the dev environment.** Despite the name, it is a legacy account from before we had proper IAM structure. It contains some old Lambda functions that are still referenced by two enterprise customers. Do not deploy to it. Do not delete anything from it without checking with the platform team.

### Terraform Structure

Infrastructure is managed via Terraform in `/infra/terraform/`. We have three module types:

- `modules/networking` — VPCs, subnets, security groups
- `modules/compute` — ECS clusters, Lambda functions
- `modules/data` — RDS instances, S3 buckets, ElastiCache

**To deploy a new service:** Follow the runbook in `/infra/docs/new-service-runbook.md`. Do not create AWS resources manually in the console — they won't be tracked and will cause drift.

### Environments

| Environment | Branch | Deploy trigger |
|---|---|---|
| Production | `main` | Manual approval after staging |
| Staging | `staging` | Auto-deploy on merge |
| Local | N/A | Docker Compose (`make dev`) |

---

## Development Setup

### Prerequisites
- Docker Desktop 4.0+
- AWS CLI v2 configured with SSO (`aws sso login --profile meridian-dev`)
- Node.js 18+ (use `nvm` — `.nvmrc` is in the repo root)
- Python 3.11+

### First-time setup

```bash
# Clone the repo
git clone git@github.com:meridian-payments/platform.git
cd platform

# Copy environment variables
cp .env.example .env.local
# You will need to fill in API keys — get these from 1Password vault "Engineering Onboarding"

# Start all services
make dev

# Run tests
make test
```

> ⚠️ **Known issue:** The first `make dev` often fails with a networking error on M1/M2 Macs due to a Docker networking conflict. Run `docker network prune` and try again. This has been a known issue since mid-2022 and is tracked in #backend-platform.

### Running individual services

```bash
# Run just the payments service
make dev service=payments

# Run with hot reload
make dev-watch service=payments
```

---

## Error Taxonomy

Our internal error codes follow this pattern: `[PREFIX]-[CODE]`

| Prefix | Meaning | Retry? |
|---|---|---|
| `CE` | Customer Error — problem is on the customer/cardholder side | No |
| `PE` | Processor Error — Stripe/Adyen/Braintree returned an error | Yes |
| `IE` | Internal Error — bug or infrastructure issue on our side | Page on-call |

**Common codes:**
- `CE-401` — Card declined. Never retry. Never escalate to customer as our error.
- `CE-403` — Insufficient funds.
- `PE-500` — Processor timeout. Retry with 30s fixed delay (max 2 retries).
- `PE-503` — Processor unavailable. Do not retry aggressively — see the retry storm incident.
- `IE-001` — Database connection failure. Auto-pages on-call.

The full enum is the source of truth: `/services/shared/error_codes.py`

> **Note:** There was a legacy Notion doc with the full error taxonomy but that Notion workspace was deprecated in 2020. The code file above is the only up-to-date reference.

---

## Known Sharp Edges

These are things that have burned multiple engineers. Read this before touching any of the relevant code.

### 1. Payment processor clients — always use the factory
Never instantiate `StripeClient()`, `AdyenClient()`, or `BraintreeClient()` directly. Use `processor_factory.py`. The factory handles credential rotation, rate limiting, and processor fallback. Direct instantiation bypasses all of this. Two production incidents have resulted from this mistake.

### 2. MQTT consumers must be idempotent
We use QoS Level 1 (at-least-once delivery). This means a message can be delivered more than once. Every consumer that processes a payment event MUST implement idempotency using the `transaction_id` as a deduplication key. Without this, customers can be double-charged. This is not optional.

### 3. The legacy_reconciliation module is different on purpose
`/services/billing/legacy_reconciliation/` uses different calculation logic than the modern reconciliation module. This is intentional — it serves 6 enterprise customers with contractual SLAs tied to the old calculation method. The code looks wrong but isn't. Do not refactor without talking to the enterprise team.

### 4. JWT vs session cookie auth — there are two systems
External API uses JWTs. Internal dashboard uses session cookies. This is not a bug or inconsistency to fix — it's deliberate. There is an open RFC to unify them but it has no timeline.

### 5. Three payment processors — all active for different reasons
- **Stripe** — primary processor, US and most markets
- **Adyen** — Southeast Asia markets where Stripe has restrictions
- **Braintree** — legacy, kept for oldest enterprise customers with contractual dependencies. Do not remove.

---

*Last updated: March 2023 — some sections may be outdated. When in doubt, ask in #backend-platform or check the relevant Slack thread.*
