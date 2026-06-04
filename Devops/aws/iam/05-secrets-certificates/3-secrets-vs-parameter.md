# Secrets Manager vs Parameter Store — Hướng Dẫn Chọn Lựa

> Cả hai dịch vụ đều lưu trữ bí mật an toàn trên AWS, nhưng phục vụ mục đích khác nhau. Bài này giúp bạn chọn đúng công cụ cho từng tình huống.

---

## 📚 Mục Lục

1. [So Sánh Toàn Diện](#so-sánh-toàn-diện)
2. [Ma Trận Quyết Định](#ma-trận-quyết-định)
3. [Use Cases Điển Hình](#use-cases-điển-hình)
4. [Chi Phí So Sánh](#chi-phí-so-sánh)
5. [Kiến Trúc Kết Hợp Cả Hai](#kiến-trúc-kết-hợp-cả-hai)
6. [Migration — Chuyển Đổi Giữa Hai Dịch Vụ](#migration--chuyển-đổi-giữa-hai-dịch-vụ)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## So Sánh Toàn Diện

### Bảng So Sánh Chi Tiết

| Tính Năng | Secrets Manager | Parameter Store Standard | Parameter Store Advanced |
|---|---|---|---|
| **Mục Đích Chính** | Secrets với rotation | Config + simple secrets | Config + secrets nâng cao |
| **Chi Phí Lưu Trữ** | $0.40/secret/tháng | **Miễn phí** | $0.05/param/tháng |
| **Chi Phí API** | $0.05/10K calls | **Miễn phí** | $0.05/10K calls |
| **Kích Thước Tối Đa** | **65KB** | 4KB | 8KB |
| **Số Lượng Tối Đa** | Không giới hạn thực tế | 10,000 | 100,000 |
| **Auto-Rotation** | ✅ Native (Lambda) | ❌ | ❌ (dùng Secrets Manager) |
| **Phân Cấp (Hierarchy)** | ❌ Chỉ flat name | ✅ `/app/env/key` | ✅ `/app/env/key` |
| **Cross-Account** | ✅ Resource policy | ❌ | ❌ |
| **Multi-Region Replication** | ✅ | ❌ | ❌ |
| **Database Native Integration** | ✅ RDS, Redshift, DocumentDB | ❌ | ❌ |
| **Versioning** | ✅ (AWSCURRENT/PENDING/PREVIOUS) | ✅ (số phiên bản) | ✅ + labels |
| **Parameter Policies** | N/A | ❌ | ✅ (TTL, Notification) |
| **CloudFormation Dynamic Ref** | `resolve:secretsmanager` | `resolve:ssm` | `resolve:ssm-secure` |
| **ECS Native Secrets** | ✅ | ✅ | ✅ |
| **Lambda Powertools** | ✅ | ✅ | ✅ |
| **VPC Endpoint** | ✅ Interface endpoint | ✅ Interface endpoint | ✅ |
| **Audit via CloudTrail** | ✅ | ✅ | ✅ |
| **AWS Config Rules** | ✅ (rotation check) | ✅ | ✅ |

---

## Ma Trận Quyết Định

### Flowchart Chọn Dịch Vụ

```
BẮT ĐẦU
    │
    ▼
[Bạn cần lưu loại dữ liệu gì?]
    │
    ├── Configuration data (hostname, port, feature flags)
    │       └──→ Parameter Store Standard (miễn phí)
    │
    └── Credentials (passwords, API keys, tokens)
            │
            ▼
        [Cần auto-rotation không?]
            │
            ├── Có, rotation tự động theo lịch
            │       └──→ Secrets Manager ✅
            │
            └── Không, rotation thủ công
                    │
                    ▼
                [Cần cross-account access?]
                    │
                    ├── Có
                    │       └──→ Secrets Manager ✅
                    │
                    └── Không
                            │
                            ▼
                        [Cần replication đa vùng?]
                            │
                            ├── Có
                            │       └──→ Secrets Manager ✅
                            │
                            └── Không
                                    │
                                    ▼
                                [Chi phí quan trọng?]
                                    │
                                    ├── Có, tối thiểu chi phí
                                    │       └──→ Parameter Store SecureString ✅
                                    │
                                    └── Không quan trọng
                                            └──→ Cả hai đều được, dùng Secrets Manager
```

### Bảng Quyết Định Nhanh

| Tình Huống | Khuyến Nghị | Lý Do |
|---|---|---|
| Database password cần rotation 30 ngày | **Secrets Manager** | Native rotation cho RDS |
| API key của bên thứ ba, ít thay đổi | **Parameter Store SecureString** | Không cần rotation, tiết kiệm chi phí |
| Config hostname, port, URL | **Parameter Store String** | Miễn phí, không cần mã hóa |
| Feature flags | **Parameter Store String** | Miễn phí, thay đổi thường xuyên |
| OAuth refresh token | **Secrets Manager** | Cần rotation khi token expire |
| SSH private key | **Secrets Manager** | 65KB, cross-account, rotation |
| App config có nhiều môi trường | **Parameter Store** | Hierarchy `/app/prod/`, `/app/dev/` |
| Secret chia sẻ với 5 AWS accounts | **Secrets Manager** | Resource policy cross-account |
| Certificate PEM (có thể lớn) | **Secrets Manager** hoặc **ACM** | 65KB vs ACM quản lý tốt hơn |
| Temporary credentials (8 giờ) | **Parameter Store Advanced** | Parameter Policies TTL |

---

## Use Cases Điển Hình

### Trường Hợp 1: Startup Nhỏ, Tối Thiểu Chi Phí

```
Yêu Cầu:
- 1 ứng dụng, 3 môi trường (dev, staging, prod)
- 10 database passwords
- 5 API keys
- Không cần rotation tự động (team nhỏ, tự rotate thủ công)

Giải Pháp: Parameter Store SecureString
├── /myapp/prod/db/password     (SecureString)
├── /myapp/prod/stripe/api-key  (SecureString)
├── /myapp/staging/db/password  (SecureString)
└── ...

Chi Phí: $0 (Standard tier, miễn phí hoàn toàn)
Trade-off: Phải nhớ rotate thủ công, không có dashboard rotation status
```

### Trường Hợp 2: Fintech Production — Yêu Cầu Compliance

```
Yêu Cầu:
- Tuân thủ PCI-DSS: database credentials phải rotate mỗi 90 ngày
- Multi-region: ap-southeast-1 và ap-northeast-1
- Cross-account: 5 microservices ở 5 accounts khác nhau
- Audit đầy đủ ai đọc secret khi nào

Giải Pháp: Secrets Manager
├── prod/payments/db-credentials     → rotation 30 ngày, replicate Tokyo
├── prod/payments/stripe-secret-key  → rotation 90 ngày
└── prod/payments/jwt-signing-key    → rotation 365 ngày

Chi Phí: ~$15/tháng cho 10 secrets + API calls
Lợi Ích: Audit trail đầy đủ, tự động rotation, cross-account
```

### Trường Hợp 3: SaaS Platform — Hàng Trăm Microservices

```
Yêu Cầu:
- 50 microservices, mỗi service có 5-10 config keys
- Mix: non-sensitive config + passwords + API keys
- Feature flags thay đổi thường xuyên
- Budget eo hẹp

Giải Pháp: Kết Hợp Cả Hai

Parameter Store (cho config và non-critical secrets):
/services/{service-name}/prod/
  ├── database/host         (String — miễn phí)
  ├── database/port         (String — miễn phí)
  ├── redis/host            (String — miễn phí)
  ├── feature-flags/*       (String — miễn phí)
  └── internal-api-key      (SecureString — miễn phí)

Secrets Manager (cho critical credentials cần rotation):
prod/{service-name}/
  ├── database-password     ($0.40/secret/tháng × 50 = $20/tháng)
  └── payment-api-key       (nếu cần)

Tổng Chi Phí: ~$25/tháng vs $200/tháng (nếu dùng toàn bộ Secrets Manager)
```

### Trường Hợp 4: GitHub Actions CI/CD

```
Yêu Cầu:
- GitHub Actions cần AWS credentials để deploy
- Không dùng long-lived IAM credentials (OIDC thay thế)
- Cần lưu external service credentials (Docker Hub, Npm token)

Giải Pháp:
- AWS credentials: OIDC (không lưu vào Parameter Store hay Secrets Manager)
- Docker Hub password: Secrets Manager (rotation 90 ngày)
- NPM token: Parameter Store SecureString (ít thay đổi, tiết kiệm)

Trong GitHub Actions:
  aws-actions/configure-aws-credentials@v4 dùng OIDC
  Sau đó gọi SSM/Secrets Manager API để lấy secret khác
```

---

## Chi Phí So Sánh

### Scenario: 100 Secrets, 1 Triệu API Calls/Tháng

```
Secrets Manager:
  Lưu trữ:    100 secrets × $0.40    = $40.00
  API calls:  1M ÷ 10K × $0.05      = $5.00
  Tổng:                              = $45.00/tháng

Parameter Store Standard:
  Lưu trữ:    Miễn phí
  API calls:  Miễn phí (đến 40 TPS)
  Tổng:                              = $0.00/tháng

Parameter Store Advanced:
  Lưu trữ:    100 params × $0.05     = $5.00
  API calls:  1M ÷ 10K × $0.05      = $5.00
  Tổng:                              = $10.00/tháng
```

### Break-Even Analysis (Phân Tích Điểm Hòa Vốn)

Secrets Manager đáng đầu tư hơn khi:
- Cần auto-rotation (tránh breach từ stale credentials)
- Team không có quy trình rotation thủ công đáng tin
- Đang trong ngành được regulated (PCI-DSS, HIPAA)
- Cost of breach >> cost of service ($40/tháng << $millions breach)

---

## Kiến Trúc Kết Hợp Cả Hai

Đây là pattern phổ biến nhất trong production:

```
┌─────────────────────────────────────────────────────────┐
│                    Application Tier                      │
│                                                          │
│  Lấy config khi khởi động:                              │
│  Parameter Store → hostname, port, feature flags        │
│                                                          │
│  Lấy secrets khi cần:                                   │
│  Secrets Manager → database password, API keys          │
└─────────────────────────────────────────────────────────┘
         │                          │
         ▼                          ▼
┌─────────────────┐      ┌─────────────────────────┐
│ Parameter Store │      │    Secrets Manager       │
│                 │      │                          │
│ /app/prod/      │      │ prod/app/db-creds        │
│   db/host ─────┼──────┼→ {user, password, host}  │
│   db/port       │      │                          │
│   features/*    │      │ prod/app/stripe-key      │
│   redis/host    │      │ → "sk_live_..."          │
└─────────────────┘      └─────────────────────────┘
         │                          │
         └──────────┬───────────────┘
                    ▼
              KMS CMK (mã hóa cả hai)
```

### Terraform — Provision Cả Hai

```hcl
# Parameter Store cho config
locals {
  app_config = {
    "db/host"         = module.rds.endpoint
    "db/port"         = "5432"
    "redis/host"      = module.elasticache.endpoint
    "feature/darkmode" = "false"
  }
}

resource "aws_ssm_parameter" "app_config" {
  for_each = local.app_config
  name     = "/myapp/prod/${each.key}"
  type     = "String"
  value    = each.value
}

# Secrets Manager cho credentials
resource "aws_secretsmanager_secret" "db_creds" {
  name       = "prod/myapp/db-credentials"
  kms_key_id = aws_kms_key.main.arn

  rotation_lambda_arn = aws_lambda_function.rotate_db.arn

  rotation_rules {
    automatically_after_days = 30
  }
}

# IAM policy cho ứng dụng
resource "aws_iam_role_policy" "app_secrets" {
  name = "app-secrets-access"
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["ssm:GetParameter", "ssm:GetParametersByPath"]
        Resource = "arn:aws:ssm:*:*:parameter/myapp/prod/*"
      },
      {
        Effect   = "Allow"
        Action   = ["secretsmanager:GetSecretValue"]
        Resource = aws_secretsmanager_secret.db_creds.arn
      },
      {
        Effect   = "Allow"
        Action   = ["kms:Decrypt"]
        Resource = aws_kms_key.main.arn
        Condition = {
          StringEquals = {
            "kms:ViaService" = [
              "ssm.ap-southeast-1.amazonaws.com",
              "secretsmanager.ap-southeast-1.amazonaws.com"
            ]
          }
        }
      }
    ]
  })
}
```

---

## Migration — Chuyển Đổi Giữa Hai Dịch Vụ

### Từ Parameter Store → Secrets Manager

**Khi nào cần migrate:**
- Cần bật auto-rotation
- Cần share cross-account
- Cần multi-region replication

```python
import boto3
import json

ssm = boto3.client('ssm')
secrets = boto3.client('secretsmanager')

def migrate_param_to_secret(param_path: str, secret_name: str):
    """
    Copy SecureString từ Parameter Store sang Secrets Manager.
    """
    # 1. Lấy giá trị từ Parameter Store
    response = ssm.get_parameter(
        Name=param_path,
        WithDecryption=True
    )
    value = response['Parameter']['Value']
    description = f"Migrated from Parameter Store: {param_path}"

    # 2. Tạo secret trong Secrets Manager
    try:
        secrets.create_secret(
            Name=secret_name,
            Description=description,
            SecretString=value
        )
        print(f"✅ Đã tạo secret: {secret_name}")
    except secrets.exceptions.ResourceExistsException:
        secrets.put_secret_value(
            SecretId=secret_name,
            SecretString=value
        )
        print(f"✅ Đã cập nhật secret: {secret_name}")

    # 3. Sau khi verify ứng dụng dùng được secret mới,
    #    xóa parameter cũ (sau vài ngày)
    print(f"⚠️  Nhớ xóa parameter cũ: {param_path}")


# Migrate hàng loạt
params_to_migrate = [
    ("/myapp/prod/db/password", "prod/myapp/db-password"),
    ("/myapp/prod/stripe/key",  "prod/myapp/stripe-api-key"),
]

for param_path, secret_name in params_to_migrate:
    migrate_param_to_secret(param_path, secret_name)
```

### Từ Secrets Manager → Parameter Store

**Khi nào cần migrate:**
- Giảm chi phí khi không cần rotation
- Tích hợp với service cần hierarchy

```python
def migrate_secret_to_param(secret_name: str, param_path: str):
    """
    Copy secret từ Secrets Manager sang Parameter Store.
    """
    # Lấy secret
    response = secrets.get_secret_value(SecretId=secret_name)
    value = response.get('SecretString', '')

    # Tạo SecureString trong Parameter Store
    ssm.put_parameter(
        Name=param_path,
        Type='SecureString',
        Value=value,
        Overwrite=True
    )
    print(f"✅ Đã tạo parameter: {param_path}")
```

---

## Checklist Quyết Định

```
Dùng Secrets Manager khi:
✅ Cần auto-rotation (mỗi 30-90 ngày)
✅ Database credentials (RDS, Redshift, DocumentDB)
✅ Cần share secret với AWS account khác
✅ Cần replication đa vùng (multi-region)
✅ Secret > 4KB (đến 65KB)
✅ Cần SDK tự động retry khi rotation xảy ra
✅ Đang trong PCI-DSS, HIPAA scope

Dùng Parameter Store khi:
✅ Configuration data (host, port, URLs)
✅ Feature flags (thay đổi thường xuyên)
✅ Cần phân cấp rõ ràng (/app/env/component/key)
✅ Muốn tối thiểu chi phí
✅ Cần tham chiếu trực tiếp trong CloudFormation (resolve:ssm)
✅ Parameter Policies TTL (Advanced tier)
✅ Không cần rotation, team tự quản lý lifecycle

Dùng Cả Hai:
✅ Config → Parameter Store, Credentials → Secrets Manager
✅ Khi scale lớn và cần tối ưu chi phí lẫn tính năng
```

---

## Câu Hỏi Phỏng Vấn

**Q1: Câu hỏi phỏng vấn kinh điển: "Secrets Manager hay Parameter Store?"**

> Không có câu trả lời tuyệt đối. Dùng **Secrets Manager** khi cần auto-rotation, cross-account sharing, hoặc native database integration. Dùng **Parameter Store** khi cần hierarchy cho config, tối thiểu chi phí, hoặc feature flags. Trong thực tế, dùng cả hai: Parameter Store cho config thông thường, Secrets Manager cho credentials nhạy cảm cần rotation.

**Q2: Tại sao không dùng Parameter Store SecureString thay vì Secrets Manager hoàn toàn?**

> Vì Parameter Store SecureString không có auto-rotation native — bạn phải tự viết Lambda và EventBridge schedule. Không có resource policy cho cross-account (chỉ IAM policy). Không có multi-region replication. Đối với credentials database trong production, chi phí $0.40/secret/tháng của Secrets Manager là đáng đầu tư so với rủi ro credential breach.

**Q3: Làm sao đảm bảo ứng dụng không bị downtime trong khi Secrets Manager đang rotate?**

> Dùng **multi-user rotation strategy**: Secrets Manager duy trì hai database user (A và B), luân phiên rotate. Khi rotate user A, user B vẫn hoạt động — ứng dụng tiếp tục kết nối. SDK AWS Secrets Manager tự động retry `GetSecretValue` với version `AWSCURRENT` mới. Thiết kế ứng dụng: bắt exception `AuthenticationFailed` và retry GetSecretValue để lấy credentials mới.

**Q4: Có thể dùng Parameter Store cho Kubernetes Secrets không?**

> Có, qua **External Secrets Operator (ESO)** — một Kubernetes operator cho phép tạo K8s Secret từ Parameter Store hoặc Secrets Manager. ESO tự động sync theo interval (ví dụ mỗi 5 phút). Cách khác là dùng **AWS Secrets Store CSI Driver** để mount secrets trực tiếp như file vào Pod. Đây là cách tốt hơn lưu credentials vào K8s Secret thông thường (không mã hóa tốt).

**Q5: Khi rotate secret trong Secrets Manager, ứng dụng SDK cần làm gì?**

> SDK không tự động biết khi nào secret được rotate. Ứng dụng cần: (1) Không cache secret quá lâu (max 5 phút cache TTL), hoặc (2) Bắt exception `AuthenticationFailed` khi dùng credentials cũ → gọi lại `GetSecretValue` để lấy mới. AWS Lambda Powertools `parameters` provider tự xử lý điều này qua `max_age` parameter.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
