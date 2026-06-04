# Caching Strategies — Chiến Lược Bộ Nhớ Đệm

> Caching (bộ nhớ đệm) là kỹ thuật lưu lại kết quả của các bước tốn thời gian (như tải thư viện, compile code) để sử dụng lại trong các lần chạy pipeline tiếp theo, tránh thực hiện lại công việc không cần thiết.

---

## 🧠 Khái Niệm Cốt Lõi

### Cache Là Gì?

```
Lần chạy 1 (Cache Miss — Bộ Đệm Không Có):
  npm install → tải 500 package → 3 phút
  → Lưu node_modules vào cache

Lần chạy 2 (Cache Hit — Bộ Đệm Có Sẵn):
  restore_cache → khôi phục node_modules → 15 giây
  → Tiết kiệm 2 phút 45 giây (92%)
```

### Vòng Đời Cache

```
Pipeline chạy
    │
    ▼
restore_cache ──→ Tìm cache key khớp
    │                   │
    │               Có (Hit) ──→ Giải nén → Dùng ngay
    │               Không (Miss) ──→ Tiếp tục không có cache
    ▼
Thực hiện công việc (install, build...)
    │
    ▼
save_cache ──→ Nén thư mục → Lưu lên CircleCI storage
    │
    ▼
Cache sẵn sàng cho pipeline tiếp theo
```

---

## ⚙️ Cú Pháp Cơ Bản

### `save_cache` — Lưu Bộ Đệm

```yaml
- save_cache:
    key: cache-key-{{ checksum "package-lock.json" }}
    paths:
      - ~/.npm            # Thư mục cần lưu
      - node_modules      # Có thể lưu nhiều thư mục
```

| Tham Số | Mô Tả | Bắt Buộc |
| ------- | ----- | --------- |
| `key` | Tên định danh duy nhất của cache | ✅ |
| `paths` | Danh sách thư mục cần lưu vào cache | ✅ |

### `restore_cache` — Khôi Phục Bộ Đệm

```yaml
- restore_cache:
    keys:
      - cache-key-{{ checksum "package-lock.json" }}  # Khớp chính xác
      - cache-key-                                      # Fallback — Dự Phòng
```

| Tham Số | Mô Tả |
| ------- | ----- |
| `keys` | Danh sách key theo thứ tự ưu tiên, dừng lại khi khớp đầu tiên |

---

## 🔑 Cache Keys — Khóa Bộ Đệm

### Template Variables — Biến Template

CircleCI hỗ trợ các biến đặc biệt trong cache key:

| Biến | Ý Nghĩa | Ví Dụ Giá Trị |
| ---- | ------- | ------------- |
| `{{ checksum "file" }}` | Hash SHA256 của nội dung file | `abc123def456...` |
| `{{ .Branch }}` | Tên nhánh hiện tại | `main`, `feature/auth` |
| `{{ .Revision }}` | Git commit SHA đầy đủ | `a1b2c3d4...` |
| `{{ .BuildNum }}` | Số thứ tự build | `42` |
| `{{ epoch }}` | Unix timestamp hiện tại | `1716000000` |
| `{{ arch }}` | Kiến trúc CPU | `x86_64`, `arm64` |

### Chiến Lược Cache Key Hiệu Quả

**Nguyên tắc:** Cache key càng cụ thể → càng ít stale (cũ); càng chung → càng nhiều hit nhưng có thể lỗi thời.

```yaml
# Tốt nhất: Khớp chính xác → fallback về partial → fallback về bare key
- restore_cache:
    keys:
      # Cấp 1: Chính xác theo nội dung lock file và OS
      - v1-deps-{{ arch }}-{{ checksum "package-lock.json" }}
      # Cấp 2: Chỉ theo nội dung lock file
      - v1-deps-{{ checksum "package-lock.json" }}
      # Cấp 3: Prefix chung — dùng cache gần nhất bất kỳ
      - v1-deps-
```

### Cache Versioning — Đánh Số Phiên Bản Cache

Thêm tiền tố phiên bản `v1-`, `v2-`... để **buộc xóa cache** khi cần:

```yaml
# Trước khi thay đổi Node version hoặc cấu trúc thư mục:
- save_cache:
    key: v1-node-{{ checksum "package-lock.json" }}

# Sau khi muốn cache mới hoàn toàn:
- save_cache:
    key: v2-node-{{ checksum "package-lock.json" }}
```

> **Lý do:** CircleCI không cung cấp nút "xóa cache" trực tiếp — tăng phiên bản là cách duy nhất để invalidate (vô hiệu hóa cache cũ).

---

## 📦 Ví Dụ Theo Từng Ngôn Ngữ / Công Cụ

### Node.js — npm

```yaml
jobs:
  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - node-v1-{{ checksum "package-lock.json" }}
            - node-v1-
      - run:
          name: Cài đặt dependencies — Install Dependencies
          command: npm ci  # Dùng npm ci thay npm install để nhất quán
      - save_cache:
          key: node-v1-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm  # Cache npm global, không cache node_modules
```

> **Tại sao cache `~/.npm` thay vì `node_modules`?**
> `~/.npm` là npm content-addressable cache, nhỏ hơn và an toàn hơn để cache. `node_modules` có thể bị corrupt nếu thay đổi giữa các OS.

### Node.js — yarn

```yaml
      - restore_cache:
          keys:
            - yarn-v1-{{ checksum "yarn.lock" }}
            - yarn-v1-
      - run: yarn install --frozen-lockfile
      - save_cache:
          key: yarn-v1-{{ checksum "yarn.lock" }}
          paths:
            - ~/.yarn/cache    # Yarn berry cache
            - ~/.cache/yarn    # Yarn classic cache
```

### Python — pip

```yaml
      - restore_cache:
          keys:
            - pip-v1-{{ checksum "requirements.txt" }}
            - pip-v1-
      - run:
          name: Cài đặt Python dependencies
          command: |
            python -m venv venv
            . venv/bin/activate
            pip install -r requirements.txt
      - save_cache:
          key: pip-v1-{{ checksum "requirements.txt" }}
          paths:
            - ./venv        # Lưu cả virtualenv
```

### Java — Maven

```yaml
      - restore_cache:
          keys:
            - maven-v1-{{ checksum "pom.xml" }}
            - maven-v1-
      - run: mvn dependency:go-offline -q
      - save_cache:
          key: maven-v1-{{ checksum "pom.xml" }}
          paths:
            - ~/.m2/repository    # Maven local repository
```

### Java — Gradle

```yaml
      - restore_cache:
          keys:
            - gradle-v1-{{ checksum "build.gradle" }}-{{ checksum "gradle/wrapper/gradle-wrapper.properties" }}
            - gradle-v1-{{ checksum "build.gradle" }}
            - gradle-v1-
      - run: ./gradlew dependencies
      - save_cache:
          key: gradle-v1-{{ checksum "build.gradle" }}-{{ checksum "gradle/wrapper/gradle-wrapper.properties" }}
          paths:
            - ~/.gradle/caches
            - ~/.gradle/wrapper
```

### Go — modules

```yaml
      - restore_cache:
          keys:
            - go-mod-v1-{{ checksum "go.sum" }}
            - go-mod-v1-
      - run: go mod download
      - save_cache:
          key: go-mod-v1-{{ checksum "go.sum" }}
          paths:
            - /home/circleci/go/pkg/mod
```

### Ruby — bundler

```yaml
      - restore_cache:
          keys:
            - gems-v1-{{ checksum "Gemfile.lock" }}
            - gems-v1-
      - run: bundle install --path vendor/bundle
      - save_cache:
          key: gems-v1-{{ checksum "Gemfile.lock" }}
          paths:
            - vendor/bundle
```

---

## 🏗️ Cache Nhiều File Lock

Khi project phụ thuộc vào nhiều file:

```yaml
- restore_cache:
    keys:
      # Kết hợp checksum của cả 2 file
      - deps-v1-{{ checksum "package-lock.json" }}-{{ checksum "yarn.lock" }}
      - deps-v1-{{ checksum "package-lock.json" }}
      - deps-v1-
```

---

## ⚠️ Cache Invalidation — Vô Hiệu Hóa Bộ Đệm

### Khi Nào Cache Bị Invalidate?

```
1. Key thay đổi:
   - File lock thay đổi → checksum thay đổi → key mới → cache miss
   - Tăng version prefix (v1 → v2)

2. Cache hết hạn:
   - CircleCI tự động xóa cache sau 15 ngày không dùng

3. Giới hạn storage:
   - Tổ chức vượt quá storage limit → cache cũ bị xóa
```

### Chiến Lược Tránh Stale Cache — Cache Lỗi Thời

```yaml
# ❌ Sai: Key cố định, không bao giờ invalidate
- save_cache:
    key: my-cache
    paths:
      - node_modules

# ✅ Đúng: Key thay đổi theo nội dung file
- save_cache:
    key: my-cache-{{ checksum "package-lock.json" }}
    paths:
      - node_modules
```

---

## 🔄 Cache Trong Workflow Có Nhiều Job

```yaml
version: 2.1

jobs:
  install:
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
      - persist_to_workspace:  # Workspace khác cache: xem file 2-workspace.md
          root: .
          paths:
            - node_modules

  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - attach_workspace:
          at: .
      # Không cần restore_cache vì node_modules đến từ workspace
      - run: npm test

  lint:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run: npm run lint

workflows:
  ci:
    jobs:
      - install
      - test:
          requires:
            - install
      - lint:
          requires:
            - install
```

---

## 📊 Cache vs Workspace — So Sánh Nhanh

| Đặc Điểm | Cache | Workspace |
| -------- | ----- | --------- |
| **Phạm vi** | Xuyên suốt nhiều pipeline | Trong một workflow duy nhất |
| **Mục đích** | Tái sử dụng dependencies | Truyền artifact giữa jobs |
| **TTL** | 15 ngày | Hết khi workflow kết thúc |
| **Ghi đè** | Không (immutable — bất biến) | Không |
| **Tốc độ** | Chậm hơn (nén/giải nén) | Nhanh hơn |

> Xem chi tiết tại [2-workspace.md](./2-workspace.md)

---

## 🛠️ Debug Cache — Gỡ Lỗi Bộ Đệm

### Kiểm Tra Cache Hit/Miss

Trong log của bước `restore_cache`, CircleCI hiển thị:

```bash
# Cache Hit — Bộ Đệm Có Sẵn:
Found a cache for key: node-v1-abc123

# Cache Miss — Bộ Đệm Không Tìm Thấy:
No cache found for key: node-v1-abc123
Searching for any cache matching: node-v1-
Found a cache for key: node-v1-  ← Sử dụng fallback
```

### Tính Checksum Thủ Công

```bash
# Tính SHA256 của file để dự đoán cache key
sha256sum package-lock.json
# Hoặc dùng CircleCI CLI:
circleci local execute --job build
```

### Buộc Cache Mới (Force Invalidate)

```bash
# Tăng version prefix trong config:
# v1-deps- → v2-deps-
# Commit và push → pipeline mới sẽ tạo cache hoàn toàn mới
```

---

## 🏆 Best Practices — Thực Hành Tốt Nhất

### 1. Luôn Dùng `restore_cache` Trước Khi Install

```yaml
# ✅ Đúng thứ tự
steps:
  - checkout
  - restore_cache: ...    # Khôi phục trước
  - run: npm ci           # Install (sẽ nhanh nếu cache hit)
  - save_cache: ...       # Lưu sau

# ❌ Sai thứ tự
steps:
  - checkout
  - run: npm ci           # Luôn chạy đầy đủ, bỏ qua cache
  - restore_cache: ...    # Vô nghĩa
  - save_cache: ...
```

### 2. `save_cache` Đặt Sau `restore_cache` và Install

Nếu cache đã tồn tại với key đó, CircleCI sẽ bỏ qua `save_cache` (không ghi đè). Điều này là bình thường.

### 3. Dùng `~/.npm` Thay Vì `node_modules` Với npm

```yaml
paths:
  - ~/.npm          # ✅ An toàn, nhỏ hơn, platform-independent
  # Không dùng:
  # - node_modules  # ⚠️ Lớn, có thể bị lỗi giữa các môi trường
```

### 4. Nhiều Fallback Keys

```yaml
keys:
  - v1-{{ checksum "package-lock.json" }}   # Chính xác nhất
  - v1-{{ .Branch }}-                        # Theo nhánh
  - v1-                                      # Bất kỳ cache nào
```

### 5. Tách Cache Theo Loại

```yaml
# Cache riêng biệt cho từng loại dependencies
- save_cache:
    key: npm-v1-{{ checksum "package-lock.json" }}
    paths:
      - ~/.npm

- save_cache:
    key: cypress-v1-{{ checksum "package-lock.json" }}
    paths:
      - ~/.cache/Cypress   # Cache riêng cho Cypress binary
```

---

## 💬 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Sự khác biệt giữa `cache` và `workspace` trong CircleCI?**

A: Cache (bộ đệm) tồn tại xuyên suốt nhiều pipeline để tái sử dụng dependencies như `node_modules` — giúp tránh download lại. Workspace (không gian làm việc) chỉ tồn tại trong một workflow để truyền artifact (như file build) từ job này sang job khác trong cùng pipeline.

**Q: Cache key được tạo như thế nào và khi nào nên invalidate?**

A: Cache key dùng `{{ checksum "lockfile" }}` để hash nội dung file. Khi file lock thay đổi (thêm/xóa package), checksum thay đổi → key mới → cache miss → install lại từ đầu. Để buộc invalidate, tăng version prefix từ `v1-` lên `v2-`.

**Q: Tại sao cache của tôi không được dùng dù key trông đúng?**

A: Các nguyên nhân phổ biến: (1) `save_cache` chưa chạy lần nào (pipeline đầu tiên luôn miss); (2) checksum file lock khác do line ending Windows/Unix; (3) cache bị xóa sau 15 ngày; (4) key có template variable bị sai.

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn Thành
