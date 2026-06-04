# 📊 Cost Monitoring — Giám Sát Chi Phí AWS

> Hướng dẫn toàn diện về giám sát và kiểm soát chi phí AWS — từ AWS Cost Explorer (Khám Phá Chi Phí), AWS Budgets (Ngân Sách), Cost Anomaly Detection (Phát Hiện Bất Thường Chi Phí), Trusted Advisor (Cố Vấn Đáng Tin Cậy), đến Tagging Strategy (Chiến Lược Gán Nhãn) và Cost Allocation (Phân Bổ Chi Phí).

---

## 📚 Mục Lục

1. [Tại Sao Cost Monitoring Quan Trọng?](#tại-sao)
2. [AWS Cost Explorer](#cost-explorer)
3. [AWS Budgets](#aws-budgets)
4. [Cost Anomaly Detection](#anomaly-detection)
5. [AWS Trusted Advisor](#trusted-advisor)
6. [Tagging Strategy — Chiến Lược Gán Nhãn](#tagging)
7. [Cost Allocation Tags](#cost-allocation)
8. [AWS Cost and Usage Report — CUR](#cur)
9. [Dashboard & Báo Cáo Tổng Hợp](#dashboard)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 💡 Tại Sao Cost Monitoring Quan Trọng? {#tại-sao}

### "Cloud Bill Shock" — Sốc Hóa Đơn Cloud

Các sự cố chi phí phổ biến:
- **Infinite loop Lambda** — hàm gọi chính nó → chi phí tăng theo hàm mũ
- **Forgotten S3 bucket** với versioning bật — backup tăng mà không ai chú ý
- **Data transfer costs** — không biết egress pricing, chuyển dữ liệu liên region
- **Reserved Instance** hết hạn → tự động chuyển sang On-Demand giá cao
- **RDS snapshot** tích lũy hàng năm mà không xóa
- **NAT Gateway** xử lý traffic không cần thiết

### Nguyên Tắc Cost Visibility (Hiển Thị Chi Phí)

```
Không thể kiểm soát những gì bạn không đo được.

Cost Monitoring Framework:
├── VISIBILITY: Mọi người đều thấy chi phí của team mình
├── ACCOUNTABILITY: Mỗi team chịu trách nhiệm chi phí service của họ
├── ALERTING: Cảnh báo sớm khi chi phí vượt ngưỡng
└── ACTION: Quy trình rõ ràng khi phát hiện anomaly
```

---

## 🔍 AWS Cost Explorer {#cost-explorer}

### Tổng Quan

AWS Cost Explorer cung cấp **giao diện trực quan** để phân tích chi phí lịch sử và dự báo chi phí tương lai.

**Phí:** $0.01 per API request (web console miễn phí)

### Tính Năng Chính

#### 1. Cost & Usage Analysis (Phân Tích Chi Phí & Sử Dụng)

```bash
# API: Xem chi phí theo service trong tháng 1/2024
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics "BlendedCost" "UnblendedCost" "UsageQuantity" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --output table

# Group by Tag (cần bật Cost Allocation Tags trước)
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=TAG,Key=Team \
  --output table
```

#### 2. Cost Forecasting (Dự Báo Chi Phí)

```bash
# Dự báo chi phí 3 tháng tới
aws ce get-cost-forecast \
  --time-period Start=2024-02-01,End=2024-05-01 \
  --metric BLENDED_COST \
  --granularity MONTHLY \
  --prediction-interval-level 80 \
  --query 'ForecastResultsByTime[*].{
    Period:TimePeriod.Start,
    Forecast:MeanValue,
    LowerBound:PredictionIntervalLowerBound,
    UpperBound:PredictionIntervalUpperBound
  }' \
  --output table
```

#### 3. RI & Savings Plans Utilization

```bash
# RI Utilization — % RI đang được dùng
aws ce get-reservation-utilization \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --query 'UtilizationsByTime[*].{
    Period:TimePeriod.Start,
    Utilization:Total.UtilizationPercentage,
    OnDemandEquivalent:Total.OnDemandCostOfRIHoursUsed,
    UnusedCost:Total.UnusedAmortizedUpfrontFeeForBillingPeriod
  }'

# SP Coverage — % spending được cover bởi SP
aws ce get-savings-plans-coverage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY
```

### Lọc Chi Phí Theo Nhiều Chiều

```bash
# Chi phí EC2 của team Backend tháng này
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --filter '{
    "And": [
      {"Dimensions": {"Key": "SERVICE", "Values": ["Amazon EC2"]}},
      {"Tags": {"Key": "Team", "Values": ["Backend"]}}
    ]
  }' \
  --metrics BlendedCost
```

---

## 💰 AWS Budgets {#aws-budgets}

### Tổng Quan

AWS Budgets cho phép đặt **ngưỡng chi phí** và nhận **alerts (cảnh báo)** hoặc kích hoạt **automated actions (hành động tự động)** khi chi phí gần đạt hoặc vượt ngưỡng.

**Phí:** 2 budgets đầu tiên miễn phí, sau đó $0.02/budget/ngày

### Các Loại Budget

| Loại                          | Mô Tả                                         |
| ----------------------------- | --------------------------------------------- |
| **Cost Budget**               | Giới hạn tổng chi phí theo tháng/quý/năm      |
| **Usage Budget**              | Giới hạn lượng sử dụng (EC2 giờ, S3 GB, ...)  |
| **RI Utilization Budget**     | Cảnh báo nếu RI utilization < X%              |
| **RI Coverage Budget**        | Cảnh báo nếu RI coverage < X%                 |
| **Savings Plans Utilization** | Cảnh báo nếu SP utilization < X%              |
| **Savings Plans Coverage**    | Cảnh báo nếu SP coverage < X%                 |

### Tạo Budget Với CLI

```bash
# Tạo Monthly Cost Budget với email alerts
aws budgets create-budget \
  --account-id "$(aws sts get-caller-identity --query Account --output text)" \
  --budget '{
    "BudgetName": "Monthly-EC2-Budget",
    "BudgetLimit": {"Amount": "5000", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "Service": ["Amazon EC2"]
    }
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {"SubscriptionType": "EMAIL", "Address": "devops@company.com"}
      ]
    },
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {"SubscriptionType": "SNS", "Address": "arn:aws:sns:us-east-1:123456789:cost-critical"}
      ]
    },
    {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 110,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {"SubscriptionType": "EMAIL", "Address": "cfo@company.com"}
      ]
    }
  ]'
```

### Budget Actions — Tự Động Phản Ứng

Budget Actions cho phép AWS **tự động thực hiện hành động** khi budget threshold đạt:

```bash
# Tạo Budget Action: Apply SCP khi vượt 100% budget
aws budgets create-budget-action \
  --account-id "123456789" \
  --budget-name "Monthly-EC2-Budget" \
  --notification-type ACTUAL \
  --action-type APPLY_SCP_POLICY \
  --action-threshold '{
    "ActionThresholdValue": 100,
    "ActionThresholdType": "PERCENTAGE"
  }' \
  --definition '{
    "ScpActionDefinition": {
      "PolicyId": "p-xxxxxxxxxxxx",
      "TargetIds": ["123456789"]
    }
  }' \
  --execution-role-arn "arn:aws:iam::123456789:role/BudgetActionsRole" \
  --approval-model AUTOMATIC
```

```
Budget Actions có thể:
├── APPLY_IAM_POLICY: Attach IAM policy để restrict permissions
├── APPLY_SCP_POLICY: Attach Service Control Policy (chặn service cụ thể)
└── RUN_SSM_DOCUMENTS: Chạy SSM document (ví dụ: stop EC2 instances)
```

### Multi-Team Budget Setup

```bash
#!/bin/bash
# Tạo budget cho mỗi team dựa trên Tag

TEAMS=("backend" "frontend" "data-engineering" "mlops")
MONTHLY_LIMITS=(2000 800 1500 3000)

for i in "${!TEAMS[@]}"; do
  TEAM="${TEAMS[$i]}"
  LIMIT="${MONTHLY_LIMITS[$i]}"
  
  aws budgets create-budget \
    --account-id "$(aws sts get-caller-identity --query Account --output text)" \
    --budget "{
      \"BudgetName\": \"Team-${TEAM}-Monthly\",
      \"BudgetLimit\": {\"Amount\": \"${LIMIT}\", \"Unit\": \"USD\"},
      \"TimeUnit\": \"MONTHLY\",
      \"BudgetType\": \"COST\",
      \"CostFilters\": {
        \"TagKeyValue\": [\"user:Team\$${TEAM}\"]
      }
    }" \
    --notifications-with-subscribers "[{
      \"Notification\": {
        \"NotificationType\": \"ACTUAL\",
        \"ComparisonOperator\": \"GREATER_THAN\",
        \"Threshold\": 80,
        \"ThresholdType\": \"PERCENTAGE\"
      },
      \"Subscribers\": [
        {\"SubscriptionType\": \"EMAIL\", \"Address\": \"${TEAM}@company.com\"}
      ]
    }]"
  
  echo "Created budget for team: $TEAM (limit: $LIMIT)"
done
```

---

## 🔔 Cost Anomaly Detection {#anomaly-detection}

### Tổng Quan

AWS Cost Anomaly Detection sử dụng **Machine Learning** để tự động phát hiện chi phí bất thường (cost spikes — tăng đột biến) mà không cần tự đặt ngưỡng.

**Phí:** Miễn phí cho dịch vụ này, chỉ trả cho Cost Explorer API calls.

### Cách Hoạt Động

```
Machine Learning Model:
1. Học "normal" spending patterns (chu kỳ hàng ngày, hàng tuần, theo mùa)
2. Phát hiện khi chi phí thực tế deviation (lệch) đáng kể
3. Alert khi anomaly vượt threshold bạn đặt (tuyệt đối $ hoặc % impact)

Ưu điểm so với Budget alerts:
├── Tự động điều chỉnh baseline (không cần update thủ công)
├── Phát hiện anomaly trong ngày (không phải chỉ cuối tháng)
└── Xác định root cause: service nào, resource nào gây ra
```

### Tạo Anomaly Monitor & Subscription

```bash
# Tạo Anomaly Monitor cho toàn bộ account
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "Account-Wide-Monitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }' \
  --query 'MonitorArn'

# Tạo Monitor cho một service cụ thể (EC2)
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "EC2-Monitor",
    "MonitorType": "CUSTOM",
    "MonitorSpecification": "{\"And\": [{\"Dimensions\": {\"Key\": \"SERVICE\", \"Values\": [\"Amazon EC2\"]}}]}"
  }'

# Tạo Subscription nhận alert qua SNS
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "Daily-Anomaly-Alert",
    "MonitorArnList": ["arn:aws:ce::123456789:anomalymonitor/..."],
    "Subscribers": [
      {
        "Address": "arn:aws:sns:us-east-1:123456789:cost-alerts",
        "Type": "SNS"
      }
    ],
    "Threshold": 100,
    "Frequency": "DAILY"
  }'
```

### Xem Anomalies Đã Phát Hiện

```bash
# Xem anomalies trong 30 ngày qua
aws ce get-anomalies \
  --date-interval Start=2024-01-01,End=2024-02-01 \
  --query 'Anomalies[*].{
    AnomalyId:AnomalyId,
    Service:RootCauses[0].Service,
    Region:RootCauses[0].Region,
    Impact:AnomalyScore.MaxScore,
    TotalImpact:Impact.TotalImpact,
    StartDate:AnomalyStartDate,
    EndDate:AnomalyEndDate
  }' \
  --output table
```

---

## 🎯 AWS Trusted Advisor {#trusted-advisor}

### Tổng Quan

Trusted Advisor (Cố Vấn Đáng Tin Cậy) phân tích AWS environment theo **5 pillar** (trụ cột) và đưa ra recommendations:

1. **Cost Optimization (Tối Ưu Chi Phí)**
2. **Performance (Hiệu Suất)**
3. **Security (Bảo Mật)**
4. **Fault Tolerance (Dung Sai Lỗi)**
5. **Service Limits (Giới Hạn Dịch Vụ)**

**Phí:** Basic checks miễn phí cho mọi account. Full access cần Business/Enterprise Support.

### Cost Optimization Checks

```
Trusted Advisor Cost Checks (Kiểm Tra Chi Phí):

✅ Low Utilization Amazon EC2 Instances
   → EC2 instances có CPU < 10% avg và Network < 5 MB trong 14 ngày

✅ Idle Load Balancers
   → Load Balancers không có requests trong 14 ngày

✅ Underutilized Amazon EBS Volumes
   → EBS volumes không attach hoặc IOPS rất thấp

✅ Unassociated Elastic IP Addresses
   → Elastic IPs không gắn với running instance ($0.005/giờ/IP)

✅ Amazon RDS Idle DB Instances
   → RDS không có connection trong 7 ngày

✅ Reserved Instance Optimization
   → Gợi ý mua RI dựa trên On-Demand usage

✅ Amazon Route 53 Latency Resource Record Sets
   → Route 53 latency records không cần thiết
```

### Tích Hợp Trusted Advisor Với Automation

```python
import boto3

def get_trusted_advisor_cost_checks():
    """Lấy tất cả Trusted Advisor cost optimization recommendations"""
    
    support = boto3.client('support', region_name='us-east-1')
    
    # Lấy tất cả checks
    checks = support.describe_trusted_advisor_checks(language='en')
    
    cost_checks = [
        check for check in checks['checks']
        if check['category'] == 'cost_optimizing'
    ]
    
    results = []
    for check in cost_checks:
        result = support.describe_trusted_advisor_check_result(
            checkId=check['id'],
            language='en'
        )
        
        check_result = result['result']
        if check_result['status'] in ['warning', 'error']:
            results.append({
                'name': check['name'],
                'status': check_result['status'],
                'estimated_savings': check_result.get('estimatedMonthlySavings', 0),
                'flagged_resources': len(check_result.get('flaggedResources', []))
            })
    
    # Sort by estimated savings
    results.sort(key=lambda x: x['estimated_savings'], reverse=True)
    
    for r in results:
        print(f"{r['name']}: {r['flagged_resources']} resources, "
              f"save ${r['estimated_savings']:.2f}/month")
    
    return results

get_trusted_advisor_cost_checks()
```

---

## 🏷️ Tagging Strategy — Chiến Lược Gán Nhãn {#tagging}

### Tại Sao Tagging Quan Trọng?

Không có tags → Không thể biết:
- Service nào tốn bao nhiêu tiền?
- Team nào chịu trách nhiệm chi phí?
- Environment nào (prod/staging/dev) đang dùng bao nhiêu?

### Tag Taxonomy (Phân Loại Tag) Được Khuyến Nghị

```
Mandatory Tags (Tag Bắt Buộc):
├── Environment: production | staging | development | testing
├── Team: backend | frontend | data-engineering | mlops
├── Project: payment-service | user-service | analytics-platform
├── Owner: nguyenvan@company.com
└── CostCenter: CC-001 | CC-002 (mã trung tâm chi phí)

Recommended Tags (Tag Được Khuyến Nghị):
├── Application: api-gateway | worker | database
├── Tier: web | app | data
├── AutoShutdown: true | false (dùng cho Lambda tự tắt dev instances)
└── Backup: daily | weekly | none
```

### Enforce Tagging với AWS Config

```json
// AWS Config Rule: Yêu cầu tags bắt buộc trên EC2 instances
{
  "ConfigRuleName": "required-tags",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "REQUIRED_TAGS"
  },
  "InputParameters": {
    "tag1Key": "Environment",
    "tag2Key": "Team",
    "tag3Key": "Project",
    "tag4Key": "Owner"
  },
  "Scope": {
    "ComplianceResourceTypes": [
      "AWS::EC2::Instance",
      "AWS::EC2::Volume",
      "AWS::RDS::DBInstance",
      "AWS::Lambda::Function",
      "AWS::ECS::Service"
    ]
  }
}
```

### Tag Policies với AWS Organizations

```json
// Tag Policy để enforce chuẩn hóa tag values
{
  "tags": {
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": ["production", "staging", "development", "testing"]
      },
      "enforced_for": {
        "@@assign": ["ec2:instance", "ec2:volume", "rds:db"]
      }
    },
    "Team": {
      "tag_key": {
        "@@assign": "Team"
      },
      "tag_value": {
        "@@assign": ["backend", "frontend", "data-engineering", "mlops", "platform"]
      }
    }
  }
}
```

### Tự Động Gán Tags

```python
import boto3

def auto_tag_new_instances(event, context):
    """Lambda triggered by EventBridge khi EC2 instance mới được tạo"""
    
    ec2 = boto3.client('ec2')
    
    # Lấy thông tin từ CloudTrail event
    instance_id = event['detail']['responseElements']['instancesSet']['items'][0]['instanceId']
    requester_arn = event['detail']['userIdentity']['arn']
    requester_email = extract_email_from_arn(requester_arn)
    
    # Gán tag Owner tự động
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[
            {'Key': 'Owner', 'Value': requester_email},
            {'Key': 'LaunchDate', 'Value': event['time'][:10]},
            {'Key': 'CreatedBy', 'Value': 'auto-tagger'}
        ]
    )
    
    # Kiểm tra mandatory tags
    response = ec2.describe_instances(InstanceIds=[instance_id])
    instance = response['Reservations'][0]['Instances'][0]
    existing_tags = {tag['Key']: tag['Value'] for tag in instance.get('Tags', [])}
    
    mandatory = ['Environment', 'Team', 'Project']
    missing = [tag for tag in mandatory if tag not in existing_tags]
    
    if missing:
        # Gửi cảnh báo qua SNS
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789:tagging-alerts',
            Subject=f'Missing Tags: {instance_id}',
            Message=f'Instance {instance_id} launched by {requester_email} is missing tags: {missing}'
        )
    
    return {'instance_id': instance_id, 'missing_tags': missing}
```

---

## 💲 Cost Allocation Tags {#cost-allocation}

### Kích Hoạt Cost Allocation Tags

```bash
# Xem tags chưa được kích hoạt
aws ce list-cost-allocation-tags \
  --status Inactive \
  --type UserDefined

# Kích hoạt tags để dùng trong Cost Explorer filtering
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status '[
    {"TagKey": "Environment", "Status": "Active"},
    {"TagKey": "Team", "Status": "Active"},
    {"TagKey": "Project", "Status": "Active"},
    {"TagKey": "CostCenter", "Status": "Active"}
  ]'

# Lưu ý: Mất 24 giờ để tags xuất hiện trong Cost Explorer
# Chỉ áp dụng cho costs phát sinh SAU khi activate
```

### Chargeback vs Showback

```
Chargeback (Tính Phí Ngược): Chi phí thực sự được charge cho từng team/project.
  → Team A phải trả cho IT $2,000/tháng cho infrastructure của họ.
  → Khuyến khích trách nhiệm tài chính.

Showback (Hiển Thị Chi Phí): Chỉ hiển thị chi phí để awareness, không charge thực.
  → Team A thấy họ dùng $2,000/tháng nhưng IT vẫn trả.
  → Tốt cho bước đầu FinOps, ít friction (ma sát) hơn.

Recommendation: Bắt đầu với Showback → Dần chuyển sang Chargeback
khi culture FinOps đã hình thành.
```

---

## 📄 AWS Cost and Usage Report — CUR {#cur}

### Tổng Quan

CUR (Cost and Usage Report — Báo Cáo Chi Phí và Sử Dụng) là bộ data chi tiết nhất về chi phí AWS, được lưu vào S3 và có thể query bằng Athena.

```bash
# Tạo CUR delivery to S3
aws cur put-report-definition \
  --report-definition '{
    "ReportName": "detailed-cost-report",
    "TimeUnit": "DAILY",
    "Format": "Parquet",
    "Compression": "Parquet",
    "AdditionalSchemaElements": ["RESOURCES"],
    "S3Bucket": "my-cost-reports-bucket",
    "S3Prefix": "cost-reports",
    "S3Region": "us-east-1",
    "AdditionalArtifacts": ["ATHENA"],
    "RefreshClosedReports": true,
    "ReportVersioning": "OVERWRITE_REPORT"
  }'
```

### Query CUR Với Athena

```sql
-- Tạo Athena table từ CUR data
CREATE EXTERNAL TABLE cost_report (
  bill_billing_period_start_date STRING,
  line_item_product_code STRING,
  line_item_usage_type STRING,
  line_item_unblended_cost DOUBLE,
  resource_tags_user_team STRING,
  resource_tags_user_environment STRING
)
STORED AS PARQUET
LOCATION 's3://my-cost-reports-bucket/cost-reports/';

-- Top 10 services theo chi phí tháng này
SELECT 
  line_item_product_code AS service,
  SUM(line_item_unblended_cost) AS total_cost,
  resource_tags_user_team AS team
FROM cost_report
WHERE bill_billing_period_start_date = '2024-01-01'
GROUP BY line_item_product_code, resource_tags_user_team
ORDER BY total_cost DESC
LIMIT 10;

-- Chi phí EC2 theo team
SELECT 
  resource_tags_user_team AS team,
  SUM(line_item_unblended_cost) AS ec2_cost
FROM cost_report
WHERE line_item_product_code = 'AmazonEC2'
  AND bill_billing_period_start_date = '2024-01-01'
GROUP BY resource_tags_user_team
ORDER BY ec2_cost DESC;
```

---

## 📈 Dashboard & Báo Cáo Tổng Hợp {#dashboard}

### Cost Monitoring Weekly Checklist (Danh Sách Kiểm Tra Hàng Tuần)

```
Every Monday Morning (Mỗi Thứ Hai Sáng):
├── 🔴 Xem Cost Anomaly Detection alerts từ tuần trước
├── 🟡 Check EC2 instances với CPU < 5% trong 7 ngày
├── 🟡 Xem Trusted Advisor: Idle Load Balancers, unattached EBS
├── 🟢 Review RI/SP Utilization (mục tiêu: > 90%)
└── 🟢 Check Savings Plans Coverage (mục tiêu: > 80%)

Every Month (Đầu Mỗi Tháng):
├── Review chi phí theo team/project, so sánh với budget
├── Xem Compute Optimizer recommendations
├── Evaluate new instance families (có thể migrate sang gen mới?)
├── Review Reserved Instances sắp hết hạn trong 90 ngày
└── Update Savings Plans nếu workload thay đổi đáng kể

Every Quarter (Mỗi Quý):
├── Deep-dive cost review với các team
├── Review và update tagging compliance
├── Đánh giá kiến trúc có cơ hội migrate sang cheaper service không?
└── Benchmark chi phí so với industry standards
```

### Tạo CloudWatch Dashboard Cho Cost Metrics

```python
import boto3
import json

def create_cost_dashboard():
    """Tạo CloudWatch dashboard theo dõi billing metrics"""
    
    cw = boto3.client('cloudwatch')
    
    dashboard_body = {
        "widgets": [
            {
                "type": "metric",
                "properties": {
                    "title": "Monthly Estimated Charges",
                    "metrics": [
                        ["AWS/Billing", "EstimatedCharges", "Currency", "USD"]
                    ],
                    "period": 86400,
                    "stat": "Maximum",
                    "view": "timeSeries",
                    "region": "us-east-1"
                }
            },
            {
                "type": "metric",
                "properties": {
                    "title": "Charges By Service",
                    "metrics": [
                        ["AWS/Billing", "EstimatedCharges", "Currency", "USD", "ServiceName", "Amazon EC2"],
                        ["...", "ServiceName", "AWS Lambda"],
                        ["...", "ServiceName", "Amazon ECS"],
                        ["...", "ServiceName", "Amazon RDS"]
                    ],
                    "period": 86400,
                    "stat": "Maximum",
                    "view": "timeSeries"
                }
            }
        ]
    }
    
    cw.put_dashboard(
        DashboardName='Cost-Monitoring',
        DashboardBody=json.dumps(dashboard_body)
    )
    
    print("Cost monitoring dashboard created!")

create_cost_dashboard()
```

### Cost Monitoring Maturity Model (Mô Hình Trưởng Thành Giám Sát Chi Phí)

```
Level 1 — BASIC (Cơ Bản):
  ✅ Monthly budget alerts
  ✅ Xem Cost Explorer hàng tháng
  ✅ Tags trên resources chính

Level 2 — STANDARD (Tiêu Chuẩn):
  ✅ All Level 1
  ✅ Cost Anomaly Detection
  ✅ Team-level budgets với alerts
  ✅ Weekly cost review process
  ✅ RI/SP utilization tracking

Level 3 — ADVANCED (Nâng Cao):
  ✅ All Level 2
  ✅ CUR + Athena for deep analysis
  ✅ Chargeback/Showback implemented
  ✅ Automated rightsizing reports
  ✅ Cost per feature/request tracking
  ✅ FinOps culture với accountability

Level 4 — ELITE (Xuất Sắc):
  ✅ All Level 3
  ✅ Real-time cost dashboards
  ✅ ML-based cost forecasting
  ✅ Automated remediation
  ✅ Unit economics (cost per transaction, per user)
```

---

## 🎤 Câu Hỏi Phỏng Vấn {#câu-hỏi-phỏng-vấn}

**Q: Làm sao bạn xây dựng cost visibility cho một startup growing fast?**
> (1) Ngay lập tức: Bật Cost Anomaly Detection và Budget alerts (email khi > 80% budget). (2) Tuần 1: Gán mandatory tags (Environment, Team, Project) cho tất cả resources. (3) Tháng 1: Kích hoạt Cost Allocation Tags, bắt đầu Showback report mỗi tuần. (4) Tháng 2-3: Implement Budget Actions để tự động restrict nếu cost explodes. (5) Sau khi culture hình thành: Chuyển sang Chargeback để các team tự chịu trách nhiệm.

**Q: Khi nào bạn sẽ dùng Cost Anomaly Detection thay vì Budgets?**
> Dùng cả hai. Budget alerts tốt cho ngưỡng tuyệt đối ("cảnh báo khi > $5,000/tháng"). Anomaly Detection tốt cho phát hiện bất thường tương đối ("chi phí EC2 tăng 300% so với tuần trước" — dù vẫn dưới budget). Anomaly Detection dùng ML nên tự động adjust theo growth — không cần update threshold thủ công khi công ty scale.

**Q: Một junior dev vô tình tạo ra Lambda infinite loop tốn $10,000 trong một ngày. Bạn ngăn chặn điều này thế nào?**
> (1) Phát hiện: Cost Anomaly Detection sẽ alert trong vòng vài giờ. (2) Ngăn chặn: Lambda có thể set reserved concurrency = 0 để throttle ngay. (3) Dài hạn: Budget Action tự động apply IAM policy restrict Lambda invocations khi cost vượt ngưỡng. (4) Prevention (Phòng Ngừa): Code review kiểm tra recursive calls, set timeout hợp lý, dùng Lambda dead letter queue cho error handling.

**Q: Làm sao implement FinOps trong team hiện tại?**
> (1) Start với Showback — hàng tuần gửi report chi phí cho mỗi team để awareness. (2) Educate engineers về cloud cost — training về pricing, Savings Plans. (3) Đặt cost targets cho mỗi sprint/quarter. (4) Celebrate savings — public recognition khi team optimize thành công. (5) Embed cost trong design reviews — "solution này tốn bao nhiêu/tháng?" là câu hỏi bắt buộc. (6) Dần dần chuyển từ Showback sang Chargeback khi culture đã hình thành.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
