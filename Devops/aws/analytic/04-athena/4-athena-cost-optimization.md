# Athena Cost Optimization — Tối Ưu Chi Phí Amazon Athena

> Athena tính phí $5 per TB dữ liệu quét. Tài liệu này hướng dẫn toàn diện cách giảm chi phí thông qua kỹ thuật format dữ liệu, partitioning, quản lý workload, và kiểm soát ngân sách.

---

## 📚 Mục Lục

1. [Hiểu Mô Hình Tính Phí](#1-hiểu-mô-hình-tính-phí)
2. [Giảm Dữ Liệu Quét — Kỹ Thuật Cốt Lõi](#2-giảm-dữ-liệu-quét--kỹ-thuật-cốt-lõi)
3. [Workgroups — Quản Lý Workload](#3-workgroups--quản-lý-workload)
4. [Query Controls — Kiểm Soát Truy Vấn](#4-query-controls--kiểm-soát-truy-vấn)
5. [S3 Storage Optimization](#5-s3-storage-optimization)
6. [Athena CTAS Để Tối Ưu Data Layout](#6-athena-ctas-để-tối-ưu-data-layout)
7. [Saved Queries và Query Caching](#7-saved-queries-và-query-caching)
8. [Monitoring Chi Phí Với CloudWatch](#8-monitoring-chi-phí-với-cloudwatch)
9. [Cost Allocation Tagging](#9-cost-allocation-tagging)
10. [Tổng Hợp — Checklist Tối Ưu Chi Phí](#10-tổng-hợp--checklist-tối-ưu-chi-phí)

---

## 1. Hiểu Mô Hình Tính Phí

### Cấu Trúc Phí Athena

```
Athena Pricing (2024):
├── SQL queries:   $5.00 per TB data scanned (dữ liệu quét)
│                  Minimum charge: 10 MB per query
│
├── Athena for Spark (notebook/interactive):
│   └── $0.005 per DPU-hour
│
└── Không tính phí cho:
    ├── DDL queries (CREATE TABLE, DROP TABLE, ALTER TABLE)
    ├── Failed queries
    ├── Cancelled queries (nếu hủy trước khi bắt đầu)
    └── Metadata operations (SHOW TABLES, DESCRIBE, ...)

Lưu ý thêm:
├── S3 GET requests: $0.0004/1,000 requests
├── S3 storage: $0.023/GB/tháng
└── Data transfer: $0.09/GB (nếu ra ngoài region)
```

### Ví Dụ Tính Chi Phí

```
Bảng dữ liệu log: 1 TB CSV (không nén)
Query: SELECT COUNT(*) WHERE status_code = 500

Không tối ưu:
→ Quét toàn bộ 1 TB
→ Chi phí: 1 TB × $5 = $5.00/query

Sau khi chuyển sang Parquet ZSTD + partition theo ngày:
→ Chỉ quét cột status_code (1% của data) = ~10 GB raw
→ Sau compression (60% giảm): ~4 GB scanned  
→ Sau partition filter (query 1 ngày / 365 ngày): ~11 MB
→ Chi phí: 11 MB × $5/TB = $0.00005/query
→ Tiết kiệm: 99.99%
```

---

## 2. Giảm Dữ Liệu Quét — Kỹ Thuật Cốt Lõi

### Kỹ Thuật 1: Columnar Format + Compression

```
Impact: Giảm 60-95% dữ liệu quét

Nguyên lý:
- Columnar: Chỉ đọc columns được SELECT/WHERE (không đọc cả row)
- Compression: Giảm bytes thực sự đọc từ S3

Best practice:
- Format: Parquet (khuyến nghị) hoặc ORC
- Compression: ZSTD (tốt nhất) hoặc SNAPPY (nhanh nhất)
- Row group size: 128 MB (mặc định Parquet — tốt cho Athena)
```

Xem [2-athena-performance.md](./2-athena-performance.md) để biết chi tiết kỹ thuật.

### Kỹ Thuật 2: Partition Pruning

```
Impact: Giảm 70-99% dữ liệu quét (tùy query pattern)

Nguyên lý:
- Thêm partition columns vào WHERE clause
- Athena đọc Glue Catalog → biết files nào cần đọc
- Files không liên quan bị bỏ qua hoàn toàn

Ví dụ:
-- Quét 100%: Không có partition filter
SELECT SUM(revenue) FROM events;

-- Quét ~0.3% (1 ngày / 365 ngày):
SELECT SUM(revenue) FROM events WHERE dt = '2024-03-15';
```

### Kỹ Thuật 3: Predicate Pushdown Với Parquet Statistics

```
Parquet row group statistics (từ footer):
- min value, max value, null count cho mỗi column
- Athena đọc footer trước → bỏ qua row groups không thỏa điều kiện

Ví dụ:
Row Group 1: amount min=10, max=100
Row Group 2: amount min=150, max=500
Row Group 3: amount min=600, max=1200

Query: WHERE amount > 400
→ Bỏ qua Row Group 1 hoàn toàn (max=100 < 400)
→ Đọc Row Group 2 và 3
→ Tiết kiệm 33%
```

### Kỹ Thuật 4: Column Projection — Tránh SELECT *

```sql
-- ❌ Quét tất cả columns (với Parquet, đọc toàn bộ row groups)
SELECT * FROM orders WHERE year = 2024;

-- ✅ Chỉ quét columns cần thiết
SELECT order_id, customer_id, amount FROM orders WHERE year = 2024;

-- Với 20-column Parquet table:
-- SELECT * → 100% column data
-- SELECT 3 cols → ~15% column data
-- → Tiết kiệm ~85% dữ liệu quét
```

---

## 3. Workgroups — Quản Lý Workload

### Workgroup Là Gì?

**Workgroup** (Nhóm Công Việc) trong Athena cho phép:
- Tách biệt query resources giữa các teams
- Set limits (giới hạn) về dữ liệu quét
- Kiểm soát chi phí theo team/project
- Cấu hình output location riêng
- Theo dõi metrics theo workgroup

### Tạo Workgroup

```bash
# Tạo workgroup cho team Data Analytics với giới hạn chi phí
aws athena create-work-group \
  --name "data-analytics-team" \
  --configuration '{
    "ResultConfiguration": {
      "OutputLocation": "s3://my-results/data-analytics/"
    },
    "EnforceWorkGroupConfiguration": true,
    "PublishCloudWatchMetricsEnabled": true,
    "BytesScannedCutoffPerQuery": 107374182400,
    "RequesterPaysEnabled": false
  }' \
  --description "Workgroup cho team Data Analytics — giới hạn 100 GB/query"
```

### Cấu Hình Workgroup Quan Trọng

```json
{
  "Configuration": {
    "ResultConfiguration": {
      "OutputLocation": "s3://results/team-a/",
      "EncryptionConfiguration": {
        "EncryptionOption": "SSE_KMS",
        "KmsKey": "arn:aws:kms:..."
      }
    },
    
    "BytesScannedCutoffPerQuery": 10737418240,
    
    "EnforceWorkGroupConfiguration": true,
    
    "PublishCloudWatchMetricsEnabled": true,
    
    "EngineVersion": {
      "SelectedEngineVersion": "Athena engine version 3"
    }
  }
}
```

### BytesScannedCutoffPerQuery — Giới Hạn Dữ Liệu Quét

```
BytesScannedCutoffPerQuery: 10737418240  (= 10 GB)

Cơ chế hoạt động:
1. Query bắt đầu chạy
2. Athena track bytes đã quét
3. Nếu vượt 10 GB → query bị hủy tự động
4. User nhận error: "Query was cancelled because it exceeded the data scanned limit"

Lưu ý:
- Query bị hủy KHÔNG bị tính phí (chỉ phí phần đã quét)
- Minimum vẫn là 10 MB
- Không áp dụng cho DDL queries
```

### Workgroup Per Team Example

```
Organization: TechCorp
├── Primary workgroup (default): admin operations
│   └── No limit (trusted users)
│
├── Workgroup "data-science": 
│   └── Limit: 1 TB/query — data scientists có thể chạy large queries
│
├── Workgroup "reporting":
│   └── Limit: 100 GB/query — dashboards và scheduled reports
│
└── Workgroup "sandbox":
    └── Limit: 10 GB/query — testing và exploration

IAM Policy — chỉ cho phép user/role dùng workgroup nhất định:
{
  "Effect": "Allow",
  "Action": "athena:StartQueryExecution",
  "Resource": "arn:aws:athena:*:*:workgroup/reporting"
}
```

---

## 4. Query Controls — Kiểm Soát Truy Vấn

### Estimate Cost Trước Khi Chạy Query

```python
import boto3

def estimate_query_cost(query: str, database: str) -> dict:
    """
    Ước tính chi phí query TRƯỚC khi chạy thực sự.
    Dùng Explain để phân tích, hoặc mock scan với LIMIT 0.
    """
    athena = boto3.client('athena')
    
    # Dry run: chỉ validate query, không scan data
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={'Database': database},
        ResultConfiguration={
            'OutputLocation': 's3://results/dry-run/'
        }
    )
    
    # Lấy statistics sau khi chạy
    execution = athena.get_query_execution(
        QueryExecutionId=response['QueryExecutionId']
    )
    
    bytes_scanned = execution['QueryExecution']['Statistics'].get(
        'DataScannedInBytes', 0
    )
    estimated_cost_usd = (bytes_scanned / (1024**4)) * 5
    
    return {
        'bytes_scanned': bytes_scanned,
        'gb_scanned': bytes_scanned / (1024**3),
        'estimated_cost_usd': estimated_cost_usd
    }
```

### Query Cost Guard — Wrapper Kiểm Tra Chi Phí

```python
import boto3
import time
from typing import Optional

class AthenaQueryGuard:
    """
    Wrapper chạy Athena query với kiểm tra chi phí.
    Tự động hủy nếu vượt ngưỡng chi phí.
    """
    
    def __init__(self, max_gb_threshold: float = 100.0):
        self.athena = boto3.client('athena')
        self.max_bytes = max_gb_threshold * (1024**3)
    
    def run_query(
        self,
        query: str,
        database: str,
        output_location: str,
        workgroup: str = 'primary'
    ) -> Optional[str]:
        
        # Bắt đầu query
        response = self.athena.start_query_execution(
            QueryString=query,
            QueryExecutionContext={'Database': database},
            ResultConfiguration={'OutputLocation': output_location},
            WorkGroup=workgroup
        )
        
        query_id = response['QueryExecutionId']
        print(f"Query started: {query_id}")
        
        # Monitor và kiểm tra
        while True:
            execution = self.athena.get_query_execution(
                QueryExecutionId=query_id
            )
            
            state = execution['QueryExecution']['Status']['State']
            stats = execution['QueryExecution'].get('Statistics', {})
            bytes_scanned = stats.get('DataScannedInBytes', 0)
            
            # Cảnh báo nếu đang quét quá nhiều
            if bytes_scanned > self.max_bytes * 0.8:
                print(f"⚠️  WARNING: Đã quét {bytes_scanned/(1024**3):.1f} GB")
            
            if state == 'SUCCEEDED':
                cost = (bytes_scanned / (1024**4)) * 5
                print(f"✅ Hoàn thành. Quét: {bytes_scanned/(1024**3):.2f} GB. Chi phí: ${cost:.4f}")
                return query_id
            elif state in ['FAILED', 'CANCELLED']:
                reason = execution['QueryExecution']['Status'].get('StateChangeReason', '')
                print(f"❌ Query {state}: {reason}")
                return None
            
            time.sleep(2)
```

### Named Queries — Truy Vấn Được Đặt Tên

Lưu các query thường dùng để tránh viết lại và kiểm soát chi phí:

```bash
# Tạo named query
aws athena create-named-query \
  --name "daily-revenue-report" \
  --description "Báo cáo doanh thu theo ngày — tối ưu với partition filter" \
  --database sales_db \
  --query-string "SELECT order_year, order_month, SUM(amount) AS revenue
                  FROM orders
                  WHERE order_year = year(current_date)
                  GROUP BY order_year, order_month
                  ORDER BY order_month"
```

---

## 5. S3 Storage Optimization

### Tổ Chức Dữ Liệu Để Tối Ưu Athena Cost

```
Cấu trúc thư mục được đề xuất:
s3://datalake/
├── raw/                          ← Bronze layer — data gốc, CSV/JSON
│   └── source-system/table/year=2024/month=01/
│
├── processed/                    ← Silver layer — Parquet, đã làm sạch
│   └── domain/table/year=2024/month=01/
│
└── analytics/                    ← Gold layer — Parquet, đã tổng hợp
    └── subject-area/report/year=2024/month=01/

Best practices:
- Analytics layer: Parquet ZSTD, partition theo query pattern
- Raw layer: giữ nguyên format gốc, không query Athena trực tiếp
- Processed layer: Parquet, partition theo time + category
```

### S3 Intelligent Tiering Và Athena

```
S3 Storage Classes và Athena:
┌──────────────────┬───────────────┬──────────────────────────────┐
│ Storage Class    │ Retrieval Cost│ Athena Compatibility         │
├──────────────────┼───────────────┼──────────────────────────────┤
│ S3 Standard      │ Miễn phí GET  │ ✅ Tốt nhất, không overhead  │
│ S3 Intelligent   │ Miễn phí GET  │ ✅ Tốt, tự động tier         │
│ S3 Standard-IA   │ $0.01/GB      │ ⚠️  Có retrieval cost         │
│ S3 One Zone-IA   │ $0.01/GB      │ ⚠️  Có retrieval cost         │
│ S3 Glacier Instant│ $0.03/GB     │ ⚠️  Có retrieval cost         │
│ S3 Glacier Flex  │ $0.01-0.03/GB │ ❌ Không hỗ trợ trực tiếp    │
│ S3 Glacier Deep  │ $0.0025/GB    │ ❌ Không hỗ trợ               │
└──────────────────┴───────────────┴──────────────────────────────┘

Khuyến nghị:
- Hot data (query thường xuyên): S3 Standard
- Warm data (query đôi khi): S3 Intelligent Tiering
- Cold data (ít query): Cân nhắc retrieval cost vs Athena cost
```

### S3 Lifecycle Cho Data Tiering

```json
{
  "Rules": [
    {
      "ID": "analytics-data-tiering",
      "Status": "Enabled",
      "Filter": {"Prefix": "analytics/"},
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "INTELLIGENT_TIERING"
        },
        {
          "Days": 90,
          "StorageClass": "STANDARD_IA"
        }
      ],
      "Expiration": {"Days": 365}
    },
    {
      "ID": "delete-athena-query-results",
      "Status": "Enabled",
      "Filter": {"Prefix": "query-results/"},
      "Expiration": {"Days": 7}
    }
  ]
}
```

---

## 6. Athena CTAS Để Tối Ưu Data Layout

### Dùng CTAS Để Compact Và Reformat

```sql
-- Scenario: CSV data cũ cần chuyển sang Parquet và partition
-- Bước 1: Tạo bảng mới với layout tốt hơn
CREATE TABLE analytics.orders_optimized
WITH (
    format                   = 'PARQUET',
    parquet_compression      = 'ZSTD',
    partitioned_by           = ARRAY['order_year', 'order_month'],
    external_location        = 's3://datalake/analytics/orders-v2/',
    
    -- Bucketing (tùy chọn) — cải thiện join performance
    bucketed_by              = ARRAY['customer_id'],
    bucket_count             = 64
)
AS
SELECT
    order_id,
    customer_id,
    amount,
    status,
    order_date,
    year(order_date)  AS order_year,
    month(order_date) AS order_month
FROM raw.orders_csv
WHERE order_date >= DATE '2020-01-01';

-- Bước 2: Cập nhật ứng dụng để dùng bảng mới
-- Bước 3: Drop bảng cũ sau khi verify
DROP TABLE raw.orders_csv;
```

### UNLOAD — Export Kết Quả Ra S3 Với Format Tùy Chỉnh

```sql
-- UNLOAD tương tự CTAS nhưng không tạo table entry trong Catalog
UNLOAD (
    SELECT customer_id, SUM(amount) AS total_spend
    FROM orders
    WHERE order_year = 2024
    GROUP BY customer_id
)
TO 's3://reports/customer-spend-2024/'
WITH (
    format = 'PARQUET',
    compression = 'SNAPPY'
);
```

---

## 7. Saved Queries và Query Caching

### Query Result Reuse — Tái Sử Dụng Kết Quả Query

Athena hỗ trợ **Query Result Reuse** (Tái Sử Dụng Kết Quả Truy Vấn): nếu cùng một query đã chạy gần đây với cùng dữ liệu, Athena có thể trả về kết quả cũ từ cache mà không cần chạy lại.

```python
# Bật query result reuse
response = athena.start_query_execution(
    QueryString='SELECT COUNT(*) FROM orders WHERE order_year = 2024',
    QueryExecutionContext={'Database': 'sales_db'},
    ResultConfiguration={
        'OutputLocation': 's3://results/athena/'
    },
    ResultReuseConfiguration={
        'ResultReuseByAgeConfiguration': {
            'Enabled': True,
            'MaxAgeInMinutes': 60  # Dùng lại kết quả trong vòng 60 phút
        }
    }
)
```

### Khi Nào Result Reuse Hoạt Động?

```
✅ Reuse khi:
- Cùng query text (case-insensitive)
- Cùng workgroup
- Kết quả chưa hết hạn (< MaxAgeInMinutes)
- Underlying data KHÔNG thay đổi (Athena kiểm tra ETag S3)

❌ Không reuse khi:
- Query text khác (dù kết quả giống nhau)
- Data ở S3 đã thay đổi
- Query dùng non-deterministic functions: NOW(), CURRENT_DATE, RANDOM()
```

---

## 8. Monitoring Chi Phí Với CloudWatch

### Dashboard Chi Phí Athena

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

# Tạo CloudWatch Alarm cảnh báo khi chi phí tăng cao
cloudwatch.put_metric_alarm(
    AlarmName='athena-daily-cost-alert',
    AlarmDescription='Cảnh báo khi Athena quét > 1 TB/ngày',
    MetricName='DataScannedInBytes',
    Namespace='AWS/Athena',
    Dimensions=[
        {
            'Name': 'WorkGroup',
            'Value': 'primary'
        }
    ],
    Statistic='Sum',
    Period=86400,       # 24 giờ
    EvaluationPeriods=1,
    Threshold=1099511627776,  # 1 TB in bytes
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=[
        'arn:aws:sns:ap-southeast-1:123456789:athena-cost-alerts'
    ]
)
```

### Cost Report Script

```python
import boto3
from datetime import datetime, timedelta

def generate_athena_cost_report(days: int = 7) -> dict:
    """Tạo báo cáo chi phí Athena 7 ngày gần nhất."""
    
    athena = boto3.client('athena')
    
    # Lấy lịch sử query executions
    total_bytes = 0
    query_count = 0
    expensive_queries = []
    
    paginator = athena.get_paginator('list_query_executions')
    
    cutoff_time = datetime.now() - timedelta(days=days)
    
    for page in paginator.paginate():
        for query_id in page['QueryExecutionIds']:
            try:
                execution = athena.get_query_execution(
                    QueryExecutionId=query_id
                )
                
                qe = execution['QueryExecution']
                submit_time = qe['Status'].get('SubmissionDateTime')
                
                if submit_time and submit_time.replace(tzinfo=None) < cutoff_time:
                    continue
                
                stats = qe.get('Statistics', {})
                bytes_scanned = stats.get('DataScannedInBytes', 0)
                
                total_bytes += bytes_scanned
                query_count += 1
                
                if bytes_scanned > 10 * (1024**3):  # > 10 GB
                    expensive_queries.append({
                        'query_id': query_id,
                        'bytes_scanned_gb': bytes_scanned / (1024**3),
                        'cost_usd': (bytes_scanned / (1024**4)) * 5,
                        'query': qe.get('Query', '')[:200]
                    })
            except Exception:
                continue
    
    total_cost = (total_bytes / (1024**4)) * 5
    
    return {
        'period_days': days,
        'total_queries': query_count,
        'total_tb_scanned': total_bytes / (1024**4),
        'total_cost_usd': total_cost,
        'avg_cost_per_query': total_cost / max(query_count, 1),
        'expensive_queries': sorted(
            expensive_queries, 
            key=lambda x: x['cost_usd'], 
            reverse=True
        )[:10]
    }
```

---

## 9. Cost Allocation Tagging

### Tag Workgroup Theo Team/Project

```bash
# Tag Athena workgroup
aws athena tag-resource \
  --resource-arn "arn:aws:athena:ap-southeast-1:123456789:workgroup/data-analytics" \
  --tags '[
    {"Key": "Team", "Value": "DataAnalytics"},
    {"Key": "CostCenter", "Value": "CC-12345"},
    {"Key": "Project", "Value": "CustomerInsights"},
    {"Key": "Environment", "Value": "production"}
  ]'
```

### AWS Cost Explorer Cho Athena

```
Trong AWS Cost Explorer (Trình Khám Phá Chi Phí):
1. Service: Amazon Athena
2. Group by: Tag → Team
3. Xem chi phí theo team/project
4. Thiết lập Budget alerts (cảnh báo ngân sách) theo tag

Sử dụng AWS Budgets:
- Tạo budget theo service + tag
- Nhận email khi sắp đạt 80%, 100% ngân sách
- Auto-action: disable IAM policy khi vượt ngưỡng
```

---

## 10. Tổng Hợp — Checklist Tối Ưu Chi Phí

### Tier 1 — Tác Động Cao, Dễ Thực Hiện (Làm Ngay)

```
☑ 1. Chuyển CSV/JSON → Parquet với SNAPPY/ZSTD compression
      Impact: Giảm 60-90% chi phí
      Cách làm: Dùng CTAS hoặc Glue ETL job

☑ 2. Thêm partition theo thời gian (year/month/day)
      Impact: Giảm 70-99% tùy query filter
      Cách làm: Thiết kế partition key khi tạo table

☑ 3. Tránh SELECT * — chỉ query columns cần thiết
      Impact: Giảm 30-80% với bảng nhiều columns
      Cách làm: Review tất cả queries hiện tại

☑ 4. Bật BytesScannedCutoffPerQuery trong workgroup
      Impact: Ngăn accidental expensive queries
      Cách làm: Set 10-100 GB tùy team
```

### Tier 2 — Tác Động Trung Bình, Cần Effort

```
☑ 5. Compact small files (< 10 MB) thành 128-512 MB
      Impact: Giảm 20-50% execution time và cost
      Cách làm: Scheduled CTAS job hàng ngày/tuần

☑ 6. Dùng Partition Projection cho time-series data
      Impact: Loại bỏ catalog lookup overhead
      Cách làm: ALTER TABLE PROPERTIES

☑ 7. Enable Query Result Reuse cho repeated queries
      Impact: 0% cost cho cached queries
      Cách làm: Set MaxAgeInMinutes = 60 phút

☑ 8. Tạo workgroup riêng cho từng team với limits
      Impact: Kiểm soát chi phí theo team
      Cách làm: CloudFormation / Terraform
```

### Tier 3 — Tác Động Thấp, Nâng Cao

```
☑ 9. Thu thập và sử dụng table statistics cho CBO
      Impact: Query planning tốt hơn
      Cách làm: ANALYZE TABLE định kỳ

☑ 10. S3 Intelligent Tiering cho warm data
       Impact: Giảm S3 storage cost
       Cách làm: S3 Lifecycle rules

☑ 11. Monitor CloudWatch metrics + cost alerts
       Impact: Phát hiện chi phí bất thường sớm
       Cách làm: CloudWatch Dashboard + Alarms

☑ 12. Cost allocation tags để chargeback
       Impact: Accountability theo team/project
       Cách làm: Tag workgroups + AWS Budgets
```

### Tóm Tắt ROI Ước Tính

```
Tình huống ban đầu: 10 TB CSV/ngày, 100 queries/ngày
Chi phí ban đầu: ~$5,000/ngày

Sau khi áp dụng:
1. Parquet ZSTD:            -80% → $1,000/ngày
2. Partition (query 1 tháng/12 tháng): -90% → $100/ngày
3. SELECT specific columns: -50% → $50/ngày
4. Result reuse (30% queries hit): -30% → $35/ngày

Tổng tiết kiệm: 99.3% — từ $5,000 xuống $35/ngày
Tiết kiệm hàng tháng: ~$146,000 (từ $150,000 → $1,050)
```

---

## 🔑 Key Principles Tối Ưu Chi Phí

```
1. SCAN LESS — Quét ít bytes hơn là mục tiêu số 1
   → Columnar format + partition + column projection

2. CONTROL COST — Đặt giới hạn để không bị "bill shock"
   → Workgroup BytesScannedCutoffPerQuery
   → CloudWatch Alarms + AWS Budgets

3. REUSE RESULTS — Tránh chạy cùng query nhiều lần
   → Query Result Reuse (60 phút TTL)
   → Saved Queries (Named Queries)

4. MONITOR AND ITERATE — Theo dõi và cải tiến liên tục
   → Track DataScannedInBytes per query
   → Identify và optimize top expensive queries hàng tuần

5. ORGANIZE DATA WELL — Data layout quyết định query cost
   → Compact small files
   → Medallion Architecture (Bronze/Silver/Gold)
   → Query Gold layer thay vì Bronze
```

---

**Hoàn thành module 04-athena!** Quay lại [README.md](./README.md) để xem tổng quan module.

**Module tiếp theo:** [05-redshift/README.md](../05-redshift/README.md) — Amazon Redshift MPP Data Warehouse
