# NLP & Text AI (Xử Lý Ngôn Ngữ Tự Nhiên) trên AWS — Toàn Diện

> NLP — Natural Language Processing — Xử Lý Ngôn Ngữ Tự Nhiên là lĩnh vực AI giúp máy tính hiểu, phân tích và tạo ra ngôn ngữ con người. Trên AWS, ba dịch vụ chủ lực là **Amazon Comprehend** (Hiểu Ngôn Ngữ Tự Nhiên), **Amazon Comprehend Medical** (Comprehend Chuyên Biệt Y Tế) và **Amazon Translate** (Dịch Thuật Máy Thần Kinh) — đều là AI Services được quản lý hoàn toàn (fully-managed), gọi qua API, không cần ML expertise (chuyên môn học máy).

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|---|---|---|
| `README.md` | Tổng quan NLP trên AWS, so sánh dịch vụ | ✅ |
| `1-comprehend-fundamentals.md` | Sentiment, Entities, Key phrases, Language detection | ✅ |
| `2-comprehend-custom.md` | Custom Classification, Custom NER, training data | ✅ |
| `3-comprehend-medical.md` | Medical entities, ICD-10-CM, RxNorm, PHI detection | ✅ |
| `4-translate.md` | Real-time & batch translation, Custom terminology | ✅ |

---

## 🎯 NLP Là Gì Và Dùng Để Làm Gì?

NLP (Natural Language Processing — Xử Lý Ngôn Ngữ Tự Nhiên) cho phép máy tính phân tích và rút trích thông tin có nghĩa từ văn bản:

- **Sentiment Analysis** (Phân Tích Cảm Xúc / Tình Cảm): Phân loại văn bản thành Positive (tích cực), Negative (tiêu cực), Neutral (trung lập), Mixed (hỗn hợp)
- **Named Entity Recognition — NER** (Nhận Dạng Thực Thể Có Tên): Tìm và phân loại các thực thể như người, địa điểm, tổ chức, ngày tháng trong văn bản
- **Key Phrase Extraction** (Trích Xuất Cụm Từ Khóa): Tìm các cụm từ quan trọng nhất trong văn bản
- **Language Detection** (Phát Hiện Ngôn Ngữ): Tự động xác định ngôn ngữ của văn bản
- **Text Classification** (Phân Loại Văn Bản): Gán văn bản vào các danh mục đã định nghĩa
- **Machine Translation** (Dịch Thuật Máy): Tự động dịch từ ngôn ngữ này sang ngôn ngữ khác

### Vị Trí Trong Hệ Sinh Thái AWS AI

```
┌─────────────────────────────────────────────────────────────────┐
│  Tầng 3: AI Services (Dịch Vụ AI Được Quản Lý)  ◄── BẠN Ở ĐÂY │
│  Comprehend │ Comprehend Medical │ Translate │ Rekognition       │
├─────────────────────────────────────────────────────────────────┤
│  Tầng 2: ML Services                                            │
│   Amazon SageMaker (dùng khi cần custom NLP model phức tạp)    │
├─────────────────────────────────────────────────────────────────┤
│  Tầng 1: ML Framework & Infrastructure                          │
│  HuggingFace Transformers │ PyTorch │ spaCy │ GPU Instances     │
└─────────────────────────────────────────────────────────────────┘
```

**Quy tắc chọn:** Dùng AI Services (Comprehend/Translate) khi bài toán nằm trong khả năng có sẵn — nhanh, không cần data, không cần ML expertise. Chỉ dùng SageMaker custom model khi yêu cầu rất đặc thù.

---

## 🗂️ So Sánh Dịch Vụ NLP Trên AWS

| Dịch Vụ | Mục Đích | Input | Output | Tính Phí |
|---|---|---|---|---|
| **Comprehend** | Phân tích ngôn ngữ tổng quát | Văn bản (UTF-8) | Sentiment, Entities, Key phrases, Topics | Per 100 ký tự |
| **Comprehend Custom** | NLP tùy chỉnh (phân loại / NER) | Văn bản + dữ liệu huấn luyện | Custom labels / custom entities | Training-hour + Endpoint-hour |
| **Comprehend Medical** | Phân tích văn bản y tế | Ghi chú lâm sàng (tiếng Anh) | Medical entities, ICD-10, RxNorm, PHI | Per 100 ký tự |
| **Translate** | Dịch thuật đa ngôn ngữ | Văn bản / tài liệu | Văn bản đã dịch | Per ký tự dịch |

### Khi Nào Dùng Cái Nào?

```
Bạn cần xử lý gì?
│
├─ Phân tích cảm xúc, tìm thực thể, từ khóa trong văn bản tổng quát?
│   └─► Comprehend (detect-sentiment, detect-entities, detect-key-phrases)
│
├─ Phân loại văn bản theo danh mục riêng của bạn?
│   └─► Comprehend Custom Classification (huấn luyện với dữ liệu riêng)
│
├─ Tìm thực thể đặc thù của domain bạn (VD: tên sản phẩm, mã hàng)?
│   └─► Comprehend Custom Entity Recognition
│
├─ Văn bản y tế — bệnh án, toa thuốc, ghi chú lâm sàng?
│   └─► Comprehend Medical (phát hiện entity y tế + ICD-10-CM + RxNorm)
│
├─ Dịch văn bản sang ngôn ngữ khác (real-time hoặc batch)?
│   └─► Amazon Translate (hỗ trợ 75+ ngôn ngữ)
│
└─ Bài toán NLP rất đặc thù (VD: fine-tune BERT cho domain riêng)?
    └─► SageMaker với HuggingFace container
```

---

## 🆚 Comprehend vs Bedrock/LLM cho NLP

| Tiêu Chí | Amazon Comprehend | Bedrock / LLM |
|---|---|---|
| **Tốc độ** | Rất nhanh, latency thấp | Chậm hơn (generation) |
| **Chi phí** | Thấp (per ký tự) | Cao hơn (per token) |
| **Độ chính xác NER** | Cao cho entity tiêu chuẩn | Linh hoạt hơn |
| **Custom training** | Có (Custom Classification / NER) | Fine-tuning hoặc prompting |
| **Ngôn ngữ hỗ trợ** | Đa ngôn ngữ (100+) | Phụ thuộc model |
| **Phù hợp** | Phân tích hàng loạt, production pipeline | Bài toán phức tạp, open-ended |

> **Quy tắc thực tế:** Comprehend cho **structured NLP tasks** (phân loại, NER, sentiment) ở quy mô lớn; Bedrock/LLM cho **open-ended tasks** (tóm tắt, hỏi đáp, tạo nội dung).

---

## 🚀 Quick Start — Gọi Comprehend & Translate

### Cài Đặt Client (boto3)

```python
import boto3

# Comprehend client (phân tích văn bản)
comprehend = boto3.client("comprehend", region_name="us-east-1")

# Translate client (dịch thuật)
translate = boto3.client("translate", region_name="us-east-1")
```

### Phân Tích Cảm Xúc (Detect Sentiment)

```python
def analyze_sentiment(text: str, language_code: str = "en") -> dict:
    """Phân tích cảm xúc của đoạn văn bản."""
    response = comprehend.detect_sentiment(
        Text=text,
        LanguageCode=language_code   # "en", "vi" (tiếng Việt), "ja", "zh"...
    )
    return {
        "sentiment": response["Sentiment"],                              # POSITIVE / NEGATIVE / NEUTRAL / MIXED
        "scores": response["SentimentScore"]                             # Dict với điểm xác suất từng loại
    }

result = analyze_sentiment("AWS AI services are incredibly powerful!")
print(result["sentiment"])       # → POSITIVE
```

### Dịch Văn Bản (Translate Text)

```python
def translate_text(text: str, source_lang: str, target_lang: str) -> str:
    """Dịch văn bản từ ngôn ngữ này sang ngôn ngữ khác."""
    response = translate.translate_text(
        Text=text,
        SourceLanguageCode=source_lang,  # "auto" để tự phát hiện
        TargetLanguageCode=target_lang   # "vi", "en", "ja", "fr"...
    )
    return response["TranslatedText"]

translated = translate_text("Hello, how are you?", "en", "vi")
print(translated)    # → "Xin chào, bạn có khỏe không?"
```

---

## 🔑 Khái Niệm Chung Cần Nắm

### Language Code (Mã Ngôn Ngữ)

Comprehend và Translate đều dùng chuẩn **BCP-47** (ví dụ: `en`, `vi`, `ja`, `zh`, `fr`, `de`). Nếu không biết ngôn ngữ của văn bản đầu vào, dùng `detect_dominant_language` trước hoặc đặt `SourceLanguageCode="auto"` trong Translate.

### Batch vs Real-time (Theo Lô vs Thời Gian Thực)

| Chế Độ | API Comprehend | API Translate | Phù Hợp |
|---|---|---|---|
| **Real-time** (Thời Gian Thực) | `detect_*` / `batch_detect_*` | `translate_text` | < 5.000 ký tự, ít tài liệu |
| **Async Batch** (Xử Lý Theo Lô) | `start_*_analysis_job` | `start_text_translation_job` | Hàng nghìn tài liệu lớn |

Batch Job lưu kết quả vào **S3**, dùng khi cần xử lý khối lượng lớn.

### Confidence Score (Điểm Tin Cậy)

Hầu hết kết quả Comprehend đều kèm `Score` (0–1) — mức độ chắc chắn của model. Đặt ngưỡng phù hợp với use case để lọc kết quả:
- Automation hoàn toàn: ngưỡng cao (0.9+)
- Hỗ trợ con người: ngưỡng thấp hơn để không bỏ sót

---

## 📊 Kiến Trúc Điển Hình

### Pipeline Phân Tích Feedback Khách Hàng

```
Khách hàng gửi feedback (form / email / chat)
      │
      ▼
S3 Bucket  ──(S3 Event)──►  Lambda Function
                                  │
                                  ├──► Comprehend detect-sentiment
                                  ├──► Comprehend detect-entities (tìm sản phẩm được đề cập)
                                  ├──► Comprehend detect-key-phrases
                                  │
                                  ▼
                          DynamoDB (lưu kết quả phân tích)
                                  │
                          ┌───────┴────────┐
                          ▼               ▼
                    QuickSight        SNS Alert
                 (dashboard BI)   (nếu sentiment âm)
```

### Pipeline Đa Ngôn Ngữ (Multi-language)

```
Tài liệu ngôn ngữ bất kỳ
      │
      ▼
Translate (auto-detect → dịch sang tiếng Anh)
      │
      ▼
Comprehend (phân tích trên bản tiếng Anh)
      │
      ▼
Kết quả: Sentiment, Entities, Key phrases
      │
      ▼
Translate (dịch kết quả text lại ngôn ngữ gốc nếu cần)
```

> **Lưu ý:** Comprehend hỗ trợ đa ngôn ngữ trực tiếp, nhưng độ chính xác cao nhất là tiếng Anh. Với ngôn ngữ ít phổ biến, dịch sang tiếng Anh trước sẽ cho kết quả tốt hơn.

### Pipeline Xử Lý Hồ Sơ Y Tế

```
Ghi chú lâm sàng (bác sĩ nhập tay hoặc scan → Textract OCR)
      │
      ▼
Comprehend Medical
      ├──► Phát hiện Medical Entities (MEDICATION, CONDITION, PROCEDURE...)
      ├──► Map sang ICD-10-CM (mã bệnh quốc tế)
      ├──► Map sang RxNorm (mã thuốc chuẩn)
      └──► PHI Detection & De-identification (ẩn danh hóa thông tin cá nhân)
            │
            ▼
      Dữ liệu đã ẩn danh → FHIR / Data Warehouse / Analytics
```

---

## 💰 Lưu Ý Chi Phí

| Dịch Vụ | Mô Hình Giá | Mẹo Tiết Kiệm |
|---|---|---|
| Comprehend built-in | Per 100 ký tự (tối thiểu 300 ký tự/request) | Batch nhiều text vào một request |
| Comprehend Custom | Training-hour + Endpoint-hour hoặc Async-job | **Tắt endpoint (delete) khi không dùng** — tính phí liên tục |
| Comprehend Medical | Per 100 ký tự (đắt hơn Comprehend thường) | Chỉ gọi với văn bản y tế thực sự |
| Translate | Per ký tự dịch | Dùng Custom Terminology đúng lần đầu để tránh dịch lại |

> **Free Tier:** Comprehend miễn phí 50.000 đơn vị/tháng (12 tháng đầu); Translate miễn phí 2 triệu ký tự/tháng (12 tháng đầu).

---

## 🔗 Điều Hướng Module

| Chủ Đề | File |
|---|---|
| Comprehend fundamentals — Sentiment, Entities, Key phrases | [1-comprehend-fundamentals.md](1-comprehend-fundamentals.md) |
| Comprehend Custom — phân loại và NER tùy chỉnh | [2-comprehend-custom.md](2-comprehend-custom.md) |
| Comprehend Medical — entity y tế, ICD-10-CM, PHI | [3-comprehend-medical.md](3-comprehend-medical.md) |
| Amazon Translate — dịch thuật đa ngôn ngữ | [4-translate.md](4-translate.md) |

---

## 🎯 Câu Hỏi Phỏng Vấn Nhanh

1. **Comprehend vs Textract — khác nhau thế nào?** → Comprehend phân tích *ý nghĩa ngôn ngữ* trong văn bản (sentiment, entities, topics); Textract *trích xuất văn bản* từ tài liệu scan (OCR + form + table). Thường kết hợp: Textract lấy text → Comprehend phân tích.

2. **Khi nào dùng Comprehend Medical thay vì Comprehend thường?** → Khi văn bản là ghi chú lâm sàng, bệnh án, toa thuốc — Comprehend Medical được huấn luyện riêng để hiểu thuật ngữ y tế và map sang chuẩn ICD-10-CM/RxNorm.

3. **Amazon Translate hỗ trợ bao nhiêu ngôn ngữ?** → 75+ ngôn ngữ, dịch theo cặp. Có thể tự phát hiện ngôn ngữ nguồn bằng `SourceLanguageCode="auto"`.

4. **Custom Terminology trong Translate là gì?** → Cho phép bạn định nghĩa bản dịch bắt buộc cho các thuật ngữ chuyên ngành (VD: tên thương hiệu, thuật ngữ kỹ thuật) — Translate sẽ ưu tiên dùng từ điển này thay vì dịch tự động.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
