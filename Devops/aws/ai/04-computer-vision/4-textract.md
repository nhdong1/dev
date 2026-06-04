# Amazon Textract — Trích Xuất Tài Liệu & OCR

> Amazon Textract là dịch vụ AI được quản lý hoàn toàn (fully-managed), vượt xa OCR (Optical Character Recognition — Nhận Dạng Ký Tự Quang Học) truyền thống. Ngoài việc đọc chữ, Textract còn **hiểu cấu trúc tài liệu** — trích xuất key-value pairs (cặp khóa-giá trị) từ form, dữ liệu từ table (bảng), và các trường chuyên biệt từ hóa đơn, biên lai, giấy tờ tùy thân.

---

## 🎯 Textract Khác OCR Truyền Thống Thế Nào?

| Khả Năng | OCR Truyền Thống | Amazon Textract |
|---|---|---|
| Đọc text thô | ✅ | ✅ |
| Hiểu cấu trúc form (key-value) | ❌ | ✅ |
| Trích xuất bảng (rows/columns) | ❌ | ✅ |
| Hiểu trường chuyên biệt (hóa đơn, ID) | ❌ | ✅ (Analyze Expense / ID) |
| Giữ quan hệ giữa các phần tử | ❌ | ✅ (Block relationships) |

---

## 🛠️ Các API Chính Của Textract

| API | Mục Đích | Sync | Async |
|---|---|---|---|
| **DetectDocumentText** | OCR thuần — đọc toàn bộ text | `detect_document_text` | `start_document_text_detection` |
| **AnalyzeDocument** | Form + Table + Query + Signature | `analyze_document` | `start_document_analysis` |
| **AnalyzeExpense** | Hóa đơn, biên lai (invoice/receipt) | `analyze_expense` | `start_expense_analysis` |
| **AnalyzeID** | Giấy tờ tùy thân (passport, bằng lái) | `analyze_id` | — |

### Sync vs Async — Quy Tắc Chọn

| | Synchronous (Đồng Bộ) | Asynchronous (Bất Đồng Bộ) |
|---|---|---|
| **Input** | Ảnh, PDF 1 trang | PDF/TIFF nhiều trang |
| **Cách gọi** | `analyze_*` trả ngay | `start_*` + `get_*` + SNS |
| **Giới hạn** | Tài liệu nhỏ (1 trang) | Tài liệu lớn (nhiều trang) |

---

## 📄 1. DetectDocumentText (OCR Thuần)

Khi chỉ cần đọc text mà không quan tâm cấu trúc — rẻ nhất.

```python
import boto3

textract = boto3.client("textract", region_name="us-east-1")

def detect_text(bucket: str, key: str) -> str:
    """Trích xuất toàn bộ text từ tài liệu 1 trang."""
    response = textract.detect_document_text(
        Document={"S3Object": {"Bucket": bucket, "Name": key}}
    )

    lines = []
    for block in response["Blocks"]:
        if block["BlockType"] == "LINE":
            lines.append(block["Text"])
    return "\n".join(lines)
```

### Mô Hình Block — Cốt Lõi Của Textract

Textract trả về một danh sách **Blocks** liên kết với nhau theo quan hệ phân cấp:

```
PAGE (Trang)
 └── LINE (Dòng)
      └── WORD (Từ)

KEY_VALUE_SET (Cặp Khóa-Giá Trị — form)
 ├── KEY   (khóa, VD: "Tên:")
 └── VALUE (giá trị, VD: "Nguyễn Văn A")

TABLE (Bảng)
 └── CELL (Ô)
      └── WORD
```

Mỗi block có `Id`, `BlockType`, `Confidence`, `Geometry` (bounding box) và `Relationships` (liên kết tới block con).

---

## 📋 2. AnalyzeDocument — Forms, Tables, Queries

API mạnh nhất, kích hoạt qua `FeatureTypes`:

| FeatureType | Trích Xuất |
|---|---|
| `FORMS` | Key-value pairs (cặp khóa-giá trị) |
| `TABLES` | Bảng có hàng/cột |
| `QUERIES` | Hỏi tài liệu bằng câu hỏi tự nhiên |
| `SIGNATURES` | Phát hiện vị trí chữ ký |
| `LAYOUT` | Cấu trúc bố cục (tiêu đề, đoạn, header/footer) |

### Trích Xuất Form (Key-Value)

```python
def analyze_form(bucket: str, key: str) -> dict:
    """Trích xuất các cặp key-value từ form."""
    response = textract.analyze_document(
        Document={"S3Object": {"Bucket": bucket, "Name": key}},
        FeatureTypes=["FORMS"]
    )

    blocks = {b["Id"]: b for b in response["Blocks"]}
    key_values = {}

    for block in response["Blocks"]:
        if block["BlockType"] == "KEY_VALUE_SET" and "KEY" in block.get("EntityTypes", []):
            key_text = _get_text(block, blocks)
            # Tìm VALUE liên kết với KEY này
            value_text = ""
            for rel in block.get("Relationships", []):
                if rel["Type"] == "VALUE":
                    for vid in rel["Ids"]:
                        value_text = _get_text(blocks[vid], blocks)
            key_values[key_text] = value_text
    return key_values


def _get_text(block, blocks_map) -> str:
    """Ghép text từ các WORD block con."""
    text = ""
    for rel in block.get("Relationships", []):
        if rel["Type"] == "CHILD":
            for cid in rel["Ids"]:
                child = blocks_map[cid]
                if child["BlockType"] == "WORD":
                    text += child["Text"] + " "
    return text.strip()
```

### Queries — Hỏi Tài Liệu Bằng Ngôn Ngữ Tự Nhiên

Tính năng mạnh: thay vì parse thủ công, **hỏi trực tiếp** điều cần lấy:

```python
def query_document(bucket: str, key: str):
    """Hỏi tài liệu các câu hỏi cụ thể."""
    response = textract.analyze_document(
        Document={"S3Object": {"Bucket": bucket, "Name": key}},
        FeatureTypes=["QUERIES"],
        QueriesConfig={
            "Queries": [
                {"Text": "Số hóa đơn là gì?", "Alias": "invoice_number"},
                {"Text": "Tổng tiền là bao nhiêu?", "Alias": "total"},
                {"Text": "Ngày phát hành?", "Alias": "date"},
            ]
        }
    )

    for block in response["Blocks"]:
        if block["BlockType"] == "QUERY_RESULT":
            print(f"{block['Text']} (tin cậy: {block['Confidence']:.1f}%)")
```

---

## 🧾 3. AnalyzeExpense — Hóa Đơn & Biên Lai

Chuyên dụng cho invoice (hóa đơn) và receipt (biên lai) — tự hiểu các trường nghiệp vụ mà không cần định nghĩa.

```python
def analyze_expense(bucket: str, key: str):
    """Trích xuất dữ liệu từ hóa đơn/biên lai."""
    response = textract.analyze_expense(
        Document={"S3Object": {"Bucket": bucket, "Name": key}}
    )

    for doc in response["ExpenseDocuments"]:
        # Summary fields — các trường tổng hợp (tổng tiền, ngày, nhà cung cấp)
        for field in doc["SummaryFields"]:
            field_type = field.get("Type", {}).get("Text", "")
            value = field.get("ValueDetection", {}).get("Text", "")
            print(f"{field_type}: {value}")

        # Line items — từng dòng sản phẩm trong hóa đơn
        for group in doc.get("LineItemGroups", []):
            for item in group["LineItems"]:
                fields = {f["Type"]["Text"]: f["ValueDetection"]["Text"]
                          for f in item["LineItemExpenseFields"]}
                print(f"  Item: {fields}")
```

**Trường tự nhận diện:** VENDOR_NAME (nhà cung cấp), TOTAL (tổng), SUBTOTAL, TAX (thuế), INVOICE_RECEIPT_DATE (ngày), DUE_DATE, ITEM (mặt hàng), QUANTITY (số lượng), UNIT_PRICE (đơn giá).

---

## 🪪 4. AnalyzeID — Giấy Tờ Tùy Thân

Trích xuất thông tin từ passport, bằng lái, ID card — hiểu các trường chuẩn.

```python
def analyze_id(bucket: str, key: str):
    """Trích xuất thông tin từ giấy tờ tùy thân."""
    response = textract.analyze_id(
        DocumentPages=[{"S3Object": {"Bucket": bucket, "Name": key}}]
    )

    for doc in response["IdentityDocuments"]:
        for field in doc["IdentityDocumentFields"]:
            field_type = field["Type"]["Text"]      # VD: FIRST_NAME, DATE_OF_BIRTH
            value = field["ValueDetection"]["Text"]
            print(f"{field_type}: {value}")
```

**Trường chuẩn:** FIRST_NAME, LAST_NAME, DATE_OF_BIRTH, DOCUMENT_NUMBER, EXPIRATION_DATE, ADDRESS, ID_TYPE.

---

## ⚙️ Xử Lý Tài Liệu Nhiều Trang (Async)

PDF/TIFF nhiều trang **bắt buộc** dùng async:

```python
import time

def analyze_multipage_pdf(bucket: str, key: str):
    """Phân tích PDF nhiều trang qua API bất đồng bộ."""
    # Bước 1: Khởi động job
    start = textract.start_document_analysis(
        DocumentLocation={"S3Object": {"Bucket": bucket, "Name": key}},
        FeatureTypes=["FORMS", "TABLES"]
    )
    job_id = start["JobId"]

    # Bước 2: Polling đến khi xong
    while True:
        result = textract.get_document_analysis(JobId=job_id)
        status = result["JobStatus"]
        if status in ("SUCCEEDED", "FAILED"):
            break
        time.sleep(5)

    # Bước 3: Lấy toàn bộ kết quả (phân trang qua NextToken)
    blocks = result["Blocks"]
    next_token = result.get("NextToken")
    while next_token:
        page = textract.get_document_analysis(JobId=job_id, NextToken=next_token)
        blocks.extend(page["Blocks"])
        next_token = page.get("NextToken")
    return blocks
```

> Production nên gắn **NotificationChannel** (SNS) vào `start_*` để Lambda được kích hoạt khi job xong, thay vì polling.

---

## 🤝 Textract + A2I + Comprehend — IDP Pipeline

**IDP — Intelligent Document Processing** (Xử Lý Tài Liệu Thông Minh) là pattern phổ biến kết hợp nhiều dịch vụ:

```
Tài liệu (S3)
      │
      ▼
Textract (trích xuất text + form + table)
      │
      ├──► Confidence thấp? ──► Amazon A2I (Augmented AI — human review)
      │
      ▼
Comprehend (phân tích: entity, sentiment, phân loại)
      │
      ├──► Comprehend Medical (nếu là hồ sơ y tế → ICD-10, RxNorm)
      │
      ▼
DynamoDB / RDS (lưu dữ liệu có cấu trúc)
      │
      ▼
OpenSearch (cho tìm kiếm) / QuickSight (cho báo cáo)
```

---

## 🆚 Textract vs Rekognition DetectText

| Tiêu Chí | Textract | Rekognition DetectText |
|---|---|---|
| **Input lý tưởng** | Tài liệu (form, hóa đơn, hợp đồng) | Ảnh đời thực (biển báo, biển số) |
| **Cấu trúc form/table** | ✅ | ❌ |
| **Số trang** | Nhiều (async) | 1 ảnh |
| **Giới hạn từ** | Trang đầy đủ | ~100 từ/ảnh |
| **Use case** | Document processing, IDP | Scene text, biển số xe |

---

## 💰 Lưu Ý Chi Phí

| API | Chi Phí Tương Đối | Mẹo |
|---|---|---|
| DetectDocumentText | Rẻ nhất | Dùng khi chỉ cần OCR thuần |
| AnalyzeDocument (FORMS/TABLES) | Đắt hơn, tính theo feature | Chỉ bật feature cần thiết |
| AnalyzeExpense / AnalyzeID | Giá theo trang | — |

> Tính phí theo **trang xử lý** và **feature kích hoạt**. Bật cả FORMS + TABLES + QUERIES cùng lúc sẽ cộng dồn chi phí — chỉ chọn feature thực sự cần.

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Textract khác OCR thường thế nào?**

> OCR thường chỉ chuyển ảnh thành text thô. Textract hiểu **cấu trúc**: trích xuất key-value từ form, dữ liệu từ table, các trường nghiệp vụ từ hóa đơn/ID, và giữ quan hệ giữa các phần tử.

**Q: Khi nào dùng Textract, khi nào dùng Rekognition DetectText?**

> Textract cho **tài liệu** (form, hóa đơn, hợp đồng) cần hiểu cấu trúc và xử lý nhiều trang. Rekognition DetectText cho **ảnh đời thực** (biển báo, biển số xe) với lượng text ít.

**Q: Tài liệu PDF nhiều trang xử lý thế nào?**

> Bắt buộc dùng async API (`start_document_analysis` + `get_document_analysis`), kết hợp SNS notification và phân trang kết quả qua `NextToken`. Sync API chỉ hỗ trợ 1 trang.

**Q: IDP pipeline điển hình gồm những gì?**

> Textract (trích xuất) → A2I (human review cho confidence thấp) → Comprehend (phân tích entity/phân loại) → lưu vào DynamoDB/RDS → OpenSearch hoặc QuickSight cho tìm kiếm/báo cáo.

**Q: Textract Queries giải quyết vấn đề gì?**

> Thay vì viết code parse Blocks thủ công để tìm một trường, Queries cho phép **hỏi trực tiếp** bằng ngôn ngữ tự nhiên ("Tổng tiền là bao nhiêu?") và nhận đúng giá trị — giảm code và linh hoạt với layout tài liệu khác nhau.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
