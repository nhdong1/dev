# 📊 Monitoring & Observability — Giám Sát Hệ Thống Tích Hợp AWS

> Tổng quan về chiến lược giám sát toàn diện cho các dịch vụ AWS Application Integration: SQS, SNS, EventBridge, Step Functions, Kinesis và AppSync.

## 📚 Mục Lục

1. [Tại Sao Monitoring Quan Trọng](#tại-sao-monitoring-quan-trọng)
2. [Ba Trụ Cột Observability](#ba-trụ-cột-observability)
3. [Công Cụ AWS Dùng Để Giám Sát](#công-cụ-aws-dùng-để-giám-sát)
4. [Tổng Quan Các File Trong Thư Mục Này](#tổng-quan-các-file)
5. [Chiến Lược Giám Sát Theo Lớp](#chiến-lược-giám-sát-theo-lớp)
6. [Checklist Production-Ready](#checklist-production-ready)

---

## Tại Sao Monitoring Quan Trọng

Trong kiến trúc Event-Driven (Hướng Sự Kiện) và Microservices (Dịch Vụ Vi Mô), các thành phần giao tiếp qua message (tin nhắn) và event (sự kiện) bất đồng bộ — khiến việc debug (gỡ lỗi) và tracing (theo dõi luồng) phức tạp hơn nhiều so với kiến trúc monolithic (nguyên khối) truyền thống.

**Vấn đề đặc trưng của hệ thống messaging:**

- **Tin nhắn bị mắc kẹt** trong queue mà không có ai xử lý
- **Consumer lag** (Độ Trễ Người Tiêu Dùng) — consumer xử lý chậm hơn tốc độ producer gửi
- **Poison message** (Tin Nhắn Độc Hại) — tin nhắn gây lỗi liên tục, chặn toàn bộ queue
- **Silent failure** (Lỗi Thầm Lặng) — service thất bại nhưng không gây lỗi rõ ràng
- **Cascading failure** (Lỗi Dây Chuyền) — một service chậm làm chậm toàn bộ pipeline

Monitoring tốt giúp phát hiện sớm các vấn đề này trước khi chúng ảnh hưởng người dùng cuối.

---

## Ba Trụ Cột Observability

**Observability** (Khả Năng Quan Sát) là khả năng hiểu trạng thái bên trong hệ thống thông qua đầu ra bên ngoài. Gồm ba trụ cột:

### 1. Metrics (Số Liệu)

Dữ liệu định lượng đo lường theo thời gian — số tin nhắn, độ trễ, tỷ lệ lỗi.

```
Ví dụ:
- SQS: ApproximateNumberOfMessagesVisible (Số Tin Nhắn Chờ Xử Lý Xấp Xỉ)
- Kinesis: GetRecords.IteratorAgeMilliseconds (Tuổi Iterator Tính Bằng Mili Giây)
- Step Functions: ExecutionsFailed (Số Lần Thực Thi Thất Bại)
```

**Dùng để:** Cảnh báo ngưỡng, dashboard tổng quan, capacity planning (lập kế hoạch năng lực).

### 2. Logs (Nhật Ký)

Bản ghi chi tiết từng sự kiện — ai làm gì, khi nào, với kết quả gì.

```
Ví dụ:
- Lambda execution log khi xử lý SQS message
- Step Functions execution history (lịch sử thực thi)
- EventBridge rule match/no-match log
```

**Dùng để:** Debug lỗi cụ thể, kiểm tra audit trail (vết kiểm toán), phân tích nguyên nhân gốc rễ.

### 3. Traces (Dấu Vết)

Theo dõi một request/event qua nhiều service — từ đầu đến cuối.

```
Ví dụ với AWS X-Ray:
API Gateway → Lambda → SQS → Lambda (consumer) → DynamoDB
Mỗi bước được ghi lại với thời gian và metadata
```

**Dùng để:** Tìm bottleneck (điểm nghẽn), hiểu luồng xử lý end-to-end (đầu cuối tới đầu cuối).

---

## Công Cụ AWS Dùng Để Giám Sát

### Amazon CloudWatch

**CloudWatch** là dịch vụ monitoring trung tâm của AWS — thu thập metrics, lưu logs và kích hoạt alarm (cảnh báo).

| Tính Năng | Mô Tả |
|---|---|
| **CloudWatch Metrics** | Thu thập số liệu từ mọi dịch vụ AWS tự động |
| **CloudWatch Alarms** | Cảnh báo khi metric vượt ngưỡng định nghĩa |
| **CloudWatch Logs** | Lưu trữ và tìm kiếm log tập trung |
| **CloudWatch Logs Insights** | Query log bằng ngôn ngữ tương tự SQL |
| **CloudWatch Dashboards** | Bảng điều khiển trực quan tùy chỉnh |
| **CloudWatch Contributor Insights** | Phân tích top contributor gây tải cao |

### AWS X-Ray

**X-Ray** là dịch vụ Distributed Tracing (Theo Dõi Phân Tán) — theo dõi request xuyên qua nhiều service.

```
Khái niệm cốt lõi:
- Trace (Dấu Vết): toàn bộ hành trình của một request
- Segment (Phân Đoạn): phần xử lý trong một service
- Subsegment (Phân Đoạn Con): chi tiết trong một segment (vd: gọi DB)
- Sampling (Lấy Mẫu): tỷ lệ % request được trace để tiết kiệm chi phí
```

### AWS CloudTrail

**CloudTrail** ghi lại mọi API call vào AWS — dùng cho security audit (kiểm toán bảo mật) và compliance (tuân thủ). Khác với CloudWatch (monitoring runtime), CloudTrail ghi hành động quản trị (ai tạo queue, ai thay đổi policy...).

---

## Tổng Quan Các File

| File | Nội Dung | Khi Nào Đọc |
|---|---|---|
| [1-key-metrics.md](1-key-metrics.md) | Metrics quan trọng theo từng dịch vụ | Thiết lập dashboard lần đầu |
| [2-cloudwatch-alarms.md](2-cloudwatch-alarms.md) | Cấu hình alarm, ngưỡng khuyến nghị | Trước khi deploy production |
| [3-xray-tracing.md](3-xray-tracing.md) | X-Ray distributed tracing end-to-end | Debug latency và bottleneck |
| [4-dlq-monitoring.md](4-dlq-monitoring.md) | Giám sát DLQ, phát hiện poison message | Xử lý tin nhắn thất bại |

---

## Chiến Lược Giám Sát Theo Lớp

### Lớp 1: Infrastructure Metrics (Số Liệu Hạ Tầng)

Giám sát tài nguyên cơ bản — queue depth, throughput, error rate.

```
SQS: ApproximateNumberOfMessagesVisible > threshold → scale consumer
Kinesis: IteratorAgeMilliseconds > threshold → thêm shard hoặc tối ưu consumer
```

### Lớp 2: Business Metrics (Số Liệu Kinh Doanh)

Giám sát ý nghĩa kinh doanh — số đơn hàng xử lý/phút, tỷ lệ thanh toán thành công.

```
Custom metric: OrdersProcessedPerMinute
Alarm: OrdersProcessedPerMinute < 100 → cảnh báo on-call engineer
```

### Lớp 3: End-to-End Tracing (Theo Dõi Đầu Cuối)

Theo dõi hành trình của từng request qua toàn bộ hệ thống với X-Ray.

```
Ví dụ flow:
API Gateway [5ms] → Lambda Producer [10ms] → SQS [queue] → Lambda Consumer [50ms] → DynamoDB [8ms]
Total: 73ms — nếu tăng lên 500ms, X-Ray cho thấy bottleneck ở bước nào
```

### Lớp 4: Alerting & Incident Response (Cảnh Báo & Phản Ứng Sự Cố)

Kết nối alarm với hành động thực tế — gọi điện, gửi Slack, tự động scale.

```
CloudWatch Alarm → SNS Topic → PagerDuty / Slack / OpsGenie
                             → Lambda tự động xử lý (auto-remediation)
```

---

## Checklist Production-Ready

### Trước Khi Deploy

- [ ] Đã tạo CloudWatch Dashboard cho tất cả dịch vụ tích hợp
- [ ] Đã cấu hình alarm cho queue depth và consumer lag
- [ ] Đã bật X-Ray tracing trên Lambda và API Gateway
- [ ] Đã cấu hình DLQ cho mọi SQS queue và SNS subscription
- [ ] Đã thiết lập alarm khi DLQ nhận tin nhắn đầu tiên
- [ ] Đã cấu hình log retention (thời gian lưu log) — tránh chi phí log vô tận
- [ ] Đã test alarm bằng cách kích hoạt thủ công

### Trong Production

- [ ] Review dashboard hàng ngày trong sprint đầu
- [ ] Điều chỉnh ngưỡng alarm sau khi có baseline thực tế
- [ ] Phân tích X-Ray trace khi có latency spike (đột biến độ trễ)
- [ ] Kiểm tra DLQ định kỳ — xử lý tin nhắn lỗi trước khi hết TTL (Time To Live — Thời Gian Sống)
- [ ] Tạo runbook (sổ tay vận hành) cho từng loại alarm

---

## Liên Kết Tài Liệu

| Chủ Đề Liên Quan | Vị Trí |
|---|---|
| Dead Letter Queue cơ bản | [01-sqs/3-dead-letter-queue.md](../01-sqs/3-dead-letter-queue.md) |
| Kinesis Shard và throughput | [05-kinesis/4-shard-management.md](../05-kinesis/4-shard-management.md) |
| Step Functions error handling | [04-step-functions/3-error-handling.md](../04-step-functions/3-error-handling.md) |
| Circuit Breaker pattern | [08-patterns/7-circuit-breaker.md](../08-patterns/7-circuit-breaker.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
