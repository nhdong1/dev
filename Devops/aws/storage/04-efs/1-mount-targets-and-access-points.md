# EFS Mount Targets & Access Points — Điểm Gắn Kết & Điểm Truy Cập

> Mount Target (Điểm Gắn Kết) là cầu nối mạng cho phép client kết nối đến EFS qua giao thức NFS. Access Point (Điểm Truy Cập) là lối vào ứng dụng với danh tính POSIX và đường dẫn gốc cố định, tăng cường bảo mật cho môi trường multi-tenant (nhiều người thuê).

---

## 📌 Mount Target — Điểm Gắn Kết

### Mount Target Là Gì?

Mount Target là một **network interface** (giao diện mạng) được tạo trong một Subnet thuộc một AZ — Availability Zone (Vùng Sẵn Sàng). Mỗi EFS file system cần ít nhất một Mount Target để client có thể kết nối.

```
Amazon EFS File System
         │
         ├── Mount Target (AZ us-east-1a) ←── IP: 10.0.1.55 (ENI trong subnet)
         ├── Mount Target (AZ us-east-1b) ←── IP: 10.0.2.55
         └── Mount Target (AZ us-east-1c) ←── IP: 10.0.3.55
```

### Quy Tắc Mount Target

| Quy Tắc | Chi Tiết |
|---------|---------|
| **Một Mount Target / AZ** | Chỉ được tạo một Mount Target duy nhất trong mỗi AZ |
| **Port** | NFS port **2049** (TCP) phải được mở trong Security Group |
| **DNS** | Tự động có DNS record: `fs-xxxxxxxx.efs.region.amazonaws.com` |
| **IP tĩnh** | Mỗi Mount Target có một private IP tĩnh trong subnet |
| **ENI** | Mỗi Mount Target là một ENI — Elastic Network Interface (Giao Diện Mạng Linh Hoạt) |

### Kiến Trúc Multi-AZ (Khuyến Nghị Production)

```
VPC (10.0.0.0/16)
├── AZ us-east-1a
│   ├── Subnet: 10.0.1.0/24
│   │   └── Mount Target IP: 10.0.1.55  ←── EC2, ECS tasks trong AZ này
│   └── Security Group: efs-sg (cho phép TCP 2049 từ EC2 SG)
│
├── AZ us-east-1b
│   ├── Subnet: 10.0.2.0/24
│   │   └── Mount Target IP: 10.0.2.55  ←── EC2, ECS tasks trong AZ này
│   └── Security Group: efs-sg
│
└── AZ us-east-1c
    ├── Subnet: 10.0.3.0/24
    │   └── Mount Target IP: 10.0.3.55  ←── EC2, ECS tasks trong AZ này
    └── Security Group: efs-sg
```

> **Best Practice:** Tạo Mount Target trong **mỗi AZ** nơi bạn có EC2/ECS. Điều này giúp tránh cross-AZ data transfer (truyền dữ liệu qua AZ), giảm latency và chi phí.

### Tạo Mount Target

```bash
# Tạo Mount Target trong mỗi AZ
aws efs create-mount-target \
  --file-system-id fs-xxxxxxxx \
  --subnet-id subnet-aaa111 \
  --security-groups sg-efs-access

# Xem danh sách Mount Targets
aws efs describe-mount-targets \
  --file-system-id fs-xxxxxxxx
```

### Security Group Cho Mount Target

```json
{
  "SecurityGroupRules": [
    {
      "Type": "Inbound",
      "Protocol": "TCP",
      "Port": 2049,
      "Source": "sg-ec2-instances",
      "Description": "Cho phép EC2 instances kết nối NFS"
    }
  ]
}
```

---

## 🎯 Access Points — Điểm Truy Cập

### Access Point Là Gì?

Access Point là một **application-specific entry point** (điểm vào theo từng ứng dụng) vào EFS file system. Mỗi Access Point có thể:

- **Enforce POSIX user identity** — Ép buộc UID (User ID) và GID (Group ID) cụ thể, bất kể client muốn dùng UID nào
- **Set root directory** — Đặt một thư mục gốc riêng cho từng ứng dụng (client chỉ thấy subtree của mình)
- **Enforce permissions** — Đặt permission cho thư mục gốc khi tạo lần đầu

```
EFS File System (/)
├── /app-a/          ←── Access Point A (UID=1000, GID=1000) → app-a chỉ thấy /app-a/
├── /app-b/          ←── Access Point B (UID=2000, GID=2000) → app-b chỉ thấy /app-b/
└── /shared/         ←── Access Point C (read-only) → shared resources
```

### Tại Sao Cần Access Points?

| Vấn Đề Không Có Access Points | Giải Pháp Với Access Points |
|-------------------------------|------------------------------|
| Container A có thể ghi đè dữ liệu của Container B | Mỗi container dùng Access Point riêng với UID khác nhau |
| Client có thể leo thang root để truy cập toàn bộ file system | Access Point ghim client vào thư mục con cụ thể |
| Khó audit ai truy cập dữ liệu gì | CloudTrail log theo từng Access Point ARN |
| Lambda function cần shared storage nhưng phải an toàn | Lambda mount Access Point với POSIX identity cố định |

### Tạo Access Point

```bash
# Tạo Access Point cho ứng dụng web
aws efs create-access-point \
  --file-system-id fs-xxxxxxxx \
  --root-directory "Path=/webapp,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=755}" \
  --posix-user "Uid=1000,Gid=1000" \
  --tags Key=Name,Value=webapp-access-point
```

```json
{
  "AccessPoint": {
    "FileSystemId": "fs-xxxxxxxx",
    "RootDirectory": {
      "Path": "/webapp",
      "CreationInfo": {
        "OwnerUid": 1000,
        "OwnerGid": 1000,
        "Permissions": "755"
      }
    },
    "PosixUser": {
      "Uid": 1000,
      "Gid": 1000
    }
  }
}
```

### Mount Qua Access Point

```bash
# Mount qua EFS mount helper với Access Point
sudo mount -t efs \
  -o tls,accesspoint=fsap-xxxxxxxx \
  fs-xxxxxxxx:/ /mnt/webapp

# Hoặc trong /etc/fstab
fs-xxxxxxxx:/ /mnt/webapp efs tls,accesspoint=fsap-xxxxxxxx,_netdev 0 0
```

---

## 🔐 IAM Authorization — Ủy Quyền Qua IAM

### EFS IAM Authorization Là Gì?

EFS hỗ trợ IAM-based authorization — Ủy quyền dựa trên IAM bổ sung thêm lớp kiểm soát **trên tầng NFS**. Khi bật, mọi kết nối NFS phải kèm theo IAM identity hợp lệ.

> Tính năng này yêu cầu **EFS mount helper** (`amazon-efs-utils`) và bật TLS (Transport Layer Security — Bảo mật Tầng Truyền Tải).

### EFS File System Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:role/WebAppRole"
      },
      "Action": [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite"
      ],
      "Resource": "arn:aws:elasticfilesystem:us-east-1:123456789:file-system/fs-xxxxxxxx"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### EFS IAM Actions (Hành Động IAM)

| IAM Action | Ý Nghĩa |
|------------|---------|
| `elasticfilesystem:ClientMount` | Cho phép mount (đọc) |
| `elasticfilesystem:ClientWrite` | Cho phép ghi dữ liệu |
| `elasticfilesystem:ClientRootAccess` | Cho phép truy cập với UID=0 (root) |

### IAM Policy Cho EC2 Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite"
      ],
      "Resource": "arn:aws:elasticfilesystem:us-east-1:123456789:file-system/fs-xxxxxxxx",
      "Condition": {
        "StringEquals": {
          "elasticfilesystem:AccessPointArn": "arn:aws:elasticfilesystem:us-east-1:123456789:access-point/fsap-xxxxxxxx"
        }
      }
    }
  ]
}
```

---

## 🐳 EFS Với ECS & EKS

### ECS Task Definition (Định Nghĩa Task ECS)

```json
{
  "volumes": [
    {
      "name": "efs-volume",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-xxxxxxxx",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "fsap-xxxxxxxx",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "webapp",
      "mountPoints": [
        {
          "sourceVolume": "efs-volume",
          "containerPath": "/data",
          "readOnly": false
        }
      ]
    }
  ]
}
```

### EKS PersistentVolume (Ổ Đĩa Lâu Dài trong Kubernetes)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany        # Nhiều pod cùng đọc/ghi
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-xxxxxxxx::fsap-xxxxxxxx   # FileSystemId::AccessPointId
```

---

## 🔧 EFS Mount Helper — Công Cụ Mount EFS

`amazon-efs-utils` là gói công cụ AWS cung cấp để mount EFS dễ dàng hơn so với NFS thuần túy.

```bash
# Cài đặt trên Amazon Linux 2
sudo yum install -y amazon-efs-utils

# Cài đặt trên Ubuntu/Debian
sudo apt-get install -y amazon-efs-utils

# Mount với TLS (mã hóa in-transit)
sudo mount -t efs -o tls fs-xxxxxxxx:/ /mnt/efs

# Mount với TLS + IAM authorization
sudo mount -t efs -o tls,iam fs-xxxxxxxx:/ /mnt/efs

# Mount với Access Point + TLS + IAM
sudo mount -t efs \
  -o tls,iam,accesspoint=fsap-xxxxxxxx \
  fs-xxxxxxxx:/ /mnt/efs
```

### So Sánh Mount Options (Tùy Chọn Mount)

| Option | Ý Nghĩa | Khi Nào Dùng |
|--------|---------|--------------|
| `tls` | Mã hóa in-transit | Luôn dùng trong production |
| `iam` | Xác thực qua IAM | Khi EFS file system policy bật IAM |
| `accesspoint=fsap-xxx` | Dùng Access Point cụ thể | Môi trường multi-tenant |
| `nfsvers=4.1` | NFS version 4.1 | Dùng khi mount NFS thuần túy |
| `_netdev` | Chờ mạng trước khi mount | Trong `/etc/fstab` |

---

## 📊 Tóm Tắt: Mount Target vs Access Point

| | Mount Target | Access Point |
|-|-------------|-------------|
| **Vai Trò** | Network endpoint (cổng mạng) | Application entry point (cổng ứng dụng) |
| **Cấp Độ** | Cấp mạng (Layer 3/4) | Cấp ứng dụng (Layer 7) |
| **Tạo Ở Đâu** | Trong Subnet / AZ | Trên file system |
| **Số Lượng** | Tối đa 1 / AZ | Tối đa 120 / file system |
| **Bảo Mật** | Security Groups (port 2049) | IAM + POSIX identity |
| **Mục Đích** | Kết nối mạng | Phân cách dữ liệu theo ứng dụng |

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao nên tạo Mount Target trong mỗi AZ?**
A: Để tránh cross-AZ traffic — nếu EC2 ở AZ-a nhưng Mount Target chỉ ở AZ-b, mọi I/O đều phải truyền qua AZ, tốn phí và tăng latency. Mount Target trong cùng AZ sẽ tối ưu cả chi phí lẫn hiệu suất.

**Q: Access Point có thể giới hạn storage quota không?**
A: Không trực tiếp. Access Point chỉ enforce POSIX identity và root directory. Để giới hạn quota, cần dùng POSIX permissions trên thư mục hoặc các công cụ ngoài.

**Q: Lambda function có thể mount EFS không?**
A: Có — Lambda hỗ trợ EFS mount qua Access Points từ 2020. Lambda phải ở cùng VPC với Mount Target và cấu hình `fileSystemConfigs` trong function configuration.

**Q: Khác nhau giữa NFS mount thuần và EFS mount helper?**
A: Mount helper tự động xử lý TLS, IAM authentication, retry logic, và stunnel (đường hầm bảo mật). NFS mount thuần túy không hỗ trợ IAM authorization.

---

## 🔗 Điều Hướng

- [← README.md](./README.md) — Tổng quan EFS
- [→ 2-performance-modes.md](./2-performance-modes.md) — Chế Độ Hiệu Suất
- [→ 5-efs-vs-ebs-vs-s3.md](./5-efs-vs-ebs-vs-s3.md) — Bảng So Sánh
