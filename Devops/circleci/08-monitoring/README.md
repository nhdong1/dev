# 🔍 Giám Sát & Xử Lý Sự Cố Pipeline — Monitoring & Troubleshooting Module

> Hướng dẫn quan sát pipeline (Pipeline — Luồng Chạy CI/CD), chẩn đoán lỗi có hệ thống, debug qua SSH, xử lý OOM/timeout và giảm flaky tests (Kiểm Thử Không Ổn Định).

## 📚 Mục Lục Module

| File | Chủ Đề | Độ Ưu Tiên |
|------|---------|------------|
| [1-pipeline-insights.md](./1-pipeline-insights.md) | Insights dashboard — Giám sát success rate, duration | ⭐⭐⭐ Vận hành |
| [2-ssh-debugging.md](./2-ssh-debugging.md) | Rerun with SSH — Gỡ lỗi trực tiếp trên runner | ⭐⭐⭐ Khi pipeline đỏ |
| [3-common-errors.md](./3-common-errors.md) | OOM, timeout, exit code, cache miss | ⭐⭐⭐ Tham khảo hàng ngày |
| [4-flaky-tests.md](./4-flaky-tests.md) | Phát hiện và xử lý flaky tests | ⭐⭐ Chất lượng CI |

---

## 🎯 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- Dùng **Pipeline Insights** để theo dõi health pipeline và ưu tiên tối ưu (chi tiết tối ưu: xem [05-optimization/5-pipeline-insights.md](../05-optimization/5-pipeline-insights.md))
- **Rerun with SSH** để tái hiện lỗi trên môi trường giống production CI
- Đọc log/step output và map lỗi với **nguyên nhân gốc** (OOM, timeout, permission, network)
- Giảm **false failures** do flaky tests bằng quy trình và cấu hình phù hợp
- Kết hợp **notifications** ([07-integration/6-notifications.md](../07-integration/6-notifications.md)) để team phản ứng nhanh

---

## 🔄 Quy Trình Xử Lý Sự Cố Điển Hình

```
Pipeline FAILED
      │
      ▼
┌─────────────────┐
│ 1. Xác định job │  ← Workflow graph, job đỏ đầu tiên
│    / step lỗi   │
└────────┬────────┘
         ▼
┌─────────────────┐     Không rõ nguyên nhân?
│ 2. Đọc log +    │──────────────────────────────┐
│    exit code    │                              ▼
└────────┬────────┘                    ┌─────────────────┐
         │                             │ 3. Rerun with   │
         │ Lỗi môi trường / tương tác  │    SSH          │
         ▼                             └────────┬────────┘
┌─────────────────┐                              │
│ 4. Tra 3-       │◄─────────────────────────────┘
│ common-errors   │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 5. Insights:    │  Flaky? → 4-flaky-tests.md
│    trend, flaky │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 6. Fix + verify │  Rerun pipeline / PR
└─────────────────┘
```

---

## 🔑 Khái Niệm Cốt Lõi

| Khái Niệm | Giải Thích Ngắn |
|-----------|-----------------|
| **Pipeline** — Luồng Chạy | Một lần chạy CI/CD gắn với commit/PR |
| **Workflow** — Luồng Công Việc | Tập job và quan hệ `requires` trong một pipeline |
| **Exit Code** — Mã Thoát | `0` = thành công; khác `0` = step/job fail (thường là 1) |
| **OOM** — Out Of Memory | Container/job hết RAM, bị kernel kill |
| **Resource Class** — Lớp Tài Nguyên | CPU/RAM của runner (xem `01-fundamentals/4-resource-classes.md`) |
| **Flaky Test** — Kiểm Thử Không Ổn Định | Cùng commit đôi khi pass, đôi khi fail không do code |

---

## ⚡ Quick Reference — Khi Nào Đọc File Nào?

| Triệu Chứng | File |
|-------------|------|
| Pipeline chậm dần, success rate giảm | [1-pipeline-insights.md](./1-pipeline-insights.md) |
| Cần chạy lệnh thủ công trong container CI | [2-ssh-debugging.md](./2-ssh-debugging.md) |
| `Killed`, `137`, timeout, permission denied | [3-common-errors.md](./3-common-errors.md) |
| Test đỏ ngẫu nhiên, pass khi rerun | [4-flaky-tests.md](./4-flaky-tests.md) |

---

## 🔗 Liên Kết Module Liên Quan

| Nhu Cầu | Module |
|---------|--------|
| Tối ưu duration, bottleneck | [05-optimization/](../05-optimization/) |
| Slack/email khi fail | [07-integration/6-notifications.md](../07-integration/6-notifications.md) |
| Resource class, executor | [01-fundamentals/](../01-fundamentals/) |
| Caching, parallelism | [05-optimization/1-caching-strategies.md](../05-optimization/1-caching-strategies.md) |

---

## ✅ Checklist Module

- [ ] Biết mở Insights và đọc success rate, duration theo job
- [ ] Đã thử Rerun with SSH ít nhất một lần (và hiểu giới hạn bảo mật)
- [ ] Nhận diện OOM (`137`, `Killed`) và tăng resource class hoặc giảm parallelism
- [ ] Có quy trình ghi nhận flaky test (ticket + retry policy)
- [ ] Cấu hình thông báo khi pipeline `main` fail

**Thời gian học ước tính:** 3–4 giờ
