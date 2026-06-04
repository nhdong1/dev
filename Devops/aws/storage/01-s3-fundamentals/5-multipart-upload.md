# Multipart Upload — Tải Lên Nhiều Phần

> Multipart Upload (Tải lên nhiều phần) cho phép upload file lớn lên S3 bằng cách chia nhỏ thành các phần và upload song song — bắt buộc với file > 5GB, khuyến nghị cho file > 100MB.

---

## 1. Tại Sao Cần Multipart Upload?

### Giới Hạn của Single PUT Upload

| Giới Hạn | Giá Trị |
|---------|--------|
| Kích thước tối đa qua single PUT | 5 GB |
| Kích thước object tối đa trong S3 | 5 TB |
| File từ 5GB đến 5TB | **Bắt buộc** dùng Multipart Upload |

### Vấn Đề Với Upload Lớn Qua Single PUT

```
Sự cố mạng giữa chừng → Phải bắt đầu lại từ đầu (mất thời gian, băng thông)
Không thể song song hóa → Tốc độ bị giới hạn bởi một kết nối duy nhất
Không có resume capability → Mất progress khi disconnect
```

---

## 2. Cách Multipart Upload Hoạt Động

### Quy Trình Ba Bước

```
Bước 1: Khởi tạo (Initiate)
─────────────────────────────
Client → S3: CreateMultipartUpload
S3 → Client: UploadId = "VXBsb2FkIElEIGZvciA2aWWpbmcncyBteS1tb3ZpZS5tMnRzIGluIHVzLWVhc3QtMQ"

Bước 2: Upload từng phần (Upload Parts)
─────────────────────────────────────────
Client → S3: UploadPart(UploadId, PartNumber=1, data[0..100MB])
S3 → Client: ETag: "etag-part-1"

Client → S3: UploadPart(UploadId, PartNumber=2, data[100..200MB])
S3 → Client: ETag: "etag-part-2"

... (có thể song song, thứ tự tùy ý)

Client → S3: UploadPart(UploadId, PartNumber=N, data[last chunk])
S3 → Client: ETag: "etag-part-N"

Bước 3: Hoàn tất (Complete)
────────────────────────────
Client → S3: CompleteMultipartUpload(UploadId, [{PartNumber:1, ETag:...}, ...])
S3: Ghép tất cả parts → Object hoàn chỉnh
S3 → Client: ETag final = MD5(etag1+etag2+...+etagN)-N
```

### Sơ Đồ Song Song

```
File 1GB chia thành 10 phần × 100MB:

Part 1 [0–100MB]   ──────────────────────► S3
Part 2 [100–200MB] ──────────────────────► S3
Part 3 [200–300MB] ──────────────────────► S3  (song song)
...
Part 10[900MB–1GB] ──────────────────────► S3

Thay vì tuần tự:
Part 1 ──► S3 ──► Part 2 ──► S3 ──► ... Part 10 (lâu hơn nhiều)
```

---

## 3. Giới Hạn Kỹ Thuật

| Tham Số | Giá Trị |
|--------|--------|
| Số parts tối đa | **10,000** parts |
| Kích thước tối thiểu mỗi part | **5MB** (ngoại trừ part cuối cùng) |
| Kích thước tối đa mỗi part | **5GB** |
| Kích thước object tối đa | **5TB** |
| UploadId hết hạn | Không hết hạn tự động (phải cleanup thủ công hoặc lifecycle) |

---

## 4. Triển Khai Với boto3 (Python)

### Cách Đơn Giản Dùng Transfer Manager

```python
import boto3
from boto3.s3.transfer import TransferConfig

def upload_large_file(file_path: str, bucket: str, key: str) -> None:
    """Upload file lớn với cấu hình tối ưu."""
    s3_client = boto3.client('s3', region_name='ap-southeast-1')

    # Cấu hình Transfer Manager
    config = TransferConfig(
        multipart_threshold=100 * 1024 * 1024,      # File > 100MB → dùng multipart
        multipart_chunksize=100 * 1024 * 1024,       # Mỗi part = 100MB
        max_concurrency=10,                          # 10 kết nối song song
        use_threads=True
    )

    # Upload — boto3 tự chia phần và ghép lại
    s3_client.upload_file(
        file_path,
        bucket,
        key,
        Config=config,
        Callback=ProgressCallback(file_path)
    )

class ProgressCallback:
    """Hiển thị tiến trình upload."""
    def __init__(self, file_path: str):
        import os
        self._total = os.path.getsize(file_path)
        self._uploaded = 0

    def __call__(self, bytes_transferred: int):
        self._uploaded += bytes_transferred
        percent = (self._uploaded / self._total) * 100
        print(f"\rUpload: {percent:.1f}%", end="")
```

### Cách Thủ Công (Hiểu Rõ Từng Bước)

```python
import boto3
import math
import os

def multipart_upload_manual(file_path: str, bucket: str, key: str, part_size_mb: int = 100):
    """Multipart upload thủ công — minh họa từng bước."""
    s3_client = boto3.client('s3', region_name='ap-southeast-1')
    part_size = part_size_mb * 1024 * 1024
    file_size = os.path.getsize(file_path)

    # Bước 1: Khởi tạo multipart upload
    response = s3_client.create_multipart_upload(
        Bucket=bucket,
        Key=key,
        ServerSideEncryption='AES256',
        StorageClass='STANDARD_IA'
    )
    upload_id = response['UploadId']
    print(f"UploadId: {upload_id}")

    parts = []
    try:
        # Bước 2: Upload từng phần
        with open(file_path, 'rb') as f:
            part_number = 1
            while True:
                data = f.read(part_size)
                if not data:
                    break

                print(f"Uploading part {part_number}...")
                part_response = s3_client.upload_part(
                    Bucket=bucket,
                    Key=key,
                    PartNumber=part_number,
                    UploadId=upload_id,
                    Body=data
                )

                parts.append({
                    'PartNumber': part_number,
                    'ETag': part_response['ETag']
                })
                part_number += 1

        # Bước 3: Hoàn tất upload
        complete_response = s3_client.complete_multipart_upload(
            Bucket=bucket,
            Key=key,
            UploadId=upload_id,
            MultipartUpload={'Parts': parts}
        )
        print(f"Upload hoàn tất: {complete_response['Location']}")

    except Exception as e:
        # Hủy bỏ khi có lỗi — giải phóng storage
        print(f"Lỗi: {e} — Đang hủy multipart upload...")
        s3_client.abort_multipart_upload(
            Bucket=bucket,
            Key=key,
            UploadId=upload_id
        )
        raise
```

---

## 5. Upload Song Song Với concurrent.futures

```python
import concurrent.futures
import threading
import boto3

def parallel_multipart_upload(file_path: str, bucket: str, key: str,
                               part_size_mb: int = 100, max_workers: int = 10):
    """Upload các phần song song để tối đa tốc độ."""
    s3_client = boto3.client('s3', region_name='ap-southeast-1')
    part_size = part_size_mb * 1024 * 1024
    file_size = os.path.getsize(file_path)
    num_parts = math.ceil(file_size / part_size)

    # Khởi tạo
    upload_id = s3_client.create_multipart_upload(Bucket=bucket, Key=key)['UploadId']
    parts = [None] * num_parts
    lock = threading.Lock()

    def upload_part(part_number: int, offset: int, length: int):
        with open(file_path, 'rb') as f:
            f.seek(offset)
            data = f.read(length)

        response = s3_client.upload_part(
            Bucket=bucket, Key=key,
            PartNumber=part_number + 1,
            UploadId=upload_id, Body=data
        )
        with lock:
            parts[part_number] = {'PartNumber': part_number + 1, 'ETag': response['ETag']}

    # Upload song song
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = []
        for i in range(num_parts):
            offset = i * part_size
            length = min(part_size, file_size - offset)
            futures.append(executor.submit(upload_part, i, offset, length))

        concurrent.futures.wait(futures)

    # Hoàn tất
    s3_client.complete_multipart_upload(
        Bucket=bucket, Key=key, UploadId=upload_id,
        MultipartUpload={'Parts': parts}
    )
```

---

## 6. Dọn Dẹp Incomplete Uploads — Vấn Đề Chi Phí

### Vấn Đề

Multipart upload **bị bỏ dở** (do lỗi, client ngắt kết nối) vẫn chiếm dung lượng lưu trữ và tốn phí — AWS tính phí cho từng phần đã upload dù chưa hoàn tất.

```bash
# Xem các incomplete multipart uploads
aws s3api list-multipart-uploads --bucket my-bucket

# Hủy một upload cụ thể
aws s3api abort-multipart-upload \
  --bucket my-bucket \
  --key large-file.zip \
  --upload-id "VXBsb2FkIElE..."
```

### Lifecycle Rule Tự Động Dọn

**Quan trọng:** Thêm lifecycle rule này vào mọi bucket có multipart upload:

```json
{
  "Rules": [
    {
      "ID": "Cleanup incomplete multipart uploads",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

```bash
# Áp dụng lifecycle rule
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle-cleanup.json
```

---

## 7. Multipart Upload Qua AWS CLI

```bash
# AWS CLI tự động dùng multipart upload cho file lớn
aws s3 cp large-file-10gb.zip s3://my-bucket/ \
  --expected-size 10737418240

# Điều chỉnh chunk size và concurrency
aws configure set default.s3.multipart_chunksize 64MB
aws configure set default.s3.multipart_threshold 64MB
aws configure set default.s3.max_concurrent_requests 20

# Upload với progress bar
aws s3 cp large-file.zip s3://my-bucket/ --no-progress
```

---

## 8. Presigned URL Cho Multipart Upload

Cho phép client upload trực tiếp lên S3 mà không qua server, ngay cả với file lớn:

```python
def create_presigned_multipart_upload(bucket: str, key: str, num_parts: int) -> dict:
    """Tạo presigned URL cho từng phần của multipart upload."""
    s3_client = boto3.client('s3', region_name='ap-southeast-1')

    # Khởi tạo
    upload_id = s3_client.create_multipart_upload(Bucket=bucket, Key=key)['UploadId']

    # Tạo presigned URL cho từng phần
    presigned_urls = []
    for part_number in range(1, num_parts + 1):
        url = s3_client.generate_presigned_url(
            ClientMethod='upload_part',
            Params={
                'Bucket': bucket,
                'Key': key,
                'UploadId': upload_id,
                'PartNumber': part_number
            },
            ExpiresIn=3600  # 1 giờ
        )
        presigned_urls.append({'part_number': part_number, 'url': url})

    return {
        'upload_id': upload_id,
        'key': key,
        'presigned_urls': presigned_urls
    }

# API endpoint để frontend nhận URLs và upload trực tiếp
# Frontend: PUT từng phần lên presigned URL, gửi ETag về backend, backend complete
```

---

## 9. So Sánh: Single PUT vs Multipart Upload

| | Single PUT | Multipart Upload |
|--|-----------|----------------|
| **Giới hạn kích thước** | 5GB | 5TB |
| **Khả năng resume** | Không | Có |
| **Upload song song** | Không | Có (10 luồng hoặc hơn) |
| **Tốc độ với file lớn** | Chậm | Nhanh hơn đáng kể |
| **Overhead** | Thấp (1 request) | Cao hơn (nhiều requests) |
| **File nhỏ (<100MB)** | Tốt hơn | Không cần thiết |
| **File lớn (>100MB)** | Không hiệu quả | Khuyến nghị |

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao part tối thiểu 5MB (trừ part cuối)?**
> AWS áp đặt giới hạn này để ngăn lạm dụng — upload file bằng hàng nghìn phần nhỏ làm tốn metadata storage và tăng overhead. Part cuối không cần vì nó là phần còn lại của file, không nhất thiết đủ 5MB.

**Q: ETag của object upload bằng Multipart có phải MD5 không?**
> Không. ETag của Multipart Upload có dạng `MD5(MD5_part1 + MD5_part2 + ...)-N` với N là số parts. Không thể dùng để verify MD5 của toàn bộ file. AWS gần đây hỗ trợ checksum SHA-256/CRC32 riêng để verify toàn vẹn.

**Q: Incomplete multipart uploads gây vấn đề gì?**
> Tốn chi phí lưu trữ vì các phần đã upload vẫn chiếm dung lượng. Phải dùng lifecycle rule `AbortIncompleteMultipartUpload` hoặc gọi `abort_multipart_upload` trong error handler.

**Q: Có thể thay đổi storage class trong khi đang multipart upload không?**
> Có. Chỉ định `StorageClass` trong `create_multipart_upload`. Không thể thay đổi sau khi đã khởi tạo — phải hủy và bắt đầu lại.

---

## Điều Hướng

- **Quay lại:** [4-presigned-urls.md](./4-presigned-urls.md) — Presigned URLs
- **Tổng quan module:** [README.md](./README.md) — S3 Fundamentals Overview
- **Module tiếp theo:** [../02-s3-advanced/](../02-s3-advanced/) — S3 Nâng Cao
- **Index:** [../INDEX.md](../INDEX.md) — Toàn bộ chỉ mục

---

**Cập Nhật Lần Cuối:** 2026-05-15
