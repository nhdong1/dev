# 07 — Testing — Kiểm Thử Hạ Tầng Terraform

> Kiểm thử hạ tầng là lớp bảo vệ cuối cùng trước khi thay đổi lên production. Terraform có hệ sinh thái công cụ phong phú từ validate đơn giản đến integration test — Kiểm thử tích hợp — thực tế trên cloud.

---

## 🎯 Tại Sao Phải Kiểm Thử Hạ Tầng?

Không giống như ứng dụng thông thường, lỗi hạ tầng có thể:

- **Gây downtime** — Dịch vụ ngừng hoạt động — tức thì
- **Tạo lỗ hổng bảo mật** — Security vulnerability — khó phát hiện sau
- **Phát sinh chi phí không kiểm soát** — Uncontrolled cost — nếu tài nguyên sai cấu hình
- **Ảnh hưởng nhiều team** — vì hạ tầng là nền tảng chung

Kiểm thử hạ tầng giúp phát hiện vấn đề sớm, trước khi đến tay người dùng.

---

## 📐 Kim Tự Tháp Kiểm Thử Hạ Tầng

```
                    ┌─────────────────────┐
                    │   End-to-End Tests  │  ← Chậm, tốn kém, ít nhất
                    │   (môi trường thực) │
                   ┌┴─────────────────────┴┐
                   │  Integration Tests    │  ← Trung bình
                   │  (Terratest, cloud)   │
                  ┌┴───────────────────────┴┐
                  │   Compliance Tests      │  ← Nhanh
                  │   (Checkov, OPA, tfsec) │
                 ┌┴─────────────────────────┴┐
                 │     Static Analysis       │  ← Rất nhanh
                 │  (validate, fmt, TFLint)  │
                └───────────────────────────┘
                          Nhiều nhất
```

**Nguyên tắc:** Càng xuống dưới kim tự tháp càng chạy nhiều và thường xuyên hơn.

---

## 📁 Nội Dung Chương Này

| File | Chủ Đề | Công Cụ | Thời Gian |
|------|--------|---------|-----------|
| [1-validate-fmt.md](1-validate-fmt.md) | Kiểm tra cú pháp & định dạng | `terraform validate`, `terraform fmt` | 30 phút |
| [2-tflint.md](2-tflint.md) | Linting — Kiểm tra cú pháp nâng cao | TFLint | 45 phút |
| [3-terratest.md](3-terratest.md) | Unit & Integration Testing | Terratest (Go) | 2-3 giờ |
| [4-checkov-opa.md](4-checkov-opa.md) | Compliance Testing — Kiểm thử tuân thủ | Checkov, OPA | 2 giờ |
| [5-test-strategy.md](5-test-strategy.md) | Chiến lược kiểm thử toàn diện | Tổng hợp | 1 giờ |

---

## 🛠️ Công Cụ Trong Hệ Sinh Thái

### Tầng 1: Static Analysis — Phân Tích Tĩnh

| Công Cụ | Mục Đích | Tốc Độ | Cài Đặt |
|---------|----------|--------|---------|
| `terraform validate` | Kiểm tra cú pháp HCL | < 1 giây | Có sẵn trong Terraform |
| `terraform fmt` | Định dạng code tự động | < 1 giây | Có sẵn trong Terraform |
| TFLint | Linting, best practices, provider rules | Vài giây | `brew install tflint` |

### Tầng 2: Security & Compliance — Bảo Mật & Tuân Thủ

| Công Cụ | Mục Đích | Chuẩn Hỗ Trợ |
|---------|----------|--------------|
| Checkov | Quét bảo mật và compliance | CIS, HIPAA, PCI-DSS, SOC2 |
| tfsec | Security scanner — Quét lỗ hổng bảo mật | AWS, GCP, Azure best practices |
| OPA — Open Policy Agent | Policy as Code — Chính Sách Dưới Dạng Mã | Custom policies — Chính sách tùy chỉnh |
| Sentinel | Policy enforcement — Bắt buộc chính sách | Chỉ dùng với Terraform Cloud/Enterprise |

### Tầng 3: Integration Testing — Kiểm Thử Tích Hợp

| Công Cụ | Ngôn Ngữ | Mục Đích |
|---------|----------|----------|
| Terratest | Go | Deploy thật, test thật, destroy |
| terraform test | HCL (built-in từ v1.6) | Test framework tích hợp sẵn |
| Kitchen-Terraform | Ruby | Test kitchen integration |
| Pytest-Terraform | Python | Test với Python |

---

## 🔄 Testing Pipeline — Đường Ống Kiểm Thử Trong CI/CD

```yaml
# Thứ tự chạy trong CI/CD pipeline
stages:
  - static:       # Chạy trong < 30 giây
      - terraform fmt -check
      - terraform validate
      - tflint

  - security:     # Chạy trong < 2 phút
      - checkov -d .
      - tfsec .

  - plan:         # Chạy trong 1-5 phút
      - terraform plan

  - integration:  # Chạy trong 10-30 phút (chỉ trên nhánh main/feature lớn)
      - terratest

  - apply:        # Chỉ sau khi tất cả pass
      - terraform apply
```

---

## ✅ Checklist Nhanh Theo Giai Đoạn

### Trước Khi Commit — Pre-commit

```bash
terraform fmt -recursive       # Tự động định dạng
terraform validate             # Kiểm tra cú pháp
tflint --recursive             # Linting nâng cao
```

### Trong CI/CD — Trên Pull Request

```bash
checkov -d . --quiet           # Security scan
tfsec . --soft-fail            # Security scan bổ sung
terraform plan -out=tfplan     # Preview thay đổi
```

### Trước Khi Merge Vào Main

```bash
# Chạy Terratest nếu có thay đổi lớn
go test ./test/... -timeout 30m -v
```

---

## 💡 Nguyên Tắc Testing Cho Hạ Tầng

1. **Luôn test từ tầng thấp lên cao** — validate trước, integration test sau
2. **Fail fast — Thất bại sớm** — dừng pipeline ngay khi có lỗi đầu tiên
3. **Test phải idempotent — Lặp lại được** — chạy nhiều lần cho cùng kết quả
4. **Destroy sau khi test** — không để lại tài nguyên tốn tiền
5. **Test trong môi trường riêng** — không test trực tiếp trên staging/prod
6. **Document expected behavior — Hành vi kỳ vọng** — test là tài liệu sống

---

## 🚀 Bắt Đầu Nhanh

```bash
# 1. Cài đặt công cụ
brew install tflint
pip install checkov
# Terratest cần Go: https://go.dev/dl/

# 2. Chạy kiểm tra cơ bản
cd your-terraform-project/
terraform fmt -recursive -check
terraform validate
tflint --init && tflint
checkov -d .

# 3. Xem kết quả và sửa lỗi
```

---

## 🔗 Điều Hướng

| Trước | Tiếp Theo |
|-------|-----------|
| [06-cicd/](../06-cicd/) — Tích hợp CI/CD | [08-monitoring/](../08-monitoring/) — Giám sát & Drift Detection |

---

**Cập Nhật:** 2026-05-12 | **Độ Khó:** ⭐⭐⭐ | **Thời Gian Học:** 6-8 giờ
