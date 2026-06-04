# Cài Đặt Codec Video — H.264, H.265, AV1

> Cấu hình codec video đúng là yếu tố quyết định **chất lượng**, **kích thước file**, và **khả năng tương thích** của output. Phần này đi sâu vào từng thông số của H.264, H.265 và AV1 trong MediaConvert.

## 📚 Mục Lục

1. [Tổng Quan Codec Video Trong MediaConvert](#1-tổng-quan-codec-video-trong-mediaconvert)
2. [Rate Control Mode — Chế Độ Kiểm Soát Tốc Độ Bit](#2-rate-control-mode--chế-độ-kiểm-soát-tốc-độ-bit)
3. [H.264 / AVC — Advanced Video Coding](#3-h264--avc--advanced-video-coding)
4. [H.265 / HEVC — High Efficiency Video Coding](#4-h265--hevc--high-efficiency-video-coding)
5. [AV1 — AOMedia Video 1](#5-av1--aomedia-video-1)
6. [Bitrate Ladder — Thang Tốc Độ Bit](#6-bitrate-ladder--thang-tốc-độ-bit)
7. [Cài Đặt Nâng Cao](#7-cài-đặt-nâng-cao)

---

## 1. Tổng Quan Codec Video Trong MediaConvert

### Codec Được Hỗ Trợ

| Codec | Tên Đầy Đủ | Định Dạng Container | Use Case |
|-------|-----------|---------------------|---------|
| **H_264** | H.264 / AVC — Advanced Video Coding | HLS, DASH, CMAF, MP4 | Phổ biến nhất, mọi thiết bị |
| **H_265** | H.265 / HEVC — High Efficiency Video Coding | HLS (fMP4), DASH, CMAF | 4K, HDR, tiết kiệm bandwidth |
| **AV1** | AV1 — AOMedia Video 1 | DASH, CMAF | Modern web (Chrome/Firefox), tiết kiệm nhất |
| **VP9** | VP9 | WebM, DASH | YouTube, Chrome, legacy |
| **MPEG2** | MPEG-2 | MPEG-TS | Broadcast, satellite |
| **PRORES** | Apple ProRes | MOV, MXF | Professional editing archive |
| **FRAME_CAPTURE** | Frame Capture | RAW (JPG/PNG) | Thumbnail generation |

### Chọn Codec Theo Use Case

```
Cần hỗ trợ mọi thiết bị (iOS cũ, Android cũ, Smart TV)?
  → H.264 (H_264)

Content 4K/HDR hoặc muốn tiết kiệm ~50% bandwidth?
  → H.265 (H_265) — cần thiết bị iOS 11+, Android 5+

Modern web/OTT, muốn tiết kiệm bandwidth tối đa, chấp nhận encode lâu hơn?
  → AV1

Lưu trữ master cho editing, chất lượng lossless/near-lossless?
  → ProRes (PRORES)

Broadcast truyền thống (satellite, DTT)?
  → MPEG2
```

---

## 2. Rate Control Mode — Chế Độ Kiểm Soát Tốc Độ Bit

**Rate Control Mode** là thông số quan trọng nhất, quyết định cách codec phân bổ bitrate:

### 2.1 CBR — Constant Bitrate (Tốc Độ Bit Cố Định)

```
Bitrate: ─────────────────────────── (luôn cố định ở 5 Mbps)
Video:   [Cảnh đơn giản] [Cảnh phức tạp] [Cảnh chuyển động nhanh]
Chất lượng: [Quá tốt, lãng phí] [Vừa đủ] [Chất lượng kém hơn]
```

- **Ưu điểm:** Dễ dự đoán băng thông, phù hợp broadcast cố định
- **Nhược điểm:** Lãng phí bitrate ở cảnh đơn giản; kém chất lượng ở cảnh phức tạp
- **Dùng cho:** Broadcast truyền thống, truyền qua vệ tinh, CDN cũ cần bitrate cố định

### 2.2 VBR — Variable Bitrate (Tốc Độ Bit Biến Đổi)

```
Bitrate: ───╮    ╭──╮          ╭───╮
            │    │  │          │   │
            ╰────╯  ╰──────────╯   ╰── (thay đổi theo độ phức tạp)
Chất lượng: [Đều, nhất quán ở mọi cảnh]
```

- **Ưu điểm:** Chất lượng nhất quán, file nhỏ hơn CBR
- **Nhược điểm:** Bitrate không dự đoán được, buffer cần đủ lớn
- **Cài đặt:** `MaxBitrate` (bitrate tối đa), `Bitrate` (bitrate mục tiêu)

### 2.3 QVBR — Quality-Defined Variable Bitrate (Tốc Độ Bit Biến Đổi Theo Chất Lượng)

**Đây là mode được AWS khuyến nghị cho streaming VOD.**

```
QVBR logic:
  - Cảnh đơn giản (tĩnh, ít chi tiết) → dùng bitrate thấp
  - Cảnh phức tạp (chuyển động, chi tiết cao) → tăng bitrate
  - Luôn giữ Quality Level mục tiêu, không quan tâm bitrate cụ thể

Kết quả:
  - Chất lượng đồng đều hơn VBR
  - File nhỏ hơn CBR ~40–60%
  - Chất lượng tốt hơn CBR cùng kích thước file
```

**Cài đặt QVBR:**
```json
"H264Settings": {
  "RateControlMode": "QVBR",
  "MaxBitrate": 5000000,        // Bitrate tối đa (bps) — giới hạn trên
  "QvbrSettings": {
    "QvbrQualityLevel": 8,      // Mức chất lượng: 1–10 (7–9 là best practice)
    "QvbrQualityLevelFineTune": 0.0  // Tinh chỉnh nhỏ: -0.5 đến +0.5
  }
}
```

**Chọn QvbrQualityLevel:**

| Mức | Ý Nghĩa | Use Case |
|-----|---------|---------|
| 6–7 | Chất lượng thấp | Mobile, internet chậm |
| **8** | **Cân bằng tốt nhất** | **Streaming VOD tiêu chuẩn** |
| 9 | Chất lượng cao | Premium content, 4K |
| 10 | Gần lossless | Archive (không dùng cho streaming) |

### 2.4 MULTIPLEX — Ghép Nhiều Luồng (ít phổ biến hơn)

Dùng trong môi trường broadcast MPEG-2 TS khi cần ghép nhiều kênh trong 1 multiplex. Không phổ biến cho VOD streaming.

### So Sánh Rate Control Modes

| Mode | Chất Lượng | File Size | Dự Đoán BW | Dùng Cho |
|------|-----------|-----------|-----------|---------|
| **CBR** | Không đồng đều | Lớn nhất | ✅ Dễ dự đoán | Broadcast, live |
| **VBR** | Tốt | Trung bình | ❌ Khó | VOD (cũ) |
| **QVBR** | Tốt nhất | Nhỏ nhất | ❌ Khó | **VOD streaming** |

---

## 3. H.264 / AVC — Advanced Video Coding

### Thông Số Cơ Bản

```json
{
  "CodecSettings": {
    "Codec": "H_264",
    "H264Settings": {
      "RateControlMode": "QVBR",
      "MaxBitrate": 5000000,
      "QvbrSettings": { "QvbrQualityLevel": 8 },

      "CodecProfile": "HIGH",         // Baseline / Main / High
      "CodecLevel": "AUTO",           // AUTO để MediaConvert tự chọn

      "FramerateControl": "INITIALIZE_FROM_SOURCE",  // Giữ nguyên framerate gốc
      "GopSize": 2.0,                 // GOP — Group of Pictures: 2 giây (thường dùng)
      "GopSizeUnits": "SECONDS",

      "SceneChangeDetect": "ENABLED", // Tự động thêm I-frame khi chuyển cảnh
      "AdaptiveQuantization": "HIGH", // Tối ưu phân bổ bits trong frame
      "EntropyEncoding": "CABAC",     // CABAC — Context-Adaptive Binary Arithmetic Coding: nén tốt hơn
      "NumberReferenceFrames": 3,
      "NumberBFramesBetweenReferenceFrames": 2,

      "FieldEncoding": "PAFF",        // PAFF — Picture Adaptive Frame Field: cho interlaced content
      "Telecine": "NONE",
      "InterlaceMode": "PROGRESSIVE"  // PROGRESSIVE — progressive scan (không interlaced)
    }
  }
}
```

### Profile H.264

**Profile — Cấu Hình** xác định tập tính năng codec được dùng:

| Profile | B-frames | CABAC | 8x8 Transform | Use Case |
|---------|----------|-------|---------------|---------|
| **BASELINE** | ❌ | ❌ | ❌ | Mobile cũ, low-power, conferencing |
| **MAIN** | ✅ | ✅ | ❌ | Mobile hiện đại, web streaming |
| **HIGH** | ✅ | ✅ | ✅ | **OTT, Blu-ray, Netflix, YouTube** |
| **HIGH 10** | ✅ | ✅ | ✅ | 10-bit color depth (HDR) |

> **Trong production VOD, luôn dùng HIGH profile** trừ khi có yêu cầu đặc biệt hỗ trợ thiết bị rất cũ.

### Level H.264

**Level — Mức** giới hạn độ phức tạp (resolution, bitrate, buffer):

| Level | Max Resolution | Max Bitrate | Use Case |
|-------|---------------|-------------|---------|
| 3.0 | 720×480 @ 30fps | 10 Mbps | SD content |
| 3.1 | 1280×720 @ 30fps | 14 Mbps | 720p streaming |
| **4.0** | **1920×1080 @ 30fps** | **20 Mbps** | **1080p streaming** |
| 4.1 | 1920×1080 @ 30fps | 50 Mbps | 1080p high-motion |
| 5.0 | 3840×2160 @ 30fps | 135 Mbps | 4K |
| 5.1 | 3840×2160 @ 60fps | 300 Mbps | 4K high-frame-rate |
| `AUTO` | — | — | **Để MediaConvert tự chọn** |

### GOP Settings (Cài Đặt Nhóm Ảnh)

**GOP — Group of Pictures — Nhóm Ảnh** là nhóm các frame từ I-frame đến I-frame tiếp theo:

```
GOP structure (GopSize = 60 frames @ 30fps = 2 giây):
I  B  B  P  B  B  P  B  B  P  B  B  P  B  B  P  B  B  P  I
↑─────────────────────── GOP = 2 giây ─────────────────────↑
```

| Thông Số | Giá Trị Khuyến Nghị | Ý Nghĩa |
|----------|---------------------|---------|
| `GopSize` | **2.0 giây** (VOD streaming) | Thời lượng mỗi GOP |
| `GopSizeUnits` | `SECONDS` | Đơn vị: giây hoặc frame |
| `GopBReference` | `ENABLED` | Cho phép B-frames tham chiếu B-frames khác |
| `GopClosedCadence` | `1` | Đóng GOP sau mỗi segment (quan trọng cho HLS) |
| `SceneChangeDetect` | `ENABLED` | Thêm I-frame tự động khi phát hiện cảnh mới |

---

## 4. H.265 / HEVC — High Efficiency Video Coding

### Tại Sao Dùng H.265?

```
Cùng chất lượng hình ảnh:
  H.264:  5 Mbps cho 1080p
  H.265:  2.5 Mbps cho 1080p (tiết kiệm ~50% bandwidth)

  H.265:  8 Mbps cho 4K UHD
  H.264: 15–20 Mbps cho 4K UHD (H.265 tiết kiệm ~55%)
```

**Đánh đổi (Trade-off):**
- Encode chậm hơn H.264 khoảng 2–3× (tốn CPU/GPU hơn, tốn tiền MediaConvert hơn)
- Decode cần thiết bị hỗ trợ HEVC (iOS 11+, Android 5.0+, modern PC/TV)
- Không hỗ trợ trong MPEG-2 TS — phải dùng fMP4/CMAF

### Cấu Hình H.265

```json
{
  "CodecSettings": {
    "Codec": "H_265",
    "H265Settings": {
      "RateControlMode": "QVBR",
      "MaxBitrate": 8000000,
      "QvbrSettings": { "QvbrQualityLevel": 8 },

      "CodecProfile": "MAIN_10_HIGH",  // MAIN / MAIN_10 / MAIN_422_8BIT / MAIN_422_10BIT
      "CodecLevel": "AUTO",

      "FramerateControl": "INITIALIZE_FROM_SOURCE",
      "GopSize": 2.0,
      "GopSizeUnits": "SECONDS",
      "SceneChangeDetect": "ENABLED",

      "Tiles": "ENABLED",              // Tiles: tăng tốc encode/decode cho 4K
      "UnregisteredSeiTimecode": "DISABLED",
      "WriteMp4PackagingType": "HVC1"  // HVC1 vs HEV1: iOS yêu cầu HVC1
    }
  }
}
```

### H.265 Profile

| Profile | Bit Depth | Chroma Sampling | Use Case |
|---------|-----------|-----------------|---------|
| **MAIN** | 8-bit | 4:2:0 | Standard HD content |
| **MAIN_10** | 10-bit | 4:2:0 | HDR10, Dolby Vision |
| **MAIN_422_8BIT** | 8-bit | 4:2:2 | Professional video |
| **MAIN_422_10BIT** | 10-bit | 4:2:2 | High-end broadcast |

> **HDR — High Dynamic Range — Dải Động Cao:** cần profile MAIN_10 (10-bit color depth). HDR10 và Dolby Vision đều yêu cầu 10-bit.

### Lưu Ý Quan Trọng Cho H.265

```
⚠️ HLS + H.265: cần dùng fMP4 container (không phải MPEG-2 TS)
  "ContainerSettings": { "Container": "M3U8", "M3u8Settings": { ... } }
  → Video codec trong segment phải là fMP4 (fragmented MP4)

⚠️ iOS yêu cầu "WriteMp4PackagingType": "HVC1"
  HVC1 vs HEV1 là 2 cách đặt codec tag trong MP4:
  - HVC1: iOS/Apple yêu cầu
  - HEV1: Android/Windows

⚠️ Không dùng H.265 với MPEG-2 TS container — không tương thích
```

---

## 5. AV1 — AOMedia Video 1

### Tổng Quan AV1

**AV1** được phát triển bởi **Alliance for Open Media — Liên Minh Phương Tiện Mở** (Google, Netflix, Amazon, Apple...) với mục tiêu:
- **Royalty-free** — Miễn phí bản quyền (không như H.264/H.265)
- Hiệu quả nén tốt hơn H.265 khoảng 20–30%
- Hỗ trợ rộng trên Chrome (từ 2018), Firefox, Android, Samsung TV

```
So sánh hiệu quả nén (cùng chất lượng 1080p):
  H.264:  5.0 Mbps
  H.265:  2.5 Mbps  (tiết kiệm 50% so với H.264)
  AV1:    1.7 Mbps  (tiết kiệm 66% so với H.264)
```

### Cấu Hình AV1

```json
{
  "CodecSettings": {
    "Codec": "AV1",
    "Av1Settings": {
      "RateControlMode": "QVBR",
      "MaxBitrate": 3000000,
      "QvbrSettings": { "QvbrQualityLevel": 8 },

      "GopSize": 60,                  // GOP theo frame (AV1 dùng frame, không phải giây)
      "NumberBFramesBetweenReferenceFrames": 15,
      "Slices": 2,                    // Parallel encoding slices (tăng tốc)
      "BitDepth": "BIT_8",            // BIT_8 hoặc BIT_10 (HDR)
      "FilmGrainSynthesis": "DISABLED",
      "FramerateControl": "INITIALIZE_FROM_SOURCE",
      "SpatialAdaptiveQuantization": "ENABLED"  // Tối ưu phân bổ bits theo vùng
    }
  }
}
```

### Hạn Chế Hiện Tại Của AV1

| Hạn Chế | Mô Tả |
|---------|-------|
| **Encode chậm** | Chậm hơn H.264 khoảng 10–20× (tốn kém khi encode MediaConvert) |
| **iOS Safari** | Safari chưa hỗ trợ AV1 đầy đủ (đến iOS 16+ có hardware decode) |
| **Smart TV cũ** | Không hỗ trợ, cần firmware mới |
| **Hardware decode** | Chỉ chip mới (Apple M-series, Intel 11th gen+, NVIDIA Ampere+) |

**Kết luận về AV1:** Lý tưởng cho **modern web (Chrome/Firefox/Android)** và các nền tảng như YouTube, Netflix. Chưa phải lựa chọn chính cho OTT cần hỗ trợ rộng thiết bị cũ.

---

## 6. Bitrate Ladder — Thang Tốc Độ Bit

**Bitrate Ladder — Thang Tốc Độ Bit** là tập hợp các rendition (phiên bản) với độ phân giải và bitrate khác nhau, dùng cho ABR — Adaptive Bitrate Streaming.

### Bitrate Ladder H.264 Tiêu Chuẩn (VOD)

| Rendition | Resolution | Framerate | MaxBitrate (QVBR) | QvbrLevel |
|-----------|-----------|-----------|-------------------|-----------|
| 2160p UHD | 3840×2160 | 30fps | 15,000 Kbps | 9 |
| **1080p FHD** | **1920×1080** | **30fps** | **5,000 Kbps** | **8** |
| 720p HD | 1280×720 | 30fps | 3,000 Kbps | 8 |
| 540p | 960×540 | 30fps | 1,500 Kbps | 7 |
| 480p | 854×480 | 30fps | 1,000 Kbps | 7 |
| 360p | 640×360 | 30fps | 600 Kbps | 7 |
| 240p | 426×240 | 30fps | 300 Kbps | 6 |

> **ABR player** (hls.js, Shaka Player, ExoPlayer) sẽ tự động chọn rendition phù hợp dựa trên bandwidth và thiết bị của viewer.

### Bitrate Ladder H.265 (Tiết Kiệm Bandwidth Hơn ~50%)

| Rendition | Resolution | MaxBitrate (QVBR) | So Sánh H.264 |
|-----------|-----------|-------------------|--------------|
| 4K UHD | 3840×2160 | 8,000 Kbps | H.264 cần ~15,000 Kbps |
| 1080p FHD | 1920×1080 | 2,500 Kbps | H.264 cần ~5,000 Kbps |
| 720p HD | 1280×720 | 1,500 Kbps | H.264 cần ~3,000 Kbps |

### Cấu Hình Resolution Trong MediaConvert

```json
"VideoDescription": {
  "Width": 1920,
  "Height": 1080,
  "ScalingBehavior": "DEFAULT",     // DEFAULT giữ aspect ratio (tỷ lệ khung hình)
  "Sharpness": 50,                  // 0–100: độ sắc nét sau scale
  "AfdSignaling": "NONE",
  "DropFrameTimecode": "ENABLED",
  "RespondToAfd": "NONE",
  "AntiAlias": "ENABLED",           // Chống răng cưa khi scale xuống
  "VideoPreprocessors": {
    "Deinterlacer": {
      "Algorithm": "INTERPOLATE",   // Xử lý interlaced content (video từ camera cũ)
      "Control": "NORMAL",
      "Mode": "DEINTERLACE"
    }
  }
}
```

---

## 7. Cài Đặt Nâng Cao

### Deinterlacing — Khử Quét Xen Kẽ

Video từ camera broadcast truyền thống thường là **interlaced** (quét xen kẽ). Cần deinterlace trước khi streaming:

```json
"VideoPreprocessors": {
  "Deinterlacer": {
    "Mode": "DEINTERLACE",         // DEINTERLACE, INVERSE_TELECINE, ADAPTIVE
    "Algorithm": "INTERPOLATE",    // INTERPOLATE, INTERPOLATE_TICKER, BLEND, BLEND_TICKER
    "Control": "NORMAL"
  }
}
```

| Mode | Dùng Khi |
|------|---------|
| `DEINTERLACE` | Video interlaced thông thường (camera analog/broadcast) |
| `INVERSE_TELECINE` | Film 24fps được telecine thành 29.97fps NTSC |
| `ADAPTIVE` | Tự động phát hiện interlaced/progressive theo scene |

### Color Space Conversion — Chuyển Đổi Không Gian Màu

```json
"VideoDescription": {
  "ColorMetadata": "INSERT",
  "VideoPreprocessors": {
    "ColorCorrector": {
      "ColorSpaceConversion": "FORCE_601",   // Hoặc FORCE_709, FORCE_HDR10, FORCE_HLG_2020
      "Hdr10Metadata": {
        "RedPrimaryX": 17500,
        "RedPrimaryY": 41400,
        // ... HDR10 metadata cho màn hình HDR
        "MaxContentLightLevel": 1000,
        "MaxFrameAverageLightLevel": 200
      }
    }
  }
}
```

| Chuyển Đổi | Ý Nghĩa |
|-----------|---------|
| `FORCE_601` | BT.601 — chuẩn SD (Standard Definition) |
| `FORCE_709` | BT.709 — chuẩn HD (High Definition) |
| `FORCE_HDR10` | HDR10 — HDR với 10-bit, static metadata |
| `FORCE_HLG_2020` | HLG — Hybrid Log-Gamma — chuẩn HDR cho broadcast |

### Image Inserter — Chèn Watermark (Dấu Bản Quyền)

```json
"VideoPreprocessors": {
  "ImageInserter": {
    "InsertableImages": [
      {
        "ImageX": 20,                 // Vị trí X (pixel từ trái)
        "ImageY": 20,                 // Vị trí Y (pixel từ trên)
        "Layer": 1,
        "ImageInserterInput": "s3://my-bucket/watermark-logo.png",
        "Opacity": 50,                // Độ trong suốt: 0–100
        "Width": 200,
        "Height": 80,
        "StartTime": "00:00:00:00",
        "Duration": 0                 // 0 = hiển thị suốt video
      }
    ]
  }
}
```

### Noise Reducer — Giảm Nhiễu

```json
"VideoPreprocessors": {
  "NoiseReducer": {
    "Filter": "TEMPORAL",            // SPATIAL, TEMPORAL, CONSERVE, SOFTNESS, LANCZOS
    "FilterSettings": {
      "Strength": 3                  // 0–3: mức độ giảm nhiễu
    }
  }
}
```

---

## Tóm Tắt: Cấu Hình Codec Cho Từng Scenario

### Scenario 1: VOD Standard (95% trường hợp)

```json
{
  "Codec": "H_264",
  "H264Settings": {
    "RateControlMode": "QVBR",
    "MaxBitrate": 5000000,
    "QvbrSettings": { "QvbrQualityLevel": 8 },
    "CodecProfile": "HIGH",
    "GopSize": 2.0,
    "GopSizeUnits": "SECONDS",
    "SceneChangeDetect": "ENABLED"
  }
}
```

### Scenario 2: Premium 4K HDR

```json
{
  "Codec": "H_265",
  "H265Settings": {
    "RateControlMode": "QVBR",
    "MaxBitrate": 8000000,
    "QvbrSettings": { "QvbrQualityLevel": 9 },
    "CodecProfile": "MAIN_10_HIGH",
    "WriteMp4PackagingType": "HVC1",
    "GopSize": 2.0,
    "GopSizeUnits": "SECONDS"
  }
}
```

### Scenario 3: Archive / ProRes

```json
{
  "Codec": "PRORES",
  "ProresSettings": {
    "CodecProfile": "APPLE_PRORES_422",  // 422 LT, 422, 422 HQ, 4444
    "FramerateControl": "INITIALIZE_FROM_SOURCE",
    "SlowPal": "DISABLED",
    "Telecine": "NONE"
  }
}
```

---

**Phần Tiếp Theo:** [4-audio-captions.md](./4-audio-captions.md) — Multi-track audio và phụ đề
