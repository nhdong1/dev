# 🔧 AWS Compute — Xử Lý Sự Cố (Troubleshooting)

> Hướng dẫn chẩn đoán và khắc phục sự cố có hệ thống cho EC2, Auto Scaling, Lambda, và Container (ECS/EKS) — bao gồm phương pháp tiếp cận, công cụ chẩn đoán, và checklist trước khi go-live.

## 📚 Mục Lục

1. [Triết Lý Xử Lý Sự Cố](#triết-lý-xử-lý-sự-cố)
2. [Công Cụ Chẩn Đoán Chính](#công-cụ-chẩn-đoán-chính)
3. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
4. [Ma Trận Sự Cố Phổ Biến](#ma-trận-sự-cố-phổ-biến)
5. [Quy Trình Triage Nhanh](#quy-trình-triage-nhanh)

---

## 🎯 Triết Lý Xử Lý Sự Cố

### Nguyên Tắc Cốt Lõi

Xử lý sự cố hiệu quả không phải là may mắn — đó là **phương pháp có hệ thống**:

```
1. QUAN SÁT   → Thu thập triệu chứng, không phán đoán ngay
2. GIẢI THUYẾT → Đưa ra giả thuyết dựa trên bằng chứng
3. KIỂM TRA   → Kiểm chứng từng giả thuyết một cách có kiểm soát
4. KHẮC PHỤC  → Áp dụng giải pháp và xác nhận hiệu quả
5. PHÒNG NGỪA → Root cause analysis, cải thiện để không tái diễn
```

### Phân Loại Mức Độ Sự Cố

| Mức Độ | Ký Hiệu | Mô Tả | Thời Gian Phản Hồi |
|--------|---------|-------|-------------------|
| **P0 — Critical** (Nghiêm Trọng) | 🔴 | Production down hoàn toàn | < 15 phút |
| **P1 — High** (Cao) | 🟠 | Tính năng core bị ảnh hưởng | < 1 giờ |
| **P2 — Medium** (Trung Bình) | 🟡 | Degraded performance (Hiệu Suất Giảm) | < 4 giờ |
| **P3 — Low** (Thấp) | 🟢 | Ảnh hưởng nhỏ, có workaround | < 24 giờ |

---

## 🛠️ Công Cụ Chẩn Đoán Chính

### AWS Console & CLI

```bash
# EC2 — Xem trạng thái instance
aws ec2 describe-instance-status --instance-ids i-1234567890abcdef0

# EC2 — Xem System Log (Nhật Ký Hệ Thống) để chẩn đoán boot failures
aws ec2 get-console-output --instance-id i-1234567890abcdef0

# Auto Scaling — Xem activity history (Lịch Sử Hoạt Động)
aws autoscaling describe-scaling-activities --auto-scaling-group-name my-asg

# Lambda — Xem recent invocations (Các Lần Gọi Gần Đây)
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '1 hour ago' +%s000)

# ECS — Xem stopped tasks và lý do dừng
aws ecs describe-tasks --cluster my-cluster --tasks TASK_ARN
```

### CloudWatch — Điểm Khởi Đầu Chẩn Đoán

```bash
# Xem metrics EC2 trong 1 giờ qua
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# CloudWatch Log Insights — Tìm lỗi trong Lambda
aws logs start-query \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'filter @message like /ERROR/ | stats count(*) by bin(5m)'
```

### AWS Systems Manager — SSM (Quản Lý Hệ Thống)

```bash
# Kết nối vào EC2 không cần SSH (không cần mở port 22)
aws ssm start-session --target i-1234567890abcdef0

# Chạy lệnh trên nhiều EC2 cùng lúc (Run Command)
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=production" \
  --parameters 'commands=["df -h", "free -m", "top -bn1 | head -5"]'
```

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📄 [1-ec2-issues.md](1-ec2-issues.md) — Sự Cố EC2

- **SSH/RDP Connection Failures** — Không thể kết nối vào instance
- **EC2 Status Checks** — System check vs Instance check failures
- **Instance Recovery** — Tự động phục hồi EC2 khi có sự cố phần cứng
- **Boot Failures** — Instance không khởi động được
- **Disk Space & Performance** — Đầy ổ đĩa, CPU/Memory bottleneck

### 📄 [2-auto-scaling-issues.md](2-auto-scaling-issues.md) — Sự Cố Auto Scaling

- **Launch Failures** — Instance không thể khởi chạy trong ASG
- **Scaling Imbalance** — Phân phối không đều giữa các AZ
- **Cooldown Issues** — Scaling quá nhanh hoặc quá chậm
- **Capacity Problems** — Không đạt desired capacity
- **Health Check Failures** — Instance bị terminate nhầm

### 📄 [3-lambda-debugging.md](3-lambda-debugging.md) — Gỡ Lỗi Lambda

- **Timeout Errors** — Function chạy quá thời gian giới hạn
- **Out of Memory — OOM** — Hết bộ nhớ trong quá trình thực thi
- **Permission Errors** — IAM role thiếu quyền cần thiết
- **Cold Start Latency** — Độ trễ khi khởi động lạnh
- **Throttling** — Bị giới hạn số lần gọi đồng thời

### 📄 [4-container-issues.md](4-container-issues.md) — Sự Cố Container

- **ECS Task Failures** — Task không thể khởi động hoặc bị crash
- **OOM Killed** — Container bị hệ thống tắt do hết bộ nhớ
- **Image Pull Failures** — Không thể tải container image
- **Networking Issues** — Container không kết nối được với nhau
- **EKS Pod Issues** — Pod ở trạng thái Pending/CrashLoopBackOff

### 📄 [5-production-checklist.md](5-production-checklist.md) — Checklist Trước Go-Live

- **Pre-deployment Checklist** — Kiểm tra trước khi triển khai
- **Security Validation** — Xác nhận cấu hình bảo mật
- **Monitoring Setup** — Đảm bảo observability đầy đủ
- **Rollback Plan** — Kế hoạch quay lại phiên bản trước
- **Post-deployment Validation** — Xác nhận sau khi triển khai

---

## 🗺️ Ma Trận Sự Cố Phổ Biến

| Triệu Chứng | Dịch Vụ | Nguyên Nhân Phổ Biến Nhất | File Tham Khảo |
|-------------|---------|--------------------------|----------------|
| Không SSH được vào EC2 | EC2 | Security Group chặn port 22 | [1-ec2-issues.md](1-ec2-issues.md) |
| Instance bị terminate liên tục | Auto Scaling | Health check fail, bootstrap lỗi | [2-auto-scaling-issues.md](2-auto-scaling-issues.md) |
| Lambda timeout mọi lần gọi | Lambda | Kết nối VPC không có NAT Gateway | [3-lambda-debugging.md](3-lambda-debugging.md) |
| ECS task exit code 137 | ECS | OOM — Container hết bộ nhớ | [4-container-issues.md](4-container-issues.md) |
| ASG không scale out khi tải cao | Auto Scaling | Cooldown period còn hiệu lực | [2-auto-scaling-issues.md](2-auto-scaling-issues.md) |
| Lambda cold start > 5 giây | Lambda | VPC Lambda không có Provisioned Concurrency | [3-lambda-debugging.md](3-lambda-debugging.md) |
| ECS task ở trạng thái PENDING mãi | ECS | Thiếu tài nguyên CPU/Memory trong cluster | [4-container-issues.md](4-container-issues.md) |
| EC2 2/2 status checks fail | EC2 | Lỗi phần cứng phía AWS | [1-ec2-issues.md](1-ec2-issues.md) |

---

## ⚡ Quy Trình Triage Nhanh

### Khi Production Down (Sản Xuất Ngừng Hoạt Động)

```
BƯỚC 1 — XÁC ĐỊNH PHẠM VI (2-3 phút)
├── Tất cả user bị ảnh hưởng hay một nhóm?
├── Tất cả region hay một region cụ thể?
└── Bắt đầu từ khi nào? Có deployment gần đây không?

BƯỚC 2 — KIỂM TRA NHANH AWS HEALTH DASHBOARD (1 phút)
├── https://health.aws.amazon.com — Sự cố từ phía AWS?
└── AWS Service Health Dashboard cho từng region

BƯỚC 3 — KIỂM TRA MONITORING (3-5 phút)
├── CloudWatch Dashboard — Metrics bất thường?
├── CloudWatch Alarms — Alarm nào đang ALARM state?
└── Recent deployments — CodeDeploy, ECS update, Lambda version?

BƯỚC 4 — CHẨN ĐOÁN THEO DỊCH VỤ
├── EC2 → Xem Status Checks trong AWS Console
├── Lambda → Xem CloudWatch Logs, check throttling metrics
├── ECS → Xem stopped tasks, describe-tasks để xem stopReason
└── Auto Scaling → Xem scaling activities, launch failures

BƯỚC 5 — ROLLBACK NẾU CẦN (Luôn có sẵn plan này)
├── Lambda → Alias pointing to previous version
├── ECS → Force new deployment với task definition cũ
├── EC2/ASG → Launch template version rollback
└── CodeDeploy → Automatic rollback on alarm
```

### Checklist 5 Phút Khi Nhận Alert

```
□ Đọc alert message đầy đủ — metric nào, threshold nào?
□ Xem CloudWatch graph 1 giờ qua — trend như thế nào?
□ Kiểm tra có deployment nào trong 30 phút qua không?
□ Xem error logs — có pattern lặp lại không?
□ Check downstream dependencies — database, external API?
□ Xác định blast radius — bao nhiêu % user bị ảnh hưởng?
□ Escalate hoặc bắt đầu fix dựa trên severity
```

---

## 📖 Tham Khảo Nhanh — Exit Codes Phổ Biến

| Exit Code | Ý Nghĩa | Nguyên Nhân Thường Gặp |
|-----------|---------|----------------------|
| `0` | Thành công | — |
| `1` | Lỗi chung | Script/application error |
| `137` | OOM Killed | Container hết memory limit |
| `139` | Segmentation Fault | Memory corruption trong application |
| `143` | SIGTERM — Graceful shutdown | Container bị stop bình thường |
| `255` | Exit Unknown | SSH/entrypoint không tìm thấy |

---

## 🔗 Điều Hướng

| Chủ Đề | File |
|--------|------|
| Sự cố EC2 | [1-ec2-issues.md](1-ec2-issues.md) |
| Sự cố Auto Scaling | [2-auto-scaling-issues.md](2-auto-scaling-issues.md) |
| Gỡ lỗi Lambda | [3-lambda-debugging.md](3-lambda-debugging.md) |
| Sự cố Container (ECS/EKS) | [4-container-issues.md](4-container-issues.md) |
| Checklist Trước Go-Live | [5-production-checklist.md](5-production-checklist.md) |
| Quay lại Tổng Quan | [../README.md](../README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
