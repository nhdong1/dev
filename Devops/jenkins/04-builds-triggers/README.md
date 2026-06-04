# 04 — Build Triggers và Quản Lý Build

> Tổng quan về cách kích hoạt build trong Jenkins: từ Webhook tức thì, lịch Cron định kỳ, tham số hóa build, đến lưu trữ và chia sẻ Artifact.

---

## Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Các Loại Build Trigger](#các-loại-build-trigger)
3. [Nội Dung Chi Tiết](#nội-dung-chi-tiết)
4. [So Sánh Nhanh](#so-sánh-nhanh)
5. [Lộ Trình Học](#lộ-trình-học)

---

## Tổng Quan

**Build Trigger** (cơ chế kích hoạt build) là cách Jenkins biết *khi nào* cần chạy một job. Chọn đúng trigger ảnh hưởng trực tiếp đến:

- **Tốc độ phản hồi** — Webhook phản hồi trong vài giây, Poll SCM mất vài phút
- **Tải hệ thống** — Poll SCM liên tục gây tải không cần thiết
- **Độ linh hoạt** — Build Parameters cho phép kiểm soát thủ công theo ngữ cảnh
- **Truy xuất nguồn gốc** — Artifact Fingerprint giúp theo dõi từng build

---

## Các Loại Build Trigger

### 1. Webhook (Kích Hoạt Qua Sự Kiện)

Jenkins nhận HTTP POST từ GitHub/GitLab khi có sự kiện (push, pull request, merge). Đây là cách **được khuyến nghị** vì phản hồi tức thì và không tốn tài nguyên polling.

```
Developer push → GitHub → Webhook POST → Jenkins → Trigger build
```

**File:** [1-webhooks.md](1-webhooks.md)

---

### 2. Cron Scheduling và Poll SCM (Lên Lịch Định Kỳ)

| Phương pháp | Mô tả | Khi nào dùng |
|-------------|-------|--------------|
| **Cron** | Chạy job theo lịch cố định, dù có thay đổi code hay không | Báo cáo đêm, build release định kỳ |
| **Poll SCM** | Jenkins tự kiểm tra repository theo lịch, chỉ build khi có thay đổi | Khi không thể dùng Webhook |

**File:** [2-cron-scheduling.md](2-cron-scheduling.md)

---

### 3. Build Parameters (Tham Số Build)

Cho phép người dùng truyền tham số khi khởi động build thủ công hoặc qua API. Phù hợp cho:

- Deploy lên môi trường cụ thể (`dev`, `staging`, `production`)
- Chọn phiên bản để build
- Bật/tắt tính năng kiểm thử

**Các loại tham số:** `String`, `Choice`, `Boolean`, `Password`, `File`

**File:** [3-build-parameters.md](3-build-parameters.md)

---

### 4. Build Artifacts (Kết Quả Build)

Quản lý các file được sinh ra trong quá trình build:

| Khái niệm | Mô tả |
|-----------|-------|
| **Archive Artifacts** | Lưu trữ file (JAR, WAR, binary) để tải về sau |
| **Stash/Unstash** | Truyền file giữa các stage hoặc agent trong cùng pipeline |
| **Fingerprint** | Tạo hash MD5 để theo dõi artifact qua nhiều job |

**File:** [4-build-artifacts.md](4-build-artifacts.md)

---

## So Sánh Nhanh

| Trigger | Độ Trễ | Tải Hệ Thống | Khó Cấu Hình | Khi Nào Dùng |
|---------|--------|--------------|--------------|--------------|
| Webhook | ~1 giây | Rất thấp | Trung bình | Luôn ưu tiên nếu có thể |
| Poll SCM | 1–15 phút | Trung bình | Thấp | Khi firewall chặn webhook |
| Cron | Đúng lịch | Thấp | Thấp | Build định kỳ, báo cáo |
| Build Parameters | Thủ công | Không có | Thấp | Deploy, release thủ công |
| Upstream Trigger | Ngay sau job cha | Thấp | Thấp | Pipeline nhiều bước |

---

## Lộ Trình Học

```
Bước 1: Hiểu Webhook ──────► 1-webhooks.md
        ↓
Bước 2: Nắm Cron/Poll SCM ──► 2-cron-scheduling.md
        ↓
Bước 3: Build Parameters ───► 3-build-parameters.md
        ↓
Bước 4: Quản lý Artifacts ──► 4-build-artifacts.md
```

### Checklist Năng Lực

#### Beginner
- [ ] Cấu hình GitHub Webhook để trigger build khi push
- [ ] Hiểu cú pháp Cron cơ bản (`H/15 * * * *`)
- [ ] Thêm String Parameter vào job

#### Intermediate
- [ ] Bảo mật Webhook bằng Secret Token
- [ ] Dùng `stash`/`unstash` để truyền artifact giữa các stage
- [ ] Cấu hình `choice` parameter cho môi trường deploy

#### Advanced
- [ ] Thiết kế Webhook với phân loại sự kiện (push vs PR vs tag)
- [ ] Implement Artifact Fingerprinting cho truy xuất nguồn gốc
- [ ] Parameterized Trigger từ upstream job sang downstream job

---

## Câu Hỏi Phỏng Vấn Liên Quan

- **Webhook vs Poll SCM: khi nào dùng cái nào và tại sao?**
- **H symbol trong cron Jenkins có ý nghĩa gì? Tại sao không dùng `* * * * *`?**
- **Stash và Archive Artifact khác nhau như thế nào?**
- **Làm thế nào để bảo mật Webhook endpoint không bị giả mạo?**
- **Build Parameters được dùng như thế nào trong Scripted Pipeline?**

---

**Thời Gian Học:** 3–4 giờ
**Độ Khó:** ⭐⭐ Trung Bình
**Cập Nhật:** 2026-05-10
