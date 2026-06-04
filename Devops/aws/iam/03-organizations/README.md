# AWS Organizations & Multi-Account Strategy — Chiến Lược Đa Tài Khoản

> **AWS Organizations** là dịch vụ quản lý tập trung nhiều tài khoản AWS trong một tổ chức duy nhất — cho phép áp dụng chính sách bảo mật, kiểm soát chi phí và tự động hóa quản trị ở quy mô doanh nghiệp.

---

## 📚 Nội Dung Module

| File | Chủ Đề | Độ Ưu Tiên |
|---|---|---|
| [1-organizations-setup.md](1-organizations-setup.md) | Thiết lập Organizations — OU hierarchy, tài khoản thành viên | ⭐⭐⭐ Bắt buộc |
| [2-service-control-policies.md](2-service-control-policies.md) | SCPs — cú pháp, ví dụ thực tế, giới hạn và lỗi thường gặp | ⭐⭐⭐ Bắt buộc |
| [3-control-tower.md](3-control-tower.md) | AWS Control Tower — landing zone tự động, guardrails | ⭐⭐ Quan trọng |
| [4-account-vending.md](4-account-vending.md) | Account Vending — tự động tạo và cấu hình tài khoản mới | ⭐⭐ Quan trọng |

---

## 🧭 Tổng Quan Nhanh

### Tại Sao Cần Nhiều Tài Khoản AWS?

Câu hỏi phổ biến: *"Tại sao không dùng một tài khoản rồi phân chia bằng IAM?"*

**Đơn tài khoản — giới hạn thực tế:**

```
Vấn đề 1: Blast Radius (Phạm Vi Ảnh Hưởng)
  → Một lỗi cấu hình IAM hoặc credentials bị rò rỉ
    có thể ảnh hưởng toàn bộ workloads

Vấn đề 2: Service Limits (Giới Hạn Dịch Vụ)
  → AWS limits áp dụng per-account
    (ví dụ: 5 VPCs, 100 S3 buckets mặc định)

Vấn đề 3: Audit & Compliance (Kiểm Toán & Tuân Thủ)
  → Khó tách biệt phạm vi audit cho từng team/môi trường

Vấn đề 4: Cost Visibility (Minh Bạch Chi Phí)
  → Khó phân bổ chi phí theo team/product
```

**Đa tài khoản — lợi ích:**

```
✅ Workload Isolation     — lỗi ở môi trường dev không ảnh hưởng production
✅ Security Boundary      — IAM không thể vượt qua ranh giới tài khoản
✅ Blast Radius Control   — giới hạn ảnh hưởng khi sự cố xảy ra
✅ Compliance Scope       — khoanh vùng phạm vi audit/tuân thủ
✅ Independent Limits     — mỗi account có service limits riêng
✅ Cost Attribution       — chi phí rõ ràng theo account/team
```

---

## 🏗️ Kiến Trúc AWS Organizations

### Cấu Trúc Phân Cấp

```
Root (Gốc tổ chức)
├── Management Account (Tài Khoản Quản Lý — trước đây gọi là Master)
│   └── Toàn quyền quản lý Organizations
│
├── OU: Security (Đơn Vị Tổ Chức: Bảo Mật)
│   ├── Log Archive Account     — lưu trữ CloudTrail, Config logs
│   └── Security Tooling Account — GuardDuty, Security Hub admin
│
├── OU: Infrastructure (Hạ Tầng)
│   ├── Network Account         — Transit Gateway, DNS, Direct Connect
│   └── Shared Services Account — AMI registry, artifact store
│
├── OU: Workloads (Khối Lượng Công Việc)
│   ├── OU: Production (Sản Xuất)
│   │   ├── Prod Account A
│   │   └── Prod Account B
│   └── OU: Non-Production (Phi Sản Xuất)
│       ├── Dev Account
│       ├── Staging Account
│       └── Sandbox Account
│
└── OU: Exceptions (Ngoại Lệ)
    └── Legacy Account          — tài khoản cũ cần xử lý riêng
```

### Các Thành Phần Cốt Lõi

| Thành Phần | Mô Tả | Vai Trò |
|---|---|---|
| **Management Account** (Tài Khoản Quản Lý) | Tài khoản tạo tổ chức | Quản trị toàn bộ, không chạy workload |
| **Member Accounts** (Tài Khoản Thành Viên) | Tài khoản được mời/tạo | Chạy workloads thực tế |
| **Root** (Gốc) | Đơn vị tổ chức cao nhất | Chứa toàn bộ accounts/OUs |
| **OU** (Organizational Unit — Đơn Vị Tổ Chức) | Nhóm tài khoản theo logic | Phân cấp và áp dụng chính sách |
| **SCP** (Service Control Policy — Chính Sách Kiểm Soát Dịch Vụ) | Giới hạn quyền IAM tối đa | Guardrail cho toàn OU/Account |

---

## 🎯 Mô Hình Landing Zone

**Landing Zone** (Vùng Đáp) là môi trường multi-account được thiết lập chuẩn ban đầu, bao gồm:

```
Landing Zone Components:
├── Account Structure    — OU hierarchy chuẩn
├── Identity             — IAM Identity Center (SSO) cho toàn tổ chức
├── Logging              — CloudTrail + Config tập trung về Log Archive
├── Network              — VPC baseline, Transit Gateway (nếu cần)
├── Security Baseline    — GuardDuty, Security Hub bật mặc định
└── Guardrails           — SCPs ngăn hành vi nguy hiểm
```

---

## 📋 Mô Hình Multi-Account Phổ Biến

### Mô Hình 1: Phân Tầng Theo Môi Trường (Nhỏ–Trung)

```
Organization
├── Management Account
├── OU: Shared Services
│   └── Shared Account (networking, monitoring, tools)
├── OU: Production
│   └── Prod Account
└── OU: Non-Production
    ├── Dev Account
    └── Staging Account
```

**Khi dùng:** Team nhỏ, 1–3 sản phẩm, yêu cầu tuân thủ không phức tạp

### Mô Hình 2: Phân Tầng Theo Workload (Trung–Lớn)

```
Organization
├── Management Account
├── OU: Security
│   ├── Log Archive Account
│   └── Security Tooling Account
├── OU: Infrastructure
│   ├── Network Account
│   └── Shared Services Account
└── OU: Workloads
    ├── OU: Product-A
    │   ├── Product-A-Prod
    │   └── Product-A-Dev
    └── OU: Product-B
        ├── Product-B-Prod
        └── Product-B-Dev
```

**Khi dùng:** Nhiều product teams, yêu cầu isolation cao, compliance phức tạp

### Mô Hình 3: AWS Recommended (Doanh Nghiệp Lớn)

Đây là mô hình AWS khuyến nghị trong tài liệu **AWS Security Reference Architecture**:

```
Organization Root
├── Security OU
│   ├── Log Archive
│   └── Security Tooling
├── Infrastructure OU
│   ├── Network
│   └── Shared Services
├── Sandbox OU
│   └── Individual sandbox accounts (1 per dev)
├── Workloads OU
│   ├── OU: Domain-A
│   └── OU: Domain-B
├── Policy Staging OU       ← thử nghiệm SCP mới trước khi rollout
└── Suspended OU            ← tài khoản cần đóng/kiểm tra
```

---

## 🔑 Các Tính Năng Quan Trọng

### Consolidated Billing (Thanh Toán Hợp Nhất)

```
✅ Tự động khi dùng Organizations
✅ Volume discounts — tổng hợp usage để đủ điều kiện giảm giá
✅ Reserved Instance sharing — RI/Savings Plans chia sẻ giữa accounts
✅ Single invoice — một hóa đơn cho toàn tổ chức
```

### Delegated Administration (Quản Trị Ủy Quyền)

Cho phép tài khoản thành viên quản lý một số dịch vụ AWS thay Management Account:

```
Dịch vụ hỗ trợ Delegated Admin:
- GuardDuty       → security tooling account làm admin
- Security Hub    → security tooling account làm admin
- AWS Config      → security tooling account làm admin
- IAM Access Analyzer
- Inspector
- Macie
- Firewall Manager
```

### Service Trust Policies (Chính Sách Tin Cậy Dịch Vụ)

Bật/tắt tích hợp giữa AWS Organizations và các dịch vụ AWS:

```bash
# Bật tích hợp với CloudTrail
aws organizations enable-aws-service-access \
  --service-principal cloudtrail.amazonaws.com

# Xem danh sách tích hợp đang bật
aws organizations list-aws-service-access-for-organization
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao không dùng một AWS account với nhiều VPC thay vì multi-account?**

> Multi-account cung cấp ranh giới bảo mật cứng mà IAM không thể vượt qua. VPCs trong cùng account vẫn chia sẻ IAM namespace, service limits, và blast radius. Với multi-account, compromise ở một account không tự động ảnh hưởng account khác.

**Q: Management Account nên chứa workloads không?**

> Không. Management Account có quyền đặc biệt đối với toàn tổ chức và không thể bị giới hạn bởi SCPs. Chạy workloads trên Management Account tăng rủi ro đáng kể — thay vào đó, chỉ dùng cho quản trị Organizations.

**Q: SCPs và IAM Policies khác nhau như thế nào?**

> SCPs (Service Control Policies) là guardrails — giới hạn quyền *tối đa* có thể cấp trong account, nhưng không tự cấp quyền. IAM Policies là authorization — thực sự cấp quyền cho principal. Quyền thực tế là phần giao nhau của SCPs và IAM Policies.

**Q: Có bao nhiêu tài khoản thì nên dùng AWS Control Tower?**

> Control Tower hữu ích từ 5–10 tài khoản trở lên, hoặc khi cần tự động hóa việc tạo tài khoản mới với cấu hình chuẩn. Với 2–4 tài khoản, cấu hình thủ công vẫn quản lý được.

---

## 🗺️ Lộ Trình Học Module Này

```
Bước 1: [1-organizations-setup.md]         — Thiết lập Organizations, OU hierarchy (45 phút)
Bước 2: [2-service-control-policies.md]    — Viết và áp dụng SCPs (60 phút)
Bước 3: [3-control-tower.md]               — Landing zone tự động (45 phút)
Bước 4: [4-account-vending.md]             — Tự động hóa tạo tài khoản (30 phút)
Lab:    Tạo Organizations với 3 accounts, áp dụng SCP deny region (90 phút)
```

---

## 🔗 Tiếp Theo

Sau khi hoàn thành module này, tiếp tục với:

- **[04-encryption-kms/](../04-encryption-kms/README.md)** — Mã hóa dữ liệu với KMS
- **[02-identity-federation/1-iam-identity-center.md](../02-identity-federation/1-iam-identity-center.md)** — SSO đa tài khoản
- **[07-monitoring-auditing/](../07-monitoring-auditing/README.md)** — CloudTrail tập trung cho Organizations

---

**Thời Gian Học Ước Tính:** 3–4 giờ lý thuyết + 4–6 giờ thực hành lab
