# Amazon Lex Fundamentals — Nền Tảng Xây Dựng Chatbot AI

> Amazon Lex V2 là dịch vụ managed (được quản lý hoàn toàn) để xây dựng chatbot và voice assistant (trợ lý giọng nói) sử dụng Deep Learning (Học Sâu) — cùng công nghệ tạo ra Alexa

---

## 🎯 Amazon Lex là gì?

**Amazon Lex** cung cấp hai khả năng cốt lõi:

1. **ASR — Automatic Speech Recognition** (Nhận Dạng Giọng Nói Tự Động): Chuyển đổi giọng nói thành văn bản
2. **NLU — Natural Language Understanding** (Hiểu Ngôn Ngữ Tự Nhiên): Hiểu ý nghĩa và mục đích câu nói

```
Người dùng nói: "Tôi muốn đặt vé máy bay từ Hà Nội đến TP.HCM ngày mai"
        ↓ ASR (Speech → Text)
"Tôi muốn đặt vé máy bay từ Hà Nội đến TP.HCM ngày mai"
        ↓ NLU (Text → Intent + Slots)
Intent: BookFlight
Slots:
  - departure_city: "Hà Nội"
  - arrival_city: "TP.HCM"
  - travel_date: "ngày mai" → 2026-06-04
```

---

## 🏗️ Kiến Trúc Amazon Lex V2

### Thành Phần Cơ Bản

```
Bot (Con Rô-Bốt Trò Chuyện)
└── Bot Alias (Phiên Bản Bot)
    └── Bot Version (Phiên Bản Cụ Thể)
        └── Locale (Ngôn Ngữ: en_US, vi_VN...)
            ├── Intent 1: BookFlight
            │   ├── Sample Utterances (Câu Mẫu)
            │   ├── Slots (Khe Tham Số)
            │   │   ├── departure_city [Required]
            │   │   ├── arrival_city [Required]
            │   │   └── travel_date [Required]
            │   ├── Slot Elicitation Prompts (Câu Hỏi Thu Thập Slot)
            │   ├── Confirmation Prompt (Câu Xác Nhận)
            │   └── Fulfillment (Xử Lý Kết Quả)
            ├── Intent 2: CheckBalance
            └── FallbackIntent (Dự Phòng)
```

---

## 📌 Intent — Mục Đích Hội Thoại

### Intent là gì?

**Intent** (Mục Đích) là hành động hoặc yêu cầu mà người dùng muốn thực hiện. Mỗi bot có thể có nhiều intents.

### Anatomy of an Intent (Cấu Trúc Một Intent)

```json
{
  "intentName": "BookFlight",
  "description": "Đặt vé máy bay cho người dùng",
  "sampleUtterances": [
    "Tôi muốn đặt vé",
    "Book a flight",
    "Mua vé từ {departure_city} đến {arrival_city}",
    "Cho tôi vé bay ngày {travel_date}"
  ],
  "slots": ["departure_city", "arrival_city", "travel_date"],
  "fulfillmentActivity": "Lambda function ARN hoặc Return to application"
}
```

### Các Loại Intent

| Loại Intent | Mô Tả | Ví Dụ |
|------------|-------|-------|
| **Custom Intent** (Tùy Chỉnh) | Do bạn tự định nghĩa | `BookFlight`, `CheckBalance` |
| **Built-in Intent** (Có Sẵn) | AWS cung cấp sẵn | `AMAZON.HelpIntent`, `AMAZON.CancelIntent`, `AMAZON.StopIntent` |
| **FallbackIntent** (Dự Phòng) | Kích hoạt khi không nhận dạng được | Xử lý "tôi không hiểu" |

### Sample Utterances (Câu Mẫu) — Bao Nhiêu Là Đủ?

```
Quy Tắc Thực Tế:
- Tối thiểu: 15-20 utterances mỗi intent
- Tốt: 30-50 utterances đa dạng
- Đừng duplicate (lặp lại) — Lex đã tự xử lý paraphrases (diễn đạt khác)
- Bao gồm cả câu có slots và không có slots

Ví dụ cho BookFlight:
✅ "Đặt vé"                            (ngắn, không slot)
✅ "Tôi muốn đặt vé máy bay"           (trung bình)
✅ "Book a flight to {arrival_city}"   (có slot)
✅ "Bay từ {departure_city} ngày mai"  (có slot, ngắn)
❌ "Đặt vé máy bay" + "Mua vé máy bay" (quá giống nhau)
```

---

## 🎰 Slot — Khe Thông Tin

### Slot là gì?

**Slot** (Khe Tham Số) là thông tin cụ thể mà bot cần thu thập để thực hiện một intent. Lex tự động trích xuất (extract) slots từ câu nói của người dùng.

### Slot Configuration (Cấu Hình Khe)

```
Slot: departure_city
├── Slot Type: AMAZON.City (Built-in) hoặc custom type
├── Required: true / false
├── Elicitation Prompt (Câu Hỏi Thu Thập):
│       "Bạn muốn bay từ thành phố nào?"
├── Max Retries (Số Lần Thử Lại): 2
└── Validation (Kiểm Tra): Lambda function
```

### Built-in Slot Types (Kiểu Slot Tích Hợp Sẵn)

| Slot Type | Nhận Dạng Được | Ví Dụ |
|-----------|--------------|-------|
| `AMAZON.Date` | Ngày tháng | "ngày mai", "thứ 2 tuần tới", "2026-06-10" |
| `AMAZON.Time` | Giờ | "lúc 3 giờ chiều", "14:00" |
| `AMAZON.Number` | Số | "năm", "5", "hai mươi" |
| `AMAZON.Duration` | Khoảng thời gian | "2 giờ", "30 phút" |
| `AMAZON.City` | Tên thành phố | "Hà Nội", "New York" |
| `AMAZON.EmailAddress` | Email | "user@example.com" |
| `AMAZON.PhoneNumber` | Số điện thoại | "0912345678" |
| `AMAZON.Airline` | Hãng hàng không | "Vietnam Airlines", "VietJet" |
| `AMAZON.Currency` | Tiền tệ | "100 đô", "500.000 đồng" |

---

## 🔄 Multi-turn Dialog — Hội Thoại Nhiều Lượt

### Luồng Slot Filling (Điền Khe Tuần Tự)

```
Bot: "Xin chào! Tôi có thể giúp gì cho bạn?"
User: "Tôi muốn đặt vé"

Bot: (Intent = BookFlight, departure_city = ?)
    "Bạn muốn bay từ thành phố nào?"
User: "Từ Hà Nội"

Bot: (departure_city = Hà Nội, arrival_city = ?)
    "Bạn muốn đến đâu?"
User: "TP.HCM"

Bot: (arrival_city = TP.HCM, travel_date = ?)
    "Bạn muốn bay ngày nào?"
User: "Ngày mai"

Bot: (All slots filled → Confirmation Prompt)
    "Xác nhận: Vé từ Hà Nội đến TP.HCM ngày 04/06/2026. Đúng không?"
User: "Đúng rồi"

Bot: (→ Fulfillment → Lambda → DynamoDB)
    "Đã đặt vé thành công! Mã đặt chỗ: VN-2026-001"
```

### Dialog States (Trạng Thái Hội Thoại)

| Trạng Thái | Ý Nghĩa |
|-----------|---------|
| `ElicitIntent` | Bot đang hỏi người dùng muốn làm gì |
| `ElicitSlot` | Bot đang hỏi để điền một slot còn thiếu |
| `ConfirmIntent` | Bot đang xác nhận intent và slots với người dùng |
| `Fulfilled` | Slots đã đủ, đang thực thi fulfillment |
| `ReadyForFulfillment` | Sẵn sàng thực thi (trả về application để xử lý) |
| `Failed` | Hội thoại thất bại (max retries exceeded) |
| `Closing` | Bot đang kết thúc hội thoại |

---

## ⚡ Fulfillment — Xử Lý Kết Quả

### Các Phương Thức Fulfillment

```
1. Return to Application (Trả Về Ứng Dụng)
   Bot → trả DialogState="ReadyForFulfillment" cho client
   Client tự xử lý dựa trên intent name + slot values
   → Dùng khi: Client-side logic đơn giản

2. AWS Lambda Function (Hàm Lambda AWS)
   Bot → gọi Lambda → Lambda xử lý nghiệp vụ → trả response
   → Dùng khi: Cần gọi API, truy vấn DB, logic phức tạp
```

### Lambda Request từ Lex

```json
{
  "messageVersion": "1.0",
  "invocationSource": "FulfillmentCodeHook",
  "userId": "user-123",
  "currentIntent": {
    "name": "BookFlight",
    "slots": {
      "departure_city": "Hà Nội",
      "arrival_city": "TP.HCM",
      "travel_date": "2026-06-04"
    },
    "confirmationStatus": "Confirmed"
  },
  "bot": {
    "name": "TravelBot",
    "alias": "production",
    "version": "1"
  },
  "sessionAttributes": {
    "user_tier": "premium"
  }
}
```

### Lambda Response trả về Lex

```python
def lambda_handler(event, context):
    intent_name = event['currentIntent']['name']
    slots = event['currentIntent']['slots']

    if intent_name == 'BookFlight':
        departure = slots['departure_city']
        arrival = slots['arrival_city']
        travel_date = slots['travel_date']

        # Gọi API đặt vé (giả lập)
        booking_id = create_booking(departure, arrival, travel_date)

        return {
            "dialogAction": {
                "type": "Close",
                "fulfillmentState": "Fulfilled",
                "message": {
                    "contentType": "PlainText",
                    "content": f"Đã đặt vé thành công! Mã: {booking_id}"
                }
            },
            "sessionAttributes": {
                "last_booking_id": booking_id
            }
        }
```

### Dialog Action Types (Loại Hành Động Hội Thoại)

| `dialogAction.type` | Ý Nghĩa |
|--------------------|---------|
| `ElicitIntent` | Yêu cầu Lex hỏi lại intent |
| `ElicitSlot` | Yêu cầu Lex hỏi lại một slot cụ thể |
| `ConfirmIntent` | Yêu cầu Lex xác nhận intent |
| `Delegate` | Để Lex tự quyết định bước tiếp theo |
| `Close` | Kết thúc hội thoại (Fulfilled hoặc Failed) |

---

## 🔗 Kết Hợp Với Các Dịch Vụ AWS

### Lex + Amazon Connect (IVR — Interactive Voice Response)

```
Người Gọi Điện
    ↓ (điện thoại)
Amazon Connect (Cloud Contact Center — Trung Tâm Liên Hệ Đám Mây)
    ↓ (text stream)
Amazon Lex (xử lý ngôn ngữ, nhận intent)
    ↓ (fulfillment)
Lambda → CRM API (lấy thông tin khách hàng)
    ↓ (nếu không giải quyết được)
Amazon Connect → Chuyển đến agent người thật
```

**Lợi ích:** Tự động hóa IVR thông minh — giảm tải agent, xử lý 24/7, giảm chi phí vận hành.

### Lex + Amazon Kendra (Knowledge Base Search)

```
User: "Chính sách hoàn tiền của công ty là gì?"
    ↓
Lex (Intent: SearchKnowledge, Slot: query = "chính sách hoàn tiền")
    ↓ Lambda
Amazon Kendra (Tìm Kiếm Tài Liệu Thông Minh)
    ↓ Tìm trong tài liệu PDF, Word nội bộ
Trả lời với đoạn trích từ tài liệu
```

### Lex + DynamoDB (Session State Persistence)

```python
# Lưu session state (trạng thái phiên) vào DynamoDB để nhớ ngữ cảnh
def save_session(user_id: str, context: dict):
    table = dynamodb.Table('lex-sessions')
    table.put_item(Item={
        'userId': user_id,
        'context': context,
        'ttl': int(time.time()) + 3600  # Hết hạn sau 1 giờ
    })
```

---

## 💻 Gọi Lex V2 API với Python boto3

### Gửi Text Message đến Bot

```python
import boto3

lex_client = boto3.client('lexv2-runtime', region_name='us-east-1')

def chat_with_lex(user_message: str, session_id: str) -> str:
    response = lex_client.recognize_text(
        botId='ABCDEF123456',              # Bot ID từ Console
        botAliasId='TSTALIASID',          # Alias ID (TSTALIASID cho TestBotAlias)
        localeId='vi_VN',                  # Hoặc 'en_US'
        sessionId=session_id,              # Unique session per user
        text=user_message
    )

    # Lấy messages từ response
    messages = response.get('messages', [])
    if messages:
        return messages[0]['content']

    # Kiểm tra dialog state
    dialog_state = response['sessionState']['dialogAction']['type']
    print(f"Dialog State: {dialog_state}")
    print(f"Intent: {response['sessionState'].get('intent', {}).get('name')}")

    return "Xin lỗi, tôi không hiểu yêu cầu của bạn."

# Sử dụng
session_id = "user-session-001"
while True:
    user_input = input("Bạn: ")
    if user_input.lower() in ['quit', 'thoát']:
        break
    bot_response = chat_with_lex(user_input, session_id)
    print(f"Bot: {bot_response}")
```

### Gửi Audio (Voice) đến Bot

```python
import boto3

lex_client = boto3.client('lexv2-runtime', region_name='us-east-1')

def voice_chat_with_lex(audio_bytes: bytes, session_id: str) -> str:
    response = lex_client.recognize_utterance(
        botId='ABCDEF123456',
        botAliasId='TSTALIASID',
        localeId='en_US',
        sessionId=session_id,
        requestContentType='audio/l16; rate=16000; channels=1',
        responseContentType='text/plain;charset=utf-8',
        inputStream=audio_bytes
    )

    return response['inputTranscript']  # Text đã nhận dạng
```

### Quản Lý Session Attributes

```python
# Đặt session attributes (thuộc tính phiên) — thông tin nhớ xuyên suốt hội thoại
response = lex_client.recognize_text(
    botId='ABCDEF123456',
    botAliasId='TSTALIASID',
    localeId='en_US',
    sessionId=session_id,
    text=user_message,
    sessionState={
        'sessionAttributes': {
            'userId': 'user-001',
            'user_tier': 'premium',
            'language_preference': 'vi'
        }
    }
)
```

---

## 🧪 Kiểm Tra & Debug

### Test Bot trong AWS Console

```
Console → Amazon Lex → Bot → Test
  ↓
Gõ text hoặc nói để test
  ↓
Xem:
  - Intent nhận dạng được
  - Slot values trích xuất được
  - Dialog state hiện tại
  - Session attributes
  - Lambda response (nếu có)
```

### CloudWatch Logs cho Lex

```python
# Bật conversation logs (nhật ký hội thoại) trong Bot Alias settings
{
  "conversationLogs": {
    "logSettings": [
      {
        "logType": "TEXT",
        "destination": "CloudWatch",
        "cloudWatch": {
          "cloudWatchLogGroupArn": "arn:aws:logs:us-east-1:123456:log-group:/aws/lex/TravelBot",
          "logPrefix": "TravelBot"
        }
      }
    ]
  }
}
```

---

## 🚨 Lỗi Phổ Biến & Cách Xử Lý

| Lỗi | Nguyên Nhân | Cách Xử Lý |
|-----|------------|-----------|
| Intent không nhận dạng được | Thiếu utterances đa dạng | Thêm nhiều sample utterances hơn |
| Slot trích xuất sai | Slot type không phù hợp | Dùng built-in type phù hợp hoặc custom type |
| Lambda timeout | Lambda xử lý chậm | Tăng timeout Lambda, tối ưu query DB |
| `ResourceNotFoundException` | Bot ID hoặc Alias ID sai | Kiểm tra lại botId và botAliasId |
| Session expired | Session hết hạn sau 5 phút không hoạt động | Lưu context trong DynamoDB, restore khi cần |
| `AccessDeniedException` | IAM role thiếu quyền | Thêm `lex:RecognizeText` permission |

---

## 📋 Checklist Triển Khai Lex Bot

**Design Phase (Giai Đoạn Thiết Kế):**
- [ ] Xác định danh sách tất cả intents cần có
- [ ] Với mỗi intent: liệt kê 20+ sample utterances
- [ ] Xác định slots required và optional cho mỗi intent
- [ ] Thiết kế conversation flow (luồng hội thoại) cho từng intent

**Development Phase (Giai Đoạn Phát Triển):**
- [ ] Tạo bot và configure locale (ngôn ngữ)
- [ ] Thêm intents và sample utterances
- [ ] Cấu hình slots và elicitation prompts
- [ ] Viết Lambda fulfillment function
- [ ] Thêm error handling trong Lambda

**Testing Phase (Giai Đoạn Kiểm Tra):**
- [ ] Test mỗi intent với ít nhất 10 utterances khác nhau
- [ ] Test multi-turn dialog đầy đủ
- [ ] Test FallbackIntent với câu không liên quan
- [ ] Test Lambda error handling

**Production Phase (Giai Đoạn Production):**
- [ ] Bật conversation logs trong CloudWatch
- [ ] Configure CloudWatch alarms cho error rate
- [ ] Thiết lập bot versioning và aliases
- [ ] Document (ghi lại tài liệu) all intents và slots

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Module | [README.md](./README.md) — Tổng quan Conversational AI |
| → Tiếp theo | [2-lex-advanced.md](./2-lex-advanced.md) — Lex nâng cao |
| ↑ Chỉ mục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03 | **Phiên Bản:** 1.0
