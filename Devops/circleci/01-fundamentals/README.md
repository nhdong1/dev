# 01 — Nền Tảng CircleCI — Fundamentals

> Module giới thiệu CI/CD (Continuous Integration / Continuous Deployment — Tích Hợp Liên Tục / Triển Khai Liên Tục) và kiến trúc cốt lõi của CircleCI. Đây là điểm khởi đầu bắt buộc trước khi học các module nâng cao.

---

## 📚 Nội Dung Module Này

| File | Chủ Đề | Độ Khó |
| ---- | ------ | ------ |
| [1-cicd-concepts.md](1-cicd-concepts.md) | CI vs CD (Delivery) vs CD (Deployment) — phân biệt khái niệm | ⭐ |
| [2-circleci-architecture.md](2-circleci-architecture.md) | Pipeline → Workflow → Job → Step — cấu trúc phân cấp | ⭐ |
| [3-executors.md](3-executors.md) | Docker, Machine, macOS, Windows executor | ⭐⭐ |
| [4-resource-classes.md](4-resource-classes.md) | Resource Class — CPU/RAM và chiến lược tối ưu chi phí | ⭐⭐ |
| [5-circleci-vs-alternatives.md](5-circleci-vs-alternatives.md) | So sánh Jenkins, GitHub Actions, GitLab CI | ⭐⭐ |

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn có thể:

- [ ] Giải thích CI, CD Delivery và CD Deployment — sự khác biệt và quan hệ giữa ba khái niệm
- [ ] Mô tả kiến trúc phân cấp của CircleCI: Pipeline → Workflow → Job → Step
- [ ] Chọn đúng Executor — Môi Trường Thực Thi cho từng tình huống
- [ ] Hiểu Resource Class — Lớp Tài Nguyên và tác động đến hiệu năng, chi phí
- [ ] So sánh CircleCI với Jenkins, GitHub Actions, GitLab CI một cách có lập luận

---

## ⚡ Tóm Tắt Nhanh — TL;DR

### CI/CD là gì?

```
CI — Continuous Integration — Tích Hợp Liên Tục:
  Developer push code → tự động build + test → phát hiện lỗi sớm

CD Delivery — Continuous Delivery — Chuyển Giao Liên Tục:
  Code luôn sẵn sàng để release → cần approval thủ công để deploy

CD Deployment — Continuous Deployment — Triển Khai Liên Tục:
  Tự động deploy lên production sau khi test xanh, không cần can thiệp
```

### Kiến Trúc CircleCI

```
Pipeline (toàn bộ quá trình)
  └── Workflow (nhóm các job, định nghĩa thứ tự chạy)
        └── Job (đơn vị công việc, chạy trên 1 executor)
              └── Step (lệnh cụ thể: checkout, run, cache...)
```

### Executor — Môi Trường Thực Thi

```
Docker   → Nhẹ, nhanh, phù hợp hầu hết use case
Machine  → Full VM, cần Docker-in-Docker hoặc systemd
macOS    → Build iOS/macOS app
Windows  → Build .NET, Windows-specific app
```

---

## 🗺️ Lộ Trình Học Module Này

```
1. Đọc 1-cicd-concepts.md         (30–45 phút)
   → Nắm vững 3 khái niệm CI, CD Delivery, CD Deployment

2. Đọc 2-circleci-architecture.md (45–60 phút)
   → Hiểu cấu trúc phân cấp, tạo pipeline "Hello World" đầu tiên

3. Đọc 3-executors.md             (45–60 phút)
   → Thử nghiệm Docker executor với image thực tế

4. Đọc 4-resource-classes.md      (30 phút)
   → Biết cách chọn resource class tối ưu chi phí

5. Đọc 5-circleci-vs-alternatives.md (30 phút)
   → Chuẩn bị cho câu hỏi "Tại sao chọn CircleCI?"
```

---

## 🔑 Câu Hỏi Kiểm Tra Nhanh

Trả lời được các câu này thì bạn đã nắm vững module:

1. Sự khác biệt cốt lõi giữa **CI** và **CD** là gì?
2. **Pipeline** khác **Workflow** ở điểm nào?
3. Khi nào nên dùng **Machine executor** thay vì **Docker executor**?
4. **Resource Class `large`** phù hợp với tình huống nào?
5. Tại sao **CircleCI** có thể nhanh hơn **GitHub Actions** trong nhiều trường hợp?

---

## 📖 Học Theo Thứ Tự Đề Xuất

```
[Bắt đầu ở đây] → 1-cicd-concepts.md
                      ↓
               2-circleci-architecture.md
                      ↓
               3-executors.md
                      ↓
               4-resource-classes.md
                      ↓
               5-circleci-vs-alternatives.md
                      ↓
[Module tiếp theo] → ../02-configuration/README.md
```

---

**Thời Gian Ước Tính:** 3–4 giờ  
**Mức Độ:** Beginner — Người Mới  
**Điều Kiện Tiên Quyết:** Biết cơ bản về Git và khái niệm lập trình
