# 🌐 DNS Issues — Xử Lý Sự Cố Route 53 & DNS Resolution

> Hướng dẫn chẩn đoán các vấn đề DNS (Domain Name System — Hệ Thống Tên Miền) trong AWS, từ Route 53 public/private hosted zones đến DNS resolution failures trong VPC.

---

## 📚 Mục Lục

1. [Tổng Quan DNS Trong AWS](#tổng-quan-dns-trong-aws)
2. [Kịch Bản 1: Public Domain Không Phân Giải Được](#kịch-bản-1-public-domain-không-phân-giải-được)
3. [Kịch Bản 2: Private Hostname Không Phân Giải Trong VPC](#kịch-bản-2-private-hostname-không-phân-giải-trong-vpc)
4. [Kịch Bản 3: DNS Failover Không Tự Động](#kịch-bản-3-dns-failover-không-tự-động)
5. [Kịch Bản 4: Propagation Delay — Thay Đổi DNS Chưa Có Hiệu Lực](#kịch-bản-4-propagation-delay--thay-đổi-dns-chưa-có-hiệu-lực)
6. [Kịch Bản 5: Hybrid DNS — On-Premises Không Phân Giải AWS](#kịch-bản-5-hybrid-dns--on-premises-không-phân-giải-aws)
7. [Công Cụ Debug DNS](#công-cụ-debug-dns)
8. [Checklist Nhanh](#checklist-nhanh)

---

## 🏗️ Tổng Quan DNS Trong AWS

```
Bên ngoài VPC (Public DNS):
  Client → Internet DNS Resolver → Route 53 Public Hosted Zone → IP Public

Bên trong VPC (Private DNS):
  EC2 → 169.254.169.253 (AmazonProvidedDNS) → Route 53 Private Hosted Zone → IP Private
                                              → Public DNS nếu không có private zone
```

### VPC DNS Settings (Cài Đặt DNS VPC)

Hai setting quan trọng trong VPC:

| Setting | Mô Tả | Mặc Định |
|---------|-------|---------|
| `enableDnsSupport` | Bật DNS resolver (169.254.169.253) | `true` |
| `enableDnsHostnames` | EC2 có DNS hostname không | `false` với custom VPC, `true` với default VPC |

Nếu muốn dùng Private Hosted Zone hoặc EC2 DNS hostnames:
```bash
# Bật enableDnsHostnames
aws ec2 modify-vpc-attribute \
  --vpc-id <vpc-id> \
  --enable-dns-hostnames

# Bật enableDnsSupport
aws ec2 modify-vpc-attribute \
  --vpc-id <vpc-id> \
  --enable-dns-support
```

---

## 🔴 Kịch Bản 1: Public Domain Không Phân Giải Được

### Triệu Chứng

```bash
nslookup myapp.example.com
# Server: ...
# ** server can't find myapp.example.com: NXDOMAIN
```

### Luồng Kiểm Tra

```
1. Record tồn tại trong Hosted Zone?
   ↓
2. Hosted Zone đúng domain?
   ↓
3. Name Servers (NS) của domain trỏ đúng vào Route 53?
   ↓
4. TTL cũ còn được cache?
```

### Bước 1: Kiểm Tra Record Tồn Tại

```bash
# Liệt kê tất cả records trong hosted zone
aws route53 list-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --query 'ResourceRecordSets[?Name==`myapp.example.com.`]'
```

**Lưu ý:** Route 53 luôn có dấu chấm (`.`) ở cuối tên domain.

### Bước 2: Kiểm Tra Hosted Zone Đúng Domain

```bash
# Xem tất cả hosted zones
aws route53 list-hosted-zones \
  --query 'HostedZones[].[Name, Id, Config.PrivateZone]'
```

**Lỗi hay gặp:** Tạo record trong `example.com.` nhưng domain thực tế là `example.com.vn.`

### Bước 3: Kiểm Tra Name Servers

Name Servers (Máy Chủ Tên Miền) của domain registrar phải trỏ vào NS records của Route 53 hosted zone.

```bash
# Xem NS records trong Route 53
aws route53 list-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --query 'ResourceRecordSets[?Type==`NS`]'

# So sánh với NS của domain (dùng dig)
dig NS example.com +short

# Kiểm tra delegation từ root
dig +trace myapp.example.com
```

**Nếu NS không khớp:** Vào domain registrar (GoDaddy, Namecheap, v.v.) và cập nhật nameservers.

### Bước 4: Test Trực Tiếp Route 53

```bash
# Kiểm tra Route 53 trả lời gì (bỏ qua cache)
dig @8.8.8.8 myapp.example.com
dig @1.1.1.1 myapp.example.com

# Hỏi trực tiếp NS của Route 53
dig @ns-1234.awsdns-10.org myapp.example.com
```

---

## 🔵 Kịch Bản 2: Private Hostname Không Phân Giải Trong VPC

### Triệu Chứng

```bash
# Từ EC2 trong VPC:
nslookup myservice.internal.example.com
# ** server can't find myservice.internal.example.com: NXDOMAIN
```

### Luồng Kiểm Tra

```
1. VPC có enableDnsSupport = true không?
   ↓
2. Private Hosted Zone tồn tại với đúng domain?
   ↓
3. Private Hosted Zone đã liên kết (associate) với VPC này?
   ↓
4. Record tồn tại trong Private Hosted Zone?
   ↓
5. EC2 đang query đúng DNS server (169.254.169.253)?
```

### Bước 1: Kiểm Tra VPC DNS Settings

```bash
aws ec2 describe-vpc-attribute \
  --vpc-id <vpc-id> \
  --attribute enableDnsSupport

aws ec2 describe-vpc-attribute \
  --vpc-id <vpc-id> \
  --attribute enableDnsHostnames
```

### Bước 2: Kiểm Tra Private Hosted Zone Và VPC Association

```bash
# Xem private hosted zones
aws route53 list-hosted-zones \
  --query 'HostedZones[?Config.PrivateZone==`true`].[Name,Id]'

# Kiểm tra VPC nào được liên kết với hosted zone
aws route53 get-hosted-zone \
  --id <zone-id> \
  --query 'VPCs'
```

**Nếu VPC chưa được liên kết:**
```bash
# Liên kết VPC với Private Hosted Zone
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id <zone-id> \
  --vpc VPCRegion=ap-southeast-1,VPCId=<vpc-id>
```

### Bước 3: Test DNS Từ Trong EC2

```bash
# Kiểm tra DNS server mà EC2 đang dùng
cat /etc/resolv.conf

# Kết quả mong đợi:
# nameserver 10.0.0.2 (AmazonProvidedDNS cho CIDR 10.0.0.0/16 → 10.0.0.2)
# hoặc
# nameserver 169.254.169.253 (Link-local DNS — DNS cục bộ)

# Query trực tiếp AmazonProvidedDNS
nslookup myservice.internal.example.com 169.254.169.253
```

**AmazonProvidedDNS IP:** Luôn là `VPC CIDR base + 2` (ví dụ: VPC 10.0.0.0/16 → DNS server là 10.0.0.2).

### Bước 4: Conflict Giữa Public và Private Hosted Zone

Khi có cả public `example.com` và private `example.com`:
- EC2 trong VPC liên kết với private zone → **chỉ thấy records trong private zone**
- Records trong public zone bị "shadow" (che khuất) bởi private zone

**Giải pháp:** Tạo cùng record trong private zone với giá trị internal IP.

---

## 🟡 Kịch Bản 3: DNS Failover Không Tự Động

### Triệu Chứng

Primary server đã down nhưng traffic vẫn đến primary thay vì failover sang secondary.

### Luồng Kiểm Tra

```
1. Health Check của Route 53 đang báo Healthy hay Unhealthy?
   ↓
2. Health Check endpoint có đúng không?
   ↓
3. TTL của DNS record có quá cao không?
   ↓
4. Routing Policy có phải Failover không?
```

### Bước 1: Kiểm Tra Trạng Thái Health Check

```bash
# Xem tất cả health checks
aws route53 list-health-checks \
  --query 'HealthChecks[].[Id, HealthCheckConfig.FullyQualifiedDomainName, HealthCheckConfig.Port, HealthCheckConfig.Type]'

# Xem trạng thái health check cụ thể
aws route53 get-health-check-status \
  --health-check-id <hc-id> \
  --query 'HealthCheckObservations[].[IPAddress, StatusReport.Status]'
```

### Bước 2: Xem Chi Tiết Health Check Configuration

```bash
aws route53 get-health-check \
  --health-check-id <hc-id> \
  --query 'HealthCheck.HealthCheckConfig'
```

**Điểm kiểm tra:**
- `FullyQualifiedDomainName` hoặc `IPAddress`: endpoint có đúng không?
- `Port`: port có đúng không?
- `ResourcePath`: path `/health` có trả về 200 không?
- `FailureThreshold`: cần bao nhiêu lần fail mới chuyển trạng thái?
- `RequestInterval`: 10s hay 30s?

### Bước 3: Kiểm Tra Security Group Cho Route 53 Health Checkers

Route 53 health checkers cần truy cập endpoint từ IP ranges của AWS.

```bash
# Lấy IP ranges của Route 53 health checkers
curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | \
  python3 -c "import json,sys; data=json.load(sys.stdin); \
  [print(p['ip_prefix']) for p in data['prefixes'] if p.get('service')=='ROUTE53_HEALTHCHECKS']"
```

**Security Group của endpoint phải ALLOW inbound từ những IP này.**

Cách đơn giản hơn: ALLOW inbound từ `0.0.0.0/0` trên port health check, hoặc dùng AWS managed prefix list.

### Bước 4: Kiểm Tra Routing Policy

```bash
# Xem DNS records và routing policy
aws route53 list-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --query 'ResourceRecordSets[?Name==`myapp.example.com.`].[Type,Failover,HealthCheckId,TTL]'
```

**Failover records phải có:**
- Record PRIMARY với `Failover: PRIMARY` và `HealthCheckId`
- Record SECONDARY với `Failover: SECONDARY`

---

## ⏳ Kịch Bản 4: Propagation Delay — Thay Đổi DNS Chưa Có Hiệu Lực

### Hiểu Về TTL (Time To Live — Thời Gian Sống)

```
TTL = 300 giây (5 phút):
  Resolver cache record này trong 5 phút
  → Thay đổi DNS có thể mất tới 5 phút để có hiệu lực

TTL = 86400 giây (1 ngày):
  → Thay đổi DNS có thể mất tới 1 ngày để propagate toàn cầu
```

### Kiểm Tra TTL Hiện Tại

```bash
dig myapp.example.com +noall +answer
# myapp.example.com. 299 IN A 1.2.3.4
#                   ^^^
#                   TTL còn lại trong cache (seconds)
```

### Giảm TTL Trước Khi Thay Đổi Lớn

**Best practice khi chuẩn bị migration (Di Chuyển):**

```
Ngày -7: Giảm TTL từ 86400 → 300
Ngày  0: Thực hiện thay đổi IP (propagation chỉ mất 5 phút)
Ngày +1: Tăng TTL lại 86400 nếu muốn
```

### Kiểm Tra Propagation Toàn Cầu

```bash
# Dùng công cụ trực tuyến hoặc query nhiều DNS servers
dig @8.8.8.8 myapp.example.com     # Google DNS
dig @1.1.1.1 myapp.example.com     # Cloudflare DNS
dig @9.9.9.9 myapp.example.com     # Quad9 DNS
dig @208.67.222.222 myapp.example.com  # OpenDNS

# Flush DNS cache trên máy local
sudo systemd-resolve --flush-caches  # Linux
ipconfig /flushdns                    # Windows
sudo dscacheutil -flushcache          # macOS
```

---

## 🏢 Kịch Bản 5: Hybrid DNS — On-Premises Không Phân Giải AWS

### Mô Hình Hybrid DNS

```
On-Premises Network ← Direct Connect/VPN → AWS VPC
     │                                          │
     ▼                                          ▼
Corporate DNS Server ←─────────────────→ Route 53 Resolver
(10.100.0.1)                              Inbound Endpoint (10.0.1.5)
```

### Bước 1: Kiểm Tra Route 53 Resolver Endpoints

```bash
# Xem Inbound Endpoints (Điểm Cuối Inbound — nhận DNS queries từ on-premises)
aws route53resolver list-resolver-endpoints \
  --filters Name=Direction,Values=INBOUND \
  --query 'ResolverEndpoints[].[Id, Name, Status, IpAddresses[*].Ip]'

# Xem Outbound Endpoints (Điểm Cuối Outbound — gửi DNS queries ra on-premises)
aws route53resolver list-resolver-endpoints \
  --filters Name=Direction,Values=OUTBOUND \
  --query 'ResolverEndpoints[].[Id, Name, Status, IpAddresses[*].Ip]'
```

### Bước 2: Kiểm Tra Resolver Rules

```bash
# Xem forwarding rules (Quy Tắc Chuyển Tiếp)
aws route53resolver list-resolver-rules \
  --query 'ResolverRules[].[Id, Name, DomainName, RuleType, TargetIps]'
```

**Ví dụ rule cần có:**
```
Domain: corp.internal  → Forward đến → 10.100.0.1:53 (Corporate DNS)
```

### Bước 3: Kiểm Tra Security Group Của Resolver Endpoint

Resolver Endpoint có ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi) trong VPC, cần Security Group:
```
Inbound:  UDP/TCP port 53 từ on-premises CIDR
Outbound: UDP/TCP port 53 đến Corporate DNS IP
```

### Bước 4: Test Từ On-Premises

```bash
# Từ on-premises server, query Inbound Endpoint của Route 53 Resolver
nslookup myservice.internal.example.com <inbound-endpoint-ip>

# Nếu không trả lời → kiểm tra Direct Connect/VPN còn up không
# Nếu trả lời sai → kiểm tra Private Hosted Zone và VPC association
```

---

## 🛠️ Công Cụ Debug DNS

### dig — Công Cụ Mạnh Nhất

```bash
# Query cơ bản
dig myapp.example.com

# Chỉ hiện answer section
dig myapp.example.com +short

# Trace đường phân giải từ root
dig +trace myapp.example.com

# Query record type cụ thể
dig TXT myapp.example.com
dig MX example.com
dig NS example.com
dig CNAME www.example.com

# Query DNS server cụ thể
dig @169.254.169.253 myservice.internal

# Xem SOA (Start of Authority — Nguồn Quyền Hạn) record
dig SOA example.com
```

### nslookup

```bash
# Cơ bản
nslookup myapp.example.com

# Chỉ định DNS server
nslookup myapp.example.com 169.254.169.253

# Interactive mode
nslookup
> server 8.8.8.8
> set type=ANY
> example.com
```

### Route 53 Resolver Query Logs (Nhật Ký Truy Vấn)

```bash
# Bật query logging cho VPC
aws route53resolver create-resolver-query-log-config \
  --name "vpc-dns-logs" \
  --destination-arn "arn:aws:logs:ap-southeast-1:123456789:log-group:/aws/route53/query-logs"

# Liên kết với VPC
aws route53resolver associate-resolver-query-log-config \
  --resolver-query-log-config-id <config-id> \
  --resource-id <vpc-id>
```

**Query trong CloudWatch Logs Insights:**
```sql
-- Tìm NXDOMAIN responses
fields @timestamp, queryName, queryType, responseCode
| filter responseCode = "NXDOMAIN"
| sort @timestamp desc
| limit 100

-- Xem queries cho một domain cụ thể
fields @timestamp, srcAddr, queryName, responseCode
| filter queryName like /myservice/
| sort @timestamp desc
```

---

## ✅ Checklist Nhanh

### Public DNS

```
□ Record tồn tại trong hosted zone (kiểm tra tên chính xác, có dấu . cuối)?
□ Domain registrar nameservers trỏ đúng vào Route 53 NS records?
□ Record type đúng (A, CNAME, ALIAS)?
□ CNAME không được dùng cho apex domain (example.com) — dùng ALIAS thay?
□ TTL đã hết hạn cache cũ?
```

### Private DNS Trong VPC

```
□ VPC enableDnsSupport = true?
□ VPC enableDnsHostnames = true (nếu cần EC2 DNS names)?
□ Private Hosted Zone đã associate với VPC?
□ Record tồn tại trong private hosted zone?
□ Không có conflict với public zone cùng tên?
□ EC2 đang dùng AmazonProvidedDNS (không phải custom DNS server)?
```

### DNS Failover

```
□ Health Check trạng thái thực sự là Unhealthy?
□ Security Group endpoint cho phép Route 53 health checker IPs?
□ Routing Policy = Failover?
□ Record PRIMARY có Health Check ID được gán?
□ TTL đủ thấp để failover nhanh (60-300s)?
```

### Hybrid DNS

```
□ Resolver Inbound/Outbound Endpoint ở trạng thái Operational?
□ Forwarding rules có đúng domain → đúng DNS server IP?
□ Security Group của Resolver Endpoint cho phép port 53?
□ Direct Connect/VPN tunnel còn active?
□ On-premises DNS configured để forward AWS domains đến Inbound Endpoint?
```

---

## 📚 Tài Liệu Liên Quan

- [../04-dns-route53/4-health-checks.md](../04-dns-route53/4-health-checks.md) — Health Checks chi tiết
- [../04-dns-route53/5-resolver.md](../04-dns-route53/5-resolver.md) — Route 53 Resolver & Hybrid DNS
- [../04-dns-route53/3-routing-policies.md](../04-dns-route53/3-routing-policies.md) — Routing Policies

---

**Cập Nhật Lần Cuối:** 2026-05-14
