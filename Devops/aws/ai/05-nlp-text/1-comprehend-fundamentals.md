# Amazon Comprehend — Phân Tích Ngôn Ngữ Tự Nhiên Cơ Bản

> Amazon Comprehend là dịch vụ NLP — Natural Language Processing — Xử Lý Ngôn Ngữ Tự Nhiên được quản lý hoàn toàn (fully-managed) của AWS. Dịch vụ sử dụng các model deep learning (học sâu) đã được pre-trained (huấn luyện sẵn) để phân tích văn bản: nhận diện cảm xúc, thực thể, cụm từ khóa, ngôn ngữ, chủ đề — chỉ qua lời gọi API, không cần ML expertise (chuyên môn học máy).

---

## 🎯 Amazon Comprehend Làm Được Gì?

| Tính Năng | API | Mô Tả |
|---|---|---|
| **Sentiment Analysis** (Phân Tích Cảm Xúc) | `detect_sentiment` | POSITIVE / NEGATIVE / NEUTRAL / MIXED |
| **Entity Recognition** (Nhận Dạng Thực Thể) | `detect_entities` | Người, địa điểm, tổ chức, ngày, số lượng... |
| **Key Phrase Extraction** (Trích Xuất Cụm Từ Khóa) | `detect_key_phrases` | Cụm từ quan trọng nhất trong văn bản |
| **Language Detection** (Phát Hiện Ngôn Ngữ) | `detect_dominant_language` | Xác định ngôn ngữ, hỗ trợ 100+ ngôn ngữ |
| **Syntax Analysis** (Phân Tích Cú Pháp) | `detect_syntax` | POS tag — Part-of-Speech — Nhãn Từ Loại |
| **Targeted Sentiment** (Cảm Xúc Theo Đối Tượng) | `detect_targeted_sentiment` | Cảm xúc hướng đến từng thực thể cụ thể |
| **Batch Processing** (Xử Lý Theo Lô) | `batch_detect_*` | Xử lý tối đa 25 văn bản trong 1 request |
| **Async Jobs** (Tác Vụ Bất Đồng Bộ) | `start_*_analysis_job` | Phân tích hàng nghìn tài liệu lớn từ S3 |

---

## 😊 1. Sentiment Analysis (Phân Tích Cảm Xúc)

Phân loại văn bản theo 4 loại cảm xúc: POSITIVE (tích cực), NEGATIVE (tiêu cực), NEUTRAL (trung lập), MIXED (hỗn hợp — vừa tích cực vừa tiêu cực).

```python
import boto3

comprehend = boto3.client("comprehend", region_name="us-east-1")

def detect_sentiment(text: str, language_code: str = "en") -> dict:
    """Phân tích cảm xúc tổng thể của văn bản."""
    response = comprehend.detect_sentiment(
        Text=text,
        LanguageCode=language_code
    )
    return {
        "sentiment": response["Sentiment"],            # Nhãn cảm xúc chính
        "scores": response["SentimentScore"]           # Điểm xác suất từng loại (tổng ≈ 1.0)
    }

# Ví dụ
result = detect_sentiment("The product quality is great but delivery was very slow.")
print(result["sentiment"])     # → MIXED
print(result["scores"])
# {
#   "Positive": 0.48,
#   "Negative": 0.40,
#   "Neutral": 0.08,
#   "Mixed": 0.04
# }
```

### Cách Đọc SentimentScore

- `Sentiment` trả về nhãn có xác suất **cao nhất** trong `SentimentScore`
- Trường hợp MIXED: cả `Positive` và `Negative` đều cao; đây là văn bản có cả hai chiều cảm xúc
- Ngưỡng quyết định: nếu cần phân biệt rõ hơn, tự đặt threshold (ngưỡng) trên `SentimentScore` thay vì dùng nhãn mặc định

### Ngôn Ngữ Được Hỗ Trợ Cho Sentiment

`en` (Anh), `es` (Tây Ban Nha), `fr` (Pháp), `de` (Đức), `it` (Ý), `pt` (Bồ Đào Nha), `ar` (Ả Rập), `hi` (Hindi), `ja` (Nhật), `ko` (Hàn), `zh` (Trung giản thể), `zh-TW` (Trung phồn thể).

> **Lưu ý:** Tiếng Việt (`vi`) **chưa được Comprehend hỗ trợ** cho Sentiment. Cần dịch sang tiếng Anh bằng Amazon Translate trước.

---

## 🏷️ 2. Entity Recognition (Nhận Dạng Thực Thể)

Phát hiện và phân loại các **named entities** (thực thể có tên) trong văn bản — người, tổ chức, địa điểm, ngày tháng, số lượng, v.v.

```python
def detect_entities(text: str, language_code: str = "en") -> list:
    """Phát hiện các thực thể (người, địa điểm, tổ chức...) trong văn bản."""
    response = comprehend.detect_entities(
        Text=text,
        LanguageCode=language_code
    )

    for entity in response["Entities"]:
        print(
            f"[{entity['Type']}] '{entity['Text']}' "
            f"(tin cậy: {entity['Score']:.2f}, "
            f"vị trí: {entity['BeginOffset']}-{entity['EndOffset']})"
        )
    return response["Entities"]

detect_entities("Jeff Bezos founded Amazon in Bellevue, Washington in 1994.")
# → [PERSON] 'Jeff Bezos' (tin cậy: 0.99, vị trí: 0-10)
# → [ORGANIZATION] 'Amazon' (tin cậy: 0.98, vị trí: 19-25)
# → [LOCATION] 'Bellevue, Washington' (tin cậy: 0.97, vị trí: 29-49)
# → [DATE] '1994' (tin cậy: 0.96, vị trí: 53-57)
```

### Các Loại Entity Được Hỗ Trợ

| Loại Entity | Ví Dụ |
|---|---|
| `PERSON` | "Jeff Bezos", "Elon Musk" |
| `ORGANIZATION` | "Amazon", "AWS", "Google" |
| `LOCATION` | "Seattle", "Việt Nam", "số 1 Lý Thái Tổ" |
| `DATE` | "January 2024", "hôm qua", "ngày 3/6" |
| `QUANTITY` | "5 triệu USD", "100 đơn vị" |
| `EVENT` | "World Cup 2026", "re:Invent" |
| `TITLE` | "CEO", "Giám Đốc Điều Hành" |
| `COMMERCIAL_ITEM` | "iPhone 15 Pro", "Toyota Camry" |
| `OTHER` | Thực thể không thuộc các loại trên |

### BeginOffset và EndOffset

`BeginOffset` / `EndOffset` là vị trí ký tự (character index) trong chuỗi gốc — dùng để highlight (tô sáng) entity trong UI hoặc trích xuất context xung quanh.

---

## 🔑 3. Key Phrase Extraction (Trích Xuất Cụm Từ Khóa)

Tìm các **cụm từ quan trọng** — những danh từ và noun phrase (cụm danh từ) mang ý nghĩa chính trong văn bản.

```python
def detect_key_phrases(text: str, language_code: str = "en") -> list:
    """Trích xuất các cụm từ khóa trong văn bản."""
    response = comprehend.detect_key_phrases(
        Text=text,
        LanguageCode=language_code
    )

    phrases = [
        {"text": kp["Text"], "score": kp["Score"]}
        for kp in response["KeyPhrases"]
        if kp["Score"] >= 0.9    # Chỉ lấy cụm từ có độ tin cậy cao
    ]
    return phrases

result = detect_key_phrases(
    "Amazon Web Services offers a comprehensive suite of cloud computing services "
    "including machine learning, storage, and database solutions."
)
# → [
#     {"text": "Amazon Web Services", "score": 0.99},
#     {"text": "a comprehensive suite", "score": 0.98},
#     {"text": "cloud computing services", "score": 0.97},
#     {"text": "machine learning", "score": 0.99},
#     {"text": "database solutions", "score": 0.95}
# ]
```

### Ứng Dụng Thực Tế Của Key Phrase Extraction

- **Auto-tagging** (Gắn Thẻ Tự Động): Tự động thêm tags cho bài viết, sản phẩm, ticket hỗ trợ
- **Search indexing** (Lập Chỉ Mục Tìm Kiếm): Xây dựng inverted index (chỉ mục đảo) từ key phrases để tìm kiếm nhanh
- **Content summarization** (Tóm Tắt Nội Dung): Kết hợp với LLM để tóm tắt dựa trên key phrases
- **Trending topics** (Chủ Đề Thịnh Hành): Aggregate key phrases từ nhiều tài liệu để tìm topic phổ biến

---

## 🌍 4. Language Detection (Phát Hiện Ngôn Ngữ)

Tự động xác định ngôn ngữ của văn bản — hỗ trợ hơn **100 ngôn ngữ**.

```python
def detect_language(text: str) -> list:
    """Phát hiện ngôn ngữ của văn bản, trả về danh sách xếp hạng."""
    response = comprehend.detect_dominant_language(Text=text)

    for lang in response["Languages"]:
        print(f"Ngôn ngữ: {lang['LanguageCode']}, Điểm: {lang['Score']:.3f}")
    return response["Languages"]

detect_language("Xin chào, tôi là người Việt Nam.")
# → Ngôn ngữ: vi, Điểm: 0.998

detect_language("Bonjour, comment allez-vous?")
# → Ngôn ngữ: fr, Điểm: 0.999
```

### Kết Hợp Language Detection + Translate + Comprehend

```python
def analyze_any_language(text: str) -> dict:
    """Phân tích văn bản ở bất kỳ ngôn ngữ nào."""
    translate_client = boto3.client("translate")

    # Bước 1: Phát hiện ngôn ngữ
    lang_response = comprehend.detect_dominant_language(Text=text)
    source_lang = lang_response["Languages"][0]["LanguageCode"]

    # Bước 2: Dịch sang tiếng Anh nếu cần
    if source_lang != "en":
        translated = translate_client.translate_text(
            Text=text,
            SourceLanguageCode=source_lang,
            TargetLanguageCode="en"
        )["TranslatedText"]
    else:
        translated = text

    # Bước 3: Phân tích bằng Comprehend (tiếng Anh)
    sentiment = comprehend.detect_sentiment(Text=translated, LanguageCode="en")
    entities = comprehend.detect_entities(Text=translated, LanguageCode="en")

    return {
        "original_language": source_lang,
        "sentiment": sentiment["Sentiment"],
        "entities": [e["Text"] for e in entities["Entities"]]
    }
```

---

## 🔤 5. Syntax Analysis (Phân Tích Cú Pháp)

Phân tách văn bản thành từng **token** (đơn vị từ) và gán **POS tag** (Part-of-Speech — Nhãn Từ Loại) cho mỗi từ.

```python
def detect_syntax(text: str, language_code: str = "en") -> None:
    """Phân tích cú pháp — xác định từ loại của từng từ."""
    response = comprehend.detect_syntax(
        Text=text,
        LanguageCode=language_code
    )

    for token in response["SyntaxTokens"]:
        print(
            f"'{token['Text']}' → {token['PartOfSpeech']['Tag']} "
            f"({token['PartOfSpeech']['Score']:.2f})"
        )

detect_syntax("AWS quickly processes millions of documents.")
# → 'AWS' → PROPN (0.99)       — Proper Noun (Danh Từ Riêng)
# → 'quickly' → ADV (0.98)     — Adverb (Trạng Từ)
# → 'processes' → VERB (0.97)  — Verb (Động Từ)
# → 'millions' → NUM (0.99)    — Numeral (Số)
# → 'documents' → NOUN (0.99)  — Noun (Danh Từ)
```

### Các POS Tag Phổ Biến

| Tag | Từ Loại (Tiếng Việt) | Ví Dụ |
|---|---|---|
| `NOUN` | Danh Từ | "document", "model" |
| `VERB` | Động Từ | "analyze", "train" |
| `ADJ` | Tính Từ | "accurate", "fast" |
| `ADV` | Trạng Từ | "quickly", "very" |
| `PROPN` | Danh Từ Riêng | "Amazon", "AWS" |
| `NUM` | Số | "100", "million" |
| `PUNCT` | Dấu Câu | ".", "," |
| `DET` | Mạo Từ | "the", "a", "an" |
| `PRON` | Đại Từ | "it", "they", "we" |

---

## 🎯 6. Targeted Sentiment (Cảm Xúc Hướng Vào Đối Tượng Cụ Thể)

Phân tích cảm xúc **gắn liền với từng entity** trong văn bản — chi tiết hơn so với sentiment tổng thể.

```python
def detect_targeted_sentiment(text: str, language_code: str = "en") -> None:
    """Phân tích cảm xúc theo từng đối tượng được đề cập."""
    response = comprehend.detect_targeted_sentiment(
        Text=text,
        LanguageCode=language_code
    )

    for entity in response["Entities"]:
        for mention in entity["Mentions"]:
            print(
                f"Entity: '{mention['Text']}' "
                f"→ Cảm xúc: {mention['MentionSentiment']['Sentiment']} "
                f"({mention['MentionSentiment']['SentimentScore']})"
            )

text = "The camera on the iPhone is amazing, but the battery life is disappointing."
detect_targeted_sentiment(text)
# → Entity: 'camera' → Cảm xúc: POSITIVE
# → Entity: 'iPhone' → Cảm xúc: MIXED
# → Entity: 'battery life' → Cảm xúc: NEGATIVE
```

> **Lợi ích:** Tốt hơn nhiều so với sentiment tổng thể trong các trường hợp review có nhiều đối tượng — giúp biết chính xác khía cạnh nào được khen, khía cạnh nào bị chê (Aspect-Based Sentiment Analysis — Phân Tích Cảm Xúc Theo Khía Cạnh).

---

## 📦 7. Batch Processing (Xử Lý Hàng Loạt)

### Batch Real-time — Tối Đa 25 Văn Bản / Request

```python
def batch_detect_sentiment(texts: list, language_code: str = "en") -> list:
    """Phân tích sentiment cho nhiều văn bản cùng lúc (tối đa 25)."""
    response = comprehend.batch_detect_sentiment(
        TextList=texts,
        LanguageCode=language_code
    )

    results = []
    for item in response["ResultList"]:
        results.append({
            "index": item["Index"],
            "sentiment": item["Sentiment"],
            "scores": item["SentimentScore"]
        })

    # Kiểm tra lỗi (văn bản quá dài, ký tự không hợp lệ...)
    for error in response["ErrorList"]:
        print(f"Lỗi tại index {error['Index']}: {error['ErrorMessage']}")

    return results
```

### Async Batch Job — Hàng Nghìn Tài Liệu Từ S3

```python
def start_sentiment_analysis_job(
    input_s3_uri: str,
    output_s3_uri: str,
    data_access_role_arn: str,
    language_code: str = "en"
) -> str:
    """Khởi động async job phân tích sentiment cho toàn bộ tài liệu trên S3."""
    response = comprehend.start_sentiment_detection_job(
        InputDataConfig={
            "S3Uri": input_s3_uri,               # s3://bucket/prefix/
            "InputFormat": "ONE_DOC_PER_FILE"    # hoặc ONE_DOC_PER_LINE
        },
        OutputDataConfig={"S3Uri": output_s3_uri},
        DataAccessRoleArn=data_access_role_arn,  # IAM Role để truy cập S3
        LanguageCode=language_code,
        JobName="sentiment-job-2026"
    )
    return response["JobId"]
```

**Input format:**
- `ONE_DOC_PER_FILE`: Mỗi file là một tài liệu (phổ biến nhất)
- `ONE_DOC_PER_LINE`: Mỗi dòng trong file là một tài liệu (phù hợp file CSV)

---

## 📊 Giới Hạn Và Quota (Hạn Mức)

| Giới Hạn | Giá Trị |
|---|---|
| Kích thước văn bản tối đa (real-time) | 5.000 byte (~5.000 ký tự ASCII) |
| Số văn bản tối đa (batch real-time) | 25 văn bản/request |
| Kích thước tài liệu (async job) | Tối đa 1 MB/file |
| Số tài liệu async | Không giới hạn cứng (xử lý song song) |
| Ngôn ngữ hỗ trợ | 100+ (language detection); ~12 (sentiment/entities) |

> **Tip về giới hạn 5.000 byte:** Nếu văn bản dài hơn, chia thành các đoạn (chunk) tối đa 5.000 ký tự, phân tích từng đoạn rồi aggregate kết quả (ví dụ: lấy trung bình weighted của SentimentScore).

---

## 🏗️ Kiến Trúc Ứng Dụng Thực Tế

### Hệ Thống Phân Tích Đánh Giá Sản Phẩm

```
Đánh giá sản phẩm (Review) từ website / app
      │
      ▼
SQS Queue  ──►  Lambda Function
                      │
                      ├──► detect_sentiment → tính điểm NPS (Net Promoter Score)
                      ├──► detect_key_phrases → tìm điểm mạnh/yếu
                      ├──► detect_targeted_sentiment → tìm khía cạnh cụ thể
                      │
                      ▼
                DynamoDB (lưu kết quả phân tích)
                      │
                      ▼
              QuickSight Dashboard
              (biểu đồ sentiment trend theo thời gian)
```

### Pipeline Phân Loại Ticket Hỗ Trợ

```
Email / Chat từ khách hàng
      │
      ▼
detect_dominant_language → nếu không phải tiếng Anh → Translate → tiếng Anh
      │
      ▼
detect_sentiment → NEGATIVE → ưu tiên cao, cảnh báo ngay
      │
detect_entities → tìm Product, OrderId được đề cập
      │
detect_key_phrases → xác định vấn đề chính
      │
      ▼
Tự động gán nhãn ticket + route đến đúng team
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Comprehend hỗ trợ ngôn ngữ nào cho sentiment analysis?**

> 12 ngôn ngữ chính (tiếng Anh, Tây Ban Nha, Pháp, Đức, Ý, Bồ Đào Nha, Ả Rập, Hindi, Nhật, Hàn, Trung giản thể, Trung phồn thể). Tiếng Việt chưa được hỗ trợ trực tiếp — cần dịch qua Amazon Translate trước.

**Q: Khi văn bản dài hơn 5.000 byte thì xử lý thế nào?**

> Chia văn bản thành các đoạn (chunk) ≤ 5.000 ký tự theo ranh giới câu (sentence boundary), phân tích từng đoạn rồi aggregate kết quả — ví dụ: average weighted sentiment score theo độ dài đoạn.

**Q: Targeted Sentiment khác detect_sentiment tổng thể thế nào?**

> `detect_sentiment` trả về cảm xúc tổng thể của toàn văn bản. `detect_targeted_sentiment` phân tích cảm xúc gắn với từng entity cụ thể — hữu ích cho review có nhiều đối tượng (VD: khen camera, chê pin của cùng một điện thoại).

**Q: Batch real-time vs async job — khi nào dùng cái nào?**

> Batch real-time (`batch_detect_*`): ≤ 25 văn bản, cần kết quả ngay, latency thấp. Async job (`start_*`): hàng nghìn–triệu tài liệu lớn trên S3, xử lý nền, không cần kết quả ngay — phù hợp ETL pipeline.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
