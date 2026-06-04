# High Availability & Fault Tolerance — Tính Sẵn Sàng Cao & Khả Năng Chịu Lỗi trên AWS Compute

> High Availability (HA — Tính Sẵn Sàng Cao) là khả năng hệ thống tiếp tục hoạt động dù có sự cố phần cứng, mạng hay phần mềm. Fault Tolerance (Khả Năng Chịu Lỗi) là khả năng hệ thống vẫn hoạt động bình thường khi một hoặc nhiều thành phần bị lỗi — không chỉ tiếp tục hoạt động mà không gây gián đoạn cho người dùng.

## 📚 Mục Lục

1. [Tại Sao HA & Fault Tolerance Quan Trọng](#tại-sao-quan-trọng)
2. [Các Khái Niệm Cốt Lõi](#các-khái-niệm-cốt-lõi)
3. [Các Chủ Đề Trong Module Này](#các-chủ-đề-trong-module-này)
4. [Kiến Trúc HA Tổng Quan](#kiến-trúc-ha-tổng-quan)
5. [Lộ Trình Học](#lộ-trình-học)
6. [Câu Hỏi Phỏng Vấn Nhanh](#câu-hỏi-phỏng-vấn-nhanh)

---

## 🎯 Tại Sao Quan Trọng

Mọi hệ thống sản xuất đều sẽ gặp sự cố. Câu hỏi không phải là "nếu" mà là "khi nào". Thiết kế HA đúng cách giúp:

| Vấn Đề                        | Không Có HA                   | Có HA                             |
| ----------------------------- | ----------------------------- | --------------------------------- |
| AZ bị sập                     | Toàn bộ app ngưng hoạt động   | Tự động failover sang AZ khác     |
| Instance phần cứng lỗi        | Downtime khi khởi động lại    | Auto Scaling tạo instance mới     |
| Deploy gây lỗi                | Tất cả người dùng bị ảnh hưởng | Chỉ % nhỏ bị ảnh hưởng (canary)  |
| Traffic đột biến x10          | App chậm / crash              | Auto Scaling mở rộng tự động      |
| Health check thất bại         | Instance lỗi vẫn nhận traffic  | ELB loại bỏ instance khỏi pool    |

### SLA, SLO, SLI — Ba Khái Niệm Cần Nắm

- **SLA** — Service Level Agreement (Thỏa Thuận Mức Dịch Vụ): Cam kết pháp lý với khách hàng. Ví dụ: "99.9% uptime mỗi tháng"
- **SLO** — Service Level Objective (Mục Tiêu Mức Dịch Vụ): Mục tiêu nội bộ, thường cao hơn SLA. Ví dụ: "99.95% uptime"
- **SLI** — Service Level Indicator (Chỉ Số Mức Dịch Vụ): Số liệu đo lường thực tế. Ví dụ: "tỷ lệ request thành công trong 30 ngày qua"

```
SLI (đo lường) → SLO (mục tiêu) → SLA (cam kết)
```

### Uptime Mapping — Bảng Quy Đổi Uptime

| SLA        | Downtime / Năm | Downtime / Tháng | Downtime / Tuần |
| ---------- | -------------- | ---------------- | --------------- |
| 99%        | 3.65 ngày      | 7.2 giờ          | 1.68 giờ        |
| 99.9%      | 8.76 giờ       | 43.8 phút        | 10.1 phút       |
| 99.95%     | 4.38 giờ       | 21.9 phút        | 5 phút          |
| 99.99%     | 52.6 phút      | 4.38 phút        | 1.01 phút       |
| 99.999%    | 5.26 phút      | 26.3 giây        | 6.05 giây       |

---

## 🔑 Các Khái Niệm Cốt Lõi

### RTO & RPO — Hai Số Liệu Định Nghĩa HA

- **RTO** — Recovery Time Objective (Mục Tiêu Thời Gian Phục Hồi): Hệ thống phải phục hồi trong bao lâu sau sự cố? Ví dụ: RTO = 5 phút nghĩa là app phải online trở lại trong 5 phút.
- **RPO** — Recovery Point Objective (Mục Tiêu Điểm Phục Hồi): Có thể chấp nhận mất dữ liệu tối đa bao nhiêu? Ví dụ: RPO = 1 giờ nghĩa là nếu hệ thống sập, dữ liệu tối đa 1 giờ trước vẫn còn.

```
Sự cố xảy ra
     │
     ▼
[RPO: mất dữ liệu tối đa?] ←── Last backup/snapshot
     │
     ▼
[RTO: phục hồi trong bao lâu?]
     │
     ▼
Hệ thống hoạt động trở lại
```

**Chi phí và RTO/RPO nghịch chiều:** RTO/RPO càng nhỏ → chi phí HA càng cao.

### Availability Zone (AZ) & Region

- **AZ** — Availability Zone (Vùng Khả Dụng): Data center hoặc cụm data center riêng biệt trong một Region. Cách nhau ≥100km, kết nối bằng cáp quang tốc độ cao, độ trễ <2ms.
- **Region** (Vùng): Tập hợp ≥2 AZ cùng địa lý (ví dụ: `ap-southeast-1` = Singapore).
- **Edge Location** (Vị Trí Biên): Điểm hiện diện của CloudFront CDN, Route 53 DNS.

### Fault Domain (Miền Lỗi)

Nhóm tài nguyên chia sẻ cùng điểm lỗi duy nhất (single point of failure — SPOF). Thiết kế HA = phân tán tài nguyên qua nhiều fault domain.

---

## 📁 Các Chủ Đề Trong Module Này

### [1. Multi-AZ Design](./1-multi-az-design.md)

Multi-AZ (Đa Vùng Khả Dụng) là nền tảng của mọi kiến trúc HA trên AWS:

- Kiến trúc Multi-AZ cho EC2 + ELB
- Multi-AZ cho ECS Service và EKS Node Groups
- Multi-AZ cho RDS, ElastiCache
- Multi-Region (Đa Vùng) Patterns: Active-Passive, Active-Active
- Failover (Chuyển Đổi Dự Phòng) tự động với Route 53

### [2. Scaling Strategies](./2-scaling-strategies.md)

Chiến lược mở rộng phù hợp với từng loại workload:

- Reactive Scaling (Co Giãn Phản Ứng): Scale dựa trên metric hiện tại
- Predictive Scaling (Co Giãn Dự Báo): ML dự đoán nhu cầu trước
- Scheduled Scaling (Co Giãn Theo Lịch): Scale theo giờ/ngày biết trước
- Horizontal vs Vertical Scaling (Co Giãn Ngang vs Dọc)
- Scale-out vs Scale-in strategies

### [3. Health Checks & Recovery](./3-health-checks-recovery.md)

Phát hiện lỗi nhanh và tự động phục hồi:

- EC2 Status Checks (Kiểm Tra Trạng Thái EC2): System check, Instance check
- ELB Health Checks (Kiểm Tra Sức Khỏe ELB): HTTP, HTTPS, TCP
- ECS Health Checks: Container-level và Service-level
- Auto Recovery (Tự Động Phục Hồi): CloudWatch Alarm → EC2 recovery
- Circuit Breaker Pattern (Mô Hình Ngắt Mạch)

### [4. Deployment Strategies](./4-deployment-strategies.md)

Deploy không gây downtime:

- Rolling Update (Cập Nhật Cuộn): Thay thế instance từng phần
- Blue/Green Deployment (Triển Khai Xanh/Lam): Hai môi trường song song
- Canary Release (Phát Hành Canary): Traffic nhỏ sang version mới
- Immutable Deployment (Triển Khai Bất Biến): Luôn tạo instance mới
- In-place vs Out-of-place deployment

### [5. Capacity Planning](./5-capacity-planning.md)

Đảm bảo đủ tài nguyên trước khi cần:

- Load Testing (Kiểm Thử Tải): Stress test, Spike test, Soak test
- Capacity Forecasting (Dự Báo Năng Lực): Growth trends, seasonality
- Pre-scaling (Mở Rộng Trước): Warmup trước sự kiện lớn
- AWS Compute Optimizer insights
- Bottleneck Analysis (Phân Tích Nút Thắt Cổ Chai)

---

## 🏗️ Kiến Trúc HA Tổng Quan

### Kiến Trúc 3-Tier HA Chuẩn Trên AWS

```
                        Internet
                            │
                    ┌───────▼────────┐
                    │   Route 53     │  DNS Failover
                    │  (DNS HA)      │  Health checks
                    └───────┬────────┘
                            │
               ┌────────────▼────────────┐
               │  Application Load       │  Multi-AZ
               │  Balancer (ALB)         │  Cross-zone LB
               └──────┬──────────┬───────┘
                      │          │
            ┌─────────▼──┐  ┌────▼────────┐
            │   AZ-1a    │  │   AZ-1b     │   Presentation Layer
            │  Web Tier  │  │  Web Tier   │   (EC2 / ECS / Lambda)
            └─────────┬──┘  └────┬────────┘
                      │          │
               ┌──────▼──────────▼──────┐
               │  Internal Load Balancer│   App Layer
               └──────┬──────────┬──────┘
                      │          │
            ┌─────────▼──┐  ┌────▼────────┐
            │   AZ-1a    │  │   AZ-1b     │   Application Layer
            │  App Tier  │  │  App Tier   │   (EC2 / ECS / Lambda)
            └─────────┬──┘  └────┬────────┘
                      │          │
               ┌──────▼──────────▼──────┐
               │   RDS Multi-AZ         │   Data Layer
               │  Primary │  Standby    │   Sync replication
               └──────────────────────── ┘
```

### Komponen HA AWS Tương Ứng

| Tầng              | Dịch Vụ AWS                           | HA Mechanism                          |
| ----------------- | ------------------------------------- | ------------------------------------- |
| DNS               | Route 53                              | Health checks, failover routing       |
| Load Balancer     | ALB / NLB                             | Multi-AZ, cross-zone load balancing   |
| Compute           | EC2 + ASG                             | Multi-AZ ASG, health check replacement|
| Compute (Container)| ECS + Fargate                        | Multi-AZ task placement               |
| Compute (K8s)     | EKS                                   | Multi-AZ node groups, pod disruption  |
| Database          | RDS Multi-AZ                          | Synchronous standby, auto failover    |
| Cache             | ElastiCache (Multi-AZ)                | Replica failover                      |
| Storage           | S3                                    | 11 nines durability, 3 AZ replication |
| Queue             | SQS                                   | Managed HA, 3-AZ redundant            |

---

## 🎓 Lộ Trình Học

### Người Mới — Beginner

```
1. Hiểu khái niệm AZ, Region, và tại sao Multi-AZ quan trọng
2. Tạo ASG span across nhiều AZ
3. Cấu hình ALB với EC2 target group Multi-AZ
4. Hiểu health check của ELB và EC2
```

### Trung Cấp — Intermediate

```
1. Thiết kế full 3-tier HA architecture
2. So sánh Rolling vs Blue/Green vs Canary deployment
3. Cấu hình RDS Multi-AZ với failover testing
4. Tích hợp health checks với Auto Scaling
```

### Nâng Cao — Advanced

```
1. Multi-Region Active-Active với Route 53 latency routing
2. Chaos Engineering (Kỹ Thuật Hỗn Loạn) — cố tình inject lỗi
3. Thiết kế cho RPO=0 / RTO<1 phút
4. Capacity planning với ML-based Predictive Scaling
```

---

## ❓ Câu Hỏi Phỏng Vấn Nhanh

**Q: Sự khác biệt giữa High Availability và Fault Tolerance?**
> HA: Hệ thống tiếp tục hoạt động với downtime tối thiểu. FT: Hệ thống hoạt động không gián đoạn dù có thành phần lỗi. FT đòi hỏi chi phí cao hơn vì cần redundancy đầy đủ.

**Q: Khi nào dùng Multi-AZ vs Multi-Region?**
> Multi-AZ: Bảo vệ chống lỗi data center đơn, đủ cho hầu hết ứng dụng. Multi-Region: Khi cần DR (Disaster Recovery — Phục Hồi Thảm Họa) cho sự cố toàn region, compliance yêu cầu dữ liệu ở nhiều địa lý, hoặc cần giảm latency cho users toàn cầu.

**Q: RTO và RPO khác nhau thế nào?**
> RTO đo "hệ thống phục hồi sau bao lâu", RPO đo "có thể mất tối đa bao nhiêu dữ liệu". Cả hai càng nhỏ càng tốn kém.

**Q: Blue/Green vs Canary deployment khác nhau thế nào?**
> Blue/Green: 100% traffic chuyển sang version mới (nhanh rollback). Canary: Tăng dần % traffic (ít rủi ro hơn, phát hiện lỗi sớm).

---

## 🔗 Điều Hướng Module

| Bước | File                                                   | Nội Dung                           |
| ---- | ------------------------------------------------------ | ---------------------------------- |
| 1    | [1-multi-az-design.md](./1-multi-az-design.md)         | Kiến trúc Multi-AZ & Multi-Region  |
| 2    | [2-scaling-strategies.md](./2-scaling-strategies.md)   | Chiến lược co giãn                 |
| 3    | [3-health-checks-recovery.md](./3-health-checks-recovery.md) | Health checks & tự động phục hồi |
| 4    | [4-deployment-strategies.md](./4-deployment-strategies.md) | Blue/Green, Canary, Rolling      |
| 5    | [5-capacity-planning.md](./5-capacity-planning.md)     | Load testing & capacity planning   |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
