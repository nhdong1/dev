# Computer Vision (Thị Giác Máy Tính) trên AWS — Toàn Diện

> Computer Vision — Thị Giác Máy Tính là lĩnh vực AI giúp máy tính "nhìn" và hiểu nội dung trong ảnh và video. Trên AWS, hai dịch vụ chủ lực là **Amazon Rekognition** (Nhận Diện Hình Ảnh & Video) và **Amazon Textract** (Trích Xuất Tài Liệu) — đều là AI Services được quản lý hoàn toàn (fully-managed), gọi qua API, không cần ML expertise (chuyên môn học máy).

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|---|---|---|
| `README.md` | Tổng quan Computer Vision trên AWS, so sánh dịch vụ | ✅ |
| `1-rekognition-image.md` | Labels, Faces, Text, Moderation, PPE detection | ✅ |
| `2-rekognition-video.md` | Activities, Celebrities, phân tích video streaming | ✅ |
| `3-rekognition-custom-labels.md` | Custom training, Auto-labeling, đánh giá model | ✅ |
| `4-textract.md` | OCR, Analyze Document, Forms, Tables, Expense | ✅ |

---

## 🎯 Computer Vision Là Gì?

Computer Vision (Thị Giác Máy Tính) cho phép phần mềm phân tích và rút trích thông tin có ý nghĩa từ dữ liệu hình ảnh:

- **Image Classification** (Phân Loại Ảnh): Gán nhãn cho toàn bộ ảnh (VD: "mèo", "ô tô")
- **Object Detection** (Phát Hiện Đối Tượng): Tìm vị trí và bounding box (khung bao) của từng vật thể
- **Facial Analysis** (Phân Tích Khuôn Mặt): Phát hiện, so sánh, nhận dạng khuôn mặt
- **Text Detection / OCR** (Nhận Diện Văn Bản — Optical Character Recognition — Nhận Dạng Ký Tự Quang Học): Đọc chữ trong ảnh
- **Content Moderation** (Kiểm Duyệt Nội Dung): Phát hiện nội dung không phù hợp
- **Segmentation** (Phân Đoạn): Tách từng pixel thuộc về object nào

### Vị Trí Trong Hệ Sinh Thái AWS AI

```
┌─────────────────────────────────────────────────────────────────┐
│  Tầng 3: AI Services (Dịch Vụ AI Được Quản Lý)  ◄── BẠN Ở ĐÂY │
│  Rekognition │ Textract │ Comprehend │ Polly │ Transcribe       │
├─────────────────────────────────────────────────────────────────┤
│  Tầng 2: ML Services                                            │
│   Amazon SageMaker (dùng khi cần custom CV model phức tạp)      │
├─────────────────────────────────────────────────────────────────┤
│  Tầng 1: ML Framework & Infrastructure                          │
│  TensorFlow │ PyTorch │ OpenCV │ GPU Instances                  │
└─────────────────────────────────────────────────────────────────┘
```

**Quy tắc chọn:** Dùng AI Services (Rekognition/Textract) khi bài toán nằm trong khả năng có sẵn của chúng — nhanh, rẻ, không cần data. Chỉ dùng SageMaker custom model khi yêu cầu rất đặc thù mà dịch vụ managed không đáp ứng.

---

## 🗂️ So Sánh Dịch Vụ Computer Vision Trên AWS

| Dịch Vụ | Mục Đích | Loại Input | Output | Tính Phí |
|---|---|---|---|---|
| **Rekognition Image** | Phân tích ảnh tĩnh | Ảnh (JPEG/PNG) | Labels, faces, text, moderation | Per image (mỗi ảnh) |
| **Rekognition Video** | Phân tích video | Video (S3 hoặc stream) | Activities, faces, celebrities | Per phút video |
| **Rekognition Custom Labels** | Nhận diện object tùy chỉnh | Ảnh + nhãn của bạn | Custom labels | Training-hour + inference-hour |
| **Textract** | Đọc tài liệu / OCR | Tài liệu scan, PDF | Text, forms, tables | Per trang tài liệu |

### Khi Nào Dùng Cái Nào?

```
Bạn cần xử lý gì?
│
├─ Ảnh tĩnh — nhận diện object/khuôn mặt/text chung chung?
│   └─► Rekognition Image (detect-labels, detect-faces, detect-text)
│
├─ Video — phát hiện hoạt động, người nổi tiếng, kiểm duyệt?
│   └─► Rekognition Video (stored hoặc streaming)
│
├─ Nhận diện object đặc thù của bạn (VD: logo công ty, linh kiện lỗi)?
│   └─► Rekognition Custom Labels (train với ảnh riêng)
│
├─ Tài liệu có chữ — hóa đơn, form, hợp đồng, bảng biểu?
│   └─► Textract (OCR + cấu trúc form/table)
│
└─ Bài toán CV rất đặc thù (segmentation y khoa, 3D...)?
    └─► SageMaker custom model
```

---

## 🆚 Rekognition Custom Labels vs SageMaker

| Tiêu Chí | Rekognition Custom Labels | SageMaker (custom CV) |
|---|---|---|
| **ML expertise cần** | Thấp — chỉ cần ảnh + nhãn | Cao — cần biết deep learning |
| **Dữ liệu cần** | Ít (vài chục → vài trăm ảnh) | Nhiều, tự chuẩn bị pipeline |
| **Thời gian triển khai** | Giờ | Ngày → tuần |
| **Kiểm soát architecture** | Không (AWS lo) | Hoàn toàn |
| **Phù hợp** | Use case CV phổ biến, tùy chỉnh nhẹ | Yêu cầu rất đặc thù, nghiên cứu |

---

## 🚀 Quick Start — Gọi Rekognition & Textract

### Cài Đặt Client (boto3)

```python
import boto3

# Rekognition client (xử lý ảnh/video)
rekognition = boto3.client("rekognition", region_name="us-east-1")

# Textract client (xử lý tài liệu)
textract = boto3.client("textract", region_name="us-east-1")
```

### Phát Hiện Nhãn Trong Ảnh (Detect Labels)

```python
def detect_labels(bucket: str, key: str, max_labels: int = 10):
    """Phát hiện các object/scene trong ảnh lưu trên S3."""
    response = rekognition.detect_labels(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        MaxLabels=max_labels,
        MinConfidence=80  # Chỉ lấy nhãn có độ tin cậy >= 80%
    )

    for label in response["Labels"]:
        print(f"{label['Name']}: {label['Confidence']:.1f}%")
```

### Đọc Text Trong Tài Liệu (Textract)

```python
def extract_text(bucket: str, key: str):
    """Trích xuất toàn bộ text từ tài liệu trên S3."""
    response = textract.detect_document_text(
        Document={"S3Object": {"Bucket": bucket, "Name": key}}
    )

    lines = [block["Text"] for block in response["Blocks"]
             if block["BlockType"] == "LINE"]
    return "\n".join(lines)
```

---

## 🔑 Khái Niệm Chung Cần Nắm

### Confidence Score (Điểm Tin Cậy)

Mọi kết quả Rekognition/Textract đều kèm **Confidence** (0–100%) — mức độ chắc chắn của model:

- Đặt `MinConfidence` (ngưỡng tin cậy tối thiểu) để lọc kết quả nhiễu
- Use case quan trọng (an ninh, y tế): đặt ngưỡng cao (95%+) hoặc thêm human review (kiểm duyệt bởi người)
- Use case khám phá: ngưỡng thấp hơn để không bỏ sót

### Bounding Box (Khung Bao Đối Tượng)

Tọa độ trả về dạng **tỷ lệ (ratio) 0–1** so với kích thước ảnh, không phải pixel tuyệt đối:

```
{
  "Width": 0.25,    # Chiều rộng box = 25% chiều rộng ảnh
  "Height": 0.40,   # Chiều cao box = 40% chiều cao ảnh
  "Left": 0.10,     # Cạnh trái cách mép trái 10%
  "Top": 0.30       # Cạnh trên cách mép trên 30%
}
```

Để vẽ lên ảnh: nhân các giá trị này với chiều rộng/cao thực tế (pixel) của ảnh.

### Sync vs Async (Đồng Bộ vs Bất Đồng Bộ)

| Chế Độ | API | Phù Hợp |
|---|---|---|
| **Synchronous** (Đồng Bộ) | `detect_*` — trả kết quả ngay | Ảnh nhỏ, tài liệu 1 trang |
| **Asynchronous** (Bất Đồng Bộ) | `start_*` + `get_*` — chạy nền | Video, PDF nhiều trang, tài liệu lớn |

Async dùng **SNS** (Simple Notification Service — Dịch Vụ Thông Báo) để báo job hoàn thành, hoặc polling (hỏi định kỳ) `get_*` bằng JobId.

---

## 📊 Kiến Trúc Điển Hình

### Pipeline Xử Lý Ảnh Upload

```
User upload ảnh
      │
      ▼
S3 Bucket  ──(S3 Event)──►  Lambda Function
                                  │
                                  ├──► Rekognition detect-labels / moderation
                                  │
                                  ├──► DynamoDB (lưu metadata + labels)
                                  │
                                  └──► SNS (cảnh báo nếu nội dung vi phạm)
```

### Pipeline Xử Lý Tài Liệu (Document Processing)

```
PDF/ảnh hóa đơn
      │
      ▼
S3  ──►  Lambda  ──►  Textract (Analyze Document / Expense)
                            │
                            ▼
                  Trích xuất Forms + Tables
                            │
                  ┌─────────┴──────────┐
                  ▼                    ▼
            DynamoDB              Comprehend (phân tích text)
        (dữ liệu cấu trúc)      (entity, sentiment)
```

---

## 💰 Lưu Ý Chi Phí

| Dịch Vụ | Mô Hình Giá | Mẹo Tiết Kiệm |
|---|---|---|
| Rekognition Image | Per image processed | Free Tier: 5.000 ảnh/tháng (12 tháng đầu) |
| Rekognition Video | Per phút video | Cắt video, chỉ phân tích đoạn cần thiết |
| Custom Labels | Training-hour + **inference-hour** | **Tắt model (stop) khi không dùng** — tính phí theo giờ kể cả idle! |
| Textract | Per trang; AnalyzeDocument đắt hơn DetectText | Dùng DetectText nếu chỉ cần OCR thuần |

> **Cảnh báo quan trọng:** Rekognition Custom Labels tính phí **inference-hour** giống SageMaker endpoint — luôn `stop_project_version` khi không dùng để tránh hóa đơn bất ngờ.

---

## 🔗 Điều Hướng Module

| Chủ Đề | File |
|---|---|
| Rekognition Image — phân tích ảnh tĩnh | [1-rekognition-image.md](1-rekognition-image.md) |
| Rekognition Video — phân tích video | [2-rekognition-video.md](2-rekognition-video.md) |
| Rekognition Custom Labels — model tùy chỉnh | [3-rekognition-custom-labels.md](3-rekognition-custom-labels.md) |
| Textract — OCR & trích xuất tài liệu | [4-textract.md](4-textract.md) |

---

## 🎯 Câu Hỏi Phỏng Vấn Nhanh

1. **Rekognition vs Textract — khác nhau thế nào?** → Rekognition cho computer vision tổng quát (object, face, scene); Textract chuyên đọc tài liệu (OCR + form + table có cấu trúc).
2. **Khi nào cần Custom Labels thay vì detect-labels built-in?** → Khi cần nhận diện object đặc thù không có trong tập nhãn chung của AWS (VD: logo riêng, sản phẩm cụ thể, lỗi sản xuất).
3. **Confidence Score dùng để làm gì?** → Lọc kết quả nhiễu và quyết định có cần human review hay không cho use case nhạy cảm.
4. **Xử lý video dài thì dùng API nào?** → Async API (`start_*` + `get_*`) kết hợp SNS notification, vì video không thể xử lý đồng bộ.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
