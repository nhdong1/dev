# Workspace — Không Gian Làm Việc Chung

> Workspace (không gian làm việc chung) cho phép truyền file và artifact từ job này sang job khác trong cùng một workflow, tránh phải thực hiện lại các bước tốn thời gian như build hay compile.

---

## 🧠 Khái Niệm Cốt Lõi

### Workspace Là Gì?

```
Workflow: build → test → deploy

Job: build
  ├── checkout
  ├── npm run build          → tạo ra ./dist/
  └── persist_to_workspace   → lưu ./dist/ lên workspace storage

Job: test
  ├── attach_workspace       → lấy ./dist/ từ workspace
  └── npm test               → test trên file đã build sẵn

Job: deploy
  ├── attach_workspace       → lấy ./dist/ từ workspace
  └── aws s3 sync dist/ ...  → deploy artifact đã có sẵn
```

### Tại Sao Cần Workspace?

**Không có workspace:**
```
build job: checkout → npm install → npm run build → (artifact bị mất)
test job:  checkout → npm install → npm run build → npm test  ← build lại!
deploy:    checkout → npm install → npm run build → deploy    ← build lại!
```

**Có workspace:**
```
build job: checkout → npm install → npm run build → persist_to_workspace
test job:  attach_workspace → npm test      ← dùng artifact từ build
deploy:    attach_workspace → deploy        ← dùng artifact từ build
```

---

## ⚙️ Cú Pháp

### `persist_to_workspace` — Lưu Vào Workspace

```yaml
- persist_to_workspace:
    root: /home/circleci/project  # Thư mục gốc (thường là . hoặc /tmp)
    paths:
      - dist/           # Đường dẫn tương đối so với root
      - build/
      - coverage/
```

| Tham Số | Mô Tả | Ghi Chú |
| ------- | ----- | ------- |
| `root` | Thư mục gốc tuyệt đối | Thường là `.` (project directory) |
| `paths` | Danh sách thư mục/file cần lưu | Đường dẫn tương đối so với `root` |

### `attach_workspace` — Gắn Workspace

```yaml
- attach_workspace:
    at: /home/circleci/project  # Nơi giải nén workspace (thường là .)
```

| Tham Số | Mô Tả |
| ------- | ----- |
| `at` | Thư mục đích để giải nén nội dung workspace |

---

## 🔄 Luồng Hoạt Động Chi Tiết

```
persist_to_workspace:
  root: .                    # /home/circleci/project/
  paths:
    - dist/                  # Lưu: /home/circleci/project/dist/

attach_workspace:
  at: .                      # Giải nén tại: /home/circleci/project/
                             # Kết quả: /home/circleci/project/dist/ có sẵn
```

### Workspace Có Thể Tích Lũy — Cumulative Workspace

Nhiều job có thể `persist_to_workspace` và workspace **tích lũy** tất cả các file:

```
Job A: persist dist/       → workspace có: dist/
Job B: persist coverage/   → workspace có: dist/ + coverage/
Job C: attach_workspace    → nhận cả dist/ + coverage/
```

---

## 📋 Ví Dụ Thực Tế

### Ví Dụ 1: Node.js Build → Test → Deploy

```yaml
version: 2.1

jobs:
  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - node-v1-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: node-v1-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm
      - run: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist/
            - node_modules/  # Truyền node_modules để test job không cần install lại

  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run:
          name: Chạy kiểm thử
          command: npm test  # Không cần npm install!

  test-e2e:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run:
          name: Kiểm thử E2E — End-to-End Testing
          command: npm run test:e2e

  deploy-staging:
    docker:
      - image: cimg/node:20.0
    steps:
      - attach_workspace:
          at: .              # Không cần checkout! Chỉ cần dist/
      - run:
          name: Triển khai lên staging
          command: aws s3 sync dist/ s3://my-app-staging

workflows:
  ci-cd:
    jobs:
      - build
      - test:
          requires:
            - build
      - test-e2e:
          requires:
            - build
      - deploy-staging:
          requires:
            - test
            - test-e2e
          filters:
            branches:
              only: main
```

### Ví Dụ 2: Go — Compile Once, Test Everywhere

```yaml
jobs:
  compile:
    docker:
      - image: cimg/go:1.21
    steps:
      - checkout
      - restore_cache:
          keys:
            - go-mod-v1-{{ checksum "go.sum" }}
      - run: go mod download
      - save_cache:
          key: go-mod-v1-{{ checksum "go.sum" }}
          paths:
            - /home/circleci/go/pkg/mod
      - run:
          name: Biên dịch binary — Compile Binary
          command: go build -o bin/app ./cmd/app
      - persist_to_workspace:
          root: .
          paths:
            - bin/

  test-unit:
    docker:
      - image: cimg/go:1.21
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: go test ./... -short

  test-integration:
    docker:
      - image: cimg/go:1.21
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: go test ./... -run Integration

workflows:
  build-test:
    jobs:
      - compile
      - test-unit:
          requires: [compile]
      - test-integration:
          requires: [compile]
```

### Ví Dụ 3: Monorepo — Nhiều Service

```yaml
jobs:
  build-frontend:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: cd frontend && npm ci && npm run build
      - persist_to_workspace:
          root: .
          paths:
            - frontend/dist/

  build-backend:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: cd backend && npm ci && npm run build
      - persist_to_workspace:
          root: .
          paths:
            - backend/dist/

  deploy:
    docker:
      - image: cimg/aws-cli:latest
    steps:
      - attach_workspace:
          at: .
      # workspace giờ có cả frontend/dist/ và backend/dist/
      - run: aws s3 sync frontend/dist/ s3://my-app-fe
      - run: aws s3 sync backend/dist/ s3://my-app-be

workflows:
  deploy-all:
    jobs:
      - build-frontend
      - build-backend
      - deploy:
          requires:
            - build-frontend
            - build-backend
```

---

## 🆚 Workspace vs Cache — So Sánh Chi Tiết

| Tiêu Chí | Workspace | Cache |
| -------- | --------- | ----- |
| **Phạm vi thời gian** | Trong một workflow | Xuyên nhiều pipeline (15 ngày) |
| **Mục đích chính** | Truyền artifact giữa jobs | Tái sử dụng dependencies |
| **Nội dung điển hình** | `dist/`, binary, test report | `node_modules`, `~/.m2`, `~/.gradle` |
| **Immutable** | Có thể tích lũy nhiều lần | Immutable (không ghi đè) |
      | **Sau workflow** | Bị xóa | Giữ 15 ngày |
| **Khi nên dùng** | Artifact cần dùng ngay trong workflow | Dependencies ổn định, ít thay đổi |

### Khi Nào Dùng Gì?

```
Workspace — Dùng khi:
  ✅ Build một lần, test/deploy nhiều job
  ✅ Test report cần tổng hợp ở job khác
  ✅ Binary compiled cần deploy
  ✅ Artifact tạm thời trong workflow

Cache — Dùng khi:
  ✅ npm/pip/maven dependencies ít thay đổi
  ✅ Muốn tái sử dụng qua nhiều pipeline
  ✅ Dependencies mất > 1 phút để install
  ✅ Docker layer cache
```

---

## 🔬 Workspace Với Parallelism

Khi dùng `parallelism` (song song hóa), mỗi container chạy một phần test. Sau đó dùng workspace để tổng hợp kết quả:

```yaml
jobs:
  test:
    parallelism: 4
    docker:
      - image: cimg/node:20.0
    steps:
      - attach_workspace:
          at: .
      - run:
          command: |
            circleci tests glob "src/**/*.test.js" | \
            circleci tests split --split-by=timings | \
            xargs npx jest --json --outputFile=test-results-${CIRCLE_NODE_INDEX}.json
      - persist_to_workspace:
          root: .
          paths:
            - test-results-*.json  # Mỗi container lưu kết quả riêng

  report:
    docker:
      - image: cimg/node:20.0
    steps:
      - attach_workspace:
          at: .
      # Nhận tất cả test-results-0.json, test-results-1.json, ...
      - run:
          command: node merge-results.js test-results-*.json

workflows:
  ci:
    jobs:
      - build
      - test:
          requires: [build]
      - report:
          requires: [test]
```

---

## ⚠️ Những Lỗi Thường Gặp — Common Pitfalls

### Lỗi 1: `root` và `at` Không Khớp

```yaml
# ❌ Sai: root là /tmp nhưng at là .
persist_to_workspace:
  root: /tmp/workspace
  paths: [dist/]

attach_workspace:
  at: .    # dist/ sẽ được giải nén tại ./dist/ (từ /tmp/workspace/dist/)
           # ← Đúng! Nhưng cần hiểu rõ đường dẫn kết quả
```

### Lỗi 2: Quên Checkout Trước Attach

```yaml
# ❌ Sai: attach_workspace ghi đè lên directory rỗng
steps:
  - attach_workspace:
      at: .         # Gắn workspace nhưng chưa có source code
  - run: npm test   # Lỗi vì thiếu package.json!

# ✅ Đúng: checkout trước
steps:
  - checkout             # Lấy source code
  - attach_workspace:
      at: .              # Gắn artifact vào thư mục đã có code
  - run: npm test
```

> **Ngoại lệ:** Nếu workspace đã có đủ mọi thứ cần thiết (kể cả source), có thể bỏ `checkout`.

### Lỗi 3: Persist Quá Nhiều — Workspace Quá Lớn

```yaml
# ❌ Persist toàn bộ thư mục — chậm
persist_to_workspace:
  root: .
  paths:
    - .    # Toàn bộ project, bao gồm .git, node_modules, ...

# ✅ Chỉ persist những gì cần thiết
persist_to_workspace:
  root: .
  paths:
    - dist/     # Chỉ artifact cần deploy
```

---

## 📏 Giới Hạn Workspace

| Giới Hạn | Giá Trị |
| -------- | ------- |
| Kích thước tối đa | Không có giới hạn cứng, nhưng >1GB sẽ chậm |
| TTL (Time-To-Live) | Tồn tại cho đến khi workflow kết thúc |
| Số lần persist | Không giới hạn, tích lũy |
| Số job attach | Không giới hạn |

---

## 💬 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Workspace và Cache khác nhau như thế nào?**

A: Cache (bộ nhớ đệm) tồn tại xuyên suốt nhiều pipeline (15 ngày) để tái sử dụng dependencies — dùng cho `node_modules`, `.m2`. Workspace (không gian làm việc) chỉ sống trong một workflow để truyền artifact từ job build sang job test/deploy — dùng cho `dist/`, binary đã compiled.

**Q: Khi nào dùng workspace thay vì checkout + build lại?**

A: Dùng workspace khi: (1) Build tốn nhiều thời gian (> 2 phút); (2) Nhiều job downstream cần cùng artifact; (3) Muốn đảm bảo tất cả job test cùng một binary — tránh race condition nếu code thay đổi giữa chừng.

**Q: Có thể dùng workspace giữa các workflow không?**

A: Không, workspace chỉ tồn tại trong phạm vi một workflow. Để chia sẻ xuyên pipeline, dùng Cache hoặc lưu artifact lên S3/Artifactory/GCS.

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn Thành
