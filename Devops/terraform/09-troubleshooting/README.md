# 09 — Troubleshooting — Xử Lý Sự Cố Terraform

> Hướng dẫn chẩn đoán và khắc phục các vấn đề thường gặp trong Terraform — từ lỗi nhỏ hàng ngày đến sự cố production nghiêm trọng.

---

## 🎯 Mục Tiêu Của Phần Này

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Chẩn đoán và phục hồi khi state file — file trạng thái — bị hỏng
- [ ] Giải quyết Dependency Cycle — Vòng phụ thuộc vòng tròn — trong resource graph
- [ ] Xử lý lỗi provider: timeout, rate limit, authentication failure
- [ ] Dùng `terraform import` và `moved` block để quản lý tài nguyên hiện có
- [ ] Bật TF\_LOG — biến môi trường bật ghi nhật ký — để debug chi tiết
- [ ] Áp dụng production checklist trước mỗi lần apply

---

## 📁 Nội Dung Phần Này

| File                          | Chủ Đề                                           | Độ Quan Trọng |
| ----------------------------- | ------------------------------------------------ | ------------- |
| `1-state-corruption.md`       | State file hỏng — phát hiện và phục hồi         | ⭐⭐⭐       |
| `2-dependency-issues.md`      | Dependency Cycle — vòng phụ thuộc — và cách fix | ⭐⭐⭐       |
| `3-provider-errors.md`        | Lỗi provider — timeout, rate limit, auth        | ⭐⭐⭐       |
| `4-import-moved.md`           | terraform import & moved block                  | ⭐⭐⭐       |
| `5-debug-mode.md`             | TF\_LOG=DEBUG — chế độ gỡ lỗi chi tiết          | ⭐⭐         |
| `6-production-checklist.md`   | Checklist trước khi apply vào production        | ⭐⭐⭐       |

---

## 🔍 Sơ Đồ Chẩn Đoán Nhanh

```
Gặp lỗi Terraform?
│
├── Lỗi khi chạy plan/apply?
│   ├── "Error acquiring the state lock" → 3-state-locking
│   ├── "Cycle:" trong output → 2-dependency-issues.md
│   ├── "Error: Provider produced inconsistent result" → 3-provider-errors.md
│   └── "Error refreshing state" → 1-state-corruption.md
│
├── Tài nguyên đã tồn tại ngoài Terraform?
│   └── terraform import / moved block → 4-import-moved.md
│
├── Không hiểu lỗi, cần thêm thông tin?
│   └── Bật TF_LOG=DEBUG → 5-debug-mode.md
│
└── Trước khi apply production?
    └── Chạy qua checklist → 6-production-checklist.md
```

---

## ⚡ Lệnh Khẩn Cấp Hay Dùng

```bash
# Xem toàn bộ resource trong state
terraform state list

# Kiểm tra chi tiết một resource
terraform state show aws_instance.web

# Xóa resource khỏi state (không xóa tài nguyên thực)
terraform state rm aws_instance.web

# Bật debug log đầy đủ
TF_LOG=DEBUG terraform apply 2>&1 | tee /tmp/tf-debug.log

# Unlock state khi bị kẹt
terraform force-unlock <LOCK_ID>

# Refresh state từ cloud (đồng bộ lại)
terraform refresh

# Kiểm tra state có bị hỏng không
terraform state pull | jq .
```

---

## 🚨 Phân Loại Mức Độ Sự Cố

### Mức 1 — Cảnh Báo (Warning) — Không ảnh hưởng ngay

- Deprecated — Lỗi thời — attribute trong provider
- TFLint — Công cụ kiểm tra lint — cảnh báo style
- Plan hiển thị thay đổi không mong muốn nhỏ

**Xử lý:** Ghi nhận, fix trong sprint tiếp theo.

### Mức 2 — Lỗi Chức Năng (Functional Error) — Cần fix trước khi deploy

- Provider authentication failure — Lỗi xác thực provider
- Resource dependency không đúng
- State drift — Lệch state — phát hiện qua `terraform plan`

**Xử lý:** Fix ngay, không deploy cho đến khi resolved.

### Mức 3 — Sự Cố Nghiêm Trọng (Critical Incident) — Ảnh hưởng production

- State file bị corrupt — hỏng hoàn toàn
- State lock không thể giải phóng — deadlock
- Apply đang chạy bị interrupt — ngắt giữa chừng

**Xử lý:** Kích hoạt incident response, không thao tác thêm cho đến khi có kế hoạch rõ ràng.

---

## 📚 Thứ Tự Học Khuyến Nghị

```
Bắt đầu từ đây:
1. state-corruption.md      ← Hiểu rủi ro lớn nhất
2. dependency-issues.md     ← Lỗi phổ biến khi viết code
3. provider-errors.md       ← Lỗi hàng ngày khi vận hành
4. import-moved.md          ← Kỹ năng thiết yếu khi migrate
5. debug-mode.md            ← Công cụ điều tra khi bí
6. production-checklist.md  ← Quy trình trước mỗi deployment
```

---

## 🔗 Liên Kết Với Các Phần Khác

| Vấn Đề                        | Xem Thêm Tại                                      |
| ----------------------------- | ------------------------------------------------- |
| State lock deadlock           | `02-state-management/3-state-locking.md`          |
| State recovery nâng cao       | `02-state-management/5-state-recovery.md`         |
| Rollback sau apply thất bại   | `06-cicd/5-rollback-strategy.md`                  |
| Drift detection tự động       | `08-monitoring/1-drift-detection.md`              |
| Kiểm thử để tránh sự cố       | `07-testing/5-test-strategy.md`                   |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
