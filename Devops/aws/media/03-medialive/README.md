# AWS Elemental MediaLive — Mã Hoá Video Trực Tiếp (Live Video Encoding)

> AWS Elemental MediaLive là dịch vụ mã hoá video trực tiếp (live encoding) được quản lý hoàn toàn bởi AWS. Dịch vụ nhận luồng video đầu vào thô (raw stream) và mã hoá thành nhiều độ phân giải/bitrate để phát trực tiếp đến hàng triệu người xem, không cần quản lý phần cứng encoder.

## 📚 Mục Lục (Table of Contents)

1. [MediaLive Là Gì?](#1-medialive-là-gì)
2. [Kiến Trúc Tổng Quan](#2-kiến-trúc-tổng-quan)
3. [Các Khái Niệm Cốt Lõi](#3-các-khái-niệm-cốt-lõi)
4. [Pipeline Live Streaming](#4-pipeline-live-streaming)
5. [Nội Dung Chi Tiết](#5-nội-dung-chi-tiết)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#6-câu-hỏi-phỏng-vấn-thường-gặp)

---

## 1. MediaLive Là Gì?

**AWS Elemental MediaLive** là dịch vụ **live video encoding** (mã hoá video trực tiếp) được quản lý hoàn toàn bởi AWS. Trong khi MediaConvert xử lý video từ file (VOD), MediaLive xử lý **luồng video thời gian thực** (real-time stream) liên tục 24/7.

### Vị Trí Trong Pipeline Live Streaming

```
                 ┌────────────────── LIVE STREAMING PIPELINE ───────────────────┐
                 │                                                               │
[Encoder/Camera]  →  [MediaLive]  →  [MediaPackage]  →  [CloudFront]  →  [Viewer]
  OBS / Wirecast      Channel         Packaging           CDN toàn cầu    HLS player
  Hardware encoder    Encode +        HLS/DASH/CMAF       phân phối       trình duyệt
  RTMP/RTP input      multi-bitrate   DRM + time-shift    adaptive        / mobile
                      output          origin server       bitrate
```

### So Sánh MediaLive vs Các Giải Pháp Khác

| Tiêu Chí | MediaLive | Amazon IVS | Tự dựng FFmpeg |
|----------|-----------|------------|----------------|
| **Quản lý hạ tầng** | Không cần | Không cần | Phải tự quản lý |
| **Độ trễ (Latency)** | 6–30 giây (standard HLS) | < 5 giây (low-latency) | Tuỳ cấu hình |
| **Người xem đồng thời** | Hàng triệu (qua CloudFront) | Hàng triệu | Phụ thuộc hạ tầng |
| **Tính năng broadcast** | Đầy đủ (SCTE-35, DVB-Sub...) | Cơ bản | Cần tự xây |
| **Redundancy** | Standard (2 pipelines) | Tích hợp sẵn | Phải tự xây |
| **Use case điển hình** | Truyền hình, thể thao, tin tức | Livestream bán hàng, game | Dev/test |
| **Chi phí** | Theo giờ chạy channel | Theo giờ ingest + data | Chi phí EC2 cố định |

> **Khi nào dùng MediaLive thay vì IVS?**
> - Cần tính năng broadcast chuyên nghiệp (SCTE-35, Motion Graphics, input switching)
> - Có nhiều loại input phức tạp (MediaConnect, UDP/RTP từ satellite)
> - Cần tích hợp DRM và đóng gói nâng cao qua MediaPackage
> - Quy mô lớn với SLA nghiêm ngặt (kênh truyền hình OTT)

---

## 2. Kiến Trúc Tổng Quan

### Các Thành Phần Chính

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AWS ELEMENTAL MEDIALIVE                         │
│                                                                     │
│  ┌───────────┐    ┌──────────────────────────────────────────────┐  │
│  │  Input    │    │              Channel                          │  │
│  │           │    │                                              │  │
│  │ RTMP Push │    │  ┌────────────┐   ┌────────────────────────┐ │  │
│  │ RTMP Pull │───▶│  │  Pipeline A │   │     Output Groups      │ │  │
│  │ RTP/UDP   │    │  │  (Encode)  │──▶│  HLS │ DASH │ Archive  │ │  │
│  │ HLS Pull  │    │  └────────────┘   └────────────────────────┘ │  │
│  │ MediaConn │    │  ┌────────────┐           │                  │  │
│  │ MP4 (file)│    │  │  Pipeline B │           │                  │  │
│  └───────────┘    │  │  (Standby) │           │                  │  │
│                   │  └────────────┘           │                  │  │
│                   └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                                │
                          ┌─────────────────────┼────────────────────┐
                          ▼                     ▼                    ▼
                    MediaPackage            S3 Bucket           RTMP Push
                    (Packaging +          (Archive +          (Facebook /
                     DRM origin)           VOD clip)          YouTube Live)
```

### Luồng Xử Lý (Processing Flow)

```
1. Encoder gửi luồng video vào MediaLive Input (RTMP/RTP)
2. Channel nhận luồng, xử lý qua Video/Audio Selector
3. Encode thành nhiều độ phân giải (1080p, 720p, 480p, 360p)
4. Tạo output theo cấu hình Output Group
5. Gửi đến đích: MediaPackage / S3 / RTMP
6. MediaPackage đóng gói thành HLS/DASH, phân phối qua CloudFront
7. Viewer nhận stream, player ABR tự chọn bitrate phù hợp
```

---

## 3. Các Khái Niệm Cốt Lõi

### 3.1 Channel (Kênh Mã Hoá)

**Channel** là tài nguyên trung tâm của MediaLive. Mỗi Channel đại diện cho một luồng phát trực tiếp độc lập.

**Trạng thái Channel (Channel States):**

| Trạng Thái | Ý Nghĩa | Ghi Chú |
|-----------|---------|---------|
| `IDLE` | Channel đã tạo, chưa chạy | Không tốn phí encoding |
| `STARTING` | Channel đang khởi động | Quá trình mất 1–3 phút |
| `RUNNING` | Channel đang chạy, encode live | **Tính phí theo giờ** |
| `STOPPING` | Channel đang dừng | |
| `DELETED` | Channel đã xoá | Không thể khôi phục |

> **Quan trọng về chi phí:** MediaLive tính phí khi Channel ở trạng thái `RUNNING`, không phụ thuộc vào việc có viewer hay không.

**Cấu trúc Channel:**
```
Channel
├── Input Specification     — Codec đầu vào, độ phân giải tối đa, bitrate tối đa
├── Input Attachments[]     — Kết nối với các Input (RTMP, RTP, HLS...)
│   ├── Input Settings
│   ├── Audio Selectors     — Chọn track âm thanh từ đầu vào
│   ├── Caption Selectors   — Chọn track phụ đề từ đầu vào
│   └── Video Selector      — Chọn track video, xử lý deinterlace
├── Encoder Settings
│   ├── Video Descriptions[]    — Cấu hình encode từng độ phân giải
│   ├── Audio Descriptions[]    — Cấu hình encode âm thanh
│   └── Caption Descriptions[]  — Cấu hình encode phụ đề
└── Output Groups[]         — Đích đến: MediaPackage, S3, RTMP...
```

### 3.2 Input (Đầu Vào)

**Input** là nguồn video đầu vào của Channel. MediaLive hỗ trợ nhiều loại input:

| Loại Input | Giao Thức | Push/Pull | Mô Tả |
|-----------|----------|-----------|-------|
| **RTMP Push** | RTMP | Push | Encoder gửi tới MediaLive endpoint |
| **RTMP Pull** | RTMP | Pull | MediaLive kéo từ URL RTMP |
| **RTP Push** | RTP/UDP | Push | Dùng cho broadcast chuyên nghiệp |
| **HLS Pull** | HLS | Pull | MediaLive kéo từ HLS manifest URL |
| **MediaConnect** | ZIXI/RTP | Push | Vận chuyển video chất lượng cao |
| **MP4 Pull** | HTTP/S3 | Pull | File MP4 (dùng làm slate/filler) |
| **Elemental Link** | Elemental | Push | Hardware encoder AWS |

### 3.3 Pipeline (Đường Ống Xử Lý)

**Pipeline** là instance thực sự thực hiện encoding. Channel Standard có 2 pipeline (A và B) chạy song song để đảm bảo redundancy — tính dự phòng.

```
STANDARD CHANNEL (2 pipelines — Khuyến nghị cho production):
┌─────────────────────────────────────────────────────┐
│                    Channel                           │
│  Input ──▶ Pipeline A (Primary)   ──▶ Output        │
│  Input ──▶ Pipeline B (Secondary) ──▶ Output        │
│                                                     │
│  Cả hai pipeline encode độc lập, output ghi lên     │
│  cùng đích. Nếu A lỗi, B tự động tiếp quản.         │
└─────────────────────────────────────────────────────┘

SINGLE PIPELINE CHANNEL (1 pipeline — Tiết kiệm chi phí):
┌─────────────────────────────────────────────────────┐
│                    Channel                           │
│  Input ──▶ Pipeline A (Primary)   ──▶ Output        │
│                                                     │
│  Không có dự phòng. Nếu pipeline lỗi → mất stream.  │
└─────────────────────────────────────────────────────┘
```

### 3.4 Output Group (Nhóm Đầu Ra)

**Output Group** xác định đích đến và định dạng của luồng output. Một Channel có thể có nhiều Output Group:

```
Channel
├── Output Group 1: HLS → MediaPackage   ← Phân phối đến viewer
├── Output Group 2: Archive → S3          ← Lưu trữ để VOD sau này
└── Output Group 3: RTMP Push → YouTube   ← Phát đồng thời lên YouTube
```

### 3.5 Bitrate Ladder (Thang Tốc Độ Bit)

**Bitrate ladder** là tập hợp các cấu hình encode với độ phân giải và bitrate khác nhau, cho phép ABR — Adaptive Bitrate Streaming — Phát Trực Tuyến Thích Ứng Tốc Độ Bit:

```
Ví dụ Bitrate Ladder cho live streaming HD:

Rendition (phiên bản)   Resolution   Video Bitrate   Audio Bitrate
─────────────────────   ──────────   ─────────────   ─────────────
1080p High              1920×1080    5,000 Kbps      192 Kbps
720p                    1280×720     3,000 Kbps      128 Kbps
540p                     960×540     1,500 Kbps      128 Kbps
360p                     640×360       800 Kbps       96 Kbps
270p (Mobile)            480×270       400 Kbps       64 Kbps
```

---

## 4. Pipeline Live Streaming

### Use Case 1: Phát Sóng Thể Thao (Sports Broadcasting)

```
Camera/OB Van (xe phát sóng lưu động)
        │ Contribution feed (SDI/IP)
        ▼
AWS Elemental Live (hardware encoder)
        │ RTMP/RTP Push
        ▼
MediaLive Channel (Standard — 2 pipelines)
  Input: 2 RTMP endpoints (Pipeline A + B)
  Encoding:
    - 1080p @ 5 Mbps (H.264)
    - 720p  @ 3 Mbps
    - 360p  @ 800 Kbps
  Captions: DVB-Sub → WebVTT conversion
        │
        ├──▶ MediaPackage Channel
        │         │
        │         ├── HLS endpoint (viewer)
        │         └── DASH endpoint (viewer)
        │                 │
        │           CloudFront CDN
        │                 │
        │           Viewer HLS.js player (web)
        │           iOS/Android native player
        │
        └──▶ S3 Archive (full recording)
```

### Use Case 2: Kênh Truyền Hình OTT 24/7

```
Multiple Sources (nhiều nguồn):
  - Live studio feed (RTMP)
  - Satellite feed (MediaConnect)
  - VOD filler (MP4 Pull S3)
        │
        ▼
MediaLive Channel (Standard)
  Input Switching (chuyển nguồn theo lịch):
    08:00–12:00 → Live studio (tin tức sáng)
    12:00–13:00 → VOD filler (phim tài liệu)
    13:00–14:00 → Satellite (tin tức trưa)
    ...

  SCTE-35 markers (đánh dấu điểm quảng cáo) mỗi 30 phút
        │
        ▼
MediaPackage → MediaTailor (SSAI)
        │
        ▼
CloudFront → Viewer
(Quảng cáo được cá nhân hoá theo viewer)
```

### Use Case 3: Concert/Event Trực Tiếp (Low-Latency)

```
Live Band on Stage
        │ Camera → Mixer → Encoder
        ▼
OBS Studio (RTMP Push)
        │
        ▼
MediaLive Channel (Standard)
  Cấu hình low-latency:
    - Segment length: 2 giây (thay vì 6 giây)
    - Partial segments: ON
        │
        ▼
MediaPackage (Low-latency HLS endpoint)
        │
        ▼
CloudFront (Low-latency Cache)
        │
        ▼
Viewer (độ trễ ~3–8 giây từ camera đến màn hình)
```

---

## 5. Nội Dung Chi Tiết

```
03-medialive/
├── README.md                    ← [BẠN ĐANG Ở ĐÂY] Tổng quan
├── 1-channel-input.md           Input types, Security Groups, Input specification
├── 2-encoding-settings.md       Video/Audio Descriptions, Bitrate ladder, H.264/H.265
├── 3-redundancy-failover.md     Standard vs Single pipeline, Input failover, Automatic failover
├── 4-output-destinations.md     MediaPackage, S3, RTMP Push, UDP/TS, HLS output group
└── 5-schedule-scte35.md         Schedule Actions, SCTE-35 ad markers, Input switching
```

### Bản Đồ Học Tập Đề Xuất

```
Bắt đầu tại đây (README)
         │
         ▼
1-channel-input.md          ← Hiểu các loại đầu vào trước
         │
         ▼
2-encoding-settings.md      ← Cấu hình bitrate ladder
         │
         ▼
3-redundancy-failover.md    ← Thiết lập HA cho production
         │
         ▼
4-output-destinations.md    ← Kết nối với MediaPackage/S3
         │
         ▼
5-schedule-scte35.md        ← Nâng cao: lịch phát + quảng cáo
```

---

## 6. Câu Hỏi Phỏng Vấn Thường Gặp

### Câu hỏi cơ bản

**Q: MediaLive khác gì Amazon IVS?**

> **MediaLive** là dịch vụ broadcast chuyên nghiệp, hỗ trợ nhiều loại input (RTMP, RTP, MediaConnect, HLS Pull), có tính năng SCTE-35 ad markers, Motion Graphics overlay, DVB-Sub caption, input switching và Standard channel với 2 pipelines dự phòng. Phù hợp cho kênh truyền hình OTT, phát sóng thể thao, sự kiện lớn.
>
> **Amazon IVS** — Interactive Video Service là dịch vụ đơn giản hơn, chỉ nhận RTMP input, nhưng có độ trễ rất thấp (< 5 giây), tích hợp sẵn timed metadata và IVS Chat. Phù hợp cho livestream tương tác (bán hàng, gaming, giáo dục).

**Q: MediaLive tính phí như thế nào?**

> Theo **giờ chạy Channel** (per channel-hour), dựa trên:
> - **Input codec và bitrate** (HD vs UHD, tốc độ bit đầu vào)
> - **Output codec** (H.264 rẻ hơn H.265)
> - **Loại Channel** (Standard = 2× Single vì 2 pipelines)
> - **Output bitrate** tổng cộng
>
> Channel chạy nhưng không có viewer vẫn bị tính phí — hãy `STOP` channel khi không dùng.

**Q: Standard Channel vs Single Pipeline Channel khác gì nhau?**

> - **Standard Channel** chạy 2 pipelines (A và B) song song. Nếu một pipeline lỗi, pipeline kia tự động tiếp quản mà không mất stream. Chi phí gần gấp đôi Single.
> - **Single Pipeline Channel** chỉ có 1 pipeline, nếu lỗi sẽ mất stream cho đến khi khôi phục (vài phút). Rẻ hơn ~50%, dùng được cho môi trường dev/test hoặc stream không quan trọng.

### Câu hỏi nâng cao

**Q: Làm thế nào xử lý khi input nguồn bị mất (Input Loss)?**

> MediaLive có cơ chế **Input Loss Behavior** — Hành Vi Khi Mất Đầu Vào:
> 1. **Emit black** — phát màn hình đen (mặc định)
> 2. **Repeat last frame** — lặp lại frame cuối cùng
> 3. **Emit a slate** — chèn video dự phòng (slate image hoặc MP4)
> 4. **Input failover** — tự động chuyển sang input dự phòng (nếu đã cấu hình)
>
> Kết hợp với CloudWatch Alarm `InputLossSeconds` để cảnh báo kịp thời.

**Q: SCTE-35 là gì và tại sao cần thiết?**

> **SCTE-35** — Society of Cable Telecommunications Engineers 35 là chuẩn tín hiệu đánh dấu điểm chèn quảng cáo trong luồng video. MediaLive có thể:
> 1. Nhận SCTE-35 từ encoder nguồn và chuyển tiếp (passthrough)
> 2. Chèn SCTE-35 markers theo lịch Schedule Actions
>
> Downstream, MediaPackage và MediaTailor đọc markers này để biết khi nào chèn quảng cáo cá nhân hoá — đây là nền tảng của hệ thống quảng cáo OTT.

**Q: Làm thế nào giảm độ trễ (latency) của live stream?**

> Ba cách chính:
> 1. **Giảm HLS segment length**: từ 6 giây xuống 2–4 giây (tăng số request nhưng giảm latency)
> 2. **Dùng Low-Latency HLS (LL-HLS)**: MediaPackage v2 hỗ trợ LL-HLS với partial segments
> 3. **Dùng Amazon IVS Real-time**: WebRTC channel của IVS đạt < 1 giây latency, phù hợp tương tác 2 chiều
>
> Trade-off: segment ngắn hơn → nhiều file hơn → tăng origin load và CDN request cost.

---

## 🔗 Tài Liệu Tham Khảo

- [MediaLive User Guide](https://docs.aws.amazon.com/medialive/latest/ug/)
- [MediaLive API Reference](https://docs.aws.amazon.com/medialive/latest/apireference/)
- [MediaLive Pricing](https://aws.amazon.com/medialive/pricing/)
- [Getting Started: Create a Channel](https://docs.aws.amazon.com/medialive/latest/ug/getting-started.html)

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phần Trước:** [02-mediaconvert/](../02-mediaconvert/README.md) — VOD Transcoding
**Phần Tiếp Theo:** [04-mediapackage/](../04-mediapackage/README.md) — Packaging & DRM
