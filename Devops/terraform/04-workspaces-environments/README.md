# 04 — Workspaces & Environments — Quản Lý Đa Môi Trường

> Chiến lược quản lý hạ tầng Terraform cho nhiều môi trường (dev, staging, production) một cách an toàn, rõ ràng và không bị nhầm lẫn.

---

## 📚 Mục Lục Phần Này

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-workspaces.md](./1-workspaces.md) | Terraform Workspaces — Không Gian Làm Việc — giới hạn và khi nào nên dùng | ⭐⭐ |
| [2-environment-separation.md](./2-environment-separation.md) | Chiến lược phân tách dev / staging / prod | ⭐⭐ |
| [3-tfvars-management.md](./3-tfvars-management.md) | Quản lý `.tfvars` theo từng môi trường | ⭐⭐ |
| [4-terragrunt-intro.md](./4-terragrunt-intro.md) | Terragrunt — Giải pháp DRY cho đa môi trường | ⭐⭐⭐ |
| [5-backend-per-env.md](./5-backend-per-env.md) | Backend riêng biệt cho từng môi trường | ⭐⭐ |

---

## 🎯 Tại Sao Phần Này Quan Trọng?

Quản lý đa môi trường là **kỹ năng thiết yếu** mà mọi kỹ sư Terraform cần nắm vững. Những tai nạn phổ biến nhất trong thực tế:

- Apply nhầm vào **production** thay vì staging
- State file bị **trộn lẫn** giữa các môi trường
- Cấu hình **không nhất quán** giữa dev và prod
- Secrets của prod bị lộ khi làm việc ở dev
- Không biết môi trường nào đang có thay đổi gì

---

## 🗺️ Tổng Quan Các Chiến Lược

### 1. Terraform Workspaces — Không Gian Làm Việc

```
Phù hợp: Hạ tầng nhỏ, ít khác biệt giữa các môi trường
Không phù hợp: Cấu hình khác nhau nhiều, cần backend riêng biệt
```

### 2. Directory Separation — Phân Tách Thư Mục

```
environments/
├── dev/
├── staging/
└── prod/
```

```
Phù hợp: Cấu hình khác nhau nhiều giữa các môi trường
Nhược điểm: Dễ bị lệch — drift — giữa các môi trường
```

### 3. Terragrunt — Wrapper Tool

```
Phù hợp: Tổ chức lớn, nhiều môi trường, cần DRY
Nhược điểm: Phải học thêm công cụ, thêm độ phức tạp
```

---

## ⚡ Nguyên Tắc Vàng

### Nguyên Tắc 1: Cách Ly Hoàn Toàn (Isolation)

Mỗi môi trường **phải có state file riêng biệt**. Không bao giờ dùng chung state.

### Nguyên Tắc 2: Prod Phải Khó Apply Hơn

Production cần thêm bước xác nhận, approval, hoặc chỉ CI/CD mới được apply.

### Nguyên Tắc 3: Infra Code Giống Nhau, Config Khác Nhau

Cùng một module, cùng một code — chỉ thay đổi giá trị biến theo môi trường.

### Nguyên Tắc 4: Không Commit Secrets

`.tfvars` chứa secrets không được commit lên git. Dùng secret manager hoặc CI/CD variables.

---

## 📊 So Sánh Các Phương Pháp

| Tiêu Chí | Workspaces | Directory Separation | Terragrunt |
|----------|-----------|---------------------|------------|
| Độ phức tạp | Thấp | Trung bình | Cao |
| DRY — Don't Repeat Yourself | ❌ Kém | ❌ Kém | ✅ Tốt |
| Cách ly state | ✅ Có | ✅ Tốt nhất | ✅ Tốt |
| Cấu hình khác nhau nhiều | ❌ Khó | ✅ Tốt | ✅ Tốt |
| Phù hợp team nhỏ | ✅ | ✅ | ❌ |
| Phù hợp enterprise | ❌ | ✅ | ✅ |
| Học nhanh | ✅ | ✅ | ❌ |

---

## 🔗 Liên Kết Với Các Phần Khác

- **02-state-management/** — Backend và locking cho từng môi trường
- **03-modules/** — Module design để dùng lại code qua các môi trường
- **05-security/** — Quản lý secrets riêng theo môi trường
- **06-cicd/** — CI/CD pipeline với môi trường được phân tách

---

## ✅ Checklist Kỹ Năng

Sau khi hoàn thành phần này, bạn sẽ:

- [ ] Giải thích được ưu/nhược điểm của Terraform Workspaces
- [ ] Thiết kế cấu trúc thư mục cho dự án đa môi trường
- [ ] Quản lý `.tfvars` files an toàn và hiệu quả
- [ ] Cấu hình backend riêng biệt cho từng môi trường
- [ ] Hiểu khi nào nên dùng Terragrunt
- [ ] Trả lời câu hỏi phỏng vấn về environment management

---

**Thời Gian Học Ước Tính:** 4-6 giờ
**Cập Nhật:** 2026-05-12
