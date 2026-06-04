# SSM Patch Manager — Vá Lỗi Tự Động Cho Fleet EC2

> **Patch Manager** (Trình Quản Lý Vá Lỗi) là tính năng của AWS Systems Manager tự động hóa quá trình vá lỗi (patching) hệ điều hành và ứng dụng trên EC2 instances và on-premises servers — theo lịch, theo nhóm, theo baseline đã định sẵn.

---

## 📚 Mục Lục

1. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
2. [Patch Baseline — Đường Cơ Sở Vá Lỗi](#patch-baseline--đường-cơ-sở-vá-lỗi)
3. [Patch Groups — Nhóm Vá Lỗi](#patch-groups--nhóm-vá-lỗi)
4. [Maintenance Windows — Cửa Sổ Bảo Trì](#maintenance-windows--cửa-sổ-bảo-trì)
5. [Quy Trình Patching Hoàn Chỉnh](#quy-trình-patching-hoàn-chỉnh)
6. [Báo Cáo Compliance](#báo-cáo-compliance)
7. [Tình Huống Thực Tế](#tình-huống-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm Cốt Lõi

### Tam Giác Vá Lỗi SSM

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    Patch Manager                            │
│                                                             │
│         ┌──────────────────────────────────┐               │
│         │         Patch Baseline           │               │
│         │  "Quy tắc nào được phép patch"   │               │
│         └──────────────┬───────────────────┘               │
│                        │                                   │
│         ┌──────────────▼───────────────────┐               │
│         │          Patch Group             │               │
│         │  "Nhóm instance nào áp dụng"    │               │
│         └──────────────┬───────────────────┘               │
│                        │                                   │
│         ┌──────────────▼───────────────────┐               │
│         │      Maintenance Window          │               │
│         │  "Khi nào chạy, bao lâu"         │               │
│         └──────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Luồng Patching Tổng Quan

```
Scheduled Trigger
      │
      ▼
Maintenance Window bắt đầu
      │
      ▼
Chọn instances theo Patch Group tag
      │
      ▼
Áp dụng Patch Baseline để lọc patches
      │
      ▼
SSM Agent tải và cài patches từ OS repository
      │
      ▼
Reboot (nếu cần) theo cấu hình
      │
      ▼
Ghi kết quả vào SSM Compliance + S3
      │
      ▼
Maintenance Window kết thúc
```

---

## Patch Baseline — Đường Cơ Sở Vá Lỗi

### Patch Baseline Là Gì?

**Patch Baseline** (Đường Cơ Sở Vá Lỗi) định nghĩa tập hợp các patches được **tự động approve** (chấp thuận tự động) và các patches bị **reject** (từ chối). Đây là "bộ lọc" quyết định patch nào sẽ được cài.

### AWS-Managed Patch Baselines (Sẵn Có)

| Baseline | OS | Mô Tả |
|----------|----|--------|
| `AWS-AmazonLinux2DefaultPatchBaseline` | Amazon Linux 2 | Critical + Important, auto-approve sau 7 ngày |
| `AWS-AmazonLinux2023DefaultPatchBaseline` | Amazon Linux 2023 | Critical + Important, auto-approve sau 7 ngày |
| `AWS-UbuntuDefaultPatchBaseline` | Ubuntu | Critical + Important |
| `AWS-WindowsServerDefaultPatchBaseline` | Windows | Critical + Important |
| `AWS-CentOSDefaultPatchBaseline` | CentOS | Critical + Important |

### Tạo Custom Patch Baseline

```bash
aws ssm create-patch-baseline \
  --name "MyProductionPatchBaseline" \
  --description "Production servers - Critical only, 14-day delay" \
  --operating-system AMAZON_LINUX_2 \
  --approval-rules '{
    "PatchRules": [
      {
        "PatchFilterGroup": {
          "PatchFilters": [
            {
              "Key": "SEVERITY",
              "Values": ["Critical", "Important"]
            },
            {
              "Key": "CLASSIFICATION",
              "Values": ["Security", "Bugfix"]
            }
          ]
        },
        "ApproveAfterDays": 14,
        "ComplianceLevel": "CRITICAL",
        "EnableNonSecurity": false
      }
    ]
  }' \
  --rejected-patches "kernel*" \
  --rejected-patches-action BLOCK \
  --tags "Key=Environment,Value=Production"
```

### Các Thuộc Tính Patch Baseline

**Classification (Phân Loại Patch):**
```
Security        — Bản vá bảo mật (security fix)
Bugfix          — Sửa lỗi thông thường
Enhancement     — Cải tiến tính năng
Recommended     — AWS khuyến nghị
Newpackage      — Package mới
```

**Severity (Mức Độ Nghiêm Trọng):**
```
Critical        — Phải vá ngay lập tức (CVE có CVSS ≥ 9.0)
Important       — Vá trong vòng vài ngày (CVSS 7.0–8.9)
Medium          — Vá theo lịch bình thường (CVSS 4.0–6.9)
Low             — Vá khi tiện (CVSS < 4.0)
Unspecified     — Chưa phân loại
```

**ApproveAfterDays:** Số ngày chờ sau khi patch được release trước khi tự động approve.

---

## Patch Groups — Nhóm Vá Lỗi

### Patch Group Là Gì?

**Patch Group** (Nhóm Vá Lỗi) là cơ chế phân nhóm EC2 instances bằng **Tag** để áp dụng Patch Baseline khác nhau cho từng nhóm. Một instance chỉ thuộc một Patch Group tại một thời điểm.

### Thiết Lập Patch Group

```bash
# Bước 1: Gắn tag "Patch Group" vào EC2 instances
# Tên tag bắt buộc là "Patch Group" (có khoảng trắng)
aws ec2 create-tags \
  --resources i-1234567890abcdef0 \
  --tags Key="Patch Group",Value="production-web-servers"

# Bước 2: Đăng ký Patch Group vào Patch Baseline
aws ssm register-patch-baseline-for-patch-group \
  --baseline-id pb-0abc123def456789 \
  --patch-group "production-web-servers"
```

### Chiến Lược Patch Group Theo Môi Trường

```
Ví dụ phân nhóm cho công ty:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Patch Group: "dev-servers"                                 │
│    → Baseline: Approve all patches immediately              │
│    → Maintenance Window: Thứ 2, 09:00 AM                   │
│                                                             │
│  Patch Group: "staging-servers"                             │
│    → Baseline: Critical + Important, approve 7 ngày         │
│    → Maintenance Window: Thứ 3, 02:00 AM                   │
│                                                             │
│  Patch Group: "production-web"                              │
│    → Baseline: Critical only, approve 14 ngày               │
│    → Maintenance Window: Thứ 7, 01:00 AM (dùng wave)       │
│                                                             │
│  Patch Group: "production-database"                         │
│    → Baseline: Critical only, approve 21 ngày               │
│    → Maintenance Window: Chủ nhật, 01:00 AM (manual approve)│
└─────────────────────────────────────────────────────────────┘
```

---

## Maintenance Windows — Cửa Sổ Bảo Trì

### Maintenance Window Là Gì?

**Maintenance Window** (Cửa Sổ Bảo Trì) định nghĩa **thời điểm** và **thời lượng** cho phép chạy các tác vụ vận hành (patching, automation, run command). Giúp đảm bảo patching không xảy ra trong giờ cao điểm.

### Tạo Maintenance Window

```bash
# Tạo Maintenance Window chạy Thứ 7 lúc 01:00 AM UTC, mỗi 2 tuần
aws ssm create-maintenance-window \
  --name "ProdPatching-BiWeekly-Saturday" \
  --description "Production patching every 2 weeks on Saturday" \
  --schedule "cron(0 1 ? * SAT#1 *)" \
  --duration 4 \
  --cutoff 1 \
  --allow-unassociated-targets false

# schedule: Cron expression (giờ UTC)
# duration: Thời lượng tối đa (giờ)
# cutoff: Ngừng bắt đầu task mới khi còn X giờ trước khi đóng window
```

### Cú Pháp Cron Phổ Biến

```
cron(phút giờ ngày tháng ngàyTrongTuần năm)

Ví dụ:
cron(0 2 * * ? *)           = Mỗi ngày lúc 02:00 AM UTC
cron(0 2 ? * SUN *)         = Mỗi Chủ Nhật lúc 02:00 AM UTC  
cron(0 2 ? * 7#1 *)         = Thứ 7 đầu tiên mỗi tháng
cron(0 2 1/7 * ? *)         = Mỗi 7 ngày, bắt đầu ngày 1 tháng

Rate expression:
rate(7 days)                = Mỗi 7 ngày
rate(14 days)               = Mỗi 14 ngày
```

### Thêm Targets Vào Maintenance Window

```bash
# Đăng ký Patch Group làm target
aws ssm register-target-with-maintenance-window \
  --window-id mw-0abc123def456789 \
  --resource-type INSTANCE \
  --targets "Key=tag:Patch Group,Values=production-web-servers" \
  --name "ProductionWebServers" \
  --description "All production web server instances"
```

### Thêm Task Patching Vào Maintenance Window

```bash
aws ssm register-task-with-maintenance-window \
  --window-id mw-0abc123def456789 \
  --targets "Key=WindowTargetIds,Values=<target-id>" \
  --task-arn "arn:aws:ssm:ap-southeast-1::document/AWS-RunPatchBaseline" \
  --task-type RUN_COMMAND \
  --max-concurrency "20%" \
  --max-errors "5%" \
  --priority 1 \
  --task-invocation-parameters '{
    "RunCommand": {
      "Parameters": {
        "Operation": ["Install"],
        "RebootOption": ["RebootIfNeeded"]
      },
      "CloudWatchOutputConfig": {
        "CloudWatchOutputEnabled": true,
        "CloudWatchLogGroupName": "/ssm/patch-manager/production"
      }
    }
  }'
```

### Tham Số Quan Trọng

| Tham Số | Giá Trị | Ý Nghĩa |
|---------|---------|---------|
| `Operation` | `Scan` | Chỉ quét, không cài patch |
| `Operation` | `Install` | Quét và cài patch |
| `RebootOption` | `RebootIfNeeded` | Reboot nếu patch yêu cầu |
| `RebootOption` | `NoReboot` | Không reboot (cần reboot thủ công sau) |
| `max-concurrency` | `20%` | Chỉ patch 20% instances cùng lúc (rolling) |
| `max-errors` | `5%` | Dừng nếu hơn 5% instances lỗi |

---

## Quy Trình Patching Hoàn Chỉnh

### Chiến Lược Rolling Patch (Vá Lỗi Cuốn Chiếu)

```
Wave 1 (10% instances): Patch → Kiểm tra → Xác nhận OK
        ↓
Wave 2 (20% instances): Patch → Kiểm tra → Xác nhận OK
        ↓
Wave 3 (70% instances): Patch → Kiểm tra → Done
```

### Tích Hợp Với Auto Scaling Group

```bash
# Thay vì patch in-place (vá tại chỗ), dùng chiến lược replace:
# 1. Tạo AMI mới đã patch
# 2. Update Launch Template
# 3. Trigger Rolling Update trên ASG

# Automation document để tạo patched AMI
aws ssm create-document \
  --name "CreatePatchedAMI" \
  --document-type Automation \
  --content '{
    "schemaVersion": "0.3",
    "mainSteps": [
      {
        "name": "runPatchBaseline",
        "action": "aws:runCommand",
        "inputs": {
          "DocumentName": "AWS-RunPatchBaseline",
          "Parameters": {"Operation": ["Install"]}
        }
      },
      {
        "name": "createImage",
        "action": "aws:createImage",
        "inputs": {
          "InstanceId": "{{ InstanceId }}",
          "ImageName": "PatchedAMI-{{ global:DATE }}"
        }
      }
    ]
  }'
```

---

## Báo Cáo Compliance

### Xem Trạng Thái Patch Compliance

```bash
# Xem instances non-compliant (thiếu critical patches)
aws ssm list-compliance-summaries \
  --filters "Key=ComplianceType,Values=Patch" \
  --query 'ComplianceSummaryItems[?Status==`NON_COMPLIANT`]'

# Xem chi tiết patch missing trên một instance
aws ssm list-compliance-items \
  --resource-ids i-1234567890abcdef0 \
  --resource-types ManagedInstance \
  --filters "Key=ComplianceType,Values=Patch" "Key=Status,Values=NON_COMPLIANT"

# Xem patch summary toàn fleet
aws ssm describe-instance-patch-states-for-patch-group \
  --patch-group "production-web-servers" \
  --query 'InstancePatchStates[*].[InstanceId,MissingCount,CriticalNonCompliantCount,LastNoRebootInstallOperationTime]' \
  --output table
```

### Dashboard Compliance Trong Console

```
Systems Manager → Patch Manager → Dashboard
→ Xem: Compliance Summary, Patch by State, Critical Patches Missing
```

### Tích Hợp Với AWS Config

```
AWS Config Rule: ec2-managedinstance-patch-compliance-status-check
→ Tự động đánh dấu NON_COMPLIANT nếu instance thiếu patches
→ Tích hợp với Security Hub để báo cáo tập trung
```

---

## Tình Huống Thực Tế

### Tình Huống 1: Emergency Patching — CVE Nghiêm Trọng

```bash
# Log4Shell (CVE-2021-44228) cần patch ngay lập tức

# Bước 1: Quét tất cả instances để tìm affected
aws ssm send-command \
  --document-name "AWS-RunPatchBaseline" \
  --targets "Key=tag:Environment,Values=production,staging,dev" \
  --parameters '{"Operation":["Scan"]}' \
  --comment "Emergency scan for Log4Shell CVE-2021-44228"

# Bước 2: Patch dev ngay
aws ssm send-command \
  --document-name "AWS-RunPatchBaseline" \
  --targets "Key=tag:Environment,Values=dev" \
  --parameters '{"Operation":["Install"],"RebootOption":["RebootIfNeeded"]}'

# Bước 3: Sau khi dev OK, patch staging, sau đó prod
# (Dùng max-concurrency thấp cho prod)
aws ssm send-command \
  --document-name "AWS-RunPatchBaseline" \
  --targets "Key=tag:Patch Group,Values=production-web-servers" \
  --max-concurrency "10%" \
  --max-errors "2%" \
  --parameters '{"Operation":["Install"],"RebootOption":["RebootIfNeeded"]}'
```

### Tình Huống 2: Patch Validation Sau Khi Cài

```bash
# Chạy health check sau patch
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Patch Group,Values=production-web-servers" \
  --parameters '{
    "commands": [
      "systemctl is-active nginx || exit 1",
      "systemctl is-active myapp || exit 1",
      "curl -sf http://localhost/health || exit 1",
      "echo VALIDATION_PASSED"
    ]
  }'
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Patch Manager gồm những thành phần chính nào?**

> Ba thành phần chính: (1) **Patch Baseline** — định nghĩa patch nào được chấp thuận (approve) dựa trên severity, classification, và tuổi của patch; (2) **Patch Group** — phân nhóm instances bằng tag để áp dụng baseline phù hợp; (3) **Maintenance Window** — lên lịch thời điểm và thời lượng thực hiện patching.

**Q: Sự khác biệt giữa Scan và Install trong Patch Manager?**

> `Scan` (Quét) chỉ kiểm tra và báo cáo xem instance có missing patches không — không cài gì. `Install` (Cài Đặt) thực sự tải và cài patches theo Patch Baseline đã cấu hình. Workflow chuẩn: chạy Scan trước để biết scope, sau đó chạy Install trong Maintenance Window.

**Q: Một instance có thể thuộc nhiều Patch Group không?**

> Không. Mỗi instance chỉ thuộc một Patch Group tại một thời điểm (dựa trên tag "Patch Group"). Nếu instance không có tag này, nó dùng Default Patch Baseline của OS đó.

### Nâng Cao

**Q: Thiết kế chiến lược patching zero-downtime cho fleet 500 production web servers?**

> **(1) Patch Group & Baseline:** Tạo baseline chỉ approve Critical patches sau 14 ngày. Chia servers thành Patch Groups: `prod-wave-1` (10%), `prod-wave-2` (30%), `prod-wave-3` (60%).
>
> **(2) Maintenance Window:** Cấu hình chạy Thứ 7 lúc 02:00 AM với duration 4 giờ. `max-concurrency=10%`, `max-errors=5%`.
>
> **(3) Tích hợp Load Balancer:** Trước khi patch, drain connections qua AWS-UpdateLinuxAmi Automation document. Sau patch, kiểm tra health trước khi đưa trở lại LB.
>
> **(4) Thay thế (Blue/Green):** Với ASG, tạo AMI đã patch, update Launch Template, trigger Instance Refresh với `MinHealthyPercentage=80%` — không cần patch in-place.
>
> **(5) Báo cáo:** Sau mỗi wave, xem Patch Compliance dashboard. Nếu CriticalNonCompliantCount > 0, alert on-call team.

**Q: ApproveAfterDays có tác dụng gì với emergency patching?**

> `ApproveAfterDays` ngăn cài patch ngay khi được release — cho thời gian để AWS và cộng đồng phát hiện regression (lỗi do patch mới). Nhưng với emergency CVE như Log4Shell hay PrintNightmare, không thể chờ. Giải pháp: (1) Tạo Approved Patches list thủ công để override `ApproveAfterDays`; hoặc (2) Tạm thời tạo custom baseline với `ApproveAfterDays=0` chỉ cho CVE cụ thể, sau khi patch xong thì rollback về baseline cũ.

---

**Liên Quan:** [Session Manager](./1-session-manager.md) | [Parameter Store](./3-parameter-store.md) | [Run Command & Automation](./4-run-command-automation.md)
