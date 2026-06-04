# ElastiCache Security — Bảo Mật: VPC, Encryption, Auth Tokens, IAM

> Bảo mật ElastiCache theo mô hình Defense in Depth (Phòng Thủ Nhiều Lớp): ElastiCache **không có** network endpoint công khai — chỉ truy cập được từ trong VPC. Kết hợp VPC isolation, encryption (mã hóa), auth tokens (token xác thực), và IAM để đạt bảo mật toàn diện.

---

## 🛡️ Mô Hình Bảo Mật Toàn Diện

```
                         Internet
                            │
                   ❌ KHÔNG THỂ TRUY CẬP
                            │
┌───────────────────────────▼────────────────────────────────────┐
│                      VPC Boundary (Ranh Giới VPC)               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │            Public Subnet (Mạng Con Công Khai)            │   │
│  │  ┌─────────────────────┐                                │   │
│  │  │   Load Balancer     │                                │   │
│  │  │   (ALB)             │                                │   │
│  │  └──────────┬──────────┘                                │   │
│  └─────────────┼──────────────────────────────────────────┘   │
│                │                                                 │
│  ┌─────────────▼──────────────────────────────────────────┐   │
│  │           Private Subnet (Mạng Con Riêng Tư)            │   │
│  │                                                          │   │
│  │  ┌─────────────────┐     ┌──────────────────────────┐  │   │
│  │  │  App Servers    │────►│  ElastiCache Redis       │  │   │
│  │  │  (EC2/ECS)      │TLS  │  (Private Subnet ONLY)   │  │   │
│  │  │  Security Group │     │  Security Group          │  │   │
│  │  └─────────────────┘     └──────────────────────────┘  │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Security Layers (Lớp Bảo Mật):                                │
│  1. VPC Isolation (Cô Lập VPC) — không có public endpoint      │
│  2. Security Groups (Nhóm Bảo Mật) — tường lửa cấp instance   │
│  3. Subnet Groups (Nhóm Mạng Con) — private subnets only       │
│  4. Encryption in Transit (Mã Hóa Khi Truyền) — TLS           │
│  5. Encryption at Rest (Mã Hóa Khi Lưu) — KMS                 │
│  6. Auth Tokens (Token Xác Thực) — Redis AUTH password         │
│  7. IAM Policies (Chính Sách IAM) — quản lý cluster           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ VPC & Network Security — Bảo Mật Mạng

### Subnet Groups (Nhóm Mạng Con)

ElastiCache buộc phải đặt trong **private subnets** (mạng con riêng tư). Không có option tạo public endpoint:

```hcl
# Subnet Group — Xác định ElastiCache sẽ triển khai trên subnets nào
resource "aws_elasticache_subnet_group" "redis" {
  name = "redis-private-subnets"
  
  subnet_ids = [
    aws_subnet.private_az_a.id,  # 10.0.10.0/24 — AZ ap-southeast-1a
    aws_subnet.private_az_b.id,  # 10.0.20.0/24 — AZ ap-southeast-1b
    aws_subnet.private_az_c.id,  # 10.0.30.0/24 — AZ ap-southeast-1c
  ]
  
  description = "ElastiCache Redis subnet group — private subnets only"
  
  tags = { Environment = "production" }
}
```

### Security Groups (Nhóm Bảo Mật) — Tường Lửa Cấp Instance

```hcl
# Security Group cho ElastiCache Redis
resource "aws_security_group" "redis" {
  name        = "redis-sg"
  description = "Security group for ElastiCache Redis"
  vpc_id      = aws_vpc.main.id

  # CHỈ cho phép traffic từ App Security Group
  ingress {
    description     = "Redis từ App Servers"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app_servers.id]  # Không dùng IP tĩnh
  }

  # Không có egress rules — Redis không cần kết nối ra ngoài
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "redis-security-group" }
}

# Security Group cho App Servers
resource "aws_security_group" "app_servers" {
  name        = "app-sg"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id
  
  # App servers cần kết nối ra Redis
  egress {
    description     = "Kết nối đến Redis"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.redis.id]
  }
}
```

**Nguyên tắc Least Privilege (Đặc Quyền Tối Thiểu):**

```
✅ Đúng:  Chỉ App Security Group được truy cập Redis port 6379
✅ Đúng:  Dùng Security Group reference, không dùng IP/CIDR cứng
❌ Sai:   Mở 0.0.0.0/0 cho port 6379
❌ Sai:   Cho phép SSH vào Redis node
❌ Sai:   Đặt ElastiCache trong public subnet
```

---

## 2️⃣ Encryption in Transit — Mã Hóa Khi Truyền Tải

### TLS/SSL Encryption (Mã Hóa TLS/SSL)

Khi bật `transit_encryption_enabled`, mọi kết nối giữa client và Redis đều được mã hóa bằng TLS 1.2+:

```hcl
resource "aws_elasticache_replication_group" "redis_secure" {
  replication_group_id = "redis-production"
  
  # Bật mã hóa khi truyền tải — BẮT BUỘC cho production
  transit_encryption_enabled = true
  
  # Khi transit_encryption = true, auth_token BẮT BUỘC phải có
  auth_token = var.redis_auth_token
  
  # ...các config khác
}
```

### Kết Nối Client Với TLS

```python
import redis
import ssl

# Kết nối có TLS
redis_client = redis.Redis(
    host='my-redis.abc.cache.amazonaws.com',
    port=6379,
    password='MySecureAuthToken123!',  # Auth token
    ssl=True,                          # Bật TLS
    ssl_cert_reqs=ssl.CERT_NONE,       # Với self-signed cert (AWS dùng cert hợp lệ)
    decode_responses=True
)

# Kiểm tra kết nối
redis_client.ping()  # → True
```

```go
// Go — kết nối Redis TLS
import (
    "crypto/tls"
    "github.com/go-redis/redis/v9"
)

client := redis.NewClient(&redis.Options{
    Addr:     "my-redis.abc.cache.amazonaws.com:6379",
    Password: "MySecureAuthToken123!",
    TLSConfig: &tls.Config{
        MinVersion: tls.VersionTLS12,
    },
})
```

---

## 3️⃣ Encryption at Rest — Mã Hóa Khi Lưu Trữ

### KMS Integration (Tích Hợp KMS — Key Management Service)

ElastiCache mã hóa data at rest (dữ liệu lưu trên đĩa — RDB snapshots, AOF files) bằng KMS:

```hcl
# Tùy chọn 1: Dùng AWS Managed Key (Khóa Được Quản Lý Bởi AWS)
resource "aws_elasticache_replication_group" "redis_encrypted" {
  replication_group_id = "redis-encrypted"
  
  at_rest_encryption_enabled = true
  # Không chỉ định kms_key_id → dùng AWS managed key (aws/elasticache)
}

# Tùy chọn 2: Dùng Customer Managed Key (Khóa Do Khách Hàng Quản Lý) — Kiểm Soát Cao Hơn
resource "aws_kms_key" "redis_kms" {
  description             = "KMS key cho ElastiCache Redis encryption"
  deletion_window_in_days = 7
  
  # Key policy — ai được dùng key này
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "Enable IAM User Permissions"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
        Action    = "kms:*"
        Resource  = "*"
      },
      {
        Sid    = "Allow ElastiCache Service"
        Effect = "Allow"
        Principal = { Service = "elasticache.amazonaws.com" }
        Action = ["kms:Decrypt", "kms:GenerateDataKey*", "kms:CreateGrant"]
        Resource = "*"
      }
    ]
  })
}

resource "aws_kms_alias" "redis_kms" {
  name          = "alias/elasticache-redis"
  target_key_id = aws_kms_key.redis_kms.key_id
}

resource "aws_elasticache_replication_group" "redis_cmk" {
  replication_group_id = "redis-cmk"
  
  at_rest_encryption_enabled = true
  kms_key_id                 = aws_kms_key.redis_kms.arn  # Customer Managed Key
}
```

### Phạm Vi Mã Hóa at Rest

```
Được mã hóa:
  ✅ RDB snapshots (ảnh chụp định kỳ) trên đĩa
  ✅ AOF files trên đĩa
  ✅ ElastiCache backups trên S3
  ✅ Data được swap ra đĩa (trong trường hợp memory pressure)

KHÔNG được mã hóa bởi at-rest encryption:
  ❌ Data trong RAM (bộ nhớ) — đây là in-memory, cần transit encryption + network isolation
```

---

## 4️⃣ Auth Tokens — Token Xác Thực (Redis AUTH Password)

### Cấu Hình Auth Token

Auth token là mật khẩu Redis (lệnh `AUTH`). Bắt buộc khi `transit_encryption_enabled = true`:

```hcl
# Tạo auth token ngẫu nhiên
resource "random_password" "redis_auth" {
  length  = 64
  special = false  # Redis auth token chỉ cho phép printable ASCII, không dấu cách
}

# Lưu vào AWS Secrets Manager (Quản Lý Bí Mật)
resource "aws_secretsmanager_secret" "redis_auth" {
  name = "production/elasticache/redis-auth-token"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "redis_auth" {
  secret_id     = aws_secretsmanager_secret.redis_auth.id
  secret_string = random_password.redis_auth.result
}

# Truyền vào ElastiCache
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "redis-production"
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result
  # auth_token phải: 16-128 ký tự, chỉ printable ASCII, không dấu cách
}
```

### Rotate Auth Token (Xoay Vòng Token) — Không Downtime

```bash
# Bước 1: Thêm token mới (dual-auth mode — chấp nhận cả 2 token)
aws elasticache modify-replication-group \
    --replication-group-id my-redis \
    --auth-token "NewSecureToken456!" \
    --auth-token-update-strategy SET  # Chế độ: SET = thêm token mới, giữ token cũ

# Bước 2: Cập nhật ứng dụng để dùng token mới
# (deploy code mới với token mới — không downtime vì cả 2 token đều hoạt động)

# Bước 3: Xóa token cũ (chỉ còn token mới)
aws elasticache modify-replication-group \
    --replication-group-id my-redis \
    --auth-token "NewSecureToken456!" \
    --auth-token-update-strategy ROTATE  # Chế độ: ROTATE = xóa token cũ
```

### Ứng Dụng Đọc Token Từ Secrets Manager

```python
import boto3
import json
import redis

def get_redis_auth_token() -> str:
    """Đọc Redis auth token từ AWS Secrets Manager"""
    secrets_client = boto3.client('secretsmanager', region_name='ap-southeast-1')
    
    response = secrets_client.get_secret_value(
        SecretId='production/elasticache/redis-auth-token'
    )
    return response['SecretString']

def create_redis_client() -> redis.Redis:
    auth_token = get_redis_auth_token()
    
    return redis.Redis(
        host='my-redis.abc.cache.amazonaws.com',
        port=6379,
        password=auth_token,
        ssl=True,
        decode_responses=True,
        socket_timeout=5,
        socket_connect_timeout=5,
        retry_on_timeout=True
    )
```

---

## 5️⃣ IAM — Identity and Access Management

### Lưu Ý Quan Trọng

ElastiCache **không hỗ trợ IAM database authentication** như RDS. IAM chỉ kiểm soát quyền **quản lý cluster** (tạo, xóa, modify), không kiểm soát quyền đọc/ghi data:

```
IAM Controls (IAM Kiểm Soát):
  ✅ elasticache:CreateReplicationGroup
  ✅ elasticache:DeleteReplicationGroup
  ✅ elasticache:ModifyReplicationGroup
  ✅ elasticache:DescribeReplicationGroups
  ✅ elasticache:CreateSnapshot / DeleteSnapshot
  ✅ elasticache:AddTagsToResource

IAM KHÔNG kiểm soát:
  ❌ Đọc/ghi key trong Redis
  ❌ SELECT/SET/GET operations (dùng Auth Token thay thế)
```

### IAM Policy Mẫu (Principle of Least Privilege — Đặc Quyền Tối Thiểu)

```hcl
# Policy cho Application — chỉ cần đọc config, không manage cluster
resource "aws_iam_policy" "app_elasticache_read" {
  name        = "AppElastiCacheRead"
  description = "Cho phép app đọc thông tin ElastiCache cluster"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadElastiCacheConfig"
        Effect = "Allow"
        Action = [
          "elasticache:DescribeReplicationGroups",
          "elasticache:DescribeCacheClusters",
          "elasticache:ListTagsForResource"
        ]
        Resource = "arn:aws:elasticache:ap-southeast-1:123456789:replicationgroup:redis-production"
      },
      {
        Sid    = "ReadSecretsManagerForRedisAuth"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:production/elasticache/*"
      }
    ]
  })
}

# Policy cho DevOps — manage cluster nhưng không xóa
resource "aws_iam_policy" "devops_elasticache_manage" {
  name = "DevOpsElastiCacheManage"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ManageElastiCache"
        Effect = "Allow"
        Action = [
          "elasticache:Describe*",
          "elasticache:List*",
          "elasticache:ModifyReplicationGroup",
          "elasticache:CreateSnapshot",
          "elasticache:RestoreReplicationGroupFromS3"
        ]
        Resource = "*"
      },
      {
        # Cần approve riêng để xóa
        Sid    = "DenyDeleteWithoutApproval"
        Effect = "Deny"
        Action = [
          "elasticache:DeleteReplicationGroup",
          "elasticache:DeleteCacheCluster"
        ]
        Resource = "*"
      }
    ]
  })
}
```

### ElastiCache User Groups — RBAC Cho Redis 6+

Redis 6.0 giới thiệu **ACL** (Access Control Lists — Danh Sách Kiểm Soát Truy Cập), ElastiCache hỗ trợ qua User Groups:

```hcl
# Tạo User với quyền hạn chế
resource "aws_elasticache_user" "readonly_user" {
  user_id       = "redis-readonly"
  user_name     = "readonly"
  access_string = "on ~* &* -@all +@read"
  # Giải thích:
  # on       = user được bật
  # ~*       = có thể truy cập tất cả keys
  # &*       = có thể pub/sub tất cả channels
  # -@all    = từ chối tất cả commands
  # +@read   = cho phép tất cả read commands

  engine = "REDIS"
  passwords = ["ReadOnlyPassword123!"]
}

resource "aws_elasticache_user" "writer_user" {
  user_id       = "redis-writer"
  user_name     = "writer"
  access_string = "on ~app:* -@all +@read +@write +@hash +@string"
  # ~app:*  = chỉ truy cập keys bắt đầu bằng "app:"
  # +@write = cho phép ghi
  
  engine    = "REDIS"
  passwords = ["WriterPassword456!"]
}

# User Group — gom users lại để assign cho cluster
resource "aws_elasticache_user_group" "production" {
  engine        = "REDIS"
  user_group_id = "production-users"
  user_ids      = [
    "default",                          # User mặc định của Redis
    aws_elasticache_user.readonly_user.user_id,
    aws_elasticache_user.writer_user.user_id
  ]
}

# Assign User Group vào Replication Group
resource "aws_elasticache_replication_group" "redis_rbac" {
  replication_group_id       = "redis-rbac"
  transit_encryption_enabled = true
  user_group_ids             = [aws_elasticache_user_group.production.user_group_id]
  # Lưu ý: Khi dùng user_group_ids, không dùng auth_token
}
```

---

## 6️⃣ Compliance & Audit — Tuân Thủ & Kiểm Toán

### CloudTrail Integration (Tích Hợp CloudTrail)

CloudTrail ghi lại mọi API call liên quan đến management plane (tầng quản lý) của ElastiCache:

```json
// Ví dụ CloudTrail event khi ai đó xóa ElastiCache cluster
{
  "eventName": "DeleteReplicationGroup",
  "eventSource": "elasticache.amazonaws.com",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "nguyen.van.a",
    "arn": "arn:aws:iam::123456789:user/nguyen.van.a"
  },
  "sourceIPAddress": "203.0.113.42",
  "requestParameters": {
    "replicationGroupId": "redis-production",
    "retainPrimaryCluster": false
  },
  "eventTime": "2026-05-15T10:30:00Z",
  "awsRegion": "ap-southeast-1"
}
```

### Config Rules (Quy Tắc Config) — Kiểm Tra Tự Động

```hcl
# AWS Config rule: Đảm bảo ElastiCache bật encryption at rest
resource "aws_config_config_rule" "elasticache_encryption_at_rest" {
  name = "elasticache-at-rest-encryption-enabled"

  source {
    owner             = "AWS"
    source_identifier = "ELASTICACHE_REPL_GRP_ENCRYPTED_AT_REST"
  }
  
  # Tự động detect compliance vi phạm
}

# AWS Config rule: Đảm bảo bật automatic failover
resource "aws_config_config_rule" "elasticache_failover" {
  name = "elasticache-automatic-failover-enabled"

  source {
    owner             = "AWS"
    source_identifier = "ELASTICACHE_REPL_GRP_AUTO_FAILOVER_ENABLED"
  }
}
```

---

## 7️⃣ Security Checklist (Danh Sách Kiểm Tra Bảo Mật)

### Triển Khai Production

```
VPC & Network:
  ✅ ElastiCache trong private subnet — không có public endpoint
  ✅ Security Group chỉ cho phép App Security Group trên port 6379
  ✅ Không dùng 0.0.0.0/0 trong Security Group rules
  ✅ Multi-AZ với replica ở AZ khác nhau

Encryption (Mã Hóa):
  ✅ transit_encryption_enabled = true (TLS)
  ✅ at_rest_encryption_enabled = true (KMS)
  ✅ auth_token được set (Redis AUTH password)
  ✅ Auth token lưu trong Secrets Manager, không hardcode

Access Control (Kiểm Soát Truy Cập):
  ✅ IAM policy tuân thủ least privilege
  ✅ Xét dùng ElastiCache User Groups (ACL) với Redis 6+
  ✅ Rotate auth token định kỳ (90 ngày hoặc khi nhân viên rời công ty)
  ✅ Không dùng default user nếu dùng User Groups

Monitoring (Giám Sát):
  ✅ CloudTrail bật ghi mọi management API calls
  ✅ CloudWatch alarms cho memory, connection, evictions
  ✅ AWS Config rules kiểm tra encryption và failover
  ✅ GuardDuty để phát hiện unusual activity (hoạt động bất thường)
```

---

## 🎓 Câu Hỏi Phỏng Vấn

### Q: "Làm thế nào bảo mật ElastiCache Redis trên AWS?"

**Trả lời mẫu:**
> "Tôi áp dụng Defense in Depth với nhiều lớp. Đầu tiên là network isolation: ElastiCache luôn trong private subnet, Security Group chỉ cho phép App tier theo security group reference, không mở ra internet.
>
> Thứ hai là encryption: bật `transit_encryption_enabled` cho TLS và `at_rest_encryption_enabled` với Customer Managed KMS key. Khi transit encryption bật, auth token là bắt buộc — tôi lưu token trong Secrets Manager và application đọc từ đó, không hardcode.
>
> Thứ ba là access control: IAM policy least privilege cho management operations. Với Redis 6+, tôi dùng ElastiCache User Groups để implement RBAC — ví dụ readonly user cho monitoring service, read-write user giới hạn key prefix cho từng service.
>
> Cuối cùng là audit: CloudTrail ghi mọi management API calls, AWS Config rules tự động kiểm tra compliance, và rotate auth token theo schedule."

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
