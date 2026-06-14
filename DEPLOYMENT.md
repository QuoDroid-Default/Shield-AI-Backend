# Shield-AI Backend — Deployment Readiness

## Current Status

**Codebase: Production-ready** | **Infrastructure: Staging required** | **Operations: Not yet set up**

- 11 sprints complete (28 stories)
- Cross-cutting security hardening applied
- 3619 tests passing (all unit/mock-based)

---

## What's Built

### Middleware Pipeline
| Component | Position | Story |
|-----------|----------|-------|
| Audit Logger | 1 | SHIELD-23 |
| Security Headers | 2 | SHIELD-2 |
| WAF & Rate Limiting | 3 | SHIELD-1 |
| Response Sanitizer | 4 | Sprint 3 |
| Callback Verifier | 5 | SHIELD-20 |
| SSRF Validator | 6 | SHIELD-19 |
| Code Validator | 11 | SHIELD-29 |

### Infrastructure (AWS-only)
- **Compute:** AWS ECS Fargate (Terraform)
- **CDN/Edge:** CloudFront SaaS multi-tenant distribution with AWS WAF (edge + regional)
- **Kubernetes:** Helm chart with NetworkPolicies (default-deny) and Pod Security Standards (restricted)
- **Database:** PostgreSQL with Row-Level Security (tenant isolation)
- **Cache:** Redis for rate limiting and config caching

> **Note:** A Cloudflare edge module exists in `terraform/cloudflare-edge/` but is **not used** — Cloudflare's required features (WAF managed rulesets, advanced rate limiting) are only available on their Enterprise plan. AWS CloudFront + AWS WAF provides equivalent edge protection at lower cost.

### Security Features
- HMAC-SHA256 callback verification with secret rotation
- Pluggable secrets management (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault)
- AST-based Python + regex-based JavaScript code validation
- SSRF protection with DNS cache and IPv6 transition range blocking
- Structured JSON logging with webhook dispatch for security events
- Audit logging with async queue and background batch flusher
- RFC 9116 security.txt and Vulnerability Disclosure Policy

### CI/CD & Supply Chain
- Composite GitHub Action: SAST, SCA, secrets scanning, container scanning
- SARIF aggregation for GitHub Security tab
- CycloneDX SBOM generation (Python, JS, Go, Docker)
- 9 SOC 2 / ISO 27001 policy templates

### Cross-Cutting Hardening (Latest)
- All error responses run through security pipeline
- App CRUD routes customer-scoped (IDOR fix)
- PostgreSQL `command_timeout=30`, Redis `socket_timeout=5`
- Async DNS validation in webhook dispatch
- Constant-time callback secret iteration
- `logger.error` (not `.exception`) in sensitive error handlers
- getattr dynamic-attribute bypass detection
- Never-allowed imports blocklist in code validation API

---

## Pre-Deployment Checklist

### Phase 1: Staging Environment
- [ ] Configure AWS credentials and region (`us-east-1` required for CloudFront WAF)
- [ ] `terraform init && terraform plan` for each module:
  - [ ] `terraform/proxy-ecs/`
  - [ ] `terraform/db-proxy-ecs/`
  - [ ] `terraform/waf/`
  - [ ] `terraform/security-headers/`
  - [ ] `terraform/cloudfront-saas/`
- [ ] `terraform apply` to staging account
- [ ] Verify ECS tasks are running and healthy
- [ ] Verify ALB target group health checks pass

### Phase 2: Database Setup
- [ ] Provision PostgreSQL (RDS recommended, 13+ for RLS)
- [ ] Set `DATABASE_URL` environment variable
- [ ] Run migrations (`run_migrations()` in lifespan)
- [ ] Verify RLS setup: `shieldai_owner` and `shieldai_app` roles created
- [ ] Verify RLS policies on `customers`, `apps`, `audit_logs`, `webhooks` tables
- [ ] Test tenant isolation: confirm cross-tenant queries return empty
- [ ] Provision Redis (ElastiCache recommended)
- [ ] Set `REDIS_URL` environment variable

### Phase 3: Secrets & Configuration
- [ ] Choose secrets provider (AWS/GCP/Vault) and configure backend
- [ ] Set `SECRETS_PROVIDER` and provider-specific env vars
- [ ] Generate and store callback verification secrets
- [ ] Set `ORIGIN_VERIFY_SECRET` for CloudFront origin bypass prevention (via `TF_VAR_` or `-var`)
- [ ] Configure `config.yaml` or environment variables for:
  - [ ] `upstream_url` — target application URL
  - [ ] `enabled_features` — feature flags per middleware
  - [ ] `rate_limit` settings
  - [ ] `security_headers` overrides
  - [ ] `code_validator` settings (opt-in, default off)
  - [ ] `audit_logging` — PostgreSQL batch flush settings
  - [ ] `structured_logging` — log level, webhook URLs

### Phase 4: Integration Testing
- [ ] Deploy proxy with a test upstream application
- [ ] Verify proxy forwards requests correctly (GET, POST, PUT, DELETE)
- [ ] Verify security headers are present on all responses
- [ ] Verify rate limiting triggers at configured threshold
- [ ] Verify WAF blocks malicious payloads (SQLi, XSS)
- [ ] Verify SSRF validator blocks internal IPs in callbacks
- [ ] Verify audit logs are written to PostgreSQL
- [ ] Verify webhook dispatch for security events
- [ ] Test customer onboarding flow end-to-end:
  - [ ] Create onboarding → ACM cert issued
  - [ ] DNS validation → cert validated
  - [ ] Tenant created → CloudFront distribution updated
  - [ ] Status transitions through `certificate_pending → active`
- [ ] Test offboarding flow (cleanup resources)

### Phase 5: Load Testing
- [ ] Establish baseline: requests/sec at acceptable latency (p50, p95, p99)
- [ ] Test with concurrent tenants (verify RLS under load)
- [ ] Test audit queue behavior under burst traffic (verify drop counter)
- [ ] Test rate limiter accuracy under sustained load
- [ ] Identify bottleneck (proxy, database, upstream) and tune:
  - [ ] ECS task count / CPU / memory
  - [ ] PostgreSQL connection pool size (`min_size`, `max_size`)
  - [ ] Redis connection limits
  - [ ] Audit batch size and flush interval

### Phase 6: Monitoring & Alerting
- [ ] Configure log aggregation (CloudWatch Logs, Datadog, or similar)
- [ ] Create dashboards for:
  - [ ] Request rate and latency (p50/p95/p99)
  - [ ] Error rate (4xx, 5xx)
  - [ ] Rate limit triggers per tenant
  - [ ] WAF block count by rule
  - [ ] Audit queue depth and drop count
  - [ ] PostgreSQL connection pool utilization
  - [ ] Redis memory and connection count
- [ ] Set up alerts for:
  - [ ] Error rate > threshold
  - [ ] Latency p99 > threshold
  - [ ] Audit drops > 0
  - [ ] Database connection pool exhaustion
  - [ ] ECS task health check failures
  - [ ] Certificate expiration (< 30 days)
- [ ] Configure PagerDuty / OpsGenie / Slack for on-call notifications

### Phase 7: Operational Readiness
- [ ] Write runbooks for:
  - [ ] Proxy restart / rollback procedure
  - [ ] Database failover
  - [ ] Secret rotation procedure
  - [ ] Customer onboarding troubleshooting
  - [ ] Incident response for security events
- [ ] Set up backup strategy:
  - [ ] PostgreSQL automated snapshots (daily, 30-day retention)
  - [ ] Configuration backup (config.yaml, Terraform state)
- [ ] Document API endpoints (OpenAPI/Swagger for config and onboarding APIs)
- [ ] Establish change management process (PR review, staging → prod promotion)

### Phase 8: Production Deployment
- [ ] Deploy to production with a single low-risk tenant
- [ ] Monitor for 48 hours — verify no anomalies
- [ ] Gradually onboard additional tenants
- [ ] Enable CloudFront distribution for customer domains
- [ ] Verify security.txt is accessible at `/.well-known/security.txt`

---

## Architecture Overview

```
Customer Domain (CNAME)
       │
       ▼
┌──────────────────────┐
│  CloudFront (CDN)    │
│  + AWS WAF (edge)    │
│  + Security Headers  │
│  + TLS 1.2+          │
└──────────┬───────────┘
           │ X-ShieldAI-Origin-Verify
           ▼
┌──────────────────────┐
│  ALB (regional)      │
│  + AWS WAF (regional)│
│  + Origin verify     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  ECS Fargate         │
│  ┌──────────────────┐│
│  │  ShieldAI Proxy  ││
│  │  ┌──────────────┐││
│  │  │  Middleware   │││
│  │  │  Pipeline     │││
│  │  └──────┬───────┘││
│  └─────────┼────────┘│
└────────────┼─────────┘
             │
  ┌──────────┼──────────┐
  ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌──────────┐
│ RDS    │ │ Elasti-│ │ Upstream │
│ Postgre│ │ Cache  │ │ App      │
│ SQL    │ │ Redis  │ │          │
└────────┘ └────────┘ └──────────┘
```

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `REDIS_URL` | Yes | Redis connection string |
| `UPSTREAM_URL` | Yes | Target application URL |
| `SECRETS_PROVIDER` | No | `aws`, `gcp`, or `vault` (default: env vars) |
| `AWS_ACCESS_KEY_ID` | Yes | AWS credentials (or use IAM role) |
| `AWS_SECRET_ACCESS_KEY` | Yes | AWS credentials (or use IAM role) |
| `LOG_LEVEL` | No | `debug`, `info`, `warning`, `error` (default: `info`) |
| `ORIGIN_VERIFY_SECRET` | If CloudFront | Shared secret for origin bypass prevention |
| `AWS_REGION` | Yes | AWS region (`us-east-1` required for CloudFront WAF) |

---

## Test Suite

```
Total: 3619 tests
├── Unit tests (proxy modules)
├── Story tests (per-sprint acceptance)
├── Attack simulation tests (security hardening)
└── Infrastructure tests (Terraform, Helm, GitHub Actions)

Run: pytest tests/ -q
```
