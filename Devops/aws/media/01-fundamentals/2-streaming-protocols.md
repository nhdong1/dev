# Streaming Protocols — Giao Thức Truyền Phát Video

> So sánh toàn diện các giao thức streaming: HLS, DASH, RTMP, SRT, WebRTC — hiểu rõ vai trò của từng giao thức trong pipeline media để chọn đúng cho mỗi use case.

## 📚 Mục Lục

1. [Phân Loại Giao Thức](#1-phân-loại-giao-thức)
2. [HLS — HTTP Live Streaming](#2-hls--http-live-streaming)
3. [DASH — Dynamic Adaptive Streaming over HTTP](#3-dash--dynamic-adaptive-streaming-over-http)
4. [RTMP — Real-Time Messaging Protocol](#4-rtmp--real-time-messaging-protocol)
5. [SRT — Secure Reliable Transport](#5-srt--secure-reliable-transport)
6. [WebRTC — Web Real-Time Communication](#6-webrtc--web-real-time-communication)
7. [Bảng So Sánh Tổng Hợp](#7-bảng-so-sánh-tổng-hợp)
8. [Chọn Giao Thức Cho Use Case](#8-chọn-giao-thức-cho-use-case)
9. [Trong AWS Media Services](#9-trong-aws-media-services)

---

## 1. Phân Loại Giao Thức

Giao thức streaming phục vụ 2 mục đích khác nhau trong pipeline:

```
INGEST PROTOCOLS              DELIVERY PROTOCOLS
(Giao thức nhập luồng)        (Giao thức phân phối)
─────────────────────         ──────────────────────
RTMP    ─┐                    HLS   ──→ Apple devices, web, smart TV
SRT     ─┼─→ MediaLive/IVS   DASH  ──→ Android, web, smart TV
RIST    ─┘   (Encode)         CMAF  ──→ Tất cả (kết hợp HLS+DASH)
                               WebRTC ─→ Realtime < 500ms (IVS)

Latency:  ~1-5 giây             ~5-30 giây (HLS/DASH)
                                 < 500ms (WebRTC)
```

> **Key insight:** RTMP và SRT là giao thức để **gửi** luồng từ encoder đến server (ingest). HLS và DASH là giao thức để **phát** luồng từ server đến viewer (delivery). Đây là điểm phân biệt quan trọng thường bị nhầm.

---

## 2. HLS — HTTP Live Streaming

### Tổng Quan

**HLS** — HTTP Live Streaming — Phát Trực Tiếp Qua HTTP: giao thức streaming phổ biến nhất thế giới, do Apple phát triển năm 2009. Phương pháp: chia video thành các segment nhỏ và phân phối qua HTTP thông thường.

```
[Encoder] → [Origin Server] → [CDN] → [Player]

Origin Server chứa:
├── index.m3u8 (Master Playlist — danh sách rendition)
├── 1080p/
│   ├── playlist.m3u8 (Media Playlist — danh sách segment)
│   ├── seg_001.ts
│   ├── seg_002.ts
│   └── ...
└── 720p/
    ├── playlist.m3u8
    └── ...
```

### Cách HLS Hoạt Động (VOD)

```
1. Player tải Master Playlist (index.m3u8)
2. Chọn rendition phù hợp (dựa trên bandwidth và resolution)
3. Tải Media Playlist (playlist.m3u8 của rendition đó)
4. Tải từng segment (.ts) theo thứ tự
5. Decode và phát ngay khi có đủ buffer
```

### Cách HLS Hoạt Động (Live)

```
Media Playlist cho Live (rolling window):
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6        ← mỗi segment ~6 giây
#EXT-X-MEDIA-SEQUENCE:147      ← sequence number (tăng dần)

#EXTINF:6.006,
seg_147.ts
#EXTINF:6.006,
seg_148.ts
#EXTINF:6.006,
seg_149.ts                      ← segment mới nhất

↑ Playlist cập nhật mỗi ~6 giây, xoá segment cũ, thêm segment mới
```

### HLS Latency

```
Standard HLS:
- Segment duration: 6-10 giây
- Player buffer: 2-3 segments
- Total latency: 20-30 giây từ camera đến viewer

Low-Latency HLS (LL-HLS — Apple 2019):
- Partial segments: 0.2 giây / partial
- Blocking playlist reload
- Total latency: 2-5 giây
```

### LL-HLS — Low-Latency HLS

LL-HLS giảm latency bằng cách chia nhỏ hơn:

```
Standard HLS:  ├────seg_1 (6s)────┤├────seg_2 (6s)────┤
LL-HLS:        ├p1┤├p2┤├p3┤├p4┤├p5┤├p6┤ ← partial segments (1s)
               └─────── seg_1 (6s) ───────┘
```

Player không phải chờ đủ segment — có thể bắt đầu phát từ partial segment đầu tiên.

### HLS Encryption & DRM

```
AES-128 Encryption (cơ bản):
#EXT-X-KEY:METHOD=AES-128,URI="https://keyserver.com/key",IV=0x1234...

FairPlay DRM (Apple):
#EXT-X-KEY:METHOD=SAMPLE-AES,URI="skd://keyid..",KEYFORMAT="com.apple.streamingkeydelivery"
```

### AWS & HLS

- **MediaConvert**: xuất HLS output group → S3 → CloudFront
- **MediaLive**: channel output → MediaPackage (HLS endpoint) hoặc S3
- **MediaPackage**: origin packaging → HLS endpoint với DRM
- **IVS**: phát lại HLS (standard latency ~15s hoặc low latency ~5s)

---

## 3. DASH — Dynamic Adaptive Streaming over HTTP

### Tổng Quan

**DASH** — Dynamic Adaptive Streaming over HTTP — Phát Thích Ứng Động Qua HTTP: chuẩn quốc tế mở (ISO/IEC 23009-1, 2012), không do công ty nào kiểm soát. Tương tự HLS nhưng dùng XML manifest (MPD — Media Presentation Description) và fMP4 segments.

### Cấu Trúc MPD (Media Presentation Description)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<MPD xmlns="urn:mpeg:dash:schema:mpd:2011"
     type="dynamic"
     minimumUpdatePeriod="PT6S"
     timeShiftBufferDepth="PT30S">
  <Period>
    <AdaptationSet mimeType="video/mp4" codecs="avc1.640028">
      <!-- Rendition 1080p -->
      <Representation id="1" bandwidth="5000000" width="1920" height="1080">
        <SegmentTemplate
          timescale="90000"
          media="1080p/seg-$Number$.m4s"
          initialization="1080p/init.mp4"
          startNumber="1" duration="540000"/>
      </Representation>
      <!-- Rendition 720p -->
      <Representation id="2" bandwidth="2500000" width="1280" height="720">
        <SegmentTemplate ... />
      </Representation>
    </AdaptationSet>
    <AdaptationSet mimeType="audio/mp4" codecs="mp4a.40.2">
      <Representation id="3" bandwidth="192000">
        <SegmentTemplate ... />
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

### HLS vs DASH — So Sánh Chi Tiết

| Tiêu Chí | HLS | DASH |
|----------|-----|------|
| **Nguồn gốc** | Apple, 2009 | ISO (open), 2012 |
| **Manifest format** | Text (.m3u8) | XML (.mpd) |
| **Segment format** | .ts hoặc .fmp4 | .m4s (fMP4) |
| **iOS/Safari support** | ✅ Native | ❌ JS library cần |
| **Android/Chrome** | ✅ JS library | ✅ Native (ExoPlayer) |
| **Samsung Smart TV** | ✅ | ✅ |
| **DRM** | FairPlay | Widevine, PlayReady |
| **Multi-period** | Hạn chế | ✅ Tốt (cho ad insertion) |
| **Low latency** | LL-HLS (2019) | LL-DASH (CMAF Chunk) |
| **Codec linh hoạt** | H.264, H.265 | H.264, H.265, AV1, VP9 |

### Tại Sao Cần Cả HLS Lẫn DASH?

```
Apple ecosystem (iPhone, iPad, Safari Mac):  ← CHỈ hỗ trợ FairPlay DRM + HLS native
Android (Chrome, ExoPlayer):                 ← Widevine DRM, DASH native
Windows (Edge):                              ← PlayReady DRM

OTT platform không thể chọn 1 trong 2:
→ Phải encode và lưu trữ BOTH HLS và DASH
→ CMAF giải quyết vấn đề này (dùng chung segment)
```

---

## 4. RTMP — Real-Time Messaging Protocol

### Tổng Quan

**RTMP** — Real-Time Messaging Protocol — Giao Thức Tin Nhắn Thời Gian Thực: giao thức do Adobe phát triển, ban đầu cho Flash Player. Hiện tại **RTMP không còn dùng cho playback** (Flash chết năm 2020) nhưng **vẫn là giao thức ingest phổ biến nhất** để gửi luồng từ OBS/encoder đến server.

```
OBS / Encoder phần cứng
     │
     │ RTMP (TCP port 1935)
     ▼
MediaLive / IVS Ingest Endpoint
rtmp://live.example.com:1935/app/stream-key
     │
     │ Encode + Package
     ▼
HLS/DASH để phát tới viewer
```

### RTMP vs RTMPS

- **RTMP**: không mã hoá (port 1935) — không dùng trong môi trường production
- **RTMPS**: RTMP over TLS/SSL (port 443) — **bắt buộc dùng** trong production

```
IVS (Amazon Interactive Video Service) chỉ nhận RTMPS:
rtmps://a1b2c3d4e5f6.global-contribute.live-video.net:443/app/
```

### RTMP Stream Key — Khóa Luồng

**Stream key** — khóa luồng là chuỗi ngẫu nhiên xác thực quyền publish:

```
RTMPS URL:   rtmps://endpoint.example.com:443/app/
Stream Key:  sk_us-east-1_XXXXXXXXXXXXXXXXXX

OBS Config:
  Server:     rtmps://endpoint.example.com:443/app/
  Stream Key: sk_us-east-1_XXXXXXXXXXXXXXXXXX
```

> **Bảo mật:** Stream key là secret — ai có key này có thể publish lên channel của bạn. Cần xoay (rotate) key định kỳ.

### Giới Hạn RTMP

| Hạn Chế | Chi Tiết |
|---------|---------|
| **Latency** | 1-5 giây (tốt cho ingest nhưng không phải realtime) |
| **Firewall** | Port 1935 thường bị block trong corporate network |
| **Codec** | Chỉ H.264 + AAC (không hỗ trợ H.265 hay AV1) |
| **Packet loss** | TCP → retransmit → latency tăng khi mạng xấu |
| **Playback** | ❌ Không dùng cho playback tới end-user nữa |

---

## 5. SRT — Secure Reliable Transport

### Tổng Quan

**SRT** — Secure Reliable Transport — Vận Chuyển An Toàn Đáng Tin Cậy: giao thức mã nguồn mở (Haivision, 2017) được thiết kế để thay thế RTMP cho broadcast và OTT ingest hiện đại. Dùng **UDP** thay vì TCP nhưng có cơ chế phục hồi lỗi (ARQ — Automatic Repeat Request).

### Tại Sao SRT Ra Đời?

```
Vấn đề của RTMP:
  ├── TCP: khi packet loss → retransmit → latency tăng đột biến
  ├── Chỉ hỗ trợ H.264/AAC
  ├── Firewall friendly kém (port 1935)
  └── Không có encryption tích hợp

SRT giải quyết:
  ├── UDP + ARQ: phục hồi packet loss mà không tăng latency nhiều
  ├── AES-128/256 encryption tích hợp
  ├── Chạy trên port 443 (HTTPS port) → không bị firewall block
  └── Codec agnostic (bất kỳ codec nào trong MPEG-TS)
```

### SRT Modes (Chế Độ Kết Nối)

```
Caller (client) → Listener (server):
  srt://server.example.com:4200
  Dùng khi encoder kết nối đến MediaLive SRT listener

Listener (server) ← Caller (client):
  Encoder lắng nghe, server kết nối đến
  Hiếm dùng hơn

Rendezvous:
  Cả hai bên kết nối đến IP:port của nhau đồng thời
  Dùng cho peer-to-peer qua NAT
```

### SRT Latency Buffer

SRT dùng **latency buffer** để tái lắp ráp packet lỗi:

```
Truyền thống (RTMP/TCP):  Packet loss → Retransmit → Latency spike
SRT:                       Packet loss → ARQ trong latency buffer → ổn định

SRT Latency Buffer:
  - Mặc định: 120ms
  - Khuyến nghị production: 3-4x RTT (Round Trip Time)
  - Ví dụ: RTT 50ms → buffer 200ms → tổng latency ~220ms
```

### SRT trong AWS

**AWS MediaLive** hỗ trợ SRT Caller input:

```
MediaLive Input → SRT Caller
  Source A: srt://encoder-a.example.com:4200
  Source B: srt://encoder-b.example.com:4200 (backup)

MediaLive channel → nhận SRT stream từ remote encoder
```

**SRT vs RTMP trong Broadcast Production:**
- OBS Studio hỗ trợ SRT (từ v26)
- FFmpeg, Haivision, Zixi encoder hỗ trợ SRT
- Đang trở thành chuẩn ingest mới của broadcast industry

---

## 6. WebRTC — Web Real-Time Communication

### Tổng Quan

**WebRTC** — Web Real-Time Communication — Giao Tiếp Web Thời Gian Thực: chuẩn mở W3C/IETF cho phép trình duyệt và ứng dụng mobile giao tiếp audio/video trực tiếp peer-to-peer mà không cần plugin. Đây là công nghệ phía sau Zoom, Google Meet, Discord video.

```
Latency đặc trưng:
  RTMP/SRT (ingest):    1-5 giây
  HLS/DASH (delivery):  5-30 giây
  WebRTC:               < 500ms (thường 50-200ms)
```

### Cách WebRTC Hoạt Động

```
SIGNALING (qua WebSocket / HTTP):
  Browser A → [Signaling Server] → Browser B
  Trao đổi: SDP offer/answer, ICE candidates

ICE — Interactive Connectivity Establishment — Thiết Lập Kết Nối Tương Tác:
  Tìm đường kết nối trực tiếp qua NAT:
  ├── STUN — Session Traversal Utilities for NAT: tìm public IP/port
  └── TURN — Traversal Using Relays around NAT: relay khi P2P không được

DATA CHANNEL (UDP/DTLS):
  Audio: Opus codec (mặc định)
  Video: H.264, VP8, VP9, AV1
  Encryption: SRTP — Secure Real-time Transport Protocol (bắt buộc)
```

### WebRTC cho Live Streaming (1-to-many)

WebRTC ban đầu là peer-to-peer (1-to-1), nhưng live streaming cần 1-to-many. Giải pháp: **SFU — Selective Forwarding Unit — Bộ Chuyển Tiếp Chọn Lọc**:

```
Không dùng SFU (P2P mesh):       Dùng SFU:
  Viewer 1 ─┐                     Publisher
  Viewer 2 ─┼─ Publisher           │
  Viewer 3 ─┘                      ▼
  N viewers = N×(N-1) connections  [SFU Server]
  Scale kém                        ├── Viewer 1
                                   ├── Viewer 2
                                   └── Viewer N
                                   1 connection từ publisher
                                   Scale được
```

### Amazon IVS Real-time (WebRTC)

**Amazon IVS** — Interactive Video Service — Dịch Vụ Video Tương Tác: Amazon xây dựng SFU infrastructure, cung cấp WebRTC streaming < 300ms cho IVS Real-time stage:

```
IVS Real-time Stage (WebRTC):
  - Latency: < 300ms (sub-second)
  - Max participants: 12 publishers, 10,000 viewers (via trickle-ice)
  - Use case: talkshows, Q&A, game shows tương tác cao
  - SDK: JavaScript, iOS, Android

IVS Standard/Low-latency (HLS):
  - Standard: ~15 giây (HLS)
  - Low-latency: ~5 giây (LL-HLS)
  - Scale: không giới hạn viewers
```

### WHIP & WHEP — WebRTC Ingest/Egress Chuẩn Mới

**WHIP** — WebRTC-HTTP Ingest Protocol: chuẩn mới (2022) cho phép dùng WebRTC để ingest (thay RTMP) vào server:
```
OBS → WHIP → IVS / Cloudflare / Mux
```

**WHEP** — WebRTC-HTTP Egress Protocol: phân phối WebRTC từ server đến trình duyệt qua HTTP signaling đơn giản.

---

## 7. Bảng So Sánh Tổng Hợp

| Tiêu Chí | HLS | DASH | RTMP | SRT | WebRTC |
|----------|-----|------|------|-----|--------|
| **Vai trò** | Delivery | Delivery | Ingest | Ingest | Cả hai |
| **Transport** | HTTP (TCP) | HTTP (TCP) | TCP | UDP | UDP (DTLS) |
| **Latency** | 5-30s | 5-30s | 1-5s | 0.2-2s | < 500ms |
| **Scale** | Không giới hạn | Không giới hạn | Kém (stateful) | Kém | SFU cần |
| **iOS/Safari** | ✅ Native | ❌ JS cần | ❌ Flash chết | ❌ | ✅ Native |
| **Android/Chrome** | ✅ JS | ✅ Native | ❌ | ❌ | ✅ Native |
| **DRM** | FairPlay | Widevine, PlayReady | ❌ | ❌ | DTLS (khác) |
| **Encryption** | AES-128 / FairPlay | CENC | ❌ (RTMP) / TLS (RTMPS) | AES-128/256 | SRTP bắt buộc |
| **Firewall friendly** | ✅ (port 80/443) | ✅ | ❌ (port 1935) | ✅ (port 443) | Cần TURN |
| **Codec support** | H.264, H.265 | H.264, H.265, AV1 | H.264 only | Bất kỳ | H.264, VP8/9, AV1 |
| **CDN cached** | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Use case chính** | Streaming tới viewer | Streaming tới viewer | OBS → server | Broadcast ingest | Video call, interactive |
| **AWS service** | MediaPackage, IVS | MediaPackage | MediaLive, IVS | MediaLive | IVS Real-time |

---

## 8. Chọn Giao Thức Cho Use Case

### Cho Delivery (Phát Tới Viewer)

```
Cần phủ thiết bị rộng nhất (iOS + Android + Smart TV)?
→ CMAF (HLS + DASH dùng chung segment) hoặc HLS+DASH dual encoding

Chỉ cần iOS / Safari?
→ HLS + FairPlay

Chỉ cần Android / Chrome?
→ DASH + Widevine

Cần latency < 5 giây?
→ LL-HLS hoặc LL-DASH

Cần latency < 1 giây (interactive)?
→ WebRTC (IVS Real-time, Amazon Chime, Agora, Millicast)
```

### Cho Ingest (Từ Encoder Đến Server)

```
Dùng OBS Studio và target simplicity?
→ RTMPS (phổ biến nhất, OBS default)

Cần quality cao cho broadcast production?
→ SRT (chịu mạng không ổn định tốt hơn RTMP)

Cần realtime bidirectional (talk show, interview)?
→ WebRTC (WHIP ingest)

Cần ingest từ IP camera / IoT device?
→ RTSP (Real Time Streaming Protocol) → Kinesis Video Streams
```

### Decision Matrix

| Scenario | Ingest | Delivery |
|----------|--------|----------|
| **YouTube-style VOD** | Upload file | HLS + DASH |
| **Live sports broadcast** | SRT / Contribution link | HLS + DASH + CMAF |
| **Twitch-style gaming** | RTMPS | HLS (LL-HLS cho subscribers) |
| **Interactive shopping live** | RTMPS | IVS (WebRTC hoặc LL-HLS) |
| **Video conference** | WebRTC | WebRTC |
| **OTT linear channel** | MediaConnect (SMPTE 2110) | HLS + DASH + CMAF + DRM |

---

## 9. Trong AWS Media Services

### MediaLive — Input Types Hỗ Trợ

```
RTMP Push:      Encoder → push RTMPS đến MediaLive endpoint
RTMP Pull:      MediaLive → pull từ RTMP URL (e.g., another encoder)
HLS Pull:       MediaLive → pull HLS stream (e.g., upstream encoder output)
SRT Caller:     MediaLive → dial vào SRT listener trên encoder
MP4 File:       S3 URL, dùng cho testing hoặc scheduled content
MediaConnect:   AWS MediaConnect (RIST, Zixi, ST 2110)
```

### IVS — Amazon Interactive Video Service

```
IVS Standard / Low-latency channel:
  Ingest:   RTMPS (OBS, hardware encoder)
  Delivery: HLS (standard ~15s, low-latency ~5s)
  SDK:      iOS, Android, JavaScript

IVS Real-time stage (WebRTC):
  Ingest:   WebRTC (WHIP) hoặc IVS Broadcast SDK
  Delivery: WebRTC < 300ms, tối đa 12 publishers + 10,000 viewers
  SDK:      iOS, Android, JavaScript
```

### MediaPackage — Output Endpoints

```
HLS endpoint:    → Apple devices, Smart TV
DASH endpoint:   → Android, Smart TV
CMAF endpoint:   → Tất cả (HLS+DASH dùng chung segment)
MS Smooth:       → Windows, Xbox (legacy)
```

---

## ❓ Câu Hỏi Ôn Tập

1. Tại sao RTMP vẫn phổ biến cho ingest dù đã cũ và Flash chết? Khi nào nên chuyển sang SRT?
2. Giải thích tại sao OTT platform phải duy trì cả HLS và DASH thay vì chọn một. CMAF giải quyết điều này thế nào?
3. WebRTC dùng P2P — tại sao cần SFU cho live streaming 1-to-many?
4. Latency của HLS là 15-30 giây — tại sao không giảm xuống bằng cách dùng segment 1 giây thay vì 6 giây?
5. Trong MediaLive, khi nào chọn RTMP Pull thay vì RTMP Push?

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---------|----------|-------------|
| [1-codec-container.md](./1-codec-container.md) | **2-streaming-protocols.md** | [3-adaptive-bitrate.md](./3-adaptive-bitrate.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn thành
