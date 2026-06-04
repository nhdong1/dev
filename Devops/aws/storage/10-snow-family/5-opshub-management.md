# OpsHub — Giao Diện Quản Lý Snow Devices

> AWS OpsHub là ứng dụng **GUI — Graphical User Interface — Giao Diện Đồ Họa** miễn phí giúp quản lý thiết bị Snow Family mà không cần dùng CLI. Nó chạy trên laptop của bạn và kết nối trực tiếp đến thiết bị Snow qua mạng cục bộ — không cần internet.

---

## Tại Sao OpsHub Ra Đời?

Trước OpsHub (ra mắt 2020), người dùng phải dùng **Snowball Edge Client CLI**:

```bash
# Cách cũ — CLI phức tạp
snowballEdge configure \
  --endpoint https://192.168.1.100 \
  --manifest-file /home/user/manifest.bin \
  --unlock-code 12345-67890-12345-67890-12345

snowballEdge describe-device
snowballEdge start-service --service-id s3

aws s3 cp /local/data s3://mybucket/ \
  --recursive \
  --endpoint-url https://192.168.1.100:8443 \
  --no-verify-ssl

# Cần biết nhiều lệnh, dễ sai typo
```

OpsHub thay thế bằng giao diện kéo-thả trực quan, phù hợp với **IT generalist** hoặc người không quen CLI.

---

## Cài Đặt OpsHub

```
Hệ điều hành hỗ trợ:
  - macOS 10.14+
  - Windows 10 / Windows Server 2016+
  - Linux (Ubuntu 18.04+, RHEL 7+)

Tải về:
  - Console: Snow Family → Create Job → Download OpsHub
  - Hoặc tìm "AWS OpsHub" trên aws.amazon.com

Yêu cầu kết nối:
  - Laptop và Snow device phải cùng mạng LAN
  - Không cần internet (offline hoàn toàn)
  - Port: 443 (HTTPS) + 8443

Kích thước: ~200 MB, không cần AWS credentials cài đặt
```

---

## Tính Năng Chính

### 1. Unlock Device (Mở Khóa Thiết Bị)

```
Quy trình:
  1. Mở OpsHub trên laptop
  2. Nhập IP của Snow device
  3. Import manifest file (.bin) từ S3 Job
  4. Nhập Unlock Code (29 ký tự)
  5. OpsHub kết nối và unlock thiết bị

Lưu ý:
  - Manifest + Unlock Code là 2 yếu tố xác thực độc lập
  - Sai unlock code quá nhiều lần → thiết bị lock hoàn toàn
  - Cần liên hệ AWS Support để reset
```

### 2. Quản Lý File (File Management)

```
Trong OpsHub:
  ┌────────────────────────────────────────────┐
  │  📁 Local Computer     📦 Snow Device      │
  │  ├── /data/             ├── s3://bucket1/  │
  │  ├── /backup/           ├── s3://bucket2/  │
  │  └── /media/            └── nfs://share/   │
  │                                            │
  │  [Drag & Drop] hoặc [Copy] / [Sync]        │
  │  Progress bar với estimated time           │
  └────────────────────────────────────────────┘

Hỗ trợ:
  - Drag & drop từ Explorer/Finder
  - Folder copy với recursive
  - Filter theo file type, date, size
  - Pause và resume transfer
```

### 3. Quản Lý EC2 Instances (Trên Thiết Bị)

```
OpsHub → Compute → Launch Instance

Giống AWS Console nhưng cho local device:
  - Chọn AMI (Amazon Machine Image) đã upload lên device
  - Chọn instance type (sbe1.large, sbe-c.xlarge,...)
  - Cấu hình VPC, security group
  - Launch, Stop, Start, Terminate

Monitor instance:
  - CPU usage
  - Memory usage
  - Network I/O
  - Console output (như EC2 serial console)
```

### 4. Quản Lý Network

```
Cấu hình IP:
  - Giao diện mạng (RJ-45, SFP+)
  - Static IP hoặc DHCP
  - MTU — Maximum Transmission Unit — Đơn Vị Truyền Tối Đa

View network stats:
  - Throughput hiện tại (MB/s)
  - Packets received/sent
  - Error rate
  - Connection status
```

### 5. Device Status Dashboard

```
Tổng quan real-time:
  ┌─────────────────────────────────────────────┐
  │  Snowball Edge Storage Optimized            │
  │  Status: ✅ Unlocked and Ready              │
  │                                             │
  │  Storage:  45.2 TB / 80 TB used (56%)      │
  │  [████████████░░░░░░░░░░░░░░░░░]           │
  │                                             │
  │  Network:  ↑ 987 MB/s  ↓ 12 MB/s          │
  │  Temp:     Operating normally (35°C)        │
  │  Uptime:   3 days 14 hours                  │
  │                                             │
  │  Active transfers: 3                        │
  │  Files copied: 1,247,832                    │
  └─────────────────────────────────────────────┘
```

---

## Cluster Management (Quản Lý Cụm Thiết Bị)

```
Khi có Snowball Edge cluster (3–16 thiết bị):

OpsHub → Devices → View Cluster

  Device 1 (Primary):  ✅ Healthy  |  28.4 TB
  Device 2:            ✅ Healthy  |  27.9 TB
  Device 3:            ⚠️ Warning  |  28.1 TB  ← 1 disk degraded

Actions available:
  - Initiate data rebalancing
  - View cluster health
  - Remove degraded node
  - Add replacement node

Tự động failover:
  - OpsHub hiển thị cảnh báo ngay khi node có vấn đề
  - Cluster vẫn online với N-1 node (erasure coding)
```

---

## OpsHub vs CLI — Khi Nào Dùng Gì?

| Tình Huống | OpsHub | CLI |
|---|---|---|
| IT generalist không quen CLI | ✅ Tốt hơn | ❌ Khó |
| Automation (scripting hàng loạt) | ❌ Không hỗ trợ | ✅ Tốt hơn |
| Copy nhanh vài folder | ✅ Drag-drop | ⚠️ Cần lệnh |
| Monitoring cluster nhiều thiết bị | ✅ Dashboard | ⚠️ Phải query từng cái |
| Tích hợp vào CI/CD pipeline | ❌ Không hỗ trợ | ✅ Bắt buộc |
| Troubleshoot real-time | ✅ Visual | ✅ Raw output |
| Training người mới | ✅ Intuitive | ❌ Steep learning curve |

---

## Kịch Bản Thực Tế Sử Dụng OpsHub

### Kịch Bản 1: Nhân Viên IT Copy Dữ Liệu Văn Phòng

```
Bối cảnh: Nhân viên IT không rành AWS nhận Snowball Edge
Dùng OpsHub:

1. Mở OpsHub → Connect Device → Nhập IP: 192.168.10.50
2. Upload manifest.bin từ S3 (đã tải trước)
3. Nhập unlock code từ email AWS
4. Thiết bị unlock → Dashboard hiện lên
5. Kéo thả D:\CompanyData\ vào s3://migration-bucket/
6. Nhìn progress bar, đợi xong
7. Xác nhận files count khớp
8. Gói thiết bị, ship về AWS

→ Hoàn thành mà không cần biết AWS CLI
```

### Kịch Bản 2: Kỹ Sư Quản Lý Cluster 5 Thiết Bị

```
Bối cảnh: Migration 400TB, dùng 5 Snowball Edge
Dùng OpsHub:

1. OpsHub → Cluster View → Connect tất cả 5 device
2. Xem storage allocation trên từng device
3. Phân bổ data source cho từng device
4. Monitor throughput tổng (giải quyết bottleneck)
5. Device 3 báo disk warning → điều chỉnh tải
6. Export report (CSV) tổng kết files transferred

→ Quản lý cluster phức tạp qua 1 giao diện
```

---

## Giới Hạn Của OpsHub

```
❌ Không hỗ trợ automation/scripting
❌ Không có REST API để integrate với tool khác
❌ Không chạy headless (cần màn hình)
❌ Phải cài trên từng laptop — không phải web interface
❌ Không sync preferences giữa các máy
❌ Chỉ kết nối local network (không remote qua internet)
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**Q: OpsHub có thể thay thế hoàn toàn Snowball Edge Client CLI không?**

> Cho người dùng thông thường, OpsHub đủ dùng. Nhưng cho automation, scripting, hoặc tích hợp vào pipeline, vẫn cần CLI. Trong thực tế, nhiều team dùng OpsHub để monitor và CLI để transfer để tận dụng ưu điểm của cả hai.

**Q: OpsHub có tính phí không?**

> Miễn phí hoàn toàn — đây là công cụ quản lý đi kèm Snow Family. Chi phí chỉ tính trên thiết bị Snow và data transfer, không phải OpsHub.

**Q: Nếu không có OpsHub, làm sao quản lý Snow device?**

> Dùng Snowball Edge Client CLI (snowballEdge commands) + AWS S3 API với endpoint URL trỏ vào device IP. Cả hai đều hoạt động offline, chỉ cần kết nối LAN đến thiết bị.

---

**Trạng Thái:** ✅ Hoàn thành  
**Cập Nhật:** 2026-05-16  
**Liên Kết:** [README](./README.md) | [Snowcone](./1-snowcone.md) | [Snow vs DataSync](./4-snow-vs-datasync.md)
