# ⚙️ CircleCI — Lộ Trình Học Toàn Diện

> Hướng dẫn đầy đủ về CircleCI — nền tảng CI/CD (Continuous Integration / Continuous Deployment — Tích Hợp Liên Tục / Triển Khai Liên Tục) từ cơ bản đến nâng cao, bao gồm cấu hình pipeline, tối ưu hóa, bảo mật và chuẩn bị phỏng vấn.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
4. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)
5. [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng — Fundamentals (Tuần 1–2)**

- [ ] Hiểu CI/CD và vai trò của CircleCI
- [ ] Cấu trúc file `.circleci/config.yml`
- [ ] Job, Step, Workflow — khái niệm cơ bản
- [ ] Executor (Docker, Machine, macOS) — Môi Trường Thực Thi
- [ ] Chạy pipeline đầu tiên

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3–6)**

- [ ] Workflow nâng cao: Sequential, Parallel, Approval
- [ ] Caching — Bộ Nhớ Đệm và Workspace — Không Gian Làm Việc Chung
- [ ] Orbs — Gói Tái Sử Dụng: sử dụng và tạo orb tùy chỉnh
- [ ] Environment Variables — Biến Môi Trường và Contexts — Ngữ Cảnh Bảo Mật
- [ ] Test Parallelism — Song Song Hóa Kiểm Thử

### **Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7–10)**

- [ ] Dynamic Config — Cấu Hình Động và Path Filtering — Lọc Theo Đường Dẫn
- [ ] Self-Hosted Runner — Máy Chạy Tự Quản Lý
- [ ] Tích hợp cloud: AWS, GCP, Azure
- [ ] Monitoring — Giám Sát pipeline và tối ưu thời gian build
- [ ] Security hardening — Tăng Cường Bảo Mật pipeline

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] Matrix Jobs — Công Việc Ma Trận
- [ ] Custom Orb development — Phát Triển Orb Tùy Chỉnh
- [ ] Chiến lược triển khai: Blue/Green, Canary
- [ ] Cost optimization — Tối Ưu Chi Phí
- [ ] Pipeline as Code — Pipeline Dưới Dạng Mã

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực | Mức Độ Ưu Tiên | Thời Gian | Trạng Thái |
| -------- | -------------- | --------- | ---------- |
| **Cấu hình `.circleci/config.yml`** | ⭐⭐⭐ | 1 tuần | - |
| **Workflow Design — Thiết Kế Luồng Công Việc** | ⭐⭐⭐ | 1 tuần | - |
| **Caching & Workspace — Bộ Đệm & Không Gian Làm Việc** | ⭐⭐⭐ | 1 tuần | - |
| **Orbs — Gói Tích Hợp Tái Sử Dụng** | ⭐⭐⭐ | 1 tuần | - |
| **Security & Secrets — Bảo Mật & Quản Lý Bí Mật** | ⭐⭐⭐ | 1 tuần | - |
| **Test Parallelism — Song Song Hóa Kiểm Thử** | ⭐⭐⭐ | 3 ngày | - |
| **Tích Hợp Docker & Kubernetes** | ⭐⭐⭐ | 1 tuần | - |
| **Monitoring & Troubleshooting — Giám Sát & Xử Lý Sự Cố** | ⭐⭐ | 3 ngày | - |
| **Dynamic Config — Cấu Hình Động** | ⭐⭐ | 3 ngày | - |
| **Self-Hosted Runner — Máy Chạy Tự Quản Lý** | ⭐⭐ | 3 ngày | - |

---

## 🗂️ Tổng Quan Chủ Đề

### 📁 **1. Nền Tảng — Fundamentals** (`01-fundamentals/`)

- CI/CD là gì và tại sao cần CircleCI
- Kiến trúc CircleCI: Pipeline → Workflow → Job → Step
- Executor — Môi Trường Thực Thi: Docker, Machine, macOS, Windows
- Orbs — Gói Tích Hợp, Resource Class — Lớp Tài Nguyên
- So sánh CircleCI với Jenkins, GitHub Actions, GitLab CI

### 📁 **2. Cấu Hình — Configuration** (`02-configuration/`)

- Cú pháp file `.circleci/config.yml` — YAML schema đầy đủ
- **version: 2.1** — phiên bản hiện tại và tính năng nổi bật
- Executors: Docker multi-image, Machine (VM — Virtual Machine), macOS
- Parameters — Tham Số: pipeline params, job params, command params
- Environment Variables — Biến Môi Trường: built-in, project-level, org-level
- Commands — Lệnh Tái Sử Dụng và Jobs được tái sử dụng

### 📁 **3. Workflow — Luồng Công Việc** (`03-workflows/`)

- Sequential Workflow — Luồng Tuần Tự: requires, dependencies
- Parallel Workflow — Luồng Song Song: tăng tốc độ, giảm thời gian chờ
- Approval Job — Công Việc Cần Duyệt: manual gate trước khi deploy
- Scheduling — Lập Lịch: cron-based pipeline triggers
- Branch & Tag Filters — Bộ Lọc Nhánh & Nhãn
- Fan-out / Fan-in Pattern — Mô Hình Phân Tỏa / Tập Hợp

### 📁 **4. Orbs — Gói Tích Hợp** (`04-orbs/`)

- Orb là gì và cách dùng orb từ CircleCI Registry
- Certified Orbs — Orb Được Chứng Nhận: aws-cli, docker, kubernetes
- Partner Orbs — Orb Của Đối Tác: Datadog, Snyk, Slack
- Viết Inline Orb — Orb Nội Tuyến cho dự án nhỏ
- Phát triển và publish Custom Orb — Orb Tùy Chỉnh
- Orb versioning — Quản Lý Phiên Bản Orb và best practices

### 📁 **5. Tối Ưu Hóa — Optimization** (`05-optimization/`)

- **Caching — Bộ Nhớ Đệm**: `save_cache`, `restore_cache`, cache keys
- **Workspace — Không Gian Làm Việc Chung**: `persist_to_workspace`, `attach_workspace`
- **Test Splitting — Phân Chia Kiểm Thử**: timing-based, file-based
- **Parallelism — Song Song Hóa**: `parallelism` key, `circleci tests split`
- **Resource Class — Lớp Tài Nguyên**: chọn CPU/RAM phù hợp
- Pipeline Insights — Thống Kê Pipeline: phân tích bottleneck

### 📁 **6. Bảo Mật — Security** (`06-security/`)

- Environment Variables — Biến Môi Trường: project, context, built-in
- Contexts — Ngữ Cảnh: quản lý secret tập trung theo tổ chức
- OIDC — OpenID Connect: xác thực không cần long-lived credentials
- Restricted Contexts — Ngữ Cảnh Giới Hạn và security groups
- IP Ranges — Dải IP: whitelist cho firewall
- Audit Log — Nhật Ký Kiểm Toán và compliance
- Secret Rotation — Xoay Vòng Bí Mật best practices

### 📁 **7. Tích Hợp — Integration** (`07-integration/`)

- **Docker**: build, tag, push image lên ECR/GCR/Docker Hub
- **Kubernetes**: deploy lên EKS, GKE, AKS qua orbs
- **AWS**: aws-cli orb, S3, Lambda, ECS, CodeDeploy
- **GCP**: gcp-cli, Cloud Run, GKE
- **Terraform**: plan & apply trong pipeline
- **Slack**: thông báo build success/failure
- **Snyk / SonarQube**: SAST — Static Application Security Testing

### 📁 **8. Giám Sát & Xử Lý Sự Cố — Monitoring & Troubleshooting** (`08-monitoring/`)

- CircleCI Insights — Thống Kê: success rate, duration, flaky tests
- SSH Debugging — Gỡ Lỗi Qua SSH: `rerun with SSH`
- Xử lý lỗi thường gặp: OOM, timeout, cache miss
- Flaky Tests — Kiểm Thử Không Ổn Định: phát hiện và xử lý
- Pipeline hiệu năng thấp: chẩn đoán và tối ưu

### 📁 **9. Nâng Cao — Advanced** (`09-advanced/`)

- **Dynamic Config — Cấu Hình Động**: setup workflow, continuation
- **Path Filtering — Lọc Theo Đường Dẫn**: monorepo CI/CD
- **Matrix Jobs — Công Việc Ma Trận**: test nhiều version song song
- **Self-Hosted Runner — Máy Chạy Tự Quản Lý**: on-premise, custom hardware
- **Pipeline Values & Logic**: điều kiện `when`, `unless`
- Monorepo Strategy — Chiến Lược Monorepo

### 📁 **10. Chuẩn Bị Phỏng Vấn — Interview Prep** (`10-interview-prep/`)

- Top 20 câu hỏi phỏng vấn CircleCI / CI/CD
- System Design Scenarios — Bài Toán Thiết Kế Hệ Thống
- Câu chuyện incident theo phương pháp STAR
- So sánh CI/CD tools: CircleCI vs Jenkins vs GitHub Actions
- Bài tập tối ưu pipeline thực tế

---

## 🔧 Cấu Trúc File `.circleci/config.yml` Cơ Bản

```yaml
version: 2.1

# Orbs — Gói Tích Hợp tái sử dụng
orbs:
  node: circleci/node@5.1.0
  aws-cli: circleci/aws-cli@4.0.0

# Executors — Môi Trường Thực Thi tái sử dụng
executors:
  node-executor:
    docker:
      - image: cimg/node:20.0
    resource_class: medium  # 2 vCPU, 4GB RAM

# Commands — Lệnh Tái Sử Dụng
commands:
  restore-npm-cache:
    steps:
      - restore_cache:
          keys:
            - npm-v1-{{ checksum "package-lock.json" }}
            - npm-v1-

# Jobs — Công Việc
jobs:
  test:
    executor: node-executor
    parallelism: 4          # Chia thành 4 luồng song song
    steps:
      - checkout
      - restore-npm-cache
      - node/install-packages
      - save_cache:
          key: npm-v1-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm
      - run:
          name: Chạy kiểm thử song song
          command: |
            circleci tests glob "src/**/*.test.js" | \
            circleci tests split --split-by=timings | \
            xargs npx jest

  build:
    executor: node-executor
    steps:
      - checkout
      - node/install-packages
      - run: npm run build
      - persist_to_workspace:   # Lưu artifact sang job tiếp theo
          root: .
          paths:
            - dist/

  deploy:
    executor: node-executor
    steps:
      - attach_workspace:       # Lấy artifact từ job build
          at: .
      - aws-cli/setup
      - run: aws s3 sync dist/ s3://my-bucket --delete

# Workflows — Luồng Công Việc
workflows:
  ci-cd-pipeline:
    jobs:
      - test:
          filters:
            branches:
              only: /.*/       # Chạy trên mọi nhánh
      - build:
          requires:
            - test             # Chỉ build sau khi test xanh
      - approve-deploy:
          type: approval       # Cần duyệt thủ công
          requires:
            - build
          filters:
            branches:
              only: main
      - deploy:
          requires:
            - approve-deploy   # Chỉ deploy sau khi được duyệt
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề | Thư Mục | Mức Độ Ưu Tiên |
| ------- | ------- | -------------- |
| Bắt đầu nhanh | [01-fundamentals/](./01-fundamentals/) | Bắt đầu ở đây |
| Cấu hình YAML | [02-configuration/](./02-configuration/) | Thiết yếu |
| Workflow nâng cao | [03-workflows/](./03-workflows/) | Thiết yếu |
| Orbs phổ biến | [04-orbs/](./04-orbs/) | Quan trọng |
| Tối ưu pipeline | [05-optimization/](./05-optimization/) | Quan trọng |
| Bảo mật secrets | [06-security/](./06-security/) | Thiết yếu |
| Tích hợp cloud | [07-integration/](./07-integration/) | Theo dự án |
| Xử lý sự cố | [08-monitoring/](./08-monitoring/) | Tham khảo |
| Dynamic Config | [09-advanced/](./09-advanced/) | Nâng cao |
| Phỏng vấn | [10-interview-prep/](./10-interview-prep/) | Trước phỏng vấn |

---

## 📊 Ma Trận Kỹ Năng — Skill Matrix

### Người Mới Bắt Đầu — Beginner (0–6 tháng)

- [ ] Hiểu CI/CD là gì và lợi ích của CircleCI
- [ ] Tạo và chạy pipeline đầu tiên
- [ ] Cấu hình job: `checkout`, `run`, `store_artifacts`
- [ ] Sử dụng Docker executor cơ bản
- [ ] Hiểu sự khác biệt giữa Job và Workflow

### Trung Cấp — Intermediate (6 tháng–2 năm)

- [ ] Thiết kế workflow với requires, filters, approval
- [ ] Cấu hình caching và workspace hiệu quả
- [ ] Sử dụng Contexts để quản lý secrets an toàn
- [ ] Tích hợp orbs: AWS, Docker, Slack
- [ ] Song song hóa test với `parallelism` và test splitting
- [ ] Debug pipeline bằng SSH

### Nâng Cao — Advanced (2+ năm)

- [ ] Phát triển Custom Orb — Orb Tùy Chỉnh và publish
- [ ] Triển khai Dynamic Config cho monorepo
- [ ] Cài đặt và quản lý Self-Hosted Runner
- [ ] Tối ưu chi phí với resource class và pipeline logic
- [ ] Xây dựng chiến lược CI/CD cho tổ chức
- [ ] OIDC integration với cloud providers

---

## 🆚 So Sánh CircleCI với Các Công Cụ Khác

### CircleCI vs Jenkins

```
CircleCI:
  + Cloud-native, không cần quản lý server
  + Cấu hình bằng YAML đơn giản
  + Orbs giúp tái sử dụng cấu hình
  + Free tier hào phóng
  - Ít plugin hơn Jenkins
  - Ít kiểm soát infrastructure hơn

Jenkins:
  + Hệ sinh thái plugin rất lớn (1800+ plugins)
  + Hoàn toàn tự quản lý, kiểm soát tối đa
  + On-premise phù hợp với yêu cầu bảo mật cao
  - Cần đội DevOps để vận hành và bảo trì
  - Cấu hình phức tạp, Groovy DSL khó học
```

### CircleCI vs GitHub Actions

```
CircleCI:
  + Tốc độ build nhanh hơn (resource class cao hơn)
  + Orbs ecosystem đa dạng
  + SSH debugging tích hợp sẵn
  - Phí tính theo credit
  - Không tích hợp sâu với GitHub ecosystem

GitHub Actions:
  + Tích hợp tự nhiên với GitHub (PR, Issues)
  + Free cho public repos không giới hạn
  + Marketplace actions phong phú
  - Tốc độ đôi khi chậm hơn CircleCI
  - Ít tính năng advanced CI/CD hơn
```

---

## 🚀 Bắt Đầu Nhanh

### Bước 1: Kết Nối Repository — Kho Mã Nguồn

```bash
# 1. Đăng nhập circleci.com bằng GitHub/Bitbucket/GitLab
# 2. Chọn repository muốn kết nối
# 3. CircleCI tự detect .circleci/config.yml
```

### Bước 2: Tạo File Cấu Hình Tối Giản

```yaml
# .circleci/config.yml
version: 2.1

jobs:
  hello-world:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - run:
          name: Xin chào từ CircleCI
          command: echo "Pipeline đầu tiên chạy thành công!"

workflows:
  say-hello:
    jobs:
      - hello-world
```

### Bước 3: Thiết Lập Môi Trường Thực Hành

```bash
# Cài CircleCI CLI — Giao Diện Dòng Lệnh
curl -fLSs https://raw.githubusercontent.com/CircleCI-Public/circleci-cli/main/install.sh | bash

# Xác thực cấu hình trước khi push
circleci config validate

# Chạy job cục bộ (yêu cầu Docker)
circleci local execute --job hello-world
```

### Bước 4: Học Theo Thứ Tự

```
1. Đọc 01-fundamentals/ (2–3 giờ)
2. Thực hành 02-configuration/ (2–4 giờ)
3. Xây dựng workflow trong 03-workflows/ (2–4 giờ)
4. Áp dụng 05-optimization/ cho dự án thực (3–5 giờ)
5. Bảo mật pipeline với 06-security/ (2–3 giờ)
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Theo Chủ Đề

#### Nền Tảng CI/CD

- [ ] CI/CD là gì? Giải thích sự khác biệt giữa CI, CD (Delivery) và CD (Deployment)
- [ ] CircleCI hoạt động như thế nào? Giải thích kiến trúc tổng quan
- [ ] Khi nào nên dùng CircleCI thay vì Jenkins hay GitHub Actions?

#### Cấu Hình Pipeline

- [ ] Giải thích sự khác biệt giữa Job, Step, Workflow, Pipeline
- [ ] Executor là gì? Khi nào dùng Docker vs Machine executor?
- [ ] Làm thế nào để tái sử dụng cấu hình? (Commands, Executors, Orbs)
- [ ] Parameters trong CircleCI hoạt động như thế nào?

#### Tối Ưu Hóa

- [ ] Caching hoạt động như thế nào? Cache key được tạo ra sao?
- [ ] Workspace khác Cache ở điểm nào?
- [ ] Giải thích Test Splitting và Parallelism
- [ ] Bạn đã tối ưu pipeline để giảm build time như thế nào?

#### Bảo Mật

- [ ] Environment Variables vs Contexts — cái nào an toàn hơn và khi nào dùng?
- [ ] OIDC là gì và tại sao tốt hơn long-lived credentials?
- [ ] Làm thế nào để rotate secrets mà không downtime pipeline?

#### Nâng Cao

- [ ] Dynamic Config là gì? Khi nào cần dùng?
- [ ] Giải thích chiến lược CI/CD cho monorepo
- [ ] Self-hosted Runner phù hợp với trường hợp nào?

Xem `10-interview-prep/` để có đầy đủ câu hỏi và câu trả lời mẫu.

---

## ✅ Checklist Tự Đánh Giá

Trước khi phỏng vấn hoặc nhận dự án mới, kiểm tra:

- [ ] Có thể giải thích sự khác biệt giữa Job, Workflow, Pipeline từ trí nhớ
- [ ] Có thể viết cấu hình `.circleci/config.yml` từ đầu không cần tài liệu
- [ ] Có thể thiết kế workflow với approval gate cho môi trường production
- [ ] Hiểu rõ khi nào dùng `save_cache` vs `persist_to_workspace`
- [ ] Có thể cấu hình Contexts để bảo mật secrets cho từng môi trường
- [ ] Biết cách debug pipeline bị lỗi bằng SSH hoặc logs
- [ ] Có thể tích hợp CircleCI với AWS/GCP để deploy ứng dụng
- [ ] Hiểu Dynamic Config và khi nào áp dụng cho monorepo
- [ ] Có thể giải thích OIDC và tại sao tốt hơn static credentials
- [ ] Đã có ít nhất 1 câu chuyện thực tế về tối ưu hoặc xử lý sự cố pipeline

---

## 📖 Tài Liệu Tham Khảo

### Tài Liệu Chính Thức

- [CircleCI Documentation](https://circleci.com/docs/)
- [CircleCI Orb Registry](https://circleci.com/developer/orbs)
- [CircleCI Discuss](https://discuss.circleci.com/)

### Đọc Thêm

- **"Continuous Delivery"** — Jez Humble & David Farley — Kinh điển về CD
- **"The DevOps Handbook"** — Gene Kim et al. — Tư duy DevOps toàn diện
- CircleCI Blog: [circleci.com/blog](https://circleci.com/blog/)

### Công Cụ Hỗ Trợ

- **CircleCI CLI** — Giao Diện Dòng Lệnh: validate config, local execute
- **VS Code YAML Extension** — Gợi ý cú pháp YAML
- **Pre-commit hooks** — Chạy `circleci config validate` trước khi commit

---

## 📋 Cách Sử Dụng Tài Liệu Này

### Tự Học — Self-Study

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học) theo từng giai đoạn
2. Đọc lý thuyết rồi ngay lập tức thực hành
3. Xây dựng pipeline thực cho dự án cá nhân
4. Ghi lại những vấn đề gặp phải và cách giải quyết

### Chuẩn Bị Phỏng Vấn

1. Tập trung vào `10-interview-prep/` để nắm câu hỏi phổ biến
2. Hiểu sâu cấu hình YAML trong `02-configuration/`
3. Ôn lại optimization trong `05-optimization/`
4. Chuẩn bị 2–3 câu chuyện thực tế (phương pháp STAR)

### Trong Công Việc — On the Job

1. Dùng `02-configuration/` làm tài liệu tham khảo nhanh
2. Tra cứu `08-monitoring/` khi gặp sự cố pipeline
3. Xem `09-advanced/` cho các bài toán phức tạp như monorepo
4. Kiểm tra `06-security/` trước khi release lên production

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình theo mức độ (Beginner / Intermediate / Advanced)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Thực hành với dự án thực hoặc repo demo
├─ 5️⃣  Hoàn thành từng module và tích các checklist
├─ 6️⃣  Xây dựng pipeline production-ready cho portfolio
└─ 7️⃣  Chuẩn bị phỏng vấn với 10-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
