# Legora OS Platform Engineering - Implementation Guide

## Quick Start for Platform Engineers

This guide provides step-by-step instructions to implement the platform architecture.

---

## Part 1: Local Development Setup

### Prerequisites
```bash
# Install required tools
brew install terraform helm kubectl docker-compose awscli vault

# Verify installations
terraform version      # >= 1.0
helm version          # >= 3.0
kubectl version       # >= 1.27
docker --version      # >= 20.0
```

### 1. Clone and Initialize Repository

```bash
git clone https://github.com/legora/solutionsforlegora.git
cd solutionsforlegora

# Create directory structure
mkdir -p infrastructure/{terraform,helm,k8s,scripts}
mkdir -p backend/{src,tests,observability,security}
mkdir -p frontend/{src,tests}
mkdir -p platform/{templates,cli,docs}
```

### 2. Set Up Local Development Environment

Create `docker-compose.yml` for local development:

```yaml
version: '3.9'

services:
  # MongoDB for development
  mongo:
    image: mongo:6
    container_name: legora-mongo-dev
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: devpassword
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis for caching
  redis:
    image: redis:7-alpine
    container_name: legora-redis-dev
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # Backend FastAPI service
  backend:
    build: ./backend
    container_name: legora-backend-dev
    ports:
      - "8000:8000"
    environment:
      ENVIRONMENT: development
      MONGODB_URI: mongodb://root:devpassword@mongo:27017/legora?authSource=admin
      REDIS_URL: redis://redis:6379/0
      LOG_LEVEL: DEBUG
    volumes:
      - ./backend:/app
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload

  # Frontend Next.js service
  frontend:
    build: ./frontend
    container_name: legora-frontend-dev
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8000
      NEXT_PUBLIC_ENVIRONMENT: development
    volumes:
      - ./frontend:/app
    depends_on:
      - backend
    command: npm run dev

volumes:
  mongo_data:
    driver: local

networks:
  default:
    name: legora-dev
```

### 3. Initialize Infrastructure with Terraform

```bash
cd infrastructure/terraform/environments/dev

# Initialize Terraform
terraform init

# Plan infrastructure
terraform plan -out=tfplan

# Apply configuration
terraform apply tfplan

# Save outputs for later use
terraform output -json > ../../../outputs.json
```

### 4. Configure Kubernetes Locally

```bash
# Create local Kubernetes cluster (minikube or kind)
minikube start --nodes=3 --cpus=4 --memory=8192

# Or using kind
kind create cluster --name legora-dev --config=- <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
EOF

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

### 5. Install and Configure Vault

```bash
# Start Vault in development mode
vault server -dev

# In another terminal, export Vault address
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='<token-from-dev-server>'

# Create secret engine
vault secrets enable -path=secret kv-v2

# Store secrets
vault kv put secret/legora/development/mongodb \
  username=legora_dev \
  password=dev_password \
  connection_string="mongodb://legora_dev:dev_password@mongo:27017/legora"

vault kv put secret/legora/development/jwt \
  signing_key="$(openssl rand -hex 32)" \
  refresh_key="$(openssl rand -hex 32)"

vault kv put secret/legora/development/ai-services \
  openai_api_key="sk-..." \
  huggingface_token="hf_..."
```

---

## Part 2: Infrastructure Deployment

### Step 1: Set Up Remote State Management

```hcl
# infrastructure/terraform/global/backend.tf

terraform {
  backend "s3" {
    bucket         = "legora-terraform-state-prod"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Create S3 bucket and DynamoDB table for state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "legora-terraform-state-prod"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name           = "terraform-locks"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

# Deploy this first before deploying environments
# aws s3 mb s3://legora-terraform-state-prod
# cd infrastructure/terraform/global && terraform apply
```

### Step 2: Deploy Staging Environment

```bash
cd infrastructure/terraform/environments/staging

# Configure AWS credentials
export AWS_PROFILE=legora-staging
export AWS_REGION=us-east-1

# Initialize and plan
terraform init
terraform plan -out=tfplan -var-file=staging.tfvars

# Review and apply
terraform apply tfplan

# Capture outputs
terraform output -json > ../../../staging-outputs.json

# Update kubeconfig
aws eks update-kubeconfig \
  --name legora-staging \
  --region us-east-1 \
  --profile legora-staging
```

### Step 3: Deploy Production Environment

```bash
cd infrastructure/terraform/environments/prod

# Configure AWS credentials (production account)
export AWS_PROFILE=legora-production
export AWS_REGION=us-east-1

# Plan with approval
terraform plan -out=tfplan -var-file=prod.tfvars

# Review plan output thoroughly
cat tfplan | less

# Apply with confirmation
terraform apply tfplan

# Verify deployment
kubectl get nodes -n production
kubectl get svc -n production
```

---

## Part 3: CI/CD Pipeline Setup

### Step 1: Create GitHub Actions Workflows

```bash
mkdir -p .github/workflows

# Create files in .github/workflows/:
# - ci.yml (main CI/CD pipeline)
# - deploy-staging.yml (staging deployment)
# - deploy-prod.yml (production deployment)
# - security-scan.yml (security scanning)
```

### Step 2: Configure GitHub Secrets

```bash
# Add these secrets to GitHub repository settings

# AWS Credentials
gh secret set AWS_ACCOUNT_ID --body "123456789012"
gh secret set AWS_ROLE_ARN --body "arn:aws:iam::123456789012:role/GitHubActionsRole"

# Kubernetes
gh secret set KUBECONFIG_STAGING --body "$(cat ~/.kube/config)"
gh secret set KUBECONFIG_PRODUCTION --body "$(cat ~/.kube/config-prod)"

# Vercel
gh secret set VERCEL_TOKEN --body "vercel_token_here"
gh secret set VERCEL_ORG_ID --body "org_id"
gh secret set VERCEL_PROJECT_ID_FRONTEND --body "project_id"

# Container Registry
gh secret set GCR_SA_KEY --body "$(cat gcr-sa-key.json)"

# Vault
gh secret set VAULT_ADDR --body "https://vault.legora.internal"
gh secret set VAULT_TOKEN --body "token_here"

# Slack/PagerDuty
gh secret set SLACK_WEBHOOK --body "https://hooks.slack.com/..."
gh secret set SLACK_WEBHOOK_CRITICAL --body "https://hooks.slack.com/..."
gh secret set PAGERDUTY_WEBHOOK --body "https://events.pagerduty.com/..."

# FOSSA (License compliance)
gh secret set FOSSA_API_KEY --body "fossa_key_here"
```

### Step 3: Test CI/CD Pipeline

```bash
# Push a test branch to trigger CI
git checkout -b test/ci-pipeline
git commit --allow-empty -m "test: trigger CI pipeline"
git push origin test/ci-pipeline

# Monitor workflow in GitHub Actions
# https://github.com/legora/solutionsforlegora/actions
```

---

## Part 4: Observability Stack Deployment

### Step 1: Deploy Prometheus & Grafana

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Deploy Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f infrastructure/helm/charts/monitoring/prometheus/values.yaml

# Verify deployment
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Port forward to access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Access at http://localhost:3000 (admin/prom-operator)
```

### Step 2: Deploy Loki & Promtail for Logging

```bash
# Add Loki Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Deploy Loki
helm install loki grafana/loki-stack \
  -n monitoring \
  -f infrastructure/helm/charts/monitoring/loki/values.yaml

# Verify
kubectl get pods -n monitoring | grep loki
```

### Step 3: Deploy Jaeger for Tracing

```bash
# Deploy Jaeger
helm install jaeger jaegertracing/jaeger \
  -n monitoring \
  --set query.port=16686

# Port forward
kubectl port-forward -n monitoring svc/jaeger-query 16686:16686
# Access at http://localhost:16686
```

### Step 4: Configure Alert Rules

```bash
# Apply PrometheusRule
kubectl apply -f infrastructure/k8s/monitoring/prometheus-rules.yaml

# Verify alerts loaded
kubectl get prometheusrule -n monitoring
```

---

## Part 5: Security Configuration

### Step 1: Enable RBAC in Kubernetes

```bash
# Create namespaces with network policies
kubectl apply -f infrastructure/k8s/base/namespaces.yaml

# Apply network policies
kubectl apply -f infrastructure/k8s/base/network-policies.yaml

# Create RBAC roles and bindings
kubectl apply -f infrastructure/k8s/base/rbac.yaml

# Verify
kubectl get networkpolicies -A
kubectl get roles -A
```

### Step 2: Set Up Pod Security Standards

```yaml
# infrastructure/k8s/base/pod-security-standards.yaml

apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: legora-restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  readOnlyRootFilesystem: true
  seLinux:
    rule: 'MustRunAs'
    seLinuxOptions:
      level: "s0:c123,c456"
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp:legora-restricted
rules:
  - apiGroups: ['policy']
    resources: ['podsecuritypolicies']
    verbs: ['use']
    resourceNames: ['legora-restricted']

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: psp:legora-restricted
roleRef:
  kind: ClusterRole
  name: psp:legora-restricted
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: Group
    name: system:serviceaccounts
    apiGroup: rbac.authorization.k8s.io
```

```bash
# Apply
kubectl apply -f infrastructure/k8s/base/pod-security-standards.yaml
```

### Step 3: Configure TLS/SSL

```bash
# Generate self-signed certificate for local development
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# Create Kubernetes secret
kubectl create secret tls legora-tls \
  --cert=cert.pem \
  --key=key.pem \
  -n production

# In production, use cert-manager with Let's Encrypt
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

---

## Part 6: Developer Platform Deployment

### Step 1: Build and Publish legora-cli

```bash
cd platform/cli

# Build Python CLI
pip install build
python -m build

# Publish to PyPI
pip install twine
python -m twine upload dist/*

# Users can then install with:
# pip install legora-cli
# legora --help
```

### Step 2: Deploy Developer Portal

```bash
# Create developer portal namespace
kubectl create namespace developer-portal

# Deploy as FastAPI service
kubectl apply -f infrastructure/k8s/developer-portal/ -n developer-portal

# Expose via Ingress
kubectl apply -f infrastructure/k8s/ingress/developer-portal-ingress.yaml

# Verify
kubectl get pods -n developer-portal
kubectl get ing -n developer-portal
```

### Step 3: Set Up Component Library

```bash
# Frontend components
cd frontend/component-library
npm init
npm install
npm publish

# Backend utils
cd ../../backend/component-library
python -m build
python -m twine upload dist/*
```

---

## Part 7: Compliance & Audit Setup

### Step 1: Enable API Audit Logging

```yaml
# infrastructure/k8s/audit-policy.yaml

apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests at RequestResponse level
  - level: RequestResponse
    omitStages:
      - RequestReceived
    resources:
      - group: ""
        resources: ["pods", "services", "secrets"]
    namespaces: ["production", "staging"]
  
  # Log authentication events
  - level: Metadata
    verbs: ["create", "update", "patch", "delete"]
    users: ["system:unauthenticated"]

# Apply to Kubernetes API server startup
# Add flag: --audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

### Step 2: Configure Centralized Audit Logging

```python
# backend/observability/compliance_reporter.py

import logging
from datetime import datetime, timedelta
from typing import Dict, List
import json

class ComplianceReporter:
    """Generate compliance reports from audit logs"""
    
    def __init__(self, audit_log_path: str):
        self.audit_log_path = audit_log_path
    
    def generate_gdpr_report(self, days: int = 30) -> Dict:
        """Generate GDPR compliance report"""
        
        report = {
            "generated_at": datetime.utcnow().isoformat(),
            "period_days": days,
            "data_access_events": self._count_events("document.view"),
            "data_deletion_events": self._count_events("document.delete"),
            "user_export_events": self._count_events("data.export"),
            "policy_violations": self._count_events("policy.violation"),
        }
        
        return report
    
    def generate_soc2_report(self, days: int = 30) -> Dict:
        """Generate SOC2 compliance report"""
        
        report = {
            "generated_at": datetime.utcnow().isoformat(),
            "access_control": {
                "mfa_enforced": self._check_mfa(),
                "rbac_configured": self._check_rbac(),
                "unauthorized_access_attempts": self._count_events("auth.failed"),
            },
            "encryption": {
                "tls_enabled": self._check_tls(),
                "at_rest_encryption": self._check_encryption_at_rest(),
            },
            "change_management": {
                "config_changes": self._count_events("config.changed"),
                "user_provisioning": self._count_events("user.created"),
            },
        }
        
        return report
    
    def _count_events(self, event_type: str) -> int:
        """Count events of a specific type in audit log"""
        # Implementation would parse audit log
        return 0
    
    def _check_mfa(self) -> bool:
        """Verify MFA is enforced"""
        return True
    
    def _check_rbac(self) -> bool:
        """Verify RBAC is configured"""
        return True
    
    def _check_tls(self) -> bool:
        """Verify TLS is enabled"""
        return True
    
    def _check_encryption_at_rest(self) -> bool:
        """Verify encryption at rest"""
        return True
```

### Step 3: Schedule Compliance Audits

```bash
# Create Kubernetes CronJob for compliance checks
kubectl apply -f infrastructure/k8s/compliance/audit-cronjob.yaml

# Example CronJob
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: compliance-audit
  namespace: production
spec:
  schedule: "0 2 * * 0"  # Weekly at 2 AM Sunday
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: compliance-checker
            image: gcr.io/legora-os/compliance-checker:latest
            env:
            - name: AUDIT_LOG_PATH
              value: /var/log/legora/audit.log
            volumeMounts:
            - name: audit-logs
              mountPath: /var/log/legora
          volumes:
          - name: audit-logs
            hostPath:
              path: /var/log/legora
          restartPolicy: OnFailure
EOF
```

---

## Part 8: Verification & Testing

### Health Check Script

```bash
#!/bin/bash
# infrastructure/scripts/health-check.sh

set -e

echo "🔍 Legora OS Platform Health Check"
echo "=================================="

# Check Kubernetes
echo "✓ Checking Kubernetes cluster..."
kubectl get nodes

# Check MongoDB
echo "✓ Checking MongoDB connection..."
kubectl exec -n production deployment/legora-backend -- \
  python -c "from pymongo import MongoClient; MongoClient(os.getenv('MONGODB_URI')).admin.command('ping')"

# Check Redis
echo "✓ Checking Redis connection..."
kubectl exec -n production deployment/legora-backend -- \
  redis-cli -h redis ping

# Check APIs
echo "✓ Checking backend API..."
curl -s http://legora-backend:8000/health | jq

echo "✓ Checking frontend..."
curl -s http://legora-frontend:3000/

# Check monitoring
echo "✓ Checking Prometheus..."
curl -s http://prometheus:9090/-/healthy

# Check Vault
echo "✓ Checking Vault..."
curl -s http://vault:8200/v1/sys/health | jq

echo ""
echo "✅ All health checks passed!"
```

### Smoke Test Script

```bash
#!/bin/bash
# infrastructure/scripts/smoke-tests.sh

set -e

API_URL=${1:-http://localhost:8000}
FRONTEND_URL=${2:-http://localhost:3000}

echo "🧪 Running smoke tests against $API_URL"

# Test API health
echo "Testing API health..."
curl -f "$API_URL/health" || exit 1

# Test authentication
echo "Testing authentication..."
curl -f "$API_URL/auth/login" -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test"}' || true

# Test document operations
echo "Testing document upload..."
curl -f "$API_URL/documents/upload" -X POST \
  -F "file=@test-document.pdf" || true

# Test frontend
echo "Testing frontend..."
curl -f "$FRONTEND_URL" | grep -q "Legora" || exit 1

echo "✅ Smoke tests passed!"
```

### Load Test Script (k6)

```javascript
// infrastructure/scripts/load-test.js
// Run with: k6 run load-test.js

import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp-up
    { duration: '5m', target: 100 },   // Stay at 100
    { duration: '2m', target: 0 },     // Ramp-down
  ],
};

export default function() {
  const url = 'http://localhost:8000';
  
  // Test document upload
  const uploadRes = http.post(`${url}/documents/upload`, {
    title: 'Test Document',
    content: 'Lorem ipsum...',
  });
  
  check(uploadRes, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  // Test document retrieval
  const docId = uploadRes.json('id');
  const getRes = http.get(`${url}/documents/${docId}`);
  
  check(getRes, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  
  sleep(1);
}
```

---

## Part 9: Monitoring Dashboard Setup

### Create Custom Grafana Dashboard

```json
{
  "dashboard": {
    "title": "Legora OS Overview",
    "panels": [
      {
        "title": "API Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total{job='legora-backend'}[5m])"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total{job='legora-backend',status=~'5..'}[5m])"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))"
          }
        ]
      }
    ]
  }
}
```

---

## Troubleshooting Guide

### Common Issues & Solutions

#### Pod CrashLoopBackOff

```bash
# Check pod logs
kubectl logs <pod-name> -n production

# Describe pod for events
kubectl describe pod <pod-name> -n production

# Check resource limits
kubectl top pods -n production
```

#### MongoDB Connection Issues

```bash
# Test connection string
kubectl exec -it deployment/legora-backend -n production -- \
  mongosh "mongodb://user:pass@mongo:27017/legora"

# Check MongoDB logs
kubectl logs -n production deployment/legora-backend | grep -i mongo
```

#### Deployment Stuck in Rolling Update

```bash
# Check rollout status
kubectl rollout status deployment/legora-backend -n production

# Rollback if needed
kubectl rollout undo deployment/legora-backend -n production

# Force replace
kubectl delete pod <pod-name> -n production --grace-period=0 --force
```

---

## Next Steps

1. ✅ Complete Part 1: Local Development Setup
2. ✅ Complete Part 2: Infrastructure Deployment
3. ✅ Complete Part 3: CI/CD Pipeline Setup
4. ✅ Complete Part 4: Observability Stack
5. ✅ Complete Part 5: Security Configuration
6. ✅ Complete Part 6: Developer Platform
7. ✅ Complete Part 7: Compliance Setup
8. ✅ Complete Part 8: Verification & Testing
9. ✅ Schedule team onboarding and training

---

**Document maintained by**: Platform Engineering Team  
**Last updated**: August 2026

