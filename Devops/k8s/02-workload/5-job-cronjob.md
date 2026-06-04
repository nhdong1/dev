# Job và CronJob — Tác Vụ Một Lần và Định Kỳ

> Job tạo một hoặc nhiều Pod để thực hiện tác vụ đến khi hoàn thành rồi dừng. CronJob tạo Job theo lịch cron định kỳ — dành cho batch processing, database migration, backup và mọi tác vụ có điểm kết thúc.

## Mục Lục

1. [Job — Tác Vụ Một Lần](#job--tác-vụ-một-lần)
2. [Các Mô Hình Chạy Job](#các-mô-hình-chạy-job)
3. [Xử Lý Thất Bại](#xử-lý-thất-bại)
4. [CronJob — Tác Vụ Định Kỳ](#cronjob--tác-vụ-định-kỳ)
5. [Cú Pháp Cron](#cú-pháp-cron)
6. [ConcurrencyPolicy — Chính Sách Đồng Thời](#concurrencypolicy--chính-sách-đồng-thời)
7. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Job — Tác Vụ Một Lần

**Job** tạo một hoặc nhiều Pod, theo dõi đến khi Pod hoàn thành thành công. Khi đạt đủ số lượng completion, Job được đánh dấu là thành công.

**Khác biệt cốt lõi với Deployment:**
- Deployment: Pod restart khi dừng → thiết kế cho dịch vụ chạy liên tục
- Job: Pod dừng khi hoàn thành → thiết kế cho tác vụ có điểm kết thúc

**Trường hợp sử dụng:**
- Database migration (chạy script thay đổi schema)
- Batch data processing (xử lý dữ liệu hàng loạt)
- Tạo báo cáo, export dữ liệu
- Gửi email hàng loạt
- One-time setup task (tạo bucket S3, seed dữ liệu ban đầu)
- Machine learning training job

### Manifest Job Cơ Bản

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: production

spec:
  # ─── Số lần hoàn thành cần đạt ──────────────────────
  completions: 1          # chạy đúng 1 lần thành công (mặc định: 1)

  # ─── Số Pod chạy song song ─────────────────────────
  parallelism: 1          # chạy 1 Pod tại một thời điểm (mặc định: 1)

  # ─── Thử lại tối đa khi thất bại ──────────────────
  backoffLimit: 3         # thử lại tối đa 3 lần trước khi đánh dấu Job thất bại

  # ─── Tự xoá sau khi hoàn thành ────────────────────
  ttlSecondsAfterFinished: 300   # tự xoá Job sau 5 phút hoàn thành (cần feature gate TTLAfterFinished)

  # ─── Timeout tổng thể ─────────────────────────────
  activeDeadlineSeconds: 600     # Job phải hoàn thành trong 10 phút, không thì bị terminated

  template:
    metadata:
      labels:
        job-name: db-migration

    spec:
      # Job KHÔNG nên restart container — dùng OnFailure hoặc Never
      restartPolicy: OnFailure   # restart container trong cùng Pod nếu thất bại

      containers:
        - name: migration
          image: my-app:1.2.3
          command: ["python", "manage.py", "migrate", "--noinput"]

          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url

          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

### restartPolicy cho Job

| Giá Trị | Hành Vi | Dùng Khi |
| ------- | ------- | --------- |
| `OnFailure` | Restart container trong **cùng Pod** khi thất bại | Cần giữ Pod để debug dễ hơn |
| `Never` | Không restart — tạo **Pod mới** khi thất bại | Muốn log từng lần thất bại riêng biệt |

> `Always` không được phép dùng với Job.

---

## Các Mô Hình Chạy Job

### 1. Completions cố định (Non-parallel)

```yaml
spec:
  completions: 1
  parallelism: 1
```

Chạy 1 Pod, đợi hoàn thành. Đơn giản nhất.

### 2. Parallel với Fixed Completion Count (Xử lý song song với số lần cố định)

```yaml
spec:
  completions: 10    # cần 10 lần hoàn thành thành công
  parallelism: 3     # chạy tối đa 3 Pod cùng lúc
```

Dùng cho: xử lý 10 file, chạy 10 task độc lập. Kubernetes tạo Pod mới tự động cho đến khi đạt đủ 10 completions.

```
Bước 1: [Pod-1] [Pod-2] [Pod-3]      (3 Pod chạy song song)
Bước 2: [Pod-4] [Pod-5] [Pod-6]      (3 Pod mới sau khi Pod 1-3 hoàn thành)
Bước 3: [Pod-7] [Pod-8] [Pod-9]
Bước 4: [Pod-10]
```

### 3. Parallel với Work Queue (Xử lý hàng đợi công việc)

```yaml
spec:
  parallelism: 5     # chạy 5 worker song song
  completions: 5     # kết thúc khi đủ 5 worker hoàn thành
```

Mỗi Pod worker đọc task từ queue (SQS, RabbitMQ, Redis), xử lý đến khi queue trống, rồi thoát với exit code 0.

---

## Xử Lý Thất Bại

### backoffLimit — Giới Hạn Thử Lại

```yaml
spec:
  backoffLimit: 4    # thử lại tối đa 4 lần
```

Kubernetes theo dõi số lần thất bại. Khi vượt `backoffLimit`, Job được đánh dấu `Failed` và không tạo thêm Pod.

**Backoff delay** (độ trễ thử lại) tăng theo hàm mũ: 10s → 20s → 40s → 80s... tối đa 6 phút, để tránh gây tải lên hệ thống phụ thuộc.

### activeDeadlineSeconds — Thời Hạn Tổng Thể

```yaml
spec:
  activeDeadlineSeconds: 3600   # Job phải xong trong 1 giờ
```

Ngay cả khi `backoffLimit` chưa đạt, khi `activeDeadlineSeconds` hết, Job bị terminate với lý do `DeadlineExceeded`. Dùng để tránh Job treo vô thời hạn.

### Pod Failure Policy (Kubernetes 1.26+)

Cho phép xử lý thất bại theo loại lỗi:

```yaml
spec:
  backoffLimit: 6
  podFailurePolicy:
    rules:
      # Nếu container thoát với code 42 (logic lỗi, không thể retry) → đánh dấu Job thất bại ngay
      - action: FailJob
        onExitCodes:
          containerName: main
          operator: In
          values: [42]

      # Nếu bị evict (node thiếu resource) → không tính vào backoffLimit, thử lại
      - action: Ignore
        onPodConditions:
          - type: DisruptionTarget
```

---

## CronJob — Tác Vụ Định Kỳ

**CronJob** tạo Job theo lịch cron định kỳ. Mỗi lần theo lịch, CronJob controller tạo một Job mới, và Job đó tạo Pod.

```
CronJob (backup-db)
    ↓ mỗi ngày 2:00 sáng
  Job (backup-db-2026050200)
    ↓
  Pod (backup-db-2026050200-xk9p2)
```

### Manifest CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-database
  namespace: production

spec:
  # ─── Lịch chạy (cú pháp cron) ─────────────────────
  schedule: "0 2 * * *"          # mỗi ngày lúc 2:00 sáng

  # ─── Timezone (Kubernetes 1.25+) ──────────────────
  timeZone: "Asia/Ho_Chi_Minh"   # nếu không đặt: dùng UTC

  # ─── Xử lý khi Job cũ chưa hoàn thành ───────────
  concurrencyPolicy: Forbid      # không tạo Job mới nếu Job cũ chưa xong

  # ─── Giữ lịch sử Job ─────────────────────────────
  successfulJobsHistoryLimit: 3  # giữ 3 Job thành công gần nhất
  failedJobsHistoryLimit: 1      # giữ 1 Job thất bại gần nhất

  # ─── Bỏ qua nếu không khởi động đúng giờ ────────
  startingDeadlineSeconds: 300   # bỏ qua nếu trễ hơn 5 phút

  # ─── Tạm dừng CronJob ─────────────────────────────
  suspend: false                 # đặt true để tạm ngừng không tạo Job mới

  # ─── Template cho Job được tạo ────────────────────
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      ttlSecondsAfterFinished: 86400   # xoá Job sau 1 ngày

      template:
        spec:
          restartPolicy: OnFailure

          containers:
            - name: backup
              image: postgres:15
              command:
                - bash
                - -c
                - |
                  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                  pg_dump $DATABASE_URL | gzip > /backup/db_${TIMESTAMP}.sql.gz
                  aws s3 cp /backup/db_${TIMESTAMP}.sql.gz s3://my-backups/postgres/
                  echo "Backup hoàn thành: db_${TIMESTAMP}.sql.gz"

              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: db-secret
                      key: url
                - name: AWS_REGION
                  value: ap-southeast-1

              resources:
                requests:
                  memory: "128Mi"
                  cpu: "100m"
                limits:
                  memory: "256Mi"
                  cpu: "500m"

              volumeMounts:
                - name: backup-tmp
                  mountPath: /backup

          volumes:
            - name: backup-tmp
              emptyDir: {}

          # ServiceAccount có quyền truy cập S3 (qua IRSA trên EKS)
          serviceAccountName: backup-sa
```

---

## Cú Pháp Cron

```
┌─────── phút (0–59)
│ ┌───── giờ (0–23)
│ │ ┌─── ngày trong tháng (1–31)
│ │ │ ┌─ tháng (1–12)
│ │ │ │ ┌ ngày trong tuần (0–6, 0=Chủ Nhật)
│ │ │ │ │
* * * * *
```

### Ví Dụ Phổ Biến

| Lịch | Ý Nghĩa |
| ----- | ------- |
| `* * * * *` | Mỗi phút |
| `0 * * * *` | Đầu mỗi giờ |
| `0 2 * * *` | Mỗi ngày lúc 2:00 sáng |
| `0 2 * * 0` | Mỗi Chủ Nhật lúc 2:00 sáng |
| `0 0 1 * *` | Ngày đầu tiên mỗi tháng lúc 0:00 |
| `*/15 * * * *` | Mỗi 15 phút |
| `0 9-17 * * 1-5` | Mỗi giờ từ 9:00 đến 17:00, Thứ Hai đến Thứ Sáu |
| `@daily` | Mỗi ngày lúc 0:00 (tương đương `0 0 * * *`) |
| `@hourly` | Mỗi giờ (tương đương `0 * * * *`) |

> Dùng [crontab.guru](https://crontab.guru) để kiểm tra cú pháp cron.

---

## ConcurrencyPolicy — Chính Sách Đồng Thời

Kiểm soát hành vi khi lịch kế tiếp đến mà Job trước vẫn chưa hoàn thành:

| Giá Trị | Hành Vi | Dùng Khi |
| ------- | ------- | --------- |
| `Allow` (mặc định) | Tạo Job mới ngay cả khi Job cũ đang chạy | Job độc lập, chạy song song được |
| `Forbid` | Bỏ qua lịch nếu Job cũ chưa xong | Job cần chạy tuần tự (backup, migration) |
| `Replace` | Xoá Job cũ và tạo Job mới | Chỉ cần kết quả gần nhất nhất |

**Ví dụ nguy hiểm không dùng `Forbid`:** CronJob backup database chạy mỗi giờ. Nếu backup hôm đó mất 2 giờ mà không dùng `Forbid`, sẽ có 2 Job backup chạy đồng thời → xung đột file, gấp đôi tải trên database.

---

## Ví Dụ Thực Tế

### Database Migration Với Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-v2-schema
  namespace: production
  annotations:
    # Tích hợp với Helm: không chạy lại khi upgrade nếu Job đã thành công
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded

spec:
  backoffLimit: 0          # không retry — migration thất bại cần điều tra ngay
  activeDeadlineSeconds: 300

  template:
    spec:
      restartPolicy: Never

      initContainers:
        # Đảm bảo database sẵn sàng trước khi chạy migration
        - name: wait-for-db
          image: postgres:15
          command:
            - sh
            - -c
            - until pg_isready -h postgres-service -U postgres; do sleep 2; done

      containers:
        - name: migrate
          image: my-app:2.0.0
          command: ["alembic", "upgrade", "head"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
```

### Cleanup Job — Dọn Dẹp Dữ Liệu Cũ

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-records
  namespace: production

spec:
  schedule: "0 3 * * *"       # mỗi ngày 3:00 sáng
  timeZone: "Asia/Ho_Chi_Minh"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3

  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 7200   # tối đa 2 giờ

      template:
        spec:
          restartPolicy: OnFailure

          containers:
            - name: cleanup
              image: my-app:latest
              command:
                - python
                - -c
                - |
                  import psycopg2
                  import os
                  conn = psycopg2.connect(os.environ['DATABASE_URL'])
                  cur = conn.cursor()
                  cur.execute("""
                    DELETE FROM events
                    WHERE created_at < NOW() - INTERVAL '90 days'
                  """)
                  deleted = cur.rowcount
                  conn.commit()
                  print(f"Đã xoá {deleted} records cũ hơn 90 ngày")
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: db-secret
                      key: url
```

### Chạy Job Thủ Công Từ CronJob

```bash
# Tạo Job ngay lập tức từ CronJob (không cần chờ lịch)
kubectl create job --from=cronjob/backup-database manual-backup-20260510

# Xem trạng thái Job
kubectl get job manual-backup-20260510

# Xem log
kubectl logs -l job-name=manual-backup-20260510
```

---

## Câu Hỏi Phỏng Vấn

### Q: Sự khác biệt giữa `restartPolicy: OnFailure` và `restartPolicy: Never`?

- **`OnFailure`**: Khi container thất bại, kubelet restart **cùng Pod** đó. Pod giữ nguyên tên, dễ debug nhưng log bị ghi đè.
- **`Never`**: Khi container thất bại, Job tạo **Pod mới**. Mỗi lần thất bại có Pod riêng với log riêng — dễ theo dõi lịch sử lỗi hơn.

Trong production: `Never` thường được ưu tiên để giữ log rõ ràng hơn và tránh trạng thái cũ ảnh hưởng lần chạy mới.

### Q: `ttlSecondsAfterFinished` là gì và tại sao quan trọng?

Không có `ttlSecondsAfterFinished`, Job và Pod của nó tồn tại mãi sau khi hoàn thành — gây lãng phí etcd storage và làm namespace lộn xộn. `ttlSecondsAfterFinished: 3600` tự động xoá Job và Pod con sau 1 giờ hoàn thành. Lưu ý: đặt đủ dài để có thời gian debug nếu cần.

### Q: CronJob bị miss (bỏ lỡ) khi nào?

CronJob controller kiểm tra lịch định kỳ. Nếu controller bị lỗi hoặc cluster bị downtime, CronJob có thể bị miss. `startingDeadlineSeconds` kiểm soát hành vi:
- Nếu `startingDeadlineSeconds` không đặt: miss thì tạo Job cho lần cuối cùng bị miss khi phục hồi
- Nếu đặt `startingDeadlineSeconds: 60`: bỏ qua hoàn toàn nếu trễ hơn 60 giây
- Nếu miss hơn 100 lần: CronJob ngừng tạo Job và báo lỗi

### Q: Làm sao để debug Job thất bại?

```bash
# Xem trạng thái Job và số lần retry
kubectl describe job <job-name>

# Tìm Pod của Job (kể cả Pod đã Failed)
kubectl get pods -l job-name=<job-name> --field-selector status.phase=Failed

# Xem log Pod thất bại
kubectl logs <pod-name>

# Nếu Pod đã bị xoá, tăng ttlSecondsAfterFinished hoặc dùng backoffLimit cao hơn
```

---

**Xem Tiếp:** [health-probes.md](./health-probes.md) — Liveness, Readiness và Startup Probe
