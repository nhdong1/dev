# Profiling — Phân Tích Hiệu Năng Ứng Dụng

> Profiling (Phân Tích Hiệu Năng) là quá trình đo lường chính xác nơi ứng dụng dành
> thời gian và tài nguyên. Không bao giờ tối ưu mà không profile trước — đây là nguyên
> tắc số một của performance engineering (Kỹ Thuật Hiệu Năng).

---

## 📋 Mục Tiêu

- [ ] Hiểu sự khác biệt giữa **sampling** và **instrumentation** profiling
- [ ] Sử dụng **async-profiler** để tạo flame graph (Biểu Đồ Ngọn Lửa)
- [ ] Thu thập và phân tích **JFR** (Java Flight Recorder — Máy Ghi Bay Java) recordings
- [ ] Sử dụng **Spring Boot Actuator** metrics để phát hiện bottleneck
- [ ] Phân tích **thread dumps** (Ảnh Chụp Luồng) và **heap dumps** (Ảnh Chụp Heap)
- [ ] Debug **memory leak** (Rò Rỉ Bộ Nhớ) và **CPU spike** (Đột Biến CPU)

---

## 1. Hai Phương Pháp Profiling

### Sampling Profiler (Profiler Lấy Mẫu)

```
Cơ chế:
  - Định kỳ (ví dụ: mỗi 1ms) lấy mẫu stack trace của tất cả threads
  - Đếm tần suất mỗi method xuất hiện trong mẫu
  - Method xuất hiện nhiều nhất = nơi CPU dành nhiều thời gian nhất

Ưu điểm:
  ✓ Overhead thấp (~1–3%) — có thể dùng trong production
  ✓ Không thay đổi behavior của ứng dụng

Nhược điểm:
  ✗ Không chính xác tuyệt đối — chỉ là xác suất thống kê
  ✗ Có thể bỏ sót methods chạy rất nhanh

Công cụ: async-profiler, JFR
```

### Instrumentation Profiler (Profiler Gắn Công Cụ)

```
Cơ chế:
  - Inject (Tiêm) bytecode vào mỗi method để đo thời gian chính xác
  - Đo CHÍNH XÁC từng method call

Ưu điểm:
  ✓ Chính xác tuyệt đối

Nhược điểm:
  ✗ Overhead cao (~10–50%) — không dùng production
  ✗ Thay đổi JIT optimization (Tối Ưu Hóa JIT) behavior

Công cụ: JProfiler, YourKit, IntelliJ IDEA Profiler
```

---

## 2. async-profiler — Công Cụ Profiling Production-Safe

### Cài Đặt

```bash
# Linux/macOS
curl -L https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz \
  | tar -xz

# Hoặc dùng Docker
docker run --pid=host --privileged \
  -v /proc:/proc \
  -v ./profiling:/output \
  grafana/async-profiler \
  -d 60 -f /output/profile.html <PID>
```

### Profiling Cơ Bản

```bash
# Tìm PID của ứng dụng Spring Boot
PID=$(jps | grep MyApplication | awk '{print $1}')

# 1. Profile CPU trong 60 giây, output HTML flame graph
./asprof -d 60 -f profile.html $PID

# 2. Profile allocation (Cấp Phát Bộ Nhớ) — tìm chỗ tạo nhiều objects
./asprof -e alloc -d 60 -f allocation.html $PID

# 3. Profile lock contention (Tranh Chấp Khóa) — tìm deadlocks và bottlenecks
./asprof -e lock -d 60 -f locks.html $PID

# 4. Profile wall clock time (Thời Gian Thực) — bao gồm I/O và wait time
./asprof -e wall -d 60 -f wall.html $PID
```

### Đọc Flame Graph (Biểu Đồ Ngọn Lửa)

```
Flame Graph được đọc từ dưới lên:
  - Đáy (bottom): Main thread / entry point
  - Đỉnh (top): Leaf methods — nơi CPU thực sự dành thời gian
  - Chiều rộng (width): Tỷ lệ thời gian

┌─────────────────────────────────────────────────────────────┐
│  java/util/HashMap.get  ← Đây là hotspot! Wide = nhiều CPU  │
├───────────────────────────┬─────────────────────────────────┤
│  Service.processItems     │  Serializer.serialize           │
├──────────────────┬────────┴─────────────────────────────────┤
│  Controller.api  │  background threads                      │
├──────────────────┴──────────────────────────────────────────┤
│                    main thread                               │
└─────────────────────────────────────────────────────────────┘

→ Tìm methods rộng ở gần đỉnh → đây là chỗ cần tối ưu

Màu sắc:
  - Đỏ/Cam (Java code): Application code
  - Vàng (native): JVM internals
  - Xanh (kernel): OS / I/O
```

### Profiling Trong Docker Container

```bash
# Chạy async-profiler trong container (cần --privileged)
docker exec -it my-container bash
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar -xzf async-profiler-3.0-linux-x64.tar.gz

# Profile PID 1 (thường là Java process trong container)
./asprof -d 60 -f /tmp/profile.html 1

# Copy ra host để xem
docker cp my-container:/tmp/profile.html ./profile.html
```

---

## 3. JFR — Java Flight Recorder

### JFR Là Gì?

```
JFR (Java Flight Recorder — Máy Ghi Bay Java):
  - Built-in (Tích Hợp Sẵn) vào JDK — không cần cài thêm
  - Low overhead (~1–2%)
  - Thu thập: CPU, memory, GC, threads, exceptions, I/O, class loading
  - Output: .jfr file — mở bằng JMC (JDK Mission Control)
  - Miễn phí từ Java 11+
```

### Thu Thập JFR Recording

```bash
# Bắt đầu recording trong 60 giây
jcmd <PID> JFR.start duration=60s filename=/tmp/recording.jfr

# Hoặc bắt đầu ngay khi khởi động JVM
JAVA_OPTS="\
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=duration=60s,filename=/tmp/recording.jfr,settings=profile"

# Bắt đầu recording liên tục (circular buffer — Bộ Đệm Vòng)
jcmd <PID> JFR.start name=continuous settings=profile maxage=5m maxsize=100m

# Dump recording hiện tại ra file
jcmd <PID> JFR.dump name=continuous filename=/tmp/now.jfr

# Dừng recording
jcmd <PID> JFR.stop name=continuous
```

### JFR Settings Profiles

```
Settings profiles được bundle sẵn trong JDK:
  default: Overhead thấp, đủ cho production monitoring
  profile: Chi tiết hơn, overhead ~2%, phù hợp cho debugging

Tùy chỉnh settings:
  $JAVA_HOME/lib/jfr/default.jfc  ← Template file
  $JAVA_HOME/lib/jfr/profile.jfc
```

### Phân Tích JFR Với JMC

```
JMC (JDK Mission Control — Trung Tâm Điều Khiển JDK):
  1. Tải về: https://jdk.java.net/jmc/
  2. File → Open... → chọn .jfr file

Tabs quan trọng trong JMC:
  - Method Profiling (Phân Tích Phương Thức): Hottest methods
  - Memory (Bộ Nhớ): Allocation rate, GC pauses
  - Threads (Luồng): Thread states, contention
  - Exception (Ngoại Lệ): Thrown exceptions
  - I/O: File và socket operations
  - Event Browser: Tất cả events chi tiết
```

### Continuous JFR Trong Production

```yaml
# application.yml — Spring Boot Actuator JFR endpoint
management:
  endpoints:
    web:
      exposure:
        include: jfr  # Spring Boot 3.2+ — JFR endpoint
```

```bash
# Spring Boot Actuator JFR endpoint
curl -X POST http://localhost:8080/actuator/jfr/start \
  -H "Content-Type: application/json" \
  -d '{"duration": "PT60S"}'

curl http://localhost:8080/actuator/jfr/download > recording.jfr
```

---

## 4. Spring Boot Actuator Metrics

### Cấu Hình Metrics

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: metrics, health, info, threaddump, heapdump
  endpoint:
    health:
      show-details: always
  metrics:
    tags:
      application: ${spring.application.name}
      env: ${spring.profiles.active:default}
```

### Các Metrics Quan Trọng

```bash
# HTTP request metrics
GET /actuator/metrics/http.server.requests

# JVM memory
GET /actuator/metrics/jvm.memory.used
GET /actuator/metrics/jvm.memory.max
GET /actuator/metrics/jvm.gc.pause

# Thread pool
GET /actuator/metrics/executor.active
GET /actuator/metrics/executor.pool.size
GET /actuator/metrics/executor.queue.size

# Database
GET /actuator/metrics/hikaricp.connections.active
GET /actuator/metrics/hikaricp.connections.pending

# Cache
GET /actuator/metrics/cache.gets?tag=result:hit
GET /actuator/metrics/cache.gets?tag=result:miss
```

### Custom Metrics Với Micrometer

```java
@Service
public class OrderService {

    private final MeterRegistry meterRegistry;
    private final Counter orderCreatedCounter;
    private final Timer orderProcessingTimer;
    private final Gauge pendingOrdersGauge;

    public OrderService(MeterRegistry meterRegistry, OrderRepository repository) {
        this.meterRegistry = meterRegistry;

        // Counter (Đếm): đếm số lần sự kiện xảy ra
        this.orderCreatedCounter = Counter.builder("orders.created.total")
            .description("Total orders created")
            .register(meterRegistry);

        // Timer (Bộ Hẹn Giờ): đo thời gian thực hiện
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Time to process an order")
            .register(meterRegistry);

        // Gauge (Đồng Hồ Đo): giá trị tại một thời điểm
        this.pendingOrdersGauge = Gauge.builder("orders.pending.count",
                repository, r -> r.countByStatus("PENDING"))
            .description("Current pending orders count")
            .register(meterRegistry);
    }

    public Order createOrder(CreateOrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(request);
            orderCreatedCounter.increment();
            return order;
        });
    }
}
```

---

## 5. Thread Dump Analysis (Phân Tích Ảnh Chụp Luồng)

### Lấy Thread Dump

```bash
# Cách 1: kill signal (không dừng app)
kill -3 <PID>  # In ra stderr

# Cách 2: jstack
jstack <PID> > thread_dump.txt

# Cách 3: jcmd
jcmd <PID> Thread.print > thread_dump.txt

# Cách 4: Actuator endpoint
curl http://localhost:8080/actuator/threaddump > thread_dump.json
```

### Đọc Thread Dump

```
"http-nio-8080-exec-1" #42 daemon prio=5 os_prio=0 tid=... nid=0x... waiting on condition
   java.lang.Thread.State: WAITING (parking)
     at sun.misc.Unsafe.park(Native Method)
     - parking to wait for  <0x0000000123> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
     at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
     at com.example.service.OrderService.waitForPayment(OrderService.java:89)
     at com.example.controller.OrderController.checkout(OrderController.java:45)

Thread States (Trạng Thái Luồng):
  RUNNABLE: Đang chạy hoặc sẵn sàng chạy
  BLOCKED: Đang chờ monitor lock (khóa monitor)
  WAITING: Đang chờ vô thời hạn (wait(), join(), park())
  TIMED_WAITING: Đang chờ có timeout (sleep(), wait(timeout))
```

### Phát Hiện Deadlock Trong Thread Dump

```
Found one Java-level deadlock:
=============================
"Thread-1" waiting to lock monitor <0xABC> (owned by "Thread-2")
"Thread-2" waiting to lock monitor <0xDEF> (owned by "Thread-1")
← Thread 1 và Thread 2 đang chờ nhau → Deadlock!

Giải pháp:
  1. Luôn acquire locks theo cùng thứ tự trong mọi thread
  2. Dùng tryLock() với timeout thay vì lock()
  3. Dùng ReentrantLock với timeout
```

---

## 6. Phân Tích Heap Dump Cho Memory Leak

### Lấy Heap Dump

```bash
# Khi ứng dụng đang chạy (không dừng)
jmap -dump:format=b,file=/tmp/heap.hprof <PID>

# Với jcmd (khuyến nghị)
jcmd <PID> GC.heap_dump /tmp/heap.hprof

# Actuator endpoint (cần xác nhận — file có thể rất lớn)
curl http://localhost:8080/actuator/heapdump -o heap.hprof
```

### Phân Tích Với Eclipse MAT

```
1. Tải Eclipse MAT: https://eclipse.dev/mat/
2. File → Open Heap Dump → chọn .hprof file
3. Chạy: Reports → Leak Suspects Report (Báo Cáo Nghi Vấn Rò Rỉ)

Leak Suspects tìm:
  - Objects chiếm > 10% heap
  - Collections lớn bất thường
  - Nhiều instances của cùng 1 class

OQL (Object Query Language) để điều tra:
  SELECT * FROM java.util.HashMap WHERE size() > 10000
  SELECT * FROM java.lang.String
```

### Memory Leak Patterns Phổ Biến Trong Spring

```java
// LEAK 1: Static collection tích lũy data
public class EventTracker {
    private static final List<Event> events = new ArrayList<>();  // ← Không bao giờ được clear!

    public void track(Event e) {
        events.add(e);  // Tích lũy mãi mãi → OOM
    }
}

// LEAK 2: ThreadLocal không được remove
public class RequestContext {
    private static ThreadLocal<User> currentUser = new ThreadLocal<>();

    public void set(User user) {
        currentUser.set(user);
        // Nếu không gọi remove() → thread pool giữ reference mãi mãi!
    }

    // FIX: Dùng OncePerRequestFilter để đảm bảo cleanup
    // hoặc always call currentUser.remove() sau khi dùng xong
}

// LEAK 3: Event listener không được unregister
@Component
public class MyListener implements ApplicationListener<SomeEvent> {
    // Nếu listener được đăng ký động nhưng không unregister
    // Spring giữ reference → object không được GC
}
```

---

## 7. Continuous Profiling (Profiling Liên Tục) Với Grafana Pyroscope

```yaml
# Tích hợp Pyroscope với Spring Boot — continuous profiling
# pom.xml
<dependency>
    <groupId>io.pyroscope</groupId>
    <artifactId>agent</artifactId>
    <version>0.12.0</version>
</dependency>
```

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        // Khởi động Pyroscope agent
        PyroscopeAgent.start(new Config.Builder()
            .setApplicationName("my-spring-app")
            .setProfilingEvent(EventType.ITIMER)
            .setServerAddress("http://pyroscope:4040")
            .build()
        );

        SpringApplication.run(Application.class, args);
    }
}
```

---

## 8. Workflow Debug Performance Issue

```
1. SYMPTOM DETECTION (Phát Hiện Triệu Chứng)
   └─ Grafana alert: P99 latency tăng từ 50ms → 500ms
   └─ Actuator: CPU usage tăng 80%

2. QUICK DIAGNOSIS (Chẩn Đoán Nhanh)
   └─ Thread dump: Nhiều threads ở state BLOCKED → lock contention?
   └─ Actuator metrics: hikaricp.connections.pending > 0 → pool exhaustion?
   └─ Slow query log: Queries > 500ms → missing index?

3. PROFILE (Phân Tích Sâu)
   └─ async-profiler 60s → flame graph
   └─ JFR recording → JMC analysis
   └─ Heap dump nếu memory tăng

4. IDENTIFY ROOT CAUSE (Tìm Nguyên Nhân Gốc)
   └─ Flame graph: HashMap.get() chiếm 40% CPU
   → Điều tra: Collection size bao nhiêu? Có dùng đúng equals/hashCode không?

5. FIX (Sửa)
   └─ Targeted fix — chỉ sửa chỗ được xác định qua profiling

6. VALIDATE (Xác Nhận)
   └─ Load test trước/sau để so sánh metrics
   └─ Profile lại để xác nhận hotspot đã giảm
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa sampling và instrumentation profiler?**

```
Sampling (Lấy Mẫu):
  - Overhead thấp (~1%), dùng được production
  - Thống kê xác suất — không 100% chính xác
  - Công cụ: async-profiler, JFR

Instrumentation (Gắn Công Cụ):
  - Overhead cao (10–50%), chỉ dùng development
  - Chính xác tuyệt đối mọi method call
  - Công cụ: JProfiler, YourKit
```

**Q: Khi nào dùng heap dump vs thread dump?**

```
Thread dump (Ảnh Chụp Luồng):
  → Khi ứng dụng bị treo (hanging)
  → Khi response time cao bất thường
  → Khi nghi ngờ deadlock hay lock contention
  → Overhead: Gần như zero

Heap dump (Ảnh Chụp Heap):
  → Khi memory usage tăng liên tục (memory leak)
  → Khi OOM xảy ra
  → Overhead: Dừng ứng dụng vài giây, file rất lớn (GB)
  → Chỉ dùng khi cần thiết
```

---

## ✅ Checklist

- [ ] async-profiler hoặc JFR có thể chạy trong staging environment
- [ ] JFR continuous recording được cấu hình với circular buffer cho production
- [ ] Spring Boot Actuator `/threaddump` và `/heapdump` endpoints được cấu hình
- [ ] `-XX:+HeapDumpOnOutOfMemoryError` được bật với path ghi được
- [ ] Custom metrics được thêm cho business-critical operations
- [ ] Grafana dashboard hiển thị key performance metrics
- [ ] Quy trình debug performance issue đã được document

---

**Xem tiếp:** [6-load-testing.md](6-load-testing.md) — Load Testing với Gatling & k6
