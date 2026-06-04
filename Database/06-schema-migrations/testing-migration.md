# Testing Migration Trên Staging

Xác thực migration hoàn toàn trước khi đưa vào production.

## Tại Sao Testing Staging Quan Trọng?

```
Sự khác biệt giữa staging và production:

Data volume:
  Staging: 1,000 hàng  → Migration chạy 0.1 giây
  Production: 100M hàng → Migration chạy 3 giờ! (timeout!)

Data distribution:
  Staging: Data đều đẹp → Migration OK
  Production: Data có edge cases (NULL, special chars, encoding) → Migration lỗi!

Concurrent load:
  Staging: 0 connections → Migration không bị lock
  Production: 1000 connections → Migration bị block 30 giây

→ Chỉ test trên staging với DATA PRODUCTION-LIKE mới có giá trị!
```

---

## Nguyên Tắc Staging Data

### Tạo Staging Data Production-Like

```bash
# Cách 1: Anonymize và copy từ production
pg_dump production_db \
    --no-owner \
    --no-acl \
    | python anonymize.py \
    | psql staging_db

# Script anonymize.py đơn giản:
# - Thay email bằng user_123@example.com
# - Thay tên bằng "User 123"
# - Giữ nguyên số lượng hàng, kiểu dữ liệu, phân bố

# Cách 2: Generate synthetic data với cùng volume:
# pgbench, Faker, hoặc custom scripts
pgbench -i -s 100 staging_db  # 10M hàng orders

# Cách 3: Subset thực tế (production fraction):
pg_dump production_db \
    --table=orders \
    | head -1000000 \  # Lấy 1M hàng
    | psql staging_db
```

### Làm Mới Staging Định Kỳ

```bash
#!/bin/bash
# refresh-staging.sh - Chạy hàng tuần

echo "Refreshing staging database..."

# 1. Dump production (anonymized):
pg_dump $PROD_DB_URL \
    | python scripts/anonymize.py \
    > /tmp/staging_refresh.sql

# 2. Drop và tạo lại staging:
psql $STAGING_DB_URL -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"

# 3. Restore:
psql $STAGING_DB_URL < /tmp/staging_refresh.sql

# 4. Chạy tất cả migrations (từ đầu):
flyway -url=$STAGING_DB_URL migrate

echo "Staging refresh complete. Row counts:"
psql $STAGING_DB_URL -c "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC LIMIT 10;"
```

---

## Test Plan Cho Migration

### Template Test Plan

```markdown
# Test Plan: Migration V10 — Thêm cột status vào users

## Thông Tin
- Migration: V10__Add_status_column.sql
- Ngày test: 2026-04-24
- Tester: [Tên]
- Môi trường: staging-prod-like (10M users, 50M orders)

## Pre-conditions
- [ ] Staging DB đã được refresh từ production (tuần này)
- [ ] Backup staging đã tạo
- [ ] Row counts ghi lại:
  - users: _______
  - orders: _______

## Test Cases

### TC-1: Migration chạy thành công
Steps:
  1. flyway info (kiểm tra version hiện tại)
  2. flyway migrate
  3. flyway info (kiểm tra V10 = Success)
Expected: Migration thành công, không có error
Actual: [Điền vào sau khi test]
Pass/Fail: [P/F]

### TC-2: Thời gian thực thi acceptable
Steps:
  1. Ghi thời điểm bắt đầu
  2. Chạy migration
  3. Ghi thời điểm kết thúc
Expected: < 5 phút cho staging (10M users)
Actual: [Điền]
Pass/Fail: [P/F]

### TC-3: Không lock bảng (concurrent access)
Steps:
  1. Trong session 1: Bắt đầu migration
  2. Trong session 2: SELECT COUNT(*) FROM users; (ngay sau khi migration bắt đầu)
Expected: SELECT chạy bình thường, không chờ đợi
Actual: [Điền]
Pass/Fail: [P/F]

### TC-4: Backfill hoàn chỉnh
Steps:
  1. SELECT COUNT(*) FROM users WHERE status IS NULL;
Expected: 0
Actual: [Điền]
Pass/Fail: [P/F]

### TC-5: Code cũ vẫn hoạt động (backward compat)
Steps:
  1. Chạy app v_cũ trên staging sau migration
  2. Thực hiện các operations cơ bản
Expected: Không có lỗi mới
Actual: [Điền]
Pass/Fail: [P/F]

### TC-6: Code mới hoạt động đúng
Steps:
  1. Chạy app v_mới trên staging sau migration
  2. Test các tính năng liên quan đến status
Expected: Tất cả features hoạt động đúng
Actual: [Điền]
Pass/Fail: [P/F]

## Kết Quả
- Tổng: X/6 test cases passed
- Đánh giá: APPROVED / REJECTED

## Ghi Chú Đặc Biệt
[Bất kỳ vấn đề nào phát hiện]
```

---

## Automated Migration Tests

### pytest Suite Cho Migration

```python
# tests/test_migration_v10.py
import pytest
import psycopg2
import subprocess
import time

STAGING_DB = "postgresql://admin:pass@staging-db:5432/mydb"

@pytest.fixture(scope="session")
def before_migration():
    """Snapshot trạng thái trước migration."""
    conn = psycopg2.connect(STAGING_DB)
    cur = conn.cursor()

    cur.execute("SELECT COUNT(*) FROM users")
    users_count = cur.fetchone()[0]

    cur.execute("SELECT COUNT(*) FROM orders")
    orders_count = cur.fetchone()[0]

    conn.close()
    return {"users": users_count, "orders": orders_count}

@pytest.fixture(scope="session")
def run_migration(before_migration):
    """Chạy migration và đo thời gian."""
    start = time.time()
    result = subprocess.run(
        ["flyway", f"-url={STAGING_DB}", "migrate"],
        capture_output=True,
        text=True
    )
    duration = time.time() - start

    return {
        "success": result.returncode == 0,
        "duration": duration,
        "stdout": result.stdout,
        "before": before_migration
    }

def test_migration_succeeds(run_migration):
    assert run_migration["success"], f"Migration failed: {run_migration['stdout']}"

def test_migration_duration_acceptable(run_migration):
    max_duration = 300  # 5 phút
    assert run_migration["duration"] < max_duration, \
        f"Migration took {run_migration['duration']:.1f}s (max: {max_duration}s)"

def test_status_column_exists(run_migration):
    conn = psycopg2.connect(STAGING_DB)
    cur = conn.cursor()
    cur.execute("""
        SELECT column_name, data_type, column_default, is_nullable
        FROM information_schema.columns
        WHERE table_name = 'users' AND column_name = 'status'
    """)
    col = cur.fetchone()
    conn.close()

    assert col is not None, "Column 'status' not found"
    assert col[1] == 'character varying', f"Wrong type: {col[1]}"
    assert col[3] == 'NO', "Column should be NOT NULL"

def test_no_null_status_values(run_migration):
    conn = psycopg2.connect(STAGING_DB)
    cur = conn.cursor()
    cur.execute("SELECT COUNT(*) FROM users WHERE status IS NULL")
    null_count = cur.fetchone()[0]
    conn.close()

    assert null_count == 0, f"Found {null_count} NULL status values"

def test_status_values_valid(run_migration):
    conn = psycopg2.connect(STAGING_DB)
    cur = conn.cursor()
    cur.execute("""
        SELECT DISTINCT status FROM users
        WHERE status NOT IN ('active', 'inactive', 'deleted', 'dormant')
    """)
    invalid = cur.fetchall()
    conn.close()

    assert len(invalid) == 0, f"Invalid status values found: {invalid}"

def test_row_count_preserved(run_migration):
    conn = psycopg2.connect(STAGING_DB)
    cur = conn.cursor()
    cur.execute("SELECT COUNT(*) FROM users")
    after_count = cur.fetchone()[0]
    conn.close()

    before_count = run_migration["before"]["users"]
    assert after_count == before_count, \
        f"Row count changed: {before_count} → {after_count}"

def test_index_created_and_valid(run_migration):
    conn = psycopg2.connect(STAGING_DB)
    cur = conn.cursor()
    cur.execute("""
        SELECT i.indisvalid
        FROM pg_indexes pi
        JOIN pg_index i ON i.indexrelid = (
            SELECT oid FROM pg_class WHERE relname = pi.indexname
        )
        WHERE pi.tablename = 'users'
          AND pi.indexname = 'idx_users_status'
    """)
    result = cur.fetchone()
    conn.close()

    assert result is not None, "Index idx_users_status not found"
    assert result[0] == True, "Index idx_users_status is not valid"

def test_index_used_in_queries(run_migration):
    conn = psycopg2.connect(STAGING_DB)
    cur = conn.cursor()
    cur.execute("EXPLAIN SELECT * FROM users WHERE status = 'active'")
    plan = "\n".join(row[0] for row in cur.fetchall())
    conn.close()

    assert "Index" in plan, f"Index not used. Plan:\n{plan}"

def test_backward_compatibility(run_migration):
    """Code cũ (không biết về cột status) vẫn hoạt động."""
    conn = psycopg2.connect(STAGING_DB)
    cur = conn.cursor()
    # Simulate old code: SELECT chỉ các cột cũ
    cur.execute("SELECT id, email, name, created_at FROM users LIMIT 10")
    rows = cur.fetchall()
    conn.close()

    assert len(rows) > 0, "Old queries failed after migration"
```

### Chạy Tests Trong CI/CD

```yaml
# .github/workflows/migration-test.yml
name: Migration Tests

on:
  pull_request:
    paths:
      - 'db/migration/**'

jobs:
  test-migration:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s

    steps:
    - uses: actions/checkout@v3

    - name: Load staging data snapshot
      run: |
        psql -h localhost -U postgres testdb < tests/fixtures/staging_snapshot.sql

    - name: Run migration tests
      run: |
        pip install pytest psycopg2-binary
        pytest tests/test_migration_v10.py -v --tb=short

    - name: Upload test report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: migration-test-report
        path: test-report.html
```

---

## Performance Testing Migration

```bash
# Đo thời gian migration với các kích thước data:

# Test 1: 100K rows (hiện tại):
time flyway migrate
# → 2 seconds

# Test 2: 1M rows (tăng 10x):
# Thêm data vào staging:
pgbench -i -s 10 testdb
time flyway migrate
# → 18 seconds (extrapolate: production sẽ mất ~X minutes)

# Test 3: Lock impact:
# Trong terminal 1:
flyway migrate &

# Trong terminal 2 (ngay sau):
psql -c "SELECT COUNT(*) FROM users;" testdb
# Kiểm tra response time — phải < 100ms nếu không lock
```

---

## Checklist Testing Migration

**Môi Trường Staging:**
- [ ] Data volume tương đương production (ít nhất 10%)
- [ ] Data distribution tương tự (có NULL, edge cases)
- [ ] Concurrent load được simulate

**Chức Năng:**
- [ ] Migration chạy thành công (flyway info = Success)
- [ ] Thời gian thực thi acceptable cho production
- [ ] Không lock bảng trong quá trình migrate (test concurrent access)
- [ ] Backfill hoàn chỉnh (không có NULL không mong muốn)
- [ ] Giá trị dữ liệu đúng
- [ ] Index valid và được dùng

**Tương Thích:**
- [ ] Code cũ vẫn hoạt động sau migration
- [ ] Code mới hoạt động đúng
- [ ] API responses không thay đổi (nếu không cần)

**Automated Tests:**
- [ ] pytest suite viết đầy đủ
- [ ] CI/CD pipeline chạy tự động khi có migration mới
- [ ] Test report được lưu lại

---

> **Điểm Mấu Chốt:** Test migration trên staging với data volume production-like là sự đầu tư tốt nhất. Một giờ test staging = Tránh được hàng giờ incident trong production.
