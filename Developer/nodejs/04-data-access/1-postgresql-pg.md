# PostgreSQL với `pg` Driver — Connection Pool và Parameterized Queries

> Module `pg` (node-postgres) là PostgreSQL client chính thức cho Node.js. Hiểu driver cấp thấp giúp debug ORM issues và viết raw SQL khi cần performance tối đa.

## Mục Lục

1. [`pg` Là Gì](#pg-là-gì)
2. [Cài Đặt và Kết Nối](#cài-đặt-và-kết-nối)
3. [Connection Pool (Bể Kết Nối)](#connection-pool-bể-kết-nối)
4. [Parameterized Queries (Truy Vấn Tham Số Hóa)](#parameterized-queries-truy-vấn-tham-số-hóa)
5. [Transactions Cơ Bản](#transactions-cơ-bản)
6. [Error Handling](#error-handling)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## `pg` Là Gì

`pg` là low-level driver, không phải ORM. Cung cấp:

| Tính Năng | Mô Tả |
| --------- | ----- |
| **Client** | Single connection — phù hợp script, migration one-off |
| **Pool** | Connection pool — production standard |
| **Prepared Statements** | Parameterized queries tự động qua `$1`, `$2` |
| **COPY** | Bulk insert hiệu năng cao |
| **SSL/TLS** | Kết nối encrypted tới managed DB (RDS, Cloud SQL) |

```bash
npm install pg
# TypeScript
npm install pg @types/pg
```

---

## Cài Đặt và Kết Nối

### Single Client

```javascript
const { Client } = require('pg');

const client = new Client({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME || 'myapp',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: true } : false,
});

async function main() {
  await client.connect();
  const result = await client.query('SELECT NOW()');
  console.log(result.rows[0]);
  await client.end();
}

main().catch(console.error);
```

### Connection String

```javascript
const { Client } = require('pg');

const client = new Client({
  connectionString: process.env.DATABASE_URL,
  // postgres://user:password@host:5432/database
});
```

---

## Connection Pool (Bể Kết Nối)

Trong web server, **mỗi request không nên tạo Client mới**. Dùng `Pool` để tái sử dụng connections.

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,              // Số connection tối đa trong pool
  idleTimeoutMillis: 30000,  // Đóng idle connection sau 30s
  connectionTimeoutMillis: 2000, // Timeout khi chờ connection
});

// Express route handler
app.get('/users', async (req, res, next) => {
  try {
    const { rows } = await pool.query('SELECT id, email, name FROM users LIMIT 100');
    res.json(rows);
  } catch (err) {
    next(err);
  }
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await pool.end();
});
```

### Pool Sizing

```
Công thức tham khảo (PostgreSQL):
  pool_size = (num_cores * 2) + effective_spindle_count

Với nhiều app instances:
  pool_per_instance = floor(db_max_connections * 0.8 / num_instances)

Ví dụ: DB max_connections=100, 4 PM2 instances
  → ~20 connections/instance
```

### Pool Events

```javascript
pool.on('connect', () => console.log('New client connected'));
pool.on('error', (err) => {
  console.error('Unexpected pool error', err);
  // Không throw — pool tự reconnect
});
```

---

## Parameterized Queries (Truy Vấn Tham Số Hóa)

**Bắt buộc** để ngăn SQL Injection.

```javascript
// ✅ AN TOÀN — parameterized
const userId = req.params.id;
const result = await pool.query(
  'SELECT * FROM users WHERE id = $1',
  [userId]
);

// ✅ Multiple parameters
await pool.query(
  'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *',
  [email, name]
);

// ❌ NGUY HIỂM — SQL Injection
await pool.query(`SELECT * FROM users WHERE email = '${email}'`);
// Input: '; DROP TABLE users; -- → disaster
```

### Dynamic WHERE Clause

```javascript
async function findUsers(filters) {
  const conditions = [];
  const values = [];
  let paramIndex = 1;

  if (filters.email) {
    conditions.push(`email = $${paramIndex++}`);
    values.push(filters.email);
  }
  if (filters.status) {
    conditions.push(`status = $${paramIndex++}`);
    values.push(filters.status);
  }

  const where = conditions.length ? `WHERE ${conditions.join(' AND ')}` : '';
  const sql = `SELECT * FROM users ${where} ORDER BY created_at DESC`;

  return pool.query(sql, values);
}
```

---

## Transactions Cơ Bản

```javascript
async function transferFunds(fromId, toId, amount) {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    const debit = await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1 RETURNING balance',
      [amount, fromId]
    );
    if (debit.rowCount === 0) {
      throw new Error('Insufficient funds');
    }

    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );

    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release(); // Trả connection về pool
  }
}
```

> **Lưu ý:** Transaction phải dùng **cùng một client** từ pool, không dùng `pool.query()` trực tiếp.

---

## Error Handling

```javascript
const { DatabaseError } = require('pg');

app.use((err, req, res, next) => {
  if (err instanceof DatabaseError) {
    // PostgreSQL error codes: https://www.postgresql.org/docs/current/errcodes-appendix.html
    switch (err.code) {
      case '23505': // unique_violation
        return res.status(409).json({ error: 'Duplicate entry' });
      case '23503': // foreign_key_violation
        return res.status(400).json({ error: 'Referenced record not found' });
      case '23502': // not_null_violation
        return res.status(400).json({ error: 'Required field missing' });
      default:
        console.error('DB error:', err.code, err.detail);
        return res.status(500).json({ error: 'Database error' });
    }
  }
  next(err);
});
```

### Common PostgreSQL Error Codes

| Code | Tên | Ý Nghĩa |
| ---- | --- | ------- |
| `23505` | unique_violation | Duplicate unique key |
| `23503` | foreign_key_violation | FK constraint failed |
| `23502` | not_null_violation | NOT NULL constraint |
| `42P01` | undefined_table | Table không tồn tại |
| `57014` | query_canceled | Query timeout |

---

## Best Practices

1. **Luôn dùng Pool** trong web application, không dùng single Client
2. **Parameterized queries** — không bao giờ string interpolation cho user input
3. **`client.release()`** trong `finally` block khi dùng `pool.connect()`
4. **Connection string từ env** — không hardcode credentials
5. **Graceful shutdown** — `pool.end()` khi process terminate
6. **Query timeout** — set `statement_timeout` cho long-running queries
7. **Index** — `EXPLAIN ANALYZE` để verify query plan

```javascript
// Query với timeout
await pool.query({
  text: 'SELECT * FROM large_table WHERE ...',
  values: [param],
  statement_timeout: 5000, // 5 giây
});
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: `pool.query()` vs `pool.connect()` — khác gì?

**Trả lời:** `pool.query()` tự lấy client, chạy query, release client — tiện cho single query. `pool.connect()` lấy client để giữ qua nhiều queries (transactions) — phải `release()` thủ công.

### Câu 2: Tại sao không tạo connection mới mỗi request?

**Trả lời:** Mỗi TCP connection + PostgreSQL backend process tốn ~5–10MB RAM và ~1–3ms handshake. Với 1000 req/s, tạo connection mới sẽ exhaust DB `max_connections` và tăng latency đáng kể.

### Câu 3: `RETURNING *` trong INSERT có lợi ích gì?

**Trả lời:** Trả về row vừa insert (bao gồm auto-generated `id`, `created_at`) trong một round-trip, tránh phải SELECT lại.

### Câu 4: Khi nào dùng raw `pg` thay vì ORM?

**Trả lời:** Complex reporting queries, bulk operations, performance-critical paths, hoặc khi ORM generate SQL không tối ưu. ORM cho CRUD thông thường; raw SQL cho edge cases.

---

**Xem tiếp:** [2-prisma-orm.md](./2-prisma-orm.md) — ORM type-safe phổ biến nhất trong ecosystem Node.js hiện đại.
