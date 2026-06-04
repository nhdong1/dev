# 🔧 AWS Network Troubleshooting — Xử Lý Sự Cố Mạng Toàn Diện

> Hướng dẫn chẩn đoán và xử lý sự cố mạng AWS một cách có hệ thống — từ kết nối EC2 cơ bản đến các vấn đề phức tạp về DNS, Load Balancer và CloudFront CDN (Content Delivery Network — Mạng Phân Phối Nội Dung).

---

## 📚 Mục Lục

1. [Tổng Quan Troubleshooting Framework](#tổng-quan-troubleshooting-framework)
2. [Các Chủ Đề Trong Section Này](#các-chủ-đề-trong-section-này)
3. [Nguyên Tắc Chẩn Đoán](#nguyên-tắc-chẩn-đoán)
4. [Công Cụ Chẩn Đoán AWS](#công-cụ-chẩn-đoán-aws)
5. [Ma Trận Sự Cố Phổ Biến](#ma-trận-sự-cố-phổ-biến)
6. [Quy Trình Xử Lý Sự Cố Khẩn Cấp](#quy-trình-xử-lý-sự-cố-khẩn-cấp)

---

## 🎯 Tổng Quan Troubleshooting Framework

Troubleshooting (Xử Lý Sự Cố) mạng AWS đòi hỏi tư duy có hệ thống theo từng lớp của mô hình OSI (Open Systems Interconnection — Kết Nối Hệ Thống Mở):

```
Lớp 7 — Application (Ứng Dụng):  DNS, HTTP 4xx/5xx, CloudFront errors
Lớp 4 — Transport (Truyền Tải):   TCP/UDP, port access, NLB
Lớp 3 — Network (Mạng):           IP routing, Route Tables, NACL
Lớp 2 — Data Link (Liên Kết):     ENI, Security Groups, VPC
```

### Phương Pháp "Divide and Conquer" (Chia Để Trị)

```
Bước 1: Thu thập triệu chứng (Gather Symptoms)
         ↓
Bước 2: Xác định phạm vi ảnh hưởng (Scope Impact)
         ↓
Bước 3: Kiểm tra từng lớp từ thấp lên cao (Layer-by-Layer)
         ↓
Bước 4: Xác nhận giả thuyết bằng công cụ (Validate Hypothesis)
         ↓
Bước 5: Áp dụng giải pháp và xác minh (Fix & Verify)
         ↓
Bước 6: Ghi lại nguyên nhân gốc rễ (Document Root Cause)
```

---

## 📁 Các Chủ Đề Trong Section Này

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-connectivity-debug.md](./1-connectivity-debug.md) | EC2 không kết nối được — chẩn đoán từng bước | ⭐⭐ |
| [2-security-group-nacl-debug.md](./2-security-group-nacl-debug.md) | Gỡ lỗi Security Group & NACL | ⭐⭐ |
| [3-dns-issues.md](./3-dns-issues.md) | Route 53 & DNS resolution failures | ⭐⭐⭐ |
| [4-load-balancer-issues.md](./4-load-balancer-issues.md) | ALB/NLB health check failures, 503 errors | ⭐⭐⭐ |
| [5-cloudfront-issues.md](./5-cloudfront-issues.md) | Cache miss, origin errors, SSL issues | ⭐⭐⭐ |
| [6-production-checklist.md](./6-production-checklist.md) | Checklist trước khi đưa vào production | ⭐⭐ |

---

## 🧭 Nguyên Tắc Chẩn Đoán

### 1. Kiểm Tra Theo Luồng Dữ Liệu (Follow the Data Path)

```
Client → Route 53 → CloudFront → ALB → Security Group → EC2
```

Luôn bắt đầu từ điểm gần nhất với người dùng và đi dần vào trong.

### 2. Phân Biệt "Không Thể Kết Nối" vs "Kết Nối Bị Từ Chối"

| Triệu Chứng | Nguyên Nhân Thường Gặp |
|-------------|----------------------|
| `Connection timed out` (Kết nối hết thời gian) | Security Group chặn, NACL chặn, route không có |
| `Connection refused` (Kết nối bị từ chối) | Ứng dụng không chạy, port sai, firewall từ chối |
| `NXDOMAIN` (Tên miền không tồn tại) | DNS record không tồn tại, hosted zone sai |
| `503 Service Unavailable` | Target group không có healthy target |
| `502 Bad Gateway` | Backend trả về lỗi hoặc không phản hồi |

### 3. Nguyên Tắc "Minimal Blast Radius" (Phạm Vi Ảnh Hưởng Tối Thiểu)

Khi sửa sự cố production (Môi Trường Sản Xuất):
- Test thay đổi trên môi trường staging trước
- Thay đổi từng cái một, không thay đổi nhiều thứ cùng lúc
- Có rollback plan (Kế Hoạch Quay Lại) trước khi thay đổi

---

## 🛠️ Công Cụ Chẩn Đoán AWS

### Công Cụ Tích Hợp Sẵn (Built-in Tools)

| Công Cụ | Mục Đích | Ưu Điểm |
|---------|---------|---------|
| **VPC Reachability Analyzer** | Kiểm tra đường đi packet từ A đến B | Không cần gửi traffic thật |
| **Network Access Analyzer** | Tìm access path không mong muốn | Tự động phát hiện misconfiguration |
| **VPC Flow Logs** | Ghi lại metadata của tất cả traffic | Phân tích sau sự cố (post-mortem) |
| **CloudWatch Logs Insights** | Query logs với SQL-like syntax | Phân tích nhanh lượng log lớn |
| **Route 53 Resolver Query Logs** | Ghi lại DNS queries | Debug DNS resolution failures |
| **ALB Access Logs** | Chi tiết mọi request đến ALB | Tìm 4xx/5xx patterns |
| **CloudFront Standard Logs** | Access logs từ edge locations | Debug cache behavior |

### Lệnh CLI Hay Dùng

```bash
# Kiểm tra reachability từ EC2 A đến EC2 B
aws ec2 create-network-insights-path \
  --source <instance-id-A> \
  --destination <instance-id-B> \
  --protocol tcp \
  --destination-port 443

# Xem VPC Flow Logs trong CloudWatch
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs \
  --filter-pattern "[version, account, eni, source, destination, srcport, destport, protocol, packets, bytes, start, end, action=REJECT, status]"

# Kiểm tra DNS từ trong VPC
nslookup myapp.internal.company.com 169.254.169.253

# Test kết nối TCP đến port cụ thể
nc -zv <ip-address> 443
```

---

## 📊 Ma Trận Sự Cố Phổ Biến

### Sự Cố Kết Nối

| Triệu Chứng | Nguyên Nhân #1 | Nguyên Nhân #2 | Nguyên Nhân #3 |
|-------------|---------------|---------------|---------------|
| EC2 không ping được | Security Group chặn ICMP | NACL chặn | Instance chưa khởi động xong |
| SSH timeout | Port 22 bị chặn trong SG | NACL inbound rule thiếu | SSH service không chạy |
| Web app không trả lời | Security Group chặn 443/80 | Route Table thiếu IGW | App crash |
| Database connection fail | Security Group RDS chưa mở | Sai subnet (private vs public) | Sai password/credentials |

### Sự Cố DNS

| Triệu Chứng | Nguyên Nhân #1 | Nguyên Nhân #2 |
|-------------|---------------|---------------|
| Domain không phân giải được | Record chưa tạo hoặc sai | TTL (Time To Live) chưa hết hạn cache cũ |
| Internal hostname không phân giải | enableDnsHostnames = false | Private hosted zone không liên kết VPC |
| Failover không tự động | Health check chưa thất bại | Routing policy sai loại |

### Sự Cố Load Balancer

| Triệu Chứng | Nguyên Nhân #1 | Nguyên Nhân #2 |
|-------------|---------------|---------------|
| 503 Service Unavailable | Tất cả targets unhealthy | Target group rỗng |
| Unhealthy targets | App trả về sai health check path | Security Group không cho ALB vào |
| SSL certificate error | Certificate không match domain | Expired certificate |

---

## 🚨 Quy Trình Xử Lý Sự Cố Khẩn Cấp

### P0 — Production Down Hoàn Toàn (Môi Trường Sản Xuất Ngưng Hoàn Toàn)

```
T+0m:  Acknowledge incident, thông báo team
T+5m:  Xác định scope (tất cả user hay một số?)
T+10m: Kiểm tra status page của AWS (status.aws.amazon.com)
T+15m: Rollback deployment gần nhất nếu có thay đổi
T+20m: Kiểm tra CloudWatch alarms và metrics
T+30m: Escalate nếu chưa tìm ra nguyên nhân
```

### P1 — Dịch Vụ Bị Suy Giảm (Service Degraded)

```
T+0m:  Ghi nhận triệu chứng chi tiết
T+15m: Thu thập logs (VPC Flow Logs, ALB Access Logs, App Logs)
T+30m: Kiểm tra từng layer theo thứ tự
T+60m: Implement fix hoặc workaround tạm thời
T+2h:  Root cause analysis (Phân Tích Nguyên Nhân Gốc Rễ)
```

### Checklist Nhanh 5 Phút Khi Có Sự Cố

```
□ AWS Service Health Dashboard có incident không?
□ Có deployment/change nào gần đây không?
□ CloudWatch Alarms có bật đỏ không?
□ EC2 instances có đang running không?
□ Auto Scaling Group có đang scale không?
□ Load Balancer có healthy targets không?
□ DNS records còn trỏ đúng không?
```

---

## 🔗 Điều Hướng

| Vào | Ra |
|-----|-----|
| [08-monitoring/](../08-monitoring/) — Công cụ giám sát | [10-interview-prep/](../10-interview-prep/) — Chuẩn bị phỏng vấn |
| [02-security/](../02-security/) — Security Groups & NACL | [1-connectivity-debug.md](./1-connectivity-debug.md) — Bắt đầu debug |

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
