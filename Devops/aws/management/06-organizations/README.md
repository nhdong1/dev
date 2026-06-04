# AWS Organizations — Quản Lý Đa Tài Khoản (Multi-Account Management)

> **AWS Organizations** là dịch vụ quản lý tập trung cho nhiều tài khoản AWS — cho phép nhóm account vào cây tổ chức (organizational hierarchy), áp dụng chính sách kiểm soát (SCP — Service Control Policy — Chính Sách Kiểm Soát Dịch Vụ) và hợp nhất thanh toán (Consolidated Billing — Hóa Đơn Hợp Nhất). Đây là nền tảng bắt buộc cho mọi môi trường doanh nghiệp nghiêm túc trên AWS.

---

## 📚 Mục Lục

1. [Tại Sao Cần Nhiều AWS Account?](#tại-sao-cần-nhiều-aws-account)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#các-khái-niệm-cốt-lõi)
4. [Luồng Hoạt Động Điển Hình](#luồng-hoạt-động-điển-hình)
5. [Các File Trong Module Này](#các-file-trong-module-này)
6. [So Sánh Nhanh: Organizations vs Control Tower](#so-sánh-nhanh)
7. [Câu Hỏi Phỏng Vấn Phổ Biến](#câu-hỏi-phỏng-vấn-phổ-biến)

---

## Tại Sao Cần Nhiều AWS Account?

### Vấn Đề Khi Dùng Một Account Duy Nhất

```
Một account cho tất cả → Dev vô tình xóa dữ liệu production →
Không có security boundary giữa các team →
Blast radius (Phạm Vi Ảnh Hưởng) của sự cố lan toàn bộ công ty →
Khó kiểm soát chi phí theo team/dự án →
Không thể enforce policy khác nhau cho môi trường khác nhau
```

### Lợi Ích Của Multi-Account Strategy (Chiến Lược Đa Tài Khoản)

| Lợi Ích | Mô Tả |
|---------|-------|
| **Security Boundary** (Ranh Giới Bảo Mật) | Compromise 1 account không ảnh hưởng account khác |
| **Blast Radius Isolation** (Cô Lập Phạm Vi Ảnh Hưởng) | Sự cố, lỗi cấu hình giới hạn trong một account |
| **Billing Separation** (Tách Biệt Thanh Toán) | Theo dõi chi phí theo team, project, môi trường |
| **Compliance Isolation** (Cô Lập Tuân Thủ) | PCI-DSS scope giới hạn trong account chuyên dụng |
| **Service Quota Isolation** (Cô Lập Hạn Mức Dịch Vụ) | Quota của team A không ảnh hưởng team B |
| **Least Privilege** (Quyền Tối Thiểu) | Developer không có quyền vào account production |

---

## Kiến Trúc Tổng Quan

```
Organization Root (Gốc Tổ Chức)
│
├── Management Account (Tài Khoản Quản Lý) ← Trước gọi là "Master Account"
│   ├── Quản lý toàn bộ Organization
│   ├── Thanh toán hợp nhất (Consolidated Billing)
│   └── KHÔNG nên chạy workload ứng dụng ở đây
│
├── OU: Security (Đơn Vị Tổ Chức: Bảo Mật)
│   ├── Log Archive Account — Centralized logging
│   └── Security Tooling Account — GuardDuty, Security Hub
│
├── OU: Infrastructure
│   ├── Shared Services Account — DNS, AD, transit networking
│   └── Network Account — Transit Gateway, VPN
│
├── OU: Workloads (Khối Lượng Công Việc)
│   ├── OU: Production
│   │   ├── Account: app-prod
│   │   └── Account: data-prod
│   │
│   └── OU: Non-Production
│       ├── Account: app-dev
│       ├── Account: app-staging
│       └── Account: data-dev
│
└── OU: Sandbox (Môi Trường Thử Nghiệm)
    └── Account: engineer-sandbox-*
```

### Các Thành Phần Chính

```
AWS Organizations
├── Root — Điểm gốc của toàn bộ cây tổ chức (chỉ có 1)
├── Management Account — Account điều hành Organizations
├── OU (Organizational Unit) — Đơn Vị Tổ Chức — nhóm account
├── Member Account — Tài Khoản Thành Viên — leaf nodes
├── SCP (Service Control Policy) — Chính sách gắn vào Root/OU/Account
└── Tag Policy — Chính sách chuẩn hóa tags
```

---

## Các Khái Niệm Cốt Lõi

### Management Account vs Member Account

| Tiêu Chí | Management Account | Member Account |
|----------|-------------------|----------------|
| **Vai trò** | Điều hành toàn Organization | Chạy workload thực tế |
| **Billing** | Nhận hóa đơn hợp nhất | Không có hóa đơn riêng |
| **SCP** | SCP không giới hạn Management Account | SCP áp dụng đầy đủ |
| **Số lượng** | Chỉ 1 | Không giới hạn (soft limit) |
| **Thực hành tốt** | Không chạy workload | Chạy workload theo mục đích |

### OU (Organizational Unit — Đơn Vị Tổ Chức)

- Cây tổ chức có thể sâu tối đa **5 cấp** (Root → OU cấp 1 → ... → Account)
- Một OU có thể chứa account và OU con
- SCP gắn vào OU áp dụng cho tất cả account trong OU đó (kế thừa)
- Account chỉ thuộc **1 OU** tại một thời điểm

### SCP (Service Control Policy — Chính Sách Kiểm Soát Dịch Vụ)

- Giới hạn **tối đa quyền** (maximum permissions) cho tất cả IAM entities trong account
- **Không** cấp quyền — chỉ giới hạn quyền đã có
- SCP **không áp dụng** cho Management Account
- SCP áp dụng theo cơ chế **AND** giữa các cấp (phải được phép ở mọi cấp)

> Chi tiết xem [2-scp-policies.md](./2-scp-policies.md)

---

## Luồng Hoạt Động Điển Hình

### Tạo Organization Và Mời Account

```bash
# 1. Tạo Organization (chạy trong Management Account)
aws organizations create-organization --feature-set ALL

# 2. Tạo OU
aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name Production

# 3. Tạo account mới (Account Vending)
aws organizations create-account \
  --email app-prod@company.com \
  --account-name "App Production"

# 4. Mời account đã tồn tại
aws organizations invite-account-to-organization \
  --target '{"Type": "EMAIL", "Id": "existing@company.com"}'

# 5. Di chuyển account vào OU
aws organizations move-account \
  --account-id 123456789012 \
  --source-parent-id r-xxxx \
  --destination-parent-id ou-xxxx-yyyyyyyy
```

### Áp Dụng SCP

```bash
# Tạo SCP
aws organizations create-policy \
  --name "DenyLeaveOrganization" \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-leave-org.json

# Gắn SCP vào OU
aws organizations attach-policy \
  --policy-id p-xxxxxxxxxxxx \
  --target-id ou-xxxx-yyyyyyyy
```

---

## Các File Trong Module Này

| File | Nội Dung |
|------|----------|
| [1-account-structure.md](./1-account-structure.md) | Management Account, Member Accounts, OU hierarchy, Root |
| [2-scp-policies.md](./2-scp-policies.md) | SCP deny list vs allow list, kế thừa, ví dụ thực tế |
| [3-consolidated-billing.md](./3-consolidated-billing.md) | Billing hợp nhất, volume discount, RI/Savings Plans sharing |
| [4-delegated-admin.md](./4-delegated-admin.md) | Delegated Administrator, Trusted Access với AWS services |
| [5-multi-account-patterns.md](./5-multi-account-patterns.md) | Account vending, OU design patterns, account types |

---

## So Sánh Nhanh

### AWS Organizations vs AWS Control Tower

| Tiêu Chí | AWS Organizations | AWS Control Tower |
|----------|------------------|-------------------|
| **Bản chất** | Dịch vụ cốt lõi | Lớp tự động hóa trên Organizations |
| **Setup** | Thủ công, linh hoạt | Tự động hóa, có opinionated defaults |
| **Guardrails** | SCP tự viết | Guardrail có sẵn (150+) |
| **Account tạo mới** | CLI/API | Account Factory tự động |
| **Dashboard** | Cơ bản | Compliance dashboard đầy đủ |
| **Phù hợp** | Team có kinh nghiệm, tùy chỉnh cao | Bắt đầu nhanh, best practice có sẵn |

> Organizations là **nền tảng** — Control Tower **xây trên nền đó**. Mọi Control Tower đều cần Organizations, không phải ngược lại.

### Feature Sets (Tính Năng Organizations)

| Feature Set | Consolidated Billing | SCP | Tag Policy | AI Services Opt-Out |
|-------------|---------------------|-----|------------|---------------------|
| **CONSOLIDATED_BILLING** | ✅ | ❌ | ❌ | ❌ |
| **ALL** | ✅ | ✅ | ✅ | ✅ |

---

## Câu Hỏi Phỏng Vấn Phổ Biến

### Cơ Bản

**Q: Tại sao nên dùng nhiều AWS account thay vì một account duy nhất?**
> Nhiều account mang lại **security boundary** thực sự — không thể vượt qua bằng IAM policy. Mỗi account là ranh giới cô lập blast radius, giúp tách biệt môi trường (dev/staging/prod), kiểm soát chi phí theo team, và giới hạn compliance scope (ví dụ: PCI-DSS chỉ trong account xử lý thanh toán).

**Q: SCP khác IAM Policy thế nào?**
> **IAM Policy** cấp quyền cho từng user/role trong account. **SCP** giới hạn *tối đa* những gì IAM policy có thể cấp, áp dụng cho toàn bộ account. SCP không cấp quyền, chỉ thu hẹp quyền — ngay cả root user của account cũng bị SCP giới hạn (trừ Management Account).

**Q: Management Account có bị SCP giới hạn không?**
> **Không.** SCP không áp dụng cho Management Account. Đây là lý do cần bảo vệ Management Account đặc biệt — không chạy workload, bật MFA bắt buộc, không tạo nhiều IAM users ở đây.

### Nâng Cao

**Q: Thiết kế OU hierarchy như thế nào cho doanh nghiệp 50 team?**
> Không thiết kế OU theo team mà theo **môi trường và bảo mật requirements**:
> - `Security OU` → Log Archive, Security Tooling
> - `Infrastructure OU` → Shared Services, Networking
> - `Workloads/Production OU` → Production accounts (SCP nghiêm ngặt)
> - `Workloads/NonProd OU` → Dev, Staging (SCP thoải mái hơn)
> - `Sandbox OU` → Engineer sandbox (SCP giới hạn region, không có production data)
> Thiết kế theo môi trường giúp áp dụng SCP đúng cấp, không phải tạo SCP riêng cho từng team.

**Q: Delegated Administrator là gì và dùng khi nào?**
> Cho phép **uỷ quyền quản lý** một AWS service cụ thể (Security Hub, GuardDuty, Config...) cho một member account thay vì phải làm mọi thứ từ Management Account. Best practice: có một Security Tooling account là Delegated Admin cho các security services — tránh cấp quá nhiều quyền cho Management Account.

---

## 🔗 Điều Hướng

| Trước | Module Này | Tiếp Theo |
|-------|-----------|-----------|
| [05-cloudformation/](../05-cloudformation/README.md) | **06-organizations/** | [07-control-tower/](../07-control-tower/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
