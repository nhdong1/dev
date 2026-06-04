# 2. Cache Key Strategies — Chiến Lược Khóa Cache

> Cache key (khóa cache) xác định cache nào sẽ được khôi phục. Chiến lược khóa tốt = cache hit rate cao + tránh dùng cache bị stale (lỗi thời).

---

## 📚 Mục Lục

1. [Cache Key Anatomy — Giải Phẫu Khóa Cache](#cache-key-anatomy)
2. [hashFiles() — Hàm Băm File](#hashfiles)
3. [Restore Keys — Khóa Khôi Phục Dự Phòng](#restore-keys)
4. [Cache Key Patterns — Mẫu Khóa Phổ Biến](#cache-key-patterns)
5. [Cache Invalidation — Làm Mới Cache](#cache-invalidation)
6. [Granular vs Broad Caching — Cache Hẹp vs Rộng](#granular-vs-broad-caching)
7. [Cross-Branch Cache Sharing — Chia Sẻ Cache Giữa Branches](#cross-branch-cache-sharing)
8. [Monorepo Caching — Cache Trong Dự Án Lớn](#monorepo-caching)
9. [Các Lỗi Thường Gặp](#các-lỗi-thường-gặp)

---

## Cache Key Anatomy

### Cấu Trúc Key Chuẩn

```
${{ runner.os }}-${{ ecosystem }}-${{ version }}-${{ hashFiles(lockfile) }}

Ví dụ:
Linux-npm-node20-a3f2b1c8d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9
  │     │    │         │
  │     │    │         └── Hash SHA-256 của package-lock.json (16 ký tự)
  │     │    └─────────── Phiên bản Node.js (ảnh hưởng đến binary compatibility)
  │     └──────────────── Hệ sinh thái (npm, pip, gradle, ...)
  └────────────────────── OS (Linux, Windows, macOS)
```

### Quy Tắc Vàng

```
Key phải thay đổi KHI VÀ CHỈ KHI dependencies thực sự thay đổi
          ▲
          └── Quá nhạy (thay đổi quá thường) → cache miss liên tục → không có lợi
              Quá bền (ít thay đổi)           → dùng cache stale   → bugs tinh vi
```

---

## `hashFiles()`

### Cú Pháp

```yaml
${{ hashFiles('glob-pattern') }}
${{ hashFiles('**/package-lock.json') }}
${{ hashFiles('go.sum', 'go.mod') }}
```

### Cách Hoạt Động

- Tính SHA-256 hash của tất cả files khớp pattern
- Nếu bất kỳ file nào thay đổi → hash khác → cache key khác → cache miss
- Trả về chuỗi hex 64 ký tự (SHA-256)
- Các files được sắp xếp theo tên trước khi hash (đảm bảo determinism)

### Các Pattern Phổ Biến

```yaml
# Một lock file
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}

# Lock file ở bất kỳ thư mục con nào (monorepo)
key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}

# Nhiều files (thứ tự không quan trọng, kết quả là hash tổng hợp)
key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}

# Kết hợp config file và lock file
key: ${{ runner.os }}-pip-${{ hashFiles('pyproject.toml', 'poetry.lock') }}
```

### Lưu Ý Quan Trọng

```yaml
# SAI — hashFiles() không nhận variable, phải là string literal
key: ${{ runner.os }}-npm-${{ hashFiles(env.LOCK_FILE) }}

# ĐÚNG — string literal
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}

# Nếu pattern không khớp file nào → empty string (key sẽ không có hash)
# → mỗi run sẽ là cache miss → nguy hiểm!
```

---

## Restore Keys — Khóa Khôi Phục Dự Phòng

### Cơ Chế Lookup

```yaml
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
restore-keys: |
  ${{ runner.os }}-npm-
  ${{ runner.os }}-
```

**Thứ tự tìm kiếm:**

```
1. Tìm exact match: "Linux-npm-a3f2b1c8..."   → HIT → khôi phục ngay
                                                ↓ MISS
2. Tìm prefix match: "Linux-npm-"              → Dùng cache gần nhất có prefix này
                                                ↓ MISS
3. Tìm prefix match: "Linux-"                 → Dùng cache gần nhất có prefix này
                                                ↓ MISS
4. Không tìm thấy gì → cache miss hoàn toàn
```

### Tại Sao Dùng Restore Keys?

```
Tình huống: developer thêm 1 package mới vào package.json
  → package-lock.json thay đổi → hash mới → key mới → cache miss

Không có restore-keys: npm ci chạy từ đầu (tải toàn bộ packages)

Có restore-keys "Linux-npm-":
  → Khôi phục cache cũ (có ~490 trong 500 packages)
  → npm ci chỉ tải 10 packages mới
  → Nhanh hơn nhiều!
```

### Restore Key Granularity — Mức Độ Chi Tiết

```yaml
# Restore keys theo thứ tự từ cụ thể đến tổng quát
restore-keys: |
  ${{ runner.os }}-node20-npm-           # Cùng OS + Node version + ecosystem
  ${{ runner.os }}-npm-                  # Cùng OS + ecosystem
  ${{ runner.os }}-                      # Chỉ cùng OS (ít hữu ích hơn)
```

---

## Cache Key Patterns — Mẫu Khóa Phổ Biến

### Pattern 1: OS + Ecosystem + Lock File (Cơ Bản)

```yaml
# Phù hợp cho: hầu hết projects
key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
restore-keys: ${{ runner.os }}-npm-
```

### Pattern 2: OS + Version + Ecosystem + Lock File (Nâng Cao)

```yaml
# Phù hợp khi: test nhiều versions, binary compatibility quan trọng
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node-version }}

  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node${{ matrix.node-version }}-npm-${{ hashFiles('package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node${{ matrix.node-version }}-npm-
        ${{ runner.os }}-npm-
```

### Pattern 3: Branch-Scoped Cache (Cache Giới Hạn Theo Branch)

```yaml
# Phù hợp khi: muốn cache riêng cho main và feature branches
key: ${{ runner.os }}-${{ github.ref_name }}-npm-${{ hashFiles('package-lock.json') }}
restore-keys: |
  ${{ runner.os }}-main-npm-      # Fallback về main branch cache
  ${{ runner.os }}-npm-
```

### Pattern 4: Weekly Rotation (Xoay Vòng Hàng Tuần)

```yaml
# Phù hợp khi: muốn force refresh cache mỗi tuần để tránh stale deps
- name: Lấy ngày trong tuần (0-6)
  id: date
  run: echo "week=$(date +%U)" >> $GITHUB_OUTPUT

- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-week${{ steps.date.outputs.week }}-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-week${{ steps.date.outputs.week }}-
      ${{ runner.os }}-npm-
```

### Pattern 5: Composite Key Cho Nhiều Package Files

```yaml
# Phù hợp khi: monorepo với nhiều package.json
key: ${{ runner.os }}-npm-${{ hashFiles(
  'package-lock.json',
  'apps/web/package-lock.json',
  'apps/api/package-lock.json',
  'packages/shared/package-lock.json'
) }}
```

### Pattern 6: Content-Addressed Cache (Cache Theo Nội Dung)

```yaml
# Phù hợp khi: cache kết quả build (không chỉ dependencies)
- name: Hash source files
  id: source-hash
  run: echo "hash=${{ hashFiles('src/**/*.ts', 'tsconfig.json') }}" >> $GITHUB_OUTPUT

- uses: actions/cache@v4
  with:
    path: dist/
    key: ${{ runner.os }}-build-${{ steps.source-hash.outputs.hash }}
    # Không cần restore-keys vì build output phải khớp chính xác source
```

---

## Cache Invalidation — Làm Mới Cache

### Invalidation Tự Động (Tốt Nhất)

Cache tự động invalid khi lock file thay đổi — không cần làm gì thêm:

```yaml
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
# package-lock.json thay đổi → hash khác → key khác → cache miss → cache mới
```

### Manual Invalidation Bằng Cache Buster

```yaml
# Thêm CACHE_VERSION secret hoặc variable, tăng số lên để force refresh
key: ${{ runner.os }}-npm-v${{ vars.CACHE_VERSION }}-${{ hashFiles('package-lock.json') }}
```

**Cách dùng:**
1. Đặt `vars.CACHE_VERSION = 1` trong repository variables
2. Khi cần xóa cache: tăng lên `2` → tất cả key cũ không khớp → cache fresh

### Manual Invalidation Bằng Ngày Tháng

```yaml
- name: Lấy ngày hiện tại
  id: date
  run: echo "date=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT

- uses: actions/cache@v4
  with:
    path: ~/.npm
    # Cache mới mỗi ngày, fallback về ngày trước
    key: ${{ runner.os }}-npm-${{ steps.date.outputs.date }}-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-${{ steps.date.outputs.date }}-
      ${{ runner.os }}-npm-
```

### Xóa Cache Qua GitHub UI / API

```bash
# Xóa cache qua GitHub CLI
gh cache list --repo owner/repo
gh cache delete <cache-id> --repo owner/repo

# Xóa tất cả cache của một branch
gh cache list --repo owner/repo --branch feature/my-feature | \
  awk '{print $1}' | \
  xargs -I{} gh cache delete {} --repo owner/repo
```

---

## Granular vs Broad Caching — Cache Hẹp vs Rộng

### So Sánh

| Tiêu Chí | Granular Cache (Cache Hẹp) | Broad Cache (Cache Rộng) |
|---|---|---|
| **Cache hit rate** | Thấp hơn (key thay đổi thường hơn) | Cao hơn |
| **Tính chính xác** | Cao (luôn có đúng dependencies) | Có thể có stale deps |
| **Độ phức tạp** | Cao hơn | Đơn giản hơn |
| **Phù hợp với** | Production CI/CD nghiêm ngặt | Development / fast iteration |

### Ví Dụ Granular Cache

```yaml
# Cache riêng cho mỗi loại dependency
- name: Cache production deps
  uses: actions/cache@v4
  with:
    path: ~/.npm/prod
    key: ${{ runner.os }}-npm-prod-${{ hashFiles('package-lock.json') }}

- name: Cache dev deps
  uses: actions/cache@v4
  with:
    path: ~/.npm/dev
    key: ${{ runner.os }}-npm-dev-${{ hashFiles('package-lock.json') }}
```

### Ví Dụ Broad Cache (Thực Tế Hơn)

```yaml
# Một cache duy nhất cho tất cả
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: ${{ runner.os }}-npm-
```

---

## Cross-Branch Cache Sharing — Chia Sẻ Cache Giữa Branches

### Cơ Chế Mặc Định

GitHub Actions cho phép PR branches truy cập cache từ:
1. Branch hiện tại
2. Branch base (nhánh đích của PR)
3. Nhánh `main`/`master`

```
feature/login  → có thể dùng cache từ:
  ├── feature/login (chính nó)
  ├── main (nhánh mặc định)
  └── không thể dùng cache từ feature/payment
```

### Tối Ưu Cho PR Workflows

```yaml
# Trên main: lưu cache "warm" để PR dùng
name: Warm Cache (Khởi Động Cache)
on:
  push:
    branches: [main]

jobs:
  warm-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-main-${{ hashFiles('package-lock.json') }}
      - run: npm ci
```

```yaml
# Trên PR: dùng cache từ main nếu không có cache riêng
name: CI
on: pull_request

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ github.sha }}-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-main-      # Dùng cache từ main branch
            ${{ runner.os }}-npm-
      - run: npm ci
```

---

## Monorepo Caching — Cache Trong Dự Án Lớn

### Strategy 1: Cache Theo Workspace

```yaml
strategy:
  matrix:
    app: [frontend, backend, shared]

steps:
  - uses: actions/cache@v4
    with:
      path: apps/${{ matrix.app }}/node_modules
      key: ${{ runner.os }}-${{ matrix.app }}-${{ hashFiles(format('apps/{0}/package-lock.json', matrix.app)) }}
      restore-keys: |
        ${{ runner.os }}-${{ matrix.app }}-
```

### Strategy 2: Cache Toàn Bộ node_modules (Turborepo / Nx)

```yaml
# Nx / Turborepo có remote caching riêng, nhưng vẫn cần local cache
- uses: actions/cache@v4
  with:
    path: |
      node_modules
      apps/*/node_modules
      packages/*/node_modules
    key: ${{ runner.os }}-monorepo-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-monorepo-
```

### Strategy 3: Phân Tách Cache Cho Từng Layer

```yaml
# Layer 1: Root dependencies (ít thay đổi nhất)
- uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-root-${{ hashFiles('package-lock.json') }}

# Layer 2: Shared package (thay đổi trung bình)
- uses: actions/cache@v4
  with:
    path: packages/shared/node_modules
    key: ${{ runner.os }}-shared-${{ hashFiles('packages/shared/package-lock.json') }}

# Layer 3: App-specific (thay đổi thường xuyên nhất)
- uses: actions/cache@v4
  with:
    path: apps/frontend/node_modules
    key: ${{ runner.os }}-frontend-${{ hashFiles('apps/frontend/package-lock.json') }}
```

---

## Các Lỗi Thường Gặp

### Lỗi 1: Cache Không Bao Giờ Hit

```yaml
# SAI — key quá cụ thể, thay đổi mỗi run
key: ${{ runner.os }}-npm-${{ github.sha }}-${{ github.run_id }}

# ĐÚNG — key thay đổi theo lock file
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
```

### Lỗi 2: Dùng node_modules Thay Vì npm Store

```yaml
# KHÔNG NÊN — node_modules rất lớn, upload/download chậm
path: node_modules

# NÊN — npm store nhỏ hơn, portable hơn
path: ~/.npm
```

### Lỗi 3: Cache Toàn Bộ Project

```yaml
# TUYỆT ĐỐI KHÔNG — cache cả thư mục project bao gồm .git
path: .

# ĐÚNG — chỉ cache thư mục dependencies cụ thể
path: ~/.npm
```

### Lỗi 4: Không Có Restore Keys → Cache Miss Khi Lock File Thay Đổi

```yaml
# Thiếu restore-keys → mỗi khi thêm package, phải npm ci từ đầu
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
# restore-keys bị bỏ qua!

# ĐÚNG — có restore keys dự phòng
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
restore-keys: ${{ runner.os }}-npm-
```

### Lỗi 5: Cache Secrets hoặc Credentials

```yaml
# NGUY HIỂM — .env có thể chứa secrets
path: |
  ~/.npm
  .env              # ← KHÔNG BAO GIỜ cache file này

# AN TOÀN — chỉ cache dependency store
path: ~/.npm
```

---

## 📊 Tóm Tắt Best Practices

| Nguyên Tắc | Mô Tả |
|---|---|
| Dùng `hashFiles()` với lock file | Đảm bảo cache invalid khi deps thay đổi |
| Luôn có `restore-keys` | Tận dụng cache một phần khi lock file đổi |
| Cache store, không phải `node_modules` | `~/.npm` nhỏ và portable hơn `node_modules` |
| Thêm OS vào key | Tránh dùng Linux cache trên macOS |
| Thêm ecosystem version nếu cần | `node20` vs `node18` có thể khác nhau |
| Không cache credentials | Nguy cơ bảo mật |
| Kiểm tra cache hit rate | Dưới 50% → xem lại chiến lược |

---

## 🔗 Liên Kết Tiếp Theo

- [3-artifacts.md](./3-artifacts.md) — Quản lý artifacts
- [4-performance-optimization.md](./4-performance-optimization.md) — Tối ưu hiệu năng

---

**Cập Nhật Lần Cuối:** 2026-05-12
