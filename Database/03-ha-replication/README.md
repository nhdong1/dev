# Tính Sẵn Sàng Cao & Nhân Bản

## Nội Dung

Phần này bao gồm kiến trúc HA, nhân bản và chiến lược failover.

## Các Tài Liệu

| Tài liệu | Mô tả |
|----------|-------|
| [Các Loại Nhân Bản](cac-loai-replication.md) | Single-Leader, Multi-Leader, Leaderless, Cascading |
| [Đồng Bộ vs Bất Đồng Bộ](dong-bo-va-bat-dong-bo.md) | Đánh đổi an toàn và hiệu suất |
| [Chiến Lược Failover](chien-luoc-failover.md) | Thủ công → Bán tự động → Hoàn toàn tự động |

## Nguyên Tắc Cốt Lõi

1. **Single-Leader** là lựa chọn mặc định — đơn giản và đã được chứng minh
2. **Nhân bản đồng bộ** cho RPO=0, nhưng tăng độ trễ
3. **Semisync** là cân bằng tốt nhất cho hầu hết hệ thống
4. **Quorum** ngăn split-brain — cần ít nhất 3 node
5. **Kiểm tra failover** thường xuyên — không bao giờ chờ đến sự cố thực

## Tam Giác CAP Trong Nhân Bản

```
Tính Nhất Quán (Consistency)
         /\
        /  \
       /    \
      / ACID \
     /_______ \
Tính Sẵn Sàng  Khả Năng Chịu Lỗi
(Availability)  (Partition Tolerance)

→ Khi phân vùng mạng: Phải chọn C hoặc A
→ Đồng bộ = C (nhất quán nhưng có thể giảm sẵn sàng)
→ Bất đồng bộ = A (sẵn sàng nhưng eventual consistency)
```

## Checklist Triển Khai HA

```
□ Single-Leader setup (primary + ≥1 replica)
□ Replication mode quyết định (async/sync/semisync)
□ Giám sát lag nhân bản liên tục
□ Cảnh báo khi lag > ngưỡng (thường 1-5 giây)
□ Failover strategy quyết định (thủ công/bán tự/tự động)
□ Split-brain prevention cấu hình
□ Kiểm tra failover hàng quý (game day)
□ Runbook failover được viết và thực hành
□ Backup độc lập với nhân bản
□ RPO/RTO tài liệu hóa và truyền đạt
```
