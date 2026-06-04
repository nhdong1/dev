# 3 — State Locking — Khoá Trạng Thái và Deadlock

> State Locking — Khoá trạng thái — là cơ chế ngăn hai quá trình đồng thời ghi vào state file, tránh race condition — điều kiện tranh chấp — và corrupt state.

---

## 📚 Mục Lục

1. [State Locking Là Gì?](#1-state-locking-là-gì)
2. [Cơ Chế Hoạt Động Theo Backend](#2-cơ-chế-hoạt-động-theo-backend)
3. [Lock Info — Thông Tin Khoá](#3-lock-info--thông-tin-khoá)
4. [Deadlock — Khoá Chết và Stale Lock — Khoá Cũ](#4-deadlock--khoá-chết-và-stale-lock--khoá-cũ)
5. [Cưỡng Bức Mở Khoá — Force Unlock](#5-cưỡng-bức-mở-khoá--force-unlock)
6. [Lock Timeout — Thời Gian Chờ Khoá](#6-lock-timeout--thời-gian-chờ-khoá)
7. [Disable Locking — Tắt Khoá Khi Cần Thiết](#7-disable-locking--tắt-khoá-khi-cần-thiết)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. State Locking Là Gì?

Hãy tưởng tượng hai developer cùng chạy `terraform apply` trên cùng một project:

```
Timeline:
  T=0s   Developer A: bắt đầu apply
  T=1s   Developer B: bắt đầu apply (cùng lúc)
  T=2s   Developer A: đọc state → thấy 5 resources
  T=2s   Developer B: đọc state → thấy 5 resources (cùng snapshot cũ)
  T=10s  Developer A: tạo resource #6, cập nhật state → 6 resources
  T=10s  Developer B: tạo resource #7, cập nhật state → 6 resources (ghi đè)
  Kết quả: Resource #6 của Developer A mất khỏi state!
           Terraform nghĩ nó không tồn tại → lần apply tiếp theo sẽ tạo lại hoặc gây lỗi
```

State Locking giải quyết vấn đề này bằng cách đảm bảo chỉ một process được ghi vào state tại một thời điểm.

---

## 2. Cơ Chế Hoạt Động Theo Backend

### S3 + DynamoDB

```
Khi apply bắt đầu:
  Terraform ghi item vào DynamoDB:
    {
      "LockID": "mycompany-state-prod/prod/vpc/terraform.tfstate",
      "Info": {
        "ID": "abc123",
        "Operation": "OperationTypeApply",
        "Who": "developer@hostname",
        "Created": "2026-05-12T10:00:00Z",
        "Path": "prod/vpc/terraform.tfstate"
      }
    }

Khi process khác thử apply:
  Terraform đọc DynamoDB → thấy LockID đã tồn tại
  → Báo lỗi và dừng lại

Khi apply hoàn thành:
  Terraform xóa item khỏi DynamoDB → state được mở khoá
```

### GCS — Google Cloud Storage

GCS dùng **object locking** tích hợp sẵn — không cần service riêng:

```
Terraform tạo file .tflock trên GCS cùng thư mục với state
Khi apply xong, file .tflock bị xóa
```

### Azure Blob Storage

Azure dùng **blob lease** — thuê blob độc quyền:

```
Terraform acquire lease (thuê quyền ghi) trên blob
Lease TTL: 60 giây (tự động gia hạn khi apply đang chạy)
Khi apply xong, lease được release (giải phóng)
```

### Terraform Cloud

Terraform Cloud quản lý locking tự động ở application level — không cần cấu hình.

---

## 3. Lock Info — Thông Tin Khoá

Khi state đang bị khoá, Terraform hiển thị thông tin chi tiết:

```
╷
│ Error: Error acquiring the state lock
│
│ Error message: ConditionalCheckFailedException: The conditional request failed
│ Lock Info:
│   ID:        abc123def-4567-89ab-cdef-0123456789ab
│   Path:      prod/vpc/terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       jenkins@ci-runner-01
│   Version:   1.7.0
│   Created:   2026-05-12 10:00:00.000000 +0000 UTC
│   Info:
╵
```

**Đọc thông tin này để quyết định hành động:**

| Trường | Thông Tin Cần Chú Ý |
|--------|---------------------|
| `Who` | Ai đang giữ khoá — người hay CI/CD runner |
| `Operation` | Apply hay plan hay destroy đang chạy |
| `Created` | Khoá được tạo lúc mấy giờ — đã lâu chưa |
| `ID` | Dùng để force unlock nếu cần |

---

## 4. Deadlock — Khoá Chết và Stale Lock — Khoá Cũ

### Stale Lock — Khoá Cũ

Xảy ra khi process giữ khoá bị crash — sập — hoặc bị kill mà không kịp giải phóng khoá:

```
Tình huống:
  CI/CD runner bắt đầu terraform apply → acquire lock
  Runner bị kill (timeout, OOM, network loss)
  Lock vẫn còn trong DynamoDB
  → Mọi apply tiếp theo đều bị block mãi mãi
```

**Cách nhận biết stale lock:**
- Lock tồn tại nhiều giờ hoặc nhiều ngày
- `Who` trỏ đến process đã không còn chạy
- `Operation` là apply/plan nhưng không có activity

### Deadlock — Khoá Chết

Trong Terraform, deadlock thực sự hiếm vì chỉ có một loại lock cho toàn bộ state file (không phải per-resource). Tuy nhiên, có trường hợp giả deadlock:

```
Process A giữ lock cho state file X
Process A cần đọc state file Y (cross-state dependency)
State file Y đang bị Process B giữ lock
Process B cần đọc state file X để hoàn thành
→ Circular wait — Chờ vòng tròn
```

**Giải pháp:** Thiết kế dependency graph rõ ràng, không để cross-state circular dependency.

---

## 5. Cưỡng Bức Mở Khoá — Force Unlock

Khi chắc chắn khoá là stale (không có process nào đang thực sự apply), có thể force unlock:

```bash
# Lấy Lock ID từ thông báo lỗi
terraform force-unlock LOCK_ID

# Ví dụ:
terraform force-unlock abc123def-4567-89ab-cdef-0123456789ab
```

**⚠️ Cảnh báo nghiêm trọng:**

```
KHÔNG bao giờ force unlock khi:
  ✗ Không chắc process giữ khoá có còn đang chạy không
  ✗ Không biết operation đang thực hiện là gì
  ✗ Apply đang ở giữa chừng (sẽ gây state corrupt — hỏng)

CHỈ force unlock khi:
  ✓ Chắc chắn process đã chết (runner bị kill, timeout)
  ✓ Đã kiểm tra không có apply nào đang chạy
  ✓ Đã backup state trước khi unlock (nếu có thể)
```

**Quy trình an toàn khi force unlock:**

```bash
# 1. Xác nhận không có apply nào đang chạy
#    (kiểm tra CI/CD, slack với team)

# 2. Với S3: xem trực tiếp DynamoDB để confirm lock đã cũ
aws dynamodb get-item \
  --table-name terraform-state-locks \
  --key '{"LockID": {"S": "mycompany-state/prod/vpc/terraform.tfstate"}}'

# 3. Backup state nếu cần
aws s3 cp s3://mycompany-state/prod/vpc/terraform.tfstate \
          ./terraform.tfstate.backup.$(date +%Y%m%d_%H%M%S)

# 4. Force unlock
terraform force-unlock abc123def-4567-89ab-cdef-0123456789ab

# 5. Chạy plan để verify state còn nguyên vẹn
terraform plan
```

**Với DynamoDB — xóa lock trực tiếp:**

```bash
# Xóa lock item trực tiếp từ DynamoDB (thay thế force-unlock)
aws dynamodb delete-item \
  --table-name terraform-state-locks \
  --key '{"LockID": {"S": "mycompany-state/prod/vpc/terraform.tfstate"}}'
```

---

## 6. Lock Timeout — Thời Gian Chờ Khoá

Mặc định, Terraform **không chờ** — nếu state bị khoá sẽ fail ngay lập tức. Có thể cấu hình:

```bash
# Chờ tối đa 5 phút trước khi báo lỗi
terraform apply -lock-timeout=5m

# Chờ tối đa 10 phút
terraform plan -lock-timeout=10m
```

**Khi nào dùng lock-timeout:**

```
Hữu ích trong CI/CD khi nhiều job có thể chạy gần cùng lúc:
  → Pipeline A đang apply, sắp xong
  → Pipeline B bắt đầu, chờ Pipeline A hoàn thành
  → Nếu không dùng timeout, B sẽ fail ngay và cần retry thủ công
```

**Lưu ý:** `-lock-timeout=0s` (mặc định) = không chờ, fail ngay.

---

## 7. Disable Locking — Tắt Khoá Khi Cần Thiết

```bash
# Tắt locking (CỰC KỲ NGUY HIỂM — chỉ dùng trong trường hợp đặc biệt)
terraform apply -lock=false
terraform plan -lock=false
```

**Trường hợp hợp lệ duy nhất để dùng `-lock=false`:**

```
1. Read-only operations trong môi trường dev — chỉ đọc, không ghi
2. Emergency recovery khi backend locking service gặp sự cố
3. Local state (không có locking) dùng trong test

Tuyệt đối KHÔNG dùng trong production với remote backend!
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q: State Locking hoạt động như thế nào với S3 backend?

**Trả lời mẫu:**
> S3 không có native locking, nên Terraform dùng DynamoDB để implement distributed lock. Khi apply bắt đầu, Terraform ghi một item vào DynamoDB với LockID là path của state file và metadata (ai, khi nào, operation gì). Nếu item đó đã tồn tại, Terraform báo lỗi state is locked. Khi apply hoàn thành (dù thành công hay thất bại), item bị xóa để release lock. Đây là lý do tại sao S3 backend cần đi kèm DynamoDB.

### Q: Làm gì khi gặp state lock mà không ai đang apply?

**Trả lời mẫu:**
> Đây là stale lock — khoá cũ — thường do CI/CD runner bị crash. Quy trình xử lý: (1) xác nhận không có process nào thực sự đang chạy bằng cách kiểm tra CI/CD dashboard và hỏi team; (2) đọc lock metadata để xem lock được tạo lúc nào và bởi ai; (3) nếu chắc chắn là stale, dùng `terraform force-unlock LOCK_ID`; (4) chạy `terraform plan` để verify state còn nguyên vẹn.

### Q: Tại sao GCS không cần DynamoDB như S3 để locking?

**Trả lời mẫu:**
> GCS có native object locking support tích hợp sẵn trong API. Terraform GCS backend dùng atomic object operations của GCS để acquire lock — tạo một file `.tflock` với điều kiện "chỉ tạo nếu chưa tồn tại". S3 không có operation tương tự, nên cần DynamoDB với conditional writes để implement distributed lock. Azure tương tự GCS — dùng blob lease tích hợp sẵn của Azure Storage.

---

## 🔗 Đọc Tiếp

- [4-state-commands.md](./4-state-commands.md) — Các lệnh thao tác state
- [5-state-recovery.md](./5-state-recovery.md) — Phục hồi khi state bị corrupt

---

**Cập Nhật Lần Cuối:** 2026-05-12
