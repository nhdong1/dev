# 📋 Launch Templates — Mẫu Khởi Chạy Instance

> **Launch Template** — Mẫu Khởi Chạy — định nghĩa cấu hình đầy đủ của EC2 instance sẽ được Auto Scaling Group tạo ra. Launch Template là phiên bản mới và được khuyến nghị thay thế **Launch Configuration** (Cấu Hình Khởi Chạy) cũ.

---

## 📚 Mục Lục

1. [Launch Template vs Launch Configuration](#launch-template-vs-launch-configuration)
2. [Cấu Hình Launch Template](#cấu-hình-launch-template)
3. [Versioning — Quản Lý Phiên Bản](#versioning)
4. [Instance Requirements — Yêu Cầu Instance Linh Hoạt](#instance-requirements)
5. [Tạo Launch Template Bằng AWS CLI](#tạo-launch-template-bằng-aws-cli)
6. [Cập Nhật ASG Khi Launch Template Thay Đổi](#cập-nhật-asg-khi-launch-template-thay-đổi)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ⚔️ Launch Template vs Launch Configuration

### So Sánh Tổng Quan

| Tính Năng                               | Launch Template | Launch Configuration |
| --------------------------------------- | :-------------: | :------------------: |
| **Versioning** (phiên bản hóa)          | ✅              | ❌                   |
| **Mixed Instance Policy** (hỗn hợp)    | ✅              | ❌                   |
| **Spot + On-Demand trong cùng ASG**     | ✅              | ❌                   |
| **Dedicated Hosts**                     | ✅              | ❌                   |
| **Capacity Reservations**               | ✅              | ❌                   |
| **T2/T3 Unlimited Burst**               | ✅              | ❌                   |
| **Elastic GPU & Elastic Inference**     | ✅              | ❌                   |
| **Instance Requirements** (linh hoạt)  | ✅              | ❌                   |
| **RAM Disk ID**                         | ✅              | ✅                   |
| **Kernel ID**                           | ✅              | ✅                   |
| **Còn được AWS hỗ trợ phát triển**     | ✅              | ❌ (Legacy)          |

### Kết Luận

> **Luôn dùng Launch Template** cho mọi ASG mới. Launch Configuration là legacy (cũ) — AWS không thêm tính năng mới và khuyến nghị migrate (chuyển đổi) sang Launch Template.

---

## ⚙️ Cấu Hình Launch Template

### Các Thông Số Quan Trọng

```json
{
  "LaunchTemplateName": "web-server-lt",
  "LaunchTemplateData": {

    // AMI — Amazon Machine Image (Ảnh Máy Ảo)
    // Định nghĩa OS và phần mềm được cài sẵn
    "ImageId": "ami-0abcdef1234567890",

    // Instance Type (Loại Instance)
    // Xác định CPU, RAM, Network bandwidth
    "InstanceType": "t3.medium",

    // Key Pair (Cặp Khóa SSH)
    // Để SSH vào instance
    "KeyName": "my-production-keypair",

    // Security Groups (Nhóm Bảo Mật)
    // Quy tắc firewall
    "SecurityGroupIds": [
      "sg-0123456789abcdef0",
      "sg-0fedcba9876543210"
    ],

    // IAM Instance Profile (Hồ Sơ Quyền Truy Cập)
    // Role gắn với instance để gọi AWS APIs
    "IamInstanceProfile": {
      "Name": "WebServerInstanceProfile"
    },

    // EBS Volumes (Ổ Đĩa Block Storage)
    "BlockDeviceMappings": [
      {
        "DeviceName": "/dev/xvda",
        "Ebs": {
          "VolumeSize": 50,
          "VolumeType": "gp3",
          "Iops": 3000,
          "Throughput": 125,
          "DeleteOnTermination": true,
          "Encrypted": true,
          "KmsKeyId": "arn:aws:kms:us-east-1:..."
        }
      }
    ],

    // Network Interfaces (Giao Diện Mạng)
    "NetworkInterfaces": [
      {
        "DeviceIndex": 0,
        "AssociatePublicIpAddress": false,
        "SubnetId": "subnet-0123456789abcdef0",
        "Groups": ["sg-0123456789abcdef0"]
      }
    ],

    // User Data (Dữ Liệu Khởi Tạo) — Base64 encoded
    // Script chạy lần đầu khi instance khởi động
    "UserData": "IyEvYmluL2Jhc2gKCiMgVXBkYXRlIHN5c3RlbQp5dW0gdXBkYXRlIC15Cg==",

    // Monitoring (Giám Sát)
    // true = detailed monitoring (1-phút intervals) thay vì 5-phút
    "Monitoring": {
      "Enabled": true
    },

    // EBS Optimized (Tối Ưu I/O Đĩa)
    // Băng thông riêng cho EBS I/O
    "EbsOptimized": true,

    // Instance Initiated Shutdown Behavior
    // "terminate" = khi tắt OS từ bên trong → terminate instance
    // "stop" = khi tắt OS từ bên trong → stop instance (giữ lại)
    "InstanceInitiatedShutdownBehavior": "terminate",

    // Placement (Vị Trí Vật Lý)
    "Placement": {
      "Tenancy": "default"
    },

    // Tags (Thẻ Phân Loại)
    "TagSpecifications": [
      {
        "ResourceType": "instance",
        "Tags": [
          {"Key": "Environment", "Value": "Production"},
          {"Key": "Application", "Value": "WebServer"}
        ]
      },
      {
        "ResourceType": "volume",
        "Tags": [
          {"Key": "Environment", "Value": "Production"}
        ]
      }
    ]
  }
}
```

### User Data — Script Khởi Tạo Instance

```bash
#!/bin/bash
# Script này chạy một lần khi instance khởi động lần đầu tiên

# Cập nhật packages
yum update -y

# Cài đặt ứng dụng web
yum install -y nginx

# Lấy metadata của instance để biết mình đang chạy ở AZ nào
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

# Cấu hình ứng dụng
cat > /var/www/html/index.html << EOF
<h1>Web Server</h1>
<p>Instance: $INSTANCE_ID | AZ: $AZ</p>
EOF

# Khởi động nginx và enable auto-start
systemctl start nginx
systemctl enable nginx

# Ghi log để debug
echo "Bootstrap completed at $(date)" >> /var/log/bootstrap.log
```

**Lưu ý quan trọng về User Data:**
- Chỉ chạy **một lần duy nhất** khi instance khởi động lần đầu
- Chạy với quyền **root**
- Phải được **Base64 encode** khi truyền qua API
- Giới hạn **16 KB** dung lượng
- Không phù hợp cho cấu hình phức tạp → dùng **AWS Systems Manager** hoặc **cloud-init**

---

## 🔖 Versioning — Quản Lý Phiên Bản

### Tại Sao Versioning Quan Trọng?

```
Launch Template Version History (Lịch Sử Phiên Bản):

Version 1  (2024-01-15): AMI=ami-001, Type=t3.small   ← OldestLaunchTemplate
Version 2  (2024-03-20): AMI=ami-002, Type=t3.medium
Version 3  (2024-06-10): AMI=ami-003, Type=t3.medium  ← $Latest

ASG đang dùng: Version=$Default (= Version 2 được set làm default)

→ Có thể rollback bất kỳ lúc nào bằng cách đổi $Default về Version 1
→ Không cần tạo lại ASG khi muốn cập nhật cấu hình
```

### Hai Alias Đặc Biệt

| Alias       | Ý Nghĩa                                        | Khi Nào Dùng                              |
| ----------- | ----------------------------------------------- | ----------------------------------------- |
| **$Latest** | Luôn trỏ vào version mới nhất tự động           | Dev/Test environments                     |
| **$Default**| Version được chỉ định làm mặc định              | Production (kiểm soát chặt hơn)          |

```bash
# Set version 3 làm default
aws ec2 modify-launch-template \
  --launch-template-id lt-0123456789abcdef0 \
  --default-version "3"

# ASG dùng $Default (production best practice — không bị surprise update)
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "web-servers-asg" \
  --launch-template "LaunchTemplateName=web-server-lt,Version=\$Default"

# ASG dùng $Latest (dev/test — luôn dùng version mới nhất)
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "web-servers-asg-dev" \
  --launch-template "LaunchTemplateName=web-server-lt,Version=\$Latest"
```

### Tạo Version Mới

```bash
# Tạo version mới từ version hiện tại, chỉ thay đổi AMI
aws ec2 create-launch-template-version \
  --launch-template-id lt-0123456789abcdef0 \
  --source-version "2" \
  --version-description "Updated to new AMI with security patches" \
  --launch-template-data '{"ImageId": "ami-0newimage1234567"}'

# Kết quả: Version 3 giống Version 2 nhưng AMI mới
```

---

## 🎯 Instance Requirements — Yêu Cầu Instance Linh Hoạt

Thay vì chỉ định cứng 1 instance type, bạn có thể định nghĩa **yêu cầu phần cứng** và để AWS chọn instance phù hợp:

### Cách Hoạt Động

```
Thay vì: InstanceType = "c5.xlarge" (cố định)

Dùng Instance Requirements:
  VCpuCount: min=4, max=8
  MemoryMiB: min=8192, max=16384    (8-16 GB RAM)
  InstanceGenerations: [current]    (chỉ gen hiện tại, không dùng gen cũ)
  ExcludedInstanceTypes: [t*]       (loại trừ dòng T — burstable)

AWS tự động chọn từ pool: c5.xlarge, c5a.xlarge, c6i.xlarge, m5.xlarge...
```

### Cấu Hình Instance Requirements

```json
{
  "InstanceRequirements": {
    "VCpuCount": {
      "Min": 4,
      "Max": 8
    },
    "MemoryMiB": {
      "Min": 8192,
      "Max": 16384
    },
    "InstanceGenerations": ["current"],
    "ExcludedInstanceTypes": ["t2.*", "t3.*", "t3a.*"],
    "CpuManufacturers": ["intel", "amd"],
    "BurstablePerformance": "excluded",
    "BareMetal": "excluded",
    "SpotMaxPricePercentageOverLowestPrice": 50
  }
}
```

**Ưu điểm của Instance Requirements:**
- Tối đa hóa khả năng fulfill Spot (Spot thường có capacity giới hạn)
- Tự động dùng instance mới hơn khi được release
- Tối ưu chi phí bằng cách chọn instance rẻ nhất đáp ứng yêu cầu

---

## 💻 Tạo Launch Template Bằng AWS CLI

```bash
# Tạo Launch Template hoàn chỉnh
aws ec2 create-launch-template \
  --launch-template-name "web-server-lt" \
  --version-description "Production web server v1" \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "t3.medium",
    "KeyName": "production-keypair",
    "SecurityGroupIds": ["sg-0123456789abcdef0"],
    "IamInstanceProfile": {
      "Name": "WebServerInstanceProfile"
    },
    "BlockDeviceMappings": [
      {
        "DeviceName": "/dev/xvda",
        "Ebs": {
          "VolumeSize": 50,
          "VolumeType": "gp3",
          "DeleteOnTermination": true,
          "Encrypted": true
        }
      }
    ],
    "Monitoring": {"Enabled": true},
    "EbsOptimized": true,
    "TagSpecifications": [
      {
        "ResourceType": "instance",
        "Tags": [
          {"Key": "Environment", "Value": "Production"},
          {"Key": "Name", "Value": "WebServer"}
        ]
      }
    ],
    "UserData": "'"$(base64 -w 0 << 'EOF'
#!/bin/bash
yum update -y
yum install -y nginx
systemctl start nginx
systemctl enable nginx
EOF
)"'"
  }'

# Xem danh sách Launch Template versions
aws ec2 describe-launch-template-versions \
  --launch-template-name "web-server-lt" \
  --query 'LaunchTemplateVersions[*].{Ver:VersionNumber,Desc:VersionDescription,Default:DefaultVersion}'
```

---

## 🔄 Cập Nhật ASG Khi Launch Template Thay Đổi

Khi bạn tạo Launch Template version mới (ví dụ: AMI mới, instance type mới), các instance hiện có trong ASG **không tự động được cập nhật**. Có hai cách để áp dụng thay đổi:

### Cách 1: Instance Refresh — Làm Mới Instance (Khuyến Nghị)

```bash
# Khởi động Instance Refresh — rolling replacement của tất cả instances
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name "web-servers-asg" \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300,
    "CheckpointPercentages": [20, 50, 100],
    "CheckpointDelay": 600
  }'
```

**Cách hoạt động:**
```
Desired = 10 instances, MinHealthyPercentage = 90%

Tại bất kỳ thời điểm nào: ít nhất 9 instances phải healthy

Bước 1: Terminate 1 instance cũ → Launch 1 instance mới
Bước 2: Chờ instance mới InService và pass health check
Bước 3: Lặp lại cho đến khi tất cả instances đã được refresh
```

### Cách 2: Terminate Thủ Công (Đơn Giản Hơn Nhưng Rủi Ro Hơn)

```bash
# Terminate từng instance thủ công — ASG sẽ launch instance mới với template mới
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id i-0123456789abcdef0 \
  --no-should-decrement-desired-capacity
```

---

## ✅ Best Practices

### Bảo Mật Launch Template

```
1. Không bao giờ hardcode secrets trong UserData:
   ❌ export DB_PASSWORD="my-secret-password"
   ✅ Dùng AWS Secrets Manager:
      DB_PASSWORD=$(aws secretsmanager get-secret-value \
        --secret-id prod/db/password --query SecretString --output text)

2. Luôn encrypt EBS volumes:
   "Encrypted": true, "KmsKeyId": "arn:aws:kms:..."

3. Không gán public IP trừ khi cần thiết:
   "AssociatePublicIpAddress": false

4. Dùng IAM Instance Profile thay vì access keys:
   "IamInstanceProfile": {"Name": "AppInstanceProfile"}
```

### Version Management — Quản Lý Phiên Bản

```
1. Luôn thêm VersionDescription mô tả thay đổi
   "Version 3: Updated AMI to include January 2024 security patches"

2. Production ASG dùng $Default (không dùng $Latest)
   → Kiểm soát khi nào version mới được áp dụng

3. Không xóa version cũ ngay — giữ lại để rollback
   → Xóa sau khi đã verify version mới ổn định (2 tuần)

4. Dùng tag để tracking launch template
   {"Key": "git-commit", "Value": "abc123def456"}
```

### Tối Ưu EBS

```
gp3 volumes:
  → Chọn gp3 thay vì gp2 (rẻ hơn 20%, performance tốt hơn)
  → gp3 baseline: 3000 IOPS, 125 MB/s throughput (không tính phí thêm)
  → Tăng IOPS/throughput nếu cần (có phí thêm, nhưng độc lập với volume size)

io2 volumes (chỉ khi cần):
  → Database workload cần IOPS rất cao (> 16,000)
  → Durability 99.999% (5 nines) thay vì 99.8%-99.9% của gp3

DeleteOnTermination: true
  → Luôn để true cho root volume (tránh orphaned volumes gây tốn kém)
  → Data volumes có thể false nếu cần bảo toàn data sau terminate
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Launch Template khác Launch Configuration ở điểm nào quan trọng nhất?**

> Điểm quan trọng nhất là **versioning** — Launch Template hỗ trợ nhiều phiên bản và rollback, trong khi Launch Configuration không thể chỉnh sửa sau khi tạo (immutable). Ngoài ra, Launch Template là yêu cầu bắt buộc để dùng Mixed Instance Policy (kết hợp On-Demand và Spot trong cùng ASG) và Instance Requirements (định nghĩa yêu cầu phần cứng thay vì instance type cụ thể).

**Q: Tại sao không nên dùng `$Latest` cho Production ASG?**

> Khi ASG được cấu hình với `$Latest`, bất kỳ ai tạo Launch Template version mới sẽ vô tình thay đổi cấu hình mà các instance mới launch dùng. Điều này có thể dẫn đến production incident ngoài ý muốn. Production nên dùng `$Default` — một version được kiểm soát rõ ràng, chỉ thay đổi khi có quyết định có chủ đích.

**Q: Khi cập nhật AMI mới, làm thế nào để áp dụng cho tất cả instances trong ASG mà không gây downtime?**

> Tạo Launch Template version mới với AMI mới → Cập nhật ASG để dùng version mới → Dùng **Instance Refresh** với `MinHealthyPercentage=90` để rolling replace từng instance. Instance Refresh tự động terminate instance cũ, launch instance mới, chờ pass health check rồi tiếp tục với instance tiếp theo. Toàn bộ quá trình zero-downtime và có thể cancel nếu phát hiện vấn đề.

---

## 🔗 Điều Hướng

| Trước                                              | Tiếp Theo                                         |
| -------------------------------------------------- | ------------------------------------------------- |
| [← ASG Cơ Bản](1-auto-scaling-groups.md)           | [Scaling Policies →](3-scaling-policies.md)       |

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
