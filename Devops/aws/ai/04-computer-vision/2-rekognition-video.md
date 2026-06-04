# Amazon Rekognition Video — Phân Tích Video

> Amazon Rekognition Video mở rộng khả năng computer vision sang video — phát hiện hoạt động (activities), người nổi tiếng, khuôn mặt, object, kiểm duyệt nội dung và theo dõi người qua thời gian. Video được xử lý theo hai chế độ: **Stored Video** (video lưu trữ trên S3, bất đồng bộ) và **Streaming Video** (video trực tiếp qua Kinesis).

---

## 🎬 Hai Chế Độ Phân Tích Video

| Chế Độ | Nguồn Video | Cách Hoạt Động | Phù Hợp |
|---|---|---|---|
| **Stored Video** (Video Lưu Trữ) | File trên S3 | Async — `start_*` rồi `get_*` | Video on-demand, phân tích offline |
| **Streaming Video** (Video Trực Tiếp) | Kinesis Video Stream | Real-time qua Stream Processor | Camera giám sát, live event |

---

## 📦 Phần 1: Stored Video Analysis (Phân Tích Video Lưu Trữ)

Video không thể xử lý đồng bộ (mất thời gian), nên Rekognition dùng mô hình **bất đồng bộ (asynchronous)**:

```
1. start_* (khởi động job)  →  trả về JobId ngay lập tức
2. Rekognition xử lý nền    →  hoàn thành sau vài giây → vài phút
3. SNS notification         →  thông báo job xong (hoặc polling get_*)
4. get_* (lấy kết quả)      →  dùng JobId để lấy kết quả phân tích
```

### Các Loại Phân Tích Stored Video

| Tính Năng | Start API | Get API |
|---|---|---|
| **Label Detection** (Object/Scene/Activity) | `start_label_detection` | `get_label_detection` |
| **Face Detection** (Khuôn Mặt) | `start_face_detection` | `get_face_detection` |
| **Face Search** (Tìm Người Trong Collection) | `start_face_search` | `get_face_search` |
| **Person Tracking** (Theo Dõi Người) | `start_person_tracking` | `get_person_tracking` |
| **Celebrity Recognition** (Người Nổi Tiếng) | `start_celebrity_recognition` | `get_celebrity_recognition` |
| **Content Moderation** (Kiểm Duyệt) | `start_content_moderation` | `get_content_moderation` |
| **Text Detection** (Văn Bản) | `start_text_detection` | `get_text_detection` |
| **Segment Detection** (Phân Cảnh) | `start_segment_detection` | `get_segment_detection` |

### Ví Dụ: Phát Hiện Label Trong Video (Polling)

```python
import boto3
import time

rekognition = boto3.client("rekognition", region_name="us-east-1")

def detect_labels_in_video(bucket: str, key: str):
    """Phát hiện object/activity trong video lưu trên S3 (dùng polling)."""
    # Bước 1: Khởi động job phân tích
    start_response = rekognition.start_label_detection(
        Video={"S3Object": {"Bucket": bucket, "Name": key}},
        MinConfidence=80
    )
    job_id = start_response["JobId"]
    print(f"Đã khởi động job: {job_id}")

    # Bước 2: Polling (hỏi định kỳ) cho đến khi xong
    while True:
        result = rekognition.get_label_detection(JobId=job_id)
        status = result["JobStatus"]
        if status in ("SUCCEEDED", "FAILED"):
            break
        print("Đang xử lý...")
        time.sleep(5)

    if status == "FAILED":
        raise RuntimeError("Job phân tích thất bại")

    # Bước 3: Đọc kết quả — mỗi label kèm Timestamp (mili-giây)
    for item in result["Labels"]:
        ts_seconds = item["Timestamp"] / 1000
        label = item["Label"]
        print(f"[{ts_seconds:.1f}s] {label['Name']}: {label['Confidence']:.1f}%")
```

### Timestamp — Đặc Trưng Quan Trọng Của Video

Khác với ảnh tĩnh, kết quả video luôn kèm **Timestamp** (mốc thời gian, đơn vị mili-giây tính từ đầu video) — cho biết object/hoạt động xuất hiện ở **thời điểm nào**:

```
[0.0s]   Person: 99.2%
[2.5s]   Person: 98.7%, Dog: 95.1%
[5.0s]   Person: 97.3%, Running: 91.5%   ← phát hiện hoạt động "chạy"
```

Điều này cho phép tạo timeline, nhảy tới đoạn chứa nội dung cần thiết, hoặc đếm thời lượng xuất hiện.

### Ví Dụ: Tích Hợp SNS (Production Pattern)

Trong production, nên dùng SNS thay vì polling để tiết kiệm chi phí và phản hồi nhanh:

```python
def start_video_moderation_with_sns(bucket, key, sns_topic_arn, role_arn):
    """Khởi động kiểm duyệt video, nhận thông báo qua SNS khi xong."""
    response = rekognition.start_content_moderation(
        Video={"S3Object": {"Bucket": bucket, "Name": key}},
        NotificationChannel={
            "SNSTopicArn": sns_topic_arn,   # Topic SNS nhận thông báo
            "RoleArn": role_arn             # IAM role cho Rekognition publish lên SNS
        },
        MinConfidence=70
    )
    return response["JobId"]

# Lambda subscribe vào SNS topic này sẽ được kích hoạt khi job xong,
# rồi gọi get_content_moderation(JobId) để lấy kết quả.
```

---

## 📡 Phần 2: Streaming Video Analysis (Phân Tích Video Trực Tiếp)

Phân tích video **real-time** từ camera trực tiếp — phù hợp giám sát an ninh, phát hiện sự kiện tức thời.

### Kiến Trúc Streaming

```
Camera (IP/RTSP)
      │
      ▼
Kinesis Video Stream  ──►  Rekognition Stream Processor
   (luồng video)              (xử lý liên tục)
                                    │
                          ┌─────────┴──────────┐
                          ▼                     ▼
              Kinesis Data Stream        S3 (lưu evidence)
              (kết quả phân tích)
                          │
                          ▼
              Lambda / KCL Consumer  ──► cảnh báo, lưu DB
```

### Hai Loại Stream Processor

| Loại | Phân Tích | Đầu Ra |
|---|---|---|
| **Face Search** | So khớp khuôn mặt với Collection real-time | Kinesis Data Stream |
| **Connected Home / Label Detection** | Phát hiện người, thú cưng, gói hàng | Notification (SNS) + S3 |

### Tạo Stream Processor (Face Search)

```python
def create_stream_processor(name, kvs_arn, kds_arn, collection_id, role_arn):
    """Tạo Stream Processor tìm khuôn mặt real-time từ Kinesis Video Stream."""
    rekognition.create_stream_processor(
        Name=name,
        Input={
            "KinesisVideoStream": {"Arn": kvs_arn}   # Nguồn video
        },
        Output={
            "KinesisDataStream": {"Arn": kds_arn}    # Đích chứa kết quả
        },
        Settings={
            "FaceSearch": {
                "CollectionId": collection_id,        # Collection khuôn mặt để so khớp
                "FaceMatchThreshold": 90
            }
        },
        RoleArn=role_arn
    )

# Khởi động xử lý
rekognition.start_stream_processor(Name="security-cam-1")
# Dừng khi không cần (tránh tính phí liên tục)
# rekognition.stop_stream_processor(Name="security-cam-1")
```

---

## 🎞️ Segment Detection (Phát Hiện Phân Đoạn) — Cho Media & Entertainment

Hữu ích cho ngành truyền thông: tự động tìm các điểm chuyển cảnh và đoạn kỹ thuật trong video.

| Segment Type | Mô Tả | Use Case |
|---|---|---|
| **Technical Cues** (Tín Hiệu Kỹ Thuật) | Black frames (khung đen), color bars, end credits (cảnh cuối), studio logo | Tự động chèn quảng cáo, bỏ qua credits |
| **Shot Detection** (Phát Hiện Cảnh Quay) | Điểm camera cắt cảnh | Tạo thumbnail, biên tập tự động |

```python
def detect_segments(bucket, key, sns_topic_arn, role_arn):
    """Phát hiện điểm chuyển cảnh và technical cue trong video."""
    response = rekognition.start_segment_detection(
        Video={"S3Object": {"Bucket": bucket, "Name": key}},
        SegmentTypes=["TECHNICAL_CUE", "SHOT"],
        NotificationChannel={"SNSTopicArn": sns_topic_arn, "RoleArn": role_arn}
    )
    return response["JobId"]
```

---

## 📊 So Sánh Stored vs Streaming

| Tiêu Chí | Stored Video | Streaming Video |
|---|---|---|
| **Nguồn** | S3 | Kinesis Video Stream |
| **Độ trễ** | Phút (batch) | Real-time (giây) |
| **Mô hình gọi** | Async start/get + SNS | Stream Processor liên tục |
| **Tính phí** | Per phút video xử lý | Theo thời gian processor chạy |
| **Use case** | VOD, phân tích kho video | Camera an ninh, live monitoring |

---

## 💰 Lưu Ý Chi Phí Video

- **Stored Video**: tính phí theo **phút video xử lý** cho mỗi loại phân tích → chạy nhiều loại trên cùng video = cộng dồn chi phí
- **Streaming**: tính phí theo **thời gian Stream Processor hoạt động** → luôn `stop_stream_processor` khi không giám sát
- **Mẹo:** Cắt/lấy mẫu (sample) khung hình thay vì phân tích toàn bộ video nếu không cần độ chi tiết theo từng giây

---

## 📐 Kiến Trúc Thực Tế: Hệ Thống Kiểm Duyệt Video UGC

```
User upload video
      │
      ▼
S3  ──(S3 Event)──►  Lambda (Starter)
                         │
                         └──► start_content_moderation (+ SNS channel)
                                    │
                          Rekognition xử lý nền
                                    │
                                    ▼
                          SNS "job-complete"  ──►  Lambda (Collector)
                                                        │
                                                        ├─ get_content_moderation
                                                        ├─ Có vi phạm → chặn + A2I review
                                                        └─ DynamoDB (lưu kết quả + timeline)
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Tại sao Rekognition Video dùng async mà không sync như ảnh?**

> Video chứa nhiều khung hình, xử lý mất nhiều thời gian (giây → phút), không phù hợp với một HTTP request đồng bộ. Async cho phép khởi động job rồi nhận kết quả qua SNS hoặc polling.

**Q: Stored Video vs Streaming Video — khi nào dùng cái nào?**

> Stored cho video đã có sẵn trên S3 (VOD, kho video) — phân tích offline theo batch. Streaming cho video trực tiếp real-time (camera an ninh) qua Kinesis Video Stream — cần phản hồi tức thời.

**Q: Làm sao biết object xuất hiện ở đoạn nào trong video?**

> Mỗi kết quả video kèm **Timestamp** (mili-giây từ đầu video), cho phép xây dựng timeline và nhảy tới đúng đoạn chứa nội dung.

**Q: Streaming Face Search hoạt động thế nào?**

> Tạo Stream Processor gắn Kinesis Video Stream (input) với một Face Collection. Khi có khuôn mặt khớp ≥ threshold, kết quả được đẩy ra Kinesis Data Stream để consumer (Lambda/KCL) xử lý và cảnh báo real-time.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
