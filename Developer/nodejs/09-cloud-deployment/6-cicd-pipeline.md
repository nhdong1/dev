# CI/CD Pipeline — GitHub Actions, Automated Testing và Rollback

> CI/CD (Continuous Integration/Continuous Deployment — Tích Hợp/Triển Khai Liên Tục) tự động hóa quy trình từ code commit đến production — test, build, deploy, và rollback khi có vấn đề.

## Mục Lục

1. [CI/CD Concepts](#cicd-concepts)
2. [Pipeline Stages Cho Node.js](#pipeline-stages-cho-nodejs)
3. [GitHub Actions Workflow Cơ Bản](#github-actions-workflow-cơ-bản)
4. [Build và Push Docker Image](#build-và-push-docker-image)
5. [Deploy lên Kubernetes](#deploy-lên-kubernetes)
6. [Environment Strategy](#environment-strategy)
7. [Automated Rollback](#automated-rollback)
8. [Security trong CI/CD](#security-trong-cicd)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CI/CD Concepts

```
Developer Push Code
        │
        ▼
┌───────────────┐
│   CI Pipeline │  Continuous Integration
│               │
│  Lint ──► Test ──► Build ──► Scan
└───────┬───────┘
        │ ✅ All checks pass
        ▼
┌───────────────┐
│   CD Pipeline │  Continuous Deployment
│               │
│  Push Image ──► Deploy Staging ──► Smoke Test ──► Deploy Prod
└───────────────┘
```

| Khái Niệm | Mô Tả |
| --------- | ----- |
| **CI** (Continuous Integration) | Tự động test và build mỗi khi merge code |
| **CD** (Continuous Delivery) | Tự động deploy lên staging, manual approve cho prod |
| **CD** (Continuous Deployment) | Tự động deploy lên prod sau khi pass tests |
| **Pipeline** | Chuỗi automated steps |
| **Artifact** | Output của build (Docker image, binary) |
| **Rollback** | Revert về version trước khi deploy fail |

---

## Pipeline Stages Cho Node.js

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  Lint   │──►│  Test   │──►│  Build  │──►│  Scan   │──►│ Deploy  │
│ ESLint  │   │ Jest    │   │ Docker  │   │ Trivy   │   │ K8s/VM  │
│ TypeChk │   │ Supertest│  │ tsc     │   │ npm aud │   │         │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
     │              │              │              │              │
   ~30s           ~2min          ~1min          ~30s          ~2min
```

| Stage | Mục Đích | Fail Action |
| ----- | -------- | ----------- |
| **Lint** | Code style, type errors | Block merge |
| **Unit Test** | Logic correctness | Block merge |
| **Integration Test** | API + DB interaction | Block merge |
| **Build** | Compile TypeScript, build Docker image | Block deploy |
| **Security Scan** | CVE trong dependencies và image | Block deploy (critical) |
| **Deploy Staging** | Deploy lên staging environment | Notify team |
| **Smoke Test** | Verify staging hoạt động | Block prod deploy |
| **Deploy Prod** | Rolling update production | Auto rollback nếu fail |

---

## GitHub Actions Workflow Cơ Bản

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '22'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    needs: lint

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    env:
      DATABASE_URL: postgresql://test:test@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm run test:coverage

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: coverage/

  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm run build
```

---

## Build và Push Docker Image

```yaml
# .github/workflows/deploy.yml
name: Build & Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest,enable={{is_default_branch}}

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

---

## Deploy lên Kubernetes

```yaml
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig

      - name: Deploy to staging
        run: |
          kubectl set image deployment/nodejs-api \
            api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-and-push.outputs.image-tag }} \
            -n staging
          kubectl rollout status deployment/nodejs-api -n staging --timeout=300s

      - name: Smoke test
        run: |
          STAGING_URL="${{ vars.STAGING_URL }}"
          for i in {1..10}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$STAGING_URL/health")
            if [ "$STATUS" = "200" ]; then
              echo "Smoke test passed"
              exit 0
            fi
            sleep 10
          done
          echo "Smoke test failed"
          exit 1

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production  # Requires manual approval

    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v3

      - name: Deploy to production
        run: |
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          kubectl set image deployment/nodejs-api \
            api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-and-push.outputs.image-tag }} \
            -n production
          kubectl rollout status deployment/nodejs-api -n production --timeout=300s
```

---

## Environment Strategy

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Development │────►│   Staging   │────►│ Production  │
│             │     │             │     │             │
│ Local/PR    │     │ Pre-prod    │     │ Live users  │
│ preview     │     │ mirror prod │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
     Auto              Auto +              Manual
     on PR             smoke test          approval
```

| Environment | Trigger | Approval | Data |
| ----------- | ------- | -------- | ---- |
| **Development** | Mỗi PR | Không | Synthetic/test data |
| **Staging** | Merge to main | Không | Anonymized prod copy |
| **Production** | Staging pass | Manual hoặc auto | Real data |

### GitHub Environments

```yaml
deploy-production:
  environment:
    name: production
    url: https://api.example.com
```

Cấu hình protection rules trong GitHub Settings → Environments → Required reviewers.

---

## Automated Rollback

### Strategy 1: kubectl rollout undo

```yaml
      - name: Deploy with rollback on failure
        run: |
          kubectl set image deployment/nodejs-api \
            api=$IMAGE \
            -n production

          if ! kubectl rollout status deployment/nodejs-api -n production --timeout=300s; then
            echo "Rollout failed — rolling back"
            kubectl rollout undo deployment/nodejs-api -n production
            kubectl rollout status deployment/nodejs-api -n production --timeout=300s
            exit 1
          fi
```

### Strategy 2: Smoke Test sau Deploy

```yaml
      - name: Post-deploy smoke test
        id: smoke
        run: |
          PROD_URL="${{ vars.PROD_URL }}"
          sleep 30  # Chờ pods ready

          ERROR_RATE=$(curl -s "$PROD_URL/metrics" | grep 'http_requests_total{status_code="500"}' | awk '{print $2}')
          if [ "$ERROR_RATE" -gt 10 ]; then
            echo "High error rate detected"
            exit 1
          fi

      - name: Rollback on smoke test failure
        if: failure() && steps.smoke.outcome == 'failure'
        run: |
          kubectl rollout undo deployment/nodejs-api -n production
```

### Strategy 3: Blue-Green Deployment

```
Blue (current):  [v1] [v1] [v1]  ← đang nhận traffic
Green (new):     [v2] [v2] [v2]  ← deploy và test

Smoke test pass → switch traffic Blue → Green
Smoke test fail → delete Green, Blue vẫn chạy
```

---

## Security trong CI/CD

| Practice | Implementation |
| -------- | -------------- |
| **Secrets trong GitHub Secrets** | `secrets.DATABASE_URL`, không hardcode |
| **Least privilege tokens** | `GITHUB_TOKEN` scoped permissions |
| **Image scanning** | Trivy/Snyk trong pipeline |
| **Dependency audit** | `npm audit --audit-level=high` |
| **Signed commits** | GPG sign, branch protection |
| **OIDC thay vì long-lived keys** | GitHub OIDC → AWS/GCP role assumption |

```yaml
      - name: npm audit
        run: npm audit --audit-level=high

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'nodejs-api'
          path: '.'
          format: 'HTML'
```

### OIDC với AWS (Không Cần Static Keys)

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: ap-southeast-1
```

---

## Best Practices

1. **Fast feedback** — lint và unit tests chạy trước, < 5 phút cho CI
2. **Immutable artifacts** — Docker image tagged với git SHA
3. **Same image everywhere** — staging và prod dùng cùng image, khác config
4. **Database migrations** — chạy trước deploy, backward compatible
5. **Feature flags** — decouple deploy từ release
6. **Pipeline as code** — workflow files trong repo, version controlled
7. **Notify on failure** — Slack/PagerDuty integration

### Database Migration trong Pipeline

```yaml
      - name: Run migrations
        run: |
          kubectl run migration-${{ github.sha }} \
            --image=$IMAGE \
            --restart=Never \
            --command -- npx prisma migrate deploy
          kubectl wait --for=condition=complete pod/migration-${{ github.sha }} --timeout=120s
```

### Caching Dependencies

```yaml
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'  # Cache node_modules tự động
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Đáp Án Ngắn |
| ------- | ----------- |
| CI vs CD khác gì? | CI: auto test/build; CD: auto deploy lên environments |
| Tại sao dùng git SHA làm image tag? | Immutable, traceable, dễ rollback |
| Rolling update vs Blue-green? | Rolling: gradual; Blue-green: instant switch, dễ rollback |
| Smoke test là gì? | Quick health check sau deploy — verify app hoạt động cơ bản |
| Làm sao rollback trong K8s? | `kubectl rollout undo` hoặc deploy lại image tag cũ |
| Tại sao staging mirror production? | Catch environment-specific issues trước prod |
