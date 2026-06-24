# Module Systems — CommonJS vs ESM

> Hệ thống module quyết định cách tổ chức, import/export code trong Node.js. Hiểu rõ CommonJS và ESM (ECMAScript Modules — Module Chuẩn ECMAScript) là bắt buộc cho mọi Backend developer.

## Mục Lục

1. [Tại Sao Cần Module System](#tại-sao-cần-module-system)
2. [CommonJS (CJS)](#commonjs-cjs)
3. [ES Modules (ESM)](#es-modules-esm)
4. [Kích Hoạt ESM Trong Node.js](#kích-hoạt-esm-trong-nodejs)
5. [So Sánh CommonJS vs ESM](#so-sánh-commonjs-vs-esm)
6. [Interoperability (Tương Thích Chéo)](#interoperability-tương-thích-chéo)
7. [Dynamic Import](#dynamic-import)
8. [Module Resolution (Giải Quyết Module)](#module-resolution-giải-quyết-module)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Module System

Module system giải quyết:

- **Encapsulation (Đóng Gói):** Mỗi file là một module với scope riêng
- **Reusability (Tái Sử Dụng):** Export/import functions, classes, constants
- **Dependency Management:** Khai báo rõ ràng phụ thuộc giữa các file
- **Tree Shaking:** ESM cho phép bundler loại bỏ dead code (code không dùng)

```
project/
├── src/
│   ├── index.js          ← entry point
│   ├── routes/
│   │   └── users.js      ← import service
│   ├── services/
│   │   └── userService.js ← import db
│   └── db/
│       └── connection.js
```

---

## CommonJS (CJS)

CommonJS là module system **mặc định** của Node.js từ đầu, dùng `require()` và `module.exports`.

### Export

```javascript
// math.js
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

// Named export — gán property lên exports
exports.add = add;
exports.subtract = subtract;

// Hoặc module.exports (ghi đè toàn bộ)
module.exports = { add, subtract };

// Export single function/class
module.exports = class Calculator {
  add(a, b) { return a + b; }
};
```

### Import

```javascript
// Destructuring import
const { add, subtract } = require('./math');

// Import toàn bộ module object
const math = require('./math');
math.add(1, 2);

// Built-in module — không cần path
const fs = require('fs');
const path = require('path');

// npm package
const express = require('express');

// require() là synchronous (đồng bộ) — load và execute ngay lập tức
```

### Module Caching

```javascript
// counter.js
let count = 0;
module.exports = {
  increment: () => ++count,
  getCount: () => count,
};

// app.js
const c1 = require('./counter');
const c2 = require('./counter');
c1.increment();
console.log(c2.getCount()); // 1 — cùng instance (singleton cache)
```

Node.js cache module sau lần `require()` đầu tiên — `require.cache` chứa tất cả cached modules.

---

## ES Modules (ESM)

ESM là **chuẩn JavaScript** (`import`/`export`), được browser và Node.js hiện đại hỗ trợ.

### Named Export / Import

```javascript
// userService.js
export const VERSION = '1.0.0';

export function findById(id) {
  return db.query('SELECT * FROM users WHERE id = ?', [id]);
}

export class UserNotFoundError extends Error {
  constructor(id) {
    super(`User ${id} not found`);
    this.name = 'UserNotFoundError';
  }
}
```

```javascript
// routes/users.js
import { findById, UserNotFoundError, VERSION } from '../services/userService.js';
// Lưu ý: ESM yêu cầu file extension .js (trong Node.js)
```

### Default Export / Import

```javascript
// app.js
export default class App {
  constructor() {
    this.name = 'My API';
  }
}

// index.js
import App from './app.js';
// Tên import có thể tùy chọn với default export
import MyApplication from './app.js';
```

### Re-export

```javascript
// index.js — barrel file (file tổng hợp export)
export { findById, createUser } from './userService.js';
export { default as App } from './app.js';
export * from './validators.js';
```

### import.meta

```javascript
// ESM-only: metadata về module hiện tại
console.log(import.meta.url);  // file:///path/to/module.js

import { fileURLToPath } from 'url';
import { dirname } from 'path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

---

## Kích Hoạt ESM Trong Node.js

Có 3 cách kích hoạt ESM:

| Cách | Ví Dụ |
| ---- | ----- |
| File extension `.mjs` | `app.mjs`, `import from './utils.mjs'` |
| `"type": "module"` trong package.json | Tất cả `.js` files là ESM |
| Dynamic `import()` | Hoạt động trong cả CJS và ESM |

```json
// package.json — toàn bộ project dùng ESM
{
  "name": "my-api",
  "type": "module",
  "main": "src/index.js"
}
```

```json
// package.json — CommonJS (mặc định, không cần khai báo)
{
  "name": "my-api",
  "main": "src/index.js"
}
```

**File `.cjs`:** Luôn được xử lý như CommonJS, kể cả khi `"type": "module"`.

---

## So Sánh CommonJS vs ESM

| Khía Cạnh | CommonJS | ESM |
| --------- | -------- | --- |
| **Syntax** | `require()` / `module.exports` | `import` / `export` |
| **Loading** | Synchronous (đồng bộ) | Asynchronous (bất đồng bộ) |
| **Static analysis** | Không — `require()` có thể dynamic | Có — `import` phải ở top level |
| **Tree shaking** | Không hỗ trợ | Hỗ trợ |
| **Top-level await** | Không | Có (ESM only) |
| **Default trong Node.js** | Có (`.js` không có `"type": "module"`) | Cần kích hoạt |
| **`__dirname`** | Có sẵn | Dùng `import.meta.url` |

### Top-Level Await (Chỉ ESM)

```javascript
// database.js (ESM)
import pg from 'pg';

const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });

// Top-level await — chờ kết nối trước khi export
await pool.query('SELECT 1');

export default pool;
```

---

## Interoperability (Tương Thích Chéo)

### ESM Import CommonJS

```javascript
// ESM file import CJS module — default import
import express from 'express';        // module.exports
import { readFile } from 'fs';         // named exports từ CJS

// CJS default export → ESM default import
import pkg from './legacy-module.cjs';
```

### CommonJS Import ESM

```javascript
// KHÔNG thể: const app = require('./esm-module.js'); // Error

// Phải dùng dynamic import (trả về Promise)
async function loadEsm() {
  const { default: app } = await import('./esm-module.js');
  return app;
}

// Hoặc top-level trong async IIFE
(async () => {
  const utils = await import('./utils.mjs');
})();
```

### Dual Package Hazard

Package publish cả CJS và ESM có thể gây duplicate instance. Giải pháp:

```json
{
  "name": "my-lib",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

---

## Dynamic Import

`import()` là function trả về Promise — load module tại runtime.

```javascript
// Conditional loading
async function loadPlugin(name) {
  const plugin = await import(`./plugins/${name}.js`);
  return plugin.default;
}

// Lazy loading — giảm startup time
async function handleUpload(req, res) {
  const multer = await import('multer');
  // ...
}

// Trong CommonJS
async function main() {
  const { createServer } = await import('http');
  const server = createServer(/* ... */);
}
```

**Use cases:** Code splitting, conditional features, plugin systems.

---

## Module Resolution (Giải Quyết Module)

Khi `require('express')` hoặc `import from 'express'`, Node.js tìm module theo thứ tự:

```
1. Core modules (fs, http, path...) — ưu tiên cao nhất
2. Relative path (./utils, ../config)
3. node_modules/ — đi lên cây thư mục
```

```
project/
├── node_modules/
│   └── express/
│       └── package.json  → "main": "index.js"
├── src/
│   └── app.js  → require('express') tìm ở ../node_modules/express
```

### package.json exports field

```json
{
  "name": "my-package",
  "exports": {
    ".": "./dist/index.js",
    "./utils": "./dist/utils.js"
  }
}
```

Chỉ paths trong `exports` mới accessible — encapsulation tốt hơn.

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Dùng ESM cho project mới | Chuẩn JavaScript, tree-shaking, top-level await |
| Dùng `.cjs` cho config cần CJS | `jest.config.cjs`, một số tool chưa hỗ trợ ESM |
| Barrel files (`index.js`) cẩn thận | Có thể gây circular dependency |
| Tránh circular dependency | A imports B, B imports A — refactor hoặc lazy import |
| Luôn dùng file extension trong ESM imports | Node.js ESM yêu cầu `.js` (không `.ts`) |

### Circular Dependency Ví Dụ

```javascript
// a.js
const b = require('./b');
module.exports = { name: 'A', b };

// b.js
const a = require('./a'); // a chưa export xong — partial export
module.exports = { name: 'B', a };
```

**Giải pháp:** Tách shared code ra module thứ ba, hoặc dùng lazy `require()` bên trong function.

---

## Câu Hỏi Phỏng Vấn

### Câu 1: `module.exports` vs `exports`?

**Gợi ý trả lời:** `exports` là reference đến `module.exports`. Gán `exports.foo = bar` hoạt động. Nhưng `exports = { foo: bar }` **không hoạt động** vì reassign local variable, không ảnh hưởng `module.exports`. Dùng `module.exports = ...` khi export single object/function.

### Câu 2: Tại sao ESM hỗ trợ tree-shaking?

**Gợi ý trả lời:** ESM `import`/`export` là static — bundler phân tích tại build time, biết chính xác exports nào được dùng. CommonJS `require()` có thể dynamic (`require(variable)`), bundler không biết trước sẽ load gì.

### Câu 3: Làm sao dùng `__dirname` trong ESM?

**Gợi ý trả lời:**

```javascript
import { fileURLToPath } from 'url';
import { dirname, join } from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const configPath = join(__dirname, 'config.json');
```

### Câu 4: `"type": "module"` ảnh hưởng gì đến Jest/testing?

**Gợi ý trả lời:** Jest mặc định dùng CommonJS. Với ESM project cần cấu hình: `"extensionsToTreatAsEsm"`, `transform`, hoặc dùng Vitest (hỗ trợ ESM native). Đây là lý do nhiều project vẫn giữ CommonJS hoặc dùng TypeScript compile sang CJS.

---

**Xem tiếp:** [4-npm-ecosystem.md](./4-npm-ecosystem.md) — quản lý packages và dependencies.
