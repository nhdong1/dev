# AWS Systems Manager (SSM) — Tổng Quan & Bộ Công Cụ Vận Hành

> **SSM — AWS Systems Manager** là bộ công cụ vận hành (operational toolkit) toàn diện cho phép quản lý EC2 instances, on-premises servers, và multi-cloud workloads từ một giao diện trung tâm — không cần SSH, không cần bastion host, không cần mở port.

---

## 📚 Mục Lục

1. [Tại Sao Cần SSM?](#tại-sao-cần-ssm)
2. [Các Năng Lực Chính](#các-năng-lực-chính)
3. [Kiến Trúc & Thành Phần](#kiến-trúc--thành-phần)
4. [SSM Agent & Điều Kiện Tiên Quyết](#ssm-agent--điều-kiện-tiên-quyết)
5. [Ma Trận Tính Năng](#ma-trận-tính-năng)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
7. [Điều Hướng Nội Dung](#điều-hướng-nội-dung)

---

## Tại Sao Cần SSM?

### Vấn Đề Truyền Thống

```
Quản lý fleet EC2 kiểu cũ:
┌──────────────────────────────────────────────────────────────┐
│  Operator ──SSH──▶ Bastion Host ──SSH──▶ EC2 Instance        │
│                                                              │
│  Vấn đề:                                                     │
│  • Port 22 mở → Attack surface (bề mặt tấn công)            │
│  • Key pair phải phân phối, rotate thủ công                  │
│  • Không có audit log đầy đủ                                  │
│  • Bastion host = Single point of failure                    │
│  • On-premises servers không thể dùng cách này               │
└──────────────────────────────────────────────────────────────┘
```

### Giải Pháp SSM

```
Quản lý fleet với SSM:
┌──────────────────────────────────────────────────────────────┐
│  Operator ──HTTPS──▶ SSM Endpoint ──▶ SSM Agent trên EC2     │
│                                                              │
│  Lợi ích:                                                    │
│  • Không cần port 22, không cần key pair                     │
│  • Mọi session được ghi log vào S3/CloudWatch                │
│  • IAM policy kiểm soát ai được access gì                    │
│  • Hỗ trợ on-premises, VMware, multi-cloud                   │
│  • Tích hợp sâu với AWS services                             │
└──────────────────────────────────────────────────────────────┘
```

---

## Các Năng Lực Chính

### 1. Truy Cập Phiên Làm Việc (Session Management)

**Session Manager** — SSH-less access (Truy Cập Không Cần SSH) vào EC2 và on-premises servers.

```
Trường hợp dùng:
✅ Debug instance không có public IP
✅ Môi trường production không cho phép mở port 22
✅ Cần audit trail (dấu vết kiểm toán) của mọi lệnh chạy
✅ Port forwarding cho RDS, Redis không expose ra ngoài
```

### 2. Vá Lỗi Tự Động (Patch Management)

**Patch Manager** — Tự động vá (patching) OS và ứng dụng theo lịch với zero downtime.

```
Trường hợp dùng:
✅ Vá hàng trăm EC2 instances theo lịch bảo trì
✅ Phân nhóm instances theo môi trường (dev/staging/prod)
✅ Báo cáo compliance patch cho kiểm toán
✅ Emergency patching khi có CVE (Common Vulnerabilities and Exposures) nghiêm trọng
```

### 3. Quản Lý Cấu Hình & Secret (Configuration & Secret Management)

**Parameter Store** — Lưu trữ cấu hình (parameters) và secret theo phân cấp.

```
Trường hợp dùng:
✅ Database connection strings cho ứng dụng
✅ API keys, config values theo môi trường
✅ Tích hợp với Lambda, ECS, EKS không cần hardcode secret
✅ Versioning và audit trail cho config changes
```

### 4. Thực Thi Lệnh Hàng Loạt (Bulk Command Execution)

**Run Command & Automation** — Thực thi lệnh và runbooks trên toàn bộ fleet.

```
Trường hợp dùng:
✅ Cài đặt phần mềm trên 500 EC2 cùng lúc
✅ Thu thập thông tin chuẩn đoán khi có sự cố
✅ Restart service hàng loạt
✅ Tự động hóa runbook phức tạp (multi-step operations)
```

### 5. Kiểm Kê Phần Mềm (Software Inventory)

**Inventory** — Biết chính xác phần mềm gì đang chạy trên từng instance.

```
Trường hợp dùng:
✅ Audit phần mềm cài đặt trên fleet
✅ Phát hiện phần mềm không được phép
✅ License compliance (tuân thủ bản quyền)
✅ Tích hợp với Config để theo dõi thay đổi
```

---

## Kiến Trúc & Thành Phần

### Sơ Đồ Tổng Quan SSM

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS Systems Manager                          │
│                                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐   │
│  │   Session   │  │    Patch     │  │      Parameter          │   │
│  │   Manager   │  │   Manager    │  │        Store            │   │
│  └──────┬──────┘  └──────┬───────┘  └───────────┬─────────────┘   │
│         │                │                      │                  │
│  ┌──────▼──────┐  ┌──────▼───────┐  ┌───────────▼─────────────┐   │
│  │    Run      │  │  Automation  │  │       Inventory          │   │
│  │   Command   │  │  (Runbooks)  │  │     & Compliance         │   │
│  └──────┬──────┘  └──────┬───────┘  └───────────┬─────────────┘   │
│         │                │                      │                  │
│  ┌──────▼──────┐  ┌──────▼───────┐  ┌───────────▼─────────────┐   │
│  │ Distributor │  │  OpsCenter   │  │      State Manager       │   │
│  │  (Packages) │  │  (OpsItems)  │  │  (Configuration Drift)   │   │
│  └─────────────┘  └──────────────┘  └─────────────────────────┘   │
│                                                                     │
│  ─────────────────── SSM Agent (Trên mọi Node) ─────────────────── │
│                                                                     │
│  EC2 Instances │ On-Premises Servers │ VMware VMs │ Edge Devices    │
└─────────────────────────────────────────────────────────────────────┘
```

### Các Tầng Dịch Vụ SSM

| Tầng | Chức Năng | Dịch Vụ |
|------|-----------|---------|
| **Access** (Truy Cập) | Kết nối vào node | Session Manager |
| **Execute** (Thực Thi) | Chạy lệnh/script | Run Command, Automation |
| **Configure** (Cấu Hình) | Quản lý trạng thái | State Manager, Parameter Store |
| **Maintain** (Bảo Trì) | Patch và phân phối | Patch Manager, Distributor |
| **Observe** (Quan Sát) | Giám sát và kiểm kê | Inventory, Compliance, OpsCenter |

---

## SSM Agent & Điều Kiện Tiên Quyết

### SSM Agent — Thành Phần Cốt Lõi

**SSM Agent** là phần mềm chạy trên node (EC2 hoặc on-premises), nhận lệnh từ SSM service và thực thi.

```
Kiến trúc kết nối:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  EC2 Instance                                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │  SSM Agent ──HTTPS outbound──▶ ssm.region.amazonaws.com │
│  │           ──HTTPS outbound──▶ ec2messages.region...     │
│  │           ──HTTPS outbound──▶ ssmmessages.region...     │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  Không cần:  ✗ Inbound port 22   ✗ Public IP           │
│  Cần có:     ✓ Outbound HTTPS    ✓ IAM role             │
└─────────────────────────────────────────────────────────┘
```

### Điều Kiện Cài SSM Agent

| Điều Kiện | Chi Tiết |
|-----------|----------|
| **SSM Agent cài sẵn** | Amazon Linux 2, Amazon Linux 2023, Ubuntu 16.04+, Windows Server 2008+ |
| **IAM Instance Profile** | Role có policy `AmazonSSMManagedInstanceCore` |
| **Network outbound** | HTTPS đến SSM endpoints (qua IGW hoặc VPC Endpoint) |
| **SSM VPC Endpoint** | Dùng khi instance trong private subnet không có NAT Gateway |

### IAM Policy Tối Thiểu

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:UpdateInstanceInformation",
        "ssmmessages:CreateControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:OpenDataChannel"
      ],
      "Resource": "*"
    }
  ]
}
```

> Thực tế dùng AWS Managed Policy `AmazonSSMManagedInstanceCore` thay vì tự viết.

### Đăng Ký On-Premises Server

```bash
# Tạo activation để đăng ký server on-premises
aws ssm create-activation \
  --iam-role service-role/AmazonEC2RunCommandRoleForManagedInstances \
  --registration-limit 10 \
  --region ap-southeast-1

# Output: ActivationId + ActivationCode
# Dùng trên server on-premises:
amazon-ssm-agent -register \
  -code <ActivationCode> \
  -id <ActivationId> \
  -region ap-southeast-1
```

---

## Ma Trận Tính Năng

### So Sánh Các Tính Năng Truy Cập

| Tiêu Chí | Session Manager | SSH Truyền Thống | Bastion Host |
|----------|----------------|-----------------|--------------|
| **Port 22 cần mở** | ❌ Không | ✅ Cần | ✅ Cần |
| **Key pair** | ❌ Không cần | ✅ Bắt buộc | ✅ Bắt buộc |
| **Audit log** | ✅ Tự động | ❌ Không | ⚠️ Tùy cấu hình |
| **IAM control** | ✅ Fine-grained | ❌ Không | ⚠️ Hạn chế |
| **Private subnet** | ✅ Hỗ trợ | ❌ Cần NAT | ✅ Cần thêm bastion |
| **On-premises** | ✅ Hỗ trợ | ✅ Hỗ trợ | ✅ Hỗ trợ |
| **Chi phí** | Miễn phí | Miễn phí | Chi phí instance |

### So Sánh Parameter Store vs Secrets Manager

| Tiêu Chí | SSM Parameter Store | AWS Secrets Manager |
|----------|---------------------|---------------------|
| **Loại dữ liệu** | Config + secret | Secret (password, key) |
| **Chi phí** | Standard: Miễn phí | $0.40/secret/tháng |
| **Auto rotation** | ❌ Không có | ✅ Tích hợp sẵn |
| **Cross-account** | ⚠️ Thủ công | ✅ Native support |
| **Versioning** | ✅ Có | ✅ Có |
| **KMS encryption** | ✅ SecureString | ✅ Mặc định |
| **Dùng khi nào** | Config, non-sensitive params | DB password, API keys cần rotate |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Cơ Bản

**Q: SSM Session Manager khác SSH ở điểm nào?**

> Session Manager không cần port 22, không cần key pair, không cần bastion host. Mọi session được ghi log vào S3 hoặc CloudWatch Logs. Quyền truy cập được kiểm soát bởi IAM policy thay vì SSH key.

**Q: SSM Agent cần những gì để hoạt động?**

> Cần: (1) SSM Agent cài trên instance, (2) IAM Instance Profile với quyền SSM, (3) Kết nối outbound HTTPS đến SSM endpoints. Không cần inbound port mở.

**Q: Parameter Store vs Secrets Manager — chọn cái nào?**

> Dùng Parameter Store cho config values và secrets đơn giản không cần auto-rotation. Dùng Secrets Manager khi cần auto-rotation cho database passwords, API keys — Secrets Manager tích hợp sẵn với RDS, Redshift, DocumentDB.

### Nâng Cao

**Q: Làm thế nào patch fleet 500 EC2 mà không downtime?**

> Dùng Patch Manager với Maintenance Windows. Chia fleet thành Patch Groups (ví dụ: 10% mỗi wave). Cấu hình Patch Baseline với Critical/Important patches. Kết hợp với Auto Scaling để terminate và launch instance mới đã được patch.

**Q: Thiết kế giải pháp không dùng bastion host cho môi trường production?**

> (1) Bật SSM Session Manager, (2) Cấu hình VPC Endpoints cho SSM (ssm, ssmmessages, ec2messages), (3) Đặt EC2 trong private subnet không cần public IP/NAT, (4) Dùng IAM policy kiểm soát ai được session vào instance nào, (5) Bật session logging vào S3 encrypted + CloudWatch Logs.

**Q: Run Command vs Automation — khác nhau gì?**

> Run Command: Thực thi lệnh một bước ngay lập tức trên nhiều instance. Automation: Workflow nhiều bước phức tạp, có thể pause để chờ approval, tích hợp với AWS services khác (EC2, RDS, IAM). Ví dụ: Restart Apache → Run Command. Triển khai AMI mới với blue/green → Automation.

---

## Điều Hướng Nội Dung

| File | Chủ Đề | Tầm Quan Trọng |
|------|--------|---------------|
| [1-session-manager.md](./1-session-manager.md) | SSH-less access, port forwarding, audit logs | ⭐⭐⭐ Thiết yếu |
| [2-patch-manager.md](./2-patch-manager.md) | Patch baselines, Patch groups, maintenance windows | ⭐⭐⭐ Thiết yếu |
| [3-parameter-store.md](./3-parameter-store.md) | Standard vs Advanced, SecureString, versioning | ⭐⭐⭐ Thiết yếu |
| [4-run-command-automation.md](./4-run-command-automation.md) | Run Command, SSM Documents, Automation runbooks | ⭐⭐⭐ Thiết yếu |
| [5-inventory-compliance.md](./5-inventory-compliance.md) | Software inventory, State Manager, config compliance | ⭐⭐ Quan trọng |
| [6-distributor-opscenter.md](./6-distributor-opscenter.md) | Package distribution, OpsItems, OpsCenter | ⭐⭐ Quan trọng |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Module:** 04 / 11 — AWS Management & Governance
