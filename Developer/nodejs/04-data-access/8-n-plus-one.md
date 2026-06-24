# N+1 Problem — Eager Loading, DataLoader và Query Optimization

> N+1 problem là anti-pattern phổ biến nhất gây API chậm trong Node.js applications. Hiểu cách detect, fix, và prevent là kỹ năng phỏng vấn bắt buộc.

## Mục Lục

1. [N+1 Problem Là Gì](#n1-problem-là-gì)
2. [Cách Detect N+1](#cách-detect-n1)
3. [Fix với Eager Loading](#fix-với-eager-loading)
4. [Fix với Batch Query](#fix-với-batch-query)
5. [DataLoader Pattern](#dataloader-pattern)
6. [Query Optimization Techniques](#query-optimization-techniques)
7. [ORM-Specific Solutions](#orm-specific-solutions)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## N+1 Problem Là Gì

**N+1** xảy ra khi code thực hiện **1 query** load N parent records, rồi **N queries** load related data trong loop.

```typescript
// ❌ N+1 — 1 query users + N queries posts
const users = await prisma.user.findMany(); // 1 query

for (const user of users) {
  user.posts = await prisma.post.findMany({
    where: { authorId: user.id },
  }); // N queries (1 per user)
}
// 100 users = 101 queries!
```

```
Timeline:
Query 1:  SELECT * FROM users                    → 100 rows
Query 2:  SELECT * FROM posts WHERE author_id=1  → 5 rows
Query 3:  SELECT * FROM posts WHERE author_id=2  → 3 rows
...
Query 101: SELECT * FROM posts WHERE author_id=100 → 2 rows

Total: 101 round-trips to database
```

### Impact

| Users | Queries | Latency (1ms/query) |
| ----- | ------- | ------------------- |
| 10 | 11 | ~11ms |
| 100 | 101 | ~101ms |
| 1000 | 1001 | ~1 second |

Với network latency thực tế (5–20ms/query), 100 users có thể mất **0.5–2 giây** chỉ cho DB queries.

---

## Cách Detect N+1

### 1. Enable Query Logging

```typescript
// Prisma
const prisma = new PrismaClient({ log: ['query'] });

// TypeORM
{ logging: true }

// Sequelize
{ logging: console.log }
```

### 2. APM Tools

- **Datadog**, **New Relic** — trace DB query count per request
- **Prisma Optimize** — detect N+1 automatically

### 3. Code Review Red Flags

```typescript
// 🚩 Loop + async DB call
for (const item of items) {
  await db.query(...);
}

// 🚩 map + async without batching
await Promise.all(items.map(async (item) => {
  return await db.findRelated(item.id);
}));

// 🚩 GraphQL resolver without DataLoader
posts: async (parent) => {
  return Post.find({ authorId: parent.id }); // N+1 per author
}
```

---

## Fix với Eager Loading

Load parent và children trong **một query** (JOIN):

### Prisma `include`

```typescript
// ✅ 1 query với JOIN
const users = await prisma.user.findMany({
  include: {
    posts: {
      where: { published: true },
      orderBy: { createdAt: 'desc' },
      take: 5,
    },
  },
});
```

### TypeORM `relations`

```typescript
const users = await userRepository.find({
  relations: ['posts'],
});

// Query Builder
const users = await userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .getMany();
```

### Sequelize `include`

```typescript
const users = await User.findAll({
  include: [{ model: Post, as: 'posts' }],
});
```

### Mongoose `populate`

```typescript
const posts = await Post.find().populate('author', 'name email');
```

### Generated SQL (Prisma)

```sql
SELECT u.*, p.*
FROM users u
LEFT JOIN posts p ON p.author_id = u.id
WHERE p.published = true
ORDER BY p.created_at DESC;
```

---

## Fix với Batch Query

Khi eager loading không phù hợp (deep nesting, conditional loading):

```typescript
// ✅ 2 queries thay vì N+1
const users = await prisma.user.findMany();
const userIds = users.map((u) => u.id);

const posts = await prisma.post.findMany({
  where: { authorId: { in: userIds } },
});

// Group posts by authorId
const postsByAuthor = posts.reduce((acc, post) => {
  (acc[post.authorId] ??= []).push(post);
  return acc;
}, {} as Record<string, Post[]>);

const usersWithPosts = users.map((user) => ({
  ...user,
  posts: postsByAuthor[user.id] ?? [],
}));
```

```
Query 1: SELECT * FROM users           → 100 rows
Query 2: SELECT * FROM posts WHERE author_id IN (1,2,...,100) → all posts

Total: 2 queries regardless of user count
```

---

## DataLoader Pattern

Batch và cache requests trong single tick — đặc biệt quan trọng cho **GraphQL**:

```typescript
import DataLoader from 'dataloader';

// Batch function — gọi 1 lần với tất cả keys
const postLoader = new DataLoader(async (authorIds: readonly string[]) => {
  const posts = await prisma.post.findMany({
    where: { authorId: { in: [...authorIds] } },
  });

  // DataLoader yêu cầu return array cùng thứ tự với keys
  const postsByAuthor = authorIds.map((id) =>
    posts.filter((p) => p.authorId === id)
  );

  return postsByAuthor;
});

// Sử dụng — mỗi call được batch
async function getUserWithPosts(userId: string) {
  const user = await prisma.user.findUnique({ where: { id: userId } });
  const posts = await postLoader.load(userId); // Batched!
  return { ...user, posts };
}

// GraphQL resolver
const resolvers = {
  User: {
    posts: (parent) => postLoader.load(parent.id),
  },
};
```

### DataLoader Lifecycle

```
Request starts ──► Create new DataLoader instances
                      │
    Resolver 1 ──► loader.load(userId: 1)  ─┐
    Resolver 2 ──► loader.load(userId: 2)  ─┼──► Batch: IN (1, 2, 3)
    Resolver 3 ──► loader.load(userId: 3)  ─┘
                      │
Request ends ──► DataLoader discarded (cache cleared)
```

> **Quan trọng:** Tạo DataLoader **per request**, không share across requests.

### NestJS + GraphQL

```typescript
@Injectable({ scope: Scope.REQUEST })
export class PostLoader {
  constructor(private prisma: PrismaService) {}

  readonly batchPosts = new DataLoader<string, Post[]>(async (authorIds) => {
    const posts = await this.prisma.post.findMany({
      where: { authorId: { in: [...authorIds] } },
    });
    return authorIds.map((id) => posts.filter((p) => p.authorId === id));
  });
}
```

---

## Query Optimization Techniques

### 1. Select Only Needed Fields

```typescript
// ❌ Load tất cả columns
const users = await prisma.user.findMany({ include: { posts: true } });

// ✅ Chỉ fields cần thiết
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    posts: { select: { id: true, title: true } },
  },
});
```

### 2. Pagination

```typescript
// Cursor-based (khuyến nghị cho large datasets)
const users = await prisma.user.findMany({
  take: 20,
  cursor: lastUserId ? { id: lastUserId } : undefined,
  skip: lastUserId ? 1 : 0,
  orderBy: { id: 'asc' },
});

// Offset-based (đơn giản nhưng chậm với large offset)
const users = await prisma.user.findMany({
  take: 20,
  skip: page * 20,
});
```

### 3. Database Indexes

```prisma
model Post {
  authorId String @map("author_id")
  @@index([authorId])        // Index cho FK queries
  @@index([authorId, published]) // Composite cho filtered queries
}
```

### 4. EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT u.*, p.*
FROM users u
LEFT JOIN posts p ON p.author_id = u.id;

-- Kiểm tra: Seq Scan (bad) vs Index Scan (good)
```

### 5. Denormalization (Khi Cần)

```typescript
// Thêm postCount vào User table — tránh COUNT query
model User {
  postCount Int @default(0)
}

// Update via trigger hoặc application logic
```

---

## ORM-Specific Solutions

| ORM | Eager Load | Batch | N+1 Detection |
| --- | ---------- | ----- | ------------- |
| **Prisma** | `include`, `select` | `$queryRaw` với `IN` | Query logging, Prisma Optimize |
| **TypeORM** | `relations`, `leftJoinAndSelect` | Query Builder `IN` | Query logging |
| **Sequelize** | `include` | `where: { id: { [Op.in]: ids } }` | `benchmark: true` |
| **Mongoose** | `populate` | `$in` query + manual group | `debug: true` |
| **GraphQL** | DataLoader | DataLoader | Apollo Studio traces |

### Prisma Relation Load Strategy

```prisma
// prisma/schema.prisma — Prisma 4.16+
model User {
  posts Post[] @relation(loadStrategy: "join") // JOIN (default)
  // hoặc loadStrategy: "query" — separate optimized queries
}
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Giải thích N+1 problem và cách fix?

**Trả lời:** 1 query load N parents + N queries load children trong loop. Fix: eager loading (`include`/`join`), batch query với `IN` clause, hoặc DataLoader cho GraphQL. Mục tiêu: giảm từ N+1 queries xuống 1–2 queries.

### Câu 2: Eager loading có downside không?

**Trả lời:** Có — Cartesian product khi join nhiều one-to-many relations (user với posts VÀ comments → duplicate user data). Giải pháp: separate queries (`loadStrategy: "query"`), batch loading, hoặc limit nested data.

### Câu 3: DataLoader khác gì eager loading?

**Trả lời:** Eager loading load tất cả upfront — biết trước cần gì. DataLoader batch on-demand trong request — phù hợp GraphQL khi client chọn fields khác nhau. DataLoader cũng cache trong request scope.

### Câu 4: Làm sao đo improvement sau fix N+1?

**Trả lời:** Enable query logging, đếm queries per request trước/sau. Dùng APM trace latency. Benchmark với `autocannon` hoặc `k6`. Target: query count không scale linearly với data size.

### Câu 5: N+1 trong GraphQL nghiêm trọng hơn REST?

**Trả lời:** Có — client tự chọn nested fields, resolver chạy per parent node. REST endpoint thường fixed shape. GraphQL **bắt buộc** DataLoader hoặc query planning (Hasura, Prisma nested writes).

---

**Hoàn thành chủ đề:** Quay lại [README.md](./README.md) hoặc tiếp tục với [05-security/](../05-security/) — JWT authentication và OWASP.
