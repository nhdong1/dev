# Bundle Optimization — Tối Ưu Kích Thước Gói JavaScript

> **Bundle Optimization** (Tối Ưu Gói) là tập hợp các kỹ thuật giảm thiểu lượng JavaScript phải tải về và thực thi, bao gồm: **tree-shaking** (loại bỏ code không dùng), **code splitting** (tách thành nhiều chunk), **lazy loading** (tải lười biếng), và tối ưu **vendor dependencies** (phụ thuộc thư viện bên thứ ba).

---

## 📌 Mục Lục

1. [Tại Sao Bundle Size Quan Trọng?](#1-tại-sao-bundle-size-quan-trọng)
2. [Phân Tích Bundle — Tìm Thủ Phạm](#2-phân-tích-bundle--tìm-thủ-phạm)
3. [Tree-shaking — Loại Bỏ Code Chết](#3-tree-shaking--loại-bỏ-code-chết)
4. [Tối Ưu Dependencies — Phụ Thuộc](#4-tối-ưu-dependencies--phụ-thuộc)
5. [Image Optimization — Tối Ưu Hình Ảnh](#5-image-optimization--tối-ưu-hình-ảnh)
6. [Font Optimization — Tối Ưu Phông Chữ](#6-font-optimization--tối-ưu-phông-chữ)
7. [Compression và Caching](#7-compression-và-caching)
8. [Vite Build Configuration — Cấu Hình Build](#8-vite-build-configuration--cấu-hình-build)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Bundle Size Quan Trọng?

### Tác Động Theo Mạng Và Thiết Bị

```
Bundle 2MB JavaScript:

Mạng 4G (40 Mbps):
  Download: ~400ms
  Parse: ~1,500ms (trên điện thoại tầm trung)
  Execute: ~300ms
  Total blocking: ~2,200ms

Mạng 3G (7.2 Mbps):
  Download: ~2,200ms
  Parse: ~1,500ms
  Total blocking: ~4,000ms → hơn 50% user mobile sẽ thoát!

Bundle 200KB JavaScript (sau tối ưu):
  Download (4G): ~40ms
  Parse: ~150ms
  Total blocking: ~200ms → trải nghiệm tốt
```

### JavaScript Cost — Chi Phí JavaScript

```
JavaScript đắt hơn cùng kích thước image/font vì:
1. Download: giống nhau
2. Parse (Phân tích cú pháp): JavaScript cần parse thành AST
3. Compile (Biên dịch): JIT compilation
4. Execute (Thực thi): Chạy code

Image 200KB → Download → Display → Done
JS 200KB → Download → Parse → Compile → Execute → Interactive
```

### Ngưỡng Mục Tiêu Thực Tế

```
Initial bundle (code tải ngay lần đầu):
  ✅ Xuất sắc: < 100KB (gzip)
  ✅ Tốt: < 200KB (gzip)
  ⚠️ Chấp nhận: < 300KB (gzip)
  ❌ Kém: > 500KB (gzip)

Tổng JavaScript:
  ✅ Tốt: < 500KB (gzip)
  ⚠️ Cần xem xét: 500KB - 1MB (gzip)
  ❌ Vấn đề: > 1MB (gzip)

Quy tắc 170KB của Alex Russell:
  Toàn bộ tài nguyên trang (JS + CSS + Fonts + Images cho above-the-fold)
  nên < 170KB trên 3G để có TTI < 5s
```

---

## 2. Phân Tích Bundle — Tìm Thủ Phạm

### Rollup Plugin Visualizer (Vite)

```bash
npm install --save-dev rollup-plugin-visualizer
```

```javascript
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      open: true, // Tự động mở report trong browser sau build
      gzipSize: true, // Hiển thị kích thước sau gzip
      brotliSize: true, // Hiển thị kích thước sau brotli
      filename: "bundle-stats.html",
      template: "treemap", // "treemap" | "sunburst" | "network"
    }),
  ],
});
```

```bash
npm run build
# → Tự động mở bundle-stats.html
# → Treemap cho thấy thư viện nào chiếm nhiều nhất
```

### Đọc Kết Quả Bundle Analysis

```
Treemap (Sơ Đồ Cây) điển hình của ứng dụng chưa tối ưu:

┌─────────────────────────────────────────────────┐
│                                                 │
│   moment.js (330KB) ← Chỉ dùng formatDate()!  │
│                                                 │
├─────────────────┬───────────────────────────────┤
│ lodash (531KB)  │  chart.js (287KB)             │
│ ← Chỉ dùng     │  ← Chỉ dùng ở trang Analytics│
│   _.get() và   │                               │
│   _.merge()     ├────────────────────────────── │
│                 │  react-icons (190KB)           │
│                 │  ← Dùng 5 icons!              │
├─────────────────┴───────────────────────────────┤
│  Your app code (85KB) ← Phần code thực sự      │
└─────────────────────────────────────────────────┘

→ 85KB code thực, 1.3MB vendor → tỉ lệ 1:15 rất tệ
```

### import-cost Extension (VSCode/Cursor)

```
Extension "Import Cost" hiển thị ngay trong editor:

import moment from "moment"; // 330KB 😱
import { format } from "date-fns"; // 6.8KB ✅
import _ from "lodash"; // 531KB 😱
import { get, merge } from "lodash"; // 531KB (vẫn full bundle!)
import get from "lodash/get"; // 1.2KB ✅
import merge from "lodash/merge"; // 12KB ✅
```

### bundlephobia.com — Kiểm Tra Online

```
Trước khi cài thư viện, kiểm tra tại bundlephobia.com:

Nhập tên package → xem:
- Size (kích thước gốc)
- Gzipped (sau gzip)
- Download time (thời gian tải)
- Similar packages (thay thế nhẹ hơn)

Ví dụ:
moment → 72.1KB gzip (có thể thay bằng date-fns: 13.4KB)
lodash → 24.5KB gzip (có thể dùng lodash-es hoặc native JS)
```

---

## 3. Tree-shaking — Loại Bỏ Code Chết

**Tree-shaking** (Loại Bỏ Code Không Dùng) là quá trình build tool phân tích import/export và loại bỏ code không bao giờ được dùng.

### Điều Kiện Để Tree-shaking Hoạt Động

```javascript
// ✅ ES Modules (ESM) — Tree-shaking hoạt động
// Vì: static import có thể phân tích tại compile time
import { useState, useEffect } from "react"; // Chỉ bundle useState và useEffect

// ❌ CommonJS (CJS) — Tree-shaking KHÔNG hoạt động
const { useState } = require("react"); // Bundle toàn bộ react module

// ✅ Named import từ library hỗ trợ ESM
import { format, parseISO } from "date-fns"; // Chỉ bundle format và parseISO

// ❌ Default import toàn bộ library
import _ from "lodash"; // Bundle toàn bộ lodash (531KB)
import moment from "moment"; // Bundle toàn bộ moment (330KB)
```

### Kiểm Tra Library Có Hỗ Trợ Tree-shaking Không

```json
// package.json của thư viện
{
  "name": "my-library",
  "main": "dist/cjs/index.js", // CommonJS — không tree-shakeable
  "module": "dist/esm/index.js", // ESM — tree-shakeable ✅
  "sideEffects": false // Báo hiệu: không có side effects, an toàn để remove
}
```

### sideEffects — Khai Báo Side Effect

```json
// package.json của ứng dụng của bạn
{
  "sideEffects": [
    "*.css", // CSS files có side effects (inject styles)
    "*.scss",
    "./src/polyfills.js", // Polyfills có side effects
    "./src/analytics.js" // Tracking setup có side effects
  ]
  // Hoặc false: toàn bộ code không có side effects
}
```

```javascript
// File có side effects:
// polyfills.js — import chỉ để chạy code, không export gì
import "core-js/stable"; // Bổ sung polyfills vào global

// Nếu "sideEffects": false → build tool sẽ xóa import này!
// → Cần khai báo file này trong sideEffects array
```

---

## 4. Tối Ưu Dependencies — Phụ Thuộc

### Thay Thế Thư Viện Nặng Bằng Thư Viện Nhẹ

```javascript
// ❌ moment.js — 330KB gzip, không tree-shakeable
import moment from "moment";
const formatted = moment(date).format("DD/MM/YYYY");

// ✅ date-fns — tree-shakeable, chỉ bundle hàm dùng
import { format } from "date-fns";
const formatted = format(date, "dd/MM/yyyy"); // ~6KB

// ✅ Temporal API (tương lai, native)
// Hoặc dùng Intl.DateTimeFormat (native, 0KB)
const formatted = new Intl.DateTimeFormat("vi-VN", {
  day: "2-digit", month: "2-digit", year: "numeric"
}).format(date);
```

```javascript
// ❌ lodash — 531KB gzip
import _ from "lodash";
const result = _.get(obj, "a.b.c", defaultValue);
const merged = _.merge({}, objA, objB);

// ✅ Lodash/es — tree-shakeable
import get from "lodash-es/get";
import merge from "lodash-es/merge";

// ✅ Hoặc native JavaScript (không cần thư viện)
const result = obj?.a?.b?.c ?? defaultValue; // Optional chaining
const merged = { ...objA, ...objB }; // Spread operator (shallow merge)
```

```javascript
// ❌ react-icons — import toàn bộ icon set
import { FaHome, FaUser, FaBell } from "react-icons/fa";
// Dù chỉ dùng 3 icons, vẫn tải một phần lớn

// ✅ @phosphor-icons/react — tree-shakeable tốt hơn
import { House, User, Bell } from "@phosphor-icons/react";

// ✅ Hoặc SVG trực tiếp cho ứng dụng nhỏ
function HomeIcon() {
  return (
    <svg viewBox="0 0 24 24" width="24" height="24">
      <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z" />
    </svg>
  );
}
```

### Bundle Splitting Thư Viện

```javascript
// vite.config.js — Manual chunks cho vendor libraries
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // React core — ít thay đổi → cache lâu dài
          "vendor-react": ["react", "react-dom"],

          // Routing
          "vendor-router": ["react-router-dom"],

          // Data fetching
          "vendor-query": ["@tanstack/react-query"],

          // State management
          "vendor-state": ["zustand", "@reduxjs/toolkit"],

          // UI components — thay đổi theo version
          "vendor-ui": ["@radix-ui/react-dialog", "@radix-ui/react-dropdown-menu"],

          // Heavy libraries — chỉ tải khi cần (nếu không lazy load được)
          "vendor-charts": ["recharts", "d3"],
        }
      }
    }
  }
});
```

### Kiểm Tra Duplicate Dependencies — Phụ Thuộc Trùng Lặp

```bash
# Tìm thư viện bị bundle nhiều lần (version khác nhau)
npx npm-dedupe

# Hoặc dùng bundlephobia để kiểm tra
npx duplicate-package-checker-webpack-plugin

# Với pnpm
pnpm why react # Xem tại sao react được cài

# Ví dụ vấn đề: cả lodash@4.x và lodash@3.x đều được bundle
# → Dùng resolutions/overrides để force một version duy nhất
```

```json
// package.json — Force single version
{
  "resolutions": {
    "lodash": "4.17.21" // npm/yarn
  },
  "pnpm": {
    "overrides": {
      "lodash": "4.17.21" // pnpm
    }
  }
}
```

---

## 5. Image Optimization — Tối Ưu Hình Ảnh

Images thường chiếm 50-70% bandwidth — tối ưu ảnh có thể cải thiện LCP đáng kể.

### Modern Image Formats — Định Dạng Ảnh Hiện Đại

```
JPEG/PNG (cũ):
  JPEG: 100KB, hỗ trợ 100% browser

WebP (2010):
  WebP: 65KB (tiết kiệm 35%), hỗ trợ 95% browser

AVIF (2019):
  AVIF: 45KB (tiết kiệm 55%), hỗ trợ 90% browser (2024)

→ Dùng AVIF với fallback WebP với fallback JPEG
```

```html
<!-- HTML: picture element với multiple sources -->
<picture>
  <source srcset="hero.avif" type="image/avif" />
  <source srcset="hero.webp" type="image/webp" />
  <img src="hero.jpg" alt="Hero image" width="800" height="600" />
</picture>
```

### Native Lazy Loading

```html
<!-- loading="lazy": browser tự tải ảnh khi sắp vào viewport -->
<img
  src="product.webp"
  alt="Sản phẩm"
  loading="lazy"
  width="400"
  height="300"
/>

<!-- loading="eager": tải ngay (mặc định, dùng cho above-the-fold images) -->
<img
  src="hero.webp"
  alt="Hero"
  loading="eager"
  fetchpriority="high"
/>
```

### Next.js Image Component

```jsx
import Image from "next/image";

// next/image tự động:
// - Chuyển đổi sang WebP/AVIF
// - Lazy loading
// - Placeholder blur trong khi tải
// - Responsive srcset
// - Prevent CLS (giữ kích thước trước khi tải)

function ProductCard({ product }) {
  return (
    <div>
      <Image
        src={product.image}
        alt={product.name}
        width={400}
        height={300}
        placeholder="blur" // Hiện blur placeholder
        blurDataURL={product.blurHash} // Tiny base64 image
        priority={product.isFeatured} // Preload ảnh nổi bật
      />
    </div>
  );
}
```

### Responsive Images — Ảnh Đáp Ứng

```html
<!-- srcset và sizes: browser chọn ảnh phù hợp kích thước màn hình -->
<img
  srcset="
    hero-480.webp 480w,
    hero-800.webp 800w,
    hero-1200.webp 1200w
  "
  sizes="
    (max-width: 600px) 480px,
    (max-width: 900px) 800px,
    1200px
  "
  src="hero-1200.webp"
  alt="Hero"
/>
<!-- Người dùng mobile (480px) chỉ tải hero-480.webp (~30KB thay vì 300KB) -->
```

---

## 6. Font Optimization — Tối Ưu Phông Chữ

### Font Loading Strategies — Chiến Lược Tải Font

```css
/* font-display: swap — Dùng system font ngay, thay bằng web font khi sẵn sàng
   → Tránh FOIT (Flash of Invisible Text — Văn Bản Vô Hình Thoáng Qua)
   → Có thể có FOUT (Flash of Unstyled Text — Văn Bản Chưa Được Định Kiểu) */
@font-face {
  font-family: "Inter";
  src: url("/fonts/Inter-Regular.woff2") format("woff2");
  font-weight: 400;
  font-display: swap;
}
```

### Preload Critical Fonts — Tải Trước Font Quan Trọng

```html
<head>
  <!-- Preload font sẽ dùng ngay (above-the-fold text) -->
  <link
    rel="preload"
    href="/fonts/Inter-Regular.woff2"
    as="font"
    type="font/woff2"
    crossorigin
  />
</head>
```

### Google Fonts Optimization

```html
<!-- ❌ Cách cũ — tải toàn bộ -->
<link href="https://fonts.googleapis.com/css2?family=Inter" rel="stylesheet" />

<!-- ✅ Tối ưu: preconnect + subset -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link
  href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap&subset=latin"
  rel="stylesheet"
/>
```

### Variable Fonts — Font Biến Đổi

```css
/* Variable font: 1 file thay vì nhiều file weight */
@font-face {
  font-family: "Inter Variable";
  src: url("/fonts/InterVariable.woff2") format("woff2-variations");
  font-weight: 100 900; /* Hỗ trợ mọi weight từ 100 đến 900 */
  font-display: swap;
}

/* Dùng bất kỳ weight nào */
h1 { font-weight: 750; } /* Không cần file riêng! */
p { font-weight: 380; }
```

---

## 7. Compression và Caching

### Gzip và Brotli

```nginx
# nginx.conf — Bật compression
# Brotli (nén tốt hơn Gzip ~15-25%)
brotli on;
brotli_comp_level 6;
brotli_types text/plain application/javascript text/css application/json;

# Gzip (fallback cho browser cũ)
gzip on;
gzip_comp_level 6;
gzip_types text/plain application/javascript text/css application/json;
```

```bash
# Kiểm tra compression size sau build
du -sh dist/assets/*.js
# Hoặc
ls -la dist/assets/*.js

# Xem kích thước sau gzip
gzip -k dist/assets/index-xxxxx.js
ls -la dist/assets/index-xxxxx.js.gz
```

### Cache Headers — Header Bộ Nhớ Đệm

```
Chiến lược cache:
- Hash-named files (VD: main.a8f3c2b1.js): Cache 1 năm
  → Tên file thay đổi khi nội dung thay đổi → cache luôn hợp lệ
- index.html: Không cache (hoặc max 1 giờ)
  → Cần trình duyệt tải phiên bản mới nhất

Cache-Control: public, max-age=31536000, immutable
  → Dùng cho: JS chunks, CSS, ảnh đã hash
  
Cache-Control: no-cache
  → Dùng cho: index.html (để luôn check version mới)
```

```nginx
# nginx.conf
location /assets/ {
  # Files với hash trong tên → cache 1 năm
  expires 1y;
  add_header Cache-Control "public, immutable";
}

location = /index.html {
  # HTML không cache
  add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

---

## 8. Vite Build Configuration — Cấu Hình Build

### Cấu Hình Đầy Đủ Cho Production

```javascript
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { visualizer } from "rollup-plugin-visualizer";
import { compression } from "vite-plugin-compression2";

export default defineConfig(({ mode }) => ({
  plugins: [
    react(),
    // Tạo gzip và brotli files để server phục vụ
    compression({ algorithm: "gzip" }),
    compression({ algorithm: "brotliCompress" }),
    // Bundle analysis (chỉ khi có ANALYZE=true)
    mode === "analyze" && visualizer({ open: true, gzipSize: true }),
  ].filter(Boolean),

  build: {
    // Target browsers — code ít polyfills hơn
    target: "es2020",

    // Cảnh báo nếu chunk > 500KB
    chunkSizeWarningLimit: 500,

    rollupOptions: {
      output: {
        // Tách vendor thành chunks riêng
        manualChunks: (id) => {
          if (id.includes("node_modules")) {
            // React ecosystem — stable, cache lâu
            if (id.includes("react") || id.includes("react-dom")) {
              return "vendor-react";
            }
            // Routing
            if (id.includes("react-router")) {
              return "vendor-router";
            }
            // Các vendor khác — group theo tên package đầu
            const packageName = id
              .toString()
              .split("node_modules/")[1]
              .split("/")[0];
            return `vendor-${packageName}`;
          }
        },
        // Đặt tên file với hash để cache busting
        chunkFileNames: "assets/[name]-[hash].js",
        entryFileNames: "assets/[name]-[hash].js",
        assetFileNames: "assets/[name]-[hash][extname]",
      }
    }
  },

  // Tối ưu dependencies — pre-bundle với esbuild
  optimizeDeps: {
    include: [
      "react",
      "react-dom",
      "@tanstack/react-query",
      "react-router-dom",
    ],
  },
}));
```

### Phân Tích Bundle Size Trong CI

```bash
# package.json scripts
{
  "scripts": {
    "build": "vite build",
    "build:analyze": "ANALYZE=true vite build",
    "size": "npm run build && du -sh dist/assets/*.js | sort -h"
  }
}

# Chạy phân tích
npm run build:analyze
```

---

## 9. Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Tree-shaking là gì và yêu cầu gì để hoạt động?**

A: Tree-shaking là quá trình build tool loại bỏ code JavaScript không bao giờ được dùng (dead code — code chết) khỏi bundle cuối. Yêu cầu:
1. Thư viện phải dùng **ES Modules** (ESM) — vì static import/export có thể phân tích tại compile time. CommonJS (require) không tree-shakeable.
2. Khai báo `"sideEffects": false` trong package.json để build tool biết an toàn khi remove code
3. Dùng **named imports** thay vì default import toàn bộ library: `import { format } from "date-fns"` thay vì `import * as dateFns from "date-fns"`

---

**Q: Làm thế nào để phân tích và cải thiện bundle size?**

A: Quy trình:
1. **Phân tích**: Dùng `rollup-plugin-visualizer` (Vite) hoặc `webpack-bundle-analyzer` để xem treemap
2. **Xác định vấn đề**: Tìm thư viện chiếm nhiều dung lượng, duplicate dependencies
3. **Hành động**:
   - Thay thư viện nặng bằng thư viện nhẹ hơn (moment → date-fns, lodash → lodash-es)
   - Code splitting cho code ít dùng (React.lazy)
   - Tree-shaking: dùng named imports
   - Kiểm tra `bundlephobia.com` trước khi cài thư viện mới

---

**Q: Gzip và Brotli khác nhau như thế nào?**

A:
- **Gzip**: Thuật toán nén cũ hơn, hỗ trợ 100% browser, compress ratio tốt
- **Brotli**: Thuật toán nén mới (Google), hỗ trợ 95%+ browser, compress ratio tốt hơn Gzip 15-25% cho text/JavaScript

Thực hành tốt nhất: Tạo cả hai (`.br` và `.gz`), server phục vụ Brotli nếu browser hỗ trợ, fallback về Gzip.

---

### Câu Hỏi Nâng Cao

**Q: Giải thích chiến lược cache cho static assets trong SPA.**

A: **Cache busting** (Làm Mới Cache) với content hashing:
- Build tool tạo file tên có hash: `main.a8f3c2b1.js`
- Hash thay đổi khi nội dung thay đổi → URL mới → cache miss → tải version mới
- Hash không đổi khi code không đổi → cache hit → tải từ cache
- `Cache-Control: public, max-age=31536000, immutable` cho các file này → cache 1 năm
- `index.html` phải `no-cache` để browser luôn lấy HTML mới nhất (chứa links đến chunk đúng)

Chiến lược tách vendor chunks đặc biệt hiệu quả: React thay đổi ít hơn code của bạn → `vendor-react.js` có thể cache lâu dù code bạn thay đổi thường xuyên.

---

**Q: Khi nào nên tách vendor chunk riêng biệt?**

A: Tách vendor chunk khi:
- Thư viện **stable** — ít thay đổi version (React, React DOM)
- Thư viện **nặng** — nhiều KB (Chart.js, Monaco Editor)
- Thư viện **dùng chung** nhiều routes — không nên tải lại mỗi route

Không nên tách khi:
- Thư viện nhỏ (< 10KB) — overhead của HTTP request không đáng
- Thư viện chỉ dùng ở 1 route — code split theo route là đủ
- Quá nhiều chunks nhỏ — HTTP/2 giảm overhead nhưng vẫn có giới hạn

---

## 🔗 Điều Hướng

- **Trước đó:** [5-profiling.md](./5-profiling.md) — React DevTools Profiler
- **Tiếp theo:** [07-testing/](../07-testing/) — Kiểm thử React
- **Liên quan:** [2-code-splitting.md](./2-code-splitting.md) — React.lazy và dynamic import
- **Chỉ mục:** [INDEX.md](../INDEX.md)
