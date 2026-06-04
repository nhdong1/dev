# Amazon MSK — Managed Streaming for Apache Kafka (Kafka Được Quản Lý)

> Amazon MSK (Managed Streaming for Apache Kafka) là dịch vụ fully managed (hoàn toàn được quản lý) giúp bạn xây dựng và vận hành ứng dụng sử dụng Apache Kafka để xử lý streaming data (dữ liệu luồng thời gian thực) mà không cần tự quản lý hạ tầng Kafka.

## 📚 Mục Lục Module

1. [Tổng Quan MSK](#tổng-quan-msk)
2. [Các File Trong Module](#các-file-trong-module)
3. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
4. [Khi Nào Dùng MSK](#khi-nào-dùng-msk)
5. [MSK vs Kinesis — Chọn Cái Nào](#msk-vs-kinesis--chọn-cái-nào)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## 🎯 Tổng Quan MSK

### MSK Là Gì?

Amazon MSK là dịch vụ managed Kafka trên AWS. AWS lo toàn bộ việc:
- Provision (Cung Cấp) broker instances (máy chủ broker)
- Cài đặt, vá lỗi và cập nhật phần mềm Kafka & ZooKeeper/KRaft
- Monitoring (Giám Sát) và alerting (Cảnh Báo) cơ bản
- Replicate (Sao Chép) dữ liệu giữa các Availability Zones (Vùng Sẵn Sàng)
- Tự động thay thế broker khi bị lỗi

Bạn chỉ cần quan tâm đến: Topics (Chủ Đề), Producers (Nhà Sản Xuất), Consumers (Người Tiêu Thụ) và business logic (Logic Nghiệp Vụ).

### Các Phiên Bản MSK

| Phiên Bản                   | Mô Tả                                                                   |
| --------------------------- | ----------------------------------------------------------------------- |
| **MSK Provisioned**         | Tự chọn broker type và số lượng — dự đoán được chi phí và hiệu suất    |
| **MSK Serverless**          | Auto-scale (Tự Động Mở Rộng) không cần quản lý capacity (dung lượng)   |
| **MSK Connect**             | Managed Kafka Connect (Kết Nối Kafka Được Quản Lý) — connector workers |
| **MSK Replicator**          | Sao chép dữ liệu giữa các MSK cluster cross-region (nhiều vùng)        |

---

## 📁 Các File Trong Module

| File                        | Nội Dung                                                                      | Độ Khó |
| --------------------------- | ----------------------------------------------------------------------------- | ------ |
| `README.md`                 | Tổng quan, kiến trúc tổng quát, khi nào dùng MSK                             | ⭐     |
| `1-msk-architecture.md`     | Brokers, Topics, Partitions, Consumer Groups — kiến trúc chi tiết            | ⭐⭐⭐ |
| `2-msk-vs-kinesis.md`       | So sánh MSK và Kinesis Data Streams — trade-offs, tiêu chí lựa chọn          | ⭐⭐   |
| `3-msk-security.md`         | TLS, SASL/SCRAM, IAM Authentication — bảo mật toàn diện                      | ⭐⭐⭐ |

---

## 🏗️ Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                        Amazon MSK Cluster                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    ZooKeeper / KRaft                      │   │
│  │           (Quản lý metadata và leader election)           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
│  │  Broker 1   │   │  Broker 2   │   │  Broker 3   │           │
│  │  AZ: us-    │   │  AZ: us-    │   │  AZ: us-    │           │
│  │  east-1a    │   │  east-1b    │   │  east-1c    │           │
│  │             │   │             │   │             │           │
│  │  Topic-A    │   │  Topic-A    │   │  Topic-A    │           │
│  │  Part-0(L)  │   │  Part-1(L)  │   │  Part-0(R)  │           │
│  │  Part-1(R)  │   │  Part-2(L)  │   │  Part-1(R)  │           │
│  └─────────────┘   └─────────────┘   └─────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
        ↑                                         ↓
  Producers                                  Consumers
  (Nhà Sản Xuất)                         (Người Tiêu Thụ)
  
  - Microservices         Consumer Group A    Consumer Group B
  - IoT devices           (Order Service)     (Analytics App)
  - Log shippers          - Consumer 1        - Consumer 1
  - CDC connectors        - Consumer 2        - Consumer 2
```

- **L** = Leader Partition (Phân Vùng Dẫn Đầu) — nhận reads/writes
- **R** = Replica Partition (Phân Vùng Bản Sao) — sao lưu dự phòng

---

## 📋 Khi Nào Dùng MSK

### ✅ Nên Chọn MSK Khi

- **Đã có Kafka ecosystem:** Đang dùng Kafka on-premises (tại chỗ) và muốn migrate (di chuyển) lên cloud
- **Kafka-native features:** Cần Kafka Streams, Kafka Connect, KSQL — các công cụ chỉ có trong Kafka ecosystem
- **High throughput (Thông Lượng Cao):** Hàng triệu messages/giây với latency (độ trễ) thấp
- **Multi-consumer patterns:** Nhiều consumer group (nhóm người tiêu thụ) độc lập đọc cùng một topic
- **Long retention (Lưu Giữ Lâu Dài):** Cần lưu dữ liệu weeks/months (không giới hạn như Kinesis 7 ngày)
- **Kafka Streams / KSQL:** Xử lý stateful streaming (luồng có trạng thái) phức tạp

### ❌ Không Nên Chọn MSK Khi

- **AWS-native workloads:** Muốn tích hợp đơn giản với Lambda, S3, Firehose → Dùng Kinesis
- **Simple fan-out (Phân Phối Đơn Giản):** 1-2 consumer không cần Kafka full feature set
- **Serverless & low ops:** Không muốn quản lý bất kỳ broker configuration nào
- **Cost-sensitive small workloads:** MSK có overhead chi phí tối thiểu cho broker instances

---

## ⚡ MSK vs Kinesis — Chọn Cái Nào

| Tiêu Chí                        | MSK (Kafka)                         | Kinesis Data Streams               |
| ------------------------------- | ------------------------------------ | ---------------------------------- |
| **Độ phức tạp vận hành**        | Trung bình (MSK lo infra, bạn lo Kafka config) | Thấp (fully managed hoàn toàn)  |
| **Throughput tối đa**           | Rất cao (không giới hạn theo thiết kế) | Cao (1 MB/s mỗi shard)           |
| **Retention (Lưu Giữ)**         | Không giới hạn (tuỳ cấu hình)       | Tối đa 365 ngày                    |
| **Consumer model**              | Pull (Kéo) — consumer tự kéo data   | Pull + Push (Lambda trigger)       |
| **Tích hợp AWS native**         | Trung bình (cần Lambda ESM)          | Cao (Lambda, Firehose tích hợp sẵn)|
| **Kafka ecosystem**             | ✅ Đầy đủ                            | ❌ Không có                        |
| **Chi phí khởi điểm**           | Cao hơn (phải có ít nhất 2-3 brokers)| Thấp hơn (pay per shard)          |
| **Replay (Phát Lại Dữ Liệu)**   | ✅ Linh hoạt theo offset              | ✅ Theo sequence number            |

**Quy tắc đơn giản:**
- Đã có Kafka workload → MSK
- Bắt đầu mới, AWS-native → Kinesis Data Streams

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

1. MSK khác Kafka tự dựng (self-managed) thế nào?
2. Giải thích Consumer Group (Nhóm Người Tiêu Thụ) và tại sao nó quan trọng
3. Khi nào bạn chọn MSK thay vì Kinesis Data Streams?
4. MSK Serverless khác MSK Provisioned thế nào?
5. Làm thế nào để secure (bảo mật) MSK cluster?

Xem chi tiết trong các file con của module này.

---

## 🔗 Điều Hướng Module

| Bước | File                    | Nội Dung                      |
| ---- | ----------------------- | ----------------------------- |
| 1    | `1-msk-architecture.md` | Nắm vững kiến trúc Kafka/MSK  |
| 2    | `2-msk-vs-kinesis.md`   | So sánh & tiêu chí lựa chọn  |
| 3    | `3-msk-security.md`     | Bảo mật production-grade      |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Module:** 10/12
**Trạng Thái:** ✅ Hoàn Thành
