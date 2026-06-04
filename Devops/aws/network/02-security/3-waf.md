# AWS WAF — Web Application Firewall (Tường Lửa Ứng Dụng Web)

> AWS WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web) bảo vệ ứng dụng web khỏi các tấn công Layer 7 (Tầng Ứng Dụng) phổ biến như SQL Injection (Tiêm SQL), XSS (Cross-Site Scripting — Kịch Bản Chéo Trang), và bot attacks (Tấn Công Bot). WAF hoạt động ở tầng HTTP/HTTPS — khác với Security Groups và NACL chỉ hoạt động ở Layer 3/4.

---

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#1-khái-niệm-cơ-bản)
2. [Kiến Trúc WAF](#2-kiến-trúc-waf)
3. [Web ACL — Danh Sách Kiểm Soát Truy Cập Web](#3-web-acl--danh-sách-kiểm-soát-truy-cập-web)
4. [Rule Types — Các Loại Quy Tắc](#4-rule-types--các-loại-quy-tắc)
5. [Managed Rule Groups — Nhóm Rules Quản Lý Sẵn](#5-managed-rule-groups--nhóm-rules-quản-lý-sẵn)
6. [Rate-Based Rules — Quy Tắc Giới Hạn Tốc Độ](#6-rate-based-rules--quy-tắc-giới-hạn-tốc-độ)
7. [Bot Control — Kiểm Soát Bot](#7-bot-control--kiểm-soát-bot)
8. [Logging & Monitoring — Ghi Log & Giám Sát](#8-logging--monitoring--ghi-log--giám-sát)
9. [WAF Pricing — Chi Phí](#9-waf-pricing--chi-phí)
10. [Best Practices — Thực Hành Tốt Nhất](#10-best-practices--thực-hành-tốt-nhất)
11. [Ví Dụ Thực Tế](#11-ví-dụ-thực-tế)
12. [Câu Hỏi Phỏng Vấn](#12-câu-hỏi-phỏng-vấn)

---

## 1. Khái Niệm Cơ Bản

### WAF Hoạt Động Ở Đâu?

```
Internet
    │
    ▼
┌──────────────────────────────────────────────┐
│            AWS CloudFront (CDN)              │  ◄── WAF có thể đặt ở đây
│         hoặc Application Load Balancer       │  ◄── hoặc đây
│         hoặc API Gateway                     │  ◄── hoặc đây
│         hoặc AppSync (GraphQL)               │  ◄── hoặc đây
└──────────────────────────────────────────────┘
    │   HTTP Request với headers, body, URI
    ▼
┌──────────────────────────────────────────────┐
│              AWS WAF                         │
│  ┌────────────────────────────────────────┐  │
│  │  Web ACL (Web Access Control List)     │  │
│  │    Rule 1: AWS Managed Rules (Core)    │  │
│  │    Rule 2: Rate limit 1000/5min        │  │
│  │    Rule 3: Block SQL Injection         │  │
│  │    Rule 4: IP Blocklist                │  │
│  │    Default: ALLOW                      │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
    │
    ▼
  Backend Application
```

### Tấn Công Mà WAF Bảo Vệ

| Tấn Công | Mô Tả | WAF Action |
|----------|-------|------------|
| **SQL Injection** | Tiêm câu lệnh SQL vào input | Block/Count |
| **XSS** (Cross-Site Scripting) | Tiêm script độc hại vào trang web | Block/Count |
| **CSRF** (Cross-Site Request Forgery) | Giả mạo request từ người dùng | Block/Count |
| **Path Traversal** | Duyệt thư mục trái phép `../../../etc/passwd` | Block |
| **Log4Shell** | Khai thác lỗ hổng Log4j | AWS Managed Rules |
| **SSRF** (Server-Side Request Forgery) | Server gọi URL độc hại | Block |
| **Bot attacks** | Automated scanning, credential stuffing | Bot Control |
| **DDoS Layer 7** | HTTP flood tới ứng dụng | Rate-based rules + Shield |

### Tích Hợp Với Dịch Vụ AWS

```
WAF có thể gắn vào (Associate with):
  ✅ Amazon CloudFront Distribution
  ✅ Application Load Balancer (ALB) — trong regional scope
  ✅ Amazon API Gateway (REST API)
  ✅ AWS AppSync (GraphQL API)
  ✅ Amazon Cognito User Pool
  ✅ AWS App Runner

KHÔNG hỗ trợ:
  ❌ Network Load Balancer (NLB) — Layer 4, không có HTTP context
  ❌ Classic Load Balancer
```

---

## 2. Kiến Trúc WAF

### Scope — Phạm Vi

| Scope | Dùng Cho | Vị Trí |
|-------|----------|--------|
| **CloudFront (Global)** | WAF gắn vào CloudFront | Region: us-east-1 (bắt buộc) |
| **Regional** | WAF gắn vào ALB, API Gateway, AppSync | Cùng region với resource |

**Lưu ý quan trọng:** Web ACL dùng cho CloudFront **phải được tạo tại us-east-1**, ngay cả khi resource ở region khác.

### Luồng Xử Lý Request

```
HTTP Request → WAF Web ACL
                  │
          Đánh giá rules theo priority (ưu tiên)
                  │
         ┌────────┴────────┐
         │                 │
       Match            No Match
         │                 │
    Rule Action      Tiếp tục rule tiếp theo
    ┌────┴────────────────┐
    │ ALLOW               │ BLOCK          │ COUNT          │ CAPTCHA
    │ Tiếp tục            │ Trả về 403     │ Đếm + tiếp tục │ Hiện CAPTCHA
    └─────────────────────┘
         │
    Nếu không rule nào match →  Default Action (ALLOW hoặc BLOCK)
```

---

## 3. Web ACL — Danh Sách Kiểm Soát Truy Cập Web

### Web ACL là gì?

Web ACL (Web Access Control List — Danh Sách Kiểm Soát Truy Cập Web) là container chứa các rules WAF. Một Web ACL có thể liên kết với nhiều resources.

```
Web ACL "prod-waf-acl"
│
├── Rule Group 1: AWSManagedRulesCommonRuleSet     (Priority 1)
├── Rule Group 2: AWSManagedRulesKnownBadInputsRuleSet (Priority 2)
├── Custom Rule: Rate limit per IP                 (Priority 3)
├── Custom Rule: Block specific countries          (Priority 4)
├── Custom Rule: Block known bad IPs               (Priority 5)
│
└── Default Action: ALLOW
```

### Capacity Units (WCUs — Đơn Vị Dung Lượng WAF)

Mỗi rule tiêu thụ một số WCU (WAF Capacity Units). Web ACL có giới hạn 5000 WCU.

| Rule Type | WCU Tiêu Thụ |
|-----------|-------------|
| Rule với 1 condition đơn giản | 1-2 WCU |
| Managed Rule Group (AWS) | 700-1500 WCU tùy nhóm |
| Rate-based rule | 2 WCU |
| Bot Control | 25 WCU |
| CAPTCHA challenge | 1 WCU thêm vào |

---

## 4. Rule Types — Các Loại Quy Tắc

### 4.1. Regular Rules (Quy Tắc Thông Thường)

Kiểm tra từng request dựa trên điều kiện cụ thể.

**Điều Kiện Có Thể Kiểm Tra:**

```
IP Address / IP Set
  → Block request từ danh sách IP cụ thể

Geographic Match (Địa Lý)
  → Block traffic từ một số quốc gia nhất định

String Match (Khớp Chuỗi)
  → Tìm pattern trong: URI, Header, Body, Query String, Method

Regex Match (Khớp Regex — Regular Expression)
  → Kiểm tra pattern phức tạp hơn bằng biểu thức chính quy

Size Constraint (Ràng Buộc Kích Thước)
  → Block request có body > 8KB (anti-DDoS)

SQL Injection Match
  → Phát hiện pattern SQL injection

XSS Match (Cross-Site Scripting)
  → Phát hiện script injection
```

### 4.2. Rule Groups (Nhóm Quy Tắc)

Tập hợp các rules, có thể reuse và share:

```
Rule Group "custom-security-rules":
  ├── Block SQLi in query strings
  ├── Block XSS in request body
  ├── Block bad user-agents
  └── Block known malicious IPs

Dùng lại Rule Group này cho nhiều Web ACLs khác nhau.
```

### 4.3. Managed Rule Groups (Nhóm Rules Quản Lý Sẵn)

AWS và đối tác quản lý — xem chi tiết ở phần 5.

### Actions (Hành Động) Cho Rules

| Action | Mô Tả |
|--------|-------|
| **ALLOW** | Cho phép request tiếp tục đến backend |
| **BLOCK** | Chặn request, trả về HTTP 403 |
| **COUNT** | Đếm request (không chặn) — dùng để test rules |
| **CAPTCHA** | Hiển thị CAPTCHA challenge cho browser |
| **CHALLENGE** | Silent challenge (không hiển thị) để phát hiện bots |

---

## 5. Managed Rule Groups — Nhóm Rules Quản Lý Sẵn

### AWS Managed Rule Groups (Miễn Phí Với Shield Advanced; Tính Phí Riêng Với WAF)

| Rule Group | Mô Tả | WCU |
|-----------|-------|-----|
| **AWSManagedRulesCommonRuleSet** | Core Rule Set — SQL Injection, XSS, path traversal, protocol anomalies | 700 |
| **AWSManagedRulesAdminProtectionRuleSet** | Bảo vệ admin pages | 100 |
| **AWSManagedRulesKnownBadInputsRuleSet** | Block input patterns known to be malicious | 200 |
| **AWSManagedRulesSQLiRuleSet** | SQL Injection chuyên biệt | 200 |
| **AWSManagedRulesLinuxRuleSet** | Bảo vệ cho Linux workloads | 200 |
| **AWSManagedRulesWindowsRuleSet** | Bảo vệ cho Windows workloads | 200 |
| **AWSManagedRulesPHPRuleSet** | Bảo vệ ứng dụng PHP | 100 |
| **AWSManagedRulesWordPressRuleSet** | Bảo vệ WordPress | 100 |
| **AWSManagedRulesAmazonIpReputationList** | Block IPs có reputation xấu (AWS intelligence) | 25 |
| **AWSManagedRulesAnonymousIpList** | Block VPNs, Tor, proxies | 50 |

### Marketplace Managed Rules (Trả Phí Thêm)

Các nhà cung cấp như Imperva, F5, Fortinet cung cấp rules chuyên biệt qua AWS Marketplace.

### Override Actions (Ghi Đè Hành Động)

Khi dùng Managed Rule Group, có thể override một số rules:

```
AWSManagedRulesCommonRuleSet:
  └── Rule "SizeRestrictions_BODY": Override COUNT (thay vì BLOCK)
      → Vì ứng dụng của bạn cần nhận body lớn hơn giới hạn mặc định
```

Dùng **COUNT mode** để test trước khi BLOCK:
```
Giai đoạn test: Tất cả rules → COUNT (ghi log nhưng không block)
Sau khi verify: Chuyển sang BLOCK
```

---

## 6. Rate-Based Rules — Quy Tắc Giới Hạn Tốc Độ

### Mục Đích

Chặn IP đang gửi quá nhiều requests trong khoảng thời gian ngắn — bảo vệ chống HTTP flood.

### Cách Cấu Hình

```
Rate-Based Rule:
  Limit: 1000 requests
  Time Window: 5 minutes (300 seconds)
  Aggregation Key: IP address
  Scope Down: (tùy chọn) chỉ áp dụng cho URI /login
  Action: BLOCK

→ Nếu IP nào gửi > 1000 requests trong 5 phút → BLOCK
```

### Aggregation Keys (Khóa Tổng Hợp)

| Key | Mô Tả |
|-----|-------|
| **IP** | Mỗi IP riêng biệt | 
| **Forwarded IP** | IP trong header X-Forwarded-For |
| **HTTP Method** | Groupby HTTP method |
| **Header** | Groupby một header cụ thể |
| **Query Argument** | Groupby một query parameter |
| **URI Path** | Groupby URI path |
| **Custom Keys** | Kết hợp nhiều fields |

### Ví Dụ Rate Limiting Theo URI

```
Rule 1: Rate limit /login endpoint
  Limit: 20 requests / 5 minutes per IP
  Scope: URI path starts with /login
  Action: BLOCK (trả về 429 Too Many Requests)

Rule 2: Rate limit /api endpoint  
  Limit: 500 requests / 1 minute per IP
  Scope: URI path starts with /api
  Action: BLOCK

Rule 3: Global rate limit
  Limit: 2000 requests / 5 minutes per IP
  Scope: All requests
  Action: BLOCK
```

---

## 7. Bot Control — Kiểm Soát Bot

### AWS WAF Bot Control

Tính năng nâng cao (cần enable riêng) phân loại và kiểm soát bot traffic:

```
Bot Categories:
  ✅ Verified Bots (Bot đã xác minh — cho phép):
      - Googlebot, Bingbot (search engine crawlers)
      - Amazon bots
      
  ⚠️  Unverified Bots (Bot chưa xác minh — cần xem xét):
      - Monitoring bots
      - Marketing bots
      
  ❌  Malicious Bots (Bot độc hại — chặn):
      - Credential stuffing bots
      - Content scraping bots
      - DDoS bots
```

### Targeted Bot Control (Kiểm Soát Bot Nâng Cao)

Dùng JavaScript challenges và CAPTCHA để phân biệt người dùng thật vs. bots:

```
Challenge Flow:
  1. Browser nhận JavaScript challenge
  2. Browser tính toán puzzle (vô hình với người dùng)
  3. WAF xác nhận giải pháp
  4. Nếu pass → cho phép request
  5. Nếu fail → CAPTCHA hoặc BLOCK
```

---

## 8. Logging & Monitoring — Ghi Log & Giám Sát

### WAF Logging

WAF có thể ghi log đến:
- **Amazon CloudWatch Logs** (gần real-time)
- **Amazon S3** (batch, phân tích lớn)
- **Amazon Kinesis Data Firehose** (streaming analytics)

### Thông Tin Trong Log

```json
{
  "timestamp": 1620345600000,
  "formatVersion": 1,
  "webaclId": "arn:aws:wafv2:us-east-1:123456789:global/webacl/prod-waf/...",
  "action": "BLOCK",
  "terminatingRuleId": "AWSManagedRulesCommonRuleSet",
  "terminatingRuleType": "MANAGED_RULE_GROUP",
  "httpRequest": {
    "clientIp": "203.0.113.5",
    "country": "CN",
    "headers": [{"name": "User-Agent", "value": "sqlmap/1.5"}],
    "uri": "/api/users?id=1' OR '1'='1",
    "args": "id=1' OR '1'='1",
    "httpMethod": "GET"
  },
  "ruleGroupList": [
    {
      "ruleGroupId": "AWSManagedRulesCommonRuleSet",
      "terminatingRule": {"ruleId": "SQLi_QUERYARGUMENTS", "action": "Block"}
    }
  ]
}
```

### CloudWatch Metrics (Số Liệu CloudWatch)

| Metric | Mô Tả |
|--------|-------|
| **AllowedRequests** | Số requests được cho phép |
| **BlockedRequests** | Số requests bị chặn |
| **CountedRequests** | Số requests bị count |
| **PassedRequests** | Requests đi qua (không match rule nào) |

---

## 9. WAF Pricing — Chi Phí

### Mô Hình Tính Phí

| Thành Phần | Giá (us-east-1) |
|-----------|----------------|
| Web ACL | $5/tháng |
| Rule (custom rule) | $1/tháng/rule |
| 1 triệu requests | $0.60 |
| Managed Rule Group | $1/tháng/group + $0.60/triệu req |
| Bot Control | Phí thêm (cần xem AWS pricing page) |

### Ước Tính Chi Phí Ví Dụ

```
Scenario: 10 triệu requests/tháng, 5 managed rule groups, 3 custom rules

Web ACL:               $5
Custom Rules (3):      $3
Managed Rule Groups (5): $5 × 5 = $25
Requests:              10M × $0.60/1M = $6

Tổng ước tính: ~$39/tháng
```

**Lưu ý:** WAF miễn phí nếu dùng Shield Advanced (included).

---

## 10. Best Practices — Thực Hành Tốt Nhất

### ✅ Nên Làm

1. **Bắt đầu với COUNT mode**, sau đó chuyển sang BLOCK khi đã verify không có false positives.

2. **Dùng AWS Managed Rules** làm baseline — đừng viết tất cả từ đầu.

3. **Rate limiting theo endpoint nhạy cảm:**
   ```
   /login, /register, /forgot-password → Rate limit thấp (anti-brute-force)
   /api → Rate limit vừa
   Global → Rate limit cao
   ```

4. **Bật WAF Logging** và gửi đến CloudWatch Logs để phân tích.

5. **Tạo Dashboard CloudWatch** theo dõi blocked requests trends.

6. **Kiểm tra false positives định kỳ** — Managed Rules đôi khi block requests hợp lệ.

7. **Kết hợp với Shield Advanced** cho bảo vệ DDoS toàn diện.

### ❌ Không Nên Làm

1. **Không bật BLOCK mode ngay** — test với COUNT trước 1-2 tuần.
2. **Không dùng WAF thay thế** cho secure coding — WAF là lớp bổ sung.
3. **Không bỏ qua rule conflicts** — rules có thể cancel nhau.
4. **Không để Web ACL không liên kết resource** — vẫn tính phí.

---

## 11. Ví Dụ Thực Tế

### Ví Dụ 1: WAF Cho E-commerce Application

```
Web ACL: "ecommerce-prod-waf"

Rules (theo thứ tự priority):
  Priority 1:  AWSManagedRulesAmazonIpReputationList    → BLOCK
  Priority 2:  AWSManagedRulesCommonRuleSet             → BLOCK
  Priority 3:  AWSManagedRulesSQLiRuleSet               → BLOCK
  Priority 4:  AWSManagedRulesKnownBadInputsRuleSet     → BLOCK
  Priority 5:  Rate limit /checkout: 50 req/5min/IP     → BLOCK
  Priority 6:  Rate limit /api/login: 10 req/5min/IP    → BLOCK
  Priority 7:  Block countries (tùy business)           → BLOCK
  Priority 8:  Bot Control (Targeted)                   → CHALLENGE
  Priority 9:  Global rate limit: 2000 req/5min/IP      → BLOCK

Default Action: ALLOW

Associate: CloudFront Distribution + ALB
```

### Ví Dụ 2: WAF Cho API Backend

```
Web ACL: "api-backend-waf"

Rules:
  Priority 1:  IP Allowlist (only known API partners)   → ALLOW (nếu match, skip qua rules khác)
  Priority 2:  AWSManagedRulesCommonRuleSet             → COUNT (bật COUNT trước để test)
  Priority 3:  Rate limit /api/v1: 1000 req/1min/IP     → BLOCK
  Priority 4:  Block oversized bodies > 10MB            → BLOCK

Default Action: ALLOW

Associate: API Gateway
```

### Ví Dụ 3: Tạo Custom Rule Bằng AWS CLI

```bash
# Tạo IP Set cho danh sách IP cần block
aws wafv2 create-ip-set \
  --name "BlockedIPs" \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses "198.51.100.0/24" "203.0.113.0/24"

# Tạo rule tham chiếu IP Set đó trong Web ACL
# (Thường làm qua Console hoặc CloudFormation/Terraform)
```

---

## 12. Câu Hỏi Phỏng Vấn

### Câu 1: WAF khác gì với Security Group?

**Trả lời:**
- **Security Group:** Hoạt động ở Layer 3/4 (IP, Port, Protocol) — kiểm soát TCP connections
- **WAF:** Hoạt động ở Layer 7 (HTTP/HTTPS) — kiểm tra nội dung request, headers, URI, body

WAF hiểu HTTP context — có thể phát hiện SQL injection trong query string, XSS trong body. Security Group chỉ thấy IP và port.

---

### Câu 2: Giải thích quy trình triển khai WAF an toàn cho production.

**Trả lời:**
```
Bước 1: Tạo Web ACL với tất cả rules ở COUNT mode
Bước 2: Associate với resource (ALB/CloudFront)
Bước 3: Monitor logs 1-2 tuần để phát hiện false positives
Bước 4: Tune rules — loại bỏ false positives bằng exception rules
Bước 5: Chuyển từng rule sang BLOCK mode (bắt đầu từ rule ít rủi ro nhất)
Bước 6: Monitor closely sau mỗi lần chuyển sang BLOCK
Bước 7: Set up alerts khi blocked requests tăng đột biến
```

---

### Câu 3: Rate limiting trong WAF dùng cho mục đích gì?

**Trả lời:**
Rate-based rules giới hạn số requests từ một IP trong khoảng thời gian, bảo vệ chống:
- **Brute force** trên /login — thử mật khẩu nhiều lần
- **Credential stuffing** — thử list username/password đánh cắp
- **HTTP flood DDoS** — gửi nhiều requests để làm quá tải server
- **API abuse** — gọi API quá giới hạn cho phép

Kết hợp với Shield Advanced cho bảo vệ DDoS đầy đủ.

---

### Câu 4: Khi nào dùng WAF vs Network Firewall?

**Trả lời:**
- **WAF:** Layer 7, HTTP/HTTPS context, ứng dụng web — SQL injection, XSS, bot protection
- **Network Firewall:** Layer 3-7, toàn bộ TCP/UDP traffic, deep packet inspection, IPS/IDS, chặn domain names, protocol anomaly detection

Dùng cả hai cho môi trường high-security: WAF bảo vệ ứng dụng web, Network Firewall bảo vệ toàn bộ mạng.

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Tiếp Theo |
|-------|----------|-----------|
| [2-network-acls.md](./2-network-acls.md) | **3-waf.md** | [4-shield.md](./4-shield.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
