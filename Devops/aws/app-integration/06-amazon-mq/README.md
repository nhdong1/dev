# Amazon MQ — Managed Message Broker (Môi Giới Tin Nhắn Được Quản Lý)

> Amazon MQ là dịch vụ message broker được quản lý hoàn toàn, tương thích với các giao thức nhắn tin tiêu chuẩn công nghiệp, giúp di chuyển ứng dụng legacy lên AWS mà không cần thay đổi code.

## 📚 Mục Lục Module

| File | Nội Dung | Mức Độ |
|---|---|---|
| `README.md` | Tổng quan, khi nào dùng, so sánh với SQS/SNS | Cơ bản |
| `1-activemq-vs-rabbitmq.md` | So sánh Apache ActiveMQ và RabbitMQ | Trung cấp |
| `2-migration-guide.md` | Hướng dẫn di chuyển từ on-premises lên Amazon MQ | Nâng cao |

---

## 🎯 Amazon MQ Là Gì?

**Amazon MQ** là dịch vụ managed message broker (môi giới tin nhắn được quản lý) hỗ trợ hai engine phổ biến:

- **Apache ActiveMQ** — message broker Java truyền thống, hỗ trợ nhiều giao thức
- **RabbitMQ** — message broker hiện đại, nổi tiếng với tính linh hoạt và plugin ecosystem

Amazon MQ được thiết kế đặc biệt cho **trường hợp migration (di chuyển)** — khi bạn đang chạy ActiveMQ hoặc RabbitMQ on-premises và muốn chuyển lên AWS mà **không cần viết lại ứng dụng**.

---

## 🏗️ Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                         Amazon MQ                               │
│                                                                 │
│  ┌──────────────┐    ┌─────────────────────────────────────┐   │
│  │  Producers   │    │         Broker Instance             │   │
│  │ (Nhà Sản     │───▶│                                     │   │
│  │  Xuất)       │    │  ┌─────────┐    ┌────────────────┐  │   │
│  └──────────────┘    │  │ Queue   │    │    Topic       │  │   │
│                      │  │(Hàng    │    │ (Chủ Đề —     │  │   │
│  ┌──────────────┐    │  │ Đợi)    │    │ Pub/Sub)       │  │   │
│  │  Consumers   │◀───│  └─────────┘    └────────────────┘  │   │
│  │ (Người Tiêu  │    │                                     │   │
│  │  Dùng)       │    │  Engine: ActiveMQ hoặc RabbitMQ     │   │
│  └──────────────┘    └─────────────────────────────────────┘   │
│                                                                 │
│  Giao thức hỗ trợ: AMQP, MQTT, STOMP, OpenWire, WebSocket     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📡 Giao Thức Được Hỗ Trợ

| Giao Thức | Tên Đầy Đủ | Mô Tả | Engine |
|---|---|---|---|
| **AMQP** | Advanced Message Queuing Protocol (Giao Thức Xếp Hàng Tin Nhắn Nâng Cao) | Tiêu chuẩn công nghiệp, phổ biến nhất | ActiveMQ, RabbitMQ |
| **MQTT** | Message Queuing Telemetry Transport (Vận Chuyển Từ Xa Xếp Hàng Tin Nhắn) | Nhẹ, dành cho IoT | ActiveMQ |
| **STOMP** | Simple Text Oriented Messaging Protocol (Giao Thức Nhắn Tin Định Hướng Văn Bản Đơn Giản) | Giao thức văn bản, đơn giản | ActiveMQ |
| **OpenWire** | Giao thức nhị phân riêng của ActiveMQ | Hiệu suất cao cho Java clients | ActiveMQ |
| **WebSocket** | Kết nối hai chiều qua HTTP | Dùng với STOMP over WebSocket | ActiveMQ |

---

## 🆚 Amazon MQ vs SQS vs SNS

Đây là câu hỏi quan trọng nhất khi học Amazon MQ:

| Tiêu Chí | Amazon MQ | Amazon SQS | Amazon SNS |
|---|---|---|---|
| **Mục đích chính** | Migration từ on-premises | Cloud-native queue | Cloud-native pub/sub |
| **Giao thức** | AMQP, MQTT, STOMP, OpenWire | AWS SDK / HTTP | AWS SDK / HTTP |
| **Thay đổi code** | Không cần (nếu dùng chuẩn) | Cần viết lại | Cần viết lại |
| **Scalability** (Khả Năng Mở Rộng) | Giới hạn theo instance | Không giới hạn | Không giới hạn |
| **Serverless** (Phi Máy Chủ) | Không — cần provision broker | Có | Có |
| **Quản lý** | Vẫn cần quản lý broker | Fully managed | Fully managed |
| **Chi phí cơ bản** | Cao hơn (instance cost) | Thấp hơn (pay per message) | Thấp hơn (pay per message) |
| **Transaction** (Giao Dịch) | Hỗ trợ XA Transaction | Không | Không |
| **Message Priority** (Ưu Tiên Tin Nhắn) | Có | Không | Không |
| **Redelivery Policy** (Chính Sách Tái Giao Vận) | Tùy chỉnh linh hoạt | Visibility Timeout | Không áp dụng |

### Quy Tắc Vàng Khi Chọn

```
Câu hỏi 1: Bạn đang migration từ on-premises không?
  ├── Có → Dùng Amazon MQ (giữ nguyên giao thức, không viết lại code)
  └── Không → Tiếp tục câu hỏi 2

Câu hỏi 2: Bạn cần fan-out (khuếch tán) một-tới-nhiều?
  ├── Có → Dùng SNS (hoặc EventBridge nếu cần event routing)
  └── Không → Tiếp tục câu hỏi 3

Câu hỏi 3: Bạn cần queue điểm-tới-điểm đơn giản?
  └── Có → Dùng SQS
```

---

## 🚀 Khi Nào Dùng Amazon MQ?

### ✅ Nên Dùng Amazon MQ

1. **Di chuyển ứng dụng legacy** — App đang dùng ActiveMQ/RabbitMQ on-premises, muốn lift-and-shift (nâng và di chuyển) lên AWS
2. **Yêu cầu giao thức chuẩn** — Phải dùng AMQP, MQTT, STOMP vì client libraries đã cố định
3. **Hệ thống đa ngôn ngữ** — Java, .NET, Python, Ruby, Go đều cần kết nối qua cùng protocol
4. **Tính năng nâng cao của broker** — Message priority (ưu tiên), XA transaction (giao dịch XA), selector (bộ chọn lọc), virtual destinations (đích ảo)
5. **Quy định tuân thủ** — Yêu cầu dùng message broker đạt chứng nhận JMS (Java Message Service) hay AMQP

### ❌ Không Nên Dùng Amazon MQ

1. **Ứng dụng mới trên AWS** — Dùng SQS/SNS/EventBridge — scalability tốt hơn, chi phí thấp hơn
2. **Cần scale tự động không giới hạn** — SQS không giới hạn throughput, MQ bị giới hạn bởi instance type
3. **Serverless architecture** — SQS/SNS tích hợp Lambda tốt hơn nhiều
4. **Chi phí quan trọng** — SQS rẻ hơn đáng kể cho low-to-medium traffic

---

## 🏭 Deployment Options (Tùy Chọn Triển Khai)

### Single-Instance Broker (Broker Đơn Lẻ)

```
┌─────────────────────────┐
│   Availability Zone A   │
│  ┌───────────────────┐  │
│  │  Broker Instance  │  │
│  │  (EBS Storage)    │  │
│  └───────────────────┘  │
└─────────────────────────┘
```

- **Dùng cho**: Development (phát triển), testing (kiểm thử)
- **Đặc điểm**: Đơn giản, chi phí thấp nhất, không có HA (High Availability — Tính Sẵn Sàng Cao)
- **Downtime**: Có downtime khi maintenance hoặc lỗi

### Active/Standby HA (Chủ động / Dự phòng)

```
┌─────────────────────────┐    ┌─────────────────────────┐
│   Availability Zone A   │    │   Availability Zone B   │
│  ┌───────────────────┐  │    │  ┌───────────────────┐  │
│  │  ACTIVE Broker    │  │    │  │  STANDBY Broker   │  │
│  │  (đang phục vụ)   │◀─┼────┼─▶│  (chờ sẵn sàng)   │  │
│  └────────┬──────────┘  │    │  └────────┬──────────┘  │
└───────────┼─────────────┘    └───────────┼─────────────┘
            │                              │
            └──────────┬───────────────────┘
                       │
              ┌────────▼───────────┐
              │  Amazon EFS        │
              │ (Shared Storage —  │
              │  Lưu Trữ Dùng      │
              │  Chung)            │
              └────────────────────┘
```

- **Dùng cho**: Production workloads
- **Failover** (Chuyển Dự Phòng): Tự động, thường trong vòng 30 giây
- **Storage**: Amazon EFS dùng chung giữa active và standby

### RabbitMQ Cluster (Cụm RabbitMQ)

```
┌─────────────────────────────────────────────────────┐
│               RabbitMQ Cluster                      │
│                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │ Node 1   │──▶│ Node 2   │──▶│ Node 3   │        │
│  │ (AZ-a)   │   │ (AZ-b)   │   │ (AZ-c)   │        │
│  └──────────┘   └──────────┘   └──────────┘        │
│                                                     │
│         Network Load Balancer (NLB)                 │
│     (Bộ Cân Bằng Tải Mạng — điểm vào duy nhất)     │
└─────────────────────────────────────────────────────┘
```

- **Dùng cho**: RabbitMQ với high availability và higher throughput
- **Quorum Queues** (Hàng Đợi Quorum): Đảm bảo dữ liệu được replica (sao chép) qua nhiều node

---

## 💰 Mô Hình Chi Phí

Amazon MQ tính phí theo **instance hour** (giờ instance) và **storage** (lưu trữ):

| Thành Phần | Cách Tính | Ghi Chú |
|---|---|---|
| **Broker instance** | $/giờ theo instance type | Không dừng được như EC2 |
| **Storage** (EBS/EFS) | $/GB-tháng | EFS dùng cho HA deployments |
| **Data transfer** (Truyền Dữ Liệu) | $/GB | Tính cho data out |

### So Sánh Chi Phí Nhanh

```
Tình huống: 1 triệu message/ngày, 1KB mỗi message

Amazon MQ (mq.m5.large, single):  ~$150-200/tháng (cố định)
Amazon SQS Standard:               ~$0.40/tháng    (pay-per-use)

→ SQS rẻ hơn 375-500 lần cho workload nhỏ
→ MQ chỉ nên dùng khi có yêu cầu kỹ thuật bắt buộc
```

---

## 🔒 Bảo Mật

### Encryption (Mã Hóa)

- **In-transit** (Trong quá trình truyền): TLS 1.2/1.3 cho tất cả kết nối
- **At-rest** (Khi lưu trữ): AWS KMS (Key Management Service — Dịch Vụ Quản Lý Khóa) với customer-managed keys

### Network Access (Truy Cập Mạng)

```
Internet          VPC (Virtual Private Cloud)
                  ┌──────────────────────────┐
Client ──[TLS]──▶ │  ┌────────────────────┐  │
                  │  │   Amazon MQ        │  │
                  │  │   Broker           │  │
                  │  └────────────────────┘  │
                  │  Private Subnet          │
                  └──────────────────────────┘
```

- Broker chạy trong **VPC** (Virtual Private Cloud — Đám Mây Riêng Ảo)
- Hỗ trợ public endpoint (đầu cuối công khai) hoặc private endpoint (đầu cuối riêng)
- Security Group (Nhóm Bảo Mật) kiểm soát port access
- Port mặc định: 61616 (OpenWire), 5671 (AMQP), 8883 (MQTT), 61614 (STOMP)

---

## 📊 Metrics Quan Trọng Cần Theo Dõi

| Metric | Ý Nghĩa | Ngưỡng Cảnh Báo |
|---|---|---|
| `TotalMessageCount` | Tổng số message trong tất cả queue | Tăng liên tục → consumer bị chậm |
| `ConsumerCount` | Số consumer đang kết nối | = 0 → không có consumer, message tích tụ |
| `ProducerCount` | Số producer đang kết nối | Bất thường → kiểm tra kết nối |
| `EnqueueCount` | Số message được thêm vào | Đo throughput producer |
| `DequeueCount` | Số message được lấy ra | Đo throughput consumer |
| `MemoryUsage` | Phần trăm RAM đã dùng | > 80% → cần scale up |
| `StorePercentUsage` | Phần trăm storage đã dùng | > 70% → cần tăng storage |
| `TempPercentUsage` | Phần trăm temp storage đã dùng | > 70% → kiểm tra large messages |

---

## 🎓 Checklist Kiến Thức

Sau khi học module này, bạn cần nắm được:

### Cơ Bản

- [ ] Amazon MQ là gì và dùng engine nào
- [ ] Tại sao dùng MQ thay vì SQS/SNS (và ngược lại)
- [ ] Các giao thức được hỗ trợ
- [ ] Deployment options: single vs HA

### Trung Cấp

- [ ] So sánh chi tiết ActiveMQ và RabbitMQ
- [ ] Khi nào chọn ActiveMQ, khi nào chọn RabbitMQ
- [ ] Cấu hình HA với Active/Standby hoặc RabbitMQ Cluster
- [ ] Các tính năng nâng cao: message priority, selector, virtual destination

### Nâng Cao

- [ ] Chiến lược migration từ on-premises
- [ ] Performance tuning và capacity planning
- [ ] Monitoring và troubleshooting
- [ ] Cost optimization

---

## 🔗 Điều Hướng

| Bước Trước | Bước Tiếp Theo |
|---|---|
| [05-kinesis/](../05-kinesis/README.md) — Amazon Kinesis | [1-activemq-vs-rabbitmq.md](./1-activemq-vs-rabbitmq.md) — So Sánh Engine |

Sau module này: [07-appsync/](../07-appsync/README.md) — AWS AppSync

---

**Cập Nhật Lần Cuối:** 2026-05-18
