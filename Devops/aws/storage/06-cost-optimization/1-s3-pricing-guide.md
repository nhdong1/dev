# Bảng Giá S3 — Storage, Requests, Data Transfer

> Hiểu cấu trúc chi phí Amazon S3 là nền tảng để tối ưu hóa chi phí lưu trữ. S3 tính phí theo ba chiều: dung lượng lưu trữ, số lượng API requests và lưu lượng truyền dữ liệu ra ngoài.

## 📚 Mục Lục

1. [Cấu Trúc Chi Phí S3](#1-cấu-trúc-chi-phí-s3)
2. [Chi Phí Lưu Trữ Theo Storage Class](#2-chi-phí-lưu-trữ-theo-storage-class)
3. [Chi Phí API Requests](#3-chi-phí-api-requests)
4. [Chi Phí Data Transfer](#4-chi-phí-data-transfer)
5. [Các Khoản Phí Ẩn](#5-các-khoản-phí-ẩn)
6. [Tính Toán Chi Phí Thực Tế](#6-tính-toán-chi-phí-thực-tế)
7. [Chiến Lược Giảm Chi Phí S3](#7-chiến-lược-giảm-chi-phí-s3)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Cấu Trúc Chi Phí S3

### Ba Thành Phần Chính

```
Tổng Chi Phí S3 = Lưu Trữ + Requests + Data Transfer (Truyền Dữ Liệu)

┌──────────────────┬────────────────────┬────────────────────┐
│   LƯU TRỮ        │    REQUESTS        │  DATA TRANSFER     │
│   (Storage)      │   (API Calls)      │   (Egress)         │
├──────────────────┼────────────────────┼────────────────────┤
│ Tính theo GB/    │ Tính theo số lần   │ Tính theo GB ra    │
│ tháng            │ gọi API            │ ngoài AWS          │
│                  │                   │                    │
│ Phụ thuộc vào   │ GET, PUT, LIST,    │ Ra Internet: có phí│
│ storage class    │ DELETE, HEAD       │ Vào S3: miễn phí  │
│                  │                   │ Cùng region: miễn  │
└──────────────────┴────────────────────┴────────────────────┘
```

### Chu Kỳ Thanh Toán

- Tính theo **tháng lịch** (calendar month)
- Dung lượng tính theo **GB-month** — trung bình trọng số theo thời gian trong tháng
- Ví dụ: Lưu 100GB trong 15 ngày = 50 GB-month (= 100 × 15/30)

---

## 2. Chi Phí Lưu Trữ Theo Storage Class

### Bảng Giá (us-east-1, 2026)

| Storage Class | Giá/GB-month | Dung Lượng Tối Thiểu | Thời Gian Tối Thiểu |
|--------------|-------------|----------------------|---------------------|
| **S3 Standard** | $0.023 | Không | Không |
| **S3 Intelligent-Tiering** | $0.023 (frequent) / $0.0125 (infrequent) | Không | Không |
| **S3 Standard-IA** | $0.0125 | 128 KB | 30 ngày |
| **S3 One Zone-IA** | $0.01 | 128 KB | 30 ngày |
| **S3 Glacier Instant Retrieval** | $0.004 | 128 KB | 90 ngày |
| **S3 Glacier Flexible Retrieval** | $0.0036 | 40 KB | 90 ngày |
| **S3 Glacier Deep Archive** | $0.00099 | 40 KB | 180 ngày |

### Lưu Ý Quan Trọng Về Dung Lượng Tối Thiểu

```
Mỗi object trong Standard-IA, Glacier được tính ít nhất theo kích thước tối thiểu:

Object 1 KB trong Standard-IA → Bị tính 128 KB
Object 50 KB trong Standard-IA → Bị tính 128 KB
Object 200 KB trong Standard-IA → Bị tính 200 KB (đúng kích thước)

→ Object nhỏ không nên để ở Standard-IA hoặc Glacier!
  Chi phí thực cao hơn nhiều so với Standard với object nhỏ.
```

### Lưu Ý Về Thời Gian Lưu Trữ Tối Thiểu

```
Xóa object trước thời gian tối thiểu → Vẫn bị tính phí cho đủ thời gian tối thiểu

Ví dụ:
- Lưu object vào Standard-IA ngày 1
- Xóa ngày 15 (sau 15 ngày, trước 30 ngày tối thiểu)
- Bị tính phí cho 30 ngày đầy đủ

→ Không chuyển object xuống IA/Glacier nếu sẽ xóa trong thời gian ngắn!
```

---

## 3. Chi Phí API Requests

### Bảng Giá Requests (us-east-1)

| Loại Request | Giá | Ví Dụ |
|-------------|-----|--------|
| **PUT, COPY, POST, LIST** | $0.005 / 1,000 requests | Upload, copy, list bucket |
| **GET, SELECT, và tất cả các loại khác** | $0.0004 / 1,000 requests | Download, metadata |
| **DELETE** | Miễn phí | Xóa object |
| **LIFECYCLE Transitions** | $0.01 / 1,000 transitions | Chuyển storage class |

### Tính Toán Nhanh

```
Scenario: 1 triệu GET requests/ngày = 30 triệu requests/tháng
  Chi phí requests = 30,000,000 / 1,000 × $0.0004 = $12/tháng

Scenario: 100,000 PUT requests/ngày = 3 triệu requests/tháng
  Chi phí requests = 3,000,000 / 1,000 × $0.005 = $15/tháng
```

### Requests Thực Tế Cần Tính

```
Web application phục vụ ảnh cho người dùng:
  - Người dùng xem ảnh profile → GET request
  - Upload ảnh mới → PUT request
  - CDN cache miss → GET request

Nếu không dùng CDN → Mỗi page view = nhiều GET requests → Chi phí cao
Nếu dùng CloudFront → CDN cache hit = không tốn S3 request
```

### Tối Ưu Requests

```
Vấn đề: Ứng dụng gọi ListObjects nhiều lần để tìm object
Giải pháp: Cache danh sách object hoặc dùng database index

Vấn đề: Ứng dụng check HEAD trước mỗi GET
Giải pháp: Xử lý 404 exception thay vì check trước

Vấn đề: Lambda function list bucket mỗi invocation
Giải pháp: Cache kết quả với TTL (Time to Live — Thời Gian Tồn Tại)
```

---

## 4. Chi Phí Data Transfer

### Bảng Giá Data Transfer

| Hướng Truyền | Chi Phí |
|-------------|---------|
| **Internet → S3** (Inbound — Vào) | **Miễn phí** |
| **S3 → Internet** (Outbound — Ra) đầu 1 GB/tháng | **Miễn phí** |
| **S3 → Internet** $0.09/GB (đến 10 TB) | $0.09/GB |
| **S3 → Internet** $0.085/GB (10–50 TB) | $0.085/GB |
| **S3 → Internet** $0.07/GB (50–150 TB) | $0.07/GB |
| **S3 → EC2 cùng Region** | **Miễn phí** |
| **S3 → EC2 khác Region** | $0.02/GB |
| **S3 → CloudFront** | **Miễn phí** |

### Ví Dụ Chi Phí Data Transfer

```
Website với ảnh: 50,000 user/ngày, mỗi user download 2 MB ảnh từ S3

Transfer/ngày = 50,000 × 2 MB = 100 GB/ngày
Transfer/tháng = 3,000 GB = 3 TB

Chi phí data transfer = 3,000 GB × $0.09 = $270/tháng (Chỉ transfer!)
Ngoài ra còn chi phí lưu trữ và requests.

→ Giải pháp: Dùng CloudFront (CDN)
  S3 → CloudFront: Miễn phí
  CloudFront → User: ~$0.0085/GB (rẻ hơn nhiều và cache giảm requests)
```

### S3 Transfer Acceleration — Tăng Tốc Truyền Tải

```
Mục đích: Tăng tốc upload từ xa (user ở xa region AWS)
Cơ chế: Route qua CloudFront edge location gần nhất → Tới S3 qua AWS backbone

Chi phí thêm:
  - $0.04/GB transfer vào (cho file upload nhanh hơn)
  - $0.08/GB transfer ra

Khi nào dùng:
  ✅ Upload file lớn từ vùng địa lý xa region S3
  ✅ Người dùng phân tán toàn cầu upload lên cùng bucket
  ❌ Không hiệu quả nếu user gần region S3
```

---

## 5. Các Khoản Phí Ẩn

### Phí Retrieve (Truy Xuất) từ IA và Glacier

| Storage Class | Phí Retrieve | Thời Gian Truy Xuất |
|--------------|-------------|---------------------|
| Standard-IA | $0.01/GB | Tức thì |
| One Zone-IA | $0.01/GB | Tức thì |
| Glacier Instant | $0.03/GB | Tức thì |
| Glacier Flexible — Expedited (Khẩn) | $0.03/GB | 1–5 phút |
| Glacier Flexible — Standard (Thường) | $0.01/GB | 3–5 giờ |
| Glacier Flexible — Bulk (Hàng Loạt) | $0.0025/GB | 5–12 giờ |
| Glacier Deep Archive — Standard | $0.02/GB | 12 giờ |
| Glacier Deep Archive — Bulk | $0.0025/GB | 48 giờ |

### Phí S3 Replication — Nhân Bản

```
CRR/SRR (Cross/Same-Region Replication — Sao Chép Liên/Cùng Vùng):
  - Phí PUT cho object sao chép: $0.005/1,000 objects
  - Data transfer liên vùng: $0.02/GB (CRR)
  - Lưu trữ ở bucket đích: Tính theo storage class của bucket đích

Ví dụ: Replication 1 TB data sang region khác/tháng:
  Transfer: 1,000 GB × $0.02 = $20
  Lưu trữ đích: 1,000 GB × $0.023 = $23
  Tổng thêm: $43/tháng
```

### Phí S3 Object Lambda — Lambda Xử Lý Object

```
Khi dùng S3 Object Lambda (biến đổi data khi GET):
  - Lambda invocations: Tính theo giá Lambda thường
  - Data processing: $0.0049/GB
  - Requests: $0.005/1,000 requests
```

### Phí Lifecycle Transitions — Chuyển Tầng

```
Mỗi lần lifecycle rule chuyển object sang class khác:
  Sang Standard-IA / One Zone-IA: $0.01/1,000 objects
  Sang Glacier Instant: $0.02/1,000 objects
  Sang Glacier Flexible/Deep Archive: $0.05/1,000 objects

Ví dụ: 1 triệu object cần chuyển xuống Glacier mỗi tháng:
  Chi phí transition: 1,000,000 / 1,000 × $0.05 = $50
```

---

## 6. Tính Toán Chi Phí Thực Tế

### Ví Dụ 1: Media Storage (Lưu Trữ Truyền Thông)

```
Scenario: Nền tảng chia sẻ ảnh
  - 10,000 ảnh mới/ngày, trung bình 2 MB/ảnh
  - Upload 1 tháng → 300,000 ảnh, 600 GB mới
  - Tổng đang lưu: 5 TB (5,000 GB)
  - Truy cập: 1 triệu GET/ngày, 100,000 PUT/ngày
  - CDN đã cache: 90% requests không hit S3

Tính toán tháng:
  Lưu trữ: 5,000 GB × $0.023 = $115
  GET requests (10% hit S3): 100,000/ngày × 30 × $0.0004/1,000 = $1.2
  PUT requests: 100,000/ngày × 30 × $0.005/1,000 = $15
  Data transfer (10% hit S3): 100,000 × 2 MB × 10% × $0.09/GB = $1.8

  Tổng ước tính: ~$133/tháng

Nếu không dùng CDN:
  GET requests: 1,000,000/ngày × 30 × $0.0004/1,000 = $12
  Data transfer: 1,000,000 × 2 MB × $0.09/GB = $180
  Tổng: ~$319/tháng (chi phí cao hơn $186 vì không có CDN)
```

### Ví Dụ 2: Log Archive (Lưu Trữ Log)

```
Scenario: Lưu trữ log ứng dụng 7 năm
  - Tạo 100 GB log/ngày
  - Sau 30 ngày: chuyển Standard → Standard-IA
  - Sau 90 ngày: chuyển → Glacier Flexible
  - Sau 7 năm: xóa

Phân tích dung lượng sau 1 năm:
  Standard (30 ngày gần nhất): 100 GB × 30 = 3,000 GB
  Standard-IA (ngày 31–90): 100 GB × 60 = 6,000 GB
  Glacier Flexible (ngày 91–365): 100 GB × 275 = 27,500 GB

Chi phí lưu trữ/tháng (sau khi đã ổn định):
  Standard: 3,000 GB × $0.023 = $69
  Standard-IA: 6,000 GB × $0.0125 = $75
  Glacier Flexible: 27,500 GB × $0.0036 = $99

  Tổng: ~$243/tháng

Nếu để tất cả trong Standard:
  36,500 GB × $0.023 = $840/tháng
  → Tiết kiệm được $597/tháng = 71% chi phí!
```

### Công Cụ Tính Chi Phí

```bash
# AWS Pricing Calculator (online tool — công cụ tính giá online)
# Truy cập: calculator.aws/pricing/2/

# AWS CLI — xem chi phí hiện tại
aws ce get-cost-and-usage \
  --time-period Start=2026-04-01,End=2026-05-01 \
  --granularity MONTHLY \
  --metrics "BlendedCost" "UsageQuantity" \
  --filter '{
    "Dimensions": {
      "Key": "SERVICE",
      "Values": ["Amazon Simple Storage Service"]
    }
  }' \
  --group-by '[{"Type":"DIMENSION","Key":"USAGE_TYPE"}]'
```

---

## 7. Chiến Lược Giảm Chi Phí S3

### 1. Tối Ưu Storage Class

```
Nguyên tắc: Dữ liệu truy cập ít hơn → Lưu ở class rẻ hơn

Access Pattern → Storage Class phù hợp:
  Hàng ngày     → Standard
  Hàng tuần     → Standard hoặc Intelligent-Tiering
  Hàng tháng    → Standard-IA
  Hàng quý      → Glacier Instant
  Hàng năm/hiếm → Glacier Flexible hoặc Deep Archive
  Không rõ      → Intelligent-Tiering
```

### 2. Dùng CloudFront CDN

```
Lợi ích kép:
  1. S3 → CloudFront miễn phí (không tốn data transfer)
  2. CloudFront cache giảm số GET requests đến S3
  3. CloudFront → User rẻ hơn S3 → User

Áp dụng cho:
  ✅ Static website assets (JS, CSS, images)
  ✅ Video streaming
  ✅ Software downloads
  ❌ Private data cần authentication phức tạp (cần cấu hình thêm)
```

### 3. Xóa Object Thường Xuyên

```bash
# Bật expiration lifecycle rule — quy tắc xóa tự động
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-logs-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "delete-old-logs",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Expiration": {"Days": 2555}
    }]
  }'
  # 2555 ngày = 7 năm
```

### 4. Xóa Incomplete Multipart Uploads — Dọn Dẹp Upload Dang Dở

```bash
# Upload dở không hoàn thành vẫn tốn phí!
# Bật lifecycle để xóa tự động sau 7 ngày

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "cleanup-incomplete-multipart",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }]
  }'
```

### 5. Tối Ưu Versioning

```
Versioning tốt cho bảo vệ dữ liệu nhưng dễ làm chi phí tăng nhanh:
  - Mỗi lần PUT object = 1 version mới
  - Version cũ vẫn tốn tiền lưu trữ
  - Không có lifecycle → Chi phí tăng vô hạn

Giải pháp: Lifecycle rule cho version cũ

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "expire-old-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90,
        "NewerNoncurrentVersions": 3
      }
    }]
  }'
  # Giữ 3 version gần nhất, xóa version >90 ngày
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: S3 tính phí theo những tiêu chí nào?**

> S3 tính phí theo ba thành phần chính: (1) Dung lượng lưu trữ tính theo GB-month, phụ thuộc vào storage class — Standard $0.023, Glacier Deep Archive $0.00099/GB-month. (2) API Requests — PUT/LIST tốn hơn GET, thường $0.005 và $0.0004 per 1,000 requests. (3) Data Transfer ra ngoài AWS — $0.09/GB ra Internet, miễn phí vào S3 và giữa S3 và EC2 cùng region.

**Q: Tại sao không nên để tất cả object nhỏ trong Standard-IA?**

> Standard-IA có dung lượng tính phí tối thiểu là 128KB. Object 1KB sẽ bị tính như 128KB, tức là chi phí thực tế gấp 128 lần so với kích thước thật. Với object nhỏ truy cập thường xuyên, Standard thường rẻ hơn Standard-IA. Ngoài ra Standard-IA còn có phí retrieve $0.01/GB, nên nếu truy cập thường xuyên thì tổng chi phí có thể cao hơn Standard.

**Q: Làm thế nào để giảm chi phí data transfer S3?**

> Ba chiến lược chính: (1) Dùng CloudFront — S3 sang CloudFront miễn phí, CDN cache giảm cả số requests lẫn transfer; (2) EC2 cùng region — traffic S3 ↔ EC2 cùng region miễn phí nên deploy ứng dụng gần data; (3) VPC Endpoint — traffic không qua internet, tránh phí NAT Gateway và data transfer.

**Q: Incomplete multipart uploads ảnh hưởng chi phí như thế nào?**

> Khi upload bị gián đoạn mà không abort, các phần đã upload (parts) vẫn chiếm dung lượng và bị tính phí lưu trữ như object thông thường nhưng object chưa hoàn thành. Giải pháp là cấu hình lifecycle rule với `AbortIncompleteMultipartUpload` để tự động xóa sau một số ngày nhất định, thường 7 ngày.

---

**Phiên Bản:** 1.0
**Cập Nhật:** 2026-05-16
