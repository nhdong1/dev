# Vite, ESBuild, Turbopack — Build Tooling Hiện Đại

> Build tool (Công Cụ Build) là phần mềm chuyển đổi source code (TypeScript, JSX, CSS modules, ...) thành JavaScript/CSS mà trình duyệt hiểu được. Vite, ESBuild và Turbopack đại diện cho thế hệ build tools mới — nhanh hơn Webpack hàng chục đến hàng trăm lần

---

## 1. Tại Sao Webpack Không Còn Đủ?

### Vấn Đề Với Webpack

```
Webpack (JavaScript-based bundler):
  Dev server start: 30-60 giây cho large project
  HMR (Hot Module Replacement — Thay Thế Module Nóng): 1-10 giây
  Full build: 60-120 giây

Vì sao chậm?
  1. Bundle toàn bộ app trước khi server start
  2. JavaScript đơn luồng — không tận dụng được CPU đa nhân
  3. Module resolution phức tạp, nhiều plugin
```

---

## 2. Vite — Build Tool Thế Hệ Mới

### Cách Vite Giải Quyết Vấn Đề

Vite có **hai chế độ** hoạt động khác nhau:

**Development Mode — Chế Độ Phát Triển:**

```
Vite dev server không bundle — dùng ESM (ES Modules — Module ES) gốc của trình duyệt:

Browser yêu cầu App.tsx
  → Vite transform NGAY LẬP TỨC (dùng ESBuild — cực nhanh)
  → Gửi ES Module về browser
  → Browser tự resolve imports

Kết quả: Server start < 300ms (không cần bundle toàn bộ trước)
HMR: < 50ms (chỉ transform file thay đổi)
```

**Production Mode — Chế Độ Sản Xuất:**

```
Vite dùng Rollup (JavaScript bundler) để tạo optimized bundle:
  - Tree-shaking — Loại bỏ code không dùng
  - Code splitting tự động
  - Asset optimization
```

### Tạo Project React với Vite

```bash
# Tạo project React + TypeScript
npm create vite@latest my-app -- --template react-ts

# Hoặc chọn template interactive
npm create vite@latest

# Templates có sẵn:
# react, react-ts, react-swc, react-swc-ts
# vue, vanilla, svelte, lit, preact, ...
```

```bash
cd my-app
npm install
npm run dev    # Dev server tại http://localhost:5173
npm run build  # Build production vào dist/
npm run preview # Xem trước production build
```

### Cấu Trúc Project Vite

```
my-react-app/
├── public/            ← Static assets (không transform)
│   └── vite.svg
├── src/
│   ├── main.tsx       ← Entry point
│   ├── App.tsx
│   ├── App.css
│   └── assets/
├── index.html         ← HTML template (entrypoint thực sự)
├── vite.config.ts     ← Cấu hình Vite
├── tsconfig.json
└── package.json
```

### `vite.config.ts` — Cấu Hình Vite

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [
    react(), // Plugin React — JSX transform, Fast Refresh
    // react-swc dùng SWC thay Babel — nhanh hơn
    // reactSWC()
  ],

  // Path alias — Đường dẫn tắt
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@hooks': path.resolve(__dirname, './src/hooks'),
    },
  },

  // Dev server config
  server: {
    port: 3000,
    open: true,   // Tự mở browser
    proxy: {
      // Proxy API calls tránh CORS trong dev
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },

  // Build config
  build: {
    outDir: 'dist',
    sourcemap: true,        // Source maps cho production debugging
    minify: 'esbuild',      // Dùng ESBuild để minify (nhanh hơn Terser)
    rollupOptions: {
      output: {
        // Manual code splitting — Tách code thủ công
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom'],
        },
      },
    },
  },

  // Environment variables — Biến môi trường (phải có prefix VITE_)
  // Dùng: import.meta.env.VITE_API_URL
});
```

### Vite Plugins Phổ Biến

```typescript
import react from '@vitejs/plugin-react';          // Plugin React chính thức
import { visualizer } from 'rollup-plugin-visualizer'; // Phân tích bundle size
import checker from 'vite-plugin-checker';          // TypeScript type checking
import { VitePWA } from 'vite-plugin-pwa';          // PWA support
import svgr from 'vite-plugin-svgr';                // Import SVG như React component

export default defineConfig({
  plugins: [
    react(),
    svgr(),
    checker({ typescript: true }),
    VitePWA({ registerType: 'autoUpdate' }),
    visualizer({ open: true }), // Mở báo cáo bundle sau build
  ],
});
```

### Environment Variables — Biến Môi Trường

```bash
# .env.local
VITE_API_URL=http://localhost:8080
VITE_APP_NAME=My App

# .env.production
VITE_API_URL=https://api.production.com
```

```typescript
// Trong code:
const apiUrl = import.meta.env.VITE_API_URL;
const isProd = import.meta.env.PROD;
const isDev = import.meta.env.DEV;
const mode = import.meta.env.MODE; // "development" | "production"
```

---

## 3. ESBuild — Trình Biên Dịch Siêu Nhanh

ESBuild — Công Cụ Build cực nhanh được viết bằng **Go** (ngôn ngữ lập trình từ Google), không phải JavaScript.

### Tại Sao ESBuild Nhanh?

```
JavaScript (Node.js):
  - Đơn luồng (single-threaded)
  - JIT compiled (Just-In-Time — Biên Dịch Ngay Lúc Chạy)
  - Garbage collection overhead
  - Parse nặng (nếu webpack)

Go (ESBuild):
  - Đa luồng thực sự (parallel processing)
  - AOT compiled (Ahead-of-Time — Biên Dịch Trước)
  - Memory efficient
  - Ít allocations hơn
```

**Benchmark thực tế:**

```
Bundle react + react-dom (production):
  Webpack:  ~2500ms
  Rollup:   ~1500ms
  Parcel:   ~1200ms
  ESBuild:  ~37ms   ← 40-100x nhanh hơn!
```

### ESBuild API

```bash
# CLI — Command Line Interface
esbuild src/index.tsx \
  --bundle \
  --minify \
  --sourcemap \
  --target=es2020 \
  --outfile=dist/bundle.js
```

```javascript
// JavaScript API
import * as esbuild from 'esbuild';

await esbuild.build({
  entryPoints: ['src/index.tsx'],
  bundle: true,
  minify: true,
  sourcemap: true,
  target: ['chrome90', 'firefox88', 'safari14'],
  outdir: 'dist',
  external: ['react', 'react-dom'], // Không bundle dependencies này
  plugins: [],
});
```

### Hạn Chế Của ESBuild

- **Không có TypeScript type checking** — chỉ transpile, không kiểm tra types
- **Không có code splitting phức tạp** như Rollup
- **Plugin API** ít mạnh mẽ hơn Webpack

→ Vì vậy Vite dùng ESBuild để **transform** (dev) và Rollup để **bundle** (production).

---

## 4. Turbopack — Bundler Rust-Based

Turbopack — Trình Đóng Gói dựa trên **Rust** (ngôn ngữ lập trình hiệu năng cao), được Vercel phát triển như "người kế nhiệm" của Webpack.

### Tích Hợp Với Next.js

```bash
# Bật Turbopack trong Next.js dev
next dev --turbopack

# package.json
{
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build"  # Build vẫn dùng Webpack (2026)
  }
}
```

### So Sánh Turbopack vs Vite

| Tiêu Chí | Turbopack | Vite |
| -------- | --------- | ---- |
| **Ngôn ngữ core** | Rust | Go (ESBuild) + JavaScript (Rollup) |
| **HMR speed** | < 10ms (claimed) | < 50ms |
| **Cold start** | Rất nhanh | Nhanh |
| **Hệ sinh thái** | Next.js-only | Framework-agnostic |
| **Maturity** | Đang phát triển | Ổn định, production-ready |
| **Plugin system** | Hạn chế | Vite plugins + Rollup plugins |
| **Phù hợp** | Next.js projects | Mọi framework |

---

## 5. SWC — Trình Biên Dịch JavaScript/TypeScript

SWC (Speedy Web Compiler — Trình Biên Dịch Web Nhanh) viết bằng Rust, thay thế Babel.

```bash
# Plugin Vite dùng SWC (nhanh hơn Babel ~20x)
npm install -D @vitejs/plugin-react-swc

# vite.config.ts
import react from '@vitejs/plugin-react-swc';

export default defineConfig({
  plugins: [react()], // SWC xử lý JSX transform
});
```

**Babel vs SWC:**

| | Babel | SWC |
| - | ----- | --- |
| **Ngôn ngữ** | JavaScript | Rust |
| **Transform speed** | 1x (baseline) | ~20x nhanh hơn |
| **Plugin ecosystem** | Rất lớn | Đang phát triển |
| **Config** | `.babelrc`, `babel.config.js` | `.swcrc` |
| **Dùng bởi** | Legacy projects | Next.js (default), Vite (optional) |

---

## 6. So Sánh Toàn Diện Build Tools

| Tool | Dev Speed | Build Speed | Use Case | Status |
| ---- | --------- | ----------- | -------- | ------ |
| **Vite** | ⚡⚡⚡ | ⚡⚡ | SPA, SSR không Next.js | ✅ Production-ready |
| **Turbopack** | ⚡⚡⚡ | N/A (chưa stable) | Next.js only | 🚧 Đang phát triển |
| **ESBuild** | N/A (low-level) | ⚡⚡⚡ | Library bundling, Vite internals | ✅ Stable |
| **Webpack** | ⚡ | ⚡ | Legacy, complex config | ✅ Stable nhưng chậm |
| **Parcel** | ⚡⚡ | ⚡⚡ | Zero-config projects | ✅ Niche |
| **Rollup** | N/A | ⚡⚡ | Library bundling | ✅ Stable |
| **CRA (Create React App)** | ❌ | ❌ | Không dùng mới | ⚠️ Deprecated |

---

## 7. Câu Hỏi Phỏng Vấn

### Q: Vite nhanh hơn Webpack vì lý do gì trong dev mode?

**A:** Vite không bundle khi khởi động dev server. Thay vào đó, nó dùng **ESM gốc của trình duyệt** (Native Browser ESM) — chỉ transform file được request khi trình duyệt yêu cầu. ESBuild (Go-based) xử lý transform cực nhanh. Kết quả: dev server start < 300ms so với 30-60 giây của Webpack. HMR cũng nhanh hơn vì chỉ invalidate module thay đổi, không rebuild cả bundle.

### Q: Vì sao Vite production build vẫn dùng Rollup thay ESBuild?

**A:** ESBuild không hỗ trợ code splitting phức tạp (dynamic imports, chunking strategy linh hoạt) và thiếu một số tối ưu hóa output mà Rollup làm tốt hơn (tree-shaking tinh vi, chunk naming, ...). Rollup tạo ra bundle sạch và nhỏ hơn cho production. Vite chọn "best of both worlds" — ESBuild để speed trong dev, Rollup cho chất lượng bundle trong production.

### Q: Nên dùng Vite hay Turbopack cho dự án mới?

**A:**
- **Next.js project** → Dùng Turbopack (tích hợp sẵn với `--turbopack`, đang ổn định hóa)
- **React SPA / Remix / khác** → Dùng Vite (mature, ecosystem lớn, plugin phong phú)
- Tránh Webpack cho dự án mới — trừ khi cần plugin đặc thù chưa có trên Vite

---

## ✅ Checklist

- [ ] Tạo project React với Vite và TypeScript
- [ ] Cấu hình path aliases trong `vite.config.ts`
- [ ] Setup proxy API để tránh CORS trong development
- [ ] Cấu hình environment variables với prefix `VITE_`
- [ ] Thêm `rollup-plugin-visualizer` để phân tích bundle size
- [ ] So sánh được Vite vs CRA vs Turbopack khi phỏng vấn

---

**Tài Liệu Tham Khảo:**
- [Vite Docs](https://vitejs.dev/guide/)
- [ESBuild Docs](https://esbuild.github.io/)
- [Turbopack Docs](https://turbo.build/pack)
- [SWC Docs](https://swc.rs/)
- [Why Vite — Evan You](https://vitejs.dev/guide/why.html)
