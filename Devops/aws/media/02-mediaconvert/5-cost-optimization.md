# Tối Ưu Chi Phí MediaConvert

> Chi phí MediaConvert có thể chiếm phần lớn ngân sách nếu không được tối ưu đúng cách. Phần này hướng dẫn cách tính toán, dự báo và giảm chi phí tối đa trong môi trường production.

## 📚 Mục Lục

1. [Mô Hình Định Giá MediaConvert](#1-mô-hình-định-giá-mediaconvert)
2. [So Sánh Ba Loại Queue](#2-so-sánh-ba-loại-queue)
3. [Tính Toán Chi Phí Thực Tế](#3-tính-toán-chi-phí-thực-tế)
4. [Reserved Queue — Hướng Dẫn Mua Và Vận Hành](#4-reserved-queue--hướng-dẫn-mua-và-vận-hành)
5. [Spot Pricing — Tận Dụng Capacity Dư](#5-spot-pricing--tận-dụng-capacity-dư)
6. [Chiến Lược Tối Ưu Job](#6-chiến-lược-tối-ưu-job)
7. [Monitoring Chi Phí](#7-monitoring-chi-phí)
8. [Tổng Hợp Best Practices](#8-tổng-hợp-best-practices)

---

## 1. Mô Hình Định Giá MediaConvert

### Nguyên Tắc Tính Phí

MediaConvert tính phí theo **phút xử lý của output** (không phải input), và phân loại theo **độ phân giải cao nhất** trong output:

```
Phí = Số phút xử lý × Đơn giá theo tier độ phân giải

Tier độ phân giải:
  SD  (Standard Definition)  — dưới 720p (< 1280×720)
  HD  (High Definition)      — 720p đến dưới 4K (1280×720 → 3839×2159)
  UHD (Ultra High Definition) — 4K trở lên (≥ 3840×2160)
```

> **Lưu ý quan trọng:** Nếu Job có 1 output 1080p và 1 output 360p, **tất cả phút xử lý trong Job đó tính theo giá HD** (vì output cao nhất là HD).

### Đơn Giá On-demand (Region us-east-1, 2026)

| Tier | Độ Phân Giải | Đơn Giá (USD/phút) |
|------|-------------|-------------------|
| **SD** | < 720p | $0.0075 |
| **HD** | 720p–1080p | $0.0150 |
| **UHD** | 4K+ | $0.0450 |

> Giá có thể thay đổi theo region và thời gian. Kiểm tra [trang giá chính thức](https://aws.amazon.com/mediaconvert/pricing/) trước khi tính toán.

### Ví Dụ Tính Phí Cơ Bản

```
Video 30 phút, Job tạo output:
  - 1080p (HD) → output 1 của Job
  - 720p  (HD) → output 2 của Job
  - 360p  (SD) → output 3 của Job

Phân loại: HD (vì có output ≥ 720p)
Phí = 30 phút × 3 outputs × $0.0150 = $1.35

Lưu ý: Số lượng output KHÔNG tăng thêm phí per-output
        Phí tính theo tổng phút, không nhân số output!

Thực ra: Phí = 30 phút × $0.0150 = $0.45 (chỉ tính theo độ dài video, không nhân output)
```

> **Đính chính:** MediaConvert tính phí theo **tổng số phút output video** (tổng cộng tất cả outputs). Mỗi output tính riêng nên:
>
> `Phí = (phút video × số outputs HD × $0.0150) + (phút video × số outputs SD × $0.0075)`

### Ví Dụ Tính Phí Đầy Đủ

```
Video 60 phút, outputs:
  1080p × 1 = 60 phút HD
  720p  × 1 = 60 phút HD
  480p  × 1 = 60 phút SD
  360p  × 1 = 60 phút SD

Phí = (60 × 2) × $0.0150 + (60 × 2) × $0.0075
    = 120 × $0.0150 + 120 × $0.0075
    = $1.80 + $0.90
    = $2.70 / job
```

---

## 2. So Sánh Ba Loại Queue

### Bảng So Sánh Toàn Diện

| Tiêu Chí | On-demand | Reserved | Spot |
|----------|-----------|----------|------|
| **Đơn giá** | $0.0150/phút HD | ~$0.0068/phút HD | ~$0.0068/phút HD |
| **Tiết kiệm** | — (baseline) | ~54% | ~54% |
| **Cam kết** | Không | 1 tháng hoặc 12 tháng | Không |
| **Đảm bảo capacity** | ❌ (có thể phải chờ) | ✅ (guaranteed) | ❌ (có thể bị ngắt) |
| **Bị ngắt giữa chừng** | ❌ | ❌ | ✅ (khi AWS cần capacity) |
| **Phù hợp** | Workload bất định | Workload đều đặn | Batch không gấp |
| **Thời gian setup** | Ngay lập tức | Cần tạo Reserved Queue | Cần tạo Spot Queue |

### Khi Nào Dùng Loại Queue Nào

```
Câu hỏi quyết định:

1. Bạn có thể dự đoán workload không?
   ├── Không (workload thất thường) → On-demand
   └── Có (đều đặn mỗi ngày) → tiếp tục...

2. Job có SLA — Service Level Agreement (cam kết thời gian) không?
   ├── Có (người dùng đang chờ) → Reserved
   └── Không (batch background) → Spot

3. Workload dự kiến bao nhiêu giờ/ngày?
   ├── < 4 giờ HD/ngày → On-demand (Reserved không đủ ROI)
   ├── 4–8 giờ HD/ngày → Cân nhắc Reserved (tính ROI)
   └── > 8 giờ HD/ngày → Reserved gần như chắc chắn tiết kiệm tiền
```

---

## 3. Tính Toán Chi Phí Thực Tế

### Công Thức Tổng Quát

```
Monthly Cost (On-demand) =
  ∑(video_duration_minutes × output_count_HD × $0.0150
    + video_duration_minutes × output_count_SD × $0.0075)
    across all jobs per month
```

### Ví Dụ: Nền Tảng E-learning Nhỏ

```
Thông số:
  - 50 video/ngày, mỗi video trung bình 20 phút
  - 4 outputs HD mỗi job (1080p, 720p, 480p, 360p)
    (480p và 360p là SD nhưng trong job có HD → tính theo HD rule? Không!)
    (Thực ra: 480p, 360p tính SD nếu resolution output là SD)

Tính lại:
  - 2 outputs HD: 1080p + 720p = 2 × 20 phút × $0.0150 = $0.60/job
  - 2 outputs SD: 480p + 360p = 2 × 20 phút × $0.0075 = $0.30/job
  - Tổng/job = $0.90

Monthly cost = 50 jobs/ngày × 30 ngày × $0.90 = $1,350/tháng

Với Reserved Queue:
  = $1,350 × (1 - 0.54) = $621/tháng + phí Reserved slot

Tiết kiệm = $1,350 - $621 = $729/tháng
```

### Ví Dụ: Platform OTT Trung Bình

```
Thông số:
  - 200 video/ngày, trung bình 45 phút/video
  - 5 outputs UHD: 4K, 1080p, 720p, 480p, 360p

UHD outputs (4K): 1 × 45 phút × $0.0450 = $2.025/job
HD outputs (1080p, 720p): 2 × 45 phút × $0.0150 = $1.35/job
SD outputs (480p, 360p): 2 × 45 phút × $0.0075 = $0.675/job
Tổng/job = $4.05

On-demand: 200 × 30 × $4.05 = $24,300/tháng
Reserved:  ~$24,300 × 0.46 = ~$11,178/tháng + phí slot
Tiết kiệm: ~$13,122/tháng (~$157,000/năm)
```

---

## 4. Reserved Queue — Hướng Dẫn Mua Và Vận Hành

### RTS — Reserved Transcoding Slot (Slot Xử Lý Đặt Trước)

```
1 RTS = khả năng xử lý 1 Job cùng một lúc

Ví dụ:
  Bạn mua 3 RTS:
  - Lúc này có 10 jobs đang chờ
  - 3 jobs chạy song song (tối đa)
  - 7 jobs xếp hàng chờ
  → Muốn xử lý song song nhiều hơn → mua thêm RTS
```

### Cam Kết Và Đơn Giá RTS

| Cam Kết | Đơn Giá RTS HD | Tiết Kiệm vs On-demand |
|---------|---------------|------------------------|
| **1 tháng** | ~$500–600/tháng/RTS | ~40% |
| **12 tháng** | ~$450–500/tháng/RTS | ~54% |

> **Tính ROI trước khi mua:** 1 RTS xử lý tối đa ~44,640 phút/tháng (30 ngày × 24h × 60 phút = 43,200 phút). Nếu thực tế chạy được >50% công suất → Reserved Queue tiết kiệm.

### Quy Trình Mua Reserved Queue

```
1. Đánh giá workload hiện tại:
   - Tổng phút HD processing / tháng
   - Thời điểm peak (giờ cao điểm)
   - SLA yêu cầu

2. Tính số RTS cần thiết:
   RTS = ceil(peak_concurrent_jobs)
   Ví dụ: peak 5 jobs song song → mua 5 RTS

3. Tạo Reserved Queue trong Console:
   MediaConvert → Queues → Create queue → Reserved
   → Chọn RTS count, commit duration (1 month / 12 months)
   → Confirm purchase

4. Gán Job vào Reserved Queue:
   Thêm vào Job settings:
   "Queue": "arn:aws:mediaconvert:REGION:ACCOUNT:queues/MyReservedQueue"

5. Monitor utilization:
   CloudWatch → MediaConvert → Queue metrics
   Target: 70–90% utilization cho Reserved Queue
```

### Chiến Lược Hybrid Queue

```
Production architecture đề xuất:

[Reserved Queue]  ← Jobs ưu tiên cao (user-facing)
   Số RTS: phủ 80% workload thông thường

[On-demand Queue] ← Overflow khi Reserved Queue đầy
   Tự động fallback khi Reserved overloaded

[Spot Queue]      ← Non-urgent batch processing
   Archive conversion, re-encode cũ, QA testing

Logic routing (Lambda/Step Functions):
  if job.priority == "HIGH" or job.sla < 30min:
      → Reserved Queue
  elif job.priority == "BATCH" and job.deadline > 24h:
      → Spot Queue
  else:
      → On-demand Queue
```

---

## 5. Spot Pricing — Tận Dụng Capacity Dư

### Cách Spot Hoạt Động

```
AWS có capacity (máy tính) nhàn rỗi
         │
MediaConvert Spot Queue dùng capacity đó
         │
Giá rẻ hơn ~54% so với On-demand
         │
Khi AWS cần capacity cho workload khác
         │
Job đang chạy CÓ THỂ BỊ NGẮT (interrupted)
         │
Job bị ngắt → trạng thái CANCELED → bạn phải submit lại
```

### Xử Lý Job Spot Bị Ngắt

```python
import boto3
import time

def submit_with_spot_retry(job_settings, max_retries=5):
    """Submit job vào Spot Queue, retry nếu bị ngắt."""
    spot_queue_arn = "arn:aws:mediaconvert:...:queues/MySpotQueue"
    ondemand_queue_arn = "arn:aws:mediaconvert:...:queues/Default"

    client = boto3.client("mediaconvert", endpoint_url=ENDPOINT)

    for attempt in range(max_retries):
        queue = spot_queue_arn if attempt < 3 else ondemand_queue_arn
        job_settings["Queue"] = queue

        job_id = client.create_job(**job_settings)["Job"]["Id"]

        # Chờ job hoàn thành (với timeout)
        result = wait_for_job(client, job_id, timeout_minutes=120)

        if result["status"] == "COMPLETE":
            return job_id
        elif result["status"] == "CANCELED":
            # Job bị ngắt bởi Spot → retry
            wait_time = 2 ** attempt * 60  # Exponential backoff theo phút
            time.sleep(wait_time)
            continue
        else:
            raise Exception(f"Job failed: {result['errorMessage']}")

    raise Exception("Max spot retries exceeded")
```

### Khi Nào Dùng Spot Queue

```
Phù hợp với Spot:
  ✅ Archive conversion (chuyển đổi kho video cũ không ai đang xem)
  ✅ Re-encode để chuyển từ H.264 sang H.265 (tiết kiệm storage)
  ✅ Tạo multiple renditions cho nội dung cũ ít xem
  ✅ CI/CD testing và QA (chấp nhận job thỉnh thoảng bị ngắt)
  ✅ Processing nội dung upload vào đêm/cuối tuần (không SLA)

KHÔNG phù hợp:
  ❌ User upload video và đang chờ xem ngay
  ❌ Live event VOD clips (thời gian xử lý quan trọng)
  ❌ Pipeline có SLA cam kết hoàn thành trong X phút
```

---

## 6. Chiến Lược Tối Ưu Job

### 6.1 Một Job Cho Tất Cả Outputs (Job Consolidation)

```
❌ Sai: Nhiều Job riêng lẻ
  Job 1: Video gốc → 1080p HLS
  Job 2: Video gốc → 720p HLS   (đọc lại cùng input từ S3)
  Job 3: Video gốc → 360p HLS   (đọc lại cùng input từ S3)
  Chi phí: 3 × $0.45 = $1.35 + 3× S3 egress cost

✅ Đúng: 1 Job, nhiều Output Groups
  Job 1: Video gốc → {1080p HLS, 720p HLS, 360p HLS, thumbnail}
  Chi phí: $0.45 (3 phút HD + SD) + 1× S3 egress cost

Lý do: MediaConvert đọc input 1 lần duy nhất, encode tất cả outputs trong 1 pass.
```

### 6.2 Dùng QVBR Thay CBR/VBR

```
Cùng chất lượng hình ảnh, file size so sánh:
  CBR (5 Mbps fixed):  → 30 phút video = 1.125 GB
  VBR (target 4 Mbps): → 30 phút video ≈ 900 MB
  QVBR (level 8):      → 30 phút video ≈ 650–750 MB (nhỏ hơn 30–40%)

Tiết kiệm storage S3: 30–40%
Tiết kiệm bandwidth CloudFront: 30–40%
→ Giảm được cả chi phí S3 và CDN egress, không chỉ MediaConvert
```

### 6.3 Dùng H.265 Khi Phù Hợp

```
Scenario: 1000 video × 30 phút × 1080p

Với H.264 QVBR-8:
  Bitrate trung bình ≈ 4 Mbps → 30 phút × 4 Mbps = 900 MB/video
  Storage: 1000 × 900 MB = 900 GB/tháng

Với H.265 QVBR-8:
  Bitrate trung bình ≈ 2.2 Mbps → 30 phút × 2.2 Mbps ≈ 495 MB/video
  Storage: 1000 × 495 MB ≈ 495 GB/tháng

Tiết kiệm storage: ~405 GB/tháng × $0.023/GB = ~$9.3/tháng (nhỏ)
Tiết kiệm CDN egress: Nếu mỗi video xem 100 lần:
  100,000 × (900 - 495) MB × $0.009/GB = ~$365/tháng (đáng kể)

Nhược điểm H.265: MediaConvert encode chậm hơn ~2× → tốn thêm processing cost
→ Tính toán cụ thể: nếu CDN egress savings > extra encoding cost → dùng H.265
```

### 6.4 Input Clipping — Tránh Encode Nội Dung Không Cần

```json
"Inputs": [{
  "InputClippings": [{
    "StartTimecode": "00:05:00:00",
    "EndTimecode": "00:25:00:00"
  }]
}]
```

Chỉ encode đoạn từ phút 5 đến phút 25 (20 phút) thay vì video gốc 60 phút.
→ Tiết kiệm 66% chi phí MediaConvert cho use case này.

### 6.5 Thumbnail Chỉ Với Frame Capture (Không Encode Video Riêng)

```
❌ Sai: Tạo Job riêng chỉ để lấy thumbnail (encode 1 frame nhưng tính phí cả video)

✅ Đúng: Thêm Frame Capture output vào Job encode chính
  Job: encode HLS + thêm 1 output Frame Capture
  → Thumbnail được tạo miễn phí trong cùng Job
  (Frame Capture không tính phí thêm vì chỉ là 1 output thêm vào Job đang chạy)
```

### 6.6 Tránh Unnecessary Re-encode

```
Chiến lược "Encode Once, Serve Many":

Lần đầu upload:
  - Encode đầy đủ bitrate ladder (2160p, 1080p, 720p, 480p, 360p)
  - Lưu trữ tất cả outputs trong S3
  - Chi phí cao hơn nhưng làm 1 lần

Sau này:
  - Không encode lại khi thêm định dạng mới (nếu đã có master quality)
  - Dùng lại outputs đã có
  - Chỉ encode lại khi thực sự cần (thay đổi watermark, sửa phụ đề v.v.)

Ngoại lệ encode lại hợp lý:
  - Chuyển từ H.264 sang H.265 (tiết kiệm bandwidth dài hạn)
  - Thêm DRM encryption
  - Thay đổi output format (ví dụ: thêm CMAF)
```

---

## 7. Monitoring Chi Phí

### AWS Cost Explorer

```
Lọc chi phí MediaConvert:
  Service: AWS Elemental MediaConvert
  Dimension: By Usage Type (Minutes of video processed — SD/HD/UHD)

Tag tracking (đặt tag cho mọi Job):
  "UserMetadata": {
    "project": "elearning-platform",
    "environment": "production",
    "content-type": "course-video"
  }
→ Dùng Cost Allocation Tags để phân tích chi phí theo project
```

### CloudWatch Dashboard Chi Phí

**Metrics quan trọng để theo dõi:**

| Metric | Mục Tiêu | Cảnh Báo |
|--------|---------|---------|
| `JobsComplete` | Tăng đều | Đột ngột giảm → pipeline bị vỡ |
| `JobsErrored` | = 0 | > 0 → có lỗi cần xử lý |
| `StandbyTime` | < 5 phút | > 30 phút → Reserved Queue thiếu RTS |
| `TranscodingTime` | Ổn định | Tăng đột biến → Job phức tạp bất thường |
| Reserved Queue Utilization | 70–90% | < 50% → mua thừa RTS; > 95% → cần thêm RTS |

### Budget Alert — Cảnh Báo Vượt Ngân Sách

```
Thiết lập AWS Budget:
  Service: AWS Elemental MediaConvert
  Amount: $X/tháng (ngân sách MediaConvert)
  Alert at: 80% (cảnh báo sớm), 100% (cảnh báo vượt)
  Notification: Email + SNS → Slack
```

---

## 8. Tổng Hợp Best Practices

### Kiểm Tra Trước Khi Vận Hành

```
Checklist tối ưu chi phí MediaConvert:

Job Design:
  ☐ Gom tất cả outputs vào 1 Job (không tách nhiều Job cho 1 video)
  ☐ Dùng QVBR thay CBR/VBR
  ☐ Thêm Frame Capture thumbnail vào Job chính (không Job riêng)
  ☐ Dùng Input Clipping nếu chỉ cần 1 đoạn video

Codec Strategy:
  ☐ H.264 cho universal compatibility
  ☐ H.265 nếu CDN bandwidth savings > extra encoding cost
  ☐ Tránh dùng UHD outputs nếu content không thực sự 4K

Queue Strategy:
  ☐ Reserved Queue cho > 8 giờ HD/ngày
  ☐ Spot Queue cho batch/archive không gấp
  ☐ On-demand Queue cho overflow và workload bất định

Monitoring:
  ☐ Cost Allocation Tags trên mọi Job
  ☐ Budget Alert thiết lập sẵn
  ☐ CloudWatch Dashboard theo dõi Queue utilization
  ☐ Monthly cost review trong Cost Explorer
```

### Công Thức Ra Quyết Định Queue

```
Tính toán Reserved Queue ROI:

monthly_minutes_HD = video_per_month × avg_duration_min × outputs_HD
monthly_cost_ondemand = monthly_minutes_HD × $0.0150

RTS_needed = peak_concurrent_jobs
monthly_cost_reserved = RTS_needed × $500 (1 month commit)
                      + monthly_minutes_HD × $0.0068

savings = monthly_cost_ondemand - monthly_cost_reserved
payback_period = monthly_cost_reserved / (monthly_cost_ondemand / 12)

if savings > 0 và payback_period < 6 tháng:
    → Mua Reserved Queue
else:
    → Tiếp tục On-demand
```

### Tóm Tắt Ưu Tiên Tối Ưu

```
Mức độ ưu tiên (Impact × Ease):

1. [HIGH] Gom outputs vào 1 Job         → Tiết kiệm ngay, không thay đổi kiến trúc
2. [HIGH] Dùng QVBR thay CBR             → Giảm file size 30–40%, ít effort
3. [MEDIUM] Reserved Queue               → Tiết kiệm 54%, cần commit tài chính
4. [MEDIUM] Spot cho batch               → Tiết kiệm 54%, cần xử lý retry logic
5. [MEDIUM] H.265 cho nội dung premium  → Tiết kiệm CDN, tăng encoding cost
6. [LOW] Input Clipping                  → Chỉ áp dụng khi có thể bỏ đoạn đầu/cuối
7. [LOW] Tắt output SD không cần thiết  → Tiết kiệm nhỏ, cần đánh giá UX impact
```

---

## 💡 Ví Dụ Thực Tế: Startup E-learning Tối Ưu Chi Phí

```
Tình huống ban đầu:
  - 100 video/ngày, 20 phút/video
  - 3 outputs: 1080p, 720p, 360p
  - Dùng CBR, On-demand Queue
  - Chi phí: ~$2,700/tháng

Sau tối ưu:
  Bước 1: Gom outputs → 1 Job (đã làm đúng, không thay đổi)
  Bước 2: Đổi sang QVBR level 8 → file nhỏ hơn 35%
  Bước 3: Mua 2 Reserved RTS (12 tháng) → giảm 54% processing cost
  Bước 4: Off-hours batch → Spot Queue → thêm 15% tiết kiệm

Kết quả:
  Chi phí MediaConvert: $2,700 → $950/tháng (tiết kiệm 65%)
  Chi phí S3: giảm 35% (nhờ QVBR)
  Chi phí CloudFront: giảm 35% (nhờ QVBR)
  Tổng tiết kiệm: ~$2,000+/tháng
```

---

**Phần Trước:** [4-audio-captions.md](./4-audio-captions.md) — Audio đa track và phụ đề
**Quay Lại Tổng Quan:** [README.md](./README.md) — MediaConvert Overview
**Module Tiếp Theo:** [../03-medialive/README.md](../03-medialive/README.md) — Live video encoding với MediaLive
