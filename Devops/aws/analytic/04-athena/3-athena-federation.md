# Athena Federated Query — Truy Vấn Liên Kết Đa Nguồn Dữ Liệu

> Athena Federated Query (Truy Vấn Liên Kết) cho phép chạy một câu SQL duy nhất để join dữ liệu từ nhiều nguồn khác nhau: S3, RDS, DynamoDB, Redshift, Redis, HBase, và hơn thế nữa — không cần di chuyển dữ liệu.

---

## 📚 Mục Lục

1. [Federated Query Là Gì?](#1-federated-query-là-gì)
2. [Kiến Trúc Lambda Connector](#2-kiến-trúc-lambda-connector)
3. [Pre-built Connectors Có Sẵn](#3-pre-built-connectors-có-sẵn)
4. [Thiết Lập Connector — Ví Dụ MySQL/RDS](#4-thiết-lập-connector--ví-dụ-mysqlrds)
5. [Thiết Lập DynamoDB Connector](#5-thiết-lập-dynamodb-connector)
6. [Viết Federated Query](#6-viết-federated-query)
7. [Performance Considerations](#7-performance-considerations)
8. [Bảo Mật Federated Query](#8-bảo-mật-federated-query)
9. [Custom Connector — Tự Viết Connector](#9-custom-connector--tự-viết-connector)
10. [Use Cases và Anti-patterns](#10-use-cases-và-anti-patterns)

---

## 1. Federated Query Là Gì?

### Bài Toán Trước Khi Có Federated Query

```
Tình huống: Muốn phân tích đơn hàng (trong RDS MySQL) kết hợp
với dữ liệu hành vi người dùng (trong S3) và thông tin session (DynamoDB)

Cách cũ:
1. Export data từ MySQL → S3 (Glue/DMS — Database Migration Service)
2. Export data từ DynamoDB → S3 (DynamoDB Export hoặc Lambda)
3. Chờ ETL hoàn thành (1-24 giờ)
4. Query trên S3 bằng Athena
→ Data stale (lỗi thời), pipeline phức tạp, chi phí ETL cao

Với Federated Query:
1. Viết một SQL query
2. Athena tự kết nối đến MySQL, DynamoDB, và S3 song song
3. Merge kết quả
→ Real-time hoặc near-real-time, không cần ETL
```

### Định Nghĩa

**Athena Federated Query** cho phép Athena hoạt động như một **query federation layer** (lớp liên kết truy vấn), kết nối đến nhiều data source thông qua **Lambda-based connectors** (connector dựa trên Lambda).

---

## 2. Kiến Trúc Lambda Connector

### Luồng Thực Thi Federated Query

```
┌─────────────────────────────────────────────────────────────────┐
│                         Athena Engine                            │
│                                                                   │
│  SQL: SELECT o.order_id, u.name, s.session_duration              │
│       FROM mysql.orders o                                         │
│       JOIN s3.user_events u ON o.customer_id = u.user_id         │
│       JOIN dynamo.sessions s ON o.session_id = s.session_id      │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Query Planner                          │    │
│  │  Nhận diện 3 data sources → Tạo 3 subplans độc lập      │    │
│  └──────┬──────────────────────┬────────────────────────────┘   │
│         │                      │                                  │
└─────────┼──────────────────────┼──────────────────────────────────┘
          │                      │
    ┌─────▼──────┐    ┌──────────▼──────┐    ┌────────────────┐
    │  Lambda    │    │    Lambda        │    │  S3 Native     │
    │  Connector │    │    Connector     │    │  (trực tiếp)   │
    │  (MySQL)   │    │    (DynamoDB)    │    │                │
    └─────┬──────┘    └──────────┬───────┘    └────────┬───────┘
          │                      │                      │
    ┌─────▼──────┐    ┌──────────▼───────┐    ┌────────▼───────┐
    │  RDS MySQL  │    │  Amazon DynamoDB │    │  S3 Parquet    │
    └────────────┘    └──────────────────┘    └────────────────┘
          │                      │                      │
          └──────────────────────┴──────────────────────┘
                                 │
                    Athena merge kết quả từ 3 sources
                                 │
                    S3 Output Bucket (kết quả cuối)
```

### Vai Trò Của Lambda Connector

Lambda connector thực hiện hai nhiệm vụ chính:

```
1. GetSplits (Lấy danh sách splits):
   Connector phân chia data source thành các "splits" 
   → Mỗi split là một phần dữ liệu đọc song song
   
   Ví dụ MySQL: mỗi shard của table = 1 split
   Ví dụ DynamoDB: mỗi partition = 1 split
   Ví dụ S3: mỗi file = 1 split

2. ReadWithConstraint (Đọc dữ liệu có filter):
   Đẩy predicates xuống data source (pushdown)
   → MySQL: thêm WHERE vào SQL gửi xuống MySQL
   → DynamoDB: dùng FilterExpression
   → Giảm lượng data truyền về Athena
```

### Spill Location (Vị Trí Tràn Bộ Nhớ)

```
Khi Lambda không đủ bộ nhớ để chứa partial results:
Lambda → Ghi tạm ra S3 (spill bucket)
Athena → Đọc từ S3 spill bucket

→ Cần cấu hình spill bucket khi deploy connector
```

---

## 3. Pre-built Connectors Có Sẵn

AWS cung cấp sẵn các connector trên **AWS Serverless Application Repository — SAR** (Kho Ứng Dụng Serverless AWS):

| Connector | Data Source | Package SAR |
|-----------|-------------|-------------|
| **AthenaMySQLConnector** | MySQL, Amazon Aurora MySQL | `AthenaJdbcConnector` |
| **AthenaPostgreSQLConnector** | PostgreSQL, Aurora PostgreSQL | `AthenaJdbcConnector` |
| **AthenaDynamoDBConnector** | Amazon DynamoDB | `AthenaDynamoDBConnector` |
| **AthenaRedisConnector** | Redis, Amazon ElastiCache | `AthenaRedisConnector` |
| **AthenaHBaseConnector** | Apache HBase trên EMR | `AthenaHBaseConnector` |
| **AthenaDocumentDBConnector** | Amazon DocumentDB | `AthenaDocumentDBConnector` |
| **AthenaCloudwatchConnector** | CloudWatch Logs | `AthenaCloudwatchConnector` |
| **AthenaCloudwatchMetricsConnector** | CloudWatch Metrics | `AthenaCloudwatchMetricsConnector` |
| **AthenaTDSConnector** | Microsoft SQL Server | `AthenaJdbcConnector` |
| **AthenaOracleConnector** | Oracle Database | `AthenaJdbcConnector` |
| **AthenaKafkaConnector** | Apache Kafka / MSK | `AthenaKafkaConnector` |
| **AthenaOpenSearchConnector** | Amazon OpenSearch | `AthenaOpenSearchConnector` |

---

## 4. Thiết Lập Connector — Ví Dụ MySQL/RDS

### Bước 1: Deploy Lambda Connector Từ SAR

```bash
# Deploy qua AWS CLI
aws serverlessrepo create-cloud-formation-template \
  --application-id arn:aws:serverlessrepo:us-east-1:292517598671:applications/AthenaMySQLConnector \
  --semantic-version 2023.11.1

# Hoặc deploy qua Console:
# AWS Serverless Application Repository → Search "AthenaMySQLConnector" → Deploy
```

### Bước 2: Cấu Hình Lambda Environment Variables

```
Lambda Function: AthenaMySQLConnector

Environment Variables:
┌────────────────────────────────────────────────────────────┐
│ default              │ mysql://hostname:3306/mydb           │
│                      │ (connection string mặc định)         │
├────────────────────────────────────────────────────────────┤
│ default_connection_string │ ${ssm:/athena/mysql/conn}       │
│                      │ (đọc từ SSM Parameter Store)         │
├────────────────────────────────────────────────────────────┤
│ spill_bucket         │ my-athena-spill-bucket               │
│                      │ (bucket để ghi khi tràn bộ nhớ)     │
├────────────────────────────────────────────────────────────┤
│ spill_prefix         │ athena-spill/mysql/                  │
└────────────────────────────────────────────────────────────┘
```

### Bước 3: Lưu Connection String Vào SSM Parameter Store

```bash
# Lưu connection string an toàn trong SSM Parameter Store
# (không để lộ password trong Lambda config)
aws ssm put-parameter \
  --name "/athena/mysql/production" \
  --value "mysql://username:password@rds-endpoint:3306/database" \
  --type "SecureString" \
  --key-id "alias/aws/ssm"
```

### Bước 4: Đăng Ký Data Source Trong Athena

```bash
# Tạo Data Catalog trong Athena trỏ vào Lambda connector
aws athena create-data-catalog \
  --name "mysql-production" \
  --type "LAMBDA" \
  --parameters '{"function":"arn:aws:lambda:ap-southeast-1:123456789:function:AthenaMySQLConnector"}' \
  --description "Production MySQL RDS"
```

---

## 5. Thiết Lập DynamoDB Connector

### Deploy và Cấu Hình

```bash
# Deploy AthenaDynamoDBConnector từ SAR
aws cloudformation create-stack \
  --stack-name AthenaDynamoDBConnector \
  --template-url https://... \
  --parameters \
    ParameterKey=AthenaCatalogName,ParameterValue=dynamo \
    ParameterKey=SpillBucket,ParameterValue=my-athena-spill \
    ParameterKey=SpillPrefix,ParameterValue=athena-spill/dynamo

# Đăng ký catalog
aws athena create-data-catalog \
  --name "dynamo" \
  --type "LAMBDA" \
  --parameters '{"function":"arn:aws:lambda:ap-southeast-1:123456789:function:AthenaDynamoDBConnector"}'
```

### Lưu Ý Với DynamoDB

```
DynamoDB không có schema cố định → Connector cần inferSchema:
- Connector scan mẫu items để suy luận schema
- Nên khai báo schema tường minh trong Glue Catalog để tránh scan

DynamoDB không hỗ trợ SQL predicates đầy đủ:
- Chỉ partition key và sort key có thể pushdown hiệu quả
- Các filter khác phải scan toàn bộ table → tốn read capacity
- Cẩn thận với RCU (Read Capacity Units) cost!
```

---

## 6. Viết Federated Query

### Cú Pháp Data Catalog Trong Query

```sql
-- Cú pháp: catalog_name.database_name.table_name

-- Query bảng trong MySQL (qua Lambda connector)
SELECT * FROM "mysql-production"."ecommerce"."orders" LIMIT 10;

-- Query bảng trong DynamoDB
SELECT * FROM "dynamo"."default"."user_sessions" LIMIT 10;

-- Query S3 (default catalog)
SELECT * FROM "AwsDataCatalog"."sales_db"."events" LIMIT 10;
```

### Federated Join — Join Đa Nguồn

```sql
-- JOIN giữa MySQL và S3 Parquet
SELECT
    o.order_id,
    o.customer_id,
    o.amount,
    o.order_date,
    e.event_type,
    e.page_url,
    e.session_duration_seconds
FROM "mysql-production"."ecommerce"."orders" o
JOIN "AwsDataCatalog"."analytics"."user_events" e
    ON o.customer_id = e.user_id
    AND o.session_id = e.session_id
WHERE
    o.order_date >= DATE '2024-01-01'
    AND e.dt >= '2024-01-01'    -- partition filter cho S3
ORDER BY o.amount DESC
LIMIT 100;
```

### Query Multi-Source Với CTE

```sql
-- Kết hợp 3 nguồn: MySQL (RDS), DynamoDB, và S3
WITH
-- Đơn hàng từ MySQL
orders AS (
    SELECT order_id, customer_id, amount, order_date
    FROM "mysql-production"."ecommerce"."orders"
    WHERE order_date >= DATE '2024-01-01'
),

-- Session info từ DynamoDB
sessions AS (
    SELECT session_id, user_id, duration_seconds, device_type
    FROM "dynamo"."default"."user_sessions"
),

-- Hành vi người dùng từ S3
events AS (
    SELECT user_id, session_id, COUNT(*) AS page_views
    FROM "AwsDataCatalog"."analytics"."page_views"
    WHERE dt >= '2024-01-01'
    GROUP BY user_id, session_id
)

-- Kết hợp tất cả
SELECT
    o.order_id,
    o.amount,
    s.duration_seconds AS session_duration,
    s.device_type,
    e.page_views AS pages_before_purchase
FROM orders o
JOIN sessions s ON o.customer_id = s.user_id
JOIN events e   ON o.customer_id = e.user_id
                AND s.session_id = e.session_id
ORDER BY o.amount DESC;
```

### Query CloudWatch Logs

```sql
-- Tìm lỗi trong Lambda logs
SELECT
    log_stream,
    timestamp,
    message
FROM "cloudwatch"."default"."/aws/lambda/my-function"
WHERE message LIKE '%ERROR%'
  AND timestamp >= 1704067200000  -- Unix timestamp milliseconds
ORDER BY timestamp DESC
LIMIT 50;
```

---

## 7. Performance Considerations

### Predicate Pushdown (Đẩy Điều Kiện Xuống Data Source)

```
Mục tiêu: Lọc dữ liệu tại nguồn, không phải sau khi đã kéo về Athena

MySQL connector:
→ WHERE clauses được chuyển thành SQL WHERE trong query đến MySQL
→ Hiệu quả với các index MySQL

DynamoDB connector:
→ Chỉ partition key / sort key conditions được pushdown
→ Các filter khác: Lambda fetch rồi filter → tốn RCU

Redis connector:
→ Chỉ key lookups được pushdown
→ Scan toàn bộ: không pushdown → rất chậm với Redis lớn
```

### Lambda Memory và Timeout

```
Cấu hình Lambda connector:
- Memory: 512 MB đến 3 GB (tuỳ data source và query complexity)
- Timeout: 15 phút (Lambda maximum)
- Concurrency: Mặc định 5 concurrent executions per connector

Lưu ý:
- Nếu Athena query timeout 30 phút, Lambda có thể bị invoke nhiều lần
- Tăng Lambda memory = tăng tốc độ + tăng chi phí
- Với large tables, cần tăng memory và tuning splits
```

### Estimated Cost Model

```
Chi phí Federated Query = Chi phí Athena + Chi phí Lambda + Chi phí Data Source

Athena: $5/TB quét (kể cả data từ external sources)
Lambda: ~$0.0000166667/GB-second execution
Data Source: MySQL read I/O, DynamoDB RCU, Redis read...

Ví dụ: Query join 1 GB MySQL data + 100 GB S3 Parquet
Athena cost: 101 GB / 1024 ≈ 0.1 TB × $5 = $0.49
Lambda cost: ~$0.01 (nhỏ, thường bỏ qua)
MySQL: ít ảnh hưởng (read I/O của RDS)
Total: ~$0.50
```

---

## 8. Bảo Mật Federated Query

### IAM Permissions Cho Lambda Connector

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-athena-spill-bucket",
        "arn:aws:s3:::my-athena-spill-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter"
      ],
      "Resource": "arn:aws:ssm:*:*:parameter/athena/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:athena-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "*"
    }
  ]
}
```

### Mã Hóa Spill Data

```
Khi Lambda spill dữ liệu ra S3:
- Dữ liệu có thể chứa sensitive information
- Nên enable SSE-KMS encryption cho spill bucket
- Đảm bảo chỉ Lambda function có quyền đọc spill data
```

### Network — VPC Integration

```
Nếu data source nằm trong VPC (RDS trong private subnet):
1. Lambda connector phải deploy trong cùng VPC
2. Cần Security Group cho phép Lambda kết nối đến RDS port
3. NAT Gateway hoặc VPC Endpoint cho S3 spill bucket

Architecture:
┌─────────────────────────────────────┐
│              VPC                     │
│  ┌──────────────┐  ┌──────────────┐ │
│  │   Lambda     │  │   RDS MySQL  │ │
│  │  Connector   │──│  (private    │ │
│  │  (trong VPC) │  │   subnet)    │ │
│  └──────┬───────┘  └──────────────┘ │
│         │                            │
│  ┌──────▼────────────────────────┐  │
│  │  VPC Endpoint for S3          │  │
│  │  (spill bucket không qua NAT) │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

---

## 9. Custom Connector — Tự Viết Connector

Khi không có connector có sẵn, bạn có thể viết connector bằng **Athena Query Federation SDK** (Java).

### Cấu Trúc Custom Connector

```java
// Maven dependency
// com.amazonaws:aws-athena-federation-sdk:2023.11.1

public class MyCustomConnector extends CompositeHandler {

    public MyCustomConnector() {
        super(new MyMetadataHandler(), new MyRecordHandler());
    }
}

// MetadataHandler: trả về schema và splits
public class MyMetadataHandler extends GlueMetadataHandler {

    @Override
    public GetTableResponse doGetTable(
            BlockAllocator allocator,
            GetTableRequest request) {
        // Trả về schema của table
        Schema schema = SchemaBuilder.newBuilder()
            .addField("id", Types.MinorType.INT.getType())
            .addField("name", Types.MinorType.VARCHAR.getType())
            .build();
        
        return new GetTableResponse(
            request.getCatalogName(),
            request.getTableName(),
            schema
        );
    }

    @Override
    public GetSplitsResponse doGetSplits(
            BlockAllocator allocator,
            GetSplitsRequest request) {
        // Chia data thành splits để đọc song song
        Set<Split> splits = new HashSet<>();
        splits.add(Split.newBuilder(makeSpillLocation(request), makeEncryptionKey())
            .add("shard_id", "0")
            .build());
        return new GetSplitsResponse(request.getCatalogName(), splits);
    }
}

// RecordHandler: đọc dữ liệu thực tế
public class MyRecordHandler extends RecordHandler {

    @Override
    protected void readWithConstraint(
            BlockSpiller spiller,
            ReadRecordsRequest request,
            QueryStatusChecker queryStatusChecker) {
        // Đọc data từ source với filters
        // Ghi vào spiller
        spiller.writeRows((Block block, int rowNum) -> {
            block.setValue("id", rowNum, 1);
            block.setValue("name", rowNum, "example");
            return 1; // 1 row written
        });
    }
}
```

---

## 10. Use Cases và Anti-patterns

### Use Cases Phù Hợp ✅

```
1. Real-time data enrichment (Làm Giàu Dữ Liệu Thời Gian Thực):
   - Join S3 historical data với RDS current data
   - Không cần ETL pipeline riêng

2. Operational analytics (Phân Tích Vận Hành):
   - Query production database cho báo cáo tần suất thấp
   - Không muốn copy data vào data warehouse chỉ để chạy ad-hoc query

3. Data exploration (Khám Phá Dữ Liệu):
   - Analyst muốn xem dữ liệu từ nhiều hệ thống mà không cần
     chờ ETL setup

4. CloudWatch log analysis (Phân Tích Log):
   - Debug production issues bằng cách join logs với business data

5. Multi-account analytics (Phân Tích Đa Tài Khoản):
   - Kết hợp data từ nhiều AWS accounts thông qua cross-account Lambda
```

### Anti-patterns Cần Tránh ❌

```
1. High-frequency queries trên production database:
   ❌ Federated query đến RDS chạy 1000 lần/ngày
   → Tạo read pressure lên production DB
   → Dùng Read Replica hoặc copy data sang S3 trước

2. Large table full scan qua Lambda:
   ❌ SELECT * FROM dynamo.sessions (100 triệu records)
   → Tiêu tốn DynamoDB RCU + Lambda execution time
   → Dùng DynamoDB Export sang S3 rồi query Athena trên S3

3. Thay thế ETL hoàn toàn:
   ❌ Chạy federated query mỗi giờ thay vì ETL pipeline
   → Tốn kém hơn ETL batch
   → Tạo coupling chặt giữa data consumers và sources
   → Dùng Federated Query cho ad-hoc, dùng ETL cho scheduled reports

4. OLTP-style queries:
   ❌ SELECT * FROM rds.orders WHERE order_id = '12345' (row lookup)
   → Chi phí Athena minimum là 10 MB — lãng phí cho point lookup
   → Dùng trực tiếp RDS hoặc DynamoDB cho OLTP lookups
```

---

## 🔑 Tóm Tắt Key Points

```
1. Federated Query = SQL duy nhất → join đa nguồn dữ liệu
2. Lambda connector là cầu nối giữa Athena và data source ngoài S3
3. Predicate pushdown quan trọng — filter tại source để giảm data transfer
4. Spill bucket cần được cấu hình và bảo mật đúng cách
5. Không phải mọi filter đều được pushdown — DynamoDB chỉ pushdown partition/sort key
6. Dùng SSM Parameter Store cho connection strings, không hardcode trong Lambda
7. Chỉ dùng Federated Query cho ad-hoc queries, không phải high-frequency scheduled jobs
8. VPC integration cần thiết khi data source trong private network
```

---

**Tiếp Theo:** [4-athena-cost-optimization.md](./4-athena-cost-optimization.md) — Tối ưu chi phí: workgroup budgets, query controls, và các kỹ thuật giảm chi phí nâng cao
