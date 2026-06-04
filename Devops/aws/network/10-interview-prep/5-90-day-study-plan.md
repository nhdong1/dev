# 📅 Kế Hoạch Học 90 Ngày — AWS Networking Mastery

> Lộ trình học có cấu trúc giúp bạn đi từ beginner (người mới) đến confident AWS Networking engineer trong 90 ngày. Mỗi tuần có mục tiêu rõ ràng, tài liệu cụ thể, và deliverables (sản phẩm đầu ra) đo lường được.

---

## 🎯 Mục Tiêu Cuối 90 Ngày

Sau 90 ngày, bạn có thể:

- ✅ Thiết kế VPC production-ready từ đầu không cần tham khảo
- ✅ Giải thích bất kỳ AWS Networking service nào với trade-offs
- ✅ Debug network connectivity issues một cách có hệ thống
- ✅ Trả lời top 20 interview questions tự tin
- ✅ Vẽ multi-region architecture trên whiteboard trong 20 phút
- ✅ Có 3 incident stories sẵn sàng theo STAR framework
- ✅ Có hands-on experience với tất cả core networking services

---

## 📊 Overview Timeline

```
Tháng 1 (Ngày 1-30):   Nền Tảng Vững Chắc
├── Tuần 1: VPC, Subnets, IGW, NAT
├── Tuần 2: Security Groups, NACLs, WAF
├── Tuần 3: Load Balancing (ALB, NLB)
└── Tuần 4: Route 53 & DNS

Tháng 2 (Ngày 31-60):  Kỹ Năng Cốt Lõi
├── Tuần 5: CloudFront CDN
├── Tuần 6: Connectivity (VPN, Direct Connect)
├── Tuần 7: Advanced Networking (TGW, PrivateLink)
└── Tuần 8: Monitoring & Troubleshooting

Tháng 3 (Ngày 61-90):  Chuyên Sâu & Phỏng Vấn
├── Tuần 9: Lab Project tổng hợp
├── Tuần 10: System Design luyện tập
├── Tuần 11: Interview preparation
└── Tuần 12: Mock interviews & refinement
```

---

## 📅 THÁNG 1: Nền Tảng — Foundation (Ngày 1-30)

### Tuần 1: VPC Fundamentals (Ngày 1-7)

**Mục tiêu:** Hiểu sâu kiến trúc VPC và thiết kế subnet

| Ngày | Nội Dung                                           | Tài Liệu                                    | Action Item                               |
| ---- | -------------------------------------------------- | ------------------------------------------- | ----------------------------------------- |
| 1    | VPC concepts, CIDR notation, IP addressing         | `01-vpc-fundamentals/1-vpc-architecture.md` | Vẽ sơ đồ VPC với 3 tiers từ bộ nhớ       |
| 2    | Subnets — Public/Private/Isolated                  | `01-vpc-fundamentals/2-subnets.md`          | Lab 1 bước 1-2: Tạo VPC + Subnets        |
| 3    | Route Tables, Internet Gateway                     | `01-vpc-fundamentals/3-route-tables.md`     | Lab 1 bước 3: Route Tables               |
| 4    | NAT Gateway vs NAT Instance                        | `01-vpc-fundamentals/4-nat-gateway.md`      | Lab 1 bước 4-5: NAT GW + verify          |
| 5    | VPC Peering, Resource Sharing                      | `01-vpc-fundamentals/5-vpc-peering.md`      | Đọc và vẽ sơ đồ peering topology         |
| 6    | Review + Practice questions                        | `10-interview-prep/1-INTERVIEW_GUIDE.md` Q1-4 | Trả lời Câu 1-4 không nhìn notes       |
| 7    | Rest + Consolidation                               | Ôn lại toàn tuần                            | Viết summary 1 trang về VPC design       |

**Deliverables tuần 1:**
- [ ] Lab 1 hoàn thành (VPC 3-tier created từ CLI)
- [ ] Vẽ VPC diagram từ bộ nhớ không sai
- [ ] Trả lời được Câu 1-4 trong Interview Guide

---

### Tuần 2: Security — Bảo Mật Mạng (Ngày 8-14)

**Mục tiêu:** Master Security Groups, NACLs, WAF, Shield

| Ngày | Nội Dung                                | Tài Liệu                                    | Action Item                                 |
| ---- | --------------------------------------- | ------------------------------------------- | ------------------------------------------- |
| 8    | Security Groups — stateful, rules       | `02-security/1-security-groups.md`          | Lab 2: Tạo SGs và test stateful behavior    |
| 9    | Network ACLs — stateless, evaluation    | `02-security/2-network-acls.md`             | Lab 2: Reproduce NACL ephemeral port issue  |
| 10   | WAF — rules, managed rules, rate limit  | `02-security/3-waf.md`                      | Tạo WAF WebACL với AWSManagedRules          |
| 11   | Shield Standard & Advanced, DDoS        | `02-security/4-shield.md`                   | Đọc case study DDoS attack                  |
| 12   | AWS Network Firewall, inspection        | `02-security/5-network-firewall.md`         | Thiết kế security architecture trên giấy   |
| 13   | Review Câu 5-7 trong Interview Guide    | `10-interview-prep/1-INTERVIEW_GUIDE.md`    | Practice STAR story về security incident    |
| 14   | Rest + Consolidation                    |                                             | Chuẩn bị Security STAR story               |

**Deliverables tuần 2:**
- [ ] Có thể giải thích stateful vs stateless không sai
- [ ] Biết khi nào dùng WAF, khi nào dùng Security Group
- [ ] Draft STAR story về security incident (dù fictitious)

---

### Tuần 3: Load Balancing (Ngày 15-21)

**Mục tiêu:** Thành thạo ALB, NLB, Target Groups, Health Checks

| Ngày | Nội Dung                                      | Tài Liệu                                    | Action Item                              |
| ---- | --------------------------------------------- | ------------------------------------------- | ---------------------------------------- |
| 15   | ALB — Layer 7, routing rules, headers         | `03-load-balancing/1-alb.md`                | Lab 3: Tạo ALB với path-based routing    |
| 16   | NLB — Layer 4, static IP, use cases           | `03-load-balancing/2-nlb.md`                | So sánh ALB vs NLB trên giấy            |
| 17   | Target Groups, Health Checks                  | `03-load-balancing/3-target-groups.md`      | Test health check states (healthy/unhealthy) |
| 18   | SSL/TLS Termination với ACM                   | `03-load-balancing/4-ssl-tls.md`            | Request ACM certificate và attach to ALB |
| 19   | Advanced: Cross-zone LB, sticky sessions      | `03-load-balancing/5-advanced-patterns.md`  | Enable/disable cross-zone, observe behavior |
| 20   | Review Câu 8-10 trong Interview Guide         | `10-interview-prep/1-INTERVIEW_GUIDE.md`    | Tự test: giải thích ALB vs NLB không notes |
| 21   | Rest + Lab review                             |                                             | Verify Lab 3 hoàn thành                 |

**Deliverables tuần 3:**
- [ ] ALB với path-based routing hoạt động
- [ ] Giải thích cross-zone load balancing bằng diagram
- [ ] Biết khi nào nên dùng NLB thay vì ALB

---

### Tuần 4: Route 53 & DNS (Ngày 22-30)

**Mục tiêu:** Nắm vững tất cả 7 routing policies và DNS concepts

| Ngày | Nội Dung                                    | Tài Liệu                                    | Action Item                                    |
| ---- | ------------------------------------------- | ------------------------------------------- | ---------------------------------------------- |
| 22   | DNS fundamentals, record types              | `04-dns-route53/1-dns-fundamentals.md`      | Query DNS với dig/nslookup cho example.com      |
| 23   | Hosted Zones — Public vs Private            | `04-dns-route53/2-hosted-zones.md`          | Tạo Private Hosted Zone, test trong VPC         |
| 24   | Routing Policies (Simple, Weighted)         | `04-dns-route53/3-routing-policies.md`      | Setup Weighted routing: 80/20 split             |
| 25   | Routing Policies (Latency, Failover, Geo)   | `04-dns-route53/3-routing-policies.md`      | Setup Failover routing với health checks        |
| 26   | Health Checks & DNS Failover Automation     | `04-dns-route53/4-health-checks.md`         | Test automatic failover khi origin goes down    |
| 27   | Route 53 Resolver, Hybrid DNS               | `04-dns-route53/5-resolver.md`              | Đọc và diagram hybrid DNS architecture          |
| 28   | Review Câu 11-13 trong Interview Guide      | `10-interview-prep/1-INTERVIEW_GUIDE.md`    | Giải thích CNAME vs Alias cho người không biết  |
| 29   | Tháng 1 Review — VPC + Security + LB + DNS |                                             | Mock quiz: 15 câu hỏi tự test                  |
| 30   | Rest + Reflection                           |                                             | Viết reflection: Tháng 1 học được gì?         |

**Deliverables tháng 1:**
- [ ] Có thể vẽ 3-tier VPC + ALB + Route 53 không sai
- [ ] Giải thích 7 routing policies với use case
- [ ] Chuẩn bị ít nhất 1 STAR story

---

## 📅 THÁNG 2: Kỹ Năng Cốt Lõi — Core Skills (Ngày 31-60)

### Tuần 5: CloudFront CDN (Ngày 31-37)

**Mục tiêu:** Master CloudFront architecture, caching, security

| Ngày | Nội Dung                                     | Tài Liệu                                          | Action Item                                      |
| ---- | -------------------------------------------- | ------------------------------------------------- | ------------------------------------------------ |
| 31   | CloudFront architecture, edge locations      | `05-cdn-cloudfront/1-cloudfront-basics.md`        | Tạo CloudFront distribution với ALB origin       |
| 32   | Cache behaviors, TTL, Cache Policies         | `05-cdn-cloudfront/2-cache-behaviors.md`          | Lab 4: CloudFront + S3 với OAC                   |
| 33   | OAC, OAI, S3 bucket protection               | `05-cdn-cloudfront/3-origin-access.md`            | Verify S3 direct access → 403 Forbidden          |
| 34   | Lambda@Edge vs CloudFront Functions          | `05-cdn-cloudfront/4-lambda-edge.md`              | Deploy simple CF Function (add security headers) |
| 35   | Price Classes, compression, optimization     | `05-cdn-cloudfront/5-performance-cost.md`         | Đo cache hit ratio trước và sau optimization     |
| 36   | Review + Câu hỏi interview CloudFront        | `10-interview-prep/1-INTERVIEW_GUIDE.md` Q 11-13 | Giải thích OAC vs OAI scenario                  |
| 37   | Rest + Lab review                            |                                                   | Lab 4 checklist verification                    |

---

### Tuần 6: Connectivity Hybrid (Ngày 38-44)

**Mục tiêu:** Hiểu sâu VPN, Direct Connect, Transit Gateway

| Ngày | Nội Dung                                     | Tài Liệu                                      | Action Item                               |
| ---- | -------------------------------------------- | --------------------------------------------- | ----------------------------------------- |
| 38   | Site-to-Site VPN — concepts, components      | `06-connectivity/1-site-to-site-vpn.md`       | Diagram VPN architecture từ bộ nhớ        |
| 39   | Client VPN — remote access, auth             | `06-connectivity/2-client-vpn.md`             | Setup Client VPN (optional — có phí nhỏ)  |
| 40   | Direct Connect — concepts, VIF types, HA     | `06-connectivity/3-direct-connect.md`         | So sánh DX vs VPN — bảng trade-offs        |
| 41   | Direct Connect Gateway — multi-VPC, region   | `06-connectivity/4-direct-connect-gateway.md` | Diagram DX Gateway + TGW architecture     |
| 42   | Transit Gateway — hub-and-spoke              | `06-connectivity/5-transit-gateway.md`        | Thiết kế TGW route domains cho 3 VPCs     |
| 43   | Review Câu 14-16 trong Interview Guide       | `10-interview-prep/1-INTERVIEW_GUIDE.md`      | Giải thích TGW vs VPC Peering trade-offs  |
| 44   | Rest                                         |                                               | STAR story về connectivity project        |

---

### Tuần 7: Advanced Networking (Ngày 45-51)

**Mục tiêu:** VPC Endpoints, PrivateLink, Global Accelerator

| Ngày | Nội Dung                                     | Tài Liệu                                              | Action Item                               |
| ---- | -------------------------------------------- | ----------------------------------------------------- | ----------------------------------------- |
| 45   | VPC Endpoints — Gateway vs Interface         | `07-advanced-networking/1-vpc-endpoints.md`           | Tạo S3 Gateway Endpoint, verify routing   |
| 46   | AWS PrivateLink — endpoint services          | `07-advanced-networking/2-privatelink.md`             | Thiết kế PrivateLink cho internal service |
| 47   | Global Accelerator — Anycast, use cases      | `07-advanced-networking/3-global-accelerator.md`      | So sánh GA vs CloudFront (khi nào dùng)   |
| 48   | Elastic IP, ENI, secondary IPs               | `07-advanced-networking/4-elastic-ip-eni.md`          | Allocate EIP, associate/disassociate       |
| 49   | IPv6 trong VPC — dual-stack                  | `07-advanced-networking/5-ipv6.md`                    | Enable IPv6 trên VPC (không tốn phí)      |
| 50   | Review Câu 16 + cost optimization            | `10-interview-prep/1-INTERVIEW_GUIDE.md`              | Tính toán tiết kiệm chi phí với VPC Endpoints |
| 51   | Rest                                         |                                                       | Draft STAR story về cost optimization     |

---

### Tuần 8: Monitoring & Troubleshooting (Ngày 52-60)

**Mục tiêu:** Debug network issues một cách chuyên nghiệp

| Ngày | Nội Dung                                       | Tài Liệu                                               | Action Item                                 |
| ---- | ---------------------------------------------- | ------------------------------------------------------ | ------------------------------------------- |
| 52   | VPC Flow Logs — setup, format, analysis        | `08-monitoring/1-vpc-flow-logs.md`                     | Lab 5: Enable flow logs + Athena queries    |
| 53   | CloudWatch metrics cho networking              | `08-monitoring/2-cloudwatch-networking.md`             | Tạo CloudWatch alarm cho ALB latency        |
| 54   | Network Access Analyzer                        | `08-monitoring/3-network-access-analyzer.md`           | Chạy Network Access Analyzer trên VPC       |
| 55   | Reachability Analyzer                          | `08-monitoring/4-reachability-analyzer.md`             | Test EC2 → RDS path với Reachability Analyzer |
| 56   | Connectivity Debug methodology                 | `09-troubleshooting/1-connectivity-debug.md`           | Simulate connectivity issue + debug từng bước |
| 57   | Security Group & NACL debug                    | `09-troubleshooting/2-security-group-nacl-debug.md`    | Intentional break + fix SG/NACL             |
| 58   | ALB/NLB issues, DNS issues                     | `09-troubleshooting/4-load-balancer-issues.md`         | Practice debug scenarios                    |
| 59   | Production checklist review                    | `09-troubleshooting/6-production-checklist.md`         | Go through checklist cho VPC đã tạo         |
| 60   | Tháng 2 Review                                 |                                                        | Mock quiz: 20 câu hỏi mixed topics          |

**Deliverables tháng 2:**
- [ ] Lab 4 + Lab 5 hoàn thành
- [ ] Có thể debug network issues trong 5-10 phút có hệ thống
- [ ] Có 2 STAR stories chuẩn bị tốt

---

## 📅 THÁNG 3: Chuyên Sâu & Phỏng Vấn (Ngày 61-90)

### Tuần 9: Lab Project Tổng Hợp (Ngày 61-67)

**Mục tiêu:** Xây dựng hoàn chỉnh một production-ready AWS network

**Project: E-commerce Network Stack**

```
Requirements:
- VPC với 3-tier architecture (6 subnets, 2 AZs)
- ALB với HTTPS, path-based routing
- CloudFront distribution với S3 + ALB origins
- Route 53 với failover routing
- WAF với rate limiting
- VPC Flow Logs với Athena setup
- S3 Gateway Endpoint (tiết kiệm NAT cost)
- Everything as code với Terraform (tùy chọn)
```

| Ngày | Công Việc                                              | Ghi Chú                                  |
| ---- | ------------------------------------------------------ | ---------------------------------------- |
| 61   | VPC + Subnets + IGW + NAT + SGs                        | Foundation layer                         |
| 62   | ALB + Target Groups + Health Checks + ACM              | Traffic distribution layer               |
| 63   | CloudFront + OAC + S3                                  | CDN layer                                |
| 64   | Route 53 + Failover routing                            | DNS layer                                |
| 65   | WAF WebACL + Rate limiting + Managed rules             | Security layer                           |
| 66   | VPC Flow Logs + Athena + CloudWatch alarms             | Observability layer                      |
| 67   | Documentation + Architecture diagram + Cleanup plan   | Tài liệu hóa toàn bộ kiến trúc          |

---

### Tuần 10: System Design Luyện Tập (Ngày 68-74)

**Mục tiêu:** Thực hành thiết kế kiến trúc trên giấy/whiteboard

| Ngày | Kịch Bản Thiết Kế                                          | File Tham Khảo                                  |
| ---- | ---------------------------------------------------------- | ----------------------------------------------- |
| 68   | E-commerce multi-region active-active                      | `3-system-design-scenarios.md` Scenario 1        |
| 69   | Hybrid cloud cho enterprise bank                           | `3-system-design-scenarios.md` Scenario 2        |
| 70   | Serverless API platform                                    | `3-system-design-scenarios.md` Scenario 3        |
| 71   | Global media streaming platform                            | `3-system-design-scenarios.md` Scenario 4        |
| 72   | Zero-trust network architecture                            | `3-system-design-scenarios.md` Scenario 5        |
| 73   | Self-designed: Thiết kế network cho use case bạn biết     | Kinh nghiệm thực tế + kiến thức học              |
| 74   | Timed practice: Thiết kế 1 scenario trong 30 phút        | Mock interview simulation                        |

**Nguyên tắc luyện tập:**
1. Không nhìn tài liệu — design từ bộ nhớ
2. Nói to các quyết định và trade-offs
3. Tự hỏi: "Interviewer sẽ hỏi gì về design này?"
4. Sau khi xong, xem lại scenario file và note gì còn thiếu

---

### Tuần 11: Interview Preparation (Ngày 75-81)

**Mục tiêu:** Polish câu trả lời và stories

| Ngày | Hoạt Động                                              | Mục Tiêu                                              |
| ---- | ------------------------------------------------------ | ----------------------------------------------------- |
| 75   | Ôn lại 20 câu hỏi Interview Guide — lần 1             | Identify gaps — chỗ nào còn không chắc               |
| 76   | Deep dive các topics còn yếu                          | Fill gaps trong kiến thức                             |
| 77   | Finalize 3 STAR stories                               | Mỗi story < 4 phút kể, có numbers                    |
| 78   | Practice nói to stories (không đọc)                   | Natural delivery, không robotic                       |
| 79   | Architecture drawing practice (5 diagrams)            | Vẽ không sai, nhanh trong 15-20 phút                 |
| 80   | Chuẩn bị câu hỏi hỏi ngược interviewer               | 5-7 câu hỏi thông minh                               |
| 81   | Full mock interview với người khác (nếu có thể)       | 60 phút: technical + behavioral + system design       |

---

### Tuần 12: Final Sprint (Ngày 82-90)

| Ngày | Hoạt Động                                              | Ghi Chú                                  |
| ---- | ------------------------------------------------------ | ---------------------------------------- |
| 82   | Ôn lại 20 câu hỏi lần 2 — focus vào weak areas       | Speed drill                              |
| 83   | System design practice lần 2                          | Faster, cleaner                          |
| 84   | Review production checklist                            | Refresh operational knowledge            |
| 85   | Review common trade-offs và decision framework        | Có sẵn framework khi nói chuyện          |
| 86   | Light review — không học mới                          | Giữ tinh thần thoải mái                  |
| 87   | Rest & preparation ngày phỏng vấn                     | Nghỉ ngơi đầy đủ                         |
| 88   | Phỏng vấn hoặc self-assessment                        | Execute!                                 |
| 89   | Reflection: Gì làm tốt, gì cần cải thiện             | Learning feedback loop                  |
| 90   | Next steps: Certificate? New role? Next topic?        | Định hướng tiếp theo                     |

---

## 📊 Theo Dõi Tiến Độ

### Tuần Check-in Template

Copy và điền vào mỗi cuối tuần:

```markdown
## Tuần [X] — Check-in

### Đã Hoàn Thành
- [ ] Nội dung 1
- [ ] Lab/exercise

### Chưa Hoàn Thành (và lý do)
- 

### Điều Tôi Học Tốt Nhất Tuần Này
- 

### Điều Tôi Vẫn Chưa Chắc
- 

### Kế Hoạch Tuần Tới
- 

### Confidence Score (1-10)
VPC: /10 | Security: /10 | LB: /10 | DNS: /10 | CloudFront: /10
Connectivity: /10 | Advanced: /10 | Monitoring: /10
```

---

## 🎯 Điều Chỉnh Theo Nền Tảng

### Nếu bạn là Absolute Beginner (Chưa biết AWS)

```
→ Kéo dài mỗi tháng thêm 2 tuần (tổng 4 tháng)
→ Dành thêm thời gian cho Labs
→ Xem AWS Console tour trước khi đọc tài liệu
→ Dùng AWS Free Tier — tạo account ngay
→ Tham gia AWS Community Discord/Reddit để hỏi đáp
```

### Nếu bạn có 1-2 năm kinh nghiệm AWS

```
→ Skip tuần 1 Labs (tự làm nhanh hơn) — focus vào edge cases
→ Tăng tốc tháng 1 xuống 3 tuần
→ Thêm thời gian cho Advanced topics (TGW, Network Firewall)
→ Focus nhiều hơn vào System Design scenarios
```

### Nếu bạn có 3+ năm kinh nghiệm AWS Networking

```
→ Skip Lab 1-2 (review nhanh)
→ Focus tháng 3 nhiều hơn: thêm kịch bản design phức tạp
→ Luyện explain như đang dạy junior engineer
→ Chuẩn bị cho senior/staff level questions (architecture, cost at scale)
→ Nếu nhắm ANS-C01 cert: thêm practice exams
```

---

## 📚 Resources Bổ Sung

### AWS Documentation (Tài Liệu Chính Thức)

```
VPC:         docs.aws.amazon.com/vpc/latest/userguide/
CloudFront:  docs.aws.amazon.com/cloudfront/latest/APIReference/
Route 53:    docs.aws.amazon.com/route53/latest/APIReference/
ELB:         docs.aws.amazon.com/elasticloadbalancing/latest/userguide/
```

### AWS re:Invent Videos (YouTube)

```
- "AWS re:Invent 2023: Advanced VPC Design and New Capabilities" (NET401)
- "AWS re:Invent 2023: Networking foundations" (NET201)
- "AWS re:Invent 2023: CloudFront best practices" (NET302)
- "AWS re:Invent 2022: Transit Gateway deep dive" (NET403)
```

### Exam Resources (Nếu nhắm chứng chỉ)

```
ANS-C01 — AWS Certified Advanced Networking – Specialty:
- Dùng knowledge base này làm base
- Thêm: AWS Whitepaper "Amazon VPC Connectivity Options"
- Practice exams: Tutorials Dojo, Whizlabs
```

### Tools Hữu Ích

```
draw.io / Lucidchart  → Vẽ AWS architecture diagrams
CIDR.xyz              → CIDR calculator online
subnet-calculator.io  → Subnet planning tool
CloudTrail Insights   → Phát hiện bất thường API calls
AWS Cost Explorer     → Phân tích chi phí theo service
```

---

## 💡 Lời Khuyên Để Duy Trì Nhịp Học

1. **30 phút/ngày beats 4 giờ/tuần** — Consistency quan trọng hơn intensity
2. **Active recall** — Sau khi đọc, đóng tài liệu và tóm tắt từ bộ nhớ
3. **Teach it** — Giải thích cho người khác hoặc rubber duck (nói to một mình) là cách học hiệu quả nhất
4. **Build while you learn** — Lab ngay sau khi đọc lý thuyết, đừng đợi
5. **Track metrics** — Confidence score mỗi tuần giúp bạn thấy tiến bộ
6. **Find a study buddy** — Học cùng người khác, mock interview nhau

---

**Chúc bạn thành công! AWS Networking mastery là hành trình, không phải đích đến.**

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
