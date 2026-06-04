# 🔐 Secrets & Variables — Quản Lý Bí Mật và Biến

> Hướng dẫn toàn diện về quản lý thông tin nhạy cảm trong GitHub Actions — từ Secrets (Bí Mật) và Variables (Biến) cơ bản đến OIDC (OpenID Connect — Xác Thực Không Cần Credentials Lâu Dài), tích hợp HashiCorp Vault và AWS Secrets Manager.

---

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Phân Loại Thông Tin Nhạy Cảm](#phân-loại-thông-tin-nhạy-cảm)
3. [Các File Trong Module Này](#các-file-trong-module-này)
4. [Luồng Bảo Mật Khuyến Nghị](#luồng-bảo-mật-khuyến-nghị)
5. [So Sánh Các Phương Pháp](#so-sánh-các-phương-pháp)
6. [Câu Hỏi Phỏng Vấn Phổ Biến](#câu-hỏi-phỏng-vấn-phổ-biến)

---

## Tổng Quan

Trong GitHub Actions, thông tin nhạy cảm (API keys, database passwords, cloud credentials) phải được bảo vệ nghiêm ngặt. GitHub cung cấp nhiều cơ chế:

```
Secrets (Bí Mật)         → Dữ liệu mã hóa, không hiển thị trong logs
Variables (Biến)         → Dữ liệu không mã hóa, cấu hình theo môi trường
OIDC (Xác Thực Liên Kết) → Thay thế credentials bằng token tạm thời
External Secret Managers → HashiCorp Vault, AWS Secrets Manager, v.v.
```

### Nguyên Tắc Vàng

| Nguyên Tắc | Mô Tả |
|---|---|
| **Least Privilege** (Đặc Quyền Tối Thiểu) | Chỉ cấp quyền tối thiểu cần thiết |
| **Zero Trust** (Không Tin Tưởng Mặc Định) | Luôn xác thực, không giả định an toàn |
| **Rotation** (Xoay Vòng) | Định kỳ thay đổi credentials |
| **Audit** (Kiểm Toán) | Ghi log mọi truy cập |
| **Prefer OIDC** | Dùng OIDC thay vì lưu long-lived credentials |

---

## Phân Loại Thông Tin Nhạy Cảm

```
┌─────────────────────────────────────────────────────────┐
│                    Thông Tin Nhạy Cảm                   │
├───────────────┬─────────────────┬───────────────────────┤
│   Secrets     │    Variables    │   External Managers   │
│  (Mã hóa)    │   (Không mã hóa)│   (Bên ngoài)        │
├───────────────┼─────────────────┼───────────────────────┤
│ API Keys      │ App Config      │ HashiCorp Vault       │
│ Passwords     │ Feature Flags   │ AWS Secrets Manager   │
│ Private Keys  │ URLs (public)   │ Azure Key Vault       │
│ Tokens        │ Version numbers │ GCP Secret Manager    │
└───────────────┴─────────────────┴───────────────────────┘
```

### Khi Nào Dùng Gì?

```
Secrets    → Khi dữ liệu tuyệt đối không được hiển thị (passwords, tokens)
Variables  → Khi cần cấu hình theo môi trường nhưng không nhạy cảm (URLs, flags)
OIDC       → Khi cần xác thực với cloud providers (AWS, GCP, Azure)
Vault/SM   → Khi cần quản lý secrets tập trung ở enterprise scale
```

---

## Các File Trong Module Này

| File | Chủ Đề | Độ Ưu Tiên |
|---|---|---|
| [1-secrets-management.md](1-secrets-management.md) | Repository, Organization, Environment Secrets | ⭐⭐⭐ |
| [2-variables.md](2-variables.md) | Variables — phạm vi và cách dùng | ⭐⭐⭐ |
| [3-oidc.md](3-oidc.md) | OIDC — xác thực không cần long-lived credentials | ⭐⭐⭐ |
| [4-vault-integration.md](4-vault-integration.md) | Tích hợp HashiCorp Vault | ⭐⭐ |
| [5-aws-secrets-manager.md](5-aws-secrets-manager.md) | Tích hợp AWS Secrets Manager | ⭐⭐ |

---

## Luồng Bảo Mật Khuyến Nghị

### Cho Dự Án Nhỏ / Cá Nhân

```
GitHub Secrets (Repository level) → Đủ dùng
```

### Cho Dự Án Team

```
GitHub Secrets (Organization level) + OIDC cho cloud
```

### Cho Enterprise

```
OIDC + HashiCorp Vault / Cloud Secret Manager
(Không lưu long-lived credentials ở đâu cả)
```

---

## So Sánh Các Phương Pháp

| Tiêu Chí | GitHub Secrets | Variables | OIDC | Vault/SM |
|---|---|---|---|---|
| **Mã hóa** | ✅ Luôn | ❌ Không | ✅ Token tạm | ✅ Luôn |
| **Xoay vòng tự động** | ❌ Thủ công | N/A | ✅ Tự động | ✅ Có thể |
| **Audit log** | Cơ bản | Cơ bản | ✅ Chi tiết | ✅ Chi tiết |
| **Cross-repo sharing** | Organization secrets | ✅ Dễ | N/A | ✅ Dễ |
| **Độ phức tạp** | Thấp | Thấp | Trung bình | Cao |
| **Chi phí** | Miễn phí | Miễn phí | Miễn phí | Phụ thuộc |
| **Khuyến nghị cho** | Mọi dự án | Cấu hình | Cloud auth | Enterprise |

---

## Câu Hỏi Phỏng Vấn Phổ Biến

### Câu 1: Phân biệt Secrets và Variables trong GitHub Actions?

**Secrets:**
- Dữ liệu được mã hóa (encrypted at rest và in transit)
- Không bao giờ hiển thị trong logs (bị mask — che dấu)
- Không thể đọc lại sau khi đã lưu (chỉ có thể overwrite)
- Dùng cho: passwords, API keys, private keys

**Variables:**
- Không mã hóa, có thể đọc lại trong UI
- Hiển thị trong logs (cần cẩn thận)
- Dễ chia sẻ và cấu hình
- Dùng cho: URLs, app config, feature flags

### Câu 2: OIDC là gì và tại sao nên dùng?

**OIDC — OpenID Connect — Xác Thực Danh Tính Mở:**
- Thay vì lưu AWS/GCP/Azure credentials trong GitHub Secrets
- GitHub Actions tạo JWT token tạm thời cho mỗi workflow run
- Cloud provider tin tưởng GitHub's OIDC provider
- Token tự hết hạn sau khi job kết thúc

**Lợi ích:**
- Không có long-lived credentials nào bị lộ
- Không cần rotation thủ công
- Audit trail rõ ràng hơn
- Giảm attack surface (Bề Mặt Tấn Công)

### Câu 3: Làm thế nào để secrets không bị lộ trong fork PRs?

```yaml
# Secrets KHÔNG được truyền vào fork PRs theo mặc định
# Chỉ từ pull_request_target mới có secrets (CẨN THẬN với code thực thi!)

# Pattern an toàn: chỉ chạy test cơ bản với fork PRs
on:
  pull_request:        # Không có secrets — an toàn cho fork
  pull_request_target: # CÓ secrets — CHỈ dùng cho code trusted
```

### Câu 4: Khi nào nên dùng HashiCorp Vault thay GitHub Secrets?

- Khi cần dynamic secrets (secrets thay đổi mỗi lần truy cập)
- Khi có nhiều pipelines trên nhiều platforms (không chỉ GitHub)
- Khi cần fine-grained access control phức tạp
- Khi compliance yêu cầu centralized secrets management
- Khi cần secret versioning và rollback

---

## ⚡ Quick Reference

```yaml
# Dùng secret
- run: echo "${{ secrets.MY_SECRET }}"

# Dùng variable
- run: echo "${{ vars.MY_VAR }}"

# Dùng OIDC với AWS
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/MyRole
    aws-region: us-east-1

# Dùng Vault
- uses: hashicorp/vault-action@v3
  with:
    url: https://vault.example.com
    method: jwt
    path: jwt-github
    secrets: secret/data/prod password | MY_PASSWORD
```

---

**Điều Hướng:**
- ← [03-cd-deployments/README.md](../03-cd-deployments/README.md)
- → [1-secrets-management.md](1-secrets-management.md)
- [Lên INDEX](../INDEX.md)

**Cập Nhật Lần Cuối:** 2026-05-11
