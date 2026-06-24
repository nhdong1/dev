# Prisma ORM — Schema, Migrations và Prisma Client

> Prisma là ORM (Object-Relational Mapping — Ánh Xạ Đối Tượng–Quan Hệ) type-safe, phổ biến nhất trong Node.js/TypeScript projects. Schema-first approach với auto-generated client.

## Mục Lục

1. [Prisma Là Gì](#prisma-là-gì)
2. [Khởi Tạo Project](#khởi-tạo-project)
3. [Schema Definition](#schema-definition)
4. [Migrations (Di Chuyển Schema)](#migrations-di-chuyển-schema)
5. [Prisma Client — CRUD](#prisma-client--crud)
6. [Relations (Quan Hệ)](#relations-quan-hệ)
7. [Raw Queries và Transactions](#raw-queries-và-transactions)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Prisma Là Gì

Prisma gồm 3 thành phần:

| Thành Phần | Vai Trò |
| ---------- | ------- |
| **Prisma Schema** | Định nghĩa models, relations, datasource trong `schema.prisma` |
| **Prisma Migrate** | Version-controlled database migrations |
| **Prisma Client** | Auto-generated, type-safe query builder |

```bash
npm install prisma @prisma/client --save-dev
npx prisma init
```

---

## Khởi Tạo Project

```
prisma/
├── schema.prisma    # Model definitions
└── migrations/      # SQL migration files
src/
└── lib/
    └── prisma.ts    # Singleton client
```

### Singleton Prisma Client

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ||
  new PrismaClient({
    log: process.env.NODE_ENV === 'development'
      ? ['query', 'error', 'warn']
      : ['error'],
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

> Pattern singleton tránh tạo nhiều PrismaClient instances trong development (hot reload).

---

## Schema Definition

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  posts     Post[]
  profile   Profile?
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("users")
}

model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  published Boolean  @default(false)
  authorId  String   @map("author_id")
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  tags      Tag[]
  createdAt DateTime @default(now()) @map("created_at")

  @@index([authorId])
  @@map("posts")
}

model Profile {
  id     String @id @default(cuid())
  bio    String?
  userId String @unique @map("user_id")
  user   User   @relation(fields: [userId], references: [id])

  @@map("profiles")
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]

  @@map("tags")
}

enum Role {
  USER
  ADMIN
}
```

### Scalar Types

| Prisma Type | PostgreSQL | Ghi Chú |
| ----------- | ---------- | ------- |
| `String` | `text`, `varchar` | `@db.VarChar(255)` cho limit |
| `Int` | `integer` | |
| `BigInt` | `bigint` | |
| `Float` | `double precision` | |
| `Decimal` | `decimal` | Tiền tệ — dùng `Decimal` không dùng `Float` |
| `Boolean` | `boolean` | |
| `DateTime` | `timestamp` | |
| `Json` | `jsonb` | |
| `Bytes` | `bytea` | |

---

## Migrations (Di Chuyển Schema)

```bash
# Tạo migration từ schema changes
npx prisma migrate dev --name add_user_role

# Apply migrations trong production (CI/CD)
npx prisma migrate deploy

# Reset DB (chỉ development!)
npx prisma migrate reset

# Generate client sau schema change
npx prisma generate
```

### Migration Workflow

```
1. Sửa schema.prisma
2. npx prisma migrate dev --name descriptive_name
3. Prisma tạo SQL migration file trong prisma/migrations/
4. Apply migration + regenerate Prisma Client
5. Commit cả schema.prisma và migration files
```

---

## Prisma Client — CRUD

```typescript
import { prisma } from './lib/prisma';

// CREATE
const user = await prisma.user.create({
  data: {
    email: 'alice@example.com',
    name: 'Alice',
    posts: {
      create: [{ title: 'First Post', published: true }],
    },
  },
  include: { posts: true },
});

// READ
const users = await prisma.user.findMany({
  where: { role: 'USER' },
  select: { id: true, email: true, name: true },
  orderBy: { createdAt: 'desc' },
  take: 20,
  skip: 0,
});

const userById = await prisma.user.findUnique({
  where: { id: 'clx...' },
});

// UPDATE
await prisma.user.update({
  where: { id: userId },
  data: { name: 'Alice Updated' },
});

// UPDATE MANY
await prisma.post.updateMany({
  where: { authorId: userId, published: false },
  data: { published: true },
});

// DELETE
await prisma.user.delete({ where: { id: userId } });

// UPSERT
await prisma.user.upsert({
  where: { email: 'bob@example.com' },
  update: { name: 'Bob Updated' },
  create: { email: 'bob@example.com', name: 'Bob' },
});
```

### Filtering Operators

```typescript
await prisma.user.findMany({
  where: {
    AND: [
      { email: { contains: '@gmail.com' } },
      { createdAt: { gte: new Date('2024-01-01') } },
    ],
    OR: [
      { role: 'ADMIN' },
      { posts: { some: { published: true } } },
    ],
    NOT: { name: null },
  },
});
```

---

## Relations (Quan Hệ)

### Eager Loading với `include`

```typescript
// Load user kèm posts và profile — 1 query với JOIN
const userWithPosts = await prisma.user.findUnique({
  where: { id: userId },
  include: {
    posts: {
      where: { published: true },
      orderBy: { createdAt: 'desc' },
      take: 10,
    },
    profile: true,
  },
});
```

### Nested Writes

```typescript
await prisma.user.create({
  data: {
    email: 'charlie@example.com',
    profile: { create: { bio: 'Developer' } },
    posts: {
      create: [
        { title: 'Post 1' },
        { title: 'Post 2', tags: { connect: [{ id: tagId }] } },
      ],
    },
  },
});
```

### Relation Types

| Type | Ví Dụ | Prisma Syntax |
| ---- | ----- | ------------- |
| **One-to-One** | User ↔ Profile | `@relation` trên cả hai models |
| **One-to-Many** | User → Posts | `posts Post[]` trên User |
| **Many-to-Many** | Post ↔ Tags | Implicit hoặc explicit join table |

---

## Raw Queries và Transactions

### Raw SQL

```typescript
// Parameterized raw query — an toàn
const users = await prisma.$queryRaw`
  SELECT u.*, COUNT(p.id)::int as post_count
  FROM users u
  LEFT JOIN posts p ON p.author_id = u.id
  WHERE u.created_at > ${since}
  GROUP BY u.id
  ORDER BY post_count DESC
  LIMIT ${limit}
`;

// Execute without return
await prisma.$executeRaw`UPDATE users SET role = 'USER' WHERE role IS NULL`;
```

### Interactive Transactions

```typescript
const result = await prisma.$transaction(async (tx) => {
  const account = await tx.account.update({
    where: { id: fromAccountId },
    data: { balance: { decrement: amount } },
  });

  if (account.balance < 0) {
    throw new Error('Insufficient funds');
  }

  await tx.account.update({
    where: { id: toAccountId },
    data: { balance: { increment: amount } },
  });

  return account;
});
```

### Sequential Transactions

```typescript
const [deletedPosts, deletedUser] = await prisma.$transaction([
  prisma.post.deleteMany({ where: { authorId: userId } }),
  prisma.user.delete({ where: { id: userId } }),
]);
```

---

## Best Practices

1. **Singleton client** — một instance per process
2. **`select` thay `include`** khi chỉ cần subset fields — giảm data transfer
3. **Index** trong schema cho foreign keys và columns thường filter
4. **Migration trong CI** — `prisma migrate deploy` trong deploy pipeline
5. **Không dùng `prisma db push`** trong production — dùng migrations
6. **Connection pooling** — dùng Prisma Accelerate hoặc PgBouncer cho serverless
7. **Soft delete** — thêm `deletedAt DateTime?` thay vì hard delete khi cần audit

```prisma
model User {
  id        String    @id @default(cuid())
  deletedAt DateTime? @map("deleted_at")

  @@index([deletedAt])
}
```

```typescript
// Middleware soft delete
prisma.$use(async (params, next) => {
  if (params.model === 'User' && params.action === 'delete') {
    params.action = 'update';
    params.args.data = { deletedAt: new Date() };
  }
  return next(params);
});
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Prisma generate client khi nào?

**Trả lời:** Sau mỗi thay đổi `schema.prisma` — `npx prisma generate`. `migrate dev` tự chạy generate. Trong CI, thêm `prisma generate` vào build step.

### Câu 2: `findUnique` vs `findFirst`?

**Trả lời:** `findUnique` chỉ dùng với unique fields (`@id`, `@unique`) — DB dùng index lookup. `findFirst` dùng với bất kỳ `where` clause — có thể full scan nếu không có index.

### Câu 3: Prisma hoạt động thế nào với serverless (Lambda)?

**Trả lời:** Mỗi cold start tạo connection mới → connection exhaustion. Giải pháp: Prisma Data Proxy/Accelerate, PgBouncer connection pooler, hoặc `@prisma/adapter-pg` với external pool.

### Câu 4: Làm sao handle N+1 với Prisma?

**Trả lời:** Dùng `include` hoặc `select` với nested relations trong một query. Prisma generate SQL với JOIN thay vì separate queries khi có thể.

---

**Xem tiếp:** [3-typeorm.md](./3-typeorm.md) — ORM dựa trên decorators, phổ biến trong NestJS.
