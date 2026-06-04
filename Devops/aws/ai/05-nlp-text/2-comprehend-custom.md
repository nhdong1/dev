# Amazon Comprehend Custom — Phân Loại Và Nhận Dạng Thực Thể Tùy Chỉnh

> Comprehend Custom cho phép bạn huấn luyện (train) model NLP riêng trên dữ liệu của mình mà không cần ML expertise (chuyên môn học máy). Có hai loại: **Custom Classification** (Phân Loại Văn Bản Tùy Chỉnh) — gán danh mục riêng cho văn bản; và **Custom Entity Recognition — Custom NER** (Nhận Dạng Thực Thể Tùy Chỉnh) — phát hiện các loại thực thể đặc thù của domain bạn.

---

## 🎯 Khi Nào Cần Comprehend Custom?

```
Built-in Comprehend đủ dùng khi:
├─ Bạn cần Sentiment, Key phrases, standard Entities (Person/Location/Org/Date...)
└─ Không cần phân loại theo danh mục riêng

Cần Comprehend Custom khi:
├─ Cần phân loại văn bản theo danh mục đặc thù của bạn
│   VD: Ticket hỗ trợ → ["Bug", "Feature Request", "Billing", "General"]
│   VD: Email → ["Urgent", "Normal", "Spam"]
│
└─ Cần nhận dạng entity đặc thù của domain
    VD: Văn bản thương mại → tìm [Tên Sản Phẩm], [Mã SKU], [Tên Nhà Cung Cấp]
    VD: Văn bản pháp lý → tìm [Điều Khoản Hợp Đồng], [Bên Ký Kết]
```

---

## 📋 1. Custom Classification (Phân Loại Văn Bản Tùy Chỉnh)

### Khái Niệm Cơ Bản

| Khái Niệm | Ý Nghĩa |
|---|---|
| **Classifier** (Bộ Phân Loại) | Model đã được huấn luyện để gán label cho văn bản |
| **Label** (Nhãn) | Danh mục bạn muốn phân loại (VD: "Positive", "Bug", "Urgent") |
| **Training data** (Dữ Liệu Huấn Luyện) | File CSV với cột `label` và cột `text` |
| **Multi-class** (Đa Lớp) | Mỗi văn bản thuộc đúng 1 nhãn |
| **Multi-label** (Đa Nhãn) | Mỗi văn bản có thể thuộc nhiều nhãn cùng lúc |
| **Endpoint** (Điểm Cuối) | Tài nguyên deploy model để gọi real-time |

### Định Dạng Dữ Liệu Huấn Luyện

**Multi-class (CSV — phổ biến nhất):**

```csv
label,text
BILLING,"I was charged twice for my subscription this month"
BUG,"The app crashes every time I try to upload an image"
FEATURE_REQUEST,"Please add dark mode to the mobile app"
GENERAL,"How do I reset my password?"
BILLING,"My invoice shows incorrect amount for March"
BUG,"Login button doesn't work on Safari browser"
```

**Yêu cầu tối thiểu:**
- Ít nhất **10 ví dụ mỗi label** (khuyến nghị: 50–1.000 ví dụ/label để độ chính xác tốt)
- Dữ liệu càng đa dạng càng tốt — tránh các mẫu quá giống nhau
- File upload lên S3 trước khi tạo training job

**Multi-label (CSV):**

```csv
labels,text
BILLING|URGENT,"I've been charged 3 times and need a refund immediately"
BUG|FEATURE_REQUEST,"The export function is broken; also please add CSV support"
```

### Quy Trình Huấn Luyện Custom Classifier

```python
import boto3
import time

comprehend = boto3.client("comprehend", region_name="us-east-1")

def train_custom_classifier(
    classifier_name: str,
    s3_training_uri: str,   # s3://bucket/training-data.csv
    data_access_role_arn: str,
    language_code: str = "en",
    mode: str = "MULTI_CLASS"   # hoặc "MULTI_LABEL"
) -> str:
    """Tạo và huấn luyện Custom Classifier."""
    response = comprehend.create_document_classifier(
        DocumentClassifierName=classifier_name,
        DataAccessRoleArn=data_access_role_arn,
        InputDataConfig={
            "DataFormat": "COMPREHEND_CSV",
            "S3Uri": s3_training_uri,
        },
        OutputDataConfig={
            "S3Uri": "s3://my-bucket/output/classifier/"
        },
        LanguageCode=language_code,
        Mode=mode
    )
    arn = response["DocumentClassifierArn"]
    print(f"Classifier ARN: {arn}")
    return arn

def wait_for_classifier(classifier_arn: str) -> str:
    """Chờ classifier hoàn thành training (polling)."""
    while True:
        response = comprehend.describe_document_classifier(
            DocumentClassifierArn=classifier_arn
        )
        status = response["DocumentClassifierProperties"]["Status"]
        print(f"Trạng thái: {status}")

        if status == "TRAINED":
            return "TRAINED"
        elif status in ["FAILED", "STOP_REQUESTED", "STOPPED"]:
            raise Exception(f"Training thất bại: {status}")
        time.sleep(60)  # Chờ 60 giây rồi kiểm tra lại
```

### Triển Khai (Deploy) Và Sử Dụng Classifier

```python
def create_classifier_endpoint(classifier_arn: str, endpoint_name: str) -> str:
    """Tạo endpoint real-time cho classifier đã trained."""
    response = comprehend.create_endpoint(
        EndpointName=endpoint_name,
        ModelArn=classifier_arn,
        DesiredInferenceUnits=1   # 1 IU = ~100 ký tự/giây
    )
    return response["EndpointArn"]

def classify_text(endpoint_arn: str, text: str) -> dict:
    """Phân loại văn bản bằng custom classifier endpoint."""
    response = comprehend.classify_document(
        Text=text,
        EndpointArn=endpoint_arn
    )
    # Multi-class: danh sách Classes với Score
    for cls in response.get("Classes", []):
        print(f"Label: {cls['Name']}, Score: {cls['Score']:.3f}")
    return response

# Ví dụ sử dụng
endpoint_arn = "arn:aws:comprehend:us-east-1:123456789:document-classifier-endpoint/my-endpoint"
result = classify_text(endpoint_arn, "The app crashes when I open the settings menu")
# → Label: BUG, Score: 0.952
# → Label: GENERAL, Score: 0.031
# → Label: FEATURE_REQUEST, Score: 0.017
```

### Async Batch Classification (Không Cần Endpoint)

```python
def start_classification_job(
    input_s3_uri: str,
    output_s3_uri: str,
    classifier_arn: str,
    data_access_role_arn: str
) -> str:
    """Phân loại hàng nghìn tài liệu theo lô từ S3 (không cần endpoint)."""
    response = comprehend.start_document_classification_job(
        JobName="classify-tickets-2026",
        DocumentClassifierArn=classifier_arn,
        InputDataConfig={
            "S3Uri": input_s3_uri,
            "InputFormat": "ONE_DOC_PER_FILE"
        },
        OutputDataConfig={"S3Uri": output_s3_uri},
        DataAccessRoleArn=data_access_role_arn
    )
    return response["JobId"]
```

> **Quan trọng về chi phí:** Endpoint tính phí **theo giờ** (Inference Unit/giờ) kể cả khi không có traffic. Nếu chỉ cần phân tích định kỳ, dùng **async job** thay vì giữ endpoint sống liên tục — tiết kiệm đáng kể.

---

## 🏷️ 2. Custom Entity Recognition — Custom NER (Nhận Dạng Thực Thể Tùy Chỉnh)

### Khái Niệm Cơ Bản

Custom NER cho phép bạn dạy Comprehend nhận dạng các loại entity **đặc thù của domain** không có trong bộ entity mặc định:

| Domain | Entity Cần Nhận Dạng |
|---|---|
| Thương mại điện tử | `PRODUCT_NAME`, `SKU_CODE`, `SUPPLIER` |
| Pháp lý | `CONTRACT_CLAUSE`, `PARTY_NAME`, `JURISDICTION` |
| Tài chính | `TICKER_SYMBOL`, `FINANCIAL_INSTRUMENT`, `AMOUNT_CURRENCY` |
| Sản xuất | `PART_NUMBER`, `DEFECT_TYPE`, `MACHINE_ID` |

### Định Dạng Dữ Liệu Huấn Luyện NER

**Cách 1: Annotations (Chú Thích) — Khuyến Nghị**

Tạo 2 file:
1. File văn bản (corpus): mỗi dòng một tài liệu
2. File chú thích CSV với vị trí entity

```csv
File,Line,Begin,End,Type
documents.txt,0,16,24,PRODUCT_NAME
documents.txt,0,56,63,SKU_CODE
documents.txt,1,5,18,PRODUCT_NAME
```

Trong đó `Begin`/`End` là vị trí ký tự (character offset) trong dòng tương ứng.

**Cách 2: Entity List (Danh Sách Thực Thể) — Đơn Giản Hơn**

```csv
Text,Type
"iPhone 15 Pro",PRODUCT_NAME
"Samsung Galaxy S24",PRODUCT_NAME
"SKU-2024-001",SKU_CODE
"P-ABC-789-X",SKU_CODE
"Công ty TNHH Minh Đức",SUPPLIER
```

Comprehend sẽ tự tìm các chuỗi trong danh sách này trong corpus bạn cung cấp.

### Huấn Luyện Custom NER

```python
def train_custom_ner(
    recognizer_name: str,
    entity_types: list,          # [{"Type": "PRODUCT_NAME"}, {"Type": "SKU_CODE"}]
    annotations_s3_uri: str,     # File annotations CSV
    documents_s3_uri: str,       # File corpus text
    data_access_role_arn: str,
    language_code: str = "en"
) -> str:
    """Huấn luyện Custom Entity Recognizer."""
    response = comprehend.create_entity_recognizer(
        RecognizerName=recognizer_name,
        DataAccessRoleArn=data_access_role_arn,
        EntityTypes=[{"Type": t} for t in entity_types],
        InputDataConfig={
            "DataFormat": "COMPREHEND_CSV",
            "EntityTypes": [{"Type": t} for t in entity_types],
            "Annotations": {"S3Uri": annotations_s3_uri},
            "Documents": {"S3Uri": documents_s3_uri}
        },
        LanguageCode=language_code
    )
    return response["EntityRecognizerArn"]
```

### Triển Khai Và Sử Dụng Custom NER

```python
def create_ner_endpoint(recognizer_arn: str, endpoint_name: str) -> str:
    """Tạo endpoint real-time cho custom NER."""
    response = comprehend.create_endpoint(
        EndpointName=endpoint_name,
        ModelArn=recognizer_arn,
        DesiredInferenceUnits=1
    )
    return response["EndpointArn"]

def detect_custom_entities(endpoint_arn: str, text: str) -> list:
    """Phát hiện custom entities trong văn bản."""
    response = comprehend.detect_entities(
        Text=text,
        EndpointArn=endpoint_arn   # Dùng endpoint ARN thay vì LanguageCode
    )

    custom_entities = []
    for entity in response["Entities"]:
        print(
            f"[{entity['Type']}] '{entity['Text']}' "
            f"(tin cậy: {entity['Score']:.2f})"
        )
        custom_entities.append(entity)
    return custom_entities

result = detect_custom_entities(
    endpoint_arn,
    "Đơn hàng SKU-2024-001 gồm 5 chiếc iPhone 15 Pro từ nhà cung cấp Minh Đức."
)
# → [SKU_CODE] 'SKU-2024-001' (tin cậy: 0.97)
# → [PRODUCT_NAME] 'iPhone 15 Pro' (tin cậy: 0.95)
# → [SUPPLIER] 'Minh Đức' (tin cậy: 0.91)
```

---

## 📊 Đánh Giá Model (Model Evaluation)

### Các Chỉ Số Quan Trọng

| Chỉ Số | Định Nghĩa | Mục Tiêu |
|---|---|---|
| **Accuracy** (Độ Chính Xác) | Tỷ lệ dự đoán đúng / tổng số mẫu | > 85% |
| **Precision** (Độ Chính Xác Dương) | Trong số dự đoán là class X, bao nhiêu thực sự là X? | > 80% |
| **Recall** (Độ Phủ) | Trong số thực sự là class X, bao nhiêu được phát hiện? | > 80% |
| **F1 Score** | Trung bình điều hòa (harmonic mean) của Precision và Recall | > 0.80 |

```
Precision = TP / (TP + FP)
Recall    = TP / (TP + FN)
F1        = 2 × (Precision × Recall) / (Precision + Recall)

Trong đó:
  TP — True Positive: dự đoán đúng là X, thực sự là X
  FP — False Positive: dự đoán là X nhưng thực sự không phải X
  FN — False Negative: thực sự là X nhưng không được phát hiện
```

### Xem Metrics Sau Training

```python
def get_classifier_metrics(classifier_arn: str) -> dict:
    """Lấy metrics đánh giá của classifier sau khi training xong."""
    response = comprehend.describe_document_classifier(
        DocumentClassifierArn=classifier_arn
    )
    metrics = response["DocumentClassifierProperties"]["ClassifierMetadata"]["EvaluationMetrics"]
    print(f"Accuracy: {metrics['Accuracy']:.3f}")
    print(f"Precision: {metrics['Precision']:.3f}")
    print(f"Recall: {metrics['Recall']:.3f}")
    print(f"F1 Score: {metrics['F1Score']:.3f}")
    return metrics
```

### Confusion Matrix (Ma Trận Nhầm Lẫn)

Comprehend xuất **Confusion Matrix** (ma trận nhầm lẫn) vào S3 sau training — cho thấy model nhầm class nào với class nào:

```
              Predicted
              BUG   BILLING   FEATURE   GENERAL
Actual BUG   [95    3          1         1    ]
       BILLING[2    91         2         5    ]
       FEATURE[1    0          88        11   ]
       GENERAL[2    4          6         88   ]
```

Nếu thấy FEATURE và GENERAL hay bị nhầm lẫn → cần thêm training data phân biệt rõ hơn hai class này.

---

## 🔧 Mẹo Cải Thiện Chất Lượng Model

### Dữ Liệu Huấn Luyện Tốt

- **Cân bằng class** (Class balance): Số ví dụ mỗi class nên tương đương — chênh lệch quá lớn làm model bias về class đa số
- **Đa dạng biểu đạt** (Diversity): Cùng ý nghĩa nhưng diễn đạt khác nhau — model sẽ generalize (tổng quát hóa) tốt hơn
- **Làm sạch dữ liệu** (Data cleaning): Loại bỏ mẫu sai nhãn, văn bản trùng lặp, ký tự đặc biệt không liên quan

```
Ví dụ về Đa Dạng Biểu Đạt Cho Class BUG:
✅ "The login page shows a 500 error"
✅ "App crashes when I open settings"
✅ "Cannot upload files - getting an error message"
✅ "Feature X is broken since the last update"
❌ "Bug bug bug bug bug bug bug bug"  ← quá lặp lại, không tự nhiên
```

### Bao Nhiêu Dữ Liệu Là Đủ?

| Số Ví Dụ/Class | Kết Quả Kỳ Vọng |
|---|---|
| 10–50 | Kết quả cơ bản, chưa ổn định |
| 50–200 | Sử dụng được cho production nhẹ |
| 200–1.000 | F1 > 0.85 cho hầu hết use case |
| 1.000+ | Tốt nhất; phù hợp production quy mô lớn |

---

## 💰 Lưu Ý Chi Phí Custom

| Hoạt Động | Chi Phí | Mẹo |
|---|---|---|
| Training job | Per giờ training (thường 30–60 phút) | Chuẩn bị dữ liệu tốt để train ít lần |
| Endpoint real-time | Per Inference Unit (IU) / giờ — **liên tục** | **Xóa endpoint khi không dùng** |
| Async job | Per ký tự xử lý | Không cần endpoint — tiết kiệm chi phí idle |

> **Quy tắc vàng:** Chỉ dùng endpoint real-time khi cần **latency thấp** (< vài giây). Với xử lý offline hàng giờ/ngày, dùng async job — không tốn phí idle.

---

## 🏗️ Kiến Trúc Hoàn Chỉnh Custom NLP Pipeline

```
Dữ Liệu Huấn Luyện (CSV / Annotations)
      │
      ▼
S3 Bucket (training data)
      │
      ▼
Comprehend Create Classifier / Recognizer
      │
      ▼
Training Job (~30-60 phút)
      │
      ├── Evaluate Metrics (F1, Precision, Recall, Confusion Matrix)
      │        │
      │        ├─ Không đạt → Thu thập thêm dữ liệu → Re-train
      │        └─ Đạt → Tiếp tục
      │
      ▼
Deploy Endpoint (nếu cần real-time)
      │
      ▼
Lambda / API Gateway gọi classify_document / detect_entities
      │
      ▼
DynamoDB (lưu kết quả phân tích)
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Custom Classification khác Custom NER thế nào?**

> Custom Classification gán **nhãn danh mục** cho toàn bộ văn bản (VD: "Đây là ticket loại BUG"). Custom NER tìm và đánh dấu **các đoạn cụ thể** trong văn bản là entity loại nào (VD: "SKU-001" là SKU_CODE). Hai loại giải quyết bài toán hoàn toàn khác nhau.

**Q: Cần bao nhiêu dữ liệu để train Custom Classifier?**

> Tối thiểu 10 ví dụ/class; khuyến nghị 200–1.000 ví dụ/class để F1 > 0.85. Dữ liệu đa dạng quan trọng hơn số lượng thuần túy.

**Q: Tại sao không dùng Bedrock/LLM thay vì Custom Classifier?**

> Custom Classifier: **nhanh, rẻ, nhất quán** cho classification bậc cao, phù hợp xử lý hàng triệu văn bản. Bedrock/LLM: linh hoạt hơn nhưng **chậm và đắt hơn** — dùng khi classification cần hiểu ngữ cảnh phức tạp hoặc không có đủ data để train.

**Q: Endpoint tính phí thế nào khi không có request?**

> Endpoint Comprehend Custom tính phí **theo giờ kể cả khi idle** (không có request). Luôn xóa endpoint khi không cần thiết; dùng async job cho batch processing để tránh chi phí không cần thiết.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
