# 02 — State Management — Quản Lý Trạng Thái Terraform

> State — Trạng thái — là trái tim của Terraform. Hiểu sâu về state là yêu cầu bắt buộc để làm việc với Terraform trong môi trường team và production.

---

## 📚 Mục Lục Phần Này

| File | Chủ Đề | Thời Gian |
|------|--------|-----------|
| [1-state-explained.md](./1-state-explained.md) | State file là gì, chứa gì, tại sao quan trọng | 60 phút |
| [2-remote-backend.md](./2-remote-backend.md) | S3+DynamoDB, GCS, Azure Blob, Terraform Cloud | 90 phút |
| [3-state-locking.md](./3-state-locking.md) | State Locking — Khoá trạng thái — và deadlock | 60 phút |
| [4-state-commands.md](./4-state-commands.md) | terraform state list/show/mv/rm/pull/push | 60 phút |
| [5-state-recovery.md](./5-state-recovery.md) | Phục hồi khi state bị corrupt — hỏng | 60 phút |

**Tổng thời gian ước tính:** 6–8 giờ (bao gồm thực hành)

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Giải thích chính xác state file lưu gì và tại sao Terraform cần nó
- [ ] Thiết lập remote backend với S3 + DynamoDB cho team
- [ ] Hiểu và cấu hình State Locking — Khoá trạng thái — để ngăn race condition
- [ ] Dùng thành thạo các lệnh `terraform state` để quản trị tài nguyên
- [ ] Phục hồi state khi gặp sự cố corrupt — hỏng — hoặc mất dữ liệu

---

## 💡 Tại Sao State Management Quan Trọng?

State Management là chủ đề **được hỏi nhiều nhất** trong phỏng vấn Terraform vì ba lý do:

1. **Phân biệt người biết dùng vs người hiểu sâu** — Ai cũng dùng được Terraform, nhưng chỉ người hiểu state mới xử lý được sự cố production
2. **Sai lầm state rất tốn kém** — Xóa nhầm tài nguyên do state corrupt có thể mất hàng giờ khôi phục
3. **Kỹ năng teamwork** — Remote backend và locking là nền tảng để nhiều người cùng làm việc an toàn

---

## 🗺️ Lộ Trình Học Phần Này

```
Bước 1: Hiểu state là gì (1-state-explained.md)
   ↓
Bước 2: Thiết lập remote backend thực tế (2-remote-backend.md)
   ↓
Bước 3: Hiểu và test state locking (3-state-locking.md)
   ↓
Bước 4: Thực hành các lệnh state (4-state-commands.md)
   ↓
Bước 5: Chuẩn bị kịch bản recovery (5-state-recovery.md)
```

---

## 🔑 Khái Niệm Then Chốt

| Thuật Ngữ | Giải Thích |
|-----------|------------|
| **State file** | File `terraform.tfstate` lưu mapping giữa resource trong code và resource thực tế trên cloud |
| **Remote backend** | Lưu state file trên dịch vụ tập trung (S3, GCS, Terraform Cloud) thay vì máy cục bộ |
| **State locking** | Cơ chế khoá file state khi đang chạy để ngăn hai người apply cùng lúc |
| **Workspace** | Môi trường độc lập dùng chung codebase nhưng có state riêng |
| **State drift** | Sự chênh lệch giữa state file và trạng thái thực tế của hạ tầng |

---

## ⚠️ Anti-patterns Phổ Biến

```
❌ Commit terraform.tfstate vào Git
❌ Không bật State Locking khi làm việc nhóm
❌ Xóa state file thay vì dùng terraform state rm
❌ Dùng local backend trong môi trường production
❌ Không backup state trước khi thực hiện thao tác nguy hiểm
```

---

## ✅ Best Practices Tóm Tắt

```
✓ Dùng remote backend cho mọi môi trường từ staging trở lên
✓ Bật State Locking (DynamoDB với S3, GCS tự động hỗ trợ)
✓ Bật versioning trên S3 bucket lưu state
✓ Mã hoá state file at rest và in transit
✓ Dùng terraform state mv thay vì xóa-tạo lại khi refactor
✓ Backup state trước các thao tác lớn
```

---

## 🔗 Liên Kết Liên Quan

- [01-fundamentals/5-lifecycle.md](../01-fundamentals/5-lifecycle.md) — Vòng đời init→apply ảnh hưởng đến state như thế nào
- [04-workspaces-environments/README.md](../04-workspaces-environments/README.md) — Workspace và state riêng cho từng môi trường
- [09-troubleshooting/1-state-corruption.md](../09-troubleshooting/1-state-corruption.md) — Xử lý sự cố state nâng cao

---

**Cập Nhật Lần Cuối:** 2026-05-12
