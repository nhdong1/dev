# AWS Media Services — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về AWS Media Services — từ nền tảng, xử lý video, phát trực tiếp đến phân phối và cá nhân hoá nội dung

## 📁 Cấu Trúc Thư Mục (Folder Structure)

```
media/
├── README.md                                       [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                        [FILE NÀY] Chỉ mục & trạng thái tạo file
│
├── 01-fundamentals/
│   ├── README.md                                   Nền tảng media: codec, protocols, pipeline
│   ├── 1-codec-container.md                        Codec (H.264/H.265/AV1), Container (MP4/HLS/DASH)
│   ├── 2-streaming-protocols.md                    HLS, DASH, RTMP, SRT, WebRTC so sánh
│   ├── 3-adaptive-bitrate.md                       ABR — Adaptive Bitrate Streaming
│   ├── 4-drm-fundamentals.md                       DRM — Digital Rights Management cơ bản
│   └── 5-media-pipeline-concepts.md                Ingest → Encode → Package → Deliver
│
├── 02-mediaconvert/
│   ├── README.md                                   MediaConvert tổng quan: VOD transcoding
│   ├── 1-jobs-queues.md                            Job, Queue (On-demand/Reserved/Spot), Role IAM
│   ├── 2-output-groups.md                          HLS, DASH, CMAF, File Group, MS Smooth
│   ├── 3-video-codec-settings.md                   H.264, H.265, AV1: bitrate, resolution, profile
│   ├── 4-audio-captions.md                         Multi-track audio, phụ đề SRT/TTML/WebVTT
│   └── 5-cost-optimization.md                      Reserved Queue, Spot, tối ưu chi phí per-minute
│
├── 03-medialive/
│   ├── README.md                                   MediaLive tổng quan: live encoding channel
│   ├── 1-channel-input.md                          Input types: RTMP, RTP, HLS, MediaConnect, MP4
│   ├── 2-encoding-settings.md                      H.264/H.265, bitrate ladder, audio selectors
│   ├── 3-redundancy-failover.md                    Standard (2 pipelines) vs Single pipeline, failover
│   ├── 4-output-destinations.md                    MediaPackage, S3, RTMP Push, HLS, UDP/TS
│   └── 5-schedule-scte35.md                        Schedule Actions, SCTE-35 ad markers, input switch
│
├── 04-mediapackage/
│   ├── README.md                                   MediaPackage tổng quan: packaging & origin
│   ├── 1-channels-endpoints.md                     Channel (WebDAV ingest), Endpoint (HLS/DASH/CMAF)
│   ├── 2-just-in-time-packaging.md                 JIT packaging, bitrate filtering, time-delay
│   ├── 3-drm-speke.md                              SPEKE — Secure Packager and Encoder Key Exchange
│   ├── 4-time-shift-viewing.md                     Startover, catch-up TV, windowed manifest
│   └── 5-mediapackage-v2.md                        V2: harvest jobs, low-latency CMAF, scalability
│
├── 05-mediastore/
│   ├── README.md                                   MediaStore tổng quan: low-latency media storage
│   ├── 1-container-access-policy.md                Container, IAM policy, CORS configuration
│   ├── 2-lifecycle-policy.md                       Tự động xoá fragment hết hạn (live buffer)
│   └── 3-mediastore-vs-s3.md                       So sánh MediaStore và S3 cho media workloads
│
├── 06-mediatailor/
│   ├── README.md                                   MediaTailor tổng quan: SSAI và Channel Assembly
│   ├── 1-ssai-basics.md                            SSAI — Server-Side Ad Insertion: playback config
│   ├── 2-ads-integration.md                        ADS — Ad Decision Server: VAST/VMAP/VPAID
│   ├── 3-channel-assembly.md                       Channel Assembly: lắp ghép VOD thành linear channel
│   ├── 4-reporting-beacons.md                      Beacons tracking: impression, quartile, complete
│   └── 5-prefetch-ads.md                           Ad prefetching giảm độ trễ khi bắt đầu ad break
│
├── 07-ivs/
│   ├── README.md                                   IVS tổng quan: interactive live streaming
│   ├── 1-channel-ingest.md                         Channel types, RTMPS ingest, stream key
│   ├── 2-latency-modes.md                          Standard, Low-latency, Real-time (WebRTC <1s)
│   ├── 3-timed-metadata.md                         Timed metadata: đồng bộ UI tương tác với video
│   ├── 4-ivs-chat.md                               IVS Chat: phòng chat realtime tích hợp
│   └── 5-recording-playback.md                     Auto-recording to S3, Playback SDK (JS/iOS/Android)
│
├── 08-kinesis-video/
│   ├── README.md                                   KVS tổng quan: video từ thiết bị IoT & camera
│   ├── 1-producer-consumer.md                      Producer SDK (gửi video), Consumer (đọc stream)
│   ├── 2-webrtc-signaling.md                       WebRTC two-way video, signaling channel
│   ├── 3-rekognition-integration.md                Tích hợp Rekognition: face/label detection realtime
│   └── 4-retention-lifecycle.md                    Retention period, GetMedia, HLS/DASH playback
│
├── 09-cdn-delivery/
│   ├── README.md                                   CloudFront + Media tổng quan: phân phối toàn cầu
│   ├── 1-cloudfront-for-media.md                   Cache behaviors, TTL, Origin Groups cho video
│   ├── 2-signed-url-cookies.md                     Signed URL vs Signed Cookies: bảo vệ nội dung trả phí
│   ├── 3-oac-origin-security.md                    OAC — Origin Access Control bảo mật S3/MediaPackage
│   └── 4-edge-functions-media.md                   CloudFront Functions và Lambda@Edge cho media routing
│
└── 10-interview-prep/
    ├── README.md                                   Tổng quan chuẩn bị phỏng vấn & checklist
    ├── INTERVIEW_GUIDE.md                          Top 20 câu hỏi và câu trả lời mẫu chi tiết
    ├── architecture-scenarios.md                   Bài toán thiết kế: OTT platform, live streaming
    └── star-stories.md                             Mẫu câu chuyện thực tế theo STAR method
```

---

## ✅ Trạng Thái Tạo File (Creation Status)

| Chủ Đề                                      | File                        | Trạng Thái        | Chất Lượng   |
| ------------------------------------------- | --------------------------- | ----------------- | ------------ |
| **Tổng Quan & Lộ Trình**                   | README.md                   | ✅ Hoàn thành     | Toàn diện    |
| **Chỉ Mục Đầy Đủ**                         | INDEX.md                    | ✅ Hoàn thành     | Toàn diện    |
| **Nền Tảng Media**                          | 01-fundamentals/            | ✅ Hoàn thành     | Toàn diện    |
| **MediaConvert (VOD Transcoding)**          | 02-mediaconvert/            | ✅ Hoàn thành     | Toàn diện    |
| **MediaLive (Live Encoding)**               | 03-medialive/               | ✅ Hoàn thành     | Toàn diện    |
| **MediaPackage (Packaging & DRM)**          | 04-mediapackage/            | 🚧 Chờ tạo       | -            |
| **MediaStore (Media Storage)**              | 05-mediastore/              | 🚧 Chờ tạo       | -            |
| **MediaTailor (Ad Insertion)**              | 06-mediatailor/             | 🚧 Chờ tạo       | -            |
| **Amazon IVS (Interactive Live)**           | 07-ivs/                     | 🚧 Chờ tạo       | -            |
| **Kinesis Video Streams**                   | 08-kinesis-video/           | 🚧 Chờ tạo       | -            |
| **CDN & Phân Phối (CloudFront)**            | 09-cdn-delivery/            | 🚧 Chờ tạo       | -            |
| **Chuẩn Bị Phỏng Vấn**                     | 10-interview-prep/          | 🚧 Chờ tạo       | -            |

---

## 🎯 Thứ Tự Tạo File (Priority Order)

### Cao (Core media skills — bắt buộc)

- [x] `01-fundamentals/README.md` — codec, protocols, pipeline concepts ✅
- [x] `02-mediaconvert/README.md` — VOD transcoding (entry point dễ nhất) ✅
- [x] `03-medialive/README.md` — live encoding fundamentals ✅
- [ ] `04-mediapackage/README.md` — packaging, DRM, origin
- [ ] `09-cdn-delivery/README.md` — CloudFront phân phối video

### Trung Bình (Advanced media skills)

- [ ] `07-ivs/README.md` — interactive live streaming (IVS)
- [ ] `06-mediatailor/README.md` — SSAI ad insertion
- [x] `02-mediaconvert/5-cost-optimization.md` — tối ưu chi phí MediaConvert ✅
- [x] `03-medialive/3-redundancy-failover.md` — HA cho live channel ✅
- [ ] `10-interview-prep/INTERVIEW_GUIDE.md` — câu hỏi phỏng vấn

### Thấp (Reference / Niche)

- [ ] `08-kinesis-video/README.md` — KVS cho IoT/ML
- [ ] `05-mediastore/README.md` — MediaStore deep dive
- [ ] `04-mediapackage/5-mediapackage-v2.md` — V2 features
- [ ] `10-interview-prep/architecture-scenarios.md` — system design
- [ ] `10-interview-prep/star-stories.md` — STAR stories

---

## 🚀 Hướng Dẫn Sử Dụng (How to Use)

### Cho Tự Học (Self-Study)

```
1. Đọc README.md (file này tổng quan)
2. Chọn lộ trình học (Mới bắt đầu / Trung cấp / Nâng cao)
3. Bắt đầu với 01-fundamentals/ để nắm codec & protocols
4. Học dịch vụ theo use case: VOD → Live → Interactive → Ads
5. Thực hành hands-on trên AWS Console / CLI
```

### Cho Chuẩn Bị Phỏng Vấn (Interview Prep)

```
1. Đọc 10-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào dịch vụ của role mục tiêu
3. Học 02-mediaconvert/ và 03-medialive/ (hỏi nhiều nhất)
4. Hiểu kiến trúc end-to-end (README.md → Topics Overview)
5. Chuẩn bị STAR stories từ kinh nghiệm thực tế
```

### Cho Công Việc Thực Tế (On-the-Job)

```
Tra cứu nhanh:
- VOD pipeline:       02-mediaconvert/
- Live pipeline:      03-medialive/ + 04-mediapackage/
- Ad insertion:       06-mediatailor/
- Interactive live:   07-ivs/
- CDN delivery:       09-cdn-delivery/
- IoT video:          08-kinesis-video/
```

### Cho Thiết Kế Hệ Thống (System Design)

```
1. Xác định use case: VOD / Live / Interactive / Ads
2. Đọc README.md → phần "Theo Loại Use Case"
3. Tham khảo từng dịch vụ liên quan theo pipeline
4. Xem 10-interview-prep/architecture-scenarios.md
5. Tính toán chi phí với AWS Pricing Calculator
```

---

## 📊 Ước Tính Thời Gian Học (Study Time Estimates)

| Phần                               | Thời Gian    | Độ Khó  | Ưu Tiên |
| ---------------------------------- | ------------ | ------- | ------- |
| Nền tảng Media (01)                | 3-4 giờ      | ⭐      | Bắt buộc |
| MediaConvert — VOD (02)            | 4-6 giờ      | ⭐⭐    | Bắt buộc |
| MediaLive — Live (03)              | 6-8 giờ      | ⭐⭐⭐  | Bắt buộc |
| MediaPackage — Package (04)        | 4-6 giờ      | ⭐⭐    | Bắt buộc |
| MediaStore (05)                    | 1-2 giờ      | ⭐      | Nên có   |
| MediaTailor — Ads (06)             | 4-6 giờ      | ⭐⭐⭐  | Nên có   |
| IVS — Interactive Live (07)        | 3-4 giờ      | ⭐⭐    | Bắt buộc |
| Kinesis Video Streams (08)         | 3-4 giờ      | ⭐⭐    | Nên có   |
| CDN Delivery — CloudFront (09)     | 3-4 giờ      | ⭐⭐    | Bắt buộc |
| Kiến trúc nâng cao & Phỏng vấn    | 6-8 giờ      | ⭐⭐⭐  | Bắt buộc |

**Tổng: 40-55 giờ để có kiến thức AWS Media Services toàn diện**

---

## 🎓 Trình Độ Được Hỗ Trợ (Skill Levels Supported)

### Mới Bắt Đầu (0-1 năm kinh nghiệm)

- [ ] Hiểu codec (H.264, H.265), container (MP4, HLS), bitrate
- [ ] Biết sự khác nhau giữa VOD và Live streaming
- [ ] Tạo MediaConvert job đơn giản từ Console
- [ ] Biết CloudFront phân phối video như thế nào
- [ ] Phân biệt được các dịch vụ AWS Elemental

**Thời gian ước tính để đạt trình độ này:** 1-2 tháng

### Trung Cấp (1-3 năm kinh nghiệm)

- [ ] Thiết lập MediaLive channel Standard với failover
- [ ] Tích hợp MediaPackage với DRM (Widevine + FairPlay)
- [ ] Xây dựng pipeline VOD end-to-end (S3 → CloudFront)
- [ ] Triển khai IVS cho low-latency interactive live
- [ ] Cấu hình MediaTailor SSAI với ADS

**Thời gian ước tính để đạt trình độ này:** 2-3 tháng

### Nâng Cao (3-5+ năm kinh nghiệm)

- [ ] Thiết kế kiến trúc OTT cho hàng triệu concurrent viewers
- [ ] Tối ưu chi phí cho large-scale video processing
- [ ] Xử lý sự cố live stream production (24/7)
- [ ] Multi-DRM cross-platform với key rotation
- [ ] Tích hợp KVS + Rekognition cho real-time video analytics

**Thời gian ước tính để đạt trình độ này:** Học liên tục

---

## 🔗 Điều Hướng Nhanh (Quick Navigation)

| Nhu Cầu                              | Vị Trí                                                                         |
| ------------------------------------ | ------------------------------------------------------------------------------ |
| Tổng quan nhanh                      | [README.md](README.md)                                                         |
| Nền tảng codec & protocols           | [01-fundamentals/README.md](01-fundamentals/README.md)                        |
| Chuyển mã video (VOD)                | [02-mediaconvert/README.md](02-mediaconvert/README.md)                        |
| Phát trực tiếp (Live)                | [03-medialive/README.md](03-medialive/README.md)                              |
| Đóng gói & DRM                       | [04-mediapackage/README.md](04-mediapackage/README.md)                        |
| Interactive live streaming           | [07-ivs/README.md](07-ivs/README.md)                                          |
| Chèn quảng cáo SSAI                  | [06-mediatailor/README.md](06-mediatailor/README.md)                          |
| Video IoT & phân tích ML             | [08-kinesis-video/README.md](08-kinesis-video/README.md)                      |
| Phân phối CDN                        | [09-cdn-delivery/README.md](09-cdn-delivery/README.md)                        |
| Câu hỏi phỏng vấn                    | [10-interview-prep/INTERVIEW_GUIDE.md](10-interview-prep/INTERVIEW_GUIDE.md) |

---

## 📈 Theo Dõi Tiến Trình Học (Learning Progress Tracker)

Sao chép và điền vào để theo dõi tiến trình của bạn:

```markdown
## AWS Media Services — Tiến Trình Học

### Phase 1: Nền Tảng (Tuần 1-2)

- [ ] Codec: H.264, H.265, AV1
- [ ] Container: MP4, HLS, DASH, CMAF
- [ ] Giao thức: RTMP, HLS, DASH, SRT, WebRTC
- [ ] ABR — Adaptive Bitrate Streaming
- [ ] DRM cơ bản (Widevine, FairPlay, PlayReady)

### Phase 2: Xử Lý Video (Tuần 3-5)

- [ ] MediaConvert: tạo job, output groups, codec settings
- [ ] MediaLive: channel, input types, encoding, output
- [ ] MediaPackage: endpoint HLS/DASH, DRM, time-shift
- [ ] MediaStore: container, lifecycle

### Phase 3: Phân Phối & Tương Tác (Tuần 6-8)

- [ ] CloudFront cho video: cache, signed URL, OAC
- [ ] IVS: channel, latency modes, timed metadata, chat
- [ ] MediaTailor: SSAI, ADS integration, channel assembly
- [ ] Kinesis Video Streams: producer, consumer, WebRTC

### Phase 4: Kiến Trúc & Phỏng Vấn (Tuần 9-10)

- [ ] Thiết kế OTT pipeline end-to-end
- [ ] Tối ưu chi phí media workloads
- [ ] Top 20 câu hỏi phỏng vấn
- [ ] Mock interview với bài toán thiết kế
```

---

## 🎯 Tiêu Chí Thành Công (Success Criteria)

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích codec, container, bitrate mà không cần tài liệu
- [ ] Phân biệt HLS, DASH, CMAF và biết khi nào dùng cái nào
- [ ] Vẽ sơ đồ pipeline VOD và Live streaming đầy đủ
- [ ] Giải thích DRM và cách AWS Speke hoạt động

### ✅ Năng Lực Vận Hành

- [ ] Tạo và cấu hình MediaConvert job với nhiều output
- [ ] Thiết lập MediaLive channel với Standard redundancy
- [ ] Cấu hình MediaPackage với DRM endpoint
- [ ] Triển khai IVS cho low-latency live streaming
- [ ] Tích hợp CloudFront với MediaPackage/S3

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin top 20 câu hỏi AWS Media Services
- [ ] Kể được 2-3 câu chuyện thực tế theo STAR format
- [ ] Thiết kế kiến trúc OTT cho yêu cầu đặc thù
- [ ] Phân tích trade-offs giữa các dịch vụ media
- [ ] Ước tính chi phí cho media workload

---

## 💡 Mẹo Học Hiệu Quả (Pro Tips)

1. **Học qua ví dụ thực tế:** Tạo account AWS Free Tier, thử tạo MediaConvert job và IVS channel
2. **Hiểu data flow:** Luôn vẽ sơ đồ pipeline từ source đến viewer
3. **So sánh dịch vụ:** Biết khi nào dùng MediaLive vs IVS vs Kinesis Video là điểm cộng lớn trong phỏng vấn
4. **Nắm vững pricing:** Chi phí media services thường là câu hỏi thực tế quan trọng
5. **Thực hành với OBS:** OBS Studio miễn phí, dùng để test ingest vào MediaLive hoặc IVS
6. **Đọc re:Invent talks:** AWS re:Invent có nhiều video thực tế về media architecture
7. **SCTE-35 quan trọng:** Hiểu SCTE-35 ad markers là lợi thế lớn cho live broadcast projects
8. **Kiểm tra latency:** Low-latency vs standard latency ảnh hưởng trực tiếp đến architecture choice

---

## 📞 Đóng Góp (Contributing)

Phát hiện lỗi? Muốn bổ sung nội dung?

Đây là tài liệu sống, luôn được cập nhật:

- [ ] Sửa thông tin không chính xác về API / pricing
- [ ] Bổ sung ví dụ thực tế từ kinh nghiệm
- [ ] Thêm câu hỏi phỏng vấn mới gặp phải
- [ ] Cập nhật tính năng mới của AWS Elemental / IVS
- [ ] Thêm so sánh với dịch vụ của GCP / Azure

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.3 (README + INDEX + 01-fundamentals + 02-mediaconvert + 03-medialive hoàn thành)
**Trạng Thái:** ✅ README + INDEX + 01-fundamentals + 02-mediaconvert + 03-medialive hoàn thành | 🚧 Các module 04–10 đang chờ tạo
