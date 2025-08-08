# Deployment Guide

## Deploying MCP Servers to Production

This guide covers deployment strategies, platforms, and best practices for running MCP servers in production environments.

## Deployment Patterns

### Local vs Remote Deployment

#### Local Deployment
- **Use Cases**: Development, testing, trusted environments
- **Transport**: STDIO with optional HTTP wrapper
- **Security**: Container isolation, syscall restrictions
- **Benefits**: Lower latency, offline capability

#### Remote Deployment  
- **Use Cases**: Production, shared services, untrusted environments
- **Transport**: Streamable HTTP (recommended)
- **Security**: TLS, authentication, network isolation
- **Benefits**: Scalability, centralized management

### Deployment Strategies

#### Rainbow Rollout Strategy
Gradual deployment across multiple environments:
```yaml
environments:
  development:
    percentage: 100%
    duration: immediate
    validation: basic_tests
    
  staging:
    percentage: 100% 
    duration: 2h
    validation: integration_tests
    
  canary:
    percentage: 5%
    duration: 4h
    validation: performance_monitoring
    
  production:
    percentage: 100%
    duration: 24h
    validation: full_monitoring
```

#### High Availability Deployment
```yaml
high_availability:
  replicas: 3
  anti_affinity: true
  zones: ["us-east-1a", "us-east-1b", "us-east-1c"]
  load_balancing: round_robin
  health_checks:
    enabled: true
    interval: 30s
    failure_threshold: 3
```

### Blue-Green Deployment
Switch between two identical production environments:
```yaml
# Blue environment (current)
blue:
  version: 1.0.0
  status: active
  health: healthy

# Green environment (new)
green:
  version: 1.1.0
  status: standby
  health: testing

# Switch traffic to green after validation
```

### Canary Deployment
Gradually roll out changes to a subset of users:
```yaml
deployment:
  strategy: canary
  stages:
    - traffic: 5%
      duration: 1h
    - traffic: 25%
      duration: 2h
    - traffic: 50%
      duration: 4h
    - traffic: 100%
```

### Rolling Deployment
Update instances one at a time:
```yaml
deployment:
  strategy: rolling
  max_surge: 1
  max_unavailable: 0
```

## Container Packaging

### Signed Container Images
Always use containers signed by trusted providers:

```bash
# Sign container with cosign
cosign sign --key cosign.key mcp-server:latest

# Verify container signature
cosign verify --key cosign.pub mcp-server:latest

# Use admission controllers to enforce signature verification
kubectl apply -f - <<EOF
apiVersion: v1
kind: Policy
metadata:
  name: require-signed-images
spec:
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create"]
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  rules:
  - operations: ["CREATE"]
    resources: ["pods"]
    admissionReviewVersions: ["v1"]
EOF
```

### Trusted Repositories
Download MCP servers only from verified sources:

- **Official registries**: Docker Hub, ECR, GCR with verified publishers
- **Organization registries**: Internal container registries with signing
- **Curated catalogs**: Company-approved MCP server collections

### Container Security
```dockerfile
# Multi-stage build for minimal attack surface
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server

# Minimal runtime image
FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER 10001:10001
EXPOSE 8000
ENTRYPOINT ["/server"]
```

## Deployment Platforms

### Container Orchestration
- **Kubernetes**: Enterprise-grade orchestration
- **Docker Swarm**: Simple orchestration
- **Nomad**: Flexible workload orchestration
- **ECS/Fargate**: AWS managed containers

### Serverless
- **AWS Lambda**: Event-driven compute
- **Google Cloud Functions**: Auto-scaling functions
- **Azure Functions**: Serverless compute
- **Vercel/Netlify**: Edge functions

### Traditional
- **VMs**: Full control, higher overhead
- **Bare Metal**: Maximum performance
- **PaaS**: Heroku, Railway, Render

## Pre-Deployment Checklist

### Code Readiness
- [ ] All tests passing
- [ ] Security scan completed
- [ ] Dependencies updated
- [ ] Documentation current
- [ ] Version tagged

### Infrastructure
- [ ] Resources provisioned
- [ ] Network configured
- [ ] SSL certificates ready
- [ ] DNS configured
- [ ] Monitoring setup

### Configuration
- [ ] Environment variables set
- [ ] Secrets stored securely
- [ ] Feature flags configured
- [ ] Rate limits defined
- [ ] Backup configured

## Kubernetes Deployment

### Deployment Manifest
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server
  labels:
    app: mcp-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mcp-server
  template:
    metadata:
      labels:
        app: mcp-server
    spec:
      containers:
      - name: mcp-server
        image: mcp-server:1.0.0
        ports:
        - containerPort: 8000
        env:
        - name: MCP_ENV
          value: production
        - name: MCP_API_KEY
          valueFrom:
            secretKeyRef:
              name: mcp-secrets
              key: api-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mcp-server-service
spec:
  selector:
    app: mcp-server
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

## Docker Compose Deployment

### docker-compose.yml
```yaml
version: '3.8'

services:
  mcp-server:
    image: mcp-server:latest
    ports:
      - "8000:8000"
    environment:
      - MCP_ENV=production
      - MCP_DB_HOST=postgres
    depends_on:
      - postgres
      - redis
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: mcp
      POSTGRES_USER: mcp
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    secrets:
      - db_password

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

## Cloud Deployment

### AWS Deployment
```bash
# Build and push to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin $ECR_URI

docker build -t mcp-server .
docker tag mcp-server:latest $ECR_URI/mcp-server:latest
docker push $ECR_URI/mcp-server:latest

# Deploy with ECS
aws ecs update-service \
  --cluster mcp-cluster \
  --service mcp-server \
  --force-new-deployment
```

### Google Cloud Deployment
```bash
# Build and push to GCR
gcloud builds submit --tag gcr.io/$PROJECT_ID/mcp-server

# Deploy to Cloud Run
gcloud run deploy mcp-server \
  --image gcr.io/$PROJECT_ID/mcp-server \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
```

### Azure Deployment
```bash
# Build and push to ACR
az acr build --registry $ACR_NAME --image mcp-server .

# Deploy to Container Instances
az container create \
  --resource-group mcp-rg \
  --name mcp-server \
  --image $ACR_NAME.azurecr.io/mcp-server:latest \
  --dns-name-label mcp-server \
  --ports 8000
```

## Environment Configuration

### Production Settings
```python
# config/production.py
import os

class ProductionConfig:
    DEBUG = False
    TESTING = False
    
    # Server
    HOST = '0.0.0.0'
    PORT = int(os.getenv('PORT', 8000))
    WORKERS = int(os.getenv('WORKERS', 4))
    
    # Security
    SECRET_KEY = os.getenv('SECRET_KEY')
    SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    
    # Database
    DATABASE_URL = os.getenv('DATABASE_URL')
    DATABASE_POOL_SIZE = 20
    
    # Caching
    REDIS_URL = os.getenv('REDIS_URL')
    CACHE_TTL = 3600
    
    # Monitoring
    SENTRY_DSN = os.getenv('SENTRY_DSN')
    LOG_LEVEL = 'INFO'
```

## Load Balancing

### NGINX Configuration
```nginx
upstream mcp_servers {
    least_conn;
    server mcp1.example.com:8000 weight=3;
    server mcp2.example.com:8000 weight=2;
    server mcp3.example.com:8000 weight=1;
    
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    ssl_certificate /etc/ssl/certs/api.crt;
    ssl_certificate_key /etc/ssl/private/api.key;
    
    location / {
        proxy_pass http://mcp_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

## Monitoring Setup

### Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'mcp-server'
    static_configs:
      - targets: ['mcp1:8000', 'mcp2:8000', 'mcp3:8000']
    metrics_path: '/metrics'
```

### Grafana Dashboard
```json
{
  "dashboard": {
    "title": "MCP Server Monitoring",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(mcp_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(mcp_errors_total[5m])"
          }
        ]
      },
      {
        "title": "Response Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, mcp_request_duration_seconds)"
          }
        ]
      }
    ]
  }
}
```

## Post-Deployment

### Smoke Tests
```bash
#!/bin/bash
# smoke_test.sh

set -e

URL="${1:-http://localhost:8000}"

echo "Running smoke tests for $URL"

# Health check
curl -f "$URL/health" || exit 1

# Tool listing
curl -f "$URL/mcp" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' || exit 1

# Sample tool execution
curl -f "$URL/mcp" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"echo","arguments":{"text":"test"}},"id":2}' || exit 1

echo "Smoke tests passed!"
```

### Rollback Procedure
```bash
#!/bin/bash
# rollback.sh

PREVIOUS_VERSION="${1:-1.0.0}"

echo "Rolling back to version $PREVIOUS_VERSION"

# Kubernetes rollback
kubectl rollout undo deployment/mcp-server

# Docker rollback
docker service update --image mcp-server:$PREVIOUS_VERSION mcp-server

# Verify rollback
./smoke_test.sh
```

## Next Steps

- ☁️ [Cloud Deployment Details](cloud.md)
- 🏭 [On-Premise Deployment](on-premise.md)
- ☸️ [Kubernetes Deep Dive](kubernetes.md)
- 🔄 [CI/CD Pipelines](cicd.md)
- 🌐 [Edge Deployment](edge.md)