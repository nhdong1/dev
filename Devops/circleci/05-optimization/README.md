# 05 — Tối Ưu Hóa Pipeline — Optimization

> Module này bao gồm các chiến lược tối ưu hóa pipeline CircleCI: caching (bộ nhớ đệm), workspace (không gian làm việc chung), test splitting (phân chia kiểm thử), parallelism (song song hóa) và pipeline insights (thống kê & phân tích).

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn có thể:

- [ ] Thiết kế chiến lược caching hiệu quả với cache keys đúng cách
- [ ] Dùng workspace để truyền artifact giữa các job trong cùng workflow
- [ ] Phân chia test suite lớn thành các luồng song song bằng `circleci tests split`
- [ ] Cấu hình `parallelism` để giảm thời gian build đáng kể
- [ ] Đọc và phân tích Pipeline Insights để tìm bottleneck

---

## 📁 Nội Dung Module

| File | Chủ Đề | Độ Ưu Tiên |
| ---- | ------- | ---------- |
| [1-caching-strategies.md](./1-caching-strategies.md) | `save_cache`, `restore_cache`, cache keys, cache invalidation | ⭐⭐⭐ Bắt Buộc |
| [2-workspace.md](./2-workspace.md) | `persist_to_workspace`, `attach_workspace`, workspace vs cache | ⭐⭐⭐ Bắt Buộc |
| [3-test-splitting.md](./3-test-splitting.md) | `circleci tests split`, timing-based, file-based splitting | ⭐⭐⭐ Bắt Buộc |
| [4-parallelism.md](./4-parallelism.md) | `parallelism` key, phân phối test, resource class phối hợp | ⭐⭐⭐ Bắt Buộc |
| [5-pipeline-insights.md](./5-pipeline-insights.md) | Insights dashboard, bottleneck, success rate, flaky tests | ⭐⭐ Nên Học |

---

## 🗺️ Bản Đồ Khái Niệm

```
Pipeline Optimization — Tối Ưu Hóa Pipeline
│
├── Speed — Tốc Độ
│   ├── Caching (bộ nhớ đệm)
│   │   ├── Dependency cache       ← npm, pip, maven, gradle
│   │   ├── Build artifact cache   ← compiled code, docker layers
│   │   └── Custom cache           ← bất kỳ thư mục nào
│   │
│   ├── Parallelism (song song hóa)
│   │   ├── parallelism: N         ← chạy N container cùng lúc
│   │   └── Test Splitting         ← chia test đều cho N container
│   │
│   └── Workspace (không gian làm việc)
│       ├── persist_to_workspace   ← lưu artifact từ job A
│       └── attach_workspace       ← lấy artifact ở job B
│
├── Visibility — Quan Sát Được
│   └── Pipeline Insights
│       ├── Duration trends        ← xu hướng thời gian chạy
│       ├── Success rate           ← tỷ lệ thành công
│       └── Flaky test detection   ← phát hiện test không ổn định
│
└── Resource — Tài Nguyên
    └── Resource Class             ← chọn CPU/RAM phù hợp
        ├── small: 1 vCPU, 2GB
        ├── medium: 2 vCPU, 4GB   ← mặc định
        ├── large: 4 vCPU, 8GB
        └── xlarge: 8 vCPU, 16GB
```

---

## ⏱️ Tại Sao Tối Ưu Lại Quan Trọng?

### Chi Phí Thực Tế

CircleCI tính phí theo **credit** (tín dụng):

```
credit/phút = resource_class_rate × số_container

Ví dụ:
- medium (2 vCPU):  10 credits/phút
- large  (4 vCPU):  20 credits/phút
- xlarge (8 vCPU):  40 credits/phút

Pipeline 20 phút × medium = 200 credits
Pipeline tối ưu còn 8 phút = 80 credits → tiết kiệm 60%
```

### Ảnh Hưởng Đến Năng Suất

```
Pipeline 5 phút   → Developer chờ ít, feedback loop nhanh
Pipeline 30 phút  → Developer chuyển task khác → context switching
Pipeline 60 phút+ → Bottleneck toàn team, slow delivery
```

**Quy tắc:** Mỗi pipeline nên chạy xong trong **< 10 phút** để duy trì productivity.

---

## 🔧 Chiến Lược Tổng Hợp

### Pipeline Chậm — Checklist Chẩn Đoán

```
1. Đo thời gian: dùng Pipeline Insights xem job nào chậm nhất
2. Caching: dependencies có được cache không? Cache hit rate ra sao?
3. Parallelism: test suite > 5 phút? Cần split không?
4. Workspace: có đang rebuild artifact không cần thiết không?
5. Resource class: job bị OOM hoặc CPU throttle không?
6. Network: có đang download package không cần thiết không?
```

### Thứ Tự Ưu Tiên Tối Ưu

```
1. Cache dependencies trước (thường tiết kiệm 50–70% thời gian install)
2. Parallel test splitting (thường tiết kiệm 60–80% thời gian test)
3. Workspace thay vì checkout + rebuild ở mỗi job
4. Resource class phù hợp (tránh over/under provisioning)
5. Pipeline Insights để theo dõi liên tục
```

---

## 🔗 Liên Kết Với Các Module Khác

- **02-configuration/**: `save_cache`, `restore_cache` là built-in steps
- **03-workflows/**: Parallel workflow + parallelism trong job = tối ưu tối đa
- **04-orbs/**: Nhiều orb (node, python) đã tích hợp sẵn caching
- **08-monitoring/**: Pipeline Insights nằm trong Monitoring dashboard

---

## 📊 Kết Quả Thực Tế — Real-World Results

| Kỹ Thuật | Mức Tiết Kiệm Thời Gian | Ghi Chú |
| -------- | ----------------------- | ------- |
| Caching dependencies | 50–70% | Phụ thuộc số package |
| Test splitting × 4 | 60–75% | Gần tuyến tính |
| Workspace (thay rebuild) | 20–40% | Phụ thuộc build time |
| Resource class tối ưu | 10–30% | Tránh throttle |
| Kết hợp tất cả | 80–90% | Pipeline có thể từ 30' → 4' |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn Thành
