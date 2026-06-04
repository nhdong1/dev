# Kiến Thức Cơ Bản về Cơ Sở Dữ Liệu

## Nội Dung

Phần này bao gồm các kiến thức nền tảng mà mọi DBA cần nắm vững.

## Các Tài Liệu

| Tài liệu | Mô tả |
|----------|-------|
| [Tính Chất ACID](tinh-chat-acid.md) | Atomicity, Consistency, Isolation, Durability |
| [Mức Độ Isolation](muc-do-isolation.md) | Các mức cô lập transaction và đánh đổi |
| [Chiến Lược Index](chien-luoc-index.md) | B-Tree, Hash, Full-Text, Partial, Covering index |
| [Các Loại CSDL](loai-co-so-du-lieu.md) | RDBMS, NoSQL, Key-Value, Time-Series |
| [Connection Pooling](connection-pooling.md) | Gộp kết nối và tối ưu tài nguyên |

## Điểm Chính Cần Nhớ

1. **ACID** là nền tảng của mọi CSDL quan hệ — hiểu rõ từng tính chất và đánh đổi
2. **Index** là công cụ quan trọng nhất để tối ưu hiệu suất truy vấn
3. **Isolation levels** ảnh hưởng trực tiếp đến hiệu suất và tính nhất quán
4. **Connection pooling** là bắt buộc trong môi trường production
5. **Chọn đúng loại CSDL** cho từng bài toán là kỹ năng quan trọng

## Trình Tự Học

```
1. Tính chất ACID → hiểu nền tảng transaction
2. Mức độ Isolation → hiểu cách transaction tương tác
3. Chiến lược Index → tối ưu hiệu suất truy vấn
4. Các loại CSDL → chọn công cụ phù hợp
5. Connection Pooling → tối ưu tài nguyên kết nối
```
