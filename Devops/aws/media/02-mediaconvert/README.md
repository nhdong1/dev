# AWS Elemental MediaConvert — Chuyển Mã Video Theo Tệp (VOD Transcoding)

> AWS Elemental MediaConvert là dịch vụ chuyển mã video dựa trên đám mây, được thiết kế cho **VOD — Video on Demand — Video Theo Yêu Cầu**. Dịch vụ cho phép chuyển đổi file video từ định dạng này sang định dạng khác với quy mô lớn, không cần quản lý hạ tầng.

## 📚 Mục Lục (Table of Contents)

1. [MediaConvert Là Gì?](#1-mediaconvert-là-gì)
2. [Kiến Trúc Tổng Quan](#2-kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#3-các-khái-niệm-cốt-lõi)
4. [Pipeline Xử Lý VOD](#4-pipeline-xử-lý-vod)
5. [Nội Dung Chi Tiết](#5-nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#6-câu-hỏi-phỏng-vấn-thường-gặp)

---

## 1. MediaConvert Là Gì?

**AWS Elemental MediaConvert** là dịch vụ **file-based transcoding** (chuyển mã theo tệp) được quản lý hoàn toàn bởi AWS. Nó thay thế cho các hệ thống transcoding truyền thống phải tự dựng server (ví dụ: FFmpeg cluster, Zencoder, Encoding.com).

### Vị Trí Trong Pipeline Media

```
                    ┌─────────────────── VOD PIPELINE ──────────────────┐
                    │                                                    │
[Source Video]  →  [S3 Input]  →  [MediaConvert]  →  [S3 Output]  →  [CloudFront]  →  [Viewer]
  .mov, .mxf        Lưu trữ      Chuyển mã + xử lý   HLS/DASH/MP4    CDN phân phối   Trình duyệt
  Camera/NLE         gốc          (đây là bước này)    output files    toàn cầu        / ứng dụng
```

### So Sánh Với Các Giải Pháp Khác

| Tiêu Chí | MediaConvert | FFmpeg tự dựng | Elastic Transcoder (cũ) |
|----------|-------------|----------------|------------------------|
| **Quản lý server** | Không cần | Phải tự quản lý | Không cần |
| **Quy mô (Scale)** | Tự động | Tự cấu hình | Giới hạn |
| **Codec hỗ trợ** | H.264, H.265, AV1, VP9... | Hầu hết mọi codec | H.264, VP8 (ít hơn) |
| **DRM tích hợp** | Có (SPEKE) | Cần tự tích hợp | Không |
| **Captions (phụ đề)** | Đầy đủ (SRT, TTML, WebVTT...) | Cơ bản | Hạn chế |
| **Định giá** | Theo phút xử lý | Chi phí EC2 cố định | Theo phút (đắt hơn) |
| **Khuyến nghị** | ✅ Dùng cho production | Dev/test nhỏ | Không nên dùng mới |

> **Elastic Transcoder** đã bị deprecated (ngừng phát triển tính năng mới). AWS khuyến nghị chuyển sang MediaConvert.

---

## 2. Kiến Trúc Tổng Quan

### Các Thành Phần Chính

```
┌────────────────────────────────────────────────────────────────────┐
│                        AWS ELEMENTAL MEDIACONVERT                  │
│                                                                    │
│   ┌──────────┐    ┌──────────┐    ┌───────────────────────────┐   │
│   │  Queue   │    │   Job    │    │       Output Groups        │   │
│   │(Hàng đợi)│───▶│ (Tác vụ)│───▶│  HLS │ DASH │ CMAF │ MP4  │   │
│   └──────────┘    └──────────┘    └───────────────────────────┘   │
│                        │                                           │
│                   ┌────┴────┐                                      │
│                   │Settings │                                      │
│                   │- Input  │                                      │
│                   │- Video  │                                      │
│                   │- Audio  │                                      │
│                   │-Caption │                                      │
│                   └─────────┘                                      │
└────────────────────────────────────────────────────────────────────┘
           │                                      │
     S3 Input Bucket                      S3 Output Bucket
  (video gốc .mov/.mp4)                (HLS segments, DASH, MP4)
```

### Luồng Xử Lý (Processing Flow)

```
1. Tải video gốc lên S3 Input Bucket
2. Gọi API MediaConvert (hoặc Console) tạo Job với settings
3. Job được đưa vào Queue (hàng đợi)
4. MediaConvert lấy video từ S3, xử lý theo settings
5. Ghi kết quả (HLS/DASH/MP4...) vào S3 Output Bucket
6. Gửi sự kiện (SNS/CloudWatch Events) khi Job hoàn thành
7. CloudFront phân phối nội dung từ S3 Output đến viewer
```

---

## 3. Các Khái Niệm Cốt Lõi

### 3.1 Job (Tác Vụ Chuyển Mã)

**Job** là đơn vị công việc cơ bản trong MediaConvert. Mỗi Job bao gồm:

```json
{
  "Role": "arn:aws:iam::ACCOUNT:role/MediaConvertRole",
  "Queue": "arn:aws:mediaconvert:REGION:ACCOUNT:queues/Default",
  "Settings": {
    "Inputs": [...],       // Đầu vào: S3 URI + cắt ghép + audio/caption selectors
    "OutputGroups": [...], // Đầu ra: HLS, DASH, CMAF, File Group, MS Smooth
    "TimecodeConfig": {},  // Cấu hình timecode (mốc thời gian)
    "AdAvailOffset": 0     // Độ lệch điểm chèn quảng cáo
  }
}
```

**Trạng thái Job (Job States):**

| Trạng Thái | Ý Nghĩa |
|------------|---------|
| `SUBMITTED` | Job vừa được tạo, chờ trong Queue |
| `PROGRESSING` | Job đang được xử lý |
| `COMPLETE` | Job hoàn thành thành công |
| `ERROR` | Job thất bại, xem Error Message để debug |
| `CANCELED` | Job bị huỷ thủ công |

### 3.2 Queue (Hàng Đợi Xử Lý)

**Queue** kiểm soát cách Job được ưu tiên và định giá:

| Loại Queue | Mô Tả | Định Giá |
|-----------|-------|---------|
| **On-demand** | Hàng đợi mặc định, xử lý theo nhu cầu | Trả theo phút xử lý |
| **Reserved** | Mua trước capacity cố định (1 tháng hoặc 1 năm) | Cam kết trả trước, rẻ hơn ~54% |
| **Spot** | Dùng capacity dư của AWS, có thể bị ngắt | Rẻ nhất (~54% off on-demand), nhưng không đảm bảo |

### 3.3 IAM Role (Vai Trò Quyền Truy Cập)

MediaConvert cần **IAM Role — Identity and Access Management Role — Vai Trò Quản Lý Danh Tính và Quyền Truy Cập** để:
- Đọc video từ S3 Input Bucket
- Ghi kết quả vào S3 Output Bucket
- Ghi log vào CloudWatch
- (Tuỳ chọn) Mã hoá với KMS — Key Management Service

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::input-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::output-bucket/*"
    }
  ]
}
```

**Trust Policy** — Chính Sách Uỷ Quyền (cho phép MediaConvert assume role):
```json
{
  "Principal": { "Service": "mediaconvert.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

### 3.4 Output Group (Nhóm Đầu Ra)

**Output Group** xác định định dạng và cấu trúc file đầu ra. Một Job có thể có nhiều Output Group:

```
Job
├── Output Group 1: HLS Group → S3://output/hls/
│   ├── Output: 1080p (5 Mbps)
│   ├── Output: 720p  (3 Mbps)
│   └── Output: 360p  (1 Mbps)
├── Output Group 2: DASH ISO Group → S3://output/dash/
│   ├── Output: 1080p
│   └── Output: 720p
└── Output Group 3: File Group → S3://output/mp4/
    └── Output: 1080p MP4 (download)
```

---

## 4. Pipeline Xử Lý VOD

### Use Case 1: Platform Học Trực Tuyến (E-learning)

```
Giảng viên upload video (.mp4 HD)
         │
         ▼
    S3 Input Bucket
         │
         ▼
    MediaConvert Job
    - Output Group HLS:
        720p @ 3 Mbps
        480p @ 1.5 Mbps
        360p @ 800 Kbps
    - Output Group File Group:
        720p MP4 (cho phép tải xuống)
         │
         ▼
    S3 Output Bucket
         │
         ▼
    CloudFront Distribution
         │
         ▼
    Học viên xem trên web/mobile (ABR tự động chọn bitrate phù hợp)
```

### Use Case 2: Nền Tảng OTT (Netflix-style)

```
Content Ingest (nhập nội dung)
   Raw master video (.mxf / .mov ProRes)
         │
         ▼
    MediaConvert Job (phức tạp hơn)
    - Output Group CMAF:          ← Dùng chung segment cho HLS + DASH
        2160p UHD @ 15 Mbps
        1080p FHD @ 8 Mbps
        720p HD   @ 4 Mbps
        540p      @ 2 Mbps
        360p SD   @ 1 Mbps
    - Audio: AAC 2.0 + Dolby Atmos 5.1
    - Captions: SRT English + Vietnamese
    - DRM: Widevine + FairPlay (qua SPEKE)
         │
         ▼
    S3 Output → MediaPackage → CloudFront
```

### Use Case 3: Chuyển Đổi Archive (File Conversion)

```
Legacy content (.wmv, .flv, .avi)
         │
         ▼
    MediaConvert Job (File Group output)
    - Input: any legacy format
    - Output: MP4 H.264 (archive standard)
         │
         ▼
    S3 Long-term Storage (Glacier)
```

---

## 5. Nội Dung Chi Tiết

```
02-mediaconvert/
├── README.md                    ← [BẠN ĐANG Ở ĐÂY] Tổng quan
├── 1-jobs-queues.md             Job lifecycle, Queue types, IAM Role, monitoring
├── 2-output-groups.md           HLS, DASH, CMAF, File Group, MS Smooth — cấu hình chi tiết
├── 3-video-codec-settings.md    H.264, H.265, AV1: bitrate mode, resolution, profile, level
├── 4-audio-captions.md          Multi-track audio, audio normalization, SRT/TTML/WebVTT
└── 5-cost-optimization.md       Per-minute pricing, Reserved vs Spot, tối ưu chi phí thực tế
```

### Bản Đồ Học Tập Đề Xuất

```
Bắt đầu tại đây (README)
         │
         ▼
1-jobs-queues.md        ← Hiểu cơ chế Job + Queue trước
         │
         ▼
2-output-groups.md      ← Biết tạo output HLS/DASH/MP4
         │
         ▼
3-video-codec-settings.md  ← Tinh chỉnh chất lượng video
         │
         ▼
4-audio-captions.md     ← Xử lý âm thanh & phụ đề
         │
         ▼
5-cost-optimization.md  ← Tối ưu chi phí production
```

---

## 6. Câu Hỏi Phỏng Vấn Thường Gặp

### Câu hỏi cơ bản

**Q: MediaConvert khác gì Elastic Transcoder?**

> MediaConvert là thế hệ mới, hỗ trợ nhiều codec hơn (H.265, AV1), nhiều output format hơn (CMAF, DASH), có Queue Reserved/Spot để tối ưu chi phí, tích hợp DRM qua SPEKE và có Image Inserter (watermark). Elastic Transcoder đã ngừng phát triển tính năng mới.

**Q: MediaConvert tính phí như thế nào?**

> Theo **phút xử lý** (per-minute processed), tuỳ theo độ phân giải đầu ra:
> - **SD** (dưới 720p): thấp nhất
> - **HD** (720p–1080p): trung bình
> - **UHD** (trên 1080p): cao nhất
>
> Ví dụ: video 60 phút, output 1080p HD → tính 60 phút HD processing.

**Q: Một Job có thể có bao nhiêu Output Group?**

> Không giới hạn cứng (soft limit có thể request tăng). Thực tế một Job có thể có đồng thời HLS + DASH + CMAF + File Group + Thumbnail, đây là cách phổ biến để tạo multi-format output từ một lần transcoding.

### Câu hỏi nâng cao

**Q: Khi nào dùng Reserved Queue thay vì On-demand?**

> Khi có **workload dự đoán được và đều đặn** (ví dụ: mỗi ngày luôn cần transcoding ít nhất 4–8 giờ), Reserved Queue tiết kiệm ~54% chi phí. Cam kết trả trước theo tháng hoặc năm. Nếu workload không đều, On-demand linh hoạt hơn.

**Q: CMAF khác HLS và DASH như thế nào?**

> **CMAF — Common Media Application Format** dùng chung một tập file segment (`.m4s`) cho cả HLS và DASH. Thay vì tạo 2 bộ segment riêng (tốn gấp đôi storage và bandwidth origin), CMAF chỉ cần 1 bộ segment + 2 file manifest (`.m3u8` cho HLS, `.mpd` cho DASH). Tiết kiệm ~50% storage và origin egress cost.

**Q: Làm thế nào xử lý lỗi Job (Error Handling)?**

> 1. Subscribe **SNS — Simple Notification Service** hoặc **EventBridge** để nhận event khi Job ERROR
> 2. Xem `ErrorMessage` trong Job response để biết nguyên nhân
> 3. Các lỗi phổ biến: sai S3 URI, IAM Role thiếu permission, input codec không hỗ trợ, settings không hợp lệ
> 4. Dùng **CloudWatch Metrics** theo dõi `JobsErrored`, `JobsComplete` để cảnh báo tự động

---

## 🔗 Tài Liệu Tham Khảo

- [MediaConvert User Guide](https://docs.aws.amazon.com/mediaconvert/latest/ug/)
- [MediaConvert API Reference](https://docs.aws.amazon.com/mediaconvert/latest/apireference/)
- [Supported Output Codecs and Containers](https://docs.aws.amazon.com/mediaconvert/latest/ug/reference-codecs-containers.html)
- [MediaConvert Pricing](https://aws.amazon.com/mediaconvert/pricing/)

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phần Trước:** [01-fundamentals/](../01-fundamentals/README.md) — Nền tảng codec & protocols
**Phần Tiếp Theo:** [03-medialive/](../03-medialive/README.md) — Live video encoding
