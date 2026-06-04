# 📋 Top 20 Câu Hỏi Phỏng Vấn AWS Compute — Kèm Đáp Án Chi Tiết

> Tổng hợp 20 câu hỏi được hỏi nhiều nhất trong phỏng vấn AWS Compute, bao gồm đáp án mẫu, từ khóa kỹ thuật, và mẹo trả lời tự tin.

## 📚 Mục Lục

- [Nhóm 1: EC2 & Auto Scaling (Câu 1-6)](#nhóm-1-ec2--auto-scaling)
- [Nhóm 2: Serverless Lambda (Câu 7-10)](#nhóm-2-serverless-lambda)
- [Nhóm 3: Containers ECS & EKS (Câu 11-15)](#nhóm-3-containers-ecs--eks)
- [Nhóm 4: High Availability & Cost (Câu 16-18)](#nhóm-4-high-availability--cost)
- [Nhóm 5: Trade-offs & Design (Câu 19-20)](#nhóm-5-trade-offs--design)

---

## Nhóm 1: EC2 & Auto Scaling

---

### Câu 1: Giải thích các loại EC2 Instance và khi nào chọn loại nào?

**Độ khó:** ⭐⭐ | **Tần suất xuất hiện:** Rất cao

**Đáp Án Mẫu:**

AWS phân loại EC2 instance thành các họ (instance families) dựa trên tỷ lệ CPU-RAM và phần cứng đặc biệt:

| Họ Instance     | Đặc Điểm                           | Use Case Điển Hình                          |
| --------------- | ---------------------------------- | ------------------------------------------- |
| **T-series**    | Burstable CPU (CPU bùng nổ ngắn hạn) | Dev/test, low-traffic web, microservices nhỏ |
| **M-series**    | General Purpose (Đa Mục Đích), cân bằng CPU-RAM | Application servers, backend APIs          |
| **C-series**    | Compute Optimized (Tối Ưu Tính Toán) | Batch processing, ML inference, game servers |
| **R-series**    | Memory Optimized (Tối Ưu Bộ Nhớ)  | In-memory cache (Redis), SAP HANA, ML training |
| **I-series**    | Storage Optimized (Tối Ưu Lưu Trữ) | NoSQL databases, data warehousing, OLAP     |
| **G/P-series**  | GPU Accelerated (Tăng Tốc GPU)     | Deep learning, video encoding, 3D rendering |
| **Inf-series**  | Inferentia (AWS chip tùy chỉnh)    | ML inference tiết kiệm chi phí đến 70%     |

**Ví dụ ra quyết định thực tế:**
- API backend thông thường → **m6i.large** (cân bằng tốt)
- Cache layer Redis → **r6g.xlarge** (nhiều RAM, ARM giá tốt)
- Video transcoding → **c6i.2xlarge** (CPU cao)
- ML training → **p3.8xlarge** (GPU Tesla V100)

**Từ Khóa Quan Trọng:** instance family, vCPU, memory-to-CPU ratio, burstable performance, baseline CPU credit

---

### Câu 2: Phân biệt On-Demand, Reserved Instance, Spot, và Savings Plans. Khi nào dùng loại nào?

**Độ khó:** ⭐⭐ | **Tần suất xuất hiện:** Rất cao

**Đáp Án Mẫu:**

**On-Demand (Theo Yêu Cầu):**
- Giá cao nhất, không cam kết
- Dùng cho: workload không dự đoán được, dev/test ngắn hạn, spike traffic

**Reserved Instances — RI (Phiên Bản Dự Phòng):**
- Giảm 40-72% so với On-Demand khi cam kết 1-3 năm
- Standard RI: cố định instance type, không linh hoạt
- Convertible RI: đổi được instance family, linh hoạt hơn
- Dùng cho: production workload ổn định, database servers

**Spot Instances (Máy Chủ Tạm Thời):**
- Rẻ nhất (giảm 60-90%), nhưng có thể bị AWS thu hồi với 2 phút thông báo
- Dùng cho: batch processing, big data, CI/CD workers, stateless microservices
- **Không dùng cho:** database, có trạng thái, yêu cầu uptime cao

**Savings Plans (Kế Hoạch Tiết Kiệm):**
- Cam kết chi tiêu $/giờ trong 1-3 năm, linh hoạt hơn RI
- Compute Savings Plan: áp dụng cho EC2, Lambda, Fargate (linh hoạt nhất)
- EC2 Instance Savings Plan: chỉ EC2, tiết kiệm nhiều hơn (72%)

**Công Thức Quyết Định:**
```
Luôn chạy + dự đoán được → Reserved hoặc Savings Plans
Luôn chạy + nhiều dịch vụ → Compute Savings Plans
Fault-tolerant, batch → Spot
Biến động, không dự đoán → On-Demand
```

---

### Câu 3: Auto Scaling Group (ASG) hoạt động như thế nào? Giải thích các thành phần chính.

**Độ khó:** ⭐⭐ | **Tần suất xuất hiện:** Rất cao

**Đáp Án Mẫu:**

ASG — Auto Scaling Group (Nhóm Tự Động Co Giãn) là dịch vụ tự động thêm/bớt EC2 instance dựa trên demand (nhu cầu).

**3 Thông Số Cốt Lõi:**
- **Minimum Capacity (Dung Lượng Tối Thiểu):** Số instance tối thiểu luôn chạy
- **Maximum Capacity (Dung Lượng Tối Đa):** Số instance tối đa được phép tạo
- **Desired Capacity (Dung Lượng Mong Muốn):** Số instance hiện tại ASG nhắm đến

**Scaling Policies — Chính Sách Co Giãn:**

1. **Target Tracking Policy (Chính Sách Theo Dõi Mục Tiêu):** Đơn giản nhất. Ví dụ: "Giữ average CPU ở 60%". ASG tự tính toán scale-out/in.

2. **Step Scaling (Co Giãn Theo Bước):** Thêm nhiều instance hơn khi CloudWatch alarm nghiêm trọng hơn. Ví dụ: CPU 70-80% → thêm 2 instance; CPU 80-90% → thêm 4 instance.

3. **Scheduled Scaling (Co Giãn Theo Lịch):** Scale dựa trên lịch cố định. Ví dụ: 8am thêm 10 instance, 10pm bớt 10 instance.

4. **Predictive Scaling (Co Giãn Dự Báo):** AWS dùng ML để dự đoán và pre-scale trước peak traffic.

**Launch Template (Mẫu Khởi Chạy):** Định nghĩa cấu hình instance (AMI — Amazon Machine Image, instance type, Security Group, Key Pair, User Data). ASG dùng template này để tạo instance mới.

**Health Check (Kiểm Tra Sức Khỏe):**
- EC2 Health Check: kiểm tra status check của instance
- ELB Health Check (Kiểm Tra Cân Bằng Tải): kiểm tra HTTP endpoint, chính xác hơn

**Cooldown Period (Thời Gian Hồi Phục):** Thời gian chờ sau mỗi scaling event để tránh thrashing (co giãn liên tục). Mặc định 300 giây.

---

### Câu 4: EC2 Placement Groups là gì? Phân biệt Cluster, Spread, Partition.

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Trung bình

**Đáp Án Mẫu:**

Placement Group (Nhóm Vị Trí Vật Lý) kiểm soát cách EC2 instance được phân bổ trên phần cứng vật lý để tối ưu network performance (hiệu suất mạng) hoặc fault tolerance (khả năng chịu lỗi).

**Cluster Placement Group (Nhóm Vị Trí Cụm):**
- Tất cả instance trong cùng một Availability Zone (Vùng Khả Dụng) và hardware rack
- **Ưu điểm:** Latency cực thấp (< 1ms), throughput cao (10 Gbps+)
- **Nhược điểm:** Nếu rack hỏng, tất cả instance đều bị ảnh hưởng
- **Dùng cho:** HPC (High Performance Computing — Tính Toán Hiệu Suất Cao), machine learning training, distributed databases cần low-latency network

**Spread Placement Group (Nhóm Vị Trí Phân Tán):**
- Mỗi instance trên một hardware rack riêng, có thể trải qua nhiều AZ
- **Ưu điểm:** Fault isolation (Cô Lập Lỗi) tốt nhất — rack hỏng chỉ ảnh hưởng 1 instance
- **Nhược điểm:** Giới hạn 7 instance per AZ per group
- **Dùng cho:** Critical instances cần HA tối đa (primary databases, ZooKeeper, HDFS namenode)

**Partition Placement Group (Nhóm Vị Trí Phân Vùng):**
- Chia instance thành partitions, mỗi partition trên rack riêng
- Tối đa 7 partitions per AZ, mỗi partition có nhiều instance
- **Ưu điểm:** Cân bằng giữa scale và fault isolation
- **Dùng cho:** Distributed big data systems (Hadoop, Cassandra, Kafka) cần rack awareness

**Tóm Tắt Ra Quyết Định:**
```
Cần network performance cực cao  → Cluster
Cần fault isolation tối đa       → Spread
Cần distributed + rack-aware     → Partition
```

---

### Câu 5: Giải thích EC2 Instance Metadata Service (IMDS) và sự khác biệt giữa IMDSv1 và IMDSv2.

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Trung bình

**Đáp Án Mẫu:**

IMDS — Instance Metadata Service (Dịch Vụ Siêu Dữ Liệu Máy Chủ) là endpoint nội bộ `169.254.169.254` mà mọi EC2 instance có thể gọi để lấy thông tin về chính nó: instance ID, IP, IAM credentials, region, AZ, user data, v.v.

**IMDSv1 (Phiên Bản 1 — Không An Toàn):**
```bash
curl http://169.254.169.254/latest/meta-data/instance-id
```
- Không cần xác thực, bất kỳ process nào chạy trên instance đều gọi được
- Lỗ hổng: nếu ứng dụng có lỗi SSRF (Server-Side Request Forgery — Giả Mạo Yêu Cầu Phía Máy Chủ), attacker có thể đánh cắp IAM credentials

**IMDSv2 (Phiên Bản 2 — Bảo Mật Hơn):**
```bash
# Bước 1: Lấy session token (có TTL tối đa 6 giờ)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Bước 2: Dùng token trong mọi request
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```
- Yêu cầu session token (PUT request trước, GET request sau)
- Bảo vệ chống SSRF vì attacker không thể thực hiện PUT request qua server vulnerable
- AWS khuyến nghị bắt buộc dùng IMDSv2 bằng cách set `HttpTokens: required`

**Best Practice:** Enforce IMDSv2 tại account level qua SCP (Service Control Policies — Chính Sách Kiểm Soát Dịch Vụ) và Launch Template.

---

### Câu 6: Lifecycle Hooks trong Auto Scaling là gì và tại sao cần dùng?

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Trung bình

**Đáp Án Mẫu:**

Lifecycle Hooks (Móc Vòng Đời) cho phép bạn tạm dừng quá trình scale-out hoặc scale-in của ASG để thực hiện các thao tác tùy chỉnh trước khi instance đi vào hoạt động hoặc bị terminate (chấm dứt).

**Vòng Đời Lifecycle Hook:**

```
Scale-out:
Pending → Pending:Wait → [Hook Action] → Pending:Proceed → InService

Scale-in:
InService → Terminating → Terminating:Wait → [Hook Action] → Terminating:Proceed → Terminated
```

**Trạng Thái "Wait":** Instance bị giữ ở đây tối đa `HeartbeatTimeout` (mặc định 3600 giây). Trong thời gian này, bạn có thể chạy script, publish event đến SNS/SQS, hoặc gọi Lambda.

**Ví Dụ Sử Dụng Thực Tế:**

*Scale-out Hook:*
- Chờ application khởi động và register với load balancer
- Chạy smoke test (kiểm tra nhanh) trước khi đưa instance vào serve traffic
- Load data vào local cache từ S3

*Scale-in Hook:*
- Draining connections (thoát kết nối) — chờ active connections kết thúc
- Flush logs lên CloudWatch hoặc S3
- Deregister instance từ service discovery

**Ví Dụ Thực Tế:** Một hệ thống e-commerce cần drain SQS queue messages đang được process trước khi instance bị scale-in. Lifecycle hook giữ instance ở Terminating:Wait để worker hoàn thành batch hiện tại, sau đó mới terminate.

---

## Nhóm 2: Serverless Lambda

---

### Câu 7: Lambda cold start là gì? Những yếu tố ảnh hưởng và cách giảm thiểu?

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Rất cao

**Đáp Án Mẫu:**

Cold Start (Khởi Động Lạnh) xảy ra khi Lambda cần tạo một execution environment (môi trường thực thi) mới vì chưa có environment nào sẵn sàng. Lambda phải:

1. Cấp phát micro-VM (firecracker)
2. Download function package / container image
3. Khởi động runtime (Python, Node.js, Java JVM, v.v.)
4. Chạy initialization code (code ngoài handler)
5. Thực thi handler function

**Thời Gian Cold Start Điển Hình:**

| Runtime        | Cold Start Thường Gặp |
| -------------- | -------------------- |
| Node.js / Python | 100-500ms          |
| Go / .NET      | 200-600ms            |
| Java / Scala   | 1-10 giây            |
| Container image| 2-15 giây            |

**Nguyên Nhân Cold Start:**
- Hết warm execution environments (traffic giảm, function mới deploy)
- Concurrency burst (nhiều request đồng thời)
- Function deploy lại sau khi thay đổi code/config

**Cách Giảm Thiểu:**

1. **Provisioned Concurrency (Đồng Thời Được Cung Cấp):** AWS pre-warm sẵn N execution environments. Loại bỏ hoàn toàn cold start nhưng tốn tiền ngay cả khi không có request.

2. **SnapStart (Khởi Động Nhanh):** Chỉ cho Java 11+. Khởi tạo function một lần, chụp snapshot (ảnh chụp) memory và disk, khôi phục từ snapshot thay vì khởi động từ đầu. Giảm 90% cold start.

3. **Giảm kích thước package:** Dùng Lambda Layers, bỏ unused dependencies.

4. **Tránh VPC nếu không cần thiết:** Lambda trong VPC cần thêm thời gian cấp phát ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi). Dùng VPC Endpoint nếu cần truy cập dịch vụ AWS.

5. **Tăng memory:** Memory cao hơn → CPU nhiều hơn → initialization nhanh hơn.

6. **Giảm code initialization:** Chỉ khởi tạo connections khi cần, dùng lazy initialization.

**Khi Nào Cold Start Không Quan Trọng:**
- Batch processing, data pipelines (không yêu cầu latency thấp)
- Async workflows (workflow bất đồng bộ)
- Khi Provisioned Concurrency đã được cấu hình

---

### Câu 8: Phân biệt Reserved Concurrency và Provisioned Concurrency trong Lambda.

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Cao

**Đáp Án Mẫu:**

**Reserved Concurrency (Đồng Thời Được Đặt Trước):**
- Đặt giới hạn tối đa concurrent executions cho một function cụ thể
- Không tốn thêm chi phí
- Mục đích kép:
  - **Throttle function:** Ngăn function dùng hết concurrency quota của account, bảo vệ downstream (các service phía sau) như database khỏi bị overwhelmed
  - **Đảm bảo quota:** Giữ lại đủ concurrency cho function quan trọng, các function khác không "ăn mất"
- Khi đặt = 0: function bị tắt hoàn toàn (dùng để tạm dừng trigger mà không xóa function)

**Provisioned Concurrency (Đồng Thời Được Cung Cấp):**
- AWS khởi tạo sẵn N execution environments luôn ở trạng thái warm
- Loại bỏ cold start hoàn toàn cho N đó
- **Tốn tiền:** Bạn trả tiền cho N environments kể cả khi không có request
- Dùng kết hợp với Application Auto Scaling (Tự Động Co Giãn Ứng Dụng) để tự động điều chỉnh theo traffic
- Trường hợp dùng: API cần latency < 100ms, user-facing applications

**Công Thức Quyết Định:**

```
Cần giới hạn concurrency tối đa        → Reserved Concurrency
Cần loại bỏ cold start (trả thêm tiền) → Provisioned Concurrency
Cần cả hai                              → Reserved + Provisioned
```

**Ví Dụ Thực Tế:**
- Checkout API: Provisioned Concurrency = 10 (đảm bảo không cold start cho 10 concurrent users)
- Background job: Reserved Concurrency = 5 (bảo vệ database khỏi 100+ concurrent queries)

---

### Câu 9: Lambda Event Source Mapping (Ánh Xạ Nguồn Sự Kiện) hoạt động như thế nào với SQS và Kinesis?

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Cao

**Đáp Án Mẫu:**

Event Source Mapping (ESM — Ánh Xạ Nguồn Sự Kiện) là cơ chế Lambda tự động poll (thăm dò) từ các streaming/queue sources và invoke (gọi) function khi có records.

**Lambda + SQS (Simple Queue Service — Dịch Vụ Hàng Đợi Đơn Giản):**

```
SQS Queue → [Lambda polls] → Lambda function invocation
```

- Lambda poll SQS theo long polling
- Batch size: 1-10,000 messages per invocation
- **Xử lý lỗi:** Nếu function throw exception, toàn bộ batch bị retry đến MaxReceiveCount, sau đó vào DLQ (Dead Letter Queue — Hàng Đợi Thư Chết)
- **Partial batch failure:** Có thể cấu hình report partial success để chỉ retry messages thực sự fail
- Tự động scale: Lambda tăng concurrency khi SQS có nhiều messages

**Lambda + Kinesis Data Streams (Luồng Dữ Liệu Kinesis):**

```
Kinesis Shard → [Lambda polls per shard] → Lambda function invocation
```

- Mỗi shard (phân mảnh) có 1 concurrent Lambda invocation
- Records được xử lý theo thứ tự trong từng shard
- **Xử llý lỗi:** Nếu fail, Lambda retry toàn bộ batch đến expiry hoặc bis destination
- **Bisect on Error (Chia Đôi Khi Lỗi):** Lambda chia batch đôi để tìm record gây lỗi
- Enhanced Fan-out (Phát Sóng Tăng Cường): HTTP/2 push cho latency thấp hơn

**Sự Khác Biệt Quan Trọng:**

| Tính Năng              | SQS                        | Kinesis                      |
| ---------------------- | -------------------------- | ---------------------------- |
| Thứ tự                 | Không đảm bảo (FIFO queue có thể) | Đảm bảo trong shard       |
| Retention (Lưu Giữ)   | 4 ngày (tối đa 14)         | 24 giờ (tối đa 365 ngày)    |
| Scale concurrency      | Tự động theo queue depth   | 1 concurrent/shard           |
| Replay (Phát Lại)      | Không                      | Có (Consumer tua lại)        |

---

### Câu 10: Thiết kế Lambda function tối ưu — những best practices quan trọng nhất?

**Độ khó:** ⭐⭐ | **Tần suất xuất hiện:** Cao

**Đáp Án Mẫu:**

**1. Tối Ưu Initialization Code (Code Khởi Tạo):**

```python
# Tốt: Khởi tạo ngoài handler — chạy 1 lần khi cold start
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')

def handler(event, context):
    # Dùng table đã khởi tạo
    return table.get_item(Key={'id': event['id']})
```

```python
# Xấu: Khởi tạo trong handler — chạy mỗi invocation
def handler(event, context):
    dynamodb = boto3.resource('dynamodb')  # Overhead mỗi lần
    table = dynamodb.Table('users')
    return table.get_item(Key={'id': event['id']})
```

**2. Đặt Timeout (Thời Gian Chờ) Phù Hợp:**
- Không để mặc định 3 giây nếu function có thể cần lâu hơn
- Không để quá cao (tối đa 15 phút) nếu không cần — tiêu tốn concurrency

**3. Xử Lý Idempotency (Tính Lũy Đẳng):**
- Lambda có thể bị invoke nhiều lần với cùng event (at-least-once delivery)
- Dùng idempotency key (DynamoDB để deduplicate — loại trùng)

**4. Environment Variables (Biến Môi Trường) và Secrets:**
```
Không hardcode config → dùng environment variables
Secrets → lấy từ Secrets Manager khi cold start, cache trong memory
```

**5. Kích Thước Memory và CPU:**
- Lambda phân bổ CPU tỷ lệ thuận với memory
- Đôi khi tăng memory từ 512MB → 1024MB giảm execution time đủ để tiết kiệm tiền tổng thể
- Dùng Lambda Power Tuning tool để tìm điểm tối ưu

**6. Phân Tách Business Logic Khỏi Handler:**
```python
def handler(event, context):
    # Handler mỏng — chỉ parse input, delegate, return output
    user_id = event['pathParameters']['id']
    result = user_service.get_user(user_id)  # Business logic riêng
    return {'statusCode': 200, 'body': json.dumps(result)}
```

---

## Nhóm 3: Containers ECS & EKS

---

### Câu 11: Phân biệt ECS Fargate và ECS EC2 launch type. Khi nào chọn loại nào?

**Độ khó:** ⭐⭐ | **Tần suất xuất hiện:** Rất cao

**Đáp Án Mẫu:**

**ECS EC2 Launch Type:**
- Bạn quản lý EC2 instances (cluster nodes) — patch OS, capacity planning, AMI updates
- Trả tiền cho EC2 instance dù container không dùng hết resources
- Kiểm soát được instance type, storage, networking
- Có thể dùng Spot Instances → tiết kiệm chi phí đáng kể
- Phù hợp khi cần: custom AMI, GPU, high-performance storage, mature ops team

**ECS Fargate (Serverless Container):**
- AWS quản lý toàn bộ infrastructure — bạn chỉ define Task CPU và Memory
- Trả tiền theo task (vCPU giờ + GB giờ), không có idle cost nếu task không chạy
- Không thể SSH vào "server" (dùng ECS Exec để debug)
- Không hỗ trợ GPU (tính đến 2025)
- Phù hợp khi: team nhỏ, không muốn ops burden, spiky (đột biến) workload

**Bảng So Sánh Nhanh:**

| Tiêu Chí              | EC2 Launch Type              | Fargate                    |
| --------------------- | ---------------------------- | -------------------------- |
| Quản lý server        | Bạn tự quản lý               | AWS quản lý                |
| Chi phí              | EC2 instance cost            | Per task: vCPU + memory    |
| Spot support          | ✅ Có                        | ✅ Có (Fargate Spot)       |
| GPU support           | ✅ Có                        | ❌ Không                   |
| Custom AMI            | ✅ Có                        | ❌ Không                   |
| Startup time          | Phụ thuộc ASG scale          | ~30 giây per task          |
| Phù hợp cho          | High traffic, steady workload | Variable, microservices    |

**Công Thức Quyết Định:**
```
Team nhỏ + muốn focus vào code → Fargate
Cần GPU / custom OS / kiểm soát sâu → EC2 launch type
Chi phí là ưu tiên số 1 → EC2 + Spot Instances
Spiky workload không dự đoán được → Fargate
```

---

### Câu 12: ECS Task Definition và ECS Service khác nhau như thế nào?

**Độ khó:** ⭐⭐ | **Tần suất xuất hiện:** Cao

**Đáp Án Mẫu:**

**Task Definition (Định Nghĩa Task) — Bản thiết kế (blueprint):**
- JSON document định nghĩa cách chạy container
- Bao gồm: Docker image, CPU/memory, environment variables, networking mode, volumes, logging config, IAM task role
- Có version (revisions): taskDef:1, taskDef:2, v.v.
- Immutable (bất biến): một revision không thay đổi được — deploy mới → tạo revision mới
- Không tự chạy — chỉ là khuôn mẫu

**Task — Đơn vị chạy:**
- Một instance của Task Definition đang chạy
- Có thể chạy bằng cách: launch standalone task hoặc thông qua ECS Service
- Standalone task: chạy một lần và kết thúc (batch jobs, data migration)

**ECS Service — Người quản lý:**
- Đảm bảo luôn có đúng số lượng Task đang chạy (desired count)
- Tự động replace Task bị fail
- Tích hợp với Load Balancer (Cân Bằng Tải) — đăng ký/hủy task với target group
- Hỗ trợ rolling deployment, Blue/Green deployment qua CodeDeploy
- Hỗ trợ Service Auto Scaling — tự động scale số task dựa trên metrics

**Analogie dễ nhớ:**
```
Task Definition  = Công thức nấu ăn (blueprint)
Task             = Đĩa ăn đã được nấu (instance đang chạy)
ECS Service      = Người quản lý bếp (đảm bảo luôn đủ đĩa ăn phục vụ)
```

---

### Câu 13: Giải thích EKS Architecture — Control Plane và Data Plane.

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Cao

**Đáp Án Mẫu:**

EKS — Elastic Kubernetes Service (Dịch Vụ Kubernetes Đàn Hồi) cung cấp Kubernetes cluster với AWS quản lý control plane.

**Control Plane (Mặt Phẳng Điều Khiển) — AWS quản lý:**
- **etcd (Cơ Sở Dữ Liệu Trạng Thái):** Distributed key-value store lưu trữ toàn bộ cluster state
- **API Server (Máy Chủ API):** Giao tiếp điểm của tất cả components — kubectl, kubelets, controllers
- **Scheduler (Bộ Lập Lịch):** Quyết định pod chạy trên node nào dựa trên resource requirements và constraints
- **Controller Manager (Quản Lý Bộ Điều Khiển):** Chạy các control loops (vòng lặp điều khiển) — ReplicaSet, Deployment, HPA

AWS đảm bảo control plane: multi-AZ, highly available, tự động upgrade, không tính tiền instance.

**Data Plane (Mặt Phẳng Dữ Liệu) — Bạn quản lý:**
- **Worker Nodes (Node Làm Việc):** EC2 instances chạy workload thực tế
- **kubelet:** Agent trên mỗi node, nhận lệnh từ API server, quản lý pod lifecycle
- **kube-proxy:** Quản lý network rules cho service routing
- **Container Runtime (Môi Trường Chạy Container):** containerd — engine chạy container

**3 Loại Data Plane:**

1. **Managed Node Groups (Nhóm Node Được Quản Lý):** AWS quản lý EC2 lifecycle — cung cấp, update, drain, terminate. Bạn chọn instance type và scaling range. Dễ nhất để vận hành.

2. **Self-managed Nodes (Node Tự Quản Lý):** Bạn hoàn toàn kiểm soát — tự cài kubelet, tự manage AMI, tự patch. Linh hoạt nhất nhưng vất vả nhất.

3. **Fargate Profiles (Hồ Sơ Fargate):** AWS cấp phát Fargate micro-VM cho mỗi pod — serverless, không cần manage nodes. Phù hợp cho workload nhỏ, dev/test.

---

### Câu 14: IRSA — IAM Roles for Service Accounts là gì và tại sao quan trọng hơn node IAM roles?

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Cao

**Đáp Án Mẫu:**

**Vấn đề với Node IAM Role (Vai Trò IAM Của Node):**
- Mọi pod trên cùng node đều thừa hưởng IAM permissions của node
- Vi phạm nguyên tắc Least Privilege (Đặc Quyền Tối Thiểu) — pod A không cần quyền của pod B nhưng vẫn có
- Nếu một pod bị compromise (xâm phạm), attacker có toàn bộ quyền của node

**IRSA — IAM Roles for Service Accounts (Vai Trò IAM Cho Service Accounts):**
- Mỗi Kubernetes Service Account (Tài Khoản Dịch Vụ) được map với một IAM Role cụ thể
- Pod được annotate với Service Account → nhận IAM credentials riêng qua OIDC federation
- Least privilege: chỉ pod cần quyền mới có quyền đó

**Cơ Chế Hoạt Động:**
```
1. EKS tạo OIDC Identity Provider (Nhà Cung Cấp Danh Tính OIDC)
2. Bạn tạo IAM Role với trust policy cho OIDC provider
3. Annotate Kubernetes Service Account với IAM Role ARN
4. Pod dùng Service Account → nhận JWT token
5. AWS STS (Security Token Service) xác thực JWT → trả về temp credentials
6. SDK trong pod tự động dùng credentials này
```

**Ví Dụ Thực Tế:**
```yaml
# Service Account với annotation IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/s3-reader-role
---
# Pod dùng Service Account này
spec:
  serviceAccountName: s3-reader
  # Pod chỉ có quyền đọc S3, không có quyền nào khác
```

**IRSA vs Pod Identity (tính năng mới hơn):**
- EKS Pod Identity (2023+): đơn giản hơn, không cần OIDC setup thủ công, recommend cho cluster mới

---

### Câu 15: HPA — Horizontal Pod Autoscaler và Cluster Autoscaler hoạt động phối hợp thế nào?

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Trung bình

**Đáp Án Mẫu:**

**HPA — Horizontal Pod Autoscaler (Tự Động Co Giãn Pod Theo Chiều Ngang):**
- Scale số lượng pods của một Deployment/StatefulSet dựa trên metrics
- Metrics hỗ trợ: CPU utilization (mặc định), memory, custom metrics (Prometheus, SQS queue depth)
- Kiểm tra metrics mỗi 15 giây, scale-down sau khi ổn định 5 phút

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

**Cluster Autoscaler (Tự Động Co Giãn Cluster):**
- Scale số lượng nodes (EC2 instances) trong cluster
- Thêm node khi pods không thể schedule vì thiếu tài nguyên (Pending pods)
- Bớt node khi node đang underutilized (không dùng nhiều) và pods có thể chuyển sang node khác
- Tích hợp với ASG — Cluster Autoscaler điều chỉnh desired capacity của ASG

**Sự Phối Hợp (Interplay):**

```
Traffic tăng
    ↓
HPA: Tăng replica pods (ví dụ: 5 → 15 pods)
    ↓
Không đủ node capacity → Pods ở trạng thái Pending
    ↓
Cluster Autoscaler: Phát hiện Pending pods → Scale-out ASG
    ↓
Node mới join cluster → Scheduler place Pending pods lên node mới
    ↓
Traffic giảm
    ↓
HPA: Giảm replicas (15 → 5 pods)
    ↓
Cluster Autoscaler: Node dư thừa → Drain pods → Scale-in ASG
```

**Lưu Ý Quan Trọng:**
- Luôn set resource requests/limits trong pod spec — Cluster Autoscaler cần giá trị này
- Dùng PodDisruptionBudget (PDB — Ngân Sách Gián Đoạn Pod) để đảm bảo availability khi scale-in
- KEDA (Kubernetes Event-Driven Autoscaling) là lựa chọn thay thế mạnh hơn HPA cho event-driven workloads

---

## Nhóm 4: High Availability & Cost

---

### Câu 16: Thiết kế kiến trúc 3-tier web application highly available trên AWS. Walk me through từng decision.

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Rất cao

**Đáp Án Mẫu:**

**Yêu cầu ban đầu cần làm rõ:**
- Traffic: 10,000 requests/ngày hay 10,000 requests/giây?
- Availability target: 99.9% (43 phút downtime/tháng) hay 99.99%?
- RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi) và RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi) là bao nhiêu?

**Kiến Trúc Đề Xuất:**

```
Internet
    ↓
Route 53 (DNS với health checks)
    ↓
CloudFront (CDN — Mạng Phân Phối Nội Dung) + WAF (Web Application Firewall)
    ↓
[Public Subnet AZ-a]              [Public Subnet AZ-b]
 Application Load Balancer ←—————→ Application Load Balancer (multi-AZ)
    ↓                                  ↓
[Private Subnet AZ-a]             [Private Subnet AZ-b]
 EC2 / ECS Fargate                EC2 / ECS Fargate
 Auto Scaling Group               Auto Scaling Group
    ↓                                  ↓
[Private Subnet AZ-a]             [Private Subnet AZ-b]
 RDS Primary ←— Synchronous ———→  RDS Standby (Multi-AZ)
 ElastiCache                      ElastiCache (replica)
```

**Decision Points (Điểm Ra Quyết Định) và Lý Do:**

1. **Multi-AZ:** Bảo vệ khỏi AZ failure — AWS khuyến nghị tối thiểu 2 AZ, production nên 3 AZ

2. **ALB thay vì NLB:** Web application cần Layer 7 routing, path-based routing, header manipulation; NLB cho TCP/UDP performance-critical

3. **Private subnets cho app và database:** Nguyên tắc defense in depth — chỉ load balancer trong public subnet

4. **RDS Multi-AZ thay vì EC2 database:** Tự động failover ~60-120 giây, không phải quản lý replication

5. **ElastiCache Redis:** Giảm database load, session storage, rate limiting

6. **CloudFront:** Cache static assets, reduce latency globally, absorb DDoS (Distributed Denial of Service — Tấn Công Từ Chối Dịch Vụ Phân Tán)

**Trade-offs:**
- Multi-AZ tăng chi phí 2x so với Single-AZ
- ElastiCache thêm phức tạp operational nhưng đáng để giảm DB load
- RDS Multi-AZ sync replication có thể tăng write latency ~2ms

---

### Câu 17: Spot Instance interruption xảy ra thế nào? Thiết kế hệ thống chịu được interruption?

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Trung bình

**Đáp Án Mẫu:**

**Cơ Chế Spot Interruption:**
- AWS cần lấy lại capacity → gửi interruption notice (thông báo thu hồi) trước 2 phút
- Notice đến qua: EC2 Instance Metadata endpoint, EventBridge event, CloudWatch Event
- Sau 2 phút: instance bị stop hoặc terminate

**Thiết Kế Chịu Interruption (Interruption-Tolerant Design):**

**1. Stateless Applications (Ứng Dụng Không Trạng Thái):**
- Không lưu state trên instance — lưu vào S3, DynamoDB, ElastiCache
- Nếu instance bị terminate, không mất data

**2. Checkpointing (Lưu Điểm Kiểm Tra):**
```python
# Xử lý interruption notice
def handle_spot_interruption():
    # 1. Ngừng nhận request mới
    # 2. Hoàn thành request đang xử lý (nếu < 2 phút)
    # 3. Lưu state vào DynamoDB hoặc S3
    # 4. SQS: return messages về queue (change visibility timeout)
    save_checkpoint()
    deregister_from_load_balancer()
```

**3. Mixed Instance Policy trong ASG:**
```
On-Demand base: 20% (đảm bảo minimum capacity)
Spot: 80% (cost savings)
Instance types: [m5.large, m5a.large, m4.large, m5d.large]  # Đa dạng instance pool
Allocation strategy: capacity-optimized  # Chọn pool ít bị interrupt nhất
```

**4. Multiple Instance Types và AZ:**
- Dùng 3-5 instance types khác nhau — nếu một pool hết capacity, dùng pool khác
- Trải rộng qua nhiều AZ — mỗi AZ có spot market riêng

**5. Spot Interruption Handler với EventBridge:**
```
EventBridge Rule: EC2 Spot Instance Interruption Warning
    → Lambda function
    → Deregister instance từ load balancer
    → Drain connections
    → Gửi alert đến SNS (Simple Notification Service)
```

---

### Câu 18: AWS Compute Optimizer và quy trình rightsizing EC2 — giải thích end-to-end.

**Độ khó:** ⭐⭐ | **Tần suất xuất hiện:** Trung bình

**Đáp Án Mẫu:**

Rightsizing (Chọn Đúng Kích Cỡ) là quá trình điều chỉnh EC2 instance sang loại phù hợp nhất với workload thực tế — tránh over-provisioning (cấp quá nhiều tài nguyên) và under-provisioning (cấp quá ít).

**Quy Trình Rightsizing:**

**Bước 1: Thu Thập Dữ Liệu**
- Bật CloudWatch Detailed Monitoring (1 phút granularity)
- Enable Compute Optimizer (miễn phí cơ bản, trả phí cho Enhanced Infrastructure Metrics)
- Thu thập ít nhất 14 ngày data (30+ ngày tốt hơn) để thấy pattern

**Bước 2: Phân Tích với Compute Optimizer**
- Xem recommendations: Under-provisioned, Over-provisioned, Optimized
- Compute Optimizer phân tích CPU, memory (cần CloudWatch Agent), network, disk I/O
- Metrics xem xét: P99 CPU (không chỉ average), memory pressure, network throughput

**Bước 3: Phân Loại Recommendations**

| Tín Hiệu                              | Khuyến Nghị               |
| ------------------------------------- | ------------------------- |
| Average CPU < 10%, burst < 20%        | Xuống instance nhỏ hơn    |
| Average CPU > 80%, P99 > 95%          | Lên instance lớn hơn      |
| Memory > 90% thường xuyên            | Đổi sang R-series         |
| Network throughput cao                | Dùng enhanced networking  |

**Bước 4: Test và Validate**
- Thay đổi 1-2 non-production instance trước
- So sánh performance metrics trước/sau
- Dùng AWS Instance Scheduler để kiểm tra

**Bước 5: Production Rollout**
- Thay đổi Launch Template → Instance Refresh trong ASG
- Monitor CloudWatch metrics trong 24-48 giờ sau thay đổi
- Có rollback plan (plan quay lại)

**Tiết Kiệm Điển Hình:** Rightsizing thường tiết kiệm 20-30% chi phí EC2 mà không ảnh hưởng performance.

---

## Nhóm 5: Trade-offs & Design

---

### Câu 19: EC2 vs Lambda vs ECS Fargate vs EKS — khi nào chọn loại nào? Giải thích trade-offs.

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Rất cao

**Đáp Án Mẫu:**

**Framework Ra Quyết Định:**

**Chọn Lambda khi:**
- Workload event-driven, stateless, burst traffic
- Execution time < 15 phút per invocation
- Không muốn quản lý bất kỳ infrastructure nào
- Ví dụ: image resize khi upload S3, Slack bot, scheduled cleanup jobs

**Chọn ECS Fargate khi:**
- Containerized application cần chạy liên tục (long-running)
- Cần nhiều RAM (Lambda max 10GB, Fargate tối đa 120GB)
- Microservices với traffic tương đối đều
- Team không muốn quản lý EC2 nodes
- Ví dụ: API backend, microservices, web applications

**Chọn ECS EC2 hoặc EKS khi:**
- Cần GPU (machine learning inference/training)
- Cần custom OS configuration
- Cần Kubernetes ecosystem (Helm charts, Operators, service mesh)
- Multi-cloud portability là yêu cầu
- Workload lớn, mature DevOps team

**Chọn EC2 trực tiếp khi:**
- Cần toàn quyền kiểm soát OS
- Legacy applications không thể containerize
- Workload yêu cầu specific hardware (bare metal, GPU, high-memory)
- Stateful databases (thường dùng RDS thay vì EC2 database)

**Ma Trận So Sánh:**

| Tiêu Chí               | Lambda         | ECS Fargate    | ECS EC2        | EKS            |
| ---------------------- | -------------- | -------------- | -------------- | -------------- |
| Quản lý infra          | Không          | Không          | Một phần       | Một phần       |
| Max execution time     | 15 phút        | Không giới hạn | Không giới hạn | Không giới hạn |
| Cold start             | Có             | ~30 giây       | ~30 giây       | ~30 giây       |
| Pricing model          | Per ms         | Per vCPU-sec   | EC2 instance   | EC2 + EKS fee  |
| GPU support            | ❌             | ❌             | ✅             | ✅             |
| Kubernetes ecosystem   | ❌             | ❌             | ❌             | ✅             |
| Tốt cho burst traffic  | ✅✅           | ✅             | ✅             | ✅             |

---

### Câu 20: Làm thế nào để debug một Lambda function bị timeout trong production? Walk me through quy trình.

**Độ khó:** ⭐⭐⭐ | **Tần suất xuất hiện:** Cao

**Đáp Án Mẫu:**

**Quy Trình Debug Có Hệ Thống:**

**Bước 1: Xác Nhận Vấn Đề**
```
CloudWatch Metrics → Duration metric → So sánh với timeout setting
CloudWatch Logs → Tìm "Task timed out" message
X-Ray traces → Timeline của toàn bộ request
```

**Bước 2: Phân Tích Logs**
```python
# CloudWatch Log Insights query
filter @type = "REPORT"
| stats avg(@duration), max(@duration), percentile(@duration, 95) by bin(5m)
| sort by bin

# Tìm requests nào bị timeout
filter @message like /Task timed out/
| fields @timestamp, @requestId, @duration
| sort @timestamp desc
```

**Bước 3: Tìm Root Cause Phổ Biến**

*Nguyên nhân 1: External API call chậm*
- Kiểm tra: X-Ray subsegments cho HTTP calls
- Giải pháp: Thêm timeout cho HTTP client, implement retry với exponential backoff (thời gian chờ tăng theo cấp số nhân)

*Nguyên nhân 2: Database query chậm*
- Kiểm tra: CloudWatch RDS Performance Insights, slow query logs
- Giải pháp: Index optimization, connection pooling với RDS Proxy

*Nguyên nhân 3: Lambda VPC cold start*
- Kiểm tra: X-Ray → "Lambda.InitializationDuration" cao bất thường
- Giải pháp: Provisioned Concurrency, hoặc tránh VPC nếu không cần

*Nguyên nhân 4: Memory pressure gây GC pause*
- Kiểm tra: CloudWatch → Memory utilization (cần CloudWatch Lambda Insights)
- Giải pháp: Tăng memory allocation

*Nguyên nhân 5: Infinite loop hoặc deadlock*
- Kiểm tra: CloudWatch Logs → function không print log sau một điểm nhất định
- Giải pháp: Thêm logging rõ ràng, kiểm tra code path

**Bước 4: Fix và Verify**
```
1. Deploy fix với timeout cao hơn tạm thời để có thêm data
2. Enable X-Ray Active Tracing nếu chưa có
3. Thêm structured logging (JSON format) cho dễ phân tích
4. Set CloudWatch Alarm cho Duration P99 > 80% timeout value
5. Deploy fix thực sự
6. Monitor 24 giờ sau deploy
```

**Best Practice Phòng Ngừa:**
- Set timeout = (expected duration × 2) + buffer, không bao giờ để mặc định 3 giây
- Luôn enable X-Ray tracing cho Lambda in production
- Set CloudWatch Alarm khi Duration > 70% của timeout setting

---

## 📊 Tóm Tắt Từ Khóa Quan Trọng

| Thuật Ngữ                        | Giải Thích Ngắn                                    |
| -------------------------------- | -------------------------------------------------- |
| ASG (Auto Scaling Group)         | Nhóm EC2 tự động co giãn                           |
| HPA (Horizontal Pod Autoscaler)  | Tự động scale số pod trong Kubernetes              |
| IRSA (IAM Roles for Service Accounts) | Gán IAM role cho Kubernetes Service Account  |
| IMDS (Instance Metadata Service) | Endpoint lấy metadata của EC2 instance             |
| ESM (Event Source Mapping)       | Lambda tự động poll từ SQS/Kinesis                 |
| DLQ (Dead Letter Queue)          | Hàng đợi lưu messages xử lý thất bại              |
| Provisioned Concurrency          | Pre-warm Lambda environments loại bỏ cold start    |
| Reserved Concurrency             | Giới hạn max concurrent Lambda executions          |
| Fargate                          | Serverless container engine của AWS                |
| SnapStart                        | Tính năng giảm cold start cho Lambda Java          |
| Cluster Autoscaler               | Tự động scale nodes trong EKS cluster              |
| PDB (PodDisruptionBudget)        | Đảm bảo minimum available pods khi scale-in        |
| ENI (Elastic Network Interface)  | Card mạng ảo trong AWS VPC                         |
| OIDC (OpenID Connect)            | Giao thức xác thực dùng trong IRSA                |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành — 20 câu hỏi với đáp án chi tiết
