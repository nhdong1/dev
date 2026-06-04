# Schedule Actions & SCTE-35 — Lịch Phát Sóng và Đánh Dấu Quảng Cáo

> Schedule Actions cho phép tự động hoá các hành động trong live channel theo thời gian thực. SCTE-35 là chuẩn broadcast dùng để đánh dấu điểm chèn quảng cáo — hai tính năng này kết hợp tạo nên nền tảng cho kênh OTT hoàn chỉnh.

## 📚 Mục Lục

1. [Schedule Actions — Lịch Phát Sóng](#1-schedule-actions--lịch-phát-sóng)
2. [Input Switch — Chuyển Nguồn Phát](#2-input-switch--chuyển-nguồn-phát)
3. [SCTE-35 — Chuẩn Đánh Dấu Quảng Cáo](#3-scte-35--chuẩn-đánh-dấu-quảng-cáo)
4. [Splice Insert — Chèn Điểm Quảng Cáo](#4-splice-insert--chèn-điểm-quảng-cáo)
5. [Time Signal — Tín Hiệu Thời Gian](#5-time-signal--tín-hiệu-thời-gian)
6. [Return to Network — Kết Thúc Quảng Cáo](#6-return-to-network--kết-thúc-quảng-cáo)
7. [Motion Graphics Overlay — Đồ Hoạ Động](#7-motion-graphics-overlay--đồ-hoạ-động)
8. [Pipeline Pause — Tạm Dừng Pipeline](#8-pipeline-pause--tạm-dừng-pipeline)
9. [Tích Hợp Với MediaTailor](#9-tích-hợp-với-mediatailor)
10. [Quản Lý Schedule Qua AWS CLI](#10-quản-lý-schedule-qua-aws-cli)

---

## 1. Schedule Actions — Lịch Phát Sóng

**Schedule Actions** là cơ chế lên lịch các hành động sẽ xảy ra trong Channel tại thời điểm xác định hoặc theo điều kiện. Thay vì phải can thiệp thủ công vào lúc đêm khuya, bạn có thể lên lịch sẵn tất cả hành động.

### Các Loại Schedule Action

| Loại Action | Mô Tả |
|-----------|-------|
| **Input Switch** | Chuyển nguồn phát sang input khác |
| **Input Prepare** | Chuẩn bị input trước khi switch (giảm độ trễ) |
| **SCTE-35 Splice Insert** | Chèn điểm bắt đầu quảng cáo |
| **SCTE-35 Time Signal** | Gửi tín hiệu thời gian SCTE-35 |
| **SCTE-35 Return to Network** | Đánh dấu kết thúc quảng cáo |
| **Static Image Activate** | Bật overlay ảnh tĩnh |
| **Static Image Deactivate** | Tắt overlay ảnh tĩnh |
| **Motion Graphics Activate** | Bật đồ hoạ động |
| **Motion Graphics Deactivate** | Tắt đồ hoạ động |
| **Pause** | Tạm dừng pipeline (phát frame cuối) |
| **Unpause** | Tiếp tục pipeline |

### Loại Timing (Thời Điểm Thực Thi)

| Timing | Ý Nghĩa | Dùng Cho |
|--------|---------|---------|
| `FIXED` | Thực thi vào thời gian UTC cụ thể | Lịch phát sóng cố định |
| `IMMEDIATE` | Thực thi ngay khi Schedule Action được tạo | Can thiệp khẩn cấp |
| `FOLLOW_START` | Thực thi X giây sau khi một action khác bắt đầu | Chuỗi action liên tiếp |
| `FOLLOW_END` | Thực thi X giây sau khi một action khác kết thúc | Input switch liên tiếp |

---

## 2. Input Switch — Chuyển Nguồn Phát

**Input Switch** cho phép chuyển nguồn phát giữa các inputs đã được attach vào Channel. Đây là cơ chế cốt lõi của kênh truyền hình OTT: chuyển giữa chương trình trực tiếp, phim, quảng cáo, v.v.

### Ví Dụ: Lịch Phát Sóng OTT

```
08:00  → Switch to Input "morning-news-rtmp"     (tin tức sáng - live)
10:00  → Switch to Input "documentary-mp4"       (phim tài liệu - file)
12:00  → Switch to Input "noon-news-rtmp"         (tin tức trưa - live)
13:00  → Switch to Input "entertainment-mp4"     (giải trí - file)
18:00  → Switch to Input "evening-news-rtmp"     (tin tức chiều - live)
21:00  → Switch to Input "prime-time-show-rtmp"  (show tối - live)
```

### Cấu Hình Input Switch

```json
{
  "ScheduleActions": [
    {
      "ActionName": "switch-to-morning-news",
      "ScheduleActionStartSettings": {
        "FixedModeScheduleActionStartSettings": {
          "Time": "2026-06-03T01:00:00.000Z"
        }
      },
      "ScheduleActionSettings": {
        "InputSwitchSettings": {
          "InputAttachmentNameReference": "morning-news-input",
          "InputClippingSettings": {
            "InputTimecodeSource": "ZEROBASED",
            "StartTimecode": {
              "Timecode": "00:00:00:00"
            }
          }
        }
      }
    },
    {
      "ActionName": "switch-to-documentary",
      "ScheduleActionStartSettings": {
        "FixedModeScheduleActionStartSettings": {
          "Time": "2026-06-03T03:00:00.000Z"
        }
      },
      "ScheduleActionSettings": {
        "InputSwitchSettings": {
          "InputAttachmentNameReference": "documentary-mp4-input"
        }
      }
    }
  ]
}
```

### Input Prepare — Chuẩn Bị Trước Khi Switch

**Input Prepare** pre-connect đến input source trước khi Input Switch xảy ra, giảm độ trễ khi switch:

```json
{
  "ActionName": "prepare-morning-news",
  "ScheduleActionStartSettings": {
    "FollowModeScheduleActionStartSettings": {
      "ReferenceActionName": "switch-to-documentary",
      "FollowPoint": "END"
    }
  },
  "ScheduleActionSettings": {
    "InputPrepareSettings": {
      "InputAttachmentNameReference": "morning-news-input",
      "InputClippingSettings": {
        "InputTimecodeSource": "ZEROBASED"
      }
    }
  }
}
```

> **Best practice:** Luôn tạo Input Prepare action 30–60 giây trước Input Switch cho RTMP/HLS Pull inputs. Giúp tránh glitch khi switch.

---

## 3. SCTE-35 — Chuẩn Đánh Dấu Quảng Cáo

**SCTE-35** — Society of Cable Telecommunications Engineers 35 là chuẩn kỹ thuật được dùng trong broadcast để nhúng tín hiệu điều khiển vào luồng video. Tín hiệu này cho biết:
- Khi nào bắt đầu break quảng cáo (ad avail — ad availability window)
- Khi nào kết thúc break quảng cáo
- Metadata liên quan (duration, splice event ID, program ID)

### Tại Sao SCTE-35 Quan Trọng?

```
Pipeline SCTE-35 trong hệ thống OTT:

MediaLive (chèn SCTE-35 markers)
     │
     ▼
MediaPackage (đọc SCTE-35, chuyển thành HLS EXT-X-CUE-OUT markers)
     │
     ▼
MediaTailor (đọc EXT-X-CUE-OUT, chèn quảng cáo cá nhân hoá vào từng viewer)
     │
     ▼
CloudFront → Viewer
```

Không có SCTE-35 → MediaTailor không biết khi nào chèn quảng cáo → **không có SSAI — Server-Side Ad Insertion**.

### Nguồn SCTE-35

MediaLive nhận SCTE-35 từ 3 nguồn:

| Nguồn | Mô Tả |
|-------|-------|
| **Passthrough** | Nhận SCTE-35 từ input source và chuyển tiếp (với MPEG-2 TS input) |
| **Schedule Actions** | Chèn SCTE-35 theo lịch từ MediaLive Schedule |
| **API** | Chèn SCTE-35 thông qua API call (programmatic) |

---

## 4. Splice Insert — Chèn Điểm Quảng Cáo

**Splice Insert** là loại SCTE-35 message phổ biến nhất. Nó đánh dấu điểm bắt đầu **và** kết thúc của một ad break.

### Cấu Trúc Splice Insert

```
Luồng video timeline:

[... Nội dung chương trình ...]
                              ↑ Splice Point (điểm chuyển)
                              │  SCTE-35 Splice Insert (duration = 60 giây)
                              ↓
[... 60 giây quảng cáo ...]
                              ↑ Return to Network
                              │
                              ↓
[... Nội dung chương trình tiếp tục ...]
```

### Tạo Splice Insert Qua Schedule

```json
{
  "ScheduleActions": [
    {
      "ActionName": "ad-break-21h30",
      "ScheduleActionStartSettings": {
        "FixedModeScheduleActionStartSettings": {
          "Time": "2026-06-03T14:30:00.000Z"
        }
      },
      "ScheduleActionSettings": {
        "Scte35SpliceInsertSettings": {
          "SpliceEventId": 1001,
          "Duration": 1350000
        }
      }
    }
  ]
}
```

**Giải thích `Duration`:**
- Đơn vị: 90 kHz clock ticks (1 giây = 90,000 ticks)
- Ví dụ: 60 giây = 60 × 90,000 = 5,400,000
- Ví dụ: 15 giây = 15 × 90,000 = 1,350,000

> **Lưu ý:** `SpliceEventId` phải duy nhất trong phiên phát. Thường dùng số tăng dần hoặc timestamp-based ID.

---

## 5. Time Signal — Tín Hiệu Thời Gian

**Time Signal** là loại SCTE-35 message linh hoạt hơn Splice Insert, dùng để mang nhiều loại metadata qua UPID — Unique Program Identifier hoặc Segmentation Descriptor. MediaTailor và các hệ thống ad server hiện đại thường dùng Time Signal.

### Segmentation Type (Loại Phân Đoạn)

| Segmentation Type | Hex | Ý Nghĩa |
|------------------|-----|---------|
| `Program Start` | 0x10 | Bắt đầu chương trình mới |
| `Program End` | 0x11 | Kết thúc chương trình |
| `Break Start` | 0x22 | Bắt đầu ad break |
| `Break End` | 0x23 | Kết thúc ad break |
| `Provider Ad Start` | 0x34 | Bắt đầu quảng cáo của nhà cung cấp |
| `Provider Ad End` | 0x35 | Kết thúc quảng cáo của nhà cung cấp |

### Cấu Hình Time Signal

```json
{
  "ActionName": "time-signal-break-start",
  "ScheduleActionStartSettings": {
    "FixedModeScheduleActionStartSettings": {
      "Time": "2026-06-03T14:30:00.000Z"
    }
  },
  "ScheduleActionSettings": {
    "Scte35TimeSignalSettings": {
      "Scte35Descriptors": [
        {
          "Scte35DescriptorSettings": {
            "SegmentationDescriptorScte35DescriptorSettings": {
              "SegmentationEventId": 2001,
              "SegmentationCancelIndicator": "SEGMENTATION_EVENT_NOT_CANCELED",
              "SegmentationDuration": 5400000,
              "SegmentationTypeId": 52,
              "SegmentNum": 1,
              "SegmentsExpected": 1,
              "DeliveryRestrictions": {
                "ArchiveAllowedFlag": "ARCHIVE_ALLOWED",
                "DeviceRestrictions": "NONE",
                "NoRegionalBlackoutFlag": "REGIONAL_BLACKOUT",
                "WebDeliveryAllowedFlag": "WEB_DELIVERY_NOT_ALLOWED"
              }
            }
          }
        }
      ]
    }
  }
}
```

---

## 6. Return to Network — Kết Thúc Quảng Cáo

**Return to Network** đánh dấu kết thúc ad break, báo hiệu cho hệ thống downstream quay lại phát nội dung chương trình.

```json
{
  "ActionName": "return-to-network-21h31",
  "ScheduleActionStartSettings": {
    "FollowModeScheduleActionStartSettings": {
      "ReferenceActionName": "ad-break-21h30",
      "FollowPoint": "END"
    }
  },
  "ScheduleActionSettings": {
    "Scte35ReturnToNetworkSettings": {
      "SpliceEventId": 1001
    }
  }
}
```

> **Lưu ý:** `SpliceEventId` trong Return to Network phải khớp với `SpliceEventId` trong Splice Insert tương ứng.

---

## 7. Motion Graphics Overlay — Đồ Hoạ Động

**Motion Graphics** cho phép overlay đồ hoạ động (animated graphics) lên luồng video. Thường dùng cho:
- Lower thirds (tên phóng viên, thông tin tóm tắt ở góc dưới màn hình)
- Logo watermark animation
- Bug (logo kênh thường trực ở góc màn hình)
- Breaking news ticker

### Cấu Hình Motion Graphics

```json
{
  "ActionName": "activate-breaking-news-ticker",
  "ScheduleActionStartSettings": {
    "ImmediateModeScheduleActionStartSettings": {}
  },
  "ScheduleActionSettings": {
    "MotionGraphicsImageActivateSettings": {
      "Duration": 30000,
      "StartDateTime": "2026-06-03T14:30:00.000Z",
      "Url": "s3ssl://my-bucket/graphics/breaking-news-ticker.html",
      "Username": "",
      "PasswordParam": ""
    }
  }
}
```

> Motion Graphics trong MediaLive dùng HTML5 + CSS animations, được render trực tiếp lên frame video.

---

## 8. Pipeline Pause — Tạm Dừng Pipeline

**Pause** tạm dừng encoding của một pipeline, giữ nguyên frame cuối cùng. Dùng để:
- Xử lý tình huống khẩn cấp (cắt nội dung không phù hợp)
- Test failover giữa 2 pipelines

```json
{
  "ActionName": "pause-pipeline-0",
  "ScheduleActionStartSettings": {
    "ImmediateModeScheduleActionStartSettings": {}
  },
  "ScheduleActionSettings": {
    "PauseStateSettings": {
      "Pipelines": [
        {
          "PipelineId": "PIPELINE_0"
        }
      ]
    }
  }
}
```

---

## 9. Tích Hợp Với MediaTailor

**MediaTailor** — SSAI — Server-Side Ad Insertion đọc SCTE-35 markers từ MediaPackage và chèn quảng cáo cá nhân hoá vào từng viewer. Đây là pipeline đầy đủ:

```
Bước 1: MediaLive chèn SCTE-35 (ad break 60 giây)
  └── Splice Insert: SpliceEventId=1001, Duration=5,400,000 (60s × 90,000)

Bước 2: MediaPackage chuyển đổi SCTE-35 sang HLS ad markers
  └── Manifest chứa:
      #EXT-X-DATERANGE:... SCTE35-OUT=0xFC003C...
      #EXT-X-CUE-OUT:60
      ... (segments của ad break) ...
      #EXT-X-CUE-IN

Bước 3: MediaTailor nhận playback request từ viewer
  └── Gọi ADS (Ad Decision Server) lấy VAST response
  └── Thay thế 60 giây ad break bằng quảng cáo cá nhân hoá
  └── Gửi manifest đã sửa đổi về cho player của viewer

Bước 4: Player viewer phát quảng cáo (không bị ad-blocker chặn vì cùng domain)
```

### SCTE-35 Passthrough vs Generated

| Cách | Mô Tả | Dùng Khi |
|------|-------|---------|
| **Passthrough** | Chuyển tiếp SCTE-35 từ encoder nguồn | Encoder nguồn đã có SCTE-35 (broadcast hardware) |
| **Generated** | MediaLive tạo SCTE-35 theo Schedule | Encoder nguồn không có SCTE-35, tự lên lịch |

Cấu hình Passthrough trong Output Group:
```json
{
  "M2tsSettings": {
    "Scte35Source": "PASSTHROUGH",
    "Scte35Pid": "500"
  }
}
```

---

## 10. Quản Lý Schedule Qua AWS CLI

### Thêm Schedule Action

```bash
aws medialive batch-update-schedule \
  --region ap-southeast-1 \
  --channel-id CHANNEL_ID \
  --creates '{
    "ScheduleActions": [
      {
        "ActionName": "ad-break-now",
        "ScheduleActionStartSettings": {
          "ImmediateModeScheduleActionStartSettings": {}
        },
        "ScheduleActionSettings": {
          "Scte35SpliceInsertSettings": {
            "SpliceEventId": 9999,
            "Duration": 5400000
          }
        }
      }
    ]
  }'
```

### Xem Schedule Hiện Tại

```bash
aws medialive describe-schedule \
  --region ap-southeast-1 \
  --channel-id CHANNEL_ID \
  --query "ScheduleActions[*].{Name:ActionName, Time:ScheduleActionStartSettings, Type:ScheduleActionSettings}" \
  --output table
```

### Xoá Schedule Action

```bash
aws medialive batch-update-schedule \
  --region ap-southeast-1 \
  --channel-id CHANNEL_ID \
  --deletes '{
    "ActionNames": ["ad-break-21h30", "return-to-network-21h31"]
  }'
```

### Ví Dụ: Script Tự Động Lên Lịch Quảng Cáo

```python
import boto3
from datetime import datetime, timezone, timedelta

def schedule_ad_break(channel_id, break_time_utc, duration_seconds=60):
    """
    Lên lịch ad break tại thời điểm xác định.
    break_time_utc: datetime object (UTC)
    duration_seconds: độ dài quảng cáo (giây)
    """
    client = boto3.client("medialive", region_name="ap-southeast-1")

    splice_event_id = int(break_time_utc.timestamp())  # Dùng timestamp làm unique ID
    duration_ticks = duration_seconds * 90000           # Chuyển sang 90kHz clock ticks

    # Tạo Return to Network sau khi Splice Insert kết thúc
    splice_action_name = f"splice-insert-{splice_event_id}"
    return_action_name = f"return-to-network-{splice_event_id}"

    response = client.batch_update_schedule(
        ChannelId=channel_id,
        Creates={
            "ScheduleActions": [
                {
                    "ActionName": splice_action_name,
                    "ScheduleActionStartSettings": {
                        "FixedModeScheduleActionStartSettings": {
                            "Time": break_time_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z")
                        }
                    },
                    "ScheduleActionSettings": {
                        "Scte35SpliceInsertSettings": {
                            "SpliceEventId": splice_event_id,
                            "Duration": duration_ticks
                        }
                    }
                },
                {
                    "ActionName": return_action_name,
                    "ScheduleActionStartSettings": {
                        "FollowModeScheduleActionStartSettings": {
                            "ReferenceActionName": splice_action_name,
                            "FollowPoint": "END"
                        }
                    },
                    "ScheduleActionSettings": {
                        "Scte35ReturnToNetworkSettings": {
                            "SpliceEventId": splice_event_id
                        }
                    }
                }
            ]
        }
    )
    return response


# Ví dụ sử dụng:
# Lên lịch ad break 60 giây vào lúc 21:30 UTC
ad_time = datetime(2026, 6, 3, 14, 30, 0, tzinfo=timezone.utc)
schedule_ad_break("my-channel-id", ad_time, duration_seconds=60)
```

---

## Best Practices

```
✅ NÊN làm:
- Luôn tạo Return to Network action cùng lúc với Splice Insert
- Dùng timestamp-based SpliceEventId để đảm bảo tính duy nhất
- Tạo Input Prepare action 30–60 giây trước Input Switch
- Lên lịch quảng cáo sớm (ít nhất 5 phút trước khi phát)
- Monitor SCTE-35 signal qua CloudWatch để phát hiện lỗi kịp thời
- Test toàn bộ SCTE-35 pipeline với MediaTailor trên môi trường staging trước production

❌ KHÔNG nên làm:
- Dùng cùng SpliceEventId cho nhiều ad break (gây nhầm lẫn cho downstream)
- Lên lịch action trong quá khứ (sẽ không được thực thi)
- Quên Passthrough SCTE-35 trong Output Group nếu muốn MediaPackage nhận markers
- Để Schedule queue quá nhiều actions không cần thiết (khó debug)
```

---

## Tổng Kết: Pipeline Kênh OTT Đầy Đủ

```
Lịch phát hàng ngày (ví dụ):

08:00 UTC  [Schedule] Input Switch → "morning-news-live" (RTMP)
           [Schedule] SCTE-35 chèn điểm quảng cáo 8:15, 8:30, 8:45, 9:00
           [Schedule] Motion Graphics "Good Morning Channel" activate

10:00 UTC  [Schedule] Input Switch → "documentary-file" (MP4)
           [Schedule] Static Image Activate (logo kênh)

12:00 UTC  [Schedule] Input Switch → "noon-news-live" (RTMP)
           [Schedule] SCTE-35 chèn điểm quảng cáo 12:15, 12:30

           MediaLive → MediaPackage → MediaTailor (SSAI)
                       ↓ mỗi viewer nhận quảng cáo cá nhân hoá khác nhau
                       CloudFront → Viewer
```

---

**Phần Trước:** [4-output-destinations.md](./4-output-destinations.md) — Output Destinations
**Phần Kế Tiếp:** [../04-mediapackage/README.md](../04-mediapackage/README.md) — MediaPackage: Packaging & DRM
