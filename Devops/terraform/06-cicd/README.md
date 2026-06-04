# 06 — CI/CD Integration — Tích Hợp CI/CD với Terraform

> CI/CD — Continuous Integration / Continuous Delivery — Tích Hợp Liên Tục / Triển Khai Liên Tục — là nền tảng để đưa thay đổi hạ tầng vào production một cách an toàn, có kiểm soát, và tự động hoá.

---

## 🎯 Mục Tiêu Phần Này

Sau khi học xong `06-cicd/`, bạn có thể:

- Thiết lập pipeline Terraform trên **GitHub Actions** và **GitLab CI**
- Hiểu và cấu hình **Atlantis** — tự động hoá Terraform qua Pull Request
- Sử dụng **Terraform Cloud** — nền tảng quản lý Terraform hosted
- Thiết kế **Rollback Strategy** — Chiến lược khôi phục — khi có sự cố
- Áp dụng best practice bảo mật cho CI/CD pipeline

---

## 📁 Nội Dung Thư Mục

| File | Chủ Đề | Thời Gian Đọc |
|------|--------|---------------|
| `1-github-actions.md` | GitHub Actions workflow cho Terraform | 25 phút |
| `2-gitlab-ci.md` | GitLab CI/CD pipeline đầy đủ | 20 phút |
| `3-atlantis.md` | Atlantis — PR-based automation — Tự động qua PR | 20 phút |
| `4-terraform-cloud.md` | Terraform Cloud / HCP Terraform | 20 phút |
| `5-rollback-strategy.md` | Chiến lược rollback khi có sự cố | 15 phút |

---

## 🧭 Tại Sao CI/CD Quan Trọng Với Terraform?

### Vấn Đề Khi Apply Thủ Công

```
Developer A                    Developer B
     │                              │
     ├─ terraform plan (local) ─┐   ├─ terraform plan (local) ─┐
     │                          │   │                          │
     ├─ terraform apply ─────── │   ├─ terraform apply ─────── │
     │                          │   │                          │
     ▼                          │   ▼                          │
  Thành công                    │  Lỗi: State bị lock hoặc     │
                                │  Xung đột tài nguyên         │
                                └──────────────────────────────┘

Vấn đề:
- Không có review trước khi apply
- Không biết ai apply gì vào lúc nào
- Không có audit trail — nhật ký kiểm tra
- Dễ apply nhầm môi trường
- Credentials phải có trên máy developer
```

### CI/CD Giải Quyết Như Thế Nào

```
Developer
    │
    ├─ git push → Pull Request
    │                   │
    │           CI Pipeline chạy:
    │           ├─ terraform validate
    │           ├─ terraform fmt --check
    │           ├─ tflint
    │           ├─ tfsec / Checkov
    │           └─ terraform plan (kết quả hiện trong PR)
    │                   │
    │           Code Review + Plan Review
    │                   │
    │           Merge PR
    │                   │
    │           CD Pipeline chạy:
    │           └─ terraform apply (tự động hoặc thủ công approve)
    │
    ▼
  Hạ tầng được cập nhật an toàn
```

---

## 🏗️ Các Mô Hình Triển Khai CI/CD Terraform

### Mô Hình 1: Plan-in-PR, Apply-on-Merge

```
┌─────────────────────────────────────────────────────┐
│  Pull Request (feature → main)                      │
│  ┌─────────────────────────────────────────────┐    │
│  │  CI: terraform plan                         │    │
│  │  Kết quả plan hiển thị dưới dạng comment    │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
                        │ Merge
                        ▼
┌─────────────────────────────────────────────────────┐
│  CD: terraform apply                                │
│  Chạy trên branch main sau khi merge                │
└─────────────────────────────────────────────────────┘
```

**Ưu điểm:** Đơn giản, ít công cụ bổ sung
**Nhược điểm:** Plan có thể stale — cũ — nếu có thay đổi khác trước khi merge

### Mô Hình 2: Atlantis (PR-based Automation)

```
Developer comment: "atlantis plan"
    │
    ▼
Atlantis chạy plan → comment kết quả vào PR
    │
Developer comment: "atlantis apply"
    │
    ▼
Atlantis chạy apply → comment kết quả
```

**Ưu điểm:** Apply xảy ra ngay trên PR, không cần merge mới apply
**Nhược điểm:** Cần self-host Atlantis server

### Mô Hình 3: Terraform Cloud / HCP Terraform

```
git push → Webhook → Terraform Cloud
                           │
                    Run — Speculative Plan (trên PR)
                           │
                    Run — Apply (sau khi approve)
```

**Ưu điểm:** Fully managed, không cần tự quản lý infrastructure
**Nhược điểm:** Phụ thuộc vào dịch vụ bên thứ ba, chi phí theo usage

---

## 🔐 Bảo Mật Trong CI/CD Terraform

### Nguyên Tắc Quan Trọng

```
❌ KHÔNG làm:
- Hardcode AWS credentials trong workflow file
- Commit .tfvars chứa secrets vào git
- Dùng long-lived credentials trong CI

✅ NÊN làm:
- Dùng OIDC — OpenID Connect — để CI lấy credentials tạm thời
- Lưu secrets trong CI secret store (GitHub Secrets, GitLab Variables)
- Áp dụng Least Privilege — Đặc quyền tối thiểu — cho CI role
- Audit log mọi lần apply
```

### OIDC — OpenID Connect — Thay Credentials Tĩnh

```
GitHub Actions / GitLab CI
        │
        │ OIDC Token (JWT — JSON Web Token)
        ▼
   AWS STS — Security Token Service
        │
        │ Temporary Credentials (15 phút - 1 giờ)
        ▼
   IAM Role (chỉ có quyền cần thiết)
        │
        ▼
   Terraform apply
```

---

## 📋 Pipeline Stages Chuẩn

```
┌────────────────────────────────────────────────────────────┐
│                    PULL REQUEST PIPELINE                   │
├────────────┬────────────┬─────────────┬────────────────────┤
│  Validate  │    Lint    │   Security  │       Plan         │
│            │            │    Scan     │                    │
│ • fmt check│ • tflint   │ • tfsec     │ • terraform plan   │
│ • validate │ • custom   │ • Checkov   │ • Post to PR       │
│            │   rules    │             │ • Cost estimate     │
└────────────┴────────────┴─────────────┴────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                     MERGE PIPELINE                         │
├──────────────────────────┬─────────────────────────────────┤
│          Plan            │           Apply                 │
│                          │                                 │
│ • terraform plan lại     │ • terraform apply               │
│ • Lưu plan file          │ • Dùng saved plan               │
│ • Manual approve gate    │ • Notify Slack / PagerDuty      │
└──────────────────────────┴─────────────────────────────────┘
```

---

## ⚡ So Sánh Các Công Cụ CI/CD

| Tiêu Chí | GitHub Actions | GitLab CI | Atlantis | Terraform Cloud |
|----------|----------------|-----------|----------|-----------------|
| **Dễ bắt đầu** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Self-hosted** | Có thể | Có thể | Bắt buộc | Không |
| **PR Integration** | Tốt | Tốt | Xuất sắc | Tốt |
| **Chi phí** | Theo phút | Theo phút | Miễn phí (self-host) | Theo user/run |
| **RBAC — Role-Based Access Control** | Cơ bản | Tốt | Tốt | Xuất sắc |
| **Audit Trail** | Qua logs | Qua logs | Native | Native |
| **Policy as Code** | Không native | Không native | Không native | Sentinel (paid) |

---

## 🗺️ Lộ Trình Học Phần Này

```
Bước 1: Đọc 1-github-actions.md
        → Hiểu cấu trúc workflow YAML cơ bản
        → Thiết lập workflow thực tế

Bước 2: Đọc 2-gitlab-ci.md
        → So sánh với GitHub Actions
        → Hiểu .gitlab-ci.yml

Bước 3: Đọc 3-atlantis.md
        → Hiểu PR-based workflow
        → Biết khi nào nên dùng Atlantis

Bước 4: Đọc 4-terraform-cloud.md
        → Hiểu Terraform Cloud runs
        → Biết khi nào nên dùng hosted solution

Bước 5: Đọc 5-rollback-strategy.md
        → Chuẩn bị cho sự cố
        → Thiết kế rollback plan
```

---

## 💡 Checklist Trước Khi Thiết Lập CI/CD

- [ ] Remote backend đã được cấu hình (không dùng local state)
- [ ] State locking đã được bật
- [ ] Credentials không hardcode trong code
- [ ] OIDC hoặc IAM role đã được tạo cho CI
- [ ] Terraform version được pin (không dùng `latest`)
- [ ] Plan output được lưu và dùng lại cho apply
- [ ] Có ít nhất một manual approval gate trước khi apply vào production
- [ ] Audit log được bật
- [ ] Notification được cấu hình (Slack, email, PagerDuty)
- [ ] Rollback plan đã được viết và test

---

**Tiếp Theo:** Bắt đầu với [1-github-actions.md](1-github-actions.md)

---

*Cập Nhật: 2026-05-12 | Phiên Bản: 1.0*
