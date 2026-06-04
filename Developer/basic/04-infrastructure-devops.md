# Infrastructure & DevOps - Interview Questions

## Question 1: Giải thích Kubernetes architecture và core components?

**Answer:**

**Kubernetes (K8s)** là container orchestration platform để deploy, scale, và manage containerized applications.

**Architecture Overview:**
```
┌─────────────────── Control Plane ───────────────────┐
│  API Server ← etcd                                  │
│      ↓                                              │
│  Controller Manager ─── Scheduler                   │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────── Worker Nodes ────────────────────┐
│  Node 1              Node 2              Node 3     │
│  ┌────────────┐     ┌────────────┐     ┌──────────┐│
│  │ kubelet    │     │ kubelet    │     │ kubelet  ││
│  │ kube-proxy │     │ kube-proxy │     │ kube-proxy│
│  │ Container  │     │ Container  │     │ Container││
│  │ Runtime    │     │ Runtime    │     │ Runtime  ││
│  └────────────┘     └────────────┘     └──────────┘│
└─────────────────────────────────────────────────────┘
```

**Control Plane Components:**

| Component | Function |
|-----------|----------|
| **API Server** | Entry point for all REST commands |
| **etcd** | Distributed key-value store for cluster state |
| **Scheduler** | Assigns pods to nodes based on resources |
| **Controller Manager** | Runs controllers (Deployment, ReplicaSet, etc.) |

**Node Components:**

| Component | Function |
|-----------|----------|
| **kubelet** | Agent that ensures containers are running |
| **kube-proxy** | Network proxy, maintains network rules |
| **Container Runtime** | Docker, containerd, CRI-O |

**Core K8s Objects:**

```yaml
# Pod: Smallest deployable unit
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080

# Deployment: Declarative updates for Pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:1.0

# Service: Expose application
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # ClusterIP, NodePort, LoadBalancer
```

---

## Question 2: Docker best practices và multi-stage builds?

**Answer:**

**Dockerfile Best Practices:**

**1. Use Official Base Images**
```dockerfile
# Good
FROM node:18-alpine

# Avoid: Unknown/untrusted images
FROM random-user/node-custom
```

**2. Minimize Layers**
```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git

# Good: Single layer
RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*
```

**3. Order Matters (Layer Caching)**
```dockerfile
# Good: Copy package files first (rarely change)
COPY package*.json ./
RUN npm install

# Then copy source (frequently change)
COPY . .
```

**4. Multi-stage Build**
```dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**5. Non-root User**
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**6. Use .dockerignore**
```
node_modules
.git
*.md
Dockerfile
.env
```

**7. Health Checks**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

**Image Size Comparison:**
```
node:18          ~900MB
node:18-slim     ~200MB
node:18-alpine   ~100MB
distroless       ~20MB (Go apps)
```

---

## Question 3: CI/CD Pipeline design và best practices?

**Answer:**

**CI/CD Pipeline Stages:**

```
┌─────────────────────────────────────────────────────────────┐
│  Source → Build → Test → Security → Deploy → Monitor       │
│                                                             │
│  Commit → Compile → Unit Tests    → SAST    → Staging  → Prod│
│         → Package → Integration   → DAST    →          →    │
│         → Lint    → E2E           → Secrets →          →    │
└─────────────────────────────────────────────────────────────┘
```

**GitHub Actions Example:**
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm test -- --coverage

      - name: Build
        run: npm run build

  security-scan:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master

  deploy-staging:
    needs: [build-and-test, security-scan]
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          kubectl apply -f k8s/staging/

  deploy-production:
    needs: [build-and-test, security-scan]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - name: Deploy to Production
        run: |
          kubectl apply -f k8s/production/
```

**Best Practices:**

1. **Fast Feedback**: Run quick tests first
2. **Parallel Jobs**: Run independent jobs concurrently
3. **Caching**: Cache dependencies between builds
4. **Environment Separation**: staging → production
5. **Rollback Strategy**: Keep previous versions
6. **Feature Flags**: Decouple deployment from release
7. **Trunk-based Development**: Short-lived branches

**Deployment Strategies:**
```
Blue-Green:     [Blue v1] ─────→ [Green v2]
                (instant switch, easy rollback)

Canary:         [v1 90%] ─→ [v2 10%] ─→ [v2 100%]
                (gradual rollout, monitor)

Rolling Update: [Pod1 v1→v2] [Pod2 v1→v2] [Pod3 v1→v2]
                (zero downtime, incremental)
```

---

## Question 4: Monitoring, Logging, Observability - Three Pillars?

**Answer:**

**Three Pillars of Observability:**

**1. Metrics (Numbers over time)**
```
What: Numerical measurements aggregated over time
Tools: Prometheus, Grafana, DataDog

Types:
- Counter: Total requests, errors
- Gauge: Current memory, CPU usage
- Histogram: Request latency distribution
```

```yaml
# Prometheus metrics example
http_requests_total{method="GET", endpoint="/api/users", status="200"} 12345
http_request_duration_seconds_bucket{le="0.1"} 500
http_request_duration_seconds_bucket{le="0.5"} 900
```

**2. Logs (Discrete events)**
```
What: Timestamped records of discrete events
Tools: ELK Stack (Elasticsearch, Logstash, Kibana), Loki, Splunk

Best Practices:
- Structured logging (JSON)
- Correlation IDs
- Log levels (DEBUG, INFO, WARN, ERROR)
```

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "service": "user-service",
  "correlation_id": "abc-123",
  "message": "Database connection failed",
  "error": "timeout after 30s",
  "user_id": "user-456"
}
```

**3. Traces (Request flow)**
```
What: Follow request through distributed system
Tools: Jaeger, Zipkin, OpenTelemetry

Components:
- Trace: End-to-end request
- Span: Single operation within trace
- Context propagation: Pass trace ID across services
```

```
Trace ID: abc-123
├── Span: API Gateway (10ms)
│   └── Span: Auth Service (5ms)
├── Span: User Service (50ms)
│   ├── Span: Database Query (30ms)
│   └── Span: Cache Lookup (2ms)
└── Span: Email Service (100ms)
```

**Alerting Strategy:**
```yaml
# Prometheus alerting rules
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status="500"}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
```

**Golden Signals (Google SRE):**
- **Latency**: Time to service a request
- **Traffic**: Requests per second
- **Errors**: Error rate
- **Saturation**: Resource utilization

---

## Question 5: Infrastructure as Code - Terraform basics?

**Answer:**

**Terraform** là IaC tool để provision và manage cloud infrastructure.

**Core Concepts:**

```hcl
# Provider configuration
provider "aws" {
  region = "us-east-1"
}

# Resource definition
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}

# Variables
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

# Outputs
output "instance_ip" {
  value = aws_instance.web.public_ip
}

# Data sources (read existing resources)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

**Terraform Workflow:**
```bash
terraform init      # Initialize, download providers
terraform plan      # Preview changes
terraform apply     # Apply changes
terraform destroy   # Remove resources
```

**State Management:**
```hcl
# Remote state (recommended for teams)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # Locking
    encrypt        = true
  }
}
```

**Modules (Reusable components):**
```hcl
# Using a module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}
```

**Best Practices:**
1. **Version Control**: Store Terraform files in Git
2. **Remote State**: Use S3/Azure Blob for state
3. **State Locking**: Prevent concurrent modifications
4. **Workspaces**: Separate environments (dev, staging, prod)
5. **Modules**: Reusable, tested components
6. **CI/CD Integration**: Automated plan/apply in pipelines

---

## Question 6: Cloud Services comparison - AWS vs Azure vs GCP?

**Answer:**

**Service Comparison:**

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| **Compute** | EC2 | Virtual Machines | Compute Engine |
| **Serverless** | Lambda | Functions | Cloud Functions |
| **Containers** | EKS, ECS | AKS | GKE |
| **Object Storage** | S3 | Blob Storage | Cloud Storage |
| **Database (SQL)** | RDS | SQL Database | Cloud SQL |
| **Database (NoSQL)** | DynamoDB | CosmosDB | Firestore/Bigtable |
| **Message Queue** | SQS, SNS | Service Bus | Pub/Sub |
| **CDN** | CloudFront | Azure CDN | Cloud CDN |
| **DNS** | Route 53 | Azure DNS | Cloud DNS |

**Strengths:**

**AWS:**
- Largest market share, most mature
- Broadest service offering
- Best for: Enterprise, startups, anyone

**Azure:**
- Best Microsoft/Windows integration
- Strong hybrid cloud (Azure Arc)
- Best for: Enterprise, Microsoft shops

**GCP:**
- Best for data/ML (BigQuery, Vertex AI)
- Best Kubernetes (GKE - originated K8s)
- Best for: Data-heavy, ML/AI workloads

**Cost Comparison Factors:**
```
- On-demand vs Reserved vs Spot instances
- Egress costs (data transfer out)
- Storage classes and access patterns
- Regional pricing differences
```

**Multi-cloud Considerations:**
```
Pros:
- Avoid vendor lock-in
- Best-of-breed services
- Disaster recovery

Cons:
- Operational complexity
- Skill requirements
- Data transfer costs
```

---

## Question 7: Container Orchestration - Kubernetes Networking?

**Answer:**

**K8s Networking Model:**
- Every Pod gets its own IP address
- Pods can communicate without NAT
- Nodes can communicate with Pods without NAT

**Service Types:**

```yaml
# ClusterIP (internal only)
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
---
# NodePort (expose on node IP)
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080  # 30000-32767
---
# LoadBalancer (cloud provider LB)
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 443
    targetPort: 8443
```

**Ingress (HTTP/HTTPS routing):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**Network Policies:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**CNI Plugins:**
- Calico: Network policy, BGP
- Flannel: Simple overlay network
- Cilium: eBPF-based, advanced security
- Weave: Simple, encrypted

---

## Question 8: Explain Kubernetes scaling strategies?

**Answer:**

**1. Horizontal Pod Autoscaler (HPA)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
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
    scaleUp:
      stabilizationWindowSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

**2. Vertical Pod Autoscaler (VPA)**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 4Gi
```

**3. Cluster Autoscaler**
```yaml
# Scales node count based on pending pods
# Cloud provider specific configuration

# AWS: Auto Scaling Groups
# GCP: Managed Instance Groups
# Azure: Virtual Machine Scale Sets
```

**Pod Resource Management:**
```yaml
spec:
  containers:
  - name: app
    resources:
      requests:        # Scheduler uses this
        memory: "256Mi"
        cpu: "250m"
      limits:          # Cannot exceed this
        memory: "512Mi"
        cpu: "500m"
```

**Scaling Best Practices:**
1. Set appropriate requests/limits
2. Use readiness probes (don't route to unready pods)
3. Implement graceful shutdown
4. Consider scale-down delay to prevent thrashing
5. Monitor and tune thresholds

**Custom Metrics Autoscaling (KEDA):**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaledobject
spec:
  scaleTargetRef:
    name: consumer
  triggers:
  - type: rabbitmq
    metadata:
      queueName: myqueue
      queueLength: "10"  # Scale when queue > 10 messages
```

---

## Question 9: GitOps và ArgoCD workflow?

**Answer:**

**GitOps Principles:**
- Git là single source of truth
- Declarative configuration
- Automated synchronization
- Self-healing infrastructure

**GitOps Workflow:**
```
Developer → Git Push → Git Repository
                            ↓
                       ArgoCD detects change
                            ↓
                       Sync to Kubernetes
                            ↓
                       Kubernetes Cluster
```

**ArgoCD Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/my-app
    targetRevision: HEAD
    path: kubernetes/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true       # Remove deleted resources
      selfHeal: true    # Auto-correct drift
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Repository Structure:**
```
kubernetes/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── development/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

**Kustomize Overlay:**
```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: my-app-prod

resources:
- ../../base

replicas:
- name: my-app
  count: 5

images:
- name: my-app
  newTag: v1.2.3

patchesStrategicMerge:
- resources-patch.yaml
```

**Best Practices:**
1. **Separate Config Repo**: App code and K8s config in different repos
2. **Environment Branches/Folders**: dev, staging, prod
3. **Sealed Secrets**: Encrypt secrets in Git (Bitnami Sealed Secrets)
4. **Progressive Delivery**: Integrate with Argo Rollouts for canary
5. **RBAC**: Limit who can sync to production

---

## Question 10: Site Reliability Engineering (SRE) practices?

**Answer:**

**SRE Core Concepts:**

**1. Service Level Indicators (SLIs)**
```
Metrics that measure service quality:
- Availability: Successful requests / Total requests
- Latency: % requests under threshold
- Throughput: Requests per second
- Error rate: Errors / Total requests
```

**2. Service Level Objectives (SLOs)**
```yaml
# Example SLOs
availability: 99.9%          # 8.76 hours downtime/year
latency_p99: 200ms           # 99% of requests < 200ms
error_rate: 0.1%             # Max 0.1% errors

# Calculate error budget
Error Budget = 100% - SLO = 0.1%
Monthly budget (30 days) = 43.2 minutes of downtime
```

**3. Service Level Agreements (SLAs)**
```
Customer-facing commitments with consequences:
- Financial penalties
- Service credits
- SLA > SLO (internal target stricter)
```

**Error Budget Policy:**
```
If error budget exhausted:
1. Freeze feature releases
2. Focus on reliability work
3. Conduct incident reviews
4. Resume when budget recovers
```

**Incident Management:**
```
1. Detection: Alert fires
2. Triage: Assess severity, assign incident commander
3. Mitigation: Restore service (not fix root cause)
4. Resolution: Permanent fix
5. Post-mortem: Blameless review, action items

Severities:
- P1/SEV1: Service down, customer impact
- P2/SEV2: Degraded service
- P3/SEV3: Minor issue, no immediate impact
```

**Toil Reduction:**
```
Toil: Manual, repetitive, automatable work

Examples:
- Manual deployments
- Ticket-driven restarts
- Alert response without automation

Goal: Automate to spend <50% time on toil
```

**Capacity Planning:**
```
1. Measure current usage
2. Forecast growth
3. Plan capacity ahead of demand
4. Load test to validate limits

Formula:
Needed Capacity = (Current Load × Growth Factor) / Utilization Target
Example: (1000 QPS × 1.5) / 0.7 = 2143 QPS capacity
```

**Reliability Practices:**
- **Chaos Engineering**: Netflix Chaos Monkey, Gremlin
- **Game Days**: Simulated incidents for practice
- **Runbooks**: Documented incident response
- **On-call Rotation**: Sustainable schedules

---
