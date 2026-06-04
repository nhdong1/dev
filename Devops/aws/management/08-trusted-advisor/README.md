# AWS Trusted Advisor — Cố Vấn Best Practice Tự Động

> **Trusted Advisor** (Cố Vấn Đáng Tin) là dịch vụ phân tích môi trường AWS của bạn và đưa ra **khuyến nghị cải thiện** theo 5 lĩnh vực: chi phí, hiệu suất, bảo mật, khả năng chịu lỗi và giới hạn dịch vụ — hoạt động như một chuyên gia tư vấn AWS luôn trực tuyến kiểm tra hạ tầng của bạn.

---

## 📚 Mục Lục

1. [Trusted Advisor là gì và tại sao cần?](#trusted-advisor-là-gì-và-tại-sao-cần)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [5 Lĩnh Vực Kiểm Tra](#5-lĩnh-vực-kiểm-tra)
4. [Cấp Độ Support & Số Lượng Checks](#cấp-độ-support--số-lượng-checks)
5. [Trusted Advisor vs Các Dịch Vụ Liên Quan](#trusted-advisor-vs-các-dịch-vụ-liên-quan)
6. [Tích Hợp & Tự Động Hóa](#tích-hợp--tự-động-hóa)
7. [Giới Hạn & Lưu Ý Quan Trọng](#giới-hạn--lưu-ý-quan-trọng)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
9. [Điều Hướng Module](#điều-hướng-module)

---

## Trusted Advisor là gì và tại sao cần?

### Vấn Đề Không Có Trusted Advisor

Khi vận hành AWS ở quy mô lớn, rất dễ bỏ sót các vấn đề tiềm ẩn:

```
❌ EC2 instances chạy suốt nhưng không ai dùng → lãng phí chi phí
❌ S3 bucket public mà không ai biết → rủi ro bảo mật
❌ RDS chỉ có 1 AZ → single point of failure
❌ Sắp chạm Service Quota mà không hay → ứng dụng bỗng dưng lỗi
❌ Security Group mở port 22/3389 cho 0.0.0.0/0 → lỗ hổng nghiêm trọng
```

### Trusted Advisor Giải Quyết Như Thế Nào?

```
✅ Tự động quét hạ tầng theo lịch (auto-refresh mỗi tuần, hoặc on-demand)
✅ Phân loại vấn đề theo 5 lĩnh vực rõ ràng
✅ Mức độ ưu tiên: 🔴 Action Required → 🟡 Investigation Recommended → ✅ No problems
✅ Tích hợp EventBridge để tự động phản ứng khi trạng thái check thay đổi
✅ API programmatic để đọc kết quả và tích hợp vào pipeline DevOps
```

### Khi Nào Nên Dùng Trusted Advisor?

| Tình Huống                                         | Khuyến Nghị                          |
| -------------------------------------------------- | ------------------------------------ |
| Tối ưu chi phí hàng tháng                         | ✅ Xem Cost Optimization checks       |
| Đánh giá bảo mật nhanh trước audit                | ✅ Xem Security checks                |
| Chuẩn bị ra production                            | ✅ Xem Fault Tolerance checks         |
| Theo dõi giới hạn dịch vụ trước khi launch        | ✅ Xem Service Limits checks          |
| Cần kiểm tra tuân thủ chi tiết (HIPAA, PCI-DSS)  | ❌ Dùng AWS Security Hub + Config     |
| Cần custom rules theo yêu cầu riêng               | ❌ Dùng AWS Config Custom Rules       |

---

## Kiến Trúc Tổng Quan

```
┌────────────────────────────────────────────────────────────────┐
│                      AWS Trusted Advisor                        │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Check Engine                             │  │
│  │                                                          │  │
│  │  ┌────────────┐  ┌───────────┐  ┌──────────────────┐    │  │
│  │  │    Cost    │  │ Security  │  │ Fault Tolerance   │    │  │
│  │  │Optimization│  │  Checks   │  │     Checks        │    │  │
│  │  └────────────┘  └───────────┘  └──────────────────┘    │  │
│  │  ┌─────────────────┐  ┌──────────────────────────────┐  │  │
│  │  │  Performance    │  │      Service Limits           │  │  │
│  │  │    Checks       │  │         Checks                │  │  │
│  │  └─────────────────┘  └──────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            │                                   │
│            ┌───────────────┼───────────────────┐              │
│            ▼               ▼                   ▼              │
│    ┌──────────────┐ ┌────────────┐  ┌──────────────────┐      │
│    │  Console UI  │ │ Support API│  │  EventBridge     │      │
│    │  Dashboard   │ │(programmatic)│ │  (automation)    │      │
│    └──────────────┘ └────────────┘  └──────────────────┘      │
│                                             │                  │
│                               ┌─────────────┼──────────────┐  │
│                               ▼             ▼              ▼  │
│                          ┌────────┐  ┌───────────┐  ┌──────┐  │
│                          │  SNS   │  │  Lambda   │  │Slack │  │
│                          └────────┘  └───────────┘  └──────┘  │
└────────────────────────────────────────────────────────────────┘
```

### Cơ Chế Hoạt Động

```
1. Trusted Advisor quét tài nguyên trong account
       │
       ▼
2. So sánh với AWS Best Practice rules (không tùy chỉnh được)
       │
       ▼
3. Phân loại kết quả: OK ✅ / Warning ⚠️ / Error ❌
       │
       ▼
4. Cập nhật Console Dashboard + Support API
       │
       ▼
5. Phát EventBridge event khi trạng thái check thay đổi
       │
       ▼
6. Trigger Lambda / SNS / automation workflow
```

---

## 5 Lĩnh Vực Kiểm Tra

### 1. Cost Optimization — Tối Ưu Chi Phí 💰

Phát hiện tài nguyên lãng phí và cơ hội tiết kiệm:

| Check                                        | Vấn Đề Phát Hiện                                     |
| -------------------------------------------- | ---------------------------------------------------- |
| Idle EC2 Instances                           | Instance CPU < 10% trong 14 ngày                    |
| Underutilized EBS Volumes                   | Volume không được gắn hoặc I/O rất thấp             |
| Unassociated Elastic IP Addresses           | EIP tồn tại nhưng không gắn vào instance            |
| Idle RDS DB Instances                       | RDS không có kết nối trong 7 ngày                   |
| Underutilized Redshift Clusters             | Cluster CPU < 5%, kết nối = 0                       |
| Reserved Instance Optimization              | Gợi ý mua RI dựa trên usage pattern                 |
| Savings Plan Optimization                   | Gợi ý Savings Plans phù hợp                        |
| Lambda Functions with High Error Rate       | Function lỗi nhiều → gây phí retry không cần thiết |

**Ví dụ tình huống:**
```
Account có 20 EC2 c5.xlarge đang chạy 24/7
→ Trusted Advisor phát hiện 5 instance CPU < 5% trong 2 tuần
→ Khuyến nghị: Hạ xuống c5.large hoặc tắt → tiết kiệm ~$800/tháng
```

---

### 2. Performance — Hiệu Suất ⚡

Phát hiện nút thắt cổ chai (bottleneck) và cấu hình chưa tối ưu:

| Check                                        | Vấn Đề Phát Hiện                                     |
| -------------------------------------------- | ---------------------------------------------------- |
| High Utilization EC2 Instances              | CPU > 90% trong 4 ngày → cần scale up               |
| Large Number of Rules in EC2 Security Group | Quá nhiều rules làm chậm network throughput         |
| CloudFront Header Forwarding                | Forward quá nhiều headers → giảm cache hit rate     |
| EBS Provisioned IOPS Optimization          | IOPS được cấp phát nhưng không dùng hết             |
| Route 53 Alias Resource Record Sets        | Dùng A record thay vì Alias → thêm DNS lookup       |
| CloudFront Content Delivery Optimization   | Origin response time cao → gợi ý tối ưu             |

---

### 3. Security — Bảo Mật 🔒

Phát hiện cấu hình sai và lỗ hổng bảo mật:

| Check                                             | Vấn Đề Phát Hiện                                          |
| ------------------------------------------------- | --------------------------------------------------------- |
| MFA on Root Account                              | Root account chưa bật MFA — **cực kỳ nguy hiểm**        |
| IAM Access Key Rotation                          | Access key không xoay vòng trong 90 ngày                 |
| IAM Password Policy                             | Policy quá yếu (độ dài, phức tạp, expiration)            |
| S3 Bucket Permissions                           | Bucket hoặc object có quyền public đọc/ghi               |
| Security Groups — Unrestricted Access           | Port 22, 3389, 3306, 5432 mở cho 0.0.0.0/0              |
| Amazon EBS Public Snapshots                     | EBS snapshot được chia sẻ công khai                       |
| Amazon RDS Public Snapshots                     | RDS snapshot được chia sẻ công khai                       |
| CloudTrail Logging                              | CloudTrail chưa bật hoặc chưa cấu hình đúng             |
| Amazon Route 53 MX and SPF Records             | Domain thiếu SPF/DMARC → dễ bị giả mạo email            |
| Exposed Access Keys                             | Access key bị lộ trên GitHub/public source               |

**Checks quan trọng nhất (thường hỏi trong phỏng vấn):**
```
🔴 MFA on Root Account     → Không có MFA trên root = rủi ro tối cao
🔴 S3 Bucket Permissions   → Public S3 bucket = data breach tiềm năng
🔴 Security Groups 0.0.0.0/0 → Port mở rộng rãi = attack surface lớn
🔴 CloudTrail Logging      → Không có audit log = không điều tra được sự cố
```

---

### 4. Fault Tolerance — Khả Năng Chịu Lỗi 🛡️

Phát hiện cấu hình có nguy cơ single point of failure:

| Check                                             | Vấn Đề Phát Hiện                                         |
| ------------------------------------------------- | -------------------------------------------------------- |
| Amazon RDS Multi-AZ                              | RDS chỉ có 1 AZ → nếu AZ đó lỗi thì downtime           |
| Amazon RDS Backups                               | Backup retention = 0 ngày → mất data khi lỗi           |
| EBS Snapshots Age                               | Snapshot > 7 ngày → RPO quá cao                         |
| Auto Scaling Group Resources                    | ALB/ASG cấu hình sai hoặc thiếu health check           |
| Amazon EC2 Availability Zone Balance            | Instances phân bố không đều giữa các AZ                |
| Elastic Load Balancer Optimization              | ELB không phân phối đều tải sang các AZ               |
| AWS Direct Connect Backup Connectivity          | Direct Connect không có backup connection              |
| Route 53 High TTL Records                       | TTL quá cao → failover DNS chậm khi xảy ra sự cố       |
| Amazon S3 Bucket Versioning                     | Versioning chưa bật → không rollback được nếu xóa nhầm |
| VPN Tunnel Redundancy                           | VPN chỉ có 1 tunnel → không có redundancy              |

**Kiến trúc Fault Tolerant điển hình Trusted Advisor hướng đến:**
```
                    ┌─────────────────────────────┐
                    │       Route 53              │
                    │  (Health Check + Failover)  │
                    └──────────┬──────────────────┘
                               │
               ┌───────────────┼───────────────────┐
               ▼               ▼                   ▼
         ┌──────────┐   ┌──────────┐         ┌──────────┐
         │   ALB    │   │   ALB    │         │  Region  │
         │  AZ-1a   │   │  AZ-1b   │         │  Failover│
         └────┬─────┘   └────┬─────┘         └──────────┘
              │              │
         ┌────▼─────┐   ┌────▼─────┐
         │  EC2/ECS │   │  EC2/ECS │
         │  AZ-1a   │   │  AZ-1b   │
         └──────────┘   └──────────┘
              │              │
         ┌────▼──────────────▼────┐
         │    RDS Multi-AZ        │
         │  Primary (AZ-1a)       │
         │  Standby (AZ-1b)       │
         └────────────────────────┘
```

---

### 5. Service Limits — Giới Hạn Dịch Vụ 📊

Cảnh báo khi tài nguyên sắp chạm Service Quota (Hạn Mức Dịch Vụ):

| Check                                             | Ngưỡng Cảnh Báo                               |
| ------------------------------------------------- | --------------------------------------------- |
| EC2 On-Demand Instance Limits                    | Sử dụng > 80% quota → cảnh báo Yellow        |
| VPC và Subnet Limits                             | Số VPC/Subnet gần đạt tối đa                 |
| IAM Roles/Groups/Users                          | Số IAM objects gần đạt giới hạn              |
| ELB Application Load Balancers                  | Số ALB trong region gần quota                |
| RDS DB Instances                                | Số instance gần đạt giới hạn region          |
| Auto Scaling Groups                             | Số ASG trong region gần quota                |
| CloudFormation Stacks                           | Số stack gần đạt limit                       |
| SES Sending Limits                              | Quota gửi email của SES gần đầy              |

**Lưu ý quan trọng:**
```
⚠️ Trusted Advisor kiểm tra Service LIMITS (quota cũ), không phải hiện trạng thực tế
⚠️ Để tăng quota: dùng Service Quotas console hoặc hỏi AWS Support
⚠️ Không phải tất cả quotas đều có thể tăng — một số là hard limits
```

---

## Cấp Độ Support & Số Lượng Checks

Đây là điểm **hay hỏi nhất trong phỏng vấn** về Trusted Advisor:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Trusted Advisor — Checks theo Support Plan        │
│                                                                      │
│  Basic / Developer Support                                           │
│  ┌────────────────────────────────────────────────┐                 │
│  │ ~7 checks cơ bản (chỉ Security + Service Limits)│                 │
│  │  ✅ S3 Bucket Permissions                        │                 │
│  │  ✅ Security Groups — Unrestricted Access        │                 │
│  │  ✅ MFA on Root Account                          │                 │
│  │  ✅ IAM Access Key Rotation                      │                 │
│  │  ✅ EBS Public Snapshots                         │                 │
│  │  ✅ RDS Public Snapshots                         │                 │
│  │  ✅ Service Limits (một số)                      │                 │
│  └────────────────────────────────────────────────┘                 │
│                                                                      │
│  Business / Enterprise Support                                       │
│  ┌────────────────────────────────────────────────┐                 │
│  │ Toàn bộ 200+ checks (tất cả 5 lĩnh vực)        │                 │
│  │  ✅ Tất cả Cost Optimization checks              │                 │
│  │  ✅ Tất cả Performance checks                   │                 │
│  │  ✅ Tất cả Security checks                      │                 │
│  │  ✅ Tất cả Fault Tolerance checks               │                 │
│  │  ✅ Tất cả Service Limits checks                │                 │
│  │  ✅ Access via Support API (lập trình)          │                 │
│  │  ✅ EventBridge integration                     │                 │
│  │  ✅ Refresh theo yêu cầu (on-demand)            │                 │
│  └────────────────────────────────────────────────┘                 │
└──────────────────────────────────────────────────────────────────────┘
```

| Tính Năng                          | Basic/Developer | Business/Enterprise |
| ---------------------------------- | :-------------: | :-----------------: |
| Số lượng checks                    | ~7              | 200+                |
| Cost Optimization checks           | ❌              | ✅                  |
| Performance checks                 | ❌              | ✅                  |
| Security checks (đầy đủ)           | Một phần        | ✅                  |
| Fault Tolerance checks             | ❌              | ✅                  |
| Service Limits checks (đầy đủ)     | Một phần        | ✅                  |
| Support API (lập trình)            | ❌              | ✅                  |
| EventBridge integration            | ❌              | ✅                  |
| On-demand refresh                  | ❌              | ✅                  |
| Organizational view                | ❌              | ✅ (Enterprise)     |

---

## Trusted Advisor vs Các Dịch Vụ Liên Quan

### Trusted Advisor vs AWS Security Hub

| Tiêu Chí                   | Trusted Advisor                       | AWS Security Hub                          |
| -------------------------- | ------------------------------------- | ----------------------------------------- |
| **Mục đích chính**         | Best practice tổng hợp 5 lĩnh vực    | Bảo mật và compliance tập trung           |
| **Tùy chỉnh rules**        | ❌ Không (rules cố định của AWS)      | ✅ Có (custom insights)                   |
| **Frameworks**             | Không                                 | CIS, PCI-DSS, NIST, FSBP                 |
| **Nguồn dữ liệu**          | Chỉ Trusted Advisor                   | GuardDuty, Config, Inspector, Macie...    |
| **Chi phí**                | Miễn phí với Business+ Support        | Tính phí riêng                           |
| **Multi-account**          | Từng account riêng lẻ                 | ✅ Aggregation toàn Organization          |

### Trusted Advisor vs AWS Config

| Tiêu Chí                   | Trusted Advisor                       | AWS Config                                |
| -------------------------- | ------------------------------------- | ----------------------------------------- |
| **Loại checks**            | Best practice (pre-defined)           | Compliance rules (managed + custom)       |
| **Tùy chỉnh**              | ❌ Không thể thêm rule mới            | ✅ Viết Lambda custom rules               |
| **Lịch sử cấu hình**       | ❌ Không lưu history                  | ✅ Lưu toàn bộ config history             |
| **Remediation**            | ❌ Chỉ cảnh báo                       | ✅ SSM Automation remediation             |
| **Khi nào dùng**           | Cái nhìn tổng quan nhanh             | Compliance-as-code, kiểm tra chi tiết     |

### Trusted Advisor vs AWS Well-Architected Tool

| Tiêu Chí                   | Trusted Advisor                       | Well-Architected Tool                     |
| -------------------------- | ------------------------------------- | ----------------------------------------- |
| **Phạm vi**                | Kiểm tra tài nguyên thực tế           | Review kiến trúc tổng thể (interview-based)|
| **Cách thức**              | Tự động (automated scan)              | Thủ công (Q&A với architect)             |
| **Đầu ra**                 | Danh sách vấn đề cụ thể              | Improvement plan, risk report             |
| **Tần suất**               | Liên tục (refresh tự động)            | Định kỳ (quarterly review)               |

---

## Tích Hợp & Tự Động Hóa

### EventBridge Integration — Phản Ứng Tự Động

Khi trạng thái một check thay đổi (OK → WARNING → ERROR), Trusted Advisor phát event lên EventBridge:

```json
{
  "source": "aws.trustedadvisor",
  "detail-type": "Trusted Advisor Check Item Refresh Notification",
  "detail": {
    "check-name": "Security Groups - Unrestricted Access",
    "check-item-detail": {
      "Status": "error",
      "Region": "ap-southeast-1",
      "Resource": "sg-0abc123def456",
      "Port": "22",
      "Protocol": "tcp",
      "IP Range": "0.0.0.0/0"
    },
    "status": "ERROR",
    "resource_id": "sg-0abc123def456",
    "uuid": "check-uuid-here"
  }
}
```

**Ví dụ tự động hóa với EventBridge + Lambda:**
```
Security Group mở port 22/0.0.0.0/0 được tạo
       │
       ▼
Trusted Advisor phát hiện → status: ERROR
       │
       ▼
EventBridge rule bắt event "Trusted Advisor Check Item Refresh"
       │
       ▼
Lambda function kích hoạt
       │
       ├─ Revoke inbound rule 0.0.0.0/0:22
       ├─ Gửi Slack alert
       └─ Tạo OpsItem trên SSM OpsCenter
```

### Support API — Truy Cập Lập Trình

```python
import boto3

# Yêu cầu: Business hoặc Enterprise Support plan
# Region: PHẢI dùng us-east-1 cho Support API
client = boto3.client('support', region_name='us-east-1')

# 1. Lấy danh sách tất cả checks
response = client.describe_trusted_advisor_checks(language='en')
checks = response['checks']
for check in checks:
    print(f"ID: {check['id']}, Name: {check['name']}, Category: {check['category']}")

# 2. Lấy kết quả một check cụ thể
check_id = 'Pfx0RwqBli'  # VD: MFA on Root Account
result = client.describe_trusted_advisor_check_result(
    checkId=check_id,
    language='en'
)
print(result['result']['status'])  # ok / warning / error / not_available

# 3. Refresh một check (on-demand)
client.refresh_trusted_advisor_check(checkId=check_id)

# 4. Lấy tóm tắt tất cả checks
summary = client.describe_trusted_advisor_check_summaries(
    checkIds=[check['id'] for check in checks]
)
```

---

## Giới Hạn & Lưu Ý Quan Trọng

### Giới Hạn Kỹ Thuật

```
⚠️ Support API chỉ hoạt động ở us-east-1 (endpoint toàn cầu)
⚠️ Refresh tự động: mỗi tuần một lần (không phải real-time)
⚠️ On-demand refresh: có rate limit, không thể refresh liên tục
⚠️ Không thể thêm custom checks — chỉ dùng checks do AWS cung cấp
⚠️ Kết quả check có thể trễ vài giờ sau khi tài nguyên thay đổi
```

### Phạm Vi Kiểm Tra

```
✅ Kiểm tra: EC2, RDS, S3, ELB, Route 53, IAM, Security Groups, EBS
⚠️ Kiểm tra hạn chế: EKS, ECS, Lambda (một số checks)
❌ Không kiểm tra: Cấu hình bên trong OS/application
❌ Không kiểm tra: Network traffic, application performance
```

### Lưu Ý Quan Trọng Cho Phỏng Vấn

```
1. Trusted Advisor KHÔNG thay thế được AWS Config — phạm vi khác nhau
2. Trusted Advisor KHÔNG có remediation tự động — chỉ cảnh báo
   (phải kết hợp EventBridge + Lambda để tự động hóa)
3. Business Support là mức TỐI THIỂU để dùng đầy đủ Trusted Advisor
4. Support API endpoint LUÔN phải là us-east-1 dù bạn ở region nào
5. Trusted Advisor kiểm tra account-level, không phải resource-level chi tiết
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Trusted Advisor kiểm tra những lĩnh vực nào?

**Trả lời:**
Trusted Advisor kiểm tra 5 lĩnh vực:
1. **Cost Optimization** (Tối Ưu Chi Phí) — Idle resources, unused EIPs, RI optimization
2. **Performance** (Hiệu Suất) — High CPU utilization, EBS IOPS, CloudFront headers
3. **Security** (Bảo Mật) — MFA, public S3, security group 0.0.0.0/0, CloudTrail
4. **Fault Tolerance** (Khả Năng Chịu Lỗi) — RDS Multi-AZ, EBS snapshots, AZ balance
5. **Service Limits** (Giới Hạn Dịch Vụ) — Cảnh báo khi > 80% quota

### Q2: Cần Support Plan gì để dùng đầy đủ Trusted Advisor?

**Trả lời:**
- **Basic/Developer**: ~7 checks cơ bản (Security + một phần Service Limits)
- **Business/Enterprise**: Toàn bộ 200+ checks + Support API + EventBridge integration + on-demand refresh

### Q3: Trusted Advisor khác gì AWS Config?

**Trả lời:**
- **Trusted Advisor**: Checks pre-defined theo AWS best practice, không tùy chỉnh được, không có history, chỉ cảnh báo
- **AWS Config**: Rules tùy chỉnh được (Managed + Custom Lambda), lưu configuration history, hỗ trợ SSM Automation remediation tự động

### Q4: Làm sao tự động hóa phản ứng khi Trusted Advisor phát hiện vấn đề?

**Trả lời:**
Kết hợp EventBridge + Lambda:
1. Trusted Advisor phát event lên EventBridge khi check status thay đổi
2. Tạo EventBridge Rule bắt event `source: aws.trustedadvisor`
3. Trigger Lambda function để tự xử lý (revoke security group rule, gửi alert, tạo ticket...)

### Q5: Tại sao Support API của Trusted Advisor phải gọi ở us-east-1?

**Trả lời:**
AWS Support API là global endpoint đặt tại `support.us-east-1.amazonaws.com`. Dù tài nguyên của bạn ở bất kỳ region nào, Support API luôn được gọi qua endpoint us-east-1 này. Đây là thiết kế của AWS, không phải giới hạn có thể thay đổi.

---

## Điều Hướng Module

| File                          | Nội Dung                                                          |
| ----------------------------- | ----------------------------------------------------------------- |
| **README.md** (file này)      | Tổng quan Trusted Advisor, kiến trúc, so sánh dịch vụ            |
| **1-check-categories.md**     | Chi tiết 5 categories: Cost, Performance, Security, FT, Limits   |
| **2-programmatic-access.md**  | Support API, EventBridge integration, automation patterns         |

| Điều Hướng                    | Link                                                              |
| ----------------------------- | ----------------------------------------------------------------- |
| ← Trước: Control Tower        | [07-control-tower/README.md](../07-control-tower/README.md)       |
| → Tiếp: Health Dashboard      | [09-health-dashboard/README.md](../09-health-dashboard/README.md) |
| ↑ Chỉ Mục                     | [INDEX.md](../INDEX.md)                                           |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
