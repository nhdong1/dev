# 🔒 Bảo Mật GitHub Actions — Tổng Quan

> Hướng dẫn bảo mật toàn diện cho GitHub Actions — từ quyền truy cập tối thiểu (least-privilege) đến bảo vệ chuỗi cung ứng phần mềm (software supply chain security).

---

## 📚 Mục Lục Chủ Đề

| File | Nội Dung | Độ Ưu Tiên |
|---|---|---|
| [1-permissions.md](./1-permissions.md) | `permissions` block, GITHUB_TOKEN scopes | ⭐⭐⭐ Bắt buộc |
| [2-oidc-cloud-auth.md](./2-oidc-cloud-auth.md) | OIDC với AWS/GCP/Azure — không cần long-lived secrets | ⭐⭐⭐ Bắt buộc |
| [3-supply-chain.md](./3-supply-chain.md) | Pin actions by SHA, Dependabot, dependency review | ⭐⭐⭐ Bắt buộc |
| [4-code-scanning.md](./4-code-scanning.md) | CodeQL, SAST, DAST tích hợp trong CI | ⭐⭐ Nên có |
| [5-secret-scanning.md](./5-secret-scanning.md) | Phát hiện secrets bị lộ trong code | ⭐⭐ Nên có |
| [6-security-hardening.md](./6-security-hardening.md) | Checklist bảo mật toàn diện cho enterprise | ⭐⭐ Nên có |

---

## 🎯 Tại Sao Bảo Mật GitHub Actions Quan Trọng?

GitHub Actions chạy code với quyền truy cập rộng vào:

- **Repository** — đọc/ghi code, tạo branches, merge PRs
- **Secrets** — API keys, cloud credentials, database passwords
- **Cloud infrastructure** — deploy, thay đổi cấu hình hệ thống
- **Third-party services** — Slack, Jira, external APIs

Một workflow bị tấn công có thể dẫn đến:
- **Data breach** (Rò rỉ dữ liệu) — lộ secrets, source code
- **Supply chain attack** (Tấn công chuỗi cung ứng) — inject malicious code vào build
- **Privilege escalation** (Leo thang đặc quyền) — từ runner lên cloud infrastructure
- **Service disruption** (Gián đoạn dịch vụ) — xóa resources, phá hủy deployment

---

## 🏛️ Mô Hình Bảo Mật Theo Lớp

```
┌─────────────────────────────────────────────────────┐
│  Lớp 5: Supply Chain Security (Chuỗi Cung Ứng)     │
│  → Pin actions by SHA, Dependabot alerts            │
├─────────────────────────────────────────────────────┤
│  Lớp 4: Code & Secret Scanning (Quét Mã & Bí Mật)  │
│  → CodeQL, SAST, DAST, secret detection             │
├─────────────────────────────────────────────────────┤
│  Lớp 3: Cloud Authentication (Xác Thực Cloud)       │
│  → OIDC, Workload Identity, short-lived tokens      │
├─────────────────────────────────────────────────────┤
│  Lớp 2: Secrets Management (Quản Lý Bí Mật)        │
│  → Encrypted secrets, environment protection        │
├─────────────────────────────────────────────────────┤
│  Lớp 1: Least Privilege (Quyền Tối Thiểu)          │
│  → permissions block, GITHUB_TOKEN scopes           │
└─────────────────────────────────────────────────────┘
```

---

## 🚨 Top 10 Rủi Ro Bảo Mật Phổ Biến

### 1. Permissions Quá Rộng (Overly Permissive Tokens)

```yaml
# ❌ SAI — token có toàn quyền
permissions: write-all

# ✅ ĐÚNG — chỉ cấp quyền cần thiết
permissions:
  contents: read
  pull-requests: write
```

### 2. Sử Dụng Actions Không Pin SHA

```yaml
# ❌ NGUY HIỂM — version tag có thể bị thay đổi
- uses: actions/checkout@v4

# ✅ AN TOÀN — SHA không thể bị thay đổi
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

### 3. In Secrets Ra Logs

```yaml
# ❌ SAI — secrets sẽ bị lộ trong logs
- run: echo "Token = ${{ secrets.API_TOKEN }}"

# ✅ ĐÚNG — dùng biến môi trường thay vì interpolation trực tiếp
- run: ./deploy.sh
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
```

### 4. Script Injection Từ PR

```yaml
# ❌ NGUY HIỂM — tên PR có thể chứa shell injection
- run: echo "PR title: ${{ github.event.pull_request.title }}"

# ✅ AN TOÀN — sử dụng biến môi trường
- run: echo "PR title: $PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

### 5. Pull Request Từ Fork Không Được Kiểm Soát

```yaml
# ❌ NGUY HIỂM — fork PR có thể chạy với write permissions
on:
  pull_request_target:  # Trigger này có quyền secrets!
    types: [opened]

# ✅ ĐÚNG — dùng pull_request thay thế (không có secrets)
on:
  pull_request:
    types: [opened]
```

---

## 🔐 Các Khái Niệm Cốt Lõi

### GITHUB_TOKEN — Token Tự Động

- **Định nghĩa:** Token tạm thời tự động cấp cho mỗi workflow run
- **Phạm vi:** Chỉ có quyền truy cập repository hiện tại
- **Thời hạn:** Hết hạn khi workflow kết thúc
- **Cấu hình:** Có thể giới hạn qua `permissions` block

### OIDC — OpenID Connect — Xác Thực Không Cần Secrets

- **Vấn đề giải quyết:** Không cần lưu long-lived cloud credentials vào GitHub Secrets
- **Cơ chế:** GitHub phát JWT token, cloud provider xác thực và cấp short-lived token
- **Hỗ trợ:** AWS, GCP, Azure, HashiCorp Vault

### Supply Chain Attack — Tấn Công Chuỗi Cung Ứng

- **Định nghĩa:** Kẻ tấn công chiếm quyền kiểm soát một dependency (action) và inject malicious code
- **Ví dụ thực tế:** tj-actions/changed-files bị tấn công năm 2023
- **Giải pháp:** Pin actions bằng commit SHA thay vì version tag

### SAST — Static Application Security Testing — Kiểm Tra Bảo Mật Tĩnh

- **Định nghĩa:** Phân tích source code để tìm lỗ hổng bảo mật mà không cần chạy code
- **Công cụ:** CodeQL (GitHub), Semgrep, Snyk Code

### DAST — Dynamic Application Security Testing — Kiểm Tra Bảo Mật Động

- **Định nghĩa:** Kiểm tra bảo mật bằng cách tấn công ứng dụng đang chạy
- **Công cụ:** OWASP ZAP, Burp Suite

---

## 📋 Checklist Bảo Mật Nhanh

### Mức Cơ Bản (bắt buộc cho mọi repo)

- [ ] Thêm `permissions` block tường minh vào mọi workflow
- [ ] Đặt `permissions: read-all` hoặc `permissions: {}` ở đầu workflow
- [ ] Không in secrets vào logs
- [ ] Không dùng `pull_request_target` với code từ fork

### Mức Trung Cấp (khuyến nghị)

- [ ] Pin tất cả third-party actions bằng SHA
- [ ] Bật Dependabot cho Actions updates
- [ ] Enable secret scanning trên repository
- [ ] Dùng OIDC thay vì lưu cloud credentials
- [ ] Enable code scanning (CodeQL)

### Mức Nâng Cao (enterprise)

- [ ] Thiết lập required reviewers cho environment production
- [ ] Dùng Organization-level secrets với environment restrictions
- [ ] Triển khai dependency review trong PR workflow
- [ ] Audit log review hàng tuần
- [ ] Runner isolation — ephemeral self-hosted runners

---

## 🗺️ Lộ Trình Học Bảo Mật

```
Tuần 1: Nền Tảng
  → 1-permissions.md (GITHUB_TOKEN, least privilege)
  → 2-oidc-cloud-auth.md (OIDC với AWS/GCP)

Tuần 2: Supply Chain & Scanning
  → 3-supply-chain.md (pin SHA, Dependabot)
  → 4-code-scanning.md (CodeQL, SAST)
  → 5-secret-scanning.md (phát hiện secrets bị lộ)

Tuần 3: Enterprise Hardening
  → 6-security-hardening.md (checklist toàn diện)
```

---

## 🔗 Liên Kết Liên Quan

- [04-secrets-variables/3-oidc.md](../04-secrets-variables/3-oidc.md) — OIDC chi tiết
- [09-self-hosted-runners/3-security-isolation.md](../09-self-hosted-runners/3-security-isolation.md) — Runner isolation
- [05-reusable/5-marketplace-guide.md](../05-reusable/5-marketplace-guide.md) — Đánh giá actions an toàn

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
