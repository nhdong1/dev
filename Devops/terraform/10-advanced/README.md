# 10 — Advanced Topics — Chủ Đề Terraform Nâng Cao

> Phần này dành cho người đã nắm vững nền tảng Terraform và muốn đi sâu vào các kỹ thuật nâng cao, kiến trúc phức tạp và hệ sinh thái mở rộng xung quanh Terraform.

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- Sử dụng **Dynamic Blocks** — Khối Động — để tạo cấu hình linh hoạt
- Hiểu và áp dụng đúng các **Meta-arguments** — Đối Số Siêu Dữ Liệu — (`depends_on`, `lifecycle`, `provisioner`)
- Chọn đúng giữa `for_each` và `count` trong từng tình huống thực tế
- Hiểu cách viết **Custom Provider** — Provider Tùy Chỉnh — bằng Go
- Viết Terraform bằng ngôn ngữ lập trình thông thường với **CDK** — Cloud Development Kit
- Nắm vững **OpenTofu** — Nhánh Open-source của Terraform
- Thiết kế kiến trúc **Multi-region & Multi-account** — Đa Vùng & Đa Tài Khoản

---

## 📁 Các File Trong Phần Này

| File | Chủ Đề | Độ Phức Tạp |
|------|--------|-------------|
| [1-dynamic-blocks.md](1-dynamic-blocks.md) | Dynamic Blocks — Khối Động | ⭐⭐⭐ |
| [2-meta-arguments.md](2-meta-arguments.md) | Meta-arguments — depends_on, lifecycle, provisioner | ⭐⭐⭐ |
| [3-for-each-count.md](3-for-each-count.md) | for_each vs count — So Sánh Chuyên Sâu | ⭐⭐⭐ |
| [4-custom-providers.md](4-custom-providers.md) | Custom Provider — Provider Tùy Chỉnh | ⭐⭐⭐⭐ |
| [5-terraform-cdk.md](5-terraform-cdk.md) | CDK for Terraform — Python/TypeScript | ⭐⭐⭐ |
| [6-opentofu.md](6-opentofu.md) | OpenTofu — Nhánh Open-source | ⭐⭐⭐ |
| [7-multi-region-account.md](7-multi-region-account.md) | Kiến Trúc Đa Vùng & Đa Tài Khoản | ⭐⭐⭐⭐ |

---

## 🗺️ Lộ Trình Học Phần Nâng Cao

```
Điều kiện tiên quyết:
  ✅ Nắm vững 01-09 (Fundamentals → Troubleshooting)
  ✅ Đã có kinh nghiệm dùng Terraform thực tế
  ✅ Quen thuộc với ít nhất 1 cloud provider

Thứ tự học đề xuất:
  1. Dynamic Blocks → hiểu cách tạo cấu hình linh hoạt
  2. Meta-arguments → kiểm soát vòng đời tài nguyên
  3. for_each vs count → tránh lỗi phổ biến với collections
  4. Custom Providers → hiểu Terraform từ bên trong
  5. Terraform CDK → lựa chọn thay thế cho HCL
  6. OpenTofu → hiểu hệ sinh thái open-source
  7. Multi-region/account → kiến trúc enterprise
```

---

## 💡 Tại Sao Phần Nâng Cao Quan Trọng?

### Cho Phỏng Vấn Senior DevOps / Platform Engineer

Câu hỏi nâng cao thường gặp:

- "Giải thích Dynamic Blocks — khi nào dùng và khi nào không?"
- "Khác nhau giữa `count` và `for_each` trong thực tế?"
- "Thiết kế kiến trúc Terraform cho 50 AWS account?"
- "Bạn có bao giờ viết custom provider chưa? Quy trình như thế nào?"

### Cho Công Việc Thực Tế

Tình huống thực tế yêu cầu kiến thức nâng cao:

- Quản lý hạ tầng phức tạp với nhiều điều kiện
- Tổ chức dùng nhiều AWS account (Landing Zone)
- Team muốn dùng Python/TypeScript thay vì HCL
- Migrate từ Terraform sang OpenTofu (open-source)

---

## 🔗 Điều Hướng

| | |
|---|---|
| ← Phần trước | [09-troubleshooting](../09-troubleshooting/README.md) |
| → Phần sau | [11-interview-prep](../11-interview-prep/README.md) |
| ↑ Tổng quan | [README.md](../README.md) |
| 📋 Chỉ mục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Độ Khó:** Nâng Cao (3-5+ năm kinh nghiệm)
**Thời Gian Học:** 15-20 giờ
