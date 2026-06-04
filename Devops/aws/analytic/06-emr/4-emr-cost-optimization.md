# EMR Cost Optimization — Tối Ưu Chi Phí Amazon EMR

> Chi phí EMR có thể chiếm phần lớn ngân sách analytics nếu không quản lý chặt. Tài liệu này cung cấp các kỹ thuật thực chiến để giảm chi phí đáng kể mà không ảnh hưởng đến hiệu suất.

## 📚 Mục Lục

1. [Mô Hình Chi Phí EMR](#mô-hình-chi-phí-emr)
2. [Spot Instances — Instance Giá Spot](#spot-instances)
3. [Instance Fleets vs Instance Groups](#instance-fleets-vs-instance-groups)
4. [Cluster Sizing — Định Cỡ Cluster](#cluster-sizing)
5. [Auto Scaling — Tự Động Mở Rộng](#auto-scaling)
6. [Transient vs Long-running Clusters](#transient-vs-long-running-clusters)
7. [Storage Cost Optimization — Tối Ưu Chi Phí Lưu Trữ](#storage-cost-optimization)
8. [Tối Ưu Spark để Giảm Tài Nguyên](#tối-ưu-spark-để-giảm-tài-nguyên)
9. [Checklist Thực Hành](#checklist-thực-hành)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 💰 Mô Hình Chi Phí EMR

### Các Thành Phần Chi Phí

```
Tổng Chi Phí EMR = EC2 Instance Cost + EMR Surcharge + Storage Cost

┌─────────────────────────────────────────────────────┐
│  EC2 Instance Cost                                  │
│  (Chiếm ~70-80% tổng chi phí)                       │
│  = Số instance × Giờ chạy × Giá EC2               │
│                                                     │
├─────────────────────────────────────────────────────┤
│  EMR Surcharge (Phụ Phí EMR)                       │
│  (Khoảng 25% của EC2 On-Demand cho instance > 0.025/hr) │
│  = Tính thêm trên mỗi vCPU-giờ hoặc % EC2 price   │
│                                                     │
├─────────────────────────────────────────────────────┤
│  S3 Storage + Requests                              │
│  (Thường nhỏ hơn EC2, nhưng cộng dồn theo thời gian)│
│                                                     │
├─────────────────────────────────────────────────────┤
│  Data Transfer (Truyền Dữ Liệu)                    │
│  (Cross-AZ, Egress ra internet — thường nhỏ)       │
└─────────────────────────────────────────────────────┘
```

### Ví Dụ Chi Phí Thực Tế

```
Cluster: 1 Primary (m5.xlarge) + 10 Core (m5.2xlarge) + 20 Task (m5.2xlarge)
Chạy: 8 giờ/ngày × 30 ngày = 240 giờ/tháng

On-Demand:
  Primary: 1 × $0.192/hr × 240h = $46
  Core:   10 × $0.384/hr × 240h = $922
  Task:   20 × $0.384/hr × 240h = $1,843
  Tổng EC2: ~$2,811/tháng

Với Spot (70% tiết kiệm trên Task):
  Task Spot: 20 × $0.115/hr × 240h = $552
  Tổng EC2 (sau Spot): ~$1,520/tháng → Tiết kiệm ~46%
```

---

## 💡 Spot Instances — Instance Giá Spot

### Spot Instance là gì?

**Spot Instance** (Instance Giá Spot — Instance Giá Thị Trường) là EC2 capacity dư thừa của AWS được bán với giá **thấp hơn 70–90%** so với On-Demand. Đổi lại, AWS có thể **thu hồi** instance với **2 phút cảnh báo** khi cần capacity trở lại.

```
On-Demand: $1.00/hr
Spot:      $0.30/hr (tiết kiệm 70%)

Rủi ro Spot: AWS có thể terminate instance bất kỳ lúc nào
```

### Chiến Lược Dùng Spot An Toàn Trong EMR

```
KHÔNG dùng Spot cho:
❌ Primary Node — Mất Primary = Mất cluster hoàn toàn
❌ Core Node nếu dùng HDFS làm primary storage — Mất Core = Mất data HDFS

NÊN dùng Spot cho:
✅ Task Node — Không lưu HDFS → an toàn thu hồi
✅ Core Node nếu dùng S3 làm storage — HDFS chỉ là cache
```

```
Kiến trúc an toàn với Spot:
┌─────────────────────────────────────────────────────────┐
│  Primary Node: On-Demand (m5.xlarge)         → 1 node   │
│  Core Nodes:   On-Demand (m5.2xlarge)        → 2 nodes  │  ← Tối thiểu ổn định
│  Task Nodes:   Spot (r5.2xlarge, m5.2xlarge) → 20 nodes │  ← Phần lớn compute
└─────────────────────────────────────────────────────────┘
Chi phí tiết kiệm: Task nodes chiếm ~70% compute cost → giảm ~50% tổng
```

### Spot Interruption Handling — Xử Lý Khi Spot Bị Thu Hồi

**Spark xử lý Spot interruption như thế nào:**

```
Task Node bị thu hồi
        │
        ▼
YARN phát hiện NodeManager không phản hồi
        │
        ▼
Đánh dấu tasks đang chạy trên node đó là FAILED
        │
        ▼
Scheduler (Bộ Lập Lịch) tự động reschedule (lên lịch lại) tasks sang nodes còn lại
        │
        ▼
Nếu có đủ nodes → Job tiếp tục (chậm hơn)
Nếu không đủ nodes → Job thất bại sau số lần retry
```

**Cấu hình để tăng khả năng chịu lỗi Spot:**

```python
# Tăng số lần retry cho task bị fail
spark.conf.set("spark.task.maxFailures", "8")  # Mặc định 4

# Bật speculative execution (thực thi đoán trước)
# — chạy song song bản copy của task chậm
spark.conf.set("spark.speculation", "true")
spark.conf.set("spark.speculation.multiplier", "1.5")
spark.conf.set("spark.speculation.quantile", "0.9")

# Blacklist executor bị lỗi nhiều lần (Spark 3.1+ dùng "exclude")
spark.conf.set("spark.excludeOnFailure.enabled", "true")
spark.conf.set("spark.excludeOnFailure.task.maxTaskAttemptsPerExecutor", "2")
```

---

## 🚢 Instance Fleets vs Instance Groups

### Instance Groups (Nhóm Instance) — Cách Truyền Thống

```
Instance Group = Tập hợp cùng instance type
- 1 Core Instance Group: m5.2xlarge × 10
- 1 Task Instance Group: m5.2xlarge × 20 (có thể Spot)
```

**Hạn chế:**
- Mỗi group chỉ có **1 instance type**
- Nếu Spot không available (không có sẵn) cho type đó → không thể scale
- Spot capacity pool (nhóm capacity Spot) nhỏ → dễ bị thu hồi hàng loạt

### Instance Fleets (Hạm Đội Instance) — Cách Hiện Đại

```
Instance Fleet = Tập hợp nhiều instance types với phân bổ linh hoạt

Ví dụ Task Fleet:
  Target: 200 vCPU Spot capacity
  Bao gồm:
    - m5.2xlarge (8 vCPU) × up to 25
    - m5a.2xlarge (8 vCPU) × up to 25
    - r5.2xlarge (8 vCPU) × up to 25
    - m5.4xlarge (16 vCPU) × up to 13
    → EMR tự chọn combination tốt nhất từ nhiều Spot pools
```

**Tại sao Instance Fleets tốt hơn cho Spot:**

```
Instance Group (1 type):
  m5.2xlarge Spot bị thu hồi → Mất 100% Task nodes → Job fail

Instance Fleet (nhiều types):
  m5.2xlarge Spot bị thu hồi → EMR thay bằng m5a.2xlarge Spot
  → Gián đoạn nhỏ, không fail toàn bộ
```

```bash
# Tạo cluster với Instance Fleets
aws emr create-cluster \
  --name "cost-optimized-cluster" \
  --release-label emr-6.15.0 \
  --instance-fleets \
    '{
      "InstanceFleetType": "MASTER",
      "TargetOnDemandCapacity": 1,
      "InstanceTypeConfigs": [{"InstanceType": "m5.xlarge"}]
    }' \
    '{
      "InstanceFleetType": "CORE",
      "TargetOnDemandCapacity": 2,
      "InstanceTypeConfigs": [
        {"InstanceType": "m5.2xlarge"},
        {"InstanceType": "m5a.2xlarge"}
      ]
    }' \
    '{
      "InstanceFleetType": "TASK",
      "TargetSpotCapacity": 20,
      "InstanceTypeConfigs": [
        {"InstanceType": "m5.2xlarge", "WeightedCapacity": 1},
        {"InstanceType": "m5a.2xlarge", "WeightedCapacity": 1},
        {"InstanceType": "r5.2xlarge", "WeightedCapacity": 1},
        {"InstanceType": "m5.4xlarge", "WeightedCapacity": 2}
      ],
      "LaunchSpecifications": {
        "SpotSpecification": {
          "TimeoutDurationMinutes": 20,
          "TimeoutAction": "SWITCH_TO_ON_DEMAND"
        }
      }
    }' \
  --applications Name=Spark \
  --use-default-roles
```

---

## 📐 Cluster Sizing — Định Cỡ Cluster

### Tránh Over-provisioning (Cấp Phát Dư Thừa)

```
Sai lầm phổ biến:
❌ Chọn instance lớn nhất "cho chắc" → Lãng phí 50% capacity
❌ Giữ cluster chạy cả đêm dù không có job → Tiền đốt vô ích
❌ Core nodes nhiều hơn cần thiết → HDFS replication factor thấp vẫn đủ

Cách đúng:
✅ Chạy benchmark với dataset thực, đo memory usage thực tế
✅ Bắt đầu nhỏ, tăng dần → Tìm điểm optimal
✅ Terminate cluster ngay sau khi job xong (transient cluster)
```

### Sizing Formula (Công Thức Định Cỡ)

```python
# Ước tính số Core Nodes cần thiết

data_size_gb = 500          # Dataset input
spark_memory_factor = 3     # Data lớn gấp ~3x trong memory (sau deserialization)
instance_memory_gb = 32     # RAM mỗi m5.2xlarge
executor_memory_fraction = 0.75  # 75% RAM cho Spark (còn lại cho OS)

required_total_memory = data_size_gb * spark_memory_factor
memory_per_node = instance_memory_gb * executor_memory_fraction

min_nodes = required_total_memory / memory_per_node
# = 500 * 3 / (32 * 0.75) = 1500 / 24 = 62.5 → 63 nodes tối thiểu

# Thêm buffer 20%
recommended_nodes = int(min_nodes * 1.2)  # = 76 nodes
```

> **Thực tế:** Dùng công thức trên như điểm khởi đầu, sau đó benchmark và điều chỉnh dựa trên Spark UI metrics thực tế.

---

## 📈 Auto Scaling — Tự Động Mở Rộng

### Managed Scaling (Tự Động Mở Rộng Được Quản Lý)

EMR Managed Scaling tự động thêm/bớt Core và Task Nodes dựa trên YARN metrics:

```bash
aws emr put-managed-scaling-policy \
  --cluster-id j-XXXXXXXXXXXXX \
  --managed-scaling-policy '{
    "ComputeLimits": {
      "UnitType": "Instances",
      "MinimumCapacityUnits": 3,
      "MaximumCapacityUnits": 50,
      "MaximumOnDemandCapacityUnits": 10,
      "MaximumCoreCapacityUnits": 5
    }
  }'
```

**Cách hoạt động:**

```
YARN PendingVCores (core đang chờ) > 0 trong > 1 phút
    → Thêm Task Nodes (Spot nếu có)

YARN ContainerPendingRatio (tỷ lệ container đang chờ) = 0 trong > 10 phút
    → Xóa bớt Task Nodes (idle nodes)
```

### Custom Auto Scaling (Tự Động Mở Rộng Tùy Chỉnh)

```json
{
  "Constraints": {
    "MinCapacity": 2,
    "MaxCapacity": 30
  },
  "Rules": [
    {
      "Name": "ScaleOutOnHighYARNUsage",
      "Description": "Scale out khi YARN usage cao",
      "Action": {
        "SimpleScalingPolicyConfiguration": {
          "AdjustmentType": "PERCENT_CHANGE_IN_CAPACITY",
          "ScalingAdjustment": 20,
          "CoolDown": 300
        }
      },
      "Trigger": {
        "CloudWatchAlarmDefinition": {
          "ComparisonOperator": "GREATER_THAN",
          "EvaluationPeriods": 1,
          "MetricName": "YARNMemoryAvailablePercentage",
          "Namespace": "AWS/ElasticMapReduce",
          "Period": 300,
          "Statistic": "AVERAGE",
          "Threshold": 75,
          "Unit": "PERCENT"
        }
      }
    },
    {
      "Name": "ScaleInOnLowYARNUsage",
      "Description": "Scale in khi YARN usage thấp",
      "Action": {
        "SimpleScalingPolicyConfiguration": {
          "AdjustmentType": "PERCENT_CHANGE_IN_CAPACITY",
          "ScalingAdjustment": -20,
          "CoolDown": 300
        }
      },
      "Trigger": {
        "CloudWatchAlarmDefinition": {
          "ComparisonOperator": "LESS_THAN",
          "EvaluationPeriods": 3,
          "MetricName": "YARNMemoryAvailablePercentage",
          "Namespace": "AWS/ElasticMapReduce",
          "Period": 300,
          "Statistic": "AVERAGE",
          "Threshold": 20,
          "Unit": "PERCENT"
        }
      }
    }
  ]
}
```

---

## 🔄 Transient vs Long-running Clusters

### Transient Cluster (Cluster Tạm Thời) — Khuyến Nghị

```
Luồng hoạt động:
1. Job được trigger (bởi Airflow / Step Functions / EventBridge)
2. Tạo EMR cluster mới
3. Chạy job (Steps)
4. Cluster tự terminate khi xong
5. Kết quả được lưu trên S3

Chi phí: Chỉ trả tiền cho thời gian job thực sự chạy
```

```python
# Orchestrate bằng AWS Step Functions (Máy Trạng Thái Bước)
# Hoặc Apache Airflow với EmrCreateJobFlowOperator

from airflow.providers.amazon.aws.operators.emr import (
    EmrCreateJobFlowOperator,
    EmrAddStepsOperator,
    EmrStepSensor,
    EmrTerminateJobFlowOperator
)

# Airflow DAG
create_emr_cluster = EmrCreateJobFlowOperator(
    task_id='create_cluster',
    job_flow_overrides={
        'Name': 'Daily ETL Cluster',
        'ReleaseLabel': 'emr-6.15.0',
        'Instances': {
            'MasterInstanceType': 'm5.xlarge',
            'SlaveInstanceType': 'm5.2xlarge',
            'InstanceCount': 5,
            'KeepJobFlowAliveWhenNoSteps': True,
        },
        'Applications': [{'Name': 'Spark'}],
        'BootstrapActions': [...],
        'Configurations': [...],
        'AutoTerminationPolicy': {'IdleTimeout': 3600},
    },
    aws_conn_id='aws_default',
)

run_spark_job = EmrAddStepsOperator(
    task_id='run_spark_etl',
    job_flow_id="{{ task_instance.xcom_pull('create_cluster', key='return_value') }}",
    steps=[{
        'Name': 'Spark ETL',
        'ActionOnFailure': 'TERMINATE_CLUSTER',
        'HadoopJarStep': {
            'Jar': 'command-runner.jar',
            'Args': ['spark-submit', '--deploy-mode', 'cluster',
                     's3://my-bucket/scripts/etl.py']
        }
    }]
)

# Tự động terminate sau khi xong
terminate_cluster = EmrTerminateJobFlowOperator(
    task_id='terminate_cluster',
    job_flow_id="{{ task_instance.xcom_pull('create_cluster', key='return_value') }}",
    trigger_rule='all_done'  # Terminate dù success hay fail
)
```

### Long-running Cluster (Cluster Chạy Lâu Dài)

**Khi nào hợp lý:**
- Nhiều job nhỏ chạy liên tiếp (overhead tạo cluster quá lớn)
- Cần HDFS data persist giữa các jobs
- Cần Presto/Hive Metastore sẵn sàng

**Tiết kiệm chi phí với long-running cluster:**

```bash
# Auto-terminate sau N giờ idle (nhàn rỗi)
aws emr modify-cluster \
  --cluster-id j-XXXXXXXXXXXXX \
  --step-concurrency-level 5

# Cấu hình auto-termination
aws emr put-auto-termination-policy \
  --cluster-id j-XXXXXXXXXXXXX \
  --auto-termination-policy '{"IdleTimeout": 14400}'  # 4 giờ idle → terminate

# Dùng Scheduled Scaling — Scale down ban đêm
# Scale down: 23:00 → 2 Core + 2 Task
# Scale up:   06:00 → 5 Core + 30 Task
```

---

## 🗄️ Storage Cost Optimization — Tối Ưu Chi Phí Lưu Trữ

### S3 Best Practices (Thực Hành Tốt Nhất S3)

```python
# 1. Dùng định dạng Parquet thay CSV — giảm 5–10x kích thước
df.write \
  .format("parquet") \
  .option("compression", "snappy") \  # Snappy: cân bằng tốc độ và nén
  .partitionBy("year", "month", "day") \
  .save("s3://my-bucket/processed/events/")

# 2. Tránh small files — gộp output trước khi ghi
df.coalesce(100).write.parquet("s3://my-bucket/output/")

# 3. Dùng S3 Intelligent-Tiering cho data ít truy cập
# → S3 tự động chuyển sang cheaper storage tier
```

### HDFS vs S3: Cân Nhắc Chi Phí

```
HDFS Storage Cost:
= Core Node Instance Cost × số giờ chạy
= $0.384/hr × 10 nodes × 720 hr/tháng = $2,765/tháng

S3 Storage Cost:
= 10 TB × $0.023/GB/tháng = $230/tháng

→ S3 rẻ hơn ~12x so với HDFS cho cold storage

Kết luận: Chỉ dùng HDFS cho dữ liệu "nóng" đang xử lý
          Lưu trữ kết quả và dữ liệu lâu dài trên S3
```

---

## 🔧 Tối Ưu Spark để Giảm Tài Nguyên

### Giảm Số Lượng Shuffle — Ít Gây Network Traffic

```python
# Tránh multiple shuffles không cần thiết
# BAD — 2 shuffle operations
df.groupBy("category").count() \
  .join(category_meta, "category") \
  .groupBy("type").sum("count")

# GOOD — 1 shuffle với broadcast join
df.join(broadcast(category_meta), "category") \
  .groupBy("type").count()
```

### Predicate Pushdown — Đẩy Lọc Sớm

```python
# Lọc sớm nhất có thể → giảm data scan từ S3 → giảm executor memory cần thiết
df = spark.read.parquet("s3://my-bucket/events/") \
    .filter(F.col("date") >= "2024-01-01") \    # ← Pushdown vào S3 scan
    .filter(F.col("country") == "VN") \          # ← Áp dụng trước khi load vào memory
    .select("user_id", "event_type", "revenue")  # ← Column pruning
```

### Cache Thông Minh — Đừng Cache Thừa

```python
# Cache khi DataFrame được dùng nhiều lần
popular_users = df.filter("visits > 100").cache()

# Dùng nhiều lần → amortize (phân bổ) cost of caching
result1 = popular_users.join(orders, "user_id")
result2 = popular_users.join(profiles, "user_id")

# QUAN TRỌNG: Unpersist khi không cần nữa → giải phóng memory
popular_users.unpersist()
```

---

## ✅ Checklist Thực Hành

### Trước Khi Tạo Cluster

```
[ ] Đã chọn đúng instance type cho workload (memory-optimized r5 cho Spark, compute-optimized c5 cho Presto)
[ ] Đã cấu hình Task Nodes dùng Spot + Instance Fleets
[ ] Primary Node và Core Nodes dùng On-Demand (tối thiểu 2 Core)
[ ] Đã bật auto-termination hoặc cấu hình auto-terminate sau khi steps xong
[ ] Đã cấu hình log output vào S3 để debug sau này
```

### Khi Thiết Kế Job

```
[ ] Input/Output lưu trên S3 (không phải HDFS)
[ ] Dùng Parquet với Snappy compression
[ ] Có partition strategy hợp lý (không quá nhỏ, không quá lớn)
[ ] Bật AQE (Adaptive Query Execution)
[ ] Bật Dynamic Allocation
[ ] Không có collect() trên large DataFrames
```

### Sau Khi Chạy

```
[ ] Cluster đã terminated (kiểm tra AWS Console)
[ ] Kiểm tra Spark UI — có spill, skew hay GC overhead không?
[ ] Kiểm tra YARN utilization — có bao nhiêu % thực sự dùng?
[ ] Cost Explorer — so sánh chi phí thực tế vs ước tính
[ ] Nếu job chạy thường xuyên → cân nhắc EMR Serverless hoặc Glue
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Làm thế nào để giảm chi phí EMR đáng kể nhất?**

> Chiến lược hiệu quả nhất theo thứ tự: (1) Dùng Spot Instances cho Task Nodes — thường tiết kiệm 50–70% chi phí instance; (2) Dùng transient cluster (tạo mới theo job, terminate khi xong) thay vì cluster chạy 24/7; (3) Dùng Instance Fleets với nhiều instance types để tăng Spot availability; (4) Bật Auto Scaling để không over-provision; (5) Tối ưu Spark job để giảm thời gian chạy và số nodes cần thiết.

**Q: Spot Instance bị thu hồi giữa chừng — EMR xử lý thế nào?**

> YARN phát hiện NodeManager mất kết nối và đánh dấu tất cả tasks trên node đó là FAILED. Spark tự động reschedule (lập lịch lại) các tasks này sang nodes còn lại, với số lần retry được cấu hình qua `spark.task.maxFailures` (mặc định 4). Nếu dùng Instance Fleets, EMR sẽ tự động request capacity từ Spot pool khác hoặc fallback (dự phòng) sang On-Demand nếu cấu hình `TimeoutAction: SWITCH_TO_ON_DEMAND`.

**Q: Instance Fleets khác Instance Groups thế nào và tại sao nên dùng Instance Fleets?**

> Instance Groups chỉ cho phép một instance type mỗi group — nếu instance type đó không có Spot capacity, không thể scale. Instance Fleets cho phép chỉ định nhiều instance types với weight (trọng số) tương đương, EMR tự chọn combination từ nhiều Spot pools khác nhau. Khi một Spot pool cạn, EMR dùng pool khác thay thế, giảm đáng kể rủi ro Spot interruption hàng loạt. Instance Fleets là lựa chọn mặc định cho production clusters sử dụng Spot.

**Q: Khi nào nên dùng EMR thay vì Glue ETL để tối ưu chi phí?**

> Glue thường rẻ hơn cho ETL đơn giản, tốc độ vừa phải — tính phí theo DPU-giờ, không cần quản lý. EMR rẻ hơn khi: (1) cần Spot Instances giảm chi phí tối đa, (2) job cần cấu hình Spark sâu để tối ưu, (3) dataset cực lớn mà Glue DPU không đủ mạnh về throughput, (4) có sẵn kỹ năng ops cluster. Breakeven (điểm hòa vốn) thường xảy ra khi job chạy >2 giờ/ngày hoặc cần >20 DPU.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
