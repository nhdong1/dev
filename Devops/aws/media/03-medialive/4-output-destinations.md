# Output Destinations — Đích Đến Đầu Ra MediaLive

> MediaLive có thể gửi output đến nhiều đích đến cùng lúc. Hiểu rõ từng loại Output Group giúp thiết kế pipeline phù hợp: phân phối đến viewer, lưu trữ archive, hay phát đồng thời lên nhiều nền tảng.

## 📚 Mục Lục

1. [Tổng Quan Output Group](#1-tổng-quan-output-group)
2. [MediaPackage — Đóng Gói Và Phân Phối](#2-mediapackage--đóng-gói-và-phân-phối)
3. [S3 — Lưu Trữ và Archive](#3-s3--lưu-trữ-và-archive)
4. [RTMP Push — Phát Đồng Thời Lên Nền Tảng Khác](#4-rtmp-push--phát-đồng-thời-lên-nền-tảng-khác)
5. [HLS Output Group — HLS Trực Tiếp Lên S3/HTTP](#5-hls-output-group--hls-trực-tiếp-lên-s3http)
6. [UDP/TS — MPEG-2 TS Qua UDP](#6-udpts--mpeg-2-ts-qua-udp)
7. [Frame Capture — Chụp Thumbnail](#7-frame-capture--chụp-thumbnail)
8. [Đa Đích Đến (Multiple Destinations)](#8-đa-đích-đến-multiple-destinations)
9. [IAM Permissions — Quyền Cần Thiết](#9-iam-permissions--quyền-cần-thiết)

---

## 1. Tổng Quan Output Group

**Output Group** là cấu hình xác định đích đến và định dạng của một nhóm output. Một Channel có thể có nhiều Output Group cùng lúc.

### Các Loại Output Group

| Loại | Đích Đến | Dùng Cho |
|------|---------|---------|
| **MediaPackage** | AWS Elemental MediaPackage | Phát trực tiếp đến viewer qua CDN |
| **HLS Group** | S3 hoặc HTTP endpoint | Lưu HLS segments + manifest lên S3 |
| **Archive** | S3 | Lưu trữ toàn bộ luồng (MPEG-2 TS) |
| **RTMP** | RTMP server | Đẩy lên YouTube/Facebook/Twitch |
| **Frame Capture** | S3 | Chụp ảnh thumbnail định kỳ |
| **UDP** | UDP endpoint | Output MPEG-2 TS qua UDP |
| **Microsoft Smooth** | IIS/Azure | Silverlight streaming (legacy) |
| **Multiplex** | MediaLive Multiplex | Combine nhiều program vào 1 MPTS |

---

## 2. MediaPackage — Đóng Gói Và Phân Phối

**MediaPackage Output Group** là cách phổ biến nhất và được khuyến nghị nhất để phát video trực tiếp đến viewer. MediaLive gửi luồng đến MediaPackage Channel qua WebDAV — Web Distributed Authoring and Versioning, MediaPackage xử lý việc đóng gói (HLS, DASH, CMAF) và DRM.

### Tại Sao Dùng MediaPackage (Thay Vì HLS Output Trực Tiếp Lên S3)?

```
HLS Output → S3 trực tiếp:
  ❌ Không có DRM
  ❌ Không có time-shift (startover, catch-up)
  ❌ Không tự động scale origin theo viewer load
  ❌ Phải tự quản lý CloudFront + S3 origin

MediaPackage Output:
  ✅ DRM tích hợp (Widevine, FairPlay, PlayReady)
  ✅ Time-shift viewing (startover, catch-up TV)
  ✅ Tự động scale origin
  ✅ Just-in-time packaging (1 segment → nhiều định dạng)
  ✅ Tích hợp sẵn với CloudFront
```

### Cách Hoạt Động

```
MediaLive Channel (Standard — 2 pipelines)
  Pipeline A ──WebDAV PUSH──▶ MediaPackage Channel (Ingest endpoint 1)
  Pipeline B ──WebDAV PUSH──▶ MediaPackage Channel (Ingest endpoint 2)
                                        │
                                        ▼
                              MediaPackage Origin
                              ├── HLS Endpoint   → CloudFront → Viewer
                              ├── DASH Endpoint  → CloudFront → Viewer
                              └── CMAF Endpoint  → CloudFront → Viewer
```

### Cấu Hình MediaPackage Output Group

```json
{
  "OutputGroups": [
    {
      "Name": "MediaPackage Group",
      "OutputGroupSettings": {
        "MediaPackageGroupSettings": {
          "Destination": {
            "DestinationRefId": "mediapackage-dest"
          }
        }
      },
      "Outputs": [
        {
          "OutputName": "1080p",
          "VideoDescriptionName": "video-1080p",
          "AudioDescriptionNames": ["audio-128k"],
          "OutputSettings": {
            "MediaPackageOutputSettings": {}
          }
        },
        {
          "OutputName": "720p",
          "VideoDescriptionName": "video-720p",
          "AudioDescriptionNames": ["audio-128k"],
          "OutputSettings": {
            "MediaPackageOutputSettings": {}
          }
        },
        {
          "OutputName": "360p",
          "VideoDescriptionName": "video-360p",
          "AudioDescriptionNames": ["audio-128k"],
          "OutputSettings": {
            "MediaPackageOutputSettings": {}
          }
        }
      ]
    }
  ],
  "Destinations": [
    {
      "Id": "mediapackage-dest",
      "MediaPackageSettings": [
        {
          "ChannelId": "my-mediapackage-channel-id"
        }
      ]
    }
  ]
}
```

> **Lưu ý:** Với Standard Channel, cấu hình `MediaPackageSettings` chỉ cần 1 `ChannelId`. MediaLive tự động route Pipeline A đến ingest endpoint 1 và Pipeline B đến ingest endpoint 2 của cùng MediaPackage Channel.

---

## 3. S3 — Lưu Trữ và Archive

**Archive Output Group** lưu toàn bộ luồng live stream dưới dạng MPEG-2 TS segments lên S3. Phù hợp để:
- Tạo VOD archive từ live stream
- Lưu trữ dài hạn
- Replay và highlight clip

### Archive Output Group

```json
{
  "OutputGroups": [
    {
      "Name": "Archive",
      "OutputGroupSettings": {
        "ArchiveGroupSettings": {
          "Destination": {
            "DestinationRefId": "archive-s3"
          },
          "RolloverInterval": 300,
          "SegmentLength": 10
        }
      },
      "Outputs": [
        {
          "OutputName": "archive-1080p",
          "VideoDescriptionName": "video-1080p",
          "AudioDescriptionNames": ["audio-128k"],
          "OutputSettings": {
            "ArchiveOutputSettings": {
              "ContainerSettings": {
                "Container": "MPEGTS",
                "M2tsSettings": {
                  "Scte35Source": "PASSTHROUGH",
                  "Scte35Pid": "500",
                  "ProgramNum": 1,
                  "PcrPeriod": 100
                }
              },
              "Extension": "ts",
              "NameModifier": "_archive"
            }
          }
        }
      ]
    }
  ],
  "Destinations": [
    {
      "Id": "archive-s3",
      "Settings": [
        {
          "Url": "s3ssl://my-archive-bucket/live-archive/2026/06/03/stream-a/"
        },
        {
          "Url": "s3ssl://my-archive-bucket/live-archive/2026/06/03/stream-b/"
        }
      ]
    }
  ]
}
```

**RolloverInterval** (giây): MediaLive tạo file mới sau mỗi khoảng thời gian này. Ví dụ `300` = mỗi 5 phút tạo 1 file `.ts` mới.

### Tên File Archive

MediaLive đặt tên file theo pattern:
```
{DestinationPath}{NameModifier}_{timestamp}.ts

Ví dụ: s3://bucket/stream-a/archive_20260603T080000.ts
                              archive_20260603T080500.ts
                              archive_20260603T081000.ts
```

---

## 4. RTMP Push — Phát Đồng Thời Lên Nền Tảng Khác

**RTMP Output Group** cho phép MediaLive đẩy (push) luồng RTMP đến nền tảng streaming khác cùng lúc — ví dụ: phát đồng thời lên YouTube Live, Facebook Live, Twitch.

### Cấu Trúc RTMP Push

```
MediaLive Channel
  ├── MediaPackage Output  →  Viewer trên website/app của bạn
  └── RTMP Push Output     →  YouTube/Facebook/Twitch (đồng thời)
```

### Ví Dụ: Phát Lên YouTube Live

```json
{
  "OutputGroups": [
    {
      "Name": "YouTube Live",
      "OutputGroupSettings": {
        "RtmpGroupSettings": {
          "AuthenticationScheme": "COMMON",
          "CacheLength": 30,
          "CacheFullBehavior": "DISCONNECT_IMMEDIATELY",
          "RestartDelay": 15,
          "InputLossAction": "EMIT_OUTPUT",
          "AdMarkers": ["ON_CUE_POINT_SCTE35"]
        }
      },
      "Outputs": [
        {
          "OutputName": "youtube-1080p",
          "VideoDescriptionName": "video-1080p",
          "AudioDescriptionNames": ["audio-128k"],
          "OutputSettings": {
            "RtmpOutputSettings": {
              "Destination": {
                "DestinationRefId": "youtube-dest"
              },
              "ConnectionRetryInterval": 2,
              "NumRetries": 10,
              "CertificateMode": "VERIFY_AUTHENTICITY"
            }
          }
        }
      ]
    }
  ],
  "Destinations": [
    {
      "Id": "youtube-dest",
      "Settings": [
        {
          "Url": "rtmps://a.rtmp.youtube.com/live2",
          "StreamName": "YOUR_STREAM_KEY_A"
        },
        {
          "Url": "rtmps://b.rtmp.youtube.com/live2",
          "StreamName": "YOUR_STREAM_KEY_B"
        }
      ]
    }
  ]
}
```

### Các Nền Tảng RTMP Phổ Biến

| Nền Tảng | RTMP URL | Port |
|---------|---------|------|
| **YouTube Live** | `rtmps://a.rtmp.youtube.com/live2` | 443 |
| **Facebook Live** | `rtmps://live-api-s.facebook.com:443/rtmp/` | 443 |
| **Twitch** | `rtmp://live.twitch.tv/app/` | 1935 |
| **Tự host (Wowza)** | `rtmp://your-server:1935/live/` | 1935 |

> **Quan trọng:** Khi dùng Standard Channel (2 pipelines), cần 2 RTMP destinations (A và B). Nếu nền tảng đích chỉ cấp 1 stream key, điền cùng URL+key cho cả 2.

---

## 5. HLS Output Group — HLS Trực Tiếp Lên S3/HTTP

**HLS Group** ghi HLS segments (`.ts` hoặc `.fmp4`) và manifest (`.m3u8`) trực tiếp lên S3 hoặc HTTP server. Khác với MediaPackage, không có packaging tự động.

### Khi Nào Dùng HLS Group (Thay Vì MediaPackage)?

- Không cần DRM
- Không cần time-shift
- Muốn kiểm soát hoàn toàn cách tổ chức file trên S3
- Cost-sensitive: tránh chi phí MediaPackage khi scale thấp

### Cấu Hình HLS Group

```json
{
  "OutputGroups": [
    {
      "Name": "HLS to S3",
      "OutputGroupSettings": {
        "HlsGroupSettings": {
          "Destination": {
            "DestinationRefId": "hls-s3-dest"
          },
          "HlsCdnSettings": {
            "HlsS3Settings": {
              "CannedAcl": "BUCKET_OWNER_FULL_CONTROL"
            }
          },
          "Mode": "LIVE",
          "SegmentLength": 6,
          "MinSegmentLength": 0,
          "IndexNSegments": 3,
          "KeepSegments": 21,
          "ProgramDateTime": "INCLUDE",
          "ProgramDateTimePeriod": 600,
          "TimedMetadataId3Frame": "PRIV",
          "TimedMetadataId3Period": 10,
          "CodecSpecification": "RFC_4281",
          "ManifestCompression": "NONE",
          "ManifestDurationFormat": "FLOATING_POINT",
          "OutputSelection": "MANIFESTS_AND_SEGMENTS",
          "StreamInfResolution": "INCLUDE",
          "IvInManifest": "EXCLUDE",
          "InputLossAction": "EMIT_OUTPUT",
          "IncompleteSegmentBehavior": "AUTO"
        }
      },
      "Outputs": [
        {
          "OutputName": "1080p",
          "VideoDescriptionName": "video-1080p",
          "AudioDescriptionNames": ["audio-128k"],
          "OutputSettings": {
            "HlsOutputSettings": {
              "HlsSettings": {
                "StandardHlsSettings": {
                  "M3u8Settings": {
                    "Scte35Source": "PASSTHROUGH"
                  }
                }
              },
              "NameModifier": "_1080p",
              "SegmentModifier": "$Number%05d$"
            }
          }
        }
      ]
    }
  ],
  "Destinations": [
    {
      "Id": "hls-s3-dest",
      "Settings": [
        { "Url": "s3ssl://my-bucket/live/stream-a/" },
        { "Url": "s3ssl://my-bucket/live/stream-b/" }
      ]
    }
  ]
}
```

### Thông Số HLS Quan Trọng

| Thông Số | Giá Trị Thông Dụng | Ý Nghĩa |
|---------|-------------------|---------|
| `SegmentLength` | 4–6 giây | Độ dài mỗi HLS segment |
| `IndexNSegments` | 3 | Số segment trong manifest (window viewer) |
| `KeepSegments` | 21 (= 3 × SegmentLength × 60/SegLen) | Giữ 21 segment trên S3 trước khi xoá |
| `Mode` | `LIVE` hoặc `VOD` | LIVE: manifest cập nhật liên tục; VOD: manifest cố định |

---

## 6. UDP/TS — MPEG-2 TS Qua UDP

**UDP Output** gửi luồng MPEG-2 TS qua UDP đến một địa chỉ IP:port. Thường dùng cho:
- Phân phối trong mạng nội bộ (LAN/WAN broadcast)
- Kết nối với thiết bị broadcast (modulator, DVB encoder)
- IRD — Integrated Receiver Decoder trong hệ thống phân phối vệ tinh

```json
{
  "OutputGroups": [
    {
      "Name": "UDP Output",
      "OutputGroupSettings": {
        "UdpGroupSettings": {
          "InputLossAction": "EMIT_PROGRAM"
        }
      },
      "Outputs": [
        {
          "OutputSettings": {
            "UdpOutputSettings": {
              "Destination": {
                "DestinationRefId": "udp-dest"
              },
              "BufferMsec": 1000,
              "ContainerSettings": {
                "M2tsSettings": {
                  "Bitrate": 12000000,
                  "BufferModel": "MULTIPLEX",
                  "Scte35Source": "PASSTHROUGH"
                }
              }
            }
          },
          "VideoDescriptionName": "video-1080p",
          "AudioDescriptionNames": ["audio-128k"]
        }
      ]
    }
  ],
  "Destinations": [
    {
      "Id": "udp-dest",
      "Settings": [
        { "Url": "udp://192.168.1.100:5000" },
        { "Url": "udp://192.168.1.101:5000" }
      ]
    }
  ]
}
```

---

## 7. Frame Capture — Chụp Thumbnail

**Frame Capture Output Group** chụp ảnh tĩnh (JPEG/PNG) định kỳ từ luồng video và lưu lên S3. Dùng để:
- Tạo thumbnail preview cho live stream trên website
- Monitoring hình ảnh (AI/ML xử lý thumbnails)
- Compliance recording (lưu ảnh để kiểm tra nội dung)

```json
{
  "OutputGroups": [
    {
      "Name": "Thumbnails",
      "OutputGroupSettings": {
        "FrameCaptureGroupSettings": {
          "Destination": {
            "DestinationRefId": "thumbnail-s3"
          },
          "FrameCaptureCdnSettings": {
            "FrameCaptureS3Settings": {
              "CannedAcl": "PUBLIC_READ"
            }
          }
        }
      },
      "Outputs": [
        {
          "OutputSettings": {
            "FrameCaptureOutputSettings": {
              "NameModifier": "_thumb"
            }
          },
          "VideoDescriptionName": "video-thumbnail",
          "OutputName": "thumbnail"
        }
      ]
    }
  ]
}
```

Video Description cho thumbnail (resolution nhỏ):
```json
{
  "Name": "video-thumbnail",
  "Width": 640,
  "Height": 360,
  "CodecSettings": {
    "FrameCaptureSettings": {
      "CaptureInterval": 5,
      "CaptureIntervalUnits": "SECONDS",
      "Quality": 80
    }
  }
}
```

---

## 8. Đa Đích Đến (Multiple Destinations)

Đây là cấu hình thực tế của một live channel production đầy đủ:

```
MediaLive Standard Channel
  │
  ├── Output Group 1: MediaPackage
  │     → MediaPackage → CloudFront → Viewer (HLS/DASH với DRM)
  │
  ├── Output Group 2: Archive (S3)
  │     → S3 → VOD sau sự kiện, replay clip
  │
  ├── Output Group 3: RTMP Push (YouTube)
  │     → YouTube Live (phát đồng thời)
  │
  └── Output Group 4: Frame Capture (S3)
        → Thumbnail mỗi 5 giây → Website preview
```

### Lưu Ý Về Giới Hạn

| Giới Hạn | Giá Trị Mặc Định | Mô Tả |
|---------|----------------|-------|
| Output Groups / Channel | 7 | Tổng số Output Group tối đa |
| Outputs / Output Group | 20 | Số rendition trong 1 group |
| RTMP Outputs / Channel | 5 | Giới hạn số đích RTMP |

---

## 9. IAM Permissions — Quyền Cần Thiết

MediaLive cần IAM Role — Identity and Access Management Role với các quyền sau tuỳ theo Output Group:

### Quyền Cơ Bản Cho Mọi Channel

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:/aws/medialive/*"
    }
  ]
}
```

### Quyền Cho S3 Output (Archive / HLS / Thumbnail)

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-live-bucket",
        "arn:aws:s3:::my-live-bucket/*"
      ]
    }
  ]
}
```

### Quyền Cho MediaPackage Output

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "mediapackage:DescribeChannel"
      ],
      "Resource": "arn:aws:mediapackage:ap-southeast-1:*:channels/*"
    }
  ]
}
```

### Quyền Cho MediaConnect Input (nếu dùng)

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "mediaconnect:ManagedDescribeFlow",
        "mediaconnect:ManagedAddOutput",
        "mediaconnect:ManagedRemoveOutput"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Best Practices

```
✅ NÊN làm:
- Dùng MediaPackage thay HLS-to-S3 khi cần DRM hoặc time-shift
- Cấu hình Archive Output để luôn có bản lưu của live event
- Dùng s3ssl:// (thay vì s3://) cho mọi S3 destination (mã hoá TLS)
- Đặt Lifecycle Policy trên S3 bucket archive để xoá file cũ tự động
- Khi RTMP Push đến nền tảng ngoài: dùng RTMPS (443) thay RTMP (1935)

❌ KHÔNG nên làm:
- Lưu HLS segments với KeepSegments = 0 (xoá hết, viewer không xem được)
- Dùng cùng 1 S3 bucket path cho Pipeline A và B (ghi đè lên nhau)
- Quên cấp quyền IAM cho đích output mới (hay gặp lỗi AccessDenied)
- Dùng HLS Group trực tiếp cho production có viewer lớn (không scale như MediaPackage)
```

---

**Phần Trước:** [3-redundancy-failover.md](./3-redundancy-failover.md) — Redundancy & Failover
**Phần Tiếp Theo:** [5-schedule-scte35.md](./5-schedule-scte35.md) — Schedule Actions & SCTE-35
