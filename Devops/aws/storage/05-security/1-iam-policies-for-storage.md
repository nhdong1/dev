# IAM Policies cho AWS Storage — S3, EBS, EFS

> IAM — Identity and Access Management — Quản Lý Danh Tính và Truy Cập: nền tảng kiểm soát quyền truy cập cho mọi dịch vụ AWS storage.

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản IAM](#1-khái-niệm-cơ-bản-iam)
2. [IAM Policy Structure](#2-iam-policy-structure)
3. [IAM Policies cho S3](#3-iam-policies-cho-s3)
4. [IAM Policies cho EBS](#4-iam-policies-cho-ebs)
5. [IAM Policies cho EFS](#5-iam-policies-cho-efs)
6. [Best Practices](#6-best-practices)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Khái Niệm Cơ Bản IAM

### Principal — Chủ Thể

Đối tượng được xác thực và được phép thực hiện hành động:

| Principal | Mô Tả | Dùng Cho |
|-----------|--------|---------|
| **IAM User** | Người dùng dài hạn với credentials | Developer, CI/CD cũ |
| **IAM Role** | Quyền tạm thời, assume bởi service/user | EC2, Lambda, ECS tasks |
| **IAM Group** | Nhóm user chia sẻ policy | Phân nhóm team |
| **AWS Service** | EC2, Lambda, ECS tự assume role | Không cần access key |
| **Federated User** | SSO qua SAML/OIDC | Doanh nghiệp dùng AD |

### Policy Types — Loại Policy

```
┌─────────────────────────────────────────────────────────────┐
│                    Các Loại IAM Policy                       │
├──────────────────────┬──────────────────────────────────────┤
│ Identity-based       │ Gắn vào User/Role/Group              │
│ (Dựa trên danh tính) │ → Kiểm soát "ai được làm gì"        │
├──────────────────────┼──────────────────────────────────────┤
│ Resource-based       │ Gắn vào resource (S3 bucket, KMS key)│
│ (Dựa trên tài nguyên)│ → Kiểm soát "ai được truy cập cái này│
├──────────────────────┼──────────────────────────────────────┤
│ Permission Boundary  │ Giới hạn tối đa quyền của Role/User  │
│ (Ranh giới quyền)    │ → Ngăn privilege escalation          │
├──────────────────────┼──────────────────────────────────────┤
│ SCPs                 │ Áp dụng cho toàn bộ AWS Organization │
│ (Service Control)    │ → Guardrail ở cấp org               │
└──────────────────────┴──────────────────────────────────────┘
```

---

## 2. IAM Policy Structure

### Cấu Trúc JSON Cơ Bản

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TênMôTảStatement",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MyRole"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "s3:prefix": "uploads/"
        }
      }
    }
  ]
}
```

**Lưu ý quan trọng:**
- `Principal` chỉ dùng trong resource-based policies (bucket policy, trust policy)
- Identity-based policies **không** có trường `Principal`
- `Resource` là ARN — Amazon Resource Name — Tên Tài Nguyên Amazon

### Effect — Hiệu Lực

| Effect | Ý Nghĩa | Ưu Tiên |
|--------|---------|---------|
| `Allow` | Cho phép action | Mặc định bị từ chối |
| `Deny` | Từ chối tường minh | **Ghi đè mọi Allow** |

---

## 3. IAM Policies cho S3

### 3.1. Policy Chỉ Đọc — Read-Only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

**Giải thích:**
- `s3:GetObject` — tải object xuống
- `s3:ListBucket` cần ARN bucket (không có `/*`)
- `s3:GetObject` cần ARN object (có `/*`)

### 3.2. Policy Ghi Vào Prefix Cụ Thể

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::data-bucket",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["uploads/*", ""]
        }
      }
    },
    {
      "Sid": "AllowWriteToUploads",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::data-bucket/uploads/*"
    }
  ]
}
```

### 3.3. Policy Cho EC2 Backup Lên S3

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BackupToS3",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::backup-bucket",
        "arn:aws:s3:::backup-bucket/*"
      ]
    },
    {
      "Sid": "KMSForEncryption",
      "Effect": "Allow",
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxx"
    }
  ]
}
```

### 3.4. Buộc Mã Hóa Khi Upload

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::secure-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

### 3.5. Policy Cho Lambda Xử Lý S3 Events

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSourceBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectTagging"
      ],
      "Resource": "arn:aws:s3:::source-bucket/*"
    },
    {
      "Sid": "WriteDestinationBucket",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectTagging"
      ],
      "Resource": "arn:aws:s3:::destination-bucket/processed/*"
    }
  ]
}
```

### 3.6. S3 Actions Theo Nhóm

```
LIST operations (cần ARN bucket, không có /*):
  s3:ListBucket
  s3:ListBucketVersions
  s3:ListBucketMultipartUploads
  s3:GetBucketLocation

OBJECT operations (cần ARN với /*):
  s3:GetObject, s3:PutObject, s3:DeleteObject
  s3:GetObjectVersion, s3:DeleteObjectVersion
  s3:CopyObject

BUCKET MANAGEMENT (cần ARN bucket):
  s3:CreateBucket, s3:DeleteBucket
  s3:GetBucketPolicy, s3:PutBucketPolicy
  s3:GetBucketVersioning, s3:PutBucketVersioning
```

---

## 4. IAM Policies cho EBS

### 4.1. Quyền Quản Lý EBS Volume

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EBSManagement",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumeStatus",
        "ec2:CreateVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DeleteVolume"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EBSSnapshots",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot",
        "ec2:DescribeSnapshots",
        "ec2:CopySnapshot"
      ],
      "Resource": "*"
    }
  ]
}
```

### 4.2. Giới Hạn Xóa Volume Theo Tag

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDeleteOnlyDevVolumes",
      "Effect": "Allow",
      "Action": "ec2:DeleteVolume",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Environment": "dev"
        }
      }
    },
    {
      "Sid": "DenyDeleteProdVolumes",
      "Effect": "Deny",
      "Action": "ec2:DeleteVolume",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Environment": "prod"
        }
      }
    }
  ]
}
```

### 4.3. Policy Cho AWS Backup Agent

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EBSBackupPermissions",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    },
    {
      "Sid": "KMSForEBSEncryption",
      "Effect": "Allow",
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:*:*:key/*",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": true
        }
      }
    }
  ]
}
```

---

## 5. IAM Policies cho EFS

### 5.1. Quyền Mount EFS Cơ Bản

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EFSMountPermissions",
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite",
        "elasticfilesystem:ClientRootAccess",
        "elasticfilesystem:DescribeFileSystems",
        "elasticfilesystem:DescribeMountTargets"
      ],
      "Resource": "arn:aws:elasticfilesystem:ap-southeast-1:123456789012:file-system/fs-xxxxxxxx"
    }
  ]
}
```

### 5.2. EFS File System Policy — Giới Hạn Theo Access Point

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRootDirectory",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/AppServerRole"
      },
      "Action": [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite"
      ],
      "Resource": "arn:aws:elasticfilesystem:ap-southeast-1:123456789012:file-system/fs-xxxxxxxx",
      "Condition": {
        "StringEquals": {
          "elasticfilesystem:AccessPointArn": "arn:aws:elasticfilesystem:ap-southeast-1:123456789012:access-point/fsap-xxxxxxxx"
        }
      }
    },
    {
      "Sid": "DenyRootAccessWithoutAccessPoint",
      "Effect": "Deny",
      "Principal": {"AWS": "*"},
      "Action": "elasticfilesystem:ClientRootAccess",
      "Resource": "arn:aws:elasticfilesystem:ap-southeast-1:123456789012:file-system/fs-xxxxxxxx",
      "Condition": {
        "Bool": {
          "elasticfilesystem:AccessedViaMountTarget": true
        },
        "Null": {
          "elasticfilesystem:AccessPointArn": true
        }
      }
    }
  ]
}
```

### 5.3. EFS Actions Theo Nhóm

```
MANAGEMENT (quản lý file system):
  elasticfilesystem:CreateFileSystem
  elasticfilesystem:DeleteFileSystem
  elasticfilesystem:DescribeFileSystems
  elasticfilesystem:UpdateFileSystem

CLIENT (EC2/container mount):
  elasticfilesystem:ClientMount     → chỉ đọc
  elasticfilesystem:ClientWrite     → đọc + ghi
  elasticfilesystem:ClientRootAccess → root user trên NFS

ACCESS POINTS:
  elasticfilesystem:CreateAccessPoint
  elasticfilesystem:DeleteAccessPoint
  elasticfilesystem:DescribeAccessPoints
```

---

## 6. Best Practices

### 6.1. Least Privilege — Nguyên Tắc Quyền Tối Thiểu

```
❌ Tránh:
{
  "Action": "s3:*",
  "Resource": "*"
}

✅ Tốt hơn:
{
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::specific-bucket/specific-prefix/*"
}
```

### 6.2. Dùng IAM Role Thay Cho Access Key

```bash
# ❌ Không nên: hardcode credentials
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...

# ✅ Nên: gắn IAM Role vào EC2/Lambda
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxxxxxxx \
  --iam-instance-profile Name=MyAppRole
```

### 6.3. Dùng Condition Keys Để Tăng Độ Chính Xác

```json
{
  "Condition": {
    "IpAddress": {
      "aws:SourceIp": ["10.0.0.0/8", "172.16.0.0/12"]
    },
    "StringEquals": {
      "aws:RequestedRegion": "ap-southeast-1"
    },
    "Bool": {
      "aws:SecureTransport": true
    }
  }
}
```

### 6.4. Permission Boundary — Ranh Giới Quyền

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StorageOnlyBoundary",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "ec2:*Volume*",
        "ec2:*Snapshot*",
        "elasticfilesystem:*"
      ],
      "Resource": "*"
    }
  ]
}
```

Dùng Permission Boundary khi developer tự tạo Role nhưng cần giới hạn phạm vi quyền họ có thể cấp.

### 6.5. Audit IAM với AWS CLI

```bash
# Xem IAM user có quyền gì với S3
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/developer \
  --action-names s3:DeleteBucket \
  --resource-arns arn:aws:s3:::prod-bucket

# Liệt kê access keys
aws iam list-access-keys --user-name developer

# Xem credential report
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Khác biệt giữa IAM policy và S3 bucket policy?**

A: IAM policy gắn vào principal (user/role), kiểm soát những gì identity đó được làm. Bucket policy gắn vào resource (bucket), kiểm soát ai được truy cập bucket đó — kể cả cross-account. Khi cả hai cùng tồn tại, quyền cuối cùng là hợp của cả hai, trừ khi có Deny tường minh.

---

**Q: Tại sao nên dùng IAM Role thay vì IAM User với Access Key cho EC2?**

A: IAM Role cấp credentials tạm thời (STS — Security Token Service — Dịch Vụ Token Bảo Mật) tự rotate, không cần lưu trữ key. Access key tĩnh có nguy cơ bị lộ qua source code, logs, hoặc environment variables. Role còn dễ audit và thu hồi quyền hơn.

---

**Q: Permission Boundary là gì và khi nào dùng?**

A: Permission Boundary là metadata policy xác định mức quyền tối đa mà IAM entity có thể có. Ngay cả khi attach policy cấp quyền rộng hơn, effective permission vẫn bị giới hạn bởi boundary. Dùng khi trao quyền cho developer tự tạo role nhưng cần đảm bảo họ không thể tạo role có quyền vượt qua ranh giới được định sẵn — ngăn privilege escalation.

---

**Q: Condition key `aws:SecureTransport` hoạt động thế nào?**

A: `aws:SecureTransport` trả về `true` khi request gửi qua HTTPS/TLS, `false` khi gửi qua HTTP. Dùng trong Deny statement với `"Bool": {"aws:SecureTransport": false}` để buộc mọi client phải dùng HTTPS, ngăn chặn man-in-the-middle attack.

---

**Cập Nhật:** 2026-05-16
