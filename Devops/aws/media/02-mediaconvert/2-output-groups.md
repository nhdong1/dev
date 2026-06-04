# Output Groups — Định Dạng Đầu Ra MediaConvert

> Output Group xác định **định dạng container**, **cấu trúc file** và **nơi lưu** kết quả chuyển mã. Chọn đúng Output Group là yếu tố then chốt để video phát trơn tru trên mọi thiết bị và tối ưu chi phí lưu trữ.

## 📚 Mục Lục

1. [Tổng Quan Các Loại Output Group](#1-tổng-quan-các-loại-output-group)
2. [HLS Group — Apple HTTP Live Streaming](#2-hls-group--apple-http-live-streaming)
3. [DASH ISO Group — MPEG-DASH](#3-dash-iso-group--mpeg-dash)
4. [CMAF Group — Common Media Application Format](#4-cmaf-group--common-media-application-format)
5. [File Group — File Độc Lập](#5-file-group--file-độc-lập)
6. [MS Smooth Group — Microsoft Smooth Streaming](#6-ms-smooth-group--microsoft-smooth-streaming)
7. [Chiến Lược Kết Hợp Output Groups](#7-chiến-lược-kết-hợp-output-groups)

---

## 1. Tổng Quan Các Loại Output Group

MediaConvert hỗ trợ 5 loại Output Group:

| Output Group | Định Dạng Manifest | Định Dạng Segment | Dùng Cho |
|-------------|--------------------|--------------------|---------|
| **HLS** | `.m3u8` (playlist) | `.ts` hoặc `.fmp4` | iOS, Safari, Apple TV, OTT |
| **DASH ISO** | `.mpd` (Media Presentation Description) | `.m4s` | Android, Chrome, Smart TV, OTT |
| **CMAF** | `.m3u8` + `.mpd` | `.m4s` (dùng chung) | Multi-platform, tiết kiệm storage |
| **File Group** | Không có | `.mp4`, `.mov`... | Download, archive, social media |
| **MS Smooth** | `.ism` (manifest) | `.ismv` / `.isma` | Xbox, Silverlight, Azure Media |

### Quyết Định Chọn Output Group

```
Bạn cần phát trên nền tảng nào?

iOS/Safari bắt buộc?
  ├─ Có → cần HLS (hoặc CMAF)
  └─ Không ↓

Android/Chrome/Smart TV?
  ├─ Có → cần DASH (hoặc CMAF)
  └─ Không ↓

Multi-platform và muốn tiết kiệm storage?
  └─ Dùng CMAF (1 bộ segment cho cả HLS + DASH)

Cần file download hoặc archive?
  └─ Dùng File Group (MP4)

Môi trường Microsoft/Azure?
  └─ Dùng MS Smooth
```

---

## 2. HLS Group — Apple HTTP Live Streaming

### Cấu Trúc Output HLS

```
s3://output-bucket/hls/video-001/
├── master.m3u8                    ← Master Playlist — Danh Sách Phát Chính (chứa tất cả bitrate)
├── 1080p/
│   ├── 1080p.m3u8                 ← Media Playlist — Danh Sách Phát Phụ (cho 1080p)
│   ├── 1080p_00001.ts             ← Segment (đoạn video) #1
│   ├── 1080p_00002.ts             ← Segment #2
│   └── ...
├── 720p/
│   ├── 720p.m3u8
│   └── *.ts
└── 360p/
    ├── 360p.m3u8
    └── *.ts
```

**Nội dung Master Playlist:**
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p/1080p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
720p/720p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=640x360
360p/360p.m3u8
```

### Cấu Hình HLS Group

```json
{
  "OutputGroupSettings": {
    "Type": "HLS_GROUP_SETTINGS",
    "HlsGroupSettings": {
      "Destination": "s3://output-bucket/hls/video-001/",
      "SegmentLength": 6,          // Thời lượng mỗi segment (giây) — thường 4–10s
      "MinSegmentLength": 0,       // 0 = cho phép segment cuối ngắn hơn
      "SegmentControl": "SEGMENTED_FILES",  // Mỗi segment là file riêng
      "ManifestCompression": "NONE",
      "StreamInfResolution": "INCLUDE",     // Ghi resolution vào manifest
      "ClientCache": "ENABLED",
      "AudioOnlyHeader": "EXCLUDE",
      "ProgramDateTime": "EXCLUDE",
      "TimedMetadataId3Frame": "PRIV",
      "TimedMetadataId3Period": 10,
      "DirectoryStructure": "SINGLE_DIRECTORY",  // Tất cả file trong 1 thư mục
      "OutputSelection": "MANIFESTS_AND_SEGMENTS"
    }
  }
}
```

### Tuỳ Chọn Quan Trọng

**SegmentLength (Thời Lượng Segment):**

| Giá Trị | Ưu Điểm | Nhược Điểm | Use Case |
|---------|---------|------------|---------|
| **2s** | Chuyển bitrate rất nhanh | Nhiều file nhỏ, request nhiều | Low-latency live (không dùng cho VOD) |
| **4s** | Cân bằng tốt | — | Live streaming |
| **6s** | Phổ biến nhất cho VOD | — | **VOD tiêu chuẩn** |
| **10s** | Ít file, request ít | Chuyển bitrate chậm | Archive, nội dung dài |

**HLS Container (Định Dạng Segment):**

| Container | Định Dạng Segment | Hỗ Trợ |
|-----------|-------------------|---------|
| `M3U8` + `MPEG-2 TS` | `.ts` | iOS/Safari cũ (< iOS 10), universal |
| `M3U8` + `fMP4` | `.mp4` | iOS 10+, modern HLS — hỗ trợ HDR, HEVC tốt hơn |

> Ưu tiên **fMP4 (fragmented MP4)** cho nội dung mới vì hỗ trợ H.265 và HDR tốt hơn.

**Encryption — Mã Hoá:**
```json
"HlsEncryptionSettings": {
  "EncryptionMethod": "AES128",        // AES-128 standard HLS encryption
  "SpekeKeyProvider": {
    "ResourceId": "content-id-001",
    "SystemIds": ["81376844-f976-481e-a84e-cc25d39b0b33"],
    "Url": "https://speke-server.example.com/copyProtection"
  }
}
```

---

## 3. DASH ISO Group — MPEG-DASH

### Cấu Trúc Output DASH

```
s3://output-bucket/dash/video-001/
├── video-001.mpd                  ← MPD — Media Presentation Description (manifest chính)
├── video_1920x1080/
│   ├── init.mp4                   ← Initialization Segment — Segment Khởi Tạo (codec info)
│   ├── 00001.m4s                  ← Media Segment #1
│   ├── 00002.m4s                  ← Media Segment #2
│   └── ...
├── video_1280x720/
│   └── ...
└── audio_aac/
    └── ...
```

**Nội dung MPD (rút gọn):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<MPD type="static" mediaPresentationDuration="PT10M30S">
  <Period>
    <AdaptationSet mimeType="video/mp4" codecs="avc1.640028">
      <Representation id="1080p" bandwidth="5000000" width="1920" height="1080">
        <SegmentTemplate media="video_1920x1080/$Number$.m4s"
                         initialization="video_1920x1080/init.mp4" timescale="90000"/>
      </Representation>
      <Representation id="720p" bandwidth="3000000" width="1280" height="720">
        ...
      </Representation>
    </AdaptationSet>
    <AdaptationSet mimeType="audio/mp4" codecs="mp4a.40.2">
      ...
    </AdaptationSet>
  </Period>
</MPD>
```

### Cấu Hình DASH ISO Group

```json
{
  "OutputGroupSettings": {
    "Type": "DASH_ISO_GROUP_SETTINGS",
    "DashIsoGroupSettings": {
      "Destination": "s3://output-bucket/dash/video-001/",
      "SegmentLength": 6,
      "FragmentLength": 2,          // Fragment nhỏ trong segment (cho low-latency)
      "SegmentControl": "SEGMENTED_FILES",
      "MpdProfile": "MAIN_PROFILE", // MAIN_PROFILE hoặc ON_DEMAND_PROFILE
      "HbbtvCompliance": "NONE",    // Bật nếu cần cho European HbbTV — Hybrid Broadcast Broadband TV
      "WriteSegmentTimelineInRepresentation": "ENABLED"
    }
  }
}
```

### HLS vs DASH — Khi Nào Dùng Cái Nào?

| Tiêu Chí | HLS | DASH |
|----------|-----|------|
| **iOS / Safari** | ✅ Native support | ❌ Cần Shaka/dash.js |
| **Android / Chrome** | ✅ Hỗ trợ (EME/MSE) | ✅ Native |
| **Smart TV** | ✅ | ✅ |
| **DRM** | FairPlay (Apple) | Widevine (Google) |
| **Standard** | Apple proprietary | ISO standard mở |
| **HEVC/H.265** | fMP4 HLS | ✅ Tốt |
| **Low-latency** | LL-HLS — Low Latency HLS | DASH-LL |
| **Kết luận** | **Bắt buộc nếu có iOS** | **Tốt cho Android/Web** |

---

## 4. CMAF Group — Common Media Application Format

### CMAF Là Gì?

**CMAF — Common Media Application Format — Định Dạng Ứng Dụng Media Chung** là chuẩn (ISO 23000-19) cho phép **dùng chung một bộ segment** cho cả HLS và DASH. Đây là điểm khác biệt lớn nhất:

```
Trước CMAF (HLS + DASH riêng biệt):
  Video gốc → Encode → HLS Segments (50 GB) + DASH Segments (50 GB) = 100 GB trên S3

Với CMAF (HLS + DASH dùng chung):
  Video gốc → Encode → CMAF Segments (50 GB) + 2 manifest files = 50 GB trên S3
                        ↑ dùng cả cho HLS lẫn DASH
```

### Cấu Trúc Output CMAF

```
s3://output-bucket/cmaf/video-001/
├── hls/
│   └── master.m3u8               ← HLS Master Playlist (trỏ đến CMAF segments)
├── dash/
│   └── video-001.mpd             ← DASH MPD Manifest (cũng trỏ đến CMAF segments)
└── segments/                     ← Một bộ segment dùng chung cho cả 2
    ├── 1080p/
    │   ├── init.mp4
    │   └── *.m4s
    ├── 720p/
    │   └── *.m4s
    └── audio/
        └── *.m4s
```

### Cấu Hình CMAF Group

```json
{
  "OutputGroupSettings": {
    "Type": "CMAF_GROUP_SETTINGS",
    "CmafGroupSettings": {
      "Destination": "s3://output-bucket/cmaf/video-001/",
      "SegmentLength": 6,
      "FragmentLength": 2,
      "ManifestCompression": "NONE",
      "StreamInfResolution": "INCLUDE",
      "WriteDashManifest": "ENABLED",   // Tạo MPD manifest
      "WriteHlsManifest": "ENABLED",    // Tạo m3u8 manifest
      "WriteSegmentTimelineInRepresentation": "ENABLED",
      "PtsOffsetHandlingForBFrames": "MATCH_INITIAL_PTS"
    }
  }
}
```

### Khi Nào Dùng CMAF?

```
Nên dùng CMAF khi:
  ✅ Cần hỗ trợ cả iOS (HLS) và Android/Web (DASH) đồng thời
  ✅ Muốn tiết kiệm chi phí S3 storage (~50%)
  ✅ Muốn giảm origin egress bandwidth
  ✅ Dùng Low-Latency CMAF (LL-CMAF) cho live-to-VOD
  ✅ Content mới — iOS 10+, modern browsers

Không nên dùng CMAF khi:
  ❌ Cần hỗ trợ iOS/Safari rất cũ (< iOS 10)
  ❌ Player legacy không hỗ trợ fMP4 segments
  ❌ Cần MPEG-2 TS segments bắt buộc
```

---

## 5. File Group — File Độc Lập

### Cấu Trúc Output File Group

File Group tạo **file video hoàn chỉnh** (không chia segment):

```
s3://output-bucket/files/video-001/
├── video-001_1080p.mp4      ← File MP4 đầy đủ (download / progressive play)
├── video-001_720p.mp4
├── video-001_thumbnail.jpg  ← Frame tĩnh (thumbnail)
└── video-001_proxy.mp4      ← Proxy quality thấp (editor preview)
```

### Cấu Hình File Group

```json
{
  "OutputGroupSettings": {
    "Type": "FILE_GROUP_SETTINGS",
    "FileGroupSettings": {
      "Destination": "s3://output-bucket/files/video-001/"
    }
  },
  "Outputs": [
    {
      "NameModifier": "_1080p",
      "ContainerSettings": {
        "Container": "MP4",
        "Mp4Settings": {
          "CslgAtom": "INCLUDE",
          "FreeSpaceBox": "EXCLUDE",
          "MoovPlacement": "PROGRESSIVE_DOWNLOAD"  // Cho phép phát trong khi tải
        }
      },
      "VideoDescription": { ... },
      "AudioDescriptions": [{ ... }]
    }
  ]
}
```

**Container hỗ trợ trong File Group:**

| Container | Extension | Use Case |
|-----------|-----------|---------|
| `MP4` | `.mp4` | Progressive playback, download, social media |
| `MOV` | `.mov` | Editing workflow (Final Cut Pro, Premiere) |
| `MXF` | `.mxf` | Broadcast archive, professional workflow |
| `RAW` | `.raw` | Framecapture (chụp frame tĩnh thành ảnh) |

### Chèn Thumbnail (Frame Capture)

```json
{
  "Outputs": [
    {
      "NameModifier": "_thumbnail",
      "ContainerSettings": {
        "Container": "RAW"
      },
      "VideoDescription": {
        "CodecSettings": {
          "Codec": "FRAME_CAPTURE",
          "FrameCaptureSettings": {
            "FramerateNumerator": 1,
            "FramerateDenominator": 30,  // Chụp 1 frame mỗi 30 giây
            "MaxCaptures": 3,
            "Quality": 80
          }
        }
      }
    }
  ]
}
```

---

## 6. MS Smooth Group — Microsoft Smooth Streaming

### Tổng Quan

**MS Smooth Streaming — Microsoft Smooth Streaming** là định dạng streaming của Microsoft, chủ yếu dùng trong môi trường:
- Xbox One / Xbox Series
- Azure Media Services
- Windows Phone (cũ)
- Silverlight player (cũ, deprecated)

```
s3://output-bucket/smooth/video-001/
├── video-001.ism              ← Manifest chính
├── video-001.ismc             ← Client manifest
├── video-001_video_1.ismv     ← Video track (ISMV — Internet Streaming Media Video)
└── video-001_audio_1.isma     ← Audio track (ISMA — Internet Streaming Media Audio)
```

### Cấu Hình MS Smooth Group

```json
{
  "OutputGroupSettings": {
    "Type": "MS_SMOOTH_GROUP_SETTINGS",
    "MsSmoothGroupSettings": {
      "Destination": "s3://output-bucket/smooth/video-001/",
      "FragmentLength": 2,
      "ManifestEncoding": "UTF8"
    }
  }
}
```

> **Lưu ý thực tế:** MS Smooth ngày càng ít được dùng. Nếu không có yêu cầu cụ thể từ client dùng Microsoft/Azure stack, ưu tiên HLS/DASH/CMAF.

---

## 7. Chiến Lược Kết Hợp Output Groups

### Chiến Lược 1: Multi-platform VOD (Phổ Biến Nhất)

```json
"OutputGroups": [
  { "Type": "CMAF" → s3://output/cmaf/  },  // iOS + Android (dùng chung segment)
  { "Type": "FILE_GROUP" → s3://output/mp4/ } // Download + thumbnail
]
```

**Tại sao CMAF thay vì HLS + DASH riêng?**
- Tiết kiệm 50% S3 storage
- Giảm thời gian encode (chỉ encode 1 lần)
- Giảm chi phí egress từ S3

### Chiến Lược 2: Maximum Compatibility (Tối Đa Tương Thích)

```json
"OutputGroups": [
  { "Type": "HLS_GROUP" → s3://output/hls/    },  // iOS cũ (< iOS 10)
  { "Type": "DASH_ISO_GROUP" → s3://output/dash/ }, // Android/Web
  { "Type": "FILE_GROUP" → s3://output/mp4/   }    // Download
]
```

Dùng khi cần hỗ trợ thiết bị cũ, nhưng tốn ~2× storage so với CMAF.

### Chiến Lược 3: Simple VOD (Nhỏ, Đơn Giản)

```json
"OutputGroups": [
  { "Type": "HLS_GROUP" → s3://output/hls/ }  // Đủ cho hầu hết use cases
]
```

HLS được hỗ trợ rộng rãi — iOS native, Android/Chrome qua hls.js. Phù hợp cho ứng dụng startup không cần DASH riêng.

### Chiến Lược 4: Professional Workflow (Luồng Làm Việc Chuyên Nghiệp)

```json
"OutputGroups": [
  { "Type": "CMAF" → Phát streaming production },
  { "Type": "FILE_GROUP" (ProRes 422) → Lưu trữ editing-ready },
  { "Type": "FILE_GROUP" (MP4 proxy) → Preview trong CMS — Content Management System }
]
```

---

## Bảng Tổng Hợp Quyết Định

| Use Case | Output Groups Đề Xuất | Lý Do |
|----------|----------------------|-------|
| OTT platform (iOS + Android) | CMAF + File Group | Tiết kiệm storage, hỗ trợ đa nền tảng |
| Nền tảng học trực tuyến | HLS + File Group | HLS đủ dùng, MP4 cho download |
| Kênh YouTube / Social | File Group (MP4 only) | Platform tự xử lý transcoding |
| Broadcast archive | File Group (MXF/ProRes) | Chất lượng lossless cho editing |
| Azure/Microsoft stack | MS Smooth + CMAF | Tương thích Xbox + modern browser |
| Startup MVPchú | HLS Group | Nhanh, đơn giản, hỗ trợ rộng |

---

**Phần Tiếp Theo:** [3-video-codec-settings.md](./3-video-codec-settings.md) — Cấu hình codec H.264, H.265, AV1
