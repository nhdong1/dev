# Build Và Push Docker Image Trong CircleCI

> Docker — Container hóa ứng dụng: pipeline thường **build image**, **gắn tag** theo commit, **push** lên registry (ECR — Elastic Container Registry, GCR — Google Container Registry, Docker Hub) để bước deploy (ECS, Kubernetes, Cloud Run) dùng image đó.

## 📚 Mục Lục

1. [Hai Cách Build Trên CircleCI](#hai-cách-build-trên-circleci)
2. [setup_remote_docker](#setup_remote_docker)
3. [Docker Orb](#docker-orb)
4. [Tagging và Immutability](#tagging-và-immutability)
5. [Đăng Nhập Registry](#đăng-nhập-registry)
6. [Workflow Mẫu](#workflow-mẫu)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Hai Cách Build Trên CircleCI

| Cách | Khi Nào Dùng |
|------|----------------|
| **Docker executor** + `setup_remote_docker` | Job chạy trong container nhưng build image trên Docker host từ xa |
| **Machine executor** — Máy Ảo | Build trực tiếp trên VM, không cần remote Docker (phù hợp image lớn, BuildKit) |
| **docker orb** | Rút gọn build/push với tham số declarative |

---

## setup_remote_docker

Trong job dùng **Docker executor**, container job **không** chạy Docker daemon. Step `setup_remote_docker` kết nối tới Docker engine trên infrastructure CircleCI.

```yaml
jobs:
  build-image:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true   # DLC — Docker Layer Caching — tái sử dụng layer
      - run:
          name: Build image
          command: |
            docker build -t myapp:${CIRCLE_SHA1:0:7} .
            docker tag myapp:${CIRCLE_SHA1:0:7} myapp:latest
```

**Lưu ý:** `docker_layer_caching` tốn thêm credit nhưng có thể giảm đáng kể thời gian build khi Dockerfile ít thay đổi layer đầu.

---

## Docker Orb

**Orb:** `circleci/docker@2.x`

```yaml
version: 2.1

orbs:
  docker: circleci/docker@2.3.0

jobs:
  build:
    executor: docker/docker
    steps:
      - checkout
      - docker/build:
          image: myorg/myapp
          tag: ${CIRCLE_SHA1:0:7},latest
      - docker/push:
          image: myorg/myapp
          tag: ${CIRCLE_SHA1:0:7},latest
```

Với **AWS ECR**, dùng job tích hợp:

```yaml
      - docker/build_and_push:
          image: myapp
          registry: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com
          tag: ${CIRCLE_SHA1:0:7}
          aws-region: ap-southeast-1
          # Cần AWS credentials hoặc OIDC — xem 3-aws-integration.md
```

---

## Tagging và Immutability

| Chiến Lược Tag | Mô Tả |
|----------------|--------|
| `CIRCLE_SHA1` | Trace chính xác commit → image (khuyến nghị cho deploy) |
| `CIRCLE_BUILD_NUM` | Số build tăng dần, dễ đọc trong dashboard |
| `latest` | Tiện dev; **không** nên là tag duy nhất trên production |
| Semver `v1.2.3` | Release có tag Git `v*` |

```yaml
      - run:
          command: |
            SHORT_SHA=${CIRCLE_SHA1:0:7}
            docker tag myapp:$SHORT_SHA \
              $ECR_REGISTRY/myapp:$SHORT_SHA
            docker tag myapp:$SHORT_SHA \
              $ECR_REGISTRY/myapp:${CIRCLE_TAG:-$SHORT_SHA}
```

---

## Đăng Nhập Registry

### Docker Hub

```yaml
      - run:
          name: Login Docker Hub
          command: |
            echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
```

Biến lưu trong **Context** — Ngữ Cảnh, không commit vào repo.

### ECR (qua AWS CLI)

```yaml
      - run:
          command: |
            aws ecr get-login-password --region ap-southeast-1 | \
              docker login --username AWS --password-stdin \
              123456789012.dkr.ecr.ap-southeast-1.amazonaws.com
```

Ưu tiên **OIDC** + `aws-cli/setup` thay access key (xem `06-security/2-oidc-integration.md`).

---

## Workflow Mẫu

```yaml
workflows:
  docker-pipeline:
    jobs:
      - build-image:
          filters:
            branches:
              only: /.*/
      - scan-image:          # Tùy chọn: Trivy, Snyk orb
          requires:
            - build-image
      - push-production:
          requires:
            - scan-image
          filters:
            branches:
              only: main
          context: production-aws   # OIDC role ARN trong context
```

Dùng **workspace** hoặc **artifact** nếu cần truyền `image digest` sang job deploy Kubernetes.

---

## Best Practices

1. **Multi-stage Dockerfile** — giảm kích thước image production
2. **Pin base image** bằng digest (`node@sha256:...`) để tránh supply chain drift
3. **Không** embed secrets vào image; dùng secrets manager lúc runtime
4. Build trên nhánh feature; push registry production chỉ từ `main` + approval
5. Kết hợp **DLC** với cache key hợp lý (`checksum Dockerfile`)

---

## Câu Hỏi Phỏng Vấn

**Tại sao cần `setup_remote_docker` trong Docker executor?**  
Container job là môi trường cô lập không có daemon; remote Docker cho phép `docker build` chạy trên engine bên ngoài.

**Khác biệt giữa `store_artifacts` và push registry?**  
Artifacts phục vụ tải xuống/debug tạm thời; registry là nguồn sự thật cho orchestrator (K8s, ECS) kéo image.

**Làm sao tránh ghi đè image production?**  
Tag immutable theo SHA; deploy job chỉ reference digest/SHA đã qua scan và approval.
