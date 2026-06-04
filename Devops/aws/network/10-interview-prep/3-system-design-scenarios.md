# 🏗️ System Design Scenarios — Kịch Bản Thiết Kế Hệ Thống

> 5 kịch bản thiết kế kiến trúc mạng AWS thực tế thường gặp trong phỏng vấn Senior Engineer và Solutions Architect. Mỗi kịch bản có requirements, phân tích, diagram, và trade-offs.

---

## 📋 Cách Tiếp Cận System Design

Khi interviewer đưa ra kịch bản, áp dụng framework sau:

```
1. Clarify requirements (2-3 phút)
   → "Traffic volume bao nhiêu? Có multi-region không? Budget constraints?"
   → "RTO/RPO requirements là gì?"
   → "Compliance requirements không?"

2. Define success criteria
   → Availability: 99.9% hay 99.99%?
   → Latency: < 200ms p99 hay < 1 giây acceptable?
   → Throughput: 1,000 req/s hay 100,000 req/s?

3. Start with high-level, then drill down
   → Vẽ boxes lớn trước, rồi fill in details
   → Đừng bắt đầu ngay với chi tiết từng service

4. Discuss trade-offs proactively
   → Đừng đợi interviewer hỏi — chủ động nêu trade-offs

5. Consider: Cost, Security, Scalability, Reliability, Operability
```

---

## 🔴 Scenario 1: E-commerce Web Application — Multi-region HA

### Yêu Cầu

```
Business context: Ứng dụng bán hàng với 500K DAU (Daily Active Users)
Scale: 10,000 concurrent users peak, 100K/giờ vào đợt sale
Availability: 99.99% (< 52 phút downtime/năm)
Latency: < 300ms p99 cho API, < 2 giây page load toàn cầu
Regions: Users ở Southeast Asia, US, Europe
Compliance: PCI-DSS cho payment data
Data: Catalog (~10GB), Orders (transactional), Sessions, Cart
```

### Kiến Trúc

```
                    ┌─────────────────────────────────────────┐
                    │             Route 53                    │
                    │  Latency-based routing                  │
                    │  Health checks: 30s interval            │
                    └──────┬──────────┬──────────┬────────────┘
                           │          │           │
                    ┌──────▼──┐  ┌────▼────┐  ┌──▼──────┐
                    │ US-East │  │ EU-West │  │ AP-SE   │
                    │  (Primary│  │(Secondary│  │(Secondary│
                    │  Write) │  │ Read)   │  │ Read)   │
                    └────┬────┘  └────┬────┘  └────┬────┘
                         │            │              │
              ┌──────────▼────────────▼──────────────▼──────┐
              │                  CloudFront                   │
              │         Edge Locations (400+ locations)       │
              │    Cache static assets, API responses         │
              └────────────────────┬───────────────────────┘
                                   │
              ┌────────────────────▼───────────────────────┐
              │              Per-Region Stack               │
              │                                             │
              │  WAF + Shield Advanced                      │
              │         ↓                                   │
              │  ALB (HTTPS, TLS 1.3, ACM cert)            │
              │         ↓                                   │
              │  ECS Fargate (Stateless App Tier)           │
              │  Auto Scaling: min 3, max 50 tasks          │
              │         ↓                    ↓              │
              │  ElastiCache Redis      Aurora MySQL        │
              │  (Sessions, Cart)       Global Database     │
              │                         Primary → Replicas  │
              └────────────────────────────────────────────┘

Payment Service (isolated):
  ┌─────────────────────────────────────────────────────┐
  │  Isolated Subnet (CDE — Cardholder Data Environment) │
  │  ECS Service → Stripe/Payment Gateway               │
  │  → VPC Endpoint cho Secrets Manager (API keys)      │
  │  → NAT Gateway → Payment gateway (whitelist IPs)    │
  └─────────────────────────────────────────────────────┘
```

### Trade-offs

| Quyết Định                          | Lựa Chọn                          | Lý Do                                                                |
| ----------------------------------- | --------------------------------- | -------------------------------------------------------------------- |
| ALB vs NLB                          | ALB                               | Path-based routing cần thiết (/api, /static, /health)               |
| ECS Fargate vs EC2                  | Fargate                           | Không manage EC2, scale nhanh hơn, phù hợp stateless workloads      |
| Aurora vs RDS Multi-AZ              | Aurora Global Database            | Cross-region replication < 1 giây, tự động failover                  |
| ElastiCache Cluster vs Single node  | Cluster mode                      | HA cho sessions — mất session = re-login, ảnh hưởng UX              |
| CloudFront Price Class              | All Edge Locations                | Users toàn cầu — Performance > Cost với business scale này           |

### Điểm Quan Trọng Để Nói Trong Phỏng Vấn

1. **Stateless application tier**: Session trong ElastiCache, không trong local memory → horizontal scaling tự do
2. **Database writes**: Chỉ primary region viết, replicas chỉ đọc → eventual consistency acceptable cho product catalog, not for orders
3. **PCI-DSS isolation**: Payment processing phải trong isolated subnet, separate security domain
4. **Failover testing**: Thực tế test failover scenario — không chỉ document

---

## 🟡 Scenario 2: Hybrid Cloud Architecture — Kết Nối On-premises với AWS

### Yêu Cầu

```
Business context: Ngân hàng enterprise đang cloud migration
On-premises: 2 data centers ở Hà Nội và Hồ Chí Minh
AWS: Bắt đầu dùng cho analytics workloads, sẽ dần migrate
Connectivity: Cần throughput cao và latency nhất quán (financial data)
Security: Compliance yêu cầu private connection, không qua Internet
Redundancy: Không thể có single point of failure
```

### Kiến Trúc

```
On-premises Hà Nội DC                    On-premises HCM DC
        │                                        │
[Customer Gateway]                    [Customer Gateway]
        │                                        │
[Direct Connect 10Gbps]              [Direct Connect 10Gbps]
        │                                        │
[Direct Connect Location]            [Direct Connect Location]
(ví dụ: Telehouse HN)                (ví dụ: Telehouse HCM)
        │                                        │
        └──────────┬─────────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────────┐
    │           AWS Transit Gateway                │
    │           (ap-southeast-1)                  │
    │                                             │
    │  Route Tables:                              │
    │  - Prod RT: VPC-Prod ↔ On-prem             │
    │  - Nonprod RT: VPC-Dev (isolated)           │
    │  - Inspection RT: → Network Firewall VPC   │
    └──┬──────────┬──────────────┬───────────────┘
       │          │               │
  ┌────▼────┐ ┌───▼────┐   ┌─────▼────────────┐
  │ VPC-Prod│ │VPC-Dev │   │ Inspection VPC   │
  │         │ │        │   │ Network Firewall  │
  │ App tier│ │Dev envs│   │ East-West Traffic │
  │ DB tier │ │        │   │ Inspection        │
  └─────────┘ └────────┘   └──────────────────┘

Backup connectivity:
[HN DC] ──[VPN Site-to-Site]──→ [TGW]  (backup khi DX down)
[HCM DC] ──[VPN Site-to-Site]──→ [TGW] (backup khi DX down)
```

### Giải Thích Kỹ Thuật

**Tại sao 2 Direct Connect connections:**
- Redundancy: Nếu 1 DX circuit down → traffic failover sang DX còn lại
- Both DX → Transit Gateway Direct Connect Gateway → VPCs
- BGP (Border Gateway Protocol — Giao Thức Định Tuyến Biên Giới) AS Path prepending để control traffic preference

**Route 53 Resolver cho Hybrid DNS:**
```
On-premises resolve AWS hostnames:
  database.internal.aws → Route 53 Inbound Endpoint (10.0.1.4)
  → Route 53 Private Hosted Zone → 10.0.3.45

AWS resolve on-premises hostnames:
  ldap.internal.company.com → Route 53 Outbound Endpoint
  → Forwarding rule → On-premises DNS server (192.168.1.10)
```

**VPN backup configuration:**
```
Primary: Direct Connect (BGP lower AS path = preferred)
Backup: VPN Site-to-Site (higher AS path = backup)
→ Automatic failover khi DX down (BGP withdraws routes)
→ Traffic seamlessly moves to VPN tunnel
```

### Trade-offs

| Quyết Định                           | Option Chosen                      | Alternative                         | Lý Do                              |
| ------------------------------------ | ---------------------------------- | ----------------------------------- | ---------------------------------- |
| DX speed                             | 10 Gbps per connection             | 1 Gbps                              | Financial data volume + growth     |
| Redundancy strategy                  | 2 DX + VPN backup                  | 2 DX only                           | VPN là insurance với cost thấp     |
| Transit Gateway vs direct DX to VPC  | Transit Gateway                    | DX thẳng từng VPC                   | Centralized routing, easier management |
| Network Firewall placement           | Inspection VPC                     | Each VPC                            | Centralize inspection, cost efficient |

---

## 🟢 Scenario 3: Serverless API Platform — Thiết Kế Cho Serverless

### Yêu Cầu

```
Business context: Startup xây dựng API platform cho mobile app
Scale: 0-100K requests/phút (unpredictable spike)
Tech stack: Lambda, API Gateway, DynamoDB
Budget: Tối thiểu hóa idle cost
Security: Không expose VPC resources trực tiếp
Multi-tenant: Nhiều clients, cần isolation
```

### Kiến Trúc

```
Mobile App / Web Client
        │
[CloudFront Distribution]
│  Cache: GET requests với Cache-Control headers
│  WAF: Rate limiting, SQL injection, XSS protection
│  Price Class: 200 (US + EU + Asia)
        │
[API Gateway (Regional)]
│  Throttling: 10,000 req/s default, custom per API key
│  Usage Plans: Free/Pro/Enterprise tiers
│  Lambda Authorizer (JWT validation)
        │
[Lambda Functions] ─── [VPC Configuration]
│  Timeout: 30s                │
│  Memory: 512MB-3GB           │  Lambda VPC ENIs → Private Subnet
│  Concurrency: Reserved + Provisioned   │  → Security Group (SG-Lambda)
│  ARM64 (Graviton) → 20% cost reduction │
        │                       │
[DynamoDB (Global Table)]    [RDS in VPC]
│  On-demand capacity          │  (nếu cần relational)
│  DAX (DynamoDB Accelerator)  │  → SG-RDS allow from SG-Lambda only
│  for read-heavy workloads    │
        │
[Secrets Manager] ← Lambda dùng qua Interface Endpoint
[SQS + SNS]       ← Event-driven patterns
[S3]              ← Storage qua Gateway Endpoint (free)
```

### Cấu Hình Lambda Networking

```
Lambda trong VPC vs ngoài VPC:

Ngoài VPC (default):
✅ Truy cập Internet trực tiếp
✅ Không cần NAT Gateway
✅ Cold start nhanh hơn
❌ Không access private resources (RDS, ElastiCache trong VPC)

Trong VPC:
✅ Access private resources
✅ Traffic không qua Internet (dùng VPC Endpoints)
⚠️  Trước đây cold start chậm — giờ AWS cải thiện với Hyperplane ENIs
⚠️  Cần đủ IP addresses trong subnet (mỗi concurrent execution = 1 ENI)
```

**Tránh NAT Gateway cost cho Lambda:**
```
Thay vì: Lambda → NAT Gateway → S3 (tốn tiền)
Dùng:    Lambda → S3 Gateway Endpoint → S3 (miễn phí)

Thay vì: Lambda → NAT Gateway → DynamoDB (tốn tiền)  
Dùng:    Lambda → DynamoDB Gateway Endpoint → DynamoDB (miễn phí)

Thay vì: Lambda → NAT Gateway → Secrets Manager (tốn tiền)
Dùng:    Lambda → Interface Endpoint → Secrets Manager (~$7/tháng nhưng tiết kiệm data cost)
```

### Trade-offs Quan Trọng

**Lambda cold start với VPC:**
- Provisioned concurrency: Giảm cold start → tốn tiền ngay cả khi không có traffic
- Reserved concurrency: Đảm bảo capacity nhưng giới hạn max scale
- Solution: Provisioned concurrency chỉ cho critical paths (checkout, auth), on-demand cho rest

**API Gateway vs ALB cho Lambda:**
```
API Gateway:
✅ Built-in auth, throttling, usage plans, API keys
✅ Better monitoring và tracing với X-Ray
❌ Đắt hơn ($3.50/million requests vs $0.008/LCU-hour cho ALB)

ALB với Lambda:
✅ Rẻ hơn cho high-volume simple APIs
❌ Phải tự implement auth, rate limiting
```

---

## 🔵 Scenario 4: Global Content Distribution — Media Streaming Platform

### Yêu Cầu

```
Business context: Nền tảng streaming video (tương tự Netflix/YouTube nhỏ)
Content: Video-on-demand (VOD), không phải live streaming
Scale: 1 triệu users/ngày, peak 100K concurrent streams
Content size: Video files 1-10GB, thumbnails, metadata
Regions: Toàn cầu, concentrate Southeast Asia và APAC
Goal: Buffer-free streaming, giảm bandwidth cost
Security: Prevent hotlinking, DRM (Digital Rights Management — Quản Lý Quyền Kỹ Thuật Số) basic
```

### Kiến Trúc

```
Content Upload & Processing:
S3 (raw upload) → MediaConvert → S3 (processed: 360p, 720p, 1080p)
                                      ↓
                              S3 Intelligent-Tiering
                              (tự động move sang Glacier nếu không access)

Content Delivery:
                    ┌──────────────────────────────────────┐
                    │           CloudFront Distribution    │
                    │  - OAC protect S3 bucket             │
                    │  - Price Class: All Edge Locations   │
                    │  - Streaming protocol: HLS/DASH      │
                    │  - Signed URLs/Cookies (auth)        │
                    │  - Geo-restriction nếu content rights│
                    └──────────────────────────────────────┘
                                    │
                            S3 Origin (Processed Video)
                            │
                    ┌───────▼──────────────────────────────┐
                    │         API Backend (Singapore)      │
                    │                                      │
                    │  ALB → Lambda/ECS                    │
                    │  → DynamoDB (user data, watch history│
                    │  → RDS Aurora (billing, subscriptions│
                    │  → ElastiCache (metadata caching)    │
                    └──────────────────────────────────────┘

Route 53: Latency-based → API clusters per region (SG, US, EU)
```

### CloudFront Configuration Chi Tiết

```
Cache Behaviors (Hành Vi Cache) theo path pattern:

/video/*          TTL: 1 năm (immutable video files)
/thumbnail/*      TTL: 7 ngày (có thể update)
/api/*            TTL: 0 (no cache — dynamic content)
/manifest/*       TTL: 5 phút (HLS manifest — cần fresh)

Cache Keys:
- Video files: chỉ path (không include query strings)
- API: path + Authorization header (per-user)

Signed URLs cho video content:
→ Lambda@Edge tại Viewer Request:
   1. Validate JWT token từ Authorization header
   2. Check user subscription active
   3. Generate signed URL với expiration 1 giờ
   4. Redirect client đến signed URL
```

### Tối Ưu Chi Phí

```
S3 Storage:
- Videos cũ (>90 ngày không xem) → Glacier Instant Retrieval
- Videos rất cũ (>1 năm) → Glacier Deep Archive
- Dùng S3 Intelligent-Tiering → tự động

CloudFront:
- Bật Brotli/Gzip compression cho manifest files (giảm 60-80%)
- Cache thumbnails và metadata aggressively
- Origin Shield (Tấm Khiên Nguồn Gốc): Thêm 1 regional cache → giảm origin requests thêm 50-70%

Bandwidth:
- Adaptive bitrate streaming: Client tự chọn quality dựa trên bandwidth
- Start với 360p, nâng lên 1080p khi network ổn định
```

---

## 🟣 Scenario 5: Zero-Trust Network Architecture

### Yêu Cầu

```
Business context: Công ty fintech đang re-architect sau security breach
Current state: Flat network — mọi services trong cùng VPC communicate tự do
Goal: Zero-trust model — mọi communication phải được authenticate + authorize
Scale: 50 microservices, 20 teams
Compliance: SOC 2 Type II
```

### Zero-Trust Principles Áp Dụng Vào AWS Networking

```
Nguyên tắc Zero-Trust:
"Never trust, always verify" — Không tin tưởng bất cứ thứ gì, luôn verify

AWS Implementation:
1. Network segmentation (không flat network)
2. Service-to-service mTLS (mutual TLS — TLS Hai Chiều)
3. IAM authorization cho mọi API call
4. Short-lived credentials, không hard-coded
5. Continuous monitoring và anomaly detection
```

### Kiến Trúc

```
VPC Structure:
┌─────────────────────────────────────────────────────────────────┐
│                         VPC (10.0.0.0/8)                        │
│                                                                  │
│  Shared Egress VPC          Inspection VPC                      │
│  ├── NAT Gateways           ├── AWS Network Firewall            │
│  └── Centralized egress     └── All north-south traffic         │
│                                                                  │
│  Payment Domain VPC         Commerce Domain VPC                 │
│  ├── Payment Service        ├── Order Service                   │
│  ├── Fraud Detection        ├── Inventory Service               │
│  └── Isolated subnet        └── User Service                    │
│                                                                  │
│  Data Platform VPC          Identity VPC                        │
│  ├── Analytics              ├── Auth Service                    │
│  ├── Data Pipeline          └── API Gateway                     │
│  └── ML Training                                                 │
│                                                                  │
│  ├─────── Transit Gateway ────────────────────────────────┤     │
│           (Centralized routing + firewall inspection)            │
└─────────────────────────────────────────────────────────────────┘

Service Mesh Layer (trên top của network):
App Mesh / AWS PrivateLink:
Order Service → mTLS (TLS Hai Chiều) → Payment Service
→ Mutual certificate authentication
→ Authorization header with JWT
→ Logged in CloudTrail
```

### Security Controls Theo Layer

**Layer 1 — Network (Mạng):**
```
Security Groups: Whitelist only — chỉ allow specific source SG → destination SG
NACLs: Subnet-level baseline protection
Transit Gateway Route Tables: Segment domains (Payment không reach Analytics)
Network Firewall: Deep packet inspection, IDS/IPS rules
```

**Layer 2 — Identity & Access (Định Danh & Truy Cập):**
```
Service-to-service: IAM Roles + Resource-based policies
"Confused Deputy" prevention: aws:SourceAccount conditions
No long-term credentials: STS AssumeRole, auto-rotate
IRSA (IAM Roles for Service Accounts — IAM Role cho Tài Khoản Dịch Vụ): Cho EKS workloads
```

**Layer 3 — Application (Ứng Dụng):**
```
API Gateway: Request validation, throttling, auth
mTLS: Service mesh với certificate-based auth
JWT validation tại Lambda Authorizer
```

**Layer 4 — Monitoring & Response (Giám Sát & Phản Ứng):**
```
VPC Flow Logs → CloudWatch → GuardDuty phát hiện anomalies
CloudTrail: Log mọi API call
Security Hub: Tổng hợp findings từ GuardDuty, Config, Inspector
EventBridge → Lambda auto-remediation (ví dụ: isolate compromised instance)
```

### Câu Hỏi Trade-offs Nên Chủ Động Nêu

1. **Complexity vs Security**: Zero-trust tăng operational complexity đáng kể — cần mature DevOps culture
2. **Latency overhead**: mTLS adds ~1-2ms per hop — acceptable cho most use cases
3. **Cost**: Network Firewall, VPC Endpoints, Transit Gateway có chi phí — quantify ROI từ security improvement
4. **Migration path**: Không thể migrate overnight — cần phased approach (cách tiếp cận theo giai đoạn)

---

## 📐 Cheat Sheet — Symbols Cho Vẽ Sơ Đồ

```
Khi vẽ trên whiteboard, dùng ký hiệu đơn giản:

□  EC2 Instance / Container
◯  Database (RDS, DynamoDB)
⬡  S3 Bucket
▽  Load Balancer (ALB/NLB)
◇  Lambda Function
══  Direct Connect / Dedicated Line
---  VPN / Encrypted Tunnel
→   Traffic direction
↔   Bidirectional communication
[SG] Security Group boundary
[VPC] VPC boundary
[AZ] Availability Zone boundary
```

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
