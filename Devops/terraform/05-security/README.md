# 05 — Security — Bảo Mật Terraform Toàn Diện

> Bảo mật hạ tầng bắt đầu từ code — không phải từ sau khi deploy.

---

## 🎯 Mục Tiêu Của Phần Này

Sau khi hoàn thành phần này, bạn có thể:

- Quản lý secrets — bí mật — an toàn, không bao giờ lộ trong state file hay log
- Áp dụng nguyên tắc Least Privilege — Đặc Quyền Tối Thiểu — cho mọi IAM role Terraform
- Xử lý biến nhạy cảm (sensitive variables) và che giấu output
- Tích hợp công cụ phân tích tĩnh (tfsec, Checkov, Terrascan) vào CI/CD
- Thiết lập Audit Logging — Nhật Ký Kiểm Tra — cho toàn bộ thay đổi hạ tầng

---

## 📁 Nội Dung

| File                       | Chủ Đề                                                            | Độ Khó |
| -------------------------- | ----------------------------------------------------------------- | ------ |
| `1-secrets-management.md`  | Vault, AWS Secrets Manager, SOPS — quản lý bí mật                | ⭐⭐   |
| `2-iam-roles.md`           | IAM role tối thiểu — Least Privilege cho Terraform               | ⭐⭐⭐ |
| `3-sensitive-variables.md` | Biến nhạy cảm — sensitive vars, output masking                    | ⭐⭐   |
| `4-static-analysis.md`     | tfsec, Checkov, Terrascan — SAST cho Terraform                    | ⭐⭐   |
| `5-audit-logging.md`       | CloudTrail, audit logs — nhật ký kiểm tra mọi thay đổi           | ⭐⭐   |

---

## 🔑 Tại Sao Bảo Mật Terraform Quan Trọng?

### Terraform Có Quyền Rất Lớn

Terraform CI/CD runner thường được cấp quyền **tạo, sửa, xóa** mọi tài nguyên cloud. Nếu bị xâm phạm — compromised — kẻ tấn công có thể:

```
- Tạo IAM user mới với quyền admin
- Exfiltrate data từ S3/GCS bucket
- Xóa toàn bộ database production
- Mở security group cho toàn bộ internet (0.0.0.0/0)
- Tăng giới hạn chi phí lên hàng triệu USD
```

### Ba Vectơ Tấn Công Phổ Biến

```
1. Secrets lộ trong state file → đọc được plaintext từ S3
2. Overprivileged IAM role → leo thang đặc quyền
3. Thiếu kiểm tra code → resource cấu hình sai đi vào production
```

---

## 🏗️ Kiến Trúc Bảo Mật Nhiều Lớp

```
┌─────────────────────────────────────────────────────────┐
│                    Lớp 1: Phòng Ngừa                    │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│  │ Least Priv   │ │  Sensitive   │ │  Secrets Mgmt    │ │
│  │ IAM Roles    │ │  Variables   │ │  (Vault/SSM)     │ │
│  └──────────────┘ └──────────────┘ └──────────────────┘ │
├─────────────────────────────────────────────────────────┤
│                    Lớp 2: Phát Hiện                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│  │    tfsec     │ │   Checkov    │ │   Terrascan      │ │
│  │ (SAST scan)  │ │ (Compliance) │ │ (Policy check)   │ │
│  └──────────────┘ └──────────────┘ └──────────────────┘ │
├─────────────────────────────────────────────────────────┤
│                    Lớp 3: Kiểm Toán                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│  │  CloudTrail  │ │  Audit Logs  │ │  Change Tracking │ │
│  │  (AWS API)   │ │  (who/what)  │ │  (drift detect)  │ │
│  └──────────────┘ └──────────────┘ └──────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 📋 Security Checklist Nhanh

### Trước Khi Viết Code

- [ ] IAM role có áp dụng Least Privilege chưa?
- [ ] Secrets có được lưu vào secret manager thay vì hardcode không?
- [ ] State backend có bật encryption-at-rest — mã hoá khi lưu — chưa?

### Khi Viết Code

- [ ] Biến chứa password/token có đánh dấu `sensitive = true` chưa?
- [ ] Output ra ngoài có ẩn giá trị nhạy cảm chưa?
- [ ] Không có `0.0.0.0/0` trong security group ingress không cần thiết chưa?

### Trước Khi Merge

- [ ] `tfsec` chạy qua và không có HIGH/CRITICAL findings chưa?
- [ ] `checkov` scan pass không?
- [ ] Có ai review plan output trước khi apply không?

### Sau Khi Apply

- [ ] CloudTrail log đã ghi nhận thay đổi chưa?
- [ ] Không có unexpected resources được tạo ra không?
- [ ] Alert cho bất thường có hoạt động không?

---

## 🔗 Điều Hướng

| Bước Trước                                                        | Bước Tiếp Theo                                |
| ----------------------------------------------------------------- | --------------------------------------------- |
| [04-workspaces-environments](../04-workspaces-environments/)      | [06-cicd](../06-cicd/)                        |

---

## ⚡ Tóm Tắt Nhanh Cho Phỏng Vấn

**Q: Làm sao quản lý secrets trong Terraform?**
> Dùng external secret manager (Vault/AWS SSM), đọc qua `data` source. Không bao giờ hardcode trong `.tf` files. Đánh dấu `sensitive = true` cho mọi biến nhạy cảm. Bật encryption cho state backend.

**Q: IAM role cho Terraform CI/CD cần quyền gì?**
> Chỉ đúng các quyền cần thiết cho resources trong project đó. Không dùng `AdministratorAccess`. Dùng IAM Conditions — điều kiện — để giới hạn theo region/tag. Rotate credentials định kỳ.

**Q: Làm sao ngăn cấu hình sai đi vào production?**
> Chạy tfsec + Checkov trong CI/CD pipeline, block merge nếu có HIGH/CRITICAL findings. Kết hợp với OPA — Open Policy Agent — policies để enforce organizational rules.
