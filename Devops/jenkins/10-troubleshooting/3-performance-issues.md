# Performance Issues — Xử Lý Vấn Đề Hiệu Suất Jenkins

> Hướng dẫn chẩn đoán và tối ưu hóa hiệu suất Jenkins: từ JVM heap tuning (tinh chỉnh heap JVM), GC pressure (áp lực Garbage Collection), thread dump analysis (phân tích thread dump), đến slow UI (giao diện chậm) và build queue (hàng đợi build).

## Mục Lục

1. [Dấu Hiệu Jenkins Có Vấn Đề Hiệu Suất](#dấu-hiệu-jenkins-có-vấn-đề-hiệu-suất)
2. [JVM Heap — Quản Lý Bộ Nhớ](#jvm-heap--quản-lý-bộ-nhớ)
3. [GC Pressure — Áp Lực Garbage Collection](#gc-pressure--áp-lực-garbage-collection)
4. [Thread Dump — Phân Tích Luồng Xử Lý](#thread-dump--phân-tích-luồng-xử-lý)
5. [Slow UI — Giao Diện Chậm](#slow-ui--giao-diện-chậm)
6. [Build Queue — Hàng Đợi Build Dài](#build-queue--hàng-đợi-build-dài)
7. [Disk I/O — Vấn Đề Ổ Đĩa](#disk-io--vấn-đề-ổ-đĩa)
8. [Tối Ưu Pipeline Để Tăng Hiệu Suất](#tối-ưu-pipeline-để-tăng-hiệu-suất)

---

## Dấu Hiệu Jenkins Có Vấn Đề Hiệu Suất

Nhận biết sớm trước khi Jenkins ngừng hoạt động hoàn toàn:

| Triệu Chứng | Nguyên Nhân Có Thể | Mức Độ Nghiêm Trọng |
|-------------|-------------------|---------------------|
| UI phản hồi sau 5–30 giây | GC pause, CPU overload | Cao |
| Build ở queue hàng giờ | Thiếu executor, agent offline | Cao |
| `OutOfMemoryError` trong log | Heap quá nhỏ | Nghiêm trọng |
| CPU 100% liên tục | Thread deadlock hoặc infinite loop | Nghiêm trọng |
| Disk đầy, build fail | Thiếu chính sách dọn dẹp | Cao |
| Plugin timeout khi load | Nhiều plugin nặng hoặc classpath conflict | Trung bình |

---

## JVM Heap — Quản Lý Bộ Nhớ

### Xem Trạng Thái Heap Hiện Tại

```
Manage Jenkins → System Information
→ Tìm "java.vm.name", "java.runtime.version"

Manage Jenkins → Monitoring (nếu có plugin)
→ JVM metrics
```

Hoặc qua Script Console:

```groovy
// Manage Jenkins → Script Console
def runtime = Runtime.getRuntime()
def maxHeap = runtime.maxMemory() / 1024 / 1024
def totalHeap = runtime.totalMemory() / 1024 / 1024
def freeHeap = runtime.freeMemory() / 1024 / 1024
def usedHeap = totalHeap - freeHeap

println "Max Heap (giới hạn tối đa): ${maxHeap} MB"
println "Total Heap (đã cấp phát): ${totalHeap} MB"
println "Used Heap (đang dùng): ${usedHeap} MB"
println "Free Heap (còn trống): ${freeHeap} MB"
println "Heap Usage: ${(usedHeap / maxHeap * 100).round(1)}%"
```

### Cấu Hình JVM Heap

Jenkins chạy trên JVM (Java Virtual Machine — Máy Ảo Java). Heap là vùng nhớ JVM dùng để lưu objects.

**Khuyến nghị về kích thước Heap:**

| Số Lượng Job / Agent | Heap Tối Thiểu | Heap Khuyến Nghị |
|---------------------|----------------|------------------|
| < 50 job, < 5 agent | 512 MB | 1 GB |
| 50–200 job, 5–20 agent | 2 GB | 4 GB |
| 200+ job, 20+ agent | 4 GB | 8–16 GB |

**Cấu hình trên Linux (Standalone):**

```bash
# File: /etc/default/jenkins (Debian/Ubuntu)
# Hoặc: /etc/sysconfig/jenkins (RHEL/CentOS)

JAVA_OPTS="-Xms2g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ExplicitGCInvokesConcurrent \
  -Djava.awt.headless=true"

# -Xms: Initial heap size (kích thước heap ban đầu)
# -Xmx: Maximum heap size (kích thước heap tối đa)
# -XX:+UseG1GC: Dùng G1 Garbage Collector (khuyến nghị cho Jenkins)
# -XX:MaxGCPauseMillis: Mục tiêu pause time tối đa của GC (mili giây)
```

**Cấu hình trên Docker:**

```yaml
# docker-compose.yml
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    environment:
      JAVA_OPTS: >-
        -Xms2g
        -Xmx4g
        -XX:+UseG1GC
        -XX:MaxGCPauseMillis=200
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:HeapDumpPath=/var/jenkins_home/heapdump.hprof
    mem_limit: 6g  # Đặt limit Docker cao hơn heap để tránh OOM
```

**Cấu hình trên Kubernetes:**

```yaml
# Helm values hoặc deployment manifest
resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "8Gi"
    cpu: "4"

env:
- name: JAVA_OPTS
  value: >-
    -Xms2g
    -Xmx6g
    -XX:+UseG1GC
    -XX:MaxGCPauseMillis=200
    -XX:+ExplicitGCInvokesConcurrent
```

### OutOfMemoryError — Hết Bộ Nhớ

**Triệu chứng:**
```
java.lang.OutOfMemoryError: Java heap space
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

**Nguyên nhân phổ biến:**
1. Build history (lịch sử build) quá lớn, không có discard policy
2. Plugin bị memory leak (rò rỉ bộ nhớ)
3. Heap size quá nhỏ so với workload
4. Build log quá lớn (log hàng GB)

**Giải pháp:**

```groovy
// 1. Cấu hình Build Discard Policy cho tất cả job
// Manage Jenkins → Script Console

Jenkins.instance.getAllItems(Job.class).each { job ->
    def strategy = new hudson.tasks.LogRotator(
        30,    // daysToKeep: giữ build trong 30 ngày
        20,    // numToKeep: giữ tối đa 20 builds
        -1,    // artifactDaysToKeep: không giới hạn ngày cho artifact
        5      // artifactNumToKeep: giữ tối đa 5 builds có artifact
    )
    job.buildDiscarder = strategy
    job.save()
    println "Updated: ${job.name}"
}
```

```groovy
// 2. Trong Jenkinsfile — cấu hình per-pipeline
pipeline {
    options {
        buildDiscarder(logRotator(
            daysToKeepStr: '30',
            numToKeepStr: '20',
            artifactNumToKeepStr: '5'
        ))
    }
}
```

---

## GC Pressure — Áp Lực Garbage Collection

### GC (Garbage Collection — Thu Gom Rác) Là Gì?

GC là quá trình JVM tự động giải phóng bộ nhớ của các objects không còn được tham chiếu. Khi GC chạy quá thường xuyên hoặc mỗi lần chạy quá lâu (GC pause), Jenkins sẽ bị chậm hoặc ngừng phản hồi.

### Đọc GC Log

**Bật GC logging:**

```bash
JAVA_OPTS="-Xmx4g \
  -XX:+UseG1GC \
  -Xlog:gc*:file=/var/log/jenkins/gc.log:time,uptime:filecount=5,filesize=20m"
  # filecount=5: giữ 5 file log rotation
  # filesize=20m: mỗi file tối đa 20 MB
```

**Đọc GC log nhanh:**

```bash
# Xem tần suất GC
grep "GC pause" /var/log/jenkins/gc.log | tail -20

# Xem Full GC (nghiêm trọng nhất)
grep "Full GC" /var/log/jenkins/gc.log
# Full GC xảy ra thường xuyên = cần tăng heap hoặc đổi GC algorithm
```

**Dấu hiệu GC Pressure nghiêm trọng:**
- GC pause > 500ms liên tục
- Full GC xảy ra nhiều hơn 1 lần/phút
- Heap usage luôn > 85% sau mỗi GC cycle

### Chọn GC Algorithm Đúng

| GC Algorithm | Khi Nào Dùng | Cấu Hình |
|-------------|-------------|---------|
| **G1GC** (G1 Garbage Collector) | Jenkins với heap 2–32 GB | `-XX:+UseG1GC` |
| **ZGC** (Z Garbage Collector) | Heap lớn (>32 GB), cần low latency | `-XX:+UseZGC` |
| **Shenandoah** | Low latency, concurrent GC | `-XX:+UseShenandoahGC` |
| **Parallel GC** | Throughput tối đa, latency không quan trọng | `-XX:+UseParallelGC` |

**Khuyến nghị cho Jenkins:** Dùng **G1GC** với `-XX:MaxGCPauseMillis=200`.

```bash
# Cấu hình G1GC tối ưu cho Jenkins
JAVA_OPTS="-Xms2g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=16m \
  -XX:G1NewSizePercent=20 \
  -XX:G1MaxNewSizePercent=40 \
  -XX:+ExplicitGCInvokesConcurrent"
```

---

## Thread Dump — Phân Tích Luồng Xử Lý

### Thread Dump (Kết Xuất Luồng) Là Gì?

Thread dump là ảnh chụp tức thời tất cả các thread (luồng) đang chạy trong JVM tại một thời điểm. Nó giúp xác định:
- Thread nào đang bị **deadlock** (bế tắc — hai thread chờ nhau vô tận)
- Thread nào đang **block** chờ lock
- Thread nào đang **stuck** trong vòng lặp vô hạn

### Lấy Thread Dump

**Cách 1: Qua Jenkins UI**

```
Manage Jenkins → System Information → Threads
(Hoặc: {jenkins-url}/threadDump)
```

**Cách 2: Qua jstack (trên máy chủ)**

```bash
# Tìm PID của Jenkins process
pgrep -f "jenkins.war"
# Hoặc
ps aux | grep jenkins | grep -v grep

# Lấy thread dump
jstack <PID> > /tmp/jenkins-threaddump-$(date +%Y%m%d-%H%M%S).txt

# Lấy 3 thread dumps cách nhau 10 giây (để xem progression)
for i in 1 2 3; do
    jstack $(pgrep -f jenkins.war) > /tmp/threaddump-$i.txt
    sleep 10
done
```

**Cách 3: Qua Script Console**

```groovy
// Manage Jenkins → Script Console
// In tất cả threads với stack trace
Thread.getAllStackTraces().each { thread, stack ->
    println "=== Thread: ${thread.name} | State: ${thread.state} ==="
    stack.each { element ->
        println "  at ${element}"
    }
    println ""
}
```

### Đọc Thread Dump — Tìm Deadlock

```
Found 1 deadlock:
=================================
"Thread-1" waiting to lock <0x00000007d5e25ab8> (a java.lang.Object)
  which is held by "Thread-2"
"Thread-2" waiting to lock <0x00000007d5e25ab0> (a java.lang.Object)
  which is held by "Thread-1"
```

Khi phát hiện deadlock:
1. Ghi lại thông tin — đây thường là bug trong plugin
2. Restart Jenkins để giải phóng deadlock (giải pháp tạm thời)
3. Báo cáo bug lên Jenkins JIRA hoặc tác giả plugin

---

## Slow UI — Giao Diện Chậm

### Nguyên Nhân Và Giải Pháp

#### 1. Quá Nhiều Plugins Kích Hoạt

```
Manage Jenkins → Plugin Manager → Installed
→ Xem danh sách plugin đang dùng
```

Vô hiệu hóa plugin không dùng:
```
Manage Jenkins → Plugin Manager → Installed
→ Tìm plugin → Uncheck "Enabled" → Restart Jenkins
```

#### 2. Build History Quá Lớn

Jenkins load metadata của tất cả builds khi khởi động. Nhiều builds cũ = khởi động chậm và UI chậm.

```groovy
// Xóa build cũ qua Script Console
// Giữ lại 20 builds mới nhất cho mỗi job
Jenkins.instance.getAllItems(Job.class).each { job ->
    def builds = job.builds
    if (builds.size() > 20) {
        println "Cleaning ${job.name}: ${builds.size()} builds → keep 20"
        builds[20..-1].each { build ->
            build.delete()
        }
    }
}
println "Done."
```

#### 3. Jenkins Fingerprint Database Phình To

Fingerprint (dấu vân tay) là cơ chế Jenkins track file artifacts. Theo thời gian, database fingerprint có thể chiếm hàng GB.

```groovy
// Dọn dẹp orphaned fingerprints (dấu vân tay mồ côi)
// Manage Jenkins → Script Console
FingerprintCleanupThread.invoke()
println "Fingerprint cleanup triggered"
```

#### 4. Regex Chậm Trong View Filter

```
Manage Jenkins → Views
→ Xem Regular Expression filter
→ Tránh regex phức tạp như .* lồng nhau
```

### Kiểm Tra CPU Và Memory Trực Tiếp

```bash
# Xem Jenkins process consumption
top -p $(pgrep -f jenkins.war)

# Hoặc dùng htop với filter
htop -p $(pgrep -f jenkins.war)

# Xem chi tiết memory breakdown
cat /proc/$(pgrep -f jenkins.war)/status | grep -E "VmRSS|VmSwap|VmPeak"
```

---

## Build Queue — Hàng Đợi Build Dài

### Xem Tình Trạng Queue

```
Jenkins Dashboard → Build Queue (bên trái)
→ Xem "Why" column — lý do build chưa chạy được
```

Hoặc qua Script Console:

```groovy
// Xem tất cả items trong queue và lý do chờ
Jenkins.instance.queue.items.each { item ->
    println "Job: ${item.task.name}"
    println "  Why: ${item.why}"
    println "  In queue since: ${new Date(item.inQueueSince)}"
    println ""
}
```

### Lý Do Queue Phổ Biến Và Giải Pháp

| "Why" trong Queue | Nguyên Nhân | Giải Pháp |
|------------------|-------------|-----------|
| `Waiting for next available executor` | Tất cả executors bận | Thêm agent hoặc tăng số executor |
| `Waiting for agent [name]` | Agent cụ thể offline | Khởi động lại agent |
| `Waiting for agent matching [label]` | Không có agent nào có label này | Thêm agent với label phù hợp |
| `Build #N is already in progress` | Concurrent build bị tắt | Bật concurrent builds trong job config |
| `[Throttle] Waiting for builds` | Throttle Concurrent Builds Plugin đang giới hạn | Tăng limit hoặc điều chỉnh throttle rule |

### Tăng Số Executor

```
Manage Jenkins → Nodes → Built-In Node → Configure
→ Number of executors: tăng từ 2 lên 4 (hoặc 0 nếu không muốn build trên controller)
```

**Lưu ý:** Không nên để nhiều executors trên Controller node — Controller nên dùng để điều phối, không thực thi build.

```groovy
// Best practice: đặt executors=0 trên Controller
// Tất cả builds chạy trên Agent
// Manage Jenkins → Script Console
Jenkins.instance.setNumExecutors(0)
Jenkins.instance.save()
println "Controller executor count set to 0"
```

---

## Disk I/O — Vấn Đề Ổ Đĩa

### Kiểm Tra Disk Usage (Mức Dùng Ổ Đĩa)

```bash
# Xem tổng quan disk
df -h /var/lib/jenkins

# Thư mục nào chiếm nhiều nhất
du -sh /var/lib/jenkins/* | sort -rh | head -20

# Xem build logs lớn nhất
find /var/lib/jenkins/jobs -name "log" -size +100M -exec ls -lh {} \;
```

### JENKINS_HOME Cấu Trúc Và Nơi Tốn Disk

```
JENKINS_HOME/
├── jobs/                          ← Thường lớn nhất
│   └── my-job/
│       └── builds/
│           └── 1234/
│               ├── log            ← Build console log
│               └── archive/       ← Archived artifacts
├── workspace/                     ← Workspace của build
├── plugins/                       ← Plugin files
└── fingerprints/                  ← Fingerprint database
```

### Tự Động Dọn Dẹp Disk

```groovy
// Dọn dẹp workspace cho tất cả offline agents
// Manage Jenkins → Script Console
Jenkins.instance.nodes.each { node ->
    def computer = node.toComputer()
    if (!computer.online) {
        println "Skipping offline agent: ${node.name}"
        return
    }
    computer.workspaceList.each { workspaceDir ->
        println "Cleaning workspace: ${workspaceDir}"
        // Xóa workspace
        workspaceDir.deleteContents()
    }
}
```

```groovy
// Xóa build logs cũ hơn 90 ngày
// Manage Jenkins → Script Console
def cutoff = System.currentTimeMillis() - (90 * 24 * 60 * 60 * 1000L)  // 90 ngày

Jenkins.instance.getAllItems(Job.class).each { job ->
    job.builds.each { build ->
        if (build.timeInMillis < cutoff) {
            println "Deleting old build: ${job.name} #${build.number} (${new Date(build.timeInMillis)})"
            build.delete()
        }
    }
}
println "Cleanup complete."
```

---

## Tối Ưu Pipeline Để Tăng Hiệu Suất

### Parallel Stages — Chạy Song Song

```groovy
// Tăng tốc build bằng chạy các stage song song
pipeline {
    stages {
        stage('Tests') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'mvn test -Punit' }
                }
                stage('Integration Tests') {
                    steps { sh 'mvn test -Pintegration' }
                }
                stage('Security Scan') {
                    steps { sh 'trivy fs .' }
                }
            }
        }
    }
}
```

### Giảm Checkout Time

```groovy
// Shallow clone (chỉ lấy commit mới nhất) — nhanh hơn nhiều cho repo lớn
checkout([
    $class: 'GitSCM',
    branches: [[name: env.GIT_BRANCH]],
    extensions: [
        [$class: 'CloneOption',
         depth: 1,        // Chỉ lấy 1 commit
         shallow: true,
         noTags: true]    // Không lấy tags — tiết kiệm bandwidth
    ],
    userRemoteConfigs: [[url: env.GIT_URL, credentialsId: 'git-creds']]
])
```

### Maven Build Cache

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            // Mount Maven repository cache — tránh download lại dependencies mỗi build
            args '-v /var/jenkins_home/.m2:/root/.m2:rw'
        }
    }
}
```

---

**Xem Tiếp:** [4-plugin-conflicts.md](4-plugin-conflicts.md) — xử lý xung đột plugin
