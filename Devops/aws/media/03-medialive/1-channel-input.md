# Channel & Input Types — Các Loại Đầu Vào MediaLive

> Input là cửa ngõ để MediaLive nhận luồng video. Chọn đúng loại Input và cấu hình đúng Security Group là bước đầu tiên khi thiết lập một live channel.

## 📚 Mục Lục

1. [Tổng Quan Về Input](#1-tổng-quan-về-input)
2. [RTMP Push — Encoder Đẩy Vào](#2-rtmp-push--encoder-đẩy-vào)
3. [RTMP Pull — MediaLive Kéo Từ Nguồn](#3-rtmp-pull--medialive-kéo-từ-nguồn)
4. [RTP Push — Broadcast Chuyên Nghiệp](#4-rtp-push--broadcast-chuyên-nghiệp)
5. [HLS Pull — Kéo Từ Nguồn HLS](#5-hls-pull--kéo-từ-nguồn-hls)
6. [MediaConnect — Vận Chuyển Chất Lượng Cao](#6-mediaconnect--vận-chuyển-chất-lượng-cao)
7. [MP4 Pull — File Video (Slate/Filler)](#7-mp4-pull--file-video-slatefiller)
8. [Input Security Group — Kiểm Soát Truy Cập](#8-input-security-group--kiểm-soát-truy-cập)
9. [Input Specification — Khai Báo Tham Số Đầu Vào](#9-input-specification--khai-báo-tham-số-đầu-vào)
10. [Tạo Input Qua AWS CLI](#10-tạo-input-qua-aws-cli)

---

## 1. Tổng Quan Về Input

**Input** trong MediaLive là tài nguyên độc lập, được tạo trước rồi **attach** (đính kèm) vào Channel. Một Input có thể được dùng lại cho nhiều Channel (trừ RTMP Push — mỗi lần chỉ dùng được cho 1 Channel đang chạy).

### Bảng So Sánh Các Loại Input

| Loại Input | Giao Thức | Chiều | Dùng Cho | Redundancy |
|-----------|----------|-------|---------|-----------|
| **RTMP Push** | RTMP/RTMPS | Encoder → MediaLive | OBS, phần mềm encoder | 2 endpoints (A/B) |
| **RTMP Pull** | RTMP | MediaLive → Source | CDN RTMP stream có sẵn | 2 URL (A/B) |
| **RTP Push** | RTP/UDP | Encoder → MediaLive | Hardware encoder, OB van | 2 endpoints (A/B) |
| **HLS Pull** | HLS/HTTP | MediaLive → Source | HLS nguồn đã có | 2 URL (A/B) |
| **MediaConnect** | ZIXI/RTP | MediaConnect → MediaLive | Contribution feed chất lượng cao | 2 flows (A/B) |
| **MP4 Pull** | HTTP/S3 | S3/HTTP → MediaLive | Slate, filler content | 2 URL (A/B) |
| **Elemental Link** | SDI/HDMI | Hardware → MediaLive | AWS hardware encoder tại chỗ | 2 thiết bị |

> **Lưu ý về redundancy:** Channel Standard (2 pipelines) yêu cầu Input cũng có 2 endpoints (A và B). Pipeline A kết nối tới endpoint A, Pipeline B kết nối tới endpoint B.

---

## 2. RTMP Push — Encoder Đẩy Vào

**RTMP Push** là loại input phổ biến nhất. Encoder của bạn (OBS, Wirecast, FFmpeg...) **chủ động đẩy** luồng tới endpoint MediaLive cung cấp.

### Cách Hoạt Động

```
OBS / Encoder
     │
     │  RTMP Push (rtmp://endpoint:1935/live/stream-key)
     ▼
MediaLive Input Endpoint
  Pipeline A: rtmp://A_IP:1935/live/stream-key
  Pipeline B: rtmp://B_IP:1935/live/stream-key
     │
     ▼
MediaLive Channel (đang xử lý)
```

### Tạo RTMP Push Input

Khi tạo RTMP Push Input, MediaLive tự động cấp phát:
- **2 endpoints** (cho Standard Channel): mỗi endpoint gồm IP:port + stream key
- **Application name**: `/live` (mặc định, có thể tuỳ chỉnh)

```json
{
  "Name": "my-rtmp-push-input",
  "Type": "RTMP_PUSH",
  "InputSecurityGroups": ["sg-xxxxxxxx"],
  "Destinations": [
    { "StreamName": "live/mystream" }
  ]
}
```

**Sau khi tạo, MediaLive trả về:**
```
Pipeline A endpoint: rtmp://54.x.x.x:1935/live/mystream
Pipeline B endpoint: rtmp://18.x.x.x:1935/live/mystream
```

### Cấu Hình OBS Studio

```
OBS → Settings → Stream:
  Service: Custom
  Server: rtmp://54.x.x.x:1935/live
  Stream Key: mystream
```

> **Thực tế production:** Encoder phần cứng (Elemental Live, Haivision) thường hỗ trợ 2 outputs đồng thời — output 1 gửi đến Pipeline A endpoint, output 2 gửi đến Pipeline B endpoint. Đây là cách đảm bảo redundancy đầy đủ end-to-end.

---

## 3. RTMP Pull — MediaLive Kéo Từ Nguồn

**RTMP Pull** là MediaLive **chủ động kéo** luồng từ RTMP server của bạn. Hướng kết nối ngược lại với RTMP Push.

### Khi Nào Dùng RTMP Pull?

- Nguồn stream đã có sẵn trên RTMP server (ví dụ: CDN RTMP endpoint, Wowza)
- Không thể cấu hình encoder để push (network restriction)
- Tái sử dụng RTMP stream đang chạy vào nhiều pipeline

### Cấu Trúc URL

```
rtmp://source-server:1935/live/stream-key
```

```json
{
  "Name": "my-rtmp-pull-input",
  "Type": "RTMP_PULL",
  "Sources": [
    { "Url": "rtmp://source-a.example.com:1935/live/key" },
    { "Url": "rtmp://source-b.example.com:1935/live/key" }
  ]
}
```

> URL thứ 2 là backup cho Pipeline B. Nếu nguồn chỉ có 1 URL, có thể điền cùng URL cho cả A và B.

---

## 4. RTP Push — Broadcast Chuyên Nghiệp

**RTP — Real-time Transport Protocol** là giao thức vận chuyển video/audio chuyên nghiệp, thường dùng trong môi trường broadcast (đài truyền hình, sự kiện thể thao lớn).

### Đặc Điểm

- **UDP-based**: độ trễ thấp hơn TCP/RTMP, nhưng không tự recover khi mất packet
- **MPEG-2 TS container**: định dạng chuẩn broadcast, hỗ trợ DVB-Sub captions, SCTE-35 markers
- **Port**: thường dùng 5000–6000 (tuỳ cấu hình)

### Các Loại RTP

| Loại | Mô Tả | Use Case |
|------|-------|---------|
| **RTP_PUSH** | Encoder đẩy RTP/UDP đến MediaLive | Hardware encoder, satellite uplink |
| **UDP_PUSH** | Raw MPEG-2 TS qua UDP (không có RTP header) | Một số encoder cũ hoặc legacy system |

```json
{
  "Name": "my-rtp-input",
  "Type": "RTP_PUSH",
  "InputSecurityGroups": ["sg-xxxxxxxx"],
  "Destinations": [
    { "Port": "5000" },
    { "Port": "5001" }
  ]
}
```

---

## 5. HLS Pull — Kéo Từ Nguồn HLS

**HLS Pull** cho phép MediaLive kéo (pull) luồng HLS — HTTP Live Streaming từ một URL manifest đã có sẵn.

### Khi Nào Dùng HLS Pull?

- Nhận contribution feed từ đài truyền hình khác qua HLS
- Re-encode một luồng HLS đang phát để tạo bitrate ladder mới
- Test với HLS demo stream có sẵn trên internet

```json
{
  "Name": "my-hls-pull-input",
  "Type": "HLS_PULL",
  "Sources": [
    { "Url": "https://live.example.com/primary/index.m3u8" },
    { "Url": "https://backup.example.com/primary/index.m3u8" }
  ]
}
```

> **Lưu ý:** HLS Pull thêm độ trễ (vì MediaLive phải đợi segments từ nguồn). Không phù hợp khi cần độ trễ tối thiểu.

---

## 6. MediaConnect — Vận Chuyển Chất Lượng Cao

**AWS Elemental MediaConnect** là dịch vụ vận chuyển video chất lượng cao (contribution-grade transport) qua mạng IP. MediaConnect Input cho phép MediaLive nhận luồng từ MediaConnect Flow.

### Tại Sao Dùng MediaConnect?

- **Chất lượng cao, độ trễ thấp**: ZIXI protocol có cơ chế correction mà không cần tăng độ trễ nhiều
- **Bảo mật**: mã hoá end-to-end bằng AES-128/256
- **Multi-destination**: 1 MediaConnect Flow có thể phân phát đến nhiều MediaLive channels
- **Giám sát**: theo dõi packet loss, latency, bitrate realtime

### Kiến Trúc Với MediaConnect

```
On-premise Encoder (tại trường quay)
        │ ZIXI/RTP (qua Internet/AWS Direct Connect)
        ▼
AWS Elemental MediaConnect Flow
        │
        ├──▶ MediaLive Channel A (production)
        └──▶ MediaLive Channel B (backup)
```

```json
{
  "Name": "my-mediaconnect-input",
  "Type": "MEDIACONNECT",
  "MediaConnectFlows": [
    { "FlowArn": "arn:aws:mediaconnect:ap-southeast-1:123456:flow/primary" },
    { "FlowArn": "arn:aws:mediaconnect:ap-southeast-1:123456:flow/secondary" }
  ],
  "RoleArn": "arn:aws:iam::123456:role/MediaLiveAccessRole"
}
```

> MediaConnect Input yêu cầu IAM Role cho MediaLive để đọc từ MediaConnect Flow.

---

## 7. MP4 Pull — File Video (Slate/Filler)

**MP4 Pull** cho phép dùng file video (MP4 từ S3 hoặc HTTP) làm đầu vào. Đây **không phải** để phát video chính, mà chủ yếu dùng như:

- **Slate** — màn hình chờ: phát khi chưa có tín hiệu từ nguồn chính
- **Filler content**: phát trong khoảng trống giữa các chương trình
- **Test source**: test pipeline khi không có live signal

```json
{
  "Name": "slate-input",
  "Type": "MP4_FILE",
  "Sources": [
    { "Url": "s3ssl://my-bucket/slate/standby.mp4" },
    { "Url": "s3ssl://my-bucket/slate/standby.mp4" }
  ]
}
```

> Khi kết hợp với **Input Switching** qua Schedule Actions, MP4 Pull là cách phổ biến để xử lý khoảng trống lịch phát sóng tự động.

---

## 8. Input Security Group — Kiểm Soát Truy Cập

**Input Security Group** là danh sách CIDR — Classless Inter-Domain Routing — Định Tuyến Không Phân Lớp cho phép kết nối đến MediaLive Push endpoints. Chỉ áp dụng cho loại Push (RTMP Push, RTP Push).

### Tại Sao Cần Input Security Group?

MediaLive Push endpoints là public URL. Nếu không có Security Group, bất kỳ ai cũng có thể gửi stream vào channel của bạn → lãng phí tài nguyên và tốn chi phí.

### Cấu Hình

```json
{
  "WhitelistRules": [
    { "Cidr": "203.0.113.0/24" },    // IP của văn phòng / encoder on-premise
    { "Cidr": "198.51.100.50/32" },  // IP cố định của camera drone
    { "Cidr": "0.0.0.0/0" }         // KHÔNG KHUYẾN NGHỊ — mở hoàn toàn
  ]
}
```

> **Best practice:** Luôn giới hạn CIDR đến IP hoặc subnet của encoder nguồn. Tránh `0.0.0.0/0` trong môi trường production.

### Ví Dụ Tạo Security Group

```bash
aws medialive create-input-security-group \
  --whitelist-rules "Cidr=203.0.113.0/24" "Cidr=198.51.100.50/32"
```

---

## 9. Input Specification — Khai Báo Tham Số Đầu Vào

**Input Specification** là khai báo cho MediaLive biết về tham số tối đa của input, dùng để **tính giá** và cấp phát tài nguyên. Không phải giới hạn cứng — nếu input vượt quá spec sẽ bị tính giá tier cao hơn.

| Tham Số | Tùy Chọn | Ý Nghĩa |
|---------|---------|---------|
| **Input Codec** | `MPEG2`, `AVC` (H.264), `HEVC` (H.265) | Codec của luồng đầu vào |
| **Maximum Bitrate** | `MAX_10_MBPS`, `MAX_20_MBPS`, `MAX_50_MBPS` | Bitrate tối đa dự kiến |
| **Resolution** | `SD` (< 720p), `HD` (720p–1080p), `UHD` (> 1080p) | Độ phân giải tối đa |

```json
{
  "InputSpecification": {
    "Codec": "AVC",
    "Resolution": "HD",
    "MaximumBitrate": "MAX_20_MBPS"
  }
}
```

> **Lưu ý pricing:** Resolution `UHD` và bitrate `MAX_50_MBPS` tốn chi phí cao hơn đáng kể. Khai báo đúng resolution thực tế để tránh bị charge nhầm tier.

---

## 10. Tạo Input Qua AWS CLI

### Ví Dụ: Tạo RTMP Push Input Đầy Đủ

```bash
# Bước 1: Tạo Input Security Group
SECURITY_GROUP_ID=$(aws medialive create-input-security-group \
  --region ap-southeast-1 \
  --whitelist-rules "Cidr=203.0.113.0/24" \
  --query "SecurityGroup.Id" \
  --output text)

echo "Security Group ID: $SECURITY_GROUP_ID"

# Bước 2: Tạo RTMP Push Input
INPUT_ID=$(aws medialive create-input \
  --region ap-southeast-1 \
  --name "my-live-input" \
  --type RTMP_PUSH \
  --input-security-groups "$SECURITY_GROUP_ID" \
  --destinations "StreamName=live/mystream" "StreamName=live/mystream" \
  --query "Input.Id" \
  --output text)

echo "Input ID: $INPUT_ID"

# Bước 3: Xem endpoints được cấp phát
aws medialive describe-input \
  --region ap-southeast-1 \
  --input-id "$INPUT_ID" \
  --query "Input.Destinations[*].Url"
```

**Output (2 endpoints cho Standard Channel):**
```json
[
  "rtmp://54.251.x.x:1935/live/mystream",
  "rtmp://18.140.x.x:1935/live/mystream"
]
```

### Tạo HLS Pull Input

```bash
aws medialive create-input \
  --region ap-southeast-1 \
  --name "hls-contribution-input" \
  --type HLS_PULL \
  --sources \
    "Url=https://primary-source.example.com/live/index.m3u8" \
    "Url=https://backup-source.example.com/live/index.m3u8"
```

### Attach Input vào Channel

```json
{
  "InputAttachments": [
    {
      "InputAttachmentName": "live-source",
      "InputId": "INPUT_ID_HERE",
      "InputSettings": {
        "SourceEndBehavior": "CONTINUE",
        "InputFilter": "AUTO",
        "FilterStrength": 1,
        "DeblockFilter": "DISABLED",
        "DenoiseFilter": "DISABLED",
        "AudioSelectors": [
          {
            "Name": "default-audio",
            "SelectorSettings": {
              "AudioLanguageSelection": {
                "LanguageCode": "vie",
                "LanguageSelectionPolicy": "LOOSE"
              }
            }
          }
        ]
      }
    }
  ]
}
```

---

## Best Practices

```
✅ NÊN làm:
- Luôn cấu hình 2 endpoints/URL (A và B) cho Standard Channel
- Dùng Input Security Group với CIDR cụ thể, không dùng 0.0.0.0/0
- Đặt tên Input mô tả rõ ràng (nguồn, loại, môi trường)
- Khai báo Input Specification đúng với thực tế để tránh tính phí sai
- Dùng MediaConnect thay RTMP cho contribution feed quan trọng

❌ KHÔNG nên làm:
- Dùng cùng 1 URL cho cả Pipeline A và B (mất redundancy)
- Để Input Security Group mở hoàn toàn (0.0.0.0/0)
- Dùng MP4 Pull cho content chính (chỉ dùng cho slate/filler)
- Xoá Input đang được attach vào Channel đang RUNNING
```

---

**Phần Tiếp Theo:** [2-encoding-settings.md](./2-encoding-settings.md) — Video/Audio Encoding Settings, Bitrate Ladder
