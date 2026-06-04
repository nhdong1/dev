# 📦 Module 04: Orbs — Gói Tích Hợp Tái Sử Dụng

> **Orbs** là các gói cấu hình CircleCI được đóng gói, có thể tái sử dụng và chia sẻ — tương tự như thư viện trong lập trình. Thay vì viết lại cấu hình từ đầu cho mỗi dự án, orbs cho phép bạn import và dùng ngay trong vài dòng.

---

## 📚 Mục Lục Module

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-using-orbs.md](./1-using-orbs.md) | Import và sử dụng orb từ registry | ⭐ |
| [2-certified-orbs.md](./2-certified-orbs.md) | aws-cli, docker, node, kubernetes orbs | ⭐⭐ |
| [3-inline-orbs.md](./3-inline-orbs.md) | Inline Orb — Orb Nội Tuyến trong config | ⭐⭐ |
| [4-custom-orb-development.md](./4-custom-orb-development.md) | Tạo, test và publish orb tùy chỉnh | ⭐⭐⭐ |

---

## 🎯 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Giải thích orb là gì và lợi ích trong dự án thực
- [ ] Import và sử dụng orb từ CircleCI Orb Registry
- [ ] Tùy chỉnh orb thông qua parameters — tham số
- [ ] Phân biệt Certified Orb, Partner Orb và Community Orb
- [ ] Viết Inline Orb — Orb Nội Tuyến cho dự án nhỏ hoặc prototype
- [ ] Phát triển, kiểm thử và publish Custom Orb — Orb Tùy Chỉnh
- [ ] Áp dụng best practices về orb versioning và bảo mật

---

## 🧩 Orb Là Gì?

### Định Nghĩa

**Orb** là một gói YAML được đóng gói gồm ba thành phần tái sử dụng của CircleCI:

```
Orb = Commands (Lệnh) + Jobs (Công Việc) + Executors (Môi Trường Thực Thi)
```

### Vấn Đề Orbs Giải Quyết

Trước khi có orbs, mỗi dự án phải viết lại cùng một cấu hình:

```yaml
# Trước orbs — lặp lại trong mọi dự án
jobs:
  deploy:
    steps:
      - run:
          name: Cài AWS CLI
          command: |
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip awscliv2.zip
            sudo ./aws/install
      - run:
          name: Cấu hình AWS credentials
          command: |
            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
            aws configure set region $AWS_DEFAULT_REGION
      - run: aws s3 sync dist/ s3://my-bucket
```

```yaml
# Sau khi có orbs — đơn giản và sạch hơn
orbs:
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  deploy:
    steps:
      - aws-cli/setup    # Toàn bộ cài đặt gói gọn trong 1 command
      - run: aws s3 sync dist/ s3://my-bucket
```

### Kiến Trúc Orb

```
┌─────────────────────────────────────────────┐
│              CircleCI Orb Registry           │
│  ┌──────────────┐  ┌──────────────────────┐ │
│  │ Certified    │  │ Partner & Community  │ │
│  │ circleci/    │  │ company/orb-name     │ │
│  │ aws-cli      │  │ datadog/agent        │ │
│  │ node         │  │ snyk/snyk            │ │
│  │ docker       │  │ slack/notify         │ │
│  └──────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────┘
              ↓ import vào config
┌─────────────────────────────────────────────┐
│           .circleci/config.yml               │
│                                             │
│  orbs:                                      │
│    aws-cli: circleci/aws-cli@4.0.0          │
│                                             │
│  jobs:                                      │
│    deploy:                                  │
│      steps:                                 │
│        - aws-cli/setup      ← dùng command  │
│        - aws-cli/deploy-ecs ← dùng job      │
└─────────────────────────────────────────────┘
```

---

## 🏷️ Phân Loại Orbs

### 1. Certified Orbs — Orb Được Chứng Nhận

- **Tác giả:** CircleCI chính thức (`circleci/`)
- **Chất lượng:** Được kiểm tra kỹ, maintained liên tục
- **Ví dụ:** `circleci/aws-cli`, `circleci/node`, `circleci/docker`, `circleci/kubernetes`
- **Tin cậy:** Cao nhất

### 2. Partner Orbs — Orb Của Đối Tác

- **Tác giả:** Các công ty đối tác chính thức của CircleCI
- **Chất lượng:** Được CircleCI review và chứng nhận
- **Ví dụ:** `datadog/agent`, `snyk/snyk`, `terraform/terrraform`, `anchore/anchore-engine`
- **Tin cậy:** Cao

### 3. Community Orbs — Orb Cộng Đồng

- **Tác giả:** Cộng đồng CircleCI (bất kỳ ai)
- **Chất lượng:** Không đảm bảo, cần review source trước khi dùng
- **Lưu ý:** Phải bật `allow_uncertified_orbs` trong Organization Settings
- **Tin cậy:** Vừa phải — luôn review source code

### 4. Inline Orbs — Orb Nội Tuyến

- **Vị trí:** Định nghĩa thẳng trong `config.yml` (không publish lên registry)
- **Mục đích:** Tổ chức code, prototype trước khi tách thành orb riêng
- **Phạm vi:** Chỉ dùng trong 1 project

---

## ⚡ Lợi Ích Của Orbs

| Lợi Ích | Mô Tả |
|---------|-------|
| **Tái sử dụng** | Viết một lần, dùng ở nhiều project |
| **Giảm độ phức tạp** | Config ngắn hơn, dễ đọc hơn |
| **Chuẩn hóa** | Cùng cách deploy AWS trên toàn tổ chức |
| **Bảo trì tập trung** | Sửa orb một nơi → áp dụng cho tất cả project |
| **Community knowledge** | Hưởng lợi từ best practices của cộng đồng |
| **Nhanh hơn** | Không cần viết lại boilerplate code |

---

## ⚠️ Rủi Ro Và Cách Giảm Thiểu

### Rủi Ro Bảo Mật

```yaml
# NGUY HIỂM — orb của bên thứ ba có thể đọc secrets của bạn
orbs:
  unknown: some-unknown-author/some-orb@1.0.0  # Không rõ nguồn gốc
```

**Cách giảm thiểu:**

1. Ưu tiên dùng Certified và Partner Orbs
2. Luôn review source code của Community Orbs trước khi dùng
3. Pin phiên bản cụ thể (ví dụ `@4.1.0`) thay vì dùng `@volatile`
4. Dùng `semver` — Semantic Versioning: `@4` cho tất cả patch trong major 4

### Rủi Ro Phụ Thuộc — Dependency Risk

```yaml
# Rủi ro — version volatile thay đổi không kiểm soát được
orbs:
  node: circleci/node@volatile  # Tránh dùng

# An toàn hơn — pin exact version
orbs:
  node: circleci/node@5.1.0    # Reproducible build — Build Có Thể Tái Tạo
```

---

## 🗺️ Lộ Trình Học Module Này

```
┌──────────────────────────────────────────────────────┐
│  Bước 1: Dùng orb cơ bản (1-using-orbs.md)          │
│  → Hiểu cú pháp import, namespace, versioning        │
├──────────────────────────────────────────────────────┤
│  Bước 2: Certified orbs thực tế (2-certified-orbs.md)│
│  → aws-cli, docker, node, kubernetes                 │
├──────────────────────────────────────────────────────┤
│  Bước 3: Inline orbs (3-inline-orbs.md)              │
│  → Tổ chức config lớn, prototype orb                 │
├──────────────────────────────────────────────────────┤
│  Bước 4: Tạo custom orb (4-custom-orb-development.md)│
│  → Chuẩn hóa CI/CD cho toàn tổ chức                 │
└──────────────────────────────────────────────────────┘
```

---

## 🔗 Điều Hướng Module

- **Trước:** [03-workflows/](../03-workflows/) — Workflow Design
- **Tiếp theo:** [05-optimization/](../05-optimization/) — Tối Ưu Hóa Pipeline
- **Tham khảo:** [CircleCI Orb Registry](https://circleci.com/developer/orbs)

---

**Cập Nhật Lần Cuối:** 2026-05-18 | **Phiên Bản:** 1.0
