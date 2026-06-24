# NPM Ecosystem — Quản Lý Package và Dependency

> NPM (Node Package Manager — Trình Quản Lý Gói Node) là registry và công cụ quản lý dependency lớn nhất trong hệ sinh thái JavaScript. Mọi dự án Node.js Backend đều dùng NPM hoặc alternative (Yarn, pnpm).

## Mục Lục

1. [NPM Là Gì?](#npm-là-gì)
2. [package.json](#packagejson)
3. [Cài Đặt và Quản Lý Dependency](#cài-đặt-và-quản-lý-dependency)
4. [SemVer — Semantic Versioning](#semver--semantic-versioning)
5. [Lockfile](#lockfile)
6. [NPM Scripts](#npm-scripts)
7. [npx — Package Runner](#npx--package-runner)
8. [Workspaces và Monorepo](#workspaces-và-monorepo)
9. [Security và Audit](#security-và-audit)
10. [Alternative Package Managers](#alternative-package-managers)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## NPM Là Gì?

NPM gồm ba thành phần:

| Thành Phần | Mô Tả |
| ---------- | ----- |
| **Registry** | Database online chứa ~2 triệu packages (npmjs.com) |
| **CLI** | Command-line tool: `npm install`, `npm run`, `npm publish` |
| **Website** | Tìm kiếm, documentation packages |

```bash
# Khởi tạo project
npm init -y          # Tạo package.json với defaults

# Cài package
npm install express  # Production dependency
npm install -D jest  # Dev dependency (chỉ development/test)

# Chạy script
npm run dev
npm test             # alias cho npm run test
```

---

## package.json

`package.json` là manifest (bản khai báo) của project — metadata, dependencies, scripts.

```json
{
  "name": "my-api",
  "version": "1.0.0",
  "description": "REST API for user management",
  "main": "dist/index.js",
  "type": "module",
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  },
  "scripts": {
    "dev": "node --watch src/index.js",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "jest",
    "lint": "eslint src/",
    "audit:fix": "npm audit fix"
  },
  "dependencies": {
    "express": "^4.21.0",
    "pg": "^8.13.0",
    "dotenv": "^16.4.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "@types/node": "^22.0.0",
    "jest": "^29.7.0",
    "eslint": "^9.0.0"
  },
  "keywords": ["api", "nodejs", "express"],
  "author": "Your Name",
  "license": "MIT"
}
```

### Các Field Quan Trọng

| Field | Mục Đích |
| ----- | -------- |
| `name` | Tên package (unique trên registry nếu publish) |
| `version` | Phiên bản theo SemVer |
| `main` | Entry point khi `require('package-name')` |
| `type` | `"module"` cho ESM, mặc định CommonJS |
| `exports` | Control public API của package |
| `engines` | Giới hạn phiên bản Node.js/npm |
| `dependencies` | Packages cần khi chạy production |
| `devDependencies` | Packages chỉ cần khi develop/test |
| `peerDependencies` | Packages consumer phải tự cài (plugins) |
| `optionalDependencies` | Cài nếu được, bỏ qua nếu fail |
| `scripts` | Shortcut commands |
| `private` | `true` — ngăn publish nhầm lên registry |

---

## Cài Đặt và Quản Lý Dependency

```bash
# Production dependency
npm install lodash
npm install lodash@4.17.21      # Exact version
npm install lodash@^4.17.0      # Compatible với 4.x

# Dev dependency
npm install --save-dev typescript
npm install -D @types/express

# Global install (CLI tools)
npm install -g pm2
npm install -g typescript

# Gỡ package
npm uninstall lodash

# Cập nhật
npm update                        # Update trong semver range
npm install lodash@latest         # Latest version
npx npm-check-updates -u          # Update package.json ranges

# Cài từ lockfile (CI/CD)
npm ci                            # Clean install — nhanh, deterministic
```

### dependencies vs devDependencies

```
dependencies:     express, pg, jsonwebtoken     → cần khi chạy app
devDependencies:  jest, typescript, eslint      → chỉ cần khi develop/build/test
```

**Production install:** `npm install --omit=dev` hoặc `NODE_ENV=production npm ci`

---

## SemVer — Semantic Versioning

SemVer (Semantic Versioning — Phiên Bản Ngữ Nghĩa) format: `MAJOR.MINOR.PATCH`

```
4.18.2
│  │  └── PATCH: Bug fixes, backward compatible
│  └───── MINOR: New features, backward compatible
└──────── MAJOR: Breaking changes
```

### Version Range Operators

| Operator | Ý Nghĩa | Ví Dụ |
| -------- | ------- | ----- |
| (none) | Exact version | `4.18.2` |
| `^` | Compatible với major (≥, < next major) | `^4.18.0` → `>=4.18.0 <5.0.0` |
| `~` | Compatible với minor | `~4.18.0` → `>=4.18.0 <4.19.0` |
| `>=` | Lớn hơn hoặc bằng | `>=4.0.0` |
| `*` / `latest` | Bất kỳ version | Không khuyến nghị production |

```json
{
  "dependencies": {
    "express": "^4.21.0",    // 4.x.x, không lên 5.x
    "lodash": "~4.17.21",    // 4.17.x only
    "uuid": "9.0.1"          // Pin exact — security-critical
  }
}
```

**Khuyến nghị production:** Pin exact version cho critical dependencies hoặc dùng lockfile + `npm ci`.

---

## Lockfile

### package-lock.json (NPM)

Ghi chính xác version đã cài và toàn bộ dependency tree.

```
package.json:        "express": "^4.21.0"     (range — khoảng version)
package-lock.json:   "express": "4.21.1"      (exact — chính xác)
                     + tất cả sub-dependencies với exact versions
```

| Lệnh | Khi Nào Dùng |
| ---- | ------------ |
| `npm install` | Development — có thể update lockfile |
| `npm ci` | CI/CD — install chính xác từ lockfile, xoá node_modules trước |

**Luôn commit lockfile** vào git — đảm bảo mọi developer và CI cài cùng versions.

### yarn.lock / pnpm-lock.yaml

Tương tự cho Yarn và pnpm — **không mix** lockfiles trong cùng project.

---

## NPM Scripts

Scripts là shortcut cho commands thường dùng — chạy bằng `npm run <name>`.

```json
{
  "scripts": {
    "dev": "node --watch src/index.js",
    "build": "tsc && cp -r src/views dist/",
    "start": "node dist/index.js",
    "test": "jest --coverage",
    "test:watch": "jest --watch",
    "lint": "eslint 'src/**/*.js'",
    "lint:fix": "eslint 'src/**/*.js' --fix",
    "db:migrate": "prisma migrate dev",
    "db:seed": "node scripts/seed.js",
    "prestart": "npm run build",
    "postinstall": "prisma generate"
  }
}
```

### Lifecycle Hooks

```
pre<script>  → chạy TRƯỚC <script>
<script>     → script chính
post<script> → chạy SAU <script>

Ví dụ: npm start → prestart → start → poststart
```

### Truyền Arguments

```bash
npm run test -- --verbose          # Pass args sau --
npm run dev -- --port 4000
```

### Environment Variables Trong Scripts

```json
{
  "scripts": {
    "dev": "NODE_ENV=development node src/index.js"
  }
}
```

Windows cần `cross-env`:

```json
{
  "scripts": {
    "dev": "cross-env NODE_ENV=development node src/index.js"
  }
}
```

---

## npx — Package Runner

`npx` chạy package mà không cần cài global.

```bash
# Chạy package chưa cài (tải tạm)
npx create-express-api my-api

# Chạy local binary từ node_modules/.bin
npx jest
npx tsc --init
npx prisma migrate dev

# Chạy specific version
npx cowsay@1.5.0 "Hello"
```

---

## Workspaces và Monorepo

NPM Workspaces (từ npm 7+) quản lý nhiều packages trong một repository.

```
monorepo/
├── package.json          ← root workspace config
├── packages/
│   ├── api/
│   │   └── package.json
│   ├── shared/
│   │   └── package.json
│   └── worker/
│       └── package.json
└── node_modules/         ← hoisted dependencies
```

```json
// Root package.json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces"
  }
}
```

```bash
# Cài dependency cho specific workspace
npm install lodash --workspace=packages/api

# Chạy script trong workspace
npm run build --workspace=packages/api
```

**Alternatives:** pnpm workspaces, Turborepo, Nx — phổ biến hơn cho monorepo lớn.

---

## Security và Audit

```bash
# Kiểm tra vulnerabilities
npm audit

# Tự động fix (cẩn thận — có thể breaking)
npm audit fix
npm audit fix --force

# Xem outdated packages
npm outdated
```

### Best Practices Bảo Mật

| Practice | Lý Do |
| -------- | ----- |
| Chạy `npm audit` trong CI | Phát hiện CVE sớm |
| Pin critical dependencies | Tránh supply chain attack |
| Dùng `npm ci` thay vì `npm install` trong CI | Deterministic builds |
| Review packages trước khi cài | Kiểm tra downloads, maintainer, last publish |
| Dùng `.npmrc` `ignore-scripts` khi cần | Tránh malicious postinstall scripts |
| Lockfile trong git | Reproducible installs |

```ini
# .npmrc
engine-strict=true
save-exact=true
```

---

## Alternative Package Managers

| Tool | Đặc Điểm | Khi Nào Dùng |
| ---- | -------- | ------------ |
| **npm** | Mặc định, đi kèm Node.js | Mọi project, đơn giản |
| **Yarn** | Plug'n'Play, workspaces | Legacy projects, Berry features |
| **pnpm** | Content-addressable store, tiết kiệm disk | Monorepo, nhiều projects |
| **Bun** | Runtime + package manager, nhanh | Thử nghiệm, performance-critical |

```bash
# pnpm
pnpm install
pnpm add express
pnpm run dev

# Bun
bun install
bun run dev
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: `npm install` vs `npm ci`?

**Gợi ý trả lời:** `npm install` đọc package.json, có thể update lockfile, giữ node_modules hiện có. `npm ci` (clean install) xoá node_modules, install chính xác từ lockfile, nhanh hơn và deterministic. Dùng `npm ci` trong CI/CD pipeline.

### Câu 2: `^4.18.0` nghĩa là gì?

**Gợi ý trả lời:** Caret range — chấp nhận `>=4.18.0 <5.0.0`. Cho phép minor và patch updates nhưng không lên major version (có thể breaking changes). Là default khi `npm install package`.

### Câu 3: peerDependencies là gì?

**Gợi ý trả lời:** Dependencies mà package **yêu cầu consumer tự cài**. Ví dụ: plugin React cần React — khai báo `peerDependencies: { "react": "^18.0.0" }`. Tránh duplicate React instances. NPM 7+ tự cài peer deps.

### Câu 4: Tại sao không nên commit node_modules?

**Gợi ý trả lời:** node_modules rất lớn (hàng trăm MB), platform-specific (native addons), và lockfile đã đảm bảo reproducible install. CI/CD chạy `npm ci` để tạo node_modules. Thêm `node_modules/` vào `.gitignore`.

### Câu 5: `dependencies` vs `devDependencies` trong Docker production?

**Gợi ý trả lời:** Production image chỉ cần `dependencies`. Multi-stage Dockerfile: stage 1 cài all deps + build, stage 2 chỉ copy production deps và built artifacts. Dùng `npm ci --omit=dev` hoặc `NODE_ENV=production`.

---

**Xem tiếp:** [5-typescript-basics.md](./5-typescript-basics.md) — static typing cho Node.js Backend.
