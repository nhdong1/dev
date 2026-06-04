# Health Checks & Recovery — Kiểm Tra Sức Khỏe & Tự Động Phục Hồi

> Health Checks (Kiểm Tra Sức Khỏe) là cơ chế phát hiện instance hoặc service không hoạt động bình thường. Auto Recovery (Tự Động Phục Hồi) tự động thay thế hoặc khởi động lại instance lỗi — không cần người vận hành can thiệp thủ công.

## 📚 Mục Lục

1. [EC2 Status Checks](#ec2-status-checks)
2. [ELB Health Checks](#elb-health-checks)
3. [ASG Health Check Integration](#asg-health-checks)
4. [ECS Health Checks](#ecs-health-checks)
5. [EKS / Kubernetes Probes](#kubernetes-probes)
6. [Auto Recovery Patterns](#auto-recovery)
7. [Circuit Breaker Pattern](#circuit-breaker)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🖥️ EC2 Status Checks {#ec2-status-checks}

AWS tự động chạy hai loại kiểm tra mỗi phút cho mỗi EC2 instance.

### System Status Check (Kiểm Tra Trạng Thái Hệ Thống)

Kiểm tra **hạ tầng AWS** bên dưới instance. Nếu lỗi → phần cứng máy chủ hoặc mạng của AWS bị vấn đề.

**Nguyên nhân lỗi thường gặp:**
- Lỗi phần cứng máy chủ vật lý
- Sự cố mạng ảnh hưởng đến instance
- Nguồn điện của host server
- Vấn đề với hypervisor

**Hành động khắc phục:** Stop + Start instance (chuyển sang host server khác).

### Instance Status Check (Kiểm Tra Trạng Thái Instance)

Kiểm tra **phần mềm và cấu hình** bên trong instance. Nếu lỗi → OS hoặc ứng dụng bên trong có vấn đề.

**Nguyên nhân lỗi thường gặp:**
- Kernel crash (hệ điều hành bị treo)
- Network misconfiguration trong OS
- File system bị corrupt
- Memory exhaustion (hết RAM)

**Hành động khắc phục:** Reboot instance hoặc khởi động lại service.

### CloudWatch Metrics cho Status Checks

```bash
# Xem status check metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name StatusCheckFailed \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2026-05-15T00:00:00Z \
  --end-time 2026-05-15T01:00:00Z \
  --period 60 \
  --statistics Maximum
```

**Metrics quan trọng:**

| Metric                          | Giá Trị | Ý Nghĩa                            |
| ------------------------------- | ------- | ---------------------------------- |
| `StatusCheckFailed`             | 0 / 1   | 0 = OK, 1 = Lỗi (bất kỳ loại nào)|
| `StatusCheckFailed_System`      | 0 / 1   | Lỗi hạ tầng AWS                   |
| `StatusCheckFailed_Instance`    | 0 / 1   | Lỗi OS / phần mềm                 |

### Tự Động Khởi Động Lại với CloudWatch Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name ec2-auto-recover \
  --metric-name StatusCheckFailed_System \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions \
    arn:aws:automate:ap-southeast-1:ec2:recover \
    arn:aws:sns:ap-southeast-1:123456789:ops-alert
```

**EC2 Recover Action** giữ nguyên:
- Instance ID
- Elastic IP (Địa Chỉ IP Cố Định)
- EBS volumes
- Placement Groups
- Chỉ chuyển sang host server vật lý mới

---

## ⚖️ ELB Health Checks {#elb-health-checks}

ELB (Elastic Load Balancer — Cân Bằng Tải Đàn Hồi) định kỳ kiểm tra instance/target. Instance unhealthy bị loại khỏi pool, không nhận traffic cho đến khi healthy trở lại.

### Cấu Hình Health Check

```bash
aws elbv2 create-target-group \
  --name my-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-12345 \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30 \     # Kiểm tra mỗi 30 giây
  --health-check-timeout-seconds 5 \       # Timeout sau 5 giây không phản hồi
  --healthy-threshold-count 2 \            # Cần 2 lần pass liên tiếp → healthy
  --unhealthy-threshold-count 3            # Cần 3 lần fail liên tiếp → unhealthy
```

### Trạng Thái Target

```
initial   → Mới đăng ký, đang chờ health check đầu tiên
healthy   → Health check pass đủ số lần threshold
unhealthy → Health check fail đủ số lần threshold
draining  → Đang được deregister, đợi active connections hoàn thành
unused    → Target group chưa được attach vào listener rule
```

### Health Check Endpoint Tốt Là Gì?

```python
# Endpoint /health không nên:
# ❌ Gọi database để "đo" kết nối (quá chậm)
# ❌ Gọi external APIs (phụ thuộc ngoài)
# ❌ Thực hiện heavy computation

# Endpoint /health nên:
# ✅ Phản hồi nhanh (<1 giây)
# ✅ Kiểm tra internal state của app
# ✅ Trả về HTTP 200 khi healthy, 5xx khi unhealthy

@app.route('/health')
def health():
    # Kiểm tra internal components
    checks = {
        "status": "healthy",
        "database": check_db_connection(),    # Thử ping database
        "cache": check_cache_connection(),    # Thử ping Redis
        "version": "1.2.3"
    }
    
    if all(v != "error" for v in checks.values()):
        return jsonify(checks), 200
    else:
        return jsonify(checks), 503
```

### ALB vs NLB Health Check

| Tính Năng              | ALB (Application LB)           | NLB (Network LB)                |
| ---------------------- | ------------------------------- | -------------------------------- |
| Protocol               | HTTP/HTTPS                      | TCP, HTTP, HTTPS                 |
| Kiểm tra path          | Có (/health)                    | TCP: không (chỉ kết nối)        |
| Hiểu HTTP codes        | Có (2xx, 3xx)                   | Chỉ TCP handshake                |
| Phù hợp cho            | Web apps, APIs                  | Non-HTTP, TCP services           |

---

## 🔄 ASG Health Check Integration {#asg-health-checks}

ASG (Auto Scaling Group) có thể dùng **EC2 health check** hoặc **ELB health check** để quyết định thay thế instance.

### Hai Loại Health Check ASG

**EC2 (mặc định):**
- Chỉ kiểm tra EC2 status checks
- Instance bị thay thế chỉ khi EC2 status check lỗi
- Không phát hiện: app crash, 500 errors, memory leak

**ELB (khuyến nghị cho web apps):**
- Dùng kết quả health check từ ELB
- Instance bị thay thế khi ELB đánh dấu là unhealthy
- Phát hiện: app không phản hồi, trả về 5xx liên tục

```bash
# Cấu hình ASG dùng ELB health check
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --health-check-type ELB \
  --health-check-grace-period 300   # Không kiểm tra 300 giây sau launch
```

### Health Check Grace Period (Giai Đoạn Ân Hạn)

```
Instance mới được launch
        │
        ▼
Health Check Grace Period (mặc định 300 giây)
        │  Trong thời gian này, ASG bỏ qua health check failures
        │  Cho phép app khởi động, warmup, kết nối DB...
        ▼
Bắt Đầu Health Check Thực Sự
        │
        ▼
Nếu unhealthy → Terminate & Launch instance mới
```

**Điều Chỉnh Grace Period:**
- App khởi động nhanh (< 30s): Set 60-120 giây
- App nặng (Spring Boot, JVM): Set 300-600 giây
- Quá ngắn → Instance bị terminate khi đang khởi động
- Quá dài → Instance lỗi nhận traffic lâu trước khi bị replace

### Instance Refresh (Làm Mới Instance)

Thay thế tất cả instances trong ASG theo cách có kiểm soát (rolling replacement).

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name my-asg \
  --preferences '{
    "MinHealthyPercentage": 90,       # Giữ ít nhất 90% instances healthy
    "InstanceWarmup": 300,             # Cho 300 giây warmup sau launch
    "MaxHealthyPercentage": 110,       # Cho phép tạm thời 110% để replace
    "SkipMatching": true               # Bỏ qua instances đã dùng template mới
  }'
```

---

## 🐳 ECS Health Checks {#ecs-health-checks}

ECS có hai tầng health check: container-level và service-level.

### Container Health Check (Docker HEALTHCHECK)

```json
{
  "containerDefinitions": [{
    "name": "my-app",
    "image": "my-app:latest",
    "healthCheck": {
      "command": [
        "CMD-SHELL",
        "curl -f http://localhost:8080/health || exit 1"
      ],
      "interval": 30,        // Kiểm tra mỗi 30 giây
      "timeout": 5,          // Timeout sau 5 giây
      "retries": 3,          // 3 lần fail → unhealthy
      "startPeriod": 60      // Bỏ qua fail trong 60 giây đầu
    }
  }]
}
```

**Container Health Status:**
- `HEALTHY`: Lần check gần nhất pass
- `UNHEALTHY`: ≥ retries lần fail liên tiếp → ECS có thể restart task
- `UNKNOWN`: Chưa có kết quả check

### ECS Service Health với ELB

```json
{
  "loadBalancers": [{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...",
    "containerName": "my-app",
    "containerPort": 8080
  }],
  "healthCheckGracePeriodSeconds": 120
}
```

ECS Circuit Breaker (Ngắt Mạch ECS) — tự động dừng deployment nếu quá nhiều tasks fail:

```json
{
  "deploymentConfiguration": {
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true    // Tự động rollback khi deployment fail
    },
    "maximumPercent": 200,
    "minimumHealthyPercent": 100
  }
}
```

---

## ☸️ EKS / Kubernetes Probes {#kubernetes-probes}

Kubernetes có 3 loại probe (bộ dò) để kiểm tra sức khỏe pod.

### Liveness Probe (Bộ Dò Sống Còn)

Kiểm tra pod có đang **chạy** không. Nếu fail → pod bị restart.

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30    # Chờ 30 giây sau khởi động
  periodSeconds: 10          # Kiểm tra mỗi 10 giây
  timeoutSeconds: 5          # Timeout 5 giây
  failureThreshold: 3        # 3 lần fail → restart pod
```

**Khi Nào Liveness Fail → Restart Giải Quyết Vấn Đề:**
- Deadlock (khóa chết) trong ứng dụng
- Memory leak dẫn đến app treo
- Infinite loop

### Readiness Probe (Bộ Dò Sẵn Sàng)

Kiểm tra pod có **sẵn sàng nhận traffic** không. Nếu fail → loại khỏi Service endpoints.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  successThreshold: 1        # 1 lần pass → đưa vào Service
  failureThreshold: 3        # 3 lần fail → bỏ khỏi Service
```

**Khi Nào Dùng Readiness Probe:**
- Sau khởi động cần load cache, connect DB
- Đang xử lý long-running requests
- Đã quá tải, không thể nhận thêm traffic

### Startup Probe (Bộ Dò Khởi Động)

Kiểm tra ứng dụng đã **khởi động xong** chưa. Khi Startup Probe pass, liveness probe mới bắt đầu.

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30       # 30 lần × 10 giây = tối đa 5 phút để khởi động
  periodSeconds: 10
```

**Phù Hợp Cho:** JVM apps, ứng dụng cần load nhiều dữ liệu lúc khởi động.

### So Sánh 3 Probes

```
Startup Probe
    │ (pass → bắt đầu Liveness Probe)
    ▼
Liveness Probe                   Readiness Probe
    │ fail → restart pod              │ fail → remove from Service
    │ pass → pod tiếp tục             │ pass → add to Service
    ▼                                 ▼

Điểm Khác Biệt Chính:
- Liveness: "Pod có alive không?" → restart khi fail
- Readiness: "Pod có ready nhận traffic không?" → remove từ LB khi fail
- Startup: "Pod đã boot xong chưa?" → chỉ chạy một lần
```

---

## 🔁 Auto Recovery Patterns {#auto-recovery}

### Pattern 1: ASG Self-Healing

```
Instance fail health check
        │
        ▼
ASG terminate instance (sau grace period)
        │
        ▼
ASG launch replacement instance (cùng AZ hoặc AZ khác)
        │
        ▼
ELB bắt đầu route traffic khi healthy
```

**Thời Gian Phục Hồi Điển Hình:**
- ELB phát hiện unhealthy: 1-3 phút (threshold × interval)
- ASG terminate + launch: 1-2 phút
- Instance bootstrap: 1-5 phút (tùy app)
- ELB health check pass: 1-2 phút
- **Tổng: 5-13 phút** cho instance hoàn toàn mới

### Pattern 2: EC2 Auto Recovery

Dùng khi muốn **giữ nguyên instance** (ID, IP, volumes) — chỉ chuyển sang host server mới.

```
[CloudWatch Alarm: StatusCheckFailed_System ≥ 1 trong 2 phút]
        │
        ▼
[EC2 Recovery Action]
        │
        ├── Stop instance trên host lỗi
        ├── Chuyển instance sang host server khác
        └── Start instance trên host mới

Giữ nguyên: Instance ID, Elastic IP, EBS volumes, Security Groups
Mất: Instance store data (ephemeral storage)
Thời gian: 3-5 phút
```

### Pattern 3: ECS Service Restart

```
Task fail health check (Docker healthcheck)
        │
        ▼
ECS đánh dấu task UNHEALTHY
        │
        ▼
ECS Service stop task
        │
        ▼
ECS Service launch replacement task (trên instance hoặc Fargate mới)
```

### Pattern 4: RDS Failover

```
Primary RDS instance gặp sự cố
        │
        ▼
RDS phát hiện lỗi (~60 giây)
        │
        ▼
Promote Standby instance thành Primary mới
        │
        ▼
Cập nhật DNS endpoint (CNAME)
        │
        ▼
Ứng dụng kết nối lại (connection pool reconnect)

Thời gian failover: 60-120 giây
```

---

## 🔌 Circuit Breaker Pattern {#circuit-breaker}

Circuit Breaker (Ngắt Mạch) ngăn cascade failure (lỗi lan rộng) khi một service bị lỗi.

### Ba Trạng Thái Circuit Breaker

```
        ┌─────────────────────────────────┐
        │            CLOSED               │  (Bình thường)
        │  Requests pass through          │
        │  Đếm failures                   │
        └────────────┬────────────────────┘
                     │ Failures > threshold
                     ▼
        ┌────────────────────────────────┐
        │              OPEN              │  (Ngắt mạch)
        │  Tất cả requests bị từ chối    │
        │  ngay lập tức (fail fast)      │
        └────────────┬───────────────────┘
                     │ Sau timeout period
                     ▼
        ┌────────────────────────────────┐
        │           HALF-OPEN            │  (Thử nghiệm)
        │  Cho phép một số requests qua  │
        │  để kiểm tra                   │
        └────────┬──────────┬────────────┘
                 │           │
           Success        Failure
                 │           │
                 ▼           ▼
            CLOSED         OPEN
```

### Implementation trong ECS (Deployment Circuit Breaker)

```json
{
  "deploymentConfiguration": {
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }
}
```

Khi deploy mới:
- Nếu ≥ 50% tasks fail health check → circuit breaker OPEN
- `rollback: true` → tự động rollback về task definition cũ

### Retry với Exponential Backoff (Thử Lại với Độ Trễ Tăng Dần)

```python
import time
import random

def call_with_retry(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            
            # Exponential backoff với jitter (độ ngẫu nhiên)
            delay = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(delay)
            # Attempt 0: ~1s, Attempt 1: ~2s, Attempt 2: ~4s...
```

---

## 📊 Monitoring Health Checks

### CloudWatch Dashboard cho Health

```bash
# Tạo metric filter cho 5xx errors
aws logs put-metric-filter \
  --log-group-name /aws/ecs/my-service \
  --filter-name http-5xx-errors \
  --filter-pattern '[timestamp, requestId, level="ERROR", ...]' \
  --metric-transformations \
    metricName=HTTP5xxErrors,metricNamespace=MyApp,metricValue=1

# Alarm khi 5xx rate cao
aws cloudwatch put-metric-alarm \
  --alarm-name high-5xx-rate \
  --metric-name HTTP5xxErrors \
  --namespace MyApp \
  --statistic Sum \
  --period 60 \
  --threshold 10 \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:...:ops-alert
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: EC2 System Status Check vs Instance Status Check khác nhau thế nào?**
> System: Lỗi phần cứng/mạng của AWS host server → fix bằng stop/start (chuyển host). Instance: Lỗi OS hoặc phần mềm trong instance → fix bằng reboot hoặc sửa cấu hình.

**Q: Tại sao cần Health Check Grace Period trong ASG?**
> Ngăn ASG terminate instance mới trước khi ứng dụng kịp khởi động. Nếu không có grace period, ELB có thể báo unhealthy khi app đang load dependencies → ASG terminate → loop vô tận.

**Q: Liveness Probe vs Readiness Probe trong Kubernetes?**
> Liveness: "App đang chạy không?" → fail → restart pod. Readiness: "App sẵn sàng nhận request không?" → fail → loại khỏi Service endpoints (không restart). Dùng cả hai: Readiness để graceful handling khi startup, Liveness để phát hiện deadlock.

**Q: Circuit Breaker Pattern giải quyết vấn đề gì?**
> Ngăn cascade failure: Khi service B chậm/lỗi, service A không nên chờ timeout mỗi request → waste resources. Circuit Breaker phát hiện pattern lỗi và fail fast ngay lập tức, cho service B thời gian phục hồi.

---

**Tiếp Theo:** [4-deployment-strategies.md](./4-deployment-strategies.md) — Chiến Lược Triển Khai Không Gây Downtime
