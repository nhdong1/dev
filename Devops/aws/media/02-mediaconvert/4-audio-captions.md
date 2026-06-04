# Audio & Captions — Âm Thanh Đa Track và Phụ Đề

> Xử lý audio đa kênh và phụ đề đúng cách là yêu cầu bắt buộc cho bất kỳ nền tảng video nghiêm túc nào — từ phụ đề song ngữ đến Dolby Atmos 5.1 surround sound.

## 📚 Mục Lục

1. [Tổng Quan Audio Trong MediaConvert](#1-tổng-quan-audio-trong-mediaconvert)
2. [Audio Selector — Chọn Track Âm Thanh](#2-audio-selector--chọn-track-âm-thanh)
3. [Audio Codec — Định Dạng Mã Hoá Âm Thanh](#3-audio-codec--định-dạng-mã-hoá-âm-thanh)
4. [Multi-track Audio — Âm Thanh Đa Track](#4-multi-track-audio--âm-thanh-đa-track)
5. [Audio Normalization — Chuẩn Hoá Âm Lượng](#5-audio-normalization--chuẩn-hoá-âm-lượng)
6. [Captions — Phụ Đề](#6-captions--phụ-đề)
7. [Caption Selector & Mapping](#7-caption-selector--mapping)
8. [Định Dạng Phụ Đề Trong MediaConvert](#8-định-dạng-phụ-đề-trong-mediaconvert)

---

## 1. Tổng Quan Audio Trong MediaConvert

### Luồng Xử Lý Audio

```
Video gốc (Input)
├── Audio Track 1: Tiếng Anh Stereo AAC
├── Audio Track 2: Tiếng Anh Dolby 5.1
└── Audio Track 3: Tiếng Việt Stereo AAC
         │
         ▼
    MediaConvert Job
    Audio Selectors → Audio Descriptions → Output
         │
         ▼
Output HLS:
├── rendition 1080p
│   ├── Audio: Tiếng Anh AAC 2.0 (default)
│   └── Audio: Tiếng Việt AAC 2.0 (alternate)
└── rendition 720p
    ├── Audio: Tiếng Anh AAC 2.0
    └── Audio: Tiếng Việt AAC 2.0

Output File MP4: Tiếng Anh Dolby 5.1 AC-3
```

### Khái Niệm Chính

| Khái Niệm | Ý Nghĩa |
|-----------|---------|
| **Audio Selector** | Chọn track audio từ file input để xử lý |
| **Audio Description** | Định nghĩa output audio: codec, bitrate, channel layout |
| **Audio Source Name** | Liên kết Audio Description với Audio Selector |
| **Default Selection** | Track audio mặc định trong manifest khi player tải |

---

## 2. Audio Selector — Chọn Track Âm Thanh

**Audio Selector** xác định **track nào** từ input sẽ được xử lý. Input thường có nhiều track (stereo English, surround sound, alternate language).

### Cách Chọn Audio Selector

```json
"AudioSelectors": {
  "Audio Selector 1": {
    "DefaultSelection": "DEFAULT",   // Chọn track audio mặc định (track đầu tiên)
    "SelectorType": "TRACK"          // TRACK, PID, LANGUAGE_CODE
  },
  "Audio Selector 2": {
    "DefaultSelection": "NOT_DEFAULT",
    "SelectorType": "TRACK",
    "Tracks": [2]                    // Chỉ định track thứ 2 (theo số thứ tự, bắt đầu từ 1)
  },
  "Vietnamese Audio": {
    "DefaultSelection": "NOT_DEFAULT",
    "SelectorType": "LANGUAGE_CODE",
    "LanguageCode": "VIE"            // Chọn theo mã ngôn ngữ ISO 639-2
  }
}
```

**Các Loại SelectorType:**

| SelectorType | Dùng Khi |
|-------------|---------|
| `DEFAULT` | Không biết cấu trúc input, lấy track mặc định |
| `TRACK` | Biết số thứ tự track (PID trong MPEG-TS) |
| `LANGUAGE_CODE` | Input có metadata ngôn ngữ (MPEG-4, MXF) |
| `PID` | Kỹ thuật — chỉ định PID — Packet Identifier trong MPEG-TS |
| `HLS_RENDITION_GROUP` | Input là HLS live stream |
| `EXTERNAL` | Track audio từ file riêng (không nằm trong video file) |

### External Audio File — Track Audio Ngoài

Trường hợp đặc biệt: audio và video ở 2 file khác nhau (phổ biến trong workflow chuyên nghiệp):

```json
"AudioSelectors": {
  "External Vietnamese": {
    "SelectorType": "EXTERNAL_AIFF",
    "ExternalAudioFileInput": "s3://my-bucket/vietnamese-audio.wav",
    "Offset": 0                      // Độ lệch thời gian (ms) để đồng bộ audio-video
  }
}
```

---

## 3. Audio Codec — Định Dạng Mã Hoá Âm Thanh

### Các Codec Audio Hỗ Trợ

| Codec | Tên Đầy Đủ | Bitrate Thông Thường | Use Case |
|-------|-----------|---------------------|---------|
| **AAC** | Advanced Audio Coding | 64–256 Kbps | Streaming web, mobile — phổ biến nhất |
| **AC3** | Dolby Digital | 192–640 Kbps | Blu-ray, broadcast, 5.1 surround |
| **EAC3** | Dolby Digital Plus (Enhanced AC-3) | 32–6144 Kbps | Streaming HD (Netflix), Dolby Atmos |
| **MP2** | MPEG-1 Audio Layer II | 192–384 Kbps | Broadcast truyền thống, DVB |
| **MP3** | MPEG-1 Audio Layer III | 128–320 Kbps | Legacy, podcast |
| **OPUS** | Opus | 6–510 Kbps | WebRTC, real-time, voice |
| **VORBIS** | Vorbis | Variable | WebM container |
| **AIFF** | Audio Interchange File Format | PCM lossless | Professional archive |
| **WAV** | Waveform Audio | PCM lossless | Editing, archive |
| **PASSTHROUGH** | Giữ nguyên codec gốc | — | Tránh re-encode |

### AAC — Cấu Hình Chi Tiết

**AAC — Advanced Audio Coding** là codec audio tiêu chuẩn cho streaming:

```json
{
  "CodecSettings": {
    "Codec": "AAC",
    "AacSettings": {
      "Bitrate": 128000,            // 128 Kbps stereo — standard
      "SampleRate": 48000,          // 48000 Hz (48 kHz) — broadcast standard
      "Channels": 2,                // 1=mono, 2=stereo, 6=5.1
      "CodingMode": "CODING_MODE_2_0",  // Xem bảng channel layout
      "RawFormat": "NONE",
      "Specification": "MPEG4",     // MPEG2 hoặc MPEG4
      "CodecProfile": "LC"          // LC, HEV1 (HE-AAC), HEV2 (HE-AACv2)
    }
  }
}
```

**Chọn Bitrate AAC:**

| Kênh | Bitrate Tối Thiểu | Bitrate Khuyến Nghị | Bitrate Cao (Chất Lượng Cao) |
|------|------------------|--------------------|-----------------------------|
| Mono | 32 Kbps | 64 Kbps | 128 Kbps |
| Stereo 2.0 | 64 Kbps | **128 Kbps** | 192–256 Kbps |
| 5.1 Surround | 192 Kbps | **320 Kbps** | 512 Kbps |
| 7.1 Surround | 256 Kbps | 448 Kbps | 640 Kbps |

**AAC Profile:**

| Profile | Tên | Ưu Điểm | Dùng Cho |
|---------|-----|---------|---------|
| `LC` | Low Complexity | Tương thích rộng nhất | **Mọi trường hợp thông thường** |
| `HEV1` | HE-AAC v1 | Bitrate thấp, chất lượng tốt | Mobile, bandwidth hạn chế |
| `HEV2` | HE-AAC v2 | Bitrate cực thấp | Mobile mono/stereo, radio streaming |

### AC-3 / Dolby Digital — Cấu Hình 5.1 Surround

```json
{
  "CodecSettings": {
    "Codec": "AC3",
    "Ac3Settings": {
      "Bitrate": 320000,              // 320 Kbps cho 5.1 surround
      "SampleRate": 48000,
      "CodingMode": "CODING_MODE_3_2", // 3 front + 2 rear + subwoofer (5.1)
      "DialnormValue": 24,             // Dialog normalization — chuẩn hoá âm thoại
      "DrcProfile": "FILM_STANDARD",  // DRC — Dynamic Range Control — Kiểm Soát Dải Động
      "LfeFilter": "ENABLED",         // LFE — Low Frequency Enhancement — kênh subwoofer
      "MetadataControl": "FOLLOW_INPUT"
    }
  }
}
```

### Dolby Digital Plus (EAC-3) — Cho Netflix/Prime Style

```json
{
  "CodecSettings": {
    "Codec": "EAC3",
    "Eac3Settings": {
      "Bitrate": 384000,
      "SampleRate": 48000,
      "CodingMode": "CODING_MODE_3_2",
      "DialnormValue": 24,
      "DrcLine": "FILM_STANDARD",
      "DrcRf": "FILM_STANDARD",
      "LfeControl": "LFE",
      "LfeFilter": "ENABLED",
      "LoRoCenterMixLevel": -3,
      "LoRoSurroundMixLevel": -3
    }
  }
}
```

---

## 4. Multi-track Audio — Âm Thanh Đa Track

### Cấu Hình Output Đa Ngôn Ngữ

Ví dụ: video có tiếng Anh (track 1) và tiếng Việt (track 2), cần output HLS với cả 2 ngôn ngữ:

```json
{
  "Inputs": [{
    "AudioSelectors": {
      "English": {
        "DefaultSelection": "DEFAULT",
        "SelectorType": "TRACK",
        "Tracks": [1]
      },
      "Vietnamese": {
        "DefaultSelection": "NOT_DEFAULT",
        "SelectorType": "TRACK",
        "Tracks": [2]
      }
    }
  }],
  "OutputGroups": [{
    "OutputGroupSettings": { "Type": "HLS_GROUP_SETTINGS", ... },
    "Outputs": [
      {
        "NameModifier": "_1080p_en",
        "VideoDescription": { ... },
        "AudioDescriptions": [
          {
            "AudioSourceName": "English",
            "LanguageCode": "ENG",
            "LanguageCodeControl": "USE_CONFIGURED",
            "AudioTypeControl": "FOLLOW_INPUT",
            "CodecSettings": {
              "Codec": "AAC",
              "AacSettings": { "Bitrate": 128000, "SampleRate": 48000, "Channels": 2 }
            }
          }
        ]
      },
      {
        "NameModifier": "_1080p_vi",
        "VideoDescription": { ... },      // Cùng video settings
        "AudioDescriptions": [
          {
            "AudioSourceName": "Vietnamese",
            "LanguageCode": "VIE",
            "LanguageCodeControl": "USE_CONFIGURED",
            "CodecSettings": {
              "Codec": "AAC",
              "AacSettings": { "Bitrate": 128000, "SampleRate": 48000, "Channels": 2 }
            }
          }
        ]
      }
    ]
  }]
}
```

**Kết quả trong HLS Master Playlist:**
```
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio-en",LANGUAGE="en",NAME="English",DEFAULT=YES,URI="1080p_en/1080p_en.m3u8"
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio-vi",LANGUAGE="vi",NAME="Tiếng Việt",DEFAULT=NO,URI="1080p_vi/1080p_vi.m3u8"
#EXT-X-STREAM-INF:BANDWIDTH=5128000,AUDIO="audio-en"
1080p_en/1080p_en.m3u8
```

Player sẽ hiển thị menu chọn ngôn ngữ cho viewer.

### Audio Remix — Phối Lại Kênh Audio

Khi input có 5.1 surround nhưng cần output stereo (downmix):

```json
"AudioDescriptions": [{
  "AudioSourceName": "Surround 5.1",
  "RemixSettings": {
    "ChannelMapping": {
      "OutputChannels": [
        {
          "InputChannels": [0, 2, 4],  // L-front, L-surround, LFE → trái
          "InputChannelsFineTune": [0.0, -3.0, -10.0]
        },
        {
          "InputChannels": [1, 3, 5],  // R-front, R-surround, LFE → phải
          "InputChannelsFineTune": [0.0, -3.0, -10.0]
        }
      ]
    },
    "ChannelsIn": 6,    // 5.1 = 6 kênh
    "ChannelsOut": 2    // Stereo = 2 kênh
  }
}]
```

---

## 5. Audio Normalization — Chuẩn Hoá Âm Lượng

### Vấn Đề: Âm Lượng Không Đồng Đều

```
Video 1: Âm lượng quá to  (-3 LUFS)
Video 2: Âm lượng bình thường (-16 LUFS)
Video 3: Âm lượng quá nhỏ (-30 LUFS)
→ Viewer phải liên tục chỉnh volume ← trải nghiệm xấu
```

**Giải pháp: Audio Normalization** (chuẩn hoá âm lượng về mức mục tiêu)

### Chuẩn LUFS (Loudness Units Full Scale — Đơn Vị Độ To Tham Chiếu Đầy Đủ)

| Tiêu Chuẩn | LUFS Target | Phạm Vi Áp Dụng |
|-----------|-------------|-----------------|
| **EBU R128** | -23 LUFS | Broadcast châu Âu |
| **ATSC A/85** | -24 LUFS | Broadcast Mỹ (TV) |
| **Apple Podcasts** | -16 LUFS | Podcast/Audio |
| **Streaming (Netflix/YouTube)** | -14 LUFS | OTT platforms |

### Cấu Hình Audio Normalization

```json
"AudioDescriptions": [{
  "AudioNormalizationSettings": {
    "Algorithm": "ITU_BS_1770_4",       // Thuật toán đo loudness: ITU BS.1770-4
    "AlgorithmControl": "MEASURE_ONLY", // MEASURE_ONLY, CORRECT_AUDIO
    "CorrectionGateLevel": -70,         // Ngưỡng im lặng (dBFS) — không chuẩn hoá đoạn im lặng
    "LoudnessLogging": "LOG",
    "PeakCalculation": "NONE",
    "TargetLkfs": -14.0                 // Mục tiêu -14 LUFS (LKFS = LUFS)
  }
}]
```

**AlgorithmControl:**

| Giá Trị | Ý Nghĩa |
|---------|---------|
| `MEASURE_ONLY` | Chỉ đo, ghi log — không thay đổi audio |
| `CORRECT_AUDIO` | Đo và tự động điều chỉnh gain để đạt TargetLkfs |

---

## 6. Captions — Phụ Đề

### Tổng Quan Phụ Đề

**Captions — Phụ Đề** trong MediaConvert bao gồm:
- **Closed Captions** — Phụ Đề Đóng: có thể bật/tắt, thường nhúng trong stream
- **Open Captions** — Phụ Đề Mở: "đốt" vào video, không thể tắt
- **Subtitles** — Phụ Đề Bên Ngoài: file riêng biệt

### Luồng Phụ Đề

```
Input có phụ đề:
├── Embedded CEA-608/CEA-708 (trong video MPEG-2 TS)
├── SRT file (s3://bucket/subtitles-en.srt)
├── TTML file (Timed Text Markup Language — Ngôn Ngữ Đánh Dấu Văn Bản Theo Thời Gian)
└── SCC/STL/IMSC (broadcast formats)
         │
         ▼
    MediaConvert
    Caption Selectors → Caption Descriptions → Output
         │
         ▼
Output HLS:
├── WebVTT (.vtt) — embedded trong HLS manifest
├── Embedded trong segment (CEA-608)
Output DASH:
└── TTML (.ttml) — sidecar file hoặc embedded trong MPD
Output MP4:
└── TTXT embedded hoặc no captions
```

---

## 7. Caption Selector & Mapping

### Caption Selector — Chọn Nguồn Phụ Đề

```json
"CaptionSelectors": {
  "Captions Selector 1": {
    "SourceSettings": {
      "SourceType": "SRT",
      "FileSourceSettings": {
        "SourceFile": "s3://my-bucket/subtitles-en.srt",
        "TimeDelta": 0,              // Độ lệch thời gian (ms)
        "Encoding": "UTF_8",
        "Convert608To708": "DISABLED"
      }
    }
  },
  "Captions Selector 2": {
    "LanguageCode": "VIE",
    "SourceSettings": {
      "SourceType": "SRT",
      "FileSourceSettings": {
        "SourceFile": "s3://my-bucket/subtitles-vi.srt",
        "Encoding": "UTF_8"
      }
    }
  },
  "Embedded Captions": {
    "SourceSettings": {
      "SourceType": "EMBEDDED",        // Phụ đề nhúng sẵn trong video input
      "EmbeddedSourceSettings": {
        "Convert608To708": "DISABLED",
        "Source608ChannelNumber": 1,
        "TerminateCaptions": "END_OF_INPUT"
      }
    }
  }
}
```

**SourceType hỗ trợ:**

| SourceType | Định Dạng | Phổ Biến |
|------------|-----------|---------|
| `SRT` | SubRip Text — phổ biến nhất | YouTube, video online |
| `TTML` | Timed Text Markup Language | Broadcast, Netflix |
| `WEBVTT` | Web Video Text Tracks | Web player |
| `EMBEDDED` | CEA-608/CEA-708 nhúng trong video | Broadcast, Blu-ray |
| `IMSC` | Internet Media Subtitles and Captions | OTT platform |
| `SCC` | Scenarist Closed Captions | US broadcast |
| `STL` | Spruce Subtitle File | Broadcast EU |
| `ANCILLARY` | Ancillary data trong SDI | Professional broadcast |

### Caption Description — Định Nghĩa Output Phụ Đề

```json
"CaptionDescriptions": [
  {
    "CaptionSelectorName": "Captions Selector 1",   // Trỏ đến Caption Selector
    "LanguageCode": "ENG",
    "LanguageDescription": "English",
    "DestinationSettings": {
      "DestinationType": "WEBVTT",                  // Loại output
      "WebvttDestinationSettings": {
        "Accessibility": "ENABLED",
        "StylePassthrough": "ENABLED"               // Giữ nguyên style (font, màu)
      }
    }
  },
  {
    "CaptionSelectorName": "Captions Selector 2",
    "LanguageCode": "VIE",
    "LanguageDescription": "Tiếng Việt",
    "DestinationSettings": {
      "DestinationType": "WEBVTT",
      "WebvttDestinationSettings": {}
    }
  }
]
```

---

## 8. Định Dạng Phụ Đề Trong MediaConvert

### So Sánh Định Dạng Output

| Định Dạng | Container | Ưu Điểm | Nhược Điểm | Dùng Cho |
|-----------|-----------|---------|------------|---------|
| **WebVTT** | HLS, DASH | Web-friendly, style phong phú | File riêng, cần serve thêm | **Web streaming** |
| **TTML** | DASH, MP4 | XML-based, phong phú | Phức tạp hơn | Broadcast, DASH |
| **Embedded CEA-608** | MPEG-2 TS | Tương thích thiết bị cũ | Chỉ 1 ngôn ngữ, ít tính năng | Legacy HLS |
| **Embedded CEA-708** | MPEG-2 TS | Đa ngôn ngữ, tính năng tốt hơn 608 | Cần decoder hỗ trợ | Broadcast |
| **SRT** | File Group | Đơn giản, phổ biến | Ít tính năng style | Download, archive |
| **IMSC** | CMAF, DASH | Phong phú, chuẩn quốc tế | Ít player hỗ trợ | OTT cao cấp |

### WebVTT — Chuẩn Phụ Đề Web

**WebVTT — Web Video Text Tracks** là chuẩn phụ đề cho web player (H5, hls.js, Shaka Player):

```
WEBVTT

00:00:05.000 --> 00:00:08.000
Xin chào, đây là ví dụ phụ đề tiếng Việt.

00:00:10.000 --> 00:00:14.500
<c.yellow>Đây là chú thích màu vàng</c>

00:00:16.000 --> 00:00:20.000 position:50% align:center
Căn giữa màn hình
```

**Cấu hình output WebVTT trong HLS:**
```json
"DestinationSettings": {
  "DestinationType": "WEBVTT",
  "WebvttDestinationSettings": {
    "Accessibility": "ENABLED",         // Thêm tag accessibility
    "StylePassthrough": "ENABLED"       // Giữ style từ source TTML/SRT
  }
}
```

**Cấu trúc HLS với WebVTT:**
```
master.m3u8:
  #EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",LANGUAGE="en",NAME="English",URI="subs/en/en.m3u8"
  #EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",LANGUAGE="vi",NAME="Tiếng Việt",DEFAULT=NO,URI="subs/vi/vi.m3u8"
  #EXT-X-STREAM-INF:...,SUBTITLES="subs"
  1080p/1080p.m3u8
```

### Burn-in Captions — Phụ Đề Đốt Vào Video

**Burn-in** nhúng phụ đề trực tiếp vào video frame (không thể tắt):

```json
"DestinationSettings": {
  "DestinationType": "BURN_IN",
  "BurninDestinationSettings": {
    "Alignment": "CENTERED",
    "BackgroundColor": "NONE",
    "BackgroundOpacity": 0,
    "FontColor": "WHITE",
    "FontOpacity": 255,
    "FontResolution": 96,
    "FontScript": "AUTOMATIC",
    "FontSize": 0,                  // 0 = auto size theo resolution
    "OutlineColor": "BLACK",
    "OutlineSize": 2,               // Viền đen bao quanh chữ
    "ShadowColor": "BLACK",
    "ShadowOpacity": 255,
    "ShadowXOffset": 2,
    "ShadowYOffset": -2,
    "TeletextGridControl": "FIXED",
    "XPosition": 0,
    "YPosition": 0
  }
}
```

**Burn-in dùng khi:**
- Xuất video cho social media (phụ đề hiển thị ngay, không cần player hỗ trợ)
- Accessibility bắt buộc (khi không dùng được player với caption support)
- Preview / QC — Quality Control (kiểm tra phụ đề trong video)

---

## Bảng Tổng Hợp: Cấu Hình Audio & Captions Theo Use Case

| Use Case | Audio Config | Captions Config |
|----------|-------------|----------------|
| **VOD đơn ngữ** | AAC 128 Kbps stereo | WebVTT 1 ngôn ngữ |
| **OTT đa ngữ (EN + VI)** | AAC stereo × 2 ngôn ngữ | WebVTT × 2 ngôn ngữ |
| **Premium / Cinematic** | AAC 2.0 + AC-3 5.1 | WebVTT + Embedded CEA-708 |
| **Broadcast** | AC-3 5.1 @ 320 Kbps | Embedded CEA-708 |
| **Social Media (YouTube)** | AAC stereo | Burn-in hoặc SRT |
| **Archive chuyên nghiệp** | PCM / AIFF lossless | TTML |
| **Mobile bandwidth hạn chế** | AAC HE-AAC v1 @ 64 Kbps | WebVTT (file nhỏ) |

---

**Phần Tiếp Theo:** [5-cost-optimization.md](./5-cost-optimization.md) — Tối ưu chi phí: Reserved Queue, Spot, per-minute pricing
