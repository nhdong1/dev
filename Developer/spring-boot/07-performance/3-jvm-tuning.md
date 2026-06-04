# JVM Tuning — Tinh Chỉnh JVM & Garbage Collector

> JVM (Java Virtual Machine — Máy Ảo Java) quản lý bộ nhớ tự động qua GC (Garbage Collector
> — Bộ Thu Gom Rác). Cấu hình sai có thể gây GC pauses (Tạm Dừng GC) dài, OOM
> (OutOfMemoryError — Lỗi Tràn Bộ Nhớ), hoặc hiệu năng tệ. Đây là kỹ năng thiết yếu
> cho Senior Backend Developer.

---

## 📋 Mục Tiêu

- [ ] Hiểu kiến trúc **JVM Heap** (Vùng Nhớ Heap) — Young/Old Generation
- [ ] Nắm rõ sự khác biệt giữa **G1GC**, **ZGC**, và **Shenandoah**
- [ ] Cấu hình **heap size** phù hợp cho container
- [ ] Chọn và tune (Tinh Chỉnh) GC flags đúng cho workload
- [ ] Phân tích **GC logs** để phát hiện vấn đề
- [ ] Xử lý **OutOfMemoryError** và **memory leak** (Rò Rỉ Bộ Nhớ)

---

## 1. JVM Memory Layout (Bố Cục Bộ Nhớ JVM)

```
┌────────────────────────────────────────────────────────┐
│                    JVM Process                         │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │                     HEAP                        │  │
│  │                                                  │  │
│  │  ┌─────────────────┐    ┌────────────────────┐  │  │
│  │  │  Young Gen      │    │     Old Gen        │  │  │
│  │  │  (Thế Hệ Trẻ)  │    │  (Thế Hệ Già)     │  │  │
│  │  │                 │    │                    │  │  │
│  │  │ ┌────┐ ┌──────┐ │    │  Objects sống lâu │  │  │
│  │  │ │Eden│ │Surv. │ │───▶│  (Long-lived)     │  │  │
│  │  │ └────┘ └──────┘ │    │                    │  │  │
│  │  │  Đa số objects  │    │   ~70% heap size   │  │  │
│  │  │  chết ở đây     │    └────────────────────┘  │  │
│  │  └─────────────────┘                            │  │
│  │           ~30% heap size                        │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌───────────────┐    ┌──────────────────────────────┐  │
│  │  Metaspace    │    │         Stack (Ngăn Xếp)     │  │
│  │  (Class meta  │    │  Thread 1 | Thread 2 | ...   │  │
│  │   data)       │    │  frames, local vars           │  │
│  └───────────────┘    └──────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### Các Loại GC

```
Minor GC (GC Nhỏ):
  - Chỉ collect Young Generation
  - Nhanh (~10–50ms)
  - Xảy ra thường xuyên

Major GC / Full GC (GC Lớn):
  - Collect cả Heap (Young + Old)
  - Chậm (~100ms – vài giây)
  - STOP-THE-WORLD: toàn bộ threads dừng lại!
  - Cần minimize (giảm thiểu) tần suất

Concurrent GC (GC Đồng Thời):
  - G1GC, ZGC, Shenandoah
  - Chạy song song với application threads
  - Giảm thiểu stop-the-world pause time
```

---

## 2. Các Garbage Collector Chính

### 2.1 G1GC (Garbage-First GC) — Mặc Định từ Java 9+

```
Đặc điểm:
  - Chia heap thành nhiều "regions" (vùng) nhỏ bằng nhau (~1–32MB mỗi vùng)
  - Ưu tiên collect vùng có nhiều rác nhất trước (Garbage-First)
  - Balance giữa throughput (thông lượng) và latency (độ trễ)
  - Mục tiêu pause time: 200ms theo mặc định

Phù hợp với:
  ✓ Ứng dụng web API đa mục đích
  ✓ Heap từ 4GB đến 32GB
  ✓ Cần balance throughput và latency
  ✓ Đây là GC phổ biến nhất cho Spring Boot production
```

### 2.2 ZGC (Z Garbage Collector) — Java 15+ GA

```
Đặc điểm:
  - Concurrent (Đồng Thời) — hầu hết công việc chạy song song với app
  - Pause time < 1ms (cực kỳ thấp!)
  - Scalable lên hàng TB heap
  - Throughput thấp hơn G1GC ~5–15%

Phù hợp với:
  ✓ Ứng dụng nhạy cảm với latency (real-time, trading)
  ✓ Heap rất lớn (> 32GB)
  ✓ Khi GC pause > 200ms là không chấp nhận được
  ✗ Không phù hợp nếu cần maximize throughput
```

### 2.3 Shenandoah GC — Java 12+ (Red Hat)

```
Đặc điểm:
  - Tương tự ZGC — pause time rất thấp
  - Concurrent compaction (Nén Đồng Thời) — giảm heap fragmentation
  - Không có trong Oracle JDK (chỉ OpenJDK)

Phù hợp với:
  ✓ Tương tự ZGC
  ✓ OpenJDK deployments
```

### 2.4 So Sánh

```
┌─────────────────────┬────────────┬──────────────┬────────────┐
│ Tiêu Chí            │  G1GC      │   ZGC        │ Shenandoah │
├─────────────────────┼────────────┼──────────────┼────────────┤
│ Throughput          │ ⭐⭐⭐      │ ⭐⭐          │ ⭐⭐        │
│ Max Pause Time      │ ~50–200ms  │ < 1ms        │ < 10ms     │
│ Heap Size Support   │ 4GB–32GB   │ 8MB – 16TB   │ Any        │
│ Java Version        │ Java 9+    │ Java 15+ GA  │ Java 12+   │
│ Memory Overhead     │ ~10%       │ ~15–20%      │ ~5–10%     │
│ Phù Hợp Nhất        │ API thông  │ Low-latency  │ OpenJDK    │
│                     │ thường     │ systems      │ deployments│
└─────────────────────┴────────────┴──────────────┴────────────┘
```

---

## 3. Heap Sizing (Cấu Hình Kích Thước Heap)

### Các JVM Flags Quan Trọng

```bash
# Heap size
-Xms<size>          # Initial heap size (Kích Thước Heap Ban Đầu)
-Xmx<size>          # Maximum heap size (Kích Thước Heap Tối Đa)

# Khuyến nghị: -Xms = -Xmx để tránh heap resizing (Thay Đổi Kích Thước Heap)
# Heap resizing tốn CPU và gây GC overhead

# Ví dụ:
-Xms512m -Xmx512m   # Heap 512MB cố định
-Xms2g -Xmx2g       # Heap 2GB cố định
```

### Sizing Cho Container (K8s / Docker)

```
Quy tắc: heap size ≈ 75% container memory limit

Container memory limit: 1GB → -Xmx768m
Container memory limit: 2GB → -Xmx1536m
Container memory limit: 4GB → -Xmx3g

Lý do để lại 25%:
  - JVM off-heap memory (Metaspace, code cache, thread stacks)
  - Native libraries
  - OS overhead
```

```yaml
# K8s deployment.yaml — cấu hình đúng cách
apiVersion: apps/v1
kind: Deployment
spec:
  containers:
    - name: app
      image: my-spring-boot-app
      resources:
        requests:
          memory: "512Mi"
          cpu: "500m"
        limits:
          memory: "1Gi"       # Giới Hạn Bộ Nhớ Container
          cpu: "1000m"
      env:
        - name: JAVA_OPTS
          value: >-
            -Xms384m
            -Xmx768m
            -XX:+UseG1GC
            -XX:MaxGCPauseMillis=200
```

### Nguy Hiểm Của `-Xmx` Quá Lớn Trong Container

```
Container memory limit: 1GB
JAVA_OPTS: -Xmx1g  ← SAI!

JVM heap: 1GB
+ JVM overhead (Metaspace, threads, ...): ~256MB
= Tổng: ~1.25GB > 1GB container limit

Kết quả: Container bị OOMKilled (K8s kill process)
         Không phải OutOfMemoryError — là container restart đột ngột!
```

---

## 4. G1GC Tuning

### Flags G1GC Cơ Bản

```bash
# Chọn G1GC (mặc định từ Java 9+)
-XX:+UseG1GC

# Mục tiêu pause time (Thời Gian Dừng Mục Tiêu) — mặc định 200ms
-XX:MaxGCPauseMillis=200

# Số luồng GC (tối ưu theo số CPU)
-XX:ParallelGCThreads=4        # GC threads khi stop-the-world
-XX:ConcGCThreads=2            # Concurrent GC threads

# Region size (Kích Thước Vùng) — từ 1MB đến 32MB
-XX:G1HeapRegionSize=4m        # Thường để JVM tự chọn

# Tỷ lệ Mixed GC (GC Hỗn Hợp)
-XX:G1MixedGCCountTarget=8     # Số Mixed GC cycles trước khi full collect
```

### Cấu Hình G1GC Khuyến Nghị

```bash
# Spring Boot production với G1GC
JAVA_OPTS="\
  -Xms1g -Xmx1g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+G1UseAdaptiveIHOP \
  -XX:InitiatingHeapOccupancyPercent=45 \
  -XX:+UseStringDeduplication \
  -XX:+ParallelRefProcEnabled"

# Giải thích:
# G1UseAdaptiveIHOP: Tự động điều chỉnh ngưỡng bắt đầu GC concurrent
# InitiatingHeapOccupancyPercent=45: Bắt đầu concurrent GC khi heap đầy 45%
# UseStringDeduplication: Gộp các String trùng lặp (tiết kiệm RAM)
# ParallelRefProcEnabled: Xử lý references song song (nhanh hơn)
```

---

## 5. ZGC Tuning

```bash
# Chọn ZGC
-XX:+UseZGC

# Soft max heap (Heap Mềm Tối Đa) — ZGC cố gắng không vượt quá giá trị này
-XX:SoftMaxHeapSize=2g -Xmx4g   # Mềm: 2GB, cứng: 4GB

# Uncommit unused memory (Giải Phóng Bộ Nhớ Không Dùng) — từ Java 16+
-XX:+ZUncommit

# Minimum uncommit delay
-XX:ZUncommitDelay=300          # 5 phút sau khi không dùng mới giải phóng
```

```bash
# ZGC configuration cho low-latency service
JAVA_OPTS="\
  -Xms2g -Xmx4g \
  -XX:+UseZGC \
  -XX:SoftMaxHeapSize=3g \
  -XX:+ZUncommit \
  -XX:ZUncommitDelay=300"
```

---

## 6. GC Logging (Ghi Log GC)

### Bật GC Logging

```bash
# Java 9+ — Unified Logging (Ghi Log Thống Nhất)
JAVA_OPTS="\
  -Xlog:gc*:file=/app/logs/gc.log:time,level,tags:filecount=5,filesize=10m"

# Giải thích:
# gc*: Log tất cả thông tin GC
# file=/app/logs/gc.log: Ghi vào file
# time,level,tags: Format log
# filecount=5: Giữ 5 file log gần nhất (log rotation)
# filesize=10m: Mỗi file tối đa 10MB
```

### Đọc GC Log

```
Ví dụ GC log output:
[0.123s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 64M->15M(256M) 12.345ms
[1.456s][info][gc] GC(1) Pause Young (Normal) (G1 Evacuation Pause) 80M->20M(256M) 8.901ms
[15.789s][info][gc] GC(5) Pause Full (Ergonomics) 200M->150M(256M) 456.789ms ← CẢNH BÁO!

Đọc hiểu:
  GC(0)          — GC lần thứ 0
  Pause Young    — Minor GC (Thế Hệ Trẻ)
  64M->15M(256M) — Before->After(Total heap) — Thu gom được 49MB, tổng heap 256MB
  12.345ms       — Pause time (thời gian dừng)

  Full GC 456ms  — Pause time quá dài! Cần investigate
```

### Công Cụ Phân Tích GC Log

```
1. GCEasy (gcease.com) — Web tool, upload GC log, phân tích tự động
2. GCViewer — Desktop app, open source
3. JVM GC Analyzer trong IntelliJ IDEA Profiler

Chỉ Số Cần Chú Ý:
  - GC frequency (Tần Suất GC): < 1 Full GC/phút là tốt
  - Max pause time: < 200ms cho G1GC
  - Heap after GC: Nếu luôn > 80% sau GC → heap quá nhỏ, cần tăng -Xmx
  - Allocation rate (Tốc Độ Cấp Phát): Nếu quá cao → object allocation issue
```

---

## 7. OutOfMemoryError — Phân Tích và Fix

### Các Loại OOM Phổ Biến

```
java.lang.OutOfMemoryError: Java heap space
  → Heap đầy, GC không thu gom đủ
  → Có thể do memory leak hoặc heap quá nhỏ

java.lang.OutOfMemoryError: GC overhead limit exceeded
  → JVM dành > 98% thời gian cho GC mà chỉ thu gom < 2% heap
  → Heap quá nhỏ hoặc memory leak nghiêm trọng

java.lang.OutOfMemoryError: Metaspace
  → Metaspace (Class metadata) đầy
  → Thường do class loader leak, quá nhiều dynamic class generation

java.lang.OutOfMemoryError: unable to create new native thread
  → Quá nhiều threads, vượt quá giới hạn OS
  → Kiểm tra thread pool config
```

### Lấy Heap Dump (Ảnh Chụp Heap) Khi OOM

```bash
# Tự động lấy heap dump khi OOM
JAVA_OPTS="\
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/dumps/heap_dump.hprof"

# Lấy heap dump thủ công (không restart)
jmap -dump:format=b,file=heap.hprof <PID>

# Hoặc với jcmd (JVM Command)
jcmd <PID> GC.heap_dump /app/dumps/heap.hprof
```

### Phân Tích Heap Dump

```
1. Mở bằng Eclipse MAT (Memory Analyzer Tool)
2. Dùng VisualVM hoặc JProfiler
3. Tìm: "Leak Suspects" report (Báo Cáo Nghi Vấn Rò Rỉ)

Nguyên nhân phổ biến:
  - List/Map static fields tích lũy dữ liệu không giới hạn
  - Event listeners không được unregister
  - ThreadLocal values không được remove
  - Cache không có giới hạn kích thước
  - Session objects tích lũy trong memory
```

---

## 8. Thực Hành: JVM Flags Đầy Đủ Cho Production

### Spring Boot API (General Purpose)

```bash
JAVA_OPTS="\
  # Heap
  -Xms1g -Xmx1g \
  \
  # GC — G1GC cho API thông thường
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+G1UseAdaptiveIHOP \
  -XX:InitiatingHeapOccupancyPercent=45 \
  \
  # Performance
  -XX:+UseStringDeduplication \
  -XX:+ParallelRefProcEnabled \
  \
  # OOM safety
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/dumps/ \
  \
  # GC Logging
  -Xlog:gc*:file=/app/logs/gc.log:time,level,tags:filecount=5,filesize=10m \
  \
  # Container awareness (Java 10+)
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0"
```

### Low-Latency Service (ZGC)

```bash
JAVA_OPTS="\
  -Xms2g -Xmx4g \
  -XX:+UseZGC \
  -XX:SoftMaxHeapSize=3g \
  -XX:+ZUncommit \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/app/dumps/ \
  -Xlog:gc*:file=/app/logs/gc.log:time,level,tags:filecount=5,filesize=10m"
```

### Container Với `MaxRAMPercentage`

```bash
# Java 11+ — tự động tính heap dựa trên container memory
JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:InitialRAMPercentage=50.0 \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200"

# Container 1GB → heap = 768MB tự động
# Container 2GB → heap = 1.5GB tự động
# Không cần hardcode -Xmx — phù hợp với nhiều môi trường
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Khi nào chuyển từ G1GC sang ZGC?**

```
Chuyển sang ZGC khi:
  1. GC pause time > 200ms gây ra SLA violation (Vi Phạm Thỏa Thuận Dịch Vụ)
  2. Ứng dụng xử lý real-time (trading, gaming, media streaming)
  3. Heap size > 16GB
  4. P99 latency requirements rất nghiêm ngặt (< 10ms)

Giữ G1GC khi:
  1. GC pause time hiện tại chấp nhận được
  2. Cần maximize throughput
  3. Heap < 8GB
  4. Java < 15 (ZGC chưa GA)
```

**Q: Tại sao không nên đặt `-Xms` khác `-Xmx`?**

```
Khi -Xms < -Xmx:
  - JVM bắt đầu với heap nhỏ
  - Khi cần thêm memory → JVM phải expand heap (Mở Rộng Heap)
  - Heap expansion = Full GC → pause time đột ngột tăng
  - Unpredictable performance (Hiệu Năng Không Đoán Trước)

Khuyến nghị production: -Xms = -Xmx để heap size cố định
```

**Q: `-XX:+UseContainerSupport` làm gì?**

```
Trước Java 10: JVM không biết đang chạy trong container
  → Đọc total RAM của host (host có 64GB RAM)
  → -XX:MaxRAMFraction=4 → JVM dùng 16GB heap
  → Container chỉ có limit 2GB → OOMKilled!

Từ Java 10+: -XX:+UseContainerSupport (mặc định bật)
  → JVM đọc container cgroup memory limits
  → Heap sizing dựa trên container limit, không phải host RAM
  → An toàn trong K8s/Docker
```

---

## ✅ Checklist

- [ ] `-Xms` = `-Xmx` để tránh heap resizing
- [ ] Heap size ≤ 75% container memory limit
- [ ] GC được chọn phù hợp (G1GC cho general, ZGC cho low-latency)
- [ ] `-XX:MaxGCPauseMillis` phù hợp với SLA
- [ ] GC logging được bật với log rotation
- [ ] `-XX:+HeapDumpOnOutOfMemoryError` được cấu hình
- [ ] `-XX:+UseContainerSupport` được bật (Java 10+)
- [ ] JVM flags được version controlled và documented

---

**Xem tiếp:** [4-query-optimization.md](4-query-optimization.md) — Database Query Optimization
