# Legora OS - Sample Configuration Files & Code Templates

This document contains copy-paste ready configuration files for implementation.

---

## 1. Dockerfile Templates

### Backend Dockerfile

```dockerfile
# backend/Dockerfile

FROM python:3.11-slim as builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --upgrade pip setuptools wheel && \
    pip install -r requirements.txt

# Final stage
FROM python:3.11-slim

# Set environment
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH="/opt/venv/bin:$PATH" \
    HOME=/app

# Install runtime dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /opt/venv /opt/venv

# Copy application code
COPY --chown=appuser:appuser src/ .

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Start application
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Frontend Dockerfile

```dockerfile
# frontend/Dockerfile

FROM node:18-alpine as builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source
COPY . .

# Build
RUN npm run build

# Final stage
FROM node:18-alpine

WORKDIR /app

ENV NODE_ENV=production

# Install pnpm (optional, for faster installs)
RUN npm install -g pnpm

# Copy built app and deps from builder
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package*.json ./

# Create non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001

USER nextjs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:3000/ || exit 1

CMD ["npm", "start"]
```

---

## 2. Kubernetes Manifests

### Deployment with Best Practices

```yaml
# infrastructure/k8s/backend-deployment.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: legora-backend
  namespace: production

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: legora-backend-config
  namespace: production
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  MAX_WORKERS: "4"
  API_TIMEOUT: "30"

---
apiVersion: v1
kind: Secret
metadata:
  name: legora-backend-secrets
  namespace: production
type: Opaque
stringData:
  MONGODB_URI: "mongodb+srv://user:pass@mongo.example.com/legora?ssl=true"
  JWT_SECRET_KEY: "your-secret-key-here"
  REDIS_URL: "redis://:password@redis.example.com:6379/0"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legora-backend
  namespace: production
  labels:
    app: legora-backend
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  selector:
    matchLabels:
      app: legora-backend
  
  template:
    metadata:
      labels:
        app: legora-backend
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    
    spec:
      serviceAccountName: legora-backend
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      containers:
      - name: backend
        image: gcr.io/legora-os/backend:v1.0.0
        imagePullPolicy: IfNotPresent
        
        ports:
        - name: http
          containerPort: 8000
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
        
        envFrom:
        - configMapRef:
            name: legora-backend-config
        - secretRef:
            name: legora-backend-secrets
        
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        
        livenessProbe:
          httpGet:
            path: /health
            port: http
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: http
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          capabilities:
            drop:
            - ALL
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: var-cache
          mountPath: /var/cache/legora
      
      volumes:
      - name: tmp
        emptyDir:
          sizeLimit: 1Gi
      - name: var-cache
        emptyDir:
          sizeLimit: 2Gi
      
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
        
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload-type
                operator: In
                values:
                - compute

---
apiVersion: v1
kind: Service
metadata:
  name: legora-backend
  namespace: production
  labels:
    app: legora-backend
spec:
  type: ClusterIP
  selector:
    app: legora-backend
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: metrics
    protocol: TCP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: legora-backend-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: legora-backend
  
  minReplicas: 3
  maxReplicas: 10
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 15
      selectPolicy: Max

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: legora-backend-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: legora-backend
```

### NetworkPolicy for Segmentation

```yaml
# infrastructure/k8s/network-policies.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: legora-backend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: legora-backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8000

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: legora-backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: legora-backend
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # MongoDB
  - to:
    - podSelector:
        matchLabels:
          app: mongodb
    ports:
    - protocol: TCP
      port: 27017
  # Redis
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  # External APIs
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

---

## 3. FastAPI Backend Skeleton

```python
# backend/src/main.py

import logging
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.responses import JSONResponse
import os
from contextlib import asynccontextmanager

# Observability
from observability.tracing import setup_tracing
from observability.audit_logger import audit_logger

# Security
from security.rbac import get_current_user
from security.secrets import get_secrets_manager

# Setup logging
logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO"))
logger = logging.getLogger(__name__)

# Lifespan context manager
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    logger.info("Starting Legora OS Backend")
    setup_tracing(app)
    
    yield
    
    # Shutdown
    logger.info("Shutting down Legora OS Backend")

# Create FastAPI app
app = FastAPI(
    title="Legora OS API",
    description="Legal document analysis platform",
    version="1.0.0",
    lifespan=lifespan,
)

# Add security middleware
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["legora.legal", "*.legora.legal"])
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "http://localhost:3000").split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Exception handlers
@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    audit_logger.log_event(
        event_type="api.error",
        user_id=request.headers.get("X-User-ID"),
        resource=request.url.path,
        action=request.method,
        status="failed",
        metadata={"status_code": exc.status_code, "detail": exc.detail}
    )
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail},
    )

# Health endpoints
@app.get("/health")
async def health_check():
    """Simple health check"""
    return {"status": "healthy"}

@app.get("/ready")
async def readiness_check(sm = Depends(get_secrets_manager)):
    """Readiness check - verify external dependencies"""
    try:
        # Check database
        secrets = sm.get_secret("database/mongo")
        # Check Redis
        secrets = sm.get_secret("cache/redis")
        return {"status": "ready"}
    except Exception as e:
        logger.error(f"Readiness check failed: {e}")
        return JSONResponse(status_code=503, content={"status": "not ready"})

# Include routers
from routers import documents, auth, analysis

app.include_router(auth.router, prefix="/auth", tags=["Authentication"])
app.include_router(documents.router, prefix="/documents", tags=["Documents"])
app.include_router(analysis.router, prefix="/analysis", tags=["Analysis"])

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        app,
        host="0.0.0.0",
        port=int(os.getenv("PORT", 8000)),
        workers=int(os.getenv("MAX_WORKERS", 4)),
    )
```

### Sample Router (Documents)

```python
# backend/src/routers/documents.py

from fastapi import APIRouter, UploadFile, File, Depends, HTTPException
from pydantic import BaseModel
from typing import List
from datetime import datetime
import os

from security.rbac import get_current_user, require_permission, Permission
from security.encryption import EncryptionManager
from observability.audit_logger import audit_logger, AuditEventType
from pymongo import MongoClient

router = APIRouter()

class DocumentResponse(BaseModel):
    id: str
    title: str
    content_hash: str
    created_at: datetime
    classification: str

@router.post("/upload")
async def upload_document(
    file: UploadFile = File(...),
    user = Depends(require_permission(Permission.UPLOAD_DOCUMENT)),
):
    """Upload a legal document"""
    
    try:
        # Read file
        content = await file.read()
        
        # Encrypt content
        em = EncryptionManager()
        encrypted_content = em.encrypt_document(content)
        
        # Store in MongoDB
        client = MongoClient(os.getenv("MONGODB_URI"))
        db = client.legora
        documents = db.documents
        
        doc = {
            "title": file.filename,
            "content": encrypted_content,
            "user_id": user.user_id,
            "classification": "confidential",
            "created_at": datetime.utcnow(),
        }
        
        result = documents.insert_one(doc)
        doc_id = str(result.inserted_id)
        
        # Audit log
        audit_logger.log_event(
            event_type=AuditEventType.DOCUMENT_UPLOAD,
            user_id=user.user_id,
            resource=f"document/{doc_id}",
            action="upload",
            status="success",
            metadata={
                "filename": file.filename,
                "size_bytes": len(content),
                "classification": doc.get("classification"),
            }
        )
        
        return {"id": doc_id, "filename": file.filename}
    
    except Exception as e:
        audit_logger.log_event(
            event_type=AuditEventType.DOCUMENT_UPLOAD,
            user_id=user.user_id,
            resource=f"document/unknown",
            action="upload",
            status="failed",
            metadata={"error": str(e)}
        )
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/{doc_id}")
async def get_document(
    doc_id: str,
    user = Depends(require_permission(Permission.VIEW_DOCUMENT)),
):
    """Retrieve a document"""
    
    client = MongoClient(os.getenv("MONGODB_URI"))
    db = client.legora
    documents = db.documents
    
    # Find document
    from bson import ObjectId
    doc = documents.find_one({"_id": ObjectId(doc_id)})
    
    if not doc:
        raise HTTPException(status_code=404, detail="Document not found")
    
    # Verify access (simple check - in production, use more sophisticated access control)
    if doc["user_id"] != user.user_id and user.role != "admin":
        audit_logger.log_event(
            event_type=AuditEventType.POLICY_VIOLATION,
            user_id=user.user_id,
            resource=f"document/{doc_id}",
            action="access_denied",
            status="failed",
            metadata={"reason": "unauthorized_access_attempt"}
        )
        raise HTTPException(status_code=403, detail="Access denied")
    
    # Decrypt content
    em = EncryptionManager()
    decrypted_content = em.decrypt_document(doc["content"])
    
    # Audit log
    audit_logger.log_event(
        event_type=AuditEventType.DOCUMENT_VIEW,
        user_id=user.user_id,
        resource=f"document/{doc_id}",
        action="view",
        status="success",
        metadata={"title": doc["title"]}
    )
    
    return {
        "id": str(doc["_id"]),
        "title": doc["title"],
        "content": decrypted_content.decode(),
        "created_at": doc["created_at"],
    }
```

---

## 4. Next.js Frontend Skeleton

```typescript
// frontend/pages/api/documents.ts

import type { NextApiRequest, NextApiResponse } from 'next';
import { getSession } from 'next-auth/react';

type ResponseData = {
  documents?: any[];
  error?: string;
};

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse<ResponseData>
) {
  // Verify session
  const session = await getSession({ req });
  
  if (!session) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  if (req.method === 'GET') {
    try {
      // Fetch documents from backend
      const apiUrl = process.env.NEXT_PUBLIC_API_URL;
      const response = await fetch(`${apiUrl}/documents`, {
        headers: {
          'Authorization': `Bearer ${session.accessToken}`,
        },
      });

      if (!response.ok) {
        throw new Error('Failed to fetch documents');
      }

      const documents = await response.json();
      return res.status(200).json({ documents });
    } catch (error) {
      return res.status(500).json({ error: 'Internal server error' });
    }
  }

  return res.status(405).json({ error: 'Method not allowed' });
}
```

### React Component with Error Boundary

```typescript
// frontend/components/DocumentUploader.tsx

import React, { useState } from 'react';
import { useSession } from 'next-auth/react';

interface DocumentUploaderProps {
  onUpload?: (docId: string) => void;
  onError?: (error: string) => void;
}

export const DocumentUploader: React.FC<DocumentUploaderProps> = ({
  onUpload,
  onError,
}) => {
  const { data: session } = useSession();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleFileChange = async (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    
    if (!file) return;

    // Validate file
    const maxSize = 100 * 1024 * 1024; // 100 MB
    if (file.size > maxSize) {
      const err = 'File size exceeds 100 MB';
      setError(err);
      onError?.(err);
      return;
    }

    setIsLoading(true);
    setError(null);

    try {
      const formData = new FormData();
      formData.append('file', file);

      const response = await fetch(
        `${process.env.NEXT_PUBLIC_API_URL}/documents/upload`,
        {
          method: 'POST',
          body: formData,
          headers: {
            'Authorization': `Bearer ${session?.accessToken}`,
          },
        }
      );

      if (!response.ok) {
        throw new Error('Upload failed');
      }

      const data = await response.json();
      onUpload?.(data.id);
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Unknown error';
      setError(errorMessage);
      onError?.(errorMessage);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="document-uploader">
      <input
        type="file"
        onChange={handleFileChange}
        disabled={isLoading}
        accept=".pdf,.docx,.txt"
      />
      
      {isLoading && <p>Uploading...</p>}
      {error && <p className="error">{error}</p>}
    </div>
  );
};

// Error Boundary
import React from 'react';

interface ErrorBoundaryProps {
  children: React.ReactNode;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends React.Component<
  ErrorBoundaryProps,
  ErrorBoundaryState
> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error) {
    console.error('Error caught by boundary:', error);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-container">
          <h1>Something went wrong</h1>
          <p>{this.state.error?.message}</p>
        </div>
      );
    }

    return this.props.children;
  }
}
```

---

## 5. Helm Chart Values

```yaml
# infrastructure/helm/charts/legora-backend/values.yaml

replicaCount: 3

image:
  repository: gcr.io/legora-os/backend
  pullPolicy: IfNotPresent
  tag: "1.0.0"

nameOverride: ""
fullnameOverride: "legora-backend"

serviceAccount:
  create: true
  annotations: {}
  name: "legora-backend"

service:
  type: ClusterIP
  port: 80
  targetPort: 8000

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: api.legora.legal
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: legora-api-tls
      hosts:
        - api.legora.legal

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

environment:
  ENVIRONMENT: production
  LOG_LEVEL: INFO
  MAX_WORKERS: "4"

secrets:
  mongodb:
    existingSecret: legora-backend-secrets
    usernameKey: mongodb-username
    passwordKey: mongodb-password
  jwt:
    existingSecret: legora-backend-secrets
    keyKey: jwt-secret-key
```

---

## 6. GitHub Actions Workflow (Simplified)

```yaml
# .github/workflows/ci.yml (Simplified)

name: CI/CD

on:
  push:
    branches: [main, staging]
    tags: ['v*']
  pull_request:
    branches: [main, staging]

jobs:
  test:
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
        ports:
          - 6379:6379

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
        cache: 'pip'

    - name: Install dependencies
      run: |
        pip install -r backend/requirements.txt pytest pytest-cov

    - name: Run tests
      run: cd backend && pytest tests/ -v --cov

    - name: Upload coverage
      uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Build image
      uses: docker/build-push-action@v4
      with:
        context: ./backend
        push: false
        tags: legora-backend:${{ github.sha }}

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4

    - name: Deploy to staging
      run: |
        echo "Deploying to staging..."
        # Add deployment commands here
```

---

## 7. Prometheus Alerts

```yaml
# infrastructure/k8s/prometheus-rules.yaml

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: legora-alerts
  namespace: monitoring
spec:
  groups:
  - name: legora.rules
    interval: 30s
    
    rules:
    - alert: BackendHighErrorRate
      expr: |
        (sum(rate(http_requests_total{job="legora-backend",status=~"5.."}[5m])) /
         sum(rate(http_requests_total{job="legora-backend"}[5m]))) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on backend"
        description: "Error rate: {{ $value | humanizePercentage }}"

    - alert: BackendHighLatency
      expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High latency detected"
        description: "P99 latency: {{ $value }}s"

    - alert: DatabaseDown
      expr: up{job="mongodb"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "MongoDB is down"

    - alert: PodCrashLoop
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} in crash loop"
```

---

This document provides copy-paste ready configuration files. Customize them according to your specific environment before deployment.

**Last updated**: August 2026

