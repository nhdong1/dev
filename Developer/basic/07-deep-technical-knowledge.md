# Deep Technical Knowledge - Interview Questions

## Question 1: Giải thích concurrency vs parallelism và các concurrency primitives?

**Answer:**

**Concurrency vs Parallelism:**

```
Concurrency: Dealing with multiple things at once
Parallelism: Doing multiple things at once

Concurrency (Single core):
Task A: ████░░░░████░░░░████
Task B: ░░░░████░░░░████░░░░
        → Time

Parallelism (Multi-core):
Core 1 - Task A: ████████████████
Core 2 - Task B: ████████████████
                 → Time
```

**Concurrency Primitives:**

**1. Mutex (Mutual Exclusion)**
```go
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```
- Only one thread can hold the lock
- Others wait (block) until lock is released

**2. Semaphore**
```python
from threading import Semaphore

# Allow max 3 concurrent accesses
sem = Semaphore(3)

def access_resource():
    with sem:
        # Max 3 threads here concurrently
        do_work()
```
- Generalized mutex (N permits instead of 1)
- Use case: Connection pools, rate limiting

**3. Read-Write Lock**
```go
var rwmu sync.RWMutex

func read() {
    rwmu.RLock()
    defer rwmu.RUnlock()
    // Multiple readers allowed
}

func write() {
    rwmu.Lock()
    defer rwmu.Unlock()
    // Exclusive access
}
```
- Multiple readers OR one writer
- Optimized for read-heavy workloads

**4. Condition Variable**
```python
condition = threading.Condition()
queue = []

def producer():
    with condition:
        queue.append(item)
        condition.notify()  # Wake one waiter

def consumer():
    with condition:
        while not queue:
            condition.wait()  # Release lock and wait
        return queue.pop(0)
```

**5. Atomic Operations**
```go
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)  // Lock-free
}

// Compare-and-Swap (CAS)
atomic.CompareAndSwapInt64(&value, expected, new)
```
- Hardware-supported, lock-free
- Used to build higher-level primitives

**Concurrency Problems:**
```
Deadlock: Two threads waiting for each other
Livelock: Threads change state but make no progress
Starvation: Thread never gets CPU time
Race condition: Output depends on timing
```

---

## Question 2: Memory management và garbage collection deep dive?

**Answer:**

**Memory Regions:**
```
┌─────────────────────────────┐ High address
│         Stack               │ ← Local variables, function calls
├─────────────────────────────┤
│           ↓                 │
│         (free)              │
│           ↑                 │
├─────────────────────────────┤
│         Heap                │ ← Dynamic allocation
├─────────────────────────────┤
│         BSS                 │ ← Uninitialized globals
├─────────────────────────────┤
│         Data                │ ← Initialized globals
├─────────────────────────────┤
│         Text                │ ← Program code
└─────────────────────────────┘ Low address
```

**Stack vs Heap:**
| Aspect | Stack | Heap |
|--------|-------|------|
| Allocation | Auto (fast) | Manual/GC (slower) |
| Size | Limited (MB) | Large (GB) |
| Lifetime | Function scope | Until freed/GC |
| Thread safety | Thread-local | Shared |

**Garbage Collection Algorithms:**

**1. Reference Counting**
```python
# Each object tracks reference count
# Free when count reaches 0

Pros: Immediate reclamation
Cons: Cannot handle circular references
      Overhead on every assignment
```

**2. Mark-and-Sweep**
```
Phase 1 (Mark): Start from roots, mark reachable objects
Phase 2 (Sweep): Free unmarked objects

Roots: Stack variables, global variables, static fields

Pros: Handles circular references
Cons: Stop-the-world pause
```

**3. Generational GC (Used by Java, .NET, Go)**
```
Observation: Most objects die young

┌───────────────────────────────────────┐
│ Young Generation (Eden + Survivor)    │ ← Minor GC (frequent, fast)
├───────────────────────────────────────│
│ Old Generation                        │ ← Major GC (rare, slow)
├───────────────────────────────────────│
│ Permanent/Metaspace                   │ ← Class metadata
└───────────────────────────────────────┘

Objects promoted after surviving N minor GCs
```

**4. Concurrent/Incremental GC**
```
G1 GC (Java):
- Divides heap into regions
- Concurrent marking
- Prioritizes regions with most garbage
- Targets pause time goals

ZGC/Shenandoah:
- Sub-millisecond pauses
- Concurrent compaction
- Colored pointers
```

**Memory Optimization Tips:**
```java
// Object pooling (reuse expensive objects)
ObjectPool pool = new ObjectPool();
Object obj = pool.acquire();
// use obj
pool.release(obj);

// Avoid boxing/unboxing
List<int> vs List<Integer>  // Prefer primitives

// String interning
String s = new String("hello").intern();

// Off-heap memory for large data
ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
```

**GC Tuning (Java example):**
```bash
java -Xms4g -Xmx4g \           # Heap size
     -XX:+UseG1GC \             # Use G1
     -XX:MaxGCPauseMillis=200 \ # Target pause
     -XX:+PrintGCDetails        # Logging
```

---

## Question 3: Network Protocols - TCP/IP, HTTP/2, gRPC?

**Answer:**

**TCP/IP Stack:**
```
Application    [HTTP, gRPC, DNS, ...]
Transport      [TCP, UDP]
Network        [IP, ICMP]
Link           [Ethernet, WiFi]
```

**TCP Three-way Handshake:**
```
Client              Server
   │  SYN (seq=x)     │
   │ ─────────────────→
   │                  │
   │  SYN-ACK         │
   │  (seq=y, ack=x+1)│
   │ ←─────────────────
   │                  │
   │  ACK (ack=y+1)   │
   │ ─────────────────→
   │                  │
   └── Connection ────┘
```

**TCP vs UDP:**
| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | Best-effort |
| Ordering | Ordered | No ordering |
| Speed | Slower | Faster |
| Use case | HTTP, file transfer | Video streaming, DNS, gaming |

**HTTP/1.1 vs HTTP/2 vs HTTP/3:**

```
HTTP/1.1:
- One request per TCP connection (or keep-alive)
- Head-of-line blocking
- Text-based headers

HTTP/2:
- Multiplexing (multiple streams per connection)
- Binary framing
- Header compression (HPACK)
- Server push

┌─────────────────────────────────┐
│     Single TCP Connection       │
│  ┌───────┐ ┌───────┐ ┌───────┐ │
│  │Stream1│ │Stream2│ │Stream3│ │
│  └───────┘ └───────┘ └───────┘ │
└─────────────────────────────────┘

HTTP/3:
- Based on QUIC (UDP)
- No head-of-line blocking at transport layer
- Built-in encryption (TLS 1.3)
- Connection migration (change IP/port)
```

**gRPC:**
```protobuf
// Protocol Buffers definition
syntax = "proto3";

service UserService {
    rpc GetUser (UserRequest) returns (UserResponse);
    rpc ListUsers (Empty) returns (stream User);  // Server streaming
    rpc CreateUsers (stream User) returns (Summary);  // Client streaming
    rpc Chat (stream Message) returns (stream Message);  // Bidirectional
}

message UserRequest {
    string id = 1;
}

message UserResponse {
    string id = 1;
    string name = 2;
    string email = 3;
}
```

**gRPC vs REST:**
| Aspect | gRPC | REST |
|--------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 or HTTP/2 |
| Format | Protocol Buffers (binary) | JSON (text) |
| Streaming | Bidirectional | Limited |
| Code generation | Built-in | Manual/optional |
| Browser support | Via grpc-web proxy | Native |
| Use case | Microservices, high-perf | Public APIs, web |

---

## Question 4: Performance optimization techniques?

**Answer:**

**Performance Optimization Process:**

```
1. Measure → 2. Identify Bottleneck → 3. Optimize → 4. Measure Again
                    ↑                                      │
                    └──────────────────────────────────────┘
```

**Profiling Tools:**
```
CPU Profiling: Find slow functions
- Java: JProfiler, async-profiler
- Python: cProfile, py-spy
- Node.js: 0x, clinic.js
- Go: pprof

Memory Profiling: Find leaks, high usage
- Java: Eclipse MAT, VisualVM
- Python: memory_profiler, tracemalloc
- Go: pprof heap

Tracing: Request flow across services
- Jaeger, Zipkin, OpenTelemetry
```

**Common Optimizations:**

**1. Algorithm & Data Structure**
```python
# O(n²) → O(n log n) or O(n)
# Example: Use hash set instead of list for lookups

# Bad: O(n) lookup
if item in large_list: ...

# Good: O(1) lookup
if item in large_set: ...
```

**2. Database Optimization**
```sql
-- Add indexes for frequent queries
CREATE INDEX idx_users_email ON users(email);

-- N+1 query problem
-- Bad: SELECT * FROM orders; + N queries for users
-- Good: SELECT o.*, u.name FROM orders o JOIN users u

-- Pagination
-- Bad: OFFSET 10000 LIMIT 10
-- Good: WHERE id > last_seen_id LIMIT 10
```

**3. Caching**
```python
# In-memory cache
from functools import lru_cache

@lru_cache(maxsize=1000)
def expensive_function(x):
    return compute(x)

# Distributed cache
cache.set(f"user:{id}", user_data, ttl=3600)
user = cache.get(f"user:{id}") or db.get_user(id)
```

**4. Async/Concurrent Processing**
```python
# Sequential (slow)
for url in urls:
    result = fetch(url)

# Concurrent (fast)
import asyncio

async def fetch_all(urls):
    tasks = [fetch(url) for url in urls]
    return await asyncio.gather(*tasks)
```

**5. Connection Pooling**
```python
# Reuse database connections
pool = ConnectionPool(minconn=5, maxconn=20)
conn = pool.getconn()
# use connection
pool.putconn(conn)
```

**6. Batching**
```python
# Bad: Many small operations
for item in items:
    db.insert(item)

# Good: Batch operations
db.insert_many(items)
```

**7. Lazy Loading**
```python
# Don't load until needed
class LazyProperty:
    def __init__(self, func):
        self.func = func

    def __get__(self, obj, type=None):
        value = self.func(obj)
        setattr(obj, self.func.__name__, value)
        return value
```

**8. Memory Optimization**
```python
# Use generators instead of lists
def process_large_file(filename):
    with open(filename) as f:
        for line in f:  # Generator, not list
            yield process(line)

# Use __slots__ in Python classes
class Point:
    __slots__ = ['x', 'y']  # Less memory than dict
```

---

## Question 5: Multithreading patterns và thread safety?

**Answer:**

**Thread Safety Issues:**

**1. Race Condition**
```python
# Unsafe
counter = 0

def increment():
    global counter
    counter += 1  # Not atomic: read, add, write

# Running 1000 times in 10 threads ≠ 10000
```

**2. Data Race**
```c
// Thread 1          // Thread 2
x = 1;               y = x;  // May see 0 or 1
```

**Thread-Safe Patterns:**

**1. Immutability**
```python
from dataclasses import dataclass

@dataclass(frozen=True)  # Immutable
class Point:
    x: int
    y: int

# Immutable objects are inherently thread-safe
```

**2. Thread Confinement**
```python
# ThreadLocal: Each thread has its own copy
import threading
local_data = threading.local()

def get_connection():
    if not hasattr(local_data, 'connection'):
        local_data.connection = create_connection()
    return local_data.connection
```

**3. Producer-Consumer Pattern**
```python
from queue import Queue
from threading import Thread

queue = Queue()

def producer():
    for item in items:
        queue.put(item)

def consumer():
    while True:
        item = queue.get()
        process(item)
        queue.task_done()

# Start threads
Thread(target=producer).start()
for _ in range(4):  # 4 consumers
    Thread(target=consumer, daemon=True).start()

queue.join()  # Wait until all tasks done
```

**4. Read-Write Lock Pattern**
```python
from threading import RLock

class ThreadSafeDict:
    def __init__(self):
        self._dict = {}
        self._lock = RLock()

    def get(self, key):
        with self._lock:
            return self._dict.get(key)

    def set(self, key, value):
        with self._lock:
            self._dict[key] = value
```

**5. Double-Checked Locking (Singleton)**
```java
public class Singleton {
    private static volatile Singleton instance;
    private static final Object lock = new Object();

    public static Singleton getInstance() {
        if (instance == null) {  // First check (no locking)
            synchronized (lock) {
                if (instance == null) {  // Second check
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**6. Actor Model (Message Passing)**
```python
# Akka-style actors: No shared state
# Each actor has mailbox, processes messages sequentially

class Actor:
    def __init__(self):
        self.mailbox = Queue()

    def send(self, message):
        self.mailbox.put(message)

    def receive(self):
        while True:
            message = self.mailbox.get()
            self.handle(message)
```

**7. Lock-Free Data Structures**
```java
// Use atomic operations
AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();  // Atomic
counter.compareAndSet(expected, new);  // CAS
```

**Best Practices:**
```
1. Prefer immutability
2. Minimize shared state
3. Use high-level abstractions (concurrent collections)
4. Lock ordering to prevent deadlock
5. Keep critical sections small
6. Use appropriate granularity (fine vs coarse)
```

---

## Question 6: Code Review best practices?

**Answer:**

**Code Review Goals:**
- Find bugs early
- Improve code quality
- Share knowledge
- Maintain consistency
- Mentoring opportunity

**As a Reviewer:**

**1. Understand Context First**
```
- Read PR description
- Understand the problem being solved
- Check linked issues/tickets
- Run the code if needed
```

**2. Review Checklist**
```markdown
## Functionality
- [ ] Does it solve the stated problem?
- [ ] Are edge cases handled?
- [ ] Are there any bugs?

## Design
- [ ] Is the approach appropriate?
- [ ] Single Responsibility Principle?
- [ ] DRY (Don't Repeat Yourself)?

## Code Quality
- [ ] Is it readable?
- [ ] Are names meaningful?
- [ ] Is complexity manageable?

## Testing
- [ ] Are there adequate tests?
- [ ] Are edge cases tested?
- [ ] Do tests actually test the feature?

## Security
- [ ] Input validation?
- [ ] Authentication/authorization?
- [ ] No sensitive data exposed?

## Performance
- [ ] Obvious performance issues?
- [ ] N+1 queries?
- [ ] Memory leaks?
```

**3. Give Actionable Feedback**
```python
# Bad comment:
"This is wrong"

# Good comment:
"This might cause an IndexError when the list is empty.
Consider adding a check:
if items:
    return items[0]
return default_value"
```

**4. Distinguish Severity**
```
[Blocking] Must fix before merge
[Suggestion] Consider this improvement
[Nitpick] Minor style preference
[Question] Seeking clarification
```

**5. Be Constructive**
```
Instead of:
"This is a terrible way to do it"

Try:
"I think we could simplify this by using X.
It would reduce complexity from O(n²) to O(n).
What do you think?"
```

**As an Author:**

**1. Make Reviews Easy**
```
- Small, focused PRs (< 400 lines ideal)
- Descriptive title and description
- Self-review before requesting
- Add comments for complex logic
```

**PR Template:**
```markdown
## What
Brief description of changes

## Why
Link to issue/ticket, business context

## How
Technical approach, key decisions

## Testing
How was this tested?

## Screenshots
If UI changes

## Checklist
- [ ] Tests added
- [ ] Documentation updated
- [ ] No console.logs
```

**2. Respond Gracefully**
```
- Thank reviewers
- Explain reasoning if disagreeing
- Ask for clarification if unclear
- Don't take feedback personally
```

**Code Review Metrics:**
```
Time to first review: < 24 hours
Review turnaround: < 4 hours for follow-ups
PR size: < 400 lines changed
Review participation: Everyone reviews
```

---

## Question 7: System Performance Debugging?

**Answer:**

**Performance Debugging Framework:**

```
Symptoms → Metrics → Hypotheses → Validation → Fix
```

**1. Gather Symptoms**
```
- What is slow? (specific operation)
- When did it start? (recent changes?)
- How reproducible? (always vs sometimes)
- What's the impact? (latency, throughput, errors)
```

**2. Check System Metrics**

**CPU:**
```bash
# High CPU
top -H -p <pid>        # Thread CPU usage
perf top               # CPU profiling
perf record -g -p <pid>  # Detailed profiling
```

**Memory:**
```bash
# Memory issues
free -h                 # System memory
pmap -x <pid>          # Process memory map
cat /proc/<pid>/smaps  # Detailed memory

# Java heap dump
jmap -dump:format=b,file=heap.bin <pid>
```

**I/O:**
```bash
# Disk I/O
iostat -x 1            # Disk stats
iotop                  # I/O by process
lsof -p <pid>         # Open files

# Network I/O
netstat -tuln          # Listening ports
ss -s                  # Socket statistics
tcpdump -i eth0        # Packet capture
```

**3. Application-Level Profiling**

**Java:**
```bash
# CPU profiler
async-profiler -d 30 -f profile.html <pid>

# Thread dump
jstack <pid> > threads.txt

# GC logs
-Xlog:gc*:file=gc.log
```

**Python:**
```python
import cProfile
cProfile.run('my_function()', 'output.prof')

# Visualize
python -m snakeviz output.prof
```

**4. Common Root Causes**

| Symptom | Possible Cause |
|---------|----------------|
| High CPU | Infinite loop, inefficient algorithm |
| High memory | Memory leak, large cache |
| Slow response | N+1 queries, missing index, external service |
| Timeouts | Connection pool exhausted, deadlock |
| Intermittent | GC pauses, resource contention |

**5. Database Performance**
```sql
-- PostgreSQL: Find slow queries
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Check for missing indexes
EXPLAIN ANALYZE SELECT ...;
-- Look for Seq Scan on large tables
```

**6. Example Investigation**
```
Problem: API latency increased from 100ms to 2s

Step 1: Check metrics
- CPU/Memory normal
- Database latency high

Step 2: Database investigation
- Slow query log shows SELECT ... taking 1.5s
- EXPLAIN shows Seq Scan on 10M row table

Step 3: Hypothesis
- Missing index on filter column

Step 4: Validate
- Create index: CREATE INDEX idx_x ON table(column)
- Verify query uses index

Step 5: Monitor
- Confirm latency back to 100ms
```

---

## Question 8: Event-Driven Architecture và Message Processing?

**Answer:**

**Event-Driven Architecture (EDA) Patterns:**

**1. Event Notification**
```
Producer publishes event
Consumers react independently
No response expected

┌──────────┐    Event    ┌──────────┐
│ Producer │────────────→│ Consumer │
└──────────┘             └──────────┘
                         ┌──────────┐
                    ────→│ Consumer │
                         └──────────┘
```

**2. Event-Carried State Transfer**
```json
// Event contains all data needed
{
  "event": "OrderCreated",
  "data": {
    "orderId": "123",
    "userId": "456",
    "items": [...],
    "total": 99.99,
    "shippingAddress": {...}
  }
}

// Consumer doesn't need to query producer
// Increases coupling but reduces runtime dependency
```

**3. Event Sourcing**
```
Store events, not current state
Rebuild state by replaying events

Events: (immutable log)
1. AccountCreated(id=1, balance=0)
2. MoneyDeposited(id=1, amount=100)
3. MoneyWithdrawn(id=1, amount=30)

Current state: balance = 70 (computed from events)
```

**4. CQRS (Command Query Responsibility Segregation)**
```
┌─────────┐     Commands     ┌──────────────┐
│ Client  │─────────────────→│ Write Model  │
└─────────┘                  └──────────────┘
    │                              │
    │                         Events│
    │                              ↓
    │      Queries          ┌──────────────┐
    └──────────────────────→│ Read Model   │
                            └──────────────┘

Separate models for read and write
Optimized for each use case
```

**Message Processing Patterns:**

**1. Competing Consumers**
```
             ┌──────────┐
Message ────→│ Consumer │
Queue        ├──────────┤
             │ Consumer │  (Only one processes each message)
             ├──────────┤
             │ Consumer │
             └──────────┘

Use case: Load balancing, horizontal scaling
```

**2. Fan-out (Pub/Sub)**
```
             ┌──────────┐
             │ Consumer │  (All receive copy)
Topic ──────→├──────────┤
             │ Consumer │
             ├──────────┤
             │ Consumer │
             └──────────┘

Use case: Event notification, broadcasting
```

**3. Saga Pattern (Distributed Transactions)**
```
Order Service → Payment Service → Inventory Service
     ↓               ↓                  ↓
   Create         Charge            Reserve
    Order         Payment           Stock
     ↓               ↓ (failure)
   Compensa-     Refund ←──────────────┘
   ting
   Transaction

Choreography: Each service listens and reacts
Orchestration: Central coordinator controls flow
```

**Message Guarantees:**

| Guarantee | Description | Trade-off |
|-----------|-------------|-----------|
| At-most-once | May lose messages | Fastest, simplest |
| At-least-once | May duplicate | Must be idempotent |
| Exactly-once | No loss, no duplicates | Complex, expensive |

**Idempotency:**
```python
# Make operations idempotent
def process_payment(payment_id, amount):
    # Check if already processed
    if payment_exists(payment_id):
        return get_existing_result(payment_id)

    # Process new payment
    result = charge(amount)
    save_result(payment_id, result)
    return result
```

---

## Question 9: Distributed Systems Patterns?

**Answer:**

**1. Service Discovery**
```
Problem: How do services find each other?

Client-side discovery:
┌────────┐  1. Query  ┌──────────────┐
│ Client │───────────→│ Service      │
│        │←───────────│ Registry     │
│        │  2. List   │ (Consul, etcd)│
└────────┘            └──────────────┘
     │ 3. Direct call
     ↓
┌────────────┐
│ Service A  │
└────────────┘

Server-side discovery (Load Balancer):
Client → Load Balancer → Service
         (handles discovery)
```

**2. Circuit Breaker**
```
CLOSED → (failures > threshold) → OPEN
   ↑                                 ↓
   └── (success) ← HALF-OPEN ←── (timeout)

Prevents cascade failures
Fast-fail instead of waiting for timeout
```

**3. Bulkhead**
```
Isolate resources per service/function
Failure in one doesn't affect others

┌─────────────────────────────────┐
│ Thread Pool: Service A (20)     │
├─────────────────────────────────┤
│ Thread Pool: Service B (10)     │
├─────────────────────────────────┤
│ Thread Pool: Service C (10)     │
└─────────────────────────────────┘

If Service A is slow, only its pool exhausted
```

**4. Retry with Exponential Backoff**
```python
def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError:
            if attempt == max_retries - 1:
                raise
            sleep_time = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(sleep_time)
```

**5. Leader Election**
```
Problem: Only one node should do certain work

Algorithms:
- Bully Algorithm
- Ring Algorithm
- Raft/Paxos consensus

Tools:
- etcd: lease-based election
- ZooKeeper: ephemeral nodes
- Kubernetes: Lease API
```

**6. Distributed Locks**
```python
# Redlock algorithm (Redis)
def acquire_lock(resource, ttl):
    lock_value = generate_unique_id()

    # Try to set if not exists
    if redis.set(resource, lock_value, nx=True, px=ttl):
        return lock_value
    return None

def release_lock(resource, lock_value):
    # Only release if we own the lock
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    redis.eval(script, 1, resource, lock_value)
```

**7. Health Checks**
```python
@app.route('/health/live')
def liveness():
    # Is the process alive?
    return {'status': 'ok'}, 200

@app.route('/health/ready')
def readiness():
    # Can the service handle requests?
    if not db.is_connected():
        return {'status': 'not ready'}, 503
    return {'status': 'ready'}, 200
```

**8. Sidecar Pattern**
```
┌─────────────────────────────────┐
│           Pod                    │
│  ┌───────────┐  ┌─────────────┐ │
│  │   App     │  │   Sidecar   │ │
│  │           │  │ (Envoy,     │ │
│  │           │  │  Logging,   │ │
│  │           │  │  Monitoring)│ │
│  └───────────┘  └─────────────┘ │
└─────────────────────────────────┘

App doesn't need to implement cross-cutting concerns
```

---

## Question 10: API Design best practices?

**Answer:**

**RESTful API Design:**

**1. Resource Naming**
```
Good:
GET    /users              # List users
GET    /users/123          # Get user 123
POST   /users              # Create user
PUT    /users/123          # Update user 123
DELETE /users/123          # Delete user 123
GET    /users/123/orders   # User's orders

Bad:
GET    /getUsers
POST   /createUser
GET    /user/123/getOrders
```

**2. HTTP Methods Correctly**
```
GET:    Read, idempotent, cacheable
POST:   Create, not idempotent
PUT:    Full update, idempotent
PATCH:  Partial update
DELETE: Remove, idempotent
```

**3. Status Codes**
```
2xx Success:
200 OK - General success
201 Created - Resource created
204 No Content - Success, no body

4xx Client Error:
400 Bad Request - Invalid input
401 Unauthorized - Auth required
403 Forbidden - Auth valid, no permission
404 Not Found - Resource not found
409 Conflict - Resource conflict
422 Unprocessable Entity - Validation error
429 Too Many Requests - Rate limited

5xx Server Error:
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

**4. Request/Response Format**
```json
// Request: POST /users
{
  "email": "user@example.com",
  "name": "John Doe"
}

// Response: 201 Created
{
  "data": {
    "id": "123",
    "email": "user@example.com",
    "name": "John Doe",
    "createdAt": "2024-01-15T10:30:00Z"
  },
  "meta": {
    "requestId": "abc-123"
  }
}

// Error Response: 422 Unprocessable Entity
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  },
  "meta": {
    "requestId": "abc-124"
  }
}
```

**5. Pagination**
```json
GET /users?page=2&limit=20

{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  },
  "links": {
    "self": "/users?page=2&limit=20",
    "first": "/users?page=1&limit=20",
    "prev": "/users?page=1&limit=20",
    "next": "/users?page=3&limit=20",
    "last": "/users?page=8&limit=20"
  }
}
```

**6. Filtering, Sorting, Fields Selection**
```
# Filtering
GET /users?status=active&role=admin

# Sorting
GET /users?sort=-createdAt,name  # - for descending

# Field selection
GET /users?fields=id,name,email
```

**7. Versioning**
```
URL: /api/v1/users (Most common)
Header: Accept: application/vnd.api.v1+json
Query: /users?version=1
```

**8. Rate Limiting Headers**
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1609459200
Retry-After: 3600
```

**9. HATEOAS (Hypermedia)**
```json
{
  "data": {
    "id": "123",
    "status": "pending"
  },
  "links": {
    "self": "/orders/123",
    "approve": "/orders/123/approve",
    "cancel": "/orders/123/cancel"
  }
}
```

**10. Documentation**
```yaml
# OpenAPI/Swagger
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [active, inactive]
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
```

---
