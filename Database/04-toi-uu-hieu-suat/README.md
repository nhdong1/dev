# Tối Ưu Hiệu Suất CSDL

Nắm vững tối ưu truy vấn, index và điều chỉnh CSDL cho hệ thống production.

## Các Tài Liệu

| Tài liệu | Mô tả |
|----------|-------|
| [Phân Tích Truy Vấn](phan-tich-truy-van.md) | EXPLAIN ANALYZE, đọc query plan, pg_stat_statements |
| [Thiết Kế Index Nâng Cao](thiet-ke-index-nang-cao.md) | Composite, covering, partial, functional index |
| [Thống Kê & Kế Hoạch Truy Vấn](thong-ke-va-ke-hoach-truy-van.md) | ANALYZE, pg_stats, extended statistics, planner tuning |
| [Autovacuum & Bloat](autovacuum-va-bloat.md) | MVCC, vacuum tuning, bloat detection, XID wraparound |
| [Khóa & Deadlock](khoa-va-deadlock.md) | Lock types, deadlock detection, prevention strategies |
| [Phát Hiện Truy Vấn Chậm](phat-hien-truy-van-cham.md) | Slow query log, pg_stat_statements, auto_explain |
| [Chiến Lược Caching](chien-luoc-caching.md) | Buffer cache, Redis, materialized views, cache patterns |

---

## Nguyên Tắc Cốt Lõi

1. **Đo Trước, Tối Ưu Sau** — Không đoán bottleneck, dùng EXPLAIN ANALYZE và pg_stat_statements
2. **Index Không Miễn Phí** — Mỗi index làm chậm writes; chỉ tạo khi có bằng chứng
3. **Thống Kê Phải Chính Xác** — ANALYZE thường xuyên để planner có đủ thông tin
4. **Vacuum Là Bắt Buộc** — Bloat và XID wraparound là nguy cơ thực sự trong production
5. **Transaction Ngắn Nhất Có Thể** — Giảm lock contention và deadlock risk

---

## Các Loại Hiệu Suất Vấn Đề

```
TRUY VẤN CHẬM:
  Triệu chứng: Latency cao, timeout
  Nguyên nhân: Thiếu index, thống kê cũ, query viết sai
  Xem: phan-tich-truy-van.md, thiet-ke-index-nang-cao.md

THROUGHPUT THẤP:
  Triệu chứng: Không xử lý được nhiều requests đồng thời
  Nguyên nhân: Lock contention, connection pool quá nhỏ
  Xem: khoa-va-deadlock.md, chien-luoc-caching.md

DATABASE PHÌNH TO (BLOAT):
  Triệu chứng: Disk tăng nhanh, queries chậm dần
  Nguyên nhân: Dead tuples tích lũy, autovacuum không đủ tích cực
  Xem: autovacuum-va-bloat.md

CACHE HIT THẤP:
  Triệu chứng: Disk I/O cao, latency bất thường
  Nguyên nhân: shared_buffers quá nhỏ, working set > RAM
  Xem: chien-luoc-caching.md
```

---

## Quy Trình Tối Ưu Hiệu Suất

```
Bước 1: Phát Hiện
  → pg_stat_statements: Tìm top queries chậm
  → Slow query log: Log queries > threshold
  → pg_stat_activity: Xem real-time activity

Bước 2: Phân Tích
  → EXPLAIN (ANALYZE, BUFFERS): Hiểu execution plan
  → Tìm Seq Scan, Filter Removes, Sort spill

Bước 3: Xác Định Nguyên Nhân
  → Thiếu index? → Thêm index phù hợp
  → Thống kê cũ? → ANALYZE
  → Bloat? → Tune autovacuum
  → Lock wait? → Giảm transaction size

Bước 4: Tối Ưu
  → Thực hiện thay đổi
  → Đo lại (so sánh trước/sau)
  → Tài liệu hóa cải thiện

Bước 5: Monitor
  → Cài đặt alert cho các chỉ số quan trọng
  → Review hàng ngày/hàng tuần
```

---

## Các Chỉ Số Dashboard Hiệu Suất

| Chỉ số | Truy vấn | Mục tiêu | Cảnh báo |
|--------|----------|----------|---------|
| Truy vấn chậm (> 1s) | pg_stat_statements | < 5 | > 10 |
| Sequential scans | pg_stat_user_tables | < 100 | > 1000 |
| Index bloat | pgstattuple | < 20% | > 50% |
| Buffer cache hit | pg_stat_database | > 99% | < 98% |
| Autovacuum lag | pg_stat_user_tables | < 1 ngày | > 3 ngày |
| XID age | pg_database | < 1B | > 1.5B |
| Connection pool usage | pg_stat_activity | < 80% | > 90% |
| Replication lag | pg_last_wal_receive_lsn() | < 1 giây | > 5 giây |
| Lock waits | pg_locks | < 5 | > 20 |

---

## Checklist Tổng Hợp

**Cơ Bản:**
- [ ] pg_stat_statements đã bật
- [ ] slow query log đang chạy (log_min_duration_statement)
- [ ] EXPLAIN ANALYZE cho top 10 queries chậm nhất
- [ ] Index phù hợp được tạo (không thiếu, không dư)

**Bảo Trì:**
- [ ] Autovacuum được điều chỉnh cho workload
- [ ] Thống kê cập nhật (ANALYZE sau import/delete lớn)
- [ ] Bloat < 20% cho các bảng chính
- [ ] XID age được monitor (xa giới hạn 2 tỷ)

**Cấu Hình:**
- [ ] shared_buffers = 25% RAM
- [ ] effective_cache_size = 75% RAM
- [ ] work_mem phù hợp (tránh sort/hash spill)
- [ ] Connection pool đúng kích thước

**Monitoring:**
- [ ] Alert khi buffer cache hit < 99%
- [ ] Alert khi slow queries tăng
- [ ] Alert khi XID age > 1.5 tỷ
- [ ] Baseline hiệu suất được tài liệu hóa

---

## Chủ Đề Nâng Cao

- [ ] Query plan cache và prepared statements
- [ ] Bloom filters và JIT compilation
- [ ] Chiến lược partitioning
- [ ] Thống kê cấp cột (extended statistics)
- [ ] Điều chỉnh cost-based optimizer
- [ ] Thực thi truy vấn song song (parallel query)

---

> **Điểm Mấu Chốt:** Hiệu suất tốt nhất đến từ việc hiểu hệ thống — không phải từ việc áp dụng mù quáng các "best practices". Đo, phân tích, tối ưu có căn cứ, và monitor liên tục.
