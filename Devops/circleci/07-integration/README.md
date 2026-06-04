# ☁️ Tích Hợp Cloud & Công Cụ — Integration Module

> Hướng dẫn kết nối CircleCI với Docker, Kubernetes, AWS, GCP, Terraform và hệ thống thông báo — biến pipeline từ “chỉ build/test” thành luồng CD (Continuous Deployment — Triển Khai Liên Tục) end-to-end.

## 📚 Mục Lục Module

| File | Chủ Đề | Độ Ưu Tiên |
|------|---------|------------|
| [1-docker-build-push.md](./1-docker-build-push.md) | Build và push Docker image | ⭐⭐⭐ Bắt buộc |
| [2-kubernetes-deploy.md](./2-kubernetes-deploy.md) | Deploy lên EKS, GKE, AKS | ⭐⭐⭐ Theo dự án |
| [3-aws-integration.md](./3-aws-integration.md) | S3, ECR, ECS, Lambda, CodeDeploy | ⭐⭐⭐ Theo dự án |
| [4-gcp-integration.md](./4-gcp-integration.md) | GCR, Cloud Run, GKE | ⭐⭐ Theo dự án |
| [5-terraform-pipeline.md](./5-terraform-pipeline.md) | Terraform plan/apply trong pipeline | ⭐⭐ IaC — Infrastructure as Code |
| [6-notifications.md](./6-notifications.md) | Slack, email, webhook | ⭐⭐ Vận hành |

---

## 🎯 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- Thiết kế workflow **build → artifact → deploy** với Docker và registry (ECR, GCR, Docker Hub)
- Deploy ứng dụng lên **Kubernetes** (EKS — Elastic Kubernetes Service, GKE — Google Kubernetes Engine, AKS — Azure Kubernetes Service) qua orb hoặc `kubectl`
- Dùng **aws-cli** / **gcp-cli** orb với **OIDC** thay vì static keys
- Chạy **Terraform** plan trên mọi PR và apply có kiểm soát (approval) lên production
- Gửi **notifications** khi pipeline success/failure để team phản ứng nhanh

---

## 🏗️ Mô Hình Tích Hợp Điển Hình

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Source    │────►│   CircleCI   │────►│  Artifact Store │
│  Git Push   │     │  test/build  │     │  ECR/GCR/S3     │
└─────────────┘     └──────┬───────┘     └────────┬────────┘
                           │                        │
                           ▼                        ▼
                    ┌──────────────┐     ┌─────────────────┐
                    │  Terraform   │     │  K8s / ECS /    │
                    │  (optional)  │     │  Lambda / Run   │
                    └──────────────┘     └─────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Slack/Email  │
                    └──────────────┘
```

---

## 🔑 Khái Niệm Cốt Lõi

| Khái Niệm | Giải Thích Ngắn |
|-----------|-----------------|
| **Registry** — Kho Chứa Image | Nơi lưu Docker image (ECR, GCR, Docker Hub) |
| **setup_remote_docker** | Bật Docker daemon trên host CircleCI để build image trong job Docker executor |
| **Orb** — Gói Tích Hợp | Gói YAML tái sử dụng (`circleci/docker`, `circleci/aws-cli`, `circleci/kubernetes`) |
| **Context** — Ngữ Cảnh | Nơi lưu secrets/registry credentials theo môi trường (xem `06-security/`) |
| **Approval Job** | Cổng duyệt thủ công trước deploy production (xem `03-workflows/3-approval-jobs.md`) |

---

## ⚡ Quick Start — Pipeline Deploy Tối Giản

```yaml
version: 2.1

orbs:
  docker: circleci/docker@2.3.0
  aws-cli: circleci/aws-cli@4.0.0

workflows:
  build-and-deploy:
    jobs:
      - docker/push:
          image: myorg/myapp
          tag: ${CIRCLE_SHA1:0:7}
          registry: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com
          aws-region: ap-southeast-1
          filters:
            branches:
              only: main
```

Chi tiết từng bước: đọc lần lượt các file trong bảng mục lục phía trên.

---

## 🔗 Liên Kết Module Liên Quan

| Nhu Cầu | Module |
|---------|--------|
| Secrets, OIDC | [06-security/](../06-security/) |
| Orbs docker/aws/k8s | [04-orbs/2-certified-orbs.md](../04-orbs/2-certified-orbs.md) |
| Approval trước prod | [03-workflows/3-approval-jobs.md](../03-workflows/3-approval-jobs.md) |
| Workspace giữ artifact | [05-optimization/2-workspace.md](../05-optimization/2-workspace.md) |

---

## ✅ Checklist Module

- [ ] Build và push image với `setup_remote_docker` hoặc docker orb
- [ ] Deploy thử lên cluster dev/staging
- [ ] Cấu hình OIDC cho AWS hoặc Workload Identity cho GCP
- [ ] Terraform plan trên PR, apply sau approval
- [ ] Slack thông báo khi deploy production thất bại

**Thời gian học ước tính:** 6–10 giờ (tùy cloud bạn dùng)
