# 📋 Top 20 Câu Hỏi Phỏng Vấn AWS Networking — Hướng Dẫn Đầy Đủ

> Bộ câu hỏi + đáp án chi tiết được tổng hợp từ thực tế phỏng vấn tại các công ty công nghệ lớn. Mỗi câu trả lời được thiết kế để gây ấn tượng, không chỉ đúng mà còn cho thấy chiều sâu kiến thức.

---

## 🏷️ Phân Loại Câu Hỏi

| Danh Mục                     | Câu Số | Mức Độ  |
| ---------------------------- | ------ | ------- |
| VPC & Subnetting             | 1-4    | Junior+ |
| Security                     | 5-7    | Junior+ |
| Load Balancing               | 8-10   | Mid+    |
| DNS & Route 53               | 11-13  | Mid+    |
| Connectivity Hybrid          | 14-16  | Senior+ |
| Advanced & System Design     | 17-20  | Senior+ |

---

## 🔵 VPC & Subnetting

---

### Câu 1: Giải thích VPC là gì và tại sao cần dùng VPC thay vì EC2-Classic?

**Câu trả lời mẫu:**

VPC — Virtual Private Cloud (Đám Mây Riêng Ảo) — là mạng ảo được cô lập logic trong AWS, cho phép bạn toàn quyền kiểm soát môi trường mạng: chọn dải IP (CIDR — Classless Inter-Domain Routing), tạo subnets (mạng con), cấu hình route tables (bảng định tuyến) và network gateways (cổng mạng).

**Tại sao không dùng EC2-Classic:**
- EC2-Classic là mạng phẳng chia sẻ với tất cả khách hàng AWS — không có isolation (cô lập)
- VPC cung cấp network isolation hoàn toàn: EC2 instances của bạn không thể bị "nhìn thấy" bởi account khác
- VPC cho phép bạn kiểm soát inbound/outbound traffic ở nhiều lớp (Security Groups, NACLs)
- Có thể kết nối VPC với mạng on-premises (tại chỗ) qua VPN hoặc Direct Connect
- AWS đã ngừng hỗ trợ EC2-Classic từ tháng 8/2022

**Thành phần cốt lõi của VPC:**
```
VPC (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24)  → Internet Gateway → Internet
├── Private Subnet (10.0.2.0/24) → NAT Gateway → Internet (outbound only)
└── Isolated Subnet (10.0.3.0/24) → Không có Internet access
```

**Follow-up thường gặp:** "Một VPC có thể span nhiều regions không?" → Không, VPC bị giới hạn trong một region. Để kết nối nhiều regions, dùng VPC Peering (Kết Nối Ngang Hàng VPC) hoặc Transit Gateway (Cổng Trung Chuyển).

---

### Câu 2: Sự khác biệt giữa Public Subnet và Private Subnet là gì? Khi nào dùng cái nào?

**Câu trả lời mẫu:**

Sự khác biệt cốt lõi nằm ở **route table** (bảng định tuyến):

| Thuộc Tính              | Public Subnet                        | Private Subnet                           |
| ----------------------- | ------------------------------------ | ---------------------------------------- |
| Route to Internet       | 0.0.0.0/0 → Internet Gateway (IGW)   | 0.0.0.0/0 → NAT Gateway                 |
| Truy cập từ Internet    | Có thể (nếu có public IP)            | Không trực tiếp                          |
| Truy cập ra Internet    | Trực tiếp qua IGW                    | Qua NAT Gateway (một chiều - outbound)   |
| Use cases               | Load balancers, bastion hosts, NAT GW | App servers, databases, Lambda functions |

**Pattern kiến trúc chuẩn 3-tier:**
```
Internet
    ↓
[Public Subnet]  → ALB (Application Load Balancer), NAT Gateway, Bastion Host
    ↓
[Private Subnet] → Application servers (EC2, ECS containers)
    ↓
[Isolated Subnet] → RDS databases, ElastiCache (không cần Internet access)
```

**Lý do thiết kế như vậy:**
- **Defense in depth** (Phòng thủ theo chiều sâu): Nếu application server bị compromise (xâm phạm), attacker (kẻ tấn công) không thể trực tiếp truy cập database
- **Principle of Least Privilege** (Nguyên Tắc Đặc Quyền Tối Thiểu): Chỉ những gì cần internet mới đặt ở public subnet
- **Giảm attack surface** (bề mặt tấn công): Database không bao giờ cần internet — vậy tại sao phải expose?

---

### Câu 3: CIDR notation là gì? Tính toán subnet như thế nào cho VPC production?

**Câu trả lời mẫu:**

CIDR — Classless Inter-Domain Routing (Định Tuyến Liên Miền Không Phân Lớp) — là cách biểu diễn địa chỉ IP kèm prefix length (độ dài tiền tố).

**Công thức cơ bản:**
```
/n → 2^(32-n) địa chỉ IP
/24 → 2^8 = 256 IP (thực tế AWS dùng được 251 — trừ 5 địa chỉ reserved)
/26 → 2^6 = 64 IP (59 usable)
/28 → 2^4 = 16 IP (11 usable)
```

**AWS reserved addresses trong mỗi subnet (ví dụ 10.0.1.0/24):**
```
10.0.1.0   — Network address (địa chỉ mạng)
10.0.1.1   — AWS router
10.0.1.2   — AWS DNS
10.0.1.3   — AWS future use
10.0.1.255 — Broadcast address
```

**Thiết kế CIDR cho production VPC:**

```
VPC: 10.0.0.0/16 (65,536 IPs — đủ room để mở rộng)

Chia theo AZ (Availability Zone — Vùng Khả Dụng):
- AZ-a: 10.0.0.0/18   (16,384 IPs)
- AZ-b: 10.0.64.0/18  (16,384 IPs)
- AZ-c: 10.0.128.0/18 (16,384 IPs)
- Reserve: 10.0.192.0/18

Trong mỗi AZ:
- Public subnet:   /24 (256 IPs) — ít tài nguyên cần public IP
- Private subnet:  /20 (4,096 IPs) — nhiều app servers
- Database subnet: /24 (256 IPs) — ít database instances
```

**Best practice quan trọng:**
- Chọn VPC CIDR đủ lớn ngay từ đầu — không dễ thay đổi sau
- Tránh overlap với on-premises network nếu có kế hoạch hybrid connectivity
- Để lại ít nhất 1 CIDR block để expand sau này
- Dùng /16 hoặc /18 cho VPC production, /24 cho development/test

---

### Câu 4: NAT Gateway khác gì với Internet Gateway? Khi nào dùng cái nào?

**Câu trả lời mẫu:**

Đây là câu hỏi rất phổ biến vì nhiều người nhầm lẫn giữa hai khái niệm này.

| Thuộc Tính                  | Internet Gateway (IGW)                                    | NAT Gateway                                                        |
| --------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------ |
| Chức năng                   | Kết nối hai chiều giữa VPC và Internet                    | Cho phép private subnet truy cập Internet một chiều (outbound)     |
| Traffic chiều               | Inbound + Outbound                                        | Chỉ Outbound (Internet không thể initiate kết nối vào)             |
| Vị trí                      | Gắn vào VPC (một IGW cho mỗi VPC)                        | Đặt trong Public Subnet                                            |
| Public IP                   | EC2 cần public/Elastic IP để dùng IGW                    | NAT Gateway có Elastic IP của chính nó                             |
| Scalability (khả năng mở rộng) | Tự động scale, không giới hạn                          | Tự động scale đến 45 Gbps                                          |
| High Availability           | AWS managed, highly available trong AZ                    | HA trong một AZ — cần tạo một NAT GW mỗi AZ cho HA thực sự        |
| Chi phí                     | Miễn phí (trả data transfer)                              | $0.045/giờ + $0.045/GB data processed                              |

**Khi nào dùng gì:**

```
Dùng Internet Gateway khi:
→ Load balancer, bastion host, hoặc EC2 cần public IP
→ Bất kỳ resource nào cần nhận inbound traffic từ Internet

Dùng NAT Gateway khi:
→ App server (trong Private Subnet) cần download packages, updates
→ Lambda function cần gọi external API
→ ECS container cần pull Docker image từ Docker Hub

KHÔNG cần NAT Gateway khi:
→ Dùng VPC Endpoints cho S3/DynamoDB (miễn phí và không qua Internet)
→ Interface Endpoints cho ECR, Secrets Manager, v.v.
```

**Lưu ý chi phí quan trọng:** NAT Gateway là một trong những nguồn chi phí bất ngờ phổ biến nhất trong AWS. Cân nhắc dùng VPC Endpoints (Điểm Cuối VPC) để giảm traffic qua NAT Gateway.

---

## 🔴 Security — Bảo Mật Mạng

---

### Câu 5: Security Group khác gì với Network ACL? Khi nào dùng cái nào?

**Câu trả lời mẫu:**

Đây là câu hỏi cơ bản nhưng cực kỳ quan trọng — interviewer muốn nghe bạn hiểu rõ **stateful vs stateless**.

| Thuộc Tính              | Security Group (Nhóm Bảo Mật)                              | Network ACL (Danh Sách Kiểm Soát Truy Cập Mạng)                     |
| ----------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------- |
| Áp dụng cho             | Network Interface (ENI — Elastic Network Interface) của EC2 | Subnet (Mạng Con)                                                    |
| Stateful/Stateless      | **Stateful** — response traffic tự động được phép          | **Stateless** — phải define cả inbound lẫn outbound rule riêng biệt |
| Rule logic              | Chỉ có Allow rules — không có Deny                         | Có cả Allow và Deny rules (ưu tiên theo số thứ tự)                   |
| Evaluation              | Tất cả rules được evaluate cùng lúc                        | Rules được evaluate theo thứ tự số (thấp nhất trước)                 |
| Default behavior        | Chặn tất cả inbound, cho phép tất cả outbound              | Subnet mặc định: Allow tất cả                                        |
| Scope                   | Gắn với instance/ENI                                       | Gắn với subnet — áp dụng cho mọi resource trong subnet               |

**Ví dụ minh họa stateful vs stateless:**

```
Security Group (Stateful):
- Bạn tạo rule: Allow inbound TCP port 443
- Client gửi request: port 443 → Được phép ✅
- Server gửi response: port 443 → port cao (ephemeral port) → TỰ ĐỘNG được phép ✅

Network ACL (Stateless):
- Bạn tạo rule: Allow inbound TCP port 443
- Client gửi request: port 443 → Được phép ✅
- Server gửi response → Bạn PHẢI tạo outbound rule cho ephemeral ports (1024-65535) ✅
- Nếu quên outbound rule → Response bị BLOCK ❌
```

**Khi nào dùng gì:**

```
Security Group — dùng cho:
→ Kiểm soát access cho từng instance/service cụ thể
→ Áp dụng Defense in depth cho từng layer của application
→ Tạo "virtual firewall" quanh EC2, RDS, Lambda

Network ACL — dùng cho:
→ Block một IP hoặc IP range cụ thể ở subnet level (security incident response)
→ Thêm layer bảo mật bổ sung cho toàn subnet
→ Block traffic từ một subnet ra ngoài (data exfiltration prevention)
```

**Best practice:** Không cần dùng NACLs cho mọi thứ — Security Groups đã đủ cho phần lớn use cases. Chỉ dùng NACLs khi cần explicit deny ở subnet level.

---

### Câu 6: Bạn sẽ thiết kế security model như thế nào cho một ứng dụng web production?

**Câu trả lời mẫu:**

Đây là câu hỏi thiết kế — interviewer muốn thấy tư duy **defense in depth** (phòng thủ theo chiều sâu).

**Kiến trúc Security Layers (Các Lớp Bảo Mật):**

```
Layer 1 — Network Perimeter (Vành Đai Mạng):
├── AWS Shield Standard (Chống DDoS — Distributed Denial of Service tự động)
├── CloudFront với WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web) rules
└── NACLs chặn known-bad IP ranges

Layer 2 — Load Balancer:
├── ALB Security Group: chỉ allow 80/443 từ CloudFront IP ranges
└── SSL/TLS termination với ACM certificate

Layer 3 — Application Tier:
├── EC2/ECS Security Group: chỉ allow traffic từ ALB Security Group
└── Không có public IP — chỉ trong private subnet

Layer 4 — Database Tier:
├── RDS Security Group: chỉ allow port 5432/3306 từ App tier SG
├── Không có internet access (isolated subnet)
└── Encryption at rest + in transit

Layer 5 — AWS Account Level:
├── IAM roles theo Least Privilege
├── AWS Config rules kiểm tra compliance
└── GuardDuty (Phát Hiện Mối Đe Dọa) cho threat detection
```

**Câu hỏi bổ sung hay thêm:** "Nếu phát hiện một IP tấn công, bạn block ở đâu và tại sao?"

Trả lời: Block ở WAF (nhanh nhất, không cần thay đổi infrastructure), sau đó xem xét block ở NACL nếu cần subnet-level block. Security Group không có explicit deny nên không dùng để block.

---

### Câu 7: AWS WAF hoạt động như thế nào? Sự khác biệt giữa WAF rule types?

**Câu trả lời mẫu:**

WAF — Web Application Firewall (Tường Lửa Ứng Dụng Web) — hoạt động ở Layer 7 (Application Layer), kiểm tra nội dung HTTP/HTTPS request trước khi cho phép đến origin.

**WAF Rule Types (Loại Quy Tắc WAF):**

| Loại Rule                        | Mô Tả                                               | Ví Dụ                                      |
| -------------------------------- | --------------------------------------------------- | ------------------------------------------ |
| IP Set Rules                     | Cho phép/chặn dựa trên IP address                   | Block IP từ quốc gia cụ thể               |
| Managed Rule Groups              | Bộ rules được AWS/partners duy trì                  | AWSManagedRulesCommonRuleSet (OWASP Top 10) |
| Rate-based Rules                 | Giới hạn số request từ một IP trong 5 phút          | Max 2000 requests/5 phút                  |
| Regex Pattern Set Rules          | Match dựa trên regular expression trong request     | Block request chứa SQL injection patterns |
| Custom Rules                     | Kết hợp nhiều điều kiện với AND/OR logic            | Block nếu IP ở US VÀ URI chứa /admin      |

**Cách WAF attach vào services:**
```
WAF WebACL (Web Access Control List — Danh Sách Kiểm Soát Truy Cập Web)
    ↕ (attach)
CloudFront Distribution  OR  ALB  OR  API Gateway  OR  AppSync
```

**Best practices cho WAF:**
1. Bắt đầu với **Count mode** (chế độ đếm) — xem rules match gì trước khi block
2. Dùng **AWS Managed Rules** trước — bao phủ OWASP Top 10 mà không cần tự viết
3. Kết hợp với **Shield Advanced** (Bảo Vệ Nâng Cao) cho DDoS protection layer
4. Monitor WAF logs trong **CloudWatch** để detect attack patterns
5. Dùng **rate-based rules** để chặn brute force (tấn công dò mật khẩu)

---

## 🟡 Load Balancing — Cân Bằng Tải

---

### Câu 8: ALB khác gì NLB? Cho ví dụ khi nào dùng mỗi loại?

**Câu trả lời mẫu:**

| Thuộc Tính                            | ALB (Application Load Balancer)                 | NLB (Network Load Balancer)                        |
| ------------------------------------- | ----------------------------------------------- | -------------------------------------------------- |
| OSI Layer                             | Layer 7 (Application)                           | Layer 4 (Transport)                                |
| Protocol                              | HTTP, HTTPS, gRPC, WebSocket                    | TCP, UDP, TLS                                      |
| Routing decisions                     | URL path, hostname, headers, query params       | IP address, port                                   |
| Latency                               | Thấp (~ms)                                      | Cực thấp (~µs — microseconds)                      |
| Static IP                             | Không có static IP (chỉ có DNS name)            | Có static IP per AZ (Availability Zone)            |
| Preserve source IP                    | Dùng X-Forwarded-For header                     | Có thể preserve source IP trực tiếp               |
| SSL Termination                       | Có (tại ALB)                                    | Có TLS pass-through hoặc termination               |
| WebSocket                             | Native support                                  | Hỗ trợ qua TCP                                    |
| Target types                          | Instance, IP, Lambda                            | Instance, IP, ALB                                  |
| Giá                                   | Cao hơn một chút                                | Thấp hơn cho high-throughput workloads             |

**Khi nào dùng ALB:**
```
✅ Web application với path-based routing (/api → api servers, /static → S3)
✅ Microservices với host-based routing (api.example.com, app.example.com)
✅ HTTPS termination với ACM certificate
✅ Container workloads (ECS, EKS) với dynamic port mapping
✅ WebSocket connections (chat apps, real-time dashboards)
✅ gRPC APIs
```

**Khi nào dùng NLB:**
```
✅ High-performance TCP/UDP workloads (gaming servers, IoT, financial trading)
✅ Cần preserve client source IP (cho security logging, geo-based business logic)
✅ Cần static IP (compliance requirements, firewall whitelisting)
✅ NLB làm target của ALB (trong cross-zone patterns)
✅ Private Link — NLB bắt buộc làm endpoint service
✅ Latency-sensitive workloads cần microsecond response
```

**Pattern nâng cao:** NLB có thể làm target của ALB, cho phép bạn kết hợp: ALB xử lý HTTP routing → NLB xử lý TCP với static IP.

---

### Câu 9: Health check trong ALB hoạt động như thế nào? Làm sao tránh false unhealthy?

**Câu trả lời mẫu:**

Health check là cơ chế ALB định kỳ kiểm tra xem target (instance, container, IP) có healthy (khỏe mạnh) không để quyết định có route traffic đến đó không.

**Health check lifecycle:**

```
Target đăng ký vào Target Group
    ↓
ALB gửi HTTP/HTTPS request đến health check path (ví dụ: GET /health)
    ↓
Target trả về 200 OK?
    ├── Có → healthy_threshold (số lần thành công cần thiết, default: 3)
    │         → Sau khi đủ lần → Target được đánh dấu Healthy
    └── Không → unhealthy_threshold (số lần thất bại cần thiết, default: 3)
              → Sau khi đủ lần → Target được đánh dấu Unhealthy → không nhận traffic
```

**Cấu hình quan trọng:**

| Parameter                      | Mô Tả                                          | Gợi Ý Production             |
| ------------------------------ | ---------------------------------------------- | ----------------------------- |
| Health check path              | URL endpoint được check                        | `/health` hoặc `/healthz`     |
| Healthy threshold              | Số lần liên tiếp phải pass để healthy          | 2-3                           |
| Unhealthy threshold            | Số lần liên tiếp phải fail để unhealthy        | 3 (tránh false positives)     |
| Timeout                        | Thời gian chờ response                         | 5-10 giây                     |
| Interval                       | Thời gian giữa các lần check                   | 30 giây (có thể giảm nếu cần) |
| Success codes                  | HTTP status codes được coi là healthy          | `200` hoặc `200-299`          |

**Tránh false unhealthy (cảnh báo sức khỏe sai):**
1. `/health` endpoint chỉ kiểm tra xem app **có thể nhận request không** — không check database
2. Nếu check database trong health endpoint → database outage = toàn bộ instances bị unhealthy → cascade failure
3. Phân biệt **liveness probe** (app đang sống) và **readiness probe** (app sẵn sàng nhận traffic)
4. Cấu hình **deregistration delay** (thời gian chờ trước khi remove, default 300s) đủ lớn để drain connections đang active

---

### Câu 10: Cross-zone load balancing là gì? Khi nào bật, khi nào tắt?

**Câu trả lời mẫu:**

Cross-zone load balancing là tính năng cho phép load balancer phân phối traffic đồng đều đến tất cả registered targets trong **tất cả AZs (Availability Zones — Vùng Khả Dụng)**, không chỉ trong AZ của load balancer node.

**Kịch bản minh họa:**

```
Không có cross-zone LB (Default với NLB):
                          ┌─ AZ-a ─────────────────────┐
Client → LB Node AZ-a →  │  EC2-1 (50% traffic)       │
                          │  EC2-2 (50% traffic)       │
                          └─────────────────────────────┘
                          ┌─ AZ-b ─────────────────────┐
         LB Node AZ-b →  │  EC2-3 (50% traffic)       │
                          └─────────────────────────────┘
→ EC2-1 và EC2-2 mỗi cái chỉ nhận 25% của tổng traffic

Có cross-zone LB (Default với ALB):
Client → LB Nodes → Distribute đều cho EC2-1, EC2-2, EC2-3
→ Mỗi EC2 nhận 33.3% tổng traffic
```

**ALB vs NLB default:**
- **ALB**: Cross-zone LB mặc định **bật** — không tính phí data transfer
- **NLB**: Cross-zone LB mặc định **tắt** — tính phí data transfer giữa AZs nếu bật

**Khi nào bật cross-zone:**
- Khi số lượng instances không đều giữa các AZs (ví dụ: AZ-a có 5 instances, AZ-b có 2 instances)
- Khi muốn phân phối load đồng đều và không quan tâm đến cross-AZ data transfer cost

**Khi nào cân nhắc tắt:**
- NLB với workloads latency-sensitive: cross-AZ call thêm ~1ms nhưng có thể quan trọng
- Khi cost optimization quan trọng và traffic giữa AZs tốn tiền

---

## 🟢 DNS & Route 53

---

### Câu 11: Giải thích tất cả Route 53 routing policies và khi nào dùng mỗi loại?

**Câu trả lời mẫu:**

Route 53 có 7 routing policies (chính sách định tuyến):

**1. Simple Routing (Định Tuyến Đơn Giản):**
```
Dùng khi: Một resource phục vụ tất cả traffic
Ví dụ: Trỏ example.com đến 1 IP của web server
Lưu ý: Nếu có nhiều values → trả về tất cả, client chọn ngẫu nhiên
```

**2. Weighted Routing (Định Tuyến Có Trọng Số):**
```
Dùng khi: A/B testing, canary deployments (triển khai kiểm tra dần)
Ví dụ: 90% traffic → V1, 10% traffic → V2 (new version)
Weight range: 0-255; Weight 0 = không nhận traffic
```

**3. Latency-based Routing (Định Tuyến Dựa Trên Độ Trễ):**
```
Dùng khi: Multi-region, muốn user được route đến region có latency (độ trễ) thấp nhất
Ví dụ: User từ Asia → ap-southeast-1, User từ EU → eu-west-1
Lưu ý: Dựa trên network latency đo được, không phải geographic distance
```

**4. Failover Routing (Định Tuyến Chuyển Đổi Dự Phòng):**
```
Dùng khi: Active/Passive Disaster Recovery (Khôi Phục Sau Thảm Họa)
Ví dụ: Primary → us-east-1, Secondary → us-west-2 (chỉ active khi primary fail)
Yêu cầu: Health check bắt buộc trên primary record
```

**5. Geolocation Routing (Định Tuyến Dựa Trên Vị Trí Địa Lý):**
```
Dùng khi: Regulatory requirements (tuân thủ pháp lý), localization
Ví dụ: User từ EU → EU servers (GDPR data residency), User từ VN → VN content
Lưu ý: Khác Latency — Geolocation dựa vào vị trí thực, không phải performance
```

**6. Geoproximity Routing (Định Tuyến Dựa Trên Khoảng Cách Địa Lý):**
```
Dùng khi: Muốn điều chỉnh vùng phục vụ (expand/shrink) bằng bias values
Ví dụ: Tăng bias của us-east-1 → thu hút thêm traffic từ users gần đó
Yêu cầu: Phải dùng Traffic Flow (tính phí thêm)
```

**7. Multi-value Answer Routing (Định Tuyến Đa Giá Trị):**
```
Dùng khi: Client-side load balancing cơ bản (không phải ELB replacement)
Trả về: Tối đa 8 healthy records ngẫu nhiên
Khác Simple Routing: Có health checks — chỉ trả về healthy records
```

**Bảng tóm tắt nhanh:**

| Policy       | Use Case Chính               | Health Check |
| ------------ | ---------------------------- | ------------ |
| Simple       | 1 endpoint, đơn giản         | Không        |
| Weighted     | A/B testing, canary deploy   | Tùy chọn     |
| Latency      | Multi-region performance     | Tùy chọn     |
| Failover     | Active/Passive DR            | Bắt buộc     |
| Geolocation  | Compliance, localization     | Tùy chọn     |
| Geoproximity | Fine-grained traffic control | Tùy chọn     |
| Multi-value  | Basic client-side LB         | Tùy chọn     |

---

### Câu 12: Sự khác biệt giữa CNAME record và Alias record trong Route 53?

**Câu trả lời mẫu:**

Đây là câu hỏi mà nhiều người trả lời sai — có nhiều điểm khác biệt quan trọng:

| Thuộc Tính           | CNAME Record                                            | Alias Record                                                          |
| -------------------- | ------------------------------------------------------- | --------------------------------------------------------------------- |
| Standard?            | DNS chuẩn (RFC 1034)                                    | Route 53 extension (không phải DNS chuẩn)                             |
| Dùng ở root domain?  | **KHÔNG** — không thể dùng cho zone apex (example.com) | **CÓ** — có thể dùng cho zone apex (example.com)                     |
| Point to            | Bất kỳ hostname nào                                     | Chỉ AWS resources (ALB, CloudFront, S3, API Gateway, v.v.)            |
| Billing             | Tính phí mỗi DNS query                                  | **Miễn phí** khi point đến AWS resources                             |
| TTL                  | Bạn cấu hình                                            | AWS tự quản lý (thường thấp hơn)                                     |
| IP thay đổi          | Client phải resolve thêm CNAME                         | Route 53 tự động resolve và trả về IP hiện tại                       |

**Ví dụ thực tế:**

```
❌ KHÔNG hợp lệ:
example.com  CNAME  myalb-123.us-east-1.elb.amazonaws.com  (zone apex!)

✅ Hợp lệ:
example.com  ALIAS  myalb-123.us-east-1.elb.amazonaws.com  (zone apex + AWS resource)
www.example.com  CNAME  myalb-123.us-east-1.elb.amazonaws.com  (subdomain OK)
```

**Khi nào dùng Alias:**
- Trỏ root domain đến ALB, CloudFront, S3 static website, API Gateway
- Tiết kiệm chi phí DNS queries (Alias đến AWS resources là free)

**Khi nào dùng CNAME:**
- Trỏ subdomain đến hostname không phải AWS resource (ví dụ: third-party services)
- `www.example.com` → `example.com` (rồi dùng Alias cho example.com)

---

### Câu 13: Giải thích Route 53 Private Hosted Zone và use case?

**Câu trả lời mẫu:**

Private Hosted Zone (Vùng Lưu Trữ Riêng Tư) cho phép bạn resolve DNS names chỉ **trong phạm vi VPC** — không accessible từ Internet.

**So sánh Public vs Private Hosted Zone:**

| Thuộc Tính          | Public Hosted Zone                     | Private Hosted Zone                           |
| ------------------- | -------------------------------------- | --------------------------------------------- |
| Resolve từ          | Internet (bất kỳ đâu)                 | Chỉ từ trong VPC được associate               |
| Use case            | Domain name cho public website/API     | Service discovery nội bộ, internal DNS        |
| Security            | Public — ai cũng resolve được         | Private — chỉ resources trong VPC biết        |
| Billing             | $0.50/hosted zone/tháng               | $0.50/hosted zone/tháng                       |

**Use cases phổ biến:**

```
1. Internal Service Discovery (Khám Phá Dịch Vụ Nội Bộ):
   database.internal → 10.0.3.45 (RDS private IP)
   cache.internal → 10.0.4.23 (ElastiCache private IP)
   
   → Khi RDS thay thế, chỉ update DNS — không cần update app config

2. Split-horizon DNS (DNS Phân Tách Theo Ngữ Cảnh):
   api.example.com từ Internet → ALB public IP (203.0.113.10)
   api.example.com từ trong VPC → ALB private IP (10.0.1.45) (tránh hairpinning)

3. Hybrid connectivity:
   Resources on-premises resolve internal.company.com
   qua Route 53 Resolver Inbound Endpoint
```

**Cấu hình quan trọng:** Để Private Hosted Zone hoạt động, VPC phải có:
- `enableDnsHostnames: true`
- `enableDnsSupport: true`

---

## 🟠 Connectivity Hybrid — Kết Nối Hybrid

---

### Câu 14: Direct Connect khác gì VPN Site-to-Site? Khi nào dùng cái nào?

**Câu trả lời mẫu:**

| Thuộc Tính                   | AWS Direct Connect (Kết Nối Trực Tiếp)          | VPN Site-to-Site (VPN Địa Điểm-Địa Điểm)        |
| ---------------------------- | ----------------------------------------------- | ------------------------------------------------ |
| Connection type              | Dedicated physical line (đường vật lý riêng)    | Encrypted tunnel qua Internet (đường hầm mã hóa) |
| Bandwidth                    | 1 Gbps đến 100 Gbps (dedicated)                 | Tối đa ~1.25 Gbps (chia sẻ Internet)            |
| Latency                      | Thấp & nhất quán (consistent)                   | Cao hơn và biến động (variable)                  |
| Security                     | Không mã hóa mặc định (physical isolation)      | Mã hóa AES-256 (IPSec)                          |
| Reliability                  | Cao (SLA 99.99% với redundant connections)      | Phụ thuộc Internet ISP                           |
| Setup time                   | Hàng tuần đến hàng tháng                        | Vài giờ                                         |
| Cost                         | Đắt ($0.03/GB + port hourly fee)                | Rẻ hơn ($0.05/hour + $0.09/GB)                  |
| Use case                     | Production, large data transfers, compliance    | Backup, development, quick hybrid setup          |

**Khi nào dùng Direct Connect:**
```
✅ Cần bandwidth lớn (hàng TB data/tháng)
✅ Latency nhất quán là yêu cầu (financial trading, real-time apps)
✅ Compliance yêu cầu private connection (HIPAA, PCI-DSS)
✅ Long-term hybrid cloud strategy (chiến lược đám mây lai dài hạn)
✅ Migration: chuyển dữ liệu lớn từ on-premises lên AWS
```

**Khi nào dùng VPN Site-to-Site:**
```
✅ Cần kết nối nhanh, không có thời gian chờ Direct Connect provisioning
✅ Budget thấp, traffic vừa phải
✅ Backup/failover cho Direct Connect
✅ Development/staging environments
✅ Temporary connections cho projects ngắn hạn
```

**Best practice — kết hợp cả hai:**
```
Primary:  Direct Connect (bandwidth, latency)
Backup:   VPN Site-to-Site (automatic failover khi DX down)
```

---

### Câu 15: Transit Gateway giải quyết vấn đề gì mà VPC Peering không làm được?

**Câu trả lời mẫu:**

**VPC Peering (Kết Nối Ngang Hàng VPC) — Hạn Chế:**

```
Vấn đề 1: Non-transitive (Không Chuyển Tiếp)
VPC-A ↔ VPC-B ↔ VPC-C
→ VPC-A KHÔNG thể nói chuyện với VPC-C qua VPC-B
→ Phải tạo peering trực tiếp A↔C

Vấn đề 2: N*(N-1)/2 connections — không scalable
- 10 VPCs = 45 peering connections
- 50 VPCs = 1,225 peering connections
- Route tables trở nên phức tạp và khó quản lý

Vấn đề 3: Không hỗ trợ kết nối với on-premises
- VPN/Direct Connect phải attach vào từng VPC riêng lẻ
- Không có centralized connectivity
```

**Transit Gateway (Cổng Trung Chuyển) — Giải Pháp:**

```
Hub-and-Spoke Architecture (Kiến Trúc Trung Tâm và Nhánh):

VPC-A ─┐
VPC-B ─┤
VPC-C ─┼─→ Transit Gateway ←─ On-premises (qua VPN hoặc DX)
VPC-D ─┤
VPC-E ─┘

→ Mỗi VPC chỉ cần 1 attachment
→ TGW xử lý routing giữa tất cả
→ Route domains (bảng định tuyến TGW) để kiểm soát ai nói chuyện với ai
```

**Tính năng nổi bật của Transit Gateway:**

| Tính Năng                    | Mô Tả                                                         |
| ---------------------------- | ------------------------------------------------------------- |
| Route Domains                | Phân chia VPCs thành groups — Production, Dev, Shared không cross talk |
| Multicast                    | Hỗ trợ multicast traffic                                      |
| Multi-Region                 | TGW Peering giữa regions                                      |
| Network Manager              | Centralized network visibility                                |
| Bandwidth                    | 50 Gbps per VPC attachment                                    |

**Khi nào dùng VPC Peering vs Transit Gateway:**

```
VPC Peering:
✅ Số lượng VPCs nhỏ (< 5)
✅ Cost-sensitive (VPC Peering không tính phí attachment)
✅ Không cần on-premises connectivity

Transit Gateway:
✅ Nhiều VPCs (> 5)
✅ Cần kết nối on-premises tập trung
✅ Cần fine-grained routing control giữa các groups
```

---

### Câu 16: PrivateLink khác gì VPC Endpoints như thế nào?

**Câu trả lời mẫu:**

Đây là một câu hỏi gây nhầm lẫn vì PrivateLink và VPC Endpoints liên quan chặt chẽ với nhau.

**Mối quan hệ:** AWS PrivateLink là **công nghệ nền** (underlying technology), VPC Endpoints là **cách để sử dụng** PrivateLink.

**VPC Endpoints có 2 loại:**

```
1. Gateway Endpoints (Điểm Cuối Gateway):
   - Chỉ cho: S3 và DynamoDB
   - Hoạt động: Thêm route vào route table
   - Miễn phí hoàn toàn
   - Không dùng PrivateLink

2. Interface Endpoints (Điểm Cuối Interface):
   - Cho: 100+ AWS services (EC2 API, SSM, Secrets Manager, ECR, ...)
   - Hoạt động: Tạo ENI với private IP trong VPC
   - Tính phí: ~$0.01/hour + $0.01/GB
   - Dùng PrivateLink technology
```

**AWS PrivateLink — dùng để tạo Endpoint Services:**

```
Bạn có thể tạo PrivateLink Endpoint Service khi:
→ Muốn expose service của bạn cho các AWS accounts khác
→ Mà KHÔNG cần VPC Peering hoặc Internet

Ví dụ: SaaS vendor expose API qua PrivateLink
Customer VPC → Interface Endpoint → PrivateLink → Vendor NLB → Vendor service
```

**Tóm tắt:**

| | Gateway Endpoint | Interface Endpoint | Custom PrivateLink |
|---|---|---|---|
| AWS Services | S3, DynamoDB | 100+ AWS services | Service của bạn |
| Cost | Miễn phí | Có phí | Có phí |
| Technology | Route table | PrivateLink + ENI | PrivateLink + NLB |

---

## 🔴 Advanced & System Design

---

### Câu 17: Thiết kế kiến trúc multi-region active-active cho e-commerce?

**Câu trả lời mẫu:**

*Đây là câu hỏi system design — interviewer muốn thấy tư duy toàn diện, không chỉ biết dịch vụ.*

**Requirements đặt ra trước:**
```
- Traffic: Global, 100K concurrent users
- RTO (Recovery Time Objective — Thời Gian Phục Hồi Mục Tiêu): < 1 phút
- RPO (Recovery Point Objective — Điểm Phục Hồi Mục Tiêu): < 5 giây
- Availability (Độ Sẵn Sàng): 99.99%
- Regions: us-east-1, eu-west-1, ap-southeast-1
```

**Kiến trúc:**

```
                    ┌── Route 53 ──────────────────────────────────┐
                    │  Latency-based routing                       │
                    │  + Health checks per region                  │
                    └──────────┬───────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ↓                ↓                 ↓
        us-east-1         eu-west-1       ap-southeast-1
              │                │                 │
    ┌─────────┴────────┐  ┌────┴──────┐  ┌──────┴────────┐
    │ CloudFront       │  │ CloudFront│  │ CloudFront    │
    │ (Edge caching)   │  │           │  │               │
    │       ↓          │  │     ↓     │  │       ↓       │
    │ ALB → ECS/EKS   │  │ ALB → ECS│  │ ALB → ECS    │
    │       ↓          │  │     ↓     │  │       ↓       │
    │ Aurora Global DB │══╪═══════════╪══│ Aurora Global │
    │ (Write primary)  │  │  (Read    │  │ (Read replica)│
    └──────────────────┘  │  replica) │  └───────────────┘
                          └───────────┘
```

**Các thành phần chính:**

1. **Route 53 Latency + Health Checks**: Route users đến region gần nhất + automatic failover
2. **CloudFront**: Cache static assets tại edge locations, giảm latency toàn cầu
3. **ALB + ECS/EKS**: Application tier stateless trong mỗi region
4. **Aurora Global Database**: Primary write region + read replicas, replication < 1 giây
5. **ElastiCache Global Datastore**: Session và cache data sync across regions

**Trade-offs cần thảo luận:**
- **Active-active vs Active-passive**: Active-active cần xử lý distributed transactions (giao dịch phân tán) — phức tạp hơn
- **Data consistency (nhất quán dữ liệu)**: Có thể có eventual consistency cho cart, read-after-write cho orders
- **Cost**: 3x infrastructure cost — cần justify bằng business requirements

---

### Câu 18: Làm sao bảo vệ S3 bucket chỉ cho phép truy cập qua CloudFront?

**Câu trả lời mẫu:**

Đây là use case phổ biến cho static website hoặc private content delivery.

**Solution sử dụng OAC (Origin Access Control — Kiểm Soát Truy Cập Nguồn Gốc) — cách mới nhất:**

**Bước 1: Tạo CloudFront OAC**
```json
{
  "Name": "my-oac",
  "OriginAccessControlOriginType": "s3",
  "SigningBehavior": "always",
  "SigningProtocol": "sigv4"
}
```

**Bước 2: Cấu hình CloudFront Distribution**
```
Origin: my-bucket.s3.amazonaws.com
Origin Access: OAC (không phải OAI — Origin Access Identity — cũ hơn)
```

**Bước 3: Update S3 Bucket Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DISTRIBUTION_ID"
      }
    }
  }]
}
```

**Bước 4: Block Public Access trên S3 Bucket**
```
✅ Block all public access = ON
→ Giờ S3 bucket chỉ accessible qua CloudFront
```

**OAC vs OAI (Origin Access Identity — Danh Tính Truy Cập Nguồn Gốc):**
- **OAI** (legacy): IAM principal nhưng không support AWS KMS encryption, server-side encryption
- **OAC** (recommended): Hỗ trợ KMS-encrypted S3, server-side encryption, S3 Object Lambda

---

### Câu 19: Giải thích cách VPC Flow Logs giúp debug kết nối?

**Câu trả lời mẫu:**

VPC Flow Logs (Nhật Ký Luồng VPC) capture metadata về IP traffic đi qua VPC — không capture nội dung packet, chỉ capture header information.

**Format của một Flow Log record:**

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status

Ví dụ:
2 123456789012 eni-abc123 10.0.1.5 10.0.2.45 54321 443 6 10 1500 1620000000 1620000060 ACCEPT OK
2 123456789012 eni-abc123 10.0.1.5 10.0.3.45 54322 5432 6 0 0 1620000000 1620000060 REJECT OK
```

**Debug scenarios thực tế:**

**Scenario 1: EC2 không kết nối được database**
```
Query Flow Logs cho traffic từ EC2 đến RDS port 5432:
- Thấy REJECT trong Flow Logs → Problem: Security Group hoặc NACL
- Không thấy record nào → Problem: Route table hoặc EC2 chưa gửi request
- Thấy ACCEPT nhưng vẫn không kết nối → Problem: Application level (sai password, sai endpoint)
```

**Scenario 2: Phát hiện port scanning**
```
Filter: REJECT action + nhiều srcport khác nhau từ cùng srcaddr
→ IP đó đang scan ports
→ Action: Block ở WAF hoặc NACL
```

**Scenario 3: Unexpected traffic**
```
Filter: traffic đến port bất thường (ví dụ: port 4444 — thường dùng bởi malware)
→ Possible compromise (xâm phạm bảo mật)
→ Investigate ngay
```

**Nơi lưu Flow Logs:**
```
CloudWatch Logs → Dùng Insights query, real-time alerting
S3 → Dùng Athena để query, lưu trữ lâu dài, chi phí thấp
    → Bật Hive-compatible partitions để query nhanh hơn
```

**Athena query ví dụ — top rejected connections:**
```sql
SELECT srcaddr, dstport, COUNT(*) as reject_count
FROM vpc_flow_logs
WHERE action = 'REJECT'
  AND start BETWEEN 1620000000 AND 1620086400
GROUP BY srcaddr, dstport
ORDER BY reject_count DESC
LIMIT 20;
```

---

### Câu 20: Câu hỏi tổng hợp — Thiết kế network cho startup fintech cần PCI-DSS compliance?

**Câu trả lời mẫu:**

*Đây là câu hỏi senior-level — kết hợp kiến thức kỹ thuật với business/compliance requirements.*

**PCI-DSS (Payment Card Industry Data Security Standard — Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán) requirements liên quan đến network:**

```
Requirement 1: Install and maintain network security controls
Requirement 2: Apply secure configurations to all system components
Requirement 3: Protect stored account data (encryption)
Requirement 4: Protect cardholder data with strong cryptography (in transit)
```

**Kiến trúc đề xuất:**

```
Internet
    ↓
[AWS Shield Advanced] → DDoS protection, SLA-backed
    ↓
[CloudFront + WAF]    → Edge security, OWASP rules, rate limiting
    ↓
[ALB] — HTTPS only, TLS 1.2+, ACM certificate
    ↓
[Public Subnet]
    ├── Application tier (ECS/EC2 in Private Subnet)
    │       ↓
    │   [CDE — Cardholder Data Environment — Môi Trường Dữ Liệu Chủ Thẻ]
    │   [Isolated Subnet] — Network segmentation bắt buộc theo PCI-DSS
    │       ├── Payment processing service
    │       ├── Token vault (không lưu raw card data)
    │       └── Audit logging service
    │
    └── [Database Subnet] — RDS với encryption at rest (KMS)
                         — Encryption in transit bắt buộc

Outbound từ CDE:
→ VPC Endpoint cho AWS services (không qua Internet)
→ NAT Gateway cho payment gateway API (cần whitelist IPs)
→ Private Direct Connect cho bank connections (không qua Internet)
```

**Security controls bắt buộc:**

```
✅ Segment CDE (Cardholder Data Environment) trong isolated subnet
✅ Security Groups: Principle of Least Privilege — chỉ allow exactly what's needed
✅ NACLs: Additional layer, explicit deny rules
✅ VPC Flow Logs: Tất cả traffic, lưu ≥ 12 tháng (PCI requirement)
✅ CloudTrail: All API calls
✅ AWS Config: Detect configuration drift
✅ GuardDuty: Threat detection
✅ Shield Advanced: DDoS protection (Requirement 6.4)
✅ WAF: SQL injection, XSS protection (Requirement 6.3)
✅ All data encrypted: TLS 1.2+ in transit, AES-256 at rest
✅ No cardholder data stored if possible — use tokenization (mã hóa thẻ)
```

**Key trade-off cần thảo luận:** Compliance increases cost và complexity. Suggest dùng payment processor như Stripe để họ handle PCI scope — giảm compliance burden đáng kể.

---

## 📚 Câu Hỏi Bổ Sung Theo Cấp Độ

### Junior Level (0-2 năm)

- Giải thích DHCP và cách EC2 nhận IP address
- Subnet mask và wildcard mask khác nhau như thế nào?
- Elastic IP là gì? Khi nào cần dùng?
- Khi nào dùng Application Load Balancer thay vì chỉ Route 53?

### Mid Level (2-5 năm)

- Bạn sẽ debug như thế nào khi ECS container không pull được Docker image từ ECR?
- Giải thích Asymmetric Routing (Định Tuyến Không Đối Xứng) và vấn đề gây ra
- VPN tunnel BGP (Border Gateway Protocol) vs static routing — khi nào dùng BGP?
- Giải thích connection draining trong ALB target groups

### Senior Level (5+ năm)

- Thiết kế network architecture cho 500 VPCs across 3 AWS accounts
- Làm sao enforce network policies centrally không cần thay đổi từng VPC?
- Giải thích BFD (Bidirectional Forwarding Detection) trong Direct Connect
- How would you approach migrating from legacy on-premises network to AWS?

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
