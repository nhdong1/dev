# Amazon Transcribe Call Analytics — Phân Tích Cuộc Gọi Tổng Đài

> Amazon Transcribe Call Analytics — Phân Tích Cuộc Gọi là dịch vụ chuyên biệt của Transcribe, được thiết kế riêng cho **contact center** (trung tâm liên lạc / tổng đài). Ngoài việc phiên âm, dịch vụ tự động phân tích cảm xúc, phát hiện vấn đề, phân loại cuộc gọi, đo lường thời gian im lặng và gián đoạn — tất cả không cần ML expertise.

---

## 🎯 Call Analytics Làm Được Gì?

| Tính Năng | Mô Tả |
|---|---|
| **Transcription** (Phiên Âm) | Phiên âm đầy đủ với phân biệt agent/customer |
| **Sentiment Analysis** (Phân Tích Cảm Xúc) | Cảm xúc theo từng đoạn và tổng thể cho agent và customer |
| **Issue Detection** (Phát Hiện Vấn Đề) | Tự động tóm tắt vấn đề chính của cuộc gọi |
| **Action Items** (Việc Cần Làm) | Tự động trích xuất cam kết và việc cần làm sau cuộc gọi |
| **Outcome** (Kết Quả) | Xác định kết quả cuộc gọi (resolved, unresolved...) |
| **Categories** (Phân Loại) | Tự định nghĩa rules để phân loại cuộc gọi |
| **Interruptions** (Gián Đoạn) | Đếm số lần agent/customer ngắt lời nhau |
| **Non-talk Time** (Thời Gian Im Lặng) | Đo tổng thời gian im lặng trong cuộc gọi |
| **Talk Speed** (Tốc Độ Nói) | Tốc độ nói của agent (từ/phút) |
| **PII Redaction** (Ẩn Danh PII) | Tự động che thông tin cá nhân nhạy cảm |

---

## 📞 Yêu Cầu Đặc Biệt: 2-Channel Audio (Âm Thanh 2 Kênh)

Call Analytics yêu cầu file audio **stereo 2 kênh** (channel 0 và channel 1 tách biệt):
- **Channel 0**: Giọng của một bên (thường là agent — nhân viên tổng đài)
- **Channel 1**: Giọng của bên kia (thường là customer — khách hàng)

> **Lý do:** Tách riêng 2 kênh giúp phân tích cảm xúc, tốc độ nói, gián đoạn chính xác cho từng bên riêng biệt.

```
Hệ thống điện thoại / UCaaS
      │
      ├──► Kênh 0: Agent audio ──┐
      │                          ├──► Stereo WAV/MP3 → S3
      └──► Kênh 1: Customer audio┘
```

---

## 🔧 1. Batch Call Analytics (Phân Tích Cuộc Gọi Theo Lô)

### Code Mẫu — Khởi Động Job

```python
import boto3
import json

transcribe = boto3.client("transcribe", region_name="us-east-1")

def start_call_analytics_job(
    audio_s3_uri: str,
    job_name: str,
    output_s3_uri: str,
    data_access_role_arn: str,
    language_code: str = "en-US"
) -> str:
    """Khởi động Call Analytics job."""
    response = transcribe.start_call_analytics_job(
        CallAnalyticsJobName=job_name,
        Media={"MediaFileUri": audio_s3_uri},
        OutputLocation=output_s3_uri,          # s3://bucket/output/
        DataAccessRoleArn=data_access_role_arn,
        LanguageCode=language_code,
        ChannelDefinitions=[
            {
                "ChannelId": 0,
                "ParticipantRole": "AGENT"     # Kênh 0 = Agent (nhân viên)
            },
            {
                "ChannelId": 1,
                "ParticipantRole": "CUSTOMER"  # Kênh 1 = Customer (khách hàng)
            }
        ]
    )
    return response["CallAnalyticsJob"]["CallAnalyticsJobName"]


def get_call_analytics_result(job_name: str) -> dict:
    """Lấy kết quả phân tích cuộc gọi."""
    response = transcribe.get_call_analytics_job(CallAnalyticsJobName=job_name)
    job = response["CallAnalyticsJob"]

    if job["CallAnalyticsJobStatus"] == "COMPLETED":
        return job
    elif job["CallAnalyticsJobStatus"] == "FAILED":
        raise RuntimeError(f"Job thất bại: {job.get('FailureReason')}")
    return None
```

### Đọc Kết Quả Phân Tích

```python
def parse_call_analytics_result(result_json: dict) -> dict:
    """Phân tích kết quả Call Analytics JSON."""

    summary = {
        "transcript": [],
        "sentiment_by_participant": {},
        "issues": [],
        "action_items": [],
        "outcomes": [],
        "statistics": {}
    }

    # Transcript với phân biệt agent/customer
    for segment in result_json.get("Transcript", []):
        summary["transcript"].append({
            "speaker": segment["ParticipantRole"],    # "AGENT" hoặc "CUSTOMER"
            "content": segment["Content"],
            "begin_offset_ms": segment["BeginOffsetMillis"],
            "end_offset_ms": segment["EndOffsetMillis"],
            "sentiment": segment.get("Sentiment")     # Cảm xúc đoạn này
        })

    # Cảm xúc tổng thể từng bên
    for category in result_json.get("Categories", {}).get("MatchedDetails", {}).values():
        pass

    # Issues, Action Items, Outcomes (từ phần Conversation Characteristics)
    characteristics = result_json.get("ConversationCharacteristics", {})

    # Thống kê gián đoạn
    interruptions = characteristics.get("Interruptions", {})
    summary["statistics"]["total_interruptions"] = interruptions.get("TotalCount", 0)

    # Thời gian im lặng
    non_talk_time = characteristics.get("NonTalkTime", {})
    summary["statistics"]["non_talk_time_ms"] = non_talk_time.get("TotalTimeMillis", 0)

    # Tốc độ nói (agent)
    talk_speed = characteristics.get("TalkSpeed", {}).get("DetailsByParticipant", {})
    if "AGENT" in talk_speed:
        summary["statistics"]["agent_words_per_minute"] = talk_speed["AGENT"].get("AverageWordsPerMinute")

    return summary
```

---

## 💬 2. Sentiment Analysis (Phân Tích Cảm Xúc) trong Call Analytics

Sentiment được phân tích ở **3 cấp độ**:
1. **Segment-level** (Cấp Đoạn): Cảm xúc từng đoạn lời nói nhỏ
2. **Participant-level** (Cấp Người Tham Gia): Cảm xúc trung bình của agent và customer riêng biệt
3. **Period-level** (Cấp Giai Đoạn): Diễn biến cảm xúc theo thời gian trong cuộc gọi (đầu/giữa/cuối)

```python
def analyze_sentiment_trend(result_json: dict) -> dict:
    """Phân tích xu hướng cảm xúc trong cuộc gọi."""

    customer_sentiments = []
    agent_sentiments = []

    for segment in result_json.get("Transcript", []):
        sentiment_info = {
            "time_ms": segment["BeginOffsetMillis"],
            "sentiment": segment.get("Sentiment"),
            "positive_score": segment.get("SentimentScore", {}).get("Positive", 0),
            "negative_score": segment.get("SentimentScore", {}).get("Negative", 0)
        }

        if segment["ParticipantRole"] == "CUSTOMER":
            customer_sentiments.append(sentiment_info)
        else:
            agent_sentiments.append(sentiment_info)

    # Phát hiện escalation: customer bắt đầu NEGATIVE và ngày càng tệ hơn
    negative_segments = [
        s for s in customer_sentiments
        if s["sentiment"] == "NEGATIVE" and s["negative_score"] > 0.7
    ]

    return {
        "customer_segments": customer_sentiments,
        "agent_segments": agent_sentiments,
        "high_negative_count": len(negative_segments),
        "needs_escalation": len(negative_segments) >= 3   # Ngưỡng cảnh báo leo thang
    }
```

### Sentiment Scoring (Điểm Cảm Xúc)

- `POSITIVE` — Tích cực: khách hàng hài lòng, vui vẻ
- `NEGATIVE` — Tiêu cực: khách hàng bực bội, khiếu nại
- `NEUTRAL` — Trung lập: trao đổi thông tin bình thường
- `MIXED` — Hỗn hợp: có cả tích cực và tiêu cực

---

## 🏷️ 3. Categories (Phân Loại Cuộc Gọi)

Categories — Phân Loại cho phép bạn định nghĩa **rules** (quy tắc) để tự động gắn nhãn cuộc gọi dựa trên từ khóa, sentiment hoặc sự kiện đặc biệt.

### Các Loại Rule

| Loại Rule | Mô Tả | Ví Dụ |
|---|---|---|
| **Keyword Match** (Khớp Từ Khóa) | Transcript chứa từ/cụm từ cụ thể | "cancel", "refund", "speak to manager" |
| **Sentiment Match** (Khớp Cảm Xúc) | Cảm xúc của participant đạt mức nhất định | Customer sentiment = NEGATIVE |
| **Interruption Match** (Khớp Gián Đoạn) | Số lần gián đoạn vượt ngưỡng | Agent interrupts > 3 lần |
| **Non-talk Time Match** (Khớp Im Lặng) | Thời gian im lặng quá dài | Im lặng > 30 giây |
| **Sentiment Time Match** (Khớp Cảm Xúc Theo Thời Gian) | Cảm xúc tại thời điểm cụ thể trong cuộc gọi | NEGATIVE trong 10% cuối cuộc gọi |

### Tạo Category

```python
def create_call_category(category_name: str) -> None:
    """Tạo category để phân loại cuộc gọi theo rules."""
    transcribe.create_call_analytics_category(
        CategoryName=category_name,
        Rules=[
            {
                # Rule 1: Khách hàng đề cập từ "cancel" hoặc "refund"
                "NonTalkTimeFilter": None,
                "TranscriptFilter": {
                    "TranscriptFilterType": "EXACT",
                    "Targets": ["cancel", "refund", "money back"],
                    "ParticipantRole": "CUSTOMER",
                    "Negate": False
                }
            },
            {
                # Rule 2: Cảm xúc khách hàng là NEGATIVE trong nửa cuối cuộc gọi
                "SentimentFilter": {
                    "Sentiments": ["NEGATIVE"],
                    "ParticipantRole": "CUSTOMER",
                    "RelativeTimeRange": {
                        "StartPercentage": 50,   # Từ 50% thời gian cuộc gọi
                        "EndPercentage": 100     # Đến cuối cuộc gọi
                    }
                }
            }
        ]
    )
    print(f"Category '{category_name}' đã được tạo.")
```

### Ví Dụ Categories Hữu Ích Cho Tổng Đài

```python
USEFUL_CATEGORIES = {
    "Cancellation Request": ["cancel", "cancel subscription", "close account"],
    "Refund Request": ["refund", "money back", "charge reversal", "dispute"],
    "Escalation Request": ["speak to manager", "supervisor", "escalate", "complaint"],
    "Technical Issue": ["not working", "broken", "error", "bug", "can't login"],
    "Positive Feedback": ["great service", "very helpful", "satisfied", "thank you so much"]
}
```

---

## 🎙️ 4. Real-time Call Analytics (Phân Tích Cuộc Gọi Thời Gian Thực)

Dùng cho **live call monitoring** (giám sát cuộc gọi trực tiếp) — agent supervisor có thể can thiệp ngay khi phát hiện vấn đề.

```python
import asyncio
from amazon_transcribe.client import TranscribeStreamingClient
from amazon_transcribe.model import CallAnalyticsTranscriptEvent

async def real_time_call_analytics(audio_stream_generator):
    """Phân tích cuộc gọi theo thời gian thực."""
    client = TranscribeStreamingClient(region="us-east-1")

    stream = await client.start_call_analytics_stream_transcription(
        language_code="en-US",
        media_sample_rate_hz=8000,          # 8kHz cho điện thoại
        media_encoding="pcm",
        enable_partial_results_stabilization=True,
        partial_results_stability="medium"
    )

    async def handle_events():
        async for event in stream.output_stream:
            if isinstance(event, CallAnalyticsTranscriptEvent):
                results = event.call_analytics_transcript_event.call_analytics_results
                for result in results:
                    if not result.is_partial:
                        for alt in result.alternatives:
                            print(f"[FINAL] {alt.transcript}")

                        # Kiểm tra categories matched
                        for item in result.alternatives[0].items:
                            if hasattr(item, "category") and item.category:
                                print(f"[CATEGORY MATCHED]: {item.category}")

    await asyncio.gather(
        audio_stream_generator(stream.input_stream),
        handle_events()
    )
```

### Kiến Trúc Giám Sát Cuộc Gọi Thực Tế

```
Hệ thống điện thoại (Amazon Connect hoặc third-party)
      │ (Audio stream 2 kênh)
      ▼
Transcribe Call Analytics Streaming
      │
      ├──► WebSocket events → Lambda
      │         │
      │         ├── Category matched → SNS → Supervisor dashboard alert
      │         ├── NEGATIVE sentiment kéo dài → Supervisor gets notification
      │         └── Keyword "cancel" → Trigger retention script popup cho agent
      │
      └──► Transcript lưu vào S3 để phân tích sau
```

---

## 📊 5. Conversation Characteristics (Đặc Điểm Cuộc Hội Thoại)

Transcribe Call Analytics đo lường các chỉ số quan trọng:

### Interruptions (Gián Đoạn)

Ngắt lời nhau — chỉ số chất lượng dịch vụ:

```json
{
  "Interruptions": {
    "TotalCount": 5,
    "InterruptionsByInterrupter": {
      "AGENT": { "Count": 2 },
      "CUSTOMER": { "Count": 3 }
    }
  }
}
```

> Agent ngắt lời khách hàng nhiều → cần đào tạo kỹ năng lắng nghe. Khách hàng ngắt lời agent nhiều → có thể agent nói quá dài, cần cải thiện script.

### Non-talk Time (Thời Gian Im Lặng)

```json
{
  "NonTalkTime": {
    "TotalTimeMillis": 45000,
    "Instances": [
      { "BeginOffsetMillis": 12000, "EndOffsetMillis": 22000, "DurationMillis": 10000 }
    ]
  }
}
```

> Im lặng dài trong cuộc gọi → agent cần kiểm tra hệ thống, chờ xử lý — cần tối ưu quy trình.

### Talk Speed (Tốc Độ Nói)

```json
{
  "TalkSpeed": {
    "DetailsByParticipant": {
      "AGENT": { "AverageWordsPerMinute": 178 },
      "CUSTOMER": { "AverageWordsPerMinute": 134 }
    }
  }
}
```

> Agent nói quá nhanh (> 200 wpm) → khách khó hiểu, cần chậm lại.

### Talk Time (Tỉ Lệ Thời Gian Nói)

```json
{
  "TalkTime": {
    "DetailsByParticipant": {
      "AGENT": { "TotalTimeMillis": 120000 },
      "CUSTOMER": { "TotalTimeMillis": 95000 }
    }
  }
}
```

> Agent nói nhiều hơn khách hàng → có thể bình thường; nhưng nếu khách hàng gần như không nói gì → vấn đề.

---

## 🔒 6. PII Redaction (Ẩn Danh Thông Tin Cá Nhân)

Tương tự Transcribe thường nhưng được áp dụng riêng cho cả 2 kênh agent và customer:

```python
response = transcribe.start_call_analytics_job(
    CallAnalyticsJobName="analytics-with-redaction",
    Media={"MediaFileUri": "s3://bucket/call.mp3"},
    OutputLocation="s3://output-bucket/",
    DataAccessRoleArn="arn:aws:iam::123456789012:role/TranscribeRole",
    LanguageCode="en-US",
    ChannelDefinitions=[
        {"ChannelId": 0, "ParticipantRole": "AGENT"},
        {"ChannelId": 1, "ParticipantRole": "CUSTOMER"}
    ],
    Settings={
        "ContentRedaction": {
            "RedactionType": "PII",
            "RedactionOutput": "redacted_and_unredacted",  # Tạo cả 2 phiên bản
            "PiiEntityTypes": ["CREDIT_DEBIT_NUMBER", "PHONE", "SSN", "NAME"]
        }
    }
)
```

---

## 🏗️ Kiến Trúc End-to-End Contact Center Analytics

```
Amazon Connect (Tổng Đài Đám Mây)
      │
      ├── Real-time: AudioStream → Kinesis Video Streams
      │       │
      │       ▼
      │   Transcribe Call Analytics Streaming
      │       │
      │       ├── Category matched → Lambda → CRM note / Supervisor alert
      │       └── Live sentiment → Agent Assist dashboard (gợi ý script)
      │
      └── Post-call: Recording → S3
              │
              ▼
          Transcribe Call Analytics Batch
              │
              ├── Full transcript + sentiment JSON
              ├── Issues, Action Items, Outcomes
              └── Categories matched
                      │
                      ▼
              DynamoDB (lưu metadata)
              Redshift (phân tích trend)
              QuickSight (dashboard BI — Business Intelligence)
                      │
                      ▼
              KPI Dashboard:
              - CSAT (Customer Satisfaction Score — Điểm Hài Lòng Khách Hàng)
              - AHT (Average Handle Time — Thời Gian Xử Lý Trung Bình)
              - FCR (First Call Resolution — Giải Quyết Ngay Lần Đầu)
              - Agent performance score (Điểm Hiệu Suất Agent)
```

---

## 💰 Chi Phí Và Tối Ưu

| Tình Huống | Chi Phí | Tối Ưu |
|---|---|---|
| Batch Call Analytics | ~$0.03/phút audio | Chỉ phân tích cuộc gọi dài hơn 30 giây |
| Real-time Call Analytics | ~$0.04/phút audio | Bật/tắt streaming theo cấu hình supervisor |
| Lưu trữ recordings | S3 Standard ~$0.023/GB/tháng | S3 Intelligent-Tiering cho recordings cũ |

> **Lưu ý:** Transcribe Call Analytics đắt hơn Transcribe thông thường do bao gồm các tính năng phân tích bổ sung. Đánh giá kỹ có cần toàn bộ tính năng hay chỉ cần phiên âm đơn giản.

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Transcribe Call Analytics khác Transcribe thông thường thế nào?**

> Call Analytics chuyên biệt cho contact center: yêu cầu audio 2 kênh (agent/customer riêng biệt), tự động phân tích sentiment theo từng bên, phát hiện issues/action items, đo interruptions/non-talk time, hỗ trợ custom categories. Transcribe thông thường chỉ phiên âm và diarization chung.

**Q: Tại sao Call Analytics yêu cầu audio 2 kênh?**

> Tách giọng agent và customer thành 2 kênh riêng cho phép phân tích chính xác sentiment và behavior của từng bên mà không bị lẫn lộn. Nếu chỉ có mono audio (1 kênh), không thể biết agent đang bực bội hay customer — dẫn đến phân tích không có giá trị.

**Q: Issue Detection và Action Items được tạo ra như thế nào?**

> Transcribe Call Analytics dùng NLP model được huấn luyện riêng trên hàng triệu cuộc gọi tổng đài để tự động nhận dạng vấn đề chính được đề cập và các cam kết/việc cần làm. Đây là ML-generated, không phải rule-based — nên đôi khi cần human review.

**Q: Làm thế nào để cảnh báo supervisor khi phát hiện cuộc gọi có vấn đề?**

> Dùng Real-time Call Analytics + Category Rules để phát hiện ngay khi keyword/sentiment match. Kết nối với Lambda → SNS → email/Slack/CRM để thông báo supervisor trong vài giây.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
