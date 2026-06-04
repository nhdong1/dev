# Encoding Settings — Cài Đặt Mã Hoá Video & Âm Thanh

> Encoding Settings quyết định chất lượng, băng thông và chi phí của live stream. Hiểu rõ các thông số này giúp thiết kế bitrate ladder tối ưu cho từng use case.

## 📚 Mục Lục

1. [Video Description — Mô Tả Đầu Ra Video](#1-video-description--mô-tả-đầu-ra-video)
2. [H.264 (AVC) Settings](#2-h264-avc-settings)
3. [H.265 (HEVC) Settings](#3-h265-hevc-settings)
4. [Bitrate Ladder — Thang Tốc Độ Bit](#4-bitrate-ladder--thang-tốc-độ-bit)
5. [Audio Description — Cài Đặt Âm Thanh](#5-audio-description--cài-đặt-âm-thanh)
6. [Audio Selector — Chọn Track Âm Thanh](#6-audio-selector--chọn-track-âm-thanh)
7. [Caption — Phụ Đề](#7-caption--phụ-đề)
8. [Video Preprocessing — Tiền Xử Lý Video](#8-video-preprocessing--tiền-xử-lý-video)
9. [Ví Dụ Cấu Hình Thực Tế](#9-ví-dụ-cấu-hình-thực-tế)

---

## 1. Video Description — Mô Tả Đầu Ra Video

**Video Description** là cấu hình cho mỗi rendition (phiên bản) đầu ra video. Mỗi Output trong Channel có thể dùng một Video Description khác nhau.

### Các Thông Số Cơ Bản

```
Video Description
├── Name                — Tên định danh (dùng để Output tham chiếu)
├── Width / Height      — Độ phân giải đầu ra (pixels)
├── Respond To AFC      — Tự động điều chỉnh theo AFC signal
├── Sharpness           — Độ sắc nét (0–100, mặc định 50)
├── Scaling Behavior    — Cách scale nếu tỷ lệ khung hình khác (Stretch/Letterbox)
└── CodecSettings       — Cài đặt codec (H.264 hoặc H.265)
```

### Tỷ Lệ Khung Hình (Aspect Ratio) Phổ Biến

| Resolution | Aspect Ratio | Dùng Cho |
|-----------|-------------|---------|
| 1920×1080 | 16:9 | HDTV, streaming HD chuẩn |
| 1280×720  | 16:9 | HD thấp, mobile |
| 960×540   | 16:9 | Trung bình |
| 640×360   | 16:9 | Mobile thấp, băng thông hạn chế |
| 480×270   | 16:9 | Ultra-low bandwidth |
| 1920×1080 → 3840×2160 | 16:9 | UHD/4K (H.265 bắt buộc) |

---

## 2. H.264 (AVC) Settings

**H.264** — Advanced Video Coding — Mã Hoá Video Tiên Tiến là codec phổ biến nhất cho live streaming. Cân bằng tốt giữa chất lượng, băng thông và khả năng tương thích thiết bị.

### Các Thông Số Quan Trọng

#### Rate Control Mode (Chế Độ Kiểm Soát Tốc Độ Bit)

| Mode | Mô Tả | Khi Nào Dùng |
|------|-------|-------------|
| **CBR** — Constant Bitrate — Tốc Độ Bit Không Đổi | Bitrate cố định, không đổi | CDN yêu cầu bitrate cố định, broadcast chuyên nghiệp |
| **VBR** — Variable Bitrate — Tốc Độ Bit Biến Đổi | Bitrate thay đổi trong phạm vi max/min | Cân bằng chất lượng và bandwidth |
| **QVBR** — Quality-defined VBR — VBR Theo Chất Lượng | Ưu tiên chất lượng, bitrate tự điều chỉnh | VOD (không lý tưởng cho live do ABR) |

> **Khuyến nghị cho live streaming:** Dùng **CBR** hoặc **VBR** (không dùng QVBR vì QVBR có thể tạo ra bitrate không đồng đều, gây vấn đề với HLS segment sizing).

#### Profile & Level

```
Profile (mức tính năng codec):
  Baseline  — Tương thích tối đa (thiết bị cũ, low-power)
  Main      — Cân bằng (hầu hết thiết bị hiện đại)
  High      — Chất lượng tốt nhất H.264, yêu cầu decoder mạnh hơn
  High 10   — 10-bit color (HDR)

Level (giới hạn thông số kỹ thuật — liên quan đến resolution + framerate + bitrate):
  4.0  — max 1080p@30fps hoặc 720p@60fps
  4.1  — max 1080p@30fps, bitrate cao hơn
  4.2  — max 1080p@60fps
  5.0  — max 1080p@120fps / 2160p@30fps
  5.1  — max 1080p@120fps / 2160p@30fps, bitrate cao hơn
```

#### GOP (Group of Pictures — Nhóm Hình Ảnh)

**GOP** định nghĩa khoảng cách giữa các keyframe (I-frame). Với HLS, chiều dài GOP thường bằng chiều dài HLS segment:

```
Ví dụ: HLS segment = 6 giây, framerate = 30fps
→ GOP size = 6 × 30 = 180 frames

GOP cấu trúc:
I B B P B B P B B P ... (6 giây) ... I B B P B B P ...
^                                     ^
Keyframe (I-frame)                    Keyframe tiếp theo
Segment bắt đầu từ đây               Segment mới bắt đầu
```

**Quan trọng:** Mỗi HLS segment PHẢI bắt đầu bằng I-frame. Nếu GOP size không khớp với segment length → MediaPackage không thể cắt segment sạch → chất lượng kém hoặc seek lỗi.

**Cài đặt thực tế:**
- `GopSize`: 60–180 frames (2–6 giây tùy framerate)
- `GopClosedCadence`: `1` (đóng GOP — mỗi I-frame là điểm seek an toàn)
- `GopSizeUnits`: `FRAMES` hoặc `SECONDS`

#### Cấu Hình H.264 Đầy Đủ (Ví Dụ)

```json
{
  "H264Settings": {
    "RateControlMode": "CBR",
    "Bitrate": 3000000,
    "FramerateDenominator": 1,
    "FramerateNumerator": 30,
    "GopSize": 60,
    "GopSizeUnits": "FRAMES",
    "GopClosedCadence": 1,
    "Profile": "HIGH",
    "Level": "H264_LEVEL_4_1",
    "ColorMetadata": "INSERT",
    "EntropyCoding": "CABAC",
    "FlickerAq": "ENABLED",
    "SpatialAq": "ENABLED",
    "TemporalAq": "ENABLED",
    "SceneChangeDetect": "ENABLED",
    "AdaptiveQuantization": "HIGH",
    "ScanType": "PROGRESSIVE",
    "NumRefFrames": 3,
    "ParControl": "SPECIFIED",
    "ParNumerator": 1,
    "ParDenominator": 1
  }
}
```

---

## 3. H.265 (HEVC) Settings

**H.265** — High Efficiency Video Coding — Mã Hoá Video Hiệu Năng Cao (còn gọi là HEVC) cho phép đạt chất lượng tương đương H.264 với **bitrate thấp hơn ~50%**, hoặc chất lượng tốt hơn ở cùng bitrate.

### H.265 vs H.264

| Tiêu Chí | H.264 | H.265 |
|---------|-------|-------|
| **Hiệu quả nén** | Chuẩn | ~50% tốt hơn H.264 |
| **CPU encode** | Thấp hơn | Cao hơn ~2–5× |
| **Chi phí MediaLive** | Thấp hơn | Cao hơn |
| **Tương thích** | Hầu hết mọi thiết bị | Thiết bị đời mới (2016+) |
| **Use case** | HD streaming phổ thông | UHD/4K, tiết kiệm bandwidth |

### Khi Nào Dùng H.265?

- **UHD/4K content**: H.264 không đủ hiệu quả cho 4K (cần quá nhiều bitrate)
- **Tiết kiệm CDN bandwidth**: với lượng viewer lớn, giảm 50% bitrate = giảm 50% CDN cost
- **Mobile với data plan hạn chế**: chất lượng cao hơn với ít data hơn

```json
{
  "H265Settings": {
    "RateControlMode": "CBR",
    "Bitrate": 6000000,
    "FramerateDenominator": 1,
    "FramerateNumerator": 30,
    "GopSize": 60,
    "GopClosedCadence": 1,
    "Profile": "MAIN",
    "Level": "H265_LEVEL_4_1",
    "Tier": "HIGH",
    "ColorMetadata": "INSERT",
    "SceneChangeDetect": "ENABLED",
    "AdaptiveQuantization": "AUTO",
    "FlickerAq": "ENABLED"
  }
}
```

---

## 4. Bitrate Ladder — Thang Tốc Độ Bit

**Bitrate Ladder** là tập hợp các rendition (phiên bản) với độ phân giải và bitrate giảm dần. ABR — Adaptive Bitrate Streaming player sẽ tự động chọn rendition phù hợp với tốc độ mạng của viewer.

### Nguyên Tắc Thiết Kế Bitrate Ladder

```
Không nên encode 1080p @ 500 Kbps (quá thấp cho resolution đó)
Không nên encode 360p @ 5 Mbps (lãng phí bandwidth)

Mỗi rendition cần:
  bitrate     ≥ "bitrate tối thiểu cho resolution đó"
  resolution  ≤ "resolution tối đa hợp lý cho bitrate đó"
```

### Bitrate Ladder Mẫu Cho Live Streaming

#### H.264 — Thể Thao / Nội Dung Chuyển Động Nhanh

```
Rendition    Resolution    Bitrate     Audio     Tổng
──────────   ──────────    ───────     ─────     ──────
1080p60      1920×1080     6,000 Kbps  192 Kbps  6,192 Kbps
720p60       1280×720      3,500 Kbps  128 Kbps  3,628 Kbps
720p30       1280×720      2,500 Kbps  128 Kbps  2,628 Kbps
480p30        854×480      1,200 Kbps  128 Kbps  1,328 Kbps
360p30        640×360        700 Kbps   96 Kbps    796 Kbps
240p30        426×240        350 Kbps   64 Kbps    414 Kbps
```

#### H.264 — Tin Tức / Nội Dung Ít Chuyển Động

```
Rendition    Resolution    Bitrate     Audio     Tổng
──────────   ──────────    ───────     ─────     ──────
1080p30      1920×1080     4,000 Kbps  192 Kbps  4,192 Kbps
720p30       1280×720      2,000 Kbps  128 Kbps  2,128 Kbps
480p30        854×480        800 Kbps  128 Kbps    928 Kbps
360p30        640×360        400 Kbps   96 Kbps    496 Kbps
240p30        426×240        200 Kbps   64 Kbps    264 Kbps
```

#### H.265 — UHD/4K Live Event

```
Rendition    Resolution    Bitrate     Audio     Tổng
──────────   ──────────    ───────     ─────     ──────
4K30        3840×2160     12,000 Kbps  192 Kbps 12,192 Kbps
1080p60     1920×1080      5,000 Kbps  192 Kbps  5,192 Kbps
720p30      1280×720       2,500 Kbps  128 Kbps  2,628 Kbps
480p30       854×480       1,000 Kbps  128 Kbps  1,128 Kbps
```

### Quy Tắc Vàng Thiết Kế Bitrate Ladder

```
1. Khoảng cách bitrate giữa các rendition: 40–60%
   (tránh khoảng cách quá nhỏ → player chuyển liên tục; quá lớn → trải nghiệm kém)

2. Resolution jump hợp lý: 720p → 480p → 360p (tránh 1080p → 270p)

3. Số rendition tối ưu: 4–6 rendition cho web/mobile
   (quá nhiều → tốn CPU encode; quá ít → trải nghiệm kém khi mạng xấu)

4. Framerate nhất quán hoặc factor 2:
   OK:    30fps, 30fps, 30fps
   OK:    60fps, 30fps, 30fps
   TRÁNH: 30fps, 25fps, 20fps (không đồng đều)

5. Audio bitrate không cần giảm nhiều:
   Video ảnh hưởng đến bandwidth hơn audio nhiều
   64–128 Kbps AAC là đủ tốt cho hầu hết use case
```

---

## 5. Audio Description — Cài Đặt Âm Thanh

**Audio Description** định nghĩa cách encode mỗi track âm thanh đầu ra.

### Codec Âm Thanh Được Hỗ Trợ

| Codec | Mô Tả | Bitrate Thông Dụng | Dùng Cho |
|-------|-------|-------------------|---------|
| **AAC** — Advanced Audio Coding | Phổ biến nhất, tương thích cao | 64–320 Kbps | Web, mobile, OTT |
| **AC-3** — Dolby Digital | Âm thanh vòm 5.1 | 192–448 Kbps | Smart TV, set-top box |
| **EAC-3** — Dolby Digital Plus | Cải tiến AC-3, hỗ trợ Atmos | 128–640 Kbps | Premium OTT (Netflix-style) |
| **MP2** | Legacy, dùng trong MPEG-2 TS | 128–384 Kbps | Broadcast DVB |
| **AIFF/PCM** | Không nén (lossless) | Rất cao | Master archive |

### Cấu Hình AAC (Phổ Biến Nhất)

```json
{
  "AudioDescriptions": [
    {
      "Name": "audio-aac-stereo",
      "AudioSelectorName": "default-audio",
      "AudioTypeControl": "FOLLOW_INPUT",
      "LanguageCodeControl": "FOLLOW_INPUT",
      "CodecSettings": {
        "AacSettings": {
          "InputType": "NORMAL",
          "Bitrate": 128000,
          "RawFormat": "NONE",
          "Spec": "MPEG4",
          "Profile": "LC",
          "RateControlMode": "CBR",
          "SampleRate": 48000,
          "CodingMode": "CODING_MODE_2_0"
        }
      }
    }
  ]
}
```

### Coding Mode (Chế Độ Âm Thanh)

| Mode | Kênh | Dùng Cho |
|------|------|---------|
| `CODING_MODE_1_0` | Mono (1 kênh) | Podcast, radio |
| `CODING_MODE_2_0` | Stereo (2 kênh) | Hầu hết streaming |
| `CODING_MODE_5_1` | 5.1 Surround | Phim, thể thao có AC-3/EAC-3 |

---

## 6. Audio Selector — Chọn Track Âm Thanh

**Audio Selector** cho phép chọn track âm thanh cụ thể từ luồng input (khi input có nhiều track audio).

### Các Cách Chọn Track

#### Chọn Theo Ngôn Ngữ (Language Selection)

```json
{
  "AudioSelectors": {
    "vietnamese-audio": {
      "SelectorSettings": {
        "AudioLanguageSelection": {
          "LanguageCode": "vie",
          "LanguageSelectionPolicy": "STRICT"
        }
      }
    },
    "english-audio": {
      "SelectorSettings": {
        "AudioLanguageSelection": {
          "LanguageCode": "eng",
          "LanguageSelectionPolicy": "LOOSE"
        }
      }
    }
  }
}
```

#### Chọn Theo PID (Packet Identifier — Định Danh Gói — Dùng Trong MPEG-2 TS)

```json
{
  "AudioSelectors": {
    "audio-pid-101": {
      "SelectorSettings": {
        "AudioPidSelection": {
          "Pid": 101
        }
      }
    }
  }
}
```

#### Chọn Theo Track Index

```json
{
  "AudioSelectors": {
    "audio-track-1": {
      "SelectorSettings": {
        "AudioTrackSelection": {
          "Tracks": [
            { "Track": 1 }
          ]
        }
      }
    }
  }
}
```

> **Use case thực tế:** Live TV thường có nhiều track audio trong MPEG-2 TS: track 1 (tiếng Việt), track 2 (tiếng Anh), track 3 (Audio Description cho người khiếm thị). Audio Selector cho phép route từng track vào Output Group phù hợp.

---

## 7. Caption — Phụ Đề

### Định Dạng Caption Input Được Hỗ Trợ

| Định Dạng | Container | Mô Tả |
|----------|----------|-------|
| **DVB-Sub** | MPEG-2 TS | Phổ biến trong broadcast châu Âu |
| **SCTE-20** | MPEG-2 TS | Bitmap subtitle truyền thống |
| **Embedded** (CEA-608/708) | MPEG-2 TS, MP4 | Chuẩn North America |
| **Teletext** | MPEG-2 TS | Broadcast châu Âu (legacy) |
| **ARIB** | MPEG-2 TS | Broadcast Nhật Bản |

### Định Dạng Caption Output Được Hỗ Trợ

| Định Dạng | Dùng Cho |
|----------|---------|
| **WebVTT** | HLS streaming (web) |
| **TTML/DFXP** | DASH streaming |
| **Burn-in** | Nhúng phụ đề trực tiếp vào video (không thể tắt) |
| **Embedded (CEA-608/708)** | Chuyển tiếp embedded captions |

### Ví Dụ: DVB-Sub → WebVTT

```json
{
  "CaptionSelectors": {
    "dvb-sub-viet": {
      "SelectorSettings": {
        "DvbSubSourceSettings": {
          "Pid": 300,
          "OcrLanguage": "VIE"
        }
      }
    }
  }
}
```

---

## 8. Video Preprocessing — Tiền Xử Lý Video

MediaLive hỗ trợ một số bộ lọc tiền xử lý trước khi encode:

| Tính Năng | Mô Tả | Khi Dùng |
|----------|-------|---------|
| **Deinterlace** (khử xen kẽ) | Chuyển interlaced (1080i) sang progressive (1080p) | Input từ broadcast camera truyền thống |
| **Denoise** (khử nhiễu) | Giảm nhiễu hình ảnh | Input chất lượng thấp, camera analog |
| **Deblock** (khử ảnh vuông) | Giảm block artifact từ encoder nguồn | Input được compress nhiều lần |
| **Color Space** | Xử lý không gian màu (SDR/HDR) | HDR content (BT.2020, PQ, HLG) |

### Deinterlace — Rất Quan Trọng Với Broadcast Input

```json
{
  "VideoSelector": {
    "ColorSpace": "FOLLOW",
    "SelectorSettings": {
      "VideoSelectorPid": {
        "Pid": 481
      }
    }
  },
  "DeinterlaceSettings": {
    "Algorithm": "INTERPOLATE_TELETEXT",
    "Control": "FORCE_ALL_FRAMES",
    "Mode": "ADAPTIVE"
  }
}
```

---

## 9. Ví Dụ Cấu Hình Thực Tế

### Full HD Live Streaming (Thể Thao)

```json
{
  "EncoderSettings": {
    "VideoDescriptions": [
      {
        "Name": "video-1080p",
        "Width": 1920,
        "Height": 1080,
        "CodecSettings": {
          "H264Settings": {
            "RateControlMode": "CBR",
            "Bitrate": 5000000,
            "FramerateNumerator": 60,
            "FramerateDenominator": 1,
            "GopSize": 120,
            "GopSizeUnits": "FRAMES",
            "GopClosedCadence": 1,
            "Profile": "HIGH",
            "Level": "H264_LEVEL_4_2",
            "SceneChangeDetect": "ENABLED"
          }
        }
      },
      {
        "Name": "video-720p",
        "Width": 1280,
        "Height": 720,
        "CodecSettings": {
          "H264Settings": {
            "RateControlMode": "CBR",
            "Bitrate": 3000000,
            "FramerateNumerator": 30,
            "FramerateDenominator": 1,
            "GopSize": 60,
            "GopSizeUnits": "FRAMES",
            "GopClosedCadence": 1,
            "Profile": "HIGH",
            "Level": "H264_LEVEL_4_1",
            "SceneChangeDetect": "ENABLED"
          }
        }
      },
      {
        "Name": "video-360p",
        "Width": 640,
        "Height": 360,
        "CodecSettings": {
          "H264Settings": {
            "RateControlMode": "CBR",
            "Bitrate": 800000,
            "FramerateNumerator": 30,
            "FramerateDenominator": 1,
            "GopSize": 60,
            "GopSizeUnits": "FRAMES",
            "GopClosedCadence": 1,
            "Profile": "MAIN",
            "Level": "H264_LEVEL_3_1",
            "SceneChangeDetect": "ENABLED"
          }
        }
      }
    ],
    "AudioDescriptions": [
      {
        "Name": "audio-128k",
        "AudioSelectorName": "default-audio",
        "CodecSettings": {
          "AacSettings": {
            "Bitrate": 128000,
            "SampleRate": 48000,
            "CodingMode": "CODING_MODE_2_0",
            "Spec": "MPEG4",
            "Profile": "LC",
            "RateControlMode": "CBR"
          }
        }
      }
    ]
  }
}
```

---

## Best Practices

```
✅ NÊN làm:
- Dùng CBR cho live streaming (ổn định hơn VBR cho CDN caching)
- Đặt GOP size = segment length × framerate (đồng bộ với HLS packaging)
- Bật SceneChangeDetect để encode hiệu quả hơn khi cảnh thay đổi đột ngột
- Deinterlace input nếu nhận từ broadcast camera (interlaced signal)
- Tạo ít nhất 3 rendition (1 high + 1 medium + 1 low) cho ABR
- Dùng Profile HIGH cho desktop, MAIN cho thiết bị cũ

❌ KHÔNG nên làm:
- Encode resolution cao hơn resolution nguồn (upscale làm mờ ảnh)
- Dùng QVBR cho live stream khi cần ABR nhất quán
- Đặt quá nhiều rendition (> 8) gây tốn CPU và tăng chi phí MediaLive
- Bỏ qua AudioSelector khi input có nhiều track (có thể encode sai track)
```

---

**Phần Tiếp Theo:** [3-redundancy-failover.md](./3-redundancy-failover.md) — Standard vs Single Pipeline, Input Failover
