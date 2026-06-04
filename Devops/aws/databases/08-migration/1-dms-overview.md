# AWS DMS — Database Migration Service: Tổng Quan

> AWS DMS (Database Migration Service — Dịch Vụ Di Chuyển Cơ Sở Dữ Liệu) là dịch vụ managed giúp di chuyển database lên AWS với minimal downtime (thời gian ngừng hoạt động tối thiểu). DMS hỗ trợ cả homogeneous migration (di chuyển cùng loại engine) và heterogeneous migration (di chuyển khác loại engine) kết hợp với SCT.

## 📚 Mục Lục

1. [Kiến Trúc DMS](#kiến-trúc-dms)
2. [Replication Instance](#replication-instance)
3. [Endpoints](#endpoints)
4. [Task Types — Loại Task](#task-types)
5. [Monitoring & Troubleshooting](#monitoring--troubleshooting)
6. [Giới Hạn & Lưu Ý](#giới-hạn--lưu-ý)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🏗️ Kiến Trúc DMS

### Các Thành Phần Chính

```
┌───────────────────────────────────────────────────────────────────┐
│                       AWS DMS Architecture                        │
│                                                                   │
│  Source Endpoint          Replication Instance    Target Endpoint │
│  ───────────────          ────────────────────    ─────────────── │
│  ┌─────────────┐         ┌──────────────────┐    ┌─────────────┐ │
│  │  On-prem    │         │   EC2 Instance   │    │   Aurora    │ │
│  │  Oracle DB  │────────▶│  ┌────────────┐  │───▶│  PostgreSQL │ │
│  └─────────────┘  source │  │Replication │  │    └─────────────┘ │
│                  endpoint│  │   Task     │  │target              │
│  ┌─────────────┐         │  └────────────┘  │   endpoint         │
│  │  RDS MySQL  │         │                  │                    │
│  │  (source)   │         │  DMS manages:    │                    │
│  └─────────────┘         │  - Full Load     │                    │
│                           │  - CDC streaming │                    │
│                           └──────────────────┘                   │
└───────────────────────────────────────────────────────────────────┘
```

DMS hoạt động theo mô hình **3 thành phần**:
1. **Source Endpoint** — kết nối đến database nguồn
2. **Replication Instance** — máy chủ EC2 chạy quá trình di chuyển
3. **Target Endpoint** — kết nối đến database đích

---

## 🖥️ Replication Instance

### Là Gì?

Replication instance (máy chủ sao chép) là một EC2 instance được AWS DMS quản lý. Đây là nơi DMS đọc dữ liệu từ source, xử lý, rồi ghi vào target.

### Lựa Chọn Instance Class

```
Nhỏ  — dms.t3.micro / dms.t3.small
  Dùng cho: development, testing, database nhỏ < 50 GB

Vừa  — dms.c5.large / dms.c5.xlarge
  Dùng cho: production database trung bình, heterogeneous migration

Lớn  — dms.c5.4xlarge / dms.r5.4xlarge
  Dùng cho: database lớn > 1 TB, nhiều task song song
```

**Nguyên tắc chọn instance:**
- Dùng instance memory lớn hơn nếu có nhiều LOB (Large Object — Đối Tượng Lớn) như BLOB, CLOB
- Dùng instance compute lớn hơn nếu có nhiều transformations (biến đổi dữ liệu)
- Multi-AZ replication instance cho production (tránh mất task khi AZ fail)

### Storage của Replication Instance

DMS dùng local storage để:
- Buffer (đệm) dữ liệu trong quá trình Full Load
- Lưu CDC events tạm thời trước khi apply vào target
- Cache schema metadata

Mặc định 50 GB. Tăng nếu:
- Database source lớn (> 200 GB)
- CDC lag lớn (network chậm giữa source và DMS)
- Nhiều LOB columns

---

## 🔌 Endpoints

### Source Endpoints — Điểm Cuối Nguồn

DMS hỗ trợ nguồn:

| Loại | Database                                                          |
| ---- | ----------------------------------------------------------------- |
| SQL  | Oracle, SQL Server, MySQL, MariaDB, PostgreSQL, SAP ASE (Sybase) |
| AWS  | RDS (tất cả engine), Aurora, S3 (nguồn file CSV/Parquet)         |
| NoSQL| MongoDB, DocumentDB                                               |

**Cấu hình source endpoint cần:**
- Hostname / ARN của database
- Port, username, password
- SSL mode (nếu dùng mã hóa kết nối)
- Extra connection attributes (ví dụ: bật CDC cho PostgreSQL cần `slotName`)

### Target Endpoints — Điểm Cuối Đích

DMS hỗ trợ đích:

| Loại   | Database / Service                                                  |
| ------ | ------------------------------------------------------------------- |
| SQL    | RDS (MySQL, PostgreSQL, Oracle, SQL Server), Aurora                 |
| NoSQL  | DynamoDB, MongoDB, DocumentDB                                       |
| Data   | S3, Kinesis, Kafka, OpenSearch, Redshift                            |

### Test Endpoint Connection

Trước khi tạo migration task, luôn test connection:

```
DMS Console → Endpoints → Test Connection
→ Chọn replication instance
→ Click "Run Test"
→ Xem kết quả: successful / failed + error message
```

Lỗi thường gặp:
- Security group chưa mở port từ replication instance đến database
- IAM role thiếu permission (cho S3/Kinesis targets)
- Binary log chưa được bật trên MySQL source
- Replication slot chưa được tạo trên PostgreSQL source

---

## ⚙️ Task Types — Loại Task

### 1. Full Load (Tải Đầy Đủ)

```
Đặc điểm:
- Copy toàn bộ dữ liệu từ source sang target
- Dừng khi hoàn thành
- Không theo dõi thay đổi mới sau đó

Khi dùng:
- Database nhỏ có thể chấp nhận downtime
- Snapshot migration (di chuyển ảnh chụp)
- Initial load cho data warehouse

Flow:
  Source DB ──[SELECT * FROM tables]──▶ DMS ──[INSERT INTO]──▶ Target DB
```

### 2. Full Load + CDC (Phổ Biến Nhất)

```
Đặc điểm:
- Giai đoạn 1: Full Load (copy toàn bộ dữ liệu hiện có)
- Giai đoạn 2: CDC (tiếp tục đồng bộ thay đổi real-time)
- Source tiếp tục nhận writes trong suốt quá trình

Khi dùng:
- Production database cần zero hoặc minimal downtime
- Database lớn — full load mất nhiều giờ/ngày
- Trường hợp phổ biến nhất trong thực tế

Timeline:
  [Full Load ~6 giờ] → [CDC catchup ~2 giờ] → [CDC lag ≈ 0] → Cutover
```

**Quá trình chi tiết Full Load + CDC:**

```
1. DMS ghi lại SCN (System Change Number — Số Thay Đổi Hệ Thống) /
   LSN (Log Sequence Number — Số Thứ Tự Log) tại thời điểm bắt đầu load

2. Full Load: DMS đọc toàn bộ dữ liệu và ghi vào target
   - Source vẫn nhận writes bình thường
   - Các thay đổi trong giai đoạn này được lưu vào transaction log

3. CDC: Sau full load, DMS đọc transaction log từ SCN/LSN đã ghi
   - Apply các INSERT/UPDATE/DELETE bị bỏ lỡ trong full load
   - Dần bắt kịp với hiện tại (lag giảm)

4. Steady state: DMS liên tục apply thay đổi, lag ổn định < vài giây
```

### 3. CDC Only (Chỉ CDC)

```
Đặc điểm:
- Chỉ đồng bộ thay đổi, không load dữ liệu ban đầu
- Cần target đã có dữ liệu trước (pre-populated)

Khi dùng:
- Đã load dữ liệu bằng native dump/restore (nhanh hơn DMS Full Load)
  rồi dùng DMS chỉ để CDC
- Replicate dữ liệu ongoing sang data warehouse
- Fan-out: nhân bản sang nhiều target khác nhau
```

### So Sánh Task Types

| Tiêu Chí              | Full Load | Full Load + CDC | CDC Only  |
| --------------------- | --------- | --------------- | --------- |
| Dữ liệu ban đầu       | ✅        | ✅              | ❌        |
| Đồng bộ liên tục      | ❌        | ✅              | ✅        |
| Zero-downtime         | ❌        | ✅              | ✅        |
| Phức tạp              | Thấp      | Trung           | Thấp      |
| Yêu cầu transaction log | Không   | Có (CDC phase)  | Có        |

---

## 📊 Table Mapping & Transformation

### Table Mapping Rules

DMS dùng JSON rules để chọn table và cột cần di chuyển:

```json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-all-tables",
      "object-locator": {
        "schema-name": "myapp",
        "table-name": "%"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "selection",
      "rule-id": "2",
      "rule-name": "exclude-audit-table",
      "object-locator": {
        "schema-name": "myapp",
        "table-name": "audit_log"
      },
      "rule-action": "exclude"
    }
  ]
}
```

### Transformation Rules

Đổi tên schema, table, hoặc column trong quá trình di chuyển:

```json
{
  "rule-type": "transformation",
  "rule-id": "3",
  "rule-name": "lowercase-all",
  "rule-action": "convert-lowercase",
  "rule-target": "schema",
  "object-locator": {
    "schema-name": "%"
  }
}
```

Ví dụ use case: Oracle dùng UPPERCASE schema/table, PostgreSQL mặc định lowercase → cần transformation để tránh case mismatch.

---

## 📈 Monitoring & Troubleshooting

### Các Metric Quan Trọng trong DMS

| Metric                         | Ý Nghĩa                                              | Ngưỡng Cần Chú Ý       |
| ------------------------------ | ---------------------------------------------------- | ----------------------- |
| **CDCLatencySource**           | Độ trễ giữa source change và DMS đọc được           | > 60 giây → cần kiểm tra |
| **CDCLatencyTarget**           | Độ trễ giữa DMS đọc và target nhận được             | > 60 giây → bottleneck  |
| **FullLoadThroughputRowsSource** | Tốc độ đọc rows từ source (full load)             | Phụ thuộc size          |
| **FullLoadThroughputRowsTarget** | Tốc độ ghi rows vào target (full load)            | Phụ thuộc size          |
| **CPUUtilization**             | CPU của replication instance                         | > 80% → scale up        |
| **FreeStorageSpace**           | Dung lượng còn lại trên replication instance         | < 10% → tăng storage    |

### Lỗi Thường Gặp & Cách Xử Lý

**Lỗi 1: Full load không hoàn thành / treo**

```
Nguyên nhân:
- Source database bị locked (khóa)
- Network timeout giữa DMS và source/target
- Target database hết dung lượng

Xử lý:
- Check DMS task logs trong CloudWatch
- Tăng timeout trong endpoint extra attributes
- Resume task (DMS hỗ trợ resume từ điểm dừng)
```

**Lỗi 2: CDC lag tăng không giảm**

```
Nguyên nhân:
- Target database quá chậm (insert bottleneck)
- Replication instance CPU/memory quá thấp
- Transaction log trên source bị purge (xóa) trước khi DMS đọc

Xử lý:
- Scale up replication instance
- Tăng write capacity trên target (nếu DynamoDB: tăng WCU)
- Tăng log retention trên source
```

**Lỗi 3: Data type errors (lỗi kiểu dữ liệu)**

```
Nguyên nhân:
- Kiểu dữ liệu không tương thích (ví dụ: Oracle NUMBER → PostgreSQL NUMERIC)
- LOB columns quá lớn
- Character set không khớp

Xử lý:
- Dùng SCT để kiểm tra trước
- Cấu hình LOB mode trong DMS task settings
- Set target_schema trên endpoint
```

---

## ⚠️ Giới Hạn & Lưu Ý

### Những Gì DMS KHÔNG Làm Tự Động

```
✗ Chuyển đổi stored procedures (thủ tục lưu trữ)
✗ Chuyển đổi triggers
✗ Chuyển đổi views phức tạp
✗ Tạo sequences (PostgreSQL/Oracle)
✗ Migrate user permissions (phân quyền người dùng)
✗ Di chuyển foreign key constraints (ràng buộc khóa ngoại) trong full load
  (DMS disable FK checks, re-enable sau)
```

→ Các object này cần xử lý thủ công hoặc qua SCT.

### LOB — Large Object Handling

LOB (Large Object — Đối Tượng Lớn) gồm BLOB (Binary Large Object), CLOB (Character Large Object), NCLOB:

```
LOB Mode Options:
  Limited LOB mode — Chỉ copy LOB đến giới hạn kích thước
    - Nhanh hơn
    - Cắt bớt LOB lớn hơn giới hạn → mất dữ liệu nếu không cẩn thận

  Full LOB mode — Copy toàn bộ LOB
    - Chậm hơn (DMS phải SELECT riêng từng LOB row)
    - Đảm bảo không mất dữ liệu

Recommendation: Dùng Full LOB mode cho production
```

### Parallel Load — Tải Song Song

DMS có thể tải nhiều tables song song:

```
MaxFullLoadSubTasks: Số table load song song (default: 8, max: 49)
ParallelLoadQueuesPerThread: Hàng đợi song song trong mỗi table

Tăng giá trị này khi:
- Source và target instance mạnh
- Di chuyển nhiều table nhỏ
- Muốn giảm tổng thời gian full load
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: AWS DMS là gì và kiến trúc của nó gồm những thành phần nào?**

> DMS là managed service giúp di chuyển database lên AWS với minimal downtime. Kiến trúc gồm 3 thành phần: source endpoint (kết nối database nguồn), replication instance (EC2 instance thực hiện di chuyển), và target endpoint (kết nối database đích). DMS đọc dữ liệu từ source, apply bất kỳ transformation nào được cấu hình, rồi ghi vào target.

**Q: Sự khác biệt giữa Full Load, Full Load + CDC, và CDC Only?**

> Full Load copy toàn bộ dữ liệu một lần, thích hợp khi chấp nhận downtime. Full Load + CDC là phổ biến nhất: load dữ liệu ban đầu rồi tiếp tục đồng bộ thay đổi real-time, cho phép zero-downtime migration. CDC Only chỉ đồng bộ thay đổi, dùng khi đã load dữ liệu ban đầu bằng cách khác (ví dụ: native backup/restore nhanh hơn).

**Q: DMS không tự động xử lý những gì? Cần làm gì thêm?**

> DMS không tự động chuyển stored procedures, triggers, views phức tạp, sequences, và user permissions. Với heterogeneous migration cần dùng SCT (Schema Conversion Tool) để chuyển đổi schema và các database objects này. Một số objects SCT cũng không chuyển được 100% và cần chỉnh sửa thủ công.

**Q: Nếu CDCLatencySource tăng cao, nguyên nhân có thể là gì?**

> CDCLatencySource tăng cao thường do: (1) replication instance không đủ tài nguyên để đọc transaction log, (2) network chậm giữa source và DMS, (3) source database tải cao làm chậm log generation, hoặc (4) transaction log trên source bị purge quá sớm. Cần scale up replication instance, kiểm tra network, và tăng log retention period.

---

**Liên Kết:**
- [README.md](./README.md) — Tổng quan migration
- [2-sct.md](./2-sct.md) — Schema Conversion Tool
- [3-cdc-online-migration.md](./3-cdc-online-migration.md) — CDC chi tiết
- [5-cutover-runbook.md](./5-cutover-runbook.md) — Cutover planning

**Cập Nhật Lần Cuối:** 2026-05-15
