# 09 — Nâng Cao — Advanced Topics

> Module nâng cao dành cho kỹ sư có kinh nghiệm với CircleCI muốn khai thác các tính năng phức tạp: Dynamic Config — Cấu Hình Động, Path Filtering — Lọc Theo Đường Dẫn, Matrix Jobs — Công Việc Ma Trận, Self-Hosted Runner — Máy Chạy Tự Quản Lý, và chiến lược CI/CD cho monorepo.

---

## 📚 Nội Dung Module

| File | Chủ Đề | Độ Khó |
| ---- | ------- | ------ |
| [1-dynamic-config.md](1-dynamic-config.md) | Dynamic Config — Cấu Hình Động: setup workflow, continuation | ⭐⭐⭐⭐ |
| [2-path-filtering.md](2-path-filtering.md) | Path Filtering — Lọc Theo Đường Dẫn cho monorepo | ⭐⭐⭐⭐ |
| [3-matrix-jobs.md](3-matrix-jobs.md) | Matrix Jobs — Công Việc Ma Trận: test đa version | ⭐⭐⭐ |
| [4-self-hosted-runner.md](4-self-hosted-runner.md) | Self-Hosted Runner — Máy Chạy Tự Quản Lý | ⭐⭐⭐⭐ |
| [5-monorepo-strategy.md](5-monorepo-strategy.md) | Chiến lược CI/CD toàn diện cho monorepo | ⭐⭐⭐⭐⭐ |

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn có thể:

- [ ] Triển khai Dynamic Config để tạo pipeline có điều kiện theo nội dung thay đổi
- [ ] Cấu hình Path Filtering để chỉ chạy CI cho các service bị ảnh hưởng trong monorepo
- [ ] Dùng Matrix Jobs để test song song trên nhiều version ngôn ngữ / OS
- [ ] Cài đặt và vận hành Self-Hosted Runner trên infrastructure riêng
- [ ] Thiết kế chiến lược CI/CD hoàn chỉnh cho dự án monorepo quy mô lớn

---

## 🔑 Khái Niệm Cốt Lõi

### Dynamic Config — Cấu Hình Động

```
Pipeline kích hoạt
        │
        ▼
  Setup Workflow       ← Giai đoạn 1: phân tích context
  (pipeline ngắn)
        │
        ▼
  Gọi continuation/    ← Truyền config mới được tạo động
  continue-with-config
        │
        ▼
  Main Workflow        ← Giai đoạn 2: pipeline thực sự
  (được tạo động)
```

**Ứng dụng:** Tránh chạy toàn bộ pipeline khi chỉ có 1 service thay đổi trong monorepo.

### Path Filtering — Lọc Theo Đường Dẫn

Orb `circleci/path-filtering` so sánh các file thay đổi với mapping quy tắc, sau đó set pipeline parameters để bật/tắt từng job.

### Matrix Jobs — Công Việc Ma Trận

Chạy cùng một job với nhiều tổ hợp tham số khác nhau (ví dụ: Node 18, 20, 22) song song mà không cần viết lặp code.

### Self-Hosted Runner — Máy Chạy Tự Quản Lý

Agent cài trên máy chủ của tổ chức, nhận job từ CircleCI cloud và thực thi cục bộ — phù hợp với yêu cầu bảo mật, phần cứng đặc biệt, hoặc compliance.

---

## 📋 Điều Kiện Tiên Quyết

Trước khi học module này, bạn nên nắm vững:

- **02-configuration/** — YAML syntax, parameters, commands
- **03-workflows/** — workflow design cơ bản
- **05-optimization/** — caching, workspace, parallelism

---

## 🚀 Khi Nào Cần Dùng Tính Năng Nâng Cao

| Bài Toán | Giải Pháp |
| -------- | --------- |
| Monorepo với 10+ service, build toàn bộ mỗi commit quá chậm | Dynamic Config + Path Filtering |
| Cần test trên Python 3.9, 3.10, 3.11, 3.12 song song | Matrix Jobs |
| Compliance yêu cầu code không ra ngoài mạng nội bộ | Self-Hosted Runner |
| Phần cứng đặc biệt: GPU, ARM, bare metal | Self-Hosted Runner |
| Pipeline logic khác nhau tùy loại thay đổi (feature/hotfix/release) | Dynamic Config |
| Muốn tái sử dụng runner cluster cho nhiều project | Self-Hosted Runner namespace |

---

## 🔗 Điều Hướng

← [08-monitoring/](../08-monitoring/README.md) | [10-interview-prep/](../10-interview-prep/README.md) →

---

**Cập Nhật Lần Cuối:** 2026-05-20
**Độ Khó Module:** ⭐⭐⭐⭐ Nâng Cao
