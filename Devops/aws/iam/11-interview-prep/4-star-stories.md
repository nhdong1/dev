# STAR Stories — Câu Chuyện Sự Cố Bảo Mật Theo Phương Pháp STAR

> Mẫu câu chuyện sự cố bảo mật AWS theo phương pháp STAR (Situation — Task — Action — Result) để chuẩn bị cho phần behavioral interview.

---

## 📑 Mục Lục

1. [Phương Pháp STAR Cho Bảo Mật](#phương-pháp-star-cho-bảo-mật)
2. [Story 1: Xử Lý EC2 Bị Compromise](#story-1-xử-lý-ec2-bị-compromise)
3. [Story 2: Phát Hiện S3 Bucket Bị Public](#story-2-phát-hiện-s3-bucket-bị-public)
4. [Story 3: Migrate Từ Access Key Sang IAM Role](#story-3-migrate-từ-access-key-sang-iam-role)
5. [Story 4: Xây Dựng Multi-Account Structure](#story-4-xây-dựng-multi-account-structure)
6. [Story 5: Implement Secrets Manager Rotation](#story-5-implement-secrets-manager-rotation)
7. [Story 6: Phản Hồi Leaked Credentials Trên GitHub](#story-6-phản-hồi-leaked-credentials-trên-github)
8. [Câu Hỏi Behavioral Thường Gặp](#câu-hỏi-behavioral-thường-gặp)
9. [Tips Kể Chuyện Hiệu Quả](#tips-kể-chuyện-hiệu-quả)

---

## Phương Pháp STAR Cho Bảo Mật

### Cấu Trúc STAR

```
S — Situation (Tình Huống):
    Bối cảnh cụ thể — công ty gì, scale, environment, thời điểm nào
    Nên có: numbers (số lượng), context (bối cảnh), urgency (mức độ khẩn)

T — Task (Nhiệm Vụ):
    Bạn cần giải quyết vấn đề gì cụ thể?
    Vai trò của bạn là gì trong tình huống đó?
    Constraints (ràng buộc) nào bạn phải đối mặt?

A — Action (Hành Động):
    Những bước cụ thể bạn đã làm (dùng "Tôi" thay vì "Chúng tôi")
    Technical details — dịch vụ AWS nào, lệnh nào, quyết định thiết kế nào
    Tại sao bạn chọn hướng đó (show your thinking)

R — Result (Kết Quả):
    Kết quả cụ thể với số liệu
    Bài học rút ra
    Cải thiện gì sau đó
```

### Thời Gian Kể

```
Phone screen: 90 giây — 2 phút (version ngắn)
Technical round: 3–4 phút (version đầy đủ)
Behavioral round: 4–5 phút (version chi tiết + lessons learned)
```

---

## Story 1: Xử Lý EC2 Bị Compromise

### Câu Hỏi Trigger

- "Kể về lần bạn phát hiện và xử lý một security incident"
- "Bạn đã xử lý compromised resource thế nào?"
- "Kể về tình huống khẩn cấp nhất trong career bảo mật của bạn"

### Câu Chuyện

---

**Situation:**

Vào 2 giờ sáng thứ Sáu, GuardDuty gửi alert Critical — finding `CryptoCurrency:EC2/BitcoinTool.B` cho một EC2 instance trong production environment. Instance này là một phần của payment processing cluster xử lý 50.000 transactions mỗi ngày. Đây là lần đầu tiên team gặp EC2 compromise ở production.

**Task:**

Tôi là on-call engineer nhận alert đó. Nhiệm vụ là: (1) ngăn chặn thiệt hại tiếp theo, (2) giữ cho payment service hoạt động bình thường, và (3) điều tra nguyên nhân — tất cả trong SLA 2 giờ. Tôi không được terminate instance ngay vì cần preserve evidence.

**Action:**

Tôi làm theo runbook của team nhưng cũng phải improvise vài bước:

*Containment ngay lập tức (0–15 phút):*
Tôi tạo một "quarantine security group" chỉ cho phép SSH từ bastion host và không có outbound traffic. Sau đó thay security group của instance compromise bằng quarantine group này — cô lập hoàn toàn khỏi payment network mà không terminate instance.

Tiếp theo, tôi revoke session của instance profile bằng cách tạo inline deny policy vào role đó:

```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "DateLessThan": {"aws:TokenIssueTime": "2026-05-16T02:15:00Z"}
  }
}
```

Điều này invalidate tất cả temporary credentials đã issued trước thời điểm tôi phát hiện.

*Điều tra (15–60 phút):*
Tôi snapshot EBS volume ngay để preserve evidence. Sau đó dùng AWS Detective để xem behavior graph — phát hiện instance này đã bắt đầu gửi traffic tới một IP ở Romania từ 22 giờ hôm trước (4 giờ trước khi GuardDuty alert). Traffic pattern từ VPC Flow Logs cho thấy outbound port 4444 — reverse shell.

CloudTrail cho thấy 3 ngày trước có một `ec2:AuthorizeSecurityGroupIngress` từ một IAM user — mở port 8080 ra internet. Đó là initial access vector.

*Recovery:*
Tôi launch instance mới từ golden AMI (không phải snapshot bị ảnh hưởng), cập nhật load balancer để route traffic sang instance mới. Terminate instance cũ sau khi confirmed service healthy.

**Result:**

Payment service downtime: 0 phút (containerized, load balancer tự route). Evidence được preserve đầy đủ cho forensics. Tôi identify được 2 lỗ hổng: (1) không có control ngăn mở port ra internet, (2) không có IMDSv2 enforcement khiến malware dễ lấy instance credentials.

Sau incident, tôi implement SCP chặn `ec2:AuthorizeSecurityGroupIngress` với source `0.0.0.0/0`, bật GuardDuty Runtime Monitoring cho ECS, và enforce IMDSv2 qua Config Rule. MTTD (Mean Time To Detect — Thời Gian Trung Bình Phát Hiện) giảm từ 4 giờ xuống còn 3 phút.

---

### Version 90 Giây (Phone Screen)

*"GuardDuty phát hiện EC2 production đang mine crypto lúc 2 giờ sáng. Tôi cô lập instance bằng quarantine security group, revoke instance credentials để ngăn lateral movement, snapshot EBS để preserve evidence. Dùng Detective để trace back — phát hiện initial access qua port mở bởi IAM user 3 ngày trước. Recovery hoàn tất không downtime vì containerized. Sau đó implement SCP chặn mở port ra internet và enforce IMDSv2. MTTD giảm từ 4 giờ xuống 3 phút."*

---

## Story 2: Phát Hiện S3 Bucket Bị Public

### Câu Hỏi Trigger

- "Bạn đã phát hiện misconfiguration nghiêm trọng thế nào?"
- "Kể về tình huống bạn phải balance giữa security fix và business continuity"
- "Bạn có kinh nghiệm với data breach prevention không?"

### Câu Chuyện

---

**Situation:**

Trong quá trình audit định kỳ hàng quý dùng Prowler, tôi phát hiện một S3 bucket chứa báo cáo financial quarterly reports của khách hàng enterprise — khoảng 2.400 files PDF — bị public accessible. Bucket này được tạo 6 tháng trước bởi một contractor và không ai biết. Đây là dữ liệu thuộc NDA (Non-Disclosure Agreement — Thỏa Thuận Không Tiết Lộ) với khách hàng.

**Task:**

Nhiệm vụ của tôi là: (1) xác định mức độ exposure (phơi bày), (2) lock down ngay lập tức, và (3) xác định xem có ai truy cập chưa — tất cả mà không xóa bucket (vì có thể cần làm evidence). CFO và Legal team cần được thông báo trong vòng 1 giờ.

**Action:**

*Bước 1: Xác định exposure (0–10 phút):*
Tôi enable S3 server access logging ngay và check S3 access logs (bucket này may mắn có logs từ trước). Chạy query trên CloudTrail Lake để xem ai đã GetObject từ bucket:

```sql
SELECT useridentity.accountid, sourceipaddress, eventtime, requestparameters.bucketname
FROM cloudtrail_logs
WHERE eventname = 'GetObject'
  AND requestparameters.bucketname = 'sensitive-reports-bucket'
  AND useridentity.accountid != '123456789012'  -- loại trừ account mình
ORDER BY eventtime DESC;
```

Kết quả: 3 external IP addresses đã access 12 files trong 2 tháng gần đây. Các IP đó sau đó được identify là crawlers/bots thông qua threat intelligence — không phải targeted attack.

*Bước 2: Immediate remediation (10–20 phút):*
Bật S3 Block Public Access cho bucket đó ngay lập tức. Xóa bucket policy cho phép public access. Verify bằng cách thử access từ incognito browser — confirmed blocked.

*Bước 3: Scope assessment:*
Tôi chạy Macie scan toàn bộ account để kiểm tra xem có bucket nào khác tương tự không — phát hiện thêm 2 buckets có policy permissive nhưng chưa bị public hoàn toàn. Fix cả 2.

*Bước 4: Prevent recurrence:*
Tôi implement AWS Config Rule `s3-bucket-public-read-prohibited` và `s3-bucket-public-write-prohibited` với auto-remediation SSM Automation document. Nếu bucket nào vi phạm, Config sẽ tự enable Block Public Access trong vòng 5 phút.

Tôi cũng deploy SCP ở cấp Organization:
```json
{
  "Effect": "Deny",
  "Action": "s3:PutBucketPublicAccessBlock",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "s3:PublicAccessBlockConfiguration/BlockPublicAcls": "false"
    }
  }
}
```

**Result:**

Không có sensitive data được accessed bởi human (chỉ bots). Legal team confirm không cần customer notification theo GDPR vì không có evidence of human access và data đã được lock trong 20 phút. Thời gian từ phát hiện đến remediation: 20 phút. Sau đó, tổ chức triển khai monthly Macie scan và Config Rules — trong 3 tháng tiếp theo, auto-remediation trigger thêm 8 lần cho các buckets khác do developers tạo sai.

---

## Story 3: Migrate Từ Access Key Sang IAM Role

### Câu Hỏi Trigger

- "Bạn đã cải thiện security posture của tổ chức thế nào?"
- "Kể về lần bạn phải thuyết phục team thay đổi thực hành bảo mật"
- "Bạn đã xử lý technical debt bảo mật thế nào?"

### Câu Chuyện

---

**Situation:**

Khi join một startup 80 người, tôi phát hiện toàn bộ CI/CD pipeline (20+ GitHub Actions workflows) đang dùng long-lived IAM access keys lưu trong GitHub Secrets. Có 15 access keys đang active, key cũ nhất 2,5 năm chưa bao giờ rotate. Đây là classic "ticking time bomb" — nếu GitHub bị breach hoặc repository bị expose, toàn bộ production AWS bị ảnh hưởng.

**Task:**

Tôi cần migrate 20+ workflows sang OIDC federation mà không break CI/CD, trong khi team engineering đang trong sprint nước rút cho product launch 3 tuần sau. Không được disrupt development workflow.

**Action:**

*Bước 1: Audit và prioritize:*
Tôi list tất cả 15 access keys, xác định key nào dùng cho workflow nào, và scan IAM roles để xác định permissions của từng key. Ưu tiên migrate keys có broad permissions trước.

*Bước 2: Xây dựng OIDC infrastructure:*
Tôi tạo GitHub OIDC Provider trong AWS và design role naming convention: `github-{org}-{repo}-{environment}`. Viết Terraform module để tạo role mới dễ dàng:

```hcl
module "github_role" {
  source = "./modules/github-oidc-role"
  
  repo_name    = "payment-service"
  branch       = "main"
  permissions  = ["ecr:GetAuthorizationToken", "ecs:UpdateService"]
}
```

*Bước 3: Migration theo chiến lược parallel run:*
Thay vì migrate hết một lúc (rủi ro cao), tôi migrate từng workflow một. Mỗi workflow: tạo OIDC role mới → update workflow → test → delete access key sau 48 giờ nếu không có vấn đề.

*Bước 4: Convince team:*
Một số developer ban đầu resist vì "works fine". Tôi show họ data: 5 trong 15 access keys không được dùng trong 90 ngày — đó là dead credentials chưa ai biết. Tôi trình bày real breach case study (Codecov breach 2021) về impact của leaked CI/CD keys. Sau đó là dễ dàng.

**Result:**

Migration hoàn tất trong 3 tuần — trước product launch. 0 CI/CD downtime. 15 long-lived access keys được delete. Chi phí: 0 (OIDC free). Audit Committee đánh giá đây là cải thiện lớn nhất trong năm. Tôi sau đó viết internal blog post và IAM Guide cho team — được reuse bởi 3 team khác trong công ty.

---

## Story 4: Xây Dựng Multi-Account Structure

### Câu Hỏi Trigger

- "Bạn đã design và implement kiến trúc AWS scale-up thế nào?"
- "Kể về dự án bảo mật lớn nhất bạn từng làm"
- "Bạn đã làm thế nào để scale security cùng với tốc độ phát triển sản phẩm?"

### Câu Chuyện (Version Ngắn)

---

**Situation:**

Một scaleup SaaS với 150 kỹ sư đang chạy toàn bộ production, staging, và dev trên 1 AWS account. Khi một developer xóa nhầm một DynamoDB table trong production (tưởng là dev), CEO yêu cầu "fix architecture" ngay.

**Task:**

Tôi được giao lead initiative thiết kế và migrate sang multi-account structure trong 2 tháng, không ảnh hưởng product delivery.

**Action:**

Tôi bắt đầu với design sprint 2 tuần: phỏng vấn 20 stakeholders, map tất cả AWS resources hiện tại, và design OU structure phù hợp với team topology. Chọn AWS Control Tower để setup landing zone nhanh thay vì manual.

Chiến lược migration: "lift and shift accounts" — tạo account mới, migrate workload dùng CloudFormation StackSets, giữ account cũ như fallback trong 30 ngày. Triển khai IAM Identity Center để replace 50+ IAM users với SSO.

Key decision: không migrate tất cả cùng lúc. Bắt đầu bằng dev accounts (low risk), sau đó staging, cuối cùng production sau khi team đã quen với workflow mới.

**Result:**

Migration hoàn thành trong 8 tuần, 3 ngày trước deadline. Developer incident như "xóa nhầm production" không thể xảy ra nữa vì dev team không còn có quyền vào production account. Security Hub compliance score tăng từ 43% lên 87%. Thời gian onboard kỹ sư mới từ 2 ngày xuống còn 4 giờ nhờ SSO và permission sets.

---

## Story 5: Implement Secrets Manager Rotation

### Câu Hỏi Trigger

- "Bạn đã giải quyết technical debt liên quan đến secrets management thế nào?"
- "Kể về lần bạn implement auto-rotation cho credentials"
- "Bạn đã cải thiện security của database access thế nào?"

### Câu Chuyện (Version Ngắn)

---

**Situation:**

Trong security audit, phát hiện tất cả 12 database connections trong microservices đang dùng hardcoded password trong environment variables — cùng 1 password không đổi trong 2 năm. Password này lưu trong Kubernetes Secrets và 5 developer senior đều biết.

**Task:**

Migrate sang Secrets Manager với auto-rotation, zero-downtime, trong 30 ngày. Thách thức: 12 services viết bằng 4 ngôn ngữ khác nhau (Python, Node.js, Java, Go).

**Action:**

Tôi xây dựng internal SDK wrapper cho việc lấy secrets — thay vì mỗi service viết lại code, chỉ cần import và gọi `get_secret("db-password")`. SDK tự handle caching và refresh khi secret rotate.

Tôi dùng multi-user rotation strategy — tạo 2 DB users (service_a, service_b), rotate luân phiên để zero-downtime. Deploy Secrets Manager VPC endpoint để traffic không qua internet.

Migrate từng service theo rolling deployment — không phải big bang.

**Result:**

Toàn bộ 12 services migrate trong 28 ngày. 0 downtime khi rotate. Password rotation giờ xảy ra tự động mỗi 30 ngày — không cần human action. Security audit tiếp theo: zero hardcoded credentials findings. Tôi publish internal runbook — team data engineering dùng pattern tương tự cho 8 services của họ.

---

## Story 6: Phản Hồi Leaked Credentials Trên GitHub

### Câu Hỏi Trigger

- "Kể về tình huống cấp bách nhất bạn xử lý"
- "Bạn đã respond thế nào khi phát hiện credentials bị lộ?"
- "Quy trình incident response của bạn trông như thế nào?"

### Câu Chuyện (Version Ngắn)

---

**Situation:**

GitGuardian alert lúc 11 giờ đêm: AWS access key bị push lên public GitHub repository của một engineer junior — key đó có `IAMFullAccess` permission. Trong vòng 30 giây sau khi push, có các API calls từ IP lạ ở Nigeria.

**Task:**

Stop bleeding ngay lập tức, assess damage, và rebuild trust sau incident. SLA internal: 5 phút từ alert đến credentials bị invalidate.

**Action:**

*Phút 0–3:* Tôi delete access key ngay (không phải deactivate — delete hoàn toàn). Sau đó kiểm tra CloudTrail — API calls từ IP lạ đã tạo 2 IAM users mới và 1 EC2 instance trong us-east-1. Tôi disable ngay 2 IAM users đó và terminate EC2 instance.

*Phút 3–10:* Comprehensive scan với CloudTrail để xem attacker đã làm gì khác không. Tìm thấy attacker đã thử attach `AdministratorAccess` nhưng bị block vì Permission Boundary tôi đã implement cho tất cả roles (chỉ allow specific services, không allow IAM escalation).

*Sau đó:* Post-mortem với engineer junior — đây là learning experience, không phải blame. Implement git-secrets pre-commit hook cho tất cả repositories. Thêm GuardDuty S3 Protection để detect credential scanning trong S3.

**Result:**

Thời gian từ alert đến credential invalid: 2 phút 40 giây (SLA 5 phút — pass). Attacker không thể escalate vì Permission Boundaries đã chặn IAM privilege escalation. Thiệt hại thực: 1 EC2 instance $0.20 chi phí. After incident: git-secrets deployed, lần commit tiếp theo của engineer junior bị chặn vì detect test credentials — đây là "teaching moment" tốt nhất.

---

## Câu Hỏi Behavioral Thường Gặp

### Câu Hỏi & Story Phù Hợp

| Câu Hỏi | Story Phù Hợp |
|---|---|
| "Kể về security incident bạn xử lý" | Story 1, Story 6 |
| "Bạn cải thiện security posture thế nào?" | Story 2, Story 3 |
| "Dự án bảo mật lớn nhất?" | Story 4 |
| "Thuyết phục team thay đổi thực hành bảo mật?" | Story 3, Story 5 |
| "Balance security và business deadline?" | Story 4, Story 5 |
| "Làm việc dưới áp lực, khẩn cấp?" | Story 1, Story 6 |
| "Phòng ngừa data breach?" | Story 2 |
| "Technical debt bảo mật?" | Story 3, Story 5 |

### Câu Hỏi Hay Để Tự Đặt Ra (Từ Kinnh Nghiệm Của Bạn)

```
□ Lần nào bạn phát hiện misconfiguration nghiêm trọng?
□ Bạn đã migrate/improve một security system cũ thế nào?
□ Khi nào bạn phải chọn giữa shipping fast và security?
□ Bạn đã educate/train team về security thế nào?
□ Lần nào bạn bất đồng với decision của manager về bảo mật?
□ Bạn đã automate một security process tẻ nhàm thế nào?
□ Khi nào bạn phải escalate security issue lên lãnh đạo?
```

---

## Tips Kể Chuyện Hiệu Quả

### DOs (Nên Làm)

```
✅ Dùng "Tôi" thay vì "Chúng tôi" cho hành động cụ thể
   → "Tôi tạo quarantine security group" (không phải "Team chúng tôi")

✅ Cho số liệu cụ thể
   → "MTTD giảm từ 4 giờ xuống 3 phút" (không phải "cải thiện đáng kể")

✅ Mention technical details nhưng giải thích ngắn
   → "Tôi dùng IMDSv2 enforcement — để ngăn malware steal instance credentials từ metadata service"

✅ Kể về decisions và trade-offs
   → "Tôi chọn snapshot trước khi terminate để preserve evidence, dù làm chậm recovery 10 phút"

✅ Đề cập lessons learned cuối câu chuyện
   → "Sau đó tôi implement X để prevent tái diễn"
```

### DON'Ts (Không Nên)

```
❌ Kể chuyện quá dài (> 5 phút trong technical round)
❌ Blame người khác cho vấn đề (focus vào solution, không phải fault)
❌ Dùng jargon không giải thích (nếu không chắc interviewer biết)
❌ Claim credit của người khác
❌ Nói chuyện giả định ("Nếu tôi gặp tình huống đó, tôi sẽ...")
   → Phải là chuyện thật hoặc được trình bày như chuyện thật
```

### Cấu Trúc Câu Chuyện Linh Hoạt

```
Nếu bạn chưa có kinh nghiệm incident thực tế:
→ "Trong lab project của tôi, tôi mô phỏng scenario và xử lý như sau..."
→ "Khi tôi đọc về Capital One breach, tôi đã phân tích và nếu tôi ở vị trí đó..."
→ Describe một certification lab scenario (SAP-C02 labs rất phong phú)
```

---

**Cập Nhật Lần Cuối:** 2026-05-16
