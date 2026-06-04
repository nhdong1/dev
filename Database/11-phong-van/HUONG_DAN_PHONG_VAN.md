# Hướng Dẫn Chuẩn Bị Phỏng Vấn DBA

Hướng dẫn toàn diện để vượt qua phỏng vấn Quản Trị Cơ Sở Dữ Liệu.

## Tổng Quan Định Dạng Phỏng Vấn

| Loại                 | Thời gian  | Số câu hỏi | Trọng tâm                          |
| -------------------- | ---------- | ---------- | ---------------------------------- |
| Phone Screen         | 30-45 phút | 2-3        | Nền tảng, động lực, tổng quan      |
| Vòng Kỹ Thuật 1      | 60 phút    | 4-5        | Kiến thức cơ bản, tình huống       |
| Vòng Kỹ Thuật 2      | 90 phút    | 2-3        | Deep dive + câu chuyện sự cố       |
| Thiết Kế Hệ Thống    | 90 phút    | 1-2        | Thiết kế hệ thống phức tạp         |
| Văn Hóa/Phù Hợp      | 30 phút    | -          | Giá trị, làm việc nhóm, phát triển |

---

## Top 20 Câu Hỏi Phỏng Vấn DBA

### Cấp 1: Kiến Thức Cơ Bản (Bắt Buộc Phải Biết)

#### 1. Giải thích RPO và RTO. Thiết kế chiến lược backup cho CSDL mission-critical?

**Họ tìm kiếm gì:**
- Hiểu yêu cầu nghiệp vụ
- Khả năng chuyển đổi yêu cầu thành quyết định kỹ thuật
- Kiến thức về loại backup và đánh đổi
- Kinh nghiệm thực tế với công cụ backup

**Cấu trúc câu trả lời:**

```
1. Định nghĩa RPO & RTO
   RPO = mức mất dữ liệu tối đa chấp nhận (thời gian)
   RTO = thời gian ngừng tối đa chấp nhận (thời gian)

2. Bối cảnh nghiệp vụ
   "Cho hệ thống mission-critical:"
   - RPO thường < 1 giờ (thường 15-30 phút)
   - RTO thường < 30 phút

3. Thiết kế chiến lược
   - Full backup hàng tuần (baseline)
   - Differential/incremental hàng ngày
   - Transaction log backups mỗi 15 phút
   - WAL archiving cho PITR
   - Bản sao offsite (region khác)

4. Triển Khai
   - Dùng managed backup nếu có (RDS, Cloud SQL)
   - Hoặc self-managed với pgBackRest, Barman
   - Tự động hóa kiểm tra khôi phục hàng tháng

5. Xác Minh
   - Kiểm tra khôi phục từ backup hàng quý
   - Xác minh checksums
   - Tài liệu hóa RTO trong điều kiện thực tế
```

---

#### 2. Mô tả sự cố hiệu suất CSDL bạn đã xử lý. Bạn kiểm tra những chỉ số nào?

**Đây là câu chuyện sự cố của bạn (dùng định dạng STAR)**

**S - Tình Huống:**
"Tôi có CSDL PostgreSQL hỗ trợ nền tảng thương mại điện tử. Trong sự kiện sale cao điểm (lưu lượng cao), khách hàng báo cáo tải trang chậm và timeout."

**T - Nhiệm Vụ:**
"Là DBA on-call, tôi cần xác định nguyên nhân gốc và khôi phục hiệu suất trong vòng 30 phút."

**A - Hành Động:**
"Quy trình xử lý sự cố của tôi:

1. Kết nối CSDL và kiểm tra top wait events
   - IO wait cao trên bảng cụ thể

2. Phân tích truy vấn chậm
   - SELECT trên bảng 'orders' quét 10 triệu hàng
   - Index bị thiếu trên cột lọc

3. Kiểm tra kế hoạch truy vấn
   - EXPLAIN ANALYZE hiển thị sequential scan
   - Nên dùng index nhưng chưa được tạo

4. Giải pháp tạm thời
   - Thêm index trên cột order_status
   - Rebuild statistics
   - Kiểm tra truy vấn (1.5s → 50ms)

5. Giải pháp dài hạn
   - Phân tích access patterns
   - Thêm composite index (user_id, order_status)
   - Đảm bảo replica có cùng index

6. Giám sát
   - Thêm cảnh báo cho sequential scans trên bảng lớn
   - Đặt ngưỡng cho duration truy vấn > 5s"

**R - Kết Quả:**
"Hiệu suất được khôi phục trong 10 phút. Thời gian phản hồi từ 8s xuống 200ms. Phòng ngừa: Thêm khuyến nghị index tự động vào quy trình deployment."

---

#### 3. Sự khác biệt giữa blocking và deadlock? Làm thế nào giảm thiểu?

**Blocking (Chặn):**

```
Transaction A khóa hàng 1
Transaction B cố truy cập hàng 1
B phải CHỜI (block) cho đến khi A giải phóng khóa

Thời gian: Ngắn (giây) = bình thường
Thời gian: Dài (phút) = vấn đề
```

**Nguyên nhân:**
- Transaction chạy dài giữ khóa
- Index bị thiếu gây lock escalation
- Lỗi ứng dụng (kết nối không được trả về)

**Giảm thiểu blocking:**
- Giữ transaction ngắn
- Tạo index phù hợp
- Dùng isolation level thấp hơn nếu chấp nhận được
- Giám sát và kill truy vấn chạy lâu

**Deadlock:**

```
Transaction A khóa hàng 1, chờ hàng 2
Transaction B khóa hàng 2, chờ hàng 1

Chờ vòng tròn → DEADLOCK
→ CSDL phát hiện và kill một transaction (victim)
```

**Nguyên nhân:**
- Thứ tự khóa không nhất quán trong ứng dụng
- Logic transaction phức tạp
- Đồng thời cao

**Giảm thiểu deadlock:**
- Khóa tài nguyên theo cùng thứ tự
- Giữ transaction nhỏ
- Thêm index phù hợp
- Triển khai retry logic (exponential backoff)

---

#### 4. Thiết kế kiến trúc HA cho CSDL với yêu cầu uptime 99.99%

**Tính toán uptime:**

```
99.99% = 52.6 phút downtime mỗi năm
       = ~8.64 giây mỗi ngày

Đây là RẤT chặt chẽ → cần failover tự động
```

**Kiến Trúc:**

```
┌─────────────────────────────────────┐
│     Load Balancer (DNS/VIP)        │
└───────────────┬─────────────────────┘
                │
    ┌───────────┴───────────┐
    ▼                       ▼
[PRIMARY]              [REPLICA]
PostgreSQL 14         PostgreSQL 14
(Multi-AZ)           (Cùng AZ ban đầu)
    │                      │
    └──────Streaming───────┘
           Replication

┌─────────────────────────────────────┐
│   Failover Tự Động (Patroni)        │
│   - Health check mỗi 10s           │
│   - Thăng cấp replica nếu primary  │
│   - Cập nhật DNS/VIP                │
│   - Ngăn split-brain (etcd)         │
└─────────────────────────────────────┘

Thêm:
- Backup lên S3 (độc lập với replication)
- Giám sát + cảnh báo
- RTO < 1 phút (failover tự động)
- RPO < 15 giây (sync replication)
```

**Giải thích quyết định chính:**
- **Synchronous replication** → RPO = 0 (không mất dữ liệu)
- **Failover tự động** → RTO < 1 phút (ngân sách uptime chặt)
- **Quorum + etcd** → Ngăn split-brain
- **Backup độc lập** → Bảo vệ khỏi lỗi ứng dụng

---

### Cấp 2: Tình Huống Trung Cấp

#### 5. CSDL của bạn tăng 20% mỗi tháng. Làm thế nào lập kế hoạch dung lượng?

**Phân tích xu hướng:**

```
Kích thước hiện tại: 200GB
Tốc độ tăng: 20% mỗi tháng

6 tháng: 200GB × 1.2^6 = 579GB
12 tháng: 200GB × 1.2^12 = 1.5TB
```

**Các yếu tố lập kế hoạch dung lượng:**

1. **Tăng trưởng dữ liệu** (như trên)
2. **Chi phí index** (thường 10-30% dữ liệu)
3. **WAL/logs** (xem xét chiến lược log archiving)
4. **Không gian tạm** (bảo trì, temp tables)
5. **Replicas** (nhân với số replica)
6. **Backups** (thời gian lưu giữ × kích thước backup hàng ngày)
7. **Hiệu suất** (headroom cho spikes)

**Quyết định mở rộng:**

```
Mở rộng dọc (server lớn hơn):
- Ưu: Đơn giản, không cần thay đổi ứng dụng
- Nhược: Đắt tiền, rủi ro điểm lỗi đơn

Mở rộng ngang (sharding):
- Ưu: Mở rộng không giới hạn, phân phối tải
- Nhược: Phức tạp, cần thay đổi ứng dụng

Quy tắc: Dọc trước (dễ hơn), sharding khi cần (> 2TB thường)
```

---

#### 6. Hướng dẫn tôi qua schema migration cho production (không thời gian chết)

**Thách thức:**
Thêm cột mới vào bảng được dùng nhiều mà không có thời gian chết.

**Mẫu Expand-Contract:**

```
GIAI ĐOẠN 1: EXPAND (Triển khai)
- Thêm cột nullable mới
- Thêm default hoặc NULL
- Ứng dụng hiện tại vẫn dùng cột cũ
- Deploy code bỏ qua cột mới

GIAI ĐOẠN 2: MIGRATE (Song song)
- Backfill dữ liệu theo batch (tránh khóa)
  UPDATE users SET new_column = old_column
  WHERE id BETWEEN 1 AND 100000;

- Dual-write: Code ghi vào CẢ HAI cột
- Giám sát vấn đề nhất quán
- Tiếp tục cho đến khi backfill đầy đủ

GIAI ĐOẠN 3: CONTRACT (Dọn dẹp)
- Thay đổi code đọc từ cột MỚI
- Deploy lên tất cả instances
- Chờ traffic ổn định
- Giám sát lỗi
- Xóa cột cũ (sau vài ngày/tuần)
```

---

### Cấp 3: Tình Huống Thực Tế

#### 7. Cửa sổ cắt chuyển 48 giờ cho migration lớn. Checklist pre-cutover?

**Checklist Pre-Cutover:**

```
72 Giờ Trước:
☐ Full backup cuối cùng của CSDL nguồn
☐ Kiểm tra khôi phục sang môi trường staging
☐ Chạy migration script trên CSDL staging
☐ Xác thực row counts khớp chính xác
☐ Chạy smoke tests ứng dụng
☐ Brief tất cả nhóm về kế hoạch và runbook

24 Giờ Trước:
☐ Xác minh encryption keys backup
☐ Xác nhận phụ thuộc bên ngoài (không có bảo trì theo lịch)
☐ Thông báo khách hàng về cửa sổ bảo trì
☐ Brief incident commander và đội support
☐ Thiết lập monitoring dashboards
☐ Tài liệu hóa kế hoạch rollback

4 Giờ Trước:
☐ Drain connection pools (graceful shutdown)
☐ Final checkpoint - không có ghi mới vào nguồn
☐ Chuẩn bị kênh giao tiếp
☐ Định vị thành viên nhóm

Trong Migration:
☐ Dừng ghi ứng dụng
☐ Final backup trước cutover
☐ Chạy migration script
☐ Xác thực toàn vẹn dữ liệu:
   ✓ Row count theo bảng
   ✓ Checksums trên bảng quan trọng
   ✓ Kiểm tra referential integrity
   ✓ Index tồn tại và hợp lệ
☐ Kiểm tra kết nối ứng dụng
☐ Chạy smoke tests logic nghiệp vụ
☐ Start ứng dụng production
☐ Giám sát lỗi (30 phút đầu quan trọng)

Sau Cutover:
☐ Giám sát liên tục (1 giờ)
☐ Baseline hiệu suất (truy vấn, độ trễ)
☐ Backup sau cutover
☐ Đóng kênh giao tiếp
☐ Lên lịch post-mortem (ngày hôm sau)
```

---

#### 8. Các biện pháp bảo mật cho truy cập CSDL?

**Bảo Mật Nhiều Tầng:**

```
Tầng 1: MẠNG
- Mạng con riêng (không có IP công khai)
- Security group chỉ giới hạn cho app IPs
- VPN cho truy cập DBA
- TLS 1.2+ cho tất cả kết nối

Tầng 2: XÁC THỰC
- Không mật khẩu dùng chung
- IAM/Active Directory cho xác thực
- Service account riêng biệt theo ứng dụng
- MFA cho truy cập DBA console
- Xoay vòng SSH key (90 ngày)

Tầng 3: PHÂN QUYỀN
- Nguyên tắc quyền tối thiểu
- Role-based access control (RBAC)
- Phân tách schema theo ứng dụng
- Không superuser cho ứng dụng
- Vai trò chỉ đọc cho báo cáo

Tầng 4: MÃ HÓA
- Lưu trữ: AES-256 cho data files
- Truyền tải: TLS cho kết nối
- Transparent Data Encryption (TDE) nếu có
- Keystore/HSM cho quản lý key

Tầng 5: KIỂM TOÁN
- Thay đổi DDL được log (ai thay đổi schema, khi nào)
- Lần đăng nhập thất bại được log
- Lệnh đặc quyền thành công được log
- Kiểm toán truy vấn cho bảng PII/nhạy cảm
- Logs được gửi đến SIEM
```

---

### Cấp 4: Kỹ Thuật Sâu

#### 9. Giải thích transaction isolation levels và tác động hiệu suất

**Phổ:**

```
Mức Isolation         Vấn đề?              Hiệu suất    Use Case
──────────────────────────────────────────────────────────────────
Read Uncommitted   ✗ Dirty reads          Rất nhanh    Không khuyến nghị
Read Committed     ✗ Non-repeatable       Nhanh        Mặc định PostgreSQL/SQL Server
Repeatable Read    ✗ Phantom reads        Trung bình   Mặc định MySQL
Serializable       ✗ Không               Chậm         Transaction quan trọng
```

**Tác động hiệu suất:**

```
Isolation thấp hơn = Khóa ít hơn = Concurrency/throughput tốt hơn
Isolation cao hơn = Khóa nhiều hơn = Ít anomalies hơn nhưng chậm hơn

Hầu hết ứng dụng dùng Read Committed:
- An toàn tốt (không có dirty reads)
- Hiệu suất tốt (khóa tối thiểu)
- Anomalies hiếm trong thực tế
- Kiểm tra tầng ứng dụng giảm thiểu vấn đề
```

---

## Mẹo Phỏng Vấn

### Trước Phỏng Vấn

```
1. CHUẨN BỊ CÂU CHUYỆN (không chỉ kiến thức)
   - 2-3 câu chuyện sự cố (định dạng STAR)
   - Có sẵn số liệu (giải quyết trong X phút, ngăn Y vấn đề)

2. NGHIÊN CỨU CÔNG TY
   - Họ dùng CSDL nào? (Kiểm tra job description)
   - Quy mô (số người dùng, khối lượng dữ liệu)
   - Cơ sở hạ tầng (on-prem, cloud, hybrid)

3. BIẾT RESUME CỦA BẠN
   - Sẵn sàng thảo luận về bất kỳ công nghệ nào được liệt kê
   - Có ví dụ về tác động bạn tạo ra
   - Biết bạn muốn làm gì khác biệt lần này

4. CHUẨN BỊ CÂU HỎI
   - Lịch on-call rotation thường như thế nào?
   - Có bao nhiêu CSDL trong production? Quy mô là gì?
   - Pain point CSDL lớn nhất hiện tại là gì?
   - Bạn dùng công cụ gì để giám sát?
```

### Trong Phỏng Vấn

```
1. LÀM RÕ TRƯỚC KHI GIẢI QUYẾT
   "Hãy để tôi chắc chắn tôi hiểu ràng buộc..."
   - Làm rõ phạm vi vấn đề
   - Hỏi về ràng buộc
   - Hiểu đánh đổi cần thiết

2. SUY NGHĨ TO TIẾNG
   "Đây là cách tiếp cận của tôi, hãy để tôi hướng dẫn..."
   - Thể hiện lý luận, không chỉ câu trả lời
   - Giải thích tại sao bạn chọn cách này
   - Chỉ ra đánh đổi

3. THỪA NHẬN NHỮNG GÌ KHÔNG BIẾT
   "Tôi chưa làm việc với Aurora cụ thể, nhưng đây là những gì tôi sẽ làm..."
   - Trung thực được đánh giá hơn lừa dối
   - Thể hiện bạn có thể học và tìm hiểu

4. DÙNG VÍ DỤ
   "Trong vai trò trước tại [công ty]..."
   - Ví dụ cụ thể đánh bại câu trả lời chung chung
   - Số liệu/metrics làm cho tác động trở nên thực tế

5. NÓI THEO TRÌNH ĐỘ NGƯỜI PHỎNG VẤN
   - Manager: Nói về tác động, bối cảnh nghiệp vụ
   - Senior DBA: Chiều sâu kỹ thuật, tối ưu hóa
   - Engineer: Chi tiết triển khai, đánh đổi
```

### Sau Phỏng Vấn

```
1. GỬI EMAIL CẢM ƠN trong vòng 24 giờ
   - Tham chiếu điểm trò chuyện cụ thể
   - Nhắc lại sự quan tâm
   - Hỏi về timeline

2. SUY NGHĨ VỀ ĐIỂM YẾU
   - Câu hỏi nào cảm thấy yếu?
   - Nên học thêm gì?
   - Luyện tập cho vòng tiếp theo

3. CHUẨN BỊ CHO VÒNG TIẾP THEO
   - Hỏi về những gì kỳ vọng
   - Học các lĩnh vực đó
   - Chuẩn bị câu chuyện mới
```

---

## Tài Liệu Học

### Đọc Trước Phỏng Vấn

- [ ] Database Internals (Chương 1-3)
- [ ] Tài liệu PostgreSQL (Backup/Recovery, Replication)
- [ ] MySQL High Availability
- [ ] Tech blog của công ty mục tiêu

### Thực Hành Thực Tế

- [ ] Thiết lập PostgreSQL replication locally
- [ ] Thực hành khôi phục từ backup
- [ ] Phân tích truy vấn chậm với EXPLAIN
- [ ] Tạo migration script không thời gian chết
- [ ] Thiết lập monitoring dashboard

### Các Công Cụ Cần Biết

- **PostgreSQL:** pg_stat_statements, EXPLAIN ANALYZE, pg_dump, WAL archiving
- **MySQL:** slow query log, SHOW ENGINE INNODB STATUS, mysqldump, replication
- **General:** pgBouncer, pgBackRest, Barman, Prometheus, Grafana

---

## Kế Hoạch 90 Ngày

| Tuần  | Trọng tâm          | Kết quả                                    |
| ----- | ------------------ | ------------------------------------------ |
| 1-2   | Kiến thức cơ bản   | Có thể giải thích ACID, isolation từ bộ nhớ |
| 3-4   | Backup & HA        | Thiết kế chiến lược backup cho SLA cho trước |
| 5-6   | Hiệu suất          | Xử lý sự cố truy vấn chậm từ đầu đến cuối |
| 7-8   | Deep dive nền tảng | Biết rất rõ một nền tảng                   |
| 9-10  | Luyện câu chuyện   | Hoàn thiện 3 câu chuyện phỏng vấn (STAR)  |
| 11-12 | Phỏng vấn thử      | Luyện tập với ai đó                        |

---

> **Hãy nhớ:** Người phỏng vấn muốn biết bạn có thể:
>
> 1. **Suy nghĩ có hệ thống** về vấn đề
> 2. **Xem xét đánh đổi** không chỉ một giải pháp
> 3. **Làm việc dưới áp lực** và giao tiếp
> 4. **Học và thích nghi** với công nghệ mới
> 5. **Bảo vệ dữ liệu** (công việc quan trọng nhất của DBA)
>
> Chúc may mắn!
