# Amazon Translate — Dịch Thuật Máy Thần Kinh Đa Ngôn Ngữ

> Amazon Translate là dịch vụ **NMT** — Neural Machine Translation — Dịch Thuật Máy Thần Kinh được quản lý hoàn toàn (fully-managed) của AWS, cung cấp khả năng dịch văn bản chất lượng cao giữa hơn **75 ngôn ngữ** theo thời gian thực hoặc theo lô. Dịch vụ hỗ trợ tích hợp **Custom Terminology** (Thuật Ngữ Tùy Chỉnh) để bảo đảm thuật ngữ chuyên ngành được dịch chính xác theo yêu cầu của tổ chức.

---

## 🎯 Các Tính Năng Chính

| Tính Năng | API / Tính Năng | Mô Tả |
|---|---|---|
| **Real-time Translation** (Dịch Thời Gian Thực) | `translate_text` | Dịch tức thì văn bản ngắn (< 10.000 byte) |
| **Batch Translation** (Dịch Theo Lô) | `start_text_translation_job` | Dịch hàng nghìn tài liệu từ S3 |
| **Auto Language Detection** (Tự Phát Hiện Ngôn Ngữ) | `SourceLanguageCode="auto"` | Tự xác định ngôn ngữ nguồn |
| **Custom Terminology** (Thuật Ngữ Tùy Chỉnh) | `import_terminology` | Từ điển thuật ngữ bắt buộc của bạn |
| **Active Custom Translation — ACT** | Training trên parallel corpus | Model dịch tùy chỉnh theo domain |
| **Profanity Masking** (Che Từ Tục) | `Settings.Profanity="MASK"` | Tự động che/thay thế từ ngữ không phù hợp |
| **Formality** (Mức Độ Trang Trọng) | `Settings.Formality` | Dịch văn phong trang trọng hoặc thông thường |

---

## 🌍 Ngôn Ngữ Được Hỗ Trợ

Amazon Translate hỗ trợ **75+ ngôn ngữ** với dịch theo cặp ngôn ngữ (language pair). Các ngôn ngữ phổ biến:

| Mã Ngôn Ngữ | Ngôn Ngữ |
|---|---|
| `vi` | Tiếng Việt |
| `en` | Tiếng Anh |
| `zh` | Tiếng Trung Giản Thể |
| `zh-TW` | Tiếng Trung Phồn Thể |
| `ja` | Tiếng Nhật |
| `ko` | Tiếng Hàn |
| `fr` | Tiếng Pháp |
| `de` | Tiếng Đức |
| `es` | Tiếng Tây Ban Nha |
| `ar` | Tiếng Ả Rập |
| `pt` | Tiếng Bồ Đào Nha |
| `hi` | Tiếng Hindi |
| `th` | Tiếng Thái |
| `id` | Tiếng Indonesia |

> Không phải mọi cặp ngôn ngữ đều hỗ trợ dịch trực tiếp — một số cặp ít phổ biến dịch qua tiếng Anh làm trung gian (pivot language).

---

## 🚀 1. Real-time Translation (Dịch Thời Gian Thực)

```python
import boto3

translate = boto3.client("translate", region_name="us-east-1")

def translate_text(
    text: str,
    source_lang: str,
    target_lang: str,
    terminology_names: list = None
) -> dict:
    """Dịch văn bản theo thời gian thực."""
    kwargs = {
        "Text": text,
        "SourceLanguageCode": source_lang,   # "auto" để tự phát hiện
        "TargetLanguageCode": target_lang
    }

    if terminology_names:
        kwargs["TerminologyNames"] = terminology_names

    response = translate.translate_text(**kwargs)
    return {
        "translated_text": response["TranslatedText"],
        "source_language": response["SourceLanguageCode"],  # Ngôn ngữ thực sự phát hiện
        "target_language": response["TargetLanguageCode"]
    }

# Dịch từ tiếng Anh sang tiếng Việt
result = translate_text(
    "Amazon Web Services provides on-demand cloud computing platforms.",
    source_lang="en",
    target_lang="vi"
)
print(result["translated_text"])
# → "Amazon Web Services cung cấp các nền tảng điện toán đám mây theo yêu cầu."

# Auto-detect ngôn ngữ nguồn
result = translate_text(
    "Bonjour tout le monde!",
    source_lang="auto",
    target_lang="vi"
)
print(f"Ngôn ngữ phát hiện: {result['source_language']}")   # → fr
print(result["translated_text"])                              # → "Xin chào tất cả mọi người!"
```

### Giới Hạn Real-time

| Giới Hạn | Giá Trị |
|---|---|
| Kích thước văn bản tối đa | 10.000 byte UTF-8 |
| Số yêu cầu đồng thời tối đa | 250 requests/giây (có thể tăng qua quota request) |
| Thời gian phản hồi | Thường < 1 giây cho văn bản ngắn |

> **Khi văn bản dài hơn 10.000 byte:** Chia thành các đoạn (chunks) tại ranh giới câu (sentence boundary) và dịch từng đoạn. Ghép kết quả lại — tránh chia giữa câu làm mất ngữ cảnh.

---

## 📦 2. Batch Translation (Dịch Theo Lô)

Dịch hàng nghìn tài liệu lưu trên S3 — kết quả lưu về S3, không cần endpoint.

```python
def start_batch_translation(
    input_s3_uri: str,       # s3://bucket/input/ (thư mục chứa tài liệu)
    output_s3_uri: str,      # s3://bucket/output/
    source_lang: str,
    target_langs: list,      # Có thể dịch sang nhiều ngôn ngữ cùng lúc
    data_access_role_arn: str,
    content_type: str = "text/plain",
    terminology_names: list = None
) -> str:
    """Khởi động batch translation job từ S3."""
    kwargs = {
        "JobName": "batch-translate-2026",
        "InputDataConfig": {
            "S3Uri": input_s3_uri,
            "ContentType": content_type   # "text/plain", "text/html", "application/vnd.openxmlformats-officedocument..."
        },
        "OutputDataConfig": {"S3Uri": output_s3_uri},
        "DataAccessRoleArn": data_access_role_arn,
        "SourceLanguageCode": source_lang,
        "TargetLanguageCodes": target_langs    # ["vi", "ja", "fr"] — dịch sang 3 ngôn ngữ cùng lúc
    }

    if terminology_names:
        kwargs["TerminologyNames"] = terminology_names

    response = translate.start_text_translation_job(**kwargs)
    return response["JobId"]

def get_batch_job_status(job_id: str) -> str:
    """Kiểm tra trạng thái batch job."""
    response = translate.describe_text_translation_job(JobId=job_id)
    job = response["TextTranslationJobProperties"]
    print(f"Trạng thái: {job['JobStatus']}")
    print(f"Tiến độ: {job.get('Message', 'N/A')}")
    return job["JobStatus"]

# Khởi chạy
job_id = start_batch_translation(
    input_s3_uri="s3://my-bucket/docs/",
    output_s3_uri="s3://my-bucket/translated/",
    source_lang="en",
    target_langs=["vi", "ja", "zh"],
    data_access_role_arn="arn:aws:iam::123456789:role/TranslateRole"
)

# Kiểm tra sau vài phút
status = get_batch_job_status(job_id)
```

### Định Dạng File Được Hỗ Trợ

| Định Dạng | Content Type | Ghi Chú |
|---|---|---|
| Plain text (.txt) | `text/plain` | Phổ biến nhất |
| HTML (.html) | `text/html` | Giữ nguyên thẻ HTML, chỉ dịch nội dung |
| Word (.docx) | `application/vnd.openxmlformats...` | Giữ nguyên định dạng |
| Excel (.xlsx) | `application/vnd.openxmlformats...` | Giữ nguyên bảng |
| PowerPoint (.pptx) | `application/vnd.openxmlformats...` | Giữ nguyên slide |
| PDF | ❌ Không hỗ trợ | Cần convert sang text/html trước |

---

## 📖 3. Custom Terminology (Thuật Ngữ Tùy Chỉnh)

Custom Terminology cho phép bạn định nghĩa **bản dịch bắt buộc** cho các thuật ngữ quan trọng — Amazon Translate sẽ ưu tiên dùng từ điển này thay vì dịch tự động.

### Tại Sao Cần Custom Terminology?

```
Vấn đề khi dịch không có Custom Terminology:
├─ Tên thương hiệu bị dịch: "Amazon Translate" → "Amazon Dịch"  ❌
├─ Thuật ngữ kỹ thuật dịch sai: "load balancer" → "bộ cân bằng tải" ✅
│   nhưng nội bộ bạn gọi là "load balancer" không dịch  ← muốn giữ nguyên
├─ Thuật ngữ pháp lý phải nhất quán: "force majeure" phải luôn → "bất khả kháng"
└─ Tên sản phẩm nội bộ không được dịch: "SmartHub Pro" giữ nguyên

Với Custom Terminology:
└─ Bạn kiểm soát hoàn toàn cách dịch các thuật ngữ quan trọng
```

### Tạo Custom Terminology

**Định dạng CSV:**

```csv
en,vi,ja,fr
"Amazon Translate","Amazon Translate","Amazon Translate","Amazon Translate"
"load balancer","load balancer","ロードバランサー","équilibreur de charge"
"force majeure","bất khả kháng","不可抗力","force majeure"
"SmartHub Pro","SmartHub Pro","SmartHub Pro","SmartHub Pro"
"machine learning","học máy","機械学習","apprentissage automatique"
"API Gateway","API Gateway","API Gateway","API Gateway"
```

**Định dạng TMX** (Translation Memory eXchange — Định Dạng Bộ Nhớ Dịch Thuật):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<tmx version="1.4">
  <body>
    <tu>
      <tuv xml:lang="en"><seg>load balancer</seg></tuv>
      <tuv xml:lang="vi"><seg>load balancer</seg></tuv>
    </tu>
    <tu>
      <tuv xml:lang="en"><seg>Amazon Web Services</seg></tuv>
      <tuv xml:lang="vi"><seg>Amazon Web Services</seg></tuv>
    </tu>
  </body>
</tmx>
```

### Upload Và Dùng Custom Terminology

```python
def import_terminology(name: str, csv_file_path: str) -> str:
    """Upload Custom Terminology từ file CSV lên AWS."""
    with open(csv_file_path, "rb") as f:
        file_data = f.read()

    response = translate.import_terminology(
        Name=name,
        MergeStrategy="OVERWRITE",    # OVERWRITE: ghi đè nếu tên đã tồn tại
        TerminologyData={
            "File": file_data,
            "Format": "CSV"            # hoặc "TMX"
        }
    )
    arn = response["TerminologyProperties"]["Arn"]
    print(f"Đã upload terminology: {arn}")
    return name

def translate_with_terminology(
    text: str,
    source_lang: str,
    target_lang: str,
    terminology_name: str
) -> str:
    """Dịch văn bản với Custom Terminology."""
    response = translate.translate_text(
        Text=text,
        SourceLanguageCode=source_lang,
        TargetLanguageCode=target_lang,
        TerminologyNames=[terminology_name]   # Dùng terminology đã upload
    )
    return response["TranslatedText"]

# Upload thuật ngữ
import_terminology("tech-glossary", "terminology.csv")

# Dịch với thuật ngữ tùy chỉnh
result = translate_with_terminology(
    "The load balancer distributes traffic across multiple instances.",
    "en", "vi",
    "tech-glossary"
)
print(result)
# → "The load balancer phân phối lưu lượng qua nhiều instance."
# (load balancer được giữ nguyên theo định nghĩa trong terminology)

# Quản lý terminology
def list_terminologies():
    response = translate.list_terminologies()
    for t in response["TerminologyPropertiesList"]:
        print(f"{t['Name']}: {t['SourceLanguageCode']} → {t['TargetLanguageCodes']}")

def delete_terminology(name: str):
    translate.delete_terminology(Name=name)
```

---

## 🎓 4. Active Custom Translation — ACT (Dịch Tùy Chỉnh Chủ Động)

ACT cho phép huấn luyện model dịch riêng trên **parallel corpus** (kho ngữ liệu song song) của bạn — phù hợp cho domain đặc thù có nhiều thuật ngữ chuyên biệt và phong cách dịch riêng.

### Khi Nào Cần ACT?

```
Custom Terminology đủ khi:
└─ Bạn chỉ cần kiểm soát một số thuật ngữ quan trọng

ACT cần thiết khi:
├─ Phong cách dịch đặc thù (tone, formality) khác biệt lớn
├─ Domain rất chuyên biệt (pháp lý, y tế, kỹ thuật hàng không)
└─ Bạn có ≥ 5.000 cặp câu song ngữ (parallel sentences)
```

### Định Dạng Dữ Liệu Huấn Luyện ACT

**File nguồn (source):** mỗi dòng một câu tiếng Anh
**File đích (target):** mỗi dòng là bản dịch tiếng Việt tương ứng

```
# source.txt (tiếng Anh)
The elastic load balancer distributes incoming traffic.
Configure auto scaling to handle peak loads.
Deploy the application to production environment.

# target.txt (tiếng Việt — dịch chuyên ngành)
Bộ cân bằng tải co giãn phân phối lưu lượng đến.
Cấu hình auto scaling để xử lý tải đỉnh.
Triển khai ứng dụng lên môi trường production.
```

```python
def create_parallel_data(name: str, source_s3_uri: str, target_s3_uri: str) -> str:
    """Tạo Parallel Data cho ACT training."""
    response = translate.create_parallel_data(
        Name=name,
        ParallelDataConfig={
            "S3Uri": source_s3_uri,
            "Format": "TSV"    # TSV (Tab-Separated Values) hoặc TMX
        }
    )
    return response["Name"]

def create_custom_translator(
    name: str,
    source_lang: str,
    target_lang: str,
    parallel_data_name: str,
    base_model_arn: str = None    # None = dùng model mặc định của AWS
) -> str:
    """Tạo và train Custom Translator."""
    response = translate.create_text_translation_job(
        JobName=name,
        InputDataConfig={"S3Uri": "s3://...", "ContentType": "text/plain"},
        OutputDataConfig={"S3Uri": "s3://..."},
        DataAccessRoleArn="arn:aws:iam::...:role/TranslateRole",
        SourceLanguageCode=source_lang,
        TargetLanguageCodes=[target_lang],
        ParallelDataNames=[parallel_data_name]
    )
    return response["JobId"]
```

---

## ⚙️ 5. Formality Và Profanity Settings

### Formality (Mức Độ Trang Trọng)

```python
def translate_with_formality(text: str, target_lang: str, formality: str) -> str:
    """
    Dịch với mức độ trang trọng:
    formality = "FORMAL" (trang trọng) hoặc "INFORMAL" (thân mật)
    """
    response = translate.translate_text(
        Text=text,
        SourceLanguageCode="en",
        TargetLanguageCode=target_lang,
        Settings={"Formality": formality}
    )
    return response["TranslatedText"]

# Ví dụ (dịch sang tiếng Đức)
formal = translate_with_formality("How are you?", "de", "FORMAL")
informal = translate_with_formality("How are you?", "de", "INFORMAL")
# formal   → "Wie geht es Ihnen?"   (dùng "Sie" — bạn trang trọng)
# informal → "Wie geht es dir?"     (dùng "du" — bạn thân mật)
```

> **Lưu ý:** Formality chỉ hỗ trợ một số ngôn ngữ như Đức, Pháp, Ý, Tây Ban Nha, Nhật, Bồ Đào Nha. Tiếng Việt hiện chưa hỗ trợ tính năng này.

### Profanity Masking (Che Từ Không Phù Hợp)

```python
def translate_with_profanity_filter(text: str, source_lang: str, target_lang: str) -> str:
    """Dịch và tự động che từ tục bằng dấu *** trong bản dịch."""
    response = translate.translate_text(
        Text=text,
        SourceLanguageCode=source_lang,
        TargetLanguageCode=target_lang,
        Settings={"Profanity": "MASK"}    # Thay từ không phù hợp bằng ***
    )
    return response["TranslatedText"]
```

---

## 📊 Kiến Trúc Ứng Dụng Thực Tế

### Hệ Thống Dịch Thuật Nội Dung Đa Ngôn Ngữ (Content Localization)

```
Nội dung mới (tiếng Anh) được tạo ra
      │
      ▼
S3 Bucket (content staging)
      │
      ├──(trigger)──► Lambda Function
      │                      │
      │                      ├── translate_text sang vi, ja, zh, fr, es, de
      │                      │   (dùng custom-terminology cho tên sản phẩm)
      │                      │
      │                      └── Lưu kết quả → S3 (phân loại theo ngôn ngữ)
      │
      ▼
CloudFront CDN
      │
      └── Phục vụ nội dung đúng ngôn ngữ theo Accept-Language header người dùng
```

### Pipeline Phân Tích Đa Ngôn Ngữ (Multi-language Analytics)

```
Feedback khách hàng từ 50 quốc gia (nhiều ngôn ngữ)
      │
      ▼
detect_dominant_language (Comprehend)
      │
      ▼
translate_text (Amazon Translate, auto-detect → en)
      │
      ▼
detect_sentiment (Comprehend, LanguageCode="en")
detect_entities
detect_key_phrases
      │
      ▼
Aggregate kết quả → QuickSight Dashboard
(hiển thị sentiment toàn cầu theo ngôn ngữ/vùng địa lý)
```

### Real-time Chat Translation (Dịch Chat Thời Gian Thực)

```
User A (tiếng Việt) gửi tin nhắn
      │
      ▼
WebSocket API Gateway → Lambda
      │
      ├── detect_dominant_language → "vi"
      ├── translate_text "vi" → "en" (cho User B nói tiếng Anh)
      └── translate_text "vi" → "ja" (cho User C nói tiếng Nhật)
              │
              ▼
      WebSocket broadcast tin nhắn đã dịch tới từng user
```

---

## 💰 Mô Hình Chi Phí

| Tính Năng | Mô Hình Giá | Ghi Chú |
|---|---|---|
| Real-time Translation | Per ký tự dịch | Tối thiểu 100 ký tự/request |
| Batch Translation | Per ký tự dịch | Giống real-time nhưng không cần endpoint |
| Custom Terminology | Miễn phí lưu trữ | Chỉ tính phí dịch thuật khi dùng |
| Active Custom Translation | Training hour + inference | Đắt hơn đáng kể |

**Free Tier:** 2 triệu ký tự/tháng trong 12 tháng đầu.

**Mẹo tiết kiệm:**
- Cache bản dịch (ElastiCache / DynamoDB) để tránh dịch lại cùng nội dung nhiều lần
- Batch nhiều câu vào một request khi có thể (giảm request overhead)
- Với nội dung tĩnh (UI labels, documentation), dịch offline và cache — không cần real-time

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Amazon Translate vs Bedrock/LLM cho dịch thuật — khi nào dùng cái nào?**

> Translate: **nhanh, rẻ, ổn định** cho dịch hàng loạt, pipeline automation, real-time chat. Bedrock/LLM: linh hoạt hơn cho dịch cần giữ sắc thái văn học, dịch kèm giải thích, dịch trong ngữ cảnh hội thoại phức tạp — nhưng đắt và chậm hơn nhiều.

**Q: Custom Terminology hoạt động thế nào?**

> Bạn định nghĩa từ điển CSV/TMX với các cặp thuật ngữ nguồn–đích. Khi dịch, Translate scan văn bản tìm các thuật ngữ này và **bắt buộc dùng bản dịch trong từ điển** thay vì dịch tự động — đảm bảo nhất quán tên thương hiệu, thuật ngữ kỹ thuật.

**Q: Dịch tài liệu HTML có mất định dạng không?**

> Không — Translate nhận diện thẻ HTML và **chỉ dịch nội dung text**, giữ nguyên toàn bộ thẻ, thuộc tính, cấu trúc HTML. Đây là một trong những điểm mạnh so với copy-paste vào Google Translate.

**Q: Batch translation job khác real-time ở điểm gì?**

> Real-time: tối đa 10.000 byte/request, kết quả trả về ngay trong response API — phù hợp cho ứng dụng cần phản hồi tức thì. Batch: không giới hạn kích thước (nhiều file trên S3), xử lý nền, kết quả lưu về S3 — phù hợp ETL pipeline, dịch toàn bộ document library, không cần kết quả ngay.

**Q: Active Custom Translation (ACT) cần bao nhiêu dữ liệu?**

> AWS khuyến nghị tối thiểu **5.000 cặp câu song ngữ** (parallel sentences) để ACT cải thiện chất lượng có ý nghĩa. Dưới mức này, Custom Terminology thường đủ dùng và rẻ hơn nhiều.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
