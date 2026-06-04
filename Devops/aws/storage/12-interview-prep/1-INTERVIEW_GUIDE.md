# 📝 Top 20 Câu Hỏi Phỏng Vấn AWS Storage — Hướng Dẫn Đầy Đủ

> Các câu hỏi được chọn lọc từ thực tế phỏng vấn tại các công ty sử dụng AWS. Mỗi câu có câu trả lời mẫu, điểm cốt lõi cần nhớ, và lỗi thường gặp.

---

## 📊 Phân Loại Câu Hỏi

| Danh Mục | Số Câu | Tần Suất Xuất Hiện |
|----------|--------|-------------------|
| S3 & Object Storage | 7 | Rất cao — hỏi trong hầu hết phỏng vấn |
| EBS & Block Storage | 4 | Cao — đặc biệt với backend/DevOps |
| EFS & File Storage | 2 | Trung bình |
| Bảo Mật | 3 | Rất cao — câu hỏi về security luôn có |
| Kiến Trúc & Tối Ưu | 4 | Cao — thường ở vòng senior/architect |

---

## 🟢 Phần 1: S3 & Object Storage (7 Câu)

---

### Câu 1: Phân biệt S3 Standard, S3 Standard-IA, S3 Glacier và khi nào dùng từng loại?

**Câu trả lời mẫu:**

S3 có 6 storage class — lớp lưu trữ chính, chọn dựa trên tần suất truy cập và yêu cầu truy xuất:

| Storage Class | Chi Phí Lưu Trữ | Chi Phí Truy Xuất | Thời Gian Truy Xuất | Dùng Khi |
|---------------|-----------------|-------------------|---------------------|----------|
| **Standard** | Cao nhất (~$0.023/GB) | Miễn phí | Tức thì (ms) | Dữ liệu truy cập thường xuyên (>1 lần/tháng) |
| **Standard-IA** (Infrequent Access — Truy cập Không Thường Xuyên) | Thấp hơn 50% | Có phí per-GB | Tức thì (ms) | Backup, log cũ >30 ngày, DR data |
| **One Zone-IA** | Thấp hơn Standard-IA 20% | Có phí | Tức thì | Dữ liệu không quan trọng, có thể tái tạo |
| **Glacier Instant Retrieval** | Rẻ hơn 68% so với Standard | Cao hơn | Tức thì (ms) | Archive truy cập vài lần/quý |
| **Glacier Flexible Retrieval** | Rẻ hơn 68% | Cao | 1 phút đến 12 giờ | Compliance archive, DR không cần ngay |
| **Glacier Deep Archive** | Rẻ nhất (~$0.00099/GB) | Cao nhất | 12–48 giờ | Lưu trữ 7–10 năm, regulatory compliance |
| **Intelligent-Tiering** | Overhead $0.0025/1000 objects | Không phí truy cập | Tức thì (tier cao) | Dữ liệu không rõ pattern truy cập |

**Ví dụ thực tế:** Hệ thống e-commerce:
- Ảnh sản phẩm → Standard (truy cập thường xuyên)
- Log giao dịch >90 ngày → Standard-IA
- Backup database hàng năm → Glacier Deep Archive

**Điểm cốt lõi cần nhớ:**
- Standard-IA có minimum storage duration 30 ngày — tính phí ít nhất 30 ngày dù xóa sớm
- Glacier có minimum 90 ngày (Flexible) hoặc 180 ngày (Deep Archive)
- Intelligent-Tiering tốt nhất khi không dự đoán được pattern, nhưng có monitoring fee

**Lỗi thường gặp:** Quên tính retrieval cost — Glacier rẻ để lưu nhưng tốn khi đọc ra. Nếu đọc nhiều, Standard-IA hoặc Intelligent-Tiering có thể rẻ hơn tổng thể.

---

### Câu 2: S3 Presigned URL là gì? Cách hoạt động và use case?

**Câu trả lời mẫu:**

Presigned URL — URL có chữ ký tạm thời — là URL cho phép người dùng truy cập tạm thời vào một S3 object cụ thể mà không cần AWS credentials, trong một khoảng thời gian giới hạn.

**Cơ chế hoạt động:**

```
1. Backend (có IAM credentials) tạo presigned URL cho object
2. URL chứa: bucket name, object key, expiration time, chữ ký HMAC
3. Backend trả URL cho client (browser, mobile app)
4. Client dùng URL để GET/PUT object trực tiếp với S3
5. S3 xác thực chữ ký và expiration — nếu hợp lệ → cho phép
```

**Hai loại presigned URL:**

- **GET presigned URL:** Cho phép download object (tạo từ GET object permission)
- **PUT presigned URL:** Cho phép upload trực tiếp lên S3 không qua backend

**Use cases điển hình:**

1. **Download file an toàn:** Tài liệu hợp đồng trong CRM — chỉ user có quyền mới nhận URL
2. **Upload trực tiếp (Direct Upload):** User upload avatar lên S3, không đi qua server → giảm băng thông backend
3. **Chia sẻ file tạm thời:** Gửi link download báo cáo có hiệu lực 1 giờ
4. **CDN presigned:** CloudFront signed URL cho nội dung premium (video, nhạc)

**Code ví dụ (Python, boto3):**

```python
import boto3
s3 = boto3.client('s3')

# Tạo GET presigned URL — hết hạn sau 1 giờ
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'report.pdf'},
    ExpiresIn=3600  # giây
)

# Tạo PUT presigned URL — cho phép client upload trực tiếp
put_url = s3.generate_presigned_url(
    'put_object',
    Params={'Bucket': 'my-bucket', 'Key': 'uploads/user-123/avatar.jpg'},
    ExpiresIn=900  # 15 phút
)
```

**Điểm cốt lõi cần nhớ:**
- URL có hiệu lực dựa trên quyền của IAM entity tạo ra nó — nếu role bị thu hồi quyền, URL còn hạn vẫn thất bại
- Presigned URL không thể vô hiệu hóa trực tiếp — phải đợi hết hạn hoặc xóa object
- Max expiration: 7 ngày với IAM user, 1 giờ với temporary credentials (STS)

**Lỗi thường gặp:** Dùng presigned URL cho nội dung cần thu hồi ngay — không phù hợp. Trong trường hợp đó, dùng CloudFront signed cookies và invalidation.

---

### Câu 3: S3 Multipart Upload là gì? Khi nào bắt buộc phải dùng?

**Câu trả lời mẫu:**

Multipart Upload — Tải lên nhiều phần — là tính năng cho phép chia file lớn thành nhiều phần nhỏ và tải lên song song, sau đó S3 ghép lại thành object hoàn chỉnh.

**Quy trình 3 bước:**

```
1. Initiate (Khởi tạo): Gửi yêu cầu → nhận UploadId
2. Upload Parts (Tải từng phần): Tải song song các phần, nhận ETag cho mỗi phần
3. Complete (Hoàn thành): Gửi danh sách ETag → S3 ghép thành object
(hoặc Abort nếu muốn hủy và xóa các phần đã tải)
```

**Giới hạn và quy tắc:**

- Bắt buộc với file > 5 GB (S3 không cho phép PutObject với file >5GB)
- Khuyến nghị với file > 100 MB để tận dụng song song
- Mỗi phần phải từ 5 MB (trừ phần cuối), tối đa 10.000 phần
- Max object size: 5 TB

**Lợi ích:**

1. **Tốc độ:** Upload song song (parallel) tận dụng toàn bộ băng thông
2. **Độ tin cậy:** Nếu mạng đứt, chỉ cần upload lại phần lỗi, không cần upload lại toàn bộ
3. **Linh hoạt:** Bắt đầu upload khi chưa biết kích thước file cuối (streaming)

**Vấn đề phổ biến — Incomplete Multipart Uploads:**

Các phần tải lên nhưng không Complete → tồn tại mãi → tốn phí. Giải pháp: tạo S3 Lifecycle rule để tự động xóa incomplete multipart uploads sau N ngày.

```json
{
  "Rules": [{
    "Status": "Enabled",
    "AbortIncompleteMultipartUpload": {
      "DaysAfterInitiation": 7
    }
  }]
}
```

**Điểm cốt lõi cần nhớ:**
- Single PutObject tối đa 5 GB — bắt buộc dùng multipart cho file lớn hơn
- Các phần được lưu riêng và tính phí — phải Complete hoặc Abort để tránh chi phí rác

---

### Câu 4: Giải thích S3 Versioning và MFA Delete. Khi nào bật, khi nào không cần?

**Câu trả lời mẫu:**

**S3 Versioning — Quản lý phiên bản** cho phép lưu nhiều phiên bản của cùng một object. Mỗi lần PUT/DELETE đều tạo version ID mới.

**Cách hoạt động:**

```
Không có versioning:    PUT file.txt → ghi đè
Có versioning:          PUT file.txt → tạo version mới, giữ version cũ
DELETE file.txt         → tạo "delete marker", không xóa thực sự
DELETE file.txt + versionId → xóa vĩnh viễn phiên bản đó
```

**Ba trạng thái của bucket:**
- **Unversioned (mặc định):** Không có version ID
- **Versioning-enabled:** Bật versioning, tất cả object có version
- **Versioning-suspended:** Tạm dừng — object mới không có version, object cũ vẫn giữ

**MFA Delete — Xóa Có Xác Thực Hai Yếu Tố:** Yêu cầu MFA token khi:
- Thay đổi trạng thái versioning của bucket
- Xóa vĩnh viễn một phiên bản object

MFA Delete ngăn kẻ tấn công (hoặc nhân viên sai phép) xóa data dù đã có AWS credentials.

**Khi nào bật Versioning:**
- Dữ liệu quan trọng cần khôi phục nếu bị ghi đè nhầm
- Audit log cần giữ lịch sử thay đổi
- Tuân thủ compliance yêu cầu giữ data

**Khi nào không cần:**
- Bucket chứa dữ liệu có thể tái tạo (thumbnail, temp files)
- Chi phí storage tăng gấp đôi (mỗi update tạo thêm phiên bản) không xứng đáng
- Log bucket có rotation tự động — dùng lifecycle thay versioning

**Điểm cốt lõi:** Versioning + Lifecycle rule (xóa old version sau N ngày) = balance giữa bảo vệ data và kiểm soát chi phí.

---

### Câu 5: S3 Cross-Region Replication (CRR) vs Same-Region Replication (SRR) — khác nhau như thế nào?

**Câu trả lời mẫu:**

Cả hai đều sao chép object tự động từ bucket nguồn sang bucket đích. Sự khác biệt chủ yếu ở mục đích sử dụng:

| Tiêu Chí | CRR — Cross-Region Replication (Sao Chép Liên Vùng) | SRR — Same-Region Replication (Sao Chép Cùng Vùng) |
|----------|------------------------------------------------------|------------------------------------------------------|
| Vùng đích | Region khác | Cùng region |
| Chi phí | DATA transfer fee liên vùng (tốn kém hơn) | Chỉ phí replication, không phí transfer |
| Độ trễ | Vài phút (cross-region) | Nhanh hơn (cùng region) |
| Use case chính | DR — Disaster Recovery, giảm latency cho user nước ngoài | Compliance (log aggregation), test/prod separation |

**Use cases thực tế:**

CRR:
- Ứng dụng global — dữ liệu ở US East, replicate sang EU West để giảm latency cho user EU
- DR strategy — nếu `us-east-1` sập, dùng bucket ở `us-west-2`
- Data sovereignty — replicate sang region ở nước user yêu cầu

SRR:
- Tổng hợp log từ nhiều bucket production vào một bucket log tập trung
- Sync giữa bucket production và bucket test/dev
- Tuân thủ nội bộ — giữ bản sao dữ liệu nhạy cảm trong cùng region

**Điều kiện bắt buộc để dùng:**
1. Versioning phải được bật ở cả bucket nguồn và đích
2. IAM role phải có quyền đọc từ nguồn và ghi vào đích
3. Replication chỉ áp dụng cho object mới — object cũ trước khi bật không được replicate (dùng S3 Batch Replication cho object cũ)

**Lỗi thường gặp:** Nghĩ rằng xóa object ở nguồn sẽ xóa ở đích — không phải mặc định. Phải bật "Delete marker replication" riêng.

---

### Câu 6: S3 Object Lock là gì? Governance mode vs Compliance mode khác nhau thế nào?

**Câu trả lời mẫu:**

S3 Object Lock — Khóa Đối Tượng — implement mô hình WORM (Write Once Read Many — Ghi Một Lần Đọc Nhiều Lần): object bị khóa không thể xóa hay ghi đè trong thời gian quy định.

**Hai chế độ retention — thời gian giữ:**

| Chế Độ | Ai Có Thể Gỡ Lock? | Dùng Khi |
|---------|-------------------|----------|
| **Governance Mode** | Admin có quyền `s3:BypassGovernanceRetention` | Môi trường test, data không quan trọng tuyệt đối |
| **Compliance Mode** | Không ai — kể cả AWS root account | Regulatory compliance: SEC 17a-4, HIPAA, FINRA |

**Legal Hold — Giữ Pháp Lý:**

Ngoài retention period, có thể đặt Legal Hold trên object: không có ngày hết hạn, chỉ user có quyền `s3:PutObjectLegalHold` mới gỡ được. Dùng khi đang trong quá trình điều tra pháp lý.

**Ví dụ thực tế:**

```
Ngân hàng cần lưu transaction log 7 năm theo quy định MAS/SEC:
- Bật Object Lock với Compliance Mode
- Retention period = 7 năm
- Kể cả CTO cũng không xóa được — đảm bảo audit trail
```

**Điểm cốt lõi:**
- Object Lock phải bật khi tạo bucket — không bật sau được
- Áp dụng per-object hoặc default cho toàn bucket
- Kết hợp với Versioning (bắt buộc bật)

---

### Câu 7: S3 Event Notifications là gì? Kể 3 use case thực tế?

**Câu trả lời mẫu:**

S3 Event Notifications — Thông báo sự kiện S3 — cho phép S3 tự động gửi thông báo khi có sự kiện xảy ra trong bucket (PUT, DELETE, replication, Glacier restore...) đến các dịch vụ xử lý.

**Các sự kiện hỗ trợ:**
- `s3:ObjectCreated:*` — mọi loại tạo object (PUT, POST, COPY, multipart complete)
- `s3:ObjectRemoved:*` — xóa object
- `s3:ObjectRestore:*` — khôi phục từ Glacier
- `s3:Replication:*` — sự kiện replication

**Đích nhận thông báo:**

| Đích | Dùng Khi |
|------|----------|
| **Lambda** | Xử lý ngay lập tức, không cần queue | 
| **SQS** — Simple Queue Service (Dịch vụ Hàng Đợi Đơn Giản) | Cần fan-out, buffer, retry |
| **SNS** — Simple Notification Service (Dịch vụ Thông Báo Đơn Giản) | Gửi đến nhiều subscriber cùng lúc |
| **EventBridge** | Cần filter phức tạp, route đến nhiều đích |

**3 Use cases thực tế:**

1. **Xử lý ảnh tự động (Image Processing):**
   ```
   User upload avatar → S3 PUT → S3 Event → Lambda trigger →
   Lambda resize ảnh → lưu thumbnail vào bucket khác
   ```

2. **ETL Pipeline — Extract Transform Load (Trích Xuất Chuyển Đổi Nạp):**
   ```
   CSV file mới vào S3 → S3 Event → SQS → Worker EC2 nhận queue →
   Parse CSV → insert vào Redshift
   ```

3. **Thông báo real-time cho team:**
   ```
   File report quan trọng được tạo → S3 Event → SNS →
   Slack notification + Email cho team
   ```

**Điểm cốt lõi:**
- Có thể filter theo prefix và suffix của object key (ví dụ: chỉ trigger với file `.jpg` trong thư mục `uploads/`)
- EventBridge cho phép replay events — hữu ích khi consumer lỗi

---

## 🟡 Phần 2: EBS & Block Storage (4 Câu)

---

### Câu 8: Khi nào dùng gp3 vs io2? Và st1 dùng cho workload gì?

**Câu trả lời mẫu:**

EBS có 5 loại volume chính — dựa trên loại phần cứng (SSD hoặc HDD) và use case:

**SSD-based (Độ trễ thấp, IOPS quan trọng):**

| Volume | Max IOPS | Max Throughput | Chi Phí | Dùng Khi |
|--------|----------|----------------|---------|----------|
| **gp3** (General Purpose SSD — SSD Đa Dụng v3) | 16.000 | 1.000 MB/s | $0.08/GB | Hầu hết workload: boot, web server, small DB |
| **gp2** | 16.000 | 250 MB/s | $0.10/GB | Legacy — nên migrate sang gp3 |
| **io2** (Provisioned IOPS SSD — SSD IOPS Được Cấp Phát) | 64.000 | 1.000 MB/s | $0.125/GB + IOPS | Mission-critical DB: Oracle, MySQL production |
| **io2 Block Express** | 256.000 | 4.000 MB/s | Cao nhất | SAP HANA, NoSQL quy mô lớn |

**HDD-based (Throughput quan trọng, chi phí thấp):**

| Volume | Max Throughput | Max IOPS | Dùng Khi |
|--------|----------------|----------|----------|
| **st1** (Throughput Optimized HDD — HDD Tối Ưu Thông Lượng) | 500 MB/s | 500 | Sequential read/write: Kafka logs, data warehouse |
| **sc1** (Cold HDD — HDD Lạnh) | 250 MB/s | 250 | Cold data, ít truy cập, cần lưu trữ rẻ |

**Không dùng HDD cho:**
- Boot volume (phải dùng SSD)
- Database OLTP — Online Transaction Processing (Xử Lý Giao Dịch Trực Tuyến) (latency cao)

**Ví dụ quyết định nhanh:**

```
"Cần volume cho PostgreSQL production, 5.000 IOPS, chi phí reasonable"
→ gp3 với 5.000 IOPS provisioned ($0.08/GB + $0.005/IOPS)

"Cần volume cho Oracle RAC — Real Application Cluster, 50.000 IOPS, failover tức thì"
→ io2 Block Express (hỗ trợ Multi-Attach, 99.999% durability)

"Cần lưu log 1TB, đọc sequential, chi phí tối thiểu"
→ st1 ($0.045/GB)
```

**Điểm cốt lõi:** gp3 tốt hơn gp2 về mọi mặt và rẻ hơn — luôn chọn gp3 cho workload mới.

---

### Câu 9: EBS Multi-Attach là gì? Giới hạn và use case?

**Câu trả lời mẫu:**

EBS Multi-Attach — Gắn kết Nhiều — cho phép một EBS volume được gắn đồng thời vào nhiều EC2 instances trong cùng Availability Zone — Vùng Khả Dụng.

**Giới hạn:**
- Chỉ hỗ trợ với io1 và io2 volume
- Tối đa 16 EC2 instances cùng lúc
- Các instances phải trong cùng AZ (Availability Zone)
- Ứng dụng phải tự quản lý việc đọc/ghi đồng thời (S3 multi-access khác — S3 tự xử lý)
- Không hỗ trợ Windows

**Use case hợp lệ:**

1. **Clustered Databases:** Oracle RAC — Real Application Cluster — nhiều node database truy cập cùng shared storage
2. **High-availability applications:** Nếu một EC2 chết, instance khác trong cluster đã có volume mounted → giảm failover time
3. **Distributed file systems:** GPFS — General Parallel File System, Lustre on-premise

**Quan trọng:** Ứng dụng phải dùng cluster-aware file system (không phải ext4 hay XFS thông thường) để tránh data corruption — hỏng dữ liệu do ghi đồng thời.

**Lỗi thường gặp:** Mount cùng EBS vào 2 EC2 thông thường và expect Linux file system tự xử lý — sẽ gây corruption. Phải có cluster file system layer.

---

### Câu 10: EBS Snapshot hoạt động như thế nào? Incremental Snapshot là gì?

**Câu trả lời mẫu:**

EBS Snapshot — Ảnh chụp nhanh EBS — là bản sao lưu của EBS volume tại một thời điểm, được lưu vào S3 (nhưng bạn không thấy trong S3 bucket thông thường — AWS quản lý).

**Incremental Snapshot — Snapshot Gia Tăng:**

```
Snapshot 1 (toàn bộ): Chứa 100% dữ liệu gốc của volume
Snapshot 2 (incremental): Chỉ chứa các block thay đổi so với Snapshot 1
Snapshot 3 (incremental): Chỉ chứa các block thay đổi so với Snapshot 2
```

Lợi ích: Snapshot 2, 3 rất nhỏ nếu ít thay đổi → tiết kiệm storage và thời gian backup.

**Xóa Snapshot:**

Khi xóa Snapshot 2, AWS tự động chuyển dữ liệu cần thiết vào Snapshot 3 để Snapshot 3 vẫn độc lập. Không bao giờ mất dữ liệu khi xóa snapshot giữa chuỗi.

**Fast Snapshot Restore — Khôi phục Nhanh:** Bật FSR để volume từ snapshot có full performance ngay lập tức, không cần warm-up. Tốn thêm phí, nhưng quan trọng khi RTO ngắn.

**Kịch bản thực tế:**

```
Backup nightly 11PM → snapshot mỗi ngày
30 ngày retention → giữ 30 snapshot
Chỉ ngày đầu là full, 29 ngày sau là incremental → tiết kiệm ~80% storage
```

**Điều cần nhớ:**
- Tạo snapshot khi đang running — consistent nhưng có thể miss write đang bay. Tốt nhất: stop instance hoặc flush disk trước
- Copy snapshot sang region khác → cho phép restore volume ở region khác (DR)
- Snapshot có thể share với AWS account khác

---

### Câu 11: EBS vs Instance Store — khác nhau như thế nào? Khi nào dùng Instance Store?

**Câu trả lời mẫu:**

| Tiêu Chí | EBS | Instance Store — Lưu Trữ Trực Tiếp |
|----------|-----|-------------------------------------|
| **Vị trí vật lý** | Network-attached (qua mạng nội bộ) | Ổ cứng gắn trực tiếp vào host |
| **Tốc độ** | Thấp hơn (có network latency) | Cực nhanh — NVMe local |
| **Persistence (Tính bền vững)** | Bền vững — dữ liệu tồn tại khi stop/start | Tạm thời — mất hoàn toàn khi stop/terminate/crash |
| **Snapshot** | Có — backup và restore | Không có |
| **Chi phí** | Tính phí riêng (theo GB/tháng) | Bao gồm trong giá EC2 instance |
| **Kích thước** | Tùy chọn, có thể tăng | Cố định theo instance type |

**Khi nào Instance Store phù hợp:**

1. **Buffer, cache, scratch space — không quan trọng nếu mất:**
   - Redis cache temporary data
   - Batch job intermediate results
   - Spark shuffle data — dữ liệu trung gian xử lý

2. **High-performance cần latency cực thấp:**
   - i3/i3en instances với NVMe SSD đạt hàng triệu IOPS
   - Phù hợp cho Elasticsearch, Cassandra với replication tự quản lý

3. **Dữ liệu có thể tái tạo:**
   - Media transcoding temp files
   - ML training dataset đã có ở S3, copy vào local để train nhanh

**Khi nào không dùng Instance Store:**
- Database production data (Oracle, MySQL, PostgreSQL)
- Bất kỳ dữ liệu nào cần survive instance stop

---

## 🟠 Phần 3: EFS & File Storage (2 Câu)

---

### Câu 12: EFS vs EBS vs S3 — khi nào dùng cái nào?

**Câu trả lời mẫu:**

Đây là câu hỏi framework — cần trả lời dựa trên 3 tiêu chí: access pattern, protocol, và durability/availability yêu cầu.

| Tiêu Chí | S3 | EBS | EFS |
|----------|-----|-----|-----|
| **Loại** | Object Storage | Block Storage | File Storage |
| **Access** | HTTP/HTTPS API | Block device (mount 1 instance) | NFS — Network File System (mount nhiều instance) |
| **Đồng thời** | Vô hạn concurrent clients | 1 instance (trừ io2 Multi-Attach) | Hàng nghìn instances đồng thời |
| **Latency** | High (HTTP roundtrip) | Lowest (block-level) | Middle |
| **Scalability** | Unlimited | Cần resize thủ công | Tự co giãn |
| **Chi phí** | ~$0.023/GB | ~$0.08/GB (gp3) | ~$0.30/GB |
| **Use case** | Backup, assets, data lake | Boot volume, DB | Shared content, CMS, ML training |

**Quy tắc chọn nhanh:**

```
Cần chia sẻ giữa nhiều EC2/container?
  → Có → EFS (hoặc FSx tùy workload)
  → Không → Tiếp tục...

Cần block-level access (database, boot volume)?
  → Có → EBS
  → Không → Tiếp tục...

Lưu trữ file/object, truy cập qua API?
  → S3
```

**Ví dụ thực tế:**

- WordPress site với nhiều EC2 → EFS cho shared `wp-content/uploads/`
- MySQL database → EBS io2
- User avatar, media files → S3
- ML model training — nhiều GPU instances đọc cùng dataset → EFS hoặc FSx Lustre

---

### Câu 13: EFS Performance Modes và Throughput Modes — khi nào dùng cái nào?

**Câu trả lời mẫu:**

EFS có hai chiều cấu hình độc lập: Performance Mode — Chế độ Hiệu suất và Throughput Mode — Chế độ Thông lượng.

**Performance Mode (Đặt khi tạo — không đổi được):**

| Mode | Latency | Khi Dùng |
|------|---------|----------|
| **General Purpose (Đa Dụng)** | Thấp nhất (<1ms) | Mặc định — phù hợp 99% use case |
| **Max I/O** | Cao hơn (~10ms) | Hàng nghìn clients đồng thời, HPC, big data |

Thực tế: Max I/O ít được dùng vì EFS đã tự scale. Chỉ dùng nếu benchmark cho thấy General Purpose không đủ.

**Throughput Mode (Có thể đổi sau):**

| Mode | Cách Tính Throughput | Khi Dùng |
|------|---------------------|----------|
| **Bursting (Bùng Nổ)** | Tỷ lệ thuận với storage size + burst credits | Storage >1TB, access không đồng đều |
| **Provisioned (Được Cấp Phát)** | Đặt throughput cố định | Biết trước throughput cần, ít data nhưng cần throughput cao |
| **Elastic (Linh Hoạt)** | Tự động scale theo demand | Không dự đoán được workload (mới nhất, khuyến nghị) |

**Quy tắc chọn:**
- Mới triển khai → dùng Elastic Throughput (tự scale, không cần dự đoán)
- Biết chính xác throughput cần → Provisioned
- Storage >1TB, workload biến thiên → Bursting

---

## 🔴 Phần 4: Bảo Mật (3 Câu)

---

### Câu 14: Giải thích các cách mã hóa dữ liệu trong S3 (at-rest). Khi nào dùng SSE-KMS?

**Câu trả lời mẫu:**

S3 hỗ trợ 4 phương thức mã hóa khi lưu (at-rest — khi dữ liệu nằm yên trên ổ đĩa):

| Phương Thức | Ai Quản Lý Key | Overhead | Dùng Khi |
|-------------|----------------|----------|----------|
| **SSE-S3** (Server-Side Encryption với S3-managed key) | AWS quản lý hoàn toàn | Thấp nhất | Mặc định — không cần audit key |
| **SSE-KMS** (với AWS KMS — Key Management Service) | AWS KMS, bạn kiểm soát policy | Trung bình | Cần audit trail ai đọc/ghi, compliance |
| **SSE-C** (Customer-provided key — Key do khách hàng cung cấp) | Bạn cung cấp key trong header | Cao — bạn quản lý key | Không muốn AWS biết key |
| **CSE** (Client-Side Encryption — Mã hóa Phía Client) | Bạn mã hóa trước khi gửi | Cao nhất | Không tin tưởng AWS infrastructure |

**Khi nào dùng SSE-KMS:**

1. **Compliance yêu cầu audit:** Mỗi lần S3 object được đọc/ghi, KMS log vào CloudTrail — ai đọc, khi nào, từ đâu
2. **Key rotation tự động:** KMS tự rotate key hàng năm
3. **Cross-account access:** Kiểm soát chặt chẽ ai (account nào) được decrypt
4. **Envelope encryption — Mã hóa bao bì:** KMS bảo vệ data encryption key — không trực tiếp encrypt data

**Điểm cốt lõi:**
- SSE-S3 là mặc định từ 2023 — tất cả object mới đều được mã hóa dù không cấu hình
- SSE-KMS tốn KMS API call fee ($0.03/10.000 requests) — với bucket traffic cao, chi phí đáng kể
- SSE-C: AWS encrypt/decrypt nhưng không lưu key → nếu mất key → mất dữ liệu vĩnh viễn

---

### Câu 15: Sự khác nhau giữa S3 Bucket Policy và IAM Policy? Khi nào dùng cái nào?

**Câu trả lời mẫu:**

Cả hai đều kiểm soát quyền truy cập S3, nhưng về bản chất khác nhau:

| Tiêu Chí | IAM Policy | S3 Bucket Policy |
|----------|------------|-----------------|
| **Gắn vào** | IAM user/role/group | S3 bucket |
| **Phạm vi** | Một account — kiểm soát identity | Cross-account — ai từ đâu đến |
| **Format** | JSON IAM statement | JSON Resource policy |
| **Anonymous access** | Không thể (cần identity) | Có thể (Principal: "*") |
| **Điều kiện** | Nhiều condition keys | Nhiều condition keys bao gồm IP, VPC |

**Quy tắc S3 evaluation:**

S3 kết hợp cả hai:
```
ALLOW nếu: (IAM policy ALLOW) AND (Bucket policy ALLOW hoặc không có bucket policy)
DENY nếu: IAM policy DENY hoặc Bucket policy DENY (explicit deny thắng)
```

**Khi nào dùng Bucket Policy:**

1. **Cross-account access — Truy cập liên account:** IAM policy trong account A không kiểm soát được bucket trong account B → cần bucket policy
2. **Giới hạn truy cập theo IP/VPC:**
   ```json
   "Condition": {
     "StringEquals": {"aws:sourceVpc": "vpc-1234"}
   }
   ```
3. **Enforce HTTPS — Bắt buộc dùng HTTPS:**
   ```json
   "Condition": {"Bool": {"aws:SecureTransport": "false"}}
   "Effect": "Deny"
   ```
4. **Public access cho static website:** Cho phép `"Principal": "*"` đọc object

**Khi nào dùng IAM Policy:**
- Quản lý quyền cho team nội bộ (developer, admin)
- Service role (EC2, Lambda) cần truy cập S3

---

### Câu 16: VPC Endpoint cho S3 là gì? Gateway Endpoint vs Interface Endpoint khác nhau thế nào?

**Câu trả lời mẫu:**

VPC Endpoint — Điểm Cuối VPC — cho phép EC2/Lambda trong VPC truy cập S3 mà không đi qua Internet, tăng bảo mật và giảm cost data transfer.

**Hai loại:**

| | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|-|------------------|----------------------------------|
| **Cơ chế** | Thêm route vào Route Table | Tạo ENI — Elastic Network Interface trong subnet |
| **DNS** | Dùng DNS public của S3 | Tạo private DNS endpoint |
| **Chi phí** | Miễn phí | $0.01/AZ/giờ + data fee |
| **Hỗ trợ** | S3, DynamoDB | Nhiều dịch vụ hơn |
| **Cross-region** | Không | Không (trong region đó) |
| **On-premises access** | Không | Có — qua Direct Connect hoặc VPN |

**Khi nào dùng cái nào:**

- **Gateway Endpoint:** Mặc định cho S3 — miễn phí, đủ cho EC2 trong VPC
- **Interface Endpoint:** Cần truy cập S3 từ on-premises qua Direct Connect, hoặc muốn DNS private

**Lợi ích chính của Gateway Endpoint:**
- Traffic không ra Internet → không cần Internet Gateway
- Có thể dùng bucket policy restrict chỉ cho phép access từ VPC endpoint:

```json
"Condition": {
  "StringEquals": {
    "aws:sourceVpce": "vpce-1234abcd"
  }
}
```

---

## 🟣 Phần 5: Kiến Trúc & Tối Ưu (4 Câu)

---

### Câu 17: Làm thế nào thiết kế một hệ thống backup với RPO = 1 giờ, RTO = 30 phút?

**Câu trả lời mẫu:**

- **RPO — Recovery Point Objective — Mục Tiêu Điểm Khôi Phục:** Mất tối đa bao nhiêu dữ liệu? (1 giờ = backup mỗi giờ)
- **RTO — Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục:** Khôi phục trong bao lâu? (30 phút = restore xong trong 30 phút)

**Thiết kế cho database PostgreSQL:**

```
Thành phần:

1. EBS gp3 volume cho database data
   → EBS snapshot mỗi 1 giờ (đảm bảo RPO 1h)
   → Dùng AWS DLM — Data Lifecycle Manager tự động hóa

2. PostgreSQL WAL — Write-Ahead Log (Nhật ký Ghi Trước) streaming
   → Ghi liên tục vào S3 (near-realtime)
   → Cho phép point-in-time recovery với RPO thực tế <1 phút

3. Fast Snapshot Restore bật cho snapshot
   → Restore volume ngay lập tức (không cần warm-up) → đạt RTO 30 phút

4. Kiểm tra định kỳ:
   → Test restore hàng tuần trong môi trường staging
   → Đo thực tế: snapshot 1h → restore bao lâu?
```

**Điểm quan trọng:** RTO 30 phút là ambitious với snapshot-based restore. Để đảm bảo, cần:
- FSR (Fast Snapshot Restore) bật sẵn
- Automation script restore (không làm tay)
- DB snapshot + application deploy song song

---

### Câu 18: Thiết kế data lake trên S3 cho công ty có 100TB dữ liệu log hàng tháng?

**Câu trả lời mẫu:**

**Architecture 3-Layer (3 tầng) tiêu chuẩn:**

```
Raw Zone (Tầng Thô)        → Silver Zone (Tầng Xử Lý)  → Gold Zone (Tầng Phân Tích)
s3://company-raw/          → s3://company-silver/        → s3://company-gold/
- Giữ nguyên log gốc       - Cleaned, parsed             - Aggregated, business metrics
- Storage class: IA/Glacier - Storage class: Standard/IA  - Storage class: Standard
- Retention: 7 năm          - Retention: 2 năm            - Retention: 1 năm
```

**Tổ chức theo partition — phân vùng (Hive-style):**

```
s3://company-raw/
  year=2026/
    month=05/
      day=16/
        hour=14/
          logs-2026-05-16-14-00.gz
```

Partition theo thời gian giúp Athena — công cụ query — chỉ scan dữ liệu cần thiết → giảm chi phí query.

**Lifecycle Rules — Quy Tắc Vòng Đời cho 100TB/tháng:**

```
Raw Zone:
  0–30 ngày    → Standard    (query thường xuyên để debug)
  31–90 ngày   → Standard-IA (query thỉnh thoảng)
  91–365 ngày  → Glacier Instant Retrieval
  366+ ngày    → Glacier Deep Archive
```

**Tính toán chi phí ước tính:**

```
Tháng 1-3: 300TB × $0.023 = $6.900/tháng (Standard)
Tháng 4-12: chuyển sang IA → $0.0125/GB → giảm ~45%
Năm 2+: Glacier → $0.004/GB → giảm thêm 68%
```

**Thêm:**
- S3 Intelligent-Tiering cho Silver Zone — không dự đoán được access
- Athena + Glue Catalog — Data Catalog (Danh mục Dữ liệu) để query SQL trực tiếp
- Lake Formation — kiểm soát quyền truy cập per-table/per-column
- S3 Storage Lens — theo dõi toàn bộ data lake

---

### Câu 19: Làm thế nào migrate 500TB từ on-premises lên AWS? Trade-offs của các phương án?

**Câu trả lời mẫu:**

**Phân tích trước khi quyết định:**

```
500TB, đường truyền hiện tại: 100Mbps
Thời gian transfer qua network: 500TB / 100Mbps = 500.000GB × 8 / 0.1Gbps
= 40.000.000 giây ≈ 463 ngày → QUÁ LÂU
```

**Các phương án và trade-offs:**

| Phương Án | Thời Gian | Chi Phí | Khi Dùng |
|-----------|-----------|---------|----------|
| **AWS Snowball Edge** (80TB/device) | 1–2 tuần (ship) | ~$300/device + storage | 500TB → 7 devices |
| **AWS Snowmobile** (100PB/xe) | 2–4 tuần | Cao — thuê xe chuyên dụng | Petabyte scale |
| **DataSync qua Direct Connect** | Tháng | Phí Direct Connect | Nếu đã có Direct Connect, < 100TB |
| **S3 Transfer Acceleration** | Tháng | Thêm phí transfer | Network tốt, < 10TB |

**Cho 500TB → Khuyến nghị Snowball Edge:**

```
1. Order 7 Snowball Edge Storage Optimized (80TB × 7 = 560TB)
2. Ship về datacenter — mất 3–5 ngày
3. Copy data vào devices (parallel, local network) — 3–5 ngày/device
4. Ship trả AWS — 3–5 ngày
5. AWS import vào S3 — 1–2 ngày
Tổng: 2–3 tuần vs 463 ngày qua internet
```

**Không nên quên:**
- Kiểm tra data integrity — checksum trước và sau
- Plan cho delta sync — dữ liệu thay đổi trong thời gian migration → dùng DataSync cho incremental
- Test restore trước khi tắt on-premises

---

### Câu 20: Làm thế nào giảm chi phí S3 30% cho công ty không biết gì về access pattern hiện tại?

**Câu trả lời mẫu:**

**Bước 1: Phân tích hiện trạng (không đoán)**

```
Công cụ:
- S3 Storage Lens: Xem access pattern theo bucket/prefix
- S3 Inventory: Danh sách tất cả object và metadata
- S3 Analytics Storage Class Analysis: Phân tích nên chuyển IA không
```

**Bước 2: Bật S3 Intelligent-Tiering cho bucket không rõ pattern**

```
Tự động chuyển:
→ Frequent Access tier (giống Standard)
→ Infrequent Access tier (sau 30 ngày không truy cập, giảm 40%)
→ Archive Instant (sau 90 ngày, giảm 68%)
→ Archive Access (sau 90 ngày, giảm 71%)
→ Deep Archive (sau 180 ngày, giảm 95%)

Chi phí overhead: $0.0025/1000 objects — nhỏ so với savings
```

**Bước 3: Lifecycle Rules cho data có pattern rõ**

```
Log files: Standard → IA (30 ngày) → Glacier (90 ngày) → Deep Archive (365 ngày)
Backups: IA ngay → Glacier (30 ngày) → Deep Archive (90 ngày)
```

**Bước 4: Dọn rác**

```
- Xóa Incomplete Multipart Uploads (lifecycle rule)
- Xóa old versions không cần thiết (lifecycle + versioning expiration)
- Xóa delete markers cũ
```

**Bước 5: Request và Transfer costs**

```
- Dùng S3 Batch Operations thay vì lặp API → giảm request count
- CloudFront trước S3 cho public content → giảm data transfer out fees
- VPC Endpoint → không mất phí data transfer từ EC2 trong cùng region
```

**Kết quả thực tế:** Thường đạt 30–50% giảm chi phí trong 3 tháng với cách tiếp cận này.

---

## 📝 Câu Hỏi Bonus — Thường Gặp Ở Vòng Cuối

### "Kể một lần bạn xử lý S3 bucket bị public không mong muốn"

Sử dụng STAR format:
- **Situation:** Nhận CloudWatch alarm — S3 bucket production bị public
- **Task:** Xác định scope, fix ngay, prevent recurrence
- **Action:** Enable Block Public Access, review bucket policy, kiểm tra S3 Access Analyzer
- **Result:** Fix trong 15 phút, tạo AWS Config rule prevent tái phát

### "Tại sao S3 eventual consistency lại ổn với hầu hết use case?"

S3 Strong Consistency (từ 2020): S3 đã có strong consistency cho tất cả operations — không còn là eventual consistency nữa. Đây là câu hỏi bẫy — nếu interviewer hỏi về eventual consistency, cần clarify đây là kiến thức cũ.

### "Giải thích S3 durability 99.999999999% (11 nines)"

11 nines = mất 1 object trong 10 tỷ object trong 10.000 năm. AWS achieve bằng cách: tự động replicate mỗi object sang tối thiểu 3 AZ, kiểm tra integrity liên tục và tự heal.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
