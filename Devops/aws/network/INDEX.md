# AWS Networking & Content Delivery — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về AWS Networking and Content Delivery — từ VPC (Virtual Private Cloud) đến CloudFront CDN (Content Delivery Network)

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/network
├── README.md                                    [BẮT ĐẦU TỪ ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                     Chỉ mục đầy đủ (file này)
│
├── 01-vpc-fundamentals/
│   ├── README.md                                Tổng quan VPC & kiến trúc mạng AWS
│   ├── 1-vpc-architecture.md                   (Sẽ tạo) VPC, CIDR, AZ, region design
│   ├── 2-subnets.md                             (Sẽ tạo) Public/Private/Isolated subnets
│   ├── 3-route-tables.md                        (Sẽ tạo) Bảng định tuyến & Internet Gateway
│   ├── 4-nat-gateway.md                         (Sẽ tạo) NAT Gateway vs NAT Instance
│   └── 5-vpc-peering.md                         (Sẽ tạo) VPC Peering & Resource Sharing
│
├── 02-security/
│   ├── README.md                                Tổng quan bảo mật mạng AWS
│   ├── 1-security-groups.md                     (Sẽ tạo) Stateful firewall, rules, best practices
│   ├── 2-network-acls.md                        (Sẽ tạo) Stateless firewall, so sánh với SG
│   ├── 3-waf.md                                 (Sẽ tạo) Web Application Firewall, rules, managed rules
│   ├── 4-shield.md                              (Sẽ tạo) Shield Standard & Advanced, DDoS protection
│   └── 5-network-firewall.md                    (Sẽ tạo) AWS Network Firewall, inspection, logging
│
├── 03-load-balancing/
│   ├── README.md                                Tổng quan Elastic Load Balancing
│   ├── 1-alb.md                                 (Sẽ tạo) Application Load Balancer, Layer 7, routing rules
│   ├── 2-nlb.md                                 (Sẽ tạo) Network Load Balancer, Layer 4, ultra-low latency
│   ├── 3-target-groups.md                       (Sẽ tạo) Target groups, health checks, deregistration delay
│   ├── 4-ssl-tls.md                             (Sẽ tạo) SSL/TLS termination, ACM, HTTPS best practices
│   └── 5-advanced-patterns.md                   (Sẽ tạo) Cross-zone LB, sticky sessions, weighted routing
│
├── 04-dns-route53/
│   ├── README.md                                Tổng quan Route 53 & DNS
│   ├── 1-dns-fundamentals.md                    (Sẽ tạo) DNS cơ bản, record types, TTL
│   ├── 2-hosted-zones.md                        (Sẽ tạo) Public & Private hosted zones
│   ├── 3-routing-policies.md                    (Sẽ tạo) Simple, Weighted, Latency, Failover, Geo
│   ├── 4-health-checks.md                       (Sẽ tạo) Health checks, DNS failover automation
│   └── 5-resolver.md                            (Sẽ tạo) Route 53 Resolver, hybrid DNS, inbound/outbound endpoints
│
├── 05-cdn-cloudfront/
│   ├── README.md                                Tổng quan CloudFront CDN
│   ├── 1-cloudfront-basics.md                   (Sẽ tạo) Distributions, edge locations, origins
│   ├── 2-cache-behaviors.md                     (Sẽ tạo) Cache policies, TTL, cache key
│   ├── 3-origin-access.md                       (Sẽ tạo) OAC, OAI, S3 bucket protection
│   ├── 4-lambda-edge.md                         (Sẽ tạo) Lambda@Edge, CloudFront Functions
│   └── 5-performance-cost.md                    (Sẽ tạo) Price classes, compression, optimization
│
├── 06-connectivity/
│   ├── README.md                                Tổng quan kết nối hybrid & on-premises
│   ├── 1-site-to-site-vpn.md                   (Sẽ tạo) VPN Site-to-Site, Customer Gateway, VGW
│   ├── 2-client-vpn.md                          (Sẽ tạo) Client VPN, remote access, authentication
│   ├── 3-direct-connect.md                      (Sẽ tạo) Direct Connect, dedicated/hosted connections
│   ├── 4-direct-connect-gateway.md              (Sẽ tạo) DX Gateway, multi-VPC, multi-region
│   └── 5-transit-gateway.md                     (Sẽ tạo) Transit Gateway, hub-and-spoke, route domains
│
├── 07-advanced-networking/
│   ├── README.md                                Tổng quan networking nâng cao
│   ├── 1-vpc-endpoints.md                       (Sẽ tạo) Gateway & Interface endpoints, PrivateLink
│   ├── 2-privatelink.md                         (Sẽ tạo) AWS PrivateLink, endpoint services
│   ├── 3-global-accelerator.md                  (Sẽ tạo) Global Accelerator, Anycast, use cases
│   ├── 4-elastic-ip-eni.md                      (Sẽ tạo) Elastic IP, ENI, secondary IPs
│   └── 5-ipv6.md                                (Sẽ tạo) IPv6 trong VPC, dual-stack architecture
│
├── 08-monitoring/
│   ├── README.md                                Tổng quan giám sát mạng
│   ├── 1-vpc-flow-logs.md                       (Sẽ tạo) Flow logs, phân tích, CloudWatch/S3
│   ├── 2-cloudwatch-networking.md               (Sẽ tạo) Metrics cho ELB, CloudFront, VPN
│   ├── 3-network-access-analyzer.md             (Sẽ tạo) Network access analysis, compliance
│   ├── 4-reachability-analyzer.md               (Sẽ tạo) Path analysis, troubleshooting tool
│   └── 5-observability-checklist.md             (Sẽ tạo) Checklist giám sát toàn diện
│
├── 09-troubleshooting/
│   ├── README.md                                Tổng quan xử lý sự cố mạng
│   ├── 1-connectivity-debug.md                  EC2 không kết nối được — chẩn đoán từng bước
│   ├── 2-security-group-nacl-debug.md           Gỡ lỗi Security Group & NACL
│   ├── 3-dns-issues.md                          Route 53 & DNS resolution failures
│   ├── 4-load-balancer-issues.md                ALB/NLB health check failures, 503 errors
│   ├── 5-cloudfront-issues.md                   Cache miss, origin errors, SSL issues
│   └── 6-production-checklist.md                Checklist trước khi đưa vào production
│
└── 10-interview-prep/
    ├── README.md                                Tổng quan chuẩn bị phỏng vấn
    ├── 1-INTERVIEW_GUIDE.md                     Top 20 câu hỏi AWS Networking kèm đáp án
    ├── 2-star-stories.md                        Mẫu câu chuyện incident theo STAR (5 stories)
    ├── 3-system-design-scenarios.md             5 kịch bản thiết kế hệ thống thực tế
    ├── 4-hands-on-exercises.md                  5 bài tập thực hành có hướng dẫn step-by-step
    └── 5-90-day-study-plan.md                   Kế hoạch học 90 ngày có lộ trình chi tiết
```

---

## ✅ Đã Được Tạo

| Chủ Đề                              | File                                           | Trạng Thái | Chất Lượng |
| ----------------------------------- | ---------------------------------------------- | ---------- | ---------- |
| **Tổng Quan & Lộ Trình**            | README.md                                      | ✅         | Toàn diện  |
| **Chỉ Mục Đầy Đủ**                  | INDEX.md                                       | ✅         | Đầy đủ     |
| **VPC Fundamentals — Tổng Quan**    | 01-vpc-fundamentals/README.md                  | ✅         | Toàn diện  |
| **VPC Architecture & CIDR**        | 01-vpc-fundamentals/1-vpc-architecture.md      | ✅         | Toàn diện  |
| **Public/Private/Isolated Subnets** | 01-vpc-fundamentals/2-subnets.md               | ✅         | Toàn diện  |
| **Route Tables & Internet Gateway** | 01-vpc-fundamentals/3-route-tables.md          | ✅         | Toàn diện  |
| **NAT Gateway vs NAT Instance**     | 01-vpc-fundamentals/4-nat-gateway.md           | ✅         | Toàn diện  |
| **VPC Peering & Resource Sharing**  | 01-vpc-fundamentals/5-vpc-peering.md           | ✅         | Toàn diện  |
| **Bảo Mật Mạng — Tổng Quan**        | 02-security/README.md                          | ✅         | Toàn diện  |
| **Security Groups — Stateful FW**   | 02-security/1-security-groups.md               | ✅         | Toàn diện  |
| **Network ACLs — Stateless FW**     | 02-security/2-network-acls.md                  | ✅         | Toàn diện  |
| **WAF — Web Application Firewall**  | 02-security/3-waf.md                           | ✅         | Toàn diện  |
| **Shield Standard & Advanced**      | 02-security/4-shield.md                        | ✅         | Toàn diện  |
| **AWS Network Firewall**            | 02-security/5-network-firewall.md              | ✅         | Toàn diện  |
| **Load Balancing — Tổng Quan**      | 03-load-balancing/README.md                    | ✅         | Toàn diện  |
| **ALB — Application Load Balancer** | 03-load-balancing/1-alb.md                     | ✅         | Toàn diện  |
| **NLB — Network Load Balancer**     | 03-load-balancing/2-nlb.md                     | ✅         | Toàn diện  |
| **Target Groups & Health Checks**   | 03-load-balancing/3-target-groups.md           | ✅         | Toàn diện  |
| **SSL/TLS Termination & ACM**       | 03-load-balancing/4-ssl-tls.md                 | ✅         | Toàn diện  |
| **Advanced Load Balancing Patterns**| 03-load-balancing/5-advanced-patterns.md       | ✅         | Toàn diện  |
| **DNS & Route 53 — Tổng Quan**      | 04-dns-route53/README.md                       | ✅         | Toàn diện  |
| **DNS Fundamentals & Record Types** | 04-dns-route53/1-dns-fundamentals.md           | ✅         | Toàn diện  |
| **Public & Private Hosted Zones**   | 04-dns-route53/2-hosted-zones.md               | ✅         | Toàn diện  |
| **7 Routing Policies**              | 04-dns-route53/3-routing-policies.md           | ✅         | Toàn diện  |
| **Health Checks & DNS Failover**    | 04-dns-route53/4-health-checks.md              | ✅         | Toàn diện  |
| **Route 53 Resolver & Hybrid DNS**  | 04-dns-route53/5-resolver.md                   | ✅         | Toàn diện  |
| **CloudFront CDN — Tổng Quan**      | 05-cdn-cloudfront/README.md                    | ✅         | Toàn diện  |
| **CloudFront Basics & Distributions**| 05-cdn-cloudfront/1-cloudfront-basics.md      | ✅         | Toàn diện  |
| **Cache Behaviors & TTL**           | 05-cdn-cloudfront/2-cache-behaviors.md         | ✅         | Toàn diện  |
| **OAC, OAI & Origin Protection**    | 05-cdn-cloudfront/3-origin-access.md           | ✅         | Toàn diện  |
| **Lambda@Edge & CF Functions**      | 05-cdn-cloudfront/4-lambda-edge.md             | ✅         | Toàn diện  |
| **Performance & Cost Optimization** | 05-cdn-cloudfront/5-performance-cost.md        | ✅         | Toàn diện  |
| **Connectivity — Tổng Quan**        | 06-connectivity/README.md                      | ✅         | Toàn diện  |
| **Site-to-Site VPN**                | 06-connectivity/1-site-to-site-vpn.md          | ✅         | Toàn diện  |
| **Client VPN — Remote Access**      | 06-connectivity/2-client-vpn.md                | ✅         | Toàn diện  |
| **Direct Connect — Kết Nối Riêng**  | 06-connectivity/3-direct-connect.md            | ✅         | Toàn diện  |
| **Direct Connect Gateway**          | 06-connectivity/4-direct-connect-gateway.md    | ✅         | Toàn diện  |
| **Transit Gateway — Hub-and-Spoke** | 06-connectivity/5-transit-gateway.md           | ✅         | Toàn diện  |
| **Advanced Networking — Tổng Quan** | 07-advanced-networking/README.md               | ✅         | Toàn diện  |
| **VPC Endpoints & PrivateLink**     | 07-advanced-networking/1-vpc-endpoints.md      | ✅         | Toàn diện  |
| **AWS PrivateLink — Dịch Vụ Riêng** | 07-advanced-networking/2-privatelink.md        | ✅         | Toàn diện  |
| **Global Accelerator — Anycast**    | 07-advanced-networking/3-global-accelerator.md | ✅         | Toàn diện  |
| **Elastic IP & ENI**                | 07-advanced-networking/4-elastic-ip-eni.md     | ✅         | Toàn diện  |
| **IPv6 & Dual-Stack VPC**           | 07-advanced-networking/5-ipv6.md               | ✅         | Toàn diện  |
| **Giám Sát Mạng — Tổng Quan**       | 08-monitoring/README.md                        | ✅         | Toàn diện  |
| **VPC Flow Logs — Phân Tích**       | 08-monitoring/1-vpc-flow-logs.md               | ✅         | Toàn diện  |
| **CloudWatch Networking Metrics**   | 08-monitoring/2-cloudwatch-networking.md       | ✅         | Toàn diện  |
| **Network Access Analyzer**         | 08-monitoring/3-network-access-analyzer.md     | ✅         | Toàn diện  |
| **Reachability Analyzer**           | 08-monitoring/4-reachability-analyzer.md       | ✅         | Toàn diện  |
| **Observability Checklist**         | 08-monitoring/5-observability-checklist.md     | ✅         | Toàn diện  |
| **Troubleshooting — Tổng Quan**     | 09-troubleshooting/README.md                   | ✅         | Toàn diện  |
| **Connectivity Debug — EC2**        | 09-troubleshooting/1-connectivity-debug.md     | ✅         | Toàn diện  |
| **Security Group & NACL Debug**     | 09-troubleshooting/2-security-group-nacl-debug.md | ✅      | Toàn diện  |
| **DNS Issues & Route 53 Debug**     | 09-troubleshooting/3-dns-issues.md             | ✅         | Toàn diện  |
| **Load Balancer Issues**            | 09-troubleshooting/4-load-balancer-issues.md   | ✅         | Toàn diện  |
| **CloudFront Issues**               | 09-troubleshooting/5-cloudfront-issues.md      | ✅         | Toàn diện  |
| **Production Checklist**            | 09-troubleshooting/6-production-checklist.md   | ✅         | Toàn diện  |
| **Interview Prep — Tổng Quan**      | 10-interview-prep/README.md                    | ✅         | Toàn diện  |
| **Top 20 Q&A AWS Networking**       | 10-interview-prep/1-INTERVIEW_GUIDE.md         | ✅         | Toàn diện  |
| **STAR Stories — 5 Templates**      | 10-interview-prep/2-star-stories.md            | ✅         | Toàn diện  |
| **System Design Scenarios**         | 10-interview-prep/3-system-design-scenarios.md | ✅         | Toàn diện  |
| **Hands-on Exercises — 5 Labs**     | 10-interview-prep/4-hands-on-exercises.md      | ✅         | Toàn diện  |
| **Kế Hoạch Học 90 Ngày**            | 10-interview-prep/5-90-day-study-plan.md       | ✅         | Toàn diện  |

---

## 🎯 Cần Tạo Tiếp Theo (Theo Thứ Tự Ưu Tiên)

### Ưu Tiên Cao — Kỹ Năng Cốt Lõi

- [x] `01-vpc-fundamentals/README.md` — Nền tảng VPC, kiến trúc mạng ✅
- [x] `01-vpc-fundamentals/1-vpc-architecture.md` — Thiết kế VPC, CIDR planning ✅
- [x] `01-vpc-fundamentals/2-subnets.md` — Public/Private/Isolated subnets ✅
- [x] `01-vpc-fundamentals/3-route-tables.md` — Bảng định tuyến & Internet Gateway ✅
- [x] `01-vpc-fundamentals/4-nat-gateway.md` — NAT Gateway vs NAT Instance ✅
- [x] `01-vpc-fundamentals/5-vpc-peering.md` — VPC Peering & Resource Sharing ✅
- [x] `02-security/README.md` — Bảo mật mạng tổng quan ✅
- [x] `02-security/1-security-groups.md` — Security Groups chi tiết ✅
- [x] `02-security/2-network-acls.md` — Network ACLs & so sánh với SG ✅
- [x] `02-security/3-waf.md` — WAF, rules, managed rules ✅
- [x] `02-security/4-shield.md` — Shield Standard & Advanced, DDoS protection ✅
- [x] `02-security/5-network-firewall.md` — AWS Network Firewall, deep inspection ✅
- [x] `03-load-balancing/README.md` — Cân bằng tải tổng quan ✅
- [x] `03-load-balancing/1-alb.md` — Application Load Balancer ✅
- [x] `03-load-balancing/2-nlb.md` — Network Load Balancer ✅
- [x] `03-load-balancing/3-target-groups.md` — Target Groups & Health Checks ✅
- [x] `03-load-balancing/4-ssl-tls.md` — SSL/TLS Termination & ACM ✅
- [x] `03-load-balancing/5-advanced-patterns.md` — Advanced Patterns ✅
- [x] `04-dns-route53/README.md` — DNS & Route 53 tổng quan ✅
- [x] `04-dns-route53/1-dns-fundamentals.md` — DNS cơ bản, record types, TTL ✅
- [x] `04-dns-route53/2-hosted-zones.md` — Public & Private hosted zones ✅
- [x] `04-dns-route53/3-routing-policies.md` — Tất cả 7 routing policies ✅
- [x] `04-dns-route53/4-health-checks.md` — Health checks, DNS failover automation ✅
- [x] `04-dns-route53/5-resolver.md` — Route 53 Resolver, hybrid DNS ✅

### Ưu Tiên Trung Bình — Kỹ Năng Nâng Cao

- [x] `05-cdn-cloudfront/README.md` — CloudFront CDN tổng quan ✅
- [x] `05-cdn-cloudfront/1-cloudfront-basics.md` — Distributions, Edge Locations, Origins ✅
- [x] `05-cdn-cloudfront/2-cache-behaviors.md` — Cache Policies, TTL, Cache Key ✅
- [x] `05-cdn-cloudfront/3-origin-access.md` — OAC, OAI, S3 bucket protection ✅
- [x] `05-cdn-cloudfront/4-lambda-edge.md` — Lambda@Edge & CloudFront Functions ✅
- [x] `05-cdn-cloudfront/5-performance-cost.md` — Price Classes, Compression, Optimization ✅
- [x] `06-connectivity/README.md` — Kết nối hybrid tổng quan ✅
- [x] `06-connectivity/1-site-to-site-vpn.md` — Site-to-Site VPN, CGW, VGW ✅
- [x] `06-connectivity/2-client-vpn.md` — Client VPN, remote access, auth ✅
- [x] `06-connectivity/3-direct-connect.md` — Direct Connect, VIF types, HA ✅
- [x] `06-connectivity/4-direct-connect-gateway.md` — DX Gateway, multi-VPC/region ✅
- [x] `06-connectivity/5-transit-gateway.md` — Transit Gateway, hub-and-spoke ✅
- [x] `07-advanced-networking/README.md` — Advanced networking tổng quan ✅
- [x] `07-advanced-networking/1-vpc-endpoints.md` — VPC Endpoints & PrivateLink ✅
- [x] `07-advanced-networking/2-privatelink.md` — AWS PrivateLink, endpoint services ✅
- [x] `07-advanced-networking/3-global-accelerator.md` — Global Accelerator, Anycast ✅
- [x] `07-advanced-networking/4-elastic-ip-eni.md` — Elastic IP, ENI, secondary IPs ✅
- [x] `07-advanced-networking/5-ipv6.md` — IPv6 & Dual-Stack VPC ✅
- [x] `10-interview-prep/README.md` — Tổng quan & lộ trình chuẩn bị ✅
- [x] `10-interview-prep/1-INTERVIEW_GUIDE.md` — Top 20 interview questions ✅

### Ưu Tiên Thấp — Tài Liệu Tham Khảo

- [x] `08-monitoring/README.md` — Giám sát mạng tổng quan ✅
- [x] `08-monitoring/1-vpc-flow-logs.md` — VPC Flow Logs phân tích ✅
- [x] `08-monitoring/2-cloudwatch-networking.md` — CloudWatch Metrics cho ELB, CloudFront, VPN ✅
- [x] `08-monitoring/3-network-access-analyzer.md` — Phân tích quyền truy cập mạng ✅
- [x] `08-monitoring/4-reachability-analyzer.md` — Phân tích kết nối từng bước ✅
- [x] `08-monitoring/5-observability-checklist.md` — Checklist giám sát toàn diện ✅
- [x] `09-troubleshooting/README.md` — Xử lý sự cố tổng quan ✅
- [x] `09-troubleshooting/1-connectivity-debug.md` — EC2 không kết nối được ✅
- [x] `09-troubleshooting/2-security-group-nacl-debug.md` — Gỡ lỗi Security Group & NACL ✅
- [x] `09-troubleshooting/3-dns-issues.md` — Route 53 & DNS resolution failures ✅
- [x] `09-troubleshooting/4-load-balancer-issues.md` — ALB/NLB health check failures ✅
- [x] `09-troubleshooting/5-cloudfront-issues.md` — Cache miss, origin errors, SSL issues ✅
- [x] `09-troubleshooting/6-production-checklist.md` — Production checklist ✅
- [x] `10-interview-prep/2-star-stories.md` — 5 STAR story templates ✅
- [x] `10-interview-prep/3-system-design-scenarios.md` — 5 Design scenarios ✅
- [x] `10-interview-prep/4-hands-on-exercises.md` — 5 Lab exercises ✅
- [x] `10-interview-prep/5-90-day-study-plan.md` — Kế hoạch 90 ngày ✅

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Cho Mục Đích Tự Học

```
1. Bắt đầu với README.md
2. Chọn lộ trình học (Beginner / Intermediate / Advanced)
3. Đi qua từng section theo thứ tự
4. Thực hành hands-on trên AWS Free Tier sau mỗi chủ đề
5. Vẽ sơ đồ kiến trúc để kiểm tra hiểu biết
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 10-interview-prep/1-INTERVIEW_GUIDE.md
2. Nắm vững VPC, Security Groups, ALB, Route 53, CloudFront (luôn được hỏi)
3. Chuẩn bị 2-3 incident stories từ kinh nghiệm thực tế
4. Luyện tập thiết kế kiến trúc mạng trên giấy
5. Mock interview với đồng nghiệp về system design scenarios
```

### Cho Công Việc Thực Tế

```
Dùng làm tài liệu tham khảo:
- Trước khi deploy: Xem 09-troubleshooting/6-production-checklist.md
- Khi có sự cố: Vào 09-troubleshooting/ để chẩn đoán từng bước
- Thiết kế mới: Tham khảo 01-vpc-fundamentals/ và 06-connectivity/
- Bảo mật: Kiểm tra 02-security/ checklist
- Hiệu suất: Xem 05-cdn-cloudfront/ và 07-advanced-networking/
```

### Cho System Design

```
1. Đọc README.md để nắm tổng quan các dịch vụ
2. Dùng 01-vpc-fundamentals/ để thiết kế network foundation
3. Chọn load balancer phù hợp từ 03-load-balancing/
4. Cấu hình DNS và CDN từ 04-dns-route53/ và 05-cdn-cloudfront/
5. Áp dụng bảo mật từ 02-security/
```

---

## 📊 Ước Tính Thời Gian Học

| Section                          | Thời Gian | Độ Khó | Ưu Tiên         |
| -------------------------------- | --------- | ------ | --------------- |
| VPC Fundamentals                 | 6-8 giờ   | ⭐⭐   | Bắt buộc        |
| Security (SG, NACL, WAF)         | 6-8 giờ   | ⭐⭐   | Bắt buộc        |
| Load Balancing                   | 4-6 giờ   | ⭐⭐   | Bắt buộc        |
| DNS & Route 53                   | 6-8 giờ   | ⭐⭐   | Bắt buộc        |
| CloudFront CDN                   | 6-8 giờ   | ⭐⭐   | Bắt buộc        |
| Hybrid Connectivity (VPN/DX/TGW) | 8-10 giờ  | ⭐⭐⭐ | Nên học         |
| Advanced Networking              | 6-8 giờ   | ⭐⭐⭐ | Nên học         |
| Monitoring & Troubleshooting     | 4-6 giờ   | ⭐⭐   | Nên học         |
| Interview Prep                   | 4-6 giờ   | ⭐     | Trước phỏng vấn |

**Tổng cộng: 50-70 giờ để có kiến thức AWS Networking toàn diện**

---

## 🎓 Cấp Độ Kỹ Năng Hỗ Trợ

### Người Mới — Beginner (0-1 năm kinh nghiệm)

- [ ] Tạo VPC từ đầu với Public/Private subnets
- [ ] Cấu hình Security Groups cho EC2 instances
- [ ] Thiết lập ALB cho web application đơn giản
- [ ] Đăng ký domain và trỏ DNS với Route 53
- [ ] Bật CloudFront cho S3 static website

**Thời gian để thành thạo:** 1-2 tháng

### Trung Cấp — Intermediate (1-3 năm kinh nghiệm)

- [ ] Thiết kế VPC multi-tier production-ready
- [ ] Cấu hình Route 53 với multiple routing policies
- [ ] Triển khai CloudFront với custom origins và cache behaviors
- [ ] Thiết lập VPN Site-to-Site kết nối on-premises
- [ ] Phân tích VPC Flow Logs để debug kết nối

**Thời gian để nâng cấp:** 2-3 tháng thực hành

### Nâng Cao — Advanced (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc Transit Gateway hub-and-spoke cho multi-VPC
- [ ] Direct Connect với failover tự động sang VPN
- [ ] Multi-region active-active với Route 53 và Global Accelerator
- [ ] Network security-in-depth (WAF + Shield + GuardDuty + Network Firewall)
- [ ] Tự động hóa toàn bộ network stack với Terraform/CDK

**Thời gian để nâng cấp:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu             | Vị Trí                                                                           |
| ------------------- | -------------------------------------------------------------------------------- |
| Tổng quan nhanh     | [README.md](README.md)                                                           |
| Thiết kế VPC        | [01-vpc-fundamentals/README.md](01-vpc-fundamentals/README.md)                   |
| Bảo mật mạng        | [02-security/README.md](02-security/README.md)                                   |
| Cân bằng tải        | [03-load-balancing/README.md](03-load-balancing/README.md)                       |
| DNS & Route 53      | [04-dns-route53/README.md](04-dns-route53/README.md)                             |
| CloudFront CDN      | [05-cdn-cloudfront/README.md](05-cdn-cloudfront/README.md)                       |
| Hybrid connectivity | [06-connectivity/README.md](06-connectivity/README.md)                           |
| Mạng nâng cao       | [07-advanced-networking/README.md](07-advanced-networking/README.md)             |
| Giám sát mạng       | [08-monitoring/README.md](08-monitoring/README.md)                               |
| Xử lý sự cố         | [09-troubleshooting/README.md](09-troubleshooting/README.md)                     |
| Câu hỏi phỏng vấn   | [10-interview-prep/1-INTERVIEW_GUIDE.md](10-interview-prep/1-INTERVIEW_GUIDE.md) |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và theo dõi tiến độ của bạn:

```markdown
## AWS Networking — Tiến Độ Học

### Giai Đoạn 1: Nền Tảng (Tuần 1-2)

- [ ] VPC, Subnets, Route Tables
- [ ] Internet Gateway & NAT Gateway
- [ ] Security Groups & Network ACLs
- [ ] CIDR planning & IP addressing

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3-6)

- [ ] ALB & NLB configuration
- [ ] Route 53 hosted zones & record types
- [ ] Route 53 routing policies (7 loại)
- [ ] CloudFront distributions & cache behaviors
- [ ] WAF rules & managed rule groups

### Giai Đoạn 3: Nâng Cao (Tuần 7-10)

- [ ] VPN Site-to-Site setup
- [ ] Direct Connect concepts
- [ ] Transit Gateway hub-and-spoke
- [ ] VPC Endpoints & PrivateLink
- [ ] VPC Flow Logs analysis

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Global Accelerator use cases
- [ ] Multi-region architecture design
- [ ] Network security-in-depth
- [ ] Terraform for networking
- [ ] Mock interviews & system design
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Thiết kế VPC production-ready từ đầu (không cần tham khảo)
- [ ] Giải thích sự khác biệt giữa Security Group và Network ACL chính xác
- [ ] Chọn đúng Load Balancer cho từng use case
- [ ] Cấu hình Route 53 với tất cả routing policies

### ✅ Năng Lực Vận Hành

- [ ] Debug kết nối mạng một cách có hệ thống
- [ ] Phân tích VPC Flow Logs để tìm root cause
- [ ] Tối ưu CloudFront cache hit ratio
- [ ] Triển khai WAF rules bảo vệ ứng dụng web

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin 20 câu hỏi AWS Networking hàng đầu
- [ ] Kể 2-3 incident stories theo định dạng STAR
- [ ] Thiết kế multi-region, high-availability architecture
- [ ] Thảo luận trade-offs giữa các dịch vụ mạng AWS

---

## 🚀 Các Bước Tiếp Theo

### Ngay Lập Tức (Tuần Này)

1. Đọc README.md đầy đủ
2. Chọn lộ trình học phù hợp
3. Tạo AWS Free Tier account (nếu chưa có)
4. Bắt đầu với `01-vpc-fundamentals/`

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành VPC Fundamentals + Security
2. Thực hành tạo VPC 3-tier trên AWS Console
3. Bắt đầu Load Balancing & Route 53
4. Labs hands-on cho mỗi chủ đề

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành CloudFront, VPN/Direct Connect
2. Deep dive Transit Gateway
3. Chuẩn bị 2-3 incident stories
4. Mock interviews với bạn bè / đồng nghiệp

### Dài Hạn (3 Tháng Tới)

1. Nắm vững toàn bộ AWS Networking
2. Xây dựng lab project: multi-region app với CloudFront + Route 53 failover
3. Lấy chứng chỉ AWS (ANS-C01 — Advanced Networking Specialty)
4. Ứng dụng vào công việc thực tế hoặc phỏng vấn vị trí mới

---

## 💡 Lời Khuyên Thực Tế

1. **Học qua thực hành:** Tạo VPC thực tế trên AWS Console trước khi dùng Terraform
2. **Vẽ sơ đồ:** Mỗi kiến trúc đều nên được vẽ ra giấy hoặc draw.io
3. **Hiểu trade-offs:** Với mỗi dịch vụ, hỏi "khi nào dùng cái này thay vì cái kia?"
4. **Đọc error messages:** VPC Flow Logs và CloudWatch logs nói cho bạn biết vấn đề ở đâu
5. **Chi phí quan trọng:** NAT Gateway, Direct Connect, Global Accelerator đắt tiền — biết khi nào cần
6. **Security first:** Luôn áp dụng Principle of Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu)
7. **Test failover:** Luôn test thực tế kịch bản failover trước khi production
8. **Document kiến trúc:** Sơ đồ network diagram là tài sản quý giá cho team

---

## 📞 Đóng Góp

Phát hiện lỗi? Muốn thêm nội dung?

Đây là tài liệu sống. Đóng góp được chào đón:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm sections cho chủ đề chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Bổ sung hands-on exercises
- [ ] Cập nhật các tính năng AWS mới

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 2.0 (10-interview-prep hoàn thành — Knowledge Base HOÀN CHỈNH)
**Trạng Thái:** ✅ README.md & INDEX.md hoàn thành | ✅ 01-vpc-fundamentals hoàn thành | ✅ 02-security hoàn thành | ✅ 03-load-balancing hoàn thành | ✅ 04-dns-route53 hoàn thành | ✅ 05-cdn-cloudfront hoàn thành | ✅ 06-connectivity hoàn thành | ✅ 07-advanced-networking hoàn thành | ✅ 08-monitoring hoàn thành | ✅ 09-troubleshooting hoàn thành | ✅ 10-interview-prep hoàn thành
