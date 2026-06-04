# Amazon Rekognition Image — Phân Tích Ảnh Tĩnh

> Amazon Rekognition Image là API phân tích ảnh được quản lý hoàn toàn (fully-managed) của AWS, sử dụng deep learning (học sâu) đã được pre-trained (huấn luyện sẵn) để phát hiện object, khuôn mặt, văn bản, nội dung không phù hợp và nhiều hơn nữa — chỉ qua một lời gọi API, không cần huấn luyện model.

---

## 🎯 Rekognition Image Làm Được Gì?

| Tính Năng | API | Mô Tả |
|---|---|---|
| **Label Detection** (Phát Hiện Nhãn) | `detect_labels` | Nhận diện object, scene (cảnh), hoạt động |
| **Face Detection** (Phát Hiện Khuôn Mặt) | `detect_faces` | Tìm khuôn mặt + thuộc tính (tuổi, cảm xúc...) |
| **Face Comparison** (So Sánh Khuôn Mặt) | `compare_faces` | So 2 khuôn mặt xem có cùng người không |
| **Face Search** (Tìm Kiếm Khuôn Mặt) | `search_faces_by_image` | Tìm khuôn mặt trong collection (bộ sưu tập) |
| **Text Detection** (Phát Hiện Văn Bản) | `detect_text` | Đọc chữ trong ảnh (biển số, biển báo) |
| **Content Moderation** (Kiểm Duyệt) | `detect_moderation_labels` | Phát hiện nội dung nhạy cảm/không phù hợp |
| **PPE Detection** (Phát Hiện Đồ Bảo Hộ) | `detect_protective_equipment` | Phát hiện mũ, khẩu trang, găng tay bảo hộ |
| **Celebrity Recognition** (Nhận Diện Người Nổi Tiếng) | `recognize_celebrities` | Nhận diện người nổi tiếng |

---

## 🏷️ 1. Label Detection (Phát Hiện Nhãn)

Phát hiện hàng nghìn object, scene và concept trong ảnh, mỗi nhãn kèm **Confidence** (điểm tin cậy) và đôi khi **Bounding Box** (khung bao).

```python
import boto3

rekognition = boto3.client("rekognition", region_name="us-east-1")

def detect_labels(bucket: str, key: str):
    """Phát hiện object/scene trong ảnh trên S3."""
    response = rekognition.detect_labels(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        MaxLabels=15,          # Số nhãn tối đa trả về
        MinConfidence=75       # Ngưỡng tin cậy tối thiểu (%)
    )

    for label in response["Labels"]:
        print(f"{label['Name']}: {label['Confidence']:.1f}%")
        # Một số label có bounding box (vị trí cụ thể của object)
        for instance in label.get("Instances", []):
            box = instance["BoundingBox"]
            print(f"  → Vị trí: Left={box['Left']:.2f}, Top={box['Top']:.2f}")
```

### Phân Cấp Nhãn (Label Hierarchy)

Rekognition trả về **Parents** (nhãn cha) để thể hiện quan hệ phân cấp:

```
"Labrador" → Parents: ["Dog", "Pet", "Animal", "Mammal"]
"Car"      → Parents: ["Vehicle", "Transportation"]
```

Điều này giúp lọc theo mức độ khái quát mong muốn.

### Truyền Ảnh Trực Tiếp (Image Bytes)

Ngoài S3, có thể gửi ảnh trực tiếp dạng bytes (giới hạn 5 MB cho synchronous):

```python
def detect_labels_from_file(image_path: str):
    """Gửi ảnh local trực tiếp (không qua S3)."""
    with open(image_path, "rb") as image_file:
        response = rekognition.detect_labels(
            Image={"Bytes": image_file.read()},
            MaxLabels=10
        )
    return response["Labels"]
```

---

## 😀 2. Face Detection (Phát Hiện Khuôn Mặt)

Phát hiện khuôn mặt và phân tích thuộc tính chi tiết (facial attributes).

```python
def detect_faces(bucket: str, key: str):
    """Phát hiện khuôn mặt và phân tích thuộc tính."""
    response = rekognition.detect_faces(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        Attributes=["ALL"]   # "DEFAULT" (cơ bản) hoặc "ALL" (đầy đủ)
    )

    for face in response["FaceDetails"]:
        print(f"Độ tuổi ước tính: {face['AgeRange']['Low']}-{face['AgeRange']['High']}")
        print(f"Giới tính: {face['Gender']['Value']}")
        print(f"Đeo kính: {face['Eyeglasses']['Value']}")
        print(f"Mỉm cười: {face['Smile']['Value']}")

        # Emotions (cảm xúc) — trả về danh sách kèm confidence
        emotions = sorted(face["Emotions"], key=lambda e: e["Confidence"], reverse=True)
        print(f"Cảm xúc chính: {emotions[0]['Type']} ({emotions[0]['Confidence']:.1f}%)")
```

### Thuộc Tính Khuôn Mặt Trả Về

| Thuộc Tính | Ý Nghĩa |
|---|---|
| `AgeRange` | Khoảng tuổi ước tính (Low–High) |
| `Gender` | Giới tính biểu hiện qua ngoại hình |
| `Emotions` | Cảm xúc: HAPPY, SAD, ANGRY, SURPRISED, CALM... |
| `Smile`, `Eyeglasses`, `Sunglasses`, `Beard` | Có/không + confidence |
| `EyesOpen`, `MouthOpen` | Trạng thái mắt/miệng |
| `Pose` | Góc nghiêng đầu (Pitch, Roll, Yaw) |
| `Quality` | Độ sáng (Brightness) và độ nét (Sharpness) |
| `Landmarks` | Tọa độ các điểm mốc (mắt, mũi, miệng) |

> **Lưu ý đạo đức:** Các thuộc tính như tuổi, giới tính, cảm xúc là **ước lượng (prediction)**, không phải sự thật tuyệt đối. Không nên dùng cho quyết định ảnh hưởng quyền lợi con người mà không có kiểm soát Responsible AI (AI Có Trách Nhiệm).

---

## 🔍 3. Face Comparison & Face Search

### Compare Faces (So Sánh 2 Khuôn Mặt)

So sánh khuôn mặt trong ảnh nguồn (source) với ảnh đích (target) — dùng cho identity verification (xác minh danh tính).

```python
def compare_faces(source_bucket, source_key, target_bucket, target_key):
    """So sánh khuôn mặt giữa 2 ảnh."""
    response = rekognition.compare_faces(
        SourceImage={"S3Object": {"Bucket": source_bucket, "Name": source_key}},
        TargetImage={"S3Object": {"Bucket": target_bucket, "Name": target_key}},
        SimilarityThreshold=90   # Ngưỡng tương đồng tối thiểu (%)
    )

    for match in response["FaceMatches"]:
        print(f"Khớp với độ tương đồng: {match['Similarity']:.1f}%")
```

### Face Collection — Tìm Kiếm Khuôn Mặt Quy Mô Lớn

Để nhận dạng người trong cơ sở dữ liệu lớn, dùng **Collection** (bộ sưu tập face vectors — vectơ đặc trưng khuôn mặt):

```
1. Tạo Collection                  → create_collection
2. Lập chỉ mục khuôn mặt vào đó     → index_faces (lưu FaceId, không lưu ảnh gốc)
3. Tìm người từ ảnh mới             → search_faces_by_image
```

```python
def setup_and_search(collection_id, enroll_bucket, enroll_key, query_bucket, query_key):
    """Tạo collection, đăng ký khuôn mặt và tìm kiếm."""
    # Bước 1: Tạo collection (chỉ làm 1 lần)
    rekognition.create_collection(CollectionId=collection_id)

    # Bước 2: Index khuôn mặt — Rekognition lưu vector đặc trưng, không lưu ảnh
    rekognition.index_faces(
        CollectionId=collection_id,
        Image={"S3Object": {"Bucket": enroll_bucket, "Name": enroll_key}},
        ExternalImageId="user_12345",   # ID nội bộ của bạn để map về user
        DetectionAttributes=["DEFAULT"]
    )

    # Bước 3: Tìm khuôn mặt khớp từ ảnh truy vấn
    result = rekognition.search_faces_by_image(
        CollectionId=collection_id,
        Image={"S3Object": {"Bucket": query_bucket, "Name": query_key}},
        FaceMatchThreshold=95,
        MaxFaces=1
    )
    for match in result["FaceMatches"]:
        print(f"User: {match['Face']['ExternalImageId']}, "
              f"Tương đồng: {match['Similarity']:.1f}%")
```

> **Quan trọng về Privacy (Quyền Riêng Tư):** Collection lưu **face vectors** (biểu diễn toán học), không lưu ảnh gốc. Vẫn cần tuân thủ quy định về dữ liệu sinh trắc học (biometric data) tại khu vực bạn hoạt động.

---

## 📝 4. Text Detection (Phát Hiện Văn Bản Trong Ảnh)

Đọc chữ xuất hiện trong ảnh tự nhiên (biển báo, biển số xe, nhãn sản phẩm). Khác với Textract — Text Detection của Rekognition phù hợp với **ảnh đời thực**, còn Textract chuyên về **tài liệu**.

```python
def detect_text(bucket: str, key: str):
    """Đọc văn bản trong ảnh (scene text)."""
    response = rekognition.detect_text(
        Image={"S3Object": {"Bucket": bucket, "Name": key}}
    )

    for item in response["TextDetections"]:
        # Type: "LINE" (dòng) hoặc "WORD" (từ)
        if item["Type"] == "LINE":
            print(f"{item['DetectedText']} (tin cậy: {item['Confidence']:.1f}%)")
```

| Tiêu Chí | Rekognition DetectText | Textract |
|---|---|---|
| Input lý tưởng | Ảnh đời thực (biển báo, biển số) | Tài liệu scan, PDF |
| Cấu trúc form/table | ❌ | ✅ |
| Giới hạn | ~100 từ/ảnh | Trang đầy đủ |

---

## 🛡️ 5. Content Moderation (Kiểm Duyệt Nội Dung)

Phát hiện nội dung không phù hợp (explicit, suggestive, violence...) — thiết yếu cho user-generated content (nội dung do người dùng đăng).

```python
def moderate_image(bucket: str, key: str):
    """Kiểm duyệt nội dung ảnh."""
    response = rekognition.detect_moderation_labels(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        MinConfidence=60
    )

    flagged = response["ModerationLabels"]
    if flagged:
        for label in flagged:
            # ParentName: category cha (VD: "Violence")
            # Name: nhãn cụ thể (VD: "Weapon Violence")
            print(f"⚠️ {label['ParentName']} > {label['Name']}: "
                  f"{label['Confidence']:.1f}%")
        return False  # Nội dung cần chặn/review
    return True       # Nội dung an toàn
```

**Các category kiểm duyệt chính:** Explicit Nudity, Suggestive, Violence, Visually Disturbing, Drugs, Tobacco, Alcohol, Gambling, Hate Symbols.

> Kết hợp với **human-in-the-loop** (con người trong vòng lặp) qua **Amazon A2I** (Augmented AI — AI Tăng Cường) để review các trường hợp confidence trung bình.

---

## 🦺 6. PPE Detection (Phát Hiện Đồ Bảo Hộ Lao Động)

Phát hiện Personal Protective Equipment (Thiết Bị Bảo Hộ Cá Nhân) — ứng dụng cho an toàn lao động, công trường, nhà máy.

```python
def detect_ppe(bucket: str, key: str):
    """Phát hiện đồ bảo hộ trên người trong ảnh."""
    response = rekognition.detect_protective_equipment(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        SummarizationAttributes={
            "MinConfidence": 80,
            "RequiredEquipmentTypes": ["FACE_COVER", "HEAD_COVER", "HAND_COVER"]
        }
    )

    summary = response["Summary"]
    print(f"Số người tuân thủ đầy đủ: {len(summary['PersonsWithRequiredEquipment'])}")
    print(f"Số người thiếu bảo hộ: {len(summary['PersonsWithoutRequiredEquipment'])}")
```

| Loại PPE | Phát Hiện |
|---|---|
| `FACE_COVER` | Khẩu trang / che mặt |
| `HEAD_COVER` | Mũ bảo hộ |
| `HAND_COVER` | Găng tay |

---

## 🌟 7. Celebrity Recognition (Nhận Diện Người Nổi Tiếng)

```python
def recognize_celebrities(bucket: str, key: str):
    """Nhận diện người nổi tiếng trong ảnh."""
    response = rekognition.recognize_celebrities(
        Image={"S3Object": {"Bucket": bucket, "Name": key}}
    )
    for celeb in response["CelebrityFaces"]:
        print(f"{celeb['Name']} (tin cậy: {celeb['MatchConfidence']:.1f}%)")
        print(f"  Liên kết: {celeb.get('Urls', [])}")
```

---

## 📊 Kiến Trúc Ứng Dụng Thực Tế

### Hệ Thống Kiểm Duyệt Ảnh Tự Động

```
User upload ảnh
      │
      ▼
S3 Bucket  ──(S3 Event Notification)──►  Lambda
                                            │
                                            ├──► detect_moderation_labels
                                            │       │
                                            │       ├─ An toàn → publish ảnh
                                            │       └─ Vi phạm → A2I human review
                                            │
                                            └──► DynamoDB (lưu kết quả + audit log)
```

### Hệ Thống Chấm Công Bằng Khuôn Mặt

```
Camera chụp ảnh nhân viên
      │
      ▼
API Gateway → Lambda → search_faces_by_image (trên Collection nhân viên)
                              │
                              ├─ Khớp ≥ 95% → ghi nhận chấm công (DynamoDB)
                              └─ Không khớp → từ chối + cảnh báo
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: detect_faces khác search_faces_by_image thế nào?**

> `detect_faces` chỉ phát hiện và phân tích thuộc tính khuôn mặt (tuổi, cảm xúc) — không biết "đó là ai". `search_faces_by_image` so khớp khuôn mặt với một Collection đã index để **nhận dạng danh tính**.

**Q: Rekognition DetectText khác Textract khi nào?**

> DetectText đọc chữ trong ảnh đời thực (biển báo, biển số) tối đa ~100 từ. Textract chuyên đọc **tài liệu** có cấu trúc (form, bảng, hóa đơn) với độ chính xác cao hơn cho văn bản tài liệu.

**Q: Confidence threshold nên đặt bao nhiêu?**

> Tùy use case: an ninh/xác minh danh tính nên ≥ 95% kèm human review; khám phá nội dung có thể 70–80%. Không có con số "đúng" tuyệt đối — cân bằng giữa false positive (báo nhầm) và false negative (bỏ sót).

**Q: Rekognition có lưu ảnh của tôi không?**

> Với các API stateless (detect_*, compare_*) thì không lưu. Với Collection, Rekognition lưu **face vectors** (không phải ảnh gốc). Có thể chọn không cho AWS dùng dữ liệu cải thiện dịch vụ qua AI services opt-out policy.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
