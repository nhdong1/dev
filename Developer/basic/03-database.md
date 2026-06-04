# Database - Interview Questions

## Question 1: Giải thích ACID properties và tại sao chúng quan trọng?

**Answer:**

**ACID** là 4 properties đảm bảo reliability của database transactions:

**1. Atomicity (Tính nguyên tử)**
```
Transaction thực hiện hoàn toàn hoặc không thực hiện gì cả.
"All or nothing"

Example:
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- Cả hai hoặc không có gì
```

**2. Consistency (Tính nhất quán)**
```
Database luôn ở valid state trước và sau transaction.
Tất cả constraints, triggers được đảm bảo.

Example: Total balance trước và sau transfer phải bằng nhau
```

**3. Isolation (Tính độc lập)**
```
Concurrent transactions không ảnh hưởng lẫn nhau.
Mỗi transaction như đang chạy một mình.

Issues khi không có isolation:
- Dirty read: Đọc uncommitted data
- Non-repeatable read: Data thay đổi giữa 2 lần đọc
- Phantom read: Rows mới xuất hiện trong query
```

**4. Durability (Tính bền vững)**
```
Committed transactions tồn tại vĩnh viễn.
Dù system crash, data vẫn được bảo toàn (WAL - Write-Ahead Logging).
```

**ACID vs BASE:**

| ACID | BASE |
|------|------|
| Strong consistency | Eventual consistency |
| Pessimistic | Optimistic |
| Traditional RDBMS | NoSQL, distributed systems |
| Lower availability | Higher availability |

---

## Question 2: So sánh các Transaction Isolation Levels?

**Answer:**

**Isolation Levels (từ thấp → cao):**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|------------|---------------------|--------------|
| Read Uncommitted | ✓ | ✓ | ✓ |
| Read Committed | ✗ | ✓ | ✓ |
| Repeatable Read | ✗ | ✗ | ✓ |
| Serializable | ✗ | ✗ | ✗ |

**1. Read Uncommitted**
```sql
-- Transaction A
UPDATE accounts SET balance = 500 WHERE id = 1;
-- NOT YET COMMITTED

-- Transaction B có thể đọc balance = 500 (dirty read)
SELECT balance FROM accounts WHERE id = 1;
```

**2. Read Committed** (Default trong PostgreSQL, SQL Server)
```sql
-- Chỉ đọc committed data
-- Nhưng nếu đọc 2 lần, có thể khác nhau
```

**3. Repeatable Read** (Default trong MySQL InnoDB)
```sql
-- Transaction B đọc cùng row sẽ thấy cùng giá trị
-- Nhưng có thể thấy phantom rows với INSERT
```

**4. Serializable** (Strict nhất)
```sql
-- Transactions thực hiện như tuần tự
-- Performance impact cao
```

**Implementation Mechanisms:**
- **Locking**: Locks rows/tables
- **MVCC (Multi-Version Concurrency Control)**: Snapshot-based, maintain multiple versions

```sql
-- PostgreSQL: Set isolation level
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    -- operations
COMMIT;
```

**Trade-off:**
- Higher isolation → Lower concurrency → Lower throughput
- Lower isolation → Better performance → Potential anomalies

---

## Question 3: Giải thích các Indexing strategies và khi nào sử dụng?

**Answer:**

**Index** là data structure giúp tăng tốc query bằng cách tránh full table scan.

**Types of Indexes:**

**1. B-Tree Index** (Default, phổ biến nhất)
```sql
CREATE INDEX idx_users_email ON users(email);

-- Good for:
-- Equality: WHERE email = 'a@b.com'
-- Range: WHERE created_at > '2024-01-01'
-- Sorting: ORDER BY created_at
-- Prefix: WHERE name LIKE 'John%'

-- Bad for: LIKE '%keyword%'
```

**2. Hash Index**
```sql
CREATE INDEX idx_hash ON users USING HASH(email);

-- Good for: Equality only (=)
-- Bad for: Range queries
```

**3. Composite (Multi-column) Index**
```sql
CREATE INDEX idx_composite ON orders(user_id, created_at);

-- Leftmost prefix rule:
-- ✓ WHERE user_id = 1
-- ✓ WHERE user_id = 1 AND created_at > '2024-01-01'
-- ✗ WHERE created_at > '2024-01-01'  -- Won't use index
```

**4. Covering Index**
```sql
CREATE INDEX idx_covering ON orders(user_id, status, total);

-- Query uses index only (no table lookup)
SELECT status, total FROM orders WHERE user_id = 1;
```

**5. Partial Index**
```sql
CREATE INDEX idx_active_users ON users(email)
WHERE status = 'active';

-- Smaller index, faster queries on subset
```

**6. Full-Text Index**
```sql
CREATE INDEX idx_fulltext ON articles USING GIN(to_tsvector('english', content));

-- For text search
SELECT * FROM articles WHERE to_tsvector('english', content) @@ 'database';
```

**Index Selection Guidelines:**
```
DO index:
- Primary keys (auto-indexed)
- Foreign keys
- Columns in WHERE, JOIN, ORDER BY
- High cardinality columns

DON'T index:
- Small tables
- Frequently updated columns
- Low cardinality (e.g., boolean)
```

**EXPLAIN ANALYZE:**
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@email.com';
-- Index Scan vs Seq Scan
```

---

## Question 4: NoSQL databases: So sánh MongoDB, Cassandra, DynamoDB?

**Answer:**

**Comparison Table:**

| Aspect | MongoDB | Cassandra | DynamoDB |
|--------|---------|-----------|----------|
| **Type** | Document | Wide-column | Key-value/Document |
| **Query** | Rich queries | Limited (CQL) | Limited (by key) |
| **Consistency** | Strong/Eventual | Tunable | Eventual/Strong |
| **Scaling** | Sharding | Linear horizontal | Managed, auto |
| **Use case** | General purpose | Time-series, IoT | Serverless, AWS |

**MongoDB:**
```javascript
// Document model - flexible schema
{
  "_id": ObjectId("..."),
  "name": "John",
  "orders": [
    { "product": "Phone", "price": 999 }
  ]
}

// Rich queries
db.users.find({ "orders.price": { $gt: 500 } })

// Pros: Flexible schema, rich queries, aggregation pipeline
// Cons: Memory hungry, complex sharding
```

**Cassandra:**
```sql
-- Wide-column store, designed for write-heavy
CREATE TABLE time_series (
    device_id UUID,
    timestamp TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY (device_id, timestamp)
);

-- Query by partition key
SELECT * FROM time_series WHERE device_id = ? AND timestamp > ?;

-- Pros: Linear scalability, no single point of failure
-- Cons: Limited query flexibility, eventual consistency
```

**DynamoDB:**
```javascript
// Key-value with optional sort key
{
  "PK": "USER#123",       // Partition key
  "SK": "ORDER#456",      // Sort key
  "data": { ... }
}

// Single-table design pattern
// Pros: Serverless, auto-scaling, low latency
// Cons: AWS lock-in, complex data modeling
```

**When to Use:**

| Use Case | Recommended |
|----------|-------------|
| Flexible schema, rapid development | MongoDB |
| High write throughput, time-series | Cassandra |
| Serverless, AWS ecosystem | DynamoDB |
| Complex relationships | Still consider RDBMS |

---

## Question 5: SQL Query Optimization techniques?

**Answer:**

**1. Use EXPLAIN ANALYZE**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending';

-- Look for: Seq Scan → Index Scan
```

**2. Avoid SELECT ***
```sql
-- Bad
SELECT * FROM users WHERE id = 1;

-- Good: Only select needed columns
SELECT id, name, email FROM users WHERE id = 1;
```

**3. Use Appropriate Indexes**
```sql
-- For frequent query
SELECT * FROM orders WHERE user_id = ? AND status = ?;

-- Create composite index
CREATE INDEX idx_user_status ON orders(user_id, status);
```

**4. Avoid Functions on Indexed Columns**
```sql
-- Bad: Can't use index
SELECT * FROM users WHERE LOWER(email) = 'test@email.com';

-- Good: Use functional index or store lowercase
CREATE INDEX idx_email_lower ON users(LOWER(email));
```

**5. Use LIMIT for Pagination**
```sql
-- Offset pagination (slow for large offsets)
SELECT * FROM orders ORDER BY created_at LIMIT 10 OFFSET 10000;

-- Keyset pagination (faster)
SELECT * FROM orders
WHERE created_at < '2024-01-01'
ORDER BY created_at DESC
LIMIT 10;
```

**6. Batch Operations**
```sql
-- Bad: Multiple round-trips
INSERT INTO users VALUES (1, 'A');
INSERT INTO users VALUES (2, 'B');

-- Good: Single batch
INSERT INTO users VALUES (1, 'A'), (2, 'B'), (3, 'C');
```

**7. Use EXISTS instead of IN**
```sql
-- Slower for large subqueries
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE status = 'active');

-- Faster
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id AND u.status = 'active');
```

**8. Avoid N+1 Query Problem**
```sql
-- Bad: 1 query + N queries
SELECT * FROM orders;
-- For each order: SELECT * FROM users WHERE id = ?

-- Good: JOIN or batch fetch
SELECT o.*, u.name FROM orders o JOIN users u ON o.user_id = u.id;
```

---

## Question 6: Database Normalization và Denormalization?

**Answer:**

**Normalization** là process tổ chức data để reduce redundancy và improve integrity.

**Normal Forms:**

**1NF (First Normal Form)**
- Mỗi column chứa atomic values
- Không có repeating groups

```sql
-- Bad
| id | name | phones              |
| 1  | John | 111-1111, 222-2222  |

-- Good (1NF)
| id | name | phone    |
| 1  | John | 111-1111 |
| 1  | John | 222-2222 |
```

**2NF (Second Normal Form)**
- 1NF + No partial dependencies
- Non-key columns depend on entire primary key

```sql
-- Bad (partial dependency: product_name depends only on product_id)
| order_id | product_id | product_name | quantity |

-- Good (2NF): Separate tables
Orders: order_id, product_id, quantity
Products: product_id, product_name
```

**3NF (Third Normal Form)**
- 2NF + No transitive dependencies
- Non-key columns depend only on primary key

```sql
-- Bad (zip_code → city is transitive)
| id | name | zip_code | city     |

-- Good (3NF)
Users: id, name, zip_code
ZipCodes: zip_code, city
```

**When to Denormalize:**

```sql
-- Normalized (multiple JOINs, slower reads)
SELECT o.id, u.name, p.name
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id;

-- Denormalized (redundant, faster reads)
| order_id | user_name | product_name | quantity |
```

**Denormalization Strategies:**
- **Materialized Views**: Precomputed query results
- **Caching**: Store computed values (e.g., order_total)
- **Data Duplication**: Store frequently accessed data together

**Trade-offs:**

| Normalization | Denormalization |
|---------------|-----------------|
| Reduce redundancy | Increase redundancy |
| Slower reads (JOINs) | Faster reads |
| Faster writes | Slower writes |
| Data integrity | Potential inconsistency |
| OLTP systems | OLAP, read-heavy |

---

## Question 7: Giải thích Database Replication strategies?

**Answer:**

**Replication** là copying và maintaining data trên multiple servers.

**1. Single-Leader (Master-Slave) Replication**
```
        [Master]
       /   |    \
    [Replica] [Replica] [Replica]

Writes → Master only
Reads → Master or Replicas
```

**Sync vs Async Replication:**
```
Synchronous:
- Master waits for replica ack
- Stronger consistency
- Higher latency

Asynchronous:
- Master doesn't wait
- Better performance
- Potential data loss on failure
```

**2. Multi-Leader Replication**
```
[Leader A] ←→ [Leader B] ←→ [Leader C]
    ↓             ↓             ↓
[Replicas]   [Replicas]    [Replicas]

Use case: Multi-datacenter
Challenge: Conflict resolution (LWW, vector clocks)
```

**3. Leaderless (Masterless) Replication**
```
Client writes to multiple nodes
Quorum: W + R > N for consistency

Example: N=3, W=2, R=2
- Write to 2 nodes to succeed
- Read from 2 nodes, take latest
```

**Consistency Guarantees:**
- **Read-after-write**: See your own writes
- **Monotonic reads**: Never see older version after newer
- **Consistent prefix reads**: See writes in order

**Lag Monitoring:**
```sql
-- PostgreSQL: Check replication lag
SELECT pg_last_xlog_receive_location() - pg_last_xlog_replay_location() AS lag;
```

**Failover:**
```
1. Detect failure (heartbeat timeout)
2. Choose new leader (election)
3. Reconfigure clients
4. Handle split-brain scenarios
```

---

## Question 8: Giải thích Database Connection Pooling?

**Answer:**

**Connection Pooling** là maintaining a cache of database connections để reuse.

**Why Connection Pooling?**

```
Without pooling:
Request → Create connection → Execute query → Close connection
         (expensive: TCP handshake, authentication)

With pooling:
Request → Get connection from pool → Execute query → Return to pool
         (reuse existing connections)
```

**Pool Configuration:**
```java
// HikariCP (Java) example
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost/db");
config.setMaximumPoolSize(10);      // Max connections
config.setMinimumIdle(5);           // Min idle connections
config.setConnectionTimeout(30000); // Wait time for connection
config.setIdleTimeout(600000);      // Max idle time
config.setMaxLifetime(1800000);     // Max connection lifetime
```

**Pool Size Formula:**
```
connections = ((core_count * 2) + effective_spindle_count)

Example: 4 cores, SSD
connections = (4 * 2) + 1 = 9

Start small, increase based on monitoring
```

**Common Issues:**

**1. Connection Leak**
```python
# Bad: Connection not returned
conn = pool.get_connection()
conn.execute(query)
# Forgot to close/return

# Good: Use context manager
with pool.get_connection() as conn:
    conn.execute(query)
# Auto-returned
```

**2. Pool Exhaustion**
```
All connections in use → Timeout waiting for connection

Solutions:
- Increase pool size
- Find long-running queries
- Implement query timeout
```

**PostgreSQL PgBouncer:**
```
Application → PgBouncer → PostgreSQL

Modes:
- Session: One connection per session
- Transaction: Connection per transaction (recommended)
- Statement: Connection per statement
```

---

## Question 9: Data Modeling cho specific use cases?

**Answer:**

**1. E-commerce Order System:**
```sql
-- Users
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Products
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0
);

-- Orders (với status tracking)
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    status VARCHAR(50) NOT NULL,
    total DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Order Items (many-to-many with extra data)
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL
);

-- Indexes
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_order_items_order ON order_items(order_id);
```

**2. Social Media (Timeline):**
```sql
-- Fan-out on write (for read-heavy)
CREATE TABLE timeline_entries (
    user_id BIGINT,           -- Timeline owner
    tweet_id BIGINT,          -- Original tweet
    author_id BIGINT,         -- Tweet author
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at, tweet_id)
);

-- Alternative: Fan-out on read (for write-heavy)
-- Compute timeline at read time by querying followed users
```

**3. Chat Application:**
```sql
-- Messages with partition by conversation
CREATE TABLE messages (
    conversation_id UUID,
    message_id TIMEUUID,      -- Sortable by time
    sender_id BIGINT,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Recent messages query
SELECT * FROM messages
WHERE conversation_id = ?
LIMIT 50;
```

**4. Time-series Data:**
```sql
-- Partition by time bucket
CREATE TABLE metrics (
    device_id UUID,
    bucket_date DATE,         -- Partition key
    timestamp TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY ((device_id, bucket_date), timestamp)
);

-- Query within time range
SELECT * FROM metrics
WHERE device_id = ? AND bucket_date = '2024-01-15'
AND timestamp > '2024-01-15 10:00:00';
```

---

## Question 10: PostgreSQL vs MySQL - Key differences?

**Answer:**

**Comparison Table:**

| Feature | PostgreSQL | MySQL (InnoDB) |
|---------|------------|----------------|
| **ACID** | Full | Full |
| **Transactions** | MVCC | MVCC |
| **JSON Support** | Excellent (JSONB) | Good (JSON) |
| **Full-text** | Built-in | Built-in |
| **Replication** | Streaming, Logical | Master-slave |
| **Extensions** | Rich (PostGIS, etc.) | Limited |

**PostgreSQL Strengths:**

```sql
-- 1. Advanced data types
CREATE TABLE geo_data (
    id SERIAL PRIMARY KEY,
    location GEOMETRY(Point, 4326),  -- PostGIS
    metadata JSONB,                   -- Binary JSON
    tags TEXT[]                       -- Arrays
);

-- 2. Rich indexing
CREATE INDEX idx_gin ON products USING GIN(metadata);
CREATE INDEX idx_gist ON geo_data USING GIST(location);

-- 3. CTEs and Window functions
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) as rank
    FROM products
)
SELECT * FROM ranked WHERE rank <= 5;

-- 4. Table inheritance
CREATE TABLE logs (id SERIAL, message TEXT, created_at TIMESTAMP);
CREATE TABLE error_logs () INHERITS (logs);
```

**MySQL Strengths:**

```sql
-- 1. Simpler, widely deployed
-- 2. Better read performance cho simple queries
-- 3. Built-in replication simpler to setup

-- 4. Full-text search
CREATE FULLTEXT INDEX idx_content ON articles(title, body);
SELECT * FROM articles WHERE MATCH(title, body) AGAINST('database');
```

**When to Choose:**

| Use Case | Recommendation |
|----------|----------------|
| Complex queries, analytics | PostgreSQL |
| Geospatial data | PostgreSQL (PostGIS) |
| JSON-heavy workloads | PostgreSQL (JSONB) |
| Simple CRUD, web apps | Both work well |
| Read-heavy, simple queries | MySQL |
| Maximum ecosystem support | MySQL |

**Migration Considerations:**
- SQL syntax differences (e.g., LIMIT, string handling)
- Data type mappings
- Stored procedure syntax
- Replication setup differences

---
