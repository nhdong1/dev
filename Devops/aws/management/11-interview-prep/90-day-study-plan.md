# Kế Hoạch Học 90 Ngày — AWS Management & Governance

> Kế hoạch có cấu trúc để đạt kỹ năng AWS Management & Governance ở mức **mid-level** (đủ để phỏng vấn Cloud Engineer / DevOps Engineer tự tin). Mỗi tuần có mục tiêu, tài nguyên, bài tập thực hành và checkpoint tự đánh giá.

---

## Tổng Quan 3 Tháng

```
Tháng 1 (Ngày 1–30):   NỀN TẢNG
  → CloudWatch, CloudTrail, AWS Config, Trusted Advisor
  → Mục tiêu: Giải thích được 3 dịch vụ cốt lõi, hiểu khi nào dùng cái nào

Tháng 2 (Ngày 31–60):  VẬN HÀNH
  → Systems Manager, CloudFormation, CDK, Organizations
  → Mục tiêu: Thiết kế và triển khai hạ tầng IaC, vận hành fleet EC2

Tháng 3 (Ngày 61–90):  QUẢN TRỊ & PHỎNG VẤN
  → Control Tower, Cost Governance, Interview Prep
  → Mục tiêu: Thiết kế landing zone, sẵn sàng phỏng vấn senior level
```

---

## THÁNG 1: NỀN TẢNG (Ngày 1–30)

### Tuần 1: CloudWatch — Quan Sát & Giám Sát (Ngày 1–7)

#### Mục Tiêu Tuần

- [ ] Hiểu Metrics (Chỉ Số), Namespaces (Không Gian Tên), Dimensions (Chiều)
- [ ] Tạo và cấu hình CloudWatch Alarms (Cảnh Báo)
- [ ] Dùng CloudWatch Logs Insights (Phân Tích Nhật Ký) để query
- [ ] Bật CloudWatch Agent trên EC2

#### Tài Nguyên Học

| Loại              | Tài Nguyên                                              | Thời Gian  |
| ----------------- | ------------------------------------------------------- | ---------- |
| Đọc               | `01-cloudwatch/1-metrics-namespaces.md`                 | 45 phút    |
| Đọc               | `01-cloudwatch/2-alarms-composite.md`                   | 45 phút    |
| Đọc               | `01-cloudwatch/3-logs-insights.md`                      | 60 phút    |
| Đọc               | `01-cloudwatch/6-cloudwatch-agent.md`                   | 45 phút    |
| Lab               | AWS Free Tier: Tạo alarm CPU > 70% cho EC2 t2.micro     | 2 giờ      |
| Lab               | Viết Logs Insights query lọc ERROR logs                 | 1 giờ      |

#### Bài Tập Thực Hành

```bash
# Lab 1: Tạo Custom Metric
aws cloudwatch put-metric-data \
  --namespace "MyApp/Custom" \
  --metric-name "ActiveUsers" \
  --value 150 \
  --unit Count

# Lab 2: Query Logs Insights
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50

# Lab 3: Tạo Alarm qua CLI
aws cloudwatch put-metric-alarm \
  --alarm-name "high-cpu-alarm" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:my-topic
```

#### Checkpoint Cuối Tuần

- [ ] Giải thích được Composite Alarm là gì và khi nào dùng
- [ ] Tạo được alarm từ Console và CLI
- [ ] Query được CloudWatch Logs Insights
- [ ] Biết sự khác nhau giữa Metric Filter và Logs Insights

---

### Tuần 2: CloudTrail — Kiểm Toán (Ngày 8–14)

#### Mục Tiêu Tuần

- [ ] Phân biệt Management Events, Data Events, Insights Events
- [ ] Tạo Trail ghi vào S3 và CloudWatch Logs
- [ ] Điều tra sự kiện API qua Event History
- [ ] Hiểu Organization Trail cho multi-account

#### Tài Nguyên Học

| Loại   | Tài Nguyên                                              | Thời Gian |
| ------ | ------------------------------------------------------- | --------- |
| Đọc    | `02-cloudtrail/1-event-types.md`                        | 45 phút   |
| Đọc    | `02-cloudtrail/2-trails-configuration.md`               | 60 phút   |
| Đọc    | `02-cloudtrail/5-forensics-investigation.md`            | 60 phút   |
| Lab    | Tạo Trail → filter DeleteBucket events → SNS alert      | 2 giờ     |
| Lab    | Dùng Athena query CloudTrail logs trong S3              | 2 giờ     |

#### Bài Tập Thực Hành

```bash
# Lab: Tìm ai tạo Security Group trong 24h qua
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateSecurityGroup \
  --start-time $(date -d "24 hours ago" +%Y-%m-%dT%H:%M:%S) \
  --query 'Events[*].{User:Username,Time:EventTime,Detail:CloudTrailEvent}'

# Lab: Athena query - top 10 IAM users gọi API nhiều nhất hôm nay
SELECT useridentity.arn, COUNT(*) as call_count
FROM cloudtrail_logs
WHERE DATE(eventtime) = CURRENT_DATE
GROUP BY useridentity.arn
ORDER BY call_count DESC
LIMIT 10;
```

#### Checkpoint Cuối Tuần

- [ ] Tìm được "ai đã làm gì" từ CloudTrail trong dưới 5 phút
- [ ] Phân biệt được khi nào dùng Event History vs Trail
- [ ] Biết Data Events cần bật riêng và tính phí thêm
- [ ] Giải thích được Organization Trail và lợi ích

---

### Tuần 3: AWS Config — Tuân Thủ (Ngày 15–21)

#### Mục Tiêu Tuần

- [ ] Hiểu Configuration Recorder và Configuration Items
- [ ] Tạo và test Config Rules (Managed + Custom)
- [ ] Cấu hình Auto-Remediation với SSM Automation
- [ ] Hiểu Conformance Packs

#### Tài Nguyên Học

| Loại   | Tài Nguyên                                              | Thời Gian |
| ------ | ------------------------------------------------------- | --------- |
| Đọc    | `03-aws-config/1-configuration-recorder.md`             | 45 phút   |
| Đọc    | `03-aws-config/2-config-rules.md`                       | 60 phút   |
| Đọc    | `03-aws-config/4-remediation.md`                        | 60 phút   |
| Lab    | Bật Config → tạo rule `s3-bucket-versioning-enabled`    | 1.5 giờ   |
| Lab    | Tạo Custom Lambda Config Rule kiểm tra EC2 tag          | 3 giờ     |

#### Custom Config Rule Mẫu

```python
import boto3
import json

def evaluate_compliance(configuration_item, rule_parameters):
    """Kiểm tra EC2 instance có tag 'Environment' không"""
    if configuration_item['resourceType'] != 'AWS::EC2::Instance':
        return 'NOT_APPLICABLE'
    
    tags = {tag['key']: tag['value'] 
            for tag in configuration_item.get('tags', [])}
    
    if 'Environment' not in tags:
        return 'NON_COMPLIANT'
    
    return 'COMPLIANT'

def lambda_handler(event, context):
    config = boto3.client('config')
    invoking_event = json.loads(event['invokingEvent'])
    configuration_item = invoking_event['configurationItem']
    
    compliance = evaluate_compliance(configuration_item, {})
    
    config.put_evaluations(
        Evaluations=[{
            'ComplianceResourceType': configuration_item['resourceType'],
            'ComplianceResourceId': configuration_item['resourceId'],
            'ComplianceType': compliance,
            'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
        }],
        ResultToken=event['resultToken']
    )
```

#### Checkpoint Cuối Tuần

- [ ] Phân biệt được CloudTrail vs Config (câu hỏi kinh điển phỏng vấn)
- [ ] Tạo được Config Rule và xem compliance status
- [ ] Viết Custom Config Rule Lambda đơn giản
- [ ] Biết Conformance Pack là gì và dùng khi nào

---

### Tuần 4: Trusted Advisor & Health Dashboard (Ngày 22–28)

#### Mục Tiêu Tuần

- [ ] Biết 5 category của Trusted Advisor
- [ ] Tự động hóa Trusted Advisor checks qua Support API
- [ ] Phân biệt Service Health vs Personal Health Dashboard
- [ ] Tích hợp Health Events với EventBridge

#### Tài Nguyên Học

| Loại   | Tài Nguyên                                              | Thời Gian |
| ------ | ------------------------------------------------------- | --------- |
| Đọc    | `08-trusted-advisor/1-check-categories.md`              | 45 phút   |
| Đọc    | `08-trusted-advisor/2-programmatic-access.md`           | 45 phút   |
| Đọc    | `09-health-dashboard/1-personal-health.md`              | 30 phút   |
| Đọc    | `09-health-dashboard/2-eventbridge-integration.md`      | 45 phút   |
| Lab    | EventBridge rule: Health event → SNS → Email            | 1.5 giờ   |

#### Checkpoint Cuối Tuần (Tháng 1 Review)

**Tự kiểm tra — trả lời không nhìn tài liệu:**

- [ ] CloudTrail vs Config vs CloudWatch — khi nào dùng cái nào?
- [ ] Trusted Advisor Security category kiểm tra gì?
- [ ] CloudWatch Composite Alarm là gì?
- [ ] Config Remediation Action hoạt động thế nào?

---

## THÁNG 2: VẬN HÀNH (Ngày 31–60)

### Tuần 5–6: AWS Systems Manager — SSM (Ngày 31–44)

#### Mục Tiêu Hai Tuần

- [ ] Thiết lập Session Manager, không cần SSH/bastion
- [ ] Cấu hình Patch Manager với Patch Groups và Maintenance Windows
- [ ] Dùng Parameter Store cho config và secret
- [ ] Viết và chạy SSM Automation Document

#### Tài Nguyên Học

| Loại   | Tài Nguyên                                           | Thời Gian |
| ------ | ---------------------------------------------------- | --------- |
| Đọc    | `04-systems-manager/1-session-manager.md`            | 60 phút   |
| Đọc    | `04-systems-manager/2-patch-manager.md`              | 60 phút   |
| Đọc    | `04-systems-manager/3-parameter-store.md`            | 60 phút   |
| Đọc    | `04-systems-manager/4-run-command-automation.md`     | 60 phút   |
| Lab    | Setup Session Manager: EC2 không có public IP        | 2 giờ     |
| Lab    | Tạo Patch Baseline custom cho Amazon Linux 2         | 1.5 giờ   |
| Lab    | SSM Automation: restart service khi alarm triggered  | 3 giờ     |

#### Lab: Session Manager Setup

```bash
# Bước 1: Tạo IAM Role cho EC2
aws iam create-role \
  --role-name EC2-SSM-Role \
  --assume-role-policy-document file://ec2-trust-policy.json

aws iam attach-role-policy \
  --role-name EC2-SSM-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Bước 2: Launch EC2 với IAM Role, không cần key pair, không cần public IP
# (Instance phải có SSM Agent — Amazon Linux 2 có sẵn)

# Bước 3: Kết nối qua CLI
aws ssm start-session --target i-1234567890abcdef0

# Bước 4: Port forwarding (tunnel localhost:8080 → instance:80)
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["80"],"localPortNumber":["8080"]}'
```

#### Lab: Parameter Store Hierarchy

```bash
# Lưu parameter
aws ssm put-parameter \
  --name "/myapp/production/database/password" \
  --value "MyS3cr3tP@ssword" \
  --type "SecureString" \
  --key-id "alias/myapp-key"

# Đọc parameter trong ứng dụng (Python)
import boto3
ssm = boto3.client('ssm', region_name='us-east-1')
response = ssm.get_parameter(
    Name='/myapp/production/database/password',
    WithDecryption=True
)
db_password = response['Parameter']['Value']
```

#### Checkpoint Cuối Tuần 5–6

- [ ] Thiết lập Session Manager thành công, không cần port 22
- [ ] Giải thích được Patch Baseline vs Patch Group vs Maintenance Window
- [ ] Phân biệt Parameter Store vs Secrets Manager
- [ ] Viết SSM Automation Document đơn giản

---

### Tuần 7–8: CloudFormation & CDK (Ngày 45–60)

#### Mục Tiêu Hai Tuần

- [ ] Viết CloudFormation template YAML từ đầu
- [ ] Hiểu và dùng Change Sets trước khi update stack
- [ ] Triển khai StackSets lên 2 accounts
- [ ] Viết CDK app đơn giản bằng Python

#### Tài Nguyên Học

| Loại   | Tài Nguyên                                              | Thời Gian |
| ------ | ------------------------------------------------------- | --------- |
| Đọc    | `05-cloudformation/1-template-anatomy.md`               | 90 phút   |
| Đọc    | `05-cloudformation/2-stacks-lifecycle.md`               | 60 phút   |
| Đọc    | `05-cloudformation/4-change-sets-drift.md`              | 60 phút   |
| Đọc    | `05-cloudformation/3-stacksets-multiregion.md`          | 60 phút   |
| Đọc    | `05-cloudformation/6-cdk-comparison.md`                 | 45 phút   |
| Lab    | Viết template deploy VPC + EC2 + Security Group         | 4 giờ     |
| Lab    | Dùng Change Set để update Stack an toàn                 | 1.5 giờ   |
| Lab    | CDK Python: tạo S3 bucket + Lambda + EventBridge rule   | 3 giờ     |

#### Template CloudFormation Mẫu

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'EC2 instance với Security Group và CloudWatch Alarm'

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: Loại EC2 instance

  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]

Conditions:
  IsProd: !Equals [!Ref Environment, prod]

Resources:
  WebServerSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server security group
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Environment
          Value: !Ref Environment

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !Sub '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2}}'
      IamInstanceProfile: !Ref InstanceProfile
      SecurityGroupIds:
        - !Ref WebServerSG
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Name
          Value: !Sub '${Environment}-web-server'

  CPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Condition: IsProd
    Properties:
      AlarmDescription: CPU cao trong production
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: InstanceId
          Value: !Ref WebServer

Outputs:
  InstanceId:
    Value: !Ref WebServer
    Export:
      Name: !Sub '${AWS::StackName}-InstanceId'
```

#### CDK Python Mẫu

```python
from aws_cdk import (
    Stack,
    aws_s3 as s3,
    aws_lambda as lambda_,
    aws_events as events,
    aws_events_targets as targets,
    RemovalPolicy, Duration
)
from constructs import Construct

class MyAppStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs):
        super().__init__(scope, construct_id, **kwargs)

        # S3 bucket với versioning và encryption
        bucket = s3.Bucket(
            self, "AppBucket",
            versioned=True,
            encryption=s3.BucketEncryption.S3_MANAGED,
            removal_policy=RemovalPolicy.RETAIN
        )

        # Lambda function
        handler = lambda_.Function(
            self, "AppHandler",
            runtime=lambda_.Runtime.PYTHON_3_12,
            code=lambda_.Code.from_asset("lambda"),
            handler="index.handler",
            timeout=Duration.seconds(30),
            environment={"BUCKET_NAME": bucket.bucket_name}
        )

        # Cấp quyền
        bucket.grant_read_write(handler)

        # EventBridge rule: chạy mỗi giờ
        rule = events.Rule(
            self, "HourlyRule",
            schedule=events.Schedule.rate(Duration.hours(1))
        )
        rule.add_target(targets.LambdaFunction(handler))
```

#### Checkpoint Cuối Tuần 7–8

- [ ] Viết CloudFormation template có Parameters, Conditions, Outputs
- [ ] Dùng Change Set và hiểu tại sao cần trước khi update production stack
- [ ] Giải thích Drift Detection và khi nào xảy ra
- [ ] Viết CDK app cơ bản và `cdk synth` ra CloudFormation template

---

## THÁNG 3: QUẢN TRỊ & PHỎNG VẤN (Ngày 61–90)

### Tuần 9–10: Organizations, Control Tower, Cost (Ngày 61–76)

#### Mục Tiêu Hai Tuần

- [ ] Thiết kế OU structure cho doanh nghiệp giả định
- [ ] Hiểu SCP deny list vs allow list và inheritance
- [ ] Biết Control Tower Landing Zone gồm những gì
- [ ] Thiết kế cost governance với Budgets + Anomaly Detection + Tagging

#### Tài Nguyên Học

| Loại   | Tài Nguyên                                              | Thời Gian |
| ------ | ------------------------------------------------------- | --------- |
| Đọc    | `06-organizations/1-account-structure.md`               | 60 phút   |
| Đọc    | `06-organizations/2-scp-policies.md`                    | 90 phút   |
| Đọc    | `06-organizations/5-multi-account-patterns.md`          | 60 phút   |
| Đọc    | `07-control-tower/1-landing-zone-setup.md`              | 60 phút   |
| Đọc    | `07-control-tower/2-guardrails.md`                      | 60 phút   |
| Đọc    | `10-cost-governance/1-budgets-alerts.md`                | 45 phút   |
| Đọc    | `10-cost-governance/3-anomaly-detection.md`             | 45 phút   |
| Đọc    | `10-cost-governance/4-tagging-strategy.md`              | 45 phút   |
| Bài tập | Thiết kế OU structure cho công ty 5 team               | 2 giờ     |

#### Bài Tập Thiết Kế OU Structure

```
Yêu cầu giả định:
- Công ty FinTech, 5 team: Frontend, Backend, Data, Security, DevOps
- Compliance: PCI-DSS (tất cả), HIPAA (chỉ Data team)
- Môi trường: dev, staging, prod mỗi team
- Budget: giới hạn $5,000/tháng/team dev, không giới hạn prod

Thiết kế OU và SCPs của bạn:
Root
├── OU: Security          → [SCP: deny disable CloudTrail, deny leave org]
│   ├── Log Archive       → centralized logs
│   └── Security Tooling  → GuardDuty master, Security Hub master
├── OU: Infrastructure    → [SCP: deny create default VPC]
│   └── Shared Services   → Transit Gateway, Directory Services
├── OU: Workloads
│   ├── OU: Production    → [SCP: deny: rds:Delete*, ec2:TerminateInstances without approval tag]
│   │   ├── Prod-Frontend
│   │   ├── Prod-Backend
│   │   └── Prod-Data     → [SCP thêm: HIPAA controls]
│   └── OU: Non-Production → [SCP: require tag Environment, deny expensive instance types]
│       ├── Dev-Frontend
│       ├── Dev-Backend
│       └── Dev-Data
└── OU: Sandbox           → [SCP: budget limit, deny prod-grade services, 30-day auto-expire]
```

#### Checkpoint Cuối Tuần 9–10

- [ ] Giải thích SCP Allow List vs Deny List và khi nào dùng cái nào
- [ ] Biết Landing Zone gồm: Log Archive, Audit, Management accounts + Guardrails
- [ ] Thiết kế được cost governance: Budget + Alert + Tagging Policy
- [ ] Phân biệt Organizations vs Control Tower (câu hỏi phỏng vấn rất hay gặp)

---

### Tuần 11–12: Interview Preparation (Ngày 77–90)

#### Mục Tiêu Hai Tuần

- [ ] Đọc và luyện INTERVIEW_GUIDE.md (30 câu Q&A)
- [ ] Luyện 3 system design scenarios
- [ ] Chuẩn bị 2 câu chuyện STAR từ kinh nghiệm thực tế
- [ ] Mock interview tự thực hành

#### Lịch Luyện Tập

```
Ngày 77–79: INTERVIEW_GUIDE.md
  - Ngày 77: Q1–Q10 (CloudWatch + CloudTrail)
  - Ngày 78: Q11–Q20 (Config + Organizations + SSM)
  - Ngày 79: Q21–Q30 (CloudFormation + Cost + Control Tower)

Ngày 80–83: system-design-scenarios.md
  - Ngày 80: S1 — Observability platform (vẽ diagram)
  - Ngày 81: S2 — Multi-account landing zone
  - Ngày 82: S3 — Compliance automation pipeline
  - Ngày 83: S4–S6 review nhanh

Ngày 84–86: star-stories.md
  - Ngày 84: Đọc templates và examples
  - Ngày 85: Viết 2 story từ kinh nghiệm cá nhân
  - Ngày 86: Luyện kể không nhìn giấy (time: 3–4 phút/story)

Ngày 87–89: service-comparison.md
  - Ôn lại tất cả bảng so sánh
  - Đảm bảo giải thích được không cần xem tài liệu

Ngày 90: Mock Interview
  - Không nhìn tài liệu
  - Trả lời 10 câu ngẫu nhiên từ INTERVIEW_GUIDE.md
  - Tự chấm điểm theo rubric bên dưới
```

#### Rubric Tự Chấm Điểm Mock Interview

| Tiêu Chí                          | 1 (Yếu)                      | 2 (Đạt)                        | 3 (Tốt)                         |
| --------------------------------- | ---------------------------- | ------------------------------ | -------------------------------- |
| Chính xác kỹ thuật                | Sai thông tin cơ bản         | Đúng nhưng thiếu chi tiết      | Đúng + chi tiết + ví dụ         |
| Cấu trúc câu trả lời              | Lan man, không có structure  | Có structure nhưng chưa rõ     | Rõ ràng, mạch lạc, đúng trọng tâm |
| Trade-off awareness               | Không nhắc đến trade-off     | Nhắc đến 1 trade-off           | So sánh các lựa chọn + lý do   |
| Ví dụ thực tế                     | Không có ví dụ               | Ví dụ mơ hồ                    | Ví dụ cụ thể với số liệu        |
| Tự tin / Tốc độ                   | Dài dòng, ngập ngừng nhiều   | Ổn nhưng có chỗ ngập ngừng    | Trả lời tự nhiên, đúng thời gian |

**Mục tiêu:** Trung bình ≥ 2.5 điểm trên tất cả 10 câu → sẵn sàng phỏng vấn

---

## Milestone Tracker

### Cuối Tháng 1 (Ngày 30)

- [ ] Giải thích được CloudTrail vs Config vs CloudWatch trong 2 phút
- [ ] Tạo được CloudWatch Alarm và Logs Insights query
- [ ] Tạo được Config Rule và xem compliance
- [ ] Biết Trusted Advisor 5 categories

### Cuối Tháng 2 (Ngày 60)

- [ ] Thiết lập Session Manager thay thế bastion host hoàn toàn
- [ ] Viết CloudFormation template có Conditions, Outputs, Cross-stack references
- [ ] Hiểu Change Set và Drift Detection
- [ ] Thiết kế được multi-account structure trên giấy

### Cuối Tháng 3 (Ngày 90)

- [ ] Trả lời được 28/30 câu trong INTERVIEW_GUIDE.md
- [ ] Vẽ diagram system design trong 20 phút không nhìn tài liệu
- [ ] Kể được 2 câu chuyện STAR mạch lạc
- [ ] Điểm mock interview ≥ 2.5/3

---

## Tips Học Hiệu Quả

### Phương Pháp Spaced Repetition (Lặp Lại Phân Tán)

```
Lần 1 học: ngày N
Review lần 1: ngày N+1
Review lần 2: ngày N+3
Review lần 3: ngày N+7
Review lần 4: ngày N+14
Review lần 5: ngày N+30
```

Dùng Anki hoặc tự tạo flashcard cho các khái niệm chính.

### Active Recall (Nhớ Lại Chủ Động)

Sau mỗi section, đóng tài liệu và viết ra:
- 3 điều quan trọng nhất vừa học
- 2 điều chưa chắc, cần review
- 1 câu hỏi phỏng vấn liên quan

### Feynman Technique (Kỹ Thuật Feynman)

Giải thích khái niệm như đang dạy cho người không biết gì về AWS. Nếu không giải thích được đơn giản → chưa thực sự hiểu.

### Lab-First Learning

Mỗi khái niệm mới: đọc 30 phút → làm lab 60 phút → review 15 phút. Tỷ lệ đọc:làm = 1:2.

---

## Tài Nguyên Bổ Sung

### AWS Documentation (Miễn Phí)

- AWS CloudWatch User Guide
- AWS CloudTrail User Guide
- AWS Config Developer Guide
- AWS Systems Manager User Guide

### AWS Labs & Workshops (Miễn Phí)

- AWS Well-Architected Labs — Operations Excellence
- AWS Observability Workshop
- AWS Control Tower Workshop

### Certification Liên Quan

Sau khi hoàn thành 90 ngày này, bạn sẽ sẵn sàng cho:
- **AWS Certified SysOps Administrator - Associate** (SysOps-C02)
- **AWS Certified DevOps Engineer - Professional** (DOP-C02) — cần thêm CI/CD knowledge

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
