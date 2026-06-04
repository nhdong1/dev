# DynamoDB — NoSQL Serverless

> Amazon DynamoDB (Cơ Sở Dữ Liệu NoSQL Serverless — Không Máy Chủ) là dịch vụ cơ sở dữ liệu key-value và document, hoàn toàn managed, có thể mở rộng đến bất kỳ quy mô nào với độ trễ single-digit millisecond (mili-giây đơn chữ số).

## 📚 Mục Lục

1. [Tổng Quan DynamoDB](#tổng-quan-dynamodb)
2. [Kiến Trúc & Mô Hình Dữ Liệu](#kiến-trúc--mô-hình-dữ-liệu)
3. [Capacity Modes — Chế Độ Năng Lực](#capacity-modes--chế-độ-năng-lực)
4. [Indexes — Chỉ Mục Phụ](#indexes--chỉ-mục-phụ)
5. [DynamoDB Streams & Lambda](#dynamodb-streams--lambda)
6. [Transactions — Giao Dịch ACID](#transactions--giao-dịch-acid)
7. [DAX — DynamoDB Accelerator](#dax--dynamodb-accelerator)
8. [Access Patterns & Single-Table Design](#access-patterns--single-table-design)
9. [So Sánh DynamoDB vs RDS](#so-sánh-dynamodb-vs-rds)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan DynamoDB

Amazon DynamoDB là dịch vụ cơ sở dữ liệu NoSQL fully managed (được quản lý hoàn toàn) được AWS ra mắt năm 2012. Nó được xây dựng để phục vụ các ứng dụng cần độ trễ cực thấp ở bất kỳ quy mô nào — từ hàng nghìn đến hàng tỷ request mỗi ngày.

### Điểm Mạnh Chính

| Đặc Điểm                        | Mô Tả                                                                          |
| ------------------------------- | ------------------------------------------------------------------------------ |
| **Serverless**                  | Không cần quản lý máy chủ hay infra — hoàn toàn managed                      |
| **Single-digit ms latency**     | Độ trễ mili-giây đơn chữ số kể cả ở quy mô lớn                               |
| **Unlimited scale**             | Mở rộng đến hàng petabyte dữ liệu, hàng triệu TPS (Transaction Per Second)   |
| **Multi-AZ by default**         | Dữ liệu được sao chép tự động qua 3 AZ (Availability Zone — Vùng Sẵn Sàng)  |
| **Flexible schema**             | Schema linh hoạt — không cần định nghĩa cột trước                             |
| **Event-driven**                | Tích hợp với Lambda qua DynamoDB Streams (Luồng Dữ Liệu)                     |

### Khi Nào Dùng DynamoDB

**Phù Hợp:**
- Ứng dụng cần độ trễ < 10ms ở mọi quy mô
- Gaming (leaderboard, player state)
- IoT (dữ liệu cảm biến, telemetry — đo từ xa)
- Session management (quản lý phiên người dùng)
- Shopping cart (giỏ hàng), user profile (hồ sơ người dùng)
- Real-time bidding (đấu giá thời gian thực)
- Catalog (danh mục sản phẩm) với access patterns rõ ràng

**Không Phù Hợp:**
- Cần complex JOIN queries (truy vấn JOIN phức tạp)
- Ad-hoc queries (truy vấn không có mẫu định sẵn)
- OLAP (Online Analytical Processing — Xử Lý Phân Tích Trực Tuyến)
- Dữ liệu quan hệ phức tạp với nhiều relationship
- Cần strong transaction support với nhiều bảng

---

## Kiến Trúc & Mô Hình Dữ Liệu

### Kiến Trúc Nội Bộ DynamoDB

```
┌─────────────────────────────────────────────────────────┐
│                    DynamoDB Service                      │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Request   │  │   Request   │  │   Request   │    │
│  │   Router    │  │   Router    │  │   Router    │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │             │
│  ┌──────▼──────────────────────────────────▼──────┐    │
│  │              Storage Nodes (Nút Lưu Trữ)       │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐        │    │
│  │  │  Shard  │  │  Shard  │  │  Shard  │        │    │
│  │  │  (AZ-a) │  │  (AZ-b) │  │  (AZ-c) │        │    │
│  │  └─────────┘  └─────────┘  └─────────┘        │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

DynamoDB lưu trữ dữ liệu trên nhiều partition (phân vùng) được phân tán qua nhiều máy chủ. Mỗi partition được sao chép qua 3 AZ để đảm bảo durability (độ bền) 99.999999999% (11 số 9).

### Thành Phần Cơ Bản

```
Table (Bảng)
│
├── Item (Mục) — tương đương "row" trong SQL
│   ├── Partition Key (Khóa Phân Vùng) — bắt buộc
│   ├── Sort Key (Khóa Sắp Xếp) — tùy chọn
│   └── Attributes (Thuộc Tính) — các cột khác, schema linh hoạt
│
├── GSI (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu)
│   └── Index riêng với partition key khác
│
└── LSI (Local Secondary Index — Chỉ Mục Phụ Cục Bộ)
    └── Cùng partition key, sort key khác
```

### Item — Mục Dữ Liệu

```json
{
  "UserId": "user-001",          // Partition Key (string)
  "OrderId": "order-2024-001",   // Sort Key (string)
  "Status": "PENDING",           // Attribute (thuộc tính)
  "TotalAmount": 150.00,         // Number attribute
  "Items": [                     // List attribute (thuộc tính danh sách)
    {"ProductId": "p1", "Qty": 2}
  ],
  "Metadata": {                  // Map attribute (thuộc tính bản đồ)
    "CreatedAt": "2024-01-15T10:00:00Z",
    "Source": "mobile-app"
  },
  "Tags": {"electronics", "sale"} // Set attribute (thuộc tính tập hợp)
}
```

**Giới Hạn Item:**
- Kích thước tối đa: **400 KB** mỗi item
- Partition Key: tối đa 2 KB
- Sort Key: tối đa 1 KB

### Các Kiểu Dữ Liệu DynamoDB

| Loại      | Ký Hiệu | Ví Dụ                                    |
| --------- | ------- | ---------------------------------------- |
| String    | S       | `"hello"`, `"user-001"`                  |
| Number    | N       | `42`, `3.14`, `-100`                     |
| Binary    | B       | Dữ liệu nhị phân (ảnh, file nén)        |
| Boolean   | BOOL    | `true`, `false`                          |
| Null      | NULL    | `null`                                   |
| List      | L       | `[1, "two", {three: 3}]`                 |
| Map       | M       | `{"key": "value"}`                       |
| String Set | SS     | `{"a", "b", "c"}`                        |
| Number Set | NS     | `{1, 2, 3}`                              |
| Binary Set | BS     | Tập hợp giá trị nhị phân                |

---

## Capacity Modes — Chế Độ Năng Lực

Xem chi tiết: [2-capacity-modes.md](./2-capacity-modes.md)

### On-Demand Mode (Chế Độ Theo Yêu Cầu)

DynamoDB tự động điều chỉnh theo traffic — không cần cấu hình trước.

```
Tính phí theo:
- RRU (Read Request Unit — Đơn Vị Request Đọc): $0.25/triệu RRU
- WRU (Write Request Unit — Đơn Vị Request Ghi): $1.25/triệu WRU
```

**Phù Hợp Khi:**
- Traffic không dự đoán được
- Ứng dụng mới chưa biết workload
- Workload có spike (đột biến) lớn và bất thường

### Provisioned Mode (Chế Độ Được Cung Cấp Sẵn)

Cấu hình số lượng RCU (Read Capacity Unit — Đơn Vị Năng Lực Đọc) và WCU (Write Capacity Unit — Đơn Vị Năng Lực Ghi) trước.

```
1 RCU = 1 strongly consistent read/s cho item ≤ 4 KB
      = 2 eventually consistent reads/s cho item ≤ 4 KB

1 WCU = 1 write/s cho item ≤ 1 KB
```

**Phù Hợp Khi:**
- Traffic dự đoán được và ổn định
- Muốn tối ưu chi phí (rẻ hơn On-Demand ~70%)
- Có thể kết hợp với Auto Scaling (Tự Động Co Giãn)

---

## Indexes — Chỉ Mục Phụ

Xem chi tiết: [3-indexes.md](./3-indexes.md)

### GSI — Global Secondary Index (Chỉ Mục Phụ Toàn Cầu)

```
Cho phép query với partition key khác hoàn toàn với base table.
Có thể tạo bất cứ lúc nào, tối đa 20 GSI mỗi bảng.
```

### LSI — Local Secondary Index (Chỉ Mục Phụ Cục Bộ)

```
Cùng partition key với base table, nhưng sort key khác.
Phải tạo lúc tạo bảng, tối đa 5 LSI mỗi bảng.
```

---

## DynamoDB Streams & Lambda

Xem chi tiết: [4-streams-lambda.md](./4-streams-lambda.md)

DynamoDB Streams (Luồng Dữ Liệu DynamoDB) ghi lại mọi thay đổi data (INSERT, UPDATE, DELETE) theo thứ tự thời gian, cho phép trigger Lambda để xử lý event-driven (hướng sự kiện).

---

## Transactions — Giao Dịch ACID

Xem chi tiết: [5-transactions-acid.md](./5-transactions-acid.md)

DynamoDB hỗ trợ ACID transactions (Giao Dịch ACID — Atomicity, Consistency, Isolation, Durability) qua API `TransactWriteItems` và `TransactGetItems`, cho phép thao tác nguyên tử trên tối đa 100 items hoặc 4 MB dữ liệu.

---

## DAX — DynamoDB Accelerator

Xem chi tiết: [6-dax.md](./6-dax.md)

DAX (DynamoDB Accelerator — Bộ Tăng Tốc DynamoDB) là in-memory cache (bộ nhớ đệm trong RAM) fully managed cho DynamoDB, giảm độ trễ từ millisecond xuống microsecond (micro-giây) cho read-heavy workloads.

---

## Access Patterns & Single-Table Design

Xem chi tiết: [7-access-patterns.md](./7-access-patterns.md)

Single-Table Design (Thiết Kế Đơn Bảng) là kỹ thuật lưu nhiều loại entity vào một bảng DynamoDB, sử dụng composite keys (khóa tổng hợp) và index overloading (tái sử dụng index) để phục vụ tất cả access patterns hiệu quả.

---

## So Sánh DynamoDB vs RDS

| Tiêu Chí                            | DynamoDB                                          | RDS/Aurora                                         |
| ----------------------------------- | ------------------------------------------------- | -------------------------------------------------- |
| **Loại**                            | NoSQL — key-value & document                      | SQL — relational (quan hệ)                         |
| **Schema**                          | Linh hoạt, định nghĩa lúc code                   | Cố định, định nghĩa trước                          |
| **Độ Trễ**                          | Single-digit ms, nhất quán ở mọi quy mô           | Thấp nhưng tăng theo quy mô                        |
| **Khả Năng Mở Rộng**               | Vô hạn, tự động theo chiều ngang (horizontal)    | Vertical scale (chiều dọc) + Read Replicas         |
| **Query**                           | Chỉ theo primary key hoặc index                   | JOIN, subquery, aggregation phức tạp               |
| **Transactions**                    | Có, nhưng giới hạn 100 items                      | Full ACID với nhiều bảng                           |
| **Consistency**                     | Eventually consistent (mặc định) hoặc Strong      | Strong consistency mặc định                        |
| **Giá**                             | Theo request + storage                            | Theo giờ (instance) + storage                      |
| **Quản Lý**                         | Serverless, không cần quản lý                     | Cần chọn instance, patching, scaling               |
| **Use Case Điển Hình**              | Gaming, IoT, session, shopping cart               | E-commerce, ERP, CRM, banking                      |

### Quy Tắc Chọn Lựa

```
Chọn DynamoDB khi:
✅ Cần < 10ms latency ở quy mô lớn
✅ Access patterns đơn giản và rõ ràng (biết trước)
✅ Không cần JOIN phức tạp
✅ Traffic thất thường, cần auto-scale nhanh
✅ Serverless architecture (kiến trúc không máy chủ)

Chọn RDS/Aurora khi:
✅ Cần JOIN nhiều bảng
✅ Schema phức tạp với nhiều relationship
✅ Ad-hoc queries từ analyst/BI tools
✅ Strong consistency là yêu cầu bắt buộc
✅ Team đã quen SQL
```

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Partition Key (Khóa Phân Vùng) tốt có những đặc điểm gì?**

A: Partition key tốt phải có **cardinality cao** (nhiều giá trị khác nhau) và **phân bố đều** traffic. Ví dụ: `UserId` (UUID) tốt hơn `Status` (chỉ vài giá trị). Partition key xấu gây hot partition (phân vùng nóng) — một phân vùng nhận quá nhiều traffic, dẫn đến throttling (giới hạn tốc độ).

**Q: Khi nào dùng DynamoDB thay vì RDS?**

A: Dùng DynamoDB khi: (1) cần độ trễ < 10ms ở quy mô lớn; (2) access patterns rõ ràng, không cần JOIN; (3) cần serverless, tự động scale; (4) traffic không dự đoán được. Dùng RDS khi cần SQL phức tạp, JOIN nhiều bảng, hay ad-hoc queries.

**Q: Sự khác biệt giữa On-Demand và Provisioned capacity?**

A: On-Demand tính phí theo từng request, không cần cấu hình trước — phù hợp traffic không đều. Provisioned đặt trước RCU/WCU, rẻ hơn ~70% nếu traffic đoán được, có thể kết hợp Auto Scaling. Throttling xảy ra khi vượt quá provisioned capacity.

### Câu Hỏi Nâng Cao

**Q: Giải thích hot partition và cách tránh?**

A: Hot partition xảy ra khi nhiều request tập trung vào một partition — thường do partition key có cardinality thấp (ví dụ: `date`, `status`) hoặc viral item (một item được truy cập nhiều bất thường). Cách tránh: (1) chọn partition key có cardinality cao; (2) write sharding — thêm suffix ngẫu nhiên vào key; (3) dùng DAX để giảm tải read; (4) caching tại application layer.

**Q: GSI vs LSI — khi nào dùng loại nào?**

A: LSI (Local Secondary Index — Chỉ Mục Phụ Cục Bộ) dùng khi cần query trên cùng partition key nhưng sort key khác, phải tạo lúc tạo bảng, dùng storage của base table. GSI (Global Secondary Index — Chỉ Mục Phụ Toàn Cầu) dùng khi cần partition key hoàn toàn khác, tạo được bất cứ lúc nào, có capacity riêng. GSI linh hoạt hơn nhưng chỉ hỗ trợ eventually consistent reads.

**Q: DynamoDB xử lý eventual consistency (nhất quán cuối cùng) như thế nào?**

A: Mặc định, DynamoDB reads là eventually consistent — data vừa ghi có thể chưa phản ánh ngay trên tất cả replicas (bản sao). Strongly consistent reads (đọc nhất quán mạnh) đảm bảo data mới nhất nhưng tốn 2x RCU và không available qua GSI. Trong thực tế, eventually consistent đủ cho hầu hết use cases; chỉ dùng strongly consistent khi business logic yêu cầu.

---

## 📁 Nội Dung Chi Tiết

| File                                            | Chủ Đề                                                    |
| ----------------------------------------------- | --------------------------------------------------------- |
| [1-data-model.md](./1-data-model.md)           | Tables, Partition Key, Sort Key, Attributes               |
| [2-capacity-modes.md](./2-capacity-modes.md)   | On-Demand vs Provisioned, Auto Scaling, Throttling        |
| [3-indexes.md](./3-indexes.md)                 | GSI, LSI — Thiết Kế và Use Cases                         |
| [4-streams-lambda.md](./4-streams-lambda.md)   | DynamoDB Streams, Lambda Integration                      |
| [5-transactions-acid.md](./5-transactions-acid.md) | Transactions, ACID, TransactWrite/Get               |
| [6-dax.md](./6-dax.md)                         | DAX — DynamoDB Accelerator, In-Memory Cache               |
| [7-access-patterns.md](./7-access-patterns.md) | Single-Table Design, Access Patterns, Best Practices      |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn Thành
