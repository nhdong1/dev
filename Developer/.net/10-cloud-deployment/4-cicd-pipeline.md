# CI/CD Pipeline cho .NET — GitHub Actions và Azure DevOps

> CI/CD — Continuous Integration / Continuous Delivery — Tích Hợp Liên Tục / Phân Phối Liên Tục là nền tảng của DevOps hiện đại. Mỗi commit được tự động build, test, và deploy, giảm thiểu lỗi thủ công và tăng tốc chu kỳ phát hành. Bài này trình bày cách xây dựng pipeline CI/CD hoàn chỉnh cho .NET với GitHub Actions và Azure DevOps.

---

## 1. Tổng Quan CI/CD Pipeline

```
Developer
    │
    │ git push
    ▼
┌─────────────────────────────────────────────────────────────┐
│                    CI — CONTINUOUS INTEGRATION              │
│                                                             │
│  Trigger → Checkout → Restore → Build → Test → Lint/SAST  │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ (all checks pass)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    CD — CONTINUOUS DELIVERY                 │
│                                                             │
│  Build Image → Push to Registry → Deploy to Staging        │
│       → Integration Tests → Approve → Deploy to Production │
└─────────────────────────────────────────────────────────────┘
```

### Nguyên Tắc CI/CD

- **Fast feedback** — Phản hồi nhanh: build và test phải hoàn thành trong < 10 phút
- **Fail fast** — Thất bại sớm: chạy test nhanh nhất trước
- **Idempotent** — Lũy đẳng: chạy pipeline nhiều lần phải cho kết quả như nhau
- **Trunk-based development** — Phát triển dựa trên nhánh chính: commit thường xuyên vào main/master

---

## 2. GitHub Actions — CI/CD Pipeline Hoàn Chỉnh

### Cấu Trúc Workflow Files

```
.github/
└── workflows/
    ├── ci.yml          # Chạy khi có PR — Pull Request
    ├── cd-staging.yml  # Deploy lên staging khi merge vào main
    └── cd-prod.yml     # Deploy lên production khi release tag
```

### CI Workflow — Kiểm Tra Pull Request

```yaml
# .github/workflows/ci.yml
name: CI — Build & Test

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

env:
  DOTNET_VERSION: '8.0.x'
  DOTNET_SKIP_FIRST_TIME_EXPERIENCE: true
  DOTNET_CLI_TELEMETRY_OPTOUT: true

jobs:
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest

    services:
      # PostgreSQL để chạy integration tests
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      # Cache NuGet packages để tăng tốc build
      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      # Unit Tests
      - name: Run unit tests
        run: |
          dotnet test tests/MyApp.UnitTests/ \
            --no-build \
            --configuration Release \
            --logger "trx;LogFileName=unit-test-results.trx" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults

      # Integration Tests
      - name: Run integration tests
        env:
          ConnectionStrings__Default: "Host=localhost;Database=testdb;Username=postgres;Password=testpassword"
        run: |
          dotnet test tests/MyApp.IntegrationTests/ \
            --no-build \
            --configuration Release \
            --logger "trx;LogFileName=integration-test-results.trx" \
            --results-directory ./TestResults

      # Publish test results lên GitHub
      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: success() || failure()
        with:
          name: Test Results
          path: TestResults/*.trx
          reporter: dotnet-trx

      # Code coverage report
      - name: Generate coverage report
        run: |
          dotnet tool install -g dotnet-reportgenerator-globaltool
          reportgenerator \
            -reports:"./TestResults/**/coverage.cobertura.xml" \
            -targetdir:"./TestResults/CoverageReport" \
            -reporttypes:Html

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: TestResults/CoverageReport

      # Check coverage threshold — ngưỡng độ bao phủ
      - name: Check coverage threshold
        run: |
          coverage=$(grep -oP 'line-rate="\K[0-9.]+' TestResults/**/coverage.cobertura.xml | head -1)
          threshold=0.80
          if (( $(echo "$coverage < $threshold" | bc -l) )); then
            echo "Coverage $coverage is below threshold $threshold"
            exit 1
          fi

  # SAST — Static Application Security Testing — Kiểm Tra Bảo Mật Tĩnh
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run CodeQL Analysis
        uses: github/codeql-action/init@v3
        with:
          languages: csharp
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      - run: dotnet build
      - uses: github/codeql-action/analyze@v3
```

### CD Staging Workflow — Deploy Tự Động lên Staging

```yaml
# .github/workflows/cd-staging.yml
name: CD — Deploy to Staging

on:
  push:
    branches: [main]

env:
  REGISTRY: myacr.azurecr.io
  IMAGE_NAME: myapp-api
  DOTNET_VERSION: '8.0.x'

jobs:
  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Tạo image metadata — tag theo commit SHA và branch
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      # Đăng nhập Azure Container Registry
      - name: Login to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.REGISTRY }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      # Build và push với BuildKit — builder nhanh hơn
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:cache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:cache,mode=max
          build-args: |
            VERSION=${{ steps.meta.outputs.version }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}

  deploy-staging:
    name: Deploy to Staging
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Đăng nhập Azure
      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      # Cập nhật AKS deployment
      - name: Set AKS context
        uses: azure/aks-set-context@v3
        with:
          resource-group: myapp-rg
          cluster-name: myapp-aks

      - name: Deploy to AKS Staging
        run: |
          kubectl set image deployment/myapp-api \
            myapp-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            -n staging
          
          kubectl rollout status deployment/myapp-api -n staging --timeout=5m

      # Chạy smoke test sau khi deploy
      - name: Run smoke tests
        run: |
          sleep 30  # chờ deployment ổn định
          response=$(curl -s -o /dev/null -w "%{http_code}" https://staging.myapp.com/health)
          if [ "$response" != "200" ]; then
            echo "Smoke test failed: health check returned $response"
            kubectl rollout undo deployment/myapp-api -n staging
            exit 1
          fi
          echo "Smoke test passed!"

  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    # Yêu cầu manual approval — phê duyệt thủ công
    environment:
      name: production
      url: https://myapp.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Set AKS context
        uses: azure/aks-set-context@v3
        with:
          resource-group: myapp-rg
          cluster-name: myapp-aks

      - name: Deploy to AKS Production
        run: |
          kubectl set image deployment/myapp-api \
            myapp-api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            -n production
          
          kubectl rollout status deployment/myapp-api -n production --timeout=10m

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Deployed to Production: ${{ github.sha }} by ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Rollback on failure
        if: failure()
        run: |
          kubectl rollout undo deployment/myapp-api -n production
          echo "Rolled back deployment due to failure"
```

### Sử Dụng GitHub Environments để Quản Lý Secrets

```yaml
# Tạo environments trong GitHub: Settings → Environments
# production environment có:
# - Required reviewers (cần người approve)
# - Wait timer (chờ N phút)
# - Deployment branches (chỉ từ branch main)
# - Environment secrets (khác với repo secrets)

jobs:
  deploy:
    environment:
      name: production
    steps:
      - run: echo ${{ secrets.PROD_DB_PASSWORD }}  # secret của production env
```

---

## 3. Azure DevOps Pipelines — Pipeline cho Doanh Nghiệp

### Pipeline YAML Cơ Bản (azure-pipelines.yml)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    exclude:
      - '**/*.md'
      - 'docs/**'

pr:
  branches:
    include:
      - main

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'
  imageRepository: 'myapp-api'
  containerRegistry: 'myacr.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

pool:
  vmImage: ubuntu-latest

stages:
# ─── Stage 1: Build & Test ─────────────────────────────────────
- stage: BuildAndTest
  displayName: 'Build & Test'
  jobs:
  - job: Build
    displayName: 'Build, Test & Analyze'
    
    steps:
    - task: UseDotNet@2
      displayName: 'Setup .NET SDK'
      inputs:
        packageType: sdk
        version: $(dotnetVersion)

    - task: Cache@2
      displayName: 'Cache NuGet packages'
      inputs:
        key: 'nuget | "$(Agent.OS)" | **/packages.lock.json,!**/bin/**,!**/obj/**'
        restoreKeys: |
          nuget | "$(Agent.OS)"
          nuget
        path: $(NUGET_PACKAGES)

    - script: dotnet restore
      displayName: 'Restore dependencies'

    - script: dotnet build --no-restore --configuration $(buildConfiguration)
      displayName: 'Build'

    - script: |
        dotnet test tests/MyApp.UnitTests/ \
          --no-build \
          --configuration $(buildConfiguration) \
          --logger trx \
          --collect "XPlat Code Coverage" \
          --results-directory $(Agent.TempDirectory)/TestResults
      displayName: 'Run unit tests'

    - task: PublishTestResults@2
      displayName: 'Publish test results'
      condition: succeededOrFailed()
      inputs:
        testResultsFormat: VSTest
        testResultsFiles: '**/*.trx'
        searchFolder: $(Agent.TempDirectory)/TestResults

    - task: PublishCodeCoverageResults@2
      displayName: 'Publish code coverage'
      inputs:
        codeCoverageTool: Cobertura
        summaryFileLocation: '$(Agent.TempDirectory)/TestResults/**/coverage.cobertura.xml'

# ─── Stage 2: Build Docker Image ──────────────────────────────
- stage: BuildImage
  displayName: 'Build Docker Image'
  dependsOn: BuildAndTest
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  
  jobs:
  - job: BuildAndPush
    displayName: 'Build and Push to ACR'
    
    steps:
    - task: Docker@2
      displayName: 'Build and Push image'
      inputs:
        containerRegistry: 'ACR-Service-Connection'
        repository: $(imageRepository)
        command: buildAndPush
        Dockerfile: $(dockerfilePath)
        tags: |
          $(tag)
          latest

# ─── Stage 3: Deploy to Staging ───────────────────────────────
- stage: DeployStaging
  displayName: 'Deploy to Staging'
  dependsOn: BuildImage
  
  jobs:
  - deployment: DeployStaging
    displayName: 'Deploy to Staging AKS'
    environment: 'staging'
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@1
            displayName: 'Deploy to K8s'
            inputs:
              action: deploy
              kubernetesServiceConnection: 'AKS-Staging-Connection'
              namespace: staging
              manifests: |
                k8s/deployment.yaml
                k8s/service.yaml
              containers: $(containerRegistry)/$(imageRepository):$(tag)

# ─── Stage 4: Deploy to Production ────────────────────────────
- stage: DeployProduction
  displayName: 'Deploy to Production'
  dependsOn: DeployStaging
  
  jobs:
  - deployment: DeployProduction
    displayName: 'Deploy to Production AKS'
    environment: 'production'  # Có manual approval gate trong Azure DevOps
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@1
            displayName: 'Deploy to Production'
            inputs:
              action: deploy
              kubernetesServiceConnection: 'AKS-Production-Connection'
              namespace: production
              manifests: |
                k8s/deployment.yaml
                k8s/service.yaml
              containers: $(containerRegistry)/$(imageRepository):$(tag)
```

---

## 4. GitOps với ArgoCD — Declarative Deployment

GitOps — triết lý dùng Git làm source of truth — nguồn chân lý duy nhất cho infrastructure:

```
Developer commit
     │
     ▼
Git Repository (k8s manifests)
     │
     ▼
ArgoCD watches repo
     │
     ▼ (diff detected)
ArgoCD syncs to cluster
     │
     ▼
Kubernetes cluster updated
```

```yaml
# ArgoCD Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/myapp-k8s-config
    targetRevision: HEAD
    path: environments/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # xóa resource không còn trong Git
      selfHeal: true   # tự sửa nếu có drift — khác biệt với Git
    syncOptions:
      - CreateNamespace=true
```

---

## 5. Secrets trong CI/CD

### GitHub Actions Secrets

```yaml
# Cách truy cập secrets trong workflow
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

# Secrets cần tạo trong GitHub:
# Settings → Secrets and variables → Actions
# - AZURE_CREDENTIALS (JSON từ az ad sp create-for-rbac)
# - ACR_USERNAME
# - ACR_PASSWORD
# - SLACK_WEBHOOK_URL
```

```bash
# Tạo Service Principal — Đầu Mối Dịch Vụ cho GitHub Actions
az ad sp create-for-rbac \
    --name "github-actions-myapp" \
    --role contributor \
    --scopes /subscriptions/<subscription-id>/resourceGroups/myapp-rg \
    --sdk-auth
# Copy JSON output vào GitHub secret: AZURE_CREDENTIALS
```

### Azure DevOps Variable Groups

```yaml
# Dùng variable groups để share variables giữa pipelines
variables:
  - group: myapp-production-secrets   # Variable group trong Library

steps:
  - script: echo $(ConnectionString)  # Lấy từ variable group
```

---

## 6. Branch Strategy — Chiến Lược Nhánh

### GitHub Flow (Đơn Giản)

```
main ──────────────────────────────────────────► production
         ▲         ▲         ▲
    feature/A  feature/B  hotfix/X
```

- Mọi feature branch merge vào `main`
- `main` luôn deployable — có thể deploy
- Deploy production từ `main`

### GitFlow (Phức Tạp Hơn)

```
main ─────────────────────────────────────────► production
      ▲                   ▲
develop ──────────────────────────────────────► staging
      ▲         ▲         ▲
 feature/A  feature/B  release/1.2
```

---

## 7. Semantic Versioning — Đánh Số Phiên Bản Có Ngữ Nghĩa

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'  # trigger khi push tag v1.0.0, v2.1.3, etc.

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - name: Build release image
        run: |
          docker build \
            --build-arg VERSION=${{ steps.version.outputs.VERSION }} \
            -t myacr.azurecr.io/myapp-api:${{ steps.version.outputs.VERSION }} \
            -t myacr.azurecr.io/myapp-api:latest \
            .
```

---

## 8. Pipeline Best Practices

### Tối Ưu Tốc Độ Build

```yaml
# 1. Chạy các jobs độc lập song song
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps: [...]

  integration-tests:
    runs-on: ubuntu-latest
    steps: [...]

  security-scan:
    runs-on: ubuntu-latest
    steps: [...]

  build:
    needs: [unit-tests, integration-tests, security-scan]
    runs-on: ubuntu-latest
    steps: [...]

# 2. Cache aggressively
- uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}

# 3. Dùng matrix để test nhiều .NET versions
strategy:
  matrix:
    dotnet-version: ['6.0.x', '7.0.x', '8.0.x']
```

### Quality Gates — Cổng Chất Lượng

```yaml
# Không deploy nếu:
# 1. Test coverage < 80%
# 2. Critical security vulnerabilities
# 3. Build warnings
# 4. Code style violations

- name: Enforce quality gates
  run: |
    # Check coverage
    dotnet test --collect:"XPlat Code Coverage"
    
    # Check for vulnerabilities
    dotnet list package --vulnerable --include-transitive
    
    # Check warnings as errors
    dotnet build -warnaserror
```

---

## Checklist CI/CD .NET

- [ ] Pipeline chạy < 10 phút cho CI checks
- [ ] Unit tests và integration tests chạy tự động trên mọi PR
- [ ] Code coverage threshold được enforce
- [ ] Security scan (SAST, dependency check) trong pipeline
- [ ] Docker image được scan trước khi push
- [ ] Secrets không được hardcode trong pipeline YAML
- [ ] Production deployment yêu cầu manual approval
- [ ] Rollback tự động khi deployment thất bại
- [ ] Notifications (Slack/Teams) khi build fail hoặc deploy thành công
- [ ] Artifacts (Docker images) được tag với build number/commit SHA

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác nhau giữa CI và CD?**
> CI — Continuous Integration: mỗi commit được tự động build và test để phát hiện lỗi sớm. CD — Continuous Delivery: sau CI thành công, code sẵn sàng được deploy bất kỳ lúc nào (nhưng có thể cần manual approval). Continuous Deployment (khác với Delivery): mọi CI pass đều tự động deploy lên production.

**Q: Blue-Green vs Canary deployment — khi nào dùng gì?**
> Blue-Green: switch 100% traffic ngay lập tức từ v1 sang v2, rollback dễ bằng cách switch lại. Tốn gấp đôi tài nguyên. Canary: phát hành dần dần (1% → 10% → 100% traffic), cho phép phát hiện lỗi ở production với ít người dùng bị ảnh hưởng. Phức tạp hơn nhưng ít rủi ro.

**Q: Tại sao không nên commit secrets vào git?**
> Khi secrets ở trong git history, chúng ở đó mãi mãi kể cả sau khi delete file. Bất kỳ ai có access vào repo đều có thể đọc. Secrets cần được rotate — luân phiên thường xuyên. Dùng git-secrets, pre-commit hooks để phòng ngừa, và Key Vault / CI/CD secret stores cho production.

**Cập Nhật Lần Cuối:** 2026-06-02
