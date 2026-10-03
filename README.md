# TradeCore Project 2 - Cloud-Native Backend Infrastructure

A production-grade, cloud-native backend infrastructure deployed on AWS using Terraform, featuring containerized microservices, managed PostgreSQL, and CI/CD automation via GitHub Actions with OIDC authentication.

> [!IMPORTANT]
> **The AWS environment described here was fully torn down on 2026-10-03 — no resources
> remain.** This repository is a working template: use the
> [Deployment Guide](#deployment-guide) to rebuild it from scratch, and see
> **[TEARDOWN.md](TEARDOWN.md)** for the destroy runbook, the resource inventory as it
> last existed, and every gotcha hit while tearing it down.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Infrastructure Modules](#infrastructure-modules)
- [Subnet Layout](#subnet-layout)
- [CI/CD Pipeline](#cicd-pipeline)
- [Security](#security)
- [Cost Model](#cost-model)
- [Getting Started](#getting-started)
- [Deployment Guide](#deployment-guide)
- [Teardown Record](TEARDOWN.md)
- [Key Design Decisions](#key-design-decisions)
- [Project Structure](#project-structure)
- [Environment Variables](#environment-variables)
- [GitHub Secrets](#github-secrets)

---

## Architecture

```
                        ┌─────────────────────────────────────┐
                        │            Internet                 │
                        └──────────────┬──────────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────────────┐
                        │     ALB (Application Load Balancer)  │
                        │     Public Subnets - :80 → 301, :443 │
                        └──────────────┬───────────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────────────┐
                        │     ECS Fargate (Container)          │
                        │     Public Subnets - Public IPs      │
                        │     256 CPU / 512 MiB                │
                        └──────────────┬───────────────────────┘
                                       │
                          ┌────────────┴────────────┐
                          │                         │
                          ▼                         ▼
            ┌─────────────────────┐   ┌──────────────────────────┐
            │  RDS PostgreSQL 15  │   │  Secrets Manager          │
            │  Private Subnets    │   │  DB creds, JWT secret     │
            │  db.t3.micro        │   │  5 secrets                │
            └─────────────────────┘   └──────────────────────────┘
```

### Supporting Services

| Service | Purpose |
|---------|---------|
| **Amazon Cognito** | User pool provisioned, standby — app authenticates with backend JWT (see Tech Stack) |
| **AWS Amplify** | Frontend hosting (us-east-1), **console-managed** — Terraform tracks app ID/domain only |
| **Amazon ECR** | Container image registry (SHA-tagged images) |
| **Amazon CloudWatch** | Logs, 5 alarms + SNS, VPC flow logs |
| **AWS CloudTrail** | Multi-region audit trail (CMK-encrypted) to S3 log bucket |
| **Terraform State** | S3 backend with S3 native locking |
| **GitHub Actions** | Infra CI/CD + daily drift detection, OIDC (no static credentials) |
| **AWS IAM** | OIDC provider + least-privilege deployment role |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| **IaC** | Terraform ~> 1.16 |
| **Cloud Provider** | AWS (af-south-1 primary, us-east-1 for Amplify) |
| **Compute** | Amazon ECS Fargate (serverless containers) |
| **Database** | Amazon RDS PostgreSQL 15 |
| **Auth** | Backend JWT (HS256, issuer/audience-pinned); Cognito pool on standby |
| **Frontend** | AWS Amplify (console-managed) |
| **CI/CD** | GitHub Actions with OIDC |
| **State Management** | S3 + DynamoDB |
| **Secrets** | AWS Secrets Manager |

---

## Infrastructure Modules

| Module | Directory | Resources | Status |
|--------|-----------|-----------|--------|
| **Networking** | `Networking/` | VPC, Internet Gateway, 2 public subnets, 2 private subnets, route tables | Complete |
| **ALB** | `Alb/` | Application Load Balancer, security group, target group, HTTP/HTTPS listeners | Complete |
| **ECS** | `Ecs/` | ECS cluster, Fargate service, task definition, IAM roles, CloudWatch logs | Complete |
| **RDS** | `Database/` | RDS PostgreSQL 15, security group, DB subnet group | Complete |
| **Secrets Manager** | `SecretsManager/` | 5 secrets (db-host, db-name, db-user, db-password, jwt-secret) | Complete |
| **Cognito** | `Cognito/` | User pool, app client (provisioned, standby) | Complete |
| **Amplify** | *(console-managed)* | App `dpqtxdawh7h1c`; Terraform tracks ID/domain vars only | Complete |
| **Observability** | `Observability/` | 5 alarms, SNS, budget, CloudTrail trail | Complete |
| **Logs** | `terraform/logs/` | S3 log bucket (CloudTrail + ALB access logs) | Complete |
| **ECR** | `Ecr/` | `tradecore-api` repository | Complete |
| **S3** | `S3/` | App-data bucket (invoice docs) | Complete |
| **ACM** | `Acm/` | Dormant (custom-domain path; DuckDNS uses LE-import) | Complete |
| **IAM** | `terraform/iam/` | OIDC provider, GitHub Actions role with least-privilege policy | Complete |
| **State** | `terraform/state/` | S3 bucket (versioned, encrypted), DynamoDB lock table | Complete |

---

## Subnet Layout

**VPC CIDR:** `10.0.0.0/16`
**Availability Zones:** af-south-1a, af-south-1b

| Subnet | CIDR | AZ | Purpose | Resources |
|--------|------|-----|---------|-----------|
| public-0 | `10.0.0.0/24` | af-south-1a | Public | ALB, ECS tasks |
| public-1 | `10.0.1.0/24` | af-south-1b | Public | ALB, ECS tasks |
| private-0 | `10.0.2.0/24` | af-south-1a | Private | RDS (subnet group spans both) |
| private-1 | `10.0.3.0/24` | af-south-1b | Private | RDS (single-AZ, no standby) |

---

## CI/CD Pipeline

**File:** `.github/workflows/terraform.yml`

### Triggers

| Event | Action |
|-------|--------|
| Pull request to `main` (paths: `terraform/**`, `*/*.tf`, `*.tf`, `.github/workflows/**`, lockfile) | `terraform plan` |
| Push to `main` (same paths) | `terraform plan` + `terraform apply` |

### Pipeline Stages

```
Preflight (secrets check) → Security scan (Trivy secret + misconfig, blocking) → Checkout → AWS OIDC Auth → Setup Terraform → Format Check → Init → Validate → Plan → Apply
```

Plus a daily **drift detection** workflow (`.github/workflows/drift.yml`): read-only plan, fails on drift.

### Authentication

Uses **OIDC (OpenID Connect)** — no static AWS credentials stored in GitHub. The workflow assumes an IAM role via GitHub's identity token.

---

## Security

### Least-Privilege IAM

The GitHub Actions IAM role is scoped to specific resource ARNs for each service:

| Service | Permissions | Resource Scope |
|---------|-------------|----------------|
| ECS | Full management | `cluster/tradecore-*`, `service/*`, `task-definition/*:*` (+ `DeregisterTaskDefinition` on `*`, required by AWS) |
| ECR | Pull/push images | `arn:aws:ecr:*:*:repository/tradecore-*` (+ `GetAuthorizationToken` on `*`, required by AWS) |
| RDS | Full CRUD (scoped) | `db:tradecore-*`, `subgrp:tradecore-*`, snapshots, parameter groups |
| Secrets Manager | Full CRUD + version staging | `arn:aws:secretsmanager:*:*:secret:/tradecore/*` |
| CloudWatch Logs | Create/write/retention | `log-group:/ecs/tradecore/*` (+ RDS/VPC log groups) |
| ALB | Full management | `loadbalancer/app/tradecore-*`, `listener/app/tradecore-*`, `targetgroup/tradecore-*` |
| CloudTrail | Trail lifecycle | `*` (required by AWS for Create/Describe/List) |
| KMS | Trail key lifecycle | `*` (required by AWS for key administration) |
| Cognito | Full management | `arn:aws:cognito-idp:*:*:userpool/*` |
| S3 | Buckets (state, data, logs) | `arn:aws:s3:::tradecore-*` |
| DynamoDB | State lock (legacy; locking now S3-native) | `arn:aws:dynamodb:*:*:table/tradecore-*` |
| IAM | Pass roles, OIDC | `arn:aws:iam::*:role/tradecore-*` |

### Additional Security Measures

- **No static AWS credentials** — OIDC-based authentication
- **Scoped IAM policies** — All permissions limited to project-specific resource ARNs, except actions AWS requires on `*` (`ecr:GetAuthorizationToken`, `ecs:DeregisterTaskDefinition`, CloudTrail/KMS administration, list-style describes) and the Amplify exception below
- **Amplify exception** — `amplify:*` on `us-east-1` apps (`terraform/iam/main.tf:451-453`); frontend is console-managed so this is rarely exercised
- **TLS everywhere** — ALB `:443` with imported Let's Encrypt cert (`tradecore-prod.duckdns.org`, renew ~2026-11-23); `:80` 301-redirects
- **Secrets Manager** — All sensitive values injected as environment variables at runtime
- **RDS encryption** — Storage encrypted with AWS-managed key
- **S3 state encryption** — AES-256 server-side encryption
- **S3 public access blocked** — All public access prohibited on state bucket
- **RDS publicly accessible** — Set to `false`
- **ECS security group** — Inbound only from ALB

---

## Cost Model

**Budget constraint:** <$30 for project duration

| Component | Cost Driver | Est. Monthly |
|-----------|-------------|--------------|
| ECS Fargate | 2 tasks × runtime hours | ~$15-20 |
| ALB | Runs continuously while deployed | ~$5-10 |
| RDS db.t3.micro | Continuous database runtime | ~$5-10 |
| RDS storage | 20GB gp3 storage | ~$2-3 |
| Secrets Manager | 5 secrets | ~$0.40 |
| KMS (trail key) | 1 CMK with rotation | ~$1 |
| CloudTrail | Mgmt events, first trail free | ~$0 |
| S3 (state, data, logs) | Storage + requests | ~$0.10 |
| CloudWatch Logs | Log ingestion/storage | ~$1-2 |
| Amplify | Build/hosting | ~$1-5 |
| ECR | Image storage | ~$0.50 |
| Cognito | Low at capstone scale | ~$0 |
| S3 state | Very small | ~$0.01 |
| DynamoDB state lock | PAY_PER_REQUEST | ~$0.00 |
| **NAT Gateway** | **NOT deployed** | **$0** |
| **Total** | | **~$30-50** |

**Biggest cost driver:** ECS Fargate (2 tasks running continuously)

**Cost-saving decisions:**
- No NAT Gateway (ECS in public subnets with public IPs)
- Single-AZ RDS (no Multi-AZ)
- db.t3.micro instance class
- PAY_PER_REQUEST DynamoDB

---

## Getting Started

### Prerequisites

- AWS CLI v2 installed
- AWS SSO configured with `AdministratorAccess` permission set
- Terraform >= 1.16 (for local development)
- GitHub account with repository access

### 1. Configure AWS SSO

```bash
aws configure sso --profile ENOFE
# Enter:
#   SSO start URL: https://tradecore-dev.awsapps.com/start
#   SSO region: af-south-1
#   Default region: af-south-1
#   Default output: json
```

### 2. Verify Identity

```bash
aws sts get-caller-identity --profile ENOFE
```

### 3. Clone Repository

```bash
git clone https://github.com/Isiaka-Ismail/tradecore-project2.git
cd tradecore-project2
```

---

## Deployment Guide

### Phase 1: Bootstrap State Infrastructure

```bash
cd terraform

# Required variables have no defaults, so Terraform refuses to plan at all without
# them — even when targeting a single module. Export them first; placeholders are
# fine for the bootstrap, Phase 4 replaces them with real values.
export TF_VAR_environment=production
export TF_VAR_aws_profile=ENOFE
export TF_VAR_container_image=placeholder
export TF_VAR_db_name=tradecore
export TF_VAR_db_username=tradecoreDB
export TF_VAR_db_password='<YOUR_DB_PASSWORD>'
export TF_VAR_jwt_secret='<YOUR_JWT_SECRET>'
export TF_VAR_github_org_id='<GITHUB_ORG_ID>'
export TF_VAR_github_repo_id='<GITHUB_REPO_ID>'

# Initialize without backend (state not yet created)
terraform init -backend=false

# Preview state infrastructure
terraform plan -target=module.state

# Create state infrastructure (S3 bucket + DynamoDB table)
terraform apply -target=module.state

# Get state bucket and table names (terraform output accepts one NAME at a time)
terraform output state_bucket_name
terraform output state_dynamodb_table_name
```

### Phase 2: Migrate to Remote State

```bash
# backend.tf ships with an empty backend block — bucket/key/region are supplied at
# init time, exactly as CI does in .github/workflows/terraform.yml.
terraform init -migrate-state \
  -backend-config="bucket=<STATE_BUCKET>" \
  -backend-config="key=<STATE_KEY>" \
  -backend-config="region=af-south-1" \
  -backend-config="use_lockfile=true"
```

### Phase 3: Configure GitHub Secrets

Add these secrets in GitHub repository settings (Settings → Secrets and variables → Actions):

| Secret | Value | Source |
|--------|-------|--------|
| `AWS_ROLE_ARN` | `terraform output github_actions_role_arn` | IAM (deployed) |
| `DB_NAME` | `tradecore` (live truth) | Secrets Manager |
| `DB_USERNAME` | `tradecoreDB` (live truth) | Secrets Manager |
| `DB_PASSWORD` | Rotated secret value | Secrets Manager |
| `JWT_SECRET` | Rotated secret value | Secrets Manager |
| `CERTIFICATE_ARN` | Imported LE cert ARN (enables HTTPS; omit stays HTTP) | ACM console |
| `DOMAIN_NAME` | Leave unset (DuckDNS has no CNAME for ACM) | — |
| `CONTAINER_IMAGE` | ECR image URI with SHA tag | ECR console |
| `TF_STATE_BUCKET` | State bucket name | State output |
| `TF_STATE_KEY` | State key (`tradecore/<env>/terraform.tfstate`) | State output |
| `TF_LOCK_TABLE` | DynamoDB table (legacy; locking is S3-native now) | State output |
| `ORG_GITHUB_ID` / `REPO_GITHUB_ID` | Numeric GitHub IDs for OIDC trust | GitHub API |

### Phase 4: Full Deployment

```bash
# Plan full deployment (secrets travel as env vars, never CLI args)
export TF_VAR_environment=production
export TF_VAR_aws_profile=ENOFE
export TF_VAR_db_name=tradecore
export TF_VAR_db_username=tradecoreDB
export TF_VAR_db_password='<YOUR_DB_PASSWORD>'
export TF_VAR_jwt_secret='<YOUR_JWT_SECRET>'
export TF_VAR_container_image='<YOUR_ECR_IMAGE_URI>'
# Numeric IDs for the OIDC trust conditions — also required, no defaults
export TF_VAR_github_org_id='<GITHUB_ORG_ID>'
export TF_VAR_github_repo_id='<GITHUB_REPO_ID>'
# Optional: export TF_VAR_certificate_arn='<CERT_ARN>'  # enables HTTPS
terraform plan

# Apply
terraform apply

# Get GitHub Actions role ARN
terraform output github_actions_role_arn
```

### Phase 5: Test CI/CD

1. Add the `AWS_ROLE_ARN` to GitHub secrets
2. Push a change to any file in `terraform/`
3. Check GitHub Actions for successful plan/apply

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **No NAT Gateway** | Cost-saving: ECS tasks use public subnets with public IPs and Internet Gateway routing |
| **ECS in Public Subnets** | Avoids NAT Gateway cost; tasks receive public IPs for direct internet access |
| **Fargate (not EC2)** | Serverless compute — no EC2 instances to manage, pay per task |
| **awsvpc networking** | Each task gets its own ENI for better network isolation |
| **Secrets Manager injection** | Secrets injected as env vars at runtime, never in code or images |
| **OIDC authentication** | No static AWS credentials in GitHub — uses identity tokens |
| **S3 + DynamoDB state** | Remote state with locking prevents concurrent modifications |
| **Single-AZ RDS** | Cost-saving for project; no automatic failover |
| **db.t3.micro** | Smallest instance class for project budget |
| **Amplify in us-east-1** | Amplify not available in af-south-1; secondary provider for frontend only |
| **Least-privilege IAM** | GitHub Actions role scoped to only services this project uses |

---

## Project Structure

```
tradecore-project2/
├── .github/
│   └── workflows/
│       ├── terraform.yml          # CI/CD pipeline (OIDC)
│       └── drift.yml              # Daily drift detection (read-only)
├── Acm/                          # ACM module (dormant; LE-import used instead)
├── Ecr/                           # ECR repository module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── Observability/                 # Alarms, SNS, budget, CloudTrail
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── S3/                            # App-data bucket module
├── Alb/                           # Application Load Balancer module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── Cognito/                       # Cognito User Pool module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── Database/                      # RDS PostgreSQL module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── Ecs/                           # ECS Fargate module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── Networking/                    # VPC + Subnets module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── SecretsManager/                # Secrets Manager module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── terraform/                     # Root Terraform configuration
│   ├── backend.tf                 # S3 backend
│   ├── iam/                       # OIDC + GitHub Actions IAM
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── logs/                      # S3 log bucket (CloudTrail + ALB logs)
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── main.tf                    # Module wiring
│   ├── outputs.tf                 # Root outputs
│   ├── provider.tf                # AWS providers (af-south-1 + us-east-1)
│   ├── state/                     # State infrastructure (S3 + DynamoDB)
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── variables.tf               # Root variables
│   └── versions.tf                # Terraform >= 1.16
├── progress.md                    # Internal progress tracker (not committed)
├── SUMMARY.md                     # Internal summary (not committed)
└── README.md                      # This file
```

---

## Environment Variables

### ECS Container Environment

| Variable | Source | Description |
|----------|--------|-------------|
| `APP_VERSION` | Hardcoded | Application version |
| `AWS_REGION` | Hardcoded | `af-south-1` |
| `NODE_ENV` | Hardcoded | `production` (masks 500 details) |
| `FRONTEND_URL` | Terraform var | CORS allowlist (comma-separated) |
| `DB_SSL` | Terraform var | `true` (encrypted Postgres) |
| `S3_BUCKET` | S3 module | Invoice document bucket |
| `DB_HOST` | Secrets Manager | RDS endpoint |
| `DB_NAME` | Secrets Manager | Database name |
| `DB_USER` | Secrets Manager | Database username |
| `DB_PASSWORD` | Secrets Manager | Database password |
| `JWT_SECRET` | Secrets Manager | JWT signing secret |

---

## GitHub Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AWS_ROLE_ARN` | Yes | IAM role ARN for OIDC authentication |
| `DB_NAME` | Yes | PostgreSQL database name |
| `DB_PASSWORD` | Yes | PostgreSQL master password |
| `DB_USERNAME` | Yes | PostgreSQL master username |
| `JWT_SECRET` | Yes | JWT signing secret |
| `CERTIFICATE_ARN` | No | Imported cert ARN — enables HTTPS when set |
| `DOMAIN_NAME` | No | Leave unset (no CNAME validation possible) |
| `CONTAINER_IMAGE` | Yes | ECR image URI (SHA tag) |
| `TF_STATE_BUCKET` | Yes | S3 bucket name for Terraform state |
| `TF_STATE_KEY` | Yes | Terraform state file path |
| `TF_LOCK_TABLE` | Yes | DynamoDB table (legacy; locking is S3-native) |
| `ORG_GITHUB_ID` / `REPO_GITHUB_ID` | Yes | Numeric GitHub IDs for OIDC trust |

---

## License

This project is part of a TradeCore capstone submission.
