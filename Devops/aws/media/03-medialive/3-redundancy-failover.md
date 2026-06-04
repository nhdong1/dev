# Redundancy & Failover — Tính Dự Phòng và Khắc Phục Sự Cố

> Đối với các sự kiện live quan trọng (thể thao, tin tức, concert), mất tín hiệu dù chỉ vài giây cũng ảnh hưởng nghiêm trọng đến trải nghiệm người xem. MediaLive cung cấp nhiều lớp bảo vệ để đảm bảo luồng phát liên tục.

## 📚 Mục Lục

1. [Standard vs Single Pipeline Channel](#1-standard-vs-single-pipeline-channel)
2. [Input Redundancy — Dự Phòng Đầu Vào](#2-input-redundancy--dự-phòng-đầu-vào)
3. [Automatic Input Failover — Chuyển Đổi Đầu Vào Tự Động](#3-automatic-input-failover--chuyển-đổi-đầu-vào-tự-động)
4. [Pipeline Locking — Đồng Bộ Hai Pipeline](#4-pipeline-locking--đồng-bộ-hai-pipeline)
5. [Input Loss Behavior — Hành Vi Khi Mất Tín Hiệu](#5-input-loss-behavior--hành-vi-khi-mất-tín-hiệu)
6. [CloudWatch Alarms & Monitoring — Giám Sát Và Cảnh Báo](#6-cloudwatch-alarms--monitoring--giám-sát-và-cảnh-báo)
7. [Kiến Trúc Redundancy Nhiều Tầng](#7-kiến-trúc-redundancy-nhiều-tầng)
8. [Chi Phí vs Độ Tin Cậy](#8-chi-phí-vs-độ-tin-cậy)

---

## 1. Standard vs Single Pipeline Channel

### Single Pipeline Channel (Kênh Một Đường Ống)

```
Input ──▶ Pipeline A ──▶ Output
            │
            └── Nếu lỗi: STREAM BỊ MẤT
                Cần vài phút để AWS khởi động lại
```

**Đặc điểm:**
- 1 pipeline duy nhất thực hiện encoding
- Chi phí thấp hơn Standard ~50%
- Nếu pipeline lỗi (hardware failure, software crash): stream bị mất ngay lập tức
- AWS cố gắng recovery tự động, nhưng mất 2–5 phút để khởi động lại
- **Phù hợp:** môi trường dev/test, stream không quan trọng, thử nghiệm cấu hình

### Standard Channel (Kênh Tiêu Chuẩn — 2 Đường Ống Song Song)

```
Input (A) ──▶ Pipeline A (Primary) ──▶ Output A
Input (B) ──▶ Pipeline B (Standby) ──▶ Output B
                  │
                  └── Cả hai pipeline encode ĐỒNG THỜI
                      MediaPackage/S3 nhận cả 2 luồng
                      Nếu A lỗi → B tiếp quản, không gián đoạn
```

**Đặc điểm:**
- 2 pipeline chạy song song và độc lập
- Mỗi pipeline nhận input riêng (Input endpoint A/B)
- Mỗi pipeline gửi output đến đích riêng (Output A/B)
- Nếu một pipeline lỗi, pipeline kia vẫn chạy → **Zero downtime**
- Phần downstream (MediaPackage) tự động dùng luồng còn lại
- **Phù hợp:** tất cả live stream production quan trọng

### So Sánh Chi Tiết

| Tiêu Chí | Single Pipeline | Standard Channel |
|---------|----------------|-----------------|
| **Số pipeline** | 1 | 2 |
| **Chi phí** | 1× | ~2× |
| **Uptime nếu 1 pipeline lỗi** | ❌ Stream mất | ✅ Stream tiếp tục |
| **Input endpoints** | 1 | 2 (A và B) |
| **Thời gian recovery** | 2–5 phút | 0 giây (pipeline kia đang chạy) |
| **Khuyến nghị production** | Không | Có |
| **Khi nào dùng Single** | Dev/test, không quan trọng | — |

---

## 2. Input Redundancy — Dự Phòng Đầu Vào

Redundancy không chỉ là pipeline — nguồn tín hiệu đầu vào (input source) cũng cần có dự phòng.

### Mô Hình End-to-End Redundancy

```
Encoder on-premise
  ├── Output 1 ──── RTMP → Pipeline A endpoint ──▶ Pipeline A
  └── Output 2 ──── RTMP → Pipeline B endpoint ──▶ Pipeline B

Nếu encoder chính lỗi:
  ├── Backup encoder ──── RTMP → Pipeline A endpoint ──▶ Pipeline A
  └── Backup encoder ──── RTMP → Pipeline B endpoint ──▶ Pipeline B
```

### Lưu Ý Quan Trọng Về Input Endpoints

Mỗi Input endpoint của Standard Channel là **độc lập**:
- Pipeline A chỉ nhận từ endpoint A
- Pipeline B chỉ nhận từ endpoint B

Nếu encoder chỉ push đến endpoint A và bỏ endpoint B → Pipeline B mất tín hiệu → **mất dự phòng**.

> **Best practice:** Luôn cấu hình encoder gửi đến cả 2 endpoints. Phần cứng encoder chuyên nghiệp (Elemental Live, Haivision Makito) hỗ trợ dual-output đồng thời.

---

## 3. Automatic Input Failover — Chuyển Đổi Đầu Vào Tự Động

**Automatic Input Failover** (AIF) là tính năng cho phép Channel tự động chuyển sang input dự phòng khi input chính mất tín hiệu, **mà không cần can thiệp thủ công**.

### Cấu Trúc AIF

```
InputAttachments:
  ├── Input 1 (Primary)   — nguồn chính (RTMP từ studio)
  └── Input 2 (Secondary) — nguồn dự phòng (RTMP backup hoặc MP4 slate)

Khi Input 1 mất tín hiệu:
  MediaLive tự động chuyển sang Input 2
  Khi Input 1 khôi phục:
    Tuỳ cấu hình: tự động trở về Input 1 hoặc giữ nguyên Input 2
```

### Điều Kiện Kích Hoạt Failover

| Điều Kiện | Ý Nghĩa |
|----------|---------|
| `INPUT_LOSS` | Không nhận được tín hiệu từ input (tính theo giây) |
| `BLACK_SCREEN` | Video thuần đen trong khoảng thời gian quy định |
| `AUDIO_SILENCE` | Không có audio trong khoảng thời gian quy định |

### Cấu Hình Automatic Input Failover

```json
{
  "InputAttachments": [
    {
      "InputAttachmentName": "primary-studio",
      "InputId": "PRIMARY_INPUT_ID",
      "AutomaticInputFailoverSettings": {
        "SecondaryInputId": "BACKUP_INPUT_ID",
        "InputPreference": "EQUAL_INPUT_PREFERENCE",
        "FailoverConditions": [
          {
            "FailoverConditionSettings": {
              "InputLossSettings": {
                "InputLossThresholdMsec": 3000
              }
            }
          },
          {
            "FailoverConditionSettings": {
              "VideoBlackSettings": {
                "VideoBlackThresholdMsec": 5000,
                "BlackDetectThreshold": 0.1
              }
            }
          },
          {
            "FailoverConditionSettings": {
              "AudioSilenceSettings": {
                "AudioSilenceThresholdMsec": 5000,
                "AudioSelectorName": "default-audio"
              }
            }
          }
        ]
      }
    }
  ]
}
```

### InputPreference Options (Tuỳ Chọn Ưu Tiên Input)

| Giá Trị | Hành Vi |
|--------|---------|
| `EQUAL_INPUT_PREFERENCE` | Không tự động quay về Primary — dùng input hiện tại đang hoạt động |
| `PRIMARY_INPUT_PREFERRED` | Tự động quay về Primary khi Primary khôi phục |

> **Khuyến nghị:** Dùng `EQUAL_INPUT_PREFERENCE` cho live event quan trọng để tránh switching không cần thiết khi Primary vừa khôi phục (có thể gây glitch nhỏ).

---

## 4. Pipeline Locking — Đồng Bộ Hai Pipeline

Trong Standard Channel, 2 pipelines chạy độc lập. **Pipeline Locking** cố gắng đồng bộ timecode giữa A và B để khi failover xảy ra, transition mượt mà hơn.

### Cách Hoạt Động

```
Không có Pipeline Locking:
  Pipeline A: Frame 1000 → Frame 1001 → Frame 1002 ...
  Pipeline B: Frame 997  → Frame 998  → Frame 999  ...  (lệch nhau vài frame)

  Khi failover A → B: viewer thấy video giật lùi hoặc tiến vài frame

Có Pipeline Locking:
  Pipeline A: Frame 1000 → Frame 1001 → Frame 1002 ...
  Pipeline B: Frame 1000 → Frame 1001 → Frame 1002 ...  (đồng bộ)

  Khi failover A → B: transition gần như không nhận ra
```

### Cấu Hình

```json
{
  "AvailConfiguration": {
    "AvailSettings": {}
  },
  "GlobalConfiguration": {
    "PipelineLockingMode": "PIPELINE_LOCKING",
    "SupportLowFramerateInputs": "DISABLED"
  }
}
```

> Pipeline Locking chỉ có tác dụng với Standard Channel và khi cả 2 pipelines đang nhận input từ cùng nguồn (synchronized encoder output).

---

## 5. Input Loss Behavior — Hành Vi Khi Mất Tín Hiệu

Khi **một** pipeline mất tín hiệu (nhưng pipeline kia vẫn chạy), pipeline mất tín hiệu sẽ hành xử theo cấu hình `InputLossBehavior`:

### Các Tùy Chọn

| Tùy Chọn | Hành Vi | Khi Nào Dùng |
|----------|---------|-------------|
| `EMIT_PROGRAM` | Tiếp tục phát nội dung đen (black video) | Mặc định |
| `REPEAT_PREVIOUS_FRAME` | Lặp lại frame cuối cùng trước khi mất tín hiệu | Tránh flicker đột ngột |
| `EMIT_SLATE` | Chèn video slate (ảnh/video dự phòng) | Broadcast chuyên nghiệp |
| `NONE` | Không phát gì | Hiếm khi dùng |

### Cấu Hình Slate (Video Dự Phòng)

```json
{
  "GlobalConfiguration": {
    "InputLossBehavior": {
      "InputLossImageType": "SLATE",
      "InputLossImageSlate": {
        "Uri": "s3ssl://my-bucket/slate/maintenance-slate.mp4"
      },
      "InputLossImageColor": "000000",
      "RepeatFrameMsec": 1000,
      "BlackFrameMsec": 0
    }
  }
}
```

---

## 6. CloudWatch Alarms & Monitoring — Giám Sát Và Cảnh Báo

### Metrics Quan Trọng Cho Redundancy

| Metric | Namespace | Ý Nghĩa | Alarm Threshold |
|--------|----------|---------|----------------|
| `InputLossSeconds` | `MediaLive` | Số giây bị mất tín hiệu input | > 0 |
| `ActiveAlerts` | `MediaLive` | Số cảnh báo đang kích hoạt | > 0 |
| `PipelineErrors` | `MediaLive` | Lỗi trong pipeline | > 0 |
| `DroppedFrames` | `MediaLive` | Số frame bị bỏ (encoding quá tải) | > 30/phút |
| `ChannelInputErrorSeconds` | `MediaLive` | Tổng giây lỗi input | > 5 |

### Thiết Lập CloudWatch Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "MediaLive-InputLoss-channel-001" \
  --alarm-description "Alert when MediaLive input signal is lost" \
  --metric-name "InputLossSeconds" \
  --namespace "MediaLive" \
  --dimensions Name=ChannelId,Value=CHANNEL_ID \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --statistic Sum \
  --alarm-actions "arn:aws:sns:ap-southeast-1:ACCOUNT_ID:ops-alerts"
```

### Pipeline EventBridge Notifications

MediaLive tự động gửi sự kiện qua EventBridge khi có thay đổi trạng thái:

```json
{
  "source": ["aws.medialive"],
  "detail-type": [
    "MediaLive Channel Alert",
    "MediaLive Channel State Change",
    "MediaLive Input State Change"
  ]
}
```

**Ví dụ event khi Input Failover xảy ra:**
```json
{
  "source": "aws.medialive",
  "detail-type": "MediaLive Channel Alert",
  "detail": {
    "alarm_id": "InputLossError",
    "alarm_state": "SET",
    "channel_arn": "arn:aws:medialive:...",
    "message": "Input loss detected on Pipeline 0",
    "pipeline": "0"
  }
}
```

---

## 7. Kiến Trúc Redundancy Nhiều Tầng

### Level 1: Basic (Dev/Test)

```
Single Pipeline Channel
├── 1 Input endpoint
└── 1 Output Group

Downtime khi sự cố: 2–5 phút
Chi phí: Thấp nhất
```

### Level 2: Standard (Production)

```
Standard Channel (2 pipelines)
├── Input A → Pipeline A → MediaPackage → CloudFront
└── Input B → Pipeline B → MediaPackage → CloudFront

Downtime: 0 (pipeline B tiếp quản ngay)
Điểm lỗi còn lại: encoder nguồn, MediaPackage, CloudFront
```

### Level 3: Full HA (High Availability — Khả Năng Sẵn Sàng Cao) cho Broadcast Production

```
[Encoder A] ──RTMP──▶ MediaLive Standard Channel (Primary Region)
   │                      Pipeline A ──▶ MediaPackage Primary ──▶ CloudFront
   │                      Pipeline B ──▶ MediaPackage Primary
   │
[Encoder B] ──RTMP──▶ MediaLive Standard Channel (Backup Region)
                          Pipeline A ──▶ MediaPackage Backup  ──▶ CloudFront
                          Pipeline B ──▶ MediaPackage Backup

CloudFront Origin Group:
  - Primary: MediaPackage Primary
  - Failover: MediaPackage Backup
  - Auto-failover khi Primary không phản hồi (HTTP 5xx)

Downtime: ~30 giây (CloudFront failover thời gian)
Chi phí: Cao nhất
Dùng cho: Kênh truyền hình OTT, sự kiện phát toàn cầu
```

---

## 8. Chi Phí vs Độ Tin Cậy

| Kiến Trúc | Chi Phí Tương Đối | Downtime Mục Tiêu | Dùng Cho |
|----------|-----------------|-------------------|---------|
| Single Pipeline, no failover | 1× | Phút | Dev, test |
| Standard Channel | ~2× | 0 giây (pipeline) | Tất cả production |
| Standard + Input Failover | ~2× | 0–3 giây | Broadcast quan trọng |
| Standard + Pipeline Locking | ~2× | 0 giây (mượt hơn) | Live event cao cấp |
| Multi-region Active/Standby | ~4× | ~30 giây | Broadcast mission-critical |

### Câu Hỏi Quyết Định Kiến Trúc

```
1. "Mất stream trong 5 phút có ảnh hưởng nghiêm trọng không?"
   → Có → Dùng Standard Channel tối thiểu

2. "Encoder nguồn có thể lỗi không?"
   → Có → Thêm Automatic Input Failover với backup encoder

3. "Yêu cầu uptime 99.99% (< 1 giờ downtime/năm)?"
   → Có → Multi-region Active/Standby

4. "Chi phí eo hẹp nhưng vẫn cần production?"
   → Standard Channel, không cần multi-region
   → Tiết kiệm: giảm số rendition trong bitrate ladder
```

---

## Best Practices

```
✅ NÊN làm:
- Luôn dùng Standard Channel cho live stream production
- Cấu hình encoder gửi đến CẢ 2 endpoints (A và B)
- Bật Automatic Input Failover với backup input (dù là MP4 slate)
- Thiết lập CloudWatch Alarm cho InputLossSeconds > 0
- Test failover định kỳ: ngắt input A và kiểm tra pipeline B tiếp quản
- Dùng Pipeline Locking cho sự kiện quan trọng có SLA cao

❌ KHÔNG nên làm:
- Dùng Single Pipeline cho live event quan trọng (rủi ro quá cao)
- Chỉ gửi input đến 1 endpoint (mất tác dụng redundancy của Standard Channel)
- Tin tưởng hoàn toàn vào automatic recovery mà không có monitoring
- Bỏ qua kiểm tra failover trước sự kiện lớn
```

---

**Phần Trước:** [2-encoding-settings.md](./2-encoding-settings.md) — Encoding Settings, Bitrate Ladder
**Phần Tiếp Theo:** [4-output-destinations.md](./4-output-destinations.md) — Output Destinations
