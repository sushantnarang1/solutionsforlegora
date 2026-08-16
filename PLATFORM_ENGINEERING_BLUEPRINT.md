# Platform Engineering Blueprint for Legora OS
## A Comprehensive Guide to Scaling Legal Document Analysis with Modern DevOps & Platform Engineering

**Version:** 1.0  
**Last Updated:** August 2026  
**Status:** Production-Ready Architecture

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Platform Architecture Overview](#platform-architecture-overview)
3. [Infrastructure-as-Code & Environment Management](#infrastructure-as-code--environment-management)
4. [CI/CD Pipeline Design](#cicd-pipeline-design)
5. [Observability & Security Stack](#observability--security-stack)
6. [Developer Experience & Self-Service Catalog](#developer-experience--self-service-catalog)
7. [Implementation Roadmap](#implementation-roadmap)
8. [Quick Reference & Checklists](#quick-reference--checklists)

---

## Executive Summary

Legora OS processes sensitive legal documents and requires a platform engineering approach that balances **developer velocity** with **compliance rigor**, **security**, and **auditability**. This blueprint provides:

- **Self-service developer platform** with standardized workflows
- **Multi-environment infrastructure** with secrets management and policy enforcement
- **Compliance-first CI/CD** with audit trails and automated rollbacks
- **Enterprise observability** tracking PII/PHI exposure and regulatory violations
- **Legal tech-specific security** including document encryption, access controls, and retention policies

### Key Principles
✅ **Security First**: Zero-trust architecture, encryption at rest/transit, secret rotation  
✅ **Audit-Ready**: Every action logged, traceable, and complaint-ready for regulatory audits  
✅ **Developer Self-Service**: Reduce friction, minimize toil, maximize autonomy  
✅ **Cost Efficient**: Leverage Vercel + cloud-native stack; pay only for what you use  
✅ **Scalable AI/ML**: Orchestrate model training, inference, and versioning systematically  

---

## Platform Architecture Overview

### 1. System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Legora OS Platform Architecture                  │
└─────────────────────────────────────────────────────────────────────────┘

                          ┌─────────────────────────┐
                          │   Developer Workflows   │
                          │  (Local + Git-driven)   │
                          └────────────┬────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
            ┌───────▼────────┐ ┌──────▼────────┐ ┌──────▼────────┐
            │  Frontend Team │ │ Backend Team  │ │ ML/Data Team  │
            │  (Next.js)     │ │ (FastAPI)     │ │ (Training)    │
            └────────────────┘ └───────────────┘ └───────────────┘
                    │                  │                  │
                    └──────────────────┼──────────────────┘
                                       │
                         ┌─────────────▼──────────────┐
                         │   Control Plane / IDP      │
                         │  ┌────────────────────────┐│
                         │  │ Service Catalog        ││
                         │  │ - Templates            ││
                         │  │ - Blueprints           ││
                         │  │ - Policies             ││
                         │  └────────────────────────┘│
                         └─────────────┬──────────────┘
                                       │
          ┌────────────────────────────┼────────────────────────────┐
          │                            │                            │
    ┌─────▼──────┐         ┌──────────▼──────────┐        ┌────────▼────────┐
    │ CI/CD Layer│         │  Infra Management   │        │  Observability  │
    │ ┌────────┐ │         │  ┌──────────────┐   │        │  ┌────────────┐ │
    │ │GitHub  │ │         │  │ Terraform    │   │        │  │Prometheus │ │
    │ │Actions │ │         │  │ (IaC)        │   │        │  │Grafana    │ │
    │ └────────┘ │         │  │ ┌──────────┐ │   │        │  │ELK Stack  │ │
    │ ┌────────┐ │         │  │ │Secrets   │ │   │        │  └────────────┘ │
    │ │ArgoCD  │ │         │  │ │Vault     │ │   │        │  ┌────────────┐ │
    │ │        │ │         │  │ └──────────┘ │   │        │  │Audit Logs │ │
    │ └────────┘ │         │  └──────────────┘   │        │  │(Central)  │ │
    │            │         │                     │        │  └────────────┘ │
    └────────────┘         └─────────────────────┘        └─────────────────┘
          │                          │                            │
          └──────────────────────────┼────────────────────────────┘
                                     │
          ┌──────────────────────────┼──────────────────────────┐
          │                          │                          │
    ┌─────▼────────┐         ┌──────▼────────┐        ┌────────▼────────┐
    │ Vercel Prod  │         │ Cloud Staging │        │ Local/Ephemeral │
    │ ┌──────────┐ │         │ (AWS/GCP)     │        │ ┌────────────┐   │
    │ │Next.js   │ │         │ ┌──────────┐  │        │ │Docker Dev  │   │
    │ │(Edge)    │ │         │ │FastAPI   │  │        │ │ Environments│  │
    │ └──────────┘ │         │ │ Service  │  │        │ └────────────┘   │
    │ ┌──────────┐ │         │ │ (k8s)    │  │        │                  │
    │ │Serverless│ │         │ └──────────┘  │        │                  │
    │ │API       │ │         │               │        │                  │
    │ └──────────┘ │         │               │        │                  │
    └──────────────┘         └───────────────┘        └──────────────────┘
          │                          │                          │
    ┌─────▼────────┐         ┌──────▼────────┐        ┌────────▼────────┐
    │ Data Storage │         │ Cache Layer   │        │ ML Pipeline     │
    │ ┌──────────┐ │         │ ┌──────────┐  │        │ ┌────────────┐   │
    │ │MongoDB   │ │         │ │Redis     │  │        │ │Model Store │   │
    │ │Atlas     │ │         │ │Cluster   │  │        │ │ (S3/GCS)   │   │
    │ │(prod)    │ │         │ └──────────┘  │        │ │ (versioned)│   │
    │ ├──────────┤ │         │               │        │ └────────────┘   │
    │ │MongoDB   │ │         │               │        │ ┌────────────┐   │
    │ │(staging) │ │         │               │        │ │Orchestrator│   │
    │ └──────────┘ │         │               │        │ │(Airflow)   │   │
    └──────────────┘         └───────────────┘        └──────────────────┘
```

### 2. Service Boundaries & Components

#### Frontend Service (Next.js on Vercel)
- **Responsibility**: UI rendering, client-side routing, real-time document preview
- **Deployment**: Vercel Edge Functions + Vercel Serverless Functions
- **Key Features**:
  - Automatic SSL/TLS termination
  - Global CDN for static assets
  - Built-in environment variable management
  - Serverless functions for dynamic routes
  - Preview deployments for each PR

#### Backend Service (FastAPI)
- **Responsibility**: API orchestration, document processing, AI inference
- **Deployment**: Kubernetes (AWS EKS / GCP GKE) or Render/Railway for simplicity
- **Key Features**:
  - Stateless architecture for horizontal scaling
  - JWT-based auth with role-based access control (RBAC)
  - Request/response logging and audit trails
  - Rate limiting and DDoS protection
  - Webhook support for async event processing

#### Data Layer (MongoDB + Cache)
- **MongoDB Atlas**: Production data storage with enterprise security
- **Redis**: Session management, caching, rate-limit counters
- **S3/GCS**: Model artifacts, processed documents, compliance backups

#### ML/Data Pipeline
- **Model Training**: Containerized jobs on Kubernetes or Lambda
- **Model Registry**: MLflow or Hugging Face Model Hub
- **Feature Store**: Feast or Tecton for feature versioning
- **Orchestration**: Apache Airflow for complex workflows

#### Observability Stack
- **Metrics**: Prometheus + Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana) + Loki
- **Tracing**: Jaeger or Zipkin
- **Audit Logs**: Centralized tamper-proof log storage
- **Real User Monitoring**: Sentry for frontend, Application Performance Monitoring

#### Security & Compliance Layer
- **Secrets Management**: HashiCorp Vault or AWS Secrets Manager
- **Policy Engine**: OPA (Open Policy Agent) for authorization
- **Encryption**: TLS 1.3 in transit; AES-256 at rest
- **Access Control**: Okta/Auth0 SSO, MFA enforced
- **Vulnerability Scanning**: Trivy, Snyk, OWASP Dependency Check

---

## Infrastructure-as-Code & Environment Management

### 1. Multi-Environment Strategy

| Environment | Purpose | Deployment | Data | Scale |
|---|---|---|---|---|
| **Production** | Live platform, users, legal documents | Vercel (frontend) + EKS (backend) | Production MongoDB Atlas | Full HA, auto-scaling |
| **Staging** | Pre-production testing, compliance validation | Vercel Preview + EKS namespace | Staging MongoDB (sanitized data) | 80% prod scale |
| **Development** | Feature development, integration testing | Docker Compose or Render | Local/dev MongoDB | Minimal |
| **Ephemeral** | PR preview, feature branches | Vercel Preview Deployments + Docker | Temp clone of staging data | Per-PR basis |

### 2. Infrastructure-as-Code Stack

#### Recommended Tools
- **Terraform**: Multi-cloud infrastructure provisioning (AWS, GCP, or Vercel integrations)
- **Helm**: Kubernetes workload management
- **ArgoCD**: GitOps-driven deployment reconciliation
- **Pulumi** (alternative): Infrastructure as code in Python/TypeScript

#### Directory Structure

```
solutionsforlegora/
├── infrastructure/
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── vpc/
│   │   │   ├── kubernetes/
│   │   │   ├── database/
│   │   │   ├── cache/
│   │   │   ├── monitoring/
│   │   │   └── security/
│   │   ├── environments/
│   │   │   ├── prod/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   └── terraform.tfvars
│   │   │   ├── staging/
│   │   │   └── dev/
│   │   └── global/
│   │       └── backend.tf (remote state management)
│   ├── helm/
│   │   ├── charts/
│   │   │   ├── backend-service/
│   │   │   ├── monitoring/
│   │   │   └── security/
│   │   └── values/
│   │       ├── prod.yaml
│   │       ├── staging.yaml
│   │       └── dev.yaml
│   ├── k8s/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── networkpolicy.yaml
│   │   └── overlays/
│   │       ├── prod/
│   │       ├── staging/
│   │       └── dev/
│   ├── scripts/
│   │   ├── setup-cluster.sh
│   │   ├── configure-secrets.sh
│   │   └── backup-database.sh
│   └── docs/
│       ├── ARCHITECTURE.md
│       ├── DEPLOYMENT.md
│       └── DISASTER_RECOVERY.md
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── deploy-staging.yml
│       └── deploy-prod.yml
└── docs/
    └── platform/
        ├── README.md
        ├── QUICK_START.md
        └── DEVELOPER_GUIDE.md
```

### 3. Example: Terraform Configuration (Core Infrastructure)

```hcl
# infrastructure/terraform/environments/prod/main.tf

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
    vault = {
      source  = "hashicorp/vault"
      version = "~> 3.0"
    }
  }
  
  backend "s3" {
    bucket         = "legora-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "Terraform"
      Application = "LeoraOS"
    }
  }
}

# VPC with private subnets for security
module "vpc" {
  source = "../../modules/vpc"
  
  environment = "prod"
  cidr_block  = "10.0.0.0/16"
  
  availability_zones = 3
  enable_nat_gateway = true
  enable_vpn_gateway = true
}

# EKS Cluster for backend services
module "eks_cluster" {
  source = "../../modules/kubernetes"
  
  cluster_name    = "legora-prod"
  cluster_version = "1.27"
  
  vpc_id            = module.vpc.vpc_id
  private_subnets  = module.vpc.private_subnets
  
  # Enable cluster autoscaling
  enable_cluster_autoscaling = true
  min_size                   = 3
  desired_size               = 5
  max_size                   = 20
  
  # Security
  enable_logging = true
  enable_control_plane_logging = true
  
  # Add-ons
  enable_metrics_server = true
  enable_ebs_csi_driver = true
  
  tags = merge(
    local.common_tags,
    {
      Component = "Kubernetes"
    }
  )
}

# MongoDB Atlas for data
module "mongodb" {
  source = "../../modules/database"
  
  cluster_name    = "legora-prod"
  instance_size   = "M20"
  disk_size_gb    = 512
  
  # High availability
  num_shards       = 3
  replication_factor = 3
  
  # Security
  database_username = var.mongo_admin_user
  database_password = var.mongo_admin_password # From Vault
  
  # Compliance
  backup_enabled          = true
  backup_retention_days   = 90
  encryption_at_rest      = true
  
  region = "US"
  tags   = local.common_tags
}

# Redis for caching and session management
module "elasticache" {
  source = "../../modules/cache"
  
  cluster_id     = "legora-prod-redis"
  engine         = "redis"
  engine_version = "7.0"
  node_type      = "cache.r6g.xlarge"
  num_cache_nodes = 3
  
  # Multi-AZ for high availability
  automatic_failover_enabled = true
  
  # Security
  transit_encryption_enabled = true
  at_rest_encryption_enabled = true
  auth_token                 = var.redis_auth_token # From Vault
  
  subnet_group_name = module.vpc.elasticache_subnet_group
  security_group_ids = [aws_security_group.redis.id]
  
  tags = local.common_tags
}

# S3 for model artifacts and backups
resource "aws_s3_bucket" "model_artifacts" {
  bucket = "legora-model-artifacts-prod"
}

resource "aws_s3_bucket_versioning" "model_artifacts" {
  bucket = aws_s3_bucket.model_artifacts.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "model_artifacts" {
  bucket = aws_s3_bucket.model_artifacts.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "model_artifacts" {
  bucket = aws_s3_bucket.model_artifacts.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Monitoring infrastructure
module "monitoring" {
  source = "../../modules/monitoring"
  
  cluster_name   = module.eks_cluster.cluster_name
  cluster_arn    = module.eks_cluster.cluster_arn
  
  enable_prometheus = true
  enable_grafana    = true
  enable_loki       = true
  enable_jaeger     = true
  
  # Storage
  prometheus_storage_size = "100Gi"
  loki_storage_size      = "100Gi"
  
  tags = local.common_tags
}

# Vault integration for secrets management
module "vault_integration" {
  source = "../../modules/security"
  
  vault_addr   = var.vault_addr
  vault_token  = var.vault_token
  
  # Kubernetes authentication
  kubernetes_host   = module.eks_cluster.cluster_endpoint
  kubernetes_ca_cert = module.eks_cluster.cluster_ca_certificate
}

locals {
  common_tags = {
    Project     = "LeoraOS"
    Environment = "production"
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  }
}

output "eks_cluster_endpoint" {
  value = module.eks_cluster.cluster_endpoint
}

output "eks_cluster_security_group_id" {
  value = module.eks_cluster.cluster_security_group_id
}

output "mongodb_connection_string" {
  value     = module.mongodb.connection_string
  sensitive = true
}

output "redis_endpoint" {
  value = module.elasticache.primary_endpoint_address
}
```

### 4. Secrets Management Strategy

#### HashiCorp Vault Setup

```hcl
# infrastructure/terraform/modules/security/vault.tf

resource "vault_generic_secret" "mongodb_credentials" {
  path      = "secret/legora/production/mongodb"
  data_json = jsonencode({
    username = var.mongo_username
    password = var.mongo_password
    connection_string = "mongodb+srv://${var.mongo_username}:${var.mongo_password}@${var.mongo_host}"
  })
}

resource "vault_generic_secret" "jwt_secrets" {
  path      = "secret/legora/production/jwt"
  data_json = jsonencode({
    signing_key  = var.jwt_signing_key
    refresh_key  = var.jwt_refresh_key
    expiration   = "24h"
  })
}

resource "vault_generic_secret" "ai_service_keys" {
  path      = "secret/legora/production/ai-services"
  data_json = jsonencode({
    openai_api_key = var.openai_api_key
    huggingface_token = var.huggingface_token
    anthropic_api_key = var.anthropic_api_key
  })
}

resource "vault_generic_secret" "aws_credentials" {
  path      = "secret/legora/production/aws"
  data_json = jsonencode({
    access_key_id     = var.aws_access_key_id
    secret_access_key = var.aws_secret_access_key
    s3_bucket         = "legora-artifacts-prod"
  })
}

# Vault policy for Kubernetes pods
resource "vault_policy" "legora_backend" {
  name = "legora-backend"
  
  policy = <<EOH
# Allow reading all legora production secrets
path "secret/data/legora/production/*" {
  capabilities = ["read", "list"]
}

# Allow reading JWT secrets (rotating)
path "secret/data/legora/production/jwt" {
  capabilities = ["read"]
}

# Allow generating dynamic database credentials
path "database/creds/legora-backend" {
  capabilities = ["read"]
}
EOH
}

# Kubernetes auth method configuration
resource "vault_auth_backend" "kubernetes" {
  type = "kubernetes"
}

resource "vault_kubernetes_auth_backend_config" "kubernetes" {
  backend            = vault_auth_backend.kubernetes.path
  kubernetes_host    = "https://${var.kubernetes_host}"
  kubernetes_ca_cert = base64decode(var.kubernetes_ca_cert)
  issuer             = "https://kubernetes.default.svc.cluster.local"
}

resource "vault_kubernetes_auth_backend_role" "legora_backend" {
  backend                          = vault_auth_backend.kubernetes.path
  role_name                        = "legora-backend"
  bound_service_account_names      = ["legora-backend"]
  bound_service_account_namespaces = ["production"]
  token_ttl                        = 3600
  token_policies                   = [vault_policy.legora_backend.name]
}
```

#### Environment Variable Injection (Kubernetes)

```yaml
# infrastructure/k8s/base/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: legora-backend
  labels:
    app: legora-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: legora-backend
  template:
    metadata:
      labels:
        app: legora-backend
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "legora-backend"
        vault.hashicorp.com/agent-inject-secret-mongodb: "secret/data/legora/production/mongodb"
        vault.hashicorp.com/agent-inject-template-mongodb: |
          {{- with secret "secret/data/legora/production/mongodb" -}}
          export MONGODB_URI="{{ .Data.data.connection_string }}"
          {{- end }}
    spec:
      serviceAccountName: legora-backend
      containers:
      - name: backend
        image: gcr.io/legora-os/backend:{{ .Values.version }}
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 9090
          name: metrics
        
        env:
        # Non-sensitive config from ConfigMap
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: legora-config
              key: environment
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: legora-config
              key: log_level
        - name: MAX_WORKERS
          valueFrom:
            configMapKeyRef:
              name: legora-config
              key: max_workers
        
        # Pod identity for AWS/GCP
        - name: AWS_ROLE_ARN
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['iam.amazonaws.com/role']
        - name: AWS_STS_REGIONAL_ENDPOINTS
          value: "regional"
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
        
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/.cache
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - legora-backend
              topologyKey: kubernetes.io/hostname
      
      tolerations:
      - key: workload-type
        operator: Equal
        value: compute
        effect: NoSchedule
```

---

## CI/CD Pipeline Design

### 1. Pipeline Architecture & Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                          GitHub Actions CI/CD                       │
└─────────────────────────────────────────────────────────────────────┘

Developer Push to GitHub
    ↓
────────────────────────────────────────────────────────────────────
STAGE 1: Trigger & Context
────────────────────────────────────────────────────────────────────
  ✓ Checkout code
  ✓ Determine event (PR, push to main, release tag)
  ✓ Set environment variables & secrets
    ↓
────────────────────────────────────────────────────────────────────
STAGE 2: Code Quality & Security Scanning (Parallel)
────────────────────────────────────────────────────────────────────
  ✓ Lint & Format Check
    - ESLint (Frontend)
    - Black/isort/flake8 (Backend)
    - Terraform fmt (IaC)
  
  ✓ SAST (Static Application Security Testing)
    - Semgrep for Python/JavaScript
    - SonarQube for code quality
  
  ✓ Dependency Scanning
    - npm audit / pip-audit
    - Snyk for vulnerabilities
    - OWASP Dependency Check
  
  ✓ Secret Detection
    - TruffleHog / GitLeaks to prevent secrets in code
  
  ✓ License Compliance
    - FOSSA / license-checker
    ↓
────────────────────────────────────────────────────────────────────
STAGE 3: Build Artifacts (Parallel)
────────────────────────────────────────────────────────────────────
  ✓ Frontend Build
    - Build Next.js app: npm run build
    - Generate static export
    - Optimize bundle size
  
  ✓ Backend Build
    - Build Docker image: docker build -t legora-backend:$VERSION
    - Push to registry (ECR/GCR): docker push
  
  ✓ ML Model Preparation
    - Test model inference
    - Validate model compatibility
    - Build ML container
  
  ✓ Infrastructure Validation
    - Terraform validate
    - Terraform plan (for infra PRs)
    ↓
────────────────────────────────────────────────────────────────────
STAGE 4: Testing (Parallel)
────────────────────────────────────────────────────────────────────
  ✓ Unit Tests
    - Frontend: Jest
    - Backend: pytest
    - Coverage > 80%
  
  ✓ Integration Tests
    - API endpoint tests
    - Database integration
    - Cache integration
  
  ✓ Compliance & Security Tests
    - PII/PHI leak detection
    - Encryption verification
    - Audit log validation
  
  ✓ AI Model Tests
    - Model accuracy benchmarks
    - Inference latency checks
    - Robustness against adversarial inputs
  
  ✓ Load Testing (Staging only)
    - k6 or Apache JMeter
    - Verify performance under load
    ↓
────────────────────────────────────────────────────────────────────
STAGE 5: Approval Gates & Compliance Checks
────────────────────────────────────────────────────────────────────
  For production deployments:
  ✓ Manual approval from release manager
  ✓ Verify all tests passed
  ✓ Generate compliance report (SOC2/GDPR/HIPAA)
  ✓ Confirm change log updated
  ✓ Validate no breaking changes to API
    ↓
────────────────────────────────────────────────────────────────────
STAGE 6: Deploy to Target Environment
────────────────────────────────────────────────────────────────────
  
  If PR (Feature Branch):
    → Deploy to Vercel Preview Environment
    → Run smoke tests
    → Notify in PR comment
  
  If push to 'staging' branch:
    → Deploy backend to Staging EKS
    → Deploy frontend to Vercel Staging
    → Run e2e tests against staging
    → Notify Slack #deployments
  
  If release tag (v*):
    → Deploy backend to Production EKS (blue-green)
    → Deploy frontend to Vercel Production
    → Run smoke tests
    → Update DNS/routing if needed
    → Notify stakeholders
    ↓
────────────────────────────────────────────────────────────────────
STAGE 7: Post-Deployment Validation
────────────────────────────────────────────────────────────────────
  ✓ Smoke Tests (Health checks, key user flows)
  ✓ Performance Baseline (Compare to previous version)
  ✓ Error Rate Monitoring (Alert if > threshold)
  ✓ Compliance Verification (No policy violations)
  ✓ Audit Log Entry (Record deployment metadata)
    ↓
────────────────────────────────────────────────────────────────────
STAGE 8: Rollback (If issues detected)
────────────────────────────────────────────────────────────────────
  Automatic triggers:
    • Error rate > 5% for 5 minutes
    • P99 latency > 10x baseline
    • Health check failures
  
  Manual trigger:
    • On-call engineer initiates rollback
  
  Rollback process:
    → ArgoCD sync to previous stable version
    → Database migration rollback (if applicable)
    → DNS failover if needed
    → Incident created in PagerDuty
    → Team notified
```

### 2. GitHub Actions CI/CD Implementation

#### Main CI Workflow: `.github/workflows/ci.yml`

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - staging
      - develop
    tags:
      - "v*"
    paths:
      - "frontend/**"
      - "backend/**"
      - "infrastructure/**"
      - "ml/**"
      - "pyproject.toml"
      - "package.json"
  pull_request:
    branches:
      - main
      - staging
      - develop

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  REGISTRY: gcr.io
  REGISTRY_ALIAS: us-docker.pkg.dev
  PROJECT_ID: legora-os-prod
  IMAGE_NAME_BACKEND: legora-backend
  IMAGE_NAME_ML: legora-ml-service

jobs:
  # Job 1: Determine version and set outputs
  setup:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
      image_tag: ${{ steps.version.outputs.image_tag }}
      environment: ${{ steps.env.outputs.environment }}
      deploy_backend: ${{ steps.changes.outputs.backend }}
      deploy_frontend: ${{ steps.changes.outputs.frontend }}
      deploy_infra: ${{ steps.changes.outputs.infra }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Full history for versioning
      
      - name: Determine version
        id: version
        run: |
          if [[ "${{ github.ref }}" == refs/tags/* ]]; then
            VERSION=${GITHUB_REF#refs/tags/}
          else
            VERSION="${{ github.sha }}"
          fi
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "image_tag=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME_BACKEND }}:${VERSION}" >> $GITHUB_OUTPUT
      
      - name: Determine environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" || "${{ github.ref }}" == refs/tags/* ]]; then
            ENV="production"
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            ENV="staging"
          else
            ENV="development"
          fi
          echo "environment=${ENV}" >> $GITHUB_OUTPUT
      
      - name: Check changed files
        id: changes
        uses: dorny/paths-filter@v2
        with:
          filters: |
            backend:
              - 'backend/**'
              - 'pyproject.toml'
            frontend:
              - 'frontend/**'
              - 'package.json'
            infra:
              - 'infrastructure/**'

  # Job 2: Code Quality & Security (Frontend)
  lint-frontend:
    needs: setup
    if: needs.setup.outputs.deploy_frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: 'frontend/package-lock.json'
      
      - run: cd frontend && npm ci
      
      - name: Run ESLint
        run: cd frontend && npm run lint
      
      - name: Check format
        run: cd frontend && npm run format:check
      
      - name: Security audit
        run: cd frontend && npm audit --production
      
      - name: Dependency check
        run: cd frontend && npx snyk test --severity-threshold=high || true

  # Job 3: Code Quality & Security (Backend)
  lint-backend:
    needs: setup
    if: needs.setup.outputs.deploy_backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - run: |
          python -m pip install --upgrade pip
          pip install -r backend/requirements.txt
          pip install black isort flake8 bandit
      
      - name: Format check (Black)
        run: cd backend && black --check .
      
      - name: Import sorting (isort)
        run: cd backend && isort --check-only .
      
      - name: Lint (flake8)
        run: cd backend && flake8 . --max-line-length=100
      
      - name: Security scan (Bandit)
        run: cd backend && bandit -r . -ll
      
      - name: SAST (Semgrep)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/owasp-top-ten
          generateSarif: true
      
      - name: Upload SAST results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: semgrep.sarif

  # Job 4: Secret Detection
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: GitLeaks scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # Job 5: Build Frontend
  build-frontend:
    needs: [setup, lint-frontend]
    if: needs.setup.outputs.deploy_frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: 'frontend/package-lock.json'
      
      - run: cd frontend && npm ci
      
      - run: cd frontend && npm run build
      
      - name: Analyze bundle size
        run: cd frontend && npm run analyze:bundle || true
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: frontend-build
          path: frontend/.next/

  # Job 6: Build Backend
  build-backend:
    needs: [setup, lint-backend]
    if: needs.setup.outputs.deploy_backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to GCR
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: _json_key
          password: ${{ secrets.GCR_SA_KEY }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: ./backend
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ needs.setup.outputs.image_tag }}, ${{ needs.setup.outputs.image_tag_latest }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ needs.setup.outputs.version }}
            BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')

  # Job 7: Unit Tests (Backend)
  test-backend:
    needs: [setup, build-backend]
    if: needs.setup.outputs.deploy_backend == 'true'
    runs-on: ubuntu-latest
    services:
      mongo:
        image: mongo:6
        options: >-
          --health-cmd mongosh
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 27017:27017
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - run: |
          python -m pip install --upgrade pip
          pip install -r backend/requirements.txt
          pip install pytest pytest-cov pytest-asyncio
      
      - name: Run unit tests
        run: cd backend && pytest tests/ -v --cov=. --cov-report=xml --junitxml=results.xml
        env:
          MONGODB_URI: mongodb://localhost:27017/legora_test
          REDIS_URL: redis://localhost:6379/0
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./backend/coverage.xml
          flags: backend
          fail_ci_if_error: false

  # Job 8: Unit Tests (Frontend)
  test-frontend:
    needs: [setup, build-frontend]
    if: needs.setup.outputs.deploy_frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: 'frontend/package-lock.json'
      
      - run: cd frontend && npm ci
      
      - name: Run unit tests
        run: cd frontend && npm run test:unit -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./frontend/coverage/coverage-final.json
          flags: frontend

  # Job 9: Integration Tests
  integration-tests:
    needs: [setup, build-backend, build-frontend]
    if: needs.setup.outputs.environment != 'development'
    runs-on: ubuntu-latest
    services:
      mongo:
        image: mongo:6
        ports:
          - 27017:27017
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r backend/requirements.txt
          pip install pytest pytest-asyncio
          cd frontend && npm ci
      
      - name: Start backend
        run: cd backend && python -m uvicorn main:app --host 0.0.0.0 --port 8000 &
        env:
          MONGODB_URI: mongodb://localhost:27017/legora_test
          REDIS_URL: redis://localhost:6379/0
      
      - name: Wait for backend to start
        run: |
          for i in {1..30}; do
            curl -f http://localhost:8000/health && break
            sleep 1
          done
      
      - name: Run integration tests
        run: cd backend && pytest tests/integration/ -v
        env:
          API_URL: http://localhost:8000

  # Job 10: Compliance & Security Tests
  compliance-check:
    needs: [setup]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: GDPR compliance check
        run: |
          # Check for hardcoded PII patterns
          grep -r "\b\d{3}-\d{2}-\d{4}\b" backend/ frontend/ && exit 1 || true
          grep -r "password.*=.*['\"]" backend/ && exit 1 || true
      
      - name: License compliance
        uses: fossas/fossa-action@main
        with:
          api-key: ${{ secrets.FOSSA_API_KEY }}

  # Job 11: Infrastructure Validation
  validate-infrastructure:
    needs: [setup, secret-scan]
    if: needs.setup.outputs.deploy_infra == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Format Check
        run: cd infrastructure/terraform && terraform fmt -check -recursive
      
      - name: Terraform Init
        run: cd infrastructure/terraform/environments/${{ needs.setup.outputs.environment }} && terraform init -backend=false
      
      - name: Terraform Validate
        run: cd infrastructure/terraform/environments/${{ needs.setup.outputs.environment }} && terraform validate

  # Job 12: Deploy to Staging (if on staging branch)
  deploy-staging:
    needs: [setup, build-backend, build-frontend, test-backend, test-frontend, compliance-check]
    if: github.ref == 'refs/heads/staging'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
          aws-region: us-east-1
      
      - name: Update backend deployment
        run: |
          kubectl set image deployment/legora-backend \
            backend=${{ needs.setup.outputs.image_tag }} \
            -n staging
          kubectl rollout status deployment/legora-backend -n staging
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG_STAGING }}
      
      - name: Deploy frontend to Vercel
        run: |
          npm install -g vercel
          vercel --token=${{ secrets.VERCEL_TOKEN }} --prod --env=staging
        env:
          VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
          VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID_FRONTEND }}

  # Job 13: Deploy to Production (with approval)
  deploy-production:
    needs: [setup, build-backend, build-frontend, test-backend, test-frontend, compliance-check, validate-infrastructure]
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment: 
      name: production
      url: https://legora.legal
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
          aws-region: us-east-1
      
      - name: Blue-Green Deployment (Backend)
        run: |
          # Deploy to green environment
          kubectl set image deployment/legora-backend-green \
            backend=${{ needs.setup.outputs.image_tag }} \
            -n production
          
          # Run smoke tests
          kubectl run smoke-tests \
            --image=${{ needs.setup.outputs.image_tag }} \
            --rm -it \
            -- pytest tests/smoke/ \
            -n production
          
          # Switch traffic to green
          kubectl patch service legora-backend \
            -p '{"spec":{"selector":{"version":"green"}}}' \
            -n production
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG_PRODUCTION }}
      
      - name: Deploy frontend to Vercel
        run: |
          npm install -g vercel
          vercel --token=${{ secrets.VERCEL_TOKEN }} --prod
      
      - name: Create release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ needs.setup.outputs.version }}
          release_name: Release ${{ needs.setup.outputs.version }}
          draft: false
          prerelease: false
      
      - name: Notify deployment
        run: |
          curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"🚀 Legora OS deployed to production: ${{ needs.setup.outputs.version }}"}' \
            ${{ secrets.SLACK_WEBHOOK }}

  # Job 14: Rollback (if deployment fails)
  rollback:
    needs: [setup, deploy-production]
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Rollback deployment
        run: |
          kubectl patch service legora-backend \
            -p '{"spec":{"selector":{"version":"blue"}}}' \
            -n production
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG_PRODUCTION }}
      
      - name: Create incident
        run: |
          curl -X POST ${{ secrets.PAGERDUTY_WEBHOOK }} \
            -H 'Content-type: application/json' \
            --data '{"routing_key":"${{ secrets.PAGERDUTY_KEY }}","event_action":"trigger"}'
```

#### Deployment Strategy: `.github/workflows/deploy-prod.yml`

```yaml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy (e.g., v1.2.3)'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.version }}
      
      - name: Verify tag exists
        run: git show-ref --verify --quiet "refs/tags/${{ github.event.inputs.version }}"
      
      - name: Update Kubernetes deployments
        run: |
          # Update backend
          kubectl set image \
            deployment/legora-backend \
            backend=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/legora-backend:${{ github.event.inputs.version }} \
            -n production
          
          # Monitor rollout
          kubectl rollout status deployment/legora-backend -n production --timeout=10m
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG_PRODUCTION }}
      
      - name: Run smoke tests
        run: |
          for i in {1..5}; do
            curl -f https://api.legora.legal/health && break
            sleep 10
          done
```

---

## Observability & Security Stack

### 1. Comprehensive Observability Architecture

#### Metrics Collection (Prometheus)

```yaml
# infrastructure/helm/charts/monitoring/prometheus/values.yaml

prometheus:
  enabled: true
  
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    
    # Service monitors to scrape metrics
    serviceMonitorSelectorNilUsesHelmValues: false
    ruleSelector:
      matchLabels:
        prometheus: kube-prometheus
    
    # Resource limits
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 1000m
        memory: 4Gi

grafana:
  enabled: true
  
  adminPassword: $ADMIN_PASSWORD  # From Vault
  
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus:9090
        isDefault: true
      
      - name: Loki
        type: loki
        url: http://loki:3100
      
      - name: Jaeger
        type: jaeger
        url: http://jaeger-query:16686
      
      - name: Elasticsearch
        type: elasticsearch
        url: http://elasticsearch:9200
  
  dashboards:
    # Import predefined dashboards
    dashboardProviders:
      dashboardproviders.yaml:
        apiVersion: 1
        providers:
        - name: 'Legora Dashboards'
          orgId: 1
          folder: 'Legora'
          type: file
          disableDeletion: false
          editable: true
          options:
            path: /var/lib/grafana/dashboards/legora

alertmanager:
  enabled: true
  
  config:
    global:
      resolve_timeout: 5m
    
    route:
      receiver: 'default'
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      routes:
      - match:
          severity: critical
        receiver: 'critical'
        continue: true
      - match:
          severity: warning
        receiver: 'default'
    
    receivers:
    - name: 'default'
      slack_configs:
      - api_url: $SLACK_WEBHOOK_URL
        channel: '#alerts'
    
    - name: 'critical'
      slack_configs:
      - api_url: $SLACK_WEBHOOK_CRITICAL
        channel: '#critical-alerts'
      pagerduty_configs:
      - service_key: $PAGERDUTY_SERVICE_KEY

# PrometheusRule for alerting
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: legora-platform-alerts
spec:
  groups:
  - name: legora.rules
    interval: 30s
    rules:
    # Backend Service Alerts
    - alert: HighErrorRate
      expr: |
        (sum(rate(http_requests_total{job="legora-backend", status=~"5.."}[5m])) /
         sum(rate(http_requests_total{job="legora-backend"}[5m]))) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on {{ $labels.service }}"
        description: "Error rate is {{ $value | humanizePercentage }}"
    
    - alert: HighLatency
      expr: |
        histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High latency on {{ $labels.service }}"
    
    - alert: PodCrashLoop
      expr: |
        rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} in crash loop"
    
    - alert: HighMemoryUsage
      expr: |
        (container_memory_usage_bytes{pod=~"legora.*"} / 
         container_spec_memory_limit_bytes{pod=~"legora.*"}) > 0.9
      for: 5m
      labels:
        severity: warning
    
    - alert: DatabaseConnectionPoolExhausted
      expr: |
        mongodb_connections{state="current"} > 800
      for: 5m
      labels:
        severity: critical
    
    - alert: UnusuallyHighAPILatency
      expr: |
        histogram_quantile(0.95, rate(legora_document_processing_duration_seconds_bucket[5m])) > 10
      for: 5m
      labels:
        severity: warning
```

#### Distributed Tracing (Jaeger)

```python
# backend/observability/tracing.py

from fastapi import FastAPI, Request
from opentelemetry import trace, metrics
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.pymongo import PyMongoInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.prometheus import PrometheusMetricReader
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
import os

# Configure Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name=os.getenv("JAEGER_AGENT_HOST", "localhost"),
    agent_port=int(os.getenv("JAEGER_AGENT_PORT", 6831)),
)

# Create tracer provider
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Configure metrics
metric_reader = PeriodicExportingMetricReader(
    OTLPMetricExporter(endpoint=os.getenv("OTLP_EXPORTER_ENDPOINT", "http://localhost:4317"))
)
metrics.set_meter_provider(MeterProvider(metric_readers=[metric_reader]))

def setup_tracing(app: FastAPI):
    """Setup OpenTelemetry instrumentation for FastAPI app"""
    
    # Instrument FastAPI
    FastAPIInstrumentor.instrument_app(app)
    
    # Instrument databases
    PyMongoInstrumentor().instrument()
    RedisInstrumentor().instrument()
    
    # Instrument HTTP requests
    RequestsInstrumentor().instrument()
    
    # Custom middleware for request tracking
    @app.middleware("http")
    async def add_trace_context(request: Request, call_next):
        tracer = trace.get_tracer(__name__)
        
        with tracer.start_as_current_span(f"{request.method} {request.url.path}") as span:
            span.set_attribute("http.method", request.method)
            span.set_attribute("http.url", str(request.url))
            span.set_attribute("http.client_ip", request.client.host)
            
            # Add request ID for correlation
            request_id = request.headers.get("X-Request-ID", "")
            if request_id:
                span.set_attribute("request_id", request_id)
            
            response = await call_next(request)
            
            span.set_attribute("http.status_code", response.status_code)
            return response
```

#### Centralized Logging (ELK Stack + Loki)

```yaml
# infrastructure/helm/charts/monitoring/loki/values.yaml

loki:
  enabled: true
  
  config:
    auth_enabled: false
    
    ingester:
      chunk_idle_period: 5m
      max_chunk_age: 1h
      chunk_retain_period: 1m
    
    limits_config:
      enforce_metric_name: false
      reject_old_samples: true
      reject_old_samples_max_age: 168h
    
    schema_config:
      configs:
      - from: 2024-01-01
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: index_
          period: 24h
    
    server:
      http_listen_port: 3100
      grpc_listen_port: 9096
  
  persistence:
    enabled: true
    size: 50Gi
    storageClassName: gp3

promtail:
  enabled: true
  
  config:
    clients:
    - url: http://loki:3100/loki/api/v1/push
    
    scrape_configs:
    # Scrape Kubernetes logs
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_name]
        action: keep
        regex: legora.*
      
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: container
```

#### Audit Logging (Compliance-Focused)

```python
# backend/observability/audit_logger.py

import json
import logging
from datetime import datetime
from typing import Any, Dict
from enum import Enum

class AuditEventType(str, Enum):
    """Audit event types for compliance"""
    AUTH_LOGIN = "auth.login"
    AUTH_LOGOUT = "auth.logout"
    AUTH_FAILED = "auth.failed"
    DOCUMENT_UPLOAD = "document.upload"
    DOCUMENT_VIEW = "document.view"
    DOCUMENT_EXPORT = "document.export"
    DOCUMENT_DELETE = "document.delete"
    MODEL_INFERENCE = "model.inference"
    USER_CREATED = "user.created"
    USER_MODIFIED = "user.modified"
    USER_DELETED = "user.deleted"
    CONFIG_CHANGED = "config.changed"
    POLICY_VIOLATION = "policy.violation"
    DATA_EXPORT = "data.export"
    SECRET_ACCESS = "secret.access"

class AuditLogger:
    """Central audit logging for compliance (GDPR, SOC2, HIPAA)"""
    
    def __init__(self):
        # Separate audit logger to prevent mixing with application logs
        self.logger = logging.getLogger("audit")
        self.logger.setLevel(logging.INFO)
        
        # File handler for audit logs (tamper-proof storage recommended)
        handler = logging.FileHandler("/var/log/legora/audit.log")
        handler.setFormatter(logging.Formatter(
            '{"timestamp": "%(asctime)s", "level": "%(levelname)s", "message": "%(message)s"}'
        ))
        self.logger.addHandler(handler)
    
    def log_event(
        self,
        event_type: AuditEventType,
        user_id: str,
        resource: str,
        action: str,
        status: str,
        metadata: Dict[str, Any] = None,
        ip_address: str = None,
        user_agent: str = None,
    ):
        """Log audit event with compliance details"""
        
        event = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "event_type": event_type.value,
            "user_id": user_id,
            "resource": resource,
            "action": action,
            "status": status,
            "ip_address": ip_address,
            "user_agent": user_agent[:200] if user_agent else None,  # Truncate for PII
            "metadata": metadata or {},
        }
        
        # Never log sensitive data directly
        if "password" in event.get("metadata", {}):
            del event["metadata"]["password"]
        if "token" in event.get("metadata", {}):
            del event["metadata"]["token"]
        
        self.logger.info(json.dumps(event))
    
    def log_policy_violation(
        self,
        violation_type: str,
        severity: str,
        description: str,
        user_id: str = None,
    ):
        """Log policy violations for compliance audits"""
        
        event = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "event_type": AuditEventType.POLICY_VIOLATION.value,
            "violation_type": violation_type,
            "severity": severity,
            "description": description,
            "user_id": user_id,
        }
        
        self.logger.warning(json.dumps(event))

# Global audit logger instance
audit_logger = AuditLogger()

# Usage in FastAPI routes
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.security import HTTPBearer

app = FastAPI()
security = HTTPBearer()

@app.post("/documents/{doc_id}/view")
async def view_document(
    doc_id: str,
    request: Request,
    credentials = Depends(security),
):
    """View a legal document"""
    
    user_id = credentials.credentials  # From JWT token
    
    try:
        # Load and return document
        document = get_document(doc_id)
        
        # Log audit event
        audit_logger.log_event(
            event_type=AuditEventType.DOCUMENT_VIEW,
            user_id=user_id,
            resource=f"document/{doc_id}",
            action="view",
            status="success",
            ip_address=request.client.host,
            user_agent=request.headers.get("user-agent"),
            metadata={
                "document_type": document.type,
                "classification": document.classification,
            }
        )
        
        return document
    
    except Exception as e:
        audit_logger.log_event(
            event_type=AuditEventType.DOCUMENT_VIEW,
            user_id=user_id,
            resource=f"document/{doc_id}",
            action="view",
            status="failed",
            ip_address=request.client.host,
            metadata={"error": str(e)}
        )
        raise HTTPException(status_code=403, detail="Access denied")
```

### 2. Security Stack

#### Secret Management & Rotation

```python
# backend/security/secrets.py

import os
import hvac
from datetime import datetime, timedelta
from typing import Dict, Any
from functools import lru_cache
import logging

logger = logging.getLogger(__name__)

class SecretsManager:
    """Centralized secrets management with HashiCorp Vault"""
    
    def __init__(self):
        self.vault_addr = os.getenv("VAULT_ADDR", "http://vault:8200")
        self.vault_namespace = os.getenv("VAULT_NAMESPACE", "legora")
        
        # Use Kubernetes auth in production, token in dev
        if os.getenv("ENVIRONMENT") == "production":
            jwt_token = open("/var/run/secrets/kubernetes.io/serviceaccount/token").read()
            self.client = hvac.Client(
                url=self.vault_addr,
                namespace=self.vault_namespace,
            )
            self.client.auth.kubernetes.login(
                role="legora-backend",
                jwt=jwt_token,
            )
        else:
            self.client = hvac.Client(
                url=self.vault_addr,
                token=os.getenv("VAULT_TOKEN"),
                namespace=self.vault_namespace,
            )
        
        self._cache: Dict[str, Dict[str, Any]] = {}
        self._cache_time: Dict[str, datetime] = {}
        self._cache_ttl = timedelta(minutes=5)
    
    def get_secret(self, secret_path: str, refresh: bool = False) -> Dict[str, Any]:
        """Retrieve secret from Vault with caching"""
        
        # Check cache
        if not refresh and secret_path in self._cache:
            if datetime.now() - self._cache_time[secret_path] < self._cache_ttl:
                logger.info(f"Retrieving {secret_path} from cache")
                return self._cache[secret_path]
        
        # Fetch from Vault
        try:
            response = self.client.secrets.kv.v2.read_secret_version(path=secret_path)
            secret_data = response["data"]["data"]
            
            # Cache the result
            self._cache[secret_path] = secret_data
            self._cache_time[secret_path] = datetime.now()
            
            logger.info(f"Retrieved secret from Vault: {secret_path}")
            return secret_data
        
        except Exception as e:
            logger.error(f"Failed to retrieve secret {secret_path}: {e}")
            
            # Return cached value if available (stale but better than failure)
            if secret_path in self._cache:
                logger.warning(f"Using stale cached secret for {secret_path}")
                return self._cache[secret_path]
            
            raise
    
    def rotate_secret(self, secret_path: str, new_value: Dict[str, Any]):
        """Rotate a secret (e.g., database passwords, API keys)"""
        
        try:
            self.client.secrets.kv.v2.create_or_update_secret(
                path=secret_path,
                secret_data=new_value,
            )
            
            # Invalidate cache
            if secret_path in self._cache:
                del self._cache[secret_path]
                del self._cache_time[secret_path]
            
            logger.info(f"Successfully rotated secret: {secret_path}")
        
        except Exception as e:
            logger.error(f"Failed to rotate secret {secret_path}: {e}")
            raise

# Singleton instance
_secrets_manager = None

def get_secrets_manager() -> SecretsManager:
    global _secrets_manager
    if _secrets_manager is None:
        _secrets_manager = SecretsManager()
    return _secrets_manager

# Usage in FastAPI
from fastapi import Depends

async def get_db_connection_string(sm: SecretsManager = Depends(get_secrets_manager)):
    """Dependency to get database connection string"""
    secrets = sm.get_secret("database/mongo")
    return secrets["connection_string"]
```

#### Encryption at Rest & In Transit

```python
# backend/security/encryption.py

from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import os
import base64
from typing import Any
import json

class EncryptionManager:
    """End-to-end encryption for sensitive documents"""
    
    def __init__(self, master_key: str = None):
        # Use Vault-managed key in production
        if master_key is None:
            master_key = os.getenv("MASTER_ENCRYPTION_KEY", "")
        
        # Derive encryption key from master key
        if not master_key:
            raise ValueError("MASTER_ENCRYPTION_KEY not set")
        
        # Generate key using PBKDF2
        salt = b'legora_salt_v1'  # In production, use random salt from Vault
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(master_key.encode()))
        self.cipher = Fernet(key)
    
    def encrypt_document(self, document_content: bytes) -> bytes:
        """Encrypt document content"""
        return self.cipher.encrypt(document_content)
    
    def decrypt_document(self, encrypted_content: bytes) -> bytes:
        """Decrypt document content"""
        return self.cipher.decrypt(encrypted_content)
    
    def encrypt_pii(self, pii_data: str) -> str:
        """Encrypt PII field (e.g., email, phone)"""
        encrypted = self.cipher.encrypt(pii_data.encode())
        return encrypted.decode()
    
    def decrypt_pii(self, encrypted_pii: str) -> str:
        """Decrypt PII field"""
        decrypted = self.cipher.decrypt(encrypted_pii.encode())
        return decrypted.decode()

# MongoDB Document with encrypted fields
from pymongo import MongoClient
from pydantic import BaseModel

class LegalDocument(BaseModel):
    title: str
    content: bytes  # Will be encrypted before storage
    client_email: str  # PII - will be encrypted
    attorney_name: str  # PII - will be encrypted

async def store_document(document: LegalDocument, em: EncryptionManager):
    """Store document with encryption"""
    
    client = MongoClient(os.getenv("MONGODB_URI"))
    db = client.legora
    collection = db.documents
    
    doc = {
        "title": document.title,
        "content": em.encrypt_document(document.content),  # Encrypted at rest
        "client_email": em.encrypt_pii(document.client_email),  # Encrypted PII
        "attorney_name": em.encrypt_pii(document.attorney_name),  # Encrypted PII
        "created_at": datetime.utcnow(),
    }
    
    result = collection.insert_one(doc)
    return str(result.inserted_id)
```

#### RBAC & Policy Enforcement

```python
# backend/security/rbac.py

from enum import Enum
from typing import List, Optional
from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthCredentials
import jwt
import os
from datetime import datetime, timedelta

class Role(str, Enum):
    ADMIN = "admin"
    ATTORNEY = "attorney"
    PARALEGAL = "paralegal"
    CLIENT = "client"
    ANALYST = "analyst"
    AUDITOR = "auditor"

class Permission(str, Enum):
    """Fine-grained permissions"""
    UPLOAD_DOCUMENT = "upload_document"
    VIEW_DOCUMENT = "view_document"
    EXPORT_DOCUMENT = "export_document"
    DELETE_DOCUMENT = "delete_document"
    RUN_ANALYSIS = "run_analysis"
    MANAGE_USERS = "manage_users"
    VIEW_AUDIT_LOG = "view_audit_log"
    CONFIGURE_SYSTEM = "configure_system"

# Role-to-Permission mapping
ROLE_PERMISSIONS = {
    Role.ADMIN: [perm.value for perm in Permission],  # All permissions
    Role.ATTORNEY: [
        Permission.UPLOAD_DOCUMENT.value,
        Permission.VIEW_DOCUMENT.value,
        Permission.EXPORT_DOCUMENT.value,
        Permission.DELETE_DOCUMENT.value,
        Permission.RUN_ANALYSIS.value,
    ],
    Role.PARALEGAL: [
        Permission.UPLOAD_DOCUMENT.value,
        Permission.VIEW_DOCUMENT.value,
        Permission.RUN_ANALYSIS.value,
    ],
    Role.CLIENT: [
        Permission.VIEW_DOCUMENT.value,
    ],
    Role.ANALYST: [
        Permission.VIEW_DOCUMENT.value,
        Permission.RUN_ANALYSIS.value,
        Permission.VIEW_AUDIT_LOG.value,
    ],
    Role.AUDITOR: [
        Permission.VIEW_AUDIT_LOG.value,
    ],
}

class User:
    def __init__(self, user_id: str, role: Role, email: str):
        self.user_id = user_id
        self.role = role
        self.email = email
        self.permissions = ROLE_PERMISSIONS[role]

security = HTTPBearer()

async def get_current_user(credentials: HTTPAuthCredentials = Depends(security)) -> User:
    """Authenticate user and return User object"""
    try:
        token = credentials.credentials
        payload = jwt.decode(
            token,
            os.getenv("JWT_SECRET_KEY"),
            algorithms=["HS256"]
        )
        user_id = payload.get("sub")
        role = Role(payload.get("role", "client"))
        email = payload.get("email")
        
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        
        return User(user_id=user_id, role=role, email=email)
    
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

def require_permission(permission: Permission):
    """Dependency for permission-based access control"""
    async def check_permission(user: User = Depends(get_current_user)):
        if permission.value not in user.permissions:
            raise HTTPException(status_code=403, detail="Permission denied")
        return user
    return check_permission

# Usage in FastAPI routes
from fastapi import FastAPI

app = FastAPI()

@app.post("/documents/{doc_id}/export")
async def export_document(
    doc_id: str,
    user: User = Depends(require_permission(Permission.EXPORT_DOCUMENT))
):
    """Export document (attorney only)"""
    # User has required permission here
    return {"status": "exported"}

@app.delete("/users/{user_id}")
async def delete_user(
    user_id: str,
    user: User = Depends(require_permission(Permission.MANAGE_USERS))
):
    """Delete user (admin only)"""
    return {"status": "deleted"}
```

#### Vulnerability Scanning & Compliance

```yaml
# .github/workflows/security-scan.yml

name: Security & Vulnerability Scanning

on:
  push:
    branches: [main, staging]
  schedule:
    # Daily security scans
    - cron: '0 2 * * *'

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: OWASP Dependency-Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'Legora OS'
          path: '.'
          format: 'SARIF'
          args: >
            --enableExperimental
            --enable-retired
      
      - name: Upload results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'dependency-check-report.sarif'

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Scan container images
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy image --severity HIGH,CRITICAL gcr.io/${{ secrets.GCP_PROJECT_ID }}/legora-backend:latest
```

---

## Developer Experience & Self-Service Catalog

### 1. CLI Tooling (`legora-cli`)

```python
# cli/legora_cli.py
"""
Legora OS Developer CLI
Simplifies common development tasks: environment setup, deployment, debugging
"""

import click
import subprocess
import os
import sys
from pathlib import Path
from typing import Optional
import yaml
import json

class LegoraContext:
    """CLI context for managing configuration"""
    def __init__(self):
        self.config_dir = Path.home() / ".legora"
        self.config_file = self.config_dir / "config.yaml"
        self.config = self._load_config()
    
    def _load_config(self) -> dict:
        if self.config_file.exists():
            with open(self.config_file) as f:
                return yaml.safe_load(f)
        return {"environments": {}, "recent_env": None}
    
    def save_config(self):
        self.config_dir.mkdir(exist_ok=True)
        with open(self.config_file, 'w') as f:
            yaml.dump(self.config, f)

@click.group()
@click.pass_context
def cli(ctx):
    """Legora OS Developer Platform CLI"""
    ctx.ensure_object(dict)
    ctx.obj['context'] = LegoraContext()

@cli.group()
def env():
    """Environment management commands"""
    pass

@env.command()
@click.option('--name', prompt='Environment name', help='e.g., staging, production')
@click.option('--type', type=click.Choice(['local', 'staging', 'production']), prompt='Environment type')
@click.pass_context
def setup(ctx, name, type):
    """Set up a new development environment"""
    click.echo(f"Setting up {type} environment: {name}")
    
    legora_ctx = ctx.obj['context']
    
    if type == 'local':
        # Start Docker Compose
        click.echo("Starting Docker Compose services...")
        subprocess.run(["docker-compose", "up", "-d"], check=True)
        
        # Initialize databases
        click.echo("Initializing databases...")
        subprocess.run(["docker-compose", "exec", "mongo", "mongosh", "--eval", "db.createCollection('documents')"], check=False)
        
        # Load sample data
        if click.confirm("Load sample data?"):
            subprocess.run(["./scripts/load-sample-data.sh"], check=True)
    
    elif type == 'staging':
        # Connect to staging cluster
        click.echo("Configuring kubectl for staging...")
        subprocess.run([
            "aws", "eks", "update-kubeconfig",
            "--name", "legora-staging",
            "--region", "us-east-1"
        ], check=True)
        
        # Create namespace if needed
        subprocess.run(["kubectl", "create", "namespace", name, "--dry-run=client", "-o", "yaml", "|", "kubectl", "apply", "-f", "-"], check=False)
    
    # Save config
    legora_ctx.config['environments'][name] = {
        'type': type,
        'created_at': str(Path.ctime(Path.cwd()))
    }
    legora_ctx.config['recent_env'] = name
    legora_ctx.save_config()
    
    click.echo(f"✅ Environment {name} ready!")

@env.command()
@click.option('--name', help='Environment name')
@click.pass_context
def activate(ctx, name):
    """Activate an environment"""
    legora_ctx = ctx.obj['context']
    
    if name not in legora_ctx.config['environments']:
        click.echo(f"❌ Environment {name} not found")
        sys.exit(1)
    
    legora_ctx.config['recent_env'] = name
    legora_ctx.save_config()
    
    # Set environment variables
    env_type = legora_ctx.config['environments'][name]['type']
    if env_type == 'local':
        os.environ['ENVIRONMENT'] = 'development'
    elif env_type == 'staging':
        os.environ['ENVIRONMENT'] = 'staging'
    
    click.echo(f"✅ Activated environment: {name}")

@cli.group()
def deploy():
    """Deployment commands"""
    pass

@deploy.command()
@click.option('--environment', type=click.Choice(['staging', 'production']), required=True)
@click.option('--version', help='Version to deploy (default: latest)')
@click.option('--components', multiple=True, type=click.Choice(['frontend', 'backend', 'ml']), help='Components to deploy')
@click.pass_context
def rollout(ctx, environment, version, components):
    """Deploy to an environment"""
    click.echo(f"Deploying {components or 'all'} to {environment}...")
    
    # Determine version
    if not version:
        result = subprocess.run(["git", "describe", "--tags"], capture_output=True, text=True)
        version = result.stdout.strip() or "latest"
    
    # Trigger GitHub Actions workflow
    payload = {
        "environment": environment,
        "version": version,
        "components": components or ["frontend", "backend"]
    }
    
    cmd = [
        "gh", "workflow", "run", "deploy.yml",
        "-f", f"environment={environment}",
        "-f", f"version={version}"
    ]
    
    if subprocess.run(cmd, check=True):
        click.echo(f"✅ Deployment workflow triggered for {environment}")

@deploy.command()
@click.option('--environment', type=click.Choice(['staging', 'production']), required=True)
@click.pass_context
def rollback(ctx, environment):
    """Rollback to previous version"""
    if not click.confirm(f"Are you sure you want to rollback {environment}?"):
        return
    
    click.echo(f"Rolling back {environment}...")
    subprocess.run([
        "kubectl", "rollout", "undo", "deployment/legora-backend",
        "-n", environment
    ], check=True)
    
    click.echo(f"✅ Rollback complete")

@cli.group()
def test():
    """Testing commands"""
    pass

@test.command()
@click.option('--type', type=click.Choice(['unit', 'integration', 'e2e']), default='unit')
@click.option('--coverage', is_flag=True, help='Generate coverage report')
@click.pass_context
def run(ctx, type, coverage):
    """Run tests"""
    click.echo(f"Running {type} tests...")
    
    cmd = []
    if type == 'unit':
        cmd = ["pytest", "tests/unit", "-v"]
        if coverage:
            cmd.extend(["--cov", "--cov-report=html"])
    
    elif type == 'integration':
        cmd = ["pytest", "tests/integration", "-v"]
    
    elif type == 'e2e':
        cmd = ["npx", "playwright", "test", "--ui"]
    
    subprocess.run(cmd, check=False)

@cli.group()
def debug():
    """Debugging commands"""
    pass

@debug.command()
@click.option('--service', type=click.Choice(['backend', 'frontend', 'mongo']), required=True)
@click.pass_context
def logs(ctx, service):
    """Stream logs from a service"""
    if service == 'backend':
        subprocess.run(["docker-compose", "logs", "-f", "backend"])
    elif service == 'frontend':
        subprocess.run(["npm", "run", "dev"])
    elif service == 'mongo':
        subprocess.run(["docker-compose", "logs", "-f", "mongo"])

@debug.command()
@click.option('--service', type=click.Choice(['backend', 'frontend']), required=True)
@click.option('--port', type=int, default=None)
@click.pass_context
def attach(ctx, service, port):
    """Attach debugger to a service"""
    if service == 'backend':
        port = port or 5678
        click.echo(f"Attaching debugger to backend on port {port}...")
        # Instructions for IDE debugger
        click.echo(f"In your IDE, set up a Python debugger listening on localhost:{port}")
    elif service == 'frontend':
        port = port or 9229
        click.echo(f"Chrome DevTools available at chrome://devtools/?ws=localhost:{port}")

if __name__ == '__main__':
    cli(obj={})
```

### 2. Self-Service Developer Portal

```python
# platform/service_catalog.py
"""
Internal Developer Platform (IDP) - Self-service catalog
Exposes templates, blueprints, and common patterns
"""

from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import List, Dict, Optional
import yaml
from pathlib import Path

app = FastAPI(title="Legora OS Developer Portal")

# Data models
class Template(BaseModel):
    name: str
    description: str
    category: str  # e.g., "microservice", "data-pipeline", "ml-service"
    repository: str
    docs_url: str
    tags: List[str]

class Blueprint(BaseModel):
    name: str
    description: str
    template: Template
    parameters: Dict[str, str]
    estimated_cost: str

class DevEnvironment(BaseModel):
    name: str
    type: str  # local, staging, production
    status: str  # active, paused, inactive
    last_accessed: str

# Load templates from repo
def load_templates() -> List[Template]:
    """Load templates from catalog directory"""
    templates = []
    catalog_dir = Path("platform/templates")
    
    for template_file in catalog_dir.glob("*/template.yaml"):
        with open(template_file) as f:
            data = yaml.safe_load(f)
            templates.append(Template(**data))
    
    return templates

# API Endpoints
@app.get("/templates", response_model=List[Template])
async def list_templates(category: Optional[str] = None):
    """List available templates"""
    templates = load_templates()
    
    if category:
        templates = [t for t in templates if t.category == category]
    
    return templates

@app.get("/templates/{template_id}")
async def get_template(template_id: str):
    """Get template details and setup instructions"""
    templates = load_templates()
    template = next((t for t in templates if t.name == template_id), None)
    
    if not template:
        raise HTTPException(status_code=404, detail="Template not found")
    
    return {
        "template": template,
        "setup_steps": [
            "Clone template repository",
            "Configure environment variables",
            "Run docker-compose up",
            "Access service at localhost:8000"
        ]
    }

@app.post("/blueprints/create")
async def create_blueprint(name: str, template_id: str, parameters: Dict[str, str]):
    """Create a new project from template"""
    
    templates = load_templates()
    template = next((t for t in templates if t.name == template_id), None)
    
    if not template:
        raise HTTPException(status_code=404, detail="Template not found")
    
    # Clone repository with parameters
    # This would typically trigger a GitHub Actions workflow
    
    return {
        "status": "created",
        "blueprint_name": name,
        "template": template_id,
        "repository_url": f"https://github.com/legora/project-{name}"
    }

@app.get("/environments")
async def list_dev_environments(user_id: str):
    """List user's development environments"""
    
    # Query database or Kubernetes API
    environments = [
        DevEnvironment(
            name="local-dev",
            type="local",
            status="active",
            last_accessed="2024-08-15T10:30:00Z"
        ),
        DevEnvironment(
            name="feature-ai-v2",
            type="staging",
            status="active",
            last_accessed="2024-08-15T09:00:00Z"
        )
    ]
    
    return environments

@app.post("/environments/create")
async def create_preview_environment(
    branch_name: str,
    template: str,
    ttl_hours: int = 24
):
    """Create ephemeral preview environment for feature branch"""
    
    # Trigger Kubernetes deployment with TTL
    # This creates a short-lived namespace for testing
    
    return {
        "environment_name": f"preview-{branch_name}",
        "namespace": f"preview-{branch_name}",
        "url": f"https://{branch_name}.preview.legora.local",
        "expires_at": "2024-08-16T10:30:00Z"
    }
```

### 3. Component Library & Documentation

```markdown
# Legora OS Component Library

## Overview
Reusable, production-tested components for frontend and backend teams.

## Frontend Components (React/Next.js)

### DocumentUploader
Upload and preview legal documents with validation.

```typescript
import { DocumentUploader } from "@legora/components";

export function MyComponent() {
  return (
    <DocumentUploader
      maxSize={100} // MB
      acceptedFormats={[".pdf", ".docx", ".txt"]}
      onUpload={(file) => console.log(file)}
      onError={(error) => console.error(error)}
    />
  );
}
```

### LegalAnalysisPanel
Display AI-generated legal analysis results.

```typescript
import { LegalAnalysisPanel } from "@legora/components";

export function AnalysisView({ documentId }) {
  return (
    <LegalAnalysisPanel
      documentId={documentId}
      displayMode="detailed" // or "summary"
      highlightClauses={true}
    />
  );
}
```

## Backend Utilities (Python/FastAPI)

### DocumentProcessor
Process uploaded documents for analysis.

```python
from legora.services import DocumentProcessor

processor = DocumentProcessor()

# Process document
result = await processor.analyze_document(
    file_path="/path/to/document.pdf",
    analysis_type="contract_review",
    extract_clauses=True
)

print(result.clauses)  # Extracted clauses
print(result.risk_score)  # Risk assessment
```

## Data Models & Schemas

### LegalDocument Schema
```yaml
LegalDocument:
  type: object
  properties:
    id:
      type: string
      description: Unique document identifier
    title:
      type: string
    content:
      type: string
      description: Encrypted document content
    metadata:
      type: object
      properties:
        document_type:
          enum: [contract, agreement, patent, motion, other]
        classification:
          enum: [public, confidential, highly-confidential]
        created_date:
          type: string
          format: date-time
```

## Testing Patterns

### Unit Test Template

```python
# tests/test_document_processor.py

import pytest
from legora.services import DocumentProcessor

@pytest.fixture
def processor():
    return DocumentProcessor()

@pytest.mark.asyncio
async def test_analyze_contract(processor):
    result = await processor.analyze_document(
        file_path="tests/fixtures/sample_contract.pdf",
        analysis_type="contract_review"
    )
    
    assert result.document_type == "contract"
    assert len(result.clauses) > 0
    assert 0 <= result.risk_score <= 100
```

## CI/CD Integration

### GitOps Pattern
```yaml
# infrastructure/helm/values.yaml

documentProcessor:
  image: gcr.io/legora-os/backend:v1.2.3
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
```

---

## Implementation Roadmap

### Phase 1: Foundation (Months 1-2)
**Goal**: Establish baseline infrastructure and observability

**Quick Wins:**
- [ ] Set up Terraform for prod/staging infrastructure
- [ ] Implement basic CI/CD pipeline (GitHub Actions)
- [ ] Deploy Prometheus + Grafana for metrics
- [ ] Configure HashiCorp Vault for secrets management
- [ ] Establish baseline audit logging

**Deliverables:**
- Multi-environment Terraform code (prod/staging/dev)
- GitHub Actions workflow for basic CI/CD
- Prometheus dashboards for key metrics
- Vault integration for all services
- Audit logger implementation

**Effort**: 8 weeks | **Team**: 2-3 engineers (Platform, DevOps, Security)

---

### Phase 2: Developer Experience (Months 3-4)
**Goal**: Enable self-service for development teams

**Quick Wins:**
- [ ] Publish legora-cli tool (npm/pip)
- [ ] Create developer portal with template catalog
- [ ] Implement PR preview environments (Vercel)
- [ ] Establish documentation standards
- [ ] Set up internal component library

**Deliverables:**
- legora-cli with environment setup, deploy, and debug commands
- Internal developer portal (web UI)
- Verified PR preview environments
- Component library (React + Python)
- Developer onboarding documentation

**Effort**: 8 weeks | **Team**: 2 engineers (Platform, DevX)

---

### Phase 3: Security & Compliance (Months 5-6)
**Goal**: Production-grade security posture

**Quick Wins:**
- [ ] Implement encryption at rest and in transit
- [ ] Set up RBAC and fine-grained permissions
- [ ] Automate compliance scanning (SAST, dependency check)
- [ ] Configure PII/PHI detection and handling
- [ ] Establish incident response procedures

**Deliverables:**
- End-to-end encryption for documents
- Role-based access control (RBAC) with policies
- Automated security scanning in CI/CD
- PII/PHI detection and masking
- Incident response playbooks
- SOC2 Type II compliance documentation

**Effort**: 10 weeks | **Team**: 2-3 engineers (Security, Platform)

---

### Phase 4: Scalability & ML Ops (Months 7-9)
**Goal**: Support AI/ML workflows and high-scale operations

**Quick Wins:**
- [ ] Set up ML model registry and versioning (MLflow)
- [ ] Implement model serving pipeline (KServe or BentoML)
- [ ] Establish A/B testing framework
- [ ] Configure distributed tracing (Jaeger)
- [ ] Set up chaos engineering for resilience

**Deliverables:**
- MLflow or Hugging Face model registry
- Model serving infrastructure (Kubernetes)
- A/B testing framework for model experiments
- Jaeger distributed tracing
- Chaos engineering experiments
- Performance baselines and SLOs

**Effort**: 12 weeks | **Team**: 3-4 engineers (ML Ops, Platform)

---

### Phase 5: Cost Optimization & Operations (Months 10-12)
**Goal**: Optimize costs and operational excellence

**Quick Wins:**
- [ ] Implement cost monitoring (AWS Cost Explorer / GCP Billing)
- [ ] Auto-scaling policies based on workload
- [ ] Reserved instances and spot instances
- [ ] Data lifecycle management (archive old documents)
- [ ] FinOps reporting to leadership

**Deliverables:**
- Cost dashboards and alerts
- Optimized auto-scaling policies
- Reserved instance strategy
- Data retention and archival policies
- Monthly FinOps reports
- Cost optimization runbooks

**Effort**: 8 weeks | **Team**: 1-2 engineers (Platform, FinOps)

---

## Success Metrics & KPIs

### Platform Reliability
- **Uptime SLA**: 99.95% (Production)
- **Deployment Success Rate**: > 98%
- **Mean Time to Recovery (MTTR)**: < 15 minutes
- **Mean Time Between Failures (MTBF)**: > 30 days

### Developer Productivity
- **Environment Setup Time**: < 15 minutes (from `legora-cli init`)
- **PR to Deployment**: < 30 minutes (including reviews)
- **Developer Self-Service Rate**: > 80% (no platform team involvement)
- **Feature Velocity**: 2x increase in features shipped per sprint

### Security & Compliance
- **Zero Critical Vulnerabilities**: In production
- **Audit Log Completeness**: 100% of user actions logged
- **Secret Rotation Frequency**: 90 days
- **Compliance Audit Pass Rate**: 100%

### Cost Efficiency
- **Cost per API Request**: < $0.001
- **Compute Utilization**: > 70%
- **Storage Cost Reduction**: 30% YoY through archival

---

## Checklists & Quick Reference

### Pre-Production Checklist

**Infrastructure**
- [ ] Terraform code deployed to prod environment
- [ ] Database backups configured (90-day retention)
- [ ] DNS and SSL/TLS configured
- [ ] CDN enabled for static assets
- [ ] VPC security groups configured
- [ ] S3 buckets encrypted and versioned

**CI/CD**
- [ ] GitHub Actions workflows tested end-to-end
- [ ] Automatic rollback configured
- [ ] Deployment approvals enforced
- [ ] Blue-green deployment tested
- [ ] Smoke tests passing on all environments

**Security**
- [ ] Vault initialized with prod secrets
- [ ] RBAC roles defined and tested
- [ ] Encryption keys rotated
- [ ] SSL certificates valid
- [ ] Dependency scan results reviewed
- [ ] SAST scan results reviewed

**Observability**
- [ ] Prometheus scrape targets configured
- [ ] Grafana dashboards created
- [ ] Alert rules configured
- [ ] Audit logging verified
- [ ] ELK stack deployed
- [ ] Distributed tracing enabled

**Documentation**
- [ ] Architecture documentation complete
- [ ] Deployment runbook created
- [ ] Disaster recovery plan documented
- [ ] On-call procedures documented
- [ ] Developer guide published

---

## Recommended Tools & Services

### Cloud Infrastructure
- **Compute**: AWS EKS / GCP GKE for Kubernetes
- **Storage**: S3 / GCS for object storage, MongoDB Atlas for database
- **Cache**: ElastiCache Redis or Redis Cloud
- **CDN**: CloudFront / CloudFlare

### Platform & DevOps
- **IaC**: Terraform
- **Kubernetes**: Helm, ArgoCD, KServe
- **Secrets**: HashiCorp Vault
- **CI/CD**: GitHub Actions

### Observability
- **Metrics**: Prometheus, Grafana
- **Logs**: ELK Stack (Elasticsearch, Logstash, Kibana) or Loki
- **Tracing**: Jaeger, Zipkin
- **APM**: Datadog, New Relic, Elastic APM

### Security
- **Scanning**: Trivy, Snyk, SonarQube, Semgrep
- **Container Registry**: Docker Hub, GCR, ECR
- **Access Management**: Okta, Auth0
- **Compliance**: Lacework, Orca, Wiz

### ML Ops
- **Model Registry**: MLflow, Hugging Face Model Hub
- **Model Serving**: KServe, BentoML
- **Experiment Tracking**: Weights & Biases, Comet
- **Feature Store**: Feast, Tecton

---

## References & Further Reading

### Official Documentation
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [HashiCorp Vault](https://www.vaultproject.io/docs)
- [ArgoCD](https://argo-cd.readthedocs.io/)

### Compliance & Security
- [GDPR Compliance Guide](https://gdpr.eu/)
- [SOC2 Trust Service Criteria](https://www.aicpa.org/soc2)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CIS Kubernetes Benchmarks](https://www.cisecurity.org/benchmark/kubernetes)

### Best Practices
- [12 Factor App](https://12factor.net/)
- [Site Reliability Engineering (SRE) Book](https://sre.google/books/)
- [Platform Engineering Guide](https://humanitec.com/blog/what-is-platform-engineering)

---

## Support & Escalation

### Getting Help
- **Documentation Portal**: https://docs.legora.local/platform
- **Slack Channel**: #platform-engineering
- **Office Hours**: Tuesdays 2-3 PM PST
- **GitHub Discussions**: https://github.com/legora/platform/discussions

### Escalation Path
1. Search documentation and existing GitHub issues
2. Post in #platform-engineering Slack channel
3. Open GitHub issue with reproduction steps
4. Page on-call platform engineer (PagerDuty)

---

## Changelog

### v1.0 (August 2026)
- Initial blueprint release
- Core infrastructure templates
- CI/CD pipeline design
- Security stack implementation
- Developer experience foundations

---

**Document maintained by**: Platform Engineering Team  
**Last reviewed**: August 2026  
**Next review**: November 2026

