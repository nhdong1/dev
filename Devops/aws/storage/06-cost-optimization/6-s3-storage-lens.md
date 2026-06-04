# S3 Storage Lens Dashboard — Phân Tích Toàn Tổ Chức

> S3 Storage Lens — Kính Phân Tích Lưu Trữ S3 cung cấp visibility toàn tổ chức về lưu lượng sử dụng và hoạt động của S3, với insights hành động (actionable insights) để tối ưu chi phí và bảo mật. Đây là công cụ analytics mạnh nhất cho S3 ở quy mô lớn.

## 📚 Mục Lục

1. [Storage Lens Là Gì](#1-storage-lens-là-gì)
2. [Kiến Trúc và Phạm Vi](#2-kiến-trúc-và-phạm-vi)
3. [Metrics Quan Trọng](#3-metrics-quan-trọng)
4. [Free vs Advanced Metrics](#4-free-vs-advanced-metrics)
5. [Thiết Lập Storage Lens Dashboard](#5-thiết-lập-storage-lens-dashboard)
6. [Phân Tích Cost Optimization Với Storage Lens](#6-phân-tích-cost-optimization-với-storage-lens)
7. [Tích Hợp Với S3 Inventory và CloudWatch](#7-tích-hợp-với-s3-inventory-và-cloudwatch)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Storage Lens Là Gì

### Tổng Quan

```
S3 Storage Lens = Analytics Dashboard (Bảng Phân Tích) cho S3

Không cần:
  ❌ Viết code query logs thủ công
  ❌ Tự tổng hợp data từ nhiều bucket/account
  ❌ Xây dựng dashboard từ đầu

Cung cấp:
  ✅ Single pane of glass — Một màn hình cho toàn bộ S3 storage
  ✅ 29+ usage metrics (đo lường sử dụng) miễn phí
  ✅ 35+ advanced metrics trả phí ($0.20/million objects/month)
  ✅ Actionable recommendations — Đề xuất có thể hành động ngay
  ✅ Data export sang S3 để phân tích thêm với Athena/QuickSight
```

### Storage Lens vs CloudWatch vs S3 Server Access Log

```
┌──────────────────┬──────────────┬─────────────────┬────────────────┐
│ Tiêu Chí         │ Storage Lens │ CloudWatch S3   │ Server Access  │
│                  │              │ Metrics         │ Logging        │
├──────────────────┼──────────────┼─────────────────┼────────────────┤
│ Phạm vi          │ Toàn account/│ Từng bucket     │ Từng request   │
│                  │ organization │                 │                │
│ Granularity      │ Daily        │ 1 phút đến 1 ngày│ Per request   │
│ (Độ chi tiết)    │              │                 │                │
│ Historical data  │ 14 ngày      │ 15 tháng        │ Không giới hạn │
│ (Dữ liệu cũ)    │ (free tier)  │                 │ (lưu ở S3)    │
│ Cost insights    │ ✅ Rất tốt  │ ✅ Tốt          │ ❌ Tự xử lý   │
│ Setup complexity │ Đơn giản     │ Trung bình      │ Phức tạp       │
│ Chi phí          │ Free+Advanced│ CloudWatch rates│ Storage cost   │
└──────────────────┴──────────────┴─────────────────┴────────────────┘
```

---

## 2. Kiến Trúc và Phạm Vi

### Phạm Vi Của Storage Lens

```
Cấp Độ Tổ Chức (Organization Level):
  Storage Lens dashboard có thể bao gồm:
  ├── AWS Account 1 (Production)
  │   ├── us-east-1: 50 buckets
  │   ├── us-west-2: 20 buckets
  │   └── eu-west-1: 30 buckets
  ├── AWS Account 2 (Staging)
  │   └── us-east-1: 15 buckets
  └── AWS Account 3 (Dev)
      └── us-east-1: 25 buckets

Tổng: 140 buckets, nhiều accounts, nhiều regions
→ Storage Lens tổng hợp thành MỘT dashboard duy nhất
```

### Cách Dữ Liệu Được Thu Thập

```
S3 Storage Lens thu thập metrics hàng ngày:
  1. AWS scan metadata của tất cả objects trong scope
  2. Tính toán metrics (dung lượng, số object, storage class, v.v.)
  3. Cập nhật dashboard (thường available sau ~48 giờ)
  4. Optionally export sang S3 bucket được chỉ định

Lưu ý:
  - Dữ liệu có độ trễ khoảng 48 giờ (không real-time)
  - Metrics tính theo cuối ngày UTC
  - Free tier lưu 14 ngày lịch sử; Advanced lưu 15 tháng
```

---

## 3. Metrics Quan Trọng

### Storage Metrics — Đo Lường Lưu Trữ

```
Total Storage (Tổng Dung Lượng):
  - Tổng GB đang lưu trữ trên S3
  - Breakdown theo storage class (Standard, IA, Glacier, v.v.)
  - Trend theo thời gian (tăng/giảm)

Object Count (Số Object):
  - Tổng số object trên toàn bộ S3
  - Breakdown theo storage class
  - Average object size (kích thước trung bình)

Current Version Bytes/Count (Phiên Bản Hiện Tại):
  - Dung lượng và số lượng current versions
  - Quan trọng khi dùng versioning để biết "live data" là bao nhiêu

Noncurrent Version Bytes/Count (Phiên Bản Cũ):
  - Dung lượng và số lượng noncurrent versions (versions cũ)
  - Nếu cao → Cần lifecycle rule cho versioned objects

Incomplete Multipart Upload (Upload Dang Dở):
  - Dung lượng bytes từ multipart uploads chưa hoàn thành
  - Nếu >0 và không giảm → Cần AbortIncompleteMultipartUpload lifecycle
```

### Request Metrics — Đo Lường Requests (Advanced)

```
GET Requests: Số lần GET per ngày, per bucket
PUT Requests: Số lần PUT per ngày
LIST Requests: Số lần LIST (thường là request đắt nhất theo request count)
DELETE Requests: Số lần DELETE

→ Dùng để phát hiện:
  - Bucket nào có nhiều GET nhưng ít dung lượng → CDN candidates
  - Bucket nào bị spam LIST requests → Cần investigate ứng dụng
  - Bucket nào không có activity → Candidates để archive hoặc xóa
```

### Cost Efficiency Insights — Insights Hiệu Quả Chi Phí

```
% of data in non-Standard storage (% dữ liệu không ở Standard):
  - Cao → Đã tối ưu tốt
  - Thấp → Có thể còn nhiều cơ hội tiết kiệm

% of objects with lifecycle rules (% object có lifecycle rule):
  - Thấp → Nhiều bucket chưa có lifecycle policy

% of buckets with versioning enabled:
  - Giúp biết bao nhiêu bucket có versioning (có thể tăng chi phí nếu không có lifecycle cho versions)

Object size distribution (Phân bố kích thước object):
  - Nhiều object <128KB → Không phù hợp Standard-IA hay Intelligent-Tiering
```

---

## 4. Free vs Advanced Metrics

### Free Metrics — Miễn Phí

```
Bao gồm 29 usage metrics:
  ✅ Total bytes per bucket/account/organization
  ✅ Object count
  ✅ Storage class breakdown
  ✅ Current vs noncurrent versions
  ✅ Delete markers
  ✅ Incomplete multipart upload bytes
  ✅ 14 ngày lịch sử

Không có:
  ❌ Request metrics (GET/PUT/LIST counts)
  ❌ Data transfer insights
  ❌ Cost optimization recommendations chi tiết
  ❌ CloudWatch metrics publishing
```

### Advanced Metrics — Nâng Cao ($0.20/million objects/month)

```
Thêm 35+ metrics:
  ✅ Tất cả request metrics (GET, PUT, LIST, DELETE)
  ✅ Data transfer metrics (bytes in/out)
  ✅ 15 tháng lịch sử (thay vì 14 ngày)
  ✅ CloudWatch metrics publishing
    → Có thể tạo CloudWatch alarms từ Storage Lens metrics
  ✅ Prefix-level aggregation (tổng hợp theo prefix)
    → Phân tích chi tiết hơn theo "thư mục" trong bucket
  ✅ Activity metrics — Phát hiện bucket không hoạt động
  ✅ S3 Storage Lens recommendations — Đề xuất hành động

Chi phí:
  $0.20 per 1 million objects/month
  
  Ví dụ:
    10 triệu objects → $2/month (rất rẻ)
    100 triệu objects → $20/month
    1 tỷ objects → $200/month
```

### Khi Nào Nên Dùng Advanced

```
✅ Dùng Advanced khi:
  - Quản lý S3 cho tổ chức lớn (nhiều team, nhiều account)
  - Cần phát hiện bucket "zombie" không được dùng
  - Muốn tạo cost alerts từ Storage Lens trong CloudWatch
  - Cần phân tích request patterns để tối ưu ứng dụng
  - Compliance cần lịch sử 15 tháng

❌ Free đủ khi:
  - Chỉ cần xem tổng dung lượng và storage class breakdown
  - Tổ chức nhỏ với ít bucket
  - Chỉ cần snapshot hiện tại, không cần trends dài hạn
```

---

## 5. Thiết Lập Storage Lens Dashboard

### Qua AWS Console

```
1. Mở S3 Console → Storage Lens → Dashboards
2. Click "Create dashboard"
3. Điền:
   - Dashboard name: "org-storage-lens-main"
   - Home Region: Chọn region lưu data (us-east-1)
4. Scope:
   - Include all accounts in organization (nếu có Organizations)
   - Include all regions
   - Include all buckets (hoặc filter theo prefix/tags)
5. Metrics selection: Free hoặc Advanced
6. Data export (optional): Cấu hình export sang S3 bucket
7. Click "Create dashboard"
```

### Qua AWS CLI

```bash
# Tạo Storage Lens dashboard cơ bản
aws s3control put-storage-lens-configuration \
  --account-id 123456789012 \
  --config-id "org-main-dashboard" \
  --storage-lens-configuration '{
    "Id": "org-main-dashboard",
    "AccountLevel": {
      "BucketLevel": {
        "ActivityMetrics": {
          "IsEnabled": true
        }
      }
    },
    "IsEnabled": true,
    "DataExport": {
      "S3BucketDestination": {
        "Format": "Parquet",
        "OutputSchemaVersion": "V_1",
        "AccountId": "123456789012",
        "Arn": "arn:aws:s3:::my-storage-lens-exports",
        "Prefix": "storage-lens/",
        "Encryption": {
          "SSES3": {}
        }
      }
    }
  }'

# Xem danh sách dashboards
aws s3control list-storage-lens-configurations \
  --account-id 123456789012

# Xem chi tiết dashboard
aws s3control get-storage-lens-configuration \
  --account-id 123456789012 \
  --config-id "org-main-dashboard"
```

---

## 6. Phân Tích Cost Optimization Với Storage Lens

### Use Case 1: Tìm Data Chưa Được Tiered

```
Dấu hiệu trong dashboard:
  % of data in Standard class = 85% (quá cao)
  Total Standard storage: 500 TB → $11,500/month

Phân tích:
  1. Xem "Storage Class Distribution" chart
  2. Drill down theo bucket để tìm bucket lớn nhất còn ở Standard
  3. Kiểm tra "Last Modified Date Distribution" — data cũ nào vẫn ở Standard?

Hành động:
  → Thiết kế lifecycle rule cho các bucket lớn đó
  → Tiết kiệm tiềm năng: 30–60% chi phí lưu trữ
```

### Use Case 2: Tìm Incomplete Multipart Uploads

```
Trong dashboard:
  Metric: "Incomplete multipart upload bytes"
  Nếu > 0 và ổn định theo ngày → Có uploads bị bỏ dở

Hành động:
  1. Xem bucket nào có incomplete uploads nhiều nhất
  2. Thêm AbortIncompleteMultipartUpload lifecycle rule vào bucket đó

Script phát hiện:
```

```bash
# Tìm bucket có incomplete multipart uploads
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
while read bucket; do
  count=$(aws s3api list-multipart-uploads --bucket "$bucket" \
    --query 'length(Uploads)' --output text 2>/dev/null)
  if [ "$count" != "None" ] && [ "$count" -gt "0" ]; then
    echo "$bucket: $count incomplete uploads"
  fi
done
```

### Use Case 3: Tìm Bucket Không Hoạt Động (Advanced)

```
Metric cần xem (Advanced tier):
  "GET requests per day" = 0 trong 30+ ngày liên tiếp
  "PUT requests per day" = 0 trong 30+ ngày

Kết quả: Bucket zombie — tốn tiền lưu trữ, không ai dùng

Quy trình xử lý:
  1. Confirm với owner của bucket (xem tag "owner")
  2. Check có lifecycle rule để xóa không?
  3. Nếu không cần → Archive bucket (chuyển tất cả sang Glacier Deep Archive)
  4. Sau 6 tháng xác nhận không cần → Xóa bucket
```

### Use Case 4: Phát Hiện Object Size Distribution Bất Thường

```
Nếu "Average object size" = 2KB trong bucket 10 TB:
  → 5 tỷ object × 2KB = 10TB
  → 5 tỷ request monitoring nếu dùng Intelligent-Tiering = $12,500/month!
  → Object nhỏ không phù hợp Intelligent-Tiering hay Standard-IA

Giải pháp:
  1. Xem xét aggregate nhỏ objects (nếu logic cho phép)
  2. Giữ ở Standard, dùng lifecycle expiration thay tiering
  3. Không bật Intelligent-Tiering cho bucket này
```

---

## 7. Tích Hợp Với S3 Inventory và CloudWatch

### Storage Lens + CloudWatch Alarms (Advanced)

```bash
# Storage Lens Advanced publish metrics sang CloudWatch
# Sau đó tạo alarm dựa trên Storage Lens metrics

# Ví dụ: Alarm khi incomplete multipart upload tăng đột biến
aws cloudwatch put-metric-alarm \
  --alarm-name "S3-Incomplete-Multipart-Upload-High" \
  --alarm-description "Incomplete multipart upload bytes quá cao" \
  --namespace "AWS/S3/Storage-Lens" \
  --metric-name "IncompleteMultipartUploadStorageBytes" \
  --dimensions Name=storage_lens_id,Value=org-main-dashboard \
  --statistic Average \
  --period 86400 \
  --threshold 10737418240 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:storage-alerts
```

### Storage Lens Data Export + Athena

```
Storage Lens xuất data sang S3 (daily) ở format CSV hoặc Parquet.
Có thể dùng Athena để query chi tiết hơn.

Athena query ví dụ:
```

```sql
-- Tìm bucket có noncurrent version > 50GB
SELECT 
    bucket_name,
    aws_account_id,
    storage_class,
    noncurrent_version_storage_bytes / 1073741824 AS noncurrent_gb
FROM storage_lens_export
WHERE 
    dt = '2026-05-15'
    AND noncurrent_version_storage_bytes > 53687091200  -- 50GB
ORDER BY noncurrent_version_storage_bytes DESC
LIMIT 20;

-- Tìm bucket không có GET request trong 30 ngày (Advanced metrics)
SELECT 
    bucket_name,
    SUM(get_requests) AS total_get_requests,
    total_storage_bytes / 1073741824 AS storage_gb
FROM storage_lens_export
WHERE 
    dt BETWEEN '2026-04-15' AND '2026-05-15'
GROUP BY bucket_name, total_storage_bytes
HAVING SUM(get_requests) = 0
ORDER BY storage_gb DESC;
```

### S3 Inventory + Storage Lens

```
S3 Inventory — Kiểm Kê S3: Liệt kê tất cả objects với metadata
Storage Lens: Tổng hợp metrics ở bucket/account level

Kết hợp:
  1. Storage Lens → Phát hiện bucket nào có vấn đề (macro view)
  2. S3 Inventory → Drill down object level trong bucket đó (micro view)

Ví dụ workflow:
  Storage Lens: "Bucket A có 70% noncurrent versions"
  → S3 Inventory Bucket A → Tìm objects có nhiều versions nhất
  → S3 Batch Operations → Tag hoặc xóa versions không cần
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: S3 Storage Lens khác gì CloudWatch S3 metrics?**

> Storage Lens cung cấp aggregated view toàn tổ chức — một dashboard hiển thị metrics tổng hợp từ tất cả buckets, accounts và regions. CloudWatch S3 metrics hoạt động ở bucket level, cần cấu hình riêng từng bucket và phù hợp cho alerting real-time. Storage Lens tập trung vào cost optimization và usage analytics với recommendations, còn CloudWatch phù hợp hơn cho operational monitoring và alarms. Storage Lens có độ trễ ~48 giờ; CloudWatch có thể 1 phút.

**Q: Storage Lens Advanced metrics thêm gì so với Free?**

> Advanced metrics bổ sung: request metrics (GET/PUT/LIST/DELETE counts), data transfer insights, prefix-level aggregation để phân tích chi tiết hơn theo "thư mục", 15 tháng lịch sử thay vì 14 ngày, CloudWatch metrics publishing để tạo alarms, và activity metrics để phát hiện bucket không hoạt động. Chi phí $0.20/million objects/month — với 10 triệu objects chỉ $2/month, rất đáng với tổ chức lớn.

**Q: Dùng Storage Lens như thế nào để tối ưu chi phí S3?**

> Quy trình bốn bước: (1) Xem "Storage Class Distribution" — nếu phần lớn ở Standard, nhiều cơ hội tiết kiệm; (2) Xem "Incomplete Multipart Upload Bytes" — nếu > 0 và không giảm, cần AbortIncompleteMultipartUpload lifecycle; (3) Xem "Noncurrent Version Bytes" — nếu cao, cần lifecycle rule cho versioned objects; (4) Với Advanced, tìm bucket có zero request trong 30+ ngày — là candidates để archive hoặc xóa. Kết hợp với S3 Inventory để drill down chi tiết sau khi xác định bucket vấn đề.

**Q: Tại sao Storage Lens cần cấu hình "scope" khi tạo dashboard?**

> Scope xác định Storage Lens sẽ thu thập metrics từ accounts, regions và buckets nào. Với AWS Organizations, có thể include toàn bộ organization hoặc chỉ một số accounts nhất định. Scope quan trọng để: (1) Đảm bảo không bỏ sót account nào trong phân tích; (2) Kiểm soát chi phí Advanced metrics (chỉ count objects trong scope); (3) Phân quyền — chỉ account management/delegated admin mới có thể tạo dashboard org-level; account thành viên chỉ tạo được account-level dashboard.

---

**Phiên Bản:** 1.0
**Cập Nhật:** 2026-05-16
