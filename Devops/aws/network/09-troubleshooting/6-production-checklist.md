# ✅ Production Checklist — Danh Sách Kiểm Tra Trước Khi Đưa Vào Sản Xuất

> Checklist toàn diện trước khi deploy (Triển Khai) hoặc thay đổi cấu hình mạng AWS lên môi trường production (Môi Trường Sản Xuất) — giúp ngăn chặn sự cố trước khi xảy ra.

---

## 📚 Mục Lục

1. [Cách Sử Dụng Checklist Này](#cách-sử-dụng-checklist-này)
2. [VPC & Networking Foundation](#vpc--networking-foundation)
3. [Security Checklist](#security-checklist)
4. [Load Balancer Checklist](#load-balancer-checklist)
5. [DNS & Route 53 Checklist](#dns--route-53-checklist)
6. [CloudFront CDN Checklist](#cloudfront-cdn-checklist)
7. [Monitoring & Alerting Checklist](#monitoring--alerting-checklist)
8. [Disaster Recovery Checklist](#disaster-recovery-checklist)
9. [Cost Optimization Checklist](#cost-optimization-checklist)
10. [Deployment Checklist Cuối Cùng](#deployment-checklist-cuối-cùng)

---

## 📋 Cách Sử Dụng Checklist Này

### Khi Nào Dùng

| Tình Huống | Section Cần Kiểm Tra |
|------------|---------------------|
| Deploy lần đầu (Greenfield) | Tất cả sections |
| Thêm tính năng mới | Security + Load Balancer + DNS |
| Thay đổi cấu hình mạng | VPC + Security + Monitoring |
| Trước khi ra mắt (Launch) | Tất cả + Deployment Checklist |
| Sau incident | Monitoring + DR Checklist |

### Mức Độ Ưu Tiên

- 🔴 **CRITICAL** — Bắt buộc phải hoàn thành trước khi deploy
- 🟡 **IMPORTANT** — Nên hoàn thành, có thể ảnh hưởng đến sự cố
- 🟢 **BEST PRACTICE** — Khuyến nghị cho production-grade system

---

## 🌐 VPC & Networking Foundation

### CIDR Planning (Quy Hoạch Địa Chỉ IP)

```
🔴 □ CIDR của VPC đủ lớn cho growth (tăng trưởng) dài hạn?
       Khuyến nghị: /16 cho production (65,534 IPs)
🔴 □ Không có CIDR overlap với các VPC khác hoặc on-premises network?
🔴 □ Subnet sizes phù hợp với expected workload?
       Public subnet: /24 (254 IPs) — ALB, NAT Gateway
       Private subnet: /22 (1022 IPs) — Application tier
       Database subnet: /24 (254 IPs) — RDS, ElastiCache
🟡 □ Đã dự phòng CIDR range cho future VPC peering hoặc Transit Gateway?
🟢 □ Đã document IP allocation plan cho team?
```

### Subnet Architecture (Kiến Trúc Mạng Con)

```
🔴 □ Có ít nhất 2 AZ (Availability Zone — Vùng Khả Dụng) cho High Availability?
🔴 □ Resources nhạy cảm (database, internal services) ở private subnets?
🔴 □ Không expose database trực tiếp ra public subnet?
🟡 □ Có isolated subnet cho database tier (không route ra internet)?
🟡 □ NAT Gateway được triển khai trong mỗi AZ (tránh single point of failure)?
🟢 □ Subnet naming convention nhất quán và dễ hiểu?
```

### Route Tables (Bảng Định Tuyến)

```
🔴 □ Public subnet: 0.0.0.0/0 → Internet Gateway (IGW)?
🔴 □ Private subnet: 0.0.0.0/0 → NAT Gateway (đúng AZ)?
🔴 □ Database subnet: KHÔNG có route ra internet?
🟡 □ VPC Peering routes đã được thêm vào TẤT CẢ route tables liên quan?
🟡 □ Transit Gateway routes được cấu hình đúng?
🟢 □ Route table propagation (Truyền Bảng Định Tuyến) từ VGW được review?
```

---

## 🔐 Security Checklist

### Security Groups

```
🔴 □ Không có Security Group nào mở port 22/3389 cho 0.0.0.0/0?
🔴 □ Mỗi layer (ALB, App, Database) có Security Group riêng biệt?
🔴 □ Database Security Group chỉ cho phép traffic từ Application SG?
🔴 □ Security Group reference (tham chiếu) được dùng thay vì hardcode IPs nội bộ?
🟡 □ Outbound rules được review — không mở tất cả outbound nếu không cần?
🟡 □ Security Group có mô tả rõ ràng cho từng rule?
🟢 □ Định kỳ review và xóa rules không dùng nữa?
🟢 □ Dùng AWS Config Rule để phát hiện SG không tuân thủ policy?
```

### Network ACLs

```
🔴 □ NACL không chặn ephemeral ports (1024-65535) outbound — cần cho return traffic?
🟡 □ Rule numbers được tổ chức có thứ tự logic (100, 200, 300...)?
🟡 □ Có DENY rules cho IP ranges độc hại đã biết?
🟢 □ Không thêm quá nhiều NACL rules (AWS limit: 20 rules mỗi chiều)?
```

### WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web)

```
🟡 □ WAF được gán vào ALB/CloudFront cho production workloads?
🟡 □ AWS Managed Rule Groups đã được bật (OWASP Core, Known Bad Inputs)?
🟡 □ Rate limiting rules để chống DDoS (Distributed Denial of Service) layer 7?
🟡 □ WAF logging được bật để audit và analysis?
🟢 □ WAF rules được test trên staging trước?
🟢 □ IP reputation blocking từ AWS managed IP lists?
```

### Data In Transit (Dữ Liệu Khi Truyền)

```
🔴 □ Tất cả traffic giữa người dùng và ALB/CloudFront dùng HTTPS?
🔴 □ ALB listener redirect HTTP → HTTPS?
🔴 □ SSL/TLS minimum version: TLSv1.2 (không dùng TLSv1.0/1.1)?
🟡 □ Certificate auto-renewal được cấu hình (dùng ACM)?
🟡 □ Truyền dữ liệu giữa services nội bộ có mã hóa không (nếu cần compliance)?
🟢 □ HSTS (HTTP Strict Transport Security) header được bật?
```

---

## ⚖️ Load Balancer Checklist

### Application Load Balancer (ALB)

```
🔴 □ ALB được deploy trong ít nhất 2 AZ?
🔴 □ Health check path trả về HTTP 200 với response nhẹ (< 1KB)?
🔴 □ Health check interval và threshold phù hợp với app startup time?
🔴 □ SSL certificate hợp lệ, chưa expire, cover đúng domain?
🟡 □ Connection draining (Thoát Kết Nối) được bật (deregistration delay)?
🟡 □ Idle timeout phù hợp với loại ứng dụng?
       API: 30-60s
       Long-polling, WebSocket: 3600s
🟡 □ Access logs được bật và ghi vào S3?
🟡 □ Deletion protection (Bảo Vệ Xóa) được bật cho production ALB?
🟢 □ Cross-zone load balancing bật cho phân phối traffic đồng đều?
🟢 □ HTTP/2 được bật cho improved performance?
```

### Target Groups (Nhóm Mục Tiêu)

```
🔴 □ Target group không rỗng — có targets đang healthy?
🔴 □ Health check thực sự kiểm tra application health (không phải chỉ TCP)?
🟡 □ Slow start duration phù hợp với app warm-up time?
🟡 □ Load balancing algorithm phù hợp (Round Robin vs Least Outstanding Requests)?
🟢 □ Stickiness (phiên dính) chỉ được bật khi thực sự cần thiết?
```

---

## 🌐 DNS & Route 53 Checklist

### Hosted Zones

```
🔴 □ NS records của domain registrar trỏ đúng vào Route 53 NS records?
🔴 □ Không có typo trong record names (ví dụ: api.exmaple.com thay vì api.example.com)?
🟡 □ SOA record TTL phù hợp với tần suất thay đổi?
🟢 □ Private hosted zone được liên kết với đúng VPCs?
```

### DNS Records

```
🔴 □ A/AAAA records cho apex domain trỏ đúng IP hoặc dùng Alias record?
🔴 □ Không dùng CNAME cho apex domain (dùng Alias thay)?
🟡 □ TTL được điều chỉnh thấp trước khi migration (30-60s)?
🟡 □ MX, TXT (SPF, DKIM) records cho email delivery chính xác?
🟢 □ CAA (Certification Authority Authorization — Ủy Quyền Cấp Chứng Chỉ) records để kiểm soát CAs?
```

### Health Checks & Failover

```
🔴 □ Health checks được cấu hình cho failover records?
🔴 □ Health check endpoint accessible từ Route 53 health checker IPs?
🟡 □ Failover secondary record đang healthy?
🟡 □ TTL cho failover records đủ thấp (60-300s) để failover nhanh?
🟢 □ CloudWatch alarm cho Health Check status?
```

---

## ☁️ CloudFront CDN Checklist

### Distribution Configuration

```
🔴 □ Certificate ở us-east-1 và trạng thái ISSUED?
🔴 □ ViewerProtocolPolicy = redirect-to-https (không cho HTTP plain)?
🔴 □ MinimumProtocolVersion >= TLSv1.2?
🟡 □ Price Class phù hợp với audience geography (vùng người dùng)?
🟡 □ Logging được bật cho production distributions?
🟡 □ Deletion protection được bật?
```

### Cache Behaviors (Hành Vi Cache)

```
🔴 □ Static assets (CSS, JS, images) có TTL cao (86400-31536000s)?
🔴 □ Dynamic API responses có TTL = 0 hoặc no-cache?
🟡 □ Cache key chỉ bao gồm params thực sự cần thiết?
🟡 □ Compression được bật cho text-based content?
🟢 □ Cache Hit Rate > 80% sau deploy?
```

### S3 Origin

```
🔴 □ OAC (Origin Access Control — Kiểm Soát Truy Cập Nguồn Gốc) được dùng (không phải OAI cũ)?
🔴 □ S3 Bucket Policy cho phép CloudFront service principal với đúng distribution ARN?
🔴 □ S3 Block Public Access bật (người dùng chỉ access qua CloudFront)?
🟡 □ S3 Versioning được bật cho rollback capability?
```

---

## 📊 Monitoring & Alerting Checklist

### CloudWatch Alarms (Cảnh Báo CloudWatch)

```
🔴 □ Alarm cho ALB 5xx error rate > 1%?
🔴 □ Alarm cho Unhealthy Host Count > 0?
🔴 □ Alarm cho EC2 CPU > 80%?
🔴 □ Alarm cho RDS Connection Count gần limit?
🟡 □ Alarm cho ALB target response time p99 > threshold?
🟡 □ Alarm cho NAT Gateway error packet count?
🟡 □ Alarm cho VPN tunnel down?
🟡 □ Alarm cho Direct Connect connection state change?
🟢 □ Composite alarms để giảm alert fatigue (Mệt Mỏi Cảnh Báo)?
```

### VPC Flow Logs

```
🔴 □ VPC Flow Logs được bật cho production VPCs?
🟡 □ Flow Logs retention (Thời Gian Lưu Trữ) phù hợp (30-90 ngày cho compliance)?
🟡 □ Flow Logs có thể query được trong CloudWatch Logs Insights hoặc Athena?
🟢 □ Flow Logs được phân tích tự động để phát hiện bất thường?
```

### Logging

```
🔴 □ ALB access logs được bật và ghi vào S3?
🟡 □ CloudFront standard logging hoặc real-time logging được bật?
🟡 □ Route 53 query logging được bật (đặc biệt cho hybrid DNS)?
🟡 □ VPN connection logs được monitor?
🟢 □ Logs được aggregate vào centralized logging solution?
```

---

## 🆘 Disaster Recovery Checklist

### Backup & Recovery

```
🔴 □ RDS automated backups được bật với retention >= 7 ngày?
🔴 □ RDS Multi-AZ được bật cho production databases?
🟡 □ S3 versioning được bật cho critical data buckets?
🟡 □ Cross-region replication được cấu hình cho tier 1 data?
🟢 □ Recovery point objective (RPO) và recovery time objective (RTO) được define và test?
```

### Failover Testing (Kiểm Tra Chuyển Đổi Dự Phòng)

```
🔴 □ DNS failover đã được test thực tế — không chỉ cấu hình trên giấy?
🔴 □ ALB health check failover đã được test?
🟡 □ AZ failure scenario (một AZ mất) đã được test?
🟡 □ NAT Gateway failure simulation đã được test?
🟢 □ Annual/quarterly DR drills (Diễn Tập Khôi Phục) được lên kế hoạch?
```

### Multi-Region (Đa Vùng — nếu applicable)

```
🟡 □ Route 53 latency-based hoặc geolocation routing được cấu hình?
🟡 □ Global Accelerator được dùng cho consistent IP và performance?
🟡 □ Data replication lag được monitor và trong SLA (Thỏa Thuận Mức Dịch Vụ)?
🟢 □ Runbook (Hướng Dẫn Vận Hành) cho regional failover được document?
```

---

## 💰 Cost Optimization Checklist

### NAT Gateway

```
🟡 □ NAT Gateway per AZ (tránh cross-AZ traffic charges — phí traffic giữa AZ)?
🟡 □ VPC Endpoints được dùng cho S3 và DynamoDB (tránh NAT Gateway charges)?
🟢 □ Review NAT Gateway data processed metrics — có workload nào inefficient?
```

### Load Balancer

```
🟡 □ Không có ALB/NLB nào không được dùng (idle load balancers vẫn tính phí)?
🟢 □ Idle load balancers được tagged và có plan review?
```

### CloudFront

```
🟡 □ Price Class phù hợp với actual user geography (không mua edge locations không cần)?
🟢 □ Cache hit ratio cao → giảm origin requests → giảm chi phí?
🟢 □ Compression được bật → giảm data transfer charges?
```

### Direct Connect / VPN

```
🟡 □ Direct Connect capacity phù hợp với actual bandwidth usage?
🟢 □ VPN không được dùng cho traffic > 1 Gbps liên tục (cân nhắc Direct Connect)?
```

---

## 🚀 Deployment Checklist Cuối Cùng

### Trước Khi Deploy (T-24 giờ)

```
🔴 □ Tất cả critical checklist items trên đã hoàn thành?
🔴 □ Staging environment đã được test đầy đủ?
🔴 □ Rollback plan đã được prepare và document?
🔴 □ Team oncall (Trực Sự Cố) đã được thông báo về deployment window?
🟡 □ Change request (Yêu Cầu Thay Đổi) đã được approve?
🟡 □ Monitoring dashboards đã được review trạng thái baseline?
🟡 □ TTL DNS records đã được giảm xuống thấp nếu sẽ thay đổi DNS?
```

### Khi Deploy (T-0)

```
🔴 □ Không deploy vào giờ cao điểm?
🔴 □ Deploy theo từng bước (blue/green hoặc canary) nếu có thể?
🟡 □ Monitor CloudWatch metrics trong suốt quá trình deploy?
🟡 □ Sẵn sàng rollback trong vòng 5 phút nếu có vấn đề?
```

### Sau Deploy (T+30 phút đến T+2 giờ)

```
🔴 □ Tất cả health checks passing (Qua)?
🔴 □ Không có 5xx error spike (Tăng Đột Biến Lỗi) trong ALB/CloudFront metrics?
🔴 □ Kiểm tra chức năng cốt lõi (core functionality) manual?
🟡 □ DNS propagation (Truyền Bá DNS) đã hoàn tất chưa?
🟡 □ Cache behavior như mong đợi (hit/miss rates)?
🟡 □ Error rates về mức baseline trong vòng 15 phút?
🟢 □ Post-deployment monitoring trong 24 giờ đầu?
```

### Documentation (Tài Liệu)

```
🟡 □ Architecture diagram (Sơ Đồ Kiến Trúc) được cập nhật?
🟡 □ Runbook được update với configuration mới?
🟢 □ Network topology document được update?
🟢 □ Cost estimate được update sau changes?
```

---

## 📞 Escalation Matrix (Ma Trận Leo Thang)

```
Sự cố P0 (Production Down):
  T+0m:   On-call engineer tự xử lý
  T+15m:  Escalate lên Tech Lead
  T+30m:  Escalate lên Engineering Manager
  T+60m:  Mở support case với AWS (nếu nghi ngờ AWS service issue)

Sự cố P1 (Service Degraded):
  T+0m:   On-call engineer tự xử lý
  T+30m:  Escalate lên Tech Lead
  T+2h:   Escalate lên Engineering Manager nếu chưa giải quyết

AWS Support Contacts:
  Console: https://console.aws.amazon.com/support/
  CLI: aws support create-case
```

---

## 📚 Tài Liệu Liên Quan

- [1-connectivity-debug.md](./1-connectivity-debug.md) — Debug kết nối EC2
- [2-security-group-nacl-debug.md](./2-security-group-nacl-debug.md) — Debug Security Group & NACL
- [3-dns-issues.md](./3-dns-issues.md) — Debug DNS
- [4-load-balancer-issues.md](./4-load-balancer-issues.md) — Debug Load Balancer
- [5-cloudfront-issues.md](./5-cloudfront-issues.md) — Debug CloudFront
- [../08-monitoring/5-observability-checklist.md](../08-monitoring/5-observability-checklist.md) — Observability Checklist

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
