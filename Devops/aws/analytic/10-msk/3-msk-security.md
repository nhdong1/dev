# MSK Security — Bảo Mật Amazon MSK

> Bảo mật toàn diện cho Amazon MSK: TLS (Transport Layer Security — Bảo Mật Tầng Truyền Tải), SASL/SCRAM (Simple Authentication and Security Layer / Salted Challenge Response Authentication Mechanism — Cơ Chế Xác Thực Thách Thức Phản Hồi Có Muối), IAM Authentication (Xác Thực IAM), encryption at rest (mã hóa khi lưu trữ), VPC networking (Mạng VPC) và audit logging (Ghi Nhật Ký Kiểm Tra).

## 📚 Mục Lục

1. [Tổng Quan Bảo Mật MSK](#tổng-quan-bảo-mật-msk)
2. [Encryption — Mã Hóa](#encryption--mã-hóa)
3. [Authentication — Xác Thực](#authentication--xác-thực)
4. [Authorization — Phân Quyền](#authorization--phân-quyền)
5. [Network Security — Bảo Mật Mạng](#network-security--bảo-mật-mạng)
6. [Audit Logging — Ghi Nhật Ký Kiểm Tra](#audit-logging--ghi-nhật-ký-kiểm-tra)
7. [So Sánh Các Phương Thức Auth](#so-sánh-các-phương-thức-auth)
8. [Security Best Practices](#security-best-practices--thực-hành-tốt-nhất)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Bảo Mật MSK

MSK security được xây dựng theo mô hình defense-in-depth (bảo vệ theo chiều sâu) với nhiều lớp:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MSK Security Layers                           │
│                                                                  │
│  Layer 1: Network (Tầng Mạng)                                   │
│    VPC isolation, Security Groups, NACLs, Private Endpoints      │
│                                                                  │
│  Layer 2: Encryption in Transit (Mã Hóa Khi Truyền)            │
│    TLS 1.2/1.3 — mã hóa mọi kết nối client ↔ broker           │
│                                                                  │
│  Layer 3: Authentication (Xác Thực)                             │
│    IAM / SASL-SCRAM / mTLS — xác minh danh tính client         │
│                                                                  │
│  Layer 4: Authorization (Phân Quyền)                            │
│    Kafka ACLs / IAM Policies — kiểm soát ai được làm gì        │
│                                                                  │
│  Layer 5: Encryption at Rest (Mã Hóa Khi Lưu Trữ)             │
│    AWS KMS — mã hóa dữ liệu trên EBS                           │
│                                                                  │
│  Layer 6: Audit & Monitoring (Kiểm Tra & Giám Sát)             │
│    CloudTrail, CloudWatch, Broker Logs                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Encryption — Mã Hóa

### Encryption in Transit (Mã Hóa Khi Truyền)

MSK sử dụng TLS để mã hóa mọi kết nối giữa clients và brokers:

```
Cấu hình TLS khi tạo MSK cluster:

Tùy chọn 1: TLS Required (Bắt Buộc TLS) — Recommended cho Production
  encryptionInfo:
    encryptionInTransit:
      clientBroker: TLS          # Chỉ cho phép TLS
      inCluster: true            # Mã hóa broker-to-broker traffic

Tùy chọn 2: TLS Preferred (Ưu Tiên TLS)
  encryptionInfo:
    encryptionInTransit:
      clientBroker: TLS_PLAINTEXT  # Chấp nhận cả TLS và plaintext
      inCluster: true

Tùy chọn 3: Plaintext Only (Không Dùng Production)
  encryptionInfo:
    encryptionInTransit:
      clientBroker: PLAINTEXT    # Không mã hóa — chỉ dev/test
```

**Kafka client kết nối TLS:**

```python
# Python kafka-python client với TLS
from kafka import KafkaProducer
import ssl

ssl_context = ssl.create_default_context()
ssl_context.load_verify_locations('/path/to/ca-cert.pem')

producer = KafkaProducer(
    bootstrap_servers=['broker1:9094', 'broker2:9094'],
    security_protocol='SSL',
    ssl_context=ssl_context
)
```

```java
// Java Kafka client với TLS
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9094,broker2:9094");
props.put("security.protocol", "SSL");
props.put("ssl.truststore.location", "/path/to/kafka.client.truststore.jks");
props.put("ssl.truststore.password", "truststore-password");
```

### Encryption at Rest (Mã Hóa Khi Lưu Trữ)

MSK tự động mã hóa dữ liệu trên EBS bằng AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa):

```
Tùy chọn 1: AWS Managed Key (Khóa Được AWS Quản Lý)
  encryptionInfo:
    encryptionAtRest:
      dataVolumeKMSKeyId: Service-managed  # AWS tự tạo và quản lý key
  
  Ưu điểm: Đơn giản, không cần quản lý key
  Nhược điểm: Không có audit trail chi tiết, không thể rotate thủ công

Tùy chọn 2: Customer Managed Key — CMK (Khóa Do Khách Hàng Quản Lý)
  encryptionInfo:
    encryptionAtRest:
      dataVolumeKMSKeyId: arn:aws:kms:us-east-1:123456789012:key/mrk-xxxx

  Ưu điểm:
    - Kiểm soát hoàn toàn key lifecycle (vòng đời khóa)
    - Audit trail trong CloudTrail cho mọi decrypt operation
    - Có thể revoke (thu hồi) key → MSK không thể đọc dữ liệu
    - Hỗ trợ key rotation (xoay khóa) tự động

  KMS Key Policy cần thiết:
  {
    "Sid": "Allow MSK to use the key",
    "Effect": "Allow",
    "Principal": {
      "Service": "kafka.amazonaws.com"
    },
    "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey*"],
    "Resource": "*"
  }
```

---

## Authentication — Xác Thực

MSK hỗ trợ 4 phương thức xác thực, có thể kích hoạt cùng lúc nhiều phương thức:

### 1. IAM Authentication (Xác Thực IAM) — AWS-Native

IAM authentication cho phép clients xác thực bằng AWS Identity (Danh Tính AWS) — không cần quản lý credentials riêng cho Kafka:

```
Luồng xác thực IAM:

Client (EC2/Lambda/ECS)
  │
  ├─ Có IAM Role gắn với: AmazonMSKFullAccess hoặc policy tùy chỉnh
  │
  ▼
AWS IAM Service → Tạo temporary token (token tạm thời) từ STS
  │
  ▼
MSK Broker ← Verify token với IAM service
  │
  ▼
Kết nối thành công (hoặc từ chối nếu thiếu permission)
```

**Cấu hình client với IAM:**

```python
# Python với aws-msk-iam-auth library
from kafka import KafkaProducer
from aws_msk_iam_sasl_signer import MSKAuthTokenProvider

def oauth_cb(oauth_config):
    auth_token, expiry_ms = MSKAuthTokenProvider.generate_auth_token('us-east-1')
    return auth_token, expiry_ms / 1000

producer = KafkaProducer(
    bootstrap_servers=['broker1:9098', 'broker2:9098'],
    security_protocol='SASL_SSL',
    sasl_mechanism='OAUTHBEARER',
    sasl_oauth_token_provider=oauth_cb
)
```

**IAM Policy cho MSK client:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:AlterCluster",
        "kafka-cluster:DescribeCluster"
      ],
      "Resource": "arn:aws:kafka:us-east-1:123456789:cluster/my-cluster/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:*Topic*",
        "kafka-cluster:WriteData",
        "kafka-cluster:ReadData"
      ],
      "Resource": "arn:aws:kafka:us-east-1:123456789:topic/my-cluster/*/orders"
    },
    {
      "Effect": "Allow",
      "Action": "kafka-cluster:AlterGroup",
      "Resource": "arn:aws:kafka:us-east-1:123456789:group/my-cluster/*/order-processors"
    }
  ]
}
```

**IAM Actions (Hành Động IAM) quan trọng:**

| Action                              | Mô Tả                                          |
| ----------------------------------- | ---------------------------------------------- |
| `kafka-cluster:Connect`             | Kết nối vào cluster                            |
| `kafka-cluster:WriteData`           | Produce messages vào topic                     |
| `kafka-cluster:ReadData`            | Consume messages từ topic                      |
| `kafka-cluster:DescribeTopic`       | Lấy metadata của topic                         |
| `kafka-cluster:AlterGroup`          | Commit offsets (cần cho consumer)              |
| `kafka-cluster:DescribeGroup`       | Describe consumer group                        |
| `kafka-cluster:CreateTopic`         | Tạo topic mới                                  |
| `kafka-cluster:DeleteTopic`         | Xóa topic                                      |

**Port cho IAM Authentication:** `9098` (mTLS+IAM) hoặc `9198` (Zookeeper)

### 2. SASL/SCRAM — Username/Password Authentication

SASL/SCRAM (Simple Authentication and Security Layer / Salted Challenge Response Authentication Mechanism) cho phép xác thực bằng username/password được lưu trong AWS Secrets Manager (Trình Quản Lý Bí Mật):

```
Luồng SASL/SCRAM:

1. Tạo secret trong Secrets Manager:
   Secret name: AmazonMSK_cluster-name_user-name
   Value: {"username": "alice", "password": "super-secret-password"}

2. Associate secret với MSK cluster:
   aws kafka batch-associate-scram-secret \
     --cluster-arn <cluster-arn> \
     --secret-arn-list <secret-arn>

3. Client kết nối với credentials:
```

```python
# Python client với SASL/SCRAM
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers=['broker1:9096', 'broker2:9096'],
    security_protocol='SASL_SSL',
    sasl_mechanism='SCRAM-SHA-512',
    sasl_plain_username='alice',
    sasl_plain_password='super-secret-password'
)
```

```properties
# Java client properties
bootstrap.servers=broker1:9096,broker2:9096
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="alice" \
  password="super-secret-password";
```

**SCRAM Versions:**
- `SCRAM-SHA-256` — bảo mật tốt, tương thích rộng
- `SCRAM-SHA-512` — bảo mật cao hơn (khuyến nghị production)

**Port cho SASL/SCRAM:** `9096` (plaintext — không khuyến nghị) hoặc `9096` với TLS

### 3. mTLS — Mutual TLS (TLS Hai Chiều)

mTLS yêu cầu **cả hai phía** (client và broker) xác thực lẫn nhau bằng certificate (chứng chỉ số):

```
Luồng mTLS:

1. Tạo Private CA (Certificate Authority — Cơ Quan Chứng Thực) với ACM PCA:
   aws acm-pca create-certificate-authority ...
   → Tạo root CA cho organization

2. Issue (Cấp) client certificate từ CA:
   aws acm-pca issue-certificate \
     --certificate-authority-arn <ca-arn> \
     --csr file://client.csr \
     --signing-algorithm SHA256WITHRSA
   → Mỗi client nhận một certificate duy nhất

3. Cấu hình MSK trust CA (Tin Tưởng CA):
   ClientAuthentication:
     Tls:
       CertificateAuthorityArnList: [arn:aws:acm-pca:...]

4. Client kết nối với certificate:
```

```python
# Python client với mTLS
import ssl

ssl_context = ssl.create_default_context()
ssl_context.load_verify_locations('/path/to/ca-cert.pem')
ssl_context.load_cert_chain(
    certfile='/path/to/client-cert.pem',
    keyfile='/path/to/client-key.pem'
)

producer = KafkaProducer(
    bootstrap_servers=['broker1:9094', 'broker2:9094'],
    security_protocol='SSL',
    ssl_context=ssl_context
)
```

**Ưu điểm mTLS:**
- Certificate có thể mang thông tin identity (principal) — dùng cho Kafka ACLs
- Không có secret nào cần lưu trữ (chỉ có private key)
- Revoke (thu hồi) certificate dễ dàng qua ACM PCA

**Nhược điểm mTLS:**
- Quản lý certificate lifecycle phức tạp
- Certificate rotation cần cẩn thận để tránh downtime

### 4. No Authentication (Không Xác Thực) — Chỉ Dev/Test

```
Port: 9092 (plaintext) hoặc 9094 (TLS without auth)
Chỉ dùng trong VPC nội bộ với Security Groups hạn chế
KHÔNG BAO GIỜ dùng production với dữ liệu nhạy cảm
```

---

## Authorization — Phân Quyền

### Kafka ACLs (Access Control Lists — Danh Sách Kiểm Soát Truy Cập)

Khi dùng mTLS hoặc SASL/SCRAM, bạn quản lý authorization bằng Kafka ACLs:

```bash
# Tạo ACL cho producer "alice" trên topic "orders"
kafka-acls.sh \
  --bootstrap-server <broker>:9096 \
  --add \
  --allow-principal User:alice \
  --operation Write \
  --operation Describe \
  --topic orders

# Tạo ACL cho consumer group "order-processors"
kafka-acls.sh \
  --bootstrap-server <broker>:9096 \
  --add \
  --allow-principal User:alice \
  --operation Read \
  --operation Describe \
  --topic orders \
  --group order-processors

# Xem tất cả ACLs
kafka-acls.sh \
  --bootstrap-server <broker>:9096 \
  --list

# Xóa ACL
kafka-acls.sh \
  --bootstrap-server <broker>:9096 \
  --remove \
  --allow-principal User:alice \
  --operation Write \
  --topic orders
```

**Kafka ACL Operations (Thao Tác):**

| Operation   | Phạm Vi       | Mô Tả                                       |
| ----------- | ------------- | ------------------------------------------- |
| `Read`      | Topic, Group  | Đọc messages, commit offsets                |
| `Write`     | Topic         | Produce messages                            |
| `Create`    | Topic, Cluster| Tạo topic                                   |
| `Delete`    | Topic, Group  | Xóa topic hoặc consumer group               |
| `Describe`  | Topic, Group  | Lấy metadata                                |
| `Alter`     | Topic, Cluster| Thay đổi cấu hình topic/cluster             |
| `All`       | Tất cả        | Toàn quyền                                  |

### IAM-based Authorization (Khi Dùng IAM Auth)

Với IAM authentication, authorization được quản lý qua **IAM Policies** thay vì Kafka ACLs. Đây là cách AWS-native hơn và tích hợp tốt với AWS Organizations, SCPs (Service Control Policies — Chính Sách Kiểm Soát Dịch Vụ):

```json
// Fine-grained IAM policy cho MSK
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadFromSpecificTopic",
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:ReadData",
        "kafka-cluster:DescribeTopic",
        "kafka-cluster:AlterGroup",
        "kafka-cluster:DescribeGroup"
      ],
      "Resource": [
        "arn:aws:kafka:us-east-1:123456789:cluster/prod-cluster/*",
        "arn:aws:kafka:us-east-1:123456789:topic/prod-cluster/*/orders",
        "arn:aws:kafka:us-east-1:123456789:group/prod-cluster/*/order-processors-*"
      ]
    },
    {
      "Sid": "DenyWriteToProductionTopics",
      "Effect": "Deny",
      "Action": "kafka-cluster:WriteData",
      "Resource": "arn:aws:kafka:us-east-1:123456789:topic/prod-cluster/*"
    }
  ]
}
```

---

## Network Security — Bảo Mật Mạng

### VPC Configuration (Cấu Hình VPC)

MSK cluster phải nằm trong VPC và được truy cập qua private endpoints:

```
Kiến trúc mạng MSK production:

┌───────────────────────────────────────────────────────────┐
│                        VPC                                │
│                                                           │
│  ┌──────────────────┐     ┌───────────────────────────┐  │
│  │  Private Subnets  │     │    MSK Cluster Subnets    │  │
│  │  (Application)    │     │    (Isolated/Private)      │  │
│  │                  │     │                           │  │
│  │  EC2 instances   │────▶│  Broker-1 (AZ: 1a)        │  │
│  │  ECS tasks       │     │  Broker-2 (AZ: 1b)        │  │
│  │  Lambda (VPC)    │     │  Broker-3 (AZ: 1c)        │  │
│  └──────────────────┘     └───────────────────────────┘  │
│           ↑                          ↑                    │
│    Security Group:            Security Group:             │
│    Allow outbound 9092-9098   Allow inbound 9092-9098     │
│    from app servers           from app security group     │
└───────────────────────────────────────────────────────────┘

KHÔNG có public internet access vào MSK brokers
```

### Security Groups (Nhóm Bảo Mật)

```
MSK Broker Security Group:
  Inbound Rules (Quy Tắc Vào):
    Port 9092 (Plaintext — chỉ internal)
    Port 9094 (TLS)
    Port 9096 (SASL/SCRAM + TLS)
    Port 9098 (IAM + TLS)
    Port 2181 (ZooKeeper — chỉ cho internal tools)
    Port 2182 (ZooKeeper TLS)
  
  Nguồn (Source): Security Group của application servers
  KHÔNG cho phép: 0.0.0.0/0 (any internet)

Application Security Group:
  Outbound Rules (Quy Tắc Ra):
    Port 9092-9098 → MSK Broker Security Group
```

**MSK Port Reference:**

| Port | Protocol      | Authentication     | Encryption |
| ---- | ------------- | ------------------ | ---------- |
| 9092 | Plaintext     | None               | ❌         |
| 9094 | TLS           | mTLS (optional)    | ✅         |
| 9096 | SASL          | SCRAM              | ✅ (forced)|
| 9098 | SASL          | IAM                | ✅ (forced)|
| 2181 | ZooKeeper     | None               | ❌         |
| 2182 | ZooKeeper TLS | None               | ✅         |

### VPC Endpoints (Điểm Cuối VPC) cho MSK

```bash
# Tạo VPC Endpoint cho MSK (PrivateLink)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxx \
  --service-name com.amazonaws.us-east-1.kafka \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-xxxxx subnet-yyyyy \
  --security-group-ids sg-xxxxx

# Cho phép cross-account access qua PrivateLink
# MSK → VPC Endpoint → Consumer VPC (different account)
```

### Cross-Account Access (Truy Cập Chéo Tài Khoản)

```
Kiến trúc cross-account:

Account A (MSK Cluster)         Account B (Consumer Application)
┌───────────────────┐           ┌──────────────────────────┐
│  MSK Cluster      │           │  EC2 / ECS               │
│  Private DNS:     │           │  IAM Role:               │
│  broker.kafka:9098│◀──────────│  - sts:AssumeRole (Acc A)│
│                   │           │  - kafka-cluster:* (Acc A)│
└───────────────────┘           └──────────────────────────┘
         ↑
VPC Peering hoặc Transit Gateway (Cổng Quá Cảnh)
hoặc AWS PrivateLink

Steps:
1. VPC Peering giữa hai accounts
2. Update route tables (bảng định tuyến)
3. Security group rules cho cross-account CIDR
4. IAM role với trust policy cho Account B
5. Consumer trong Account B assume role của Account A
```

---

## Audit Logging — Ghi Nhật Ký Kiểm Tra

### Broker Logs (Nhật Ký Broker)

```bash
# Cấu hình broker logs khi tạo cluster (CloudFormation/Terraform)
LoggingInfo:
  BrokerLogs:
    CloudWatchLogs:
      Enabled: true
      LogGroup: /aws/msk/cluster/my-cluster
    Firehose:
      Enabled: true
      DeliveryStream: my-msk-logs-stream  # → S3 long-term storage
    S3:
      Enabled: true
      Bucket: my-msk-logs-bucket
      Prefix: msk-logs/

# Broker logs chứa:
# - Broker startup/shutdown events
# - Topic creation/deletion
# - Authentication failures
# - Partition leader changes
# - Replication lag warnings
```

### CloudTrail Integration (Tích Hợp CloudTrail)

CloudTrail tự động ghi lại mọi API call đến MSK control plane (mặt phẳng điều khiển):

```json
// Ví dụ CloudTrail event khi tạo topic
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDAXXX",
    "arn": "arn:aws:iam::123456789012:user/alice",
    "accountId": "123456789012"
  },
  "eventTime": "2026-05-17T10:30:00Z",
  "eventSource": "kafka.amazonaws.com",
  "eventName": "CreateTopic",
  "requestParameters": {
    "clusterArn": "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster",
    "topicName": "orders",
    "numPartitions": 12,
    "replicationFactor": 3
  },
  "responseElements": null,
  "sourceIPAddress": "10.0.1.100"
}
```

### CloudWatch Metrics (Chỉ Số CloudWatch) cho Security Monitoring

```
Security-relevant metrics (Chỉ số liên quan bảo mật):

kafka.network:requests-per-sec
  → Đột ngột tăng có thể là DDoS hoặc misconfigured client

kafka.server:failed-authentication-total
  → Tăng → Có client đang brute-force hoặc wrong credentials

kafka.server:unauthorized-requests-per-sec  
  → Client không có permission đang cố truy cập

CloudWatch Alarms (Cảnh Báo CloudWatch):
  - Alarm khi failed-authentication > 100/phút
  - Alarm khi unauthorized-requests > 50/phút
  - SNS notification → PagerDuty / Slack
```

---

## So Sánh Các Phương Thức Auth

| Tiêu Chí                         | IAM Auth            | SASL/SCRAM          | mTLS                 |
| -------------------------------- | ------------------- | ------------------- | -------------------- |
| **Phức tạp cấu hình**            | Thấp (AWS-native)   | Trung bình          | Cao                  |
| **Quản lý credentials**          | AWS xử lý (STS token)| Secrets Manager    | Certificate lifecycle|
| **Authorization**                | IAM Policies        | Kafka ACLs          | Kafka ACLs           |
| **Audit trail**                  | CloudTrail đầy đủ   | Broker logs         | Broker logs          |
| **Cross-account**                | ✅ IAM Role assume  | ❌ Khó              | ❌ Khó               |
| **Rotate credentials**           | Tự động (token 15ph)| Thủ công/Secrets Mgr| Certificate rotation |
| **On-premises clients**          | ❌ Cần AWS SDK      | ✅ Kafka native     | ✅ Kafka native       |
| **MSK Serverless support**       | ✅ Bắt buộc         | ❌                  | ❌                   |
| **Khuyến nghị**                  | AWS-native workloads| Hybrid/migration    | Security requirement |

**Khi nào chọn phương thức nào:**

```
IAM Auth → Khi:
  ✅ Toàn bộ consumers/producers là AWS services (EC2, Lambda, ECS)
  ✅ Muốn tập trung quản lý permissions qua IAM
  ✅ Cần audit trail CloudTrail chi tiết
  ✅ Dùng MSK Serverless (IAM là bắt buộc)

SASL/SCRAM → Khi:
  ✅ Có clients bên ngoài AWS (on-premises, third-party)
  ✅ Migrating từ Kafka self-managed với SCRAM
  ✅ Team muốn dùng Kafka-native auth (không muốn phụ thuộc AWS SDK)

mTLS → Khi:
  ✅ Compliance yêu cầu certificate-based authentication
  ✅ Cần client identity trong Kafka ACLs (dựa trên CN — Common Name của certificate)
  ✅ Zero-trust security model (mô hình bảo mật không tin tưởng)
```

---

## Security Best Practices — Thực Hành Tốt Nhất

### 1. Encryption Best Practices

```bash
# Bắt buộc TLS — không cho phép plaintext
clientBroker: TLS  # Không phải TLS_PLAINTEXT

# Dùng Customer Managed Key cho sensitive data
aws kms create-key --description "MSK encryption key"
aws kms create-alias --alias-name alias/msk-key --target-key-id <key-id>

# Rotate KMS key hàng năm
aws kms enable-key-rotation --key-id <key-id>
```

### 2. Network Best Practices

```
✅ MSK trong private subnets — không expose ra internet
✅ Security Groups chỉ allow traffic từ known application SGs
✅ Không mở port 2181 (ZooKeeper) ra ngoài MSK internal network
✅ Dùng VPC Endpoints cho cross-account nếu cần
✅ Bật VPC Flow Logs (Nhật Ký Luồng VPC) để audit network traffic
```

### 3. Authentication Best Practices

```
✅ Luôn dùng authentication — không production với unauthenticated access
✅ Ưu tiên IAM auth cho AWS-native workloads
✅ Không share credentials giữa applications — mỗi service có credentials riêng
✅ Dùng Secrets Manager để lưu SCRAM passwords — không hardcode trong code
✅ Rotate passwords định kỳ (ít nhất 90 ngày)
✅ Revoke credentials ngay khi không cần (nhân viên nghỉ việc, service ngừng)
```

### 4. Authorization Best Practices

```
Principle of Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu):
  ✅ Producer chỉ cần Write + Describe
  ✅ Consumer chỉ cần Read + Describe + AlterGroup (cho commit offset)
  ✅ Admin service mới cần Create/Delete/Alter topic
  ✅ Không dùng wildcard (*) cho topic/group resource nếu có thể

IAM Policy example — Producer:
  kafka-cluster:Connect
  kafka-cluster:WriteData
  kafka-cluster:DescribeTopic
  Resource: specific topic ARN (không phải *)

IAM Policy example — Consumer:
  kafka-cluster:Connect
  kafka-cluster:ReadData
  kafka-cluster:DescribeTopic
  kafka-cluster:DescribeGroup
  kafka-cluster:AlterGroup
  Resource: specific topic ARN + specific group ARN
```

### 5. Monitoring Best Practices

```bash
# Tạo CloudWatch alarm cho authentication failures
aws cloudwatch put-metric-alarm \
  --alarm-name "MSK-AuthFailures-High" \
  --metric-name "kafka.network:failed-authentication-total" \
  --namespace "AWS/Kafka" \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --period 60 \
  --alarm-actions arn:aws:sns:us-east-1:xxx:security-alerts

# Enable broker logs đến CloudWatch và S3
# Enable CloudTrail cho API audit
# Review IAM policies mỗi quý
```

### 6. Compliance Checklist (Danh Sách Kiểm Tra Tuân Thủ)

```
Cho PCI-DSS (Payment Card Industry Data Security Standard):
  ✅ TLS 1.2+ bắt buộc
  ✅ Encryption at rest với CMK
  ✅ Audit logging đầy đủ (CloudTrail + Broker logs)
  ✅ Network isolation trong VPC
  ✅ Access control granular (chi tiết)
  ✅ Key rotation định kỳ

Cho HIPAA (Health Insurance Portability and Accountability Act):
  ✅ Business Associate Agreement (BAA) với AWS
  ✅ Encryption at rest và in transit
  ✅ Audit logs được giữ ít nhất 6 năm
  ✅ Access logging chi tiết
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: MSK hỗ trợ những phương thức authentication nào?**

> MSK hỗ trợ 4 phương thức: (1) **Unauthenticated** — không xác thực, chỉ dev/test trong VPC nội bộ; (2) **mTLS** — mutual TLS dùng certificate từ ACM Private CA, client và broker xác thực nhau; (3) **SASL/SCRAM** — username/password lưu trong AWS Secrets Manager; (4) **IAM Authentication** — dùng AWS identity, phù hợp nhất cho AWS-native workloads. Có thể kích hoạt nhiều phương thức cùng lúc trên cùng cluster.

**Q: Encryption at rest trong MSK hoạt động thế nào?**

> MSK tự động mã hóa dữ liệu trên EBS bằng AWS KMS. Có thể dùng AWS Managed Key (đơn giản, không cần quản lý) hoặc Customer Managed Key — CMK (kiểm soát hoàn toàn, audit trail CloudTrail, có thể revoke). CMK phải có key policy cho phép `kafka.amazonaws.com` service principal thực hiện `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey*`.

### Nâng Cao

**Q: So sánh IAM Authentication và SASL/SCRAM — khi nào chọn cái nào?**

> **IAM Authentication** phù hợp khi toàn bộ clients là AWS services (EC2, Lambda, ECS) — không cần quản lý credentials vì dùng IAM Role và STS temporary tokens (tự rotate mỗi 15 phút). Audit trail đầy đủ qua CloudTrail. Hạn chế: cần AWS SDK, không tương thích clients bên ngoài AWS. **SASL/SCRAM** phù hợp khi có clients on-premises hoặc non-AWS (chỉ cần Kafka native library), migrating từ self-managed Kafka, hoặc team không muốn phụ thuộc AWS SDK. Credentials lưu trong Secrets Manager. Hạn chế: phải rotate password thủ công, audit trail ít chi tiết hơn IAM.

**Q: Làm thế nào để implement least-privilege access cho MSK với nhiều microservices?**

> Với IAM Auth: Tạo IAM Role riêng cho mỗi microservice với policy chỉ cho phép thao tác cần thiết trên topic và consumer group cụ thể — producer chỉ có `WriteData` + `DescribeTopic`, consumer chỉ có `ReadData` + `AlterGroup` + `DescribeGroup`. Resource ARN phải specify đúng topic/group, không dùng wildcard. Với SASL/SCRAM + Kafka ACLs: Tạo user riêng cho mỗi service, grant ACLs chính xác (Write/Read/Describe theo topic), không cấp `ALTER CLUSTER` cho service thông thường. Audit định kỳ ACLs và IAM policies để phát hiện permission thừa.

---

## 🔗 Điều Hướng

| Trước                                              | Module Tiếp Theo                                |
| -------------------------------------------------- | ----------------------------------------------- |
| [2-msk-vs-kinesis.md](2-msk-vs-kinesis.md) — So sánh | [../11-data-architecture/README.md](../11-data-architecture/README.md) — Kiến trúc dữ liệu |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Độ Khó:** ⭐⭐⭐ Nâng Cao
