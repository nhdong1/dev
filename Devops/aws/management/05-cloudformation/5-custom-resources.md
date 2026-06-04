# Custom Resources — Mở Rộng CloudFormation Với Lambda

> **Custom Resource** (Tài Nguyên Tùy Chỉnh) là cơ chế cho phép bạn viết logic tùy chỉnh bằng Lambda hoặc SNS để thực thi trong quá trình CloudFormation tạo, cập nhật hoặc xóa stack. Dùng khi CloudFormation không hỗ trợ native một tài nguyên, API call hoặc tác vụ cụ thể.

---

## 📚 Mục Lục

1. [Khi Nào Dùng Custom Resource?](#khi-nào-dùng-custom-resource)
2. [Kiến Trúc Custom Resource](#kiến-trúc-custom-resource)
3. [Triển Khai Custom Resource Với Lambda](#triển-khai-custom-resource-với-lambda)
4. [Giao Thức Request/Response](#giao-thức-requestresponse)
5. [Các Pattern Thực Tế](#các-pattern-thực-tế)
6. [CloudFormation Registry — Resource Providers](#cloudformation-registry--resource-providers)
7. [Lưu Ý Quan Trọng & Anti-Patterns](#lưu-ý-quan-trọng--anti-patterns)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khi Nào Dùng Custom Resource?

### Use Cases Phổ Biến

| Use Case | Mô Tả |
|----------|-------|
| **Tài nguyên AWS chưa hỗ trợ** | Resource type mới AWS chưa có CloudFormation support |
| **API bên thứ ba** | Tạo DNS record trên Cloudflare, tạo repo trên Datadog |
| **Tác vụ tự động hóa** | Seed database, tạo Cognito user pool custom attributes |
| **Lookup / Data fetch** | Lấy thông tin động không dùng được Parameter/Mapping |
| **Validation phức tạp** | Kiểm tra điều kiện trước khi tạo tài nguyên |
| **Cross-account actions** | Thao tác trên account khác từ một template |
| **Secret rotation seed** | Tạo secret ban đầu và đẩy vào Secrets Manager |

### Khi Không Nên Dùng Custom Resource

```
❌ Tài nguyên đã có CloudFormation support → Dùng native resource type
❌ Chỉ cần lấy SSM Parameter → Dùng dynamic reference {{resolve:ssm:...}}
❌ Logic quá phức tạp, khó maintain → Cân nhắc CDK hoặc Terraform
❌ Long-running tasks (>15 phút) → Lambda timeout, cần giải pháp khác
```

---

## Kiến Trúc Custom Resource

### Luồng Tổng Quan

```
CloudFormation Stack Operation (CREATE/UPDATE/DELETE)
         │
         │ Request (HTTPS POST to ServiceToken)
         ▼
┌─────────────────────────────────┐
│     Lambda Function             │
│   (Custom Resource Handler)     │
│                                 │
│  1. Parse event (RequestType)   │
│  2. Execute custom logic        │
│  3. Call pre-signed S3 URL      │
│     (hoặc SNS) với response     │
└─────────────────────────────────┘
         │
         │ Response (PUT to CloudFormation pre-signed S3 URL)
         ▼
CloudFormation nhận kết quả
├── SUCCESS → Tiếp tục tạo/update stack
└── FAILED → Rollback stack
```

### Hai Loại Custom Resource

| Loại | ServiceToken | Mô Tả |
|-----|-------------|-------|
| **Lambda-backed** | ARN của Lambda function | Phổ biến nhất — code trực tiếp |
| **SNS-backed** | ARN của SNS topic | Cho long-running tasks, custom provider |

---

## Triển Khai Custom Resource Với Lambda

### Template CloudFormation

```yaml
Resources:
  # ========================
  # Lambda function handler
  # ========================
  CustomResourceFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub "${AWS::StackName}-custom-resource-handler"
      Runtime: python3.12
      Handler: index.handler
      Role: !GetAtt CustomResourceRole.Arn
      Timeout: 300             # 5 phút — đủ cho hầu hết use cases
      Code:
        ZipFile: |
          import json
          import urllib.request

          def handler(event, context):
              print(json.dumps(event))
              
              request_type = event['RequestType']
              physical_id = event.get('PhysicalResourceId', 'custom-resource-id')
              
              try:
                  if request_type == 'Create':
                      result = create_resource(event)
                      physical_id = result['Id']
                  elif request_type == 'Update':
                      result = update_resource(event)
                  elif request_type == 'Delete':
                      delete_resource(event)
                      result = {}
                  
                  send_response(event, context, 'SUCCESS', result, physical_id)
              except Exception as e:
                  print(f"Error: {str(e)}")
                  send_response(event, context, 'FAILED', {}, physical_id,
                               reason=str(e))

          def create_resource(event):
              props = event['ResourceProperties']
              # === Custom logic ở đây ===
              return {'Id': 'created-resource-id', 'Endpoint': 'https://...'}

          def update_resource(event):
              old_props = event.get('OldResourceProperties', {})
              new_props = event['ResourceProperties']
              # === Custom update logic ===
              return {}

          def delete_resource(event):
              physical_id = event['PhysicalResourceId']
              # === Custom delete logic ===
              pass

          def send_response(event, context, status, data, physical_id, reason=''):
              response_body = json.dumps({
                  'Status': status,
                  'Reason': reason or f'See CloudWatch Logs: {context.log_stream_name}',
                  'PhysicalResourceId': physical_id,
                  'StackId': event['StackId'],
                  'RequestId': event['RequestId'],
                  'LogicalResourceId': event['LogicalResourceId'],
                  'Data': data    # Key-value pairs có thể dùng !GetAtt
              }).encode('utf-8')
              
              req = urllib.request.Request(
                  url=event['ResponseURL'],
                  data=response_body,
                  method='PUT',
                  headers={'Content-Type': '', 'Content-Length': len(response_body)}
              )
              urllib.request.urlopen(req)

  # IAM Role cho Lambda
  CustomResourceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: CustomResourcePolicy
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - route53:CreateHealthCheck
                  - route53:DeleteHealthCheck
                  - ssm:PutParameter
                  - ssm:DeleteParameter
                Resource: "*"

  # ========================
  # Custom Resource Definition
  # ========================
  MyCustomResource:
    Type: Custom::MyCustomType    # Tên tùy ý, phải bắt đầu "Custom::"
    # Hoặc dùng: AWS::CloudFormation::CustomResource
    Properties:
      ServiceToken: !GetAtt CustomResourceFunction.Arn  # BẮT BUỘC
      # Tất cả properties khác sẽ được pass vào event.ResourceProperties
      ApiEndpoint: "https://api.example.com"
      ApiKey: "{{resolve:secretsmanager:api-key}}"
      Environment: !Ref Environment
      Region: !Ref AWS::Region

Outputs:
  # Lấy data từ Custom Resource response
  CustomEndpoint:
    Value: !GetAtt MyCustomResource.Endpoint    # Từ Data.Endpoint trong response
  
  CustomId:
    Value: !GetAtt MyCustomResource.Id          # Từ Data.Id trong response
```

---

## Giao Thức Request/Response

### Event Cấu Trúc (CloudFormation Gửi Đến Lambda)

```json
{
  "RequestType": "Create",          // Create | Update | Delete
  "ResponseURL": "https://s3.amazonaws.com/...",  // Pre-signed URL để gửi kết quả
  "StackId": "arn:aws:cloudformation:ap-southeast-1:123456789012:stack/...",
  "RequestId": "unique-request-id-abc123",
  "ResourceType": "Custom::MyCustomType",
  "LogicalResourceId": "MyCustomResource",
  "PhysicalResourceId": "my-resource-abc",  // Chỉ có khi Update/Delete
  "ResourceProperties": {
    "ServiceToken": "arn:aws:lambda:...",
    "ApiEndpoint": "https://api.example.com",
    "Environment": "prod"
  },
  "OldResourceProperties": {               // Chỉ có khi Update
    "ApiEndpoint": "https://old-api.example.com",
    "Environment": "staging"
  }
}
```

### Response Cấu Trúc (Lambda Gửi Về CloudFormation)

```json
{
  "Status": "SUCCESS",                     // SUCCESS | FAILED
  "Reason": "Lý do nếu FAILED",
  "PhysicalResourceId": "my-resource-abc", // ID ổn định qua các lần update
  "StackId": "arn:aws:cloudformation:...",
  "RequestId": "unique-request-id-abc123",
  "LogicalResourceId": "MyCustomResource",
  "NoEcho": false,                         // true = ẩn Data khỏi Console
  "Data": {
    "Endpoint": "https://created-endpoint.example.com",
    "Id": "resource-id-xyz",
    "Status": "active"
    // Có thể truy cập qua !GetAtt MyCustomResource.Endpoint
  }
}
```

### Physical Resource ID — Quan Trọng

```
PhysicalResourceId là định danh ổn định của resource qua vòng đời stack.

Quy tắc:
- CREATE: Tự tạo và trả về PhysicalResourceId mới
- UPDATE: Nếu trả về PhysicalResourceId khác → CloudFormation coi như REPLACE
           (sẽ gọi Delete cho ID cũ, create cho ID mới)
           Nếu trả về cùng PhysicalResourceId → coi như UPDATE tại chỗ
- DELETE: Nhận PhysicalResourceId từ event, dùng để xóa resource

→ Nếu không muốn resource bị recreate khi update: giữ nguyên PhysicalResourceId
```

---

## Các Pattern Thực Tế

### Pattern 1: Lookup AMI ID Mới Nhất

```python
import boto3

def handler(event, context):
    if event['RequestType'] == 'Delete':
        send_response(event, context, 'SUCCESS', {}, event['PhysicalResourceId'])
        return
    
    props = event['ResourceProperties']
    ssm = boto3.client('ssm')
    
    # Lấy AMI ID mới nhất từ SSM Parameter Store
    response = ssm.get_parameter(
        Name=f"/aws/service/ami-amazon-linux-latest/{props['AmiName']}"
    )
    ami_id = response['Parameter']['Value']
    
    send_response(event, context, 'SUCCESS',
                  {'AmiId': ami_id},
                  physical_id='ami-lookup')
```

```yaml
Resources:
  LatestAMILookup:
    Type: Custom::AMILookup
    Properties:
      ServiceToken: !GetAtt AMILookupFunction.Arn
      AmiName: amzn2-ami-hvm-x86_64-gp2

  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !GetAtt LatestAMILookup.AmiId   # Kết quả từ custom resource
```

### Pattern 2: Seed DynamoDB Table Sau Khi Tạo

```python
import boto3

def handler(event, context):
    if event['RequestType'] == 'Delete':
        send_response(event, context, 'SUCCESS', {}, event.get('PhysicalResourceId', 'seed'))
        return
    
    if event['RequestType'] == 'Create':
        props = event['ResourceProperties']
        dynamodb = boto3.resource('dynamodb')
        table = dynamodb.Table(props['TableName'])
        
        # Seed data ban đầu
        with table.batch_writer() as batch:
            for item in props.get('SeedItems', []):
                batch.put_item(Item=item)
    
    send_response(event, context, 'SUCCESS', {}, physical_id='table-seeder')
```

### Pattern 3: Gọi API Bên Thứ Ba (Datadog Monitor)

```python
import json
import urllib.request

def handler(event, context):
    props = event['ResourceProperties']
    api_key = props['DatadogApiKey']
    app_key = props['DatadogAppKey']
    
    if event['RequestType'] == 'Create':
        monitor_config = {
            "type": "metric alert",
            "query": props['Query'],
            "name": props['MonitorName'],
            "message": props['AlertMessage']
        }
        
        req = urllib.request.Request(
            'https://api.datadoghq.com/api/v1/monitor',
            data=json.dumps(monitor_config).encode(),
            headers={
                'DD-API-KEY': api_key,
                'DD-APPLICATION-KEY': app_key,
                'Content-Type': 'application/json'
            }
        )
        response = json.loads(urllib.request.urlopen(req).read())
        monitor_id = str(response['id'])
        send_response(event, context, 'SUCCESS',
                      {'MonitorId': monitor_id}, monitor_id)
    
    elif event['RequestType'] == 'Delete':
        monitor_id = event['PhysicalResourceId']
        req = urllib.request.Request(
            f'https://api.datadoghq.com/api/v1/monitor/{monitor_id}',
            method='DELETE',
            headers={
                'DD-API-KEY': api_key,
                'DD-APPLICATION-KEY': app_key
            }
        )
        urllib.request.urlopen(req)
        send_response(event, context, 'SUCCESS', {}, monitor_id)
```

### Pattern 4: Dùng crhelper Library (Khuyến Nghị)

```python
# pip install crhelper → đóng gói vào Lambda layer
from crhelper import CfnResource

helper = CfnResource(
    json_logging=True,
    log_level='DEBUG',
    boto_level='CRITICAL',
    sleep_on_delete=120    # Chờ trước khi delete để CloudFormation timeout xử lý
)

@helper.create
def create(event, context):
    props = event['ResourceProperties']
    # ... logic tạo resource
    helper.Data.update({'Id': 'resource-id', 'Endpoint': 'https://...'})
    return 'my-physical-resource-id'   # Return PhysicalResourceId

@helper.update
def update(event, context):
    props = event['ResourceProperties']
    old_props = event.get('OldResourceProperties', {})
    # ... logic update
    return event['PhysicalResourceId']  # Giữ nguyên ID

@helper.delete
def delete(event, context):
    physical_id = event['PhysicalResourceId']
    # ... logic xóa resource

def handler(event, context):
    helper(event, context)
```

---

## CloudFormation Registry — Resource Providers

**CloudFormation Registry** (Danh Mục CloudFormation) cho phép đăng ký resource types và modules tùy chỉnh, sau đó dùng như native resource types.

### Các Loại Extension

| Loại | Mô Tả | Ví Dụ |
|-----|-------|-------|
| **Resource Types** | Custom resource type dùng như native | `MyCompany::Networking::VPN` |
| **Modules** | Template fragment tái sử dụng | `MyCompany::Common::VPCSetup` |
| **Hooks** | Logic chạy trước/sau stack operations | `MyCompany::Security::Validator` |

### Resource Provider vs Custom Resource

| Tiêu Chí | Custom Resource | Resource Provider |
|---------|----------------|-------------------|
| **Cú pháp** | `Type: Custom::MyType` | `Type: MyCompany::MyService::MyResource` |
| **Complexity** | Đơn giản | Phức tạp (cần CloudFormation CLI) |
| **Reusability** | Per-stack | Có thể share qua Registry |
| **Schema validation** | Không | Có (JSON Schema) |
| **Drift detection** | Không tự động | Hỗ trợ nếu implement |
| **Phù hợp khi** | One-off tasks | Tái sử dụng nhiều, muốn native feel |

### Public Registry — Third-Party Resources

```yaml
# Dùng resource type từ Public Registry (ví dụ: Datadog)
Resources:
  DatadogMonitor:
    Type: Datadog::Monitors::Monitor
    Properties:
      Type: metric alert
      Query: "avg(last_5m):avg:system.cpu.user{*} > 90"
      Name: "High CPU Alert"
      Message: "CPU quá cao! @team-ops"
      Tags:
        - "env:prod"
        - "service:api"

# Lưu ý: Phải activate resource type trong CloudFormation Registry trước
```

---

## Lưu Ý Quan Trọng & Anti-Patterns

### Lưu Ý Quan Trọng

```
1. LUÔN gửi response về CloudFormation
   → Nếu không gửi, CloudFormation chờ 1 giờ rồi timeout → FAILED
   → Dùng try/except để đảm bảo gửi response dù có lỗi

2. Idempotency (Tính Bất Biến)
   → Cùng input → cùng output, bất kể gọi bao nhiêu lần
   → Create nên kiểm tra resource đã tồn tại chưa

3. PhysicalResourceId nhất quán
   → Update phải trả về cùng PhysicalResourceId trừ khi muốn Replace

4. Xử lý Delete gracefully
   → Resource có thể đã bị xóa thủ công → Không raise error nếu not found
   → Trả về SUCCESS ngay cả khi resource không tồn tại

5. Timeout
   → Lambda timeout tối đa 15 phút
   → CloudFormation timeout stack operation (riêng biệt)
   → Cho long-running tasks: dùng Step Functions thay Lambda

6. Response URL có thời hạn
   → Pre-signed S3 URL hết hạn sau khoảng 2 giờ
   → Đảm bảo Lambda hoàn thành và gửi response trước đó
```

### Anti-Patterns Cần Tránh

```python
# ❌ Anti-Pattern 1: Không xử lý exception
def handler(event, context):
    do_something()   # Nếu exception, response không được gửi → timeout
    send_response(event, context, 'SUCCESS', {}, 'id')

# ✅ Đúng:
def handler(event, context):
    physical_id = event.get('PhysicalResourceId', 'new-resource')
    try:
        result = do_something()
        send_response(event, context, 'SUCCESS', result, physical_id)
    except Exception as e:
        send_response(event, context, 'FAILED', {}, physical_id, str(e))

# ❌ Anti-Pattern 2: Delete fails if resource not found
def delete_resource(event):
    resource_id = event['PhysicalResourceId']
    client.delete_resource(Id=resource_id)   # ResourceNotFoundException → Error

# ✅ Đúng:
def delete_resource(event):
    resource_id = event['PhysicalResourceId']
    try:
        client.delete_resource(Id=resource_id)
    except client.exceptions.ResourceNotFoundException:
        print(f"Resource {resource_id} already deleted — OK")

# ❌ Anti-Pattern 3: Thay đổi PhysicalResourceId khi update không cần thiết
def handler(event, context):
    if event['RequestType'] == 'Update':
        new_id = str(uuid.uuid4())   # Tạo ID mới → CloudFormation coi là Replace!
        send_response(event, context, 'SUCCESS', {}, new_id)

# ✅ Đúng:
def handler(event, context):
    if event['RequestType'] == 'Update':
        old_id = event['PhysicalResourceId']  # Giữ nguyên ID
        send_response(event, context, 'SUCCESS', {}, old_id)
```

---

## Câu Hỏi Phỏng Vấn

**Q: Custom Resource xử lý Delete như thế nào khi stack bị rollback?**
> Khi stack rollback do lỗi trong CREATE, CloudFormation gọi Delete trên tất cả resources đã tạo thành công — bao gồm Custom Resource. Lambda handler nhận `RequestType: Delete` và phải xóa resource. Nếu Lambda xử lý Delete thành công → rollback tiếp tục. Nếu Lambda lỗi hoặc không trả response → CloudFormation chờ timeout rồi rollback tiếp (bỏ qua resource đó).

**Q: Nếu Lambda timeout trước khi gửi response, điều gì xảy ra?**
> CloudFormation chờ đến khi pre-signed S3 URL hết hạn (thường ~2 giờ sau thời điểm operation bắt đầu), sau đó đánh dấu resource là FAILED và tiến hành rollback. Đây là lý do luôn phải implement timeout handling trong Lambda — ví dụ: kiểm tra `context.get_remaining_time_in_millis()` và gửi FAILED response chủ động trước khi Lambda timeout.

**Q: Custom Resource khác CloudFormation Registry Resource Provider ở điểm nào?**
> **Custom Resource** (`Custom::MyType`) là giải pháp nhanh, không cần schema, phù hợp cho one-off automation tasks. **Resource Provider** trong Registry có schema JSON đầy đủ, validation tự động, hỗ trợ drift detection nếu implement, và có thể share qua Public Registry. Resource Provider phù hợp khi bạn muốn build resource type dùng như native CloudFormation resource, tái sử dụng nhiều lần, hoặc share cho cộng đồng.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
