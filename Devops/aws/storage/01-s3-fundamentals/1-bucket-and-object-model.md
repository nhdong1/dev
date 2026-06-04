# Bucket và Object Model — Mô Hình Lưu Trữ Cốt Lõi của S3

> Hiểu rõ bucket, object, key, metadata, và ETag là nền tảng để làm việc hiệu quả với Amazon S3.

---

## 1. Bucket — Thùng Chứa Dữ Liệu

### Đặc điểm

- **Container** (vùng chứa) cấp cao nhất trong S3
- Tên phải **globally unique** (duy nhất toàn cầu) trên tất cả AWS accounts và regions
- Thuộc về một **AWS Region** cụ thể — dữ liệu **không tự động** di chuyển sang Region khác
- Không có giới hạn số object trong một bucket

### Quy tắc đặt tên bucket

```
✅ Hợp lệ:
  my-company-data-2026
  logs.production.app
  backup-bucket-ap-southeast-1

❌ Không hợp lệ:
  My-Bucket          (chữ hoa không được phép)
  my_bucket          (dấu gạch dưới không được phép)
  192.168.1.1        (không được trùng định dạng IP)
  my-bucket-         (không được kết thúc bằng dấu gạch ngang)
```

| Quy tắc | Chi tiết |
|---------|---------|
| Độ dài | 3–63 ký tự |
| Ký tự được phép | Chữ thường (a-z), số (0-9), dấu chấm (.), dấu gạch ngang (-) |
| Bắt đầu và kết thúc | Phải là chữ hoặc số |
| Không dùng | Chữ hoa, dấu gạch dưới, định dạng IP |

### Giới hạn mặc định

| Giới Hạn | Mặc Định | Có Thể Tăng? |
|----------|----------|--------------|
| Buckets mỗi account | 100 | Có (tối đa 1,000 qua Service Quotas) |
| Objects mỗi bucket | Không giới hạn | — |
| Kích thước tổng | Không giới hạn | — |

---

## 2. Object — Đơn Vị Lưu Trữ Cơ Bản

### Cấu trúc của một Object

```
┌─────────────────────────────────────────────────────────────┐
│                        S3 Object                            │
│                                                             │
│  ┌─────────────────────┐  ┌──────────────────────────────┐  │
│  │        Key          │  │         Value (Data)         │  │
│  │ logs/2026/app.log   │  │  <nội dung file thực tế>     │  │
│  └─────────────────────┘  └──────────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Metadata                         │    │
│  │  System: Content-Type, Content-Length, ETag, ...   │    │
│  │  User:   x-amz-meta-author, x-amz-meta-env, ...   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Version ID  │  │ Storage Class│  │   Access Control │  │
│  │  (nếu bật)   │  │  Standard/IA │  │   ACL / Policy   │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Giới hạn kích thước object

| Thao Tác | Giới Hạn |
|----------|---------|
| Kích thước tối đa một object | 5 TB (5,120 GB) |
| Tải lên qua single PUT | Tối đa 5 GB |
| File lớn hơn 5 GB | Bắt buộc dùng **Multipart Upload** |
| Khuyến nghị dùng Multipart | File > 100 MB |

---

## 3. Key — Tên Định Danh Đối Tượng

### Key là gì?

Key là **tên duy nhất** của object trong một bucket. Không phải đường dẫn file theo nghĩa thông thường — S3 là **flat namespace** (không gian phẳng), không có thư mục thực sự.

```
Bucket: my-company-logs
Key:    2026/05/15/app-server-01.log
        └───────────────────────────┘
              Đây là toàn bộ key (prefix + tên file)
```

### Prefix — Tiền Tố Giả Thư Mục

Dấu `/` trong key tạo ra **prefix** — AWS Console và nhiều công cụ hiển thị prefix như thư mục, nhưng thực ra không có thư mục vật lý:

```
aws s3 ls s3://my-bucket/logs/2026/05/
  → Liệt kê tất cả objects có prefix "logs/2026/05/"
```

### Ảnh hưởng đến hiệu suất

S3 phân phối requests dựa trên các ký tự đầu của key. Tránh đặt prefix tuần tự sẽ tránh **hot partition** (phân vùng quá tải):

```
❌ Tệ (tất cả cùng prefix, gây hot partition):
  2026-05-15-log-001.txt
  2026-05-15-log-002.txt
  2026-05-15-log-003.txt

✅ Tốt (prefix ngẫu nhiên, phân tán đều):
  a3f2/image-001.jpg
  b7c1/image-002.jpg
  e9d4/image-003.jpg
```

> S3 tự động phân vùng từ 2018 — randomize prefix không còn bắt buộc nhưng vẫn là best practice.

---

## 4. Metadata — Siêu Dữ Liệu

### System Metadata (Siêu dữ liệu hệ thống)

AWS tự động gán và quản lý:

| Key | Ý Nghĩa |
|-----|--------|
| `Content-Type` | Loại MIME của file (vd: `image/jpeg`, `text/html`) |
| `Content-Length` | Kích thước object tính bằng byte |
| `Content-MD5` | Checksum MD5 để xác minh tính toàn vẹn |
| `Last-Modified` | Thời điểm tạo hoặc cập nhật object lần cuối |
| `x-amz-version-id` | ID phiên bản (nếu versioning được bật) |
| `x-amz-storage-class` | Storage class đang dùng |
| `x-amz-server-side-encryption` | Thuật toán mã hóa phía server |

### User-defined Metadata (Siêu dữ liệu người dùng)

- Thêm vào lúc PUT object hoặc COPY
- Phải có tiền tố `x-amz-meta-`
- Tối đa **2 KB** tổng kích thước metadata
- Không thể tìm kiếm (search) trực tiếp theo metadata trong S3

```bash
# Thêm user metadata khi upload
aws s3 cp photo.jpg s3://my-bucket/photo.jpg \
  --metadata "author=Nguyen Van A,env=production,project=web-app"

# Xem metadata của object
aws s3api head-object \
  --bucket my-bucket \
  --key photo.jpg
```

**Kết quả head-object:**
```json
{
    "ContentType": "image/jpeg",
    "ContentLength": 2048576,
    "ETag": "\"d41d8cd98f00b204e9800998ecf8427e\"",
    "Metadata": {
        "author": "Nguyen Van A",
        "env": "production",
        "project": "web-app"
    },
    "StorageClass": "STANDARD",
    "LastModified": "2026-05-15T10:30:00+00:00"
}
```

---

## 5. ETag — Định Danh Phiên Bản Nội Dung

### ETag là gì?

**ETag** (Entity Tag — Thẻ Thực Thể) là chuỗi định danh duy nhất cho nội dung của một object. Dùng để:

- Xác minh tính toàn vẹn dữ liệu (data integrity)
- Phát hiện thay đổi nội dung (change detection)
- HTTP caching — trình duyệt dùng ETag để kiểm tra bộ nhớ đệm

### Cách tính ETag

**Trường hợp 1 — Upload thông thường (không Multipart):**
```
ETag = MD5(nội dung file)
Ví dụ: "d41d8cd98f00b204e9800998ecf8427e"
```

**Trường hợp 2 — Multipart Upload:**
```
ETag = MD5(ghép các MD5 của từng part) + "-" + số_parts
Ví dụ: "4d9031c7644d8081c2829f4ea23c55f7-14"
         └──────────────────────────────┘ └──┘
              MD5 của các parts               14 parts
```

> **Lưu ý quan trọng:** ETag từ Multipart Upload **không phải** là MD5 của toàn bộ file — không thể dùng để verify checksum toàn file theo cách thông thường.

### Dùng ETag để kiểm tra tính toàn vẹn

```bash
# Upload file
aws s3 cp myfile.txt s3://my-bucket/myfile.txt

# Lấy ETag từ S3
aws s3api head-object --bucket my-bucket --key myfile.txt \
  --query 'ETag' --output text
# Kết quả: "abc123..."

# Tính MD5 local (chỉ đúng nếu không dùng Multipart Upload)
md5sum myfile.txt
# Kết quả: abc123... myfile.txt

# Nếu hai giá trị trùng → file toàn vẹn
```

---

## 6. Object URL — Địa Chỉ Truy Cập

### URL Chuẩn

```
https://<bucket-name>.s3.<region>.amazonaws.com/<key>

Ví dụ:
https://my-company-data.s3.ap-southeast-1.amazonaws.com/images/logo.png
```

### Path-style URL (cũ, bị deprecated)

```
https://s3.<region>.amazonaws.com/<bucket-name>/<key>
```

### Ví Dụ Thực Tế

```
Bucket: my-company-data
Region: ap-southeast-1 (Singapore)
Key:    images/2026/logo.png

URL: https://my-company-data.s3.ap-southeast-1.amazonaws.com/images/2026/logo.png
```

---

## 7. Thao Tác Cơ Bản Qua AWS CLI

```bash
# Tạo bucket
aws s3 mb s3://my-test-bucket --region ap-southeast-1

# Upload file
aws s3 cp local-file.txt s3://my-test-bucket/folder/local-file.txt

# Upload với storage class và metadata
aws s3 cp large-backup.tar.gz s3://my-bucket/backups/large-backup.tar.gz \
  --storage-class STANDARD_IA \
  --metadata "env=prod,team=backend"

# Download file
aws s3 cp s3://my-test-bucket/folder/local-file.txt ./downloaded-file.txt

# List objects
aws s3 ls s3://my-test-bucket/folder/

# Xóa object
aws s3 rm s3://my-test-bucket/folder/local-file.txt

# Xóa bucket (phải rỗng trước)
aws s3 rb s3://my-test-bucket

# Xóa bucket kể cả nội dung bên trong
aws s3 rb s3://my-test-bucket --force
```

---

## 8. Strong Consistency — Nhất Quán Mạnh

Từ tháng 12/2020, S3 đảm bảo **strong read-after-write consistency** (nhất quán đọc-sau-ghi mạnh):

| Thao Tác | Consistency Trước 12/2020 | Consistency Từ 12/2020 |
|----------|--------------------------|------------------------|
| PUT object mới | Strongly consistent | Strongly consistent |
| PUT overwrite | Eventually consistent | **Strongly consistent** |
| DELETE | Eventually consistent | **Strongly consistent** |
| LIST | Eventually consistent | **Strongly consistent** |

**Ý nghĩa thực tế:** Sau khi ghi thành công, mọi lần đọc tiếp theo đều thấy dữ liệu mới — không cần retry hay delay.

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao tên bucket phải globally unique?**
> S3 dùng tên bucket như một phần của DNS (Domain Name System — Hệ thống Tên Miền). URL `bucket-name.s3.amazonaws.com` phải duy nhất toàn cầu để tránh xung đột DNS.

**Q: ETag có phải lúc nào cũng là MD5 không?**
> Không. Với Multipart Upload, ETag là MD5 của các MD5 ghép lại, kèm suffix `-N` (N là số parts). Với server-side encryption dùng KMS, ETag cũng không phải MD5 của nội dung gốc.

**Q: Có thể tìm kiếm object theo metadata không?**
> Không trực tiếp trong S3. Phải dùng thêm DynamoDB, Elasticsearch, hoặc S3 Inventory + Athena để tạo catalog riêng và tìm kiếm theo metadata.

**Q: S3 có thư mục thực sự không?**
> Không. S3 là flat namespace — chỉ có bucket và objects. Dấu `/` trong key được hiển thị như thư mục bởi Console và CLI nhưng không có cấu trúc thư mục vật lý. Tuy nhiên có **S3 folder** (thực chất là object có key kết thúc bằng `/` và kích thước 0).

---

## Điều Hướng

- **Tiếp theo:** [2-storage-classes.md](./2-storage-classes.md) — So sánh 6 storage classes
- **Quay lại:** [README.md](./README.md) — Tổng quan S3 Fundamentals
- **Index:** [../INDEX.md](../INDEX.md) — Toàn bộ chỉ mục

---

**Cập Nhật Lần Cuối:** 2026-05-15
