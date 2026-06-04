# Job, Queue & IAM Role — Cơ Chế Xử Lý MediaConvert

> Hiểu rõ vòng đời của Job và cơ chế Queue là nền tảng để vận hành MediaConvert hiệu quả trong môi trường production.

## 📚 Mục Lục

1. [Job — Tác Vụ Chuyển Mã](#1-job--tác-vụ-chuyển-mã)
2. [Queue — Hàng Đợi Xử Lý](#2-queue--hàng-đợi-xử-lý)
3. [IAM Role — Vai Trò Quyền Truy Cập](#3-iam-role--vai-trò-quyền-truy-cập)
4. [Tạo Job Qua AWS CLI](#4-tạo-job-qua-aws-cli)
5. [Monitoring & Alerting](#5-monitoring--alerting)
6. [Best Practices Vận Hành](#6-best-practices-vận-hành)

---

## 1. Job — Tác Vụ Chuyển Mã

### Cấu Trúc Job

**Job** là đơn vị xử lý cơ bản trong MediaConvert. Mỗi Job chứa đầy đủ thông tin về: nguồn đầu vào, cài đặt xử lý, và đầu ra mong muốn.

```
Job
├── Role ARN             — IAM Role MediaConvert dùng để truy cập S3/KMS
├── Queue ARN            — Hàng đợi sẽ xử lý Job này
├── Settings
│   ├── Inputs[]         — Danh sách file đầu vào (thường là 1, có thể ghép nhiều)
│   │   ├── FileInput    — S3 URI của video gốc
│   │   ├── AudioSelectors   — Chọn track audio từ input
│   │   ├── CaptionSelectors — Chọn track phụ đề từ input
│   │   └── VideoSelector    — Cấu hình deinterlace, color space
│   ├── OutputGroups[]   — Danh sách nhóm đầu ra (HLS, DASH, File Group...)
│   ├── TimecodeConfig   — Cách đọc timecode (mốc thời gian của video)
│   └── AdAvailOffset    — Độ lệch điểm chèn quảng cáo (giây)
└── UserMetadata         — Metadata tuỳ ý (key-value) để tracking
```

### Vòng Đời Job (Job Lifecycle)

```
Tạo Job (API/Console)
         │
         ▼
   ┌──────────┐
   │SUBMITTED │  ← Job vào hàng đợi, chưa có worker xử lý
   └──────────┘
         │
         ▼  (Có worker available trong Queue)
   ┌─────────────┐
   │ PROGRESSING │  ← MediaConvert đang xử lý Job
   └─────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌───────┐
│COMPLETE│ │ ERROR │
└────────┘ └───────┘
    ↑           ↑
Thành công   Thất bại
(output       (xem
 trên S3)     ErrorMessage)
```

**Lưu ý về thời gian chờ (Queue Wait Time):**
- Hàng đợi **On-demand** và **Spot**: có thể phải chờ nếu hết capacity
- Hàng đợi **Reserved**: luôn có capacity vì đã mua trước
- Thời gian xử lý (processing time) ≈ **2–4× độ dài video** (ví dụ: video 10 phút → xử lý khoảng 20–40 phút cho UHD)

### Input — Đầu Vào

**Định dạng input được hỗ trợ:**

| Loại | Định Dạng |
|------|-----------|
| **Video Container** | MP4, MOV, MXF, AVI, WMV, FLV, MKV, MPEG-2 TS |
| **Video Codec** | H.264, H.265, MPEG-2, ProRes, DNxHD, XAVC, RAW |
| **Audio** | AAC, MP3, WAV, AIFF, PCM, Dolby E, AC-3 |

**Input clipping — Cắt đoạn đầu vào:**
```json
"InputClippings": [
  {
    "StartTimecode": "00:01:00:00",
    "EndTimecode":   "00:05:30:00"
  }
]
```
Cho phép chỉ xử lý một đoạn của video gốc mà không cần cắt file trước.

**Input stitching — Ghép nhiều file:**
```json
"Inputs": [
  { "FileInput": "s3://bucket/intro.mp4" },
  { "FileInput": "s3://bucket/main-content.mp4" },
  { "FileInput": "s3://bucket/outro.mp4" }
]
```
MediaConvert ghép các file theo thứ tự trước khi xử lý — hữu ích để thêm intro/outro.

---

## 2. Queue — Hàng Đợi Xử Lý

### Ba Loại Queue

#### 2.1 On-demand Queue (Hàng Đợi Theo Nhu Cầu)

```
Đặc điểm:
  - Mặc định khi tạo Job không chỉ định Queue
  - AWS cấp phát worker tự động theo nhu cầu
  - Phù hợp: workload không đều, traffic đột biến
  - Giá: cao nhất trong 3 loại

Hạn chế:
  - Có thể phải chờ khi hệ thống bận (capacity throttling)
  - Không đảm bảo thời gian xử lý tối đa
```

#### 2.2 Reserved Queue — Hàng Đợi Đặt Trước (Reserved Transcoding Capacity)

```
Đặc điểm:
  - Mua trước số lượng RTS — Reserved Transcoding Slot (slot xử lý đặt trước)
  - 1 RTS = khả năng xử lý 1 Job cùng lúc
  - Cam kết 1 tháng hoặc 12 tháng (12 tháng rẻ hơn)
  - Giảm ~54% so với On-demand
  - Không bị ngắt (guaranteed capacity)

Phù hợp:
  - Workload đều đặn, dự đoán được
  - Ví dụ: mỗi ngày upload và xử lý 50–100 video
  - SLA (Service Level Agreement) nghiêm ngặt

Tính toán ROI (Return on Investment — Tỷ Lệ Hoàn Vốn):
  Nếu > 10–15 giờ HD processing/ngày → Reserved Queue tiết kiệm tiền
```

**Ví dụ tính toán:**
```
On-demand HD: $0.0075 / phút
Reserved HD (12 tháng): $0.0034 / phút (tiết kiệm 55%)

Video 30 phút, 5 outputs HD:
- On-demand:  30 × 5 × $0.0075 = $1.125 / job
- Reserved:   30 × 5 × $0.0034 = $0.510 / job (tiết kiệm $0.615)

Nếu 100 jobs/ngày:
- On-demand:  $112.5 / ngày = $3,375 / tháng
- Reserved:   $51.0 / ngày + chi phí slot ≈ $1,530–1,800 / tháng
→ Tiết kiệm ~$1,500–1,800/tháng
```

#### 2.3 Spot Queue — Hàng Đợi Spot (Spot Pricing)

```
Đặc điểm:
  - Dùng capacity dư của AWS
  - Rẻ nhất: giảm ~54% so với On-demand (tương đương Reserved)
  - CÓ THỂ BỊ NGẮT nếu AWS cần capacity đó

Khi nào dùng:
  - Transcoding không cần gấp (background batch processing)
  - Archive conversion (chuyển đổi kho video cũ)
  - Development / testing

KHÔNG dùng khi:
  - Cần output trong thời gian xác định (SLA)
  - Production pipeline người dùng chờ kết quả
```

### Quản Lý Queue

**Thiết lập Priority — Mức Ưu Tiên (0–50):**
```json
{
  "Priority": 25  // 0 = thấp nhất, 50 = cao nhất
}
```

Trong cùng một Queue, Job có Priority cao hơn được xử lý trước. Hữu ích khi có:
- Job "urgent" (preview nhanh) và Job "background" (full encode)

**Pausing Queue — Tạm Dừng Hàng Đợi:**
- Có thể pause Queue để dừng nhận Job mới (ví dụ: bảo trì hệ thống downstream)
- Job đang xử lý (`PROGRESSING`) không bị ảnh hưởng

---

## 3. IAM Role — Vai Trò Quyền Truy Cập

### Tại Sao Cần IAM Role?

MediaConvert hoạt động **trong tài khoản AWS của bạn**, cần quyền để:
1. Đọc video nguồn từ S3 Input
2. Ghi output vào S3 Output
3. (Tuỳ chọn) Ghi CloudWatch Logs
4. (Tuỳ chọn) Tạo/xoá temp objects
5. (Tuỳ chọn) Dùng KMS key để mã hoá output

### Tạo IAM Role Đúng Cách

**Bước 1: Trust Policy — Chính Sách Uỷ Quyền** (cho phép MediaConvert assume role):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "mediaconvert.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "123456789012"
        },
        "ArnLike": {
          "aws:SourceArn": "arn:aws:mediaconvert:ap-southeast-1:123456789012:*"
        }
      }
    }
  ]
}
```

> **Lưu ý bảo mật:** Thêm `Condition` để giới hạn chỉ MediaConvert trong account và region của bạn mới được dùng role này (chống Confused Deputy Attack — Tấn Công Đại Lý Bị Nhầm Lẫn).

**Bước 2: Permission Policy — Chính Sách Quyền Hạn:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3InputAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": [
        "arn:aws:s3:::my-input-bucket/*"
      ]
    },
    {
      "Sid": "S3OutputAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-output-bucket/*"
      ]
    },
    {
      "Sid": "S3BucketList",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-input-bucket",
        "arn:aws:s3:::my-output-bucket"
      ]
    }
  ]
}
```

**Thêm quyền KMS nếu dùng Server-Side Encryption (mã hoá phía máy chủ):**
```json
{
  "Sid": "KMSAccess",
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "arn:aws:kms:REGION:ACCOUNT:key/KEY_ID"
}
```

### Lỗi IAM Phổ Biến

| Lỗi | Nguyên Nhân | Giải Pháp |
|-----|-------------|-----------|
| `AccessDenied on s3:GetObject` | Role thiếu quyền đọc S3 Input | Thêm `s3:GetObject` permission |
| `AccessDenied on s3:PutObject` | Role thiếu quyền ghi S3 Output | Thêm `s3:PutObject` permission |
| `The role... is not assumable` | Trust Policy sai Service principal | Kiểm tra Trust Policy: `"Service": "mediaconvert.amazonaws.com"` |
| `KMS key not accessible` | Thiếu KMS decrypt permission | Thêm `kms:Decrypt` và `kms:GenerateDataKey` |

---

## 4. Tạo Job Qua AWS CLI

### Job Settings File (job-settings.json)

```json
{
  "Role": "arn:aws:iam::123456789012:role/MediaConvertRole",
  "Queue": "arn:aws:mediaconvert:ap-southeast-1:123456789012:queues/Default",
  "UserMetadata": {
    "contentId": "video-001",
    "environment": "production"
  },
  "Settings": {
    "TimecodeConfig": {
      "Source": "ZEROBASED"
    },
    "Inputs": [
      {
        "FileInput": "s3://my-input-bucket/raw-video.mp4",
        "AudioSelectors": {
          "Audio Selector 1": {
            "DefaultSelection": "DEFAULT"
          }
        },
        "VideoSelector": {
          "ColorSpace": "FOLLOW",
          "Rotate": "AUTO"
        }
      }
    ],
    "OutputGroups": [
      {
        "Name": "HLS Group",
        "OutputGroupSettings": {
          "Type": "HLS_GROUP_SETTINGS",
          "HlsGroupSettings": {
            "Destination": "s3://my-output-bucket/hls/video-001/",
            "SegmentLength": 6,
            "MinSegmentLength": 0
          }
        },
        "Outputs": [
          {
            "NameModifier": "_1080p",
            "VideoDescription": {
              "Width": 1920,
              "Height": 1080,
              "CodecSettings": {
                "Codec": "H_264",
                "H264Settings": {
                  "RateControlMode": "QVBR",
                  "MaxBitrate": 5000000,
                  "QvbrSettings": { "QvbrQualityLevel": 8 }
                }
              }
            },
            "AudioDescriptions": [
              {
                "AudioSourceName": "Audio Selector 1",
                "CodecSettings": {
                  "Codec": "AAC",
                  "AacSettings": { "Bitrate": 128000, "SampleRate": 48000 }
                }
              }
            ],
            "ContainerSettings": {
              "Container": "M3U8",
              "M3u8Settings": {}
            }
          }
        ]
      }
    ]
  }
}
```

### Lệnh CLI

```bash
# Lấy endpoint MediaConvert của region (bắt buộc, mỗi account có endpoint riêng)
ENDPOINT=$(aws mediaconvert describe-endpoints \
  --region ap-southeast-1 \
  --query "Endpoints[0].Url" \
  --output text)

# Tạo Job
aws mediaconvert create-job \
  --endpoint-url "$ENDPOINT" \
  --region ap-southeast-1 \
  --cli-input-json file://job-settings.json

# Kiểm tra trạng thái Job
aws mediaconvert get-job \
  --endpoint-url "$ENDPOINT" \
  --region ap-southeast-1 \
  --id "JOB_ID_HERE"

# Liệt kê Jobs gần đây
aws mediaconvert list-jobs \
  --endpoint-url "$ENDPOINT" \
  --region ap-southeast-1 \
  --status PROGRESSING
```

> **Quan trọng:** MediaConvert yêu cầu dùng **account-specific endpoint** (endpoint riêng của từng account). Không dùng regional endpoint chung như các service khác.

---

## 5. Monitoring & Alerting

### CloudWatch Metrics (Chỉ Số Theo Dõi)

| Metric | Ý Nghĩa | Cảnh Báo Khi |
|--------|---------|--------------|
| `JobsComplete` | Số Job hoàn thành | — |
| `JobsErrored` | Số Job lỗi | > 0 trong 5 phút |
| `JobsSubmitted` | Số Job được tạo | Tăng đột biến bất thường |
| `StandbyTime` | Thời gian chờ trong Queue | > 30 phút (tùy SLA) |
| `TranscodingTime` | Thời gian thực sự xử lý | Bất thường cao |

### EventBridge — Theo Dõi Sự Kiện Job

```json
{
  "source": ["aws.mediaconvert"],
  "detail-type": ["MediaConvert Job State Change"],
  "detail": {
    "status": ["ERROR", "COMPLETE", "CANCELED"]
  }
}
```

**Thiết lập pipeline thông báo:**
```
MediaConvert Job
      │ (Job ERROR/COMPLETE event)
      ▼
  EventBridge Rule
      │
      ├──▶ SNS Topic → Email/Slack notification
      ├──▶ Lambda → Auto-retry hoặc ghi log vào DynamoDB
      └──▶ SQS Queue → Downstream system xử lý tiếp
```

### Xem Log Job

Khi Job ERROR, xem nguyên nhân qua:
1. **AWS Console** → MediaConvert → Jobs → chọn Job → tab "Job summary"
2. **AWS CLI:**
```bash
aws mediaconvert get-job --id JOB_ID --endpoint-url ENDPOINT \
  --query "Job.ErrorMessage"
```
3. **CloudWatch Logs** (nếu đã enable): nhóm log `/aws/mediaconvert/ACCOUNT_ID`

---

## 6. Best Practices Vận Hành

### Thiết Kế Job

```
✅ NÊN làm:
- Đặt UserMetadata để tracking (contentId, userId, environment)
- Dùng QVBR (Quality-defined Variable Bitrate — Tốc Độ Bit Biến Đổi Theo Chất Lượng)
  thay vì CBR cho output HLS/DASH (tốt hơn về chất lượng và bandwidth)
- Một Job tạo tất cả output cùng lúc (HLS + DASH + Thumbnail) thay vì nhiều Job riêng
- Sử dụng Input Clipping để tránh encode đoạn không cần thiết

❌ KHÔNG nên làm:
- Tạo nhiều Job riêng lẻ cho cùng một video (vừa chậm, vừa tốn tiền)
- Hardcode IAM Role ARN trong code — dùng environment variable hoặc SSM Parameter
- Để Queue On-demand bị tràn khi có batch upload lớn — dùng SQS để điều phối
```

### Quản Lý Queue Hiệu Quả

```
Production workflow đề xuất:

[On-demand Queue]  ← Job ưu tiên cao (người dùng đang chờ preview)
       │
       ├── Priority 50: thumbnail generation (< 1 phút)
       ├── Priority 30: preview quality 360p (vài phút)
       └── Priority 10: full HD encode

[Reserved Queue]   ← Background encode sau khi preview OK
       └── Priority 0: full multi-bitrate production encode

[Spot Queue]       ← Batch processing không gấp
       └── Archive conversion, re-encode old content
```

### Xử Lý Lỗi Và Retry

```python
import boto3, time

def submit_job_with_retry(job_settings, max_retries=3):
    client = boto3.client("mediaconvert", endpoint_url=ENDPOINT)

    for attempt in range(max_retries):
        try:
            response = client.create_job(**job_settings)
            return response["Job"]["Id"]
        except client.exceptions.TooManyRequestsException:
            wait = 2 ** attempt  # Exponential backoff — Thời Gian Chờ Tăng Dần Theo Luỹ Thừa
            time.sleep(wait)
        except Exception as e:
            raise e  # Lỗi khác không retry

    raise Exception("Max retries exceeded")
```

---

**Phần Tiếp Theo:** [2-output-groups.md](./2-output-groups.md) — Cấu hình Output Groups: HLS, DASH, CMAF, File Group
