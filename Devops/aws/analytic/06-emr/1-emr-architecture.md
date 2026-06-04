# EMR Architecture — Kiến Trúc Amazon EMR

> Hiểu rõ kiến trúc EMR là nền tảng để vận hành, tối ưu và xử lý sự cố cluster Big Data hiệu quả.

## 📚 Mục Lục

1. [Tổng Quan Kiến Trúc](#tổng-quan-kiến-trúc)
2. [Các Loại Node](#các-loại-node)
3. [Cluster Lifecycle — Vòng Đời Cluster](#cluster-lifecycle)
4. [Storage Options — Lưu Trữ Trong EMR](#storage-options)
5. [Networking — Mạng](#networking)
6. [High Availability — Tính Sẵn Sàng Cao](#high-availability)
7. [Hands-on: Tạo Cluster Đầu Tiên](#hands-on)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🏗️ Tổng Quan Kiến Trúc

### Cluster EMR On EC2 — Cấu Trúc Đầy Đủ

```
┌──────────────────────────────────────────────────────────────────┐
│                    Amazon EMR Cluster                            │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │              Primary Node (Nút Chính)                   │     │
│  │   • ResourceManager (YARN)                              │     │
│  │   • NameNode (HDFS metadata)                            │     │
│  │   • Driver (Spark, Hive, etc.)                          │     │
│  │   • Job History Server                                  │     │
│  └─────────────────────────────────────────────────────────┘     │
│                           │                                      │
│          ┌────────────────┼────────────────┐                     │
│          ▼                ▼                ▼                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Core Node   │  │  Core Node   │  │  Task Node   │           │
│  │  (Nút Lõi)   │  │  (Nút Lõi)   │  │ (Nút Tác Vụ) │           │
│  │              │  │              │  │              │           │
│  │ NodeManager  │  │ NodeManager  │  │ NodeManager  │           │
│  │ DataNode     │  │ DataNode     │  │ (no DataNode)│           │
│  │ Executor     │  │ Executor     │  │ Executor     │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
            Amazon S3     HDFS local    DynamoDB
          (persistent)   (temporary)   (optional)
```

---

## 🖥️ Các Loại Node

### 1. Primary Node (Nút Chính) — Trước Đây Gọi Là "Master Node"

**Vai trò:**
- Điều phối toàn bộ cluster — chạy **YARN ResourceManager** (Trình Quản Lý Tài Nguyên YARN) và **HDFS NameNode** (Nút Tên HDFS)
- Tiếp nhận job submissions (yêu cầu chạy job) từ người dùng
- Chạy **Spark Driver** (Trình Điều Khiển Spark) khi dùng `cluster` deploy mode (chế độ triển khai cluster)
- Host các Web UI: Spark History Server, YARN UI, Ganglia

**Lưu ý quan trọng:**
- Chỉ có **1 Primary Node** trong EMR on EC2 thông thường (High Availability bổ sung thêm 2 nút standby)
- Nếu Primary Node bị lỗi và không có HA → **cluster sẽ dừng hoàn toàn**
- Không nên chạy workload nặng trực tiếp trên Primary Node

```bash
# Kết nối SSH vào Primary Node
ssh -i my-key.pem hadoop@ec2-xx-xxx-xxx-xxx.compute-1.amazonaws.com

# Kiểm tra trạng thái YARN
yarn node -list

# Xem YARN ResourceManager logs
sudo tail -f /var/log/hadoop-yarn/yarn-yarn-resourcemanager-*.log
```

---

### 2. Core Node (Nút Lõi)

**Vai trò:**
- Lưu trữ dữ liệu trên **HDFS** (Hadoop Distributed File System — Hệ Thống File Phân Tán Hadoop)
- Chạy **YARN NodeManager** (Trình Quản Lý Node YARN) để thực thi các task
- Chạy **Spark Executor** (Đơn Vị Thực Thi Spark) — nơi dữ liệu thực sự được xử lý

**Đặc điểm quan trọng:**
- **Không được xóa tùy tiện** — Xóa Core Node có thể gây mất dữ liệu HDFS
- EMR sẽ **từ chối** decommission Core Node nếu làm mất block (khối dữ liệu) HDFS
- Nếu workload không dùng HDFS (dùng S3 làm storage), Core Node chỉ cần ít nhất **1 node** để cluster hoạt động

```
Core Node = NodeManager (chạy task) + DataNode (lưu trữ HDFS)
Task Node = NodeManager (chạy task) only
```

---

### 3. Task Node (Nút Tác Vụ)

**Vai trò:**
- Chỉ chạy **YARN NodeManager** và **Spark Executor** — **không lưu HDFS**
- Dùng để **scale-out** (mở rộng) khả năng tính toán mà không ảnh hưởng đến dữ liệu

**Tại sao Task Node quan trọng:**
- **An toàn để dùng Spot Instance** — Vì không lưu dữ liệu, nếu bị AWS thu hồi (reclaim) → cluster vẫn hoạt động
- **Scale nhanh** — Thêm/bớt Task Node không cần rebalance HDFS
- **Giảm chi phí đáng kể** — Có thể chiếm 50–80% compute với giá Spot

```
Khi nào dùng Task Node:
✅ Workload đọc từ S3 (không cần HDFS)
✅ Cần tăng parallelism (độ song song) tạm thời
✅ Muốn dùng Spot Instances an toàn
```

---

### So Sánh Tổng Hợp 3 Loại Node

| Đặc Điểm | Primary | Core | Task |
|----------|---------|------|------|
| **HDFS DataNode** | ❌ | ✅ | ❌ |
| **YARN NodeManager** | ❌ | ✅ | ✅ |
| **Spark Executor** | ❌ (thường) | ✅ | ✅ |
| **Số lượng tối thiểu** | 1 (hoặc 3 với HA) | 1 | 0 |
| **An toàn dùng Spot** | ❌ | ⚠️ Thận trọng | ✅ |
| **Có thể xóa khi đang chạy** | ❌ | ⚠️ Hạn chế | ✅ |

---

## 🔄 Cluster Lifecycle — Vòng Đời Cluster

### Các Trạng Thái Của Cluster

```
STARTING
   │
   ▼
BOOTSTRAPPING ── (Bootstrap Actions chạy)
   │
   ▼
RUNNING ──────── (Jobs đang thực thi)
   │
   ├──── WAITING (Auto-terminate = false, không có job nào)
   │
   ▼
TERMINATING
   │
   ▼
TERMINATED
   │
   └──── TERMINATED_WITH_ERRORS (nếu có lỗi)
```

### Các Giai Đoạn Chi Tiết

**1. STARTING (Khởi Động)**
- AWS provision (cấp phát) các EC2 instance
- Cài đặt hệ điều hành, Amazon EMR agent
- Thường mất **3–5 phút**

**2. BOOTSTRAPPING (Khởi Tạo)**
- Chạy **Bootstrap Actions** (Hành Động Khởi Tạo) — script tùy chỉnh môi trường
- Cài thêm package, cấu hình JVM, tải model ML, v.v.
- Xảy ra trên **tất cả nodes** trước khi cluster sẵn sàng

```bash
# Ví dụ Bootstrap Action — cài thư viện Python
#!/bin/bash
sudo pip3 install pandas==1.5.3 scikit-learn==1.2.0 boto3
sudo pip3 install pyarrow fastparquet

# Cấu hình biến môi trường
echo "export JAVA_OPTS='-Xmx4g'" >> /etc/environment
```

**3. RUNNING / WAITING**
- Cluster sẵn sàng nhận job
- **RUNNING:** Đang thực thi ít nhất 1 step
- **WAITING:** Không có step nào — nếu `auto-terminate` = true → chuyển sang TERMINATING

**4. TERMINATING → TERMINATED**
- Tất cả data trên HDFS và instance storage bị **xóa vĩnh viễn**
- Chỉ data trên S3 là an toàn sau khi cluster terminate

---

### Steps — Các Bước Xử Lý

EMR cho phép định nghĩa **Steps** (Bước) — đơn vị công việc chạy tuần tự hoặc song song:

```
Cluster Ready
     │
     ▼
Step 1: Spark ETL (10 phút)
     │
     ▼
Step 2: Data Validation (5 phút)
     │
     ▼
Step 3: Export to S3 (3 phút)
     │
     ▼
Auto-terminate (nếu bật)
```

```python
# Tạo cluster với steps qua boto3 (AWS Python SDK)
import boto3

emr = boto3.client('emr', region_name='us-east-1')

response = emr.run_job_flow(
    Name='my-etl-cluster',
    ReleaseLabel='emr-6.15.0',
    Instances={
        'MasterInstanceType': 'm5.xlarge',
        'SlaveInstanceType': 'm5.2xlarge',
        'InstanceCount': 5,
        'KeepJobFlowAliveWhenNoSteps': False  # Auto-terminate sau khi xong
    },
    Steps=[
        {
            'Name': 'Spark ETL Job',
            'ActionOnFailure': 'TERMINATE_CLUSTER',
            'HadoopJarStep': {
                'Jar': 'command-runner.jar',
                'Args': [
                    'spark-submit',
                    '--deploy-mode', 'cluster',
                    '--class', 'com.example.ETLJob',
                    's3://my-bucket/jars/etl-job.jar',
                    '--input', 's3://my-bucket/raw/',
                    '--output', 's3://my-bucket/processed/'
                ]
            }
        }
    ],
    Applications=[{'Name': 'Spark'}, {'Name': 'Hive'}],
    ServiceRole='EMR_DefaultRole',
    JobFlowRole='EMR_EC2_DefaultRole',
    AutoTerminationPolicy={'IdleTimeout': 3600}  # Terminate sau 1h idle
)
```

---

## 💾 Storage Options — Lưu Trữ Trong EMR

### 1. HDFS — Hadoop Distributed File System (Hệ Thống File Phân Tán Hadoop)

```
Block size (Kích Thước Khối): 128 MB (mặc định)
Replication factor (Hệ Số Sao Chép): 2 (mặc định trong EMR, thay vì 3 của Hadoop tiêu chuẩn)
```

**Ưu điểm:**
- Tốc độ đọc/ghi **rất nhanh** — local disk, không qua network đến S3
- Phù hợp cho **iterative algorithms** (thuật toán lặp) như ML training, graph processing

**Nhược điểm:**
- **Mất dữ liệu khi cluster terminate** — Không persistent (không bền vững)
- Cần Core Nodes để lưu trữ — tốn chi phí dù không xử lý

**Khi nào dùng HDFS:**
- Dữ liệu trung gian (intermediate data) giữa nhiều Spark stages
- Thuật toán cần đọc cùng dataset nhiều lần (ML, iterative joins)
- Không cần lưu lâu dài sau khi job xong

---

### 2. Amazon S3 — Lưu Trữ Bền Vững

```
EMR Cluster → Đọc từ S3 (EMRFS) → Xử lý → Ghi kết quả về S3
```

**EMRFS** (EMR File System — Hệ Thống File EMR) là lớp tương thích giúp Spark/Hive đọc S3 như HDFS.

**Ưu điểm:**
- **Persistent** — Dữ liệu không mất khi cluster terminate
- **Decoupled storage and compute** (Tách Biệt Lưu Trữ và Tính Toán) — Cluster có thể terminate mà không mất data
- **Tích hợp với Glue Data Catalog** và các dịch vụ AWS khác

**Nhược điểm:**
- **Chậm hơn HDFS** — Đặc biệt với many small files (nhiều file nhỏ)
- **Eventual consistency** — Đã cải thiện nhiều với S3 Strong Consistency (từ 2020)

```
Best Practice (Thực Hành Tốt Nhất):
✅ Lưu input/output trên S3 (bền vững, rẻ)
✅ Dùng HDFS cho intermediate shuffle data
✅ Dùng S3 Committer thay thế mặc định để tránh lỗi partial write
```

---

### 3. Instance Store — Bộ Nhớ Tạm Thời Trực Tiếp

- Ổ đĩa NVMe gắn trực tiếp vào EC2 instance (i3, d2, h1 instance families)
- **Tốc độ cực cao** — Thích hợp cho Spark shuffle (hoán vị dữ liệu) và spill (tràn bộ nhớ)
- **Mất hoàn toàn** khi instance dừng/terminate

---

## 🌐 Networking — Mạng

### VPC và Subnet

```
AWS Region
└── VPC (Virtual Private Cloud — Đám Mây Riêng Ảo)
    ├── Public Subnet (Mạng Con Công Khai) — Primary Node (có thể)
    └── Private Subnet (Mạng Con Riêng Tư) — Core + Task Nodes (khuyến nghị)
```

**Khuyến nghị bảo mật:**
- **Private Subnet** cho tất cả nodes — Không expose trực tiếp ra internet
- **NAT Gateway** (Cổng NAT) cho outbound internet access (cập nhật package, gọi API)
- **Security Groups** (Nhóm Bảo Mật) riêng cho Primary và Core/Task nodes

```hcl
# Terraform — Security Group cho EMR Primary Node
resource "aws_security_group" "emr_primary" {
  name   = "emr-primary-sg"
  vpc_id = var.vpc_id

  # SSH từ bastion host (máy chủ trung gian) nội bộ
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.bastion_cidr]
  }

  # Spark UI, YARN UI — nội bộ VPC
  ingress {
    from_port   = 8088
    to_port     = 8088
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

### SSH Tunnel để Truy Cập Web UI

```bash
# Tạo SSH tunnel (đường hầm SSH) đến Primary Node
ssh -i ~/.ssh/my-key.pem \
    -N -L 8088:localhost:8088 \
    -L 18080:localhost:18080 \
    hadoop@<primary-node-public-dns>

# Sau đó truy cập:
# http://localhost:8088 → YARN ResourceManager UI
# http://localhost:18080 → Spark History Server UI
```

---

## 🔒 High Availability — Tính Sẵn Sàng Cao

### EMR Multi-Primary (HA Mode)

```
                        ZooKeeper Quorum
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
  Primary-1 (Active)    Primary-2 (Standby)   Primary-3 (Standby)
  YARN RM Leader         YARN RM Standby        YARN RM Standby
  HDFS NN Active         HDFS NN Standby        HDFS NN Standby
```

**Khi Active Primary Node bị lỗi:**
1. **ZooKeeper** (Dịch Vụ Đồng Thuận Phân Tán) phát hiện leader mất
2. Bầu chọn Standby mới làm Active trong **~30–60 giây**
3. Các job đang chạy có thể cần **retry** (thử lại) nhưng cluster không mất

**Yêu cầu:**
- Cần **3 Primary Nodes** (số lẻ để ZooKeeper có quorum — đa số phiếu)
- Instance type phải **đủ mạnh** cho cả YARN RM + HDFS NN
- Tăng chi phí ~3x cho Primary Nodes

```bash
# Tạo EMR HA Cluster qua CLI
aws emr create-cluster \
  --name "HA-Production-Cluster" \
  --release-label emr-6.15.0 \
  --instance-groups \
    InstanceGroupType=MASTER,InstanceCount=3,InstanceType=m5.2xlarge \
    InstanceGroupType=CORE,InstanceCount=10,InstanceType=m5.4xlarge \
  --ec2-attributes KeyName=my-key,SubnetId=subnet-xxx
```

---

## 🛠️ Hands-on: Tạo Cluster Đầu Tiên

### Bước 1 — Chuẩn Bị IAM Roles (Vai Trò IAM)

```bash
# Tạo default EMR roles nếu chưa có
aws emr create-default-roles

# Các roles được tạo:
# EMR_DefaultRole        → EMR service role (quyền quản lý cluster)
# EMR_EC2_DefaultRole    → EC2 instance profile (quyền cho nodes trong cluster)
```

### Bước 2 — Tạo Cluster Đơn Giản

```bash
aws emr create-cluster \
  --name "my-first-emr-cluster" \
  --release-label emr-6.15.0 \
  --applications Name=Spark Name=Hive \
  --instance-groups \
    InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m5.xlarge \
    InstanceGroupType=CORE,InstanceCount=2,InstanceType=m5.2xlarge \
  --ec2-attributes KeyName=my-key-pair \
  --use-default-roles \
  --log-uri s3://my-bucket/emr-logs/ \
  --no-auto-terminate
```

### Bước 3 — Gửi Spark Job

```bash
aws emr add-steps \
  --cluster-id j-XXXXXXXXXXXXX \
  --steps Type=Spark,Name="My Spark Job",\
ActionOnFailure=CONTINUE,\
Args=[--class,com.example.Main,\
s3://my-bucket/jars/my-app.jar,\
--input,s3://my-bucket/input/,\
--output,s3://my-bucket/output/]
```

### Bước 4 — Theo Dõi và Dọn Dẹp

```bash
# Xem trạng thái cluster
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXXX

# Xem các steps
aws emr list-steps --cluster-id j-XXXXXXXXXXXXX

# Terminate cluster khi xong (QUAN TRỌNG — tránh phát sinh phí)
aws emr terminate-clusters --cluster-ids j-XXXXXXXXXXXXX
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Core Node và Task Node là gì?**

> Core Node chạy cả YARN NodeManager (để xử lý task) lẫn HDFS DataNode (để lưu dữ liệu), trong khi Task Node chỉ chạy NodeManager. Task Node không lưu HDFS nên an toàn để dùng Spot Instance và có thể thêm/bớt linh hoạt mà không ảnh hưởng đến tính toàn vẹn dữ liệu.

**Q: Điều gì xảy ra nếu Primary Node bị lỗi?**

> Với cluster thông thường (single primary), toàn bộ cluster dừng và job bị mất — đây là single point of failure (điểm lỗi đơn). Với EMR Multi-Master HA mode, ZooKeeper bầu chọn standby node làm active trong 30–60 giây, cluster tiếp tục hoạt động.

**Q: Khi nào nên dùng HDFS thay vì S3 trong EMR?**

> Dùng HDFS cho dữ liệu trung gian (intermediate) giữa các stage của Spark khi thuật toán cần đọc nhiều lần, ví dụ ML iterative training. Dùng S3 cho input/output cuối cùng để đảm bảo persistence và tách biệt storage khỏi compute (decoupled architecture).

**Q: Bootstrap Action dùng để làm gì?**

> Bootstrap Action là script chạy trên tất cả nodes ngay sau khi OS được cài đặt nhưng trước khi cluster sẵn sàng nhận job. Dùng để cài thêm Python packages, cấu hình JVM heap size, tải model ML, thiết lập monitoring agent, v.v.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
