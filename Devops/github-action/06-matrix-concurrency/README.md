# 06 — Matrix Strategy & Concurrency Control

> Chạy song song nhiều cấu hình, kiểm soát luồng thực thi — nền tảng để xây dựng pipeline hiệu quả và tiết kiệm chi phí.

---

## 📚 Mục Lục

| File | Nội Dung | Độ Khó |
|---|---|---|
| [1-matrix-strategy.md](1-matrix-strategy.md) | Matrix Strategy — test đa phiên bản, đa OS, include/exclude | ⭐⭐ |
| [2-concurrency.md](2-concurrency.md) | Concurrency — groups, cancel-in-progress, queue | ⭐⭐ |
| [3-fan-out-fan-in.md](3-fan-out-fan-in.md) | Fan-out & Fan-in — phân tán và tập hợp kết quả | ⭐⭐⭐ |

---

## 🎯 Tổng Quan

### Matrix Strategy (Chiến Lược Ma Trận)

Matrix Strategy cho phép một job chạy song song trên nhiều cấu hình khác nhau — ví dụ test trên Node.js 18/20/22 đồng thời, hoặc trên Ubuntu + Windows + macOS cùng lúc. Thay vì viết lặp lại nhiều jobs, bạn khai báo một lần và GitHub Actions tự nhân bản.

```yaml
strategy:
  matrix:
    node: [18, 20, 22]
    os: [ubuntu-latest, windows-latest]
  # → tạo 6 jobs chạy song song
```

### Concurrency (Kiểm Soát Đồng Thời)

Concurrency Control giải quyết vấn đề khi nhiều workflow run chạy cùng lúc trên cùng một nhánh — ngăn tình trạng deploy chồng chéo, race condition (điều kiện tranh chấp), hoặc lãng phí tài nguyên.

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
  # → hủy run cũ, chỉ giữ run mới nhất
```

### Fan-out & Fan-in (Phân Tán & Tập Hợp)

Pattern nâng cao: phân tán công việc thành nhiều jobs song song (fan-out — phân tán), sau đó tập hợp kết quả về một job tổng hợp (fan-in — tập hợp). Thường dùng cho kiểm tra song song lớn hoặc build đa nền tảng.

---

## 🏗️ Kiến Trúc Tổng Quan

```
Workflow Trigger
      │
      ▼
┌─────────────┐
│  Matrix Job │  strategy.matrix → nhân bản thành N jobs
│  (fan-out)  │
└──┬──┬──┬───┘
   │  │  │
   ▼  ▼  ▼
 Job1 Job2 Job3  ← chạy song song
   │  │  │
   └──┴──┘
      │
      ▼
┌─────────────┐
│  Merge Job  │  needs: [job1, job2, job3] → fan-in
│  (fan-in)   │
└─────────────┘
```

---

## 📊 So Sánh Các Kỹ Thuật

| Kỹ Thuật | Mục Đích | Khi Dùng |
|---|---|---|
| **Matrix** | Chạy cùng logic trên nhiều cấu hình | Test đa version, đa OS |
| **Concurrency group** | Giới hạn số run đồng thời | Deploy, preview environments |
| **cancel-in-progress** | Hủy run cũ khi có run mới | Feature branches, PR checks |
| **Fan-out** | Chia nhỏ công việc lớn thành nhiều job | Build song song, sharded tests |
| **Fan-in** | Gom kết quả từ nhiều job | Aggregate reports, final deploy |

---

## ⚡ Lợi Ích Thực Tế

### Với Matrix Strategy

- **Tốc độ**: Test 3 version Node.js trong thời gian của 1 (chạy song song)
- **Coverage (Phạm Vi Bao Phủ)**: Phát hiện lỗi tương thích sớm
- **DRY (Don't Repeat Yourself — Không Lặp Lại)**: Một job definition, nhiều cấu hình

### Với Concurrency

- **An toàn**: Ngăn deploy đồng thời gây xung đột
- **Tiết kiệm**: Hủy runs không cần thiết, giảm billing minutes
- **Trật tự**: Đảm bảo chỉ run mới nhất được triển khai

---

## 🔗 Liên Kết Liên Quan

- [07-caching-performance](../07-caching-performance/) — Cache giữa các matrix jobs
- [02-ci-pipeline](../02-ci-pipeline/) — Ứng dụng matrix vào CI pipeline
- [03-cd-deployments](../03-cd-deployments/) — Concurrency trong deployment pipeline

---

**Cập Nhật:** 2026-05-11 | **Phiên Bản:** 1.0
