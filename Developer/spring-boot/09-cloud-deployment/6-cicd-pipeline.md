# CI/CD Pipeline — GitHub Actions, Jenkins & Maven/Gradle CI

> CI/CD — Continuous Integration / Continuous Deployment (Tích Hợp Liên Tục / Triển Khai Liên Tục) — tự động hóa quá trình build, test, và deploy Spring Boot application. Hướng dẫn này bao gồm thiết kế pipeline từ code đến production với GitHub Actions, Jenkins, và các best practices quản lý secrets và rollback.

---

## 1. Tổng Quan CI/CD Pipeline

```
Developer
    │
    │ git push
    ▼
Git Repository (GitHub/GitLab)
    │
    │ Trigger CI Pipeline
    ▼
┌───────────────────────────────────────────────────────────┐
│                    CI Pipeline                            │
│  1. Checkout code                                         │
│  2. Setup Java & Cache Maven/Gradle                       │
│  3. Lint & Code Analysis (SonarQube/Checkstyle)           │
│  4. Unit Tests (JUnit 5 + Mockito)                        │
│  5. Integration Tests (Testcontainers)                    │
│  6. Code Coverage Check (JaCoCo ≥ 80%)                   │
│  7. Security Scan (OWASP Dependency Check / Snyk)         │
│  8. Build JAR / Docker Image                              │
│  9. Push image đến Container Registry                     │
└───────────────────────────────────────────────────────────┘
    │
    │ (Chỉ khi merge vào main/release)
    ▼
┌───────────────────────────────────────────────────────────┐
│                    CD Pipeline                            │
│  1. Deploy đến Staging Environment (môi trường staging)   │
│  2. Smoke Tests & Integration Tests                       │
│  3. Manual Approval Gate (cổng phê duyệt thủ công)       │
│  4. Deploy đến Production                                 │
│  5. Health Check verification                             │
│  6. Notify team (Slack/Teams)                             │
└───────────────────────────────────────────────────────────┘
```

---

## 2. GitHub Actions — CI/CD Với GitHub

### Pipeline CI Cơ Bản

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop, 'release/**']
  pull_request:
    branches: [main, develop]
  # Cho phép chạy thủ công
  workflow_dispatch:

# Giới hạn concurrent runs — hủy run cũ nếu có run mới
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  JAVA_VERSION: '21'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─────────────────────────────────────────
  # Job 1: Test — Kiểm Thử
  # ─────────────────────────────────────────
  test:
    name: Test & Code Quality
    runs-on: ubuntu-latest
    
    # Services cần thiết cho integration tests
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Cần full history cho SonarQube

      - name: Setup Java ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          # Cache Maven dependencies — Bộ Nhớ Đệm Dependencies
          cache: maven

      # Cache Maven packages thủ công (backup nếu setup-java cache không hoạt động)
      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-maven-

      - name: Run tests with coverage
        run: ./mvnw verify -Pcoverage
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test

      - name: Upload test reports
        uses: actions/upload-artifact@v4
        if: always()  # Upload ngay cả khi test fail
        with:
          name: test-reports
          path: target/surefire-reports/

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: target/site/jacoco/jacoco.xml

      # SonarQube analysis — phân tích chất lượng code
      - name: SonarQube Scan
        if: github.event_name != 'pull_request'  # Chỉ scan trên main branch
        run: ./mvnw sonar:sonar
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

  # ─────────────────────────────────────────
  # Job 2: Security Scan — Quét Bảo Mật
  # ─────────────────────────────────────────
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'my-spring-app'
          path: '.'
          format: 'HTML'
        env:
          JAVA_HOME: /opt/jdk

      - name: Upload OWASP results
        uses: actions/upload-artifact@v4
        with:
          name: owasp-report
          path: reports/

  # ─────────────────────────────────────────
  # Job 3: Build & Push Docker Image
  # ─────────────────────────────────────────
  build-image:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [test]  # Chỉ build nếu test pass
    permissions:
      contents: read
      packages: write

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: maven

      # Build JAR
      - name: Build JAR
        run: ./mvnw package -DskipTests -q

      # Docker buildx — build multi-platform images
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Login vào GitHub Container Registry
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Tạo metadata cho image tags
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            # Tag với SHA commit ngắn
            type=sha,prefix={{branch}}-,format=short
            # Tag với branch name
            type=ref,event=branch
            # Tag semantic version nếu là git tag
            type=semver,pattern={{version}}
            # latest chỉ cho main branch
            type=raw,value=latest,enable={{is_default_branch}}

      # Build và push image
      - name: Build and Push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          # Cache layers từ registry để tăng tốc
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
          platforms: linux/amd64,linux/arm64

      # Scan image với Trivy
      - name: Scan Docker image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'  # Fail pipeline nếu tìm thấy lỗ hổng nghiêm trọng

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

### Pipeline CD — Deploy Đến Production

```yaml
# .github/workflows/deploy.yml
name: CD Pipeline — Deploy

on:
  workflow_run:
    workflows: ["CI Pipeline"]
    branches: [main]
    types: [completed]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]
      image-tag:
        description: 'Docker image tag to deploy'
        required: true

jobs:
  # ─────────────────────────────────────────
  # Deploy to Staging — Triển Khai Staging
  # ─────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    environment:
      name: staging
      url: https://staging.api.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v4
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.STAGING_KUBECONFIG }}

      - name: Deploy to Staging
        run: |
          # Thay thế image tag trong manifest
          sed -i "s|IMAGE_TAG|${{ github.sha }}|g" k8s/overlays/staging/kustomization.yaml
          kubectl apply -k k8s/overlays/staging/
          
          # Chờ rollout hoàn thành
          kubectl rollout status deployment/myapp \
            --namespace=staging \
            --timeout=5m

      - name: Run smoke tests
        run: |
          # Chờ service ready
          sleep 30
          # Test endpoint cơ bản
          curl --fail https://staging.api.example.com/actuator/health

  # ─────────────────────────────────────────
  # Manual Approval — Phê Duyệt Thủ Công
  # ─────────────────────────────────────────
  approval:
    name: Production Approval
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    environment:
      name: production-approval  # Environment có required reviewers
    steps:
      - name: Waiting for approval
        run: echo "Production deployment approved"

  # ─────────────────────────────────────────
  # Deploy to Production — Triển Khai Production
  # ─────────────────────────────────────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [approval]
    environment:
      name: production
      url: https://api.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v4
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.PROD_KUBECONFIG }}

      - name: Deploy to Production
        run: |
          sed -i "s|IMAGE_TAG|${{ github.sha }}|g" k8s/overlays/production/kustomization.yaml
          kubectl apply -k k8s/overlays/production/
          kubectl rollout status deployment/myapp \
            --namespace=production \
            --timeout=10m

      - name: Verify deployment health
        run: |
          # Chờ HPA ổn định
          sleep 60
          kubectl get hpa -n production
          # Kiểm tra không có pod crash
          CRASH_COUNT=$(kubectl get pods -n production -l app=myapp \
            -o jsonpath='{.items[*].status.containerStatuses[*].restartCount}' \
            | tr ' ' '\n' | awk '{sum+=$1} END {print sum}')
          if [ "$CRASH_COUNT" -gt "0" ]; then
            echo "Pods đang crash! Tiến hành rollback..."
            kubectl rollout undo deployment/myapp -n production
            exit 1
          fi

      - name: Notify Slack on success
        uses: slackapi/slack-github-action@v1
        if: success()
        with:
          payload: |
            {
              "text": "✅ Production deployment successful: ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on failure
        uses: slackapi/slack-github-action@v1
        if: failure()
        with:
          payload: |
            {
              "text": "❌ Production deployment FAILED: ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 3. Maven CI Configuration — Cấu Hình Maven Cho CI

### `pom.xml` Profiles Cho CI

```xml
<profiles>
    <!-- Profile chạy tests với coverage trong CI -->
    <profile>
        <id>coverage</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.jacoco</groupId>
                    <artifactId>jacoco-maven-plugin</artifactId>
                    <executions>
                        <execution>
                            <id>prepare-agent</id>
                            <goals><goal>prepare-agent</goal></goals>
                        </execution>
                        <execution>
                            <id>report</id>
                            <phase>test</phase>
                            <goals><goal>report</goal></goals>
                        </execution>
                        <execution>
                            <id>check</id>
                            <goals><goal>check</goal></goals>
                            <configuration>
                                <rules>
                                    <rule>
                                        <element>BUNDLE</element>
                                        <limits>
                                            <!-- Yêu cầu ≥ 80% line coverage -->
                                            <limit>
                                                <counter>LINE</counter>
                                                <value>COVEREDRATIO</value>
                                                <minimum>0.80</minimum>
                                            </limit>
                                            <!-- Yêu cầu ≥ 75% branch coverage -->
                                            <limit>
                                                <counter>BRANCH</counter>
                                                <value>COVEREDRATIO</value>
                                                <minimum>0.75</minimum>
                                            </limit>
                                        </limits>
                                    </rule>
                                </rules>
                                <excludes>
                                    <!-- Loại trừ generated code -->
                                    <exclude>**/*Application.class</exclude>
                                    <exclude>**/dto/**</exclude>
                                    <exclude>**/config/**</exclude>
                                </excludes>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
    
    <!-- Profile skip tests nhanh cho local build -->
    <profile>
        <id>fast</id>
        <properties>
            <maven.test.skip>true</maven.test.skip>
            <skipTests>true</skipTests>
        </properties>
    </profile>
</profiles>
```

### `.mvn/maven.config` — Cấu Hình Maven Mặc Định

```
# .mvn/maven.config
# Số threads parallel (theo số CPU)
--threads 1C

# Tắt transfer progress trong CI (logs sạch hơn)
--no-transfer-progress

# Batch mode — không interactive, phù hợp CI
--batch-mode
```

---

## 4. Jenkins Pipeline

### Jenkinsfile (Declarative Pipeline)

```groovy
// Jenkinsfile
pipeline {
    agent {
        // Chạy trong Docker container — không cần cài Java trên Jenkins
        docker {
            image 'eclipse-temurin:21-jdk-alpine'
            args '-v $HOME/.m2:/root/.m2'  // Cache Maven repository
        }
    }
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'my-org/myapp'
        // Credentials từ Jenkins Credentials Store
        REGISTRY_CREDS = credentials('registry-credentials')
        SONAR_TOKEN = credentials('sonar-token')
        KUBECONFIG = credentials('prod-kubeconfig')
    }
    
    options {
        // Timeout cho toàn bộ pipeline
        timeout(time: 1, unit: 'HOURS')
        // Giữ 10 builds gần nhất
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Timestamps trong logs
        timestamps()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log --oneline -5'
            }
        }
        
        stage('Test') {
            steps {
                sh './mvnw verify -Pcoverage --no-transfer-progress'
            }
            post {
                always {
                    // Publish JUnit test results
                    junit 'target/surefire-reports/**/*.xml'
                    // Publish JaCoCo coverage report
                    jacoco(
                        execPattern: 'target/*.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java',
                        minimumLineCoverage: '80',
                        minimumBranchCoverage: '75'
                    )
                }
            }
        }
        
        stage('Code Analysis') {
            when {
                branch 'main'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh './mvnw sonar:sonar --no-transfer-progress'
                }
                // Quality Gate — Cổng Chất Lượng: chờ SonarQube phân tích xong
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def gitCommit = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short HEAD'
                    ).trim()
                    
                    def imageTag = "${env.REGISTRY}/${env.IMAGE_NAME}:${gitCommit}"
                    
                    docker.build(imageTag, '--no-cache .')
                    
                    // Lưu tag để dùng ở stages sau
                    env.IMAGE_TAG = imageTag
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                // Trivy scan
                sh """
                    trivy image --exit-code 1 \
                      --severity CRITICAL,HIGH \
                      --no-progress \
                      ${env.IMAGE_TAG}
                """
            }
        }
        
        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry("https://${env.REGISTRY}", 'registry-credentials') {
                        docker.image(env.IMAGE_TAG).push()
                        // Push 'latest' tag nếu là main branch
                        if (env.BRANCH_NAME == 'main') {
                            docker.image(env.IMAGE_TAG).push('latest')
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                withKubeConfig([credentialsId: 'staging-kubeconfig']) {
                    sh """
                        kubectl set image deployment/myapp \
                          myapp=${env.IMAGE_TAG} \
                          -n staging
                        kubectl rollout status deployment/myapp \
                          -n staging \
                          --timeout=5m
                    """
                }
            }
        }
        
        stage('Integration Tests on Staging') {
            when {
                branch 'main'
            }
            steps {
                sh './mvnw test -Pintegration-tests -Dbase-url=https://staging.api.example.com'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            // Yêu cầu input từ người dùng (approval gate)
            input {
                message "Deploy ${env.IMAGE_TAG} to Production?"
                ok "Deploy"
                submitter "devops-team"
            }
            steps {
                withKubeConfig([credentialsId: 'prod-kubeconfig']) {
                    sh """
                        kubectl set image deployment/myapp \
                          myapp=${env.IMAGE_TAG} \
                          -n production
                        kubectl rollout status deployment/myapp \
                          -n production \
                          --timeout=10m
                    """
                }
            }
        }
    }
    
    post {
        success {
            slackSend(
                color: 'good',
                message: "✅ Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${env.IMAGE_TAG}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "❌ Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
            // Auto rollback production nếu deploy fail
            script {
                if (env.BRANCH_NAME == 'main') {
                    withKubeConfig([credentialsId: 'prod-kubeconfig']) {
                        sh 'kubectl rollout undo deployment/myapp -n production || true'
                    }
                }
            }
        }
        always {
            cleanWs()  // Dọn dẹp workspace sau mỗi build
        }
    }
}
```

---

## 5. Gradle CI Configuration

```groovy
// build.gradle.kts — Kotlin DSL
plugins {
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
    id("jacoco")
    id("org.sonarqube") version "5.0.0.4638"
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24"
}

// JaCoCo configuration
jacoco {
    toolVersion = "0.8.12"
}

tasks.jacocoTestReport {
    reports {
        xml.required = true  // Cho SonarQube
        html.required = true
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()  // 80% coverage tối thiểu
            }
        }
    }
}

// SonarQube
sonarqube {
    properties {
        property("sonar.projectKey", "my-project")
        property("sonar.host.url", System.getenv("SONAR_HOST_URL"))
        property("sonar.login", System.getenv("SONAR_TOKEN"))
        property("sonar.coverage.jacoco.xmlReportPaths", 
            "${project.buildDir}/reports/jacoco/test/jacocoTestReport.xml")
    }
}

// Task CI — chạy test + coverage check
tasks.register("ci") {
    dependsOn("test", "jacocoTestReport", "jacocoTestCoverageVerification")
    description = "Run CI checks: test + coverage"
}
```

---

## 6. Quản Lý Secrets Trong CI/CD

### GitHub Actions Secrets

```yaml
# Sử dụng secrets trong workflow
steps:
  - name: Build and push
    env:
      DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}
      JWT_SECRET: ${{ secrets.JWT_SECRET }}
    run: |
      # Secrets tự động được masked trong logs
      echo "Deploying with secure config..."
```

### Vault Integration Trong CI

```yaml
- name: Import secrets from Vault
  uses: hashicorp/vault-action@v3
  with:
    url: https://vault.example.com
    token: ${{ secrets.VAULT_TOKEN }}
    secrets: |
      secret/data/myapp/prod db_password | DB_PASSWORD ;
      secret/data/myapp/prod jwt_secret | JWT_SECRET

- name: Deploy with vault secrets
  run: kubectl create secret generic myapp-secrets \
    --from-literal=DB_PASSWORD="$DB_PASSWORD" \
    --from-literal=JWT_SECRET="$JWT_SECRET" \
    --dry-run=client -o yaml | kubectl apply -f -
```

---

## 7. Chiến Lược Branching & Deployment

### GitFlow Cơ Bản

```
main (production)
  ↑ merge qua PR + approval
release/1.x.0
  ↑ merge
develop
  ↑ merge qua PR
feature/TICKET-123-add-payment
```

### Deployment Strategy — Chiến Lược Triển Khai

| Chiến Lược | Mô Tả | Ưu Điểm | Nhược Điểm |
|-----------|-------|---------|------------|
| **Rolling Update** | Thay thế dần Pod cũ | Zero downtime, dễ rollback | Tạm thời có 2 version chạy song song |
| **Blue-Green** | Hai môi trường giống nhau, switch traffic | Rollback instant | Tốn gấp đôi tài nguyên |
| **Canary** | Route % nhỏ traffic đến version mới | Kiểm tra rủi ro thấp | Phức tạp hơn |
| **Recreate** | Xóa hết Pod cũ rồi tạo mới | Đơn giản | Downtime |

### Canary Deployment Với Kubernetes

```yaml
# Deployment cũ: 9 replicas (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable

---
# Deployment mới: 1 replica (10% traffic) - canary
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      track: canary

---
# Service route đến cả hai (dựa trên ratio replicas)
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp  # Match cả stable và canary
```

---

## 8. Pipeline Best Practices (Thực Hành Tốt Nhất)

### Tốc Độ Pipeline

```yaml
# Cache Maven dependencies
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

# Chạy jobs parallel khi độc lập nhau
jobs:
  test:
    ...
  security-scan:  # Chạy song song với test
    ...
  lint:           # Chạy song song với test
    ...
  build-image:
    needs: [test, security-scan, lint]  # Chờ tất cả pass
```

### Fail Fast Strategy — Thất Bại Nhanh

```yaml
# Chạy quick checks trước khi slow tests
stages:
  - compile          # Nhanh nhất: 30s
  - unit-test        # Nhanh: 2-3 phút
  - integration-test # Chậm: 5-10 phút
  - build-image      # Chậm: 3-5 phút
  - deploy-staging   # Phụ thuộc môi trường
```

### Idempotent Deployments — Triển Khai Bất Biến

```bash
# Dùng kubectl apply thay vì create (idempotent)
kubectl apply -f manifests/

# Dùng --dry-run để xem trước thay đổi
kubectl apply -f manifests/ --dry-run=server
```

---

## 9. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: CI và CD khác nhau thế nào?**

A: **CI — Continuous Integration (Tích Hợp Liên Tục):** tự động build và test mỗi khi có code mới, đảm bảo code luôn compile và test pass. **CD — Continuous Deployment (Triển Khai Liên Tục):** tự động deploy lên production sau khi CI pass, không cần can thiệp thủ công. Một số tổ chức dùng Continuous Delivery — chỉ tự động đến staging, production cần approval thủ công.

**Q: Làm thế nào để rollback nhanh khi deploy lỗi?**

A: (1) `kubectl rollout undo deployment/myapp` — rollback về revision trước trong K8s; (2) Blue-Green deployment — switch traffic ngay về environment cũ; (3) Feature flags — tắt feature mới mà không deploy lại; (4) Gitops với ArgoCD — revert commit trong Git, ArgoCD tự sync.

**Q: Pipeline nên chạy tests theo thứ tự nào?**

A: Fail fast: (1) Compile/lint — vài giây; (2) Unit tests — vài phút; (3) Integration tests — 5-10 phút; (4) Security scan — song song; (5) Build image; (6) Deploy staging + smoke tests; (7) Deploy production. Đặt tests nhanh nhất lên đầu để phát hiện lỗi sớm mà không tốn thời gian chạy slow tests.

**Q: Cách quản lý secrets trong CI/CD an toàn?**

A: (1) Không commit secrets vào Git — dùng `.gitignore` và pre-commit hooks; (2) Lưu trong CI/CD secrets store (GitHub Secrets, Jenkins Credentials); (3) Dùng Vault cho secrets phức tạp; (4) Inject vào runtime qua environment variables hoặc Kubernetes Secrets; (5) Rotate secrets định kỳ và revoke ngay khi có rủi ro.

---

## ✅ Checklist

- [ ] Pipeline có đủ 3 giai đoạn: test → build → deploy
- [ ] Test phải pass trước khi build Docker image
- [ ] Docker image được scan với Trivy/Snyk
- [ ] Image tag dùng git SHA (không dùng `latest`)
- [ ] Maven/Gradle dependencies được cache để tăng tốc
- [ ] Secrets lưu trong CI/CD secrets store, không hardcode
- [ ] Deployment có approval gate cho production
- [ ] Health check sau deploy — rollback tự động nếu fail
- [ ] Notification khi build fail hoặc deploy thành công
- [ ] Code coverage ≥ 80% được enforce trong pipeline
- [ ] SonarQube quality gate được tích hợp
- [ ] Branching strategy được định nghĩa rõ ràng
