# 🌟 STAR Stories — Câu Chuyện Incident Theo Phương Pháp STAR

> STAR là phương pháp kể chuyện kinh nghiệm hiệu quả nhất trong phỏng vấn kỹ thuật. Phần này cung cấp các template (mẫu) và ví dụ về câu chuyện thực tế liên quan đến AWS Networking.

---

## 📖 Phương Pháp STAR

```
S — Situation (Tình Huống): Bối cảnh và context
T — Task (Nhiệm Vụ): Yêu cầu và trách nhiệm của bạn  
A — Action (Hành Động): Cụ thể bạn đã làm gì
R — Result (Kết Quả): Kết quả đo lường được
```

**Nguyên tắc vàng:**
- **Situation**: Ngắn gọn — 2-3 câu. Đừng kể lể background
- **Task**: Rõ ràng vai trò của bạn. Không phải "team làm" mà "tôi chịu trách nhiệm gì"
- **Action**: Đây là phần quan trọng nhất — ít nhất 60% thời gian kể. Chi tiết cụ thể
- **Result**: Số liệu thực tế. "Cải thiện hiệu suất" không đủ — "giảm latency từ 800ms xuống 120ms" mới tốt

---

## 📋 Danh Sách Câu Hỏi Cần Chuẩn Bị Stories

Interviewer thường hỏi:

1. "Kể về một lần bạn giải quyết một network outage nghiêm trọng"
2. "Kể về một lần bạn cải thiện performance của hệ thống"
3. "Kể về một lần bạn phát hiện và xử lý security incident"
4. "Kể về một kiến trúc mạng phức tạp bạn đã thiết kế"
5. "Kể về một lần bạn đưa ra quyết định kỹ thuật khó khăn với trade-offs"

---

## 🔴 STAR Story 1: Network Outage — Mất Kết Nối Toàn Phần

### Kịch bản: Sự cố do thay đổi NACL gây mất kết nối database

---

**S — Situation (Tình Huống):**

> "Vào 2 giờ sáng một thứ Sáu, tôi nhận được PagerDuty alert rằng toàn bộ API endpoints của hệ thống e-commerce đang trả về lỗi 502 Bad Gateway. Đây là hệ thống xử lý 50,000 đơn hàng/ngày — mỗi phút downtime (ngừng hoạt động) có thể gây thiệt hại khoảng $15,000 doanh thu."

**T — Task (Nhiệm Vụ):**

> "Với tư cách là on-call engineer (kỹ sư trực), tôi chịu trách nhiệm chẩn đoán và khôi phục hệ thống trong thời gian ngắn nhất có thể, đồng thời liên lạc với stakeholders về tiến độ."

**A — Action (Hành Động):**

> "Tôi bắt đầu theo quy trình debug có hệ thống:
>
> **Bước 1 — Xác định scope (phạm vi) sự cố (3 phút đầu tiên):**
> - Kiểm tra CloudWatch Metrics cho ALB: HealthyHostCount = 0 → toàn bộ targets unhealthy
> - Kiểm tra ECS service logs: ứng dụng đang chạy nhưng không kết nối được database
> - Kiểm tra RDS: database vẫn running, connections available
>
> **Bước 2 — Kiểm tra network path (đường dẫn mạng):**
> - Dùng EC2 Instance Connect để SSH vào bastion host, test telnet đến RDS port 5432 → Connection refused
> - Kiểm tra Security Groups: rules đúng (app SG allow → RDS SG)
> - Kiểm tra Route Tables: routing đúng giữa private subnets
>
> **Bước 3 — Phát hiện nguyên nhân gốc rễ:**
> - Kiểm tra VPC Flow Logs trong CloudWatch Logs Insights:
>   ```
>   fields @timestamp, srcaddr, dstaddr, dstport, action
>   | filter action = 'REJECT' and dstport = 5432
>   | limit 20
>   ```
> - Kết quả: Toàn bộ traffic đến port 5432 bị REJECT
> - So sánh với lịch sử: Phát hiện NACL (Network ACL) của database subnet vừa được thay đổi lúc 1:50 AM
> - CloudTrail (Nhật Ký API) confirm: một engineer khác vô tình xóa outbound ephemeral port rules (1024-65535) khi troubleshooting việc khác
>
> **Bước 4 — Khắc phục:**
> - Restore NACL rules: thêm lại outbound rule allow TCP 1024-65535
> - Verify: test lại telnet → thành công
> - Monitor ALB HealthyHostCount → tăng từ 0 lên 6 trong 2 phút

**R — Result (Kết Quả):**

> "Tổng thời gian từ alert đến recovery: **22 phút**. Hệ thống hoàn toàn phục hồi với 0 data loss. Sau incident, tôi đề xuất và implement:
> - **AWS Config rule** kiểm tra NACL changes và alert ngay lập tức
> - **Change management process**: Mọi NACL/Security Group changes phải qua peer review trước khi apply production
> - **Runbook** (tài liệu quy trình) cho database connectivity issues được thêm vào wiki team
>
> Incident giúp team reduce MTTR (Mean Time To Recovery — Thời Gian Phục Hồi Trung Bình) cho loại sự cố tương tự từ giờ xuống còn dưới 15 phút."

---

**Câu hỏi follow-up thường gặp và cách trả lời:**

- *"Tại sao NACL lại gây vấn đề trong khi Security Group không?"*
  → Giải thích stateful vs stateless: Security Group tự động cho phép response traffic, NACL thì không
- *"Bạn sẽ làm gì để ngăn incident này xảy ra lại?"*
  → Đã nói trong phần Result — cho thấy bạn học được từ incident

---

## 🟡 STAR Story 2: Performance Optimization — Tối Ưu Hiệu Suất

### Kịch bản: Giảm latency bằng cách tối ưu CloudFront và Route 53

---

**S — Situation (Tình Huống):**

> "Ứng dụng SaaS (Software as a Service — Phần Mềm Dưới Dạng Dịch Vụ) của công ty đang có users ở Southeast Asia báo cáo trang web load chậm — trung bình 4-6 giây cho first contentful paint. Users ở US thì chỉ mất 1.2 giây. Điều này gây churn rate (tỷ lệ rời bỏ) cao ở thị trường Southeast Asia, ảnh hưởng đến revenue target Q3."

**T — Task (Nhiệm Vụ):**

> "Tôi được assign làm tech lead cho performance improvement project với mục tiêu: giảm load time ở Southeast Asia xuống dưới 2 giây mà không tăng infrastructure cost quá 20%."

**A — Action (Hành Động):**

> "**Bước 1 — Đo lường baseline và phân tích:**
> - Dùng CloudWatch RUM (Real User Monitoring — Giám Sát Người Dùng Thực) để đo TTFB (Time To First Byte — Thời Gian Đến Byte Đầu Tiên) từ các regions
> - Kết quả: Users ở Singapore có TTFB = 380ms, users ở US = 45ms
> - Phân tích: Toàn bộ traffic đang route đến single region us-east-1 — thiếu geographic distribution
>
> **Bước 2 — Phân tích CloudFront cache metrics:**
> - CloudFront cache hit ratio: chỉ 42% — quá thấp
> - Nguyên nhân: Cache key bao gồm nhiều query parameters không cần thiết
> - Nhiều API calls không cacheable vì không có proper Cache-Control headers
>
> **Bước 3 — Implement giải pháp theo thứ tự ưu tiên:**
>
> *Giải pháp A — CloudFront Optimization (không tốn thêm tiền):*
> - Tạo Cache Policy mới: chỉ include query params thực sự affect response
> - Thêm Cache-Control headers cho static assets: `max-age=31536000, immutable`
> - Bật CloudFront compression (Brotli + Gzip)
> - Kết quả ngay lập tức: Cache hit ratio tăng từ 42% lên 78%
>
> *Giải pháp B — Route 53 Latency-Based Routing:*
> - Deploy application stack sang ap-southeast-1 (Singapore)
> - Cấu hình Route 53 latency-based routing: Users ở Asia → ap-southeast-1
> - Kết hợp health checks để automatic failover về us-east-1 nếu Singapore down
>
> *Giải pháp C — CloudFront Price Class Update:*
> - Đổi từ PriceClass_100 (US/EU only) sang PriceClass_200 (+ Asia edge locations)
> - Tăng $45/tháng nhưng serve content từ edge locations gần hơn

**R — Kết Quả:**

> "Sau 3 tuần implement:
> - **TTFB cho Southeast Asia**: 380ms → **85ms** (giảm 78%)
> - **First Contentful Paint**: 4.6s → **1.8s** (đạt target < 2s)
> - **Cache hit ratio**: 42% → **83%** (giảm origin load 69%)
> - **Infrastructure cost tăng**: chỉ **8%** (dưới mức cho phép 20%)
> - **Business impact**: Churn rate ở Southeast Asia giảm 23% trong tháng tiếp theo"

---

## 🟢 STAR Story 3: Security Incident — Xử Lý Tấn Công DDoS

### Kịch bản: Phát hiện và ngăn chặn DDoS attack bằng WAF + Shield

---

**S — Situation (Tình Huống):**

> "Trong đợt sale event Black Friday, hệ thống thanh toán của công ty bắt đầu nhận lượng traffic tăng đột biến bất thường vào 9AM — gấp 50x traffic bình thường. CloudWatch alarms kích hoạt khi ALB request count vượt threshold. Đây có thể là DDoS (Distributed Denial of Service — Tấn Công Từ Chối Dịch Vụ Phân Tán) attack có chủ đích."

**T — Task (Nhiệm Vụ):**

> "Với tư cách là security engineer on-call, tôi cần xác định có phải attack thật không, nếu có thì ngăn chặn trong khi đảm bảo real users vẫn truy cập được."

**A — Action (Hành Động):**

> "**Bước 1 — Phân biệt legitimate traffic vs attack traffic:**
> - Phân tích CloudFront access logs: 73% requests đến từ 850 IPs trong AS (Autonomous System) của một cloud provider, không phải residential IPs
> - Request pattern: cùng User-Agent, cùng Accept-Language header, cùng query parameters → bot traffic rõ ràng
> - Requests target endpoint `/api/checkout` với tốc độ 120,000 requests/phút từ các IPs này
>
> **Bước 2 — Activate AWS Shield Advanced (bật bảo vệ nâng cao):**
> - Escalate lên AWS Support để engage Shield Response Team (SRT)
> - SRT confirm: đây là HTTP flood attack, không phải volumetric
>
> **Bước 3 — Deploy WAF rules theo thứ tự (từ ít aggressive đến nhiều aggressive):**
>
> *Rule 1 — Rate-based block (5 phút để propagate):*
> ```
> Rate limit: 2000 requests per 5 minutes per IP
> Action: Block
> ```
> → Ngay lập tức giảm 40% attack traffic nhưng attacker rotate IPs nhanh
>
> *Rule 2 — IP Set block (phân tích ongoing):*
> - Build IP set từ attack IPs: 2,800 IPs thuộc 3 ASNs
> - Deploy IP set block rule
> → Giảm thêm 35% attack traffic
>
> *Rule 3 — Bot Control Managed Rule (cuối cùng):*
> - Bật AWS Bot Control rule group
> - Challenge suspicious bot traffic với CAPTCHA
> → Hầu hết attack traffic bị chặn, real users vẫn pass
>
> **Bước 4 — Monitor và iterate:**
> - Attacker thay đổi chiến thuật sau 30 phút — thêm random delays
> - Detect qua anomaly trong request timing patterns
> - Thêm rule: block User-Agent strings chứa known bot signatures"

**R — Kết Quả:**

> "**Trong vòng 90 phút**, attack traffic giảm từ 120,000 req/phút xuống còn 1,200 req/phút (99% blocked). Legitimate user traffic không bị ảnh hưởng đáng kể — checkout success rate chỉ giảm 0.3% so với dự kiến. 
>
> **Financial impact được ngăn chặn:** Ước tính $280,000 doanh thu bị mất nếu hệ thống down trong Black Friday window.
>
> **Lessons learned (bài học):**
> - Implement WAF rules trước sự kiện lớn, không chờ incident
> - AWS Shield Advanced với SRT access là must-have cho e-commerce
> - Bot Control nên là default — chi phí $10/tháng/1M requests không đáng kể so với risk"

---

## 🔵 STAR Story 4: Architecture Design — Thiết Kế Kiến Trúc Mạng Phức Tạp

### Kịch bản: Migrate 40 VPCs về Transit Gateway hub-and-spoke

---

**S — Situation (Tình Huống):**

> "Công ty đã phát triển qua 5 năm với kiến trúc mạng không có kế hoạch: 40 VPCs across 3 AWS accounts, kết nối với nhau bằng VPC Peering. Kết quả là 780 peering connections, route tables với hàng trăm entries, và không ai hiểu rõ toàn bộ network topology. Security audit phát hiện một số VPCs có thể communicate (giao tiếp) với nhau theo những cách không mong muốn do route table conflicts (xung đột bảng định tuyến)."

**T — Task (Nhiệm Vụ):**

> "Tôi được assign lead một network modernization project: migrate toàn bộ từ peering mesh (lưới kết nối ngang hàng) sang Transit Gateway hub-and-spoke architecture — mà không gây bất kỳ downtime nào cho production workloads."

**A — Action (Hành Động):**

> "**Phase 1 — Khám phá và lập kế hoạch (2 tuần):**
> - Dùng AWS Network Access Analyzer (Bộ Phân Tích Truy Cập Mạng) để visualize toàn bộ network topology hiện tại
> - Phân loại VPCs thành: Production, Staging, Development, Shared Services
> - Thiết kế Transit Gateway route domains (miền định tuyến):
>   - `prod-rt`: Production VPCs — có thể reach Shared Services
>   - `nonprod-rt`: Dev/Staging — không thể reach Production
>   - `shared-rt`: Shared Services — có thể reach mọi domain
>   - `inspection-rt`: Tất cả inter-VPC traffic đi qua Network Firewall
>
> **Phase 2 — Setup Transit Gateway (không ảnh hưởng existing traffic):**
> - Deploy Transit Gateway trong Networking account
> - Share TGW với tất cả accounts qua AWS RAM (Resource Access Manager)
> - Configure route tables theo design
>
> **Phase 3 — Migrate từng VPC (không downtime — kỹ thuật quan trọng):**
> - Với mỗi VPC, thực hiện:
>   1. Tạo TGW attachment (kết nối TGW) — chưa routing
>   2. Test connectivity qua TGW attachment trong staging environment
>   3. Gradually migrate routes: thêm route trong TGW trước, giữ peering
>   4. Monitor traffic shift qua Flow Logs trong 24 giờ
>   5. Sau khi confirm traffic đã shift → xóa peering connections
> - Migrate 2-3 VPCs per week để kiểm soát risk
>
> **Phase 4 — Add Network Firewall (Tường Lửa Mạng) cho inspection:**
> - Deploy AWS Network Firewall trong dedicated inspection VPC
> - Route all inter-VPC traffic qua firewall cho east-west traffic inspection
> - Configure Suricata-compatible rules cho threat detection"

**R — Kết Quả:**

> "Migration hoàn thành sau 4 tháng:
> - **Zero downtime** trong toàn bộ quá trình
> - Route table entries giảm từ trung bình 200 entries mỗi VPC xuống còn **8 entries**
> - VPC Peering connections: từ **780 → 0**
> - Operational overhead: Network changes giờ chỉ cần update TGW route tables — không còn touch từng VPC
> - Security posture cải thiện: Network Firewall detect và block 340 suspicious inter-VPC connections trong tuần đầu tiên — những connections này tồn tại từ trước mà không ai biết
> - Cost: Transit Gateway cost $3,200/tháng nhưng tiết kiệm ~$800/tháng data transfer cost từ peering optimization → net cost tăng $2,400/tháng nhưng justified bởi operational improvements"

---

## 🟣 STAR Story 5: Cost Optimization — Tối Ưu Chi Phí Mạng

### Kịch bản: Giảm chi phí NAT Gateway 73% với VPC Endpoints

---

**S — Situation (Tình Huống):**

> "Trong quarterly cost review (rà soát chi phí hàng quý), tôi phát hiện AWS bill của team tăng 45% trong 3 tháng. Phân tích chi tiết cho thấy NAT Gateway data processing cost chiếm $18,000/tháng trong tổng bill $40,000 — cao bất thường so với expected traffic."

**T — Task (Nhiệm Vụ):**

> "Phân tích nguyên nhân NAT Gateway cost cao và reduce xuống dưới $6,000/tháng mà không ảnh hưởng functionality."

**A — Action (Hành Động):**

> "**Bước 1 — Phân tích traffic patterns qua NAT Gateway:**
> - Enable detailed NAT Gateway metrics trong CloudWatch
> - Dùng VPC Flow Logs với Athena để query top destination IPs:
>   ```sql
>   SELECT dstaddr, COUNT(*) as connections, SUM(bytes)/1e9 as gb_transferred
>   FROM vpc_flow_logs
>   WHERE srcaddr LIKE '10.%' AND action = 'ACCEPT'
>   GROUP BY dstaddr
>   ORDER BY gb_transferred DESC
>   LIMIT 20;
>   ```
> - Kết quả shock: **68% traffic qua NAT Gateway** đang đến AWS service IPs
>   - S3: 8.2 TB/tháng
>   - DynamoDB: 2.1 TB/tháng
>   - ECR (Elastic Container Registry): 1.8 TB/tháng (Docker image pulls)
>   - Secrets Manager: 0.4 TB/tháng
>   - CloudWatch Logs: 0.3 TB/tháng
>
> **Bước 2 — Deploy VPC Endpoints (Điểm Cuối VPC):**
>
> *Gateway Endpoints (miễn phí):*
> - S3 Gateway Endpoint: Deploy trong 30 phút → tiết kiệm ngay 8.2 TB × $0.045 = **$369/tháng** (chỉ NAT processing, không tính bandwidth)
> - DynamoDB Gateway Endpoint: Tương tự
>
> *Interface Endpoints ($0.01/hour/endpoint + $0.01/GB):*
> - ECR API + ECR DKR endpoints: $7.2/endpoint/tháng + data
> - Secrets Manager endpoint: $7.2/endpoint/tháng
> - CloudWatch Logs endpoint: $7.2/endpoint/tháng
>
> *Tính toán cost-benefit cho ECR:*
> - Before: 1.8 TB × $0.045 (NAT) = $81/tháng
> - After: $14.4 (endpoints) + 1.8 TB × $0.01 = $32.4/tháng
> - **Saving: $48.6/tháng per AZ**
>
> **Bước 3 — Tối ưu Docker image strategy:**
> - Phân tích ECR pull patterns: nhiều microservices pull same base image independently
> - Implement image layer caching tại ECS cluster level
> - Giảm unique image pulls 65%
>
> **Bước 4 — Right-size NAT Gateway:**
> - Phát hiện 3 NAT Gateways ở 3 AZs nhưng chỉ có workloads ở 2 AZs
> - Xóa 1 NAT Gateway không dùng: tiết kiệm $32/tháng × 12 = $384/năm"

**R — Kết Quả:**

> "Sau 3 tuần implement:
> - **NAT Gateway cost: $18,000 → $4,800/tháng** (giảm 73%, vượt mục tiêu $6,000)
> - Total AWS bill: $40,000 → $27,000/tháng (giảm 32.5%)
> - **Annual saving: $156,000/năm**
> - Interface Endpoint costs mới thêm: $300/tháng — ROI dương từ tháng đầu tiên
>
> Project được recognize bởi VP Engineering — dùng làm case study cho cost review process mới toàn công ty: automated cost anomaly alerts khi spending tăng > 20% in 7 ngày."

---

## 📝 Template Tự Viết STAR Story

Sử dụng template dưới đây để viết stories từ kinh nghiệm của bạn:

```markdown
### Tiêu đề: [Loại incident/project] — [Kết quả chính]

**S — Situation:**
- Thời gian: [Khi nào]
- Hệ thống: [Loại hệ thống, scale]
- Vấn đề: [Cụ thể vấn đề gì]
- Business impact: [Ảnh hưởng đến business như thế nào]

**T — Task:**
- Vai trò của bạn: [Chính xác bạn responsible cho gì]
- Mục tiêu cụ thể: [Success criteria là gì]
- Constraints: [Giới hạn thời gian, budget, technical]

**A — Action:**
Bước 1: [Tên bước — Chi tiết việc bạn làm]
  - [Cụ thể action 1]
  - [Cụ thể action 2]
  - [Quyết định kỹ thuật quan trọng và lý do]

Bước 2: [Tên bước — ...]
  ...

**R — Result:**
- Metric 1: [Con số trước] → [Con số sau]
- Metric 2: [Con số trước] → [Con số sau]
- Business impact: [Tác động kinh doanh đo lường được]
- Lessons learned: [Bạn học được gì]
```

---

## 💡 Tips Kể Story Hiệu Quả

### Làm
- ✅ Dùng numbers (con số) — "giảm 78%" thuyết phục hơn "cải thiện đáng kể"
- ✅ Nhấn mạnh **quyết định** của bạn, không chỉ hành động
- ✅ Giải thích tại sao bạn chọn cách đó thay vì alternatives (phương án thay thế)
- ✅ Kể cả khi có sự cố hoặc sai lầm — và bạn xử lý như thế nào
- ✅ Kết nối với business value (giá trị kinh doanh)

### Tránh
- ❌ Nói "chúng tôi" nhiều mà không rõ bạn cụ thể làm gì
- ❌ Kể dài dòng về background mà không đến point
- ❌ Kết quả mơ hồ — "hệ thống tốt hơn nhiều"
- ❌ Bỏ qua việc đề cập đến alternatives bạn đã consider
- ❌ Không đề cập lessons learned — thể hiện growth mindset

---

## 🎭 Cách Điều Chỉnh Cho Cấp Bậc

### Junior Engineer
Câu chuyện nên thể hiện:
- Khả năng học nhanh và implement theo hướng dẫn
- Debug có hệ thống theo checklist
- Biết khi nào cần escalate (leo thang) lên senior

### Mid-level Engineer
Câu chuyện nên thể hiện:
- Độc lập diagnose (chẩn đoán) phức tạp
- Đề xuất và implement improvements proactively
- Mentor (hướng dẫn) junior engineers trong team

### Senior Engineer
Câu chuyện nên thể hiện:
- Architecture decisions với trade-off analysis rõ ràng
- Ảnh hưởng vượt team (cross-team impact)
- Thiết lập processes để ngăn incidents tái diễn
- Cost, security, reliability consideration đồng thời

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
