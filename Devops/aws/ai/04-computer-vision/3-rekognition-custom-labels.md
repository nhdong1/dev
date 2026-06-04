# Amazon Rekognition Custom Labels — Model Nhận Diện Tùy Chỉnh

> Rekognition Custom Labels cho phép huấn luyện một model computer vision **của riêng bạn** để nhận diện các object, scene và concept **đặc thù** mà Rekognition built-in không hỗ trợ — ví dụ: logo công ty, linh kiện lỗi trên dây chuyền, giống cây trồng cụ thể. Tất cả qua giao diện đơn giản, **không cần ML expertise** (chuyên môn học máy), chỉ cần ảnh có gán nhãn.

---

## 🎯 Khi Nào Cần Custom Labels?

| Tình Huống | Dùng Gì? |
|---|---|
| Nhận diện object phổ biến (chó, xe, người) | Rekognition `detect_labels` built-in |
| Nhận diện **logo riêng** của thương hiệu bạn | **Custom Labels** |
| Phát hiện **sản phẩm lỗi** đặc thù trên dây chuyền | **Custom Labels** |
| Phân loại **giống hoa/cây** chuyên ngành | **Custom Labels** |
| Bài toán CV rất phức tạp (segmentation y khoa 3D) | SageMaker custom model |

**Nguyên tắc:** Custom Labels lấp khoảng trống giữa built-in (quá chung) và SageMaker (quá phức tạp) — đó là **transfer learning** (học chuyển giao) được quản lý, AWS đã lo phần model nền, bạn chỉ cung cấp data đặc thù.

---

## 🔄 Quy Trình Xây Dựng Custom Labels (End-to-End)

```
1. Tạo Project (Dự Án)
        │
        ▼
2. Tạo Dataset (Tập Dữ Liệu) — gán nhãn ảnh
   ├─ Image-level labels (nhãn cấp ảnh) — phân loại
   └─ Bounding box labels (nhãn khung bao) — object detection
        │
        ▼
3. Train Model (Huấn Luyện) — AWS tự chọn thuật toán + tune
        │
        ▼
4. Evaluate Model (Đánh Giá) — xem Precision, Recall, F1
        │
        ▼
5. Start Model (Khởi Động) — deploy để inference (tính phí/giờ!)
        │
        ▼
6. Detect Custom Labels (Suy Luận) — gọi API với ảnh mới
        │
        ▼
7. Stop Model (Dừng) — TẮT khi không dùng để tiết kiệm chi phí
```

---

## 📁 Bước 1–2: Project & Dataset

### Hai Loại Bài Toán

| Loại | Nhãn | Output |
|---|---|---|
| **Image Classification** (Phân Loại Ảnh) | Image-level label (nhãn cho cả ảnh) | "ảnh này là sản phẩm lỗi" |
| **Object Detection** (Phát Hiện Đối Tượng) | Bounding box + label | "có 3 lỗi tại các vị trí X, Y, Z" |

### Cách Gán Nhãn (Labeling)

| Phương Pháp | Mô Tả |
|---|---|
| **Auto-labeling từ S3 folder** | Tên thư mục = nhãn (cho classification) |
| **Rekognition Console** | Vẽ bounding box trực tiếp trên UI |
| **Amazon SageMaker Ground Truth** | Dịch vụ gán nhãn quy mô lớn, dùng human workforce |
| **Manifest file** | Import nhãn có sẵn dạng JSON (SageMaker format) |

### Tạo Project Qua API

```python
import boto3

rekognition = boto3.client("rekognition", region_name="us-east-1")

# Bước 1: Tạo project
project_response = rekognition.create_project(ProjectName="defect-detection")
project_arn = project_response["ProjectArn"]
print(f"Project ARN: {project_arn}")
```

### Cấu Trúc S3 Cho Auto-Labeling (Classification)

```
s3://my-bucket/training/
├── normal/          ← Nhãn "normal" (bình thường)
│   ├── img001.jpg
│   └── img002.jpg
└── defective/       ← Nhãn "defective" (lỗi)
    ├── img101.jpg
    └── img102.jpg
```

---

## 🏋️ Bước 3: Train Model (Huấn Luyện)

AWS tự động: chọn architecture, chia train/test, tune hyperparameter — bạn **không cần** can thiệp.

```python
def train_model(project_arn, version_name, output_bucket,
                training_manifest, testing_manifest):
    """Huấn luyện một phiên bản model Custom Labels."""
    response = rekognition.create_project_version(
        ProjectArn=project_arn,
        VersionName=version_name,
        OutputConfig={"S3Bucket": output_bucket, "S3KeyPrefix": "output/"},
        TrainingData={
            "Assets": [{"GroundTruthManifest": {
                "S3Object": {"Bucket": output_bucket, "Name": training_manifest}}}]
        },
        TestingData={
            "Assets": [{"GroundTruthManifest": {
                "S3Object": {"Bucket": output_bucket, "Name": testing_manifest}}}]
            # Hoặc: "AutoCreate": True → AWS tự tách 20% train làm test
        }
    )
    return response["ProjectVersionArn"]
```

### Dữ Liệu Tối Thiểu

- **Tối thiểu:** ~10 ảnh/nhãn để bắt đầu
- **Khuyến nghị:** 50+ ảnh/nhãn, đa dạng góc chụp, ánh sáng, nền
- **Test set:** tách riêng, KHÔNG trùng với training set (tránh data leakage — rò rỉ dữ liệu)

---

## 📈 Bước 4: Evaluate Model (Đánh Giá) — Phần Quan Trọng Nhất

Sau khi train, Rekognition cung cấp các metric đánh giá. Hiểu chúng là **bắt buộc** cho câu hỏi phỏng vấn.

### Các Metric Chính

| Metric | Công Thức | Ý Nghĩa |
|---|---|---|
| **Precision** (Độ Chính Xác) | TP / (TP + FP) | Trong các dự đoán "dương", bao nhiêu % đúng? |
| **Recall** (Độ Bao Phủ / Độ Nhạy) | TP / (TP + FN) | Trong các trường hợp dương thực tế, bắt được bao nhiêu %? |
| **F1 Score** | 2 × (P × R) / (P + R) | Trung bình điều hòa của Precision & Recall |

*(TP = True Positive — dương tính đúng; FP = False Positive — dương tính giả; FN = False Negative — âm tính giả)*

### Precision vs Recall — Trade-off (Đánh Đổi)

```
Ưu tiên PRECISION cao (giảm báo nhầm):
  → Kiểm duyệt nội dung: không muốn chặn nhầm ảnh hợp lệ
  → Đặt confidence threshold CAO

Ưu tiên RECALL cao (giảm bỏ sót):
  → Phát hiện lỗi sản xuất / khối u y tế: không được bỏ sót ca thật
  → Đặt confidence threshold THẤP
```

### Assumed Threshold & F1

Rekognition tính một **assumed threshold** (ngưỡng giả định) cho mỗi nhãn để tối ưu F1. Khi inference, bạn có thể override ngưỡng này tùy nhu cầu Precision/Recall.

```python
def get_evaluation(project_version_arn):
    """Lấy kết quả đánh giá model sau khi train xong."""
    response = rekognition.describe_project_versions(
        ProjectArn=project_version_arn.rsplit("/", 2)[0],
    )
    for version in response["ProjectVersionDescriptions"]:
        metrics = version.get("EvaluationResult", {})
        print(f"F1 Score trung bình: {metrics.get('F1Score')}")
        # Chi tiết per-label nằm trong file Summary trên S3 output
```

---

## 🚀 Bước 5–6: Start Model & Inference

### Khởi Động Model

> ⚠️ **Đây là bước phát sinh chi phí lớn nhất.** Model tính phí theo **inference-hour** (giờ suy luận) kể từ lúc start, **kể cả khi không có request nào** — giống SageMaker endpoint.

```python
def start_model(project_version_arn, min_inference_units=1):
    """Khởi động model để sẵn sàng inference."""
    rekognition.start_project_version(
        ProjectVersionArn=project_version_arn,
        MinInferenceUnits=min_inference_units   # Số đơn vị throughput
    )
    # Chờ model chuyển sang trạng thái RUNNING (vài phút)
    waiter = rekognition.get_waiter("project_version_running")
    waiter.wait(
        ProjectArn=project_version_arn.rsplit("/", 2)[0],
        VersionNames=[project_version_arn.split("/")[-2]]
    )
```

### Gọi Detect Custom Labels

```python
def detect_custom_labels(project_version_arn, bucket, key):
    """Nhận diện custom labels trên ảnh mới."""
    response = rekognition.detect_custom_labels(
        ProjectVersionArn=project_version_arn,
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        MinConfidence=70,
        MaxResults=10
    )
    for label in response["CustomLabels"]:
        print(f"{label['Name']}: {label['Confidence']:.1f}%")
        if "Geometry" in label:   # Object detection trả về bounding box
            box = label["Geometry"]["BoundingBox"]
            print(f"  Vị trí: {box}")
```

---

## 🛑 Bước 7: Stop Model — TIẾT KIỆM CHI PHÍ

```python
def stop_model(project_version_arn):
    """Dừng model khi không dùng — NGỪNG tính phí inference-hour."""
    rekognition.stop_project_version(ProjectVersionArn=project_version_arn)
```

> **Best practice:** Trong môi trường on-demand (không cần inference 24/7), dùng EventBridge Scheduler hoặc Lambda để **start model trước batch xử lý**, rồi **stop ngay sau khi xong**.

---

## 📊 So Sánh Custom Labels vs Built-in vs SageMaker

| Tiêu Chí | Built-in (detect_labels) | Custom Labels | SageMaker CV |
|---|---|---|---|
| **Nhãn** | Hàng nghìn nhãn chung của AWS | Nhãn riêng của bạn | Hoàn toàn tự định nghĩa |
| **Cần training data** | Không | Ít (10–50+ ảnh/nhãn) | Nhiều, tự chuẩn bị |
| **ML expertise** | Không | Thấp | Cao |
| **Chi phí** | Per image | Train-hour + inference-hour | Instance-hour |
| **Idle cost** | Không | **Có (model phải start)** | Có (endpoint) |
| **Kiểm soát** | Không | Hạn chế | Hoàn toàn |

---

## 🏭 Kiến Trúc Thực Tế: Phát Hiện Lỗi Sản Xuất

```
Camera trên dây chuyền chụp sản phẩm
            │
            ▼
S3 (ảnh sản phẩm)  ──(S3 Event)──►  Lambda
                                       │
                                       └──► detect_custom_labels
                                              │
                                    ┌─────────┴──────────┐
                                    ▼                    ▼
                              "normal"            "defective" (lỗi)
                                    │                    │
                              tiếp tục            ├─ Loại bỏ sản phẩm
                                                  ├─ SNS cảnh báo QA
                                                  └─ DynamoDB (log lỗi)
```

> Mô hình tối ưu chi phí: dùng **một model luôn chạy** nếu dây chuyền hoạt động liên tục; hoặc **batch + start/stop** nếu kiểm tra theo ca.

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Custom Labels khác detect_labels built-in thế nào?**

> Built-in nhận diện nhãn chung mà AWS đã train sẵn (chó, xe, người...). Custom Labels cho phép train model riêng nhận diện object đặc thù của bạn (logo, lỗi sản phẩm) bằng transfer learning — chỉ cần ít ảnh có nhãn.

**Q: Tại sao phải stop model? Chi phí tính thế nào?**

> Model Custom Labels tính phí theo **inference-hour** kể từ lúc start, kể cả khi idle (không có request). Nếu quên stop, hóa đơn tăng liên tục. Luôn stop khi không dùng, hoặc dùng pattern start-batch-stop.

**Q: Precision và Recall khác nhau thế nào? Khi nào ưu tiên cái nào?**

> Precision = % dự đoán dương đúng (giảm báo nhầm). Recall = % ca dương thực tế bắt được (giảm bỏ sót). Ưu tiên Precision cho kiểm duyệt (không chặn nhầm); ưu tiên Recall cho phát hiện lỗi/bệnh (không bỏ sót ca thật).

**Q: Cần bao nhiêu ảnh để train Custom Labels?**

> Tối thiểu ~10 ảnh/nhãn, nhưng khuyến nghị 50+ ảnh đa dạng (góc, ánh sáng, nền). Chất lượng và đa dạng dữ liệu quan trọng hơn số lượng thuần túy.

**Q: F1 Score là gì và tại sao Rekognition dùng nó?**

> F1 là trung bình điều hòa của Precision và Recall, cân bằng cả hai. Rekognition dùng F1 để chọn assumed threshold tối ưu cho mỗi nhãn — vì chỉ tối ưu một metric (chỉ Precision hoặc chỉ Recall) sẽ lệch.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
