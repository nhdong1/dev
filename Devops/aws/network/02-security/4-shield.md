# AWS Shield — Bảo Vệ Chống DDoS

> AWS Shield là dịch vụ bảo vệ chống **DDoS** (Distributed Denial of Service — Tấn Công Từ Chối Dịch Vụ Phân Tán) — loại tấn công làm ngập lụt hệ thống bằng lượng traffic khổng lồ để khiến dịch vụ không thể hoạt động. AWS Shield cung cấp hai cấp độ: **Shield Standard** (miễn phí, tự động) và **Shield Advanced** (trả phí, cho doanh nghiệp lớn).

---

## 📚 Mục Lục

1. [DDoS Là Gì?](#1-ddos-là-gì)
2. [Shield Standard — Cơ Bản Miễn Phí](#2-shield-standard--cơ-bản-miễn-phí)
3. [Shield Advanced — Nâng Cao Cho Doanh Nghiệp](#3-shield-advanced--nâng-cao-cho-doanh-nghiệp)
4. [So Sánh Standard vs Advanced](#4-so-sánh-standard-vs-advanced)
5. [SRT — Shield Response Team](#5-srt--shield-response-team)
6. [DDoS Cost Protection](#6-ddos-cost-protection)
7. [Tích Hợp Với AWS WAF](#7-tích-hợp-với-aws-waf)
8. [Kiến Trúc Bảo Vệ DDoS Toàn Diện](#8-kiến-trúc-bảo-vệ-ddos-toàn-diện)
9. [Best Practices — Thực Hành Tốt Nhất](#9-best-practices--thực-hành-tốt-nhất)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. DDoS Là Gì?

### Phân Loại DDoS Attacks

```
┌─────────────────────────────────────────────────────────────────┐
│                      DDoS Attack Types                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Volumetric Attacks (Tấn Công Khối Lượng) — Layer 3/4:         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  UDP Flood: Gửi hàng triệu UDP packets                  │   │
│  │  ICMP Flood: Ping flood                                  │   │
│  │  Amplification: NTP, DNS, Memcached reflection           │   │
│  │  Mục tiêu: Làm tắc nghẽn băng thông (bandwidth)         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Protocol Attacks (Tấn Công Giao Thức) — Layer 3/4:            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SYN Flood: Gửi SYN packets mà không hoàn thành handshake│  │
│  │  Fragmented Packet Attack                                │   │
│  │  Mục tiêu: Làm cạn kiệt tài nguyên server/firewall       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Application Attacks (Tấn Công Ứng Dụng) — Layer 7:            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HTTP Flood: Gửi hàng triệu HTTP requests hợp lệ         │   │
│  │  Slowloris: Giữ kết nối HTTP mở rất lâu                  │   │
│  │  DNS Query Flood: Ngập lụt DNS server                    │   │
│  │  Mục tiêu: Làm cạn kiệt CPU, database, application       │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Tác Động Của DDoS

- **Dịch vụ không khả dụng** cho người dùng thật
- **Chi phí tăng vọt** do traffic AWS tăng đột biến
- **Reputation tổn hại** khi downtime kéo dài
- **Mất doanh thu** trực tiếp (e-commerce, fintech)

---

## 2. Shield Standard — Cơ Bản Miễn Phí

### Đặc Điểm

| Đặc Điểm | Chi Tiết |
|----------|----------|
| **Giá** | Miễn phí — tự động cho mọi AWS customer |
| **Kích hoạt** | Tự động, không cần cấu hình |
| **Phạm vi bảo vệ** | Layer 3 & Layer 4 (Volumetric + Protocol attacks) |
| **Scope** | Tất cả AWS resources |

### Bảo Vệ Gì?

```
Shield Standard bảo vệ chống:
  ✅ SYN/UDP flood
  ✅ Reflection attacks (NTP, DNS amplification)
  ✅ Other common Layer 3/4 DDoS attacks

Không bảo vệ:
  ❌ Layer 7 application attacks (HTTP flood) — cần WAF
  ❌ Zero-day DDoS techniques mới
  ❌ Không có SRT (Shield Response Team) support
  ❌ Không có DDoS cost protection
```

### Tích Hợp Tự Động

Shield Standard tự động kích hoạt cho:
- Amazon EC2 instances
- Elastic Load Balancers (ALB, NLB, CLB)
- Amazon CloudFront
- Amazon Route 53
- AWS Global Accelerator

---

## 3. Shield Advanced — Nâng Cao Cho Doanh Nghiệp

### Đặc Điểm

| Đặc Điểm | Chi Tiết |
|----------|----------|
| **Giá** | $3,000/tháng per organization + data transfer fees |
| **Commitment** | 1 năm (subscription) |
| **Phạm vi bảo vệ** | Layer 3, 4, và 7 (với WAF) |
| **SRT Access** | 24/7 Shield Response Team |
| **DDoS Cost Protection** | Hoàn tiền phí AWS phát sinh do DDoS |

### Resources Được Bảo Vệ (Protected Resources)

```
Shield Advanced protected resources:
  ✅ EC2 instances (Elastic IP)
  ✅ Elastic Load Balancers (ALB, NLB, CLB)
  ✅ Amazon CloudFront distributions
  ✅ Amazon Route 53 hosted zones
  ✅ AWS Global Accelerator accelerators
  ✅ VPC Subnets (via Elastic IP của EC2)
```

### Tính Năng Nổi Bật

#### 1. Near Real-Time Attack Visibility (Khả Năng Nhìn Thấy Tấn Công Gần Thời Gian Thực)

```
Dashboard: Shield Advanced Events
  └── Thời gian tấn công: 14:32 - 14:47 UTC
  └── Vector: UDP reflection
  └── Peak magnitude: 2.5 Gbps
  └── Status: Mitigated
  └── Actions taken: Traffic scrubbing activated
```

#### 2. Advanced Attack Mitigation (Giảm Thiểu Tấn Công Nâng Cao)

Shield Advanced phân tích traffic patterns phức tạp:
- **Surgical mitigation:** Chỉ drop malicious traffic, không ảnh hưởng legitimate traffic
- **Anomaly detection:** Phát hiện pattern bất thường dựa trên baseline traffic
- **Proactive engagement:** SRT chủ động liên hệ khi phát hiện cuộc tấn công lớn

#### 3. AWS WAF Included (WAF Đi Kèm)

Shield Advanced **bao gồm AWS WAF miễn phí** (không tính phí Web ACL và rules khi protect resources thuộc Shield Advanced). Điều này tăng giá trị đáng kể vì WAF riêng lẻ có thể tốn $50-200+/tháng.

#### 4. Health-Based Detection (Phát Hiện Dựa Trên Sức Khỏe Ứng Dụng)

```
Kết hợp với Route 53 Health Checks:
  1. Tạo CloudWatch Alarms cho health metrics
  2. Liên kết với Shield Advanced
  3. Shield phát hiện tấn công khi health check fail
  → Mitigations kích hoạt nhanh hơn
```

---

## 4. So Sánh Standard vs Advanced

| Tính Năng | Standard | Advanced |
|-----------|----------|----------|
| **Giá** | Miễn phí | $3,000/tháng |
| **Layer 3/4 protection** | ✅ | ✅ (nâng cao hơn) |
| **Layer 7 protection** | ❌ | ✅ (qua WAF tích hợp) |
| **24/7 SRT support** | ❌ | ✅ |
| **DDoS cost protection** | ❌ | ✅ |
| **Attack visibility** | Hạn chế | Detailed reporting |
| **Near real-time metrics** | ❌ | ✅ |
| **Proactive engagement** | ❌ | ✅ |
| **Health-based detection** | ❌ | ✅ |
| **AWS WAF miễn phí** | ❌ | ✅ |
| **Global Threat Environment Dashboard** | ❌ | ✅ |

### Khi Nào Cần Shield Advanced?

| Tình Huống | Khuyến Nghị |
|-----------|-------------|
| Ứng dụng cần uptime 99.99%+ | Shield Advanced |
| Financial services / Gaming / Media | Shield Advanced |
| Workloads có SLA với khách hàng | Shield Advanced |
| Đã từng bị DDoS tấn công | Shield Advanced |
| Traffic cao bất thường theo mùa | Shield Advanced |
| Startup / Website nhỏ | Shield Standard đủ |

---

## 5. SRT — Shield Response Team

### SRT Là Gì?

**SRT** (Shield Response Team — Đội Phản Ứng Shield) là đội chuyên gia AWS có thể:

1. **24/7 support:** Gọi bất cứ lúc nào khi bị tấn công
2. **Custom mitigations:** Viết custom WAF rules khẩn cấp cho attack đang xảy ra
3. **Attack analysis:** Phân tích traffic để xác định attack vectors
4. **Proactive contact:** Chủ động liên hệ khi phát hiện anomaly lớn

### Cách Engage SRT

```
Khi bị tấn công:
  1. Mở AWS Support case (Business/Enterprise support required)
  2. Tag: "DDoS" và "Shield Advanced"
  3. Cung cấp: Affected resource ARNs, business impact
  4. SRT respond trong vài phút (với Enterprise support)
```

**Điều kiện:** Phải có AWS Support plan **Business** hoặc **Enterprise** để có SRT support.

### IAM Permission Cho SRT

Để SRT có thể hành động, cần cấp IAM role:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "drt.shield.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}
```

SRT sau đó có thể:
- Xem WAF logs và metrics
- Tạo/chỉnh sửa WAF rules khẩn cấp
- Tạo Shield mitigations

---

## 6. DDoS Cost Protection

### Vấn Đề

Khi bị DDoS, traffic tăng vọt → AWS bills tăng vọt:
- EC2 bandwidth costs
- ALB LCU (Load Balancer Capacity Units) costs
- CloudFront data transfer costs
- Route 53 DNS query costs

### Shield Advanced DDoS Cost Protection

Shield Advanced sẽ **hoàn tiền** (credit) cho phí AWS phát sinh trực tiếp do DDoS attack:

```
Ví dụ:
  Tháng bình thường: $500 data transfer
  Tháng bị DDoS:     $8,000 data transfer (tăng $7,500 do attack)
  
  Shield Advanced credit: $7,500 (đủ điều kiện)
  
Quy trình:
  1. Mở AWS Support case sau sự kiện
  2. Cung cấp Shield Event details
  3. AWS review và issue service credit
```

### Resources Được Bảo Vệ Chi Phí

```
DDoS Cost Protection áp dụng cho:
  ✅ EC2 instances
  ✅ Elastic Load Balancers
  ✅ CloudFront
  ✅ Route 53
  ✅ Global Accelerator
```

---

## 7. Tích Hợp Với AWS WAF

### Shield Advanced + WAF = Bảo Vệ Layer 3-7 Toàn Diện

```
DDoS Attack → Shield Advanced
                │
                ├── Layer 3/4 (Volumetric): Shield mitigates directly
                │   → SYN flood, UDP flood → Absorbed at AWS edge
                │
                └── Layer 7 (Application): WAF mitigates
                    → HTTP flood, SlowHTTP → WAF rate limiting, signature matching
```

### Automatic Application Layer DDoS Mitigation

Khi enable trong Shield Advanced:

```
Shield Advanced theo dõi traffic baselines
  │
  ├── Phát hiện anomaly (traffic tăng bất thường)
  │
  └── Tự động tạo WAF rate-based rules
      → Block IPs gửi requests bất thường
      → Không cần can thiệp thủ công
```

---

## 8. Kiến Trúc Bảo Vệ DDoS Toàn Diện

### Kiến Trúc Khuyến Nghị

```
Internet Traffic
       │
       ▼
┌─────────────────────────────────────────────────────┐
│         AWS Edge Network (> 300 PoPs toàn cầu)      │
│  Shield Standard: Absorb Layer 3/4 DDoS traffic     │
│  Shield Advanced: Enhanced mitigation + visibility  │
└─────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│              Amazon CloudFront                      │
│  + AWS WAF (Layer 7 protection)                     │
│  + AWS Certificate Manager (SSL/TLS)                │
└─────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│         Application Load Balancer (ALB)             │
│  + AWS WAF (additional Layer 7)                     │
│  Private subnet, no direct internet access          │
└─────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│         Backend Services (EC2, ECS, Lambda)         │
│  Security Groups (micro-segmentation)               │
└─────────────────────────────────────────────────────┘
```

### Thiết Kế Giảm Thiểu Attack Surface

```
Best practices giảm khả năng bị DDoS:

1. Route qua CloudFront → Ẩn origin IP
2. Dùng ALB thay vì expose EC2 public IP trực tiếp
3. Elastic IP chỉ cho những gì thực sự cần
4. Route 53 với health checks → automatic failover
5. Auto Scaling → absorb traffic spikes
6. CloudFront + WAF → Layer 7 filtering ở edge
```

---

## 9. Best Practices — Thực Hành Tốt Nhất

### ✅ Thiết Kế Kiến Trúc

1. **Giảm attack surface:** Không expose EC2 trực tiếp ra internet — dùng ALB và CloudFront
2. **Dùng Elastic IP cẩn thận:** Chỉ gắn EIP khi thực sự cần direct access
3. **CloudFront trước mọi origin:** Cache content + ẩn origin server
4. **Auto Scaling:** Giúp absorb traffic spikes bất thường
5. **Multi-AZ:** Tránh single point of failure khi một AZ bị tấn công

### ✅ Shield Advanced Setup

6. **Enable Shield Advanced response proactively** — không đợi đến khi bị tấn công
7. **Configure protected resources:** Liệt kê tất cả critical resources
8. **Create health checks** trong Route 53 và liên kết với Shield
9. **Grant SRT access** — IAM role sẵn sàng để SRT có thể giúp ngay
10. **Test failover scenarios** — verify kiến trúc resilient trước khi incident

### ✅ Monitoring

11. **Subscribe to Shield Advanced event notifications** qua SNS
12. **Create CloudWatch alarms** cho DDoS metrics
13. **Review Global Threat Environment Dashboard** định kỳ
14. **Document incident response playbook** cho DDoS scenarios

### ❌ Sai Lầm Phổ Biến

1. **Không enable Shield Advanced** khi có workload critical — $3,000/tháng rẻ hơn cost của 1 giờ downtime với business lớn
2. **Expose origin server IP** trực tiếp ra internet khi đã dùng CloudFront
3. **Không có SRT access** khi cần khẩn cấp — phải setup IAM role trước
4. **Không test failover** — chỉ biết failover có work không khi đã bị tấn công

---

## 10. Câu Hỏi Phỏng Vấn

### Câu 1: Giải thích sự khác biệt giữa Shield Standard và Shield Advanced.

**Trả lời:**
- **Shield Standard:** Miễn phí, tự động, bảo vệ Layer 3/4 (volumetric và protocol attacks) cho mọi AWS customers. Không cần cấu hình.
- **Shield Advanced:** $3,000/tháng, bảo vệ Layer 3/4/7 (kết hợp WAF), bao gồm WAF miễn phí, 24/7 SRT support, DDoS cost protection, real-time visibility, và automatic application layer DDoS mitigation.

---

### Câu 2: Khi nào nên bật Shield Advanced?

**Trả lời:** Khi:
1. Ứng dụng có SLA nghiêm ngặt (99.99%+) và downtime có chi phí cao
2. Đã có hoặc có khả năng cao bị DDoS (gaming, fintech, media)
3. Cần Layer 7 DDoS protection (HTTP floods)
4. Cần SRT support 24/7
5. Muốn bảo vệ chi phí AWS khi bị tấn công

---

### Câu 3: Shield bảo vệ Layer 7 thế nào?

**Trả lời:** Shield Advanced tích hợp với AWS WAF để bảo vệ Layer 7:
1. WAF miễn phí đi kèm Shield Advanced
2. Automatic application layer DDoS mitigation: Tự động tạo WAF rate-based rules khi phát hiện HTTP flood
3. SRT có thể viết custom WAF rules nhanh chóng khi cần
4. Kết hợp với Route 53 health checks để phát hiện application degradation

---

### Câu 4: DDoS Cost Protection hoạt động thế nào?

**Trả lời:** Khi bị DDoS, traffic tăng vọt khiến AWS bills tăng. Shield Advanced cung cấp service credits để hoàn tiền cho chi phí phát sinh trực tiếp do attack (EC2 bandwidth, ALB LCUs, CloudFront data transfer, Route 53 queries). Phải mở AWS Support case sau sự kiện và cung cấp Shield Event details để nhận credit.

---

### Câu 5: Thiết kế kiến trúc để tối đa hóa DDoS resilience.

**Trả lời (kiến trúc):**
```
1. CloudFront (global edge) → WAF (Layer 7) → ẩn origin IP
2. Route 53 với health checks → automatic DNS failover
3. ALB trong private subnet → không expose EC2 trực tiếp
4. Auto Scaling để absorb traffic spikes
5. Multi-AZ và multi-region cho high availability
6. Shield Advanced cho all critical resources
7. SRT access pre-configured
8. Runbook sẵn sàng cho DDoS incident response
```

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Tiếp Theo |
|-------|----------|-----------|
| [3-waf.md](./3-waf.md) | **4-shield.md** | [5-network-firewall.md](./5-network-firewall.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
