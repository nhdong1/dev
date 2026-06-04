# Kubernetes Deployment — Triển Khai Lên Kubernetes Từ Jenkins

> Kubernetes Deployment Integration cho phép Jenkins tự động triển khai ứng dụng lên Kubernetes cluster (cụm Kubernetes) bằng `kubectl` và Helm — là mắt xích cuối cùng trong pipeline CD (Continuous Delivery — Phân Phối Liên Tục).

---

## Mục Lục

1. [Tổng Quan Kiến Trúc](#1-tổng-quan-kiến-trúc)
2. [Xác Thực với Kubernetes Cluster](#2-xác-thực-với-kubernetes-cluster)
3. [Deploy Bằng kubectl](#3-deploy-bằng-kubectl)
4. [Deploy Bằng Helm](#4-deploy-bằng-helm)
5. [Multi-environment Deployment](#5-multi-environment-deployment)
6. [Rollback Strategy (Chiến Lược Rollback)](#6-rollback-strategy-chiến-lược-rollback)
7. [Deployment Verification (Xác Minh Triển Khai)](#7-deployment-verification-xác-minh-triển-khai)
8. [GitOps với Jenkins](#8-gitops-với-jenkins)
9. [Best Practices](#9-best-practices)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Kiến Trúc

```
Jenkins Pipeline → Deploy to Kubernetes
────────────────────────────────────────

Jenkins (CI/CD Server)
     │
     │  kubeconfig (file cấu hình cluster)
     ▼
Kubernetes API Server
     │
     ├── Namespace: production
     │       └── Deployment: myapp → ReplicaSet → Pods
     │
     ├── Namespace: staging
     │       └── Deployment: myapp → ReplicaSet → Pods
     │
     └── Namespace: development
             └── Deployment: myapp → ReplicaSet → Pods
```

### Hai Phương Pháp Triển Khai Chính

| Phương Pháp | Công Cụ | Phù Hợp | Độ Phức Tạp |
|-------------|---------|---------|------------|
| Imperative (Khai báo tường minh) | kubectl set image | Deploy đơn giản, nhanh | ⭐ |
| Declarative (Khai báo) | kubectl apply -f | Quản lý config theo file | ⭐⭐ |
| Package Manager | Helm | App phức tạp, multi-env | ⭐⭐⭐ |

---

## 2. Xác Thực với Kubernetes Cluster

### 2.1 Kubeconfig — File Cấu Hình Cluster

Kubeconfig là file chứa thông tin để `kubectl` kết nối và xác thực với Kubernetes cluster.

```yaml
# Cấu trúc kubeconfig điển hình
apiVersion: v1
kind: Config
clusters:
  - name: production-cluster
    cluster:
      server: https://k8s-api.mycompany.com:6443
      certificate-authority-data: <base64-encoded-CA-cert>
contexts:
  - name: production
    context:
      cluster: production-cluster
      user: jenkins-sa
      namespace: default
current-context: production
users:
  - name: jenkins-sa
    user:
      token: <service-account-token>
```

### 2.2 Lưu Kubeconfig Vào Jenkins Credentials

```
Manage Jenkins → Credentials → System → Global credentials
→ Add Credentials:
   Kind: Secret file
   File: upload file kubeconfig
   ID: kubeconfig-production
   Description: Kubernetes Production Cluster Config
```

### 2.3 Service Account (Tài Khoản Dịch Vụ) Cho Jenkins

Thay vì dùng admin kubeconfig, tạo Service Account riêng cho Jenkins với quyền hạn tối thiểu:

```yaml
# jenkins-rbac.yaml — Role-Based Access Control cho Jenkins
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-deployer
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-deploy-role
  namespace: production
rules:
  # Quyền tối thiểu cần thiết để deploy
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-deploy-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: jenkins-deployer
    namespace: production
roleRef:
  kind: Role
  name: jenkins-deploy-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Apply RBAC config
kubectl apply -f jenkins-rbac.yaml

# Lấy token của Service Account (Kubernetes v1.24+)
kubectl create token jenkins-deployer -n production --duration=8760h
```

### 2.4 Cài Plugin Kubernetes CLI

```
Plugin Manager → Cài: "Kubernetes CLI Plugin"
```

---

## 3. Deploy Bằng kubectl

### 3.1 Cập Nhật Image (Rolling Update — Cập Nhật Cuốn Chiếu)

```groovy
pipeline {
    agent any

    environment {
        K8S_NAMESPACE  = 'production'
        DEPLOYMENT     = 'myapp'
        CONTAINER      = 'myapp-container'
        IMAGE          = "myregistry/myapp:${env.BUILD_NUMBER}"
    }

    stages {
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-production']) {
                    // Phương pháp 1: kubectl set image — cập nhật image trực tiếp
                    sh """
                        kubectl set image deployment/${DEPLOYMENT} \
                            ${CONTAINER}=${IMAGE} \
                            -n ${K8S_NAMESPACE}
                    """

                    // Gắn annotation để track deploy history (lịch sử triển khai)
                    sh """
                        kubectl annotate deployment/${DEPLOYMENT} \
                            kubernetes.io/change-cause="Jenkins Build #${env.BUILD_NUMBER} - ${env.GIT_COMMIT[0..7]}" \
                            -n ${K8S_NAMESPACE} \
                            --overwrite
                    """
                }
            }
        }

        stage('Verify Rollout') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-production']) {
                    // rollout status — chờ deployment hoàn thành (hoặc timeout)
                    sh """
                        kubectl rollout status deployment/${DEPLOYMENT} \
                            -n ${K8S_NAMESPACE} \
                            --timeout=5m
                    """
                }
            }
        }
    }
}
```

### 3.2 Deploy Bằng Manifest File (File Khai Báo)

```groovy
stage('Deploy Manifests') {
    steps {
        withKubeConfig([credentialsId: 'kubeconfig-production']) {
            // Thay thế biến IMAGE_TAG trong manifest trước khi apply
            sh """
                sed -i 's|IMAGE_TAG_PLACEHOLDER|${IMAGE_TAG}|g' k8s/deployment.yaml
                kubectl apply -f k8s/
                kubectl rollout status deployment/myapp -n production --timeout=5m
            """
        }
    }
}
```

```yaml
# k8s/deployment.yaml — file manifest với placeholder
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp-container
          # IMAGE_TAG_PLACEHOLDER được thay thế bởi sed trong pipeline
          image: myregistry/myapp:IMAGE_TAG_PLACEHOLDER
          ports:
            - containerPort: 8080
          # Resource requests và limits — quan trọng cho scheduling
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          # Liveness probe — kiểm tra container còn sống không
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          # Readiness probe — kiểm tra container sẵn sàng nhận traffic chưa
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

### 3.3 Kustomize (Công Cụ Tùy Biến Manifest)

```groovy
stage('Deploy with Kustomize') {
    steps {
        withKubeConfig([credentialsId: 'kubeconfig-production']) {
            // Kustomize — tùy biến manifest theo environment mà không cần template engine
            dir('k8s/overlays/production') {
                sh """
                    kustomize edit set image myapp=myregistry/myapp:${IMAGE_TAG}
                    kustomize build . | kubectl apply -f -
                """
            }
        }
    }
}
```

---

## 4. Deploy Bằng Helm

### 4.1 Helm — Package Manager Cho Kubernetes

Helm (quản lý gói cho Kubernetes) sử dụng "chart" (gói cấu hình) để đóng gói và quản lý ứng dụng Kubernetes.

```
myapp-chart/
├── Chart.yaml          # Metadata của chart (tên, version, description)
├── values.yaml         # Giá trị mặc định
├── values-staging.yaml # Giá trị riêng cho staging
├── values-prod.yaml    # Giá trị riêng cho production
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── configmap.yaml
```

### 4.2 Jenkinsfile — Helm Deploy

```groovy
pipeline {
    agent any

    environment {
        HELM_RELEASE   = 'myapp'
        HELM_CHART_DIR = './helm/myapp-chart'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Helm Lint') {
            steps {
                // Kiểm tra cú pháp chart trước khi deploy
                sh "helm lint ${HELM_CHART_DIR}"
            }
        }

        stage('Deploy to Staging') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-staging']) {
                    sh """
                        helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_DIR} \
                            --namespace staging \
                            --create-namespace \
                            --values ${HELM_CHART_DIR}/values-staging.yaml \
                            --set image.tag=${IMAGE_TAG} \
                            --set image.repository=myregistry/myapp \
                            --wait \
                            --timeout 5m \
                            --atomic
                    """
                    // --atomic: nếu deploy fail → tự động rollback về version trước
                    // --wait:   chờ đến khi tất cả resources ready
                }
            }
        }

        stage('Integration Tests') {
            steps {
                sh './scripts/run-integration-tests.sh staging'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                // Yêu cầu xác nhận thủ công trước khi deploy production
                input message: "Deploy ${IMAGE_TAG} to Production?",
                      ok: 'Deploy',
                      submitter: 'tech-lead,devops-team'

                withKubeConfig([credentialsId: 'kubeconfig-production']) {
                    sh """
                        helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_DIR} \
                            --namespace production \
                            --values ${HELM_CHART_DIR}/values-prod.yaml \
                            --set image.tag=${IMAGE_TAG} \
                            --wait \
                            --timeout 10m \
                            --atomic \
                            --history-max 5
                    """
                    // --history-max 5: giữ tối đa 5 revision lịch sử để rollback
                }
            }
        }
    }
}
```

### 4.3 Helm Values File Mẫu

```yaml
# values.yaml — giá trị mặc định
replicaCount: 2

image:
  repository: myregistry/myapp
  tag: latest  # Được ghi đè bởi --set image.tag=${IMAGE_TAG}
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: myapp.example.com

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

```yaml
# values-prod.yaml — ghi đè cho production
replicaCount: 5

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
```

---

## 5. Multi-environment Deployment

### 5.1 Pipeline Đa Môi Trường

```groovy
pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['development', 'staging', 'production'],
            description: 'Môi trường triển khai'
        )
    }

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Build & Push Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def img = docker.build("myorg/myapp:${IMAGE_TAG}")
                        img.push("${IMAGE_TAG}")
                        img.push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def kubeCredId = "kubeconfig-${params.ENVIRONMENT}"
                    def namespace  = params.ENVIRONMENT
                    def valuesFile = "values-${params.ENVIRONMENT}.yaml"

                    // Production yêu cầu manual approval
                    if (params.ENVIRONMENT == 'production') {
                        input message: "Confirm deploy to PRODUCTION?",
                              ok: 'Confirm',
                              submitter: 'devops-lead'
                    }

                    withKubeConfig([credentialsId: kubeCredId]) {
                        sh """
                            helm upgrade --install myapp ./helm/myapp-chart \
                                --namespace ${namespace} \
                                --values ./helm/myapp-chart/${valuesFile} \
                                --set image.tag=${IMAGE_TAG} \
                                --atomic \
                                --wait \
                                --timeout 10m
                        """
                    }
                }
            }
        }
    }
}
```

---

## 6. Rollback Strategy (Chiến Lược Rollback)

### 6.1 Rollback Với kubectl

```groovy
stage('Rollback on Failure') {
    steps {
        script {
            try {
                withKubeConfig([credentialsId: 'kubeconfig-production']) {
                    sh "kubectl set image deployment/myapp myapp=myregistry/myapp:${IMAGE_TAG} -n production"
                    sh "kubectl rollout status deployment/myapp -n production --timeout=5m"
                }
            } catch (Exception e) {
                echo "Deploy failed — initiating rollback"
                withKubeConfig([credentialsId: 'kubeconfig-production']) {
                    // Rollback về revision trước đó
                    sh "kubectl rollout undo deployment/myapp -n production"
                    sh "kubectl rollout status deployment/myapp -n production --timeout=5m"
                }
                // Báo lỗi để build thất bại
                error("Deployment failed and rolled back: ${e.message}")
            }
        }
    }
}
```

### 6.2 Rollback Với Helm

```bash
# Xem lịch sử các lần release
helm history myapp -n production

# OUTPUT:
# REVISION  UPDATED                   STATUS     CHART        APP VERSION  DESCRIPTION
# 1         2026-05-01 10:00:00       superseded myapp-1.0.0  1.0.0        Install complete
# 2         2026-05-10 14:30:00       superseded myapp-1.0.1  1.0.1        Upgrade complete
# 3         2026-05-11 09:00:00       deployed   myapp-1.0.2  1.0.2        Upgrade complete

# Rollback về revision 2
helm rollback myapp 2 -n production

# Rollback về revision trước đó
helm rollback myapp -n production
```

---

## 7. Deployment Verification (Xác Minh Triển Khai)

### 7.1 Health Check Sau Deploy

```groovy
stage('Post-Deploy Verification') {
    steps {
        withKubeConfig([credentialsId: 'kubeconfig-production']) {
            script {
                // Kiểm tra tất cả pods đang chạy
                sh """
                    kubectl get pods -n production -l app=myapp
                    kubectl rollout status deployment/myapp -n production --timeout=5m
                """

                // Kiểm tra số lượng pods sẵn sàng (ready)
                def readyPods = sh(
                    script: "kubectl get deployment myapp -n production -o jsonpath='{.status.readyReplicas}'",
                    returnStdout: true
                ).trim()

                def desiredPods = sh(
                    script: "kubectl get deployment myapp -n production -o jsonpath='{.spec.replicas}'",
                    returnStdout: true
                ).trim()

                if (readyPods != desiredPods) {
                    error("Only ${readyPods}/${desiredPods} pods are ready")
                }

                echo "Deployment verified: ${readyPods}/${desiredPods} pods ready"
            }
        }
    }
}
```

### 7.2 Smoke Test (Kiểm Tra Khói — Test Nhanh Sau Deploy)

```groovy
stage('Smoke Test') {
    steps {
        script {
            // Lấy URL của service
            def serviceUrl = sh(
                script: "kubectl get service myapp -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                returnStdout: true
            ).trim()

            // Gọi health endpoint để xác nhận ứng dụng đang hoạt động
            retry(5) {
                sleep(10)
                sh "curl -f http://${serviceUrl}/health"
            }

            echo "Smoke test passed: ${serviceUrl}/health responded successfully"
        }
    }
}
```

---

## 8. GitOps với Jenkins

### 8.1 GitOps Pattern (Mô Hình GitOps)

GitOps là phương pháp sử dụng Git repository làm nguồn sự thật (source of truth) duy nhất cho cả application code và infrastructure configuration:

```
GitOps Flow với Jenkins + ArgoCD:
──────────────────────────────────

Developer push code
      │
      ▼
App Repository (code change)
      │
      │  Jenkins CI Pipeline
      ▼
Docker Image → Push to Registry
      │
      │  Jenkins CD Pipeline
      ▼
Config Repository (cập nhật image tag trong values.yaml)
      │
      │  ArgoCD / FluxCD phát hiện thay đổi
      ▼
Kubernetes Cluster (tự động sync)
```

### 8.2 Jenkins Pipeline Cập Nhật Config Repository

```groovy
stage('Update Config Repository') {
    steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'config-repo-key', keyFileVariable: 'SSH_KEY')]) {
            sh """
                export GIT_SSH_COMMAND="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no"

                # Clone config repository
                git clone git@github.com:myorg/k8s-configs.git
                cd k8s-configs

                # Cập nhật image tag trong values file
                sed -i 's|tag: .*|tag: "${IMAGE_TAG}"|g' apps/myapp/values-production.yaml

                # Commit và push thay đổi
                git config user.email "jenkins@mycompany.com"
                git config user.name "Jenkins CI"
                git add apps/myapp/values-production.yaml
                git commit -m "chore: update myapp image to ${IMAGE_TAG} [ci skip]"
                git push origin main
            """
        }
    }
}
```

---

## 9. Best Practices

### Namespace Strategy (Chiến Lược Phân Vùng Namespace)

```yaml
# Tổ chức namespace theo môi trường
namespaces:
  - development    # Dành cho feature branch builds
  - staging        # Môi trường test trước production
  - production     # Production workloads

# Mỗi namespace có:
# - Resource quota (giới hạn tài nguyên)
# - Network policy (chính sách mạng)
# - Dedicated Service Account cho Jenkins
```

### Image Pull Policy (Chính Sách Kéo Image)

```yaml
# ❌ Tránh dùng Always cho production — tốn băng thông, chậm hơn
imagePullPolicy: Always

# ✅ Dùng IfNotPresent kết hợp với immutable tags (tag cố định)
imagePullPolicy: IfNotPresent
image: myapp:1.2.3  # Tag cố định — không bao giờ bị ghi đè
```

### Deployment Strategy (Chiến Lược Triển Khai)

```yaml
spec:
  strategy:
    type: RollingUpdate          # Rolling Update — cập nhật cuốn chiếu, zero downtime
    rollingUpdate:
      maxSurge: 1                # Tạo tối đa 1 pod mới thêm vào trong lúc update
      maxUnavailable: 0          # Không cho phép pod nào unavailable trong lúc update
```

### Secrets Trong Kubernetes

```groovy
// ❌ Không truyền secret qua environment variables thông thường trong manifest
// ✅ Dùng Kubernetes Secrets và tham chiếu từ Deployment

// Tạo secret từ Jenkins pipeline
withCredentials([string(credentialsId: 'db-password', variable: 'DB_PASS')]) {
    sh """
        kubectl create secret generic myapp-secrets \
            --from-literal=db-password=${DB_PASS} \
            -n production \
            --dry-run=client -o yaml | kubectl apply -f -
    """
}
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Làm thế nào để deploy lên nhiều Kubernetes cluster từ một Jenkins pipeline?**

> Tạo nhiều Credentials trong Jenkins (một kubeconfig cho mỗi cluster), sau đó dùng `withKubeConfig` với credentialsId tương ứng cho từng stage. Có thể dùng Helm với values files khác nhau để cấu hình từng environment. Bọc trong vòng lặp hoặc parallel stages để deploy đồng thời.

**Q: Tại sao nên dùng `--atomic` flag khi Helm deploy?**

> Flag `--atomic` đảm bảo: nếu deploy thất bại (bất kỳ resource nào không vào trạng thái Ready trong thời gian timeout), Helm sẽ tự động rollback về revision trước đó và báo lỗi. Không có flag này, một deploy lỗi sẽ để lại cluster trong trạng thái không nhất quán — một số pods chạy version mới, một số chạy version cũ.

**Q: Sự khác nhau giữa Readiness Probe và Liveness Probe?**

> **Liveness Probe** — kiểm tra container còn sống không. Nếu fail, Kubernetes restart container. Dùng để phát hiện deadlock, memory leak. **Readiness Probe** — kiểm tra container đã sẵn sàng nhận traffic chưa. Nếu fail, pod bị remove khỏi Service endpoints nhưng không bị restart. Khi deploy, traffic chỉ được route tới pods đã pass Readiness Probe — đảm bảo zero-downtime deployment.

**Q: Jenkins CD vs GitOps (ArgoCD/FluxCD) — nên chọn cái nào?**

> Jenkins CD (Jenkins kiểm soát deploy trực tiếp) đơn giản hơn và phù hợp khi team đang dùng Jenkins cho CI. GitOps (ArgoCD/FluxCD theo dõi Git repo) có ưu điểm: drift detection (phát hiện cấu hình bị thay đổi ngoài quy trình), audit trail tốt hơn, và declarative hơn. Hybrid approach là phổ biến nhất: Jenkins làm CI (build, test, push image), ArgoCD làm CD (đồng bộ config từ Git vào cluster).

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
**Chủ Đề Tiếp Theo:** [4-notifications.md](4-notifications.md)
