# 🔍 AWS X-Ray Distributed Tracing — Theo Dõi Phân Tán

> Hướng dẫn toàn diện về AWS X-Ray để theo dõi request xuyên qua nhiều dịch vụ AWS, tìm bottleneck (điểm nghẽn) và debug lỗi trong kiến trúc event-driven phân tán.

## 📚 Mục Lục

1. [Tại Sao Cần Distributed Tracing](#tại-sao-cần-distributed-tracing)
2. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
3. [Tích Hợp Với Các Dịch Vụ AWS](#tích-hợp-với-các-dịch-vụ-aws)
4. [Bật X-Ray Cho Lambda](#bật-x-ray-cho-lambda)
5. [Instrument Code Thủ Công](#instrument-code-thủ-công)
6. [X-Ray Với SQS và SNS](#x-ray-với-sqs-và-sns)
7. [X-Ray Với Kinesis và Step Functions](#x-ray-với-kinesis-và-step-functions)
8. [Service Map — Bản Đồ Dịch Vụ](#service-map)
9. [Sampling Rules — Quy Tắc Lấy Mẫu](#sampling-rules)
10. [Phân Tích Trace và Debug](#phân-tích-trace-và-debug)

---

## Tại Sao Cần Distributed Tracing

Trong kiến trúc monolithic (nguyên khối), khi có lỗi, bạn xem log của một service duy nhất. Trong microservices với messaging, một request có thể đi qua:

```
User Request
    ↓
API Gateway (5ms)
    ↓
Lambda: order-validator (15ms)
    ↓
SQS: order-queue (0ms — gửi vào queue)
    ↓ [bất đồng bộ — asynchronous]
Lambda: order-processor (250ms ← ĐÂY LÀ BOTTLENECK)
    ↓
DynamoDB: orders table (8ms)
    ↓
SNS: order-confirmed (3ms)
    ↓
Lambda: email-sender (45ms)
```

**Vấn đề:** Nếu user phàn nàn "đặt hàng chậm", bạn cần biết **bước nào** chậm. Không có tracing, bạn phải xem log từng service riêng lẻ, tốn nhiều giờ. Với X-Ray, bạn thấy ngay Lambda `order-processor` chiếm 250ms.

---

## Khái Niệm Cốt Lõi

### Trace (Dấu Vết)

**Trace** là toàn bộ hành trình của một request từ đầu đến cuối, được xác định bởi một **Trace ID** (Mã Dấu Vết) duy nhất.

```
Trace ID: 1-64abc123-xyz789...
Tổng thời gian: 326ms
Trạng thái: Thành công
```

### Segment (Phân Đoạn)

**Segment** là phần xử lý trong một service cụ thể — mỗi service tạo ra một segment trong trace.

```
Segment: Lambda order-processor
  Start time: 2026-05-18T10:00:00.250Z
  End time:   2026-05-18T10:00:00.500Z
  Duration:   250ms
  HTTP status: 200
  Error: false
```

### Subsegment (Phân Đoạn Con)

**Subsegment** là chi tiết bên trong một segment — ví dụ gọi DynamoDB, gọi external API.

```
Segment: Lambda order-processor (250ms)
  └── Subsegment: DynamoDB PutItem (8ms)
  └── Subsegment: SNS Publish (3ms)
  └── Subsegment: Validate order logic (239ms ← vấn đề ở đây)
```

### Annotation và Metadata

- **Annotation** (Chú Thích): Key-value pairs có thể dùng để filter (lọc) trace — dùng cho thông tin quan trọng như `orderId`, `customerId`.
- **Metadata** (Siêu Dữ Liệu): Thông tin chi tiết hơn, không dùng để filter — dùng cho debugging.

```python
from aws_xray_sdk.core import xray_recorder

# Annotation — có thể filter trong X-Ray console
xray_recorder.current_segment().put_annotation('orderId', order_id)
xray_recorder.current_segment().put_annotation('orderStatus', 'processing')

# Metadata — không filter được nhưng lưu chi tiết hơn
xray_recorder.current_segment().put_metadata('orderDetails', {
    'items': order['items'],
    'total': order['total'],
    'shippingAddress': order['address']
})
```

### Sampling (Lấy Mẫu)

Không thể trace 100% request trong production vì chi phí và overhead. **Sampling** xác định tỷ lệ request được trace:

```
Mặc định: 5% request/giây + 1 request đầu tiên mỗi giây (reservoir)
Tùy chỉnh: Tạo Sampling Rules (Quy Tắc Lấy Mẫu) theo service, path, method
```

---

## Tích Hợp Với Các Dịch Vụ AWS

### Dịch Vụ Hỗ Trợ X-Ray Tự Động

| Dịch Vụ | Hỗ Trợ | Cách Bật |
|---|---|---|
| **AWS Lambda** | ✅ Native | Bật trong function configuration |
| **API Gateway** | ✅ Native | Bật trong stage settings |
| **AWS Step Functions** | ✅ Native | Bật trong state machine config |
| **AWS AppSync** | ✅ Native | Bật trong API settings |
| **Amazon ECS** | ✅ Sidecar | Chạy X-Ray daemon container |
| **Amazon EC2** | ✅ Agent | Cài X-Ray daemon |
| **SQS** | ⚠️ Partial | Propagate trace header thủ công |
| **SNS** | ⚠️ Partial | Propagate trace header thủ công |
| **Kinesis** | ⚠️ Partial | Propagate trace header thủ công |

### Trace Header — Đầu Mô Tả Dấu Vết

X-Ray dùng header `X-Amzn-Trace-Id` để truyền trace ID qua các service:

```
X-Amzn-Trace-Id: Root=1-64abc123-xyz789;Parent=53995c3f42cd8ad8;Sampled=1
```

- **Root**: Trace ID gốc
- **Parent**: Segment ID của service cha
- **Sampled**: 1 = trace này đang được ghi, 0 = không ghi

---

## Bật X-Ray Cho Lambda

### Cách 1: AWS Console

```
Lambda Console → Function → Configuration → Monitoring and operations tools
→ Active tracing: Enable
```

### Cách 2: AWS CLI

```bash
aws lambda update-function-configuration \
  --function-name order-processor \
  --tracing-config Mode=Active
```

### Cách 3: CloudFormation / SAM

```yaml
# SAM template
Resources:
  OrderProcessor:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handler.process
      Tracing: Active  # PassThrough hoặc Active
      Policies:
        - AWSXRayDaemonWriteAccess  # Quyền ghi vào X-Ray
```

### Cách 4: Terraform

```hcl
resource "aws_lambda_function" "order_processor" {
  function_name = "order-processor"
  
  tracing_config {
    mode = "Active"  # Hoặc "PassThrough"
  }
}

# Cấp quyền cho Lambda để ghi trace
resource "aws_iam_role_policy_attachment" "xray" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
}
```

---

## Instrument Code Thủ Công

Để có subsegment chi tiết và annotation, cần instrument (đo đạc) code thủ công.

### Python với AWS X-Ray SDK

```bash
pip install aws-xray-sdk
```

```python
import boto3
from aws_xray_sdk.core import xray_recorder, patch_all

# Patch tất cả AWS SDK calls tự động tạo subsegment
patch_all()

@xray_recorder.capture('process_order')  # Tạo subsegment với tên
def process_order(order_id: str, order_data: dict):
    # Thêm annotation để filter sau này
    xray_recorder.current_subsegment().put_annotation('orderId', order_id)
    xray_recorder.current_subsegment().put_annotation('orderType', order_data['type'])
    
    # Thêm metadata cho debugging chi tiết
    xray_recorder.current_subsegment().put_metadata('orderData', order_data)
    
    # Subsegment tự động tạo cho DynamoDB call (do patch_all())
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('orders')
    table.put_item(Item={'orderId': order_id, **order_data})
    
    # Tạo subsegment thủ công cho logic phức tạp
    with xray_recorder.in_subsegment('validate-inventory') as subsegment:
        is_available = check_inventory(order_data['items'])
        subsegment.put_annotation('inventoryAvailable', is_available)
        if not is_available:
            subsegment.add_error_flag()  # Đánh dấu lỗi
            raise Exception("Hàng tồn kho không đủ")
    
    return {'status': 'processed', 'orderId': order_id}
```

### Node.js với AWS X-Ray SDK

```javascript
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));  // Patch AWS SDK

exports.handler = async (event) => {
    const segment = AWSXRay.getSegment();
    
    // Tạo subsegment thủ công
    const subsegment = segment.addNewSubsegment('process-order');
    
    try {
        const orderId = event.orderId;
        subsegment.addAnnotation('orderId', orderId);
        
        // DynamoDB call tự động được trace (do captureAWS)
        const dynamodb = new AWS.DynamoDB.DocumentClient();
        await dynamodb.put({
            TableName: 'orders',
            Item: { orderId, ...event }
        }).promise();
        
        subsegment.close();
        return { status: 'success' };
        
    } catch (error) {
        subsegment.addError(error);
        subsegment.close(error);
        throw error;
    }
};
```

---

## X-Ray Với SQS và SNS

SQS và SNS không tự động propagate (truyền) trace header. Cần xử lý thủ công.

### Producer: Gửi Trace ID Vào SQS Message

```python
import boto3
from aws_xray_sdk.core import xray_recorder

sqs = boto3.client('sqs')

def send_order_to_queue(order_data: dict):
    # Lấy trace header hiện tại
    segment = xray_recorder.current_segment()
    trace_header = f"Root={segment.trace_id};Parent={segment.id};Sampled=1"
    
    response = sqs.send_message(
        QueueUrl='https://sqs.ap-southeast-1.amazonaws.com/123456789/order-queue',
        MessageBody=json.dumps(order_data),
        MessageAttributes={
            # Gửi trace header như message attribute
            'X-Amzn-Trace-Id': {
                'DataType': 'String',
                'StringValue': trace_header
            }
        }
    )
    return response
```

### Consumer: Đọc Trace ID Từ SQS Message

```python
from aws_xray_sdk.core import xray_recorder

def process_sqs_message(event, context):
    for record in event['Records']:
        # Đọc trace header từ message attributes
        trace_header = record.get('messageAttributes', {}).get(
            'X-Amzn-Trace-Id', {}
        ).get('stringValue')
        
        if trace_header:
            # Tiếp tục trace từ producer
            # X-Ray SDK tự động xử lý khi Lambda active tracing bật
            # Chỉ cần log trace ID để correlate (tương quan) sau
            print(f"Processing message with trace: {trace_header}")
        
        # Xử lý message bình thường với subsegment
        with xray_recorder.in_subsegment('process-sqs-record') as sub:
            order_id = json.loads(record['body'])['orderId']
            sub.put_annotation('orderId', order_id)
            do_process(record['body'])
```

### Lưu Ý Về SQS Trace Propagation

```
Giới hạn: SQS không tự động liên kết trace giữa producer và consumer.
Mỗi Lambda invocation từ SQS trigger tạo trace MỚI.

Cách giải quyết:
1. Gửi trace ID trong message body hoặc attribute (như ví dụ trên)
2. Log trace ID ở producer và consumer để manual correlation
3. Dùng correlation ID (ID tương quan) trong business logic

Tương lai: AWS đang cải thiện native SQS trace propagation
```

---

## X-Ray Với Kinesis và Step Functions

### Step Functions — Tự Động Tracing

Step Functions hỗ trợ X-Ray native. Bật trong state machine definition:

```json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:order-validator",
      "Next": "ProcessPayment"
    }
  },
  "TracingConfiguration": {
    "Enabled": true
  }
}
```

Với tracing bật, X-Ray tự động tạo trace cho:
- Mỗi lần thực thi (execution) state machine
- Mỗi state transition (chuyển đổi trạng thái)
- Mỗi Lambda invocation từ Task state

### Kinesis Consumer Với X-Ray

```python
from aws_xray_sdk.core import xray_recorder, patch_all

patch_all()

def process_kinesis_records(event, context):
    for record in event['Records']:
        # Kinesis record không có trace header
        # Tạo subsegment cho mỗi record
        shard_id = record['kinesis']['partitionKey']
        sequence = record['kinesis']['sequenceNumber']
        
        with xray_recorder.in_subsegment('kinesis-record') as sub:
            sub.put_annotation('shardId', shard_id)
            sub.put_annotation('sequenceNumber', sequence)
            
            data = base64.b64decode(record['kinesis']['data'])
            process_record(data)
```

---

## Service Map — Bản Đồ Dịch Vụ

**Service Map** (Bản Đồ Dịch Vụ) trong X-Ray Console hiển thị trực quan các service và mối liên hệ giữa chúng, kèm health status và latency.

### Đọc Service Map

```
Màu sắc của node:
- Xanh lá: OK — service đang hoạt động tốt
- Vàng: Chậm — latency cao hơn bình thường
- Đỏ: Lỗi — error rate cao

Mũi tên (edge):
- Độ dày = volume (khối lượng) request
- Màu = trạng thái sức khỏe của kết nối

Ví dụ Service Map tốt:
API Gateway → [xanh] → Lambda → [xanh] → DynamoDB

Ví dụ Service Map có vấn đề:
API Gateway → [vàng] → Lambda → [đỏ] → DynamoDB
                                          ↑
                        DynamoDB đang chậm hoặc throttle
```

### X-Ray Insights (Thông Tin Chuyên Sâu)

**X-Ray Insights** tự động phát hiện anomaly (bất thường) và tạo insight:

```
Ví dụ Insight:
"Lambda function order-processor error rate tăng từ 0.1% lên 5.2%
bắt đầu từ 10:15 AM. Root cause: DynamoDB ProvisionedThroughputExceededException.
Ảnh hưởng: 847 request thất bại."
```

---

## Sampling Rules — Quy Tắc Lấy Mẫu

Sampling Rules (Quy Tắc Lấy Mẫu) tùy chỉnh xác định tỷ lệ trace theo điều kiện:

```json
{
  "SamplingRule": {
    "RuleName": "HighPriorityOrders",
    "Priority": 1,
    "FixedRate": 1.0,
    "ReservoirSize": 100,
    "ServiceName": "order-processor",
    "ServiceType": "AWS::Lambda::Function",
    "Host": "*",
    "HTTPMethod": "*",
    "URLPath": "*",
    "ResourceARN": "*",
    "Attributes": {
      "orderType": "premium"
    }
  }
}
```

### Chiến Lược Sampling Cho Production

```
Rule 1: Premium orders — trace 100%
  Priority: 1, FixedRate: 1.0, Attribute: orderType=premium

Rule 2: Error paths — trace 100%  
  Priority: 2, FixedRate: 1.0, Attribute: error=true

Rule 3: Health checks — trace 0%
  Priority: 3, FixedRate: 0.0, URLPath: /health

Rule 4: Default — trace 5%
  Priority: 9999 (lowest), FixedRate: 0.05, ReservoirSize: 50

Kết quả: Chi phí thấp nhưng đủ dữ liệu để debug
```

### Tạo Sampling Rule Bằng CLI

```bash
aws xray create-sampling-rule --cli-input-json '{
  "SamplingRule": {
    "RuleName": "PremiumOrders",
    "Priority": 1,
    "FixedRate": 1.0,
    "ReservoirSize": 100,
    "ServiceName": "order-processor",
    "ServiceType": "AWS::Lambda::Function",
    "Host": "*",
    "HTTPMethod": "*",
    "URLPath": "*",
    "ResourceARN": "*",
    "Attributes": {}
  }
}'
```

---

## Phân Tích Trace và Debug

### Workflow Debug Với X-Ray

```
Bước 1: User báo cáo đặt hàng chậm

Bước 2: Mở X-Ray Console → Traces
  Filter: annotation.userId = "user123"
  Sắp xếp theo: Duration (lâu nhất trước)

Bước 3: Chọn trace lâu nhất
  Thấy: Tổng 2.5 giây
  Segment Lambda order-processor: 2.3 giây (bất thường — bình thường 100ms)

Bước 4: Mở rộng segment order-processor
  Subsegment: validate-inventory = 2.2 giây ← BOTTLENECK
  Subsegment: DynamoDB PutItem = 0.05 giây (bình thường)
  Subsegment: SNS Publish = 0.05 giây (bình thường)

Bước 5: Điều tra validate-inventory
  Xem code → Gọi external inventory API không có timeout
  Inventory service đang chậm → giải thích bottleneck

Bước 6: Fix
  Thêm timeout 200ms cho inventory API call
  Thêm circuit breaker (cầu dao)
  Deploy và verify qua X-Ray trace mới
```

### CloudWatch Logs Insights + X-Ray Correlation

```sql
-- Tìm tất cả log có trace ID của một request cụ thể
fields @timestamp, @message, @xrayTraceId
| filter @xrayTraceId = "1-64abc123-xyz789abc"
| sort @timestamp asc
| limit 100
```

### X-Ray API Để Automation

```python
import boto3

xray = boto3.client('xray')

# Lấy trace theo filter expression
response = xray.get_trace_summaries(
    StartTime=datetime(2026, 5, 18, 10, 0, 0),
    EndTime=datetime(2026, 5, 18, 11, 0, 0),
    FilterExpression='annotation.orderId = "order-12345"',
    Sampling=False  # Lấy tất cả, không sampling
)

for trace in response['TraceSummaries']:
    print(f"Trace ID: {trace['Id']}")
    print(f"Duration: {trace['Duration']}ms")
    print(f"Has Error: {trace.get('HasError', False)}")
    print(f"Has Fault: {trace.get('HasFault', False)}")
```

---

## Chi Phí X-Ray

```
Free Tier (Gói Miễn Phí):
- 100,000 traces ghi miễn phí/tháng
- 1,000,000 traces lấy và quét miễn phí/tháng

Sau Free Tier:
- Ghi trace: $5.00 per 1 triệu trace
- Lấy trace: $0.50 per 1 triệu trace

Ước tính:
- 10 triệu request/tháng, sampling 5% = 500,000 trace/tháng
- Chi phí ghi: (500,000 - 100,000) / 1,000,000 * $5 = $2/tháng
- Rất rẻ so với giá trị debug mang lại
```

---

## Tóm Tắt Best Practice

| Hành Động | Khuyến Nghị |
|---|---|
| **Sampling rate** | 5% mặc định + 100% cho critical paths |
| **Annotation** | Thêm orderId, userId, requestType cho mọi segment |
| **Subsegment** | Tạo cho mọi external call và business logic quan trọng |
| **Error handling** | Luôn gọi `subsegment.add_error_flag()` khi có lỗi |
| **SQS/SNS propagation** | Truyền trace ID trong message attribute |
| **Dashboard** | Bổ sung widget X-Ray error rate vào CloudWatch Dashboard |
| **Insights** | Bật X-Ray Insights để tự động phát hiện anomaly |
| **Retention** | Default 30 ngày — đủ cho hầu hết use case |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Xem Tiếp:** [4-dlq-monitoring.md](4-dlq-monitoring.md) — Giám Sát DLQ và Xử Lý Tin Nhắn Lỗi
