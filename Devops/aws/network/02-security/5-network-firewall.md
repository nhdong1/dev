# AWS Network Firewall — Tường Lửa Mạng Chuyên Dụng

> AWS Network Firewall là dịch vụ tường lửa mạng được quản lý (managed network firewall) cung cấp **deep packet inspection** (kiểm tra gói tin sâu), **IPS** (Intrusion Prevention System — Hệ Thống Ngăn Chặn Xâm Nhập), và **IDS** (Intrusion Detection System — Hệ Thống Phát Hiện Xâm Nhập) cho toàn bộ VPC traffic, không chỉ HTTP/HTTPS. Đây là lớp bảo mật nâng cao nhất cho mạng AWS, phù hợp với môi trường regulated (có quy định) như financial services, healthcare, government.

---

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#1-khái-niệm-cơ-bản)
2. [Kiến Trúc Network Firewall](#2-kiến-trúc-network-firewall)
3. [Stateful vs Stateless Rules](#3-stateful-vs-stateless-rules)
4. [Rule Groups — Nhóm Quy Tắc](#4-rule-groups--nhóm-quy-tắc)
5. [Suricata Rules — Quy Tắc Suricata](#5-suricata-rules--quy-tắc-suricata)
6. [Domain Filtering — Lọc Tên Miền](#6-domain-filtering--lọc-tên-miền)
7. [Logging & Monitoring](#7-logging--monitoring)
8. [So Sánh Với Các Dịch Vụ Khác](#8-so-sánh-với-các-dịch-vụ-khác)
9. [Triển Khai — Deployment Patterns](#9-triển-khai--deployment-patterns)
10. [Pricing — Chi Phí](#10-pricing--chi-phí)
11. [Best Practices](#11-best-practices)
12. [Câu Hỏi Phỏng Vấn](#12-câu-hỏi-phỏng-vấn)

---

## 1. Khái Niệm Cơ Bản

### Network Firewall Là Gì?

AWS Network Firewall là **next-generation firewall** (tường lửa thế hệ mới) dạng managed service:

```
Không phải AWS Network Firewall:
  ❌ Security Groups (chỉ Allow rules, instance-level)
  ❌ Network ACL (subnet-level, no DPI)
  ❌ WAF (chỉ HTTP/HTTPS Layer 7)

Là AWS Network Firewall:
  ✅ Deep Packet Inspection (DPI) cho mọi protocol
  ✅ IPS (Intrusion Prevention) — chặn threats
  ✅ IDS (Intrusion Detection) — phát hiện và alert
  ✅ Domain filtering — chặn theo domain name
  ✅ Protocol anomaly detection
  ✅ Stateful và Stateless rules
  ✅ Suricata-compatible rule format
```

### Khi Nào Cần Network Firewall?

| Tình Huống | Cần Không? |
|-----------|-----------|
| Web application bảo vệ HTTP/HTTPS | Dùng WAF thay thế |
| Chặn IP độc hại tại subnet | NACL đủ |
| Phát hiện và chặn C2C (Command & Control) traffic | ✅ Cần |
| Compliance: PCI-DSS, HIPAA, FedRAMP | ✅ Cần |
| Kiểm soát egress traffic (ra ngoài) | ✅ Cần |
| Lọc domain names trong DNS queries | ✅ Cần (DNS Firewall hoặc Network Firewall) |
| Deep inspection cho non-HTTP protocols | ✅ Cần |
| East-West traffic inspection (VPC-to-VPC) | ✅ Cần |

---

## 2. Kiến Trúc Network Firewall

### Các Thành Phần

```
┌──────────────────────────────────────────────────────────────────┐
│                       AWS Network Firewall                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    Firewall (resource)                     │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │            Firewall Policy (chính sách)              │  │  │
│  │  │                                                      │  │  │
│  │  │  Stateless Rule Group 1 (ưu tiên cao)                │  │  │
│  │  │  Stateless Rule Group 2                              │  │  │
│  │  │  Stateful Rule Group 1 (Domain List)                 │  │  │
│  │  │  Stateful Rule Group 2 (Suricata rules)              │  │  │
│  │  │  Stateful Rule Group 3 (Standard rules)              │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Firewall Endpoints (1 per AZ):                                  │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │  Endpoint AZ-1a  │  │  Endpoint AZ-1b  │  ...                │
│  │  (trong subnet   │  │  (trong subnet   │                     │
│  │  riêng của FW)   │  │  riêng của FW)   │                     │
│  └──────────────────┘  └──────────────────┘                     │
└──────────────────────────────────────────────────────────────────┘
```

### Firewall Subnet (Subnet Riêng Cho Tường Lửa)

Network Firewall yêu cầu subnet riêng trong mỗi AZ — gọi là **firewall subnet**:

```
VPC (10.0.0.0/16)
│
├── Firewall Subnet AZ-1a (10.0.0.0/28) — chứa Firewall Endpoint
├── Firewall Subnet AZ-1b (10.0.1.0/28) — chứa Firewall Endpoint
│
├── Public Subnet AZ-1a (10.0.2.0/24)  — Web tier
├── Public Subnet AZ-1b (10.0.3.0/24)
│
├── Private Subnet AZ-1a (10.0.4.0/24) — App tier
└── Private Subnet AZ-1b (10.0.5.0/24)
```

### Traffic Flow Qua Firewall

```
Internet → IGW → Route Table → Firewall Endpoint → Destination

Ingress Route Table (gắn vào IGW):
  Destination: 10.0.2.0/24  → vpce-xxx (Firewall Endpoint AZ-1a)
  Destination: 10.0.3.0/24  → vpce-yyy (Firewall Endpoint AZ-1b)

Firewall Subnet Route Table:
  Destination: 0.0.0.0/0   → IGW (sau khi firewall inspect xong)

Public Subnet Route Table:
  Destination: 0.0.0.0/0   → vpce-xxx (Firewall Endpoint, không phải IGW trực tiếp)
```

---

## 3. Stateful vs Stateless Rules

### Stateless Rules (Xử Lý Đầu Tiên)

- Xử lý **trước** stateful rules
- Không nhớ trạng thái kết nối
- Hành động: **PASS** (tiếp tục), **DROP** (chặn), **FORWARD** (chuyển sang stateful)
- Phù hợp cho: Allow/Deny đơn giản dựa trên IP, port, protocol

```
Stateless Rules ví dụ:
  Priority 1: UDP 53 → PASS              (DNS queries)
  Priority 2: TCP any 80,443 → FORWARD   (HTTP/HTTPS → stateful inspection)
  Priority 3: ICMP → PASS               (Ping)
  Default: DROP                          (Chặn tất cả còn lại)
```

### Stateful Rules (Xử Lý Sau)

- Nhớ trạng thái kết nối TCP
- Hành động: **PASS**, **DROP**, **ALERT** (log rồi cho qua)
- Deep packet inspection
- Hỗ trợ Suricata rules

```
Stateful Rules ví dụ:
  Rule 1: Domain list — DENY *.malware-c2.com
  Rule 2: Suricata rule — Alert SQL Injection attempts
  Rule 3: Protocol anomaly — Drop malformed HTTP
```

### Rule Processing Order (Thứ Tự Xử Lý)

```
Traffic vào
    │
    ▼
Stateless Rule Groups (theo priority)
    │
    ├── DROP → Traffic bị chặn ngay
    ├── PASS → Traffic đi qua (không kiểm tra stateful)
    └── FORWARD → Tiếp tục xuống stateful
                  │
                  ▼
          Stateful Rule Groups
                  │
                  ├── DROP   → Chặn
                  ├── ALERT  → Log + Cho qua
                  └── PASS   → Cho qua
                              │
                              ▼
                    Default Firewall Policy Action
                    (DROP ALL hoặc PASS)
```

---

## 4. Rule Groups — Nhóm Quy Tắc

### Hai Loại Rule Groups

1. **Stateless Rule Group:** Xử lý 5-tuple (source IP, source port, dest IP, dest port, protocol)
2. **Stateful Rule Group:** Deep packet inspection, domain filtering, Suricata rules

### Firewall Policy (Chính Sách Tường Lửa)

```
Firewall Policy chứa:
  ├── Stateless Rule Groups (ordered by priority)
  │     ├── Rule Group A (priority 1)
  │     └── Rule Group B (priority 2)
  │
  ├── Stateless Default Actions:
  │     ├── Full packets: FORWARD_TO_SF (stateful), DROP, PASS
  │     └── Fragment packets: Same options
  │
  ├── Stateful Rule Groups (ordered)
  │     ├── Domain List Rule Group
  │     ├── Suricata Rule Group
  │     └── Standard Rule Group
  │
  ├── Stateful Default Actions:
  │     └── DROP_ESTABLISHED, ALERT_ESTABLISHED, DROP_ALL, ALERT_ALL
  │
  └── TLS Inspection Configuration (optional)
```

---

## 5. Suricata Rules — Quy Tắc Suricata

### Suricata Là Gì?

**Suricata** là open-source IDS/IPS engine phổ biến. AWS Network Firewall sử dụng **Suricata-compatible rule syntax** để viết rules tùy chỉnh.

### Cú Pháp Cơ Bản

```
action protocol src_ip src_port direction dest_ip dest_port (options)

Ví dụ:
drop tcp $HOME_NET any -> $EXTERNAL_NET 4444 \
  (msg:"Possible C2 traffic on port 4444"; \
   sid:1000001; rev:1;)
```

### Ví Dụ Suricata Rules Thực Tế

```
# 1. Chặn kết nối SSH ra ngoài internet (chỉ cho phép vào Bastion)
drop tcp $INTERNAL_NETS any -> !$INTERNAL_NETS 22 \
  (msg:"Block outbound SSH - use bastion instead"; \
   sid:1000001; rev:1;)

# 2. Alert khi phát hiện SQL Injection trong HTTP
alert http any any -> $HTTP_SERVERS any \
  (msg:"SQL Injection Attempt"; \
   content:"UNION SELECT"; nocase; http.uri; \
   sid:1000002; rev:1;)

# 3. Chặn tải file thực thi từ internet
drop http $INTERNAL_NETS any -> $EXTERNAL_NET any \
  (msg:"Block executable download"; \
   content:"Content-Type: application/x-msdownload"; http.header; \
   sid:1000003; rev:1;)

# 4. Phát hiện DNS over TCP (có thể là DNS tunneling)
alert tcp $INTERNAL_NETS any -> any 53 \
  (msg:"DNS over TCP - possible tunneling"; \
   sid:1000004; rev:1;)

# 5. Chặn TLS fingerprint của Cobalt Strike (C2 tool phổ biến)
drop tls any any -> any any \
  (msg:"Cobalt Strike HTTPS C2"; \
   tls.fingerprint; content:"72:f3:dd:5c:07:6c:b5:c1:98:10:34:a8:07:90:14:0c"; \
   sid:1000005; rev:1;)
```

### Managed Suricata Rule Groups (AWS Managed)

AWS cung cấp managed Suricata rule groups:
- **ThreatSignaturesAWSManagedRules:** Threat intelligence từ AWS
- **ThreatSignaturesDoS:** Phát hiện DoS patterns
- **ThreatSignaturesBotnet:** Botnet C2 signatures
- **ThreatSignaturesMalware:** Malware communication patterns
- **ThreatSignaturesWebAttacks:** Web attack signatures

---

## 6. Domain Filtering — Lọc Tên Miền

### Domain List Rules

Cho phép lọc traffic dựa trên domain names — hữu ích cho:

```
Whitelist approach (Allowlist — Danh Sách Cho Phép):
  Chỉ cho phép kết nối đến các domain được duyệt:
  ✅ *.amazonaws.com
  ✅ *.example-partner.com
  ✅ api.payment-provider.com
  Tất cả domain khác → DENY

Blacklist approach (Blocklist — Danh Sách Chặn):
  Chặn các domain độc hại đã biết:
  ❌ *.malware-domain.com
  ❌ *.phishing-site.net
  Tất cả domain khác → ALLOW
```

### Ví Dụ Domain Filtering Cho Egress Control

```
# Rule Group Type: Domain List

Action: DENY
Protocol: HTTP, HTTPS
Domains:
  - .malware-c2.com
  - .crypto-miner.net
  - .darkweb-market.onion

---

# Hoặc whitelist approach cho strict environments:
Action: DENY  (cho tất cả không trong list)
Protocol: HTTPS
Domains (ALLOW list):
  - .amazonaws.com
  - .github.com
  - .npmjs.org
  - .docker.io
```

### Route 53 Resolver DNS Firewall vs Network Firewall

| Tính Năng | Route 53 DNS Firewall | Network Firewall |
|-----------|----------------------|------------------|
| **Lọc DNS queries** | ✅ Chặn DNS resolution | ✅ Có (domain list) |
| **Deep packet inspection** | ❌ | ✅ |
| **HTTP content filtering** | ❌ | ✅ |
| **Chi phí** | Thấp hơn | Cao hơn |
| **Khi nào dùng** | Chặn DNS lookups | Full network inspection |

---

## 7. Logging & Monitoring

### Loại Logs

| Log Type | Nội Dung | Destination |
|----------|----------|-------------|
| **Alert logs** | Traffic khớp alert rules | CloudWatch Logs, S3, Kinesis |
| **Flow logs** | Tất cả connections (5-tuple) | CloudWatch Logs, S3, Kinesis |
| **Drop logs** | Traffic bị chặn | CloudWatch Logs, S3, Kinesis |

### Ví Dụ Alert Log

```json
{
  "firewall_name": "prod-network-firewall",
  "availability_zone": "us-east-1a",
  "event_timestamp": "2026-05-14T10:23:45.123Z",
  "event": {
    "timestamp": "2026-05-14T10:23:45.123Z",
    "src_ip": "10.0.4.50",
    "src_port": 52341,
    "dest_ip": "203.0.113.5",
    "dest_port": 4444,
    "proto": "TCP",
    "alert": {
      "action": "blocked",
      "signature_id": 1000001,
      "signature": "Possible C2 traffic on port 4444",
      "category": "Command and Control",
      "severity": 1
    }
  }
}
```

### CloudWatch Alarms Khuyến Nghị

```
Alarm 1: High alert rate
  Metric: AlertCount > 100 trong 5 phút
  Action: SNS notification → PagerDuty/Slack

Alarm 2: Blocked connections to unexpected destinations
  Metric: DroppedPackets tăng > 1000%
  Action: SNS → On-call team

Alarm 3: New domain category blocked
  Metric: Filter theo alert category = "Malware"
  Action: SNS → Security team
```

---

## 8. So Sánh Với Các Dịch Vụ Khác

| Tính Năng | Security Group | NACL | WAF | Network Firewall |
|-----------|--------------|------|-----|-----------------|
| **Layer** | 3/4 | 3/4 | 7 (HTTP) | 3-7 (All) |
| **Stateful** | ✅ | ❌ | ✅ | Cả hai |
| **Phạm vi** | ENI | Subnet | HTTP endpoints | VPC-wide |
| **Deep inspection** | ❌ | ❌ | HTTP only | ✅ All protocols |
| **Domain filtering** | ❌ | ❌ | ❌ | ✅ |
| **IPS/IDS** | ❌ | ❌ | Partial | ✅ |
| **Suricata rules** | ❌ | ❌ | ❌ | ✅ |
| **Non-HTTP protocols** | IP/Port | IP/Port | ❌ | ✅ |
| **Chi phí** | Free | Free | $$ | $$$ |
| **Phức tạp** | Thấp | Thấp | Trung | Cao |

### Khi Dùng Network Firewall Thay Vì WAF?

```
Dùng WAF khi:
  → Chỉ cần bảo vệ HTTP/HTTPS applications
  → SQL Injection, XSS, Bot protection
  → Budget hạn chế
  → Không có compliance requirements phức tạp

Dùng Network Firewall khi:
  → Cần kiểm soát ALL traffic (không chỉ HTTP)
  → Cần egress filtering (EC2 gọi ra internet)
  → Compliance: PCI-DSS, HIPAA, FedRAMP, SOC2
  → Cần IPS/IDS capabilities
  → Cần East-West traffic inspection
  → Cần TLS inspection (mở TLS để inspect)
  → Môi trường với nhiều non-HTTP workloads
```

---

## 9. Triển Khai — Deployment Patterns

### Pattern 1: Centralized Ingress (Kiểm Soát Traffic Vào Tập Trung)

```
Internet
    │
    ▼
IGW (Internet Gateway)
    │
    ▼ (route qua firewall)
Network Firewall Endpoints (Firewall Subnets)
    │
    ▼ (sau khi inspect)
Public Subnets (ALB, NAT Gateway)
    │
    ▼
Private Subnets (EC2, ECS)
```

### Pattern 2: Centralized Egress (Kiểm Soát Traffic Ra Tập Trung)

```
Private Subnets (EC2)
    │
    ▼
Network Firewall (Inspect outbound)
    │
    ▼
NAT Gateway
    │
    ▼
IGW → Internet
```

### Pattern 3: East-West Inspection (Kiểm Tra Traffic Nội Bộ)

Dùng Transit Gateway để route traffic giữa các VPCs qua một Inspection VPC:

```
VPC A (Production)
    │
    ▼
Transit Gateway
    │
    ▼
Inspection VPC (có Network Firewall)
    │
    ▼
Transit Gateway
    │
    ▼
VPC B (Database VPC)
```

### Pattern 4: Full Inspection (Cả Ingress và Egress)

```
Internet
    │ Ingress
    ▼
Network Firewall
    │
    ├── Ingress → inspect → Public Subnet
    │
    └── Egress ← inspect ← Private Subnet → Internet
```

---

## 10. Pricing — Chi Phí

| Thành Phần | Giá (us-east-1) |
|-----------|----------------|
| Firewall Endpoint per AZ | $0.395/giờ (~$285/tháng/AZ) |
| Traffic processed | $0.065/GB |

### Ước Tính Chi Phí

```
Scenario: 2 AZ, 1TB traffic/tháng

Endpoint fees: 2 × $0.395 × 24 × 30 = $567/tháng
Traffic fees: 1000GB × $0.065 = $65/tháng

Tổng: ~$632/tháng

→ Đây là chi phí đáng kể, phù hợp với enterprise environments
→ Với startup hoặc small workloads, WAF + Security Groups thường đủ
```

---

## 11. Best Practices

### ✅ Thiết Kế

1. **Deploy một firewall endpoint mỗi AZ** — không chia sẻ endpoint giữa AZs để tránh cross-AZ traffic costs
2. **Bắt đầu với ALERT mode**, sau đó chuyển sang DROP khi đã verify
3. **Separate Firewall Subnets** — /28 đủ (cần ít nhất 2 IP per subnet)
4. **Centralized Egress pattern** cho multi-VPC environments với Transit Gateway
5. **TLS Inspection** cân nhắc cho deep inspection khi compliance cần (nhưng tăng latency)

### ✅ Rules Management

6. **Version control cho rules** — lưu Suricata rules trong Git
7. **Test rules trong staging** trước khi production
8. **Dùng Managed Rule Groups** làm baseline, thêm custom rules cho specific threats
9. **Set alert thresholds** — không để alerts flood (alert fatigue)
10. **Review và update rules định kỳ** — threats thay đổi liên tục

### ✅ Monitoring

11. **Enable tất cả log types** (alert, flow, drop) cho audit
12. **Gửi logs đến centralized SIEM** (Security Information and Event Management)
13. **Create CloudWatch dashboards** cho real-time visibility
14. **Set SNS alerts** cho high-severity events

---

## 12. Câu Hỏi Phỏng Vấn

### Câu 1: AWS Network Firewall khác WAF như thế nào?

**Trả lời:**
- **WAF:** Chỉ hoạt động với HTTP/HTTPS (Layer 7), bảo vệ web applications khỏi SQL injection, XSS, bots. Đơn giản hơn, rẻ hơn.
- **Network Firewall:** Hoạt động với mọi protocol (Layer 3-7), có IPS/IDS, deep packet inspection, domain filtering, Suricata rules. Phức tạp hơn, đắt hơn, dùng cho enterprise environments với compliance requirements.

---

### Câu 2: Khi nào cần Network Firewall vs chỉ WAF + Security Groups?

**Trả lời:**
Cần Network Firewall khi:
- Compliance yêu cầu IPS/IDS
- Cần kiểm soát egress traffic (EC2 gọi đến đâu trong internet)
- Cần East-West traffic inspection giữa VPCs
- Có non-HTTP workloads cần deep inspection
- Cần phát hiện C2C (Command & Control) communications
- Cần Suricata-based threat signatures

---

### Câu 3: Stateful và stateless rules trong Network Firewall khác nhau thế nào?

**Trả lời:**
- **Stateless rules:** Xử lý trước, không nhớ connection state, chỉ dựa trên 5-tuple (src IP, src port, dst IP, dst port, protocol). Hành động: PASS, DROP, FORWARD. Phù hợp cho simple allow/deny.
- **Stateful rules:** Xử lý sau, nhớ TCP state, deep packet inspection, hỗ trợ Suricata rules, domain filtering. Hành động: PASS, DROP, ALERT.

Luồng: Stateless xử lý trước → FORWARD_TO_STATEFUL → Stateful inspection.

---

### Câu 4: Giải thích Suricata rules trong AWS Network Firewall.

**Trả lời:** AWS Network Firewall sử dụng Suricata-compatible rule syntax — cùng format với Suricata open-source IPS engine. Rules có dạng:

```
action protocol src_ip src_port -> dest_ip dest_port (options; sid:xxx; rev:1;)
```

Có thể viết custom Suricata rules để detect specific threats (malware signatures, C2 patterns, protocol anomalies) hoặc dùng AWS Managed Suricata rule groups cho common threat signatures.

---

### Câu 5: Thiết kế egress filtering architecture với Network Firewall.

**Trả lời:**
```
Architecture:
  Private EC2 Instances
      │
      ▼ (Route: 0.0.0.0/0 → Firewall Endpoint)
  Network Firewall (Firewall Subnet)
      │ Domain Filtering: Allow list only trusted domains
      │ Suricata rules: Block C2C patterns
      │ Protocol rules: Block non-standard ports
      ▼ (Route: 0.0.0.0/0 → NAT Gateway)
  NAT Gateway (Public Subnet)
      │
      ▼
  Internet Gateway → Internet

Benefits:
  - EC2 chỉ kết nối được domain đã được duyệt
  - Phát hiện malware/ransomware trying to call home
  - Full audit log cho compliance
```

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Section Tiếp Theo |
|-------|----------|-------------------|
| [4-shield.md](./4-shield.md) | **5-network-firewall.md** | [03-load-balancing/README.md](../03-load-balancing/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
