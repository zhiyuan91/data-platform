# Data Platform - Contract Validator

Central platform for managing data contracts and validating producer changes using AI-powered semantic analysis.

## Architecture Overview

This platform implements **contract-first data governance** for Kafka-based data pipelines. Instead of schema registries that only validate syntax, this system uses Claude AI to perform semantic validation of producer code against data contracts.

### Key Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        Producer Repo                             │
│                  (checkout-service)                              │
│                                                                   │
│  Developer opens PR → GitHub webhook triggered                   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ repository_dispatch event
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Data Platform Repo                            │
│                   (this repository)                              │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  GitHub Actions Workflow                                │    │
│  │  (.github/workflows/validate-contracts.yml)             │    │
│  │                                                          │    │
│  │  1. Parse repo info from webhook payload                │    │
│  │  2. Generate GitHub App token (cross-repo access)       │    │
│  │  3. Checkout platform repo (contracts)                  │    │
│  │  4. Checkout producer repo (PR branch)                  │    │
│  │  5. Post "Validating..." comment to PR                  │    │
│  │  6. Run Claude Code Action ────────────┐                │    │
│  │  7. Extract validation results         │                │    │
│  │  8. Update PR comment with results     │                │    │
│  └────────────────────────────────────────┼────────────────┘    │
│                                            │                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Data Contracts (contracts/)                             │   │
│  │  - orders.yaml                                           │   │
│  │  - line-items.yaml                                       │   │
│  │                                                           │   │
│  │  Defines schema, quality rules, enum values, etc.        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                            │                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Contract Mappings (contract-mappings.yaml)              │   │
│  │                                                           │   │
│  │  Maps producer repos to their contracts:                 │   │
│  │  - vibe-coding-in-action/checkout-service → orders       │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ AI Analysis
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Claude Code Action                             │
│               (anthropics/claude-code-action@v1)                 │
│                                                                   │
│  Semantic Analysis:                                              │
│  ✓ Read contract YAML                                            │
│  ✓ Analyze Java/Kotlin producer code                            │
│  ✓ Check for:                                                    │
│    - P0 (BREAKING): Enum drift, field rename, type changes,     │
│                     unit mismatches, business rule violations    │
│    - P1 (WARNING): Default/null changes                         │
│    - P2 (WARNING): PII leakage                                  │
│  ✓ Generate formatted validation report                         │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ Results
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      PR Comment                                  │
│                                                                   │
│  🚨 BREAKING / ⚠️ WARNING / ✅ PASS                              │
│                                                                   │
│  Lists critical issues with:                                     │
│  - Field name                                                    │
│  - Problem description                                           │
│  - Code location                                                 │
│  - Suggested fix                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Repository Structure

```
data-platform/
├── contracts/                    # Data contract definitions
│   ├── orders.yaml              # Order events contract
│   └── line-items.yaml          # Line item events contract
├── contract-mappings.yaml        # Maps producer repos to contracts
├── .github/workflows/
│   └── validate-contracts.yml   # Main validation workflow
└── README.md                     # This file
```

## How Validation Works

### 1. Webhook Setup
- GitHub App installed on producer repos
- Webhook configured to send `pull_request` events to webhook handler
- Webhook handler triggers `repository_dispatch` to platform repo

### 2. Cross-Repo Access
- Workflow generates GitHub App token scoped to producer repo
- Uses token to checkout PR branch from producer
- Allows reading code without giving broad permissions

### 3. AI-Powered Analysis
Claude Code Action performs semantic validation:
- **Understands business logic**: Not just schema matching
- **Detects semantic issues**: Unit mismatches (seconds vs milliseconds)
- **Context-aware**: Understands camelCase ↔ snake_case mapping
- **Explains reasoning**: Provides actionable fix suggestions

### 4. Validation Categories

**P0 - BREAKING (🚨)**
- Enum drift: Adding values not in contract
- Field removal/rename: Missing required fields
- Type changes: String → Long, Long → String
- Unit mismatches: Seconds vs milliseconds, cents vs dollars
- Business rule violations: Negative quantities, out-of-range values

**P1 - WARNING (⚠️)**
- Default/null changes that don't break existing consumers

**P2 - WARNING (⚠️)**
- PII leakage: Sensitive data in metadata fields

## Data Contract Specification

Contracts use [Data Contract Specification](https://datacontract.com/) format:

```yaml
dataContractSpecification: 1.0.0
id: orders
info:
  title: Orders
  version: 1.0.0
  owner: Data Platform Team

servers:
  production:
    type: kafka
    topic: orders
    format: json

models:
  orders:
    fields:
      order_id:
        type: text
        required: true
        unique: true

      order_status:
        type: text
        required: true
        enum: [PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED]
        enum_evolution: strict  # No new values without contract update

      order_total:
        type: long
        description: Total order amount in cents
        required: true
        quality:
          - type: range
            min: 1000
            max: 49900
```

## Example Validation Results

### Breaking Change Detected
```markdown
## 🔍 Contract Validation Results

**Status**: 🚨 BREAKING

### Critical Issues

#### 🔴 Field Rename
- **Field**: `order_total`
- **Problem**: Field renamed from `order_total` to `amount`
- **Location**: OrderProducer.java:62, :130
- **Fix**: Change field name back to `orderTotal` or update contract first

---
**Contracts checked**: orders
```

### Pass
```markdown
## 🔍 Contract Validation Results

**Status**: ✅ PASS

No breaking changes detected.

---
**Contracts checked**: orders
```

## Why AI-Powered Validation?

Traditional schema registries (Confluent Schema Registry, AWS Glue) only validate JSON/Avro schema syntax. They miss:

- **Semantic bugs**: Sending seconds instead of milliseconds
- **Business rule violations**: Negative quantities, invalid enums
- **Field naming issues**: `amount` vs `order_total` (both valid schemas)
- **Unit mismatches**: Cents vs dollars
- **PII leakage**: Sensitive data in wrong fields

Claude analyzes the actual producer code logic to catch these issues before deployment.

## Setup Requirements

### GitHub App
- Read repository contents
- Read/write pull requests (comments)
- Webhook for pull request events

### Secrets
- `ANTHROPIC_API_KEY`: Claude API key
- `ANTHROPIC_BASE_URL`: API endpoint (optional)
- `GH_APP_ID`: GitHub App ID
- `GH_APP_PRIVATE_KEY`: GitHub App private key

### Producer Repo Setup
- Install GitHub App
- Configure webhook to point to webhook handler
- Webhook handler triggers platform repo validation

## Related Repositories

- **Producer**: [checkout-service](https://github.com/vibe-coding-in-action/checkout-service)
- **Webhook Handler**: [contract-webhook-handler](https://github.com/zhiyuan91/contract-webhook-handler)
- **Platform**: [data-platform](https://github.com/zhiyuan91/data-platform) (this repo)

## Benefits

✅ **Catch breaking changes before merge**
✅ **Semantic validation, not just syntax**
✅ **Automated PR feedback in seconds**
✅ **No manual contract reviews needed**
✅ **Enforces data governance at scale**
