# zen-pharma-backend

Spring Boot + Node.js microservices monorepo for the Zen Pharma platform. Contains 8 backend
services (7 Java 17 / Spring Boot + 1 Node.js) deployed to AWS EKS via GitOps (ArgoCD).

> **This fork (`ajay-bj`) — current status (2026-09-03):** all backend services build in CI and
> deploy to the dev environment. Every service has an image in ECR (account `304312474711`) and
> all pods are `Running`. Image tag currently deployed: `sha-4088c8f`.

> **Companion forks:**
> - [`zen-infra-ajay`](https://github.com/ajay-bj/zen-infra-ajay) — Terraform for AWS infrastructure (EKS, RDS, ECR, IAM)
> - [`zen-pharma-frontend-ajay`](https://github.com/ajay-bj/zen-pharma-frontend-ajay) — React frontend
> - [`zen-gitops-ajay`](https://github.com/ajay-bj/zen-gitops-ajay) — ArgoCD apps + Helm values

---

## Services

| Service | Description | Port | DB |
|---|---|---|---|
| `api-gateway` | Spring Cloud Gateway — routes all external traffic | 8080 | No |
| `auth-service` | JWT authentication and user management | 8081 | PostgreSQL |
| `drug-catalog-service` | Drug catalogue — search, categories, formulary | 8082 | PostgreSQL |
| `inventory-service` | Stock levels, replenishment, batch tracking | 8083 | PostgreSQL |
| `supplier-service` | Supplier management and purchase orders | 8084 | PostgreSQL |
| `manufacturing-service` | Production orders and batch manufacturing | 8085 | PostgreSQL |
| `qc-service` | Quality control — batch checks and results | 8086 | PostgreSQL (H2 embedded) |
| `notification-service` | Email/SMS notifications (Node.js 20 / Express) | 3000 | No |

---

## Repository Structure

```
zen-pharma-backend/
├── api-gateway/
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── auth-service/
│   └── ...
├── drug-catalog-service/
│   └── ...
├── inventory-service/
│   └── ...
├── manufacturing-service/
│   └── ...
├── supplier-service/
│   └── ...
├── qc-service/                    ← Quality control (Java)
│   └── ...
├── notification-service/          ← Node.js (not Java)
│   └── ...
└── .github/
    └── workflows/
        ├── _java-build.yml        ← Reusable: full Java CI pipeline
        ├── _java-pr-check.yml     ← Reusable: lightweight PR check
        ├── _node-build.yml        ← Reusable: full Node.js CI pipeline
        ├── _node-pr-check.yml     ← Reusable: lightweight Node PR check
        ├── ci-<service>.yml       ← Full build + DEV deploy + QA PR (8 files)
        ├── ci-pr-<service>.yml    ← Feature branch check (8 files)
        └── promote-prod.yml       ← Manual PROD promotion trigger
```

---

## CI Pipeline Overview

Every push to `develop` or `release/**` runs the full pipeline for the changed service
(path filter `<service>/**`):

```
1. Maven verify + JaCoCo coverage  — real PostgreSQL sidecar for DB services
   (Node services: npm ci + ESLint + Jest coverage gate ≥ 80%)
2. CodeQL SAST (security-extended)          — disabled on forks (needs Code Scanning enabled)
3. OWASP Dependency Check (Java) / npm audit (Node) — advisory, non-blocking
4. Docker build (multi-stage, non-root UID 1000)
5. Trivy image scan (HIGH/CRITICAL, ignore-unfixed) — advisory, non-blocking
6. ECR push → tag: sha-<7chars>
7. Cosign keyless sign (GitHub OIDC → Fulcio → Rekor)
8. Update envs/dev/values-<service>.yaml in zen-gitops → ArgoCD auto-syncs dev
9. Open QA promotion PR in zen-gitops
```

> **Fork note:** on a fork without GitHub Code Scanning enabled, the CodeQL and "Upload SARIF"
> steps are set to `if: false`, and the dependency/audit + Trivy gates are advisory
> (non-blocking) in the reusable workflows (`_java-build.yml`, `_node-build.yml`). This lets the
> build reach the ECR push + Cosign sign stages on a personal fork.

Feature branch pushes (`feat/*`, `fix/*`, `chore/*`) run only the lightweight
test + SAST check (no Docker/ECR).

**Authentication to AWS:** GitHub OIDC (no `AWS_ACCESS_KEY_ID` stored as a secret) — assumes
`arn:aws:iam::304312474711:role/pharma-dev-github-actions-role`.

---

## Branching Strategy

| Branch | Purpose | CI |
|---|---|---|
| `feat/*`, `fix/*`, `chore/*` | Feature development | Lightweight: test + SAST only |
| `develop` | Integration branch | Full pipeline + DEV deploy |
| `release/**` | Sprint release / hotfix | Full pipeline + DEV deploy |
| `main` | Stable / matches production | PR check only |

PROD is promoted manually via `promote-prod.yml` (workflow_dispatch with service dropdown).

---

## Local Development

### Prerequisites
- Java 17 (`sdk install java 17-tem`)
- Maven 3.9+
- Docker Desktop
- PostgreSQL 15 (for DB services)

### Run a service locally

```bash
# Auth service example
cd auth-service

# Start PostgreSQL (Docker)
docker run -d --name pharma-db \
  -e POSTGRES_DB=pharma \
  -e POSTGRES_USER=pharma \
  -e POSTGRES_PASSWORD=pharma \
  -p 5432:5432 postgres:15-alpine

# Set environment variables
export SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/pharma
export SPRING_DATASOURCE_USERNAME=pharma
export SPRING_DATASOURCE_PASSWORD=pharma
export JWT_SECRET=local-dev-secret

# Run
mvn spring-boot:run
```

### Run tests

```bash
cd auth-service
mvn verify                        # unit + integration tests + JaCoCo coverage
mvn verify -Pintegration-tests    # integration tests only
```

### Build Docker image locally

```bash
cd auth-service
docker build -t auth-service:local .
docker run -p 8081:8081 auth-service:local
```

---

## Forking this repo

See **[`implementation.md`](./implementation.md)** for the complete fork setup guide covering AWS OIDC, IAM role, ECR repos, GitHub secrets, ArgoCD, and GitOps layout.

Quick checklist (already done for the `ajay-bj` fork):
1. Update the IAM role trust policy — the `sub` condition allows `repo:ajay-bj/zen-pharma-backend-ajay:*` (missing org name is the most common failure)
2. If you use a different IAM role name than `pharma-dev-github-actions-role`, update `role-to-assume` in `.github/workflows/_java-build.yml` and `_node-build.yml`
3. Set the GitHub secrets and variable below
4. Create the 9 ECR repositories in your AWS account (one per service)
5. Set `GITOPS_REPO` to point at your own gitops repo (`ajay-bj/zen-gitops-ajay`)

---

## Required GitHub Secrets

Set in **Settings → Secrets and variables → Actions**:

| Secret | Description | This fork |
|---|---|---|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | `304312474711` |
| `GITOPS_TOKEN` | GitHub PAT with `contents: write` on your gitops repo | set on `ajay-bj/zen-gitops-ajay` |
| `NVD_API_KEY` | NIST NVD API key for OWASP Dep Check (optional, faster) | optional |

| Variable | Value |
|---|---|
| `GITOPS_REPO` | `ajay-bj/zen-gitops-ajay` |

---

## Full Deployment Guide

See [`implementation.md`](./implementation.md) for the complete setup guide including OIDC, IAM, ECR, ArgoCD, and GitOps configuration.
