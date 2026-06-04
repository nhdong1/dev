# 🎬 AWS Media Services — Roadmap

> Hướng dẫn toàn diện về AWS Media Services, bao gồm xử lý video, phát trực tiếp, phân phối nội dung, cá nhân hóa quảng cáo và chuẩn bị phỏng vấn.

## 📚 Mục Lục (Table of Contents)

1. [Lộ Trình Học Tập](#lộ-trình-học-tập)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Dịch Vụ](#tổng-quan-dịch-vụ)
4. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)

---

## 🎯 Lộ Trình Học Tập (Learning Path)

### **Phase 1: Nền Tảng (Weeks 1-2)**

- [ ] Các khái niệm Media cơ bản: Codec (bộ mã hoá/giải mã), Container (định dạng đóng gói), Bitrate (tốc độ bit), Resolution (độ phân giải)
- [ ] Giao thức streaming: HLS — HTTP Live Streaming, DASH — Dynamic Adaptive Streaming over HTTP, RTMP — Real-Time Messaging Protocol
- [ ] Mô hình quy trình media: Ingest → Encode → Package → Deliver (Nhập → Mã hoá → Đóng gói → Phân phối)
- [ ] Tổng quan hệ sinh thái AWS Elemental và các dịch vụ liên quan

### **Phase 2: Xử Lý Video Theo Tệp & Trực Tiếp (Weeks 3-5)**

- [ ] AWS Elemental MediaConvert — Chuyển mã video theo tệp (File-based transcoding)
- [ ] AWS Elemental MediaLive — Mã hoá video trực tiếp (Live video encoding)
- [ ] AWS Elemental MediaPackage — Đóng gói và bảo vệ nội dung (Packaging & DRM)
- [ ] AWS Elemental MediaStore — Lưu trữ tối ưu cho media (Media-optimized storage)

### **Phase 3: Phân Phối & Cá Nhân Hóa (Weeks 6-8)**

- [ ] AWS Elemental MediaTailor — Chèn quảng cáo cá nhân hoá (Server-side ad insertion)
- [ ] Amazon IVS — Interactive Video Service — Dịch Vụ Video Tương Tác (live streaming độ trễ thấp)
- [ ] Amazon Kinesis Video Streams — Luồng video cho phân tích & ML
- [ ] Amazon CloudFront + Media — CDN phân phối nội dung media

### **Phase 4: Kiến Trúc Nâng Cao & Phỏng Vấn (Weeks 9-10)**

- [ ] Kiến trúc OTT — Over-The-Top end-to-end trên AWS
- [ ] Bảo vệ nội dung với DRM — Digital Rights Management
- [ ] Giám sát và tối ưu chi phí cho media workloads
- [ ] Chuẩn bị phỏng vấn: câu hỏi thực tế và bài toán thiết kế

---

## 🏢 Năng Lực Cốt Lõi (Core Competencies)

| Năng Lực                              | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| ------------------------------------- | ---------- | --------- | ---------- |
| **File-based Transcoding (MediaConvert)** | ⭐⭐⭐   | 2 tuần    | -          |
| **Live Encoding (MediaLive)**         | ⭐⭐⭐   | 2 tuần    | -          |
| **Packaging & DRM (MediaPackage)**    | ⭐⭐⭐   | 1 tuần    | -          |
| **Interactive Live (IVS)**            | ⭐⭐⭐   | 1 tuần    | -          |
| **Ad Insertion (MediaTailor)**        | ⭐⭐     | 1 tuần    | -          |
| **Video Streaming (Kinesis Video)**   | ⭐⭐     | 1 tuần    | -          |
| **CDN Delivery (CloudFront + Media)** | ⭐⭐⭐   | 1 tuần    | -          |
| **Media Storage (MediaStore)**        | ⭐⭐     | 0.5 tuần  | -          |
| **OTT Architecture End-to-End**      | ⭐⭐⭐   | 2 tuần    | -          |
| **Cost Optimization for Media**       | ⭐⭐     | 0.5 tuần  | -          |

---

## 🗂️ Tổng Quan Dịch Vụ (Services Overview)

### Bản Đồ Dịch Vụ AWS Media

```
Ingest (Nhập liệu)
  ├── MediaLive Input           — Live stream RTMP/RTP/HLS đầu vào
  ├── MediaConnect              — Vận chuyển video chất lượng cao qua mạng IP
  └── Kinesis Video Streams     — Nhập video từ thiết bị IoT / camera

Encode & Process (Mã hoá & Xử lý)
  ├── MediaConvert              — Chuyển mã file video theo yêu cầu
  ├── MediaLive                 — Mã hoá luồng live real-time
  └── Rekognition Video         — Phân tích nội dung video bằng AI/ML

Package & Protect (Đóng gói & Bảo vệ)
  ├── MediaPackage              — Đóng gói HLS/DASH/CMAF, DRM tích hợp
  └── MediaStore                — Lưu trữ fragment media độ trễ thấp

Deliver (Phân phối)
  ├── CloudFront                — CDN toàn cầu phân phối video
  └── MediaTailor               — SSAI — Server-Side Ad Insertion cá nhân hoá

Interact (Tương tác)
  └── IVS — Interactive Video Service — Live stream tương tác độ trễ < 5 giây
```

---

## 🗺️ Tổng Quan Chủ Đề (Topics Overview)

### 📁 **1. Nền Tảng Media** (`01-fundamentals/`)

- Codec: H.264/AVC, H.265/HEVC, VP9, AV1 — Các chuẩn nén video phổ biến
- Container: MP4, MKV, HLS (.m3u8), DASH (.mpd) — Định dạng đóng gói
- Giao thức streaming: HLS, DASH, RTMP, SRT — Smooth Secure Reliable Transport
- ABR — Adaptive Bitrate Streaming — Phát Trực Tuyến Thích Ứng Tốc Độ Bit
- Latency (Độ trễ): Standard latency vs Low latency vs Real-time
- DRM — Digital Rights Management — Quản Lý Quyền Kỹ Thuật Số: Widevine, FairPlay, PlayReady
- CDN — Content Delivery Network — Mạng Phân Phối Nội Dung: edge caching, origin shield

### 📁 **2. MediaConvert — Chuyển Mã Video** (`02-mediaconvert/`)

- Job (tác vụ chuyển mã): Input → Settings → Output Groups → Outputs
- Output Groups: HLS, DASH ISO, File Group, CMAF, MS Smooth
- Video Codec settings: H.264, H.265, AV1 — bitrate, resolution, framerate
- Audio tracks: AAC, Dolby Digital, multi-track, audio normalization
- Captions (phụ đề): SRT, TTML, WebVTT, embedded CEA-608/708
- Image Inserter (chèn hình mờ / watermark)
- Queue (hàng đợi xử lý): On-demand, Reserved, Spot pricing
- Pricing model: phút xử lý theo độ phân giải (SD/HD/UHD)

### 📁 **3. MediaLive — Mã Hoá Trực Tiếp** (`03-medialive/`)

- Channel (kênh): Input → Video Selector → Encode Settings → Output Groups
- Input types: RTMP Push/Pull, RTP, HLS Pull, MediaConnect, MP4 (file)
- Redundancy: Standard channel (2 pipelines A/B) vs Single pipeline
- Output destinations: MediaPackage, S3, RTMP, HLS, UDP/TS
- Encoding profiles: H.264 / H.265 — bitrate ladder (thang tốc độ bit)
- Audio selector: mapping nguồn âm thanh theo PID / track
- Motion Graphics: overlay đồ hoạ động trên live stream
- Schedule Actions: chuyển cảnh, chèn SCTE-35 markers, input switching
- SCTE-35 — Society of Cable Telecommunications Engineers 35 — chuẩn đánh dấu điểm chèn quảng cáo

### 📁 **4. MediaPackage — Đóng Gói & Bảo Vệ** (`04-mediapackage/`)

- Channel (kênh nhận): nhận luồng từ MediaLive qua WebDAV hoặc REST
- Endpoint (điểm phát): HLS, DASH, CMAF — CMAF — Common Media Application Format
- Origin (nguồn phát): just-in-time packaging (đóng gói theo yêu cầu thời gian thực)
- Time-shift viewing: startover, catch-up TV (xem lại các chương trình đã phát)
- DRM tích hợp: AWS Speke — Secure Packager and Encoder Key Exchange, Widevine, FairPlay, PlayReady
- CDN Authorization: signed URL / signed cookies từ CloudFront
- MediaPackage v2: harvest jobs, low-latency CMAF, improved scalability

### 📁 **5. MediaStore — Lưu Trữ Media** (`05-mediastore/`)

- Container (thùng chứa): vùng lưu trữ tối ưu cho media fragments
- CORS — Cross-Origin Resource Sharing — Chia Sẻ Tài Nguyên Đa Nguồn gốc: cấu hình cho web player
- Access policy: IAM + resource-based policy kiểm soát truy cập
- Lifecycle policy: tự động xoá nội dung hết hạn (live buffer)
- So sánh MediaStore vs S3: độ trễ thấp hơn cho live, S3 tốt hơn cho VOD

### 📁 **6. MediaTailor — Chèn Quảng Cáo** (`06-mediatailor/`)

- SSAI — Server-Side Ad Insertion — Chèn Quảng Cáo Phía Máy Chủ (không bị ad-blocker chặn)
- Playback configuration: content source URL, ADS — Ad Decision Server URL
- ADS — Ad Decision Server — Máy Chủ Ra Quyết Định Quảng Cáo: VAST/VMAP/VPAID responses
- Ad marker passthrough: SCTE-35 từ MediaLive → MediaTailor → personalized ads
- Channel Assembly: lắp ghép VOD clip thành kênh linear (giống kênh truyền hình)
- Reporting: beacons (tín hiệu theo dõi) tracking impression, quartile, complete
- Prefetch: tải trước quảng cáo để giảm độ trễ khi chuyển sang ad break

### 📁 **7. Amazon IVS — Video Tương Tác** (`07-ivs/`)

- IVS — Interactive Video Service — Dịch Vụ Video Tương Tác: managed live streaming
- Ingest: RTMPS push từ OBS / encoder phần cứng
- Latency modes: Standard (~15s), Low (~5s), Real-time (WebRTC, <1s)
- Channel types: Standard, Basic, Advanced-SD, Advanced-HD
- Timed metadata: nhúng metadata vào luồng để đồng bộ hoá UI tương tác
- Chat (IVS Chat): phòng chat realtime tích hợp cùng luồng video
- Recording: tự động lưu vào S3 dưới dạng HLS segments
- Playback SDK: JavaScript, iOS, Android player tích hợp sẵn

### 📁 **8. Kinesis Video Streams** (`08-kinesis-video/`)

- KVS — Kinesis Video Streams — Luồng Video Kinesis: nhập và lưu trữ video từ thiết bị
- Producer SDK: gửi video từ camera / IoT device lên AWS
- Consumer: đọc video frame-by-frame hoặc theo chunk để phân tích
- WebRTC tích hợp: peer-to-peer video call / two-way streaming
- Rekognition integration: phân tích face detection, label detection real-time
- Retention (lưu trữ): cấu hình thời gian giữ dữ liệu (giờ → năm)
- Use cases: home security, factory floor monitoring, baby monitor, telemedicine

### 📁 **9. CDN & Phân Phối Nội Dung** (`09-cdn-delivery/`)

- CloudFront distribution cho media: cache behaviors, origin groups, TTL
- OAC — Origin Access Control — Kiểm Soát Truy Cập Origin: thay thế OAI bảo mật S3
- Signed URL vs Signed Cookies: kiểm soát truy cập nội dung trả phí
- CloudFront Functions vs Lambda@Edge: xử lý tại edge cho media routing
- Real-time logs: theo dõi cache hit/miss, viewer metrics
- Media-specific settings: Range GET, Cache-Control headers, CORS
- AWS Media Services + CloudFront: kiến trúc pipeline hoàn chỉnh

### 📁 **10. Chuẩn Bị Phỏng Vấn** (`10-interview-prep/`)

- Top 20 câu hỏi phỏng vấn AWS Media Services
- Bài toán thiết kế: xây dựng nền tảng OTT, live streaming platform
- STAR stories: các tình huống thực tế xử lý sự cố media
- So sánh dịch vụ: khi nào dùng MediaConvert vs MediaLive vs IVS
- Checklist kiến trúc media trên AWS

---

## 🎬 Theo Loại Use Case

### **VOD — Video on Demand — Video Theo Yêu Cầu**

```
Phù hợp: Netflix-style, nền tảng học trực tuyến, lưu trữ video
Pipeline:  S3 (source) → MediaConvert → S3 (output) → CloudFront → Viewer
Dịch vụ chính: MediaConvert, S3, CloudFront
Tài liệu: 02-mediaconvert/, 09-cdn-delivery/
```

### **Live Streaming — Phát Trực Tiếp**

```
Phù hợp: thể thao, tin tức, sự kiện trực tiếp, concert
Pipeline:  Encoder → MediaLive → MediaPackage → CloudFront → Viewer
Dịch vụ chính: MediaLive, MediaPackage, CloudFront
Tài liệu: 03-medialive/, 04-mediapackage/, 09-cdn-delivery/
```

### **Interactive Live — Phát Trực Tiếp Tương Tác**

```
Phù hợp: livestream bán hàng, game show, giáo dục tương tác
Pipeline:  OBS/Encoder → IVS → IVS Player SDK → Timed Metadata → UI
Dịch vụ chính: Amazon IVS, IVS Chat
Tài liệu: 07-ivs/
```

### **Live with Ads — Phát Trực Tiếp Có Quảng Cáo**

```
Phù hợp: kênh truyền hình OTT, free ad-supported streaming (FAST)
Pipeline:  MediaLive (SCTE-35) → MediaPackage → MediaTailor → CloudFront → Viewer
Dịch vụ chính: MediaLive, MediaPackage, MediaTailor
Tài liệu: 03-medialive/, 04-mediapackage/, 06-mediatailor/
```

### **IoT Video Analytics — Phân Tích Video IoT**

```
Phù hợp: camera an ninh, giám sát nhà máy, xe tự lái
Pipeline:  Camera/Device → KVS Producer SDK → KVS → Rekognition / Lambda
Dịch vụ chính: Kinesis Video Streams, Rekognition
Tài liệu: 08-kinesis-video/
```

---

## 🔗 Liên Kết Nhanh (Quick Links)

| Chủ Đề                        | Thư Mục                                                    | Độ Ưu Tiên           |
| ----------------------------- | ---------------------------------------------------------- | -------------------- |
| Tổng quan & Lộ trình           | [README.md](./README.md)                                   | Bắt đầu tại đây      |
| Chỉ mục đầy đủ                 | [INDEX.md](./INDEX.md)                                     | Tra cứu nhanh        |
| Chuyển mã VOD                  | [02-mediaconvert/](./02-mediaconvert/)                     | Thiết yếu            |
| Live encoding                  | [03-medialive/](./03-medialive/)                           | Thiết yếu            |
| Đóng gói & DRM                 | [04-mediapackage/](./04-mediapackage/)                     | Thiết yếu            |
| Interactive live               | [07-ivs/](./07-ivs/)                                       | Quan trọng           |
| Chèn quảng cáo                 | [06-mediatailor/](./06-mediatailor/)                       | Nâng cao             |
| Video IoT & phân tích          | [08-kinesis-video/](./08-kinesis-video/)                   | Nâng cao             |
| Phỏng vấn                      | [10-interview-prep/](./10-interview-prep/)                 | Trước phỏng vấn      |

---

## 📊 Ma Trận Kỹ Năng (Skill Matrix)

### Mới Bắt Đầu (0-1 năm kinh nghiệm)

- [ ] Hiểu khái niệm codec, container, bitrate, resolution
- [ ] Biết sự khác nhau giữa HLS và DASH
- [ ] Tạo job MediaConvert cơ bản (S3 → S3)
- [ ] Phân biệt VOD vs Live streaming
- [ ] Hiểu CDN caching cơ bản với CloudFront

### Trung Cấp (1-3 năm kinh nghiệm)

- [ ] Thiết lập MediaLive channel với Standard redundancy
- [ ] Cấu hình MediaPackage endpoint với DRM
- [ ] Tích hợp MediaTailor SSAI với SCTE-35
- [ ] Xây dựng pipeline VOD end-to-end
- [ ] Giám sát health của live channel (metrics & alarms)

### Nâng Cao (3-5+ năm kinh nghiệm)

- [ ] Thiết kế kiến trúc OTT toàn diện cho hàng triệu viewer
- [ ] Tối ưu chi phí cho large-scale transcoding
- [ ] Xử lý sự cố live stream (input loss, encoding errors)
- [ ] Triển khai multi-DRM cross-platform
- [ ] Xây dựng pipeline AI/ML phân tích nội dung video

---

## 🚀 Hướng Dẫn Bắt Đầu (Getting Started)

### Bước 1: Nắm Vững Khái Niệm Cơ Bản

```
Ưu tiên đọc trước:
- 01-fundamentals/     → codec, streaming protocols, media pipeline
- 02-mediaconvert/     → VOD transcoding (dễ bắt đầu nhất)
- 09-cdn-delivery/     → CloudFront phân phối video
```

### Bước 2: Thực Hành Hands-on

```bash
# Tạo MediaConvert job đơn giản qua AWS CLI
aws mediaconvert create-job \
  --role arn:aws:iam::ACCOUNT_ID:role/MediaConvertRole \
  --settings file://job-settings.json \
  --region us-east-1
```

### Bước 3: Xây Dựng Pipeline Hoàn Chỉnh

```
Lộ trình thực hành:
1. VOD pipeline: S3 → MediaConvert → S3 → CloudFront (Tuần 1-2)
2. Live pipeline: OBS → MediaLive → MediaPackage → CloudFront (Tuần 3-4)
3. Interactive live: OBS → IVS → Web player với timed metadata (Tuần 5)
4. Ad-supported live: MediaLive + SCTE-35 → MediaTailor (Tuần 6-7)
```

### Bước 4: Chuẩn Bị Phỏng Vấn

```
Cho mỗi dịch vụ, chuẩn bị câu chuyện STAR:
- Situation  — Bối cảnh: hệ thống media như thế nào
- Task        — Nhiệm vụ: yêu cầu cần giải quyết
- Action      — Hành động: giải pháp kỹ thuật đã thực hiện
- Result      — Kết quả: cải thiện đo lường được
```

---

## 📖 Tài Liệu Tham Khảo (Reference Materials)

### Tài Liệu Chính Thức AWS

- [AWS Elemental MediaConvert Docs](https://docs.aws.amazon.com/mediaconvert/)
- [AWS Elemental MediaLive Docs](https://docs.aws.amazon.com/medialive/)
- [AWS Elemental MediaPackage Docs](https://docs.aws.amazon.com/mediapackage/)
- [Amazon IVS Docs](https://docs.aws.amazon.com/ivs/)
- [Kinesis Video Streams Docs](https://docs.aws.amazon.com/kinesisvideostreams/)
- [AWS Elemental MediaTailor Docs](https://docs.aws.amazon.com/mediatailor/)

### Bài Đọc Thêm

- **"Video Streaming Protocols Explained"** — HLS, DASH, CMAF comparison
- **"AWS Media Services Overview"** — AWS whitepaper tổng quan
- **"Building a Live Streaming Platform on AWS"** — AWS blog
- **"OTT Platform Architecture"** — best practices từ AWS re:Invent

### Công Cụ Hữu Ích

- **MediaInfo** — Phân tích thông tin codec/container của file video
- **FFmpeg** — Công cụ dòng lệnh xử lý video (test encoding locally)
- **HLS.js** — JavaScript player cho HLS trên web
- **OBS Studio** — Phần mềm phát trực tiếp mã nguồn mở
- **AWS Elemental Live** — Hardware encoder tích hợp với AWS

---

## 🎯 Chuẩn Bị Phỏng Vấn (Interview Preparation)

### Câu Hỏi Theo Danh Mục

#### VOD & Transcoding

- [ ] Giải thích quy trình tạo MediaConvert job
- [ ] Các Output Group nào MediaConvert hỗ trợ?
- [ ] Làm thế nào tối ưu chi phí MediaConvert với Reserved Queue?
- [ ] Khác nhau giữa H.264 và H.265 (HEVC) trong use case nào?

#### Live Streaming

- [ ] Giải thích sự khác nhau giữa Standard và Single-pipeline channel
- [ ] Khi nào dùng MediaLive, khi nào dùng IVS?
- [ ] Xử lý input loss trong MediaLive như thế nào?
- [ ] SCTE-35 là gì và tại sao quan trọng trong live streaming?

#### Packaging & DRM

- [ ] Khác nhau giữa HLS và DASH, khi nào dùng CMAF?
- [ ] Giải thích cách MediaPackage xử lý just-in-time packaging
- [ ] Multi-DRM setup cho cross-platform (iOS, Android, Web)?
- [ ] Time-shift viewing hoạt động như thế nào?

#### Kiến Trúc Tổng Thể

- [ ] Thiết kế pipeline cho 1 triệu concurrent viewers
- [ ] Khác nhau giữa MediaStore và S3 cho media workflows
- [ ] Làm thế nào đảm bảo low latency cho live streaming toàn cầu?
- [ ] Chi phí tối ưu cho một nền tảng OTT nhỏ?

Xem `10-interview-prep/` để có hướng dẫn đầy đủ với câu trả lời mẫu.

---

## ✅ Danh Sách Tự Đánh Giá (Self-Assessment Checklist)

Trước phỏng vấn hoặc khi nhận dự án mới, kiểm tra:

- [ ] Có thể giải thích pipeline VOD và Live streaming đầy đủ
- [ ] Biết cấu hình MediaConvert job với nhiều output format
- [ ] Hiểu cách MediaLive xử lý redundancy và failover
- [ ] Biết cách tích hợp DRM với MediaPackage + Speke
- [ ] Có thể thiết kế kiến trúc OTT cho yêu cầu cụ thể
- [ ] Hiểu SSAI — Server-Side Ad Insertion với MediaTailor
- [ ] Biết trường hợp dùng IVS thay vì MediaLive + MediaPackage
- [ ] Có thể giải thích CDN caching strategy cho video content
- [ ] Biết cách monitor và alert cho live streaming pipeline
- [ ] Hiểu trade-offs về chi phí giữa các dịch vụ media

---

## 🗺️ Các Bước Tiếp Theo (Next Steps)

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học (Mới bắt đầu / Trung cấp / Nâng cao)
├─ 3️⃣  Bắt đầu với 01-fundamentals/ (codec, protocols)
├─ 4️⃣  Thực hành MediaConvert job trên AWS Console
├─ 5️⃣  Xây dựng live pipeline với MediaLive + MediaPackage
├─ 6️⃣  Thử nghiệm IVS với low-latency live streaming
└─ 7️⃣  Chuẩn bị câu hỏi phỏng vấn từ 10-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
