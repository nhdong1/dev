# Công Cụ Migration CSDL

So sánh và hướng dẫn sử dụng Flyway, Liquibase và sqitch.

## Tại Sao Cần Công Cụ Migration?

```
Không có công cụ:
  - Schema thay đổi theo thời gian nhưng không ai track
  - "Staging hoạt động, production bị lỗi" — khác schema!
  - Không biết production đang ở version nào
  - Không thể tái tạo schema từ đầu

Với công cụ migration:
  - Lịch sử thay đổi schema được version control
  - Tái tạo database từ đầu trong bất kỳ môi trường nào
  - Biết chính xác schema hiện tại ở version nào
  - CI/CD tích hợp — migrate tự động khi deploy
```

---

## Flyway

### Khái Niệm Cơ Bản

```
Flyway dùng versioned migration files:

db/migration/
├── V1__Create_users_table.sql
├── V2__Add_orders_table.sql
├── V3__Add_index_on_email.sql    ← Versioned (V prefix)
├── R__Refresh_analytics_view.sql ← Repeatable (R prefix, chạy khi thay đổi)
└── U3__Undo_add_index.sql        ← Undo (chỉ Flyway Teams)

Flyway track migrations trong bảng flyway_schema_history:
version | description           | success | installed_on
--------|----------------------|---------|-------------
1       | Create users table   | true    | 2026-01-01
2       | Add orders table     | true    | 2026-01-15
3       | Add index on email   | true    | 2026-02-01
```

### Cài Đặt và Cấu Hình

```bash
# Cài Flyway CLI:
wget https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.22.0/flyway-commandline-9.22.0-linux-x64.tar.gz
tar -xzf flyway-commandline-9.22.0-linux-x64.tar.gz
ln -s /opt/flyway-9.22.0/flyway /usr/local/bin/flyway

# Maven project:
# pom.xml:
# <dependency>
#     <groupId>org.flywaydb</groupId>
#     <artifactId>flyway-core</artifactId>
#     <version>9.22.0</version>
# </dependency>
```

```properties
# flyway.conf:
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=migration_user
flyway.password=migration_password
flyway.locations=filesystem:./db/migration
flyway.baselineOnMigrate=false
flyway.validateOnMigrate=true
flyway.outOfOrder=false
```

### Các Lệnh Flyway

```bash
# Migrate lên version mới nhất:
flyway migrate

# Xem trạng thái migration:
flyway info

# Output:
# +-----------+---------+---------------------+---------+
# | Category  | Version | Description         | State   |
# +-----------+---------+---------------------+---------+
# | Versioned | 1       | Create users table  | Success |
# | Versioned | 2       | Add orders table    | Success |
# | Versioned | 3       | Add index on email  | Pending |
# +-----------+---------+---------------------+---------+

# Xác thực migrations không bị sửa:
flyway validate

# Sửa trạng thái migration lỗi (sau khi fix thủ công):
flyway repair

# Tạo baseline cho database đã có sẵn:
flyway baseline
```

### Ví Dụ Migration Files

```sql
-- V1__Create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- V2__Add_orders_table.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    total DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status, created_at);

-- V3__Add_status_to_users.sql (Expand-Contract Phase 1):
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'active';
-- Nullable trước, backfill sau, constraint cuối
```

### Flyway Trong Java/Spring Boot

```java
// Spring Boot tự động chạy Flyway khi khởi động!
// application.yml:
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: myapp
    password: ${DB_PASSWORD}
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true

// Flyway chạy migration trước khi ứng dụng start
// → Đảm bảo schema luôn đúng version
```

---

## Liquibase

### Khái Niệm Cơ Bản

```
Liquibase dùng changeset (atomic change units):

db/changelog/
├── db.changelog-master.xml    ← File chính, include các file khác
├── changes/
│   ├── 001-create-users.xml
│   ├── 002-add-orders.xml
│   └── 003-add-indexes.yaml
└── rollbacks/                 ← Rollback scripts

Liquibase track trong bảng DATABASECHANGELOG:
id | author | filename          | exectype | dateexecuted
---|--------|-------------------|----------|-------------
1  | john   | 001-create-users  | EXECUTED | 2026-01-01
2  | jane   | 002-add-orders    | EXECUTED | 2026-01-15
```

### Cú Pháp Changeset

```xml
<!-- db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <include file="changes/001-create-users.xml" relativeToChangelogFile="true"/>
    <include file="changes/002-add-orders.xml" relativeToChangelogFile="true"/>
</databaseChangeLog>

<!-- changes/001-create-users.xml -->
<databaseChangeLog>
    <changeSet id="1" author="john">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="name" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMPTZ" defaultValueComputed="NOW()">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <!-- Rollback tự động (Liquibase biết cách drop table) -->
        <!-- Hoặc custom rollback: -->
        <rollback>DROP TABLE users;</rollback>
    </changeSet>

    <changeSet id="2" author="john">
        <addColumn tableName="users">
            <column name="status" type="VARCHAR(50)" defaultValue="active"/>
        </addColumn>
        <rollback>
            <dropColumn tableName="users" columnName="status"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

### Liquibase YAML Format

```yaml
# changes/002-add-orders.yaml
databaseChangeLog:
  - changeSet:
      id: 3
      author: jane
      changes:
        - createTable:
            tableName: orders
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: user_id
                  type: BIGINT
                  constraints:
                    nullable: false
              - column:
                  name: total
                  type: DECIMAL(10,2)
                  constraints:
                    nullable: false
              - column:
                  name: status
                  type: VARCHAR(50)
                  defaultValue: pending
        - addForeignKeyConstraint:
            baseTableName: orders
            baseColumnNames: user_id
            referencedTableName: users
            referencedColumnNames: id
            constraintName: fk_orders_user_id
      rollback:
        - dropTable:
            tableName: orders
```

### Các Lệnh Liquibase

```bash
# Migrate:
liquibase update

# Rollback về specific tag:
liquibase rollback --tag=v1.0

# Rollback N bước:
liquibase rollbackCount --count=2

# Xem SQL sẽ chạy (không thực thi):
liquibase updateSQL

# Xem trạng thái:
liquibase status

# Tag current state:
liquibase tag --tag=v1.0

# Generate changelog từ database hiện có:
liquibase generateChangeLog
```

---

## sqitch

### Khái Niệm Cơ Bản

```
sqitch dùng dependency graph (không phải numeric versions):

sqitch/
├── sqitch.conf
├── sqitch.plan          ← Danh sách tất cả changes
└── deploy/
│   ├── users.sql        ← Deploy script
├── revert/
│   ├── users.sql        ← Revert script
└── verify/
    ├── users.sql        ← Verify script

Đặc điểm sqitch:
- Không cần số version (dùng tên change)
- Explicit dependencies (A phải deploy trước B)
- Tích hợp Git
- Mạnh về verify step
```

### Sử Dụng sqitch

```bash
# Khởi tạo project:
sqitch init myapp --engine pg

# Thêm change mới:
sqitch add users -n "Create users table"
# Tạo ra: deploy/users.sql, revert/users.sql, verify/users.sql

sqitch add orders --requires users -n "Create orders table"
# orders phụ thuộc vào users

# Sửa deploy/users.sql:
cat > deploy/users.sql << 'EOF'
-- Deploy myapp:users
BEGIN;
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
COMMIT;
EOF

# Sửa revert/users.sql:
cat > revert/users.sql << 'EOF'
-- Revert myapp:users
BEGIN;
DROP TABLE users;
COMMIT;
EOF

# Sửa verify/users.sql:
cat > verify/users.sql << 'EOF'
-- Verify myapp:users
SELECT id, email, name, created_at FROM users WHERE FALSE;
EOF

# Deploy:
sqitch deploy db:pg://localhost/mydb

# Revert last change:
sqitch revert db:pg://localhost/mydb

# Verify:
sqitch verify db:pg://localhost/mydb

# Status:
sqitch status db:pg://localhost/mydb
```

---

## So Sánh Các Công Cụ

| Tiêu chí | Flyway | Liquibase | sqitch |
|----------|--------|-----------|--------|
| Format file | SQL / Java | XML, YAML, JSON, SQL | SQL thuần |
| Versioning | Numeric (V1, V2) | ID + author | Dependency graph |
| Rollback | Chỉ Teams/Enterprise | Built-in | Explicit scripts |
| Learning curve | Thấp | Trung bình | Cao |
| Multi-DB support | Tốt | Rất tốt | Tốt |
| Community | Lớn | Lớn | Nhỏ hơn |
| Free/Paid | Core miễn phí | Miễn phí | Miễn phí |
| Verify step | Không | Không | Built-in |

### Khi Nào Dùng Cái Nào?

```
Flyway → Đơn giản, team nhỏ, Spring Boot project
  ✓ Easy setup, ít config
  ✓ Tích hợp Spring Boot rất tốt
  ✗ Rollback chỉ có ở phiên bản trả phí

Liquibase → Cần đa nền tảng, rollback tự động, nhiều format
  ✓ Rollback tốt (XML format có rollback tự động)
  ✓ Hỗ trợ nhiều database nhất
  ✗ Verbose XML, phức tạp hơn

sqitch → Team PostgreSQL thuần, cần verify step
  ✓ Pure SQL, không abstraction
  ✓ Verify step đảm bảo migration thành công
  ✗ Learning curve cao hơn
```

---

## Migration User Permissions

```sql
-- Tạo dedicated migration user (quyền cao hơn app user):
CREATE ROLE migration_user WITH LOGIN PASSWORD 'migration_password';

-- Cấp quyền cần thiết:
GRANT CONNECT ON DATABASE mydb TO migration_user;
GRANT CREATE ON SCHEMA public TO migration_user;  -- Tạo bảng/index
GRANT USAGE ON SCHEMA public TO migration_user;

-- Trong production: Dùng chỉ khi migrate, revoke sau
-- Hoặc: Migration user chỉ có thể kết nối từ CI/CD server
```

---

## Checklist Công Cụ Migration

- [ ] Công cụ migration đã chọn và cài đặt
- [ ] Migration files được version control trong git
- [ ] Migration user có quyền phù hợp (không dùng superuser)
- [ ] Flyway/Liquibase validate checksum (phát hiện file bị sửa)
- [ ] CI/CD pipeline chạy migration tự động
- [ ] Migration chạy trong staging trước production
- [ ] Không sửa migration file đã chạy (tạo migration mới thay thế)

---

> **Điểm Mấu Chốt:** Công cụ migration quan trọng hơn schema — nó đảm bảo mọi môi trường đều đồng bộ. Chọn một công cụ và dùng nhất quán; sự nhất quán quan trọng hơn việc chọn công cụ "tốt nhất".
