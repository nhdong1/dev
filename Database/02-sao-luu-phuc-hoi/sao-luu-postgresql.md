# Sao Lưu & Phục Hồi PostgreSQL

Các chiến lược, công cụ và best practice cho PostgreSQL.

## WAL (Write-Ahead Log) Archiving

WAL là nền tảng cho tính bền vững và khôi phục của PostgreSQL. Lưu trữ WAL files để cho phép PITR (Point-in-Time Recovery).

### Thiết Lập WAL Archiving

Chỉnh sửa `/etc/postgresql/[version]/main/postgresql.conf`:

```ini
# Bật WAL archiving
wal_level = replica                    # Phải là 'replica' hoặc 'logical'
max_wal_senders = 3                    # Kết nối replication
wal_keep_size = 10GB                   # Giữ WAL gần đây locally
archive_mode = on                      # Bật archiving
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
archive_timeout = 300                  # Lưu archive ít nhất mỗi 5 phút
```

### Xác Minh Thiết Lập WAL

```sql
-- Kiểm tra thư mục WAL
SELECT * FROM pg_ls_waldir();

-- Kiểm tra WAL LSN hiện tại (Log Sequence Number)
SELECT pg_current_wal_lsn();

-- Kiểm tra archiving có hoạt động không
SELECT * FROM pg_stat_archiver;
-- archived_count phải tăng
-- failed_count phải bằng 0
```

### Cấu Trúc File WAL

```
Tên file WAL: 000000010000000000000001

Cấu trúc:
- 8 ký tự hex: timeline
- 8 ký tự hex: số log file
- 8 ký tự hex: số segment
- Mỗi file ~16MB
```

---

## Sao Lưu Logic (pg_dump)

Câu lệnh SQL để tạo lại CSDL. Tốt cho CSDL nhỏ hơn, tính di động, và sao lưu có chọn lọc.

### Sao Lưu Toàn Bộ CSDL

```bash
# Sao lưu cơ bản (SQL văn bản thuần)
pg_dump -U postgres mydb > backup.sql

# Với nén (nhỏ hơn nhiều)
pg_dump -U postgres -F c mydb > backup.dump
# -F c: định dạng tùy chỉnh (binary nén)
# -F d: định dạng thư mục (song song)
# -F t: định dạng tar

# Với mức nén
pg_dump -U postgres -F c -Z 9 mydb > backup.dump
# -Z 9: Nén tối đa
```

### Sao Lưu Có Chọn Lọc

```bash
# Sao lưu các bảng cụ thể
pg_dump -U postgres -t users -t orders mydb > tables.sql

# Sao lưu schema cụ thể
pg_dump -U postgres -n public mydb > schema_public.sql

# Loại trừ bảng cụ thể
pg_dump -U postgres --exclude-table=logs mydb > backup_no_logs.sql

# Chỉ schema (không có dữ liệu)
pg_dump -U postgres -s mydb > schema_only.sql

# Chỉ dữ liệu (không có schema)
pg_dump -U postgres -a mydb > data_only.sql
```

### Tùy Chọn Nâng Cao

```bash
# Output verbose
pg_dump -U postgres -v mydb > backup.sql

# Bao gồm câu lệnh DROP
pg_dump -U postgres --clean mydb > backup.sql

# Dump song song (nhanh hơn cho CSDL lớn)
pg_dump -U postgres -F d -j 4 mydb > backup_dir/
# -F d: định dạng thư mục
# -j 4: 4 jobs song song
```

### Khôi Phục từ Sao Lưu Logic

```bash
# Khôi phục SQL văn bản thuần
psql -U postgres -d mydb < backup.sql

# Khôi phục định dạng tùy chỉnh
pg_restore -U postgres -d mydb backup.dump

# Khôi phục định dạng thư mục (song song)
pg_restore -U postgres -d mydb -j 4 backup_dir/

# Khôi phục bảng cụ thể
pg_restore -U postgres -d mydb -t users backup.dump
```

---

## Sao Lưu Vật Lý (pg_basebackup)

Sao chép byte-by-byte các file dữ liệu. Khôi phục nhanh hơn, cho phép PITR và thiết lập replication.

### Sao Lưu Vật Lý Cơ Bản

```bash
# Sao lưu đơn giản
pg_basebackup -U postgres -D /path/to/backup -Ft -z -P

# Các flag:
# -U: User
# -D: Thư mục đích
# -Ft: Định dạng Tar
# -Fz: Tar nén
# -P: Hiển thị tiến độ
# -l: Label (đặt tên backup)
# -c: Chế độ checkpoint (fast/spread)
```

### Sao Lưu Vật Lý Có Nhãn

```bash
# Sao lưu có tên với nhãn
pg_basebackup -U postgres \
  -D /backups/base_2026_04_26 \
  -Ft -z -P \
  -l "Full backup - 2026-04-26"

# Sao lưu với WAL files đi kèm
pg_basebackup -U postgres \
  -D /backups/base_with_wal \
  -Ft -z -P \
  -X fetch \       # Bao gồm WAL files
  -c spread        # Phân tán tải checkpoint
```

### Khôi Phục từ Sao Lưu Vật Lý

```bash
# Giải nén backup (giả sử định dạng tar)
cd /var/lib/postgresql/14/main
tar -xzf /backups/base_2026_04_26/base.tar.gz

# Khởi động PostgreSQL (recovery xảy ra tự động)
pg_ctl start

# Giám sát tiến trình recovery
tail -f /var/log/postgresql/postgresql.log
```

---

## PITR (Point-in-Time Recovery) — Phục Hồi Đến Thời Điểm Cụ Thể

Khôi phục CSDL về bất kỳ thời điểm nào có WAL.

### Điều Kiện Tiên Quyết cho PITR

1. WAL archiving được bật
2. WAL files được giữ lại hoặc lưu trữ
3. Physical backup có sẵn
4. Biết thời điểm recovery mục tiêu

### Quy Trình Phục Hồi PITR

```bash
# 1. Khôi phục từ physical backup
cd /var/lib/postgresql/14/main
tar -xzf /backups/base_2026_04_26/base.tar.gz

# 2. Tạo file recovery.signal (PostgreSQL 12+)
touch /var/lib/postgresql/14/main/recovery.signal

# 3. Tạo cấu hình recovery (trong postgresql.conf)
cat > /var/lib/postgresql/14/main/postgresql.conf << 'EOF'
# Bật chế độ recovery
restore_command = 'cp /wal_archive/%f %p'

# Thời điểm mục tiêu (định dạng ISO 8601)
recovery_target_time = '2026-04-26 14:30:00'

# Hoặc mục tiêu LSN
# recovery_target_lsn = '0/1234567'

# Hành động khi đến mục tiêu
recovery_target_timeline = 'latest'
recovery_target_action = 'promote'
EOF

# 4. Khởi động PostgreSQL
pg_ctl start

# 5. Giám sát recovery
tail -f /var/log/postgresql/postgresql.log
# Khi recovery đến thời điểm mục tiêu, CSDL chuyển sang online

# 6. Xác minh recovery đã hoạt động
psql -U postgres -c "SELECT NOW();"
```

### Các Tham Số PITR

```sql
-- Các mục tiêu recovery có sẵn:
recovery_target_time = '2026-04-26 14:30:00'  -- Timestamp cụ thể
recovery_target_name = 'before_migration'     -- Savepoint có tên
recovery_target_lsn = '0/1234567'             -- Log sequence number

-- Hành động khi đến mục tiêu recovery:
recovery_target_action = 'promote'       -- Chuyển sang primary (mặc định)
recovery_target_action = 'pause'         -- Tạm dừng để xác minh
recovery_target_action = 'shutdown'      -- Tắt máy tại mục tiêu
```

---

## Chiến Lược Sao Lưu Đề Xuất cho PostgreSQL

```
┌─────────────────────────────────────────┐
│ PHYSICAL BACKUP TOÀN BỘ (Tuần/CN)     │
│ pg_basebackup → compressed tar         │
├─────────────────────────────────────────┤
│ WAL ARCHIVING (Liên tục)              │
│ archive_command → offsite storage      │
├─────────────────────────────────────────┤
│ LOGICAL BACKUP (Hàng tháng)           │
│ pg_dump → SQL văn bản để di động     │
├─────────────────────────────────────────┤
│ KIỂM TRA PHỤC HỒI (Hàng tháng)      │
│ Xác minh physical + WAL recovery       │
└─────────────────────────────────────────┘

Kết quả: RPO ≤ 5 phút (tần suất WAL), RTO ≤ 1 giờ
```

---

## Script Tự Động Hóa Sao Lưu

```bash
#!/bin/bash
# postgres_backup.sh
# Mục đích: Sao lưu hàng ngày tự động PostgreSQL

set -e

BACKUP_DIR="/backups/postgresql"
DB_NAME="production"
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_${BACKUP_DATE}.tar.gz"

# Tạo thư mục backup
mkdir -p $BACKUP_DIR

echo "Bắt đầu sao lưu PostgreSQL lúc $(date)"

# Physical backup
pg_basebackup \
    -U backup_user \
    -D /tmp/backup_temp \
    -Ft -z -P \
    -l "Daily backup $BACKUP_DATE"

# Di chuyển đến vị trí cuối
mv /tmp/backup_temp/base.tar.gz $BACKUP_FILE

# Xác minh backup
BACKUP_SIZE=$(stat -c%s "$BACKUP_FILE" 2>/dev/null || stat -f%z "$BACKUP_FILE")
echo "Sao lưu hoàn thành: $BACKUP_FILE ($(numfmt --to=iec $BACKUP_SIZE 2>/dev/null || echo $BACKUP_SIZE))"

# Chỉ giữ 8 backup gần nhất
cd $BACKUP_DIR
ls -t backup_*.tar.gz | tail -n +9 | xargs rm -f 2>/dev/null || true

echo "Sao lưu kết thúc lúc $(date)"
```

### Thiết Lập Lịch Cron

```bash
# Backup hàng ngày lúc 2 SA
0 2 * * * /scripts/postgres_backup.sh >> /var/log/postgres_backup.log 2>&1

# Full backup hàng tuần lúc 1 SA Chủ Nhật
0 1 * * 0 /scripts/postgres_backup_full.sh >> /var/log/postgres_backup.log 2>&1
```

---

## Các Sự Cố Backup PostgreSQL Thường Gặp

### Sự cố: WAL Archiving Thất Bại

```sql
SELECT * FROM pg_stat_archiver;
```

Tìm kiếm:
- `failed_count > 0`: Archive command thất bại
- `last_failed_wal`: WAL file thất bại cuối
- `last_failed_time`: Khi nào thất bại

**Giải pháp:**
- Kiểm tra quyền archive_command
- Xác minh dung lượng đích
- Kiểm tra kết nối mạng (nếu archiving từ xa)

### Sự cố: Backup Quá Lớn

**Giải pháp:**
- Dùng `-F c` (định dạng nén) cho logical backups
- Loại trừ các bảng lớn không quan trọng
- Dùng `-Z 9` để nén tối đa
- Cân nhắc sao lưu tăng dần

### Sự cố: Khôi Phục Mất Quá Nhiều Thời Gian

**Giải pháp:**
- Dùng physical backup thay vì logical
- Dùng parallel restore: `pg_restore -j 4`
- Cân nhắc warm standby để failover nhanh hơn

---

## Checklist Sao Lưu PostgreSQL

- [ ] WAL archiving được cấu hình và hoạt động
- [ ] Physical backups hàng ngày được lên lịch
- [ ] Backups được xác minh (kích thước, checksums)
- [ ] Chính sách lưu giữ backup được xác định
- [ ] Backups offsite được cấu hình
- [ ] Kiểm tra khôi phục hàng tháng được lên lịch
- [ ] Quy trình phục hồi được tài liệu hóa
- [ ] Backup user được tạo với quyền tối thiểu
- [ ] Cảnh báo giám sát backup được cấu hình
- [ ] Mã hóa backup được bật cho dữ liệu nhạy cảm

---

> **Điểm Mấu Chốt:** WAL archiving của PostgreSQL cung cấp khả năng phục hồi mạnh mẽ. Kết hợp physical backups với WAL archiving để phục hồi linh hoạt, nhanh chóng tại bất kỳ thời điểm nào.
