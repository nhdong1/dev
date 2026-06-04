# 🔐 Bảo Mật Pipeline — Security Module

> Hướng dẫn toàn diện về bảo mật trong CircleCI — từ quản lý secrets, xác thực không mật khẩu (OIDC — OpenID Connect), kiểm soát truy cập mạng, đến kiểm toán hoạt động pipeline.

## 📚 Mục Lục Module

| File | Chủ Đề | Độ Ưu Tiên |
|------|---------|------------|
| [1-contexts.md](./1-contexts.md) | Contexts — Ngữ Cảnh: Quản lý secrets tập trung | ⭐⭐⭐ Bắt buộc |
| [2-oidc-integration.md](./2-oidc-integration.md) | OIDC — OpenID Connect với AWS/GCP/Azure | ⭐⭐⭐ Bắt buộc |
| [3-ip-ranges.md](./3-ip-ranges.md) | IP Ranges — Dải IP cho firewall whitelist | ⭐⭐ Quan trọng |
| [4-audit-log.md](./4-audit-log.md) | Audit Log — Nhật Ký Kiểm Toán và compliance | ⭐⭐ Quan trọng |

---

## 🎯 Tại Sao Bảo Mật Pipeline Quan Trọng?

CI/CD pipeline — Đường Ống Tích Hợp Liên Tục/Triển Khai Liên Tục là nơi **tập trung quyền lực cao nhất** trong hệ thống kỹ thuật:

```
Pipeline có khả năng:
  ✅ Truy cập mã nguồn toàn bộ ứng dụng
  ✅ Deploy lên production server — Máy Chủ Sản Xuất
  ✅ Đọc/ghi vào database — Cơ Sở Dữ Liệu
  ✅ Push lên Docker registry — Kho Chứa Image
  ✅ Thay đổi cấu hình cloud infrastructure — Hạ Tầng Đám Mây

→ Nếu bị xâm phạm: toàn bộ hệ thống bị ảnh hưởng
```

### Các Mối Đe Dọa Phổ Biến — Common Threats

| Mối Đe Dọa | Mô Tả | Biện Pháp Đối Phó |
|------------|-------|-------------------|
| **Secret Leakage** — Rò Rỉ Bí Mật | API key lộ trong logs | Contexts, masked variables |
| **Supply Chain Attack** — Tấn Công Chuỗi Cung Ứng | Orb độc hại, image giả | Pin version, dùng certified orbs |
| **Privilege Escalation** — Leo Thang Đặc Quyền | PR từ fork truy cập secrets | Context restrictions |
| **Credential Theft** — Đánh Cắp Thông Tin Xác Thực | Long-lived tokens bị lộ | OIDC — xác thực không cần static key |
| **Insider Threat** — Mối Đe Dọa Từ Nội Bộ | Thành viên team rời đi | Audit Log, Context permissions |

---

## 🏗️ Kiến Trúc Bảo Mật Nhiều Lớp — Defense in Depth

```
┌─────────────────────────────────────────────────────────────────┐
│                    VÒNG NGOÀI — NETWORK LAYER                   │
│  IP Ranges Allowlist → Chỉ CircleCI IPs mới vào được firewall  │
├─────────────────────────────────────────────────────────────────┤
│                  LỚP GIỮA — IDENTITY LAYER                      │
│  OIDC — Xác thực ngắn hạn, không cần static credentials        │
├─────────────────────────────────────────────────────────────────┤
│                 LỚP TRONG — SECRETS LAYER                       │
│  Contexts — Phân quyền secrets theo team/môi trường             │
├─────────────────────────────────────────────────────────────────┤
│                LỚP CỐT LÕI — AUDIT LAYER                       │
│  Audit Log — Ghi lại mọi thay đổi để phát hiện bất thường      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔑 Các Khái Niệm Cốt Lõi

### 1. Hierarchy bảo mật secrets

```
Mức độ ưu tiên (thấp → cao):

Project Environment Variables    ← Ít an toàn nhất
  ↓ Override bởi
Context Environment Variables   ← An toàn hơn, quản lý tập trung
  ↓ Không thể override
Built-in CircleCI Variables     ← Hệ thống tự inject (CIRCLE_SHA1, v.v.)
```

### 2. OIDC vs Static Credentials

```
Static Credentials (cũ):             OIDC (hiện đại):
  AWS_ACCESS_KEY_ID=AKIA...            Không cần key nào cả
  AWS_SECRET_ACCESS_KEY=xxx            CircleCI tự lấy token tạm thời
  Hết hạn: không bao giờ               Hết hạn: sau 1 giờ
  Nếu lộ: rủi ro mãi mãi              Nếu lộ: tự động vô hiệu
```

### 3. Context Restrictions — Giới Hạn Ngữ Cảnh

```yaml
# Chỉ cho phép context khi chạy từ nhánh main
workflows:
  deploy:
    jobs:
      - deploy-job:
          context: production-secrets   # Secrets chỉ tiêm vào job này
          filters:
            branches:
              only: main               # Chỉ nhánh main mới được dùng
```

---

## ⚡ Quick Start — Thiết Lập Bảo Mật Cơ Bản

### Bước 1: Tạo Context cho từng môi trường

```
CircleCI Dashboard → Organization Settings → Contexts
  → New Context: "staging-secrets"
  → New Context: "production-secrets"
  → Add Environment Variable vào từng context
```

### Bước 2: Gán Context vào job trong workflow

```yaml
workflows:
  ci-cd:
    jobs:
      - test           # Không cần secrets
      - deploy-staging:
          context: staging-secrets
          requires: [test]
      - deploy-prod:
          context: production-secrets
          requires: [deploy-staging]
          filters:
            branches:
              only: main
```

### Bước 3: Thêm Security Groups cho context nhạy cảm

```
Context "production-secrets" → Security → Add Security Group
  → Restrict to: GitHub Team "senior-engineers"
  → Chỉ thành viên nhóm mới có thể dùng context này
```

### Bước 4: Bật OIDC để loại bỏ static AWS keys

```yaml
jobs:
  deploy:
    steps:
      - run:
          name: Lấy AWS credentials qua OIDC
          command: |
            # CircleCI tự inject CIRCLE_OIDC_TOKEN
            aws sts assume-role-with-web-identity \
              --role-arn $AWS_ROLE_ARN \
              --web-identity-token $CIRCLE_OIDC_TOKEN \
              --role-session-name circleci-deploy
```

---

## 📋 Security Checklist — Danh Sách Kiểm Tra Bảo Mật

### Cho Mọi Project

- [ ] Không bao giờ hardcode secrets trong `.circleci/config.yml`
- [ ] Dùng Contexts thay vì Project-level variables cho secrets nhạy cảm
- [ ] Ghim (pin) phiên bản orbs: `circleci/aws-cli@4.1.0` không phải `@latest`
- [ ] Ghim Docker image tag: `cimg/node:20.11.0` không phải `node:latest`
- [ ] Review config.yml trước khi merge PR từ contributor bên ngoài

### Cho Môi Trường Production

- [ ] Bật Context Security Groups — hạn chế ai được dùng production context
- [ ] Dùng OIDC thay vì long-lived AWS/GCP credentials
- [ ] Thiết lập IP Ranges nếu firewall cần whitelist
- [ ] Kích hoạt Audit Log export vào SIEM — Hệ Thống Quản Lý Thông Tin Bảo Mật

### Cho Tổ Chức Lớn — Enterprise

- [ ] Single Sign-On (SSO — Đăng Nhập Một Lần) với SAML/OIDC
- [ ] Restricted Context chỉ cho team DevOps/SRE
- [ ] Regular secret rotation — Xoay Vòng Định Kỳ (90 ngày)
- [ ] Audit Log review hàng tuần

---

## 🔗 Điều Hướng

| ← Trước | Module Này | Tiếp → |
|---------|-----------|--------|
| [05-optimization/](../05-optimization/README.md) | **06-security/** | [07-integration/](../07-integration/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Độ Khó:** ⭐⭐ Trung Bình
**Thời Gian Học:** 4–6 giờ
