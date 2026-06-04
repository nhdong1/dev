# Callback Pattern — waitForTaskToken

> **Callback Pattern** (Mẫu Gọi Lại) cho phép Step Functions tạm dừng execution và chờ tín hiệu từ bên ngoài trước khi tiếp tục. Đây là giải pháp cho các tình huống cần **human approval** (phê duyệt người dùng) hoặc tích hợp với **hệ thống bên thứ ba** không có API đồng bộ.

---

## 🎯 Vấn Đề Callback Giải Quyết

### Bài Toán Thực Tế

```
Bạn xây dựng hệ thống duyệt khoản vay. Quy trình:

1. Khách hàng nộp đơn vay
2. Hệ thống chấm điểm tự động (2 giây)
3. Nhân viên ngân hàng xem xét và phê duyệt (có thể mất 1-3 ngày!)
4. Hệ thống gửi hợp đồng
5. Khách hàng ký số (có thể mất vài giờ)
6. Giải ngân

Vấn đề: Làm thế nào để Step Functions "đợi" nhân viên phê duyệt
mà không tốn tài nguyên hay bị timeout?
```

### Giải Pháp: waitForTaskToken

```
Step Functions → Gửi task + taskToken cho Lambda/SQS
                                    │
                            Lambda gửi taskToken
                            cho hệ thống bên ngoài
                                    │
                            Hệ thống bên ngoài xử lý
                            (có thể mất nhiều ngày)
                                    │
                            Hệ thống gọi SendTaskSuccess/Failure
                            với taskToken
                                    │
Step Functions nhận tín hiệu và tiếp tục execution
```

---

## 🔧 Cách Hoạt Động

### 1. Định Nghĩa Task Với `.waitForTaskToken`

```json
{
  "ChoPheDuyet": {
    "Type": "Task",
    "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
    "Parameters": {
      "QueueUrl": "https://sqs.ap-southeast-1.amazonaws.com/123/ApprovalQueue",
      "MessageBody": {
        "taskToken.$": "$$.Task.Token",
        "loanApplicationId.$": "$.loanApplicationId",
        "applicantName.$": "$.applicantName",
        "amount.$": "$.amount",
        "approvalDeadline.$": "$.approvalDeadline"
      }
    },
    "HeartbeatSeconds": 86400,
    "TimeoutSeconds": 604800,
    "Catch": [
      {
        "ErrorEquals": ["States.HeartbeatTimeout", "States.Timeout"],
        "Next": "XuLyHetHan",
        "ResultPath": "$.error"
      },
      {
        "ErrorEquals": ["ApprovalRejected"],
        "Next": "XuLyTuChoi",
        "ResultPath": "$.rejectionInfo"
      }
    ],
    "Next": "TiepTucSauPheDuyet"
  }
}
```

> **`$$.Task.Token`** — Đây là **task token** duy nhất được Step Functions tạo ra cho mỗi lần task này chạy. Bạn phải lưu token này và dùng nó khi gọi callback.

### 2. Hệ Thống Bên Ngoài Xử Lý

```python
# Lambda xử lý message từ SQS
import json
import boto3

def lambda_handler(event, context):
    sfn_client = boto3.client('stepfunctions')
    
    for record in event['Records']:
        body = json.loads(record['body'])
        task_token = body['taskToken']
        loan_id = body['loanApplicationId']
        
        # Lưu vào database để nhân viên xem và xử lý sau
        save_to_approval_db(
            loan_id=loan_id,
            task_token=task_token,
            data=body
        )
        
        # Gửi email thông báo cho nhân viên phê duyệt
        send_approval_email(loan_id=loan_id)
    
    return {"status": "queued"}
```

### 3. Callback Khi Hoàn Thành

```python
# API Handler — nhân viên phê duyệt hoặc từ chối qua UI
import boto3

sfn_client = boto3.client('stepfunctions')

def approve_loan(loan_id: str, approver: str, notes: str):
    """Nhân viên phê duyệt khoản vay"""
    approval = get_approval_record(loan_id)
    task_token = approval['taskToken']
    
    # Gửi tín hiệu thành công về Step Functions
    sfn_client.send_task_success(
        taskToken=task_token,
        output=json.dumps({
            "approved": True,
            "approvedBy": approver,
            "approvalTime": datetime.utcnow().isoformat(),
            "notes": notes
        })
    )

def reject_loan(loan_id: str, reason: str):
    """Nhân viên từ chối khoản vay"""
    approval = get_approval_record(loan_id)
    task_token = approval['taskToken']
    
    # Gửi tín hiệu thất bại về Step Functions
    sfn_client.send_task_failure(
        taskToken=task_token,
        error="ApprovalRejected",
        cause=json.dumps({
            "reason": reason,
            "rejectedAt": datetime.utcnow().isoformat()
        })
    )

def send_heartbeat(loan_id: str):
    """Gửi heartbeat để Step Functions biết task vẫn đang sống"""
    approval = get_approval_record(loan_id)
    sfn_client.send_task_heartbeat(taskToken=approval['taskToken'])
```

---

## 🏗️ Kiến Trúc Đầy Đủ: Hệ Thống Phê Duyệt Khoản Vay

```
┌─────────────────────────────────────────────────────────────────┐
│                     Standard Workflow                           │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │ Xác Thực     │───▶│ Chấm Điểm   │───▶│  ChoPheDuyet    │  │
│  │ Đơn Vay      │    │ Tự Động      │    │ (waitForToken)  │  │
│  └──────────────┘    └──────────────┘    └────────┬─────────┘  │
│                                                   │ (pause)    │
│                                                   │            │
│                                    ┌──────────────▼─────────┐  │
│                                    │      (resume)           │  │
│                                    │   TiepTucSauPheDuyet   │  │
│                                    └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         │                                    ▲
         │ sendMessage                        │ sendTaskSuccess/Failure
         ▼                                    │
┌────────────────┐    ┌──────────────┐   ┌───┴───────────────┐
│   SQS Queue    │───▶│    Lambda    │   │   API Handler     │
│(Approval Queue)│    │ (processor)  │   │ (approve/reject)  │
└────────────────┘    └──────┬───────┘   └───────────────────┘
                             │                    ▲
                             ▼                    │
                      ┌──────────────┐   ┌────────┴──────────┐
                      │  DynamoDB    │   │   Nhân Viên       │
                      │(Pending Tasks│───▶   Phê Duyệt (UI)  │
                      │  + Tokens)   │   │                   │
                      └──────────────┘   └───────────────────┘
```

---

## 📋 Các Resource Hỗ Trợ `.waitForTaskToken`

| Resource | Cú Pháp | Use Case |
|---------|---------|---------|
| SQS | `arn:aws:states:::sqs:sendMessage.waitForTaskToken` | Tác vụ nền, xử lý async |
| SNS | `arn:aws:states:::sns:publish.waitForTaskToken` | Thông báo, webhook |
| Lambda | `arn:aws:states:::lambda:invoke.waitForTaskToken` | Logic phức tạp tùy chỉnh |
| ECS | `arn:aws:states:::ecs:runTask.waitForTaskToken` | Container task dài |
| API Gateway | `arn:aws:states:::apigateway:invoke.waitForTaskToken` | HTTP callback |
| EventBridge | `arn:aws:states:::events:putEvents.waitForTaskToken` | Event-driven callback |

---

## 🔄 Callback APIs (API Gọi Lại)

### SendTaskSuccess (Gửi Thành Công)

```python
sfn_client.send_task_success(
    taskToken="token-string-from-step-functions",
    output=json.dumps({
        # Output này trở thành kết quả của Task state
        "approvalStatus": "APPROVED",
        "approvedAt": "2026-05-18T10:30:00Z"
    })
)
```

### SendTaskFailure (Gửi Thất Bại)

```python
sfn_client.send_task_failure(
    taskToken="token-string-from-step-functions",
    error="ApprovalRejected",      # Error code — dùng trong Catch ErrorEquals
    cause="Reason for rejection"    # Mô tả chi tiết (tối đa 32KB)
)
```

### SendTaskHeartbeat (Gửi Nhịp Tim)

```python
sfn_client.send_task_heartbeat(
    taskToken="token-string-from-step-functions"
)
# Phải gọi trước khi HeartbeatSeconds hết hạn
# Dùng để thông báo "task vẫn đang chạy, đừng timeout"
```

---

## ⏱️ Quản Lý Timeout Và Heartbeat

```json
{
  "ChoPheDuyet": {
    "Type": "Task",
    "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
    "Parameters": { "...": "..." },
    "TimeoutSeconds": 604800,
    "HeartbeatSeconds": 3600,
    "Catch": [
      {
        "ErrorEquals": ["States.HeartbeatTimeout"],
        "Next": "XuLyKhongPhanHoi",
        "ResultPath": "$.error"
      },
      {
        "ErrorEquals": ["States.Timeout"],
        "Next": "XuLyHetHan",
        "ResultPath": "$.error"
      }
    ]
  }
}
```

| Tham Số | Giá Trị Trong Ví Dụ | Ý Nghĩa |
|---------|-------------------|---------|
| `TimeoutSeconds` | 604800 (7 ngày) | Tổng thời gian tối đa đợi callback |
| `HeartbeatSeconds` | 3600 (1 giờ) | Heartbeat phải đến trong vòng 1 tiếng |

### Chiến Lược Heartbeat

```python
import time
import threading

def process_approval_task(task_token: str):
    """Long-running task với heartbeat tự động"""
    
    # Gửi heartbeat mỗi 30 phút trong background
    def send_heartbeats():
        while not processing_complete:
            time.sleep(1800)  # 30 phút
            try:
                sfn_client.send_task_heartbeat(taskToken=task_token)
            except Exception:
                pass  # Execution có thể đã hoàn thành
    
    heartbeat_thread = threading.Thread(target=send_heartbeats, daemon=True)
    heartbeat_thread.start()
    
    # Xử lý task thực sự
    result = do_long_running_work()
    processing_complete = True
    
    sfn_client.send_task_success(
        taskToken=task_token,
        output=json.dumps(result)
    )
```

---

## 🔐 Bảo Mật Task Token

**Task token** là bí mật — ai có token này đều có thể resume execution. Cần bảo vệ:

```python
# ✅ Tốt: Lưu token trong DynamoDB với TTL và mã hóa
def save_task_token(loan_id: str, task_token: str, ttl_seconds: int = 604800):
    dynamodb.put_item(
        TableName='PendingApprovals',
        Item={
            'loanId': {'S': loan_id},
            'taskToken': {'S': task_token},  # DynamoDB mã hóa at rest
            'expiresAt': {'N': str(int(time.time()) + ttl_seconds)},
            'status': {'S': 'PENDING'}
        }
    )

# ✅ Tốt: Xác thực người dùng trước khi dùng token
def approve_loan_api(loan_id: str, approver_id: str):
    if not has_approval_permission(approver_id):
        raise PermissionDeniedError("Không có quyền phê duyệt")
    
    record = get_pending_approval(loan_id)
    if not record:
        raise NotFoundError(f"Không tìm thấy đơn vay {loan_id}")
    
    sfn_client.send_task_success(
        taskToken=record['taskToken'],
        output=json.dumps({"approvedBy": approver_id})
    )

# ❌ Không nên: Expose token trực tiếp trong API response
# ❌ Không nên: Lưu token trong local file hoặc log
```

---

## 🆚 Callback Pattern vs Activity Worker

Step Functions có hai cách để "chờ" tín hiệu bên ngoài:

| | Callback (waitForTaskToken) | Activity Worker (Công Nhân Hoạt Động) |
|--|---------------------------|--------------------------------------|
| **Cơ chế** | Push: Step Functions gửi task token ra ngoài | Pull: Worker liên tục polling để nhận task |
| **Tích hợp** | Lambda, SQS, SNS, ECS… | GetActivityTask API |
| **Độ trễ** | Thấp (ngay khi có token) | Có thể cao (polling interval) |
| **Phức tạp** | Vừa phải | Cao hơn (phải quản lý worker lifecycle) |
| **Use case** | Human approval, third-party callback | Legacy on-premises worker, long-running compute |
| **Khuyến nghị** | ✅ Được khuyến nghị | ⚠️ Chỉ dùng khi cần thiết |

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Tại sao Callback Pattern cần Standard Workflow chứ không phải Express?**

> Express Workflow có thời hạn tối đa 5 phút và không hỗ trợ `waitForTaskToken`. Callback Pattern thường dùng cho human approval có thể kéo dài nhiều ngày — chỉ Standard Workflow (tối đa 1 năm) mới đáp ứng được. Ngoài ra, Standard đảm bảo exactly-once execution, quan trọng khi liên quan đến giao dịch tài chính.

**Q: Điều gì xảy ra nếu token hết hạn trước khi callback được gọi?**

> Nếu `TimeoutSeconds` trôi qua mà không có callback, Step Functions ném `States.Timeout`. Nếu `HeartbeatSeconds` trôi qua mà không có heartbeat, ném `States.HeartbeatTimeout`. Cả hai đều cần có `Catch` để xử lý — thường là gửi thông báo cảnh báo và đóng task.

**Q: Làm thế nào để handle trường hợp nhân viên phê duyệt muốn yêu cầu thêm thông tin?**

> Implement "request more info" action: gọi `send_task_failure` với error code đặc biệt như `"AdditionalInfoRequired"`, sau đó trong `Catch` chuyển sang state thu thập thêm thông tin — có thể là một Callback Pattern mới, tạo ra vòng lặp phê duyệt nhiều bước. Hoặc dùng `send_task_success` với trạng thái trung gian như `{"status": "NEEDS_MORE_INFO"}` và để state machine quyết định bước tiếp theo.

---

## 🔗 Điều Hướng

- **Quay lại:** [3-error-handling.md](./3-error-handling.md)
- **Tiếp theo:** [5-saga-orchestration.md](./5-saga-orchestration.md) — Saga Pattern
- **Liên quan:** [README.md](./README.md) — Tổng quan Step Functions

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
