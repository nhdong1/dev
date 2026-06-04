# Bảo Mật Mạng AWS — Tổng Quan

> Security (Bảo Mật) là lớp nền tảng bắt buộc cho mọi kiến trúc AWS production. Section này bao gồm toàn bộ công cụ bảo mật mạng của AWS: từ Security Groups (Nhóm Bảo Mật), Network ACLs (Danh Sách Kiểm Soát Truy Cập Mạng), đến WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web), Shield (Chắn DDoS), và Network Firewall (Tường Lửa Mạng).

---

## 📚 Mục Lục Section

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-security-groups.md](./1-security-groups.md) | Security Groups — Stateful Firewall | ⭐⭐ |
| [2-network-acls.md](./2-network-acls.md) | Network ACLs — Stateless Firewall & So Sánh với SG | ⭐⭐ |
| [3-waf.md](./3-waf.md) | WAF — Web Application Firewall, Rules, Managed Rules | ⭐⭐⭐ |
| [4-shield.md](./4-shield.md) | Shield Standard & Advanced — Chống DDoS | ⭐⭐ |
| [5-network-firewall.md](./5-network-firewall.md) | AWS Network Firewall — Deep Packet Inspection | ⭐⭐⭐ |

---

## 🏛️ Kiến Trúc Bảo Mật Theo Lớp (Defense-in-Depth)

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  AWS Shield Advanced                                │
│  (Chống DDoS — Distributed Denial of Service)      │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  AWS WAF (Web Application Firewall)                 │
│  (Lọc HTTP/HTTPS — tấn công Layer 7)                │
│  Kết hợp với CloudFront, ALB, API Gateway           │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  VPC Boundary — Biên Giới VPC                       │
│  Network ACL (Stateless — áp dụng cho Subnet)       │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  AWS Network Firewall                               │
│  (Deep Packet Inspection — Kiểm Tra Gói Tin Sâu)   │
│  Đặt trong Inspection VPC hoặc subnet riêng         │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Security Group (Stateful — áp dụng cho ENI)        │
│  Bảo vệ từng instance EC2, container, RDS           │
└─────────────────────────────────────────────────────┘
    │
    ▼
  EC2 / ECS / RDS / Lambda
```

**Defense-in-Depth** (Bảo Vệ Theo Chiều Sâu): Không có lớp bảo mật nào là hoàn hảo — xây dựng nhiều lớp để kẻ tấn công phải vượt qua tất cả.

---

## 🔑 Khái Niệm Cốt Lõi

### Stateful vs Stateless Firewall (Tường Lửa Trạng Thái vs Phi Trạng Thái)

| Đặc Điểm | Stateful (Security Group) | Stateless (Network ACL) |
|----------|--------------------------|------------------------|
| **Theo dõi kết nối** | Có — nhớ trạng thái kết nối | Không — mỗi gói tin xử lý độc lập |
| **Return traffic** | Tự động cho phép | Phải tạo rule chiều về riêng |
| **Phạm vi áp dụng** | ENI (Network Interface) | Subnet (toàn bộ subnet) |
| **Quy tắc deny** | Không có (chỉ Allow) | Có Allow và Deny |
| **Xử lý khi hết rules** | Deny all (mặc định) | Deny all (rule cuối) |

### Ký Hiệu Cổng Phổ Biến

| Cổng | Giao Thức | Ứng Dụng |
|------|-----------|----------|
| 22 | TCP | SSH (Secure Shell — Vỏ Bọc An Toàn) |
| 80 | TCP | HTTP (HyperText Transfer Protocol) |
| 443 | TCP | HTTPS (HTTP Secure) |
| 3306 | TCP | MySQL / Aurora |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 27017 | TCP | MongoDB |
| 8080 | TCP | HTTP alternative / Application servers |

---

## 🛠️ Công Cụ Bảo Mật AWS — Bản Đồ Tổng Quan

```
Mối Đe Dọa              Công Cụ AWS                    Lớp Bảo Vệ
─────────────           ──────────────────             ──────────────
DDoS volumetric    →    Shield Standard/Advanced        Network Edge
DDoS application   →    Shield Advanced + WAF           Application Edge
Web attacks        →    WAF (SQL Injection, XSS...)     Application Layer 7
Port scanning      →    Network ACL + Security Group    Network Layer 3/4
Unauthorized SSH   →    Security Group (Port 22)        Instance Level
Lateral movement   →    Security Group (micro-segment.) Instance Level
DNS exfiltration   →    Route 53 Resolver DNS FW        DNS Layer
Deep inspection    →    Network Firewall (IPS/IDS)       Network Layer
Threat detection   →    GuardDuty                       Behavioral Analysis
Config compliance  →    AWS Config + Security Hub        Compliance Layer
```

---

## 📋 Nguyên Tắc Bảo Mật Mạng AWS

### 1. Principle of Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu)

Chỉ mở port và giao thức thực sự cần thiết. Không bao giờ mở `0.0.0.0/0` cho SSH/RDP trong production.

### 2. Micro-segmentation (Phân Đoạn Vi Mô)

Dùng Security Groups để cô lập workloads. Web tier không nên trực tiếp kết nối database tier — phải qua app tier.

### 3. Egress Filtering (Lọc Lưu Lượng Ra)

Kiểm soát traffic ra ngoài, không chỉ traffic vào. Dùng Network ACL và Security Group để giới hạn outbound.

### 4. Layered Security (Bảo Mật Nhiều Lớp)

Kết hợp Security Group + Network ACL + WAF + Shield cho production. Không phụ thuộc vào một lớp duy nhất.

### 5. Zero Trust Network (Mạng Không Tin Tưởng)

Xác thực mọi kết nối, kể cả internal traffic. Không giả định rằng traffic bên trong VPC là an toàn.

---

## ⚡ Quick Reference — Tham Khảo Nhanh

### Khi Nào Dùng Gì?

| Tình Huống | Giải Pháp |
|-----------|-----------|
| Bảo vệ instance EC2 cụ thể | Security Group |
| Chặn toàn bộ Subnet | Network ACL |
| Chống SQL Injection, XSS trên web app | WAF |
| Chống DDoS volumetric | Shield Standard (miễn phí) |
| Chống DDoS tinh vi + SRT support | Shield Advanced |
| Deep packet inspection, IPS/IDS | AWS Network Firewall |
| Kiểm soát DNS queries | Route 53 Resolver DNS Firewall |
| Phát hiện bất thường / threat intel | GuardDuty |

---

## 🎯 Checklist Bảo Mật Mạng AWS

### Cơ Bản (Bắt Buộc Cho Mọi Môi Trường)

- [ ] Security Groups: Không mở `0.0.0.0/0` cho SSH (22) hoặc RDP (3389)
- [ ] Security Groups: Chỉ mở port cần thiết cho từng tier
- [ ] Network ACL: Cấu hình default deny cho subnet nhạy cảm
- [ ] VPC Flow Logs: Bật để ghi lại traffic log
- [ ] IMDSv2: Yêu cầu trên tất cả EC2 instances

### Nâng Cao (Cho Production)

- [ ] WAF: Bật managed rule groups cho ALB / CloudFront / API Gateway
- [ ] Shield Advanced: Bật cho workloads có yêu cầu SLA cao
- [ ] GuardDuty: Bật ở mức account để phát hiện threat
- [ ] Security Hub: Tổng hợp findings từ tất cả dịch vụ security
- [ ] AWS Config: Rules kiểm tra compliance tự động
- [ ] Network Firewall: Cho môi trường regulated hoặc cần deep inspection

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Tiếp Theo |
|-------|----------|-----------|
| [01-vpc-fundamentals/](../01-vpc-fundamentals/) | **02-security/** | [03-load-balancing/](../03-load-balancing/) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
