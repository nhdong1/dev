# Sao Lưu & Phục Hồi

## Nội Dung

Phần này bao gồm tất cả kiến thức về backup và disaster recovery.

## Các Tài Liệu

| Tài liệu | Mô tả |
|----------|-------|
| [RPO & RTO](rpo-rto.md) | Mục tiêu phục hồi và thiết lập yêu cầu nghiệp vụ |
| [Chiến Lược Sao Lưu](chien-luoc-sao-luu.md) | Full, Incremental, Differential, Logic, Physical |
| [Sao Lưu PostgreSQL](sao-luu-postgresql.md) | WAL archiving, pg_dump, pg_basebackup, PITR |

## Nguyên Tắc Cốt Lõi

1. **RPO và RTO là quyết định nghiệp vụ** — công nghệ chỉ triển khai chúng
2. **Backup chưa được kiểm tra = không có backup** — luôn kiểm tra khôi phục
3. **3-2-1 Rule:** 3 bản sao, 2 phương tiện khác nhau, 1 offsite
4. **WAL archiving** cho phép PITR — khôi phục đến bất kỳ thời điểm nào
5. **Tự động hóa** backup và giám sát — con người mắc lỗi

## Cây Quyết Định Nhanh

```
RPO cần thiết < 15 phút?
├─ Có → Replication đồng bộ + WAL archiving
└─ Không → Transaction log backup mỗi 15-30 phút

RTO cần thiết < 30 phút?
├─ Có → Cần warm/hot standby
└─ Không → Khôi phục từ backup có thể ổn

CSDL > 50GB?
├─ Có → Physical backup (pg_basebackup)
└─ Không → Logical backup (pg_dump) có thể đủ
```
