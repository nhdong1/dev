# Monorepo — Turborepo, Nx và pnpm Workspaces

> Monorepo (Kho Mã Nguồn Đơn) chứa nhiều packages/projects trong một repository. Phù hợp cho modular monolith, microservices cùng codebase, hoặc shared libraries. Chủ đề cover **pnpm workspaces**, **Turborepo**, và **Nx** cho Node.js/TypeScript teams.

## Mục Lục

1. [Monorepo Là Gì](#monorepo-là-gì)
2. [Monorepo vs Polyrepo](#monorepo-vs-polyrepo)
3. [pnpm Workspaces — Nền Tảng](#pnpm-workspaces--nền-tảng)
4. [Turborepo — Build Orchestration](#turborepo--build-orchestration)
5. [Nx — Enterprise Monorepo](#nx--enterprise-monorepo)
6. [Cấu Trúc Monorepo Node.js](#cấu-trúc-monorepo-nodejs)
7. [Shared Packages Pattern](#shared-packages-pattern)
8. [CI/CD Optimization](#cicd-optimization)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Monorepo Là Gì

```
my-platform/                    ← Single Git repository
├── apps/
│   ├── api-gateway/            ← Express/Fastify app
│   ├── user-service/           ← Microservice
│   ├── order-service/          ← Microservice
│   └── admin-web/              ← Frontend (optional)
├── packages/
│   ├── shared-types/           ← TypeScript interfaces
│   ├── database/               ← Prisma schema + client
│   ├── logger/                 ← Pino wrapper
│   └── eslint-config/          ← Shared lint rules
├── package.json                ← Root workspace config
├── pnpm-workspace.yaml
├── turbo.json
└── tsconfig.base.json
```

**Lợi ích:**

| Lợi Ích | Mô Tả |
| ------- | ----- |
| **Atomic changes** | Một PR update API + shared types + consumer |
| **Code sharing** | Shared packages không cần publish npm |
| **Consistent tooling** | Một ESLint, Prettier, TypeScript config |
| **Refactoring** | Rename across apps — IDE + TypeScript find all |
| **Single CI pipeline** | Test affected packages only |

---

## Monorepo vs Polyrepo

| Tiêu Chí | Monorepo | Polyrepo |
| -------- | -------- | -------- |
| **Repo count** | 1 | N repos (1 per service) |
| **Shared code** | Workspace packages | Publish npm hoặc git submodules |
| **CI complexity** | Cần affected detection | Independent per repo |
| **Access control** | Coarse — all or nothing | Fine-grained per repo |
| **Deploy** | Independent apps từ cùng repo | Natural isolation |
| **Team autonomy** | Cần coordination | High autonomy |

**Chọn Monorepo khi:**
- Team share significant code (types, utils, config)
- Cần atomic cross-service changes
- Modular monolith hoặc microservices cùng org

**Chọn Polyrepo khi:**
- Teams hoàn toàn independent
- Different release cycles, different owners
- Open source components với external consumers

---

## pnpm Workspaces — Nền Tảng

**pnpm (Performant NPM — Trình Quản Lý Gói Hiệu Năng Cao)** workspaces link local packages qua symlinks — tiết kiệm disk, install nhanh.

### Setup

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```json
// package.json (root)
{
  "name": "my-platform",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.4.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

### Internal Dependencies

```json
// apps/user-service/package.json
{
  "name": "@myplatform/user-service",
  "dependencies": {
    "@myplatform/shared-types": "workspace:*",
    "@myplatform/database": "workspace:*",
    "@myplatform/logger": "workspace:*",
    "express": "^4.19.0"
  }
}
```

```bash
# Install — pnpm link workspace packages automatically
pnpm install

# Run script in specific app
pnpm --filter @myplatform/user-service dev

# Run in all apps
pnpm -r run build
```

### Hoisting và Isolation

pnpm dùng **content-addressable store** — mỗi package chỉ access dependencies declared trong `package.json` (strict node_modules).

---

## Turborepo — Build Orchestration

**Turborepo** cache và parallelize tasks across monorepo:

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

| Feature | Mô Tả |
| ------- | ----- |
| **Task pipeline** | `dependsOn: ["^build"]` — build dependencies trước |
| **Remote caching** | Share build cache across CI và dev machines |
| **Parallel execution** | Independent tasks chạy song song |
| **Incremental builds** | Chỉ rebuild packages thay đổi |

```bash
# Build all — respects dependency graph
turbo run build

# Build only affected by git changes
turbo run build --filter=[origin/main]

# Build user-service và dependencies
turbo run build --filter=@myplatform/user-service...
```

### Pipeline Example

```
@myplatform/shared-types  ──build──► dist/
         │
         ▼
@myplatform/database      ──build──► dist/
         │
         ▼
@myplatform/user-service  ──build──► dist/
```

---

## Nx — Enterprise Monorepo

**Nx** cung cấp thêm: project graph visualization, generators, module boundaries enforcement.

```json
// nx.json
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "cache": true
    }
  },
  "namedInputs": {
    "production": ["default", "!{projectRoot}/**/*.spec.ts"]
  }
}
```

### Module Boundaries — Enforce Architecture

```json
// .eslintrc.json — restrict imports between apps
{
  "overrides": [
    {
      "files": ["*.ts"],
      "rules": {
        "@nx/enforce-module-boundaries": [
          "error",
          {
            "allow": [],
            "depConstraints": [
              {
                "sourceTag": "scope:api",
                "onlyDependOnLibsWithTags": ["scope:shared", "scope:database"]
              },
              {
                "sourceTag": "scope:shared",
                "onlyDependOnLibsWithTags": ["scope:shared"]
              }
            ]
          }
        ]
      }
    }
  ]
}
```

### Turborepo vs Nx

| | Turborepo | Nx |
| --- | --------- | -- |
| **Focus** | Fast task runner + caching | Full monorepo platform |
| **Learning curve** | Thấp | Cao hơn |
| **Generators** | Minimal | Rich — scaffold apps/libs |
| **Boundaries** | Manual ESLint | Built-in enforcement |
| **Phù hợp** | Small-medium teams | Enterprise, large monorepos |

---

## Cấu Trúc Monorepo Node.js

### Modular Monolith Layout

```
apps/
└── api/
    └── src/
        ├── modules/
        │   ├── auth/           # Bounded context
        │   ├── orders/
        │   └── catalog/
        └── main.ts

packages/
├── domain/                     # Shared domain types
├── infrastructure/             # DB, cache adapters
└── config/                     # eslint, tsconfig
```

### Microservices Layout

```
apps/
├── api-gateway/
├── user-service/
├── order-service/
└── notification-service/

packages/
├── shared-types/
├── event-schemas/
├── logger/
└── test-utils/
```

---

## Shared Packages Pattern

### Shared Types

```typescript
// packages/shared-types/src/user.ts
export interface UserDTO {
  id: string;
  email: string;
  name: string;
  createdAt: string;
}

export interface ApiResponse<T> {
  data: T;
  meta?: { page: number; total: number };
}
```

### Shared Database Package

```typescript
// packages/database/src/index.ts
export { PrismaClient } from '@prisma/client';
export * from './client';  // Singleton Prisma instance

// packages/database/prisma/schema.prisma
// Single schema hoặc multi-schema per service
```

### Internal Package Versioning

```json
// Luôn dùng workspace protocol
"@myplatform/shared-types": "workspace:*"

// Không cần semver bump cho internal packages
// Breaking change = update all consumers trong cùng PR
```

---

## CI/CD Optimization

### Affected Detection — Chỉ Test/Build Thay Đổi

```yaml
# GitHub Actions example
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: pnpm/action-setup@v4

      - name: Build affected
        run: turbo run build test lint --filter=[origin/main...HEAD]
```

### Remote Cache

```bash
# turbo.json — enable remote cache (Vercel hoặc self-hosted)
turbo run build --remote-cache
```

### Deploy Independent Apps

```yaml
# Deploy only changed services
- name: Detect changed apps
  run: |
    CHANGED=$(turbo run build --filter=[origin/main] --dry-run=json | jq ...)
    echo "apps=$CHANGED" >> $GITHUB_OUTPUT

- name: Deploy user-service
  if: contains(steps.detect.outputs.apps, 'user-service')
  run: kubectl apply -f apps/user-service/k8s/
```

---

## Best Practices

### Do

- **Single TypeScript base config** — `tsconfig.base.json` extend per package
- **Consistent package naming** — `@org/package-name`
- **Document dependency graph** — packages không circular depend
- **Shared eslint/prettier** — `@org/eslint-config`
- **Pin Node.js version** — `.nvmrc` hoặc `engines` field

### Don't

- **Circular dependencies** — A → B → A breaks build graph
- **Apps import from apps** — Chỉ share qua `packages/`
- **God shared package** — Split theo domain, không dump everything vào `shared`
- **Skip affected CI** — Full build mọi PR waste time

### TypeScript Project References

```json
// packages/shared-types/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src"
  }
}

// apps/user-service/tsconfig.json
{
  "references": [
    { "path": "../../packages/shared-types" }
  ]
}
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Gợi Ý Trả Lời |
| ------- | ------------- |
| Monorepo vs polyrepo? | Monorepo: shared code, atomic changes. Polyrepo: team autonomy |
| pnpm vs npm workspaces? | pnpm: strict deps, disk efficient. npm/yarn: simpler, hoisted |
| Turborepo làm gì? | Task orchestration, caching, parallel builds — `dependsOn` graph |
| Module boundaries? | ESLint/Nx rules — apps không import trực tiếp apps |
| `workspace:*` protocol? | pnpm link local packages — always latest trong monorepo |
| CI optimization? | `--filter=[origin/main]` — chỉ build/test affected packages |
| Khi nào monorepo? | Shared types/utils, modular monolith, coordinated releases |

---

**Tiếp theo:** [7-error-handling-architecture.md](./7-error-handling-architecture.md) — Result pattern và global error boundary
