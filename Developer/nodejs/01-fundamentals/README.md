# Nền Tảng JavaScript & Node.js — Tổng Quan

> Chủ đề nền tảng bắt buộc trước khi đi sâu vào Backend Development với Node.js: ngôn ngữ JavaScript hiện đại, runtime Node.js, hệ thống module, quản lý package, TypeScript, và built-in modules.

## Mục Lục

1. [Tại Sao Cần Học Nền Tảng Trước](#tại-sao-cần-học-nền-tảng-trước)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Lộ Trình Học Trong Chủ Đề](#lộ-trình-học-trong-chủ-đề)
4. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
5. [Bài Tập Thực Hành](#bài-tập-thực-hành)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tại Sao Cần Học Nền Tảng Trước

Node.js không phải là một framework — nó là **JavaScript Runtime (Môi Trường Thực Thi JavaScript)** chạy ngoài trình duyệt. Để viết Backend hiệu quả, bạn cần nắm vững:

| Kỹ Năng | Lý Do Quan Trọng |
| ------- | ---------------- |
| **JavaScript ES6+** | Cú pháp hiện đại là chuẩn trong mọi codebase Node.js |
| **Node.js Runtime** | Hiểu V8, libuv, single-threaded model để debug và tối ưu |
| **Module Systems** | CommonJS vs ESM ảnh hưởng cách import, tree-shaking, bundling |
| **NPM Ecosystem** | Quản lý dependency, SemVer, security audit là kỹ năng hàng ngày |
| **TypeScript** | Chuẩn de facto trong dự án enterprise và NestJS |
| **Built-in Modules** | `fs`, `http`, `crypto` — không cần thư viện ngoài cho nhiều tác vụ cơ bản |

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    NODE.JS APPLICATION                            │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Your Code   │  │  npm packages│  │  Built-in Modules    │  │
│  │  (JS/TS)     │  │  (express,   │  │  fs, http, crypto,   │  │
│  │              │  │   prisma...) │  │  path, events...     │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         └─────────────────┼─────────────────────┘              │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Node.js Runtime (C++ bindings)              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │   │
│  │  │ V8 Engine   │  │    libuv    │  │  Node.js APIs   │ │   │
│  │  │ (JS exec)   │  │ (Event Loop,│  │  (Buffer,       │ │   │
│  │  │             │  │  thread pool)│  │   Stream, etc.) │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         Operating System (Linux / Windows / macOS)        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Lộ Trình Học Trong Chủ Đề

**Thời gian ước tính:** 6–8 giờ

| Thứ Tự | File | Nội Dung | Thời Gian |
| ------ | ---- | -------- | --------- |
| 1 | [1-javascript-es6-plus.md](./1-javascript-es6-plus.md) | let/const, arrow functions, destructuring, spread/rest | 1.5 giờ |
| 2 | [2-nodejs-runtime.md](./2-nodejs-runtime.md) | V8, libuv, process model, global objects | 1.5 giờ |
| 3 | [3-module-systems.md](./3-module-systems.md) | CommonJS vs ESM, dynamic import | 1 giờ |
| 4 | [4-npm-ecosystem.md](./4-npm-ecosystem.md) | package.json, SemVer, scripts, workspaces | 1 giờ |
| 5 | [5-typescript-basics.md](./5-typescript-basics.md) | Type system, interfaces, generics | 1.5 giờ |
| 6 | [6-builtin-modules.md](./6-builtin-modules.md) | fs, path, crypto, http, events, buffer | 1.5 giờ |

**Thứ tự học được khuyến nghị:** 1 → 2 → 3 → 4 → 5 → 6. TypeScript (5) có thể học song song với 3–4 nếu bạn đã quen JavaScript.

---

## Các Tài Liệu Chi Tiết

| File | Nội Dung Chính |
| ---- | -------------- |
| [1-javascript-es6-plus.md](./1-javascript-es6-plus.md) | Cú pháp ES6+ bắt buộc cho Backend: scope, closures, array/object methods |
| [2-nodejs-runtime.md](./2-nodejs-runtime.md) | Cách Node.js chạy JavaScript, process object, environment variables |
| [3-module-systems.md](./3-module-systems.md) | `require` vs `import`, ESM trong Node.js, interoperability |
| [4-npm-ecosystem.md](./4-npm-ecosystem.md) | Quản lý dependency, lockfile, monorepo workspaces |
| [5-typescript-basics.md](./5-typescript-basics.md) | Static typing, type inference, utility types cho API development |
| [6-builtin-modules.md](./6-builtin-modules.md) | Module lõi Node.js — đọc file, HTTP server, mã hoá, events |

---

## Bài Tập Thực Hành

### Lab 1: JavaScript ES6+ (30 phút)

```javascript
// Tạo file lab1.js — refactor code ES5 sang ES6+
// Yêu cầu: dùng destructuring, arrow function, template literal, spread
const users = [
  { id: 1, name: 'Alice', role: 'admin' },
  { id: 2, name: 'Bob', role: 'user' },
];

// Viết hàm filterByRole dùng arrow function + destructuring
// Viết hàm formatUser dùng template literal
```

### Lab 2: Module Systems (20 phút)

```
Tạo 2 file:
- math.cjs (CommonJS): export add, subtract
- greet.mjs (ESM): export greet function
- main.mjs: import từ cả hai, in kết quả
```

### Lab 3: Built-in HTTP Server (30 phút)

```javascript
// Tạo HTTP server đơn giản không dùng Express
// GET /health → { status: 'ok' }
// GET /users → đọc từ users.json bằng fs
```

### Lab 4: TypeScript Mini Project (45 phút)

```bash
mkdir ts-lab && cd ts-lab
npm init -y
npm install typescript @types/node --save-dev
npx tsc --init
# Tạo User interface, ApiResponse<T> generic, implement CRUD types
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Sự khác biệt giữa `var`, `let`, và `const`?

**Gợi ý trả lời:** `var` có function scope và hoisting (đưa khai báo lên đầu scope). `let` và `const` có block scope. `const` không cho phép gán lại biến nhưng object/array vẫn mutable (có thể thay đổi nội dung). Trong Node.js hiện đại, luôn dùng `const` mặc định, `let` khi cần reassign.

### Câu 2: Node.js là multi-threaded hay single-threaded?

**Gợi ý trả lời:** JavaScript code chạy trên **single main thread** qua Event Loop. Tuy nhiên, libuv dùng **thread pool** cho I/O blocking (file system, DNS, crypto). Worker Threads và Cluster module cho phép tận dụng multi-core cho CPU-intensive tasks.

### Câu 3: CommonJS và ESM khác nhau thế nào?

**Gợi ý trả lời:** CommonJS dùng `require()`/`module.exports`, load đồng bộ (synchronous — đồng bộ), là mặc định trong Node.js cũ. ESM dùng `import`/`export`, load bất đồng bộ, hỗ trợ static analysis và tree-shaking. Node.js hỗ trợ cả hai; file `.mjs` hoặc `"type": "module"` trong package.json kích hoạt ESM.

### Câu 4: `package.json` và `package-lock.json` khác nhau thế nào?

**Gợi ý trả lời:** `package.json` khai báo dependency với version range (ví dụ `^4.18.0`). `package-lock.json` ghi chính xác version đã cài và dependency tree — đảm bảo reproducible builds (build tái lập được) giữa các môi trường. Luôn commit lockfile.

### Câu 5: Tại sao dùng TypeScript thay vì JavaScript thuần?

**Gợi ý trả lời:** TypeScript thêm **static typing (kiểu tĩnh)** — bắt lỗi tại compile time thay vì runtime. Cải thiện IDE autocomplete, refactoring an toàn, và documentation tự động qua types. Trade-off: thêm build step và learning curve.

---

**Xem tiếp:** [1-javascript-es6-plus.md](./1-javascript-es6-plus.md) — bắt đầu với cú pháp JavaScript hiện đại.
