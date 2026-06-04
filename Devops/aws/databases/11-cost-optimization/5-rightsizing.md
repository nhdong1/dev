# Right-sizing & Storage Optimization — Định Cỡ Phù Hợp & Tối Ưu Lưu Trữ

> Hướng dẫn toàn diện về right-sizing (định cỡ phù hợp) database instances và tối ưu storage (lưu trữ) trên AWS — từ phân tích metrics, quy trình thay đổi instance size, đến các chiến lược lưu trữ tiết kiệm nhất.

## 📚 Mục Lục

1. [Right-sizing Là Gì](#right-sizing-là-gì)
2. [Công Cụ Phân Tích](#công-cụ-phân-tích)
3. [Right-sizing RDS & Aurora](#right-sizing-rds--aurora)
4. [Right-sizing DynamoDB](#right-sizing-dynamodb)
5. [Right-sizing ElastiCache](#right-sizing-elasticache)
6. [Storage Optimization — Tối Ưu Lưu Trữ](#storage-optimization)
7. [Lifecycle Management — Quản Lý Vòng Đời Dữ Liệu](#lifecycle-management)
8. [Automation Framework — Khung Tự Động Hóa](#automation-framework)
9. [Cost Reporting — Báo Cáo Chi Phí](#cost-reporting)

---

## Right-sizing Là Gì

Right-sizing là quá trình đảm bảo mỗi tài nguyên AWS được cấp phát đúng kích thước — không quá lớn (gây lãng phí chi phí) và không quá nhỏ (gây bottleneck hiệu năng).

### Vấn Đề Phổ Biến Trong Thực Tế

```
Over-provisioning (Cấp Phát Quá Mức):
- Lý do: "Just in case" (dùng khi cần)
- Kết quả: Trả tiền cho capacity không dùng
- Ước tính: 30-40% workloads AWS bị over-provisioned

Under-provisioning (Cấp Phát Thiếu):
- Lý do: Cắt giảm chi phí quá mức hoặc tính toán sai
- Kết quả: Performance degradation, throttling, poor user experience
- Khó phát hiện hơn over-provisioning

Right-sizing mục tiêu:
- Không phải "dùng ít nhất" — mà là "dùng đúng lượng cần thiết"
- Buffer 20-30% cho unexpected peaks
- Không buffer quá nhiều — đó là tiền lãng phí
```

### Lộ Trình Right-sizing

```
1. Measure (Đo Lường): Thu thập metrics 2-4 tuần
2. Analyze (Phân Tích): Xác định pattern, peaks, valleys
3. Recommend (Đề Xuất): Tính instance size phù hợp
4. Test (Kiểm Thử): Apply thay đổi trên dev/staging trước
5. Apply (Áp Dụng): Thay đổi production với rollback plan
6. Monitor (Giám Sát): Theo dõi sau thay đổi 1-2 tuần
7. Repeat (Lặp Lại): Right-sizing là quá trình liên tục
```

---

## Công Cụ Phân Tích

### AWS Compute Optimizer (Trình Tối Ưu Tính Toán)

```
Tính năng:
- Phân tích 14 ngày CloudWatch metrics tự động
- Đề xuất instance type tối ưu cho RDS
- Ước tính % tiết kiệm
- So sánh performance vs cost trade-offs

Bật Compute Optimizer:
aws compute-optimizer update-enrollment-status \
  --status Active \
  --include-member-accounts

Xem recommendations cho RDS:
aws compute-optimizer get-rds-database-recommendations \
  --account-ids 123456789012

Mỗi recommendation bao gồm:
- Current instance type
- Recommended instance type(s)
- Estimated monthly savings
- Performance risk assessment
```

### AWS Cost Explorer Rightsizing Recommendations

```
Cost Explorer → Rightsizing Recommendations → Database:
- Dựa trên 14-day utilization data
- Phân loại: Terminate (xóa), Downsize (giảm kích thước)
- Tính toán savings estimate tự động

Chi phí:
- Cost Explorer cơ bản: Miễn phí
- Rightsizing recommendations: Miễn phí
- Reservation recommendations: Miễn phí
```

### AWS Trusted Advisor (Cố Vấn Tin Cậy)

```
Checks liên quan đến database:
- "Amazon RDS Idle DB Instances": RDS instances ít hoặc không có connections
- "Amazon RDS Multi-AZ": Khi nào nên/không nên Multi-AZ
- "Amazon RDS Reserved Instance Optimization": Mua RI chưa?

Thresholds mặc định:
- "Idle" = < 4 connections/ngày + CPU < 5%
→ Xem xét xóa hoặc downsize

Trusted Advisor tiers:
- Business/Enterprise Support: Full checks
- Developer/Basic: Chỉ một số free checks
```

### CloudWatch Custom Dashboards

```bash
# Tạo CloudWatch dashboard theo dõi right-sizing metrics
aws cloudwatch put-dashboard \
  --dashboard-name "RDS-Rightsizing" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "metrics": [
            ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", "prod-db-1"],
            ["AWS/RDS", "FreeableMemory", "DBInstanceIdentifier", "prod-db-1"],
            ["AWS/RDS", "DatabaseConnections", "DBInstanceIdentifier", "prod-db-1"]
          ],
          "period": 300,
          "stat": "Average",
          "title": "RDS Key Metrics"
        }
      }
    ]
  }'
```

---

## Right-sizing RDS & Aurora

### Metrics Cần Thu Thập (4 Tuần)

```
CPU:
- CPUUtilization (Average và Maximum)
- Target: Average < 50%, Peak < 75%
- Nếu Average < 10% → Downsize an toàn

Memory:
- FreeableMemory (Bộ Nhớ Trống)
- Target: FreeableMemory > 15% total instance memory
- Nếu FreeableMemory < 100 MB → Upsize ngay

Connections:
- DatabaseConnections (Số Kết Nối Cơ Sở Dữ Liệu)
- Target: < 70% max_connections limit
- Nếu < 5% max connections → Có thể downsize

Storage I/O:
- ReadIOPS và WriteIOPS
- ReadLatency và WriteLatency
- Target Latency: < 1ms (gp3), < 0.5ms (io1)

Network:
- NetworkReceiveThroughput
- NetworkTransmitThroughput
```

### Decision Matrix — Ma Trận Quyết Định

```
Kịch bản 1: CPU thấp, Memory OK, Connections thấp
- Average CPU: 8%, Peak CPU: 25%
- FreeableMemory: 4 GB (30% total)
- Connections: 15 (limit: 500)
→ DOWNSIZE: 1-2 cấp, ước tính tiết kiệm 40-60%

Kịch bản 2: CPU cao nhưng Memory OK
- Average CPU: 70%, Peak CPU: 90%
- FreeableMemory: 3 GB (25% total)
- Connections: normal
→ Investigate: Slow queries? Unoptimized indexes?
  Nếu sau optimize vẫn cao → UPSIZE CPU (compute-optimized)
  Hoặc thêm Read Replica để scale reads

Kịch bản 3: Memory ít, CPU OK
- Average CPU: 25%
- FreeableMemory: 200 MB (< 5% total)
- Connections: normal
→ UPSIZE Memory: Chọn memory-optimized instance (r-series)
  Không nên downsize trong trường hợp này

Kịch bản 4: Connections gần limit
- Connections: 450/500 (90% max)
→ KHÔNG phải vấn đề instance size
  Dùng RDS Proxy (Proxy RDS) để pool connections
  Hoặc review connection management trong application
```

### Quy Trình Thay Đổi RDS Instance (Zero Downtime với Multi-AZ)

```bash
# Bước 1: Verify Multi-AZ trước khi apply
aws rds describe-db-instances \
  --db-instance-identifier prod-mysql-01 \
  --query 'DBInstances[0].MultiAZ'

# Bước 2: Downsize instance (thực hiện trong maintenance window)
# "apply-immediately" = ngay lập tức (sẽ có brief reboot trên single-AZ)
# Không có apply-immediately = thực hiện trong maintenance window

# Multi-AZ: Failover sang standby trước, modify primary, không có downtime
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-01 \
  --db-instance-class db.r6g.large \
  # Bỏ --apply-immediately để thực hiện trong maintenance window

# Bước 3: Monitor sau khi thay đổi
# Theo dõi 48 giờ: CPU, Memory, Connections, Latency
# Nếu performance suy giảm → Upsize lại ngay
```

### Instance Family Upgrades — Tiết Kiệm Bằng Upgrade Generation

```
AWS thường xuyên release generation mới của instance families:
- New generation: Hiệu năng cao hơn 10-20% cùng giá
  hoặc Giá thấp hơn 10-20% cùng hiệu năng

Ví dụ upgrade path:
db.r5.large → db.r6g.large: 
  - r6g dùng AWS Graviton2 (Chip Tùy Chỉnh ARM)
  - 10% cheaper + 30% better price/performance
  - Nhưng: r6g là ARM architecture → cần kiểm tra compatibility

db.r6g.large → db.r7g.large:
  - r7g dùng AWS Graviton3
  - Tốt hơn r6g ~25% price/performance
  - Cùng giá nhưng hiệu năng cao hơn

→ Upgrade generation = free performance boost
  hoặc có thể downsize 1 cấp mà vẫn cùng performance

Kiểm tra compatibility trước khi migrate sang Graviton:
- MySQL, PostgreSQL trên Aurora và RDS: Full support
- Oracle, SQL Server: Không hỗ trợ Graviton
- MariaDB: Full support
```

---

## Right-sizing DynamoDB

### Provisioned Capacity Optimization

```
CloudWatch Metrics quan trọng:
- ConsumedReadCapacityUnits (RCU Đã Dùng)
- ConsumedWriteCapacityUnits (WCU Đã Dùng)
- ProvisionedReadCapacityUnits (RCU Được Cấp Phát)
- ProvisionedWriteCapacityUnits (WCU Được Cấp Phát)

Tính utilization:
Read Utilization (%) = ConsumedRCU / ProvisionedRCU × 100

Target: 70% average utilization
Nếu < 30% consistently → Scale down hoặc đổi On-Demand
Nếu > 90% consistently → Scale up hoặc đổi On-Demand

Công thức tính WCU/RCU tối ưu:
Optimal Provisioned = Average_consumed × (1 / target_utilization)
Ví dụ: Average 50 WCU, target 70% → Provision 50/0.7 ≈ 72 WCU
```

### Auto Scaling Fine-tuning (Tinh Chỉnh Tự Động Co Giãn)

```bash
# Review và điều chỉnh Auto Scaling settings
aws application-autoscaling describe-scaling-policies \
  --service-namespace dynamodb

# Điều chỉnh target utilization
# Lower target (60%) = scale out sớm hơn, buffer lớn hơn = chi phí cao hơn
# Higher target (80%) = scale out muộn hơn, buffer nhỏ hơn = có thể throttle

# Best practice: Target = 70% để cân bằng cost và throttle prevention

# Điều chỉnh min/max capacity
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/my-table" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --min-capacity 10 \   # Tăng min nếu cold start là vấn đề
  --max-capacity 500    # Giới hạn max để kiểm soát chi phí
```

---

## Right-sizing ElastiCache

### Memory Sizing Methodology — Phương Pháp Tính Kích Thước Bộ Nhớ

```
Bước 1: Đo memory usage thực tế
redis-cli INFO memory
→ used_memory_human: Bộ nhớ đang dùng
→ maxmemory_human: Giới hạn memory
→ mem_fragmentation_ratio: Tỷ lệ phân mảnh (> 1.5 = vấn đề)

Bước 2: Dự báo data growth
- Hiện tại: 5 GB cached data
- Growth rate: 10% mỗi tháng
- 6 tháng sau: 5 × 1.1^6 ≈ 8.86 GB

Bước 3: Tính node size cần thiết
Required memory = max_dataset_size × 1.2 (20% buffer)
                + OS overhead (~500 MB)
                + Redis memory overhead

Ví dụ:
Dataset: 8 GB
Buffer: 8 × 1.2 = 9.6 GB
OS: 0.5 GB
Total: ~10.1 GB

→ Chọn cache.r7g.large (13.1 GB) ← phù hợp
→ Không cần cache.r7g.xlarge (26.3 GB) ← over-provisioned

Tiết kiệm: $239/tháng - $119/tháng = $120/tháng
```

---

## Storage Optimization — Tối Ưu Lưu Trữ

### RDS Storage Optimization

#### Migrate gp2 → gp3 (Dễ, Tiết Kiệm Ngay)

```
gp2: $0.115/GB-month
gp3: $0.092/GB-month (20% rẻ hơn)

Tiết kiệm theo storage size:
100 GB:   ($0.115 - $0.092) × 100 = $2.30/tháng
500 GB:   $11.50/tháng
1 TB:     $23.00/tháng
5 TB:     $115.00/tháng ($1,380/năm)

Migration: Zero downtime với Multi-AZ
  aws rds modify-db-instance \
    --db-instance-identifier my-rds-db \
    --storage-type gp3 \
    --apply-immediately

Nếu cần IOPS cao hơn 3,000 baseline của gp3:
  aws rds modify-db-instance \
    --db-instance-identifier my-rds-db \
    --storage-type gp3 \
    --iops 5000 \
    --storage-throughput 500  # MiBps
```

#### Reclaim Unused Allocated Storage

```
Vấn đề: RDS storage chỉ tăng, không thể giảm trực tiếp
Nếu allocated 1 TB nhưng chỉ dùng 100 GB → trả tiền 1 TB

Giải pháp giảm allocated storage:
1. Snapshot database hiện tại
2. Restore từ snapshot với storage size mới (nhỏ hơn)
3. Point application sang instance mới
4. Xóa instance cũ

Automation với AWS CloudFormation/Terraform:
- Định nghĩa storage size trong IaC (Infrastructure as Code)
- Review và adjust khi deploy mới
```

#### Aurora Storage Auto-scaling

```
Aurora storage tự động tăng khi cần:
- Bắt đầu: Sử dụng thực tế (không cần pre-allocate)
- Increments: Tăng theo 10 GB
- Giảm: Tự động shrink khi free space tăng (Aurora 3.x+)

Monitoring storage:
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraVolumeBytesLeftTotal \
  --dimensions Name=DBClusterIdentifier,Value=my-aurora-cluster \
  --statistics Minimum \
  --period 3600 \
  --start-time 2026-05-01T00:00:00Z \
  --end-time 2026-05-15T00:00:00Z

Alert nếu AuroraVolumeBytesLeftTotal < 10 GB
```

### Data Archiving — Lưu Trữ Dữ Liệu Lạnh

#### Archive Old Data sang S3 — Chi Phí Tương Phản

```
RDS storage: $0.092/GB-month (gp3)
S3 Standard: $0.023/GB-month
S3 Glacier Instant Retrieval (Lấy Lại Tức Thời): $0.004/GB-month
S3 Glacier Deep Archive (Lưu Trữ Sâu): $0.00099/GB-month

Ví dụ: 1 TB data cũ hơn 2 năm

Lưu trong RDS: $92/tháng
Lưu trong S3 Standard: $23/tháng (-75%)
Lưu trong S3 Glacier Instant: $4/tháng (-96%)
Lưu trong S3 Glacier Deep Archive: $0.99/tháng (-99%)

Chiến lược archive (lưu trữ) từng bước:
1. Data < 6 tháng: Giữ trong RDS (hot data)
2. Data 6-24 tháng: Archive sang S3 Standard
3. Data > 24 tháng: S3 Glacier Instant Retrieval
4. Data > 5 năm (compliance): S3 Glacier Deep Archive
```

```sql
-- Ví dụ: Archive orders cũ hơn 2 năm từ MySQL sang S3 (qua Aurora)
-- Dùng AWS DMS (Database Migration Service) hoặc custom Lambda

-- Bước 1: Export data sang S3
SELECT * FROM orders 
WHERE created_at < DATE_SUB(NOW(), INTERVAL 2 YEAR)
INTO OUTFILE S3 's3://my-archive-bucket/orders/2024/'
FORMAT CSV;

-- Bước 2: Xóa data đã archive (sau khi verify)
DELETE FROM orders 
WHERE created_at < DATE_SUB(NOW(), INTERVAL 2 YEAR)
LIMIT 10000;  -- Xóa theo batch, không lock table
-- Lặp lại cho đến khi hết
```

### DynamoDB Storage Optimization

#### Compression — Nén Dữ Liệu

```python
import gzip
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('my-table')

# Write với compression (nén)
def write_compressed(key, data):
    json_bytes = json.dumps(data).encode('utf-8')
    compressed = gzip.compress(json_bytes)
    
    table.put_item(Item={
        'id': key,
        'data': compressed,           # Binary data, DynamoDB lưu dưới dạng Binary
        'compressed': True
    })

# Read với decompression (giải nén)
def read_compressed(key):
    response = table.get_item(Key={'id': key})
    item = response.get('Item', {})
    
    if item.get('compressed'):
        return json.loads(gzip.decompress(item['data'].value))
    return item.get('data')

# Tiết kiệm điển hình: 50-80% kích thước item
# Item 10 KB JSON → 2-5 KB sau compress
# WCU cost giảm 5-10x
```

#### S3 cho Large Objects — Đối Tượng Lớn

```python
# Pattern: Lưu large data trong S3, chỉ giữ S3 key trong DynamoDB
import boto3
import uuid

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('products')

def store_product_with_large_description(product_id, product_name, large_html):
    # Upload large HTML description lên S3
    s3_key = f"product-descriptions/{product_id}.html"
    s3.put_object(
        Bucket='my-content-bucket',
        Key=s3_key,
        Body=large_html.encode('utf-8'),
        ContentType='text/html'
    )
    
    # Chỉ lưu S3 key trong DynamoDB (item nhỏ gọn)
    table.put_item(Item={
        'product_id': product_id,
        'name': product_name,
        'description_s3_key': s3_key   # Reference thay vì data thực tế
    })

# DynamoDB item: ~100 bytes thay vì 50 KB
# Tiết kiệm: 500x ít WCU/WRU, 500x ít storage cost trong DynamoDB
```

---

## Lifecycle Management — Quản Lý Vòng Đời Dữ Liệu

### Automated Snapshot Lifecycle

```bash
# Tạo lifecycle policy cho RDS snapshots tự động
# Dùng AWS Data Lifecycle Manager (Trình Quản Lý Vòng Đời Dữ Liệu)

aws dlm create-lifecycle-policy \
  --description "RDS Snapshot Lifecycle" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123456789:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["INSTANCE"],
    "TargetTags": [{"Key": "Environment", "Value": "production"}],
    "Schedules": [
      {
        "Name": "Daily-Snapshots",
        "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS"},
        "RetainRule": {"Count": 7},
        "CopyTags": true
      }
    ]
  }'
```

### Intelligent Tiering Cho S3 Archives

```bash
# Bật S3 Intelligent-Tiering cho archive bucket
# Tự động chuyển giữa các tier dựa trên access pattern

aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-database-archives \
  --id default \
  --intelligent-tiering-configuration '{
    "Id": "default",
    "Status": "Enabled",
    "Tierings": [
      {
        "Days": 90,
        "AccessTier": "ARCHIVE_ACCESS"
      },
      {
        "Days": 180,
        "AccessTier": "DEEP_ARCHIVE_ACCESS"
      }
    ]
  }'

# Cost saving:
# Data không access trong 90 ngày → Archive tier ($0.004/GB)
# Data không access trong 180 ngày → Deep Archive ($0.00099/GB)
# Không cần quản lý manual transitions
```

---

## Automation Framework — Khung Tự Động Hóa

### Lambda-based Right-sizing Recommendations

```python
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')
rds = boto3.client('rds')

def get_rds_cpu_stats(db_instance_id, days=14):
    """Lấy CPU statistics cho 14 ngày."""
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=days)
    
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/RDS',
        MetricName='CPUUtilization',
        Dimensions=[
            {'Name': 'DBInstanceIdentifier', 'Value': db_instance_id}
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=3600,  # 1 giờ
        Statistics=['Average', 'Maximum']
    )
    
    datapoints = response['Datapoints']
    if not datapoints:
        return None
    
    avg_cpu = sum(d['Average'] for d in datapoints) / len(datapoints)
    max_cpu = max(d['Maximum'] for d in datapoints)
    
    return {'average': avg_cpu, 'maximum': max_cpu}

def generate_rightsizing_report():
    """Tạo báo cáo right-sizing cho tất cả RDS instances."""
    instances = rds.describe_db_instances()
    report = []
    
    for db in instances['DBInstances']:
        db_id = db['DBInstanceIdentifier']
        current_class = db['DBInstanceClass']
        
        cpu_stats = get_rds_cpu_stats(db_id)
        if not cpu_stats:
            continue
        
        recommendation = None
        if cpu_stats['average'] < 10 and cpu_stats['maximum'] < 30:
            recommendation = 'STRONGLY_CONSIDER_DOWNSIZE'
        elif cpu_stats['average'] < 20 and cpu_stats['maximum'] < 50:
            recommendation = 'CONSIDER_DOWNSIZE'
        elif cpu_stats['maximum'] > 85:
            recommendation = 'CONSIDER_UPSIZE'
        else:
            recommendation = 'OPTIMAL'
        
        report.append({
            'db_instance': db_id,
            'current_class': current_class,
            'avg_cpu': f"{cpu_stats['average']:.1f}%",
            'max_cpu': f"{cpu_stats['maximum']:.1f}%",
            'recommendation': recommendation
        })
    
    return report
```

### Schedule Dev/Test Databases

```python
import boto3

rds = boto3.client('rds')

def stop_dev_databases():
    """Dừng tất cả dev/test RDS instances vào cuối ngày."""
    instances = rds.describe_db_instances()
    
    for db in instances['DBInstances']:
        tags = rds.list_tags_for_resource(
            ResourceName=db['DBInstanceArn']
        )['TagList']
        
        env_tag = next(
            (t['Value'] for t in tags if t['Key'] == 'Environment'),
            None
        )
        
        if env_tag in ['dev', 'test', 'staging']:
            if db['DBInstanceStatus'] == 'available':
                print(f"Stopping: {db['DBInstanceIdentifier']}")
                rds.stop_db_instance(
                    DBInstanceIdentifier=db['DBInstanceIdentifier']
                )

# Deploy như EventBridge Rule (Quy Tắc EventBridge) để chạy tự động
# Cron: "cron(0 18 ? * MON-FRI *)" → 6PM weekdays
# Start function: "cron(0 8 ? * MON-FRI *)" → 8AM weekdays
```

---

## Cost Reporting — Báo Cáo Chi Phí

### Thiết Lập Cost Allocation Tags (Nhãn Phân Bổ Chi Phí)

```bash
# Gắn tags cho tất cả database resources
# Tags này xuất hiện trong Cost Explorer để phân tích chi phí theo team/project

# Gắn tags cho RDS instances
aws rds add-tags-to-resource \
  --resource-name arn:aws:rds:us-east-1:123456789:db:prod-mysql-01 \
  --tags \
    Key=Team,Value=backend \
    Key=Project,Value=ecommerce \
    Key=Environment,Value=production \
    Key=CostCenter,Value=engineering-001

# Bật Cost Allocation Tags trong Billing Console
# Billing → Cost Allocation Tags → Activate user-defined tags
# (Tags chỉ hiện trong Cost Explorer sau khi được activate)
```

### Automated Monthly Cost Report

```python
import boto3
from datetime import datetime, date
from calendar import monthrange

ce = boto3.client('ce')

def get_database_costs_by_service(year, month):
    """Lấy chi phí database breakdown theo service trong tháng."""
    _, last_day = monthrange(year, month)
    start = f"{year}-{month:02d}-01"
    end = f"{year}-{month:02d}-{last_day}"
    
    response = ce.get_cost_and_usage(
        TimePeriod={'Start': start, 'End': end},
        Granularity='MONTHLY',
        Filter={
            'Dimensions': {
                'Key': 'SERVICE',
                'Values': [
                    'Amazon Relational Database Service',
                    'Amazon DynamoDB',
                    'Amazon ElastiCache'
                ]
            }
        },
        GroupBy=[
            {'Type': 'DIMENSION', 'Key': 'SERVICE'},
            {'Type': 'TAG', 'Key': 'Team'}
        ],
        Metrics=['BlendedCost']
    )
    
    return response['ResultsByTime'][0]['Groups']

# Kết quả: Chi phí theo từng database service × từng team
# Giúp charge back (phân bổ chi phí) theo team
```

---

## Checklist Right-sizing Tổng Thể

```
Thiết lập ban đầu (1 lần):
□ Bật Cost Allocation Tags cho tất cả database resources
□ Bật AWS Compute Optimizer
□ Thiết lập CloudWatch dashboards cho key metrics
□ Thiết lập budget alerts (cảnh báo ngân sách) cho mỗi service
□ Kích hoạt AWS Trusted Advisor (cần Business Support)

Hàng tuần:
□ Review Trusted Advisor recommendations
□ Check instances với CPU < 10% sustained (duy trì liên tục)
□ Check FreeableMemory < 20% cho RDS
□ Review ElastiCache evictions và hit rate

Hàng tháng:
□ Chạy right-sizing report từ Compute Optimizer
□ Review Cost Explorer → top services by cost
□ Xóa unused snapshots (> 90 ngày không dùng)
□ Review backup retention periods — có hợp lý không?
□ Đánh giá Reserved Instances utilization (target > 90%)

Hàng quý:
□ Full right-sizing audit cho tất cả instances
□ Đánh giá migration sang newer instance generations
□ Review DynamoDB capacity modes — On-Demand vs Provisioned
□ Audit dev/test instances — có cần thiết không?
□ Planning Reserved Instances renewals và purchases
□ Storage optimization review — gp2 → gp3, archive old data

Hàng năm:
□ Architecture review — có dịch vụ nào cần re-architect không?
□ Reserved Instances renewal strategy (1yr vs 3yr)
□ Evaluate new AWS service offerings (có service mới rẻ hơn không?)
□ Team cost awareness training (đào tạo nhận thức chi phí)
```

---

**Cập Nhật Lần Cuối:** 2026-05-15
**File:** 11-cost-optimization/5-rightsizing.md
