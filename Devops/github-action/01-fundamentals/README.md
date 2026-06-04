# 01 — Nền Tảng GitHub Actions

> Hiểu đúng kiến trúc và khái niệm cốt lõi là nền tảng để làm chủ mọi tính năng nâng cao của GitHub Actions.

## 📚 Mục Lục Chương

| File | Nội Dung | Độ Quan Trọng |
|---|---|---|
| [1-workflow-syntax.md](1-workflow-syntax.md) | Cú pháp YAML workflow đầy đủ | ⭐⭐⭐ |
| [2-events-triggers.md](2-events-triggers.md) | Tất cả events kích hoạt workflow | ⭐⭐⭐ |
| [3-runners.md](3-runners.md) | GitHub-hosted vs self-hosted runners | ⭐⭐⭐ |
| [4-contexts-expressions.md](4-contexts-expressions.md) | Contexts, expressions, functions | ⭐⭐⭐ |
| [5-environment-variables.md](5-environment-variables.md) | Biến môi trường mặc định & tùy chỉnh | ⭐⭐ |

---

## 🏗️ Kiến Trúc GitHub Actions

### Tổng Quan Hệ Thống

```
GitHub Repository
│
├── .github/
│   └── workflows/
│       ├── ci.yml          ← Workflow file (File Workflow)
│       ├── cd.yml
│       └── release.yml
│
Event xảy ra (push, PR, schedule...)
         │
         ▼
  GitHub Actions Platform
         │
         ├── Phân tích Workflow file
         ├── Tạo Workflow Run (Lần Chạy Workflow)
         └── Phân phối Jobs đến Runners
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
   GitHub-hosted Runner   Self-hosted Runner
   (ubuntu-latest)        (máy chủ tự quản lý)
          │
          └── Thực thi Steps theo thứ tự
```

### 5 Thành Phần Cốt Lõi

```
Events (Sự Kiện)
    │  "Điều gì kích hoạt workflow?"
    │  push, pull_request, schedule, workflow_dispatch...
    ▼
Workflows (Luồng Công Việc)
    │  "Tệp YAML định nghĩa toàn bộ quy trình"
    │  .github/workflows/*.yml
    ▼
Jobs (Công Việc)
    │  "Tập hợp các steps chạy trên cùng một runner"
    │  Có thể chạy song song hoặc tuần tự (qua needs:)
    ▼
Steps (Bước Thực Thi)
    │  "Lệnh shell hoặc action đơn lẻ trong một job"
    │  run: echo "hello" | uses: actions/checkout@v4
    ▼
Actions (Hành Động Tái Sử Dụng)
       "Đơn vị code tái sử dụng nhỏ nhất"
       Từ Marketplace hoặc tự viết
```

---

## 🔄 Vòng Đời Một Workflow Run

```
1. Trigger (Kích Hoạt)
   └── Event xảy ra → GitHub nhận diện workflow files

2. Queue (Xếp Hàng)
   └── Workflow Run được tạo, Jobs được đưa vào queue

3. Runner Pickup (Runner Nhận Việc)
   └── Runner khả dụng nhận job → cài đặt môi trường

4. Job Execution (Thực Thi Job)
   ├── Step 1: uses/run
   ├── Step 2: uses/run
   └── Step N: uses/run

5. Completion (Hoàn Thành)
   ├── success → các jobs phụ thuộc tiếp tục
   ├── failure → workflow dừng (trừ khi if: failure())
   └── cancelled → người dùng hủy thủ công
```

---

## 📋 Cấu Trúc File Workflow Tối Thiểu

```yaml
# .github/workflows/ci.yml
name: CI Pipeline              # Tên hiển thị trên GitHub UI

on:                            # Events kích hoạt
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:                       # Job ID (tùy đặt tên)
    runs-on: ubuntu-latest     # Runner sử dụng
    steps:
      - uses: actions/checkout@v4          # Action từ Marketplace
      - name: Run tests
        run: npm test                      # Lệnh shell
```

---

## 🧩 Các Khái Niệm Cần Nắm Vững

### Workflow vs Job vs Step vs Action

| Khái Niệm | Định Nghĩa | Ví Dụ |
|---|---|---|
| **Workflow** | Toàn bộ quy trình tự động hóa, định nghĩa trong file `.yml` | `ci.yml`, `deploy.yml` |
| **Job** | Nhóm steps chạy trên cùng runner, có thể song song | `build`, `test`, `deploy` |
| **Step** | Đơn vị thực thi trong job — một lệnh hoặc một action | `run: npm install` |
| **Action** | Code tái sử dụng được, đóng gói logic phức tạp | `actions/checkout@v4` |

### GitHub-hosted vs Self-hosted Runners

| Tiêu Chí | GitHub-hosted | Self-hosted |
|---|---|---|
| **Quản lý** | GitHub lo | Bạn tự quản lý |
| **Hệ điều hành** | Ubuntu, Windows, macOS | Bất kỳ OS nào |
| **Chi phí** | Tính theo phút | Tính tiền server |
| **Tốc độ khởi động** | ~30–60 giây | ~5–10 giây |
| **Mạng nội bộ** | Không truy cập được | Truy cập được |
| **Phù hợp** | Hầu hết use cases | Private network, custom hardware |

### Contexts và Expressions

```yaml
steps:
  - name: Show context values
    run: |
      echo "Repo: ${{ github.repository }}"       # tên repo
      echo "Branch: ${{ github.ref_name }}"       # tên nhánh
      echo "Actor: ${{ github.actor }}"           # người trigger
      echo "SHA: ${{ github.sha }}"               # commit hash
      echo "Event: ${{ github.event_name }}"      # loại event
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp (Topic Này)

### Câu 1: Giải thích sự khác nhau giữa Job và Step?

**Trả lời mẫu:**
- **Job** là tập hợp các steps chạy trên cùng một runner, trong cùng một môi trường. Mỗi job bắt đầu với runner sạch (fresh environment). Các jobs chạy song song mặc định, trừ khi dùng `needs:` để tạo phụ thuộc.
- **Step** là đơn vị thực thi nhỏ nhất trong một job — có thể là lệnh shell (`run:`) hoặc một action tái sử dụng (`uses:`). Các steps luôn chạy tuần tự trong cùng một job và chia sẻ filesystem.

**Điểm quan trọng:** Data không tự share giữa các jobs — phải dùng Artifacts hoặc cache.

### Câu 2: Khi nào dùng `needs:` keyword?

**Trả lời mẫu:**
`needs:` tạo dependency (phụ thuộc) giữa các jobs. Job B với `needs: [job-A]` sẽ chỉ bắt đầu sau khi Job A thành công. Dùng khi:
- Deploy chỉ được chạy sau khi test pass
- Một job cần output từ job trước
- Muốn tạo pipeline tuần tự (sequential pipeline)

### Câu 3: Sự khác nhau giữa `run:` và `uses:`?

- `run:` — chạy lệnh shell trực tiếp (bash, PowerShell, cmd)
- `uses:` — gọi một action từ Marketplace, repo khác, hoặc local directory

---

## 📂 Điều Hướng

- [→ Workflow Syntax](1-workflow-syntax.md)
- [→ Events & Triggers](2-events-triggers.md)
- [→ Runners](3-runners.md)
- [→ Contexts & Expressions](4-contexts-expressions.md)
- [→ Environment Variables](5-environment-variables.md)
- [↑ Quay lại INDEX](../INDEX.md)

---

**Cập Nhật:** 2026-05-11 | **Trạng Thái:** ✅ Hoàn Thành
