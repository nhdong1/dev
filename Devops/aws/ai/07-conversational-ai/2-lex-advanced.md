# Amazon Lex Advanced — Nâng Cao: Custom Slots, Context & Lambda Fulfillment

> Các kỹ thuật nâng cao trong Amazon Lex V2: Custom slot types (Kiểu Khe Tùy Chỉnh), Dialog Context (Ngữ Cảnh Hội Thoại), Lambda hooks (Móc Lambda), tích hợp Amazon Kendra và thiết kế enterprise chatbot

---

## 🎰 Custom Slot Types — Kiểu Khe Tùy Chỉnh

### Tại Sao Cần Custom Slot Types?

Built-in slot types (kiểu khe tích hợp sẵn) chỉ nhận dạng được dữ liệu phổ thông (ngày, số, thành phố...). Với dữ liệu nghiệp vụ cụ thể như `LoạiSảnPhẩm`, `MãPhòngBan`, `TrạngTháiĐơnHàng`, bạn cần tự định nghĩa.

### Enumeration-based Custom Slot (Kiểu Liệt Kê)

```json
{
  "slotTypeName": "ProductCategory",
  "description": "Danh mục sản phẩm trong hệ thống",
  "slotTypeValues": [
    {
      "sampleValue": { "value": "điện thoại" },
      "synonyms": [
        { "value": "phone" },
        { "value": "mobile" },
        { "value": "smartphone" }
      ]
    },
    {
      "sampleValue": { "value": "laptop" },
      "synonyms": [
        { "value": "máy tính xách tay" },
        { "value": "notebook" }
      ]
    },
    {
      "sampleValue": { "value": "phụ kiện" },
      "synonyms": [
        { "value": "accessories" },
        { "value": "accessories" }
      ]
    }
  ],
  "valueSelectionSetting": {
    "resolutionStrategy": "OriginalValue"
    // "TopResolution" — dùng giá trị khớp tốt nhất
    // "OriginalValue" — giữ nguyên từ người dùng nói
  }
}
```

### Regex-based Custom Slot (Kiểu Biểu Thức Chính Quy)

```json
{
  "slotTypeName": "OrderIdSlot",
  "description": "Mã đơn hàng theo định dạng ORD-XXXXXXXX",
  "regexFilter": {
    "pattern": "ORD-[0-9]{8}"
  }
}
```

**Dùng khi:** Mã nhân viên, số hợp đồng, mã sản phẩm có format cố định.

### Grammar-based Slot (Kiểu Ngữ Pháp — chỉ Voice)

```
Dùng GRXML (Grammar XML — Ngữ Pháp XML) để định nghĩa chính xác cách phát âm các giá trị.
Phù hợp với voice chatbot cần nhận dạng số tài khoản, mã PIN.
```

---

## 🔁 Dialog Context — Ngữ Cảnh Hội Thoại

### Active Contexts (Ngữ Cảnh Đang Hoạt Động)

**Context** cho phép một intent kích hoạt chỉ khi ngữ cảnh nhất định đang tồn tại — tạo ra luồng hội thoại phân nhánh theo trạng thái.

```
Ví dụ: "FollowUpQuestion" intent chỉ được phép sau khi "CheckBalance" đã chạy
```

**Cấu hình Context:**

```json
{
  "intentName": "FollowUpTransfer",
  "inputContexts": [
    {
      "name": "balance_checked"
    }
  ],
  "outputContexts": [
    {
      "name": "transfer_initiated",
      "timeToLiveInSeconds": 300,
      "turnsToLive": 5
    }
  ]
}
```

### Context Flow Diagram (Sơ Đồ Luồng Ngữ Cảnh)

```
User: "Số dư tài khoản là bao nhiêu?"
    → Intent: CheckBalance
    → Thực thi → Trả về số dư
    → OUTPUT CONTEXT: balance_checked (sống 5 lượt, tối đa 300 giây)

User: "Chuyển 500k cho bạn tôi"
    → INPUT CONTEXT: balance_checked (đang active)
    → Intent: FollowUpTransfer (được kích hoạt vì context khớp)
    → Slots: amount=500k, recipient=?
    → "Bạn muốn chuyển cho ai?"
```

### Session Attributes vs Slot Values vs Context

| Cơ Chế | Phạm Vi | Mục Đích |
|---------|---------|---------|
| **Slot values** | Trong một intent | Thông tin cụ thể cho intent đang xử lý |
| **Session attributes** | Toàn bộ session | Dữ liệu dùng chung: userId, language, tier |
| **Active context** | Giữa các intents | Điều hướng luồng hội thoại, guard (bảo vệ) intent |

---

## ⚡ Lambda Hooks — Móc Lambda Nâng Cao

### Hai Điểm Gắn Lambda (Lambda Hook Points)

```
Người Dùng Gửi Message
        ↓
1. [Dialog Code Hook] ← Lambda 1: Validation & Custom logic trong quá trình dialog
   - Validate slot values
   - Elicit next slot theo điều kiện
   - Thêm/xóa session attributes
        ↓
Lex quyết định dialog step tiếp theo
        ↓
2. [Fulfillment Code Hook] ← Lambda 2: Thực thi nghiệp vụ khi tất cả slots đủ
   - Gọi API backend
   - Truy vấn database
   - Gửi notification
        ↓
Lex trả kết quả về người dùng
```

### Dialog Code Hook — Validation Lambda

```python
import boto3
import json
from datetime import datetime, date

def lambda_handler(event, context):
    """
    Dialog Code Hook: Gọi MỖI LƯỢT trong hội thoại để validate và điều hướng
    """
    invocation_source = event['invocationSource']
    intent_name = event['sessionState']['intent']['name']
    slots = event['sessionState']['intent']['slots']
    session_attrs = event['sessionState'].get('sessionAttributes', {})

    # Chỉ xử lý khi đang trong quá trình dialog (không phải fulfillment)
    if invocation_source == 'DialogCodeHook':
        return handle_dialog(intent_name, slots, session_attrs, event)

    elif invocation_source == 'FulfillmentCodeHook':
        return handle_fulfillment(intent_name, slots, session_attrs)


def handle_dialog(intent_name, slots, session_attrs, event):
    """Validate slots và điều hướng dialog"""

    if intent_name == 'BookFlight':
        # Validate travel_date nếu đã có
        travel_date_slot = slots.get('travel_date')
        if travel_date_slot and travel_date_slot.get('value'):
            travel_date_str = travel_date_slot['value']['interpretedValue']
            travel_date = datetime.strptime(travel_date_str, '%Y-%m-%d').date()

            if travel_date < date.today():
                # Ngày trong quá khứ — từ chối và hỏi lại
                return elicit_slot(
                    intent_name=intent_name,
                    slots=slots,
                    slot_to_elicit='travel_date',
                    message='Ngày đi không thể là ngày trong quá khứ. Vui lòng chọn ngày khác.',
                    session_attrs=session_attrs
                )

    # Delegate — để Lex tự quyết định bước tiếp theo
    return delegate(slots, session_attrs)


def elicit_slot(intent_name, slots, slot_to_elicit, message, session_attrs):
    """Yêu cầu người dùng cung cấp lại một slot cụ thể"""
    return {
        'sessionState': {
            'sessionAttributes': session_attrs,
            'dialogAction': {
                'type': 'ElicitSlot',
                'slotToElicit': slot_to_elicit
            },
            'intent': {
                'name': intent_name,
                'slots': slots,
                'state': 'InProgress'
            }
        },
        'messages': [
            {
                'contentType': 'PlainText',
                'content': message
            }
        ]
    }


def delegate(slots, session_attrs):
    """Trả quyền điều khiển về Lex để tự quyết định bước tiếp theo"""
    return {
        'sessionState': {
            'sessionAttributes': session_attrs,
            'dialogAction': {
                'type': 'Delegate'
            },
            'intent': {
                'slots': slots,
                'state': 'InProgress'
            }
        }
    }
```

### Fulfillment Lambda — Xử Lý Nghiệp Vụ

```python
import boto3
import json
import uuid
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')

def handle_fulfillment(intent_name, slots, session_attrs):
    """Thực thi logic nghiệp vụ sau khi tất cả slots đã đầy đủ"""

    if intent_name == 'BookFlight':
        departure = slots['departure_city']['value']['interpretedValue']
        arrival = slots['arrival_city']['value']['interpretedValue']
        travel_date = slots['travel_date']['value']['interpretedValue']
        user_id = session_attrs.get('userId', 'anonymous')

        try:
            # Lưu booking vào DynamoDB
            booking_id = f"VN-{uuid.uuid4().hex[:8].upper()}"
            table = dynamodb.Table('flight-bookings')
            table.put_item(Item={
                'bookingId': booking_id,
                'userId': user_id,
                'departure': departure,
                'arrival': arrival,
                'travelDate': travel_date,
                'createdAt': datetime.utcnow().isoformat(),
                'status': 'CONFIRMED'
            })

            # Gửi email xác nhận (tùy chọn)
            # sns.publish(TopicArn='arn:aws:sns:...', Message=f"Booking confirmed: {booking_id}")

            return close(
                intent_name=intent_name,
                fulfillment_state='Fulfilled',
                message=f"✅ Đặt vé thành công!\n"
                        f"Mã đặt chỗ: {booking_id}\n"
                        f"Chuyến bay: {departure} → {arrival}\n"
                        f"Ngày: {travel_date}",
                session_attrs={**session_attrs, 'last_booking_id': booking_id}
            )

        except Exception as e:
            return close(
                intent_name=intent_name,
                fulfillment_state='Failed',
                message='Xin lỗi, có lỗi xảy ra khi đặt vé. Vui lòng thử lại sau.',
                session_attrs=session_attrs
            )

    return close(intent_name, 'Failed', 'Tôi không hiểu yêu cầu này.', session_attrs)


def close(intent_name, fulfillment_state, message, session_attrs):
    """Đóng hội thoại với kết quả cuối cùng"""
    return {
        'sessionState': {
            'sessionAttributes': session_attrs,
            'dialogAction': {
                'type': 'Close'
            },
            'intent': {
                'name': intent_name,
                'state': fulfillment_state
            }
        },
        'messages': [
            {
                'contentType': 'PlainText',
                'content': message
            }
        ]
    }
```

---

## 🔍 Tích Hợp Amazon Kendra — Trả Lời Câu Hỏi Từ Tài Liệu

### Kiến Trúc Lex + Kendra

```
User: "Chính sách nghỉ phép của công ty là gì?"
    ↓
Lex (Intent: SearchKnowledge)
    ↓ Lambda
Amazon Kendra (Index chứa tài liệu HR PDF, Confluence pages)
    ↓ Natural language search
Kết quả: Đoạn trích từ "HR Policy 2026.pdf" trang 15
    ↓
Lambda format response
    ↓
Lex trả lời người dùng
```

### Lambda gọi Kendra

```python
import boto3

kendra = boto3.client('kendra')

def search_kendra(query: str, index_id: str) -> str:
    """Tìm kiếm câu trả lời trong Kendra index"""

    response = kendra.query(
        IndexId=index_id,
        QueryText=query,
        QueryResultTypeFilter='ANSWER',  # Chỉ lấy kết quả dạng câu trả lời
        PageSize=3
    )

    results = response.get('ResultItems', [])

    if not results:
        return "Tôi không tìm thấy thông tin liên quan đến câu hỏi của bạn."

    # Lấy kết quả đầu tiên (score cao nhất)
    top_result = results[0]
    answer = top_result.get('AdditionalAttributes', [{}])[0].get('Value', {})
    answer_text = answer.get('TextWithHighlightsValue', {}).get('Text', '')
    document_title = top_result.get('DocumentTitle', {}).get('Text', 'Không rõ tài liệu')

    return f"{answer_text}\n\n📄 Nguồn: {document_title}"
```

---

## 🏢 Thiết Kế Enterprise Chatbot

### Multi-Bot Architecture (Kiến Trúc Đa Bot)

```
Routing Bot (Bot Điều Phối)
    ├── Intent: AskHR → Chuyển đến HR Bot
    ├── Intent: AskIT → Chuyển đến IT Support Bot
    ├── Intent: AskSales → Chuyển đến Sales Bot
    └── FallbackIntent → Knowledge Base Search

HR Bot:
    - CheckLeaveBalance (Kiểm tra số ngày phép còn lại)
    - RequestLeave (Xin nghỉ phép)
    - SearchHRPolicy (Tìm chính sách nhân sự)

IT Support Bot:
    - ResetPassword (Đặt lại mật khẩu)
    - ReportIssue (Báo cáo sự cố)
    - CheckTicketStatus (Kiểm tra trạng thái ticket)
```

### IAM Permissions cho Lex

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lex:RecognizeText",
        "lex:RecognizeUtterance",
        "lex:GetSession",
        "lex:PutSession",
        "lex:DeleteSession"
      ],
      "Resource": "arn:aws:lex:us-east-1:123456789012:bot-alias/ABCDEF123456/TSTALIASID"
    }
  ]
}
```

### Lex V2 Conversation Logs — Phân Tích Chất Lượng Bot

```python
import boto3
import json

logs_client = boto3.client('logs')

def analyze_bot_quality(log_group_name: str, start_time: int, end_time: int):
    """
    Phân tích nhật ký hội thoại để cải thiện bot
    - Tỷ lệ FallbackIntent: Intent nào thường không nhận dạng được
    - Slot elicitation failure rate: Slot nào khó điền
    - Session abandonment: Người dùng bỏ cuộc ở đâu
    """
    paginator = logs_client.get_paginator('filter_log_events')
    events = []

    for page in paginator.paginate(
        logGroupName=log_group_name,
        startTime=start_time,
        endTime=end_time,
        filterPattern='{ $.intent.name = "FallbackIntent" }'
    ):
        events.extend(page['events'])

    fallback_count = len(events)
    print(f"FallbackIntent count (số lần không nhận dạng được): {fallback_count}")

    # Trích xuất các câu không nhận dạng được để cải thiện utterances
    unrecognized_utterances = []
    for event in events:
        log_data = json.loads(event['message'])
        utterance = log_data.get('inputTranscript', '')
        unrecognized_utterances.append(utterance)

    return unrecognized_utterances
```

---

## 🔄 Lex Version & Alias Management (Quản Lý Phiên Bản)

### Workflow Khuyến Nghị

```
Development (Phát Triển)
    ↓ Build bot → Test
Draft version (phiên bản nháp)
    ↓ Test trong Console → OK
Create Bot Version 1 (không thể chỉnh sửa sau khi tạo)
    ↓
Bot Alias: "staging" → Version 1
    ↓ QA (Quality Assurance — Kiểm Tra Chất Lượng) team test
Bot Alias: "production" → Version 1
    ↓
Production traffic

--- Sau khi có cập nhật ---
Draft version (chỉnh sửa)
    ↓ Test
Create Bot Version 2
    ↓
Bot Alias: "production" → Version 2 (zero-downtime switch)
```

```python
import boto3

lex_mgmt = boto3.client('lexv2-models')

def promote_to_production(bot_id: str, version: str):
    """Chuyển bot alias production sang version mới"""
    response = lex_mgmt.update_bot_alias(
        botAliasId='PRODUCTION_ALIAS_ID',
        botAliasName='production',
        botId=bot_id,
        botVersion=version,
        botAliasLocaleSettings={
            'en_US': {
                'enabled': True,
                'codeHookSpecification': {
                    'lambdaCodeHook': {
                        'lambdaARN': 'arn:aws:lambda:us-east-1:123456:function:TravelBotHandler',
                        'codeHookInterfaceVersion': '1.0'
                    }
                }
            }
        },
        conversationLogSettings={
            'textLogSettings': [
                {
                    'enabled': True,
                    'destination': {
                        'cloudWatch': {
                            'cloudWatchLogGroupArn': 'arn:aws:logs:...',
                            'logPrefix': 'production'
                        }
                    }
                }
            ]
        }
    )
    print(f"Updated alias to version {version}")
```

---

## 📊 Monitoring & Metrics Quan Trọng

| Metric | Ý Nghĩa | Ngưỡng Cảnh Báo |
|--------|---------|----------------|
| **MissedUtteranceCount** | Số câu không nhận dạng được → FallbackIntent | > 20% tổng requests |
| **RuntimeRequestCount** | Tổng số requests | Dùng để dự báo chi phí |
| **RuntimeSuccessfulRequestLatency** | Độ trễ phản hồi | > 3 giây cần tối ưu |
| **RuntimeThrottledRequests** | Requests bị giới hạn tốc độ | > 0 → cần tăng quota |
| **FulfillmentLambdaErrors** | Lambda fulfillment lỗi | > 1% → cần kiểm tra Lambda |

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

def create_lex_alarm(bot_id: str, alarm_name: str):
    """Tạo CloudWatch alarm cho FallbackIntent rate cao"""
    cloudwatch.put_metric_alarm(
        AlarmName=alarm_name,
        MetricName='MissedUtteranceCount',
        Namespace='AWS/Lex',
        Dimensions=[
            {'Name': 'BotName', 'Value': 'TravelBot'},
            {'Name': 'BotVersion', 'Value': '1'},
            {'Name': 'Operation', 'Value': 'PostText'}
        ],
        Period=300,           # 5 phút
        EvaluationPeriods=3,
        Threshold=50,         # Cảnh báo nếu > 50 missed utterances trong 5 phút
        ComparisonOperator='GreaterThanThreshold',
        Statistic='Sum',
        AlarmActions=['arn:aws:sns:us-east-1:123456:bot-alerts']
    )
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về Lex Advanced

**Q: Khi nào dùng Dialog Code Hook thay vì chỉ Fulfillment Code Hook?**

> Dùng **Dialog Code Hook** khi cần: (1) Validate slot value ngay khi người dùng cung cấp (thay vì chờ đến cuối), (2) Thay đổi luồng hội thoại động (ví dụ: hỏi thêm câu dựa trên câu trả lời trước), (3) Populate slot value từ dữ liệu có sẵn (ví dụ: pre-fill userId từ session). Dùng **chỉ Fulfillment** khi logic đơn giản, chỉ cần xử lý cuối cùng.

**Q: Active Context hoạt động như thế nào?**

> Context có `timeToLiveInSeconds` và `turnsToLive`. Lex chỉ kích hoạt intent có `inputContexts` nếu context đó đang active. Sau khi context hết hạn hoặc đủ số turns, intent đó không còn được ưu tiên. Đây là cách tạo branching dialog (hội thoại phân nhánh) theo trạng thái.

**Q: Làm thế nào để migrate Lex V1 lên Lex V2?**

> AWS cung cấp migration tool trong Console. Lex V2 có cấu trúc khác (Bot → Bot Version → Bot Alias → Locale) và dùng `lexv2-runtime` API thay vì `lex-runtime`. Quan trọng: V2 có tốt hơn về conversation logs, streaming, và intent confidence scores.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Trước | [1-lex-fundamentals.md](./1-lex-fundamentals.md) — Lex Fundamentals |
| → Tiếp theo | [3-amazon-q-business.md](./3-amazon-q-business.md) — Amazon Q Business |
| ↑ Module | [README.md](./README.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03 | **Phiên Bản:** 1.0
