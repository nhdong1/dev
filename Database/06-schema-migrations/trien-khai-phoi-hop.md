# Triển Khai & Phối Hợp Với Ứng Dụng

Phối hợp migration database và deployment ứng dụng để đảm bảo zero-downtime và an toàn.

## Thứ Tự Triển Khai Quan Trọng

```
Quy tắc vàng:
  Database migration TRƯỚC khi deploy ứng dụng mới
  (Không phải đồng thời, không phải sau)

Lý do:
  - Schema mới tương thích ngược với code cũ (nhờ Expand-Contract)
  - Code mới cần schema mới đã có sẵn
  - Code cũ vẫn chạy trong quá trình deploy rolling

Trình tự:
  1. Migration → Schema mới (tương thích ngược)
  2. Deploy app → Code mới dần dần thay code cũ
  3. Contract phase → Xóa schema cũ (sau khi code cũ biến mất)
```

---

## Mô Hình Deploy Phổ Biến

### Blue-Green Deployment

```
Môi trường Blue (đang chạy):
  App v1.0 ←→ Database (Schema v1)

Môi trường Green (chuẩn bị):
  App v2.0 ←→ Database (Schema v2 = v1 + migration)

Bước 1: Chạy migration trên database chung:
  Migration: Schema v1 → Schema v2 (backward compatible!)
  Blue app vẫn chạy với Schema v2 (tương thích ngược)

Bước 2: Deploy Green (App v2.0 với Schema v2):
  Test Green

Bước 3: Switch traffic từ Blue → Green (Load Balancer)

Bước 4: Giữ Blue một thời gian (rollback nếu cần)

Bước 5: Contract: Xóa schema cũ (sau khi Blue đã tắt)
```

### Rolling Deployment (Kubernetes)

```yaml
# deployment.yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # Không có downtime
      maxSurge: 2          # Tối đa 2 pod mới cùng lúc

# Thứ tự:
# T=0: 10 pods app v1.0 đang chạy
# T=1: Migration chạy (init container hoặc job riêng)
# T=2: Pod mới app v2.0 bắt đầu
# T=3: Pod v1.0 bị terminate dần
# T=4: 10 pods app v2.0 đang chạy
# Trong T=2 đến T=3: Cả v1.0 và v2.0 chạy song song!
# → Migration phải tương thích ngược với v1.0!
```

### Init Container cho Migration (Kubernetes)

```yaml
# Chạy migration trước khi app khởi động:
spec:
  initContainers:
  - name: run-migrations
    image: myapp:v2.0
    command: ["flyway", "migrate"]
    env:
    - name: FLYWAY_URL
      value: jdbc:postgresql://db:5432/mydb
    - name: FLYWAY_USER
      valueFrom:
        secretKeyRef:
          name: db-migration-creds
          key: username
    - name: FLYWAY_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-migration-creds
          key: password
  containers:
  - name: myapp
    image: myapp:v2.0
    # App chỉ khởi động sau khi initContainer thành công
```

---

## Feature Flags: Tách Schema và Code

```python
# Pattern: Schema sẵn sàng trước, feature bật sau

# Sau Expand migration (cột status đã có):
class UserService:
    def get_user(self, user_id: int) -> dict:
        user = db.query("SELECT * FROM users WHERE id = %s", [user_id])

        # Feature flag: Dùng cột status mới
        if feature_flags.is_enabled('user_status_feature', user_id):
            return {**user, 'status': user['status']}
        else:
            # Code cũ: Bỏ qua cột status
            return {k: v for k, v in user.items() if k != 'status'}

    def update_user_status(self, user_id: int, status: str):
        if not feature_flags.is_enabled('user_status_feature', user_id):
            raise FeatureNotEnabled("Status feature not enabled")

        db.execute(
            "UPDATE users SET status = %s WHERE id = %s",
            [status, user_id]
        )
```

```python
# Dần dần rollout:
# Week 1: 0% users
# Week 2: 10% users (A/B test)
# Week 3: 50% users
# Week 4: 100% users
# Week 5: Contract migration (xóa cột cũ)
```

---

## CI/CD Pipeline Cho Migration

### GitHub Actions Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  migrate:
    name: Run Database Migrations
    runs-on: ubuntu-latest
    environment: production

    steps:
    - uses: actions/checkout@v3

    - name: Run migrations
      run: |
        flyway -url=${{ secrets.DB_URL }} \
               -user=${{ secrets.MIGRATION_USER }} \
               -password=${{ secrets.MIGRATION_PASSWORD }} \
               migrate
      env:
        FLYWAY_LOCATIONS: filesystem:./db/migration

    - name: Verify migration
      run: |
        flyway -url=${{ secrets.DB_URL }} \
               -user=${{ secrets.MIGRATION_USER }} \
               -password=${{ secrets.MIGRATION_PASSWORD }} \
               info | grep "Success"

    - name: Run post-migration tests
      run: |
        python -m pytest tests/migration/ -v

  deploy:
    name: Deploy Application
    runs-on: ubuntu-latest
    needs: migrate  # App deploy SAU khi migration thành công
    environment: production

    steps:
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/myapp \
          myapp=myapp:${{ github.sha }} \
          --record

    - name: Wait for rollout
      run: |
        kubectl rollout status deployment/myapp --timeout=300s

    - name: Smoke tests
      run: |
        ./scripts/smoke-test.sh
```

### Flyway Callback Hooks

```sql
-- Flyway callbacks chạy tự động tại các thời điểm:

-- beforeMigrate__validate_prerequisites.sql:
-- Kiểm tra điều kiện trước khi bắt đầu migrate
DO $$
BEGIN
    -- Đảm bảo không có long-running transactions
    IF EXISTS (
        SELECT 1 FROM pg_stat_activity
        WHERE state = 'active'
          AND query_start < NOW() - INTERVAL '5 minutes'
          AND query NOT LIKE '%flyway%'
    ) THEN
        RAISE EXCEPTION 'Long-running transactions detected. Migration aborted.';
    END IF;
END;
$$;

-- afterMigrate__update_schema_version.sql:
-- Cập nhật bảng tracking nội bộ sau migrate
INSERT INTO schema_changelog (version, migrated_at, migrated_by)
VALUES (
    (SELECT MAX(version) FROM flyway_schema_history WHERE success = true),
    NOW(),
    current_user
);
```

---

## Lock Management Trong Migration

### Phát Hiện Lock Trước Khi Migrate

```sql
-- Kiểm tra có lock conflict không trước khi chạy DDL:
SELECT
    pid,
    usename,
    state,
    query_start,
    NOW() - query_start AS duration,
    LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < NOW() - INTERVAL '30 seconds'
ORDER BY query_start;

-- Nếu có transactions lâu → Chờ hoặc cancel chúng trước
```

### lock_timeout Cho Migration An Toàn

```sql
-- Đặt timeout để migration fail nhanh nếu không lấy được lock:
SET lock_timeout = '5s';
SET statement_timeout = '300s';  -- Max 5 phút cho toàn bộ statement

-- Nếu bảng đang busy, DDL sẽ fail sau 5 giây thay vì chờ mãi:
ALTER TABLE users ADD COLUMN status VARCHAR(50);
-- ERROR: canceling statement due to lock timeout
-- → Retry sau, hoặc dùng maintenance window

RESET lock_timeout;
RESET statement_timeout;
```

### Retry Logic Cho Migration

```python
# Python script với retry:
import psycopg2
import time

def run_migration_with_retry(migration_sql: str, max_retries: int = 3):
    for attempt in range(max_retries):
        try:
            with psycopg2.connect(DATABASE_URL) as conn:
                with conn.cursor() as cur:
                    cur.execute("SET lock_timeout = '10s'")
                    cur.execute(migration_sql)
                conn.commit()
                print(f"Migration succeeded on attempt {attempt + 1}")
                return

        except psycopg2.errors.LockNotAvailable:
            wait_time = (2 ** attempt) * 5  # 5s, 10s, 20s
            print(f"Lock timeout on attempt {attempt + 1}, waiting {wait_time}s...")
            time.sleep(wait_time)

    raise Exception(f"Migration failed after {max_retries} attempts")
```

---

## Phối Hợp Với Maintenance Window

### Khi Cần Maintenance Window

```
Không cần maintenance window:
✓ Thêm cột nullable với default
✓ Thêm index CONCURRENTLY
✓ Thêm table mới
✓ Thêm stored procedure/function
✓ Expand phase của Expand-Contract

Cần maintenance window:
⚠ ALTER TABLE ... SET NOT NULL (nếu bảng lớn)
⚠ VACUUM FULL (rewrites toàn bộ bảng)
⚠ ALTER TYPE (thay đổi kiểu dữ liệu)
⚠ DROP TABLE / DROP COLUMN (không phải zero-downtime)
⚠ Tạo index không có CONCURRENTLY (khóa bảng)
```

### Runbook Maintenance Window

```bash
#!/bin/bash
# maintenance-migration.sh

MAINTENANCE_START=$(date -u +%Y-%m-%dT%H:%M:%SZ)
echo "=== Maintenance window started at $MAINTENANCE_START ==="

# 1. Báo cho team (Slack/PagerDuty)
curl -X POST "$SLACK_WEBHOOK" \
    -d '{"text": "🔧 Database maintenance started: Adding NOT NULL constraint to users.status"}'

# 2. Chờ traffic giảm (nếu cần)
echo "Waiting for traffic to decrease..."
sleep 30

# 3. Stop writes nếu cần (tùy trường hợp)
# Không nên dùng nếu có thể tránh

# 4. Chạy migration
echo "Running migration..."
flyway migrate
if [ $? -ne 0 ]; then
    echo "MIGRATION FAILED!"
    curl -X POST "$SLACK_WEBHOOK" \
        -d '{"text": "❌ Migration FAILED! Manual intervention required."}'
    exit 1
fi

# 5. Verify
echo "Running verification..."
psql -c "SELECT COUNT(*) FROM users WHERE status IS NULL;"  # Phải là 0

# 6. Restart app servers (nếu cần)
kubectl rollout restart deployment/myapp

# 7. Smoke tests
./smoke-test.sh

MAINTENANCE_END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
echo "=== Maintenance window ended at $MAINTENANCE_END ==="
curl -X POST "$SLACK_WEBHOOK" \
    -d "{\"text\": \"✅ Database maintenance completed at $MAINTENANCE_END\"}"
```

---

## Checklist Phối Hợp Triển Khai

**Trước Migration:**
- [ ] Migration chạy thành công trên staging
- [ ] Backup production được tạo
- [ ] Thông báo team về maintenance (nếu cần)
- [ ] Monitoring được tăng cường (alert threshold thấp hơn)
- [ ] Rollback plan sẵn sàng

**Trong Migration:**
- [ ] Chạy migration TRƯỚC khi deploy app
- [ ] Monitor lock waits và errors
- [ ] Verify migration thành công (flyway info)

**Sau Migration, Trước App Deploy:**
- [ ] Verification queries chạy OK
- [ ] Row count khớp
- [ ] Smoke test với schema mới

**Sau App Deploy:**
- [ ] Rolling update hoàn thành
- [ ] Error rate không tăng
- [ ] Latency bình thường
- [ ] Contract migration lên kế hoạch cho sprint sau

---

> **Điểm Mấu Chốt:** Migration thành công = Database migration + Application deployment + Verification đều thành công. Thiếu bất kỳ bước nào cũng chưa phải thành công hoàn toàn.
