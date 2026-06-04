# 🔔 Cấu Hình CloudWatch Alarms — Cảnh Báo CloudWatch

> Hướng dẫn thiết lập CloudWatch Alarms (Cảnh Báo CloudWatch) cho hệ thống AWS Application Integration, bao gồm ngưỡng khuyến nghị, best practice và các pattern phổ biến.

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#khái-niệm-cơ-bản)
2. [Alarm States — Trạng Thái Cảnh Báo](#alarm-states)
3. [Alarm Cho SQS](#alarm-cho-sqs)
4. [Alarm Cho Kinesis](#alarm-cho-kinesis)
5. [Alarm Cho Step Functions](#alarm-cho-step-functions)
6. [Alarm Cho SNS và EventBridge](#alarm-cho-sns-và-eventbridge)
7. [Composite Alarms — Cảnh Báo Kết Hợp](#composite-alarms)
8. [Kết Nối Alarm Với Hành Động](#kết-nối-alarm-với-hành-động)
9. [Chiến Lược Ngưỡng Thông Minh](#chiến-lược-ngưỡng-thông-minh)
10. [Ví Dụ Terraform và CDK](#ví-dụ-terraform-và-cdk)

---

## Khái Niệm Cơ Bản

**CloudWatch Alarm** (Cảnh Báo CloudWatch) giám sát một metric và thực hiện hành động khi metric vượt ngưỡng xác định trong một khoảng thời gian.

### Các Thành Phần Của Một Alarm

```
Alarm = Metric + Threshold + Period + Evaluation Period + Action

Ví dụ:
- Metric: SQS ApproximateNumberOfMessagesVisible
- Threshold (Ngưỡng): > 1000
- Period (Chu Kỳ Đo): 60 giây
- Evaluation Periods (Số Chu Kỳ Đánh Giá): 3 liên tiếp
- Action: Gửi SNS notification → Slack/PagerDuty
→ Alarm kích hoạt khi queue > 1000 trong 3 phút liên tiếp
```

### Datapoints to Alarm (Số Điểm Dữ Liệu Cần Để Báo Động)

```
M out of N evaluation periods (M trong N chu kỳ đánh giá):

Ví dụ "2 out of 3":
Period 1: metric = 1500 (BREACH — vượt ngưỡng)
Period 2: metric = 1200 (BREACH)
Period 3: metric = 800  (OK)
→ 2/3 breach → ALARM kích hoạt

"3 out of 3" (chặt hơn — tránh false positive):
→ Chỉ báo động khi vượt ngưỡng 3 lần liên tiếp
→ Ít nhạy hơn nhưng ít false alarm (cảnh báo nhầm) hơn
```

---

## Alarm States

Một alarm có ba trạng thái:

| Trạng Thái | Ý Nghĩa |
|---|---|
| **OK** | Metric đang trong ngưỡng bình thường |
| **ALARM** | Metric đã vượt ngưỡng — cần hành động |
| **INSUFFICIENT_DATA** | Chưa đủ dữ liệu để đánh giá (thường khi mới tạo alarm hoặc metric chưa có data) |

---

## Alarm Cho SQS

### 1. Queue Depth Alarm (Cảnh Báo Độ Sâu Hàng Đợi)

Báo động khi queue tích lũy quá nhiều tin nhắn chưa xử lý:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "SQS-QueueDepth-High" \
  --alarm-description "Queue depth vượt ngưỡng — consumer có thể bị chậm hoặc lỗi" \
  --metric-name ApproximateNumberOfMessagesVisible \
  --namespace AWS/SQS \
  --dimensions Name=QueueName,Value=my-order-queue \
  --statistic Average \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:alert-topic \
  --ok-actions arn:aws:sns:ap-southeast-1:123456789:alert-topic \
  --treat-missing-data notBreaching
```

### 2. Message Age Alarm (Cảnh Báo Tuổi Tin Nhắn)

Metric quan trọng hơn queue depth — phát hiện tin nhắn bị treo:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "SQS-OldestMessage-Critical" \
  --alarm-description "Tin nhắn cũ nhất trong queue đã quá 1 giờ — kiểm tra consumer ngay" \
  --metric-name ApproximateAgeOfOldestMessage \
  --namespace AWS/SQS \
  --dimensions Name=QueueName,Value=my-order-queue \
  --statistic Maximum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 3600 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:critical-alert-topic
```

### 3. DLQ Non-Empty Alarm (Cảnh Báo DLQ Có Tin Nhắn)

Ngay khi DLQ nhận tin nhắn đầu tiên phải báo động:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "SQS-DLQ-NotEmpty" \
  --alarm-description "DLQ có tin nhắn — có lỗi cần điều tra ngay" \
  --metric-name ApproximateNumberOfMessagesVisible \
  --namespace AWS/SQS \
  --dimensions Name=QueueName,Value=my-order-queue-dlq \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:critical-alert-topic \
  --treat-missing-data notBreaching
```

### Ngưỡng Khuyến Nghị Cho SQS

| Alarm | Ngưỡng | Mức Độ Nghiêm Trọng |
|---|---|---|
| Queue Depth > X | Phụ thuộc SLA — baseline × 10 | Warning (Cảnh Báo) |
| Queue Depth > 10X | Baseline × 100 | Critical (Khẩn Cấp) |
| Oldest Message > 1 giờ | 3,600 giây | Warning |
| Oldest Message > 6 giờ | 21,600 giây | Critical |
| DLQ Messages > 0 | 1 | Critical ngay lập tức |

---

## Alarm Cho Kinesis

### 1. Iterator Age Alarm (Cảnh Báo Tuổi Iterator)

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "Kinesis-IteratorAge-High" \
  --alarm-description "Consumer lag cao — Kinesis stream đang bị tụt hậu so với producer" \
  --metric-name GetRecords.IteratorAgeMilliseconds \
  --namespace AWS/Kinesis \
  --dimensions Name=StreamName,Value=my-event-stream \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 5 \
  --threshold 60000 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:alert-topic
```

### 2. Write Throttle Alarm (Cảnh Báo Ghi Bị Giới Hạn)

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "Kinesis-WriteThrottle" \
  --alarm-description "Ghi vào Kinesis bị throttle — cần thêm shard hoặc dùng Enhanced Fan-Out" \
  --metric-name WriteProvisionedThroughputExceeded \
  --namespace AWS/Kinesis \
  --dimensions Name=StreamName,Value=my-event-stream \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:alert-topic
```

### Ngưỡng Khuyến Nghị Cho Kinesis

| Alarm | Ngưỡng | Hành Động |
|---|---|---|
| Iterator Age > 60 giây | 60,000 ms | Kiểm tra consumer performance |
| Iterator Age > 5 phút | 300,000 ms | Scale consumer hoặc thêm shard |
| Write Throttle > 0 | 1 count | Thêm shard hoặc dùng batch write |
| Read Throttle > 0 | 1 count | Dùng Enhanced Fan-Out |

---

## Alarm Cho Step Functions

### 1. Execution Failure Alarm (Cảnh Báo Thực Thi Thất Bại)

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "StepFunctions-ExecutionsFailed" \
  --alarm-description "Có workflow thất bại trong 5 phút qua — kiểm tra execution history" \
  --metric-name ExecutionsFailed \
  --namespace AWS/States \
  --dimensions Name=StateMachineArn,Value=arn:aws:states:ap-southeast-1:123456789:stateMachine:OrderProcessing \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:alert-topic
```

### 2. Execution Duration Alarm (Cảnh Báo Thời Gian Thực Thi)

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "StepFunctions-SlowExecution" \
  --alarm-description "Workflow đang chạy chậm hơn SLA — P95 > 30 giây" \
  --metric-name ExecutionTime \
  --namespace AWS/States \
  --dimensions Name=StateMachineArn,Value=arn:aws:states:... \
  --extended-statistic p95 \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 30000 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:alert-topic
```

---

## Alarm Cho SNS và EventBridge

### SNS Delivery Failure Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "SNS-DeliveryFailed" \
  --alarm-description "SNS giao vận thất bại tăng cao" \
  --metric-name NumberOfNotificationsFailed \
  --namespace AWS/SNS \
  --dimensions Name=TopicName,Value=order-events-topic \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:alert-topic
```

### EventBridge Failed Invocations Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EventBridge-FailedInvocations" \
  --alarm-description "EventBridge rule target thất bại — event có thể bị mất" \
  --metric-name FailedInvocations \
  --namespace AWS/Events \
  --dimensions Name=RuleName,Value=process-order-rule \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:critical-alert-topic
```

---

## Composite Alarms — Cảnh Báo Kết Hợp

**Composite Alarm** (Cảnh Báo Kết Hợp) kết hợp nhiều alarm bằng logic AND/OR — giảm false positive và tạo cảnh báo có ngữ cảnh rõ ràng hơn.

### Ví Dụ: Phát Hiện Consumer Thực Sự Có Vấn Đề

```
Alarm A: SQS Queue Depth > 1000 (có thể do spike bình thường)
Alarm B: Lambda Errors > 0 (có lỗi)
Alarm C: Lambda Throttles > 0 (bị throttle)

Composite Alarm = A AND (B OR C)
→ Chỉ báo động khi queue sâu VÀ có vấn đề với consumer
→ Tránh báo nhầm khi chỉ có spike ngắn
```

```bash
aws cloudwatch put-composite-alarm \
  --alarm-name "SQS-ConsumerIssue-Composite" \
  --alarm-description "Queue depth cao kết hợp với lỗi hoặc throttle ở consumer" \
  --alarm-rule "ALARM(\"SQS-QueueDepth-High\") AND (ALARM(\"Lambda-Errors\") OR ALARM(\"Lambda-Throttles\"))" \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:critical-alert-topic
```

### Ví Dụ: Cảnh Báo Hệ Thống Đang Ổn Định

```bash
# Composite alarm cho biết toàn bộ pipeline đang OK
aws cloudwatch put-composite-alarm \
  --alarm-name "OrderPipeline-AllOK" \
  --alarm-rule "OK(\"SQS-QueueDepth-High\") AND OK(\"SQS-DLQ-NotEmpty\") AND OK(\"StepFunctions-ExecutionsFailed\")"
```

---

## Kết Nối Alarm Với Hành Động

### 1. Gửi Thông Báo Qua SNS

```
CloudWatch Alarm → SNS Topic → {
  Email subscription (thông báo qua email),
  SQS subscription (đưa vào ticket queue),
  Lambda subscription (xử lý tự động),
  HTTPS subscription (gọi webhook Slack/PagerDuty)
}
```

### 2. Auto Scaling Với EC2/ECS

```bash
# Ví dụ: Scale ECS service khi SQS queue depth cao
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/order-consumer \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name scale-based-on-sqs \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 100,
    "CustomizedMetricSpecification": {
      "MetricName": "ApproximateNumberOfMessagesVisible",
      "Namespace": "AWS/SQS",
      "Dimensions": [{"Name": "QueueName", "Value": "my-order-queue"}],
      "Statistic": "Average"
    }
  }'
```

### 3. Lambda Auto-Remediation (Tự Động Khắc Phục)

```python
# Lambda được trigger bởi SNS khi alarm kích hoạt
import boto3
import json

def handler(event, context):
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm_name = message['AlarmName']
    new_state = message['NewStateValue']
    
    if alarm_name == 'SQS-QueueDepth-High' and new_state == 'ALARM':
        # Tự động tăng Lambda reserved concurrency
        lambda_client = boto3.client('lambda')
        lambda_client.put_function_concurrency(
            FunctionName='order-consumer',
            ReservedConcurrentExecutions=100  # Tăng từ 10 lên 100
        )
        print("Đã tăng concurrency limit cho order-consumer")
```

---

## Chiến Lược Ngưỡng Thông Minh

### Anomaly Detection (Phát Hiện Bất Thường)

Thay vì ngưỡng tĩnh, dùng **Anomaly Detection** để AWS tự học pattern và báo động khi có bất thường:

```bash
# Tạo alarm dựa trên anomaly detection (phát hiện bất thường)
aws cloudwatch put-metric-alarm \
  --alarm-name "SQS-AnomalyDetection" \
  --alarm-description "Queue depth bất thường so với pattern lịch sử" \
  --metrics '[{
    "Id": "m1",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/SQS",
        "MetricName": "ApproximateNumberOfMessagesVisible",
        "Dimensions": [{"Name": "QueueName", "Value": "my-order-queue"}]
      },
      "Period": 60,
      "Stat": "Average"
    }
  }, {
    "Id": "ad1",
    "Expression": "ANOMALY_DETECTION_BAND(m1, 2)",
    "Label": "ApproximateNumberOfMessagesVisible (expected)"
  }]' \
  --comparison-operator GreaterThanUpperThreshold \
  --threshold-metric-id ad1 \
  --evaluation-periods 3
```

### Graduated Alerting (Cảnh Báo Phân Cấp)

Tạo hai mức alarm cho cùng một metric:

```
Level 1 — Warning (Cảnh Báo):
  SQS Oldest Message > 30 phút
  → Gửi Slack channel #alerts-warning
  → Không gọi điện, chỉ notify

Level 2 — Critical (Khẩn Cấp):
  SQS Oldest Message > 2 giờ
  → Gọi điện on-call engineer qua PagerDuty
  → Tạo incident ticket tự động
  → Escalate (leo thang) sau 15 phút không phản hồi
```

### Suppression Window (Cửa Sổ Tắt Alarm Tạm Thời)

Trong maintenance window (cửa sổ bảo trì), tắt alarm để tránh false positive:

```bash
# Tắt alarm trong 2 giờ bảo trì
aws cloudwatch disable-alarm-actions \
  --alarm-names "SQS-QueueDepth-High" "SQS-DLQ-NotEmpty"

# Sau bảo trì xong, bật lại
aws cloudwatch enable-alarm-actions \
  --alarm-names "SQS-QueueDepth-High" "SQS-DLQ-NotEmpty"
```

---

## Ví Dụ Terraform và CDK

### Terraform

```hcl
resource "aws_cloudwatch_metric_alarm" "sqs_dlq_not_empty" {
  alarm_name          = "SQS-DLQ-NotEmpty-${var.environment}"
  alarm_description   = "DLQ có tin nhắn - có lỗi cần điều tra ngay"
  
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  
  dimensions = {
    QueueName = aws_sqs_queue.dlq.name
  }
  
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 1
  threshold           = 0
  comparison_operator = "GreaterThanThreshold"
  
  alarm_actions = [aws_sns_topic.critical_alerts.arn]
  ok_actions    = [aws_sns_topic.critical_alerts.arn]
  
  treat_missing_data = "notBreaching"
  
  tags = {
    Environment = var.environment
    Service     = "order-processing"
  }
}
```

### AWS CDK (TypeScript)

```typescript
import * as cw from 'aws-cdk-lib/aws-cloudwatch';
import * as cw_actions from 'aws-cdk-lib/aws-cloudwatch-actions';
import * as sns from 'aws-cdk-lib/aws-sns';

// Tạo alarm cho DLQ
const dlqAlarm = new cw.Alarm(this, 'DLQNotEmptyAlarm', {
  alarmName: `SQS-DLQ-NotEmpty-${props.environment}`,
  alarmDescription: 'DLQ có tin nhắn - có lỗi cần điều tra ngay',
  
  metric: new cw.Metric({
    metricName: 'ApproximateNumberOfMessagesVisible',
    namespace: 'AWS/SQS',
    dimensionsMap: { QueueName: dlqQueue.queueName },
    statistic: 'Sum',
    period: cdk.Duration.minutes(1),
  }),
  
  evaluationPeriods: 1,
  threshold: 0,
  comparisonOperator: cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
  treatMissingData: cw.TreatMissingData.NOT_BREACHING,
});

// Kết nối với SNS topic để gửi thông báo
dlqAlarm.addAlarmAction(new cw_actions.SnsAction(criticalAlertTopic));
dlqAlarm.addOkAction(new cw_actions.SnsAction(criticalAlertTopic));

// Tạo dashboard
const dashboard = new cw.Dashboard(this, 'AppIntegrationDashboard', {
  dashboardName: `AppIntegration-${props.environment}`,
});

dashboard.addWidgets(
  new cw.GraphWidget({
    title: 'SQS Queue Depth',
    left: [
      new cw.Metric({
        metricName: 'ApproximateNumberOfMessagesVisible',
        namespace: 'AWS/SQS',
        dimensionsMap: { QueueName: queue.queueName },
      })
    ],
    width: 8,
  }),
  new cw.AlarmStatusWidget({
    title: 'Alarm Status',
    alarms: [dlqAlarm],
    width: 8,
  })
);
```

---

## Checklist Alarm Hoàn Chỉnh

### Trước Khi Deploy Production

- [ ] Alarm cho mọi SQS queue: depth, age, DLQ
- [ ] Alarm cho mọi Kinesis stream: iterator age, throttle
- [ ] Alarm cho mọi Step Functions state machine: failures, timeout
- [ ] Alarm cho SNS: delivery failure rate
- [ ] Alarm cho EventBridge: failed invocations
- [ ] Alarm cho Lambda consumer: errors, throttles, duration
- [ ] Composite alarm để giảm false positive
- [ ] Kiểm tra alarm hoạt động bằng cách kích hoạt thủ công
- [ ] Đảm bảo đúng người nhận được thông báo đúng kênh
- [ ] Có runbook (sổ tay) cho mỗi alarm — ai làm gì khi báo động

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Xem Tiếp:** [3-xray-tracing.md](3-xray-tracing.md) — Distributed Tracing với AWS X-Ray
