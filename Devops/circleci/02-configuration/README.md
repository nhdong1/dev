# ⚙️ Module 02 — Cấu Hình CircleCI (Configuration)

> Hướng dẫn toàn diện về cấu hình `.circleci/config.yml` — từ cú pháp YAML cơ bản đến các kỹ thuật nâng cao như Parameters — Tham Số, Commands — Lệnh Tái Sử Dụng và Environment Variables — Biến Môi Trường.

---

## 📋 Mục Lục Module

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-yaml-syntax.md](1-yaml-syntax.md) | Cú pháp YAML, anchors, aliases, schema đầy đủ | ⭐⭐ |
| [2-jobs-and-steps.md](2-jobs-and-steps.md) | Định nghĩa job, built-in steps, executor inline | ⭐⭐ |
| [3-commands.md](3-commands.md) | Commands — Lệnh Tái Sử Dụng tùy chỉnh | ⭐⭐ |
| [4-parameters.md](4-parameters.md) | Pipeline, job, command parameters | ⭐⭐⭐ |
| [5-environment-variables.md](5-environment-variables.md) | Built-in, project-level, org-level vars | ⭐⭐ |

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn có thể:

- [ ] Viết file `.circleci/config.yml` hoàn chỉnh từ đầu không cần tài liệu
- [ ] Hiểu và sử dụng đúng cú pháp YAML: anchors, aliases, multi-line strings
- [ ] Định nghĩa jobs với các loại executor khác nhau
- [ ] Tạo reusable commands — lệnh tái sử dụng để giảm lặp code
- [ ] Sử dụng parameters — tham số để tạo pipeline linh hoạt
- [ ] Quản lý environment variables — biến môi trường an toàn ở các cấp độ khác nhau

---

## 🗂️ Cấu Trúc File Config Đầy Đủ

File `.circleci/config.yml` có cấu trúc phân cấp theo thứ tự cố định:

```
version: 2.1
│
├── orbs:          # Gói tích hợp tái sử dụng từ registry
├── parameters:    # Pipeline parameters — Tham Số Pipeline
├── executors:     # Môi trường thực thi tái sử dụng
├── commands:      # Lệnh tùy chỉnh tái sử dụng
├── jobs:          # Định nghĩa các đơn vị công việc
└── workflows:     # Điều phối thứ tự thực thi jobs
```

---

## 🧩 Sơ Đồ Quan Hệ Giữa Các Thành Phần

```
Pipeline (1 lần chạy)
└── Workflow (1 hoặc nhiều)
    └── Job (chạy song song hoặc tuần tự)
        ├── Executor (môi trường: Docker/Machine/macOS)
        └── Steps (các bước thực thi)
            ├── checkout
            ├── run: <shell command>
            ├── restore_cache / save_cache
            ├── persist_to_workspace / attach_workspace
            ├── store_artifacts
            ├── store_test_results
            └── <custom command>
```

---

## 📝 Template Config Cơ Bản Theo Version

### Version 2.1 — Phiên Bản Khuyến Nghị (Hiện Tại)

```yaml
version: 2.1

# Orbs — Gói Tích Hợp
orbs:
  node: circleci/node@5.1.0

# Executors — Môi Trường Thực Thi Tái Sử Dụng
executors:
  default-executor:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

# Commands — Lệnh Tái Sử Dụng
commands:
  install-deps:
    description: "Cài đặt dependencies — phụ thuộc với cache"
    steps:
      - restore_cache:
          keys:
            - node-v1-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: node-v1-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm

# Jobs — Công Việc
jobs:
  test:
    executor: default-executor
    steps:
      - checkout
      - install-deps
      - run:
          name: Chạy kiểm thử
          command: npm test

  build:
    executor: default-executor
    steps:
      - checkout
      - install-deps
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist/

# Workflows — Luồng Công Việc
workflows:
  main:
    jobs:
      - test
      - build:
          requires:
            - test
```

---

## ⚡ Sự Khác Biệt Giữa version: 2.0 và version: 2.1

| Tính Năng | version: 2.0 | version: 2.1 |
|-----------|-------------|-------------|
| Orbs | ❌ Không hỗ trợ | ✅ Hỗ trợ đầy đủ |
| Reusable Executors | ❌ | ✅ Khai báo 1 lần, dùng nhiều nơi |
| Reusable Commands | ❌ | ✅ Custom commands |
| Pipeline Parameters | ❌ | ✅ Truyền tham số vào pipeline |
| `when` / `unless` | ❌ | ✅ Điều kiện trong steps |
| Matrix Jobs | ❌ | ✅ Chạy nhiều biến thể |

> **Luôn dùng `version: 2.1`** — đây là phiên bản hiện tại với đầy đủ tính năng.

---

## 🔑 Các Khái Niệm Then Chốt

### 1. Job — Công Việc

- Đơn vị thực thi nhỏ nhất trong CircleCI
- Mỗi job chạy trong một môi trường độc lập (container/VM riêng)
- Jobs không chia sẻ filesystem — dùng **Workspace** để truyền file

### 2. Step — Bước

- Một hành động cụ thể trong job
- Chạy tuần tự từ trên xuống dưới
- Nếu một step thất bại → job dừng (trừ khi dùng `when: always`)

### 3. Executor — Môi Trường Thực Thi

- Định nghĩa **nơi** job chạy: Docker container, Linux VM, macOS, Windows
- Có thể tái sử dụng bằng cách khai báo trong block `executors:`

### 4. Command — Lệnh Tái Sử Dụng

- Nhóm nhiều steps thành một đơn vị có thể gọi lại
- Tương tự như hàm (function) trong lập trình
- Hỗ trợ parameters — tham số để linh hoạt hóa

### 5. Parameter — Tham Số

- Biến được truyền vào pipeline, job hoặc command
- Kiểu dữ liệu: `string`, `boolean`, `integer`, `enum`, `steps`, `env_var_name`

### 6. Environment Variable — Biến Môi Trường

- Cấp độ: Built-in > Org-level Context > Project-level > Job-level
- Không bao giờ hardcode secrets trong config.yml

---

## 🧪 Kiểm Tra Cấu Hình Trước Khi Push

```bash
# Validate cú pháp config — kiểm tra tính hợp lệ của cú pháp
circleci config validate

# Process config — xem config sau khi orbs được expand
circleci config process .circleci/config.yml

# Chạy job cục bộ với Docker
circleci local execute --job <job-name>

# Chạy job cục bộ với biến môi trường
circleci local execute --job <job-name> \
  --env MY_VAR=value \
  --env OTHER_VAR=value2
```

---

## 🚦 Thứ Tự Học Đề Xuất

```
1. 1-yaml-syntax.md        → Nắm cú pháp YAML trước (30 phút)
2. 2-jobs-and-steps.md     → Hiểu cách định nghĩa job (45 phút)
3. 3-commands.md           → Tái sử dụng logic với commands (30 phút)
4. 4-parameters.md         → Linh hoạt hóa config với parameters (45 phút)
5. 5-environment-variables.md → Quản lý secrets đúng cách (30 phút)
```

**Tổng thời gian ước tính:** 3–4 giờ học + 2–4 giờ thực hành

---

## 💡 Best Practices — Thực Hành Tốt Nhất

1. **Luôn dùng `version: 2.1`** để có đầy đủ tính năng
2. **Khai báo executors tập trung** — không lặp lại image trong từng job
3. **Dùng commands cho logic lặp** — DRY (Don't Repeat Yourself)
4. **Cache key nên bao gồm checksum** của file lock để tự động invalidate
5. **Không hardcode secrets** — dùng Contexts hoặc project env vars
6. **Validate config trước khi push** — tiết kiệm thời gian chờ CI
7. **Dùng `cimg/` images** — CircleCI convenience images được tối ưu sẵn

---

**Tiếp theo:** [1-yaml-syntax.md](1-yaml-syntax.md) — Cú Pháp YAML Đầy Đủ
