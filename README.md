# Data Platform - Contract Validator

Central platform for managing data contracts and validating producer changes.

## Purpose
- Owns all data contracts for Kafka topics
- Maps producer repos to their contracts
- Runs automated validation when producers open PRs
- Uses Claude AI to analyze code changes against contracts

## Structure
```
data-platform/
├── contracts/              # Data contract definitions
│   ├── orders-latest.yaml
│   └── line-items.yaml
├── contract-mappings.yaml  # Maps repos to contracts
└── .github/workflows/
    └── validate-contracts.yml  # Validation workflow
```

## How It Works
1. Producer opens PR in their repo
2. Webhook triggers this repo's workflow
3. Workflow uses Claude Code Action to analyze changes
4. Results posted as PR comment

## Setup
See `/NEXT-STEPS.md` in the workspace root for implementation guide.

## GitHub Repo
https://github.com/zhiyuan91/data-platform
