# 📅 Kế Hoạch Học 90 Ngày — AWS Storage

> Lộ trình học có cấu trúc từ nền tảng đến sẵn sàng phỏng vấn. Kế hoạch được thiết kế cho người đã có kinh nghiệm cloud cơ bản, học 1-2 giờ/ngày.

---

## 🎯 Mục Tiêu Cuối 90 Ngày

Sau 90 ngày bạn sẽ:
- Tự tin trả lời top 20 câu hỏi phỏng vấn AWS Storage
- Có 3-4 câu chuyện STAR về storage incidents/projects thực tế
- Thiết kế được kiến trúc storage cho hệ thống phức tạp
- Tối ưu được chi phí S3/EBS trong môi trường thực tế

---

## 📊 Tổng Quan 3 Giai Đoạn

| Giai Đoạn | Tuần | Chủ Đề | Mục Tiêu |
|-----------|------|--------|----------|
| **Nền Tảng** | 1–4 | S3, EBS, EFS cơ bản | Hiểu và dùng được |
| **Kỹ Năng Cốt Lõi** | 5–8 | Bảo mật, Lifecycle, DR | Production-ready |
| **Nâng Cao & Phỏng Vấn** | 9–13 | Architecture, Cost, Mock | Interview-ready |

---

## 🟢 Giai Đoạn 1: Nền Tảng (Tuần 1–4)

### Tuần 1: S3 Fundamentals — Kiến Thức Nền Tảng S3

**Mục tiêu tuần:** Tạo và cấu hình S3 bucket, upload/download object, hiểu storage classes

**Ngày 1-2: S3 Concepts (Khái niệm S3)**

Đọc:
- `01-s3-fundamentals/1-bucket-and-object-model.md`
- `01-s3-fundamentals/2-storage-classes.md`

Thực hành (AWS Console):
```bash
# Tạo bucket
aws s3 mb s3://my-learning-bucket-$(date +%s) --region ap-southeast-1

# Upload file
echo "Hello S3" > test.txt
aws s3 cp test.txt s3://my-learning-bucket/

# List objects
aws s3 ls s3://my-learning-bucket/

# Kiểm tra storage class
aws s3api head-object --bucket my-learning-bucket --key test.txt
```

Tự kiểm tra:
- [ ] Giải thích durability 11 nines nghĩa là gì
- [ ] Biết tên 6 storage classes và khi nào dùng
- [ ] Biết sự khác biệt giữa object key, bucket, và ETag

**Ngày 3-4: Versioning và Presigned URLs**

Đọc:
- `01-s3-fundamentals/3-versioning-and-mfa-delete.md`
- `01-s3-fundamentals/4-presigned-urls.md`

Thực hành:
```bash
# Bật versioning
aws s3api put-bucket-versioning \
  --bucket my-learning-bucket \
  --versioning-configuration Status=Enabled

# Upload 3 versions của cùng file
echo "version 1" > file.txt && aws s3 cp file.txt s3://my-learning-bucket/
echo "version 2" > file.txt && aws s3 cp file.txt s3://my-learning-bucket/
echo "version 3" > file.txt && aws s3 cp file.txt s3://my-learning-bucket/

# List versions
aws s3api list-object-versions --bucket my-learning-bucket --prefix file.txt

# Tạo presigned URL (Python)
python3 -c "
import boto3
s3 = boto3.client('s3')
url = s3.generate_presigned_url('get_object',
  Params={'Bucket': 'my-learning-bucket', 'Key': 'file.txt'},
  ExpiresIn=3600)
print(url)
"
```

**Ngày 5-7: Multipart Upload và Ôn Tập**

Đọc: `01-s3-fundamentals/5-multipart-upload.md`

Bài tập cuối tuần:
- Viết tay (không nhìn tài liệu): 6 storage classes, giá ước tính, và use case của từng loại
- Giải thích Versioning cho một người không biết cloud

---

### Tuần 2: EBS — Elastic Block Store — Lưu Trữ Khối Linh Hoạt

**Mục tiêu tuần:** Tạo và attach EBS volume, hiểu các loại volume, snapshot

**Ngày 8-9: Volume Types — Loại Volume**

Đọc: `03-ebs/1-volume-types.md`

Bài tập:
```
Cho mỗi scenario, chọn EBS volume type và giải thích:
1. Boot volume cho web server thông thường
2. Database PostgreSQL production, 8.000 IOPS
3. MySQL test environment, cost tối thiểu
4. Kafka logs processing, sequential read/write nhiều
5. Oracle RAC — Real Application Cluster, 100.000 IOPS
```

Đáp án tự kiểm tra:
```
1. gp3 (đủ performance, rẻ)
2. gp3 với 8.000 IOPS provisioned (hoặc io2 nếu cần durability cao)
3. gp3 (smallest size possible)
4. st1 (Throughput Optimized HDD — HDD Tối ưu Thông lượng, sequential)
5. io2 Block Express + Multi-Attach
```

**Ngày 10-12: Snapshots và Encryption**

Đọc:
- `03-ebs/2-snapshots-and-lifecycle.md`
- `03-ebs/5-encryption-with-kms.md`

Thực hành:
```bash
# Tạo EBS snapshot
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxx \
  --description "Learning snapshot"

# List snapshots
aws ec2 describe-snapshots --owner-ids self

# Copy snapshot sang region khác (cho DR)
aws ec2 copy-snapshot \
  --source-region ap-southeast-1 \
  --source-snapshot-id snap-xxxxxxxxx \
  --region us-west-2 \
  --description "DR copy"
```

**Ngày 13-14: Instance Store và Ôn Tập**

Đọc: `03-ebs/6-ebs-vs-instance-store.md`

Quiz cuối tuần:
- [ ] Khi nào dùng Instance Store vs EBS?
- [ ] io2 vs gp3 — ở mức IOPS nào io2 bắt đầu worth it?
- [ ] EBS snapshot là incremental — nghĩa là gì về cost và storage?

---

### Tuần 3: EFS — Elastic File System — Hệ Thống Tệp Linh Hoạt

**Mục tiêu tuần:** Mount EFS vào EC2, hiểu performance và throughput modes

**Ngày 15-17: EFS Basics và Mount**

Đọc:
- `04-efs/README.md`
- `04-efs/1-mount-targets-and-access-points.md`

Thực hành:
```bash
# Tạo EFS file system
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted

# Mount vào EC2 (sau khi tạo mount target)
sudo yum install -y amazon-efs-utils
sudo mount -t efs fs-xxxxxxxxx:/ /mnt/efs

# Kiểm tra
df -h /mnt/efs
touch /mnt/efs/testfile
```

**Ngày 18-19: Performance và Throughput Modes**

Đọc:
- `04-efs/2-performance-modes.md`
- `04-efs/3-throughput-modes.md`

Flashcard:
```
General Purpose: latency thấp (<1ms), 99% use cases
Max I/O: latency cao hơn, hàng nghìn clients đồng thời

Bursting: throughput tỷ lệ với storage size
Provisioned: throughput cố định, bạn đặt trước
Elastic: tự scale (KHUYẾN NGHỊ cho mới triển khai)
```

**Ngày 20-21: EFS vs EBS vs S3**

Đọc: `04-efs/5-efs-vs-ebs-vs-s3.md`

Bài tập cuối tuần — Cho mỗi scenario, chọn storage service:
```
1. 10 EC2 web servers cần share user uploaded files
2. MySQL database trên EC2
3. 100TB data lake cho analytics
4. WordPress media files, 2 EC2 servers
5. Kafka log streaming, sequential write nhiều
6. User avatar images, truy cập qua CloudFront
```

---

### Tuần 4: Ôn Tập Giai Đoạn 1 + Mini Project

**Ngày 22-25: Review và Gap Analysis**

Tự làm quiz không nhìn tài liệu:
1. Vẽ sơ đồ: S3 storage classes và lifecycle transitions có thể
2. Điền vào bảng: EBS volume types — Max IOPS, Max Throughput, Use case
3. Giải thích: Khi nào EFS rẻ hơn EBS?
4. Trả lời: Presigned URL có thể revoke không? Tại sao?

**Ngày 26-28: Mini Project — S3 Static Website với Security**

```bash
# 1. Tạo bucket với static website hosting
aws s3api create-bucket --bucket my-static-site --region ap-southeast-1
aws s3 website s3://my-static-site/ --index-document index.html

# 2. Upload content
echo '<h1>Hello World</h1>' > index.html
aws s3 cp index.html s3://my-static-site/

# 3. Tạo bucket policy cho public read
# (Lưu vào policy.json rồi apply)

# 4. Bật versioning
aws s3api put-bucket-versioning \
  --bucket my-static-site \
  --versioning-configuration Status=Enabled

# 5. Thêm lifecycle rule: delete old versions sau 30 ngày

# 6. Bật server access logging
```

---

## 🟡 Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 5–8)

### Tuần 5: Bảo Mật Storage

**Ngày 29-31: IAM và Bucket Policies**

Đọc:
- `05-security/1-iam-policies-for-storage.md`
- `05-security/2-s3-bucket-policies.md`

Bài tập: Viết bucket policy cho từng scenario:
```json
// Scenario 1: Chỉ cho phép account 123456789012 PUT objects
// Scenario 2: Deny mọi access không phải HTTPS
// Scenario 3: Chỉ cho phép access từ VPC endpoint vpce-1234
// Scenario 4: Cho phép EC2 role arn:aws:iam::123:role/app-role đọc
```

**Ngày 32-34: Encryption — Mã Hóa**

Đọc:
- `05-security/4-encryption-at-rest.md`
- `05-security/5-encryption-in-transit.md`

So sánh nhanh:
```
SSE-S3:   AWS quản lý key → đơn giản nhất, không audit key access
SSE-KMS:  Bạn kiểm soát key policy, CloudTrail log → cần compliance
SSE-C:    Bạn cung cấp key → AWS không lưu → nếu mất key = mất data
CSE:      Mã hóa trước khi upload → AWS không thấy plaintext
```

**Ngày 35: VPC Endpoints và Block Public Access**

Đọc:
- `05-security/3-block-public-access.md`
- `05-security/6-vpc-endpoints.md`

Thực hành: Tạo Gateway Endpoint cho S3 trong VPC test

---

### Tuần 6: Lifecycle Policies và Cost Optimization

**Ngày 36-38: Lifecycle Design**

Đọc:
- `02-s3-advanced/1-lifecycle-policies.md`
- `06-cost-optimization/3-lifecycle-rules-design.md`

Bài tập thiết kế lifecycle cho 3 scenario:
```
1. Application log: retain 1 năm, 90% không access sau 30 ngày
2. Database backup: retain 7 năm theo compliance
3. User media files: không biết access pattern
```

**Ngày 39-41: S3 Replication**

Đọc:
- `02-s3-advanced/2-cross-region-replication.md`
- `02-s3-advanced/3-same-region-replication.md`

Thực hành: Cấu hình CRR giữa 2 bucket (cùng account, khác region)

**Ngày 42: S3 Object Lock và Pricing**

Đọc:
- `02-s3-advanced/4-s3-object-lock.md`
- `06-cost-optimization/1-s3-pricing-guide.md`

Bài tập tính toán:
```
Tính chi phí cho scenario:
- 10TB Standard-IA, 100GB được access/ngày
- Lifecycle rule: chuyển Glacier sau 90 ngày
- 5M requests GET/tháng
→ Total monthly cost?
```

---

### Tuần 7: Disaster Recovery và Monitoring

**Ngày 43-46: DR Strategy**

Đọc toàn bộ `08-disaster-recovery/`:
- RPO/RTO concepts và thiết kế
- CRR cho DR
- EBS snapshot strategy
- AWS Backup service

Quiz:
- [ ] RPO = 4 giờ nghĩa là gì cụ thể về backup frequency?
- [ ] Làm thế nào đạt RTO 15 phút với S3?
- [ ] Backup Vault Lock dùng khi nào?

**Ngày 47-49: CloudWatch Monitoring**

Đọc:
- `07-monitoring/1-cloudwatch-s3-metrics.md`
- `07-monitoring/2-cloudwatch-ebs-metrics.md`

Thực hành: Tạo CloudWatch alarm cho EBS BurstBalance < 20%

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EBS-BurstBalance-Low" \
  --alarm-description "EBS burst balance below 20%" \
  --metric-name BurstBalance \
  --namespace AWS/EBS \
  --statistic Average \
  --period 300 \
  --threshold 20 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:storage-alerts
```

---

### Tuần 8: Advanced S3 và Storage Gateway

**Ngày 50-53: S3 Advanced Features**

Đọc:
- `02-s3-advanced/5-s3-event-notifications.md`
- `02-s3-advanced/6-s3-batch-operations.md`

Project nhỏ: Cấu hình S3 Event Notification → Lambda → xử lý ảnh (hoặc mock)

**Ngày 54-56: Storage Gateway Hybrid**

Đọc toàn bộ `09-storage-gateway/`:
- File Gateway, Volume Gateway, Tape Gateway
- DataSync cho migration

Vẽ sơ đồ: Hybrid storage architecture cho một công ty có 50TB on-premises

---

## 🔴 Giai Đoạn 3: Nâng Cao & Chuẩn Bị Phỏng Vấn (Tuần 9–13)

### Tuần 9: FSx và Snow Family

**Ngày 57-61: FSx Variants**

Đọc toàn bộ `11-fsx/`:
- FSx Windows vs Lustre vs NetApp ONTAP vs OpenZFS
- Khi nào dùng từng loại

Flashcard quyết định nhanh:
```
Windows workload + Active Directory → FSx Windows
HPC, ML training → FSx Lustre
Enterprise migration từ NetApp → FSx NetApp ONTAP
Dev/test cần snapshots nhanh → FSx OpenZFS
```

**Ngày 62-63: Snow Family**

Đọc `10-snow-family/`:
- Snowcone vs Snowball Edge vs Snowmobile
- Khi nào Snow vs DataSync

---

### Tuần 10: System Design Practice

**Ngày 64-67: Tự Thiết Kế (Không Nhìn Tài Liệu)**

Với mỗi scenario, tự vẽ sơ đồ và giải thích (30 phút/scenario):

**Scenario A:** Startup 100.000 users cần lưu user-generated content (ảnh, video), budget $1.000/tháng

**Scenario B:** Bank cần backup transaction log 7 năm, RPO = 0, compliance MAS

**Scenario C:** E-learning platform, 500 TB video content, phân phối toàn cầu

**Scenario D:** Enterprise có 200TB on-premises files, muốn hybrid cloud trong 6 tháng

**Ngày 68-70: Review và Gap Fix**

Sau khi tự làm, so sánh với `12-interview-prep/2-system-design-scenarios.md`

Xác định gaps và ôn lại phần còn yếu.

---

### Tuần 11: Behavioral Questions và STAR Stories

**Ngày 71-73: Viết STAR Stories**

Từ kinh nghiệm thực tế của bạn, viết ít nhất 4 câu chuyện STAR:
1. Storage incident bạn xử lý
2. Cost optimization bạn thực hiện
3. Architecture decision bạn đưa ra (EBS vs EFS vs S3)
4. Security issue liên quan đến storage

Nếu chưa có kinh nghiệm thực tế: sử dụng stories mẫu trong `3-star-stories.md` và điều chỉnh theo context giả định nhưng realistic.

**Ngày 74-77: Luyện Nói To**

Luyện trả lời mỗi câu hỏi bằng cách NÓI TO, không viết:
- Record bằng điện thoại
- Nghe lại và đánh giá: rõ ràng không? Có số liệu cụ thể không? Đúng thời gian (2-3 phút)?
- Lặp lại cho đến khi tự nhiên

---

### Tuần 12: Mock Interviews — Phỏng Vấn Thử

**Ngày 78-80: Mock Interview 1 — Technical**

Tìm đồng nghiệp hoặc dùng ChatGPT/AI để mock:
- 5 câu technical questions từ `1-INTERVIEW_GUIDE.md`
- 1 system design scenario
- Feedback: điểm mạnh, điểm cần cải thiện

**Ngày 81-83: Mock Interview 2 — Behavioral**

- 4 câu behavioral questions về storage
- Đánh giá STAR format: có đủ Situation-Task-Action-Result?
- Thời gian: mỗi câu 2-3 phút

**Ngày 84-86: Mock Interview 3 — Cost & Architecture**

- Câu hỏi từ `4-cost-optimization-questions.md`
- 1 cost optimization scenario thực tế
- Đánh giá: có thể hiện tư duy business/cost-aware không?

---

### Tuần 13: Ôn Tập Cuối và Chuẩn Bị

**Ngày 87-88: Rapid Review**

Flashcard ôn nhanh — 2 giờ:
- 6 S3 storage classes → giá và use case
- 5 EBS volume types → IOPS và use case
- 4 encryption methods → ai quản lý key
- 4 gateway types → protocol và use case

**Ngày 89-90: Chiến Lược Phỏng Vấn**

Ngày 89 — Chuẩn bị logistics:
- In (hoặc mở tab) key diagrams: S3 storage classes, EBS comparison
- Chuẩn bị câu hỏi ngược lại cho interviewer
- Review company tech stack — họ dùng dịch vụ AWS nào?

Ngày 90 — Ngày trước phỏng vấn:
- Ôn nhẹ 1 giờ, tập trung vào điểm yếu nhất
- Ngủ đủ giấc (thực sự quan trọng hơn ôn thêm 2 giờ)
- Chuẩn bị 3-4 câu hỏi để hỏi lại interviewer về culture, tech challenges

---

## 📅 Lịch Học Hàng Ngày

### Ngày Bình Thường (1.5 giờ)

```
30 phút — Đọc tài liệu
30 phút — Thực hành (AWS Console hoặc CLI)
30 phút — Ôn lại hôm qua, làm quiz
```

### Cuối Tuần (3 giờ)

```
60 phút — Ôn tập cả tuần
60 phút — Thực hành lab hoặc project nhỏ
60 phút — Luyện giải thích/mock
```

---

## ✅ Milestone Checkpoints

### Cuối Tuần 4 (Giai Đoạn 1)

- [ ] Tạo và cấu hình S3 bucket từ CLI không cần Google
- [ ] Giải thích 6 storage classes không nhìn tài liệu
- [ ] Chọn đúng EBS volume type cho 5 scenario khác nhau
- [ ] Mount EFS vào 2 EC2 instances và verify shared access

### Cuối Tuần 8 (Giai Đoạn 2)

- [ ] Viết bucket policy đúng cho 3 scenario security
- [ ] Thiết kế lifecycle rule cho data có retention 7 năm
- [ ] Giải thích CRR vs SRR — use case khác nhau
- [ ] Tính toán chi phí S3 cho scenario đã cho

### Cuối Tuần 13 (Sẵn Sàng Phỏng Vấn)

- [ ] Trả lời 20/20 câu hỏi trong `1-INTERVIEW_GUIDE.md` tự tin
- [ ] Có 4 câu chuyện STAR đã luyện nhuần nhuyễn
- [ ] Hoàn thành 3 mock interviews với feedback tích cực
- [ ] Thiết kế được 3/5 system design scenarios trong 20 phút

---

## 🔧 Công Cụ và Tài Nguyên

### Thực Hành

- **AWS Free Tier:** S3 5GB miễn phí, EC2 750h/tháng miễn phí
- **AWS CLI:** Cài đặt và cấu hình `aws configure`
- **CloudShell:** Browser-based CLI, không cần cài đặt

### Học Lý Thuyết

- **AWS Documentation:** docs.aws.amazon.com — source of truth
- **AWS Skill Builder:** Khóa học miễn phí từ AWS
- **A Cloud Guru hoặc Udemy:** Có video course AWS SAA-C03

### Luyện Tập

- **ExamTopics:** Câu hỏi thi AWS certification để test knowledge
- **Whizlabs:** Practice exams
- **Peer learning:** Tìm study buddy để mock interview nhau

---

## ⚠️ Lỗi Thường Gặp Khi Học

1. **Đọc nhiều, thực hành ít:** Storage khó nhớ nếu không tự tay làm. Tỷ lệ lý tưởng: 50% đọc, 50% thực hành

2. **Học thuộc thay vì hiểu:** Nhớ rằng gp3 có 16.000 IOPS max không đủ — phải hiểu *tại sao* chọn gp3 vs io2

3. **Bỏ qua cost:** Mọi quyết định kiến trúc đều có chi phí — tập thói quen ước tính cost cho mỗi option

4. **Không chuẩn bị behavioral questions:** Technical giỏi nhưng không kể được câu chuyện thực tế → fail behavioral round

5. **Mock interview 1 mình:** Tự trả lời trong đầu rất khác với nói ra miệng trước người khác. Phải nói thành lời

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
