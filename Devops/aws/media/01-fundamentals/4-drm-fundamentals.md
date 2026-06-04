# DRM — Digital Rights Management — Quản Lý Quyền Kỹ Thuật Số

> DRM bảo vệ nội dung trả phí khỏi bị sao chép và phân phối trái phép. Đây là yêu cầu bắt buộc để phát hành nội dung có bản quyền (phim, thể thao, nhạc) trên nền tảng OTT.

## 📚 Mục Lục

1. [DRM Là Gì và Tại Sao Cần?](#1-drm-là-gì-và-tại-sao-cần)
2. [Các Hệ Thống DRM Phổ Biến](#2-các-hệ-thống-drm-phổ-biến)
3. [Kiến Trúc DRM — Luồng Hoạt Động](#3-kiến-trúc-drm--luồng-hoạt-động)
4. [Encryption — Mã Hoá Nội Dung](#4-encryption--mã-hoá-nội-dung)
5. [SPEKE — AWS DRM Integration](#5-speke--aws-drm-integration)
6. [Multi-DRM — Bảo Vệ Đa Nền Tảng](#6-multi-drm--bảo-vệ-đa-nền-tảng)
7. [Token-Based Access Control — Kiểm Soát Truy Cập Theo Token](#7-token-based-access-control--kiểm-soát-truy-cập-theo-token)
8. [Trong AWS MediaPackage](#8-trong-aws-mediapackage)

---

## 1. DRM Là Gì và Tại Sao Cần?

### Vấn Đề DRM Giải Quyết

```
Không có DRM:
  Content trên CDN → URL công khai → Bất kỳ ai cũng tải về và phân phối lại
  Netflix: phim bị download và upload lên torrent ngay sau khi phát hành

Với DRM:
  Content trên CDN → Mã hoá → Không xem được nếu không có KEY
  KEY chỉ được cấp khi:
    - Người dùng đã đăng nhập
    - Đã thanh toán subscription / mua content
    - Request từ thiết bị được uỷ quyền (authorized device)
```

### DRM vs Encryption Đơn Giản

DRM không chỉ là mã hoá file — đó là **hệ sinh thái kiểm soát quyền**:

```
AES-128 Encryption đơn giản:
  Content → encrypt → CDN → Player tải key → Decrypt → Phát
  Vấn đề: Key được truyền trong bản rõ (cleartext) → dễ bị lấy

DRM (Widevine/FairPlay):
  Content → encrypt (CENC) → CDN
  Key → License Server (bảo mật cao)
  Player → chứng minh danh tính → xin license → Decrypt trong TEE
                                                  (Trusted Execution Environment)
  Vấn đề được giải quyết:
    - Key không bao giờ expose ra ngoài hardware sandbox
    - Enforce playback rules: no recording, no screenshot, expire time
    - Hardware-level protection (L1 Widevine)
```

### Business Requirements — Yêu Cầu Kinh Doanh DRM

| Yêu Cầu | DRM Giải Pháp |
|---------|--------------|
| Chỉ subscriber mới xem được | License chỉ cấp sau khi xác thực subscription |
| Không screenshot / record màn hình | L1 Widevine, FairPlay enforce ở OS level |
| Content hết hạn sau 48h | License có expiry timestamp |
| Giới hạn xem đồng thời (3 thiết bị) | License server kiểm tra concurrency |
| Khác nhau theo địa lý (geo-restriction) | License server check IP/location |
| Offline download có thời hạn | Offline license với expiry |

---

## 2. Các Hệ Thống DRM Phổ Biến

### Widevine (Google)

```yaml
Chủ sở hữu: Google
Nền tảng: Chrome, Android, Chromecast, Android TV, Fire TV
License: Miễn phí cho integrator, Google control
```

**Security Levels — Cấp Độ Bảo Mật Widevine:**

| Level | Tên | Bảo Mật | Thiết Bị |
|-------|-----|---------|---------|
| **L1** | Hardware DRM | Cao nhất | Android 6+ (hardware TEE), Chromecast Ultra |
| **L2** | Software + Hardware | Trung bình | Ít phổ biến |
| **L3** | Software-only | Thấp nhất | PC Chrome, Android cũ, emulator |

```
L1 (Hardware TEE — Trusted Execution Environment):
  Decryption và rendering xảy ra bên trong secure processor
  OS chính không thể truy cập plaintext video
  → Cho phép phát 1080p/4K Netflix, Disney+

L3 (Software):
  Chạy trong normal browser process
  → Netflix giới hạn ở 540p (480p cho một số content)
  → Không cho phép download offline
```

**Kiểm tra Widevine Level:**
Truy cập `chrome://components` hoặc dùng test page để xem L1/L3.

### FairPlay (Apple)

```yaml
Chủ sở hữu: Apple
Nền tảng: Safari (macOS/iOS), iPhone, iPad, Apple TV, AirPlay
License: Phải đăng ký Apple Developer Program
Đặc điểm: CHỈ hoạt động trên Apple ecosystem
```

**FairPlay là bắt buộc nếu muốn DRM trên iOS/Safari:**
```
Không có FairPlay:
  iOS player → HLS stream → DRM? → Chỉ có AES-128 (dễ bypass)
  → Netflix phải từ chối phát trên browser Safari không có FairPlay

Có FairPlay:
  iOS player → HLS + FairPlay → Hardware decrypt (Secure Enclave) → Play
  → Bảo mật tương đương Widevine L1 trên Android
```

**FairPlay Streaming Protocol:**
```
Player (Safari/iOS) → SPC (Server Playback Context) → License Server
License Server      → CKC (Content Key Context) → Player → Decrypt
```

### PlayReady (Microsoft)

```yaml
Chủ sở hữu: Microsoft
Nền tảng: Edge (Windows), Internet Explorer, Xbox, Windows Media Player
License: Có phí (Microsoft license)
```

**PlayReady Security Levels:**
- **SL150**: Software DRM (thấp)
- **SL2000**: Hardware DRM trung bình (Surface, Xbox)
- **SL3000**: Hardware DRM cao nhất (Xbox, Zune, certified devices)

**Khi Nào Cần PlayReady:**
- Target: Xbox One/Series, Windows 10 Edge, smart TV Panasonic/Philips
- IPTV set-top boxes (nhiều dùng PlayReady)
- Enterprise Windows deployment

---

## 3. Kiến Trúc DRM — Luồng Hoạt Động

### Luồng DRM Đầy Đủ

```
PACKAGING (Xử lý trước — offline):
  ┌──────────────────────────────────────────────────────┐
  │ Content Key (CEK — Content Encryption Key):          │
  │   - Random 128-bit key                               │
  │   - MediaPackage/SPEKE tạo hoặc nhận từ Key Server  │
  │                                                      │
  │ Video/Audio → ENCRYPT với CEK → Encrypted Segments  │
  │ CEK → LICENSE SERVER lưu trữ (không lưu trong CDN!) │
  └──────────────────────────────────────────────────────┘

PLAYBACK (Thời gian thực — online):
  ┌──────────────────────────────────────────────────────┐
  │ 1. PLAYER tải manifest (m3u8/mpd) từ CDN             │
  │    → Phát hiện DRM: "cần FairPlay/Widevine license"  │
  │                                                      │
  │ 2. PLAYER → gửi License Request đến License Server  │
  │    Request gồm:                                      │
  │      - Device credentials (certificate)             │
  │      - Content ID (key ID)                          │
  │      - Auth token (JWT của user đã login)            │
  │                                                      │
  │ 3. LICENSE SERVER xác minh:                          │
  │      ✅ Auth token hợp lệ?                           │
  │      ✅ User có quyền xem content này?               │
  │      ✅ Device được uỷ quyền?                        │
  │      ✅ Concurrency limit chưa vượt?                 │
  │                                                      │
  │ 4. LICENSE SERVER trả về LICENSE:                    │
  │      - Content Key (CEK) — mã hoá bởi device public key │
  │      - Policy: expiry, output restrictions          │
  │                                                      │
  │ 5. PLAYER decrypt LICENSE bằng device private key   │
  │    → Có CEK trong Secure Environment (TEE)          │
  │                                                      │
  │ 6. PLAYER decrypt video segments bằng CEK → PLAY!   │
  └──────────────────────────────────────────────────────┘
```

### Key Rotation — Xoay Khoá

Để tăng bảo mật, có thể xoay CEK định kỳ:

```
Segment 1-100:   Encrypt bằng Key A
Segment 101-200: Encrypt bằng Key B  ← Key mới
Segment 201-300: Encrypt bằng Key C

Player phải xin license mới khi gặp Key ID mới trong manifest
→ Nếu license hết hạn giữa chừng → buộc user phải xác thực lại
```

---

## 4. Encryption — Mã Hoá Nội Dung

### CENC — Common Encryption — Mã Hoá Chung

**CENC** — Common Encryption — Mã Hoá Chung (ISO/IEC 23001-7): chuẩn cho phép cùng một file encrypted được decrypt bởi nhiều DRM system khác nhau (Widevine, PlayReady, FairPlay).

```
Trước CENC:
  Content → Widevine encryption  → Widevine-only file
  Content → PlayReady encryption → PlayReady-only file
  → Phải lưu trữ 2 bản

Sau CENC:
  Content → CENC encryption → 1 encrypted file
                              ├── Widevine   → có thể decrypt
                              ├── PlayReady  → có thể decrypt
                              └── FairPlay   → cần CBCS (Apple)
```

### CENC Encryption Modes

**CENC hỗ trợ 2 mode:**

```
CTR Mode (AES-CTR — Counter Mode):
  - Dùng cho: Widevine, PlayReady
  - Encrypt toàn bộ sample
  - Pattern: không có (encrypt tất cả)

CBCS Mode (AES-CBC — Cipher Block Chaining with Pattern):
  - Dùng cho: FairPlay, Widevine L1, PlayReady
  - Encrypt theo pattern (ví dụ: encrypt 1 block, skip 9 blocks)
  - Hiệu quả hơn (ít CPU hơn), iOS yêu cầu
  - CMAF dùng CBCS
```

> **Thực tế với AWS:** MediaPackage hỗ trợ cả CENC CTR (cho Widevine/PlayReady) và CBCS (cho FairPlay + CMAF). Multi-DRM thường cần cả hai.

### Widevine DASH DRM Manifest

```xml
<MPD>
  <Period>
    <AdaptationSet>
      <ContentProtection schemeIdUri="urn:uuid:EDEF8BA9-..."
                         value="Widevine">
        <cenc:pssh>AAAA...</cenc:pssh>  <!-- PSSH box chứa Widevine header -->
      </ContentProtection>
      <ContentProtection schemeIdUri="urn:uuid:9A04F079-..."
                         value="PlayReady">
        <mspr:pro>...</mspr:pro>  <!-- PlayReady header -->
      </ContentProtection>
      ...
    </AdaptationSet>
  </Period>
</MPD>
```

**PSSH — Protection System Specific Header**: chứa thông tin DRM-specific, nhúng trong manifest/init segment để player biết dùng DRM nào.

---

## 5. SPEKE — AWS DRM Integration

### SPEKE — Secure Packager and Encoder Key Exchange — Trao Đổi Khoá An Toàn Giữa Packager và Encoder

**SPEKE** là giao thức API do AWS phát triển (chuẩn hoá bởi DASH IF) để MediaPackage/MediaConvert giao tiếp với **Key Server** (hệ thống quản lý khoá bên ngoài).

```
AWS Architecture với SPEKE:

MediaLive → MediaPackage
              │
              │ SPEKE API (HTTPS)
              ▼
         ┌──────────────────┐
         │   Key Server     │  ← Có thể là:
         │   (DRM Provider) │    - AWS Elemental Delta (AWS native)
         │                  │    - Axinom DRM
         │                  │    - EZDRM
         │                  │    - BuyDRM (KeyOS)
         │                  │    - Intertrust ExpressPlay
         └──────────────────┘
              │
              │ Keys
              ▼
         MediaPackage → Encrypt content với CEK
```

### SPEKE API Flow

```
MediaPackage → Key Server (SPEKE request):
POST /drm/keys
{
  "Id": "content-id-12345",
  "DRMSystems": ["com.widevine.alpha", "com.apple.fairplay"],
  "RequestToken": "unique-request-token"
}

Key Server → MediaPackage (SPEKE response):
{
  "Id": "content-id-12345",
  "ContentKeys": [
    {
      "CommonSystemId": "urn:uuid:EDEF8BA9-79D6-4ACE-A3C8-27DCD51D21ED",
      "AlgorithmId": "aes128",
      "KeyId": "key-id-hex",
      "Key": "base64-encrypted-content-key",
      "Pssh": "base64-pssh-box"
    }
  ]
}
```

### Cấu Hình SPEKE trong MediaPackage

```json
{
  "Encryption": {
    "SpekeKeyProvider": {
      "Url": "https://keyserver.example.com/speke/v2",
      "RoleArn": "arn:aws:iam::123456789:role/MediaPackageSpekeRole",
      "SystemIds": [
        "edef8ba9-79d6-4ace-a3c8-27dcd51d21ed",  // Widevine
        "94ce86fb-07ff-4f43-adb8-93d2fa968ca2"   // FairPlay
      ],
      "ResourceId": "my-content-id"
    }
  }
}
```

---

## 6. Multi-DRM — Bảo Vệ Đa Nền Tảng

### Tại Sao Cần Multi-DRM?

```
Thực tế OTT platform:
  iOS / Safari:         FairPlay ONLY (Apple không cho dùng Widevine)
  Android / Chrome:     Widevine (hỗ trợ tốt nhất)
  Windows Edge:         PlayReady + Widevine
  Smart TV (Samsung):   Widevine + PlayReady (tuỳ model)
  Roku / Fire TV:       Widevine
  Xbox:                 PlayReady

→ Không có "one DRM for all" → phải triển khai ít nhất Widevine + FairPlay
  (+ PlayReady cho Windows/Xbox)
```

### Multi-DRM Architecture

```
                    ┌─────────────────────────────────┐
                    │          Key Server              │
                    │  (Widevine + FairPlay + PlayReady)│
                    └──────────────┬──────────────────┘
                                   │ SPEKE
                    ┌──────────────▼──────────────────┐
                    │         MediaPackage             │
                    │  CBCS encrypted content          │
                    └──────────────┬──────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
       HLS endpoint          DASH endpoint         CMAF endpoint
       (FairPlay)        (Widevine+PlayReady)    (FairPlay+Widevine)
              │                    │                    │
              ▼                    ▼                    ▼
        iPhone/Safari       Android/Chrome          Samsung TV
        Apple TV            Smart TV                Roku
```

### Multi-DRM Content Packaging

```
Với CMAF (lý tưởng cho multi-DRM):
  1 encrypted stream → serve qua:
    - HLS manifest → FairPlay → iOS
    - DASH manifest → Widevine → Android
    - DASH manifest → PlayReady → Windows

Với HLS + DASH riêng (truyền thống):
  HLS stream (CBCS) → FairPlay → iOS
  DASH stream (CTR)  → Widevine → Android
  → Phải lưu 2 bản
```

---

## 7. Token-Based Access Control — Kiểm Soát Truy Cập Theo Token

DRM kiểm soát quyền truy cập **key**, nhưng cần thêm lớp kiểm soát **manifest và segment** qua CDN.

### Signed URL / Signed Cookie (CloudFront)

```
User login → App Server xác thực → Tạo Signed URL/Cookie
Player     → Gửi request kèm Signed URL → CloudFront verify → Cho phép

CloudFront Signed URL:
https://d123456.cloudfront.net/hls/index.m3u8
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=...
  &X-Amz-Date=20260603T000000Z
  &X-Amz-Expires=86400          ← Hết hạn sau 24 giờ
  &X-Amz-SignedHeaders=host
  &X-Amz-Signature=abc123...
```

**Signed URL vs Signed Cookie:**

| | Signed URL | Signed Cookie |
|-|-----------|--------------|
| **Phạm vi** | 1 file cụ thể | Nhiều file (wildcard) |
| **Dùng cho** | 1 video | Toàn bộ HLS (manifest + segments) |
| **Triển khai** | Đơn giản hơn | Phức tạp hơn (set cookie) |
| **Use case** | VOD download | HLS streaming |

### JWT Token trong License Request

```
User login → Auth Server → JWT (JSON Web Token):
{
  "sub": "user123",
  "exp": 1760000000,
  "entitlements": ["content-456", "plan:premium"],
  "concurrency_id": "session-789"
}

Player gửi License Request:
POST /license
Authorization: Bearer <JWT>
Body: <Widevine license request bytes>

License Server kiểm tra JWT:
✅ Token valid (chữ ký đúng, chưa hết hạn)
✅ user123 có "content-456" trong entitlements
✅ Concurrency check: user123 đang xem tối đa 3 streams
→ Cấp license
```

---

## 8. Trong AWS MediaPackage

### Cấu Hình DRM Endpoint trong MediaPackage

```json
{
  "Id": "hls-fairplay-endpoint",
  "PackagingType": "HLS",
  "HlsPackage": {
    "SegmentDurationSeconds": 6,
    "Encryption": {
      "Method": "SAMPLE-AES",
      "SpekeKeyProvider": {
        "Url": "https://keyserver.example.com/speke",
        "RoleArn": "arn:aws:iam::123456789:role/MediaPackageSpekeRole",
        "SystemIds": ["94ce86fb-07ff-4f43-adb8-93d2fa968ca2"],
        "ResourceId": "fairplay-content"
      }
    }
  }
}
```

### Endpoints Multi-DRM Điển Hình

```
Channel "my-live-channel"
├── Endpoint "hls-fairplay"     → iOS/Safari (FairPlay, CBCS)
├── Endpoint "dash-widevine"    → Android/Chrome (Widevine, CTR)
├── Endpoint "cmaf-multidrm"    → Smart TV (CBCS — FairPlay+Widevine)
└── Endpoint "hls-clear"        → Preview/free content (không DRM)
```

### CloudFront + MediaPackage + DRM

```
Kiến trúc bảo mật đầy đủ:

Viewer → CloudFront (Signed URL/Cookie)
          → MediaPackage Origin (OAC — Origin Access Control)
          → [Encrypted HLS/DASH segments]

Viewer → License Server (JWT Bearer Token)
          → Xác thực entitlement
          → Trả về license
          → Player decrypt → Play
```

---

## ❓ Câu Hỏi Ôn Tập

1. Tại sao Netflix iOS phải dùng FairPlay thay vì Widevine dù Widevine phổ biến hơn?
2. Widevine L1 vs L3 khác nhau thế nào? Ảnh hưởng gì đến chất lượng phát?
3. SPEKE trong AWS là gì? Tại sao không hard-code key vào MediaPackage?
4. CENC CTR và CBCS khác nhau như thế nào? Tại sao CMAF chọn CBCS?
5. Thiết kế DRM architecture cho OTT cần hỗ trợ: iPhone, Android, Samsung Smart TV, Xbox.

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---------|----------|-------------|
| [3-adaptive-bitrate.md](./3-adaptive-bitrate.md) | **4-drm-fundamentals.md** | [5-media-pipeline-concepts.md](./5-media-pipeline-concepts.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn thành
