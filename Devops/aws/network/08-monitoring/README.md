# 08 — Monitoring (Giám Sát Mạng AWS)

> **Mức độ:** Trung cấp đến Nâng cao | **Thời gian học ước tính:** 3–4 ngày

---

## Giới Thiệu

Một kiến trúc mạng AWS hoạt động tốt không chỉ được thiết kế đúng — mà còn phải được **giám sát liên tục**. Monitoring (Giám Sát) cho phép bạn:

- **Phát hiện sớm** bất thường về lưu lượng, kết nối, hoặc bảo mật
- **Chẩn đoán** sự cố một cách có hệ thống thay vì đoán mò
- **Chứng minh tuân thủ** (compliance) với các yêu cầu kiểm toán
- **Tối ưu hiệu suất** dựa trên dữ liệu thực tế thay vì giả định

Trong môi trường production, thiếu monitoring nghĩa là bạn đang bay mù — bạn sẽ chỉ biết có sự cố khi khách hàng phàn nàn.

---

## Danh Sách Chủ Đề Con

| #  | File                                                           | Chủ đề                    | Mô tả ngắn                                                     |
|----|----------------------------------------------------------------|---------------------------|----------------------------------------------------------------|
| 1  | [1-vpc-flow-logs.md](./1-vpc-flow-logs.md)                     | VPC Flow Logs             | Ghi lại toàn bộ lưu lượng IP vào/ra VPC, subnet, hoặc ENI     |
| 2  | [2-cloudwatch-networking.md](./2-cloudwatch-networking.md)     | CloudWatch Networking     | Metrics & Alarms cho ELB, CloudFront, VPN, Transit Gateway     |
| 3  | [3-network-access-analyzer.md](./3-network-access-analyzer.md) | Network Access Analyzer   | Phân tích quyền truy cập mạng, phát hiện đường đi ngoài ý muốn |
| 4  | [4-reachability-analyzer.md](./4-reachability-analyzer.md)     | Reachability Analyzer     | Kiểm tra từng bước kết nối giữa hai điểm trong VPC             |
| 5  | [5-observability-checklist.md](./5-observability-checklist.md) | Observability Checklist   | Checklist giám sát toàn diện cho production environment        |

---

## Tại Sao Monitoring Mạng Quan Trọng?

### Kịch Bản Thực Tế

```
Tình huống 1: Rò rỉ dữ liệu nhạy cảm
─────────────────────────────────────
Không có Flow Logs → Không biết EC2 đang gửi dữ liệu ra ngoài
Có Flow Logs       → Phát hiện traffic bất thường đến IP lạ → chặn ngay

Tình huống 2: Mất kết nối bí ẩn
────────────────────────────────
Không có monitoring → Debug mất 4 tiếng, hỏi từng người
Có Reachability Analyzer → Xác định Security Group blocking trong 5 phút

Tình huống 3: ALB bị quá tải
─────────────────────────────
Không có CloudWatch Alarms → Chỉ biết khi site down, 503 errors
Có CloudWatch Alarms       → Cảnh báo trước khi đạt ngưỡng, scale kịp thời
```

---

## Tổng Quan Các Công Cụ Giám Sát Mạng AWS

### Bản Đồ Công Cụ → Vấn Đề Giải Quyết

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Network Monitoring Stack                  │
├─────────────────────┬───────────────────────────────────────────┤
│ CÔNG CỤ             │ GIẢI QUYẾT VẤN ĐỀ GÌ?                    │
├─────────────────────┼───────────────────────────────────────────┤
│ VPC Flow Logs       │ "Traffic nào đang đi qua mạng của tôi?"   │
│                     │ "Tại sao kết nối bị từ chối (REJECT)?"     │
│                     │ "Ai đang tấn công từ IP nào?"              │
├─────────────────────┼───────────────────────────────────────────┤
│ CloudWatch Metrics  │ "Load Balancer của tôi đang hoạt động     │
│                     │  như thế nào? Latency? Error rate?"        │
│                     │ "Khi nào cần cảnh báo & scale?"            │
├─────────────────────┼───────────────────────────────────────────┤
│ Network Access      │ "Có đường đi không mong muốn nào từ       │
│ Analyzer            │  internet vào DB của tôi không?"           │
│                     │ "Kiến trúc này có tuân thủ policy không?"  │
├─────────────────────┼───────────────────────────────────────────┤
│ Reachability        │ "Tại sao EC2-A không kết nối được EC2-B?"  │
│ Analyzer            │ "Security Group hay Route Table block?"     │
│                     │ "Path chính xác từ A đến B là gì?"         │
├─────────────────────┼───────────────────────────────────────────┤
│ CloudTrail          │ "Ai đã thay đổi Security Group lúc 2am?"   │
│                     │ "Route Table bị chỉnh sửa khi nào?"        │
├─────────────────────┼───────────────────────────────────────────┤
│ AWS Config          │ "Security Group có đang mở port 22 ra      │
│                     │  internet không?" (compliance check)       │
└─────────────────────┴───────────────────────────────────────────┘
```

---

## Mô Hình Observability (Khả Năng Quan Sát) Ba Trụ Cột

Trong networking, observability (khả năng quan sát) được xây dựng trên **ba trụ cột**:

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│    LOGS     │   │   METRICS   │   │   TRACES    │
│  (Nhật Ký)  │   │ (Số Liệu)   │   │ (Theo Dõi)  │
├─────────────┤   ├─────────────┤   ├─────────────┤
│ VPC Flow    │   │ CloudWatch  │   │ X-Ray cho   │
│ Logs        │   │ Metrics     │   │ application │
│             │   │             │   │ layer       │
│ CloudTrail  │   │ Network     │   │             │
│ (API Logs)  │   │ Perf Mon    │   │ VPC Flow    │
│             │   │             │   │ Logs (IP    │
│ ELB Access  │   │ Enhanced    │   │ level)      │
│ Logs        │   │ Networking  │   │             │
│             │   │ Metrics     │   │             │
│ CloudFront  │   │             │   │             │
│ Access Logs │   │             │   │             │
└─────────────┘   └─────────────┘   └─────────────┘
     "Gì đã          "Xu hướng          "Luồng
      xảy ra?"        như thế nào?"      đi đâu?"
```

---

## Lộ Trình Học Topic 08

### Ngày 1 — VPC Flow Logs

```
Buổi sáng (2h): Đọc 1-vpc-flow-logs.md
  - Cấu trúc log record
  - Các trường quan trọng (srcaddr, dstaddr, action)
  - Gửi logs đến CloudWatch Logs vs S3

Buổi chiều (2h): Hands-on
  - Bật Flow Logs cho VPC test
  - Tạo filter pattern trong CloudWatch Logs Insights
  - Phân tích REJECT records
```

### Ngày 2 — CloudWatch Networking Metrics

```
Buổi sáng (2h): Đọc 2-cloudwatch-networking.md
  - ELB metrics (RequestCount, TargetResponseTime, HTTPCode_ELB_5XX)
  - CloudFront metrics (CacheHitRate, 4xxErrorRate)
  - VPN/DX metrics (TunnelState, BytesIn/Out)

Buổi chiều (1.5h): Hands-on
  - Tạo Dashboard cho network metrics
  - Thiết lập Alarms với SNS notification
```

### Ngày 3 — Network Access & Reachability Analyzer

```
Buổi sáng (2h): Đọc file 3 và 4
  - Network Access Analyzer: phân tích quyền truy cập toàn kiến trúc
  - Reachability Analyzer: debug kết nối từng bước

Buổi chiều (2h): Hands-on
  - Chạy Reachability Analyzer cho EC2 pair có vấn đề
  - Tạo Network Access Scope để kiểm tra compliance
```

### Ngày 4 — Tổng Hợp & Checklist

```
Đọc 5-observability-checklist.md
Tự đánh giá môi trường hiện tại theo checklist
Xây dựng runbook (tài liệu vận hành) cho team
```

---

## Use Cases — Khi Nào Dùng Công Cụ Nào?

| Tình Huống                                          | Công Cụ Phù Hợp                       |
|-----------------------------------------------------|---------------------------------------|
| Debug kết nối bị từ chối giữa hai EC2               | Reachability Analyzer + Flow Logs     |
| Phát hiện port mở không mong muốn                   | Network Access Analyzer               |
| Theo dõi hiệu suất ALB theo thời gian thực          | CloudWatch Metrics + Dashboard        |
| Điều tra traffic bất thường từ IP lạ                | VPC Flow Logs → CloudWatch Insights   |
| Kiểm tra ai đã thay đổi Security Group              | CloudTrail                            |
| Đảm bảo không có EC2 nào expose port 22 ra internet | AWS Config + Network Access Analyzer  |
| Cảnh báo khi error rate ALB vượt ngưỡng             | CloudWatch Alarm + SNS                |
| Phân tích chi phí NAT Gateway theo workload         | VPC Flow Logs → Athena query          |

---

## Các Khái Niệm Cốt Lõi Cần Nắm

Trước khi đi vào từng file, cần hiểu:

- **ENI** (Elastic Network Interface — Giao Diện Mạng Ảo): mọi Flow Log đều gắn với ENI
- **ACCEPT vs REJECT**: Flow Logs ghi lại kết quả, không phải chỉ traffic đi qua
- **Metric vs Log**: Metrics là số liệu tổng hợp theo thời gian; Logs là sự kiện cụ thể từng cái một
- **Alarm vs Dashboard**: Alarm tự động cảnh báo; Dashboard hiển thị trực quan để con người theo dõi
- **Scope (Phạm Vi)**: Network Access Analyzer dùng "scope" để định nghĩa vùng kiểm tra

---

## Liên Kết Đến Các Topic Khác

- **VPC Flow Logs** phụ thuộc vào hiểu biết về [02-security/1-security-groups.md](../02-security/1-security-groups.md) để đọc ACCEPT/REJECT
- **CloudWatch Metrics cho ELB** liên quan đến [03-load-balancing/3-target-groups.md](../03-load-balancing/3-target-groups.md) (health checks)
- **Reachability Analyzer** phân tích [01-vpc-fundamentals/3-route-tables.md](../01-vpc-fundamentals/3-route-tables.md) và Security Groups
- Khi phát hiện vấn đề qua monitoring → chuyển sang [09-troubleshooting/](../09-troubleshooting/) để xử lý

---

**Tiếp Theo:** Bắt đầu với [1-vpc-flow-logs.md](./1-vpc-flow-logs.md)
