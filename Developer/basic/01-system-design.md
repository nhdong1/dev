# System Design - Interview Questions

## Question 1: Giải thích CAP Theorem và cách áp dụng trong thiết kế hệ thống phân tán?

**Answer:**

CAP Theorem phát biểu rằng một distributed system chỉ có thể đảm bảo tối đa 2 trong 3 thuộc tính:

- **Consistency (C)**: Mọi read đều nhận được dữ liệu mới nhất hoặc error
- **Availability (A)**: Mọi request đều nhận được response (không đảm bảo là dữ liệu mới nhất)
- **Partition Tolerance (P)**: Hệ thống tiếp tục hoạt động dù có network partition

**Thực tế áp dụng:**
- **CP Systems** (Consistency + Partition Tolerance): MongoDB, HBase, Redis Cluster. Phù hợp cho banking, financial transactions
- **AP Systems** (Availability + Partition Tolerance): Cassandra, DynamoDB, CouchDB. Phù hợp cho social media, real-time analytics
- **CA Systems**: Chỉ tồn tại trong single-node systems vì network partition luôn có thể xảy ra

**Trade-off trong thực tế:**
```
Banking System → Chọn CP: Không thể chấp nhận inconsistent balance
E-commerce Cart → Chọn AP: User experience quan trọng hơn, có thể merge conflicts sau
```

---

## Question 2: So sánh Microservices vs Monolith architecture. Khi nào nên chọn cái nào?

**Answer:**

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | Single deployment unit | Independent deployment per service |
| **Scaling** | Scale toàn bộ application | Scale từng service riêng lẻ |
| **Technology** | Single tech stack | Polyglot (nhiều languages/frameworks) |
| **Team Structure** | Một team lớn | Nhiều team nhỏ, autonomous |
| **Complexity** | Simple initially | Complex infrastructure |
| **Data Management** | Single database | Database per service |

**Chọn Monolith khi:**
- Team nhỏ (< 10 developers)
- MVP hoặc startup phase
- Domain chưa rõ ràng, cần iterate nhanh
- Limited DevOps expertise

**Chọn Microservices khi:**
- Team lớn, cần scale independently
- High traffic với different scaling needs
- Clear bounded contexts
- Mature DevOps practices (CI/CD, monitoring, containerization)

**Migration Strategy:**
```
Monolith → Strangler Fig Pattern → Gradually extract services → Full Microservices
```

---

## Question 3: Thiết kế một URL Shortener service như bit.ly. Giải thích các design decisions?

**Answer:**

**Functional Requirements:**
- Shorten long URL → short URL
- Redirect short URL → original URL
- Optional: Custom aliases, expiration, analytics

**Non-functional Requirements:**
- High availability, low latency (< 100ms)
- 100M URLs/day, 10:1 read/write ratio

**High-Level Design:**

```
Client → Load Balancer → API Gateway → URL Service → Database
                                    ↓
                              Cache (Redis)
```

**Key Design Decisions:**

1. **URL Encoding**: Base62 (a-z, A-Z, 0-9) với 7 characters = 62^7 = 3.5 trillion URLs
   ```
   ID: 12345 → Base62: "dnh"
   ```

2. **ID Generation**:
   - Option 1: Auto-increment DB + Base62 (simple, predictable)
   - Option 2: Snowflake ID (distributed, no single point of failure)
   - Option 3: MD5/SHA256 hash + take first 7 chars (collision possible)

3. **Database Choice**:
   - NoSQL (Cassandra/DynamoDB) cho write-heavy workload
   - Partition by short_url hash for even distribution

4. **Caching Strategy**:
   - Cache hot URLs in Redis (LRU eviction)
   - Cache-aside pattern: Check cache → miss → query DB → populate cache

5. **Rate Limiting**: Token bucket algorithm per user/IP

---

## Question 4: Giải thích các caching strategies và khi nào sử dụng mỗi strategy?

**Answer:**

**1. Cache-Aside (Lazy Loading)**
```
Read: App checks cache → miss → query DB → write to cache → return
Write: App writes to DB → invalidate cache
```
- **Pros**: Only cache what's needed, resilient to cache failures
- **Cons**: Cache miss penalty, potential stale data
- **Use case**: Read-heavy workloads, general purpose

**Lưu ý khi implement:**
- Luôn **invalidate trước rồi mới ghi** (hoặc dùng pattern invalidate + small TTL) để tránh đọc stale data.
- Xử lý **cache stampede**: dùng random TTL, mutex/lock per key, request coalescing.
- Nếu DB fail sau khi invalidate cache → có thể gây **cache penetration** (nhiều request cùng đập vào DB); dùng bulkhead / fallback hợp lý.
- Với data quan trọng, nên dùng **version trong key** (`user:123:v2`) để rollout schema mới an toàn.

**2. Write-Through**
```
Read: Always from cache
Write: App writes to cache → cache writes to DB (synchronous)
```
- **Pros**: Data consistency, no stale data
- **Cons**: Write latency, cache filled with unused data
- **Use case**: Applications requiring strong consistency

**Lưu ý khi implement:**
- Ghi cache + DB phải là **1 operation logic**: nếu ghi DB fail phải rollback cache (hoặc không update cache).
- Cần **timeout & retry policy** rõ ràng cho path ghi DB, tránh treo ứng dụng nếu DB chậm.
- Chỉ dùng cho **data size vừa phải**; dữ liệu ít được đọc nhưng vẫn bị đẩy vào cache gây tốn RAM.

**3. Write-Behind (Write-Back)**
```
Read: Always from cache
Write: App writes to cache → cache async writes to DB (batched)
```
- **Pros**: Low write latency, reduced DB load
- **Cons**: Data loss risk if cache fails before sync
- **Use case**: High write throughput, acceptable eventual consistency

**Lưu ý khi implement:**
- Cần **durable buffer** (queue, log) giữa cache và DB để hạn chế mất dữ liệu khi cache node chết.
- Thiết kế **flush policy**: theo thời gian (interval), theo batch size, hoặc combination.
- Đảm bảo **ordering** cho cùng một key (ví dụ: dùng partition theo key trong queue).
- Cần cơ chế **graceful shutdown** để flush hết buffer trước khi dừng service.

**4. Read-Through**
```
Read: App reads from cache → cache loads from DB if miss
```
- **Pros**: Simplified application logic
- **Cons**: First request always slow
- **Use case**: Read-heavy with predictable access patterns

**Lưu ý khi implement:**
- Thường do **library / cache provider** hỗ trợ; cần chuẩn hóa function loader (đảm bảo idempotent, timeout).
- Nên có cơ chế **warm-up cache** (preload hot keys) để tránh first-request penalty sau deploy.
- Xử lý **cache penetration** cho key không tồn tại: cache negative results với TTL ngắn.

**Cache Invalidation Strategies:**
- **TTL (Time-To-Live)**: Simple, but may serve stale data
- **Event-driven**: Invalidate on data change events
- **Version-based**: Include version in cache key

**Lưu ý chung khi dùng cache:**
- Định nghĩa rõ **cache key convention** (`service:entity:id:field`) để dễ debug và tránh collision.
- Chuẩn hóa **serialization** (JSON, protobuf, msgpack) và quản lý backward compatibility.
- Thêm **metrics & logging**: hit ratio, latency, error rate, size per key pattern để tối ưu sau này.
- Luôn thiết kế logic **chịu được cache down** (có fallback vào DB hoặc degrade gracefully).

---

## Question 5: Thiết kế Rate Limiter. So sánh các algorithms khác nhau?

**Answer:**

**Rate Limiting Algorithms:**

**1. Token Bucket**
```
- Bucket holds tokens (max = bucket size)
- Tokens added at fixed rate
- Request consumes 1 token
- No token → request rejected
```
- **Pros**: Allows burst traffic, smooth rate limiting
- **Cons**: Memory for storing tokens
- **Use case**: API rate limiting, network traffic shaping

**2. Leaky Bucket**
```
- Requests enter bucket (queue)
- Processed at fixed rate (leak)
- Bucket full → request rejected
```
- **Pros**: Smooth output rate, no bursts
- **Cons**: Doesn't handle burst well
- **Use case**: Traffic shaping, network congestion control

**3. Fixed Window Counter**
```
- Count requests in fixed time window (e.g., per minute)
- Reset counter at window boundary
```
- **Pros**: Simple, memory efficient
- **Cons**: Burst at window edges (2x limit possible)
- **Use case**: Simple rate limiting requirements

**4. Sliding Window Log**
```
- Store timestamp of each request
- Count requests in sliding window
```
- **Pros**: Accurate, no edge burst
- **Cons**: Memory intensive (store all timestamps)
- **Use case**: High accuracy requirements

**5. Sliding Window Counter**
```
- Combine fixed window + sliding
- Weighted count based on overlap
```
- **Pros**: Memory efficient, smooth limiting
- **Cons**: Approximate calculation
- **Use case**: Balanced accuracy and performance

**Distributed Rate Limiting:**
- Use Redis with atomic operations (INCR + EXPIRE)
- Consider race conditions with Lua scripts

---

## Question 6: Giải thích Circuit Breaker pattern và cách implement?

**Answer:**

**Circuit Breaker** là một design pattern giúp prevent cascade failures trong distributed systems.

**States:**

```
CLOSED → (failures exceed threshold) → OPEN
   ↑                                      ↓
   ← (success in half-open) ← HALF-OPEN ←
                              (after timeout)
```

1. **CLOSED**: Requests flow normally, failures counted
2. **OPEN**: Requests fail immediately (fail-fast), no calls to downstream
3. **HALF-OPEN**: Allow limited requests to test if service recovered

**Configuration Parameters:**
- `failureThreshold`: Number of failures to trip circuit (e.g., 5)
- `successThreshold`: Successes needed to close circuit (e.g., 3)
- `timeout`: Time before transitioning to half-open (e.g., 30s)

**Implementation Example (Pseudocode):**
```csharp
public class CircuitBreaker {
    private State state = CLOSED;
    private int failureCount = 0;
    private DateTime lastFailureTime;

    public async Task<T> Execute<T>(Func<Task<T>> action) {
        if (state == OPEN) {
            if (DateTime.Now - lastFailureTime > timeout) {
                state = HALF_OPEN;
            } else {
                throw new CircuitOpenException();
            }
        }

        try {
            var result = await action();
            OnSuccess();
            return result;
        } catch {
            OnFailure();
            throw;
        }
    }
}
```

**Libraries:**
- .NET: Polly
- Java: Resilience4j, Hystrix (deprecated)
- Node.js: opossum

**Best Practices:**
- Combine with retry pattern (retry before circuit breaks)
- Implement fallback mechanisms
- Monitor circuit state for alerting

---

## Question 7: Database Sharding strategies và trade-offs của mỗi approach?

**Answer:**

**Sharding** là kỹ thuật horizontal partitioning dữ liệu across multiple database instances.

**Sharding Strategies:**

**1. Range-based Sharding**
```
Shard 1: user_id 1-1M
Shard 2: user_id 1M-2M
Shard 3: user_id 2M-3M
```
- **Pros**: Simple, range queries efficient
- **Cons**: Hotspots, uneven distribution
- **Use case**: Time-series data, sequential IDs

**2. Hash-based Sharding**
```
shard_id = hash(user_id) % num_shards
```
- **Pros**: Even distribution, no hotspots
- **Cons**: Range queries across all shards, resharding is hard
- **Use case**: User data, session data

**3. Directory-based Sharding**
```
Lookup table: user_id → shard_id
```
- **Pros**: Flexible, easy to rebalance
- **Cons**: Lookup table is single point of failure
- **Use case**: Complex sharding logic

**4. Geographic Sharding**
```
Shard by user region: US, EU, APAC
```
- **Pros**: Data locality, compliance (GDPR)
- **Cons**: Uneven load, cross-region queries
- **Use case**: Global applications

**Challenges:**
- **Joins**: Cross-shard joins expensive → denormalize data
- **Transactions**: Distributed transactions (2PC) → saga pattern
- **Resharding**: Consistent hashing to minimize data movement
- **ID Generation**: Globally unique IDs (Snowflake, UUID)

**Consistent Hashing:**
```
Hash ring với virtual nodes
Add/remove node → only K/N keys remapped (K=keys, N=nodes)
```

---

## Question 8: Thiết kế Message Queue system. So sánh Kafka vs RabbitMQ?

**Answer:**

**Message Queue Use Cases:**
- Decoupling services
- Async processing
- Load leveling
- Event sourcing

**Kafka vs RabbitMQ Comparison:**

| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| **Model** | Distributed log | Traditional message broker |
| **Message Retention** | Configurable (days/weeks) | Until consumed |
| **Consumer Model** | Pull-based | Push-based |
| **Ordering** | Per partition | Per queue |
| **Throughput** | Very high (millions/sec) | High (tens of thousands/sec) |
| **Use Case** | Event streaming, log aggregation | Task queues, RPC |
| **Replay** | Yes (offset-based) | No |
| **Routing** | Topics + partitions | Exchanges + routing keys |

**Kafka Architecture:**
```
Producer → Topic (partitioned) → Consumer Group
                ↓
         Partition 0: [msg1, msg2, msg3...]
         Partition 1: [msg4, msg5, msg6...]
```

**Key Kafka Concepts:**
- **Partition**: Unit of parallelism, ordered within partition
- **Consumer Group**: Each partition consumed by one consumer in group
- **Offset**: Position in partition, consumer controls commit
- **Replication**: Leader + followers for fault tolerance

**When to Choose:**
- **Kafka**: Event sourcing, real-time analytics, high throughput, need replay
- **RabbitMQ**: Complex routing, request-reply pattern, lower latency requirement

---

## Question 9: Giải thích Eventual Consistency và các patterns để handle?

**Answer:**

**Eventual Consistency** là model trong đó hệ thống đảm bảo rằng nếu không có updates mới, eventually tất cả replicas sẽ converge về cùng một state.

**So sánh với Strong Consistency:**
- **Strong**: Read always returns latest write
- **Eventual**: Read may return stale data, but eventually consistent

**Patterns để Handle Eventual Consistency:**

**1. Read Your Writes**
```
After write → route subsequent reads to same replica
hoặc include version/timestamp in response
```

**2. Monotonic Reads**
```
User always reads từ same replica
hoặc track last-read timestamp
```

**3. Causal Consistency**
```
Vector clocks để track causality
Writes that are causally related appear in order
```

**4. Conflict Resolution Strategies:**

a) **Last Write Wins (LWW)**
```
Use timestamp, latest write wins
Simple but may lose data
```

b) **Merge/CRDT (Conflict-free Replicated Data Types)**
```
G-Counter: Only increment, merge = max per node
LWW-Register: Last write wins with timestamp
OR-Set: Add wins over remove
```

c) **Application-level Resolution**
```
Present conflicts to user (như Git merge conflicts)
Business logic decides winner
```

**Saga Pattern for Distributed Transactions:**
```
Service A → Event → Service B → Event → Service C
     ↓ (if fail at C)
Compensating transactions: C' → B' → A'
```

**Design Tips:**
- Accept eventual consistency where possible (shopping cart, likes count)
- Use strong consistency for critical data (payments, inventory)
- Design for idempotency (duplicate messages safe)

---

## Question 10: Thiết kế một Distributed Cache system (như Redis Cluster)?

**Answer:**

**Requirements:**
- High availability, low latency (< 1ms)
- Horizontal scalability
- Data persistence (optional)
- Support multiple data structures

**Architecture:**

```
Client → Hash(key) → Correct Node
     ↓
┌─────────────────────────────────────────┐
│  Node 1        Node 2        Node 3     │
│  (slots 0-5k)  (slots 5k-10k) (slots 10k-16k)
│  Master        Master        Master     │
│    ↓             ↓              ↓       │
│  Replica       Replica       Replica    │
└─────────────────────────────────────────┘
```

**Key Design Components:**

**1. Data Partitioning (Hash Slots)**
```
Redis: 16384 hash slots
slot = CRC16(key) % 16384
Each node owns a range of slots
```

**2. Consistent Hashing** (Alternative approach)
```
Virtual nodes trên hash ring
Add/remove node → minimal key redistribution
```

**3. Replication**
```
Async replication: Master → Replica
Quorum writes optional cho strong consistency
```

**4. Failure Detection**
```
Gossip protocol: Nodes exchange state information
Heartbeat timeout → mark node as potentially failed
Quorum agreement → confirm failure → promote replica
```

**5. Client-side Routing**
```
Smart client: Knows cluster topology
MOVED redirect: Try node → wrong node → redirect to correct
ASK redirect: During resharding
```

**6. Persistence Options:**
- **RDB**: Periodic snapshots
- **AOF**: Append-only file (every write logged)
- **Hybrid**: RDB + AOF for balance

**Eviction Policies:**
- `LRU`: Least Recently Used
- `LFU`: Least Frequently Used
- `TTL`: Expire based on time-to-live
- `Random`: Random eviction

**Handling Hot Keys:**
```
Problem: One key gets millions of requests
Solutions:
1. Local cache in application
2. Key splitting: key:1, key:2, key:3 (read any, aggregate)
3. Replica reads: Read from replicas for hot keys
```

**Data Structures:**
- Strings, Lists, Sets, Sorted Sets, Hashes
- Bitmaps, HyperLogLog, Streams

---
