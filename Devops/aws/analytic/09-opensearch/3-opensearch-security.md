# OpenSearch Security — Bảo Mật Amazon OpenSearch Service

> Bảo mật OpenSearch theo nhiều lớp: network isolation (Cô Lập Mạng) với VPC, fine-grained access control — FGAC (Kiểm Soát Truy Cập Tinh Tế) ở cấp index/field/document, encryption (Mã Hóa) dữ liệu khi lưu trữ và truyền tải, cùng với audit logging (Ghi Log Kiểm Tra) để đáp ứng compliance (Tuân Thủ) yêu cầu.

---

## 📚 Mục Lục

1. [Tổng Quan Mô Hình Bảo Mật](#1-tổng-quan-mô-hình-bảo-mật)
2. [Network Access — Kiểm Soát Mạng](#2-network-access--kiểm-soát-mạng)
3. [Domain Access Policy — Chính Sách Truy Cập](#3-domain-access-policy--chính-sách-truy-cập)
4. [Fine-grained Access Control (FGAC)](#4-fine-grained-access-control-fgac)
5. [Encryption — Mã Hóa](#5-encryption--mã-hóa)
6. [SAML Authentication — Xác Thực SAML](#6-saml-authentication--xác-thực-saml)
7. [Audit Logging — Ghi Log Kiểm Tra](#7-audit-logging--ghi-log-kiểm-tra)
8. [Best Practices — Thực Hành Tốt Nhất](#8-best-practices--thực-hành-tốt-nhất)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Mô Hình Bảo Mật

OpenSearch Service áp dụng mô hình **defense in depth** (Bảo Mật Theo Chiều Sâu) — nhiều lớp bảo vệ:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Lớp 1: Network Security                       │
│  VPC + Security Groups + NACLs + Private Endpoints              │
├─────────────────────────────────────────────────────────────────┤
│                    Lớp 2: Transport Security                     │
│  TLS 1.2/1.3 — Encryption in transit (Mã Hóa Truyền Tải)       │
├─────────────────────────────────────────────────────────────────┤
│                    Lớp 3: Domain Access Policy                   │
│  IAM Resource-based policy — ai được kết nối đến domain         │
├─────────────────────────────────────────────────────────────────┤
│                    Lớp 4: Authentication                         │
│  IAM / Internal users / SAML / Cognito                          │
├─────────────────────────────────────────────────────────────────┤
│                    Lớp 5: Fine-grained Access Control            │
│  Index / Document / Field level permissions                      │
├─────────────────────────────────────────────────────────────────┤
│                    Lớp 6: Encryption at Rest                     │
│  AWS KMS — mã hóa dữ liệu lưu trữ trên EBS/S3                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Network Access — Kiểm Soát Mạng

### 2.1 VPC vs Public Domain

| | **VPC Deployment** | **Public Domain** |
|--|-------------------|-------------------|
| **Endpoint** | Private IP trong VPC | Public HTTPS endpoint |
| **Truy cập từ Internet** | Không (cần VPN/Direct Connect) | Có (nếu policy cho phép) |
| **Bảo mật** | Cao hơn — network isolation | Phụ thuộc vào IAM policy |
| **Phù hợp** | Production, dữ liệu nhạy cảm | Dev/Test, internal tools |

### 2.2 VPC Deployment — Triển Khai Trong VPC

```
Internet
    │ (Không thể kết nối trực tiếp)
    ✗
    │
VPC (Virtual Private Cloud — Đám Mây Riêng Ảo)
├── Public Subnet
│   └── Application Load Balancer / Bastion Host
│           │ (chỉ kết nối nội bộ VPC)
├── Private Subnet (AZ-a)
│   └── OpenSearch Data Node 1
│   └── OpenSearch Master Node
└── Private Subnet (AZ-b)
    └── OpenSearch Data Node 2

Security Group Rules (Quy Tắc Nhóm Bảo Mật):
    Inbound: Port 443 (HTTPS) từ Security Group của application
    Outbound: Tất cả (để nodes giao tiếp với nhau)
```

### 2.3 Interface Endpoint (VPC Endpoint)

Cho phép Lambda trong VPC kết nối đến OpenSearch mà không cần Internet Gateway:

```
Lambda (trong VPC)
    │
    │ VPC Interface Endpoint (AWS PrivateLink)
    ▼
OpenSearch Domain
(Traffic không ra Internet)
```

---

## 3. Domain Access Policy — Chính Sách Truy Cập

### 3.1 Resource-based Policy — Chính Sách Dựa Trên Tài Nguyên

Domain access policy kiểm soát **ai** có thể kết nối đến OpenSearch domain ở cấp AWS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:role/my-app-role"
      },
      "Action": "es:*",
      "Resource": "arn:aws:es:ap-southeast-1:123456789:domain/my-domain/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:role/data-analyst-role"
      },
      "Action": [
        "es:ESHttpGet",
        "es:ESHttpPost"
      ],
      "Resource": "arn:aws:es:ap-southeast-1:123456789:domain/my-domain/*"
    }
  ]
}
```

### 3.2 IP-based Access — Truy Cập Theo Địa Chỉ IP

Dành cho Public domain, giới hạn theo IP (chỉ phù hợp khi IP cố định):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "*" },
      "Action": "es:*",
      "Resource": "arn:aws:es:ap-southeast-1:123456789:domain/my-domain/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": ["203.0.113.0/24", "198.51.100.5/32"]
        }
      }
    }
  ]
}
```

---

## 4. Fine-grained Access Control (FGAC)

### 4.1 FGAC — Kiểm Soát Truy Cập Tinh Tế Là Gì?

FGAC cho phép kiểm soát quyền truy cập ở mức độ rất chi tiết: không chỉ "ai được kết nối domain" mà còn "ai được đọc/ghi index nào, thậm chí field nào và document nào".

```
Không có FGAC:          Có FGAC:
                 
User A ──▶ Xem          User A ──▶ Chỉ đọc index "public-logs"
           tất cả       User B ──▶ Đọc/ghi index "app-logs"
           index        User C ──▶ Đọc "orders" nhưng KHÔNG thấy field "customer_id"
                        User D ──▶ Chỉ thấy orders của region "North" (document-level)
```

### 4.2 Roles — Vai Trò Trong FGAC

FGAC sử dụng hệ thống role (Vai Trò) riêng biệt với IAM:

```
OpenSearch Internal Role System:
├── Built-in Roles (Vai Trò Tích Hợp Sẵn)
│   ├── all_access          — Toàn quyền (tương đương admin)
│   ├── security_manager    — Quản lý security plugin
│   ├── kibana_user         — Truy cập OpenSearch Dashboards
│   ├── readall             — Đọc tất cả indices
│   └── logstash            — Quyền để Logstash ghi dữ liệu
│
└── Custom Roles (Vai Trò Tùy Chỉnh)
    └── Bạn tự tạo với quyền cụ thể
```

### 4.3 Tạo Custom Role

```json
// Tạo role "app-read-only" — chỉ đọc index "app-logs-*"
PUT _plugins/_security/api/roles/app-read-only
{
  "cluster_permissions": ["cluster:monitor/health"],
  "index_permissions": [
    {
      "index_patterns": ["app-logs-*"],
      "allowed_actions": [
        "read",
        "indices:data/read/search",
        "indices:data/read/get"
      ]
    }
  ]
}
```

### 4.4 Role Mapping — Gán Vai Trò

Kết nối IAM roles/users với OpenSearch internal roles:

```json
// Gán IAM role "data-analyst-role" với OpenSearch role "app-read-only"
PUT _plugins/_security/api/rolesmapping/app-read-only
{
  "backend_roles": [
    "arn:aws:iam::123456789:role/data-analyst-role"
  ],
  "users": ["analyst1", "analyst2"]
}
```

### 4.5 Document-level Security — Bảo Mật Cấp Tài Liệu

Mỗi user chỉ thấy documents thỏa mãn điều kiện filter:

```json
// Role "north-region-viewer" — chỉ thấy đơn hàng region North
PUT _plugins/_security/api/roles/north-region-viewer
{
  "index_permissions": [
    {
      "index_patterns": ["orders"],
      "allowed_actions": ["read"],
      "dls": "{\"term\": {\"region\": \"North\"}}"
    }
  ]
}
```

```
User thuộc role "north-region-viewer" query:
    GET /orders/_search  → chỉ trả về documents có region="North"
    
Transparent với user — họ không biết filter đang được áp dụng
```

### 4.6 Field-level Security — Bảo Mật Cấp Trường

Ẩn hoặc chỉ hiển thị các field nhất định:

```json
// Role "orders-no-pii" — ẩn thông tin cá nhân (PII — Personally Identifiable Information)
PUT _plugins/_security/api/roles/orders-no-pii
{
  "index_permissions": [
    {
      "index_patterns": ["orders"],
      "allowed_actions": ["read"],
      "fls": [
        "~customer_name",
        "~customer_email",
        "~phone_number",
        "~credit_card"
      ]
    }
  ]
}
```

Dấu `~` nghĩa là **exclude** (Loại Trừ) field đó. Không có `~` nghĩa là chỉ cho phép field đó.

---

## 5. Encryption — Mã Hóa

### 5.1 Encryption in Transit — Mã Hóa Khi Truyền Tải

Tất cả communication đến/từ OpenSearch domain bắt buộc HTTPS (TLS 1.2 hoặc 1.3):

```
Client ──HTTPS──▶ OpenSearch Domain Endpoint
                  (TLS certificate tự động được AWS cấp và rotate)

Cấu hình:
└── TLS Security Policy: Policy-Min-TLS-1-2-2019-07 (khuyến nghị)
    (Hỗ trợ TLS 1.2 và 1.3, loại bỏ các cipher yếu)
```

### 5.2 Encryption at Rest — Mã Hóa Khi Lưu Trữ

Dữ liệu trên EBS (Hot) và S3 (UltraWarm/Cold) được mã hóa bằng AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa):

```
Kích hoạt khi tạo domain:
├── Encryption at rest: Enabled
├── KMS key:
│   ├── AWS managed key (aws/es) — Không mất phí thêm, ít kiểm soát
│   └── Customer managed key (CMK — Khóa Do Khách Hàng Quản Lý)
│       └── Bạn tạo và quản lý trong KMS — kiểm soát hoàn toàn

Lưu ý quan trọng:
- Chỉ bật khi tạo domain — KHÔNG thể bật sau
- Tất cả node trong domain dùng cùng một KMS key
```

### 5.3 Node-to-node Encryption — Mã Hóa Giữa Các Node

Mã hóa traffic giữa các data nodes trong cluster — quan trọng cho production:

```
Data Node 1 ──TLS──▶ Data Node 2  (replication traffic được mã hóa)
Master Node ──TLS──▶ Data Node    (cluster management traffic)

Bật khi tạo domain:
    Node-to-node encryption: Enabled
```

---

## 6. SAML Authentication — Xác Thực SAML

### 6.1 SAML Là Gì?

**SAML** (Security Assertion Markup Language — Ngôn Ngữ Đánh Dấu Xác Nhận Bảo Mật) cho phép Single Sign-On (SSO — Đăng Nhập Một Lần) từ Identity Provider (IdP — Nhà Cung Cấp Danh Tính) như Okta, Azure AD, Google Workspace vào OpenSearch Dashboards.

```
User Browser
    │
    │ 1. Truy cập OpenSearch Dashboards
    ▼
OpenSearch Dashboards
    │
    │ 2. Redirect đến IdP (chưa đăng nhập)
    ▼
Identity Provider (Okta / Azure AD / GSuite)
    │
    │ 3. User đăng nhập với tài khoản công ty
    ▼
OpenSearch Dashboards
    │ (IdP gửi SAML assertion xác nhận identity + roles)
    │ 4. Map SAML role → OpenSearch role
    ▼
User vào được Dashboard với đúng quyền
```

### 6.2 Cấu Hình SAML trong AWS Console

```
Domain → Security configuration → SAML authentication for OpenSearch Dashboards

Thông tin cần cấu hình:
├── IdP entity ID:        Lấy từ IdP metadata
├── IdP metadata URL:     URL metadata của IdP (hoặc upload XML)
├── SAML master backend role: ARN của admin group trong IdP
└── Subject key:          Trường trong SAML assertion chứa username (thường là "email")

Sau khi cấu hình, mỗi lần vào Dashboards:
    URL: https://my-domain.ap-southeast-1.es.amazonaws.com/_dashboards
    → Tự động redirect sang Okta/Azure → Đăng nhập → Về Dashboards
```

### 6.3 Amazon Cognito Integration

Thay thế cho SAML khi muốn dùng Cognito User Pools (Nhóm Người Dùng Cognito) làm IdP:

```
Amazon Cognito User Pool
    │ (Social login: Google, Facebook, hoặc tự tạo user)
    ▼
Cognito Identity Pool ──▶ IAM Role ──▶ OpenSearch Domain
    │
    └── OpenSearch Dashboards nhận token từ Cognito
        Map Cognito group → OpenSearch role
```

---

## 7. Audit Logging — Ghi Log Kiểm Tra

### 7.1 Audit Log Là Gì?

Audit log ghi lại mọi hành động trên OpenSearch — ai đã làm gì, khi nào, với index nào. Bắt buộc cho compliance như HIPAA, PCI-DSS, SOC 2.

```
Audit Log Categories:
├── FAILED_LOGIN         — Đăng nhập thất bại
├── MISSING_PRIVILEGES   — Truy cập bị từ chối do thiếu quyền
├── INDEX_EVENT          — Tạo/xóa index
├── DOCUMENT_READ        — Đọc document (cần bật riêng — nhiều log)
├── DOCUMENT_WRITE       — Thêm/sửa/xóa document
└── OPENSEARCH_SECURITY_INDEX_ATTEMPTED_DELETE — Cố xóa security index
```

### 7.2 Kích Hoạt và Cấu Hình

```
Domain → Logs → Audit logs → Enable

Gửi đến:
├── CloudWatch Logs — xem trực tiếp trong AWS Console
└── S3 → Athena → Phân tích dài hạn

Lưu ý về chi phí:
    DOCUMENT_READ log có thể tạo ra hàng triệu records/ngày
    → Chỉ bật khi thực sự cần cho compliance
    → Dùng filter để chỉ ghi log query trên sensitive indices
```

### 7.3 Ví Dụ Audit Log Entry

```json
{
  "audit_cluster_name":    "my-domain",
  "audit_node_name":       "node-1",
  "audit_category":        "FAILED_LOGIN",
  "audit_request_origin":  "REST",
  "audit_request_initiating_user": "unknown",
  "audit_request_remote_address":  "203.0.113.5",
  "@timestamp":            "2024-01-15T10:30:45.123Z",
  "audit_request_body":    "{\"password\":\"***\"}",
  "audit_format_version":  4
}
```

---

## 8. Best Practices — Thực Hành Tốt Nhất

### 8.1 Checklist Bảo Mật

```
Khi tạo domain mới:
[ ] Triển khai trong VPC (không dùng Public endpoint nếu production)
[ ] Bật Encryption at rest (KMS)
[ ] Bật Node-to-node encryption
[ ] Bật HTTPS (TLS 1.2+) — enforce HTTPS
[ ] Bật Fine-grained access control (FGAC)
[ ] Tạo dedicated master user thay vì dùng IAM admin
[ ] Cấu hình Domain access policy restrict theo IAM role (không dùng "Principal: *")

Sau khi tạo:
[ ] Bật Audit logging cho sensitive domains
[ ] Cấu hình SAML/Cognito thay vì internal users trong production
[ ] Không lưu credentials (thông tin xác thực) trong code — dùng IAM role
[ ] Rotate (Xoay Vòng) internal user passwords định kỳ
[ ] Monitor FAILED_LOGIN events qua CloudWatch Alarms
```

### 8.2 Least Privilege — Nguyên Tắc Đặc Quyền Tối Thiểu

```
Thay vì:
    User A: all_access role  ← tránh điều này

Nên:
    App Server Role: chỉ ghi/đọc index cụ thể nó cần
    Data Analyst Role: chỉ đọc, không ghi, ẩn PII fields
    Dashboard Viewer Role: chỉ đọc index "reporting-*"
    Admin Role: full access, chỉ dùng cho maintenance
```

### 8.3 Phân Tách Domain Theo Môi Trường

```
production-domain  ─ VPC riêng, FGAC bật, Audit log bật, CMK
staging-domain     ─ VPC riêng, FGAC bật, Audit log tùy chọn
development-domain ─ Có thể Public, FGAC bật, cost-optimized
```

### 8.4 Giám Sát Bảo Mật

```
CloudWatch Alarms nên thiết lập:
├── KibanaHealthyNodes < 1           → Alert: node bị lỗi
├── Số FAILED_LOGIN > 10/phút       → Alert: brute force attempt
├── ClusterStatus.red == 1           → Alert: cluster không healthy
└── FreeStorageSpace < 20%           → Alert: sắp hết dung lượng
```

---

## 9. Câu Hỏi Phỏng Vấn

### Q1: Fine-grained Access Control (FGAC) khác gì với Domain Access Policy?

**Trả lời:**
- **Domain Access Policy** (resource-based IAM policy): Kiểm soát ở **cấp AWS** — ai (IAM user/role/account) được phép kết nối đến domain và gọi các `es:ESHttp*` actions. Đây là lớp đầu tiên.
- **FGAC**: Kiểm soát bên **trong OpenSearch** sau khi đã xác thực — ai được đọc/ghi index nào, thấy field nào, document nào. FGAC yêu cầu bật khi tạo domain và không thể tắt sau đó.
- Cả hai đều cần thiết — Domain Access Policy là lớp ngoài, FGAC là lớp trong.

### Q2: Tại sao encryption at rest phải bật khi tạo domain và không bật được sau?

**Trả lời:** Encryption at rest mã hóa ở cấp EBS volume — khi volume được tạo, nó đã được mã hóa hoặc không. AWS không hỗ trợ mã hóa lại volume đang chạy in-place. Để "bật" encryption cho domain cũ, cần: tạo domain mới với encryption bật, reindex toàn bộ dữ liệu sang domain mới, sau đó đổi endpoint. Đây là lý do tại sao checklist tạo domain rất quan trọng.

### Q3: Giải thích document-level security (DLS) và field-level security (FLS) — khi nào cần dùng?

**Trả lời:**
- **DLS (Document-Level Security — Bảo Mật Cấp Tài Liệu)**: Mỗi user/role chỉ thấy documents thỏa mãn điều kiện filter. Ví dụ: sales manager Region A chỉ thấy data của Region A trong cùng một index `orders`. Cần dùng khi data đa tenant (Đa Khách Thuê) lưu trong cùng index.
- **FLS (Field-Level Security — Bảo Mật Cấp Trường)**: Ẩn các field nhạy cảm như `salary`, `ssn`, `credit_card` với user không được phép. Cần dùng để tuân thủ GDPR, HIPAA khi cho phép truy cập data nhưng ẩn PII.

### Q4: Làm thế nào để tích hợp Okta với OpenSearch Dashboards?

**Trả lời:** Dùng SAML authentication:
1. Trong Okta: tạo SAML application, lấy metadata URL và IdP entity ID
2. Trong OpenSearch: bật FGAC, cấu hình SAML với Okta metadata URL
3. Tạo role mapping: Okta group (backend_role) → OpenSearch role
4. Kết quả: User vào Dashboards → redirect Okta → đăng nhập SSO → OpenSearch role được gán tự động theo Okta group membership

### Q5: Audit logging có ảnh hưởng đến hiệu suất không? Cần lưu ý gì?

**Trả lời:** Có — đặc biệt khi bật `DOCUMENT_READ` log. Mỗi search query tạo ra nhiều log entries. Để giảm overhead:
1. Chỉ bật các category cần thiết (thường là FAILED_LOGIN + MISSING_PRIVILEGES + INDEX_EVENT là đủ)
2. Dùng `ignore_requests` để loại trừ các endpoint health check (`/_cat/health`)
3. Gửi log đến CloudWatch Logs với retention policy phù hợp (không giữ mãi)
4. Đối với compliance cần lưu lâu: ship sang S3 → Glacier với lifecycle policy

---

## 🔗 Liên Kết Tiếp Theo

- [OpenSearch Fundamentals — Nền Tảng](./1-opensearch-fundamentals.md)
- [Thu Nạp Dữ Liệu — Kinesis, Logstash, Fluent Bit](./2-opensearch-ingestion.md)
- [OpenSearch Module Overview](./README.md)
- [Module Tiếp Theo: MSK — Managed Kafka](../10-msk/README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-17
