# 🛡️ Network Security (Bảo Mật Mạng) trên AWS

> Defense-in-depth (Phòng Thủ Theo Chiều Sâu) — chiến lược bảo mật nhiều lớp cho hạ tầng mạng AWS

## 📚 Mục Lục Module

| File | Chủ Đề | Mức Độ |
|---|---|---|
| [1-security-groups.md](1-security-groups.md) | Security Groups — Stateful Firewall ở cấp Instance | Cơ bản |
| [2-nacls.md](2-nacls.md) | NACLs — Stateless Firewall ở cấp Subnet | Cơ bản |
| [3-vpc-endpoints.md](3-vpc-endpoints.md) | VPC Endpoints — Kết Nối Riêng Tư đến Dịch Vụ AWS | Trung cấp |
| [4-privatelink.md](4-privatelink.md) | AWS PrivateLink — Kết Nối Dịch Vụ Nội Bộ | Trung cấp |
| [5-waf-setup.md](5-waf-setup.md) | WAF — Web Application Firewall | Trung cấp |
| [6-shield-ddos.md](6-shield-ddos.md) | Shield — Bảo Vệ Chống DDoS | Trung cấp |
| [7-network-firewall.md](7-network-firewall.md) | Network Firewall — Tường Lửa Lập Trình Được | Nâng cao |

---

## 🎯 Tổng Quan Defense-in-Depth

Defense-in-depth (Phòng Thủ Theo Chiều Sâu) là chiến lược bảo mật áp dụng nhiều lớp kiểm soát độc lập, đảm bảo kẻ tấn công phải vượt qua nhiều rào cản để đạt mục tiêu.

```
Internet
    │
    ▼
┌─────────────────────────────────┐
│  AWS Shield (DDoS Protection)   │  Lớp 1: Chống tấn công thể tích
│  AWS WAF (Web App Firewall)     │  Lớp 2: Lọc HTTP malicious
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│     Internet Gateway / ALB      │  Entry point vào VPC
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  Network Firewall (Stateful)    │  Lớp 3: Deep Packet Inspection
│  NACLs (Stateless Subnet)       │  Lớp 4: Kiểm soát cấp subnet
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  Security Groups (Instance SG)  │  Lớp 5: Kiểm soát cấp instance
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  VPC Endpoints / PrivateLink    │  Lớp 6: Truy cập dịch vụ nội bộ
│  (Không qua Internet)           │
└─────────────────────────────────┘
```

---

## 🏗️ Kiến Trúc VPC Chuẩn

### Mô Hình 3-Tier (Ba Tầng)

```
VPC (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24)          ← Load Balancer, NAT Gateway
│   ├── Internet Gateway route: 0.0.0.0/0
│   └── NACL: cho phép HTTP/HTTPS từ internet
│
├── Private App Subnet (10.0.2.0/24)     ← EC2 App Servers
│   ├── Route qua NAT Gateway
│   └── NACL: chỉ nhận từ Public Subnet
│
└── Private Data Subnet (10.0.3.0/24)    ← RDS, ElastiCache
    ├── Không có route ra ngoài
    └── NACL: chỉ nhận từ App Subnet
```

### Security Group Hierarchy (Phân Cấp Security Group)

```
ALB Security Group (sg-alb)
  Allow: 0.0.0.0/0 → 80, 443

App Security Group (sg-app)
  Allow: sg-alb → 8080          ← chỉ nhận từ ALB SG

DB Security Group (sg-db)
  Allow: sg-app → 5432          ← chỉ nhận từ App SG
```

---

## 📊 So Sánh Các Cơ Chế Bảo Vệ

| Cơ Chế | Cấp Độ | Stateful? | Layer OSI | Mục Đích Chính |
|---|---|---|---|---|
| **Security Groups** | Instance | ✅ Có | L3/L4 | Lọc inbound/outbound theo port/IP |
| **NACLs** | Subnet | ❌ Không | L3/L4 | Bảo vệ toàn subnet, block dải IP |
| **WAF** | Application | ❌ Không | L7 | Chặn HTTP attacks, OWASP Top 10 |
| **Shield** | Edge | N/A | L3/L4/L7 | Hấp thụ DDoS attack |
| **Network Firewall** | VPC | ✅ Có | L3–L7 | IPS/IDS, domain filtering |
| **PrivateLink/Endpoints** | VPC | N/A | N/A | Kết nối private, không qua internet |

---

## 🔑 Khái Niệm Quan Trọng

### Stateful vs Stateless Firewall

**Stateful Firewall (Tường Lửa Có Trạng Thái):**
- Theo dõi trạng thái kết nối (connection tracking)
- Tự động cho phép traffic phản hồi (return traffic)
- Ví dụ: Security Groups — nếu cho phép TCP 443 inbound, response tự động được phép outbound
- Hiệu quả hơn, ít rule hơn

**Stateless Firewall (Tường Lửa Không Trạng Thái):**
- Mỗi packet được đánh giá độc lập
- Phải định nghĩa rule cho cả 2 chiều (inbound và outbound)
- Ví dụ: NACLs — phải cho phép ephemeral ports (1024–65535) cho return traffic
- Nhanh hơn ở tốc độ cao, xử lý theo rule số thứ tự

### Egress vs Ingress Control

```
Ingress (Lưu Lượng Vào):  Internet → Firewall → VPC Resource
Egress (Lưu Lượng Ra):    VPC Resource → Firewall → Internet/Service
```

Cả hai chiều đều quan trọng:
- Ingress control: ngăn tấn công từ ngoài vào
- Egress control: ngăn data exfiltration (đánh cắp dữ liệu ra ngoài)

---

## 🚀 Thứ Tự Học

1. **Security Groups** — nền tảng bắt buộc, áp dụng hàng ngày
2. **NACLs** — bổ trợ cho Security Groups, bảo vệ subnet
3. **VPC Endpoints** — cải thiện bảo mật và giảm chi phí data transfer
4. **WAF** — bảo vệ ứng dụng web khỏi các tấn công phổ biến
5. **Shield** — chống DDoS, quan trọng với hệ thống công khai
6. **PrivateLink** — kiến trúc microservices và SaaS nâng cao
7. **Network Firewall** — kiểm soát traffic nâng cao trong VPC

---

## 📋 Checklist Bảo Mật Mạng

### Cơ Bản

- [ ] Không sử dụng default Security Group (SG mặc định)
- [ ] Áp dụng principle of least privilege cho Security Group rules
- [ ] Không mở port 22 (SSH) hoặc 3389 (RDP) cho 0.0.0.0/0
- [ ] Dùng Bastion Host hoặc AWS Systems Manager Session Manager thay SSH trực tiếp
- [ ] Đặt tài nguyên nhạy cảm vào private subnet

### Trung Cấp

- [ ] Triển khai NACLs để bảo vệ subnet
- [ ] Dùng VPC Endpoints cho S3, DynamoDB, và dịch vụ AWS thường dùng
- [ ] Bật VPC Flow Logs để theo dõi lưu lượng mạng
- [ ] Triển khai WAF trước CloudFront và ALB công khai
- [ ] Bật Shield Standard (miễn phí) cho tất cả tài nguyên

### Nâng Cao

- [ ] Triển khai AWS Network Firewall cho deep packet inspection
- [ ] Dùng PrivateLink cho kết nối giữa các VPC và dịch vụ nội bộ
- [ ] Thiết lập centralized egress filtering qua Network Firewall
- [ ] Bật Shield Advanced cho workload quan trọng
- [ ] Tích hợp WAF logs với Security Hub và CloudWatch

---

## 🔗 Liên Kết Nội Bộ

- [01-iam-fundamentals/](../01-iam-fundamentals/README.md) — IAM cho network resources
- [04-encryption-kms/](../04-encryption-kms/README.md) — Mã hóa lưu lượng VPN/TLS
- [07-monitoring-auditing/](../07-monitoring-auditing/README.md) — VPC Flow Logs, CloudTrail
- [08-threat-detection/](../08-threat-detection/README.md) — GuardDuty phân tích VPC Flow Logs
- [09-compliance-governance/](../09-compliance-governance/README.md) — Firewall Manager đa tài khoản

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
