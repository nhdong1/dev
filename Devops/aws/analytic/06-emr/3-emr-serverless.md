# EMR Serverless — Xử Lý Big Data Không Quản Lý Cluster

> EMR Serverless (EMR Không Máy Chủ) cho phép chạy Apache Spark và Hive mà không cần tạo, quản lý, hay scale cluster thủ công. AWS tự động cấp phát và thu hồi tài nguyên theo từng job.

## 📚 Mục Lục

1. [EMR Serverless là gì?](#emr-serverless-là-gì)
2. [Kiến Trúc EMR Serverless](#kiến-trúc-emr-serverless)
3. [Application và Job Run](#application-và-job-run)
4. [Pre-initialized Capacity — Tài Nguyên Khởi Tạo Trước](#pre-initialized-capacity)
5. [So Sánh: EMR Serverless vs EMR on EC2](#so-sánh)
6. [Giới Hạn và Lưu Ý](#giới-hạn-và-lưu-ý)
7. [Hands-on: Chạy Spark Job Đầu Tiên](#hands-on)
8. [Monitoring và Debugging](#monitoring-và-debugging)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🚀 EMR Serverless là gì?

### Vấn Đề Với EMR Truyền Thống

```
EMR on EC2 — Luồng Công Việc Thông Thường:
1. Ước tính tài nguyên cần thiết → thường sai ±30%
2. Chờ cluster khởi động (~10 phút)
3. Chạy job
4. Cluster idle (nhàn rỗi) 50% thời gian → lãng phí tiền
5. Job tăng đột biến? → Scale thủ công hoặc chờ Auto Scaling
6. Nhớ terminate cluster hoặc tiếp tục trả tiền
```

### EMR Serverless Giải Quyết Những Gì

```
EMR Serverless — Luồng Mới:
1. Tạo Application (một lần, lâu dài)
2. Submit job
3. AWS tự động:
   - Cấp phát worker (worker được cấp phát tự động — vCPU + RAM) trong ~1–2 phút
   - Scale lên khi cần
   - Scale xuống khi job xong
   - Thu hồi tài nguyên hoàn toàn sau timeout
4. Chỉ trả tiền cho thời gian vCPU + GB-giây thực sự dùng
```

---

## 🏗️ Kiến Trúc EMR Serverless

```
┌─────────────────────────────────────────────────────────┐
│                  EMR Serverless Application              │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │             Spark Driver (khi job chạy)          │   │
│  └────────────────────┬─────────────────────────────┘   │
│                       │                                 │
│          ┌────────────┼────────────┐                    │
│          ▼            ▼            ▼                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Worker 1   │  │   Worker 2   │  │   Worker N   │  │
│  │ (Spark Exec) │  │ (Spark Exec) │  │ (Spark Exec) │  │
│  │  vCPU + RAM  │  │  vCPU + RAM  │  │  vCPU + RAM  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│        (Tự động thêm khi cần, xóa khi xong)            │
└─────────────────────────────────────────────────────────┘
            │                              │
            ▼                              ▼
         Input: S3                    Output: S3
         (Đọc dữ liệu)               (Ghi kết quả)
```

### Mô Hình Tính Phí

```
Chi phí = Σ(vCPU-giây × $0.052/vCPU-giờ) + Σ(GB-giây × $0.0057/GB-giờ)

Ví dụ: Job chạy 10 phút với 10 workers (mỗi worker 4 vCPU, 16 GB RAM)
- vCPU: 10 workers × 4 vCPU × 10/60 giờ × $0.052 = $0.035
- RAM:  10 workers × 16 GB × 10/60 giờ × $0.0057 = $0.015
- Tổng ≈ $0.05 cho job này
```

> **Quan trọng:** Giá trên là ví dụ minh họa — kiểm tra trang định giá AWS cho vùng và thời điểm thực tế.

---

## 📦 Application và Job Run

### Application — Ứng Dụng EMR Serverless

**Application** là đơn vị quản lý tài nguyên dài hạn:

```
Application = Container chứa cấu hình + pre-initialized workers (nếu có)
            + lịch sử job runs

Một Application có thể chạy nhiều Job Runs liên tiếp
```

```bash
# Tạo EMR Serverless Application (Spark 3.4)
aws emr-serverless create-application \
  --name "production-etl-app" \
  --type SPARK \
  --release-label emr-6.15.0 \
  --initial-capacity '{
    "DRIVER": {
      "workerCount": 1,
      "workerConfiguration": {
        "cpu": "4vCPU",
        "memory": "16GB",
        "disk": "100GB"
      }
    },
    "EXECUTOR": {
      "workerCount": 10,
      "workerConfiguration": {
        "cpu": "4vCPU",
        "memory": "16GB",
        "disk": "100GB"
      }
    }
  }' \
  --maximum-capacity '{
    "cpu": "400vCPU",
    "memory": "1000GB"
  }' \
  --auto-stop-config '{
    "enabled": true,
    "idleTimeoutMinutes": 15
  }'
```

**Trạng thái Application:**

```
CREATING → CREATED → STARTING → STARTED → STOPPING → STOPPED
                                    │
                                    └── (Nhận Job Runs ở trạng thái này)
```

---

### Job Run — Lần Chạy Job

```bash
# Lấy Application ID
APP_ID="00f9xxxxxxxxxxxxxxx"

# Submit Spark Job
aws emr-serverless start-job-run \
  --application-id $APP_ID \
  --execution-role-arn "arn:aws:iam::123456789012:role/EMRServerlessRole" \
  --job-driver '{
    "sparkSubmit": {
      "entryPoint": "s3://my-bucket/scripts/etl_job.py",
      "entryPointArguments": ["--input", "s3://my-bucket/raw/", "--output", "s3://my-bucket/processed/"],
      "sparkSubmitParameters": "--conf spark.executor.cores=4 --conf spark.executor.memory=14g --conf spark.driver.cores=2 --conf spark.driver.memory=8g --conf spark.dynamicAllocation.enabled=true --conf spark.dynamicAllocation.minExecutors=2 --conf spark.dynamicAllocation.maxExecutors=50"
    }
  }' \
  --configuration-overrides '{
    "monitoringConfiguration": {
      "s3MonitoringConfiguration": {
        "logUri": "s3://my-bucket/emr-serverless-logs/"
      }
    }
  }'
```

**Theo dõi Job Run:**

```bash
JOB_RUN_ID="00f9xxxxxxxxxxxxxxx"

# Xem trạng thái job
aws emr-serverless get-job-run \
  --application-id $APP_ID \
  --job-run-id $JOB_RUN_ID

# Output trạng thái có thể là:
# SUBMITTED → PENDING → SCHEDULED → RUNNING → SUCCESS / FAILED / CANCELLED
```

---

### IAM Role cho EMR Serverless

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:GetPartitions",
        "glue:CreateTable",
        "glue:UpdateTable"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

**Trust Policy (Chính Sách Tin Tưởng) cho EMR Serverless:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "emr-serverless.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## ⚡ Pre-initialized Capacity — Tài Nguyên Khởi Tạo Trước

### Vấn Đề Cold Start (Khởi Động Lạnh)

Khi không có pre-initialized capacity:

```
Job Submit → Cấp phát workers (1–3 phút) → Khởi động JVM → Chạy job thực sự
```

Với pre-initialized capacity:

```
(Workers đã sẵn sàng từ trước)
Job Submit → Khởi động JVM (30 giây) → Chạy job thực sự
```

### Khi Nào Nên Dùng Pre-initialized Capacity

```
✅ Cần: SLA job bắt đầu < 1 phút
✅ Cần: Nhiều job nhỏ chạy liên tiếp (chi phí cold start cộng dồn lớn)
✅ Cần: Production pipeline với strict latency requirements
❌ Không cần: Batch job ban đêm, độ trễ khởi động không quan trọng
❌ Không cần: Job chạy không thường xuyên (trả phí dù không dùng)
```

```bash
# Cập nhật application để thêm pre-initialized workers
aws emr-serverless update-application \
  --application-id $APP_ID \
  --initial-capacity '{
    "DRIVER": {
      "workerCount": 1,
      "workerConfiguration": {"cpu": "2vCPU", "memory": "8GB"}
    },
    "EXECUTOR": {
      "workerCount": 5,
      "workerConfiguration": {"cpu": "4vCPU", "memory": "16GB"}
    }
  }'
```

> **Chi phí pre-initialized workers:** Được tính phí ngay cả khi không có job chạy. Nên bật `auto-stop` để tự tắt khi idle.

---

## 🔄 So Sánh: EMR Serverless vs EMR on EC2 {#so-sánh}

### So Sánh Chi Tiết

| Tiêu Chí | EMR Serverless | EMR on EC2 |
|----------|---------------|------------|
| **Quản lý infrastructure** | AWS lo hoàn toàn | Tự quản lý |
| **Thời gian khởi động** | 1–3 phút (cold) / <1 phút (warm) | 5–15 phút |
| **Scaling** | Tự động, tức thì | Auto Scaling (chậm hơn) |
| **Mô hình tính phí** | vCPU-giây + GB-giây | EC2 instance-giờ |
| **Tùy chỉnh OS/JVM** | Hạn chế | Toàn quyền |
| **Custom Docker image** | Có hỗ trợ | Có hỗ trợ (Bootstrap Action) |
| **Hive Metastore** | Glue Catalog (tích hợp sẵn) | Glue hoặc tự cài |
| **Hỗ trợ framework** | Spark, Hive | Spark, Hive, Presto, HBase, Flink... |
| **Spot Instance** | N/A (AWS tự tối ưu) | Dùng thủ công, tiết kiệm |
| **Debug / SSH** | Qua logs và Spark UI | SSH trực tiếp vào node |
| **EMR Studio** | ✅ Hỗ trợ | ✅ Hỗ trợ |

### Bảng Quyết Định Nhanh

```
Chọn EMR Serverless khi:
✅ Workload intermittent (không đều, không thường xuyên)
✅ Không muốn quản lý cluster
✅ Cần auto-scale hoàn toàn tự động
✅ Dùng S3 làm primary storage (không cần HDFS)
✅ Data Engineer muốn tập trung code, không ops cluster

Chọn EMR on EC2 khi:
✅ Cần framework ngoài Spark/Hive (Presto, HBase, Flink)
✅ Cần Spot Instance để tối ưu chi phí tối đa
✅ Cần SSH debug trực tiếp vào cluster
✅ Workload liên tục, ổn định — Reserved Instance tiết kiệm hơn
✅ Cần HDFS cho iterative ML jobs
✅ Cần cấu hình OS/JVM đặc biệt
```

---

## ⚠️ Giới Hạn và Lưu Ý

### Giới Hạn Kỹ Thuật

```
Tối đa mỗi Application:
- 1,000 job runs đồng thời
- 30 ngày lưu logs
- Pre-initialized capacity: tối đa được định nghĩa khi tạo/cập nhật app

Framework hỗ trợ:
- Apache Spark ✅
- Apache Hive ✅
- Presto ❌ (chưa hỗ trợ — dùng EMR on EC2)
- Apache Flink ❌ (chưa hỗ trợ — dùng KDA hoặc EMR on EC2)
- HBase ❌ (cần HDFS — dùng EMR on EC2)
```

### Lưu Ý Về Chi Phí

```
Khi nào EMR Serverless ĐẮT hơn EMR on EC2:
❌ Workload liên tục 24/7 → EC2 Reserved Instance rẻ hơn nhiều
❌ Nhiều job cần chạy song song → overhead khởi động cộng dồn

Khi nào EMR Serverless RẺ hơn:
✅ Job chỉ chạy vài giờ/ngày
✅ Workload không thể dự đoán (variable load)
✅ Không quản lý cluster → tiết kiệm OpEx (chi phí vận hành)
```

---

## 🛠️ Hands-on: Chạy Spark Job Đầu Tiên

### Bước 1 — Tạo IAM Role

```bash
# Tạo role
aws iam create-role \
  --role-name EMRServerlessExecutionRole \
  --assume-role-policy-document file://trust-policy.json

# Gắn policy
aws iam put-role-policy \
  --role-name EMRServerlessExecutionRole \
  --policy-name EMRServerlessS3Policy \
  --policy-document file://s3-policy.json
```

### Bước 2 — Tạo Application

```bash
aws emr-serverless create-application \
  --name "my-spark-app" \
  --type SPARK \
  --release-label emr-6.15.0 \
  --auto-stop-config '{"enabled": true, "idleTimeoutMinutes": 15}'
```

### Bước 3 — Upload Script lên S3

```python
# etl_job.py
from pyspark.sql import SparkSession
import argparse

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True)
    parser.add_argument("--output", required=True)
    args = parser.parse_args()

    spark = SparkSession.builder.appName("EMRServerlessETL").getOrCreate()

    # Đọc data từ S3
    df = spark.read.parquet(args.input)

    # Xử lý
    result = df.filter("status = 'active'") \
               .groupBy("category") \
               .count()

    # Ghi kết quả ra S3
    result.write.mode("overwrite").parquet(args.output)

    print(f"Đã xử lý {df.count()} bản ghi")
    spark.stop()

if __name__ == "__main__":
    main()
```

```bash
aws s3 cp etl_job.py s3://my-bucket/scripts/
```

### Bước 4 — Submit và Theo Dõi

```bash
# Submit job
JOB_RUN=$(aws emr-serverless start-job-run \
  --application-id $APP_ID \
  --execution-role-arn "arn:aws:iam::123456789012:role/EMRServerlessExecutionRole" \
  --job-driver '{
    "sparkSubmit": {
      "entryPoint": "s3://my-bucket/scripts/etl_job.py",
      "entryPointArguments": ["--input", "s3://my-bucket/input/", "--output", "s3://my-bucket/output/"]
    }
  }' \
  --query 'jobRunId' --output text)

echo "Job Run ID: $JOB_RUN"

# Theo dõi đến khi hoàn thành
while true; do
  STATUS=$(aws emr-serverless get-job-run \
    --application-id $APP_ID \
    --job-run-id $JOB_RUN \
    --query 'jobRun.state' --output text)
  echo "Trạng thái: $STATUS"
  if [[ "$STATUS" == "SUCCESS" || "$STATUS" == "FAILED" || "$STATUS" == "CANCELLED" ]]; then
    break
  fi
  sleep 30
done
```

---

## 📊 Monitoring và Debugging

### Xem Logs

```bash
# Logs được ghi vào S3 (nếu cấu hình)
aws s3 ls s3://my-bucket/emr-serverless-logs/$APP_ID/$JOB_RUN/

# Cấu trúc thư mục logs:
# applications/$APP_ID/jobs/$JOB_RUN/SPARK_DRIVER/stdout.gz
# applications/$APP_ID/jobs/$JOB_RUN/SPARK_DRIVER/stderr.gz
# applications/$APP_ID/jobs/$JOB_RUN/SPARK_EXECUTOR_xxx/stdout.gz

# Tải và xem Driver logs
aws s3 cp \
  s3://my-bucket/emr-serverless-logs/$APP_ID/$JOB_RUN/SPARK_DRIVER/stdout.gz - \
  | gunzip
```

### Spark UI qua Dashboard

```bash
# Lấy URL Dashboard (có thể truy cập từ EMR Console hoặc CloudWatch)
aws emr-serverless get-dashboard-for-job-run \
  --application-id $APP_ID \
  --job-run-id $JOB_RUN

# Output trả về signed URL — truy cập Spark History Server UI trong 1 giờ
```

### CloudWatch Metrics (Chỉ Số CloudWatch)

```bash
# Các metrics quan trọng:
# emrserverless/JobRunSuccessCount    — Số job thành công
# emrserverless/JobRunFailureCount    — Số job thất bại
# emrserverless/DriverMemoryUsed      — Bộ nhớ Driver đang dùng
# emrserverless/ExecutorMemoryUsed    — Bộ nhớ Executor đang dùng
# emrserverless/RunningWorkerCount    — Số workers đang chạy
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: EMR Serverless khác EMR on EC2 thế nào về mô hình vận hành?**

> EMR Serverless là fully managed (được quản lý hoàn toàn) — không cần tạo, quản lý hay terminate cluster. AWS tự động cấp phát workers khi job submit và thu hồi khi xong. EMR on EC2 cần chủ động quản lý cluster lifecycle (tạo, scale, terminate). EMR Serverless phù hợp cho team muốn tập trung vào code thay vì ops, còn EMR on EC2 cho workload cần kiểm soát sâu về infrastructure.

**Q: Pre-initialized capacity là gì và khi nào cần?**

> Pre-initialized capacity là số workers được khởi tạo trước khi có job run, giúp giảm cold start latency (độ trễ khởi động lạnh) từ 1–3 phút xuống còn <30 giây. Nên dùng khi pipeline có SLA nghiêm ngặt về thời gian bắt đầu, hoặc khi nhiều job nhỏ chạy liên tiếp và thời gian cold start cộng dồn lớn. Chi phí của pre-initialized workers được tính ngay cả khi không có job chạy.

**Q: Hạn chế lớn nhất của EMR Serverless là gì?**

> EMR Serverless chỉ hỗ trợ Spark và Hive — không có Presto, HBase, Flink. Không thể SSH vào nodes để debug trực tiếp. Không dùng được HDFS làm primary storage (vì không có persistent nodes). Với workload 24/7 liên tục, EC2 Reserved Instance thường rẻ hơn mô hình vCPU-giây. Cũng không thể dùng Spot Instance trực tiếp (mặc dù AWS ngầm tối ưu chi phí phía dưới).

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
