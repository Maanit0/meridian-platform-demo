# Meridian Platform — Engineering Repository

## Repository Structure

```
platform/
├── services/
│   ├── payments/           # Payment initiation and processor routing
│   ├── billing/            # Billing, invoicing, reconciliation
│   ├── ledger/             # Transaction ledger and event log
│   ├── auth/               # Authentication service
│   └── shared/             # Shared utilities, error codes, models
├── infra/
│   ├── terraform/          # Infrastructure as code
│   └── docs/               # Runbooks and infrastructure docs
├── api-gateway/            # Kong configuration
├── scripts/                # Developer utility scripts
└── docs/                   # Architecture decision records (ADRs)
```

---

## Architecture Decision Records

ADRs live in `/docs/adr/`. Read these before making significant changes.

| ADR | Title | Status |
|---|---|---|
| ADR-001 | Use Postgres as primary datastore | Accepted |
| ADR-003 | Multi-processor payment routing | Accepted |
| ADR-007 | Migrate internal events to MQTT | Accepted |
| ADR-009 | Separate analytics read replica | Accepted |
| ADR-012 | JWT for external API auth | Accepted |
| ADR-015 | Deprecate Braintree processor | Draft — no timeline |

---

## Services

### payments service
Handles payment initiation, processor selection, and retry logic.

**Key files:**
- `processor_factory.py` — **Always use this to get a processor client**
- `clients/base.py` — Abstract base class for processor clients
- `clients/stripe_client.py` — Stripe integration
- `clients/adyen_client.py` — Adyen integration (Southeast Asia markets)
- `clients/braintree_client.py` — Braintree integration (legacy enterprise)
- `retry.py` — Retry logic with fixed 30s delay, max 2 retries

### billing service
Handles invoicing, subscription billing, and reconciliation.

**Key files:**
- `reconciliation.py` — Modern reconciliation (all customers post-2020)
- `legacy_reconciliation/` — ⚠️ Legacy reconciliation for pre-2020 enterprise customers. Do not modify without enterprise team sign-off.
- `retry.py` — Transaction retry handling

### ledger service
Immutable transaction log. Events published to MQTT on every state change.

### shared
- `error_codes.py` — Source of truth for all internal error codes (CE/PE/IE taxonomy)
- `models.py` — Shared data models
- `idempotency.py` — Idempotency utilities for MQTT consumers

---

## Getting Started

See the [Engineering Wiki](./docs/engineering-wiki.md) for full setup instructions.

Quick start:
```bash
cp .env.example .env.local
make dev
```

---

## Commit History Highlights

These commits contain important context for understanding the codebase:

- `a3f7c2d` (Feb 2021) — "Migrate billing<>ledger communication to MQTT, remove REST polling"
- `b8e1049` (Sep 2020) — "Add analytics read replica, add query routing middleware"
- `c91d3e8` (Jun 2021) — "Add processor factory, deprecate direct client instantiation"
- `d4a2f61` (Dec 2021) — "Change retry logic: exponential backoff -> fixed 30s delay"
- `e7b90ac` (Nov 2022) — "Isolate legacy reconciliation module, add DO NOT MODIFY warning"

---

# /services/payments/processor_factory.py

```python
"""
Payment Processor Factory

IMPORTANT: Always use this factory to get a processor client.
Do NOT instantiate StripeClient, AdyenClient, or BraintreeClient directly.

The factory handles:
- Credential rotation (credentials are rotated every 30 days)
- Per-processor rate limiting
- Automatic fallback if primary processor is unavailable
- Audit logging for processor selection decisions

Two production incidents resulted from bypassing this factory:
- INC-2021-047: Direct StripeClient instantiation bypassed rate limiting,
  caused 429 errors during Black Friday peak
- INC-2022-003: Direct client instantiation used stale credentials after rotation

See ADR-003 for the multi-processor routing rationale.
"""

from typing import Optional
from services.shared.error_codes import ProcessorError
from services.payments.clients.stripe_client import StripeClient
from services.payments.clients.adyen_client import AdyenClient
from services.payments.clients.braintree_client import BraintreeClient
from services.shared.models import Transaction, CustomerAccount


PROCESSOR_ROUTING_RULES = {
    # Southeast Asia country codes — Stripe has regulatory restrictions here
    # Adyen has full coverage. Last verified: Q3 2022.
    "SE_ASIA_MARKETS": ["SG", "MY", "ID", "TH", "VN", "PH"],

    # Legacy enterprise accounts on Braintree — contractual dependency
    # These account IDs are hardcoded because we cannot change them without
    # renegotiating contracts. Project to migrate: see Jira PLAT-891 (no ETA).
    "BRAINTREE_LEGACY_ACCOUNTS": [
        "acc_legcy_0012", "acc_legcy_0047", "acc_legcy_0088",
        "acc_legcy_0103", "acc_legcy_0156", "acc_legcy_0201"
    ]
}


def get_processor(transaction: Transaction, account: CustomerAccount):
    """
    Returns the appropriate processor client for a given transaction.
    Applies routing rules, initializes with current credentials,
    and wraps with rate limiting.
    """
    processor_name = _select_processor(transaction, account)
    credentials = _get_current_credentials(processor_name)
    client = _initialize_client(processor_name, credentials)
    return _wrap_with_rate_limiter(client, processor_name)


def _select_processor(transaction: Transaction, account: CustomerAccount) -> str:
    # Legacy enterprise accounts always route to Braintree
    if account.account_id in PROCESSOR_ROUTING_RULES["BRAINTREE_LEGACY_ACCOUNTS"]:
        return "braintree"

    # Southeast Asia markets route to Adyen
    if transaction.destination_country in PROCESSOR_ROUTING_RULES["SE_ASIA_MARKETS"]:
        return "adyen"

    # Default to Stripe
    return "stripe"
```

---

# /services/payments/retry.py

```python
"""
Transaction Retry Logic

Current behavior: Fixed 30-second delay, maximum 2 retries.

WHY FIXED DELAY (not exponential backoff):
In December 2021 we changed FROM exponential backoff TO fixed delay.
The reason: during a Stripe outage in November 2021, all failed transactions
accumulated and then retried simultaneously when Stripe came back online.
This "retry storm" overloaded our system and extended the outage by ~8 minutes.
Fixed 30-second delay spreads retries more evenly.

See: INC-2021-089 postmortem, Slack #incidents 2021-11-15

WHY MAX 2 RETRIES (not 3):
Originally 3 retries. Reduced to 2 after analysis showed the 3rd retry
had <2% success rate but added significant tail latency for customers.

IMPORTANT: CE (Customer Error) codes are NEVER retried. A declined card
will not succeed on retry. Retrying wastes time and signals to the customer
that something is wrong on our end when it isn't.
"""

import time
from services.shared.error_codes import ErrorCode, ErrorPrefix


RETRY_DELAY_SECONDS = 30
MAX_RETRIES = 2


def should_retry(error_code: ErrorCode) -> bool:
    """
    Determines if a failed transaction should be retried based on error type.

    CE (Customer Error): Never retry — deterministic failure
    PE (Processor Error): Retry — transient, processor-side issue
    IE (Internal Error): Do not retry from here — page on-call instead
    """
    if error_code.prefix == ErrorPrefix.CE:
        return False
    if error_code.prefix == ErrorPrefix.PE:
        return True
    if error_code.prefix == ErrorPrefix.IE:
        # Don't retry internal errors — they indicate a bug.
        # The on-call engineer needs to investigate.
        _page_oncall(error_code)
        return False
    return False


def retry_transaction(transaction_id: str, attempt: int = 0):
    if attempt >= MAX_RETRIES:
        return {"status": "failed", "reason": "max_retries_exceeded"}

    time.sleep(RETRY_DELAY_SECONDS)
    # ... retry logic
```

---

# /services/shared/error_codes.py

```python
"""
Meridian Internal Error Taxonomy

Error codes follow the format: [PREFIX]-[CODE]

Prefixes:
  CE = Customer Error. Problem is on the customer/cardholder side.
       DO NOT retry. DO NOT surface as "our fault" to the customer.

  PE = Processor Error. Stripe, Adyen, or Braintree returned an error.
       CAN retry with fixed delay. See retry.py for logic.

  IE = Internal Error. Bug or infrastructure failure on our side.
       DO NOT retry. Page on-call immediately.

History:
  This taxonomy was established in 2019. There was a Notion doc with full
  descriptions but that workspace was deprecated. This file is now the
  source of truth. If you add a new error code, add it here with a comment.
"""

from enum import Enum


class ErrorPrefix(str, Enum):
    CE = "CE"  # Customer Error
    PE = "PE"  # Processor Error
    IE = "IE"  # Internal Error


class ErrorCode:
    # --- Customer Errors (CE) ---
    CARD_DECLINED = "CE-401"           # Generic decline. Most common. Never retry.
    INSUFFICIENT_FUNDS = "CE-403"      # Card has insufficient funds.
    CARD_EXPIRED = "CE-405"            # Card expiry date has passed.
    INVALID_CVV = "CE-410"             # CVV mismatch.
    CARD_NOT_SUPPORTED = "CE-415"      # Card type not supported in this market.

    # --- Processor Errors (PE) ---
    PROCESSOR_TIMEOUT = "PE-500"       # Processor took too long. Retry with fixed delay.
    PROCESSOR_UNAVAILABLE = "PE-503"   # Processor is down. Retry carefully — see retry storm incident.
    RATE_LIMITED = "PE-429"            # We've exceeded processor rate limit. Back off.

    # --- Internal Errors (IE) ---
    DB_CONNECTION_FAILURE = "IE-001"   # Database unreachable. Auto-pages on-call.
    LEDGER_SYNC_FAILURE = "IE-010"     # MQTT publish to ledger failed.
    PROCESSOR_FACTORY_ERROR = "IE-020" # Could not initialize processor client.
```

---

# /infra/terraform/modules/compute/main.tf (excerpt)

```hcl
# ECS Cluster — Production
# Note: We use ECS (not EKS/Kubernetes) for compute.
# This was a deliberate decision made in 2020 when the team was 8 engineers.
# Kubernetes operational overhead was considered too high for team size.
# As of 2023, team is 120 engineers and there is active discussion about
# migrating to EKS. See RFC-2023-004 in Confluence (if you have access).
# Until that RFC is decided, all new services should deploy to ECS.

resource "aws_ecs_cluster" "meridian_prod" {
  name = "meridian-prod"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    # DO NOT manually modify ECS resources in the console.
    # All changes must go through Terraform or they will be overwritten.
  }
}

# AWS Accounts:
# meridian-prod    = production (this account)
# meridian-staging = staging
# meridian-dev-old = LEGACY ACCOUNT. Not a dev environment.
#                    Contains legacy Lambdas for 2 enterprise customers.
#                    Do not deploy here. Do not delete resources here.
#                    Contact marcus.chen@ before touching anything in this account.
```
