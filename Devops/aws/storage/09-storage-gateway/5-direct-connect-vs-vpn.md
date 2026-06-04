# 🔌 Direct Connect vs VPN — Kết Nối Mạng Cho Hybrid Storage

> Hai giải pháp chính để kết nối on-premises với AWS cho hybrid storage: **AWS Direct Connect** (Kết Nối Trực Tiếp Vật Lý) và **AWS Site-to-Site VPN** (Mạng Riêng Ảo Site-to-Site). Lựa chọn đúng ảnh hưởng trực tiếp đến hiệu suất, độ tin cậy, và chi phí của Storage Gateway, DataSync, và toàn bộ kiến trúc hybrid.

---

## 📚 Mục Lục

1. [Tổng Quan So Sánh](#tổng-quan-so-sánh)
2. [AWS Direct Connect](#aws-direct-connect)
3. [AWS Site-to-Site VPN](#aws-site-to-site-vpn)
4. [Kiến Trúc Hybrid Storage Điển Hình](#kiến-trúc-hybrid-storage)
5. [Decision Framework](#decision-framework)
6. [HA và Resilience Patterns](#ha-và-resilience-patterns)
7. [Chi Phí So Sánh](#chi-phí-so-sánh)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan So Sánh

### Bảng So Sánh Toàn Diện

| Tiêu Chí | Direct Connect | Site-to-Site VPN |
|----------|----------------|------------------|
| **Loại kết nối** | Đường truyền vật lý riêng | IPSec tunnel qua internet |
| **Băng thông** | 1/10/100 Gbps | Tối đa ~1.25 Gbps |
| **Độ trễ (Latency)** | Thấp, ổn định (1-5ms) | Cao hơn, biến động (10-100ms) |
| **Jitter** (Biến Động Trễ) | Rất thấp | Cao (phụ thuộc internet) |
| **Tính nhất quán** | Cao (dedicated line) | Thấp (best-effort internet) |
| **Encryption** | Không mặc định (cần thêm MACsec) | IPSec — có sẵn |
| **Setup time** | 30-90 ngày | Vài giờ |
| **Chi phí setup** | Cao ($1,000-10,000 port fee) | Thấp (không cần) |
| **Chi phí hàng tháng** | Cao ($200-2,000+/tháng) | Thấp ($36-72/tháng) |
| **SLA** | 99.9% uptime | Không có SLA |
| **Compliance** | Mạnh (traffic không qua internet) | Vừa (mã hóa nhưng qua internet) |

---

## AWS Direct Connect

### Cách Hoạt Động

```
┌──────────────────┐         ┌─────────────────────────┐
│  On-Premises      │         │  AWS Direct Connect     │
│  Datacenter       │         │  Location               │
│                   │         │  (Colocation Facility)  │
│  Customer Router ─┼─────────┼─► Customer Router       │
│                   │         │         │               │
│  Cross Connect    │         │  AWS Router             │
│  (Cáp vật lý)     │         │         │               │
└──────────────────┘         └─────────┼───────────────┘
                                       │  AWS Backbone Network
                                       │  (Mạng Xương Sống AWS)
                                       ▼
                              ┌──────────────────────┐
                              │    AWS Region         │
                              │  ┌────────────────┐  │
                              │  │   VPC          │  │
                              │  │  ┌──────────┐  │  │
                              │  │  │  EC2/    │  │  │
                              │  │  │  Storage │  │  │
                              │  │  └──────────┘  │  │
                              │  └────────────────┘  │
                              └──────────────────────┘
```

### Thành Phần Direct Connect

```
1. DX Location (Vị Trí Direct Connect):
   ├── Cơ sở colocation của bên thứ ba (Equinix, Digital Realty)
   ├── AWS có thiết bị tại đây
   └── Khách hàng đặt router tại đây hoặc thuê đối tác

2. Cross Connect (Kết Nối Chéo):
   ├── Cáp vật lý từ router khách hàng → router AWS
   └── Thuê từ cơ sở colocation

3. Virtual Interface — VIF (Giao Diện Ảo):
   ├── Private VIF: kết nối vào VPC cụ thể
   ├── Public VIF: kết nối vào AWS public services (S3, DynamoDB)
   └── Transit VIF: kết nối vào AWS Transit Gateway
```

### Bandwidth Options (Tùy Chọn Băng Thông)

```
Dedicated Connection (Kết Nối Riêng):
├── 1 Gbps
├── 10 Gbps
└── 100 Gbps

Hosted Connection (Kết Nối Chia Sẻ qua đối tác):
├── 50 Mbps, 100 Mbps, 200 Mbps, 300 Mbps, 400 Mbps, 500 Mbps
├── 1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps
└── Nhanh hơn Dedicated, chia sẻ port với khách hàng khác của đối tác
```

### MACsec — Mã Hóa Layer 2

```
Direct Connect không mã hóa mặc định!
→ Traffic đi qua cáp vật lý shared với khách hàng khác ở DX Location

MACsec (Media Access Control Security — Bảo Mật Điều Khiển Truy Cập Phương Tiện):
├── Mã hóa hop-by-hop tại Layer 2
├── Hỗ trợ trên 1Gbps, 10Gbps, 100Gbps Dedicated
└── Yêu cầu router hỗ trợ MACsec (Cisco, Juniper mới nhất)

Nếu không dùng MACsec:
└── Dùng IPSec VPN over Direct Connect để mã hóa (tốt nhất)
```

### Tốt Cho Storage Workloads

```
Storage use cases phù hợp Direct Connect:

1. Storage Gateway File/Volume (production):
   ├── Đọc/ghi liên tục cần latency < 10ms
   └── Throughput cao: Direct Connect 10Gbps = 1.25 GB/s

2. DataSync bulk migration:
   ├── Transfer 500TB cần 10Gbps Direct Connect để hoàn thành trong vài ngày
   └── Với internet 1Gbps: ~500 ngày → không thực tế

3. Database với EBS/cloud storage:
   └── Oracle RAC, SQL Server cần block latency ổn định

4. Real-time analytics:
   └── Kinesis Firehose ingestion từ on-prem qua Direct Connect
```

---

## AWS Site-to-Site VPN

### Cách Hoạt Động

```
┌──────────────────┐                    ┌───────────────────────┐
│  On-Premises      │                    │  AWS Cloud            │
│                   │                    │                       │
│  Customer Gateway │                    │  Virtual Private      │
│  (CGW — Thiết Bị  │  IPSec Tunnel #1   │  Gateway (VGW) hoặc  │
│  VPN Của Khách) ──┼────────────────────┼─► AWS Transit Gateway │
│                   │  IPSec Tunnel #2   │                       │
│  (2 tunnels HA)  ─┼────────────────────┼─►  (2 tunnels riêng  │
│                   │  (qua internet)    │   biệt cho HA)        │
└──────────────────┘                    └───────────────────────┘
                         INTERNET
                      (Best-effort)
```

### VPN Tunnels

```
Mỗi VPN connection = 2 tunnels tự động (active/standby hoặc ECMP):

Tunnel 1: on-prem router → AWS endpoint A (us-east-1a)
Tunnel 2: on-prem router → AWS endpoint B (us-east-1b)

→ Nếu một tunnel down → failover sang tunnel còn lại (~30s)
→ 2 tunnels có thể ECMP (Equal Cost Multi-Path) cho throughput đôi
```

### Accelerated VPN (VPN Tăng Tốc)

```
Vấn đề thông thường: VPN qua internet → latency biến động
Giải pháp: Accelerated Site-to-Site VPN

Accelerated VPN:
├── Dùng AWS Global Accelerator network (mạng backbone AWS)
├── Traffic từ on-prem → AWS edge location gần nhất → AWS backbone
└── Giảm latency và jitter đáng kể

So sánh:
Regular VPN: on-prem → internet → AWS endpoint
Accelerated VPN: on-prem → AWS edge (20ms) → AWS backbone → AWS endpoint

Phù hợp cho: Storage Gateway khi không muốn chi phí Direct Connect
```

### Giới Hạn VPN

```
Bandwidth:
├── Mỗi tunnel: ~1.25 Gbps (giới hạn cứng)
├── 2 tunnels ECMP: ~2.5 Gbps tối đa
└── So với Direct Connect 10 Gbps: thấp hơn 4x

Latency:
├── Phụ thuộc internet routing
├── Cross-continent: 100-300ms
└── Không đảm bảo — biến động theo lưu lượng internet

Suitability cho Storage:
├── OK: Backup, DataSync scheduled sync
└── Không OK: Real-time database I/O, production file server
```

---

## Kiến Trúc Hybrid Storage

### Pattern 1: Direct Connect + VPN Backup

```
Thiết kế HA (High Availability — Tính Sẵn Sàng Cao) tốt nhất:

On-Premises
     │
     ├── Direct Connect 10Gbps ──────────────────► AWS VPC
     │   (Primary — Kết Nối Chính)                    │
     │                                                  │
     └── Site-to-Site VPN (Internet) ─────────────────► VGW
         (Backup — Dự Phòng)                           │
                                                        │
                                               Storage Gateway
                                               (File/Volume)

Behavior:
- Normal: traffic qua Direct Connect (thấp latency, cao bandwidth)
- DX down: BGP failover sang VPN (~60s)
- DX restored: BGP failback về Direct Connect

Cost: Direct Connect + VPN ≈ $1,200-2,500/tháng
```

### Pattern 2: VPN Only (Tiết Kiệm Chi Phí)

```
Dùng cho: Backup workloads, không-production

On-Premises
     │
     └── Site-to-Site VPN ────────────────────► AWS VPC
         (2 Tunnels Active/Active)                   │
                                                      │
                                             Tape Gateway
                                             DataSync
                                             (Backup, off-hours)

Phù hợp khi:
├── Backup chạy ban đêm (không bị ảnh hưởng latency ban ngày)
├── DataSync weekly migration (không cần real-time)
└── Môi trường dev/test

Cost: ~$73/tháng
```

### Pattern 3: Public VIF + S3 VPC Endpoint

```
Tối ưu cho File Gateway ↔ S3 qua Direct Connect:

On-Premises → Direct Connect → Public VIF → S3 Direct

Lưu ý: Public VIF kết nối vào AWS public services như S3
(không cần VPC, không cần VGW)

Lợi ích:
├── Băng thông cao cho S3 uploads
├── Không traffic qua internet
└── Giảm latency cho File Gateway

Nhược điểm:
└── Traffic không mã hóa theo mặc định
   → Thêm IPSec tunnel over Direct Connect để mã hóa
```

---

## Decision Framework

### Khi Nào Dùng Direct Connect

```
Dùng Direct Connect khi:

✅ Bandwidth cần > 2 Gbps thường xuyên
✅ Latency < 5ms là yêu cầu cứng (database, real-time)
✅ Compliance yêu cầu traffic KHÔNG qua internet
✅ Production Storage Gateway liên tục (File/Volume)
✅ DataSync migration lớn (>100TB, cần hoàn thành trong tuần)
✅ Budget cho phép ($500-2,000+/tháng)

Không phù hợp khi:
❌ Setup time quan trọng (cần trong vài ngày) → DX cần 30-90 ngày
❌ Temporary workload (migration ngắn hạn)
❌ Budget thắt chặt
```

### Khi Nào Dùng VPN

```
Dùng VPN khi:

✅ Backup và archive (latency không quan trọng)
✅ DataSync scheduled sync (chạy ban đêm, không real-time)
✅ Tape Gateway (throughput nhỏ, đã có buffer)
✅ Dev/test environment
✅ Cần kết nối trong vài giờ (không chờ DX)
✅ Budget hạn chế
✅ Backup kết nối cho Direct Connect chính

Không phù hợp khi:
❌ Production Storage Gateway liên tục cần thấp latency
❌ Transfer >50TB thường xuyên (bottleneck 1.25 Gbps)
❌ Compliance yêu cầu traffic riêng biệt hoàn toàn
```

### Decision Tree (Cây Quyết Định)

```
Tôi cần kết nối hybrid gì cho storage?

Q1: Bandwidth yêu cầu?
├── > 2 Gbps thường xuyên → Direct Connect
└── < 2 Gbps → tiếp Q2

Q2: Latency yêu cầu?
├── < 5ms ổn định → Direct Connect
└── OK với 10-100ms → tiếp Q3

Q3: Compliance?
├── Traffic phải không qua internet → Direct Connect
└── Encryption là đủ → tiếp Q4

Q4: Timeline?
├── Cần trong vài ngày → VPN (DX mất 30-90 ngày)
└── OK chờ 1-3 tháng → Direct Connect

Q5: Budget?
├── Budget cho phép → Direct Connect (tốt hơn)
└── Budget thắt chặt → VPN + Accelerated VPN
```

---

## HA và Resilience Patterns

### Direct Connect HA

```
Level 1: Single DX Connection (Không HA)
├── 1 DX location, 1 DX connection
└── SPOF (Single Point of Failure — Điểm Lỗi Đơn)

Level 2: Redundant DX Connections
├── 2 DX connections tại cùng DX location
└── Chịu được lỗi router/port, không chịu được lỗi DX location

Level 3: DX + VPN Backup
├── DX primary + VPN backup qua internet
└── Chịu được DX location down
→ Khuyến nghị cho hầu hết production

Level 4: Dual DX Locations (Tốt Nhất)
├── 2 DX connections tại 2 DX locations khác nhau
└── Chịu được lỗi hoàn toàn một DX location
→ Cho critical production, financial, healthcare
```

### Storage Gateway HA Patterns

```
File/Volume Gateway HA:

Option 1: Multiple Gateways
├── 2 gateway VMs cùng site
├── NFS/iSCSI load balance
└── Một gateway down → traffic sang gateway còn lại

Option 2: Single Gateway + Cloud Failover
├── 1 gateway on-prem
├── Data backup trên S3
└── Nếu on-prem down: launch Gateway trên EC2 (RTO ~1 giờ)

CloudWatch Alarm:
└── Monitor gateway health → SNS notification → auto-remediation Lambda
```

---

## Chi Phí So Sánh

### Direct Connect Chi Phí

```
Dedicated Connection 1 Gbps:
├── Port fee: $0.30/giờ = ~$216/tháng
├── Data transfer out (về on-prem): $0.02-0.09/GB (tùy region)
└── Data transfer in (lên AWS): miễn phí

Dedicated Connection 10 Gbps:
├── Port fee: $0.15/giờ = ~$108/tháng (per Gbps rẻ hơn)
└── (Hiệu quả hơn nếu thực sự dùng băng thông cao)

Hosted Connection (qua đối tác):
├── 50Mbps–500Mbps: $0.03/giờ = ~$22-72/tháng
└── 1Gbps: $0.06/giờ = ~$43/tháng

Tổng Direct Connect (1Gbps, us-east-1):
├── Port: ~$216/tháng
└── Data egress 10TB: 10,000 × $0.02 = $200/tháng
Total: ~$416/tháng
```

### VPN Chi Phí

```
Site-to-Site VPN connection:
├── VPN connection fee: $0.05/giờ = ~$36/tháng per connection
├── Data transfer out: $0.09/GB (standard rate)
└── Accelerated VPN: thêm ~$0.02/GB + AWS Global Accelerator fee

Tổng VPN thông thường:
├── Connection: ~$36/tháng
└── Data egress 10TB: 10,000 × $0.09 = $900/tháng
Total: ~$936/tháng

→ VPN RẺ hơn Direct Connect cho setup fee, nhưng data egress có thể đắt hơn!
```

### So Sánh Tổng Chi Phí (TCO)

```
Scenario: 10TB data transfer/tháng, 3 năm

Direct Connect 1Gbps:
├── Setup (cross connect, LOA-CFA): ~$2,000 một lần
├── Monthly: ~$416 × 36 = $14,976
└── Total 3 năm: ~$16,976

Site-to-Site VPN:
├── Setup: $0
├── Monthly: ~$936 × 36 = $33,696
└── Total 3 năm: ~$33,696

→ Direct Connect rẻ hơn ~50% sau 3 năm (data egress là yếu tố lớn)
→ VPN: chi phí thấp ban đầu, đắt dần theo data volume

Break-even point: ~6 tháng (Direct Connect trở nên có lợi hơn)
```

---

## Câu Hỏi Phỏng Vấn

**Q1: Khi nào dùng Direct Connect thay vì VPN cho hybrid storage?**

> Direct Connect khi: bandwidth >2Gbps, latency <5ms là yêu cầu, compliance không cho phép traffic qua internet, production Storage Gateway liên tục. VPN khi: backup không real-time, budget hạn chế, cần kết nối trong vài ngày, workload tạm thời.

**Q2: Direct Connect có mã hóa traffic không?**

> Không mặc định. Direct Connect là đường vật lý dedicated, nhưng traffic không mã hóa. Để mã hóa:
> - **MACsec**: mã hóa Layer 2 cho Dedicated Connection (cần router hỗ trợ)
> - **IPSec VPN over Direct Connect**: chạy VPN tunnel qua DX để có cả tốc độ lẫn mã hóa — đây là pattern phổ biến nhất trong production.

**Q3: Thiết kế HA cho hybrid storage nếu Direct Connect down?**

> Pattern tốt nhất: Direct Connect primary + Site-to-Site VPN backup. BGP route preference ưu tiên DX (lower MED/AS path). Khi DX down, BGP tự động chuyển traffic sang VPN (~60s). Khi DX restore, BGP failback. Storage Gateway sẽ tiếp tục hoạt động, chỉ chậm hơn khi trên VPN.

**Q4: Accelerated VPN là gì và khi nào dùng?**

> Accelerated Site-to-Site VPN sử dụng AWS Global Accelerator để route traffic từ on-prem đến AWS edge location gần nhất, sau đó đi qua AWS backbone (thay vì internet công cộng). Giảm latency 20-40% và ổn định hơn. Dùng khi muốn VPN nhưng cần performance tốt hơn — chi phí thêm ~$0.02/GB so với VPN thường.

**Q5: Tại sao Direct Connect cần 30-90 ngày setup?**

> Direct Connect yêu cầu:
> 1. Chọn DX location (colocation facility) gần datacenter on-prem
> 2. Gửi LOA-CFA (Letter of Authorization — Thư Ủy Quyền) cho facility
> 3. Facility kéo cross-connect cáp vật lý (~1-4 tuần)
> 4. Cấu hình BGP peering giữa router khách và router AWS
> 5. Test và verify
> → Toàn bộ quá trình vật lý + logistic → không thể instant như VPN

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
