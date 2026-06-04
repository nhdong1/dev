# Nền Tảng Media — Codec, Protocols & Pipeline

> Kiến thức nền tảng không thể thiếu trước khi học bất kỳ dịch vụ AWS Media Services nào. Nắm vững phần này giúp bạn hiểu sâu hơn MediaConvert, MediaLive, MediaPackage và các dịch vụ khác.

## 📚 Mục Lục (Table of Contents)

1. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)
2. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
3. [Pipeline Media End-to-End](#pipeline-media-end-to-end)
4. [Nội Dung Chi Tiết](#nội-dung-chi-tiết)
5. [Câu Hỏi Ôn Tập](#câu-hỏi-ôn-tập)

---

## 🗺️ Tổng Quan Chủ Đề

```
01-fundamentals/
├── README.md                    ← [BẠN ĐANG Ở ĐÂY] Tổng quan & roadmap
├── 1-codec-container.md         Codec (H.264/H.265/AV1) + Container (MP4/HLS/DASH)
├── 2-streaming-protocols.md     HLS, DASH, RTMP, SRT, WebRTC — so sánh & use cases
├── 3-adaptive-bitrate.md        ABR — Adaptive Bitrate Streaming — phát thích ứng tốc độ bit
├── 4-drm-fundamentals.md        DRM — Digital Rights Management — bảo vệ bản quyền nội dung
└── 5-media-pipeline-concepts.md Ingest → Encode → Package → Deliver — quy trình media đầy đủ
```

---

## 🔑 Khái Niệm Cốt Lõi

### Codec — Bộ Mã Hoá / Giải Mã (Coder-Decoder)

**Codec** là phần mềm hoặc phần cứng dùng để **nén** (encode) và **giải nén** (decode) dữ liệu video/audio. Không có codec, một giờ video HD thô có thể chiếm hàng trăm GB.

```
Video thô (RAW) → [ENCODER] → Video nén → [DECODER] → Video phát lại
   ~100 GB/giờ                 ~2 GB/giờ               Hiển thị trên màn hình
```

**Các codec video phổ biến:**

| Codec | Tên Đầy Đủ | Năm Ra Đời | Hiệu Quả Nén | Use Case |
|-------|-----------|------------|--------------|----------|
| **H.264 / AVC** | Advanced Video Coding | 2003 | Chuẩn | Web, mobile, broadcast |
| **H.265 / HEVC** | High Efficiency Video Coding | 2013 | Tốt hơn H.264 ~50% | 4K, HDR, bandwidth hạn chế |
| **AV1** | AOMedia Video 1 | 2018 | Tốt hơn H.265 ~30% | YouTube, Netflix (tương lai) |
| **VP9** | Video Processor 9 | 2013 | Tương đương H.265 | YouTube, Chrome |

### Container — Định Dạng Đóng Gói (Media Container Format)

**Container** là "chiếc hộp" chứa đồng thời video, audio, phụ đề và metadata. Container KHÔNG phải codec — H.264 có thể được đặt trong MP4, MKV hoặc TS.

```
Container (MP4)
├── Video track  — H.264 encoded frames
├── Audio track  — AAC encoded audio
├── Subtitle     — SRT/WebVTT text data
└── Metadata     — duration, title, creation date
```

**Các container phổ biến trong streaming:**

| Container | Extension | Dùng Cho | Đặc Điểm |
|-----------|-----------|----------|-----------|
| **MP4** | `.mp4` | VOD download, progressive playback | Phổ biến nhất, iOS/Android hỗ trợ tốt |
| **HLS** | `.m3u8` + `.ts` | HTTP Live Streaming (Apple) | Chunk-based, adaptive bitrate |
| **DASH** | `.mpd` + `.m4s` | Dynamic Adaptive Streaming | Open standard, linh hoạt hơn HLS |
| **CMAF** | `.m4s` | Common Media Application Format | Kết hợp HLS + DASH dùng chung chunk |
| **MKV** | `.mkv` | Lưu trữ, Blu-ray rip | Hỗ trợ nhiều track nhất |
| **MPEG-TS** | `.ts` | Live broadcast, satellite | Chịu lỗi tốt, truyền thống |

### Bitrate — Tốc Độ Bit

**Bitrate** (đơn vị: Kbps hoặc Mbps) là lượng dữ liệu được xử lý mỗi giây. Bitrate cao hơn → chất lượng tốt hơn → tốn băng thông hơn.

```
Tham khảo bitrate cho video:
- 360p  SD  :  500 Kbps  — video điện thoại cũ
- 720p  HD  : 2.5 Mbps  — streaming tiêu chuẩn
- 1080p FHD : 5-8 Mbps  — Netflix HD
- 4K    UHD : 15-25 Mbps — Netflix 4K, YouTube 4K
```

**CBR vs VBR vs CRF:**
- **CBR** — Constant Bitrate — Tốc Độ Bit Cố Định: bitrate không đổi, dễ dự đoán băng thông
- **VBR** — Variable Bitrate — Tốc Độ Bit Thay Đổi: tăng khi cảnh phức tạp, giảm khi cảnh đơn giản
- **CRF** — Constant Rate Factor — Hệ Số Chất Lượng Cố Định: điều chỉnh bitrate để duy trì chất lượng nhất quán

### Resolution — Độ Phân Giải

| Tên | Pixel | Tỉ Lệ Khung Hình |
|-----|-------|-----------------|
| **SD** — Standard Definition | 640×480 | 4:3 |
| **HD** — High Definition (720p) | 1280×720 | 16:9 |
| **FHD** — Full HD (1080p) | 1920×1080 | 16:9 |
| **4K / UHD** — Ultra HD | 3840×2160 | 16:9 |
| **8K** | 7680×4320 | 16:9 |

### Framerate — Tốc Độ Khung Hình (FPS — Frames Per Second)

- **24 FPS** — phim điện ảnh (cinematic look)
- **30 FPS** — truyền hình NTSC (Mỹ, Nhật)
- **25 FPS** — truyền hình PAL (Châu Âu, VN)
- **60 FPS** — game, thể thao (mượt mà hơn)
- **120 FPS** — slow-motion, VR

---

## 🔄 Pipeline Media End-to-End

Mọi hệ thống media dù VOD hay Live đều tuân theo pipeline 4 bước:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   INGEST    │───▶│   ENCODE    │───▶│   PACKAGE   │───▶│   DELIVER   │
│  (Nhập)     │    │  (Mã hoá)   │    │  (Đóng gói) │    │ (Phân phối) │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
     │                   │                   │                   │
     ▼                   ▼                   ▼                   ▼
 Camera/File          Transcode           HLS/DASH           CDN Edge
 RTMP/SRT             H.264→H.265         Segment             CloudFront
 S3 upload            Bitrate ladder      DRM encrypt         Viewer
```

### AWS Services Map theo Pipeline:

```
INGEST         →  ENCODE          →  PACKAGE          →  DELIVER
─────────────────────────────────────────────────────────────────
MediaConnect      MediaConvert        MediaPackage         CloudFront
MediaLive Input   MediaLive           MediaStore           MediaTailor
KVS Producer      Rekognition Video   S3 (processed)       IVS Player
```

---

## 📁 Nội Dung Chi Tiết

### 1. Codec & Container
**File:** [1-codec-container.md](./1-codec-container.md)

- So sánh chi tiết H.264, H.265/HEVC, AV1, VP9
- HDR — High Dynamic Range: HDR10, HDR10+, Dolby Vision, HLG
- Audio codec: AAC, Dolby Digital (AC3), Dolby Atmos (EAC3), MP3
- Container deep dive: MP4 vs MKV vs HLS vs DASH vs CMAF
- Cách chọn codec và container cho từng use case

### 2. Streaming Protocols — Giao Thức Truyền Phát
**File:** [2-streaming-protocols.md](./2-streaming-protocols.md)

- **HLS** — HTTP Live Streaming: Apple standard, #EXTM3U manifest
- **DASH** — Dynamic Adaptive Streaming over HTTP: open standard, MPD manifest
- **RTMP** — Real-Time Messaging Protocol: ingest protocol (không phải playback)
- **SRT** — Secure Reliable Transport: thay thế RTMP cho broadcast
- **WebRTC** — Web Real-Time Communication: peer-to-peer < 500ms latency
- So sánh bảng: latency, support, use case, pros/cons

### 3. Adaptive Bitrate Streaming — Phát Thích Ứng Tốc Độ Bit
**File:** [3-adaptive-bitrate.md](./3-adaptive-bitrate.md)

- Cơ chế ABR hoạt động như thế nào (segment-based switching)
- Bitrate ladder — Thang tốc độ bit: thiết kế rendition set tối ưu
- HLS vs DASH ABR algorithm khác nhau thế nào
- CMAF — Common Media Application Format: giải quyết bài toán dual-packaging
- Low-latency ABR (LL-HLS, LL-DASH): chunked transfer encoding

### 4. DRM — Digital Rights Management
**File:** [4-drm-fundamentals.md](./4-drm-fundamentals.md)

- **Widevine** (Google): Chrome, Android — L1/L2/L3 security levels
- **FairPlay** (Apple): Safari, iOS, macOS — ONLY trên Apple ecosystem
- **PlayReady** (Microsoft): Edge, Windows, Xbox — enterprise focus
- Luồng DRM: License request → Key server → Decrypt → Playback
- **SPEKE** — Secure Packager and Encoder Key Exchange: AWS standard
- Multi-DRM setup cho cross-platform OTT

### 5. Media Pipeline Concepts — Kiến Trúc Pipeline Media
**File:** [5-media-pipeline-concepts.md](./5-media-pipeline-concepts.md)

- Pipeline VOD (Video on Demand): S3 → MediaConvert → S3 → CloudFront
- Pipeline Live Streaming: Encoder → MediaLive → MediaPackage → CloudFront
- Pipeline Interactive Live: OBS → IVS → Player SDK
- Pipeline IoT Video: Camera → KVS → Rekognition
- Quyết định kiến trúc: latency vs cost vs complexity trade-offs

---

## ❓ Câu Hỏi Ôn Tập

### Mức Cơ Bản

1. H.264 và H.265 khác nhau như thế nào? Khi nào chọn H.265?
2. Container MP4 và HLS (.m3u8) có gì khác nhau?
3. Tại sao streaming không dùng MP4 raw mà phải chia thành segment?
4. Bitrate 5 Mbps nghĩa là gì? Ảnh hưởng đến chất lượng và băng thông thế nào?
5. VOD và Live streaming khác nhau ở điểm cốt lõi nào?

### Mức Trung Cấp

6. ABR — Adaptive Bitrate Streaming hoạt động như thế nào? Player quyết định chuyển rendition dựa trên gì?
7. RTMP và HLS khác nhau về vai trò: cái nào dùng để ingest, cái nào để playback?
8. SRT có ưu điểm gì so với RTMP khiến broadcast hiện đại chuyển sang SRT?
9. Tại sao OTT platform cần cả Widevine VÀ FairPlay thay vì chỉ một loại DRM?
10. CMAF giải quyết vấn đề gì mà HLS và DASH riêng lẻ không làm được?

### Mức Nâng Cao

11. Thiết kế bitrate ladder tối ưu cho nền tảng xem trên mobile và TV 4K.
12. Low-latency HLS (LL-HLS) hoạt động khác gì so với HLS truyền thống?
13. Giải thích luồng DRM: từ lúc player yêu cầu đến lúc video decrypt và phát được.
14. Khi nào dùng CBR thay vì VBR trong live streaming? Tại sao?
15. Trade-off giữa segment duration ngắn (2s) và dài (10s) trong HLS là gì?

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Tiếp Theo |
|-------|----------|-----------|
| [Media README](../README.md) | **01-fundamentals/** | [02-mediaconvert/](../02-mediaconvert/) |

| File | Chủ Đề |
|------|--------|
| [1-codec-container.md](./1-codec-container.md) | Codec H.264/H.265/AV1, Container MP4/HLS/DASH |
| [2-streaming-protocols.md](./2-streaming-protocols.md) | HLS, DASH, RTMP, SRT, WebRTC so sánh |
| [3-adaptive-bitrate.md](./3-adaptive-bitrate.md) | ABR — Adaptive Bitrate Streaming |
| [4-drm-fundamentals.md](./4-drm-fundamentals.md) | DRM — Widevine, FairPlay, PlayReady, SPEKE |
| [5-media-pipeline-concepts.md](./5-media-pipeline-concepts.md) | Ingest → Encode → Package → Deliver |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn thành
