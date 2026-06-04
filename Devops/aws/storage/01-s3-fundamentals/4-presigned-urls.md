# Presigned URLs — URL Có Chữ Ký Tạm Thời

> Presigned URL (URL có chữ ký được tạo trước) cho phép cấp quyền truy cập tạm thời vào S3 object mà không cần chia sẻ AWS credentials — rất quan trọng trong thiết kế ứng dụng web và mobile.

---

## 1. Presigned URL Là Gì?

**Presigned URL** là URL chứa thông tin xác thực AWS được nhúng vào (embedded credentials) và có thời hạn hết hiệu lực. Bất kỳ ai có URL này đều có thể thực hiện thao tác đã được ký (GET, PUT, DELETE) trong thời gian URL còn hợp lệ.

```
URL thông thường (public object):
https://my-bucket.s3.amazonaws.com/document.pdf
→ Cần public access hoặc authentication

Presigned URL:
https://my-bucket.s3.amazonaws.com/document.pdf
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKIAIOSFODNN7EXAMPLE/20260515/ap-southeast-1/s3/aws4_request
  &X-Amz-Date=20260515T100000Z
  &X-Amz-Expires=3600
  &X-Amz-SignedHeaders=host
  &X-Amz-Signature=abc123...
→ Ai có URL này đều download được trong 3600 giây (1 giờ)
```

---

## 2. Cơ Chế Hoạt Động

```
┌────────────┐    1. Request download link    ┌─────────────┐
│   Client   │ ──────────────────────────────► │  App Server │
│ (Trình     │                                 │ (Backend)   │
│  duyệt/    │ ◄────────────────────────────── │             │
│  mobile)   │    2. Trả về presigned URL      └──────┬──────┘
└────────────┘                                        │
      │                                               │ 3. Tạo presigned URL
      │ 4. Request trực tiếp đến S3 bằng URL         │    dùng AWS SDK
      │                                               ▼
      │                                    ┌──────────────────┐
      └───────────────────────────────────►│   Amazon S3      │
                                           │                  │
                                           │ 5. Xác thực chữ  │
                                           │    ký trong URL  │
                                           │                  │
                                           │ 6. Trả về file   │
                                           │    hoặc 403      │
                                           └──────────────────┘
```

**Lợi ích:**
- Backend không cần làm proxy — S3 phục vụ file trực tiếp → giảm tải server
- Không lộ AWS credentials
- Kiểm soát được thời gian truy cập

---

## 3. Tạo Presigned URL Để Download (GET)

### Bằng AWS CLI

```bash
# Tạo URL có hiệu lực 1 giờ (3600 giây)
aws s3 presign s3://my-bucket/private/report.pdf --expires-in 3600

# Kết quả (URL dài):
https://my-bucket.s3.ap-southeast-1.amazonaws.com/private/report.pdf?X-Amz-Algorithm=...
```

### Bằng Python SDK (boto3)

```python
import boto3
from botocore.exceptions import ClientError

def generate_presigned_download_url(bucket_name: str, object_key: str, expiry_seconds: int = 3600) -> str:
    """Tạo presigned URL để download file từ S3."""
    s3_client = boto3.client('s3', region_name='ap-southeast-1')

    try:
        url = s3_client.generate_presigned_url(
            ClientMethod='get_object',
            Params={
                'Bucket': bucket_name,
                'Key': object_key,
                # Tùy chọn: buộc trình duyệt download thay vì hiển thị inline
                'ResponseContentDisposition': f'attachment; filename="{object_key.split("/")[-1]}"'
            },
            ExpiresIn=expiry_seconds
        )
        return url
    except ClientError as e:
        raise RuntimeError(f"Không thể tạo presigned URL: {e}")

# Sử dụng
url = generate_presigned_download_url(
    bucket_name='my-company-docs',
    object_key='contracts/2026/contract-001.pdf',
    expiry_seconds=900  # 15 phút
)
print(url)
```

### Bằng JavaScript SDK (AWS SDK v3)

```javascript
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3Client = new S3Client({ region: "ap-southeast-1" });

async function generatePresignedDownloadUrl(bucket, key, expiresIn = 3600) {
  const command = new GetObjectCommand({ Bucket: bucket, Key: key });
  const url = await getSignedUrl(s3Client, command, { expiresIn });
  return url;
}

// Sử dụng trong Express route
app.get('/download/:fileId', async (req, res) => {
  const url = await generatePresignedDownloadUrl(
    'my-bucket',
    `files/${req.params.fileId}`,
    900 // 15 phút
  );
  res.redirect(url);
});
```

---

## 4. Tạo Presigned URL Để Upload (PUT)

Cho phép client upload trực tiếp lên S3 mà không qua backend — giảm băng thông server và tăng tốc độ upload:

```python
def generate_presigned_upload_url(
    bucket_name: str,
    object_key: str,
    content_type: str,
    max_size_bytes: int,
    expiry_seconds: int = 300
) -> str:
    """Tạo presigned URL để client upload trực tiếp lên S3."""
    s3_client = boto3.client('s3', region_name='ap-southeast-1')

    url = s3_client.generate_presigned_url(
        ClientMethod='put_object',
        Params={
            'Bucket': bucket_name,
            'Key': object_key,
            'ContentType': content_type,
            'ContentLength': max_size_bytes,
            # Mã hóa phía server
            'ServerSideEncryption': 'AES256'
        },
        ExpiresIn=expiry_seconds
    )
    return url

# Client upload bằng HTTP PUT
# curl -X PUT \
#   -H "Content-Type: image/jpeg" \
#   --data-binary @photo.jpg \
#   "https://my-bucket.s3.amazonaws.com/uploads/photo.jpg?..."
```

### Flow Upload Trực Tiếp Lên S3

```
1. Client → Backend: "Tôi muốn upload file avatar.jpg"
2. Backend → S3: generate_presigned_url(PUT, "uploads/user-123/avatar.jpg", expires=5min)
3. Backend → Client: { "upload_url": "https://...", "key": "uploads/user-123/avatar.jpg" }
4. Client → S3: PUT request với file (dùng upload_url)
5. S3 → Client: 200 OK
6. Client → Backend: "Upload xong rồi, key là uploads/user-123/avatar.jpg"
7. Backend → DB: Lưu key vào database
```

---

## 5. Presigned URL Với POST Policy (Upload Form)

Dành cho upload qua HTML form (đặc biệt hữu ích cho ứng dụng web truyền thống):

```python
def generate_presigned_post(bucket_name: str, object_key_prefix: str) -> dict:
    """Tạo presigned POST policy để upload qua HTML form."""
    s3_client = boto3.client('s3', region_name='ap-southeast-1')

    response = s3_client.generate_presigned_post(
        Bucket=bucket_name,
        Key=f'{object_key_prefix}/${{filename}}',  # Cho phép filename động
        Fields={
            'Content-Type': 'image/jpeg',
            'acl': 'private',
        },
        Conditions=[
            ['content-length-range', 1, 10 * 1024 * 1024],  # 1 byte → 10MB
            ['starts-with', '$Content-Type', 'image/'],      # Chỉ cho phép ảnh
            {'acl': 'private'},
        ],
        ExpiresIn=600  # 10 phút
    )
    return response
    # Trả về: { 'url': 'https://...', 'fields': { 'key': '...', 'AWSAccessKeyId': '...', ... } }
```

```html
<!-- Sử dụng trong HTML form -->
<form action="{{ post_data.url }}" method="post" enctype="multipart/form-data">
  {% for key, value in post_data.fields.items() %}
    <input type="hidden" name="{{ key }}" value="{{ value }}">
  {% endfor %}
  <input type="file" name="file" accept="image/*">
  <input type="submit" value="Upload">
</form>
```

---

## 6. Thời Hạn Hiệu Lực (Expiry)

| Phương Pháp Ký | Thời Hạn Tối Đa |
|----------------|-----------------|
| IAM User credentials | 7 ngày (604,800 giây) |
| IAM Role / Instance Profile | **Thời gian session của role** (thường 1 giờ) |
| STS temporary credentials | Thời gian token còn lại |

> **Quan trọng:** Nếu tạo presigned URL bằng IAM Role và đặt `ExpiresIn=7200` nhưng role session chỉ còn 1 giờ → URL sẽ hết hạn sau 1 giờ, **không phải** 2 giờ.

### Khuyến Nghị Thời Hạn

| Use Case | Thời Hạn Nên Dùng |
|----------|------------------|
| Download tài liệu ngay | 5–15 phút |
| Upload file người dùng | 5–10 phút |
| Email link download | 24 giờ |
| Chia sẻ nội bộ | 1–7 ngày |
| API callback có retry | 30 phút |

---

## 7. Bảo Mật Presigned URL

### Rủi Ro

```
❗ URL bị rò rỉ → bất kỳ ai có URL đều truy cập được trong thời gian còn hiệu lực
❗ Ghi log → URL (kèm signature) có thể xuất hiện trong access logs
❗ Crawlers → URL có thể bị index nếu paste vào web public
```

### Biện Pháp Giảm Thiểu Rủi Ro

```python
# 1. Thời hạn ngắn nhất có thể
ExpiresIn = 300  # 5 phút thay vì 3600

# 2. Dùng CloudFront Signed URL thay thế cho nội dung CDN
# → Invalidate URL mà không cần thay đổi S3

# 3. Kiểm tra IP hoặc Referer bằng bucket policy
# (chỉ khi dùng pre-signed URL với VPC endpoint)

# 4. Không log URL đầy đủ — che signature
import re
def mask_presigned_url(url: str) -> str:
    return re.sub(r'X-Amz-Signature=[^&]+', 'X-Amz-Signature=***', url)
```

### Vô Hiệu Hóa Presigned URL Sớm

S3 **không có cơ chế revoke** (thu hồi) presigned URL trực tiếp. Để vô hiệu hóa:

```
Cách 1: Xóa hoặc rotate credentials IAM User đã ký → URL ngay lập tức invalid
         (Nhưng ảnh hưởng toàn bộ URL ký bằng credentials đó)

Cách 2: Đổi tên object (key) → URL cũ trỏ về key không tồn tại → 404

Cách 3: Dùng CloudFront + Lambda@Edge để có revoke granular hơn

Cách 4: Object Lock hoặc Bucket Policy để chặn cụ thể
```

---

## 8. Presigned URL vs CloudFront Signed URL

| | Presigned URL | CloudFront Signed URL |
|--|--------------|----------------------|
| **Phục vụ qua** | S3 trực tiếp | CloudFront CDN |
| **Tốc độ** | Phụ thuộc Region | Nhanh hơn (edge caching) |
| **Revoke** | Không trực tiếp | Có (invalidation) |
| **Geo-restriction** | Không | Có |
| **Chi phí** | Rẻ hơn | Tốn thêm CloudFront |
| **Use case** | Download/Upload 1 lần | Media streaming, CDN |

---

## 9. Ví Dụ Thực Tế: API Download Có Xác Thực

```python
# FastAPI + boto3: Secure download endpoint
from fastapi import FastAPI, Depends, HTTPException
from fastapi.responses import RedirectResponse

app = FastAPI()

async def get_current_user(token: str):
    # Logic xác thực JWT token của bạn
    ...

@app.get("/api/files/{file_id}/download")
async def download_file(file_id: str, current_user = Depends(get_current_user)):
    # Kiểm tra quyền
    if not user_has_access(current_user, file_id):
        raise HTTPException(status_code=403, detail="Không có quyền truy cập")

    # Lấy S3 key từ DB
    s3_key = get_file_key_from_db(file_id)

    # Tạo presigned URL
    url = generate_presigned_download_url(
        bucket_name='my-company-docs',
        object_key=s3_key,
        expiry_seconds=300  # 5 phút
    )

    # Redirect trình duyệt thẳng đến S3
    return RedirectResponse(url=url, status_code=302)
```

---

## Câu Hỏi Phỏng Vấn

**Q: Presigned URL khác với public URL như thế nào?**
> Public URL yêu cầu object phải công khai (public-read ACL hoặc bucket policy). Presigned URL cấp quyền tạm thời cho object private — không cần thay đổi quyền object.

**Q: Tại sao nên dùng presigned URL thay vì để backend làm proxy download?**
> Presigned URL cho phép client download trực tiếp từ S3, không qua server → giảm băng thông và CPU của server, tăng throughput, tận dụng AWS network. Backend chỉ tạo URL và trả về.

**Q: Làm thế nào để presigned URL không bị lạm dụng khi bị rò rỉ?**
> (1) Đặt thời hạn ngắn (vài phút). (2) Dùng CloudFront Signed URL có thể revoke. (3) Dùng Lambda@Edge để kiểm tra thêm điều kiện (IP, user agent). (4) Log và monitor các request bất thường vào S3.

**Q: Presigned URL có hoạt động khi bucket bật Block Public Access không?**
> Có. Block Public Access ngăn truy cập **public** (không xác thực), nhưng presigned URL dùng **authenticated credentials** nhúng trong URL — hai cơ chế này độc lập nhau.

**Q: IAM Role vs IAM User — cái nào nên dùng để tạo presigned URL?**
> IAM Role được khuyến nghị vì không có long-term credentials. Nhưng cần chú ý thời hạn của role session — URL không thể có thời hạn dài hơn session còn lại. Với IAM User, có thể đặt thời hạn lên 7 ngày nhưng IAM User có access key là rủi ro bảo mật.

---

## Điều Hướng

- **Tiếp theo:** [5-multipart-upload.md](./5-multipart-upload.md) — Multipart Upload
- **Quay lại:** [3-versioning-and-mfa-delete.md](./3-versioning-and-mfa-delete.md) — Versioning và MFA Delete
- **Index:** [../INDEX.md](../INDEX.md) — Toàn bộ chỉ mục

---

**Cập Nhật Lần Cuối:** 2026-05-15
