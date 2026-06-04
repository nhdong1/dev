# Deploy Lên Kubernetes Từ CircleCI

> **Kubernetes — K8s** — Hệ Điều Phối Container: sau khi có image trên registry, pipeline áp dụng manifest (hoặc Helm chart) lên cluster **EKS**, **GKE**, hoặc **AKS**.

## 📚 Mục Lục

1. [Luồng Deploy Chuẩn](#luồng-deploy-chuẩn)
2. [kubernetes Orb](#kubernetes-orb)
3. [kubectl Thủ Công](#kubectl-thủ-công)
4. [Helm](#helm)
5. [Xác Thực Cluster](#xác-thực-cluster)
6. [Chiến Lược Deploy](#chiến-lược-deploy)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Luồng Deploy Chuẩn

```
build image → push registry → cập nhật manifest/image tag → kubectl apply / helm upgrade
```

CircleCI thường chạy job deploy **sau** test + build, trên nhánh `main`, có thể qua **approval job**.

---

## kubernetes Orb

**Orb:** `circleci/kubernetes@1.x` (kết hợp `aws-cli`, `gcp-cli` tùy cluster)

```yaml
version: 2.1

orbs:
  kubernetes: circleci/kubernetes@1.3.1
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  deploy-eks:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - aws-cli/setup:
          role-arn: $AWS_DEPLOY_ROLE_ARN
          region: ap-southeast-1
      - kubernetes/install-kubectl
      - kubernetes/install-kubeconfig:
          kubeconfig: $KUBECONFIG_DATA    # Base64 kubeconfig hoặc dùng aws eks update-kubeconfig
      - run:
          name: Cập nhật image và apply
          command: |
            kubectl set image deployment/myapp \
              myapp=123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/myapp:${CIRCLE_SHA1:0:7} \
              -n production
            kubectl rollout status deployment/myapp -n production --timeout=300s
```

### EKS — Elastic Kubernetes Service

```yaml
      - run:
          name: Cấu hình kubeconfig EKS
          command: |
            aws eks update-kubeconfig \
              --name my-cluster \
              --region ap-southeast-1
```

Role IAM cần quyền `eks:DescribeCluster` và quyền Kubernetes RBAC tương ứng (thường qua `aws-auth` ConfigMap hoặc EKS access entries).

---

## kubectl Thủ Công

Khi không dùng orb, cài `kubectl` trong step:

```yaml
      - run:
          name: Install kubectl
          command: |
            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
            chmod +x kubectl && sudo mv kubectl /usr/local/bin/
      - run:
          name: Apply manifests
          command: |
            envsubst < k8s/deployment.yaml | kubectl apply -f -
```

`envsubst` thay `${IMAGE_TAG}` từ biến môi trường CircleCI.

**Ví dụ deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: ${ECR_REGISTRY}/myapp:${IMAGE_TAG}
```

---

## Helm

```yaml
orbs:
  helm: circleci/helm@2.0.1

jobs:
  deploy:
    steps:
      - checkout
      - helm/install-helm-client
      - run:
          command: |
            helm upgrade --install myapp ./chart \
              --namespace production \
              --set image.tag=${CIRCLE_SHA1:0:7} \
              --wait --timeout 10m
```

Helm phù hợp khi nhiều môi trường (values-staging.yaml, values-prod.yaml).

---

## Xác Thực Cluster

| Cloud | Cách Phổ Biến |
|-------|----------------|
| **EKS** | `aws eks update-kubeconfig` + IAM/OIDC role |
| **GKE** | `gcloud container clusters get-credentials` + Workload Identity hoặc service account key (tránh key nếu có WI) |
| **AKS** | `az aks get-credentials` |

Lưu **kubeconfig** nhạy cảm trong Context; hoặc chỉ lưu role ARN và generate kubeconfig mỗi lần chạy pipeline.

---

## Chiến Lược Deploy

| Chiến Lược | Mô Tả Trong Pipeline |
|------------|----------------------|
| **Rolling Update** — Mặc định K8s | `kubectl set image` + `rollout status` |
| **Blue/Green** | Hai deployment/service; switch traffic bằng Service selector hoặc Ingress |
| **Canary** | Flagger, Argo Rollouts, hoặc Ingress weight — thường job riêng sau deploy cơ bản |

CircleCI chỉ **kích hoạt** bước; logic canary thường nằm trên cluster (GitOps: Argo CD, Flux).

---

## Best Practices

1. Deploy **staging** tự động; **production** cần approval + context riêng
2. Dùng **namespace** tách môi trường (`staging`, `production`)
3. Không commit kubeconfig plaintext; generate từ cloud CLI + OIDC
4. `kubectl rollout status` với timeout để job fail khi deploy kẹt
5. Lưu manifest trong repo (GitOps) hoặc Helm chart versioned

---

## Câu Hỏi Phỏng Vấn

**CircleCI deploy K8s khác GitOps (Argo CD) thế nào?**  
CircleCI push thay đổi trực tiếp qua kubectl/helm; GitOps cluster tự sync từ Git — có thể kết hợp: CI chỉ cập nhật tag trong repo, Argo sync.

**Làm sao rollback?**  
`kubectl rollout undo deployment/myapp` hoặc deploy lại image tag SHA cũ đã biết.

**HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang** có liên quan pipeline không?**  
HPA chạy trên cluster theo CPU/memory/custom metrics; pipeline chỉ cần deploy image và resource requests/limits đúng.
