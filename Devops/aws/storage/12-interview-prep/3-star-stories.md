# ⭐ STAR Stories — Câu Chuyện Phỏng Vấn Behavioral AWS Storage

> STAR — Situation (Tình huống) · Task (Nhiệm vụ) · Action (Hành động) · Result (Kết quả) — là framework chuẩn để trả lời câu hỏi behavioral trong phỏng vấn. Tài liệu này cung cấp template và ví dụ cho các tình huống storage phổ biến.

---

## 🎯 Tại Sao Cần Câu Chuyện STAR?

Câu hỏi behavioral thường bắt đầu bằng:
- *"Tell me about a time when..."*
- *"Give me an example of..."*
- *"Describe a situation where..."*

Interviewer đánh giá:
1. Cách bạn xử lý vấn đề thực tế (không phải lý thuyết)
2. Tư duy kỹ thuật và quyết định dưới áp lực
3. Kỹ năng giao tiếp và teamwork
4. Bài học rút ra — growth mindset

---

## 📋 Template STAR Cho Storage Incidents

```
SITUATION (30-45 giây):
→ Bối cảnh: hệ thống gì, scale bao nhiêu, thời điểm nào
→ Vấn đề gì xảy ra: symptom rõ ràng

TASK (15-20 giây):
→ Vai trò của bạn: lead, contributor, hay escalation point
→ Mục tiêu cần đạt: fix trong bao lâu, impact là gì

ACTION (2-3 phút):
→ Bước cụ thể bạn làm — chronological order
→ Công cụ, services, commands sử dụng
→ Quyết định khó khăn và tại sao

RESULT (30-45 giây):
→ Outcome đo được: downtime, cost savings, performance
→ Bài học và thay đổi sau đó
```

---

## Story 1: Xử Lý S3 Bucket Vô Tình Bị Public

### Câu Hỏi Dạng Này

*"Tell me about a time you discovered a security vulnerability in production."*
*"Describe a situation where you had to respond to a security incident quickly."*

### STAR Story Mẫu

**Situation:**
Lúc 2 giờ sáng, tôi nhận được alert từ hệ thống monitoring: một S3 bucket chứa dữ liệu export của khách hàng đang có traffic bất thường — gấp 50 lần bình thường. Đây là bucket production của một fintech platform xử lý khoảng 200.000 giao dịch/ngày. Sau khi check nhanh, tôi phát hiện bucket đã bị public do một pull request deploy nhầm bucket policy — developer mới đã copy template từ bucket staging (public cho test) sang production.

**Task:**
Vai trò của tôi là on-call engineer đêm đó. Mục tiêu: ngắt public access trong vòng 15 phút, sau đó assess impact (ai đã access), và prevent recurrence. Data có PII — thông tin cá nhân của khách hàng — nên đây là incident nghiêm trọng theo PDPA và tiêu chuẩn nội bộ.

**Action:**

1. **Fix ngay lập tức (T+5 phút):**
   - Chạy AWS CLI enable S3 Block Public Access ngay:
     ```bash
     aws s3api put-public-access-block \
       --bucket production-exports \
       --public-access-block-configuration \
         "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
     ```
   - Xóa bucket policy sai, revert về policy đúng từ git history

2. **Assess impact (T+5 đến T+30 phút):**
   - Query S3 Server Access Logs trong Athena để xác định IP nào đã access, object nào đã được download
   - Check CloudTrail events trong 24 giờ trước
   - Phát hiện: bucket public từ 6 giờ chiều (8 giờ), nhưng chỉ có crawler bots access — không có evidence con người tải xuống PII

3. **Notify và document (T+30 đến T+60 phút):**
   - Escalate lên CISO và legal team theo incident response playbook
   - Tạo incident ticket với timeline đầy đủ
   - Xác định exposure window và data involved

4. **Prevent recurrence:**
   - Thêm AWS Config rule: `s3-bucket-public-read-prohibited` — tự động detect và alert
   - Thêm CI/CD check: script kiểm tra bucket policy không có `"Principal": "*"` trước khi deploy
   - Bật S3 Access Analyzer cho toàn organization
   - Yêu cầu approval của security team cho mọi IAM/bucket policy change

**Result:**
- Incident được contain trong 15 phút
- Không có PII confirmed leak (logs cho thấy chỉ bots)
- Sau đó 0 public bucket incident trong 18 tháng nhờ automation check
- Case được dùng làm security training nội bộ
- Bài học: staging và production cần tách biệt hoàn toàn — kể cả bucket policy templates

---

## Story 2: Tối Ưu Chi Phí S3 Giảm 45%

### Câu Hỏi Dạng Này

*"Tell me about a time you saved money on cloud infrastructure."*
*"Describe a situation where you optimized system performance or cost."*

### STAR Story Mẫu

**Situation:**
Tháng 3 năm ngoái, Finance team cảnh báo rằng AWS bill của chúng tôi đã tăng 80% trong 6 tháng, nhưng traffic chỉ tăng 30%. S3 chiếm 40% tổng bill — khoảng $15.000/tháng. Hệ thống là một SaaS B2B với 500+ khách hàng, mỗi khách hàng có storage riêng. Không ai trong team biết tại sao chi phí tăng mạnh như vậy.

**Task:**
Tôi được giao nhiệm vụ phân tích và giảm S3 cost ít nhất 30% trong 1 tháng mà không ảnh hưởng đến performance cho end users.

**Action:**

1. **Phân tích hiện trạng (Tuần 1):**
   - Bật S3 Storage Lens để có cái nhìn toàn bộ organization
   - Chạy S3 Inventory và query bằng Athena để phân tích object distribution
   - Phát hiện:
     - 60% objects không được access trong >90 ngày
     - 15TB incomplete multipart uploads đang tồn tại (không ai complete)
     - 3 bucket đang version tất cả objects nhưng không có lifecycle rule → versions tích lũy vô tận
     - 8TB thumbnail images đang dùng Standard class dù chỉ được access vài lần/năm

2. **Quick wins — Thắng lợi nhanh (Tuần 1-2):**
   - Xóa incomplete multipart uploads: thêm lifecycle rule `AbortIncompleteMultipartUpload` 7 ngày
     → Tiết kiệm ngay $450/tháng
   - Xóa old versions: thêm lifecycle rule delete versions cũ >90 ngày
     → Tiết kiệm $1.200/tháng

3. **Lifecycle policies (Tuần 2-3):**
   - Phân loại 3 nhóm data:
     - **Hot data** (access <30 ngày): giữ Standard
     - **Warm data** (30-90 ngày): chuyển Standard-IA
     - **Cold data** (>90 ngày): Glacier Instant Retrieval
   - Triển khai lifecycle rules theo prefix (`/customer-files/`, `/exports/`, `/reports/`)
   - Test trên 1 bucket nhỏ trước 1 tuần để verify không break gì

4. **Intelligent-Tiering cho unpredictable data (Tuần 3-4):**
   - Thumbnails và generated files → chuyển sang Intelligent-Tiering
   - Access không dự đoán được — IT sẽ tự optimize

5. **CloudFront trước S3 cho public assets (Tuần 4):**
   - Static assets trước đây serve từ S3 trực tiếp → data transfer out fee cao
   - Thêm CloudFront distribution → cache hit rate 85% → giảm S3 GET requests và data transfer

**Result:**
- Tháng đầu: giảm từ $15.000 xuống $8.200/tháng (-45%)
- 3 tháng sau: ổn định ở $7.500/tháng (-50%)
- Zero performance impact — users không phàn nàn gì
- Process được document và bây giờ là quarterly cost review standard
- Bài học lớn nhất: Versioning mà không có lifecycle là "silent cost killer"

---

## Story 3: Thiết Kế và Implement Disaster Recovery

### Câu Hỏi Dạng Này

*"Tell me about a time you designed a system for high availability or disaster recovery."*
*"Describe a complex technical project you led from start to finish."*

### STAR Story Mẫu

**Situation:**
Công ty tôi cung cấp dịch vụ e-learning cho 50 trường đại học, tổng cộng 200.000 học sinh dùng platform hàng ngày. Tháng 9 năm ngoái, AWS region `ap-southeast-1` có sự cố nhỏ 45 phút — không phải outage toàn diện — nhưng chúng tôi down 2 tiếng vì không có DR plan. Đó là đầu kỳ thi — impact rất lớn, nhận được nhiều khiếu nại. CTO yêu cầu implement DR trong vòng 3 tháng.

**Task:**
Tôi là Tech Lead phụ trách thiết kế và implement toàn bộ DR strategy cho storage layer. Mục tiêu: RPO = 2 giờ, RTO = 1 giờ. Budget: không có budget tăng thêm — phải optimize từ chi phí hiện tại.

**Action:**

1. **Phân tích và chọn strategy (Tháng 1):**
   - Inventory tất cả storage: 2TB video lectures, 500GB user data, 50GB database backups
   - Đánh giá 3 options:
     - **Pilot Light (Đèn Pilot):** Chỉ replicate data, infra DR start khi cần → RTO cao (2-4h)
     - **Warm Standby (Chờ Ấm):** Data + minimal infra running → RTO 30-60 phút
     - **Multi-Region Active-Active:** Tốt nhất nhưng đắt gấp đôi
   - Chọn Warm Standby vì phù hợp budget và RTO 1 giờ

2. **Implement storage replication (Tháng 1-2):**
   - **S3 CRR — Cross-Region Replication** từ `ap-southeast-1` sang `us-west-2`:
     - Bật versioning trên cả hai bucket
     - Tạo IAM role cho replication
     - Enable CRR với filter chỉ cho video và user upload (không replicate temp files)
     - Enable Replication Time Control — RTC — đảm bảo replicate trong 15 phút
   - **EBS Snapshots cross-region:**
     - Dùng AWS Data Lifecycle Manager tạo snapshot mỗi 2 giờ
     - Auto-copy sang `us-west-2`
     - Retention 7 ngày
   - **Database:** RDS Aurora Global Database — near-realtime replication

3. **Test và document (Tháng 2):**
   - Viết runbook chi tiết: từng bước failover và failback
   - Test failover trong maintenance window đêm thứ 7:
     - Simulate primary region outage
     - Start EC2 ở DR region
     - Promote Aurora secondary → primary
     - Update Route 53 DNS → DR region
     - Verify toàn bộ feature hoạt động
   - Thời gian thực tế: 47 phút (trong target 1 giờ)

4. **Automation và monitoring (Tháng 3):**
   - CloudWatch alarm trigger SNS → notify on-call khi primary down
   - Script tự động hóa 70% failover steps (chỉ cần xác nhận manual cho bước cuối)
   - Dashboard theo dõi replication lag real-time

**Result:**
- Triển khai đúng hạn 3 tháng
- Chi phí tăng thêm: +$800/tháng (DR region infra) — trong budget
- 6 tháng sau: có 1 sự cố nhỏ ở primary, team failover trong 52 phút — không có downtime nhận thấy bởi users
- DR test hàng quý trở thành quy trình chuẩn
- Bài học: document runbook ngay từ đầu, không đợi đến khi có incident thật

---

## Story 4: Xử Lý EBS Volume Đầy Trong Production

### Câu Hỏi Dạng Này

*"Tell me about a time you had to solve a critical problem under pressure."*
*"Describe a situation where you managed a production incident."*

### STAR Story Mẫu

**Situation:**
Một thứ 6, 4 giờ chiều — giờ cao điểm — hệ thống monitoring alert: EBS volume của production database server đã đầy 100%. Database PostgreSQL bắt đầu reject writes, API trả về 500 errors. Đây là hệ thống đặt hàng của một chuỗi siêu thị với 300 cửa hàng, đang trong giờ cao điểm cuối tuần.

**Task:**
On-call engineer. Cần restore database write capability trong vòng 15 phút (đây là SLA cho production incidents). Không được restart database nếu không cần thiết.

**Action:**

1. **Immediate triage (T+2 phút):**
   - SSH vào server, chạy `df -h` → `/var/lib/postgresql` 100%
   - Kiểm tra: đâu chiếm space?
     ```bash
     du -sh /var/lib/postgresql/* | sort -rh | head -20
     ```
   - Phát hiện: WAL — Write-Ahead Log (Nhật Ký Ghi Trước) files tích lũy 80GB do replication lag với replica

2. **Immediate mitigation (T+2 đến T+8 phút):**
   - Xóa safe: old WAL files đã được archive (kiểm tra trong `pg_wal_lsn_diff`)
   - Free ~20GB → database resume writes
   - Service restore sau 8 phút

3. **Root cause fix (T+8 đến T+30 phút):**
   - Kiểm tra replication status: replica lag 2 ngày vì network issue
   - Rebuild replica
   - Xóa WAL files cũ không còn cần thiết

4. **Long-term fix:**
   - Resize EBS volume từ 500GB → 1TB (online resize — không downtime):
     ```bash
     aws ec2 modify-volume --volume-id vol-xxx --size 1000
     # Sau đó resize filesystem
     sudo resize2fs /dev/xvdb
     ```
   - Thêm CloudWatch alarm khi disk usage >75% (trước đây chỉ alert ở 90%)
   - Implement automated WAL archiving với S3
   - Tạo runbook cho "EBS volume full" scenario

**Result:**
- Downtime: 8 phút (trong SLA 15 phút)
- Không mất data
- Sau đó: alert ở 75% cho phép proactive action — không còn full-volume emergency nào trong 1 năm
- Bài học: Alert thresholds cần đủ sớm để action trước khi critical

---

## Story 5: Migrate 10TB Data Lên Cloud

### Câu Hỏi Dạng Này

*"Tell me about a complex migration project you were involved in."*
*"Describe a time you had to plan and execute a large-scale technical project."*

### STAR Story Mẫu

**Situation:**
Công ty mua lại một startup và cần migrate toàn bộ infrastructure của startup đó lên AWS. Phần khó nhất: 10TB media files (ảnh và video) đang trên một NAS on-premises cũ, kết nối qua cáp 100Mbps. Startup tiếp tục hoạt động trong quá trình migration — không có maintenance window lớn.

**Task:**
Tôi được giao plan và execute migration với yêu cầu: zero data loss, downtime <4 giờ cho việc cắt sang S3 cuối cùng, hoàn thành trong 4 tuần.

**Action:**

1. **Analysis và planning (Tuần 1):**
   - Inventory 10TB: 2 triệu files, average 5MB/file
   - Tính toán: 10TB / 100Mbps = ~10 ngày nếu dùng network liên tục
   - Quyết định: dùng AWS DataSync agent (parallel streams, checksum) thay vì `rsync` thủ công
   - Plan: sync lần đầu (initial bulk) → continuous sync trong khi app vẫn dùng NAS → cutover

2. **Initial sync (Tuần 1-2):**
   - Cài DataSync agent trên VMware tại data center
   - Tạo DataSync task: NAS → S3 bucket
   - Chạy initial sync với 128 parallel connections (DataSync default)
   - Thời gian thực tế: 8 ngày (DataSync tối ưu tốt hơn naive copy)
   - Verify: DataSync tự kiểm tra checksum — báo cáo 0 errors

3. **Delta sync và preparation (Tuần 3):**
   - Chạy DataSync scheduled sync mỗi 4 giờ để bắt kịp changes mới
   - Chuẩn bị phía application: thay đổi config để đọc từ S3 (dùng S3 URI thay vì NAS path)
   - Test với môi trường staging: simulate cutover, verify tất cả features hoạt động

4. **Cutover (Tuần 4, Saturday 11 PM):**
   - Maintenance window 4 giờ (11PM - 3AM)
   - Stop all writes đến NAS
   - Chạy final DataSync sync (chỉ delta của vài giờ → xong trong 30 phút)
   - Verify checksums final batch
   - Update application config → point to S3
   - Enable S3 in application → smoke test tất cả media features
   - Thực tế: cutover hoàn tất sau 2.5 giờ

**Result:**
- 0 data loss — verified qua checksum
- Cutover trong 2.5 giờ (trong target 4 giờ)
- Application performance cải thiện: S3 serve với CloudFront nhanh hơn NAS on-premises
- Chi phí lưu trữ giảm 30% so với maintain NAS hardware
- Bài học: DataSync checksum verification là bắt buộc — không bao giờ trust "copy xong là đúng" với dữ liệu lớn

---

## 📝 Bài Tập Tự Chuẩn Bị

Trả lời các câu hỏi sau theo format STAR từ kinh nghiệm thực tế của bạn:

1. **Incident:** Lần storage incident (S3, EBS, EFS) tệ nhất bạn xử lý — bạn học được gì?
2. **Optimization:** Lần bạn giảm chi phí lưu trữ đáng kể nhất — approach của bạn?
3. **Design decision:** Lần bạn phải chọn giữa EBS, EFS, S3 — bạn đã cân nhắc thế nào?
4. **Security:** Lần bạn phát hiện security issue liên quan đến storage — bạn xử lý thế nào?
5. **Teamwork:** Lần bạn phải giải thích storage architecture cho non-technical stakeholder?

### Tips Khi Kể STAR Story

- **Cụ thể hơn là chung chung:** "Giảm 45%" tốt hơn "giảm đáng kể"
- **Đề cập số liệu:** Data volume, downtime minutes, cost savings $
- **Nói về bạn:** Dùng "tôi" không phải "team" cho Actions
- **Kết thúc bằng bài học:** Interviewer muốn thấy growth mindset
- **Chuẩn bị 2 version:** 90-giây version và 3-phút version

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
