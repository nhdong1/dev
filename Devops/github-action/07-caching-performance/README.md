# 07. Caching & Performance — Bộ Đệm, Artifacts và Tối Ưu Hiệu Năng

> Tối ưu hóa tốc độ và chi phí GitHub Actions thông qua chiến lược cache thông minh, quản lý artifacts hiệu quả và giảm thiểu thời gian chạy workflow.

---

## 📚 Mục Lục Module

| File | Chủ Đề | Mô Tả |
|---|---|---|
| `1-caching-dependencies.md` | Cache Dependencies (Bộ Đệm Phụ Thuộc) | `actions/cache` cho npm, pip, Maven, Gradle, Go |
| `2-cache-key-strategies.md` | Cache Key Strategies (Chiến Lược Khóa Cache) | Cache key patterns, restore keys, invalidation |
| `3-artifacts.md` | Artifacts (Tệp Đầu Ra) | Upload/download artifacts, retention policy |
| `4-performance-optimization.md` | Performance Optimization (Tối Ưu Hiệu Năng) | Giảm thời gian chạy workflow |
| `5-billing-cost.md` | Billing & Cost (Tính Phí & Chi Phí) | Tính toán và tiết kiệm chi phí |

---

## 🎯 Tại Sao Caching & Performance Quan Trọng?

### Vấn Đề Thực Tế

Một workflow Node.js điển hình không có cache:

```
⏱️ npm install        → 2–4 phút (tải ~500 packages từ internet)
⏱️ Build TypeScript   → 1–2 phút
⏱️ Run tests          → 1–3 phút
────────────────────────────────
⏱️ Tổng              → 4–9 phút mỗi lần push
```

Với cache đúng cách:

```
⚡ npm install (cache hit) → 10–30 giây (khôi phục từ cache)
⚡ Build TypeScript        → 1–2 phút
⚡ Run tests               → 1–3 phút
──────────────────────────────────────
⚡ Tổng                    → 2–5 phút (tiết kiệm ~50%)
```

**Tác động kinh doanh:**
- Giảm thời gian phản hồi cho developer
- Giảm chi phí GitHub Actions minutes (đặc biệt với private repos)
- Tăng tần suất deploy — feedback nhanh hơn

---

## 🗺️ Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Actions Workflow                    │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Job 1: CI  │    │  Job 2: Build│    │ Job 3: Deploy│   │
│  │              │    │              │    │              │   │
│  │ ┌──────────┐ │    │ ┌──────────┐ │    │ ┌──────────┐ │   │
│  │ │  Cache   │ │    │ │  Cache   │ │    │ │Artifacts │ │   │
│  │ │  Restore │ │    │ │  Restore │ │    │ │ Download │ │   │
│  │ └────┬─────┘ │    │ └────┬─────┘ │    │ └──────────┘ │   │
│  │      │       │    │      │       │    │              │   │
│  │ ┌────▼─────┐ │    │ ┌────▼─────┐ │    │              │   │
│  │ │npm install│ │    │ │npm build │ │    │              │   │
│  │ └────┬─────┘ │    │ └────┬─────┘ │    │              │   │
│  │      │       │    │      │       │    │              │   │
│  │ ┌────▼─────┐ │    │ ┌────▼─────┐ │    │              │   │
│  │ │  Cache   │ │    │ │Artifact  │ │    │              │   │
│  │ │   Save   │ │    │ │  Upload  │─┼────┼──────────────┤   │
│  │ └──────────┘ │    │ └──────────┘ │    │              │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                              │
│  ┌───────────────────────┐  ┌──────────────────────────┐    │
│  │     Cache Storage     │  │    Artifact Storage       │    │
│  │  (Key-value, 10GB max)│  │  (Blob storage, 90 ngày) │    │
│  └───────────────────────┘  └──────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚡ So Sánh Cache vs Artifacts

| Tiêu Chí | Cache (Bộ Đệm) | Artifacts (Tệp Đầu Ra) |
|---|---|---|
| **Mục đích** | Tái sử dụng dependencies giữa các runs | Truyền files giữa jobs hoặc lưu trữ kết quả |
| **Phạm vi** | Across workflow runs | Trong một workflow run (hoặc download thủ công) |
| **Thời hạn** | 7 ngày không được dùng | Mặc định 90 ngày (cấu hình được) |
| **Giới hạn** | 10GB per repo | Theo plan (500MB free, 2GB Pro) |
| **Tốc độ** | Nhanh khi cache hit | Nhanh khi truyền giữa jobs |
| **Ví dụ** | `node_modules`, `.m2`, pip cache | Build outputs, test reports, Docker images |

---

## 🚀 Quick Start — Cache Trong 5 Phút

### Cache npm Dependencies

```yaml
name: CI với Cache

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Bước quan trọng — cache node_modules
      - name: Cache npm dependencies
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci
      - run: npm test
```

### Upload Artifact

```yaml
      - name: Build ứng dụng
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7
```

---

## 📊 Tổng Quan Hiệu Quả Cache Theo Ecosystem

| Ecosystem | Thư Mục Cache | Tiết Kiệm Điển Hình |
|---|---|---|
| **npm / Node.js** | `~/.npm` hoặc `node_modules` | 60–80% thời gian install |
| **pip / Python** | `~/.cache/pip` | 50–70% |
| **Maven / Java** | `~/.m2/repository` | 70–85% (maven repo rất lớn) |
| **Gradle / Java** | `~/.gradle/caches` | 65–80% |
| **Go** | `~/go/pkg/mod` | 70–85% |
| **Cargo / Rust** | `~/.cargo/registry` | 80–90% (Rust compile lâu) |
| **Composer / PHP** | `~/.composer/cache` | 50–65% |
| **CocoaPods / iOS** | `~/Library/Caches/CocoaPods` | 60–75% |

---

## 🔑 Các Khái Niệm Cốt Lõi

### Cache Key (Khóa Cache)
Chuỗi định danh duy nhất cho một cache entry. Cache hit (trúng cache) xảy ra khi key khớp chính xác.

```
runner.os-ecosystem-hash(lockfile)
   │        │              │
   │        │              └── Thay đổi khi dependencies thay đổi
   │        └─────────────── npm, pip, gradle, etc.
   └──────────────────────── linux, windows, macos
```

### Restore Keys (Khóa Khôi Phục)
Danh sách keys dự phòng — được dùng khi không tìm thấy exact match, khôi phục cache gần nhất.

### Cache Hit / Miss
- **Hit**: Key khớp → khôi phục cache ngay lập tức
- **Miss**: Không tìm thấy → chạy bước cài đặt → lưu cache mới

### Artifact Retention (Thời Hạn Lưu Trữ)
Thời gian GitHub giữ artifact trước khi tự động xóa. Mặc định 90 ngày, cấu hình được từ 1–400 ngày.

---

## 📖 Điều Hướng Module

- **Bắt đầu:** [1-caching-dependencies.md](./1-caching-dependencies.md) — Cache cơ bản cho từng ngôn ngữ
- **Nâng cao:** [2-cache-key-strategies.md](./2-cache-key-strategies.md) — Chiến lược khóa thông minh
- **Truyền dữ liệu:** [3-artifacts.md](./3-artifacts.md) — Quản lý artifacts
- **Tối ưu toàn diện:** [4-performance-optimization.md](./4-performance-optimization.md) — Giảm thời gian chạy
- **Chi phí:** [5-billing-cost.md](./5-billing-cost.md) — Tính toán và tiết kiệm

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
