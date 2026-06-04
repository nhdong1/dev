# ✅ AWS Glue Data Quality — Chất Lượng Dữ Liệu Tự Động

> Glue Data Quality tự động phân tích (profile), kiểm tra (validate) và theo dõi (monitor) chất lượng dữ liệu trong data lake — phát hiện sớm dữ liệu bẩn trước khi ảnh hưởng đến analytics và báo cáo.

## 📚 Mục Lục

1. [Data Quality Là Gì](#data-quality-là-gì)
2. [Các Thành Phần Chính](#các-thành-phần-chính)
3. [Data Quality Rules](#data-quality-rules)
4. [Data Profiling](#data-profiling)
5. [Tích Hợp Vào ETL Pipeline](#tích-hợp-vào-etl-pipeline)
6. [Monitoring và Alerting](#monitoring-và-alerting)
7. [DQDL — Ngôn Ngữ Định Nghĩa Rules](#dqdl--ngôn-ngữ-định-nghĩa-rules)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Data Quality Là Gì

**Data Quality (Chất Lượng Dữ Liệu)** là mức độ dữ liệu phù hợp với mục đích sử dụng. Dữ liệu chất lượng cao là dữ liệu:

```
6 Chiều Đo Chất Lượng Dữ Liệu:
─────────────────────────────────────────────────────────────
1. COMPLETENESS (Đầy Đủ):     Không có giá trị null quan trọng
2. ACCURACY (Chính Xác):      Giá trị đúng thực tế (age = 150 là sai)
3. CONSISTENCY (Nhất Quán):   Không mâu thuẫn giữa các bảng
4. TIMELINESS (Kịp Thời):     Dữ liệu mới nhất, không stale
5. UNIQUENESS (Duy Nhất):     Không có duplicate records
6. VALIDITY (Hợp Lệ):         Đúng format, đúng range (email, phone, date)
```

### Tại Sao Data Quality Quan Trọng?

```
Dữ liệu xấu → Phân tích sai → Quyết định kinh doanh sai

Ví dụ thực tế:
  ❌ 30% orders thiếu user_id → báo cáo conversion rate sai
  ❌ Giá sản phẩm = -100 (âm) → doanh thu tính toán sai
  ❌ Ngày đặt hàng năm 1970 (Unix epoch default) → phân tích theo thời gian sai
  ❌ Duplicate transaction IDs → đếm đôi revenue

Phát hiện sớm (tại pipeline) >> Phát hiện muộn (từ báo cáo sai)
```

### Glue Data Quality Giải Quyết Gì?

| Bài Toán                                  | Giải Pháp                                  |
| ----------------------------------------- | ------------------------------------------ |
| Không biết dữ liệu có vấn đề gì          | Data Profiling tự động thống kê            |
| Cần kiểm tra dữ liệu trước khi load      | Rules validation trong ETL pipeline        |
| Muốn theo dõi chất lượng theo thời gian  | Monitoring metrics với CloudWatch          |
| Không biết viết rules kiểm tra gì        | Rule Recommendations tự động gợi ý         |

---

## 🗂️ Các Thành Phần Chính

```
AWS Glue Data Quality
│
├── 1. Data Quality Ruleset (Tập Quy Tắc Chất Lượng)
│      └── Định nghĩa rules bằng DQDL (Data Quality Definition Language)
│
├── 2. Data Quality Evaluation (Đánh Giá Chất Lượng)
│      └── Chạy rules trên data, xuất kết quả PASS/FAIL
│
├── 3. Rule Recommendations (Gợi Ý Quy Tắc)
│      └── Glue phân tích data và tự đề xuất rules phù hợp
│
└── 4. Data Quality Results (Kết Quả Đánh Giá)
       └── Lưu vào S3, hiển thị trong console, gửi alerts
```

---

## 📋 Data Quality Rules (Quy Tắc Chất Lượng Dữ Liệu)

### DQDL — Data Quality Definition Language (Ngôn Ngữ Định Nghĩa Chất Lượng Dữ Liệu)

DQDL là ngôn ngữ declarative (khai báo) để viết data quality rules. Cú pháp đơn giản, dễ đọc:

```
Rules = [
    <RuleType> <Columns> <Condition> [with threshold <value>]
]
```

### Các Rule Types (Loại Quy Tắc) Phổ Biến

#### Completeness Rules (Kiểm Tra Đầy Đủ)

```
Rules = [
    # Cột order_id không được có null
    IsComplete "order_id",

    # Cột email không được có null
    IsComplete "email",

    # Tỷ lệ null của user_id phải < 5%
    Completeness "user_id" >= 0.95,

    # Tỷ lệ null của discount_code có thể lên đến 70% (cột tùy chọn)
    Completeness "discount_code" >= 0.30
]
```

#### Uniqueness Rules (Kiểm Tra Duy Nhất)

```
Rules = [
    # order_id phải unique hoàn toàn
    IsPrimaryKey "order_id",

    # Tỷ lệ unique của transaction_id phải >= 99%
    Uniqueness "transaction_id" >= 0.99,

    # Không cho phép duplicate
    IsUnique "session_id"
]
```

#### Validity Rules (Kiểm Tra Hợp Lệ)

```
Rules = [
    # Tuổi phải từ 0 đến 120
    ColumnValues "age" between 0 and 120,

    # Giá phải > 0
    ColumnValues "price" > 0,

    # Status chỉ được là một trong các giá trị cho phép
    ColumnValues "status" in ["pending", "processing", "completed", "cancelled"],

    # Email phải đúng format (regex)
    ColumnValues "email" matches "^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$",

    # Ngày phải sau 2020-01-01
    ColumnValues "created_at" > "2020-01-01",

    # Phone phải đủ 10 chữ số
    ColumnLength "phone_number" = 10
]
```

#### Statistical Rules (Kiểm Tra Thống Kê)

```
Rules = [
    # Giá trung bình phải trong khoảng [50, 5000]
    Mean "order_amount" between 50 and 5000,

    # Giá trị lớn nhất phải < 100,000
    Max "order_amount" < 100000,

    # Độ lệch chuẩn (standard deviation) không quá lớn
    StandardDeviation "response_time_ms" <= 500,

    # Tỷ lệ phần trăm thứ 99 (P99 latency) phải < 2000ms
    ColumnValues "response_time_ms" <= 2000 with threshold >= 0.99
]
```

#### Referential Integrity Rules (Kiểm Tra Tính Toàn Vẹn Tham Chiếu)

```
Rules = [
    # user_id trong bảng orders phải tồn tại trong bảng users
    ReferentialIntegrity "orders.user_id" "users.user_id" >= 0.99,

    # product_id phải tồn tại trong catalog
    ReferentialIntegrity "order_items.product_id" "products.product_id" = 1.0
]
```

#### Row Count Rules (Kiểm Tra Số Lượng Records)

```
Rules = [
    # Bảng phải có ít nhất 1000 records (tránh empty table)
    RowCount >= 1000,

    # Bảng không được có quá 100 triệu records (sanity check)
    RowCount <= 100000000,

    # Số records phải trong khoảng dự kiến (daily batch thường ~50K-200K)
    RowCount between 50000 and 200000
]
```

---

## 📊 Data Profiling (Phân Tích Hồ Sơ Dữ Liệu)

**Data Profiling** tự động tính toán thống kê về từng cột — giúp hiểu dữ liệu trước khi viết rules.

### Metrics Được Tính Tự Động

```
Cho mỗi cột, Glue Profiling tính:

Numeric columns (Cột Số):
  ├── Min, Max (Giá trị nhỏ nhất, lớn nhất)
  ├── Mean (Trung Bình), Median (Trung Vị)
  ├── Standard Deviation (Độ Lệch Chuẩn)
  ├── Percentiles: P25, P50, P75, P90, P95, P99
  ├── Null Count, Null Percentage (Tỷ Lệ Null)
  └── Unique Count (Số Giá Trị Duy Nhất)

String columns (Cột Chuỗi):
  ├── Min Length, Max Length, Mean Length
  ├── Null Count, Null Percentage
  ├── Unique Count
  ├── Top 10 most frequent values (10 giá trị xuất hiện nhiều nhất)
  └── Sample values (Giá Trị Mẫu)

Boolean columns (Cột Boolean):
  ├── True Count, False Count
  └── Null Count
```

### Chạy Profiling

```python
# Tạo Data Quality ruleset với profiling
import boto3

glue = boto3.client('glue', region_name='ap-southeast-1')

# Tạo ruleset
glue.create_data_quality_ruleset(
    Name='orders-quality-rules',
    Description='Rules kiểm tra chất lượng bảng orders',
    Ruleset='''
    Rules = [
        IsComplete "order_id",
        IsComplete "user_id",
        IsUnique "order_id",
        ColumnValues "amount" > 0,
        ColumnValues "status" in ["pending", "completed", "cancelled"],
        RowCount >= 1000,
        Completeness "email" >= 0.90
    ]
    ''',
    TargetTable={
        'TableName': 'orders_processed',
        'DatabaseName': 'analytics_processed'
    }
)

# Bắt đầu đánh giá
response = glue.start_data_quality_ruleset_evaluation_run(
    DataSource={
        'GlueTable': {
            'DatabaseName': 'analytics_processed',
            'TableName': 'orders_processed'
        }
    },
    Role='arn:aws:iam::123456789:role/GlueDataQualityRole',
    RulesetNames=['orders-quality-rules'],
    RunConfiguration={
        'IcebergTargetCompressionType': 'SNAPPY',
        'NumberOfWorkers': 5,
        'WorkerType': 'G.1X'
    }
)

run_id = response['RunId']
print(f"Đã bắt đầu evaluation run: {run_id}")
```

### Lấy Kết Quả Profiling

```python
# Lấy kết quả sau khi run xong
result = glue.get_data_quality_ruleset_evaluation_run(RunId=run_id)
print(f"Trạng thái: {result['Status']}")

# Xem kết quả từng rule
for rule_result in result['RuleResults']:
    status = "✅ PASS" if rule_result['Result'] == 'PASS' else "❌ FAIL"
    print(f"{status} | Rule: {rule_result['Name']} | {rule_result.get('EvaluationMessage', '')}")
```

---

## 🔗 Tích Hợp Vào ETL Pipeline

### Pattern 1: Validate Before Load (Kiểm Tra Trước Khi Tải)

```python
# Trong Glue ETL Job — validate data quality trước khi load vào Redshift
import sys
from awsglue.context import GlueContext
from awsglue.transforms import EvaluateDataQuality
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)

args = getResolvedOptions(sys.argv, ['JOB_NAME'])

# Đọc dữ liệu cần load
source_dyf = glueContext.create_dynamic_frame.from_catalog(
    database="analytics_raw",
    table_name="orders_raw"
)

# Định nghĩa data quality rules inline
dq_ruleset = """
Rules = [
    IsComplete "order_id",
    IsComplete "user_id",
    IsUnique "order_id",
    ColumnValues "amount" > 0,
    RowCount >= 100
]
"""

# Chạy Data Quality evaluation
dq_results = EvaluateDataQuality().process_rows(
    frame=source_dyf,
    ruleset=dq_ruleset,
    publishing_options={
        "dataQualityEvaluationContext": "orders-validation",
        "enableDataQualityResultsPublishing": True,
        "resultsS3Prefix": "s3://my-bucket/dq-results/orders/"
    },
    additional_options={
        "observations.scope": "ALL",      # Ghi nhận tất cả quan sát
        "performanceTuning.caching": "CACHE_NOTHING"
    }
)

# Tách records đạt / không đạt
passed_records = dq_results.select_from_expression(
    "dataQualityEvaluationResult = 'Passed'"
)
failed_records = dq_results.select_from_expression(
    "dataQualityEvaluationResult != 'Passed'"
)

print(f"Records đạt chất lượng: {passed_records.count()}")
print(f"Records KHÔNG đạt: {failed_records.count()}")

# Chỉ load records đạt vào Redshift
glueContext.write_dynamic_frame.from_jdbc_conf(
    frame=passed_records,
    catalog_connection="redshift-connection",
    connection_options={
        "dbtable": "analytics.orders",
        "database": "data_warehouse"
    }
)

# Ghi records lỗi vào quarantine zone (vùng cách ly)
glueContext.write_dynamic_frame.from_options(
    frame=failed_records,
    connection_type="s3",
    connection_options={
        "path": "s3://my-bucket/quarantine/orders/",
        "partitionKeys": ["year", "month", "day"]
    },
    format="parquet"
)
```

### Pattern 2: Quarantine Pattern (Mẫu Cách Ly)

```
┌─────────────────────────────────────────────────────────┐
│                   ETL Pipeline                           │
│                                                          │
│  [Raw Data S3]                                           │
│       │                                                  │
│       ▼                                                  │
│  [Data Quality Check]                                    │
│       │                                                  │
│       ├── PASS ──► [Processed Data S3] ──► [Redshift]   │
│       │                                                  │
│       └── FAIL ──► [Quarantine Zone S3]                  │
│                         │                                │
│                         ▼                                │
│               [Alert: SNS/Slack]                         │
│               [Manual Review Queue]                      │
└─────────────────────────────────────────────────────────┘
```

### Pattern 3: Pipeline Halt (Dừng Pipeline Khi Quality Thấp)

```python
# Dừng toàn bộ pipeline nếu chất lượng dưới ngưỡng
def check_data_quality_threshold(evaluation_run_id, min_pass_rate=0.90):
    """
    Kiểm tra tỷ lệ rules PASS.
    Nếu < min_pass_rate, raise exception để dừng job.
    """
    result = glue.get_data_quality_ruleset_evaluation_run(
        RunId=evaluation_run_id
    )

    total_rules = len(result['RuleResults'])
    passed_rules = sum(
        1 for r in result['RuleResults']
        if r['Result'] == 'PASS'
    )
    pass_rate = passed_rules / total_rules

    print(f"Data Quality: {passed_rules}/{total_rules} rules passed ({pass_rate:.1%})")

    if pass_rate < min_pass_rate:
        failed_rules = [
            r['Name'] for r in result['RuleResults']
            if r['Result'] != 'PASS'
        ]
        raise Exception(
            f"Data Quality không đạt ngưỡng {min_pass_rate:.0%}! "
            f"Rules thất bại: {', '.join(failed_rules)}"
        )

    print(f"✅ Data Quality đạt ngưỡng {min_pass_rate:.0%}")
```

---

## 📡 Monitoring và Alerting

### CloudWatch Integration (Tích Hợp CloudWatch)

```python
# Glue Data Quality tự động gửi metrics lên CloudWatch
# Tên metrics:
#   glue.dataQuality.rulesSucceeded    → Số rules PASS
#   glue.dataQuality.rulesFailed       → Số rules FAIL
#   glue.dataQuality.rulesSkipped      → Số rules bị bỏ qua

# Tạo CloudWatch Alarm khi có rule fail
import boto3

cloudwatch = boto3.client('cloudwatch', region_name='ap-southeast-1')

cloudwatch.put_metric_alarm(
    AlarmName='GlueDataQuality-OrdersTable-RulesFailed',
    AlarmDescription='Cảnh báo khi có Data Quality rules thất bại trong bảng orders',
    MetricName='glue.dataQuality.rulesFailed',
    Namespace='Glue',
    Statistic='Sum',
    Period=3600,             # 1 giờ
    EvaluationPeriods=1,
    Threshold=0,             # Cảnh báo ngay khi có 1 rule fail
    ComparisonOperator='GreaterThanThreshold',
    Dimensions=[
        {'Name': 'JobName', 'Value': 'etl-orders-job'}
    ],
    AlarmActions=[
        'arn:aws:sns:ap-southeast-1:123456789:data-quality-alerts'
    ],
    TreatMissingData='breaching'  # Coi thiếu data như vi phạm
)
```

### Gửi Alert Qua SNS → Slack/Email

```python
# Lambda function xử lý SNS notification từ Data Quality failure
import boto3
import json

sns = boto3.client('sns')

def lambda_handler(event, context):
    # Parse Glue Data Quality event
    detail = event['detail']
    job_name = detail.get('jobName', 'Unknown')
    failed_rules = detail.get('failedRules', [])

    # Format message
    message = f"""
🚨 Data Quality Alert!
Job: {job_name}
Rules Thất Bại:
{chr(10).join(f'  ❌ {r}' for r in failed_rules)}
Xem chi tiết: AWS Glue Console → Data Quality → Evaluation Runs
"""

    # Gửi SNS notification
    sns.publish(
        TopicArn='arn:aws:sns:ap-southeast-1:123456789:data-quality-alerts',
        Subject=f'[DATA QUALITY FAIL] {job_name}',
        Message=message
    )

    return {'statusCode': 200}
```

### Data Quality Dashboard (Bảng Theo Dõi)

```
Metrics nên theo dõi hàng ngày:
─────────────────────────────────────────────────────
Bảng orders:
  ✅ Completeness(order_id):   100% → 100% → 99.8% (⚠️ giảm)
  ✅ Uniqueness(order_id):     100% → 100% → 100%
  ✅ ColumnValues(amount > 0): 100% → 99.9% → 100%
  📊 RowCount:                 52,341 → 48,219 → 67,854

Bảng users:
  ✅ Completeness(email):      95.2% → 94.8% → 96.1%
  ⚠️ Uniqueness(email):       99.1% → 99.3% → 98.7% (⬇️ xuống dưới 99%)
─────────────────────────────────────────────────────
→ Phát hiện: email uniqueness giảm → có thể có bug duplicate user registration
```

---

## 📝 DQDL — Ngôn Ngữ Định Nghĩa Rules

### Tổng Hợp Cú Pháp DQDL

```
# Cú pháp cơ bản
Rules = [
    <RuleFunction> [<Column>] [<Condition>] [with threshold <fraction>]
]

# with threshold — tỷ lệ records phải thỏa mãn (0.0 đến 1.0)
ColumnValues "price" > 0 with threshold >= 0.99
→ Ít nhất 99% records phải có price > 0
```

### Bảng Tổng Hợp Rules

| Rule Function           | Mô Tả Tiếng Việt                                       | Ví Dụ                                              |
| ----------------------- | ------------------------------------------------------- | --------------------------------------------------- |
| `IsComplete`            | Cột không có null                                       | `IsComplete "user_id"`                             |
| `Completeness`          | Tỷ lệ non-null phải >= threshold                        | `Completeness "email" >= 0.95`                     |
| `IsUnique`              | Mọi giá trị phải unique                                 | `IsUnique "order_id"`                              |
| `Uniqueness`            | Tỷ lệ unique phải >= threshold                          | `Uniqueness "email" >= 0.99`                       |
| `IsPrimaryKey`          | Cột là primary key (không null, unique)                 | `IsPrimaryKey "id"`                                |
| `ColumnValues`          | Giá trị phải thỏa mãn điều kiện                         | `ColumnValues "age" between 0 and 120`             |
| `ColumnLength`          | Độ dài chuỗi phải thỏa mãn điều kiện                   | `ColumnLength "zip_code" = 5`                      |
| `ColumnCorrelation`     | Tương quan giữa hai cột phải thỏa mãn                  | `ColumnCorrelation "price" "cost" >= 0.8`          |
| `Mean`                  | Giá trị trung bình phải thỏa mãn                        | `Mean "price" between 10 and 1000`                 |
| `Sum`                   | Tổng phải thỏa mãn                                      | `Sum "quantity" > 0`                               |
| `Min`                   | Giá trị nhỏ nhất phải thỏa mãn                          | `Min "price" > 0`                                  |
| `Max`                   | Giá trị lớn nhất phải thỏa mãn                          | `Max "age" <= 150`                                 |
| `StandardDeviation`     | Độ lệch chuẩn phải thỏa mãn                             | `StandardDeviation "price" <= 100`                 |
| `RowCount`              | Số lượng records phải thỏa mãn                          | `RowCount between 1000 and 1000000`                |
| `RowCountMatch`         | Số records phải khớp với bảng khác                     | `RowCountMatch "other_table" >= 0.95`              |
| `ReferentialIntegrity`  | Giá trị cột phải tồn tại trong bảng tham chiếu          | `ReferentialIntegrity "col1" "table2.col2" >= 1.0` |
| `DataFreshness`         | Dữ liệu phải đủ mới (không stale)                      | `DataFreshness "updated_at" <= 3 hours`            |
| `DetectAnomalies`       | Phát hiện bất thường tự động (ML-based)                 | `DetectAnomalies "daily_revenue"`                  |

### Rule Recommendations (Gợi Ý Quy Tắc Tự Động)

```python
# Glue tự phân tích data và đề xuất rules phù hợp
response = glue.get_data_quality_rule_recommendation_run(
    RunId='recommendation-run-id'
)

recommended_ruleset = response['RecommendedRuleset']
print("Glue đề xuất rules sau:")
print(recommended_ruleset)

# Output ví dụ:
# Rules = [
#     IsComplete "order_id",
#     IsPrimaryKey "order_id",
#     Completeness "user_id" >= 0.9984,
#     ColumnValues "status" in ["pending", "completed", "cancelled", "refunded"],
#     ColumnValues "amount" between 0.99 and 99999.99,
#     RowCount between 45000 and 75000
# ]
```

---

## ✅ Best Practices (Thực Hành Tốt Nhất)

### 1. Phân Tầng Data Quality

```
Tier 1 — Critical Rules (Quy Tắc Quan Trọng):
  Fail → DỪNG pipeline ngay lập tức
  Ví dụ: IsComplete "order_id", IsUnique "transaction_id"

Tier 2 — Warning Rules (Quy Tắc Cảnh Báo):
  Fail → LOG + Alert nhưng KHÔNG dừng pipeline
  Ví dụ: Completeness "email" >= 0.90

Tier 3 — Informational Rules (Quy Tắc Thông Tin):
  Fail → Chỉ ghi vào dashboard, không alert
  Ví dụ: Mean "price" between 50 and 500 (để theo dõi xu hướng)
```

### 2. Viết Rules Từ Yêu Cầu Kinh Doanh

```
❌ TRÁNH: Viết rules kỹ thuật chung chung không có ý nghĩa
  ColumnValues "field_x" > 0  (field_x là gì? range đúng không?)

✅ NÊN: Viết rules phản ánh yêu cầu kinh doanh
  # Đơn hàng phải có giá trị dương (không thể hoàn tiền âm trong hệ thống này)
  ColumnValues "order_amount" > 0
  # Chỉ 5 trạng thái hợp lệ trong business workflow
  ColumnValues "order_status" in ["new", "paid", "shipped", "delivered", "cancelled"]
```

### 3. Baseline Từ Historical Data

```
Nguyên tắc: Đặt threshold dựa trên historical data thực tế, cộng buffer hợp lý

Cách làm:
  1. Chạy profiling trên 30 ngày data gần nhất
  2. Xem Completeness(email) = 94.5% average
  3. Đặt rule: Completeness "email" >= 0.90  (buffer 4.5%)
  4. Monitor: Nếu metric thường xuyên sát 90%, hạ threshold hoặc fix data source

Không đặt threshold 100% cho mọi thứ:
  ❌ Completeness "email" >= 1.0
     (Không thực tế — user có thể không cung cấp email)
  ✅ Completeness "email" >= 0.90
     (Dựa trên lịch sử, chấp nhận 10% thiếu)
```

### 4. Quarantine vs Fail Fast

```
Chiến lược Quarantine (Cách Ly):
  Dùng khi: Dữ liệu lỗi là ngoại lệ (< 5%), không muốn mất toàn bộ batch
  Cách làm: Tách records lỗi sang quarantine zone, load phần còn lại
  Hệ quả: Pipeline tiếp tục chạy, team review quarantine data sau

Chiến lược Fail Fast (Dừng Nhanh):
  Dùng khi: Lỗi ảnh hưởng đến toàn bộ dataset (schema sai, source system lỗi)
  Cách làm: Dừng pipeline, alert ngay, không load dữ liệu xấu
  Hệ quả: Downstream không nhận dữ liệu xấu, cần fix source trước

Kết hợp:
  Critical rules → Fail Fast
  Warning rules → Quarantine
```

### 5. Version Control Rules (Quản Lý Phiên Bản Rules)

```bash
# Lưu DQDL rules vào Git repository như code
# File: data-quality/orders_rules.dqdl

Rules = [
    # v1.0 (2024-01-01): Basic completeness
    IsComplete "order_id",
    IsComplete "user_id",

    # v1.1 (2024-03-15): Thêm validation sau incident giá âm
    ColumnValues "amount" > 0,
    ColumnValues "amount" <= 10000000,

    # v2.0 (2024-06-01): Thêm sau khi mở rộng sang thị trường mới
    ColumnValues "currency" in ["VND", "USD", "SGD", "THB"],

    RowCount >= 1000
]
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Glue Data Quality khác gì với việc validate dữ liệu trong ETL script thông thường?**
> ETL script validate thủ công: viết code Python/PySpark, khó tái sử dụng, không có reporting. Glue Data Quality: ngôn ngữ DQDL declarative dễ đọc, tự động gửi metrics lên CloudWatch, tạo báo cáo có thể share, có Rule Recommendations tự động. Khi scale lên hàng trăm bảng, Glue DQ dễ quản lý hơn nhiều so với viết validation logic riêng cho từng ETL job.

**Q: Chiến lược Quarantine Pattern là gì và khi nào áp dụng?**
> Quarantine Pattern tách records không đạt chất lượng sang một vùng riêng (quarantine zone trong S3) thay vì từ chối toàn bộ batch. Áp dụng khi: lỗi dữ liệu là ngoại lệ (ví dụ: 0.5% records có email sai định dạng) và bạn không muốn mất 99.5% data tốt còn lại. Team sẽ review quarantine zone định kỳ để xử lý data lỗi.

**Q: Làm sao quyết định threshold cho Data Quality rules?**
> (1) Chạy profiling trên 30-90 ngày data lịch sử để lấy baseline thực tế; (2) Đặt threshold = baseline - buffer an toàn (ví dụ: baseline 97% → threshold 95%); (3) Phân biệt rules nghiêm ngặt (primary key = 100%) và rules linh hoạt (optional field = 70%); (4) Review và điều chỉnh threshold sau mỗi quý dựa trên xu hướng thực tế.

**Q: DetectAnomalies rule hoạt động thế nào?**
> `DetectAnomalies` dùng machine learning (học máy) để học pattern bình thường của metric (ví dụ: RowCount hàng ngày) qua lịch sử, sau đó tự động phát hiện khi giá trị bất thường. Không cần đặt threshold cứng — phù hợp cho metrics có seasonality (mùa vụ) như daily_revenue (cao vào cuối tuần, thấp đầu tuần). Cần ít nhất 30 lần chạy để học đủ pattern.

**Q: Làm sao tích hợp Data Quality vào CI/CD pipeline?**
> (1) Lưu DQDL rules trong Git repository; (2) Khi deploy ETL job mới, chạy Data Quality evaluation trên test dataset; (3) CI pipeline fail nếu quality không đạt; (4) Dùng Glue Workflows để chạy DQ check tự động sau mỗi ETL job; (5) Gửi kết quả lên CloudWatch Dashboard để theo dõi trend chất lượng theo thời gian.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**File:** `03-glue/4-glue-data-quality.md`
**Module Hoàn Thành:** ✅ 03-glue — 5 files (README + 4 topics)
**Xem Tiếp:** `04-athena/` — Serverless SQL query trên S3
