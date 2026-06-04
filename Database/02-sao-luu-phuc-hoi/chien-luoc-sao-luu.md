# Chiến Lược Sao Lưu

Chọn chiến lược sao lưu đúng phụ thuộc vào kích thước CSDL, tốc độ thay đổi, yêu cầu RPO và cơ sở hạ tầng.

## Tổng Quan Các Loại Sao Lưu

### 1. Sao Lưu Toàn Bộ (Full Backup)

**Snapshot hoàn chỉnh của CSDL tại một thời điểm**

#### Ưu Điểm
- Phục hồi hoàn chỉnh, độc lập
- Có thể khôi phục toàn bộ CSDL từ một file
- Đơn giản nhất để hiểu

#### Nhược Điểm
- Kích thước lớn (toàn bộ CSDL)
- Mất nhiều thời gian hoàn thành
- Tốn tài nguyên (CPU, disk I/O)
- Không thực tế cho CSDL rất lớn

#### Dùng Khi Nào
- Sao lưu đầu tiên (baseline bắt buộc)
- Trước khi thay đổi lớn
- Baseline cho sao lưu tăng dần
- CSDL nhỏ đến trung bình

#### Ví Dụ

```bash
# PostgreSQL
pg_dump -U postgres mydb > full_backup.sql

# MySQL
mysqldump -u root -p mydb > full_backup.sql
```

---

### 2. Sao Lưu Tăng Dần (Incremental Backup)

**Chỉ các block đã thay đổi kể từ lần sao lưu cuối**

#### Ưu Điểm
- Kích thước nhỏ (chỉ các block đã thay đổi)
- Hoàn thành nhanh
- Ít tốn tài nguyên hơn
- Lý tưởng cho CSDL lớn, thay đổi chậm

#### Nhược Điểm
- Khôi phục phức tạp (cần full + tất cả incremental theo thứ tự)
- Phụ thuộc chuỗi (mất một, mất tất cả sau đó)
- Lưu trữ nhiều file

#### Dùng Khi Nào
- CSDL lớn và thay đổi thường xuyên
- Giới hạn storage/bandwidth
- Có thể chấp nhận phức tạp khi khôi phục
- Nhiều lần sao lưu tăng dần mỗi ngày

#### Độ Phức Tạp Khi Khôi Phục

```
Ngày 1: Full backup (100GB)
Ngày 2: Incremental #1 (5GB thay đổi)
Ngày 3: Incremental #2 (4GB thay đổi)

Để khôi phục về Ngày 3:
1. Khôi phục full backup
2. Áp dụng incremental #1
3. Áp dụng incremental #2
= Mất nhiều thời gian
```

---

### 3. Sao Lưu Chênh Lệch (Differential Backup)

**Tất cả thay đổi kể từ lần FULL backup cuối**

#### Ưu Điểm
- Khôi phục nhanh hơn incremental (chỉ cần full + differential cuối)
- Nhỏ hơn full backup
- Không có vấn đề phụ thuộc chuỗi

#### Nhược Điểm
- Lớn hơn incremental
- Kém hiệu quả cho CSDL thay đổi nhanh

#### Dùng Khi Nào
- Cần tốc độ khôi phục tốt hơn incremental
- SQL Server (hỗ trợ gốc)
- Lịch sao lưu hàng ngày phù hợp

#### Đơn Giản Khi Khôi Phục

```
Ngày 1: Full backup (100GB)
Ngày 2: Differential (5GB tất cả thay đổi)
Ngày 3: Differential (8GB tất cả thay đổi)

Để khôi phục về Ngày 3:
1. Khôi phục full backup
2. Áp dụng differential mới nhất (Ngày 3)
= Nhanh hơn incremental
```

---

### 4. Sao Lưu Logic (Logical Backup)

**Các câu lệnh SQL để tạo lại CSDL**

Công cụ: `pg_dump`, `mysqldump`, `expdp`

#### Ưu Điểm
- Di động qua nhiều nền tảng
- Con người có thể đọc được (có thể chỉnh sửa SQL)
- Có thể loại trừ các bảng/schema cụ thể
- Không phụ thuộc nền tảng
- Có thể tương thích phiên bản mới hơn

#### Nhược Điểm
- Khôi phục chậm hơn (thực thi lại SQL)
- RPO phụ thuộc vào lịch dump
- Lớn hơn với CSDL lớn
- Khó thực hiện PITR

#### Dùng Khi Nào
- Di chuyển giữa các phiên bản
- Cần tính di động
- Xuất tập con dữ liệu
- CSDL nhỏ đến trung bình

#### Ví Dụ

```bash
# PostgreSQL - dump đầy đủ
pg_dump -U postgres mydb > backup.sql

# Có nén
pg_dump -U postgres -F c mydb > backup.dump

# Chỉ các bảng cụ thể
pg_dump -U postgres -t users -t orders mydb > subset.sql
```

---

### 5. Sao Lưu Vật Lý (Physical Backup)

**Sao chép byte-by-byte các file dữ liệu**

Công cụ: filesystem snapshots, `xtrabackup`, `pg_basebackup`

#### Ưu Điểm
- Khôi phục nhanh hơn logic
- Có thể thực hiện PITR với WAL files
- Có thể thiết lập replicas
- Sao chép toàn bộ CSDL nhanh chóng

#### Nhược Điểm
- Đặc thù theo engine
- Cần crash recovery
- File lớn hơn
- Ít di động hơn

#### Dùng Khi Nào
- CSDL lớn
- Cần RTO nhanh
- Cần PITR
- Cần thiết lập replication

#### Ví Dụ

```bash
# PostgreSQL physical backup
pg_basebackup -U postgres -D /path/to/backup

# MySQL Percona XtraBackup
innobackupex --user=root --password=/path/to/backup
```

---

## Chiến Lược Sao Lưu Được Khuyến Nghị (OLTP)

Cho CSDL OLTP điển hình với yêu cầu uptime 99.99%:

```
┌──────────────────────────────────────────┐
│ SAO LƯU TOÀN BỘ (Hàng tuần, CN 2 SA)  │
│ RPO: Tối đa 7 ngày                      │
├──────────────────────────────────────────┤
│ SAO LƯU CHÊNH LỆCH (Hàng ngày 3 SA)   │
│ RPO: Tối đa 1 ngày                      │
├──────────────────────────────────────────┤
│ SAO LƯU TRANSACTION LOG (Mỗi 15 phút) │
│ RPO: 15 phút                            │
├──────────────────────────────────────────┤
│ BẢN SAO OFFSITE (Hàng ngày)            │
│ Bảo vệ chống sự cố toàn site           │
├──────────────────────────────────────────┤
│ KIỂM TRA KHÔI PHỤC (Hàng tháng)       │
│ Xác minh backup hoạt động              │
└──────────────────────────────────────────┘

Kết quả: RPO ≤ 15 phút, RTO ≤ 1 giờ
```

### Tại Sao Chiến Lược Này?

1. **Full backup (tuần)**: Baseline phục hồi, phù hợp với cửa sổ bảo trì tuần
2. **Differential hàng ngày**: Khôi phục nhanh (chỉ full + differential cuối)
3. **Transaction logs (15 phút)**: Point-in-time recovery không cần replica
4. **Offsite (hàng ngày)**: Bảo vệ chống mất toàn bộ site (hỏa hoạn, thiên tai)
5. **Kiểm tra (hàng tháng)**: Xác minh mọi thứ hoạt động trước khi khủng hoảng

---

## So Sánh Chiến Lược Sao Lưu

| Yếu tố           | Full    | Incremental | Differential | Logic    | Physical  |
| ---------------- | ------- | ----------- | ------------ | -------- | --------- |
| Kích thước backup | 100%   | 5-10%       | 15-25%       | 80%      | 100%      |
| Tốc độ backup    | Trung bình | Rất nhanh | Nhanh       | Chậm     | Trung bình |
| Tốc độ khôi phục | Nhanh  | Rất chậm    | Nhanh        | Chậm     | Rất nhanh |
| Độ phức tạp      | Đơn giản | Rất cao    | Trung bình   | Đơn giản | Trung bình |
| Cần storage      | Cao nhất | Thấp       | Trung bình   | Cao      | Cao       |
| Hỗ trợ PITR      | Không  | Hạn chế     | Hạn chế      | Không    | Có        |
| Tính di động     | Thấp   | Thấp        | Thấp         | Cao      | Thấp      |

---

## Chọn Chiến Lược Của Bạn

### Cây Quyết Định

```
1. Kích thước CSDL?
   - < 50GB → Dùng full backup hàng ngày
   - > 50GB → Dùng differential hoặc physical

2. Tốc độ thay đổi?
   - Thấp (< 5% hàng ngày) → Dùng differential
   - Cao (> 20% hàng ngày) → Dùng incremental hoặc physical + WAL

3. Cần PITR không?
   - Có → Dùng physical + WAL archiving
   - Không → Dùng full/differential/logic

4. Cần tính di động không?
   - Có → Dùng logical backup
   - Không → Dùng physical backup

5. Mục tiêu RTO?
   - > 4 giờ → Logical backup ổn
   - 1-4 giờ → Differential backup
   - < 1 giờ → Physical backup + WAL
```

---

## Lập Kế Hoạch Dung Lượng Lưu Trữ

### Tính Tổng Storage Sao Lưu

```
Daily_Backup_Size = Database_Size × (1 + Change_Rate)
Weekly_Backup_Size = Database_Size × 1.05
Monthly_Retention = (Daily × 30) + (Weekly × 13)

Ví dụ: CSDL 100GB, thay đổi 2% hàng ngày
- Differential hàng ngày: 100GB × 0.02 = 2GB
- 30 backup hàng ngày × 2GB = 60GB
- 13 full hàng tuần: 13 × 100GB = 1.3TB
- Tổng hàng tháng: 60GB + 1.3TB = 1.36TB
- Hàng năm: 16TB (không nén)

Với nén 50%:
- Hàng năm: 8TB (thực tế)
```

---

## Checklist Chiến Lược Sao Lưu

- [ ] Xác định yêu cầu RPO
- [ ] Xác định yêu cầu RTO
- [ ] Xác định kích thước và tốc độ tăng trưởng CSDL
- [ ] Tính tốc độ thay đổi
- [ ] Chọn loại backup (full/diff/incremental)
- [ ] Xác định tần suất backup
- [ ] Tính tổng storage cần thiết
- [ ] Lên kế hoạch thời gian lưu giữ
- [ ] Bao gồm transaction log backups
- [ ] Bao gồm bản sao offsite
- [ ] Lên lịch kiểm tra khôi phục
- [ ] Tài liệu hóa quy trình khôi phục
- [ ] Giám sát tỷ lệ thành công của backup

---

> **Điểm Mấu Chốt:** Chiến lược sao lưu tốt nhất cân bằng yêu cầu RPO/RTO với chi phí và độ phức tạp. Kiểm tra thường xuyên — backup chưa được khôi phục thử chỉ là niềm hy vọng.
