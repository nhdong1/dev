# EC2 Instance Types — Loại Máy Chủ Ảo

> **Instance Type** định nghĩa phần cứng ảo của EC2: số vCPU (CPU ảo), RAM, băng thông mạng, và loại lưu trữ. Chọn đúng instance type ảnh hưởng trực tiếp đến hiệu năng và chi phí.

## 📚 Mục Lục

1. [Quy Tắc Đặt Tên](#quy-tắc-đặt-tên)
2. [Các Instance Family](#các-instance-family)
3. [Thế Hệ Instance](#thế-hệ-instance)
4. [So Sánh Chi Tiết Từng Family](#so-sánh-chi-tiết-từng-family)
5. [Hướng Dẫn Chọn Instance Type](#hướng-dẫn-chọn-instance-type)
6. [Burstable Instances T-Series](#burstable-instances-t-series)
7. [Graviton — ARM-based Instances](#graviton--arm-based-instances)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Quy Tắc Đặt Tên

```
    m   7   g   .   2   x   large
    │   │   │       │   │     │
    │   │   │       │   │     └── Size: nano, micro, small, medium,
    │   │   │       │   │         large, xlarge, 2xlarge, ...48xlarge
    │   │   │       │   │
    │   │   │       │   └── x = extra capacity (ít dùng)
    │   │   │       │
    │   │   │       └── (dấu chấm phân cách)
    │   │   │
    │   │   └── Processor attribute:
    │   │       g = AWS Graviton (ARM)
    │   │       a = AMD
    │   │       i = Intel
    │   │       (trống) = Intel mặc định (thế hệ cũ)
    │   │
    │   └── Generation (thế hệ): 7 = thế hệ 7 (mới nhất)
    │
    └── Family (dòng): m = general purpose
```

### Ví Dụ Thực Tế

| Instance Type  | Giải Thích                                              |
| -------------- | ------------------------------------------------------- |
| `t3.micro`     | Burstable general, thế hệ 3, size micro                 |
| `m7g.large`    | General purpose, thế hệ 7, Graviton (ARM), size large  |
| `c6i.2xlarge`  | Compute optimized, thế hệ 6, Intel, 2xlarge            |
| `r6a.4xlarge`  | Memory optimized, thế hệ 6, AMD, 4xlarge               |
| `p4d.24xlarge` | GPU accelerated, thế hệ 4, deep learning, 24xlarge     |

---

## Các Instance Family

### Bảng Tổng Quan Nhanh

| Family | Loại               | Tỉ Lệ vCPU:RAM | Use Case Điển Hình                   |
| ------ | ------------------ | --------------- | ------------------------------------ |
| **T**  | Burstable General  | 1:2             | Dev/test, web nhỏ, microservices     |
| **M**  | General Purpose    | 1:4             | App server, backend API, database nhỏ |
| **C**  | Compute Optimized  | 1:2             | Web server nhiều, game server, HPC   |
| **R**  | Memory Optimized   | 1:8             | Database, cache (Redis), Spark       |
| **X**  | Memory Extreme     | 1:16+           | SAP HANA, in-memory database lớn     |
| **I**  | Storage Optimized (NVMe) | 1:4    | NoSQL DB, data warehouse, log processing |
| **D**  | Dense Storage (HDD) | 1:3            | Hadoop, HDFS, data lake              |
| **H**  | High Disk Throughput | 1:4           | Distributed file system, MapReduce   |
| **P**  | GPU — General      | varies          | Machine learning training, AI        |
| **G**  | GPU — Graphics     | varies          | Video rendering, ML inference        |
| **Inf**| Inferentia (AWS AI Chip) | varies | ML inference, cost-efficient AI      |
| **Trn**| Trainium (AWS AI Chip) | varies   | ML training, LLM fine-tuning         |
| **F**  | FPGA               | varies          | Financial analytics, genomics        |
| **U**  | High Memory (bare metal) | 1:32+  | SAP HANA scale-up, OLAP              |
| **VT** | Video Transcoding  | varies          | Real-time video processing           |
| **Mac**| macOS (Apple silicon) | varies      | iOS/macOS app development, CI/CD     |

---

## Thế Hệ Instance

AWS liên tục ra thế hệ mới. **Luôn ưu tiên thế hệ mới nhất** vì:
- Hiệu năng tốt hơn (20-40% so với thế hệ trước)
- Chi phí thấp hơn (hoặc ngang bằng)
- Bảo mật tốt hơn (Nitro System)

```
Thế hệ hiện tại (2025-2026):
  m7i, m7g, m7a  ← Mới nhất General Purpose
  c7i, c7g, c7a  ← Mới nhất Compute Optimized
  r7i, r7g, r7a  ← Mới nhất Memory Optimized
  
Vẫn phổ biến:
  m6i, m6g, c6i, r6i ← Ổn định, nhiều AZ hỗ trợ

Tránh dùng:
  m4, c4, r4    ← Thế hệ cũ, kém hiệu năng, sắp deprecated
```

### Nitro System — Nền Tảng Hypervisor Mới

**Nitro System** là hypervisor (phần mềm ảo hóa) thế hệ mới của AWS áp dụng từ thế hệ 5+:

- **Hiệu năng gần bare metal** — Offload network/storage sang hardware riêng
- **Bảo mật tốt hơn** — Isolated firmware, không có quyền truy cập từ AWS
- **Network bandwidth cao hơn** — Lên đến 200 Gbps (trên `u-` và `p4de` instances)
- **EBS-optimized mặc định** — Không cần chọn riêng như thế hệ cũ

---

## So Sánh Chi Tiết Từng Family

### General Purpose — T và M Series

#### T Series — Burstable Performance (Hiệu Năng Co Giãn)

T instances dùng mô hình **CPU Credits** (Tín Chỉ CPU):

```
CPU Credit hoạt động:
  - Khi CPU < baseline: Tích lũy credits
  - Khi CPU > baseline: Tiêu thụ credits
  - Hết credits: CPU bị throttle về baseline

Baseline CPU:
  t3.micro  → 10% của 2 vCPU
  t3.small  → 20% của 2 vCPU
  t3.medium → 20% của 2 vCPU
  t3.large  → 30% của 2 vCPU
```

**T Unlimited Mode** — Cho phép burst vô hạn nhưng tính phí thêm khi hết credits.

| Loại          | vCPU | RAM   | Dùng Khi                            |
| ------------- | ---- | ----- | ----------------------------------- |
| t3.nano       | 2    | 0.5GB | Dev/test tối giản                   |
| t3.micro      | 2    | 1GB   | Lab, static site nhỏ                |
| t3.small      | 2    | 2GB   | Internal tool, low traffic API      |
| t3.medium     | 2    | 4GB   | Small web app, dev environment      |
| t3.large      | 2    | 8GB   | Medium web app, staging             |
| t3.xlarge     | 4    | 16GB  | Medium production (traffic ổn định) |
| t3.2xlarge    | 8    | 32GB  | Production web với traffic đều       |

> **Khi nào KHÔNG dùng T series:** CPU-intensive workload liên tục (web server tải cao), database production, hoặc bất cứ thứ gì cần CPU ổn định 100%.

#### M Series — General Purpose Cân Bằng

Tỉ lệ vCPU:RAM = 1:4. Lựa chọn an toàn khi không chắc chắn.

| Loại       | vCPU | RAM    | Dùng Khi                                 |
| ---------- | ---- | ------ | ---------------------------------------- |
| m7g.medium | 1    | 4GB    | Small app server (ARM binary compatible) |
| m7g.large  | 2    | 8GB    | Backend API nhỏ-vừa                      |
| m7g.xlarge | 4    | 16GB   | App server vừa                           |
| m7g.2xlarge| 8    | 32GB   | App server lớn, Spring Boot microservice |
| m7g.4xlarge| 16   | 64GB   | Multi-threaded web backend               |
| m7g.8xlarge| 32   | 128GB  | Large application cluster                |

### Compute Optimized — C Series

Tỉ lệ vCPU:RAM = 1:2. CPU nhiều hơn RAM, dùng cho:
- Web server nhiều request đồng thời (Nginx, HAProxy)
- Game server (CPU-intensive logic)
- Video encoding, scientific modeling
- High-performance computing (HPC — Tính Toán Hiệu Năng Cao)
- Machine learning inference (nhỏ)

```
c7g.large   → 2 vCPU, 4GB   (Graviton, tiết kiệm ~20%)
c7i.large   → 2 vCPU, 4GB   (Intel, compatibility tốt hơn)
c6a.large   → 2 vCPU, 4GB   (AMD, cost-effective)
```

### Memory Optimized — R, X, Z Series

#### R Series — Memory Optimized (RAM Tối Ưu)

Tỉ lệ vCPU:RAM = 1:8. Dùng cho:
- Database server (PostgreSQL, MySQL, Oracle)
- In-memory cache (Redis, Memcached lớn)
- Apache Spark với large datasets
- SAP HANA (phiên bản nhỏ-vừa)
- Real-time big data analytics

```
r7g.large    → 2 vCPU,   16GB   (baseline production DB)
r7g.2xlarge  → 8 vCPU,   64GB   (medium PostgreSQL)
r7g.4xlarge  → 16 vCPU, 128GB   (large DB hoặc Redis cluster)
r7g.8xlarge  → 32 vCPU, 256GB   (enterprise DB)
r7g.16xlarge → 64 vCPU, 512GB   (very large DB)
```

#### X Series — Extreme Memory

Tỉ lệ vCPU:RAM có thể đạt 1:32 hoặc hơn. Dùng cho:
- SAP HANA production
- In-memory database khổng lồ
- Real-time analytics trên dữ liệu petabyte

```
x2gd.medium  →   2 vCPU,   32GB  (SAP HANA dev)
x2gd.xlarge  →   4 vCPU,   64GB
x2gd.8xlarge →  32 vCPU,  512GB
x2gd.16xlarge→  64 vCPU, 1024GB (1TB RAM!)
```

### Storage Optimized — I, D, H Series

#### I Series — NVMe SSD (NVMe — Non-Volatile Memory Express — Giao Thức Lưu Trữ Nhanh)

Instance Store NVMe: **Latency microsecond, IOPS cực cao**.

```
i3en.large  →  2 vCPU,  16GB RAM, 1.25TB NVMe
i3en.xlarge →  4 vCPU,  32GB RAM, 2.5TB  NVMe
i3en.3xlarge → 12 vCPU, 96GB RAM, 7.5TB  NVMe
```

Dùng cho:
- NoSQL database (Cassandra, DynamoDB DAX)
- Elasticsearch/OpenSearch với large index
- Data warehouse (Redshift dense storage)
- OLTP (Online Transaction Processing) với I/O cao

> **Nhắc nhở:** Dữ liệu Instance Store **mất khi stop/terminate**. Luôn replicate sang EBS hoặc S3.

### GPU Instances — P và G Series

#### P Series — Training (Huấn Luyện ML)

```
p3.2xlarge   →  8 vCPU,  61GB RAM, 1x V100 GPU
p3.8xlarge   → 32 vCPU, 244GB RAM, 4x V100 GPU
p3.16xlarge  → 64 vCPU, 488GB RAM, 8x V100 GPU
p4d.24xlarge → 96 vCPU, 1152GB RAM, 8x A100 GPU  (cao cấp nhất)
```

#### G Series — Inference và Graphics

```
g4dn.xlarge  →  4 vCPU, 16GB RAM, 1x T4 GPU  (ML inference, cost-effective)
g5.xlarge    →  4 vCPU, 16GB RAM, 1x A10G GPU (phổ biến cho AI inference)
g5.12xlarge  → 48 vCPU, 192GB RAM, 4x A10G GPU
```

---

## Hướng Dẫn Chọn Instance Type

### Decision Tree — Cây Quyết Định

```
Workload của bạn là gì?
│
├── Web/API server
│   ├── Traffic đột biến, không đều → T3/T4g (burstable)
│   └── Traffic ổn định, cao        → C7g (compute) hoặc M7g (balanced)
│
├── Database
│   ├── Tự quản lý MySQL/PostgreSQL → R7g (memory optimized)
│   ├── NoSQL (Cassandra, MongoDB)  → I3en (NVMe storage)
│   └── In-memory (Redis lớn)      → R7g hoặc X2gd
│
├── Machine Learning
│   ├── Training                   → P4d (A100 GPU)
│   ├── Inference                  → G5 hoặc Inf2 (Inferentia)
│   └── Small model inference      → C7g (cost-effective)
│
├── Batch processing / HPC
│   ├── CPU-bound                  → C7g hoặc Hpc7g
│   └── Memory-bound               → R7g hoặc X2gd
│
└── Dev / Test
    └── Mọi thứ                    → T3/T4g (tiết kiệm nhất)
```

### Bảng Chọn Nhanh Theo Use Case

| Use Case                            | Instance Gợi Ý | Lý Do                              |
| ----------------------------------- | -------------- | ---------------------------------- |
| WordPress / Blog                    | t3.small       | Traffic không đều, tiết kiệm       |
| Node.js/Go API (vừa)                | t3.medium      | Balanced, burstable                |
| Spring Boot API (production)        | m7g.xlarge     | Ổn định, ARM tiết kiệm ~20%        |
| Nginx reverse proxy                 | c7g.large      | CPU-optimized, nhiều connections    |
| PostgreSQL production               | r7g.2xlarge    | RAM cao cho buffer pool             |
| Redis cache lớn                     | r7g.xlarge     | Fit toàn bộ dataset vào RAM        |
| Elasticsearch cluster node          | i3en.xlarge    | NVMe cho inverted index            |
| Jenkins CI server                   | m7g.2xlarge    | Balanced cho build jobs            |
| ML training (PyTorch)               | p3.2xlarge     | GPU V100                           |
| ML inference API                    | g5.xlarge      | A10G GPU, cost-effective           |

---

## Burstable Instances T-Series

### Cơ Chế CPU Credits Chi Tiết

```
Mỗi vCPU kiếm được credits/giờ dựa vào loại instance:

Instance     | vCPU | Credits/giờ | Baseline CPU
-------------|------|-------------|-------------
t3.nano      |   2  |      3      |   5% × 2 vCPU
t3.micro     |   2  |      6      |  10% × 2 vCPU
t3.small     |   2  |     12      |  20% × 2 vCPU
t3.medium    |   2  |     24      |  20% × 2 vCPU
t3.large     |   2  |     36      |  30% × 2 vCPU
t3.xlarge    |   4  |     48      |  40% × 4 vCPU
t3.2xlarge   |   8  |     96      |  40% × 8 vCPU

Max credit balance (giới hạn tích lũy):
  t3.nano    → 72 credits  (~24h burst)
  t3.micro   → 144 credits (~24h burst)
  t3.small   → 288 credits (~24h burst)
```

### T Unlimited Mode

Khi bật **T Unlimited**:
- Burst vượt credit balance được phép
- Tính phí $0.05/vCPU-hour cho mỗi giờ burst vượt
- Phù hợp khi workload có spike không thường xuyên nhưng cần hiệu năng đầy đủ

```bash
# Bật T Unlimited khi launch
aws ec2 run-instances \
  --instance-type t3.medium \
  --credit-specification CpuCredits=unlimited \
  ...

# Thay đổi sau khi đã chạy
aws ec2 modify-instance-credit-specification \
  --instance-credit-specifications InstanceId=i-xxxxx,CpuCredits=unlimited
```

---

## Graviton — ARM-based Instances

**AWS Graviton** là bộ xử lý ARM 64-bit do AWS thiết kế riêng (dựa trên kiến trúc ARM Neoverse).

### So Sánh Graviton vs Intel/AMD

| Tiêu Chí           | Graviton 3 (g suffix) | Intel (i suffix) | AMD (a suffix) |
| ------------------ | --------------------- | ---------------- | -------------- |
| Chi phí            | ~20% rẻ hơn Intel     | Baseline         | ~10% rẻ hơn    |
| Hiệu năng/watt     | Tốt nhất              | Trung bình       | Tốt            |
| x86 compatibility  | Không (ARM binary)    | Có               | Có             |
| Java performance   | Tốt (JVM ARM64)       | Tốt              | Tốt            |
| Python/Go/Rust     | Tốt (build lại)       | Tốt              | Tốt            |
| Windows            | Không hỗ trợ          | Có               | Có             |
| .NET               | .NET 6+ hỗ trợ ARM64  | Tất cả           | Tất cả         |

### Khi Nào Dùng Graviton?

**Nên dùng Graviton:**
- Ứng dụng Linux (99% trường hợp)
- Java (JVM chạy ARM64 rất tốt từ JDK 11+)
- Go, Rust, Python, Node.js, PHP, Ruby
- Container workloads (Docker image ARM64)
- Tiết kiệm chi phí là ưu tiên

**Không dùng Graviton:**
- Windows workloads
- Software chỉ có binary x86_64 (không có ARM64 build)
- .NET Framework (chỉ chạy trên Windows)
- Một số ISV (Independent Software Vendor) software chưa hỗ trợ ARM64

```bash
# Kiểm tra AMI có hỗ trợ ARM64 không
aws ec2 describe-images \
  --filters "Name=architecture,Values=arm64" \
            "Name=name,Values=amzn2-ami-hvm*"
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao bạn chọn C instance thay vì M instance cho web server production?**
> C instances có tỉ lệ vCPU:RAM = 1:2 (M là 1:4), nghĩa là nhiều CPU hơn cho cùng chi phí. Web server chủ yếu xử lý request (CPU-bound), ít cần RAM (ứng dụng không cần buffer lớn). Ví dụ: `c7g.xlarge` cho 4 vCPU với 8GB RAM, trong khi `m7g.xlarge` chỉ 4 vCPU với 16GB RAM — ta trả tiền RAM không cần thiết với M.

**Q: Khi nào dùng T instance không phù hợp?**
> Khi workload cần CPU **liên tục và ổn định**: database production, video encoding, machine learning inference. T instances bị throttle về baseline CPU khi hết credits, gây latency spike bất ngờ. Cũng không dùng T cho load balancer production vì traffic đột biến tiêu hết credits nhanh.

**Q: Sự khác biệt giữa Graviton và Intel instance trong thực tế?**
> Graviton (ARM64) rẻ hơn ~20% và hiệu năng/watt tốt hơn, nhưng cần binary ARM64. Hầu hết Linux container/app đều hỗ trợ ARM64 tốt. Vấn đề thực tế: một số closed-source software hoặc legacy binary chỉ có x86_64. Quy trình migration: test trên Graviton trước, đo hiệu năng và chi phí, sau đó rollout dần.

**Q: `m7g.2xlarge` hay `m6g.4xlarge` — cái nào tốt hơn?**
> So sánh: m7g.2xlarge (8 vCPU, 32GB) vs m6g.4xlarge (16 vCPU, 64GB). Tùy workload. m7g thế hệ 7 có kiến trúc Graviton 3 nhanh hơn ~25% mỗi core. Nếu workload cần nhiều core song song → m6g.4xlarge. Nếu cần ít core nhưng nhanh hơn → m7g.2xlarge. Về chi phí, hai instance này khá tương đương nhau, nên benchmark thực tế với workload cụ thể.

---

## Liên Kết

- [← README.md](./README.md)
- [Tiếp theo: AMI & Storage →](./2-ami-storage.md)
- [AWS Instance Types Documentation](https://aws.amazon.com/ec2/instance-types/)
- [EC2 Instance Comparison Tool](https://instances.vantage.sh/)

---

**Cập Nhật:** 2026-05-14 | **Trạng Thái:** ✅ Hoàn thành
