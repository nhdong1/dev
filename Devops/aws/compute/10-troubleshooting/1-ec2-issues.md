# 🖥️ Sự Cố EC2 — Chẩn Đoán & Khắc Phục

> Hướng dẫn xử lý có hệ thống các sự cố phổ biến nhất trên EC2: không kết nối được, instance không khởi động, hiệu suất kém, và phục hồi tự động.

## 📚 Mục Lục

1. [Không Kết Nối SSH/RDP](#1-không-kết-nối-sshrдp)
2. [EC2 Status Checks Thất Bại](#2-ec2-status-checks-thất-bại)
3. [Instance Không Khởi Động](#3-instance-không-khởi-động)
4. [Hiệu Suất Kém — CPU/Memory/Disk](#4-hiệu-suất-kém)
5. [Phục Hồi Tự Động — Auto Recovery](#5-phục-hồi-tự-động)
6. [Sự Cố Mạng & Connectivity](#6-sự-cố-mạng--connectivity)

---

## 1. Không Kết Nối SSH/RDP

### Sơ Đồ Chẩn Đoán

```
Không kết nối được SSH/RDP
│
├── Instance đang Running không?
│   ├── NO → Start instance, chờ Status Checks xanh
│   └── YES → Tiếp tục
│
├── Security Group có mở đúng port?
│   ├── SSH (Linux): port 22 từ IP của bạn
│   ├── RDP (Windows): port 3389 từ IP của bạn
│   └── Kiểm tra: EC2 Console → Security → Security Groups
│
├── Network ACL — NACL (Danh Sách Kiểm Soát Mạng) có chặn không?
│   ├── NACL là stateless — cần mở cả inbound VÀ outbound
│   └── Kiểm tra: VPC Console → Network ACLs → Inbound/Outbound Rules
│
├── Route Table có route đến Internet Gateway không? (nếu Public Subnet)
│   └── 0.0.0.0/0 → igw-xxxxxxxx
│
├── Elastic IP hoặc Public IP được gán chưa?
│   └── EC2 không có Public IP → không kết nối từ internet được
│
└── Key Pair đúng không? (Linux)
    ├── Dùng đúng file .pem cho instance
    └── chmod 400 my-key.pem
```

### Checklist Xử Lý SSH Không Được

```bash
# Bước 1: Xác nhận instance đang chạy
aws ec2 describe-instances \
  --instance-ids i-1234567890abcdef0 \
  --query 'Reservations[].Instances[].State.Name'

# Bước 2: Kiểm tra Security Group rules
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxx \
  --query 'SecurityGroups[].IpPermissions'

# Bước 3: Kiểm tra IP public của instance
aws ec2 describe-instances \
  --instance-ids i-1234567890abcdef0 \
  --query 'Reservations[].Instances[].[PublicIpAddress,PublicDnsName]'

# Bước 4: Test kết nối TCP (từ máy local)
nc -zv <public-ip> 22 -w 5
# Kết quả "Connection refused" → OS nhận packet nhưng port đóng (OS issue)
# Kết quả "Connection timed out" → Security Group hoặc NACL chặn

# Bước 5: Kết nối thử bằng SSM (không cần port 22)
aws ssm start-session --target i-1234567890abcdef0
```

### Giải Pháp Khi Mất Key Pair

Nếu mất file `.pem` và không SSH được:

**Cách 1 — SSM Session Manager (Khuyến Nghị)**
```bash
# Không cần SSH, không cần port 22
# Yêu cầu: SSM Agent chạy trên instance + IAM role có ssm:StartSession
aws ssm start-session --target i-1234567890abcdef0
```

**Cách 2 — EC2 Instance Connect**
```bash
# Tạo tạm thời SSH key và push vào instance (hết hiệu lực sau 60 giây)
aws ec2-instance-connect send-ssh-public-key \
  --instance-id i-1234567890abcdef0 \
  --availability-zone us-east-1a \
  --instance-os-user ec2-user \
  --ssh-public-key file://temp-key.pub
ssh -i temp-key ec2-user@<public-ip>
```

**Cách 3 — Detach & Reattach Volume (Phương Án Cuối)**
```
1. Stop instance
2. Detach root EBS volume (ổ đĩa gốc)
3. Attach volume vào instance "cứu hộ" khác
4. Mount volume và sửa ~/.ssh/authorized_keys
5. Detach từ instance cứu hộ, reattach vào instance gốc
6. Start lại instance gốc
```

---

## 2. EC2 Status Checks Thất Bại

### Hai Loại Status Check

EC2 có hai loại kiểm tra sức khỏe khác nhau:

| Loại | Tên | Kiểm Tra Gì | Ai Xử Lý |
|------|-----|------------|----------|
| **System Status Check** | Kiểm Tra Hệ Thống | Phần cứng, mạng, nguồn điện phía AWS | AWS tự xử lý |
| **Instance Status Check** | Kiểm Tra Instance | OS boot, kernel, network interface | Bạn xử Lý |

### System Status Check Thất Bại

```
Nguyên nhân: Phần cứng vật lý phía AWS bị lỗi
Giải pháp:
  1. Stop & Start instance (KHÔNG phải Reboot)
     → AWS sẽ di chuyển instance sang phần cứng mới
  2. Hoặc bật Auto Recovery (xem phần 5)

# Stop & Start (không dùng reboot — reboot giữ nguyên host vật lý)
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 start-instances --instance-ids i-1234567890abcdef0
```

> **Lưu ý quan trọng:** `Reboot` và `Stop+Start` khác nhau. `Reboot` giữ nguyên phần cứng vật lý (host). `Stop+Start` chuyển sang phần cứng mới — đây là cách để thoát khỏi host bị lỗi.

### Instance Status Check Thất Bại

```
Nguyên nhân: OS, kernel, hoặc network driver bị lỗi
Các bước xử lý:

1. Xem System Console Output (Đầu Ra Console Hệ Thống) để xem boot logs
   aws ec2 get-console-output --instance-id i-1234567890abcdef0

2. Dấu hiệu thường gặp:
   - "GRUB boot error" → Filesystem corruption (Hỏng Hệ Thống File)
   - "Kernel panic" → OS kernel crash
   - "No space left on device" → Disk đầy, OS không boot được
   - "fsck failed" → Lỗi kiểm tra filesystem

3. Giải pháp:
   - Thử Reboot trước (đôi khi đủ)
   - Nếu không được: detach volume, mount vào instance khác, sửa lỗi
   - Xem xét restore từ EBS Snapshot (Ảnh Chụp EBS)
```

### Monitoring Status Checks với CloudWatch

```bash
# Tạo CloudWatch Alarm cho Instance Status Check
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-InstanceStatusCheck-Failed" \
  --metric-name StatusCheckFailed_Instance \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --statistic Maximum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:automate:us-east-1:ec2:recover  # Auto Recovery
```

---

## 3. Instance Không Khởi Động

### Boot Failure — Không Khởi Động Được

```
Triệu chứng: Instance ở trạng thái "pending" mãi hoặc "terminated" ngay
             sau khi launch.

Nguyên nhân thường gặp:
  A. Disk đầy (100%) — OS không boot được
  B. User Data script lỗi — gây treo hoặc crash
  C. Filesystem corruption — cần fsck
  D. Kernel không tương thích với AMI
  E. Instance type không hỗ trợ AMI architecture (arm64 vs x86_64)
```

### Đọc Console Output Để Chẩn Đoán

```bash
# Lấy console output — đây là "black box recorder" (hộp đen ghi chép)
aws ec2 get-console-output \
  --instance-id i-1234567890abcdef0 \
  --output text

# Hoặc xem Screenshot console (nếu không có text output)
aws ec2 get-console-screenshot \
  --instance-id i-1234567890abcdef0 \
  --output text \
  --query ImageData | base64 -d > screenshot.jpg
```

### Xử Lý User Data Script Lỗi

```bash
# Log của User Data nằm ở:
# Amazon Linux: /var/log/cloud-init-output.log
# Ubuntu: /var/log/cloud-init-output.log

# Kết nối vào instance và xem log
sudo cat /var/log/cloud-init-output.log | tail -100

# Các lỗi thường gặp trong User Data:
# 1. Shebang thiếu: #!/bin/bash
# 2. Lỗi syntax bash
# 3. Package không tồn tại
# 4. Timeout khi download từ internet (thiếu NAT Gateway trong private subnet)
```

---

## 4. Hiệu Suất Kém

### CPU Cao Bất Thường

```bash
# Kiểm tra CPUUtilization từ CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Maximum,Average

# Kết nối vào instance và xem process nào dùng nhiều CPU
top -b -n 1 -d 1 | head -20

# Nếu CPU Steal cao (> 10%) → Instance trên host vật lý đang bị quá tải
# Giải pháp: Stop & Start để chuyển sang host khác
vmstat 1 5  # Xem cs (context switches) và us/sy/st
```

### Memory Không Đủ

```bash
# CloudWatch KHÔNG đo Memory theo mặc định — phải cài CloudWatch Agent
# Trên instance, kiểm tra bằng:
free -h
cat /proc/meminfo

# Xem process dùng nhiều memory nhất
ps aux --sort=-%mem | head -20

# Nếu instance hết memory: Swap (Hoán Đổi Bộ Nhớ) sẽ được dùng → hiệu suất giảm mạnh
# Giải pháp ngắn hạn: Restart service ăn memory nhiều
# Giải pháp dài hạn: Rightsize lên instance type lớn hơn hoặc Memory-Optimized
```

### Disk I/O Chậm (EBS Performance)

```bash
# Xem EBS metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-xxxxxxxx \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Sum

# Kiểm tra trực tiếp trên instance
iostat -x 1 5         # Xem %util của disk
iotop -b -n 5         # Xem process đọc/ghi nhiều nhất

# Nguyên nhân phổ biến:
# - gp2 volume hết "burst credits" (Tín Dụng Burst)
#   → Nâng cấp lên gp3 (ổn định hơn, không burst credit)
# - Nhiều small random reads/writes → Cân nhắc io2 Block Express
# - EBS throughput limit bị đạt → Check VolumeQueueLength metric
```

### Disk Đầy — No Space Left

```bash
# Kiểm tra disk usage
df -h

# Tìm thư mục chiếm nhiều dung lượng nhất
du -sh /* 2>/dev/null | sort -rh | head -20
du -sh /var/log/* | sort -rh | head -10

# Các thư mục thường đầy:
# /var/log → Logs ứng dụng (thiết lập log rotation)
# /tmp → File tạm (dọn sạch định kỳ)
# /var/lib/docker → Docker images/containers cũ

# Mở rộng EBS volume không cần downtime:
# 1. Tăng kích thước volume trong AWS Console/CLI
aws ec2 modify-volume --volume-id vol-xxxxxxxx --size 100

# 2. Mở rộng filesystem (vẫn đang chạy, không cần restart)
sudo growpart /dev/xvda 1          # Mở rộng partition
sudo resize2fs /dev/xvda1          # ext4
# hoặc
sudo xfs_growfs /                  # xfs
```

---

## 5. Phục Hồi Tự Động — Auto Recovery

### Cấu Hình CloudWatch Alarm + Auto Recovery

```bash
# Tạo alarm tự động recover EC2 khi System Status Check fail
aws cloudwatch put-metric-alarm \
  --alarm-name "AutoRecover-$(INSTANCE_ID)" \
  --alarm-description "Auto recover EC2 on system status check failure" \
  --metric-name StatusCheckFailed_System \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --statistic Maximum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:automate:us-east-1:ec2:recover"
```

### Kết Quả Của Auto Recovery

```
Khi Auto Recovery kích hoạt:
✅ Instance ID giữ nguyên
✅ Private IP giữ nguyên
✅ Elastic IP giữ nguyên (nếu có)
✅ EBS volumes giữ nguyên
❌ Instance store data bị mất (không phải EBS)
❌ RAM data bị mất (reboot)

Thời gian: Thường 5-15 phút để recovery hoàn tất
```

### Auto Recovery trong Launch Template (Khi Tạo Instance Mới)

```json
{
  "MaintenanceOptions": {
    "AutoRecovery": "default"
  }
}
```

---

## 6. Sự Cố Mạng & Connectivity

### Instance Không Truy Cập Internet Được (Private Subnet)

```
Triệu chứng: Trong instance, curl google.com timeout

Nguyên nhân:
  - Instance ở Private Subnet (Mạng Con Riêng Tư)
  - Không có NAT Gateway (Cổng Dịch Địa Chỉ Mạng) hoặc NAT Instance

Kiểm tra:
  1. Instance có Public IP không? → Không có Public IP = private subnet
  2. Route Table có default route (0.0.0.0/0) không?
  3. Default route trỏ đến NAT Gateway chưa?

Giải pháp:
  A. Tạo NAT Gateway trong Public Subnet
  B. Thêm route 0.0.0.0/0 → nat-xxxxxxxx vào Route Table của Private Subnet
  C. Hoặc dùng VPC Endpoints (Điểm Cuối VPC) cho AWS services (không cần NAT)
```

### Không Kết Nối Được Giữa Hai Instance

```bash
# Kiểm tra từng layer:

# Layer 1: Security Group
# Instance A muốn kết nối Instance B trên port 8080
# → Security Group của Instance B phải có inbound rule:
#   Port 8080, Source = Security Group ID của Instance A
#   (KHÔNG dùng IP — IP có thể thay đổi, Security Group ID ổn định hơn)

# Layer 2: NACL
# NACL stateless — cần mở cả inbound (8080) VÀ outbound (ephemeral ports 1024-65535)

# Layer 3: Route Table
# Cả hai instance phải có thể route đến nhau
# (Thường tự động trong cùng VPC)

# Test từ Instance A:
telnet <instance-B-private-ip> 8080
# hoặc
nc -zv <instance-B-private-ip> 8080
```

### Latency Cao Giữa Các Instance

```
Giải pháp cho latency thấp:
  1. Đặt trong cùng Placement Group Cluster (Nhóm Vị Trí Cụm)
     → Latency xuống còn < 1ms, bandwidth tăng lên đến 100 Gbps
  2. Dùng Enhanced Networking (Mạng Tăng Cường) — SR-IOV
     → Các instance m5, c5, r5 trở lên tự động có Enhanced Networking
  3. Dùng instance types hỗ trợ EFA (Elastic Fabric Adapter)
     → Cho HPC workloads cần ultra-low latency
```

---

## 🔁 Tổng Kết — Quy Trình Chẩn Đoán EC2

```
Gặp sự cố EC2?
│
├─ KHÔNG KẾT NỐI ĐƯỢC?
│   ├─ Check Security Group port
│   ├─ Check Public IP / Elastic IP
│   ├─ Check NACL rules
│   └─ Dùng SSM Session Manager thay SSH
│
├─ STATUS CHECK FAIL?
│   ├─ System Check Fail → Stop & Start (chuyển host vật lý)
│   └─ Instance Check Fail → Xem Console Output, có thể cần fsck
│
├─ HIỆU SUẤT KÉM?
│   ├─ CPU cao → Check CloudWatch, top, có thể Rightsize instance
│   ├─ Memory hết → Check free, ps aux, Rightsize hoặc tối ưu app
│   ├─ Disk I/O chậm → Check VolumeQueueLength, cân nhắc io2/gp3
│   └─ Disk đầy → df -h, du, mở rộng EBS volume
│
└─ KHÔNG BOOT?
    ├─ Xem Console Output / Screenshot
    ├─ Check User Data logs: /var/log/cloud-init-output.log
    └─ Detach volume, mount vào instance khác để sửa
```

---

## 📖 Tham Khảo Thêm

- [2-auto-scaling-issues.md](2-auto-scaling-issues.md) — Nếu EC2 trong ASG bị terminate
- [../08-security/2-systems-manager.md](../08-security/2-systems-manager.md) — SSM Session Manager chi tiết
- [../09-monitoring/1-cloudwatch-metrics.md](../09-monitoring/1-cloudwatch-metrics.md) — CloudWatch EC2 metrics

---

**Cập Nhật Lần Cuối:** 2026-05-15
