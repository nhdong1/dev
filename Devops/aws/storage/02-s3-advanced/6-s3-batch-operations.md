# S3 Batch Operations — Thao Tác Hàng Loạt

> S3 Batch Operations (Thao Tác Hàng Loạt) cho phép thực hiện một thao tác duy nhất trên hàng tỷ S3 object trong một lần chạy — sao chép, đặt tag, khôi phục từ Glacier, áp dụng Object Lock, hoặc gọi Lambda cho mỗi object. Không cần viết code phức tạp hay quản lý infrastructure.

## 📚 Mục Lục

1. [Batch Operations Là Gì?](#1-batch-operations-là-gì)
2. [Các Loại Operation Hỗ Trợ](#2-các-loại-operation-hỗ-trợ)
3. [Luồng Hoạt Động](#3-luồng-hoạt-động)
4. [Tạo Job Batch Operations](#4-tạo-job-batch-operations)
5. [Inventory Report — Danh Sách Object Nguồn](#5-inventory-report--danh-sách-object-nguồn)
6. [Use Cases Thực Tế](#6-use-cases-thực-tế)
7. [Monitoring Và Xử Lý Lỗi](#7-monitoring-và-xử-lý-lỗi)
8. [Chi Phí](#8-chi-phí)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Batch Operations Là Gì?

S3 Batch Operations là dịch vụ managed (được quản lý hoàn toàn) thực hiện thao tác trên danh sách object được xác định trước, với:

- Quản lý tự động tiến trình, retry, và báo cáo kết quả
- Không cần viết code xử lý pagination (phân trang) hay error handling
- Scale lên đến hàng tỷ object
- Audit trail (dấu vết kiểm toán) đầy đủ qua AWS CloudTrail

### Vấn Đề Batch Operations Giải Quyết

```
Tình huống 1: Đã upload 100 triệu object trước khi bật CRR
→ Muốn replicate toàn bộ object cũ sang region khác
→ Không thể viết script thủ công (quá lâu, không có retry, không có report)

Tình huống 2: Muốn áp dụng Object Lock cho 50 triệu object hiện có
→ Làm thủ công sẽ mất hàng tuần và có thể bỏ sót

Tình huống 3: Muốn resize 2 tỷ ảnh thumbnail cũ với thuật toán mới
→ Gọi Lambda cho từng object với quản lý retry tự động

→ Batch Operations giải quyết tất cả: một job, một lần cấu hình
```

---

## 2. Các Loại Operation Hỗ Trợ

### 2.1 S3 Put Object Copy — Sao Chép Object

```
Mục đích: Sao chép object (trong cùng bucket hoặc sang bucket khác)
Use case:
- Replicate object cũ sang bucket khác (trước khi bật replication)
- Chuyển đổi encryption (SSE-S3 → SSE-KMS)
- Thay đổi storage class hàng loạt
- Sao chép metadata hoặc thay thế tag

Tham số quan trọng:
- TargetResource: ARN bucket đích
- StorageClass: Override storage class khi copy
- MetadataDirective: COPY (giữ nguyên) hoặc REPLACE (thay mới)
```

### 2.2 S3 Batch Replication — Sao Chép Hàng Loạt

```
Mục đích: Replicate object theo replication rule đã cấu hình
Use case:
- Sync object đã tồn tại trước khi bật CRR/SRR
- Re-replicate object đã fail trước đó (ReplicationStatus = FAILED)

Khác Put Object Copy:
- Dùng đúng replication rule đã cấu hình, tôn trọng destination config
- Report chi tiết theo replication status
```

### 2.3 S3 Initiate Restore — Khởi Động Restore Từ Glacier

```
Mục đích: Yêu cầu restore nhiều object từ Glacier/Deep Archive cùng lúc
Use case:
- DR drill (diễn tập phục hồi thảm họa) — test restore từ Glacier
- Regulatory audit cần truy cập dữ liệu cũ
- Di chuyển dữ liệu từ Glacier sang storage class khác

Tham số:
- ExpirationInDays: Số ngày giữ bản restore tạm thời
- GlacierJobTier: BULK (rẻ, 5-12 giờ), STANDARD (3-5 giờ), EXPEDITED (1-5 phút, đắt)
```

### 2.4 S3 Put Object Tagging — Thêm/Thay Tag

```
Mục đích: Thêm, cập nhật, hoặc xóa tag cho nhiều object
Use case:
- Retroactively (hồi tố) tag object cũ cho cost allocation
- Đánh dấu data classification (phân loại dữ liệu): PII, public, confidential
- Tag theo dự án hoặc team để theo dõi chi phí

Lưu ý:
- S3 Put Object Tagging: Thêm hoặc thay toàn bộ tag
- S3 Delete Object Tagging: Xóa tất cả tag
```

### 2.5 S3 Put Object ACL — Thay Đổi Access Control

```
Mục đích: Cập nhật ACL (Access Control List) hàng loạt
Use case:
- Chuyển từ public-read sang private cho nhiều object
- Thay đổi ownership sau khi migrate bucket

Lưu ý: AWS khuyên dùng Bucket Policy thay vì ACL cho hầu hết use case
```

### 2.6 S3 Object Lock Retention — Áp Dụng Object Lock

```
Mục đích: Áp dụng hoặc mở rộng Object Lock retention
Use case:
- Apply Object Lock cho object cũ sau khi bật Object Lock trên bucket
- Kéo dài retention period hàng loạt (vd: từ 5 năm → 7 năm theo quy định mới)

Tham số:
- Mode: GOVERNANCE hoặc COMPLIANCE
- RetainUntilDate: Ngày hết hạn retention

Lưu ý: KHÔNG thể rút ngắn retention Compliance Mode qua Batch Operations
```

### 2.7 S3 Object Lock Legal Hold — Áp Dụng Legal Hold

```
Mục đích: Bật hoặc tắt Legal Hold hàng loạt
Use case:
- Khi có litigation (vụ kiện) ảnh hưởng đến nhiều object
- Sau khi điều tra kết thúc, tắt Legal Hold hàng loạt
```

### 2.8 S3 Invoke Lambda Function — Gọi Lambda Cho Mỗi Object

```
Mục đích: Xử lý tùy chỉnh cho mỗi object, không giới hạn use case
Use case:
- Chuyển đổi format file (JPEG → WebP)
- Ẩn danh hóa PII trong file văn bản
- Kiểm tra integrity (tính toàn vẹn) của object
- Populate metadata vào database
- Virus scan hàng loạt

Lambda nhận event JSON với thông tin object, trả về kết quả success/failure/permanent failure.
```

---

## 3. Luồng Hoạt Động

```
Bước 1: Chuẩn bị Manifest (Danh Sách Object)
├── S3 Inventory Report (tự động, khuyên dùng)
└── CSV file tự tạo (object key, version ID)
          │
Bước 2: Tạo Batch Operations Job
├── Chọn operation type
├── Chỉ định manifest
├── Chỉ định IAM role
├── Cấu hình report output bucket
└── Priority và completion report
          │
Bước 3: Review & Confirm
├── AWS tính toán số object sẽ xử lý
├── Ước tính chi phí
└── Xác nhận chạy job
          │
Bước 4: Job Chạy
├── AWS quản lý workers, retry, và tiến trình
├── Track progress qua CloudWatch
└── Xử lý lỗi tự động (retry 3 lần mặc định)
          │
Bước 5: Completion Report
└── CSV report: mỗi object → succeeded/failed/skipped
```

---

## 4. Tạo Job Batch Operations

### 4.1 Qua AWS CLI

```bash
# Tạo job copy tất cả object trong inventory
aws s3control create-job \
  --account-id 123456789012 \
  --manifest '{
    "Spec": {
      "Format": "S3InventoryReport_CSV_20161130",
      "Fields": ["Bucket", "Key", "VersionId"]
    },
    "Location": {
      "ObjectArn": "arn:aws:s3:::my-inventory-bucket/data/2026-05-15T00-00Z/manifest.json",
      "ETag": "abc123def456"
    }
  }' \
  --operation '{
    "S3PutObjectCopy": {
      "TargetResource": "arn:aws:s3:::destination-bucket",
      "StorageClass": "STANDARD_IA",
      "MetadataDirective": "COPY"
    }
  }' \
  --report '{
    "Bucket": "arn:aws:s3:::my-report-bucket",
    "Format": "Report_CSV_20180820",
    "Enabled": true,
    "Prefix": "batch-reports",
    "ReportScope": "AllTasks"
  }' \
  --priority 10 \
  --role-arn "arn:aws:iam::123456789012:role/S3BatchRole" \
  --region us-east-1

# Xem trạng thái job
aws s3control describe-job \
  --account-id 123456789012 \
  --job-id "job-abc123"

# Cập nhật priority của job đang chạy
aws s3control update-job-priority \
  --account-id 123456789012 \
  --job-id "job-abc123" \
  --priority 100

# Hủy job
aws s3control update-job-status \
  --account-id 123456789012 \
  --job-id "job-abc123" \
  --requested-job-status Cancelled
```

### 4.2 IAM Role Cho Batch Operations

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::source-bucket",
        "arn:aws:s3:::source-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::destination-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::report-bucket/*",
        "arn:aws:s3:::manifest-bucket/*"
      ]
    }
  ]
}
```

Trust Policy cho IAM Role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "batchoperations.s3.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

### 4.3 Lambda Handler Cho Invoke Lambda Operation

```python
def lambda_handler(event, context):
    """
    Handler cho S3 Batch Operations Invoke Lambda.
    Mỗi invocation xử lý một task (một object).
    """
    invocation_id = event['invocationId']
    task_id = event['tasks'][0]['taskId']
    bucket = event['tasks'][0]['s3BucketArn'].split(':::')[1]
    key = urllib.parse.unquote_plus(event['tasks'][0]['s3Key'])
    version_id = event['tasks'][0].get('s3VersionId')

    try:
        # Xử lý object tùy ý ở đây
        process_object(bucket, key, version_id)
        
        result_code = 'Succeeded'
        result_string = f'Processed {key}'
        
    except PermanentError as e:
        # Lỗi vĩnh viễn — không retry (vd: object corrupt không thể xử lý)
        result_code = 'PermanentFailure'
        result_string = str(e)
        
    except TemporaryError as e:
        # Lỗi tạm thời — Batch sẽ retry
        result_code = 'TemporaryFailure'
        result_string = str(e)

    return {
        'invocationSchemaVersion': '1.0',
        'treatMissingKeysAs': 'PermanentFailure',
        'invocationId': invocation_id,
        'results': [{
            'taskId': task_id,
            'resultCode': result_code,
            'resultString': result_string
        }]
    }
```

---

## 5. Inventory Report — Danh Sách Object Nguồn

### Hai Cách Tạo Manifest

#### Cách 1: S3 Inventory Report (Khuyên Dùng)

S3 Inventory (Kiểm Kê) tự động tạo báo cáo danh sách object theo lịch:

```bash
# Bật S3 Inventory Report
aws s3api put-bucket-inventory-configuration \
  --bucket source-bucket \
  --id "daily-inventory" \
  --inventory-configuration '{
    "Destination": {
      "S3BucketDestination": {
        "Bucket": "arn:aws:s3:::inventory-bucket",
        "Format": "CSV",
        "Prefix": "inventory"
      }
    },
    "IsEnabled": true,
    "Id": "daily-inventory",
    "IncludedObjectVersions": "All",
    "OptionalFields": ["Size", "LastModifiedDate", "StorageClass", "ETag",
                       "ReplicationStatus", "ObjectLockMode", "ObjectLockRetainUntilDate"],
    "Schedule": { "Frequency": "Daily" }
  }'
```

Inventory report format cho Batch Operations:
```
source-bucket,object-key,version-id
my-bucket,photos/2026/01/img001.jpg,abc123
my-bucket,photos/2026/01/img002.jpg,def456
```

#### Cách 2: CSV File Tự Tạo

Tạo file CSV thủ công cho danh sách cụ thể:

```csv
my-bucket,specific-object-1.pdf,version-abc
my-bucket,specific-object-2.pdf,version-def
my-bucket,specific-object-3.pdf,
```

Upload lên S3 và dùng làm manifest source trong job.

---

## 6. Use Cases Thực Tế

### Use Case 1: Migrate Encryption — Chuyển Đổi Mã Hóa

```
Bài toán: Bucket hiện có 500 triệu object dùng SSE-S3, muốn chuyển sang SSE-KMS
để có audit trail đầy đủ.

Giải pháp:
1. Bật S3 Inventory để lấy danh sách toàn bộ object
2. Tạo Batch Operations job: Put Object Copy
   - TargetResource: cùng bucket (copy in-place)
   - ServerSideEncryption: SSE-KMS
   - SSEKMSKeyId: ARN của KMS key mới
3. Completion report: Kiểm tra object nào fail
4. Verify: Sample một số object xem encryption metadata

Thời gian ước tính cho 500 triệu object: 2–8 giờ tùy size
```

### Use Case 2: Backfill Replication — Sao Chép Object Cũ

```
Bài toán: Bật CRR từ us-east-1 → eu-west-1 hôm nay.
200 triệu object cũ cần được replicate.

Giải pháp:
1. Cấu hình CRR rule (chỉ apply cho object mới)
2. S3 Inventory report → manifest
3. Batch Operations: S3 Batch Replication
   - Chỉ replicate object có ReplicationStatus ≠ COMPLETED
   - Tự động dùng replication config đã cấu hình
4. Monitor completion report để detect failed replications
```

### Use Case 3: Apply Compliance Lock — Áp Dụng Object Lock Sau Khi Bật

```
Bài toán: Bucket mới bật Object Lock. 1 tỷ object cũ chưa có retention.
Regulation yêu cầu áp dụng 7 năm retention.

Giải pháp:
1. S3 Inventory → manifest với VersionId
2. Batch Operations: S3 Object Lock Retention
   - Mode: COMPLIANCE
   - RetainUntilDate: 2033-01-01T00:00:00Z
3. Batch report: Kiểm tra object nào không apply được (vd: đã có retention ngắn hơn)

Lưu ý: Không thể rút ngắn Compliance Mode qua Batch Operations
```

### Use Case 4: Mass Tagging Cho Cost Allocation

```
Bài toán: Kiểm toán phát hiện 300 triệu object không có tag "project" → không phân bổ chi phí được

Giải pháp:
1. S3 Inventory với OptionalFields: Tags
2. Filter manifest: Chỉ lấy object thiếu tag "project"
3. Batch Operations: Put Object Tagging
   - Tag: project=unknown-legacy
4. Sau đó team dùng S3 Storage Lens để phân tích

Alternative: Dùng Invoke Lambda để đặt tag thông minh hơn (đọc path để suy ra project)
```

### Use Case 5: Bulk Image Reprocessing — Xử Lý Lại Ảnh Hàng Loạt

```
Bài toán: Thuật toán thumbnail cũ kém chất lượng.
2 tỷ ảnh cần được resize lại với thuật toán mới.

Giải pháp:
1. Lambda function: Nhận object event, resize ảnh, lưu vào bucket mới
2. S3 Inventory: Lấy danh sách tất cả object trong thumbnail-bucket
3. Batch Operations: Invoke Lambda per object
4. Lambda trả về: Succeeded / TemporaryFailure / PermanentFailure
5. Completion report: Kiểm tra failure rate

Performance estimate:
- Lambda: 1 giây/ảnh × 2 tỷ ảnh = quá lâu nếu concurrency thấp
- Batch Operations tự động scale Lambda concurrency
- Với concurrency 10,000: 2 tỷ / 10,000 = 200,000 giây ≈ 2.3 ngày
```

---

## 7. Monitoring Và Xử Lý Lỗi

### CloudWatch Metrics Của Batch Operations

| Metric | Ý Nghĩa |
| ------ | ------- |
| `NumberOfTasksSucceeded` | Số object xử lý thành công |
| `NumberOfTasksFailed` | Số object fail (permanent) |
| `NumberOfTasksSkipped` | Số object bị skip (không đủ điều kiện) |
| `JobProgressPercentage` | Phần trăm hoàn thành |

### Completion Report — Báo Cáo Hoàn Thành

Sau khi job chạy xong, Batch Operations tạo báo cáo CSV:

```csv
Bucket,Key,VersionId,TaskStatus,ErrorCode,HTTPStatusCode,ResultMessage
my-bucket,object1.jpg,abc123,succeeded,,200,
my-bucket,object2.jpg,def456,failed,AccessDenied,403,Access denied
my-bucket,object3.jpg,ghi789,skipped,,404,Object not found
```

- `succeeded`: Thành công
- `failed`: Thất bại không thể recover
- `skipped`: Object không tồn tại hoặc không đủ điều kiện

### Retry Strategy — Chiến Lược Thử Lại

```
Mặc định: Retry 3 lần cho TemporaryFailure trước khi chuyển sang failed
PermanentFailure: Không retry — ghi vào completion report ngay

Với Invoke Lambda:
- Lambda trả về "TemporaryFailure" → Batch retry sau backoff (độ trễ tăng dần)
- Lambda trả về "PermanentFailure" → Không retry
- Lambda timeout → Xử lý như TemporaryFailure
```

---

## 8. Chi Phí

### Thành Phần Chi Phí

| Thành Phần | Giá |
| ---------- | ---- |
| Batch Operations job | $0.25 / job |
| Số object xử lý | $1.00 / triệu object |
| Chi phí operation | Theo loại (PUT request, Lambda invoke, v.v.) |

### Ví Dụ Tính Chi Phí

```
Scenario: Copy 1 tỷ object (Batch Replication)

Job fee:              1 × $0.25                  = $0.25
Object fee:           1,000 × $1.00              = $1,000.00
PUT request ở đích:   1,000,000,000 × $0.000005  = $5,000.00
Data transfer (CRR):  Size × $0.02               = Tùy

→ Tổng không tính data transfer: ~$6,000.25
→ Rất rẻ so với viết và chạy script tự quản lý trên EC2
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q1: Batch Operations khác gì với viết script Python/boto3 để xử lý object hàng loạt?**

> Batch Operations là managed service — AWS quản lý retry logic, fault tolerance (khả năng chịu lỗi), scaling workers, và completion reporting. Script tự viết phải tự xử lý: pagination qua ListObjects, retry khi throttle, checkpoint để resume nếu script crash, và tổng hợp báo cáo. Với hàng tỷ object, script tự viết rất dễ fail giữa chừng và khó resume. Batch Operations đáng tin cậy hơn và không cần maintain infrastructure.

**Q2: Batch Replication khác gì với CRR/SRR thông thường?**

> CRR/SRR chỉ replicate object MỚI được tạo sau khi bật replication rule. Batch Replication là công cụ one-time để backfill object đã tồn tại trước khi bật rule. Batch Replication cũng có thể re-replicate object có `ReplicationStatus = FAILED` do lỗi trước đó. Cả hai đều tuân theo cùng replication configuration đã cấu hình trên bucket.

**Q3: Lambda trả về "TemporaryFailure" trong Batch Operations nghĩa là gì?**

> Lambda trả về `TemporaryFailure` khi lỗi có thể tự khắc phục sau một thời gian (throttle, network issue, service unavailable). Batch Operations sẽ retry object đó sau backoff period. Khác với `PermanentFailure` (lỗi không thể khắc phục, vd: object corrupt) không được retry và ghi vào completion report là failed. Thiết kế đúng: chỉ dùng `PermanentFailure` cho lỗi thực sự vĩnh viễn để tránh miss xử lý object có thể được fix.

**Q4: Làm sao tạo manifest chỉ chứa object thất bại từ job trước để retry?**

> Completion report của job trước là CSV liệt kê tất cả object và trạng thái. Lọc các dòng có `TaskStatus = failed`, extract cột Bucket/Key/VersionId, tạo CSV manifest mới từ đó. Upload CSV này lên S3 và tạo job mới với manifest source là file đã lọc. Đây là workflow chuẩn khi cần retry partial failures.

**Q5: Batch Operations có thể áp dụng cho object ở Glacier không?**

> Phụ thuộc vào operation: `S3 Initiate Restore` — có thể kích hoạt restore cho object đang ở Glacier. `Put Object Copy` — không thể copy object đang ở Glacier (cần restore trước). Workflow để copy object Glacier: (1) Batch Operations `Initiate Restore` cho toàn bộ, (2) Chờ restore hoàn thành (nhận Event Notification khi xong), (3) Tạo job mới `Put Object Copy` từ bản restored.

---

## 📋 Checklist Trước Khi Chạy Job

- [ ] Kiểm tra IAM role có đủ quyền trên source bucket, destination bucket, và report bucket
- [ ] Verify manifest format đúng (CSV với đúng cột, encode UTF-8)
- [ ] Test với một subset nhỏ (100 object) trước khi chạy full job
- [ ] Cấu hình completion report output (AllTasks để có full visibility)
- [ ] Đặt CloudWatch alarm cho `NumberOfTasksFailed`
- [ ] Với Invoke Lambda: test Lambda handler với sample event trước
- [ ] Tính toán chi phí ước tính (job fee + object fee + operation fee)
- [ ] Cân nhắc priority: 1 (thấp) đến 2,147,483,647 (cao), ảnh hưởng đến thứ tự trong queue

---

**Module 02-s3-advanced hoàn thành!**

**Quay Lại:** [README.md](./README.md) — Tổng quan module
**Module Tiếp Theo:** [03-ebs/](../03-ebs/) — EBS — Elastic Block Store
