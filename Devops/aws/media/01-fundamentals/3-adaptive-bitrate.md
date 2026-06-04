# ABR — Adaptive Bitrate Streaming — Phát Thích Ứng Tốc Độ Bit

> ABR là công nghệ cốt lõi giúp Netflix, YouTube và mọi nền tảng streaming hiện đại hoạt động mượt mà trên mọi mạng từ 3G đến WiFi gigabit. Hiểu ABR là hiểu trái tim của video streaming.

## 📚 Mục Lục

1. [Tại Sao Cần ABR?](#1-tại-sao-cần-abr)
2. [Cơ Chế Hoạt Động ABR](#2-cơ-chế-hoạt-động-abr)
3. [Bitrate Ladder — Thang Tốc Độ Bit](#3-bitrate-ladder--thang-tốc-độ-bit)
4. [ABR Algorithms — Thuật Toán ABR](#4-abr-algorithms--thuật-toán-abr)
5. [CMAF và ABR](#5-cmaf-và-abr)
6. [Low-Latency ABR](#6-low-latency-abr)
7. [Thiết Kế Bitrate Ladder Tối Ưu](#7-thiết-kế-bitrate-ladder-tối-ưu)
8. [Trong AWS MediaConvert & MediaPackage](#8-trong-aws-mediaconvert--mediapackage)

---

## 1. Tại Sao Cần ABR?

### Vấn Đề Của Progressive Download

Trước ABR, video được phục vụ theo kiểu **progressive download** (tải dần):

```
Server phục vụ 1 quality (ví dụ: 1080p 5Mbps)

Viewer có băng thông tốt (10Mbps):  ✅ Xem mượt
Viewer có băng thông kém (1Mbps):   ❌ Phải buffer liên tục
Viewer trên mobile 3G:              ❌ Không thể xem được
```

**Vấn đề:** Không thể phục vụ cùng một file cho mọi người dùng với băng thông khác nhau.

### ABR Giải Quyết Thế Nào?

```
Server chuẩn bị NHIỀU rendition (bản chất lượng khác nhau):
  1080p @ 5Mbps
  720p  @ 2.5Mbps
  480p  @ 1Mbps
  360p  @ 600Kbps
  240p  @ 300Kbps

Player TỰ ĐỘNG chọn rendition phù hợp:
  Băng thông 8Mbps   → chọn 1080p ✅
  Băng thông 3Mbps   → chọn 720p ✅
  Băng thông 800Kbps → chọn 360p ✅
  Mạng kém đột ngột  → tự động hạ xuống 240p ✅
```

**Kết quả:** Viewer luôn xem được video dù mạng tốt hay kém — không bao giờ bị màn hình loading trắng.

---

## 2. Cơ Chế Hoạt Động ABR

### Segment-Based ABR (ABR Dựa Trên Segment)

ABR hoạt động theo cơ chế **segment**:

```
Video được chia thành các segment nhỏ (thường 2-10 giây):

Rendition 1080p:  |seg001.ts|seg002.ts|seg003.ts|seg004.ts|...
Rendition 720p:   |seg001.ts|seg002.ts|seg003.ts|seg004.ts|...
Rendition 480p:   |seg001.ts|seg002.ts|seg003.ts|seg004.ts|...

Player tải từng segment → có thể CHUYỂN RENDITION giữa các segment:

Đang xem:  1080p    1080p    720p↓   480p↓   720p↑   1080p↑
Segment:   seg001   seg002   seg003  seg004  seg005  seg006
Sự kiện:   Normal   Normal   Mạng↓   Mạng↓↓  Mạng↑   Mạng↑↑
```

### ABR Player Loop (Vòng Lặp Player ABR)

```
[PLAYER ABR LOOP]

1. Tải Master Playlist (index.m3u8)
2. Đọc danh sách renditions và bitrate của từng rendition
3. Đo băng thông hiện tại (throughput estimation)
4. Chọn rendition phù hợp
5. Tải segment từ rendition đó
6. Decode và phát
7. Đo throughput của segment vừa tải
8. Kiểm tra buffer hiện tại
9. Quay lại bước 3 để chọn rendition cho segment tiếp theo
```

### Throughput Estimation — Đo Băng Thông

Player ước tính băng thông bằng cách đo tốc độ tải từng segment:

```
Ví dụ segment 720p (6 giây, 2.5Mbps):
  Kích thước segment: 2.5Mbps × 6s = 15Mb = 1.875MB
  Thời gian tải thực tế: 0.8 giây
  Throughput đo được: 1.875MB / 0.8s = 2.34MB/s = 18.75Mbps

So sánh với bitrate của các rendition:
  1080p 5Mbps  → 18.75Mbps >> 5Mbps → CÓ THỂ tải 1080p
  720p 2.5Mbps → đang tải → ổn
  
→ Nâng lên 1080p cho segment tiếp theo ✅
```

### Buffer Management — Quản Lý Bộ Đệm

Player duy trì một buffer (bộ đệm) phía trước vị trí đang phát:

```
Thời gian phát hiện tại: 0:02:30
Buffer hiện tại:         0:02:30 → 0:03:00 (30 giây buffer)

Quyết định tải segment tiếp theo:
  Buffer > 30s → có thể tải quality cao hơn (conservative)
  Buffer 15-30s → duy trì quality hiện tại
  Buffer < 15s → cân nhắc hạ quality
  Buffer < 5s  → PHẢI hạ quality ngay (tránh rebuffering)
```

**Rebuffering** — Tái Đệm: khi player chạy hết buffer và phải dừng để tải thêm → trải nghiệm tệ nhất cho người dùng.

---

## 3. Bitrate Ladder — Thang Tốc Độ Bit

### Khái Niệm

**Bitrate ladder** — thang tốc độ bit: tập hợp các rendition (cặp resolution + bitrate) được sắp xếp từ thấp đến cao. Mỗi rendition phục vụ một segment băng thông khác nhau.

```
Bitrate Ladder (ví dụ Netflix-style):

Resolution  │ Bitrate  │ Target viewer
────────────┼──────────┼────────────────────────────
240p        │ 235 Kbps │ 3G, rất chậm
360p        │ 560 Kbps │ 3G tốt, 4G yếu
480p        │ 1.35 Mbps│ 4G, WiFi chậm
720p (HD)   │ 2.5 Mbps │ 4G LTE, WiFi thường
1080p (FHD) │ 4.3 Mbps │ WiFi tốt, cáp quang
1080p (HQ)  │ 8 Mbps   │ Cáp quang, download
4K (UHD)    │ 16 Mbps  │ Gigabit WiFi
```

### Nguyên Tắc Thiết Kế Bitrate Ladder

**1. Khoảng cách bitrate giữa các rendition:**
```
Quá gần:  720p@2.5Mbps, 720p@2.8Mbps → Player không đổi rendition, lãng phí
Quá xa:   720p@2.5Mbps, 1080p@8Mbps  → Jump chất lượng quá lớn, gây giật

Lý tưởng: 30-50% tăng bitrate giữa mỗi bậc
  360p@560Kbps → 480p@1.35Mbps (+141%) ← hơi xa nhưng chấp nhận được
  720p@2.5Mbps → 1080p@4.3Mbps (+72%)  ← tốt
```

**2. Resolution không nhất thiết tăng theo mỗi bậc:**
```
Tốt hơn: 1080p@4Mbps và 1080p@8Mbps (cùng resolution, khác bitrate)
Mục đích: Cải thiện chất lượng cho viewer có băng thông cao nhưng màn hình không phải 4K
```

**3. Không phải mọi resolution đều hữu ích:**
```
4K trên smartphone 5 inch: lãng phí bandwidth
→ Cần per-device adaptation (Netflix dùng AI để quyết định)
```

### Content-Aware Encoding — Mã Hoá Thích Nghi Nội Dung

Phương pháp truyền thống: encode mọi video với cùng bitrate ladder cố định.

**Vấn đề:** Video "Khoảng Trời Tĩnh" (ít chuyển động) và "Trận Bóng Đá" (nhiều chuyển động) cần bitrate khác nhau để đạt cùng chất lượng.

**Giải pháp — Per-Title Encoding (Netflix, 2015):**
```
Video ít chuyển động (hoạt hình, documentary):
  1080p chỉ cần 2Mbps để đạt chất lượng tốt
  → Hạ bitrate → tiết kiệm 60% bandwidth

Video nhiều chuyển động (thể thao, action):
  1080p cần 6Mbps mới đủ chất lượng
  → Giữ bitrate cao → đảm bảo quality
```

**AWS QVBR** — Quality-Defined Variable Bitrate — Tốc Độ Bit Thay Đổi Theo Chất Lượng: MediaConvert tự động điều chỉnh bitrate theo độ phức tạp của nội dung trong khi duy trì target chất lượng (VMAF/SSIM score).

---

## 4. ABR Algorithms — Thuật Toán ABR

### Throughput-Based Algorithm (Thuật Toán Dựa Trên Thông Lượng)

Thuật toán đơn giản nhất: chọn rendition có bitrate thấp hơn throughput đo được:

```python
# Đơn giản hoá
def select_rendition(throughput_mbps, renditions, safety_factor=0.8):
    target_bitrate = throughput_mbps * safety_factor
    # Chọn rendition cao nhất mà bitrate <= target
    return max(r for r in renditions if r.bitrate <= target_bitrate)
```

**Vấn đề:** Throughput fluctuate nhiều → rendition thay đổi liên tục → chất lượng không ổn định.

### Buffer-Based Algorithm (Thuật Toán Dựa Trên Buffer)

Dùng độ dài buffer làm tín hiệu chính thay vì throughput:

```
Buffer > threshold_high  → tăng quality (có đủ dự phòng)
Buffer trong vùng ổn định → giữ nguyên quality
Buffer < threshold_low   → giảm quality ngay lập tức
```

**Ưu điểm:** Ổn định hơn, ít switching hơn. Netflix dùng phương pháp này.

### BOLA — Bitrate Optimal Lyapunov Algorithm

Thuật toán dựa trên lý thuyết điều khiển (control theory), tối ưu hoá cả playback quality và buffer level cùng lúc. Dùng trong DASH.js.

### Model Predictive Control (MPC)

Dự đoán throughput trong tương lai (dùng lịch sử) để đưa ra quyết định tốt hơn:
- Tính toán rendition sequence tối ưu cho N segment tiếp theo
- Cân bằng: quality, switching penalty, rebuffer penalty

---

## 5. CMAF và ABR

### CMAF — Common Media Application Format — Định Dạng Ứng Dụng Media Chung

**Vấn đề:** HLS và DASH cần 2 loại segment khác nhau:
- HLS dùng `.ts` (MPEG-TS)
- DASH dùng `.m4s` (fMP4)

→ MediaPackage phải tạo và lưu 2 bản → tốn storage, CDN, compute.

**Giải pháp CMAF:**
```
Thay vì:
  encode → [HLS segments .ts]  → CloudFront → Apple
         → [DASH segments .m4s] → CloudFront → Android

Dùng CMAF:
  encode → [CMAF chunks .cmf4 / .m4s]
         → HLS manifest (.m3u8) ─┐  CÙNG segments
         → DASH manifest (.mpd) ─┘  được serve

Tiết kiệm:
  - 50% storage (1 bản thay vì 2)
  - 50% CDN cache (1 URL thay vì 2)
  - Đơn giản hoá pipeline
```

### CMAF Chunk vs CMAF Segment

```
CMAF Segment (thông thường):
  ├── 1 segment = 6 giây (= 6 * segment duration)
  └── Player tải cả segment trước khi phát

CMAF Chunk (Low-Latency):
  ├── 1 segment = 6 giây
  ├── nhưng được chia thành nhiều chunk nhỏ (~0.5-1 giây)
  └── Player dùng Chunked Transfer Encoding → nhận và phát từng chunk
                                               không cần chờ cả segment
```

---

## 6. Low-Latency ABR

### Vấn Đề Với HLS/DASH Truyền Thống

```
Standard HLS latency breakdown:
  Encode delay:     1-2 giây  (encoder latency)
  Segment duration: 6 giây    (phải chờ encode xong segment)
  CDN propagation:  1-2 giây
  Player buffer:    12-18 giây (2-3 segments)
  ───────────────────────────
  Total:            20-28 giây từ camera đến viewer
```

Với sự kiện thể thao trực tiếp, 28 giây delay là quá lớn — khán giả có thể nhận được kết quả qua mạng xã hội trước khi xem trên TV!

### LL-HLS — Low-Latency HLS (Apple, 2019)

**Giải pháp:** Partial segments + Blocking playlist reload + Server push (HTTP/2)

```
LL-HLS:
  Partial segment duration: 0.2-1 giây
  Blocking playlist reload: Player gửi request kèm _HLS_msn (sequence number)
                             Server giữ request đến khi có partial segment mới
                             → Không cần polling, giảm overhead

Latency breakdown với LL-HLS:
  Encode delay:          0.5-1 giây
  Partial segment:       0.5-1 giây
  CDN:                   0.2-0.5 giây
  Player buffer:         1-2 giây
  ─────────────────────────────────
  Total:                 2-4 giây ✅
```

### LL-DASH — Low-Latency DASH

**Giải pháp:** Chunked Transfer Encoding (CTE) — Mã Hoá Truyền Theo Khối:

```
Standard HTTP response:
  Client gửi request → Server xử lý xong → Response đầy đủ → Client nhận

Chunked Transfer Encoding (LL-DASH):
  Client gửi request → Server bắt đầu response ngay
                     → Gửi chunk 0.5s khi sẵn sàng
                     → Gửi chunk tiếp theo...
                     → Client decode và phát chunk đầu tiên trong khi chờ chunk sau
```

### Tóm Tắt Latency Modes

| Mode | Latency | Use Case |
|------|---------|----------|
| **Standard HLS/DASH** | 15-30 giây | VOD, không cần realtime |
| **LL-HLS / LL-DASH** | 2-5 giây | Live sports, news |
| **WebRTC / WHEP** | < 1 giây | Interactive live, video call |

---

## 7. Thiết Kế Bitrate Ladder Tối Ưu

### Bitrate Ladder Cho VOD (Netflix-inspired)

```
Thiết bị Mobile (màn hình < 6 inch):
  240p  @ 145 Kbps
  360p  @ 365 Kbps
  480p  @ 730 Kbps
  720p  @ 2 Mbps    ← tối đa hữu ích trên mobile

Thiết bị Desktop/Tablet (màn hình 10-27 inch):
  480p  @ 730 Kbps
  720p  @ 2 Mbps
  1080p @ 4.3 Mbps
  1080p @ 8 Mbps    ← high quality

TV 4K:
  1080p @ 4.3 Mbps
  2160p @ 16 Mbps   ← 4K UHD
  2160p @ 25 Mbps   ← 4K UHD HDR
```

### Bitrate Ladder Cho Live Streaming

Live streaming ưu tiên **stability** (ổn định) hơn **quality** do:
- Không thể pre-encode (phải encode real-time)
- Encoder có giới hạn CPU/ASIC
- Latency sensitive

```
Live SD (360p):
  240p @ 300 Kbps
  360p @ 700 Kbps  ← target

Live HD (720p):
  360p @ 700 Kbps
  540p @ 1.5 Mbps
  720p @ 2.5 Mbps  ← target

Live FHD (1080p):
  480p @ 1 Mbps
  720p @ 2.5 Mbps
  1080p @ 5 Mbps   ← target

Live 4K (UHD):
  720p  @ 2.5 Mbps
  1080p @ 5 Mbps
  2160p @ 12 Mbps  ← target (cần H.265)
```

### AWS MediaLive Bitrate Ladder Mẫu

```json
{
  "EncoderSettings": {
    "VideoDescriptions": [
      {
        "Name": "video_1080p",
        "Width": 1920, "Height": 1080,
        "CodecSettings": {
          "H264Settings": {
            "Bitrate": 5000000,
            "RateControlMode": "CBR",
            "FramerateNumerator": 30,
            "FramerateDenominator": 1
          }
        }
      },
      {
        "Name": "video_720p",
        "Width": 1280, "Height": 720,
        "CodecSettings": {
          "H264Settings": {
            "Bitrate": 2500000,
            "RateControlMode": "CBR"
          }
        }
      },
      {
        "Name": "video_480p",
        "Width": 854, "Height": 480,
        "CodecSettings": {
          "H264Settings": {
            "Bitrate": 1000000,
            "RateControlMode": "CBR"
          }
        }
      }
    ]
  }
}
```

> **CBR trong Live:** MediaLive thường dùng CBR — Constant Bitrate — Tốc Độ Bit Cố Định cho live output vì CDN và downstream router cần biết chính xác bitrate để cấp phát băng thông.

---

## 8. Trong AWS MediaConvert & MediaPackage

### MediaConvert — ABR Output Groups

MediaConvert tạo tất cả rendition trong một job:

```
Job Input (1080p source)
│
├── Output Group: Apple HLS
│   ├── Output 1: 1080p @ 5Mbps (video + audio)
│   ├── Output 2: 720p  @ 2.5Mbps (video + audio)
│   ├── Output 3: 480p  @ 1Mbps (video + audio)
│   └── → S3: s3://bucket/hls/
│
└── Output Group: DASH ISO
    ├── Output 1: 1080p @ 5Mbps
    ├── Output 2: 720p  @ 2.5Mbps
    └── → S3: s3://bucket/dash/
```

**Tự Động Tạo Master Playlist:** MediaConvert tự tạo `index.m3u8` và `index.mpd` với thông tin tất cả rendition.

### MediaPackage — Just-in-Time Packaging

**JIT Packaging** — Just-in-Time Packaging — Đóng Gói Theo Yêu Cầu Tức Thì: MediaPackage không lưu sẵn HLS và DASH riêng. Thay vào đó:

```
Live từ MediaLive → MediaPackage ingest (MPEG-TS hoặc fMP4)

Khi viewer request:
  HLS player  → GET /channel/endpoint-hls/index.m3u8
  DASH player → GET /channel/endpoint-dash/index.mpd

MediaPackage:
  → Đọc từ internal buffer
  → Tạo manifest và package segment theo đúng format (HLS/DASH/CMAF)
  → Trả về response ngay lập tức

Lợi ích JIT:
  - Không lưu trữ duplicate (1 bản ingest → nhiều format phát)
  - Thay đổi bitrate filtering mà không cần encode lại
  - Dễ dàng thêm/bớt rendition cho từng endpoint
```

### MediaPackage Bitrate Filtering

```
Endpoint có thể lọc rendition theo bitrate:
  Endpoint cho Smart TV:  tất cả rendition (240p → 4K)
  Endpoint cho Mobile:    chỉ 240p → 720p (tiết kiệm băng thông)
  Endpoint cho 4K TV:     720p → 4K (bỏ quality thấp)

Cấu hình qua StreamSelection:
  MinVideoBitsPerSecond: 500000    (≥ 500 Kbps)
  MaxVideoBitsPerSecond: 5000000   (≤ 5 Mbps)
```

---

## ❓ Câu Hỏi Ôn Tập

1. Tại sao ABR player đôi khi chọn rendition thấp hơn mức cần thiết? Đây là do thuật toán nào gây ra?
2. CMAF mang lại lợi ích gì cho CDN cost so với dual HLS+DASH packaging?
3. Tại sao live streaming dùng CBR trong khi VOD dùng QVBR/VBR? Trade-off là gì?
4. JIT Packaging trong MediaPackage hoạt động như thế nào? Tại sao hiệu quả hơn pre-packaging?
5. Bitrate ladder cho mobile và 4K TV có nên giống nhau không? Tại sao?

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---------|----------|-------------|
| [2-streaming-protocols.md](./2-streaming-protocols.md) | **3-adaptive-bitrate.md** | [4-drm-fundamentals.md](./4-drm-fundamentals.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn thành
