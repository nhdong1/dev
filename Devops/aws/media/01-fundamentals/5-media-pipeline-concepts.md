# Media Pipeline Concepts — Kiến Trúc Pipeline Media End-to-End

> Hiểu pipeline end-to-end là kỹ năng phân biệt engineer "biết dịch vụ" và engineer "hiểu hệ thống". Mọi câu hỏi phỏng vấn về thiết kế hệ thống media đều xoay quanh việc kết nối các thành phần trong pipeline này.

## 📚 Mục Lục

1. [4 Giai Đoạn Pipeline Media](#1-4-giai-đoạn-pipeline-media)
2. [VOD Pipeline — Video On Demand](#2-vod-pipeline--video-on-demand)
3. [Live Streaming Pipeline — Phát Trực Tiếp](#3-live-streaming-pipeline--phát-trực-tiếp)
4. [Interactive Live Pipeline — Phát Tương Tác](#4-interactive-live-pipeline--phát-tương-tác)
5. [Live with Ads Pipeline — Phát Có Quảng Cáo](#5-live-with-ads-pipeline--phát-có-quảng-cáo)
6. [IoT Video Analytics Pipeline](#6-iot-video-analytics-pipeline)
7. [Trade-offs Kiến Trúc](#7-trade-offs-kiến-trúc)
8. [Giám Sát và Vận Hành](#8-giám-sát-và-vận-hành)

---

## 1. 4 Giai Đoạn Pipeline Media

### Tổng Quan

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   INGEST    │───▶│   ENCODE    │───▶│   PACKAGE   │───▶│   DELIVER   │
│  (Nhập)     │    │  (Mã hoá)   │    │  (Đóng gói) │    │ (Phân phối) │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

### Giai Đoạn 1: INGEST — Nhập Luồng / Tệp

**Mục tiêu:** Đưa nội dung thô vào hệ thống xử lý.

```
VOD Ingest:
  Camera/Editor → Upload file (MP4, MOV, MXF) → S3 bucket (source)
  Trigger: S3 Event Notification → Lambda → MediaConvert job

Live Ingest:
  Camera → Encoder (OBS, hardware) → RTMPS/SRT → MediaLive input endpoint
  MediaLive cung cấp: rtmps://a1b2c3.medialive.us-east-1.amazonaws.com/app/

IoT Ingest:
  Camera/Sensor → KVS Producer SDK → Kinesis Video Stream
```

**Yếu tố kỹ thuật cần quan tâm:**
- **Redundancy** — Dự Phòng: Live cần 2 nguồn ingest (Primary + Backup)
- **Bitrate ổn định**: Encoder nên dùng CBR hoặc capped VBR
- **Network reliability**: SRT chịu lỗi tốt hơn RTMP

### Giai Đoạn 2: ENCODE — Mã Hoá / Chuyển Mã

**Mục tiêu:** Chuyển đổi nội dung thô sang nhiều định dạng phù hợp phân phối.

```
VOD Encode (Transcoding — Chuyển mã):
  Input: 1 file chất lượng cao (ProRes, H.264 master)
  Output: nhiều rendition (H.264 + H.265, nhiều resolution/bitrate)
  Tool: AWS MediaConvert
  Thời gian: phút → giờ (tùy source và output)

Live Encode (Encoding — Mã hoá thời gian thực):
  Input: live stream liên tục
  Output: nhiều rendition real-time
  Tool: AWS MediaLive
  Latency: 1-3 giây
  Đặc điểm: không thể tua lại, không có buffer trước
```

**VOD vs Live Encoding:**

| Tiêu Chí | VOD (MediaConvert) | Live (MediaLive) |
|----------|-------------------|-----------------|
| **Thời gian** | Offline (không real-time) | Real-time bắt buộc |
| **Chất lượng** | Có thể dùng 2-pass | 1-pass bắt buộc |
| **Bitrate** | QVBR (tối ưu hơn) | CBR (predictable) |
| **Error recovery** | Không giới hạn thử lại | Không thể retry frame đã qua |
| **Codec** | H.264, H.265, AV1, VP9 | H.264, H.265 |
| **Chi phí** | Per-minute (theo phút) | Per-hour (theo giờ) |

### Giai Đoạn 3: PACKAGE — Đóng Gói

**Mục tiêu:** Tổ chức segments thành định dạng streaming chuẩn (HLS, DASH, CMAF) với DRM nếu cần.

```
Packaging output:
  ├── HLS manifest (.m3u8) + segments (.ts hoặc .fmp4)
  ├── DASH manifest (.mpd) + segments (.m4s)
  └── CMAF segments (.cmf4) — dùng chung cho HLS và DASH

DRM encryption (tuỳ chọn):
  ├── FairPlay (cho HLS → iOS)
  ├── Widevine (cho DASH → Android)
  └── PlayReady (cho DASH → Windows/Xbox)

Tools:
  VOD: MediaConvert (built-in) hoặc MediaPackage (v2)
  Live: MediaPackage (just-in-time packaging)
```

### Giai Đoạn 4: DELIVER — Phân Phối

**Mục tiêu:** Đưa nội dung từ origin (nguồn gốc) đến viewer nhanh nhất có thể, gần nhất có thể.

```
CDN — Content Delivery Network — Mạng Phân Phối Nội Dung:
  Origin (S3 / MediaPackage) → CloudFront edge locations (450+ PoP toàn cầu)
                                → Viewer tải từ edge gần nhất

Tại sao CDN giảm latency:
  Không CDN: Viewer ở HN → S3 us-east-1 → 150ms+
  Có CDN:    Viewer ở HN → CloudFront Singapore edge → 10-30ms

CDN Caching:
  Segments (.ts, .m4s): Cache lâu (không thay đổi)
  Manifests (.m3u8, .mpd): Cache ngắn (live: 1-2s, VOD: dài hơn)
```

---

## 2. VOD Pipeline — Video On Demand

### Kiến Trúc Cơ Bản

```
┌──────┐     ┌──────────┐     ┌────────────────┐     ┌──────────────┐     ┌────────┐
│ S3   │────▶│  Lambda  │────▶│  MediaConvert  │────▶│  S3 Output   │────▶│  CDN   │
│Source│     │ (trigger)│     │  (transcode)   │     │ (HLS/DASH)   │     │CloudFnt│
└──────┘     └──────────┘     └────────────────┘     └──────────────┘     └────────┘
   ↑              ↑                    ↑                      ↑                ↑
 Upload          S3 Event         Job settings            segments          Viewer
 video file    notification         JSON                  manifest           plays
```

### Pipeline Chi Tiết

```
Step 1: INGEST
  Nhà sản xuất upload file lên S3:
    s3://my-source-bucket/raw/movie-001.mp4

Step 2: TRIGGER
  S3 Event → SQS Queue → Lambda function
  Lambda tạo MediaConvert job:
    Input:  s3://my-source-bucket/raw/movie-001.mp4
    Output: s3://my-output-bucket/vod/movie-001/

Step 3: TRANSCODE (MediaConvert)
  Tạo multiple renditions:
  ├── HLS: 1080p, 720p, 480p, 360p
  ├── DASH: 1080p, 720p, 480p, 360p
  └── Thumbnails (dùng Frame Capture output)

  Output structure:
    s3://my-output-bucket/vod/movie-001/
    ├── hls/
    │   ├── index.m3u8                    ← Master Playlist
    │   ├── 1080p/playlist.m3u8
    │   ├── 1080p/seg_000.ts, seg_001.ts, ...
    │   ├── 720p/...
    │   └── 360p/...
    ├── dash/
    │   ├── index.mpd                     ← MPD Manifest
    │   └── ...
    └── thumbnails/
        └── thumb_000.jpg, thumb_001.jpg, ...

Step 4: DELIVER
  CloudFront distribution:
    Origin: s3://my-output-bucket (OAC bảo mật)
    Behaviors:
      /vod/*.m3u8  → TTL 5 giây (manifest thay đổi, VOD không cần ngắn như live)
      /vod/*.ts    → TTL 1 năm (segment bất biến)
      /vod/*.mpd   → TTL 5 giây
    
  Viewer URL:
    https://d123.cloudfront.net/vod/movie-001/hls/index.m3u8
```

### Workflow Notifications

```
MediaConvert Job Status:
  SUBMITTED → PROGRESSING → COMPLETE / ERROR

EventBridge Rule → Lambda → DynamoDB (cập nhật trạng thái)
                          → SNS (thông báo admin nếu ERROR)
                          → API Gateway (webhook đến video platform)
```

### VOD với DRM

```
Thêm DRM vào pipeline:
  MediaConvert → S3 output (CMAF encrypted)
              → MediaPackage v2 (nếu cần JIT packaging với DRM)

Hoặc:
  MediaConvert Job → SPEKE endpoint → Encrypted HLS/DASH → S3 → CloudFront
```

---

## 3. Live Streaming Pipeline — Phát Trực Tiếp

### Kiến Trúc Standard

```
┌─────────────┐    ┌───────────┐    ┌──────────────┐    ┌────────────┐    ┌────────┐
│  Encoder    │───▶│MediaLive  │───▶│ MediaPackage │───▶│ CloudFront │───▶│Viewers │
│(OBS/HW)     │    │(Channel)  │    │(Channel+Endp)│    │(Distrib.)  │    │        │
└─────────────┘    └───────────┘    └──────────────┘    └────────────┘    └────────┘
RTMPS ingest        Encode           Package + DRM         CDN cache
1 source            Bitrate ladder   HLS/DASH/CMAF         Edge locations
                    H.264 multi-     Just-in-time          450+ PoPs
                    resolution       packaging
```

### Pipeline Chi Tiết

```
Step 1: INGEST (MediaLive Input)
  Input Type: RTMP Push
  Primary:   rtmps://primary.medialive.us-east-1.amazonaws.com:443/app/
  Secondary: rtmps://secondary.medialive.us-east-1.amazonaws.com:443/app/
  (Standard channel có 2 pipeline A và B cho redundancy)

Step 2: ENCODE (MediaLive Channel)
  Video Descriptions:
    1080p @ 5Mbps H.264
    720p  @ 2.5Mbps H.264
    480p  @ 1Mbps H.264
    360p  @ 600Kbps H.264
  
  Audio Descriptions:
    AAC 192Kbps Stereo (EN)
    AAC 192Kbps Stereo (VI) ← nếu có đa ngôn ngữ

  Output Destinations:
    MediaPackage Channel URL (WebDAV)

Step 3: PACKAGE (MediaPackage)
  Channel: nhận từ MediaLive qua WebDAV
  Endpoints:
    HLS:  https://mediapackage.example.com/out/v1/.../index.m3u8
    DASH: https://mediapackage.example.com/out/v1/.../index.mpd
    CMAF: https://mediapackage.example.com/out/v1/.../index.m3u8 (CMAF+FP)
  
  Time-shift buffer: 30 phút (cho catch-up viewing)

Step 4: DELIVER (CloudFront)
  Origin: MediaPackage endpoint
  Cache behaviors:
    *.m3u8  → TTL 1 giây (live manifest cập nhật liên tục)
    *.ts    → TTL 1 năm (segment bất biến sau khi ghi)
    *.m4s   → TTL 1 năm
  
  Real-time logs → Kinesis → Lambda → Elasticsearch (monitoring)
```

### Standard Channel vs Single Pipeline

```
Standard Channel (2 pipelines):
  ┌──────────────────────────────────────────────┐
  │ Pipeline A: Primary encoder → MediaPackage A │
  │ Pipeline B: Backup encoder  → MediaPackage B │
  └──────────────────────────────────────────────┘
  
  Lợi ích: Nếu Pipeline A fail → MediaPackage failover sang Pipeline B tự động
  Chi phí: Gấp đôi (2 pipelines chạy song song)
  Use case: Production broadcast, thể thao quan trọng

Single Pipeline:
  ┌──────────────────────────────────┐
  │ 1 Pipeline: Encoder → MediaPkg  │
  └──────────────────────────────────┘
  
  Lợi ích: Chi phí thấp hơn 50%
  Rủi ro: Nếu fail → downtime
  Use case: Non-critical streams, development, testing
```

### Failover trong MediaLive

```
Automatic Input Failover:
  Primary Input   → Pipeline A
  Secondary Input → Pipeline B (hot standby)
  
  Nếu Primary Input mất tín hiệu:
    MediaLive tự động chuyển sang Secondary Input
    Thời gian chuyển: < 1 giây (configurable)
  
  Conditions kích hoạt failover:
    - Input loss (không nhận được RTMP packet)
    - Video black (màn hình đen)
    - Audio silence (im lặng)
    - Input quality < threshold
```

---

## 4. Interactive Live Pipeline — Phát Tương Tác

### IVS — Interactive Video Service Standard/Low-latency

```
┌──────────┐     ┌─────────────┐     ┌────────────────────┐     ┌────────┐
│   OBS    │────▶│  IVS Ingest │────▶│   IVS Channel      │────▶│Viewers │
│ (RTMPS)  │     │  Endpoint   │     │(Encode + Package)  │     │(HLS)   │
└──────────┘     └─────────────┘     └────────────────────┘     └────────┘
                                           │
                                           ▼
                                    Timed Metadata
                                    (nhúng vào HLS)
                                           │
                                     ┌─────▼──────┐
                                     │  Web App   │
                                     │  (React)   │
                                     │  Sync UI   │
                                     └────────────┘
```

### Timed Metadata — Metadata Đồng Bộ Theo Thời Gian

**Timed metadata** cho phép đồng bộ hoá UI tương tác với video:

```
Use case: Livestream bán hàng

Luồng:
  Host giới thiệu sản phẩm → gọi IVS API gắn metadata
  PutMetadata API:
  {
    "channelArn": "arn:aws:ivs:...",
    "metadata": "{\"product_id\": \"P123\", \"action\": \"show_buy_button\"}"
  }

  Player nhận metadata event:
  player.addEventListener(PlayerEventType.TEXT_METADATA_CUE, (cue) => {
    const data = JSON.parse(cue.text);
    if (data.action === 'show_buy_button') {
      showBuyButton(data.product_id); // Hiện nút mua ngay
    }
  });
```

### IVS Real-time Stage (WebRTC)

```
┌──────────┐    ┌─────────────────────────────────┐    ┌──────────┐
│Broadcaster│──▶│          IVS Stage               │──▶│ Viewers  │
│(iOS SDK) │    │  (Managed SFU — Selective        │    │(< 300ms) │
│WebRTC    │    │   Forwarding Unit)               │    │          │
└──────────┘    │                                 │    └──────────┘
                │  Max 12 publishers              │
                │  Max 10,000 viewers             │
                └─────────────────────────────────┘

Use cases:
  - Talk show trực tiếp (host + guests)
  - Q&A session (audience hỏi, host trả lời)
  - Game show (người chơi từ xa tham gia)
  - Giáo dục tương tác (teacher + students)
```

---

## 5. Live with Ads Pipeline — Phát Có Quảng Cáo

### SCTE-35 Ad Markers — Điểm Đánh Dấu Quảng Cáo

**SCTE-35** — Society of Cable Telecommunications Engineers standard 35 — Chuẩn Đánh Dấu Điểm Chèn Quảng Cáo: chuẩn broadcast cho phép encoder đánh dấu vị trí trong luồng để chèn quảng cáo.

```
Live Stream:        [...content...][SCTE-35 splice_insert][...content...]
                                         ↑
                                    Ad break start (30 giây)

SCTE-35 marker chứa:
  - splice_event_id: ID duy nhất của ad break
  - pts_time: thời điểm chèn (Presentation Timestamp)
  - break_duration: thời lượng ad break (giây)
  - avail_num: số lượng ad slot
```

### Pipeline Live with SSAI

```
┌──────────┐  RTMPS  ┌──────────┐  WebDAV  ┌───────────┐  HTTP  ┌───────────┐
│ Encoder  │────────▶│MediaLive │─────────▶│MediaPackge│───────▶│MediaTailor│
│(SCTE-35) │         │          │          │           │        │  (SSAI)   │
└──────────┘         └──────────┘          └───────────┘        └─────┬─────┘
                                                                        │
                      ┌─────────────────────────────────────────────────▼──┐
                      │         ADS — Ad Decision Server                    │
                      │  VAST/VMAP response: ad URLs + durations            │
                      └─────────────────────────────────────────────────────┘
                                                   │
                      ┌────────────────────────────▼──┐
                      │     CloudFront Distribution    │
                      │  Serve personalized HLS stream │
                      └───────────────────────────────┘
                                        │
                               ┌────────▼────────┐
                               │    Viewer A     │  ← Thấy Ad X (theo profile)
                               │    Viewer B     │  ← Thấy Ad Y
                               └─────────────────┘
```

**SSAI** — Server-Side Ad Insertion — Chèn Quảng Cáo Phía Máy Chủ: quảng cáo được ghép vào stream trên server trước khi phát tới viewer → ad blocker không chặn được (khác với CSAI — Client-Side Ad Insertion).

---

## 6. IoT Video Analytics Pipeline

### Kinesis Video Streams Pipeline

```
┌────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────┐
│  Camera    │───▶│ KVS Producer │───▶│ Kinesis Video   │───▶│ Consumer │
│(IP Camera/ │    │    SDK       │    │    Stream       │    │(Lambda / │
│ IoT Device)│    │              │    │                 │    │ App)     │
└────────────┘    └──────────────┘    └─────────────────┘    └──────────┘
                                               │
                                    ┌──────────▼──────────┐
                                    │   Rekognition Video  │
                                    │  (Real-time AI/ML)  │
                                    │  - Face detection   │
                                    │  - Object labels    │
                                    │  - PPE detection    │
                                    └─────────────────────┘
```

### Use Cases Thực Tế

```
Factory Floor Monitoring (Giám sát sàn sản xuất):
  Camera → KVS → Rekognition (PPE detection)
               → Lambda (cảnh báo nếu worker không đeo mũ bảo hiểm)
               → SNS → Quản lý nhận alert

Smart Home / Baby Monitor:
  Camera → KVS → WebRTC signaling channel
               → Mobile app (xem realtime qua WebRTC)
               → S3 (lưu recording có chuyển động)

Retail Analytics (Phân tích bán lẻ):
  Camera → KVS → Rekognition (face counting, dwell time)
               → Kinesis Data Streams → Kinesis Analytics
               → Dashboard (heatmap, customer flow)
```

---

## 7. Trade-offs Kiến Trúc

### Latency vs Scale vs Cost

```
                    LOW LATENCY
                         ↑
                         │ WebRTC (IVS Real-time)
                         │   < 300ms, max 10K viewers
                         │
              LL-HLS/LL-DASH
              2-5 giây, unlimited viewers
                         │
              Standard HLS/DASH
              15-30 giây, unlimited viewers
                         │
                         └─────────────────────→
                    LOW SCALE                HIGH SCALE
```

```
                    LOW COST
                         ↑
                         │ IVS (managed, simple)
                         │   Low cost for small scale
                         │
              MediaLive Single Pipeline
              Cheaper, no redundancy
                         │
              MediaLive Standard Pipeline
              More expensive, HA
                         │
                         └──────────────────────→
                    LOW RELIABILITY         HIGH RELIABILITY
```

### Khi Nào Dùng Gì?

| Scenario | Giải Pháp AWS | Trade-off |
|----------|--------------|-----------|
| **Gaming livestream (Twitch-style)** | IVS Low-latency | Đơn giản, managed, không cần ops |
| **Interactive shopping live** | IVS Real-time | Sub-second nhưng max 12 publishers |
| **TV broadcast (thể thao)** | MediaLive Standard + MediaPackage | HA, scale, phức tạp hơn |
| **Netflix-style VOD** | MediaConvert + S3 + CloudFront | Offline processing, không real-time |
| **Security cameras** | KVS | IoT-optimized, ML integration |
| **Video conference** | Amazon Chime SDK / IVS Real-time | WebRTC managed |

### MediaLive vs IVS — Khi Nào Chọn Cái Nào?

```
Chọn IVS khi:
  ✅ Scale nhỏ đến trung bình (< 1 triệu viewers)
  ✅ Cần tích hợp timed metadata và chat
  ✅ Muốn managed service, ít ops overhead
  ✅ Budget thấp hơn
  ✅ Standard use case (không cần custom encoder settings)

Chọn MediaLive khi:
  ✅ Yêu cầu broadcast-grade redundancy (Standard channel)
  ✅ Input từ nhiều nguồn (RTMP, SRT, MediaConnect, MP4)
  ✅ Cần SCTE-35 ad marker passthrough
  ✅ Custom audio mapping (multi-language)
  ✅ Output đến nhiều đích (S3, MediaPackage, RTMP push)
  ✅ Enterprise broadcast production
```

---

## 8. Giám Sát và Vận Hành

### Key Metrics — Chỉ Số Quan Trọng

```
MediaLive Channel Health:
  - ActiveAlerts: số lượng alert đang active
  - InputVideoFrameRate: FPS nhận được từ encoder
  - OutputVideoFrameRate: FPS xuất ra (phải bằng input FPS)
  - DroppedFrames: số frame bị drop (lý tưởng = 0)
  - NetworkIn / NetworkOut: traffic

MediaPackage:
  - IngressBytes: dữ liệu nhận vào
  - EgressBytes: dữ liệu phát ra
  - ActiveInput: nguồn đang active (A hay B)

CloudFront:
  - CacheHitRate: tỉ lệ cache hit (target > 90% cho VOD)
  - TotalErrorRate: tỉ lệ lỗi 4xx/5xx
  - OriginLatency: độ trễ khi cache miss
  - BytesDownloaded / BytesUploaded
```

### CloudWatch Alarms — Cảnh Báo

```
Thiết lập alarms cho live channel:

alarm_1: MediaLive InputVideoFrameRate < 29 trong 2 phút
  → Action: SNS → PagerDuty → On-call engineer

alarm_2: MediaLive DroppedFrames > 0 trong 5 phút
  → Action: SNS notification

alarm_3: CloudFront CacheHitRate < 80%
  → Action: Kiểm tra TTL settings, origin response headers

alarm_4: MediaPackage 5xx errors > 1%
  → Action: SNS → Investigate origin health
```

### Runbook — Sách Hướng Dẫn Xử Lý Sự Cố

```
Sự cố: Viewer báo video bị freeze / buffering

Checklist xử lý (theo thứ tự):
  1. Kiểm tra CloudWatch → MediaLive ActiveAlerts
     → Có alert? → Xem chi tiết → Xử lý theo alert
  
  2. Kiểm tra CloudFront Real-time Dashboard
     → CacheHitRate đột ngột giảm?
     → Origin errors tăng?
  
  3. Kiểm tra MediaLive Channel health
     → InputVideoFrameRate = 0? → Encoder issue
     → DroppedFrames tăng? → Network/CPU issue
  
  4. Kiểm tra MediaPackage origin
     → Endpoint responding? → curl test manifest URL
  
  5. Nếu Input A fail → Kích hoạt failover sang Input B
     (Manual: MediaLive Console → Channel → Input → Switch)
```

---

## ❓ Câu Hỏi Phỏng Vấn Thực Tế

### Câu Hỏi Thiết Kế Hệ Thống

**Câu 1:** Thiết kế pipeline cho nền tảng thể thao trực tiếp với 1 triệu concurrent viewers, yêu cầu latency < 5 giây, hỗ trợ DRM.

```
Gợi ý trả lời:
  Ingest: OBS/Hardware encoder → RTMPS → MediaLive (Standard channel)
  Encode: MediaLive → H.264 bitrate ladder (1080p, 720p, 480p, 360p)
  Package: MediaPackage → LL-HLS endpoint + DRM (FairPlay + Widevine SPEKE)
  Deliver: CloudFront (LL-HLS manifest TTL 1s, segments TTL long)
  
  HA: MediaLive Standard (2 pipelines) + MediaPackage failover
  Latency: LL-HLS → 2-5 giây ✅
  DRM: CBCS CMAF endpoint → FairPlay (iOS) + Widevine (Android)
```

**Câu 2:** Khác nhau giữa MediaLive và IVS là gì? Khi nào bạn chọn MediaLive?

```
Gợi ý trả lời:
  IVS: Managed, đơn giản, tích hợp timed metadata và chat, 
       phù hợp interactive live (bán hàng, game show nhỏ)
  
  MediaLive: Broadcast-grade, Standard channel redundancy,
             SCTE-35 ad markers, nhiều input type (SRT, MediaConnect),
             custom audio mapping, output đến nhiều đích
  
  Chọn MediaLive khi: broadcast production, cần HA, SCTE-35, custom settings
  Chọn IVS khi: interactive, simple setup, timed metadata, chat tích hợp
```

**Câu 3:** Tại sao SSAI tốt hơn CSAI cho OTT platform?

```
Gợi ý trả lời:
  CSAI (Client-Side): JavaScript load ad từ ad server → bị ad blocker chặn
                       → ~30-40% viewer không thấy quảng cáo

  SSAI (Server-Side): MediaTailor ghép ad vào stream trên server
                      → Player chỉ thấy 1 stream liên tục, không biết là ad
                      → Ad blocker không thể phân biệt
                      → 100% impression
  
  Trade-off SSAI: Phức tạp hơn, cần ADS (Ad Decision Server) tích hợp,
                  latency khi tải ad segments
```

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---------|----------|-------------|
| [4-drm-fundamentals.md](./4-drm-fundamentals.md) | **5-media-pipeline-concepts.md** | [../02-mediaconvert/README.md](../02-mediaconvert/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn thành
