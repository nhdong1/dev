# 1 — State File Là Gì, Chứa Gì, Tại Sao Quan Trọng

> State file — File trạng thái — là nơi Terraform ghi nhớ "hạ tầng trông như thế nào sau lần apply cuối". Không có state, Terraform không thể biết phải tạo mới hay cập nhật tài nguyên nào.

---

## 📚 Mục Lục

1. [State File Là Gì?](#1-state-file-là-gì)
2. [Cấu Trúc Bên Trong State File](#2-cấu-trúc-bên-trong-state-file)
3. [Terraform Dùng State Như Thế Nào?](#3-terraform-dùng-state-như-thế-nào)
4. [State và Thực Tế Lệch Nhau — Drift](#4-state-và-thực-tế-lệch-nhau--drift)
5. [Tại Sao Không Commit State Lên Git?](#5-tại-sao-không-commit-state-lên-git)
6. [Sensitive Data — Dữ Liệu Nhạy Cảm Trong State](#6-sensitive-data--dữ-liệu-nhạy-cảm-trong-state)
7. [Local vs Remote State](#7-local-vs-remote-state)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. State File Là Gì?

State file là file JSON tên `terraform.tfstate` (hoặc `terraform.tfstate.backup`) mà Terraform tự động tạo và cập nhật sau mỗi lần `apply`.

**Vai trò của state file:**

| Vai Trò | Giải Thích |
|---------|------------|
| **Mapping** | Kết nối resource trong code (ví dụ: `aws_instance.web`) với ID thực tế trên cloud (ví dụ: `i-0abc123def`) |
| **Tracking metadata** | Lưu các thuộc tính mà API không trả về trực tiếp (ví dụ: dependency order — thứ tự phụ thuộc) |
| **Performance** | Tránh gọi API cho mỗi resource khi chạy plan (Terraform đọc state thay vì query cloud) |
| **Dependency graph** | Terraform dùng state để tính toán dependency graph — đồ thị phụ thuộc — khi apply/destroy |

---

## 2. Cấu Trúc Bên Trong State File

Ví dụ một state file đơn giản:

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 12,
  "lineage": "a3f2c1d4-...",
  "outputs": {
    "instance_ip": {
      "value": "54.12.34.56",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "i-0abc123def456789",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t3.micro",
            "public_ip": "54.12.34.56",
            "tags": {
              "Name": "web-server",
              "Environment": "production"
            }
          },
          "dependencies": [
            "aws_security_group.web_sg"
          ]
        }
      ]
    }
  ]
}
```

**Giải thích các trường quan trọng:**

| Trường | Ý Nghĩa |
|--------|---------|
| `version` | Phiên bản schema của state format |
| `serial` | Số thứ tự tăng dần mỗi lần state thay đổi — dùng để phát hiện xung đột |
| `lineage` | UUID duy nhất của state — đảm bảo không merge nhầm state của hai project |
| `resources[].mode` | `managed` (tài nguyên Terraform quản lý) hoặc `data` (data source — nguồn dữ liệu) |
| `instances[].attributes` | Tất cả thuộc tính của tài nguyên tại thời điểm apply |
| `dependencies` | Danh sách các resource mà resource này phụ thuộc vào |

---

## 3. Terraform Dùng State Như Thế Nào?

Khi chạy `terraform plan`, Terraform thực hiện **3 chiều so sánh**:

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   Code (HCL)          State File       Cloud (API)      │
│   ──────────          ──────────       ────────────     │
│   Muốn gì?            Lần trước?       Hiện tại?        │
│                                                         │
│   aws_instance        i-0abc123        i-0abc123        │
│   t3.small      vs    t3.micro    vs   t3.micro         │
│                                                         │
│   Kết luận: Cần update instance_type từ micro → small   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Quy trình ra quyết định:**

```
Nếu (resource trong code) và (không có trong state):
    → Tạo mới (create)

Nếu (resource trong code) và (có trong state) và (thuộc tính khác nhau):
    → Cập nhật (update) hoặc Xóa-tạo lại (replace)

Nếu (resource không trong code) nhưng (có trong state):
    → Xóa (destroy)

Nếu (resource trong code) và (giống hệt state):
    → Không làm gì (no-op)
```

---

## 4. State và Thực Tế Lệch Nhau — Drift

**Drift — Lệch cấu hình** xảy ra khi ai đó thay đổi hạ tầng trực tiếp trên cloud (qua console, CLI, API) mà không đi qua Terraform.

```
Ví dụ thực tế:
  State nói:   instance_type = "t3.micro"
  AWS thực tế: instance_type = "t3.small"  ← ai đó đổi tay trên console
  Code nói:    instance_type = "t3.micro"

  Kết quả khi plan:
  ~ aws_instance.web
      instance_type: "t3.small" → "t3.micro"  (revert về giá trị trong code)
```

**Cách phát hiện drift:**

```bash
# Refresh state từ thực tế cloud (không thay đổi hạ tầng)
terraform refresh

# Hoặc xem plan có thay đổi gì không (plan tự refresh)
terraform plan

# Terraform >= 1.3: dùng -refresh-only để chỉ cập nhật state
terraform plan -refresh-only
terraform apply -refresh-only
```

---

## 5. Tại Sao Không Commit State Lên Git?

Đây là lỗi phổ biến nhất của người mới học Terraform.

### Lý do 1: Dữ liệu nhạy cảm bị lộ

```json
// Ví dụ: state file sau khi tạo RDS database
{
  "attributes": {
    "password": "mySecretPassword123!",    // ← Mật khẩu database lưu plaintext!
    "connection_string": "postgresql://admin:mySecretPassword123!@..."
  }
}
```

### Lý do 2: Conflict khi làm việc nhóm

```
Developer A: pull main, apply → state cập nhật
Developer B: pull main (cũ), apply → state cũ ghi đè state mới của A
             → Terraform nghĩ resource của A không tồn tại → plan destroy!
```

### Lý do 3: State file không phải source of truth cho code

```
Git là nơi lưu code (HCL).
State file là snapshot hạ tầng — thay đổi mỗi lần apply.
Hai thứ này có vòng đời khác nhau và không nên ở cùng chỗ.
```

**Luôn thêm vào `.gitignore`:**

```gitignore
# Terraform state files
terraform.tfstate
terraform.tfstate.backup
*.tfstate
*.tfstate.*

# Terraform working directory
.terraform/
.terraform.lock.hcl   # Chỉ bỏ qua nếu không muốn pin provider versions
```

---

## 6. Sensitive Data — Dữ Liệu Nhạy Cảm Trong State

State file luôn lưu tất cả attributes của tài nguyên, **kể cả secrets** như:

- Database passwords — Mật khẩu cơ sở dữ liệu
- Private keys — Khoá riêng tư
- API keys — Khoá API
- Connection strings — Chuỗi kết nối

**Điều này có nghĩa là:**

```
Ngay cả khi dùng sensitive = true trong variable:
  variable "db_password" {
    sensitive = true  # Ẩn khỏi terminal output
  }

→ Giá trị VẪN được lưu plaintext trong state file!
→ Vì vậy bảo mật state file = bảo mật toàn bộ hạ tầng.
```

**Biện pháp bảo vệ:**

```hcl
# Với S3 backend: bật encryption at rest
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true  # ← Bắt buộc
  }
}
```

---

## 7. Local vs Remote State

### Local State — Trạng Thái Cục Bộ

```
Lưu tại: ./terraform.tfstate (thư mục hiện tại)

✓ Phù hợp: Học tập, dự án cá nhân, prototype
✗ Không phù hợp: Team > 1 người, môi trường staging/production
```

### Remote State — Trạng Thái Từ Xa

```
Lưu tại: S3, GCS, Azure Blob, Terraform Cloud, ...

✓ Nhiều người cùng truy cập an toàn
✓ State locking ngăn apply đồng thời
✓ Versioning — lịch sử thay đổi
✓ Mã hoá và phân quyền đọc/ghi
✗ Cần thiết lập ban đầu
✗ Phụ thuộc vào dịch vụ bên ngoài (nhưng thường rất ổn định)
```

**So sánh nhanh:**

| Tiêu Chí | Local | Remote (S3) | Terraform Cloud |
|----------|-------|-------------|-----------------|
| Dễ thiết lập | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| Hỗ trợ team | ❌ | ✅ | ✅ |
| State Locking | ❌ | ✅ (DynamoDB) | ✅ (tự động) |
| Versioning | ❌ | ✅ (S3) | ✅ |
| Chi phí | Miễn phí | Thấp | Miễn phí (≤5 users) |
| Phù hợp | Dev cá nhân | Team/Prod | Team/Enterprise |

---

## 8. Câu Hỏi Phỏng Vấn

### Q: State file trong Terraform là gì và tại sao cần nó?

**Trả lời mẫu:**
> State file là file JSON mà Terraform dùng để tracking — theo dõi — mapping giữa resource trong code và resource thực tế trên cloud. Terraform cần state để biết lần apply trước tạo ra những gì, từ đó tính toán minimal diff — thay đổi tối thiểu — khi plan. Không có state, Terraform không thể phân biệt "cần tạo mới" hay "cần update" tài nguyên đã tồn tại.

### Q: Tại sao không nên commit state file lên Git?

**Trả lời mẫu:**
> Vì state file thường chứa sensitive data — dữ liệu nhạy cảm — như database passwords và private keys ở dạng plaintext. Ngoài ra, state file thay đổi liên tục sau mỗi apply nên dễ gây conflict khi nhiều người cùng làm việc. Giải pháp đúng là dùng remote backend như S3 với encryption at rest.

### Q: Drift là gì và làm sao phát hiện?

**Trả lời mẫu:**
> Drift xảy ra khi hạ tầng thực tế trên cloud khác với những gì state file ghi. Nguyên nhân thường do ai đó thay đổi tay qua console hoặc CLI. Phát hiện bằng cách chạy `terraform plan` — nếu có thay đổi dù không sửa code là có drift. Với Terraform >= 1.3, có thể dùng `terraform plan -refresh-only` để chỉ cập nhật state mà không thay đổi hạ tầng.

---

## 🔗 Đọc Tiếp

- [2-remote-backend.md](./2-remote-backend.md) — Thiết lập remote backend cho team
- [3-state-locking.md](./3-state-locking.md) — Hiểu State Locking — Khoá trạng thái

---

**Cập Nhật Lần Cuối:** 2026-05-12
