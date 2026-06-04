# Codec & Container — Nền Tảng Nén và Đóng Gói Video

> Hiểu codec và container là bước đầu tiên bắt buộc để làm việc với bất kỳ hệ thống video nào, từ MediaConvert đến MediaLive.

## 📚 Mục Lục

1. [Codec Video](#1-codec-video)
2. [So Sánh Codec Chi Tiết](#2-so-sánh-codec-chi-tiết)
3. [HDR — High Dynamic Range](#3-hdr--high-dynamic-range)
4. [Codec Audio](#4-codec-audio)
5. [Container — Định Dạng Đóng Gói](#5-container--định-dạng-đóng-gói)
6. [Chọn Codec & Container Cho Use Case](#6-chọn-codec--container-cho-use-case)
7. [Trong AWS MediaConvert](#7-trong-aws-mediaconvert)

---

## 1. Codec Video

### Codec Là Gì?

**Codec** (viết tắt của **Co**der-**Dec**oder — Bộ Mã Hoá/Giải Mã) là thuật toán nén video:
- **Encoder** — Bộ Mã Hoá: chuyển video thô (RAW) thành định dạng nén nhỏ hơn
- **Decoder** — Bộ Giải Mã: tái tạo video từ dữ liệu nén để phát lại

```
Video thô (YUV 4:2:0)   →   [ENCODER H.264]   →   Video nén (bitstream)
 1920×1080 @ 30fps                                   ~5 Mbps
 ~746 Mbps (uncompressed)                           Tiết kiệm 99% dung lượng
```

### Nguyên Lý Nén Video

**Nén video khai thác 2 loại dư thừa (redundancy):**

1. **Spatial Redundancy** — Dư Thừa Không Gian: các pixel liền kề thường có màu tương tự nhau → nén trong cùng một frame (Intra-frame compression)

2. **Temporal Redundancy** — Dư Thừa Thời Gian: các frame liên tiếp thường ít thay đổi → chỉ lưu sự khác biệt giữa các frame (Inter-frame compression)

**Các loại frame (GOP — Group of Pictures — Nhóm Ảnh):**

```
I-frame (Keyframe)   — Intra-coded frame — frame đầy đủ, không phụ thuộc frame khác
P-frame (Predicted)  — Predicted frame — chỉ lưu sự khác biệt so với frame trước
B-frame (Bi-dir)     — Bidirectional frame — tham chiếu cả frame trước lẫn frame sau

GOP structure:  I  B  B  P  B  B  P  B  B  I ...
                ↑─────────────────────────↑
                        GOP = 30 frames (1 giây @ 30fps)
```

> **Lưu ý:** **GOP size** ảnh hưởng đến khả năng seek (tua) và latency. GOP ngắn hơn → seek nhanh hơn, latency thấp hơn nhưng file lớn hơn.

---

## 2. So Sánh Codec Chi Tiết

### H.264 / AVC — Advanced Video Coding

**Tiêu Chuẩn Phổ Biến Nhất Thế Giới**

```yaml
Năm ra đời: 2003
Tổ chức: ITU-T + ISO/IEC (cùng phát triển với MPEG-4 Part 10)
Hiệu quả nén: Chuẩn cơ sở (baseline)
Hardware support: Gần như mọi thiết bị (TV, phone, browser, camera)
License: Có phí (MPEG-LA pool) nhưng được miễn phí cho streaming
```

**Profile (Cấu Hình) H.264:**

| Profile | Tính Năng | Use Case |
|---------|-----------|----------|
| **Baseline** | Cơ bản nhất, không có B-frames | Mobile cũ, low-power devices |
| **Main** | B-frames, CABAC entropy coding | Broadcast, web streaming |
| **High** | 8x8 transform, tốt hơn Main ~10% | Blu-ray, Netflix, YouTube |

**Level (Mức) H.264:**
- Quy định giới hạn: resolution tối đa, bitrate tối đa, buffer size
- Level 4.0: tối đa 1080p@30fps, 20 Mbps — phổ biến nhất
- Level 5.1: tối đa 4K@30fps, 300 Mbps — cho 4K content

**Khi Nào Dùng H.264:**
- ✅ Cần hỗ trợ thiết bị rộng nhất (smart TV, set-top box cũ)
- ✅ Real-time encoding hardware (MediaLive default)
- ✅ Content 1080p trở xuống với bandwidth hợp lý
- ❌ Không nên: khi muốn tối ưu bandwidth cho 4K content

---

### H.265 / HEVC — High Efficiency Video Coding

**Thế Hệ Kế Tiếp, Hiệu Quả Hơn H.264 ~50%**

```yaml
Năm ra đời: 2013
Tổ chức: ITU-T + ISO/IEC
Hiệu quả nén: ~50% tốt hơn H.264 ở cùng chất lượng
Hardware support: Tốt trên thiết bị từ 2015 trở đi
License: Phức tạp, nhiều patent pool (MPEG-LA, HEVC Advance, Velos Media)
```

**So sánh thực tế:**
```
1080p video @ chất lượng tương đương:
  H.264: 5 Mbps
  H.265: 2.5 Mbps  ← tiết kiệm 50% bandwidth

4K video @ chất lượng tốt:
  H.264: 25 Mbps
  H.265: 12 Mbps  ← tiết kiệm bandwidth đáng kể
```

**Profile H.265:**
- **Main**: 8-bit, 4:2:0 — phổ biến nhất
- **Main 10**: 10-bit màu — cho HDR content
- **Main Still Picture**: ảnh tĩnh
- **Main 4:4:4**: full chroma — professional production

**Khi Nào Dùng H.265:**
- ✅ 4K / UHD content (bắt buộc)
- ✅ HDR content (HDR10, Dolby Vision)
- ✅ Khi muốn giảm CDN bandwidth cost cho large-scale platform
- ✅ Khán giả dùng thiết bị từ 2016 trở đi
- ❌ Không nên: encode real-time trên hardware yếu (tốn CPU ~4x H.264)

---

### AV1 — AOMedia Video 1

**Codec Mở, Miễn Phí, Hiệu Quả Nhất Hiện Tại**

```yaml
Năm ra đời: 2018
Tổ chức: Alliance for Open Media (Google, Netflix, Amazon, Apple, Microsoft...)
Hiệu quả nén: ~30% tốt hơn H.265 / ~50% tốt hơn H.264
Hardware support: Đang mở rộng (2020+ GPU/CPU)
License: Hoàn toàn miễn phí, không patent
```

**Tại Sao AV1 Ra Đời:**
- H.265 có vấn đề về license phức tạp và tốn kém
- Google (YouTube), Netflix, Amazon muốn tránh phí bản quyền
- AV1 được thiết kế để thay thế H.265 trong streaming dài hạn

**Hiệu Quả AV1 Thực Tế (Netflix nghiên cứu):**
```
Cùng chất lượng VMAF 80:
  H.264:  3.5 Mbps
  H.265:  2.2 Mbps (-37%)
  AV1:    1.5 Mbps (-57% vs H.264, -32% vs H.265)
```

**Hạn Chế AV1:**
- Encode chậm hơn H.265 rất nhiều (10-50x với software encoder)
- Hardware encoder vẫn đang phát triển (NVIDIA RTX 40xx, Intel Arc hỗ trợ)
- Chưa phổ biến trên smart TV cũ và set-top box

**Khi Nào Dùng AV1:**
- ✅ VOD pre-encoding (không real-time) — YouTube, Netflix
- ✅ Muốn tiết kiệm CDN cost tối đa cho large library
- ✅ Target thiết bị modern (Chrome 70+, Android 10+, Samsung 2020+)
- ❌ Không phù hợp: live encoding real-time (chưa đủ hardware support)

---

### VP9 — Video Processor 9

```yaml
Năm ra đời: 2013
Tổ chức: Google
Hiệu quả nén: Tương đương H.265
License: Miễn phí (Google)
Use case: YouTube (cả VP9 và AV1), Chromebook
```

VP9 là "người tiền nhiệm" của AV1 từ Google. Hiện YouTube vẫn dùng VP9 cho nhiều content, nhưng đang dần chuyển sang AV1.

---

### Bảng So Sánh Tổng Hợp

| Tiêu Chí | H.264 | H.265 | AV1 | VP9 |
|----------|-------|-------|-----|-----|
| **Năm ra đời** | 2003 | 2013 | 2018 | 2013 |
| **Hiệu quả nén** | Cơ sở | +50% | +80% | +50% |
| **License** | Có phí | Phức tạp | Miễn phí | Miễn phí |
| **Encode speed** | Nhanh | Vừa | Chậm | Vừa |
| **Device support** | Rộng nhất | Tốt | Đang mở rộng | Chrome/Android |
| **AWS MediaConvert** | ✅ | ✅ | ✅ | ✅ |
| **AWS MediaLive** | ✅ | ✅ | ❌ | ❌ |
| **Best for** | Compatibility | 4K, HDR | VOD tối ưu | YouTube |

---

## 3. HDR — High Dynamic Range

**HDR** — High Dynamic Range — Dải Tương Phản Động Cao: công nghệ hiển thị màu sắc và độ sáng vượt trội so với SDR — Standard Dynamic Range — Dải Tương Phản Động Tiêu Chuẩn.

```
SDR (Standard):  100 nits peak brightness, 8-bit color (16.7 triệu màu)
HDR (High):     1000-10000 nits peak brightness, 10-bit color (1 tỷ màu)
```

### Các Chuẩn HDR:

| Chuẩn | Bit Depth | Metadata | Hỗ Trợ | AWS MediaConvert |
|-------|-----------|----------|---------|-----------------|
| **HDR10** | 10-bit | Static | Rộng nhất (Samsung, LG, Sony) | ✅ |
| **HDR10+** | 10-bit | Dynamic (per-scene) | Samsung, Amazon | ✅ |
| **Dolby Vision** | 12-bit | Dynamic (per-frame) | Apple, LG, Roku | ✅ |
| **HLG** — Hybrid Log-Gamma | 10-bit | Không cần | Broadcast (BBC, NHK) | ✅ |

**Static vs Dynamic Metadata:**
- **Static metadata**: thông tin HDR áp dụng cho toàn bộ video (HDR10)
- **Dynamic metadata**: thông tin HDR thay đổi theo từng cảnh/frame (HDR10+, Dolby Vision) → chất lượng tốt hơn

**Color Space — Không Gian Màu:**
- **Rec.709** — SDR standard (HDTV)
- **Rec.2020** — HDR standard, dải màu rộng hơn nhiều
- **DCI-P3** — Digital Cinema Initiatives, dùng trong rạp chiếu phim

---

## 4. Codec Audio

### AAC — Advanced Audio Coding

```yaml
Tiêu chuẩn: ISO/IEC 14496-3 (MPEG-4 Audio)
Bitrate phổ biến: 128 Kbps (stereo), 192-256 Kbps (high quality)
Channels: Stereo (2.0), 5.1 surround
Use case: Streaming phổ biến nhất — YouTube, Spotify, Apple Music
AWS support: MediaConvert (AAC), MediaLive (AAC)
```

**AAC Profile:**
- **AAC-LC** — Low Complexity: phổ biến nhất, hiệu quả tốt
- **HE-AAC v1** — High Efficiency: dùng SBR, tốt ở bitrate thấp (< 80 Kbps)
- **HE-AAC v2**: thêm PS (Parametric Stereo), tốt ở 24-48 Kbps

### Dolby Digital (AC-3) và Dolby Digital Plus (E-AC-3)

```yaml
AC-3 (Dolby Digital):
  - Bitrate: 192-640 Kbps
  - Channels: tối đa 5.1
  - Use case: Blu-ray, DVD, broadcast TV (Mỹ)

E-AC-3 (Dolby Digital Plus / Dolby Atmos):
  - Bitrate: 32 Kbps - 6 Mbps
  - Channels: tối đa 7.1.4 (Atmos)
  - Use case: Netflix 4K, Amazon Prime, Apple TV+
  - AWS MediaConvert: ✅ hỗ trợ E-AC-3 (Dolby Atmos passthrough/encode)
```

### So Sánh Audio Codec

| Codec | Bitrate Tốt | Channels | License | Use Case |
|-------|------------|----------|---------|----------|
| **AAC-LC** | 128-256 Kbps | ≤ 5.1 | Có phí nhưng rẻ | Web streaming, mobile |
| **HE-AAC** | 48-128 Kbps | ≤ 5.1 | Có phí | Mobile low bandwidth |
| **AC-3** | 192-448 Kbps | ≤ 5.1 | Dolby license | Broadcast, disc |
| **E-AC-3** | 96 Kbps - 6 Mbps | ≤ 7.1.4 | Dolby license | Premium streaming |
| **MP3** | 128-320 Kbps | Stereo | Hết hạn patent | Legacy, music |

---

## 5. Container — Định Dạng Đóng Gói

### Tại Sao Cần Container?

Container giải quyết bài toán: làm sao đóng gói **video + audio + phụ đề + metadata** vào cùng một file/stream và đảm bảo chúng phát đồng bộ?

```
Container = "hộp" chứa nhiều track:
┌─────────────────────────────────────────┐
│  Container (MP4)                        │
│  ├─ Video track: H.264 @ 5 Mbps        │
│  ├─ Audio track 1: AAC EN @ 192 Kbps   │
│  ├─ Audio track 2: AAC VI @ 192 Kbps   │
│  ├─ Subtitle: WebVTT (EN)              │
│  └─ Metadata: duration, chapters...    │
└─────────────────────────────────────────┘
```

### MP4 (MPEG-4 Part 14)

```yaml
Extension: .mp4, .m4v, .m4a (audio only)
Codec support: H.264, H.265, AV1, AAC, MP3, ...
Streaming: Progressive download hoặc fragmented MP4
Use case: VOD, local file, mobile download
```

**MP4 Fragmented (fMP4):**
- Cấu trúc MP4 thông thường có moov atom ở cuối → không stream được khi đang download
- **fMP4** — Fragmented MP4: chia thành các fragment độc lập → hỗ trợ streaming, adaptive bitrate
- CMAF và DASH đều dùng fMP4 làm container cho segment

### HLS — HTTP Live Streaming

```yaml
Tạo bởi: Apple (2009)
Extension: .m3u8 (manifest), .ts hoặc .fmp4 (segments)
Codec video: H.264, H.265 (HEVC)
Codec audio: AAC, AC-3, E-AC-3
Adaptive bitrate: ✅ (multiple renditions)
DRM: FairPlay (Apple), AES-128 encryption
```

**Cấu Trúc HLS:**
```
Master Playlist (index.m3u8)
├── Rendition 1: 1080p 5Mbps (1080p.m3u8)
│   ├── segment_000.ts
│   ├── segment_001.ts
│   └── ...
├── Rendition 2: 720p 2.5Mbps (720p.m3u8)
│   ├── segment_000.ts
│   └── ...
└── Rendition 3: 360p 800Kbps (360p.m3u8)
    └── ...
```

**HLS Master Playlist mẫu:**
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720,CODECS="avc1.64001f,mp4a.40.2"
720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360,CODECS="avc1.42001e,mp4a.40.2"
360p/index.m3u8
```

### DASH — Dynamic Adaptive Streaming over HTTP

```yaml
Tạo bởi: ISO/IEC (chuẩn mở, 2012)
Extension: .mpd (manifest — Media Presentation Description), .m4s (segment)
Codec video: H.264, H.265, AV1, VP9 — linh hoạt hơn HLS
Codec audio: AAC, AC-3, E-AC-3
Adaptive bitrate: ✅
DRM: Widevine, PlayReady (qua Common Encryption — CENC)
```

**Điểm Khác Biệt HLS vs DASH:**

| Tiêu Chí | HLS | DASH |
|----------|-----|------|
| **Tạo bởi** | Apple (proprietary) | ISO (open standard) |
| **Manifest** | .m3u8 (text) | .mpd (XML) |
| **Segment** | .ts hoặc .fmp4 | .m4s (fMP4) |
| **iOS/Safari** | ✅ Native | ❌ Cần thư viện JS |
| **Android/Chrome** | ✅ (thư viện) | ✅ Native |
| **DRM** | FairPlay | Widevine + PlayReady |
| **Codec linh hoạt** | Hạn chế | Rất linh hoạt |

### CMAF — Common Media Application Format

```yaml
Tạo bởi: Apple + Microsoft (2016), chuẩn hoá ISO 2018
Mục tiêu: Giải quyết dual-packaging problem (phải tạo cả HLS lẫn DASH)
Segment format: fMP4 (dùng chung cho cả HLS và DASH)
DRM: CBCS (Cipher Block Chaining with Pattern Encryption)
```

**Vấn Đề CMAF Giải Quyết:**
```
TRƯỚC CMAF (Dual Packaging):
Encoder → MediaPackage → HLS segments (.ts)    → Apple devices
                       → DASH segments (.m4s)  → Android, Smart TV
                       → Lưu trữ 2x, xử lý 2x

SAU CMAF (Single Packaging):
Encoder → MediaPackage → CMAF chunks (.cmf4)
                       → HLS manifest (.m3u8) ─┐  Cùng segment
                       → DASH manifest (.mpd) ─┘  dùng chung
```

**Tại Sao CMAF Quan Trọng với AWS:**
- **MediaPackage** hỗ trợ CMAF endpoint → tiết kiệm storage và CDN cost
- **LL-CMAF** — Low-Latency CMAF: chunked transfer encoding giảm latency xuống 2-5 giây

### MPEG-TS — MPEG Transport Stream

```yaml
Extension: .ts
Use case: Broadcast TV, satellite, cable, MediaLive output
Đặc điểm: Chịu lỗi tốt (error resilience), thiết kế cho môi trường không ổn định
Không dùng cho: Progressive download, adaptive streaming hiện đại
```

---

## 6. Chọn Codec & Container Cho Use Case

### Decision Tree — Sơ Đồ Quyết Định

```
Cần encode video cho mục đích gì?
│
├── VOD (Video on Demand)?
│   ├── Target thiết bị cũ (2014 trở về trước)?
│   │   └── H.264 + MP4/HLS
│   ├── 4K / HDR content?
│   │   └── H.265 + CMAF (HLS+DASH)
│   └── Muốn tối ưu bandwidth tối đa?
│       └── AV1 (hoặc H.265 nếu cần real-time)
│
├── Live Streaming?
│   ├── Cần phủ rộng nhất?
│   │   └── H.264 + HLS
│   ├── 4K live stream?
│   │   └── H.265 + HLS/DASH
│   └── Real-time encoding hardware?
│       └── H.264 hoặc H.265 (AV1 chưa đủ hardware)
│
└── Interactive / Low-latency?
    └── H.264 + WebRTC hoặc LL-HLS
```

### Bảng Use Case Thực Tế

| Use Case | Video Codec | Container | Audio | Ghi Chú |
|----------|------------|-----------|-------|---------|
| **Netflix VOD** | H.264, H.265, AV1 | CMAF | AAC, E-AC-3 | Multi-codec per device |
| **YouTube** | H.264, VP9, AV1 | DASH, HLS | AAC, Opus | AV1 cho modern browsers |
| **Live Sports HD** | H.264 | HLS | AAC | Max compatibility |
| **Live Sports 4K** | H.265 | HLS, DASH | E-AC-3 | HDR thêm giá trị |
| **Livestream (IVS)** | H.264 | HLS | AAC | Low-latency mode |
| **Video Conference** | H.264 | WebRTC (RTP) | Opus | Realtime < 200ms |
| **OTT Platform** | H.264 + H.265 | CMAF | AAC + AC-3 | Dual codec, single package |

---

## 7. Trong AWS MediaConvert

### Các Codec AWS MediaConvert Hỗ Trợ

**Video:**
- H.264 (AVC)
- H.265 (HEVC) — kể cả HDR10, HDR10+, Dolby Vision
- AV1
- VP9
- MPEG-2 (legacy broadcast)
- ProRes (Apple ProRes — professional production)
- Frame Capture (thumbnail extraction)

**Audio:**
- AAC (LC, HE-AAC v1, HE-AAC v2)
- Dolby Digital (AC-3)
- Dolby Digital Plus (E-AC-3) — kể cả Dolby Atmos
- MP3
- WAV/PCM (uncompressed)
- AIFF

### Ví Dụ Cấu Hình MediaConvert Job

```json
{
  "Settings": {
    "OutputGroups": [
      {
        "Name": "Apple HLS",
        "OutputGroupSettings": {
          "Type": "HLS_GROUP_SETTINGS",
          "HlsGroupSettings": {
            "SegmentLength": 6,
            "Destination": "s3://my-bucket/hls/"
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
                "CodecSettings": {
                  "Codec": "AAC",
                  "AacSettings": {
                    "Bitrate": 192000,
                    "SampleRate": 48000,
                    "CodingMode": "CODING_MODE_2_0"
                  }
                }
              }
            ]
          }
        ]
      }
    ]
  }
}
```

**QVBR** — Quality-Defined Variable Bitrate — Tốc Độ Bit Thay Đổi Theo Chất Lượng: AWS khuyến nghị dùng QVBR thay vì CBR để giảm chi phí storage/CDN trong khi duy trì chất lượng nhất quán.

---

## ❓ Câu Hỏi Ôn Tập

1. Tại sao AV1 có hiệu quả nén tốt hơn H.265 nhưng chưa được dùng rộng rãi cho live streaming?
2. CMAF giải quyết vấn đề gì so với việc dùng HLS và DASH riêng lẻ?
3. Tại sao HDR10+ và Dolby Vision tốt hơn HDR10 mặc dù cùng 10-bit?
4. HLS dùng `.ts` và `.fmp4` làm segment — sự khác biệt là gì? Cái nào hiện đại hơn?
5. Trong MediaConvert, tại sao AWS khuyến nghị QVBR thay vì CBR?

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---------|----------|-------------|
| [README.md](./README.md) | **1-codec-container.md** | [2-streaming-protocols.md](./2-streaming-protocols.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn thành
