# 2 — Lake Formation Security — Bảo Mật & Kiểm Soát Truy Cập

> AWS Lake Formation cung cấp fine-grained access control (kiểm soát truy cập tinh tế) ở cấp độ bảng, cột, hàng, và cell — vượt xa khả năng của IAM S3 policies truyền thống. Phần này giải thích chi tiết từng cơ chế bảo mật và cách triển khai thực tế.

---

## 🔐 Mô Hình Bảo Mật Của Lake Formation

### Hai Lớp Bảo Mật Kết Hợp

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Kiểm Tra Quyền Truy Cập                          │
│                                                                      │
│  Request từ User / Role                                              │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────────────┐                                             │
│  │   IAM Policy Check  │  ← Lớp 1: IAM phải cho phép gọi           │
│  │   (Kiểm Tra IAM)    │    Glue/Athena/Redshift Spectrum           │
│  └─────────┬───────────┘                                             │
│            │ Passed (Vượt Qua)                                       │
│            ▼                                                         │
│  ┌─────────────────────────────────┐                                 │
│  │   Lake Formation Permission     │  ← Lớp 2: Lake Formation       │
│  │   Check (Kiểm Tra Quyền LF)     │    kiểm tra có quyền truy      │
│  │                                 │    cập bảng/cột/hàng không     │
│  │   • Table permissions           │                                 │
│  │   • Column permissions          │                                 │
│  │   • Row filters                 │                                 │
│  │   • LF-Tag expressions          │                                 │
│  └─────────┬───────────────────────┘                                 │
│            │ Granted (Được Cấp)                                      │
│            ▼                                                         │
│       Truy Cập Dữ Liệu ✅                                            │
└──────────────────────────────────────────────────────────────────────┘
```

**Nguyên tắc quan trọng:** Cả hai lớp đều phải pass — IAM **VÀ** Lake Formation. Nếu IAM allow nhưng LF deny → bị từ chối. Nếu LF allow nhưng IAM deny → bị từ chối.

---

## 👑 Lake Formation Administrators — Quản Trị Viên

### Data Lake Administrator (Quản Trị Viên Hồ Dữ Liệu)

Là IAM user/role có toàn quyền trong Lake Formation:
- Đăng ký S3 locations làm data lake storage
- Tạo/xóa databases trong Glue Catalog
- Cấp quyền cho principal (user/role) khác
- Tạo và gán LF-Tags

```bash
# Thêm Data Lake Administrator qua AWS CLI
aws lakeformation put-data-lake-settings \
  --data-lake-settings '{
    "DataLakeAdmins": [
      {
        "DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/DataLakeAdminRole"
      }
    ]
  }'
```

### Data Steward (Quản Lý Dữ Liệu)

Không phải admin đầy đủ, nhưng có thể:
- Cấp quyền trên subset dữ liệu họ được giao quản lý
- Tạo LF-Tags trong phạm vi được cho phép

---

## 🗂️ Resource-Based Permissions — Phân Quyền Theo Tài Nguyên

### Table Permissions (Quyền Trên Bảng)

```python
import boto3

lf_client = boto3.client('lakeformation')

# Cấp quyền SELECT (Chọn) trên bảng cho một role
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/AnalystRole'
    },
    Resource={
        'Table': {
            'DatabaseName': 'analytics_db',
            'Name': 'customer_transactions'
        }
    },
    Permissions=['SELECT'],
    PermissionsWithGrantOption=[]  # Không cho phép re-grant (cấp lại cho người khác)
)
```

### Các Loại Permissions

| Permission | Mô Tả | Áp Dụng Cho |
| ---------- | ----- | ----------- |
| `SELECT` | Đọc dữ liệu | Table, Column, Database |
| `INSERT` | Thêm dữ liệu | Table |
| `DELETE` | Xóa dữ liệu | Table |
| `ALTER` | Thay đổi schema | Table, Database |
| `CREATE_TABLE` | Tạo bảng mới | Database |
| `DROP` | Xóa bảng/database | Table, Database |
| `DESCRIBE` | Xem metadata nhưng không thấy dữ liệu | Table, Database |
| `DATA_LOCATION_ACCESS` | Truy cập S3 location | S3 Location |

---

## 🔒 Column-Level Security — Bảo Mật Cấp Độ Cột

### Ẩn Cột Nhạy Cảm

Dùng để bảo vệ PII (Personally Identifiable Information — Thông Tin Nhận Dạng Cá Nhân):

```python
# Cấp SELECT chỉ trên một số cột — ẩn cột email và ssn
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/JuniorAnalystRole'
    },
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'analytics_db',
            'Name': 'customers',
            'ColumnNames': ['customer_id', 'name', 'country', 'signup_date']
            # Không bao gồm: email, phone, ssn, credit_card_number
        }
    },
    Permissions=['SELECT']
)
```

### Column Wildcard với Exclusion (Ngoại Lệ)

Thay vì liệt kê cột được phép, liệt kê cột bị ẩn:

```python
# Cho phép tất cả cột TRỪ các cột PII
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/AnalystRole'
    },
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'analytics_db',
            'Name': 'customers',
            'ColumnWildcard': {
                'ExcludedColumnNames': ['ssn', 'credit_card_number', 'bank_account']
            }
        }
    },
    Permissions=['SELECT']
)
```

---

## 🔍 Row-Level Security — Bảo Mật Cấp Độ Hàng

### Data Filters (Bộ Lọc Dữ Liệu)

Row-level security trong Lake Formation dùng **data filters** — biểu thức SQL xác định hàng nào được phép truy cập:

```python
# Bước 1: Tạo data filter
lf_client.create_data_cells_filter(
    TableData={
        'TableCatalogId': '123456789012',
        'DatabaseName': 'analytics_db',
        'TableName': 'sales_data',
        'Name': 'us_east_filter',
        'RowFilter': {
            'FilterExpression': "region = 'us-east-1'"
            # Chỉ các hàng có region = 'us-east-1' mới hiển thị
        },
        'ColumnWildcard': {}  # Tất cả cột đều hiển thị
    }
)

# Bước 2: Gán filter cho role
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/USEastAnalystRole'
    },
    Resource={
        'DataCellsFilter': {
            'TableCatalogId': '123456789012',
            'DatabaseName': 'analytics_db',
            'TableName': 'sales_data',
            'Name': 'us_east_filter'
        }
    },
    Permissions=['SELECT']
)
```

### Kết Hợp Row + Column Filter

```python
# Filter: chỉ thấy hàng của region us-east-1, và ẩn cột salary
lf_client.create_data_cells_filter(
    TableData={
        'TableCatalogId': '123456789012',
        'DatabaseName': 'analytics_db',
        'TableName': 'employees',
        'Name': 'us_east_no_salary_filter',
        'RowFilter': {
            'FilterExpression': "region = 'us-east-1' AND department != 'Executive'"
        },
        'ColumnNames': ['employee_id', 'name', 'role', 'department', 'region']
        # Ẩn cột: salary, bonus, performance_rating
    }
)
```

---

## 🏷️ LF-Tags — Tag-Based Access Control (Kiểm Soát Truy Cập Theo Thẻ)

### LF-Tags Là Gì?

**LF-Tags** (Lake Formation Tags — Nhãn Lake Formation) là cơ chế ABAC (Attribute-Based Access Control — Kiểm Soát Truy Cập Theo Thuộc Tính). Thay vì cấp quyền từng resource riêng lẻ, bạn:

1. Gán tag lên database/table/column
2. Cấp quyền cho principal dựa trên biểu thức tag

```
Truyền thống (Resource-based):
- Cấp quyền table_A cho role_1
- Cấp quyền table_B cho role_1
- Cấp quyền table_C cho role_1
- ... (100 bảng = 100 grants)

LF-Tags (Attribute-based):
- Gán tag domain=finance lên table_A, table_B, table_C
- Cấp quyền: role_1 được đọc tất cả resource có tag domain=finance
- Khi tạo table_D với tag domain=finance → role_1 tự động có quyền ✅
```

### Tạo và Quản Lý LF-Tags

```python
# Tạo LF-Tag key với các giá trị cho phép
lf_client.create_lf_tag(
    TagKey='classification',
    TagValues=['public', 'internal', 'confidential', 'restricted']
)

lf_client.create_lf_tag(
    TagKey='domain',
    TagValues=['finance', 'marketing', 'hr', 'engineering', 'product']
)

lf_client.create_lf_tag(
    TagKey='environment',
    TagValues=['dev', 'staging', 'production']
)

# Gán tag lên database
lf_client.add_lf_tags_to_resource(
    Resource={
        'Database': {
            'Name': 'finance_analytics'
        }
    },
    LFTags=[
        {'TagKey': 'domain', 'TagValues': ['finance']},
        {'TagKey': 'environment', 'TagValues': ['production']}
    ]
)

# Gán tag lên cột cụ thể (cột chứa PII)
lf_client.add_lf_tags_to_resource(
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'finance_analytics',
            'Name': 'customers',
            'ColumnNames': ['ssn', 'credit_card_number']
        }
    },
    LFTags=[
        {'TagKey': 'classification', 'TagValues': ['restricted']}
    ]
)
```

### Cấp Quyền Dựa Trên LF-Tags

```python
# Cấp quyền: Finance team được đọc mọi resource có tag domain=finance
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/FinanceTeamRole'
    },
    Resource={
        'LFTagPolicy': {
            'ResourceType': 'TABLE',
            'Expression': [
                {
                    'TagKey': 'domain',
                    'TagValues': ['finance']
                },
                {
                    'TagKey': 'environment',
                    'TagValues': ['production']
                }
            ]
        }
    },
    Permissions=['SELECT', 'DESCRIBE']
)

# Senior Data Steward được đọc tất cả classification TRỪ restricted
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/SeniorAnalystRole'
    },
    Resource={
        'LFTagPolicy': {
            'ResourceType': 'TABLE',
            'Expression': [
                {
                    'TagKey': 'classification',
                    'TagValues': ['public', 'internal', 'confidential']
                    # Không bao gồm 'restricted'
                }
            ]
        }
    },
    Permissions=['SELECT']
)
```

---

## 🌐 Cross-Account Data Sharing — Chia Sẻ Dữ Liệu Giữa Tài Khoản

### Kiến Trúc Cross-Account

```
┌─────────────────────────────────┐    ┌────────────────────────────────┐
│   Account A — Producer          │    │   Account B — Consumer         │
│   (Tài Khoản Nhà Sản Xuất)     │    │   (Tài Khoản Người Dùng)       │
│                                 │    │                                 │
│  ┌──────────────────┐           │    │  ┌────────────────────────┐    │
│  │  Glue Catalog    │──────────▶│    │  │  Shared Catalog Entry  │    │
│  │  (database A)    │  AWS RAM  │    │  │  (read-only view)      │    │
│  └──────────────────┘  Share    │    │  └──────────┬─────────────┘    │
│                                 │    │             │                   │
│  ┌──────────────────┐           │    │  ┌──────────▼─────────────┐    │
│  │  S3 Bucket A     │◀──────────│────│──│  Athena Query          │    │
│  │  (data stays     │  Direct   │    │  │  (queries S3 in A)     │    │
│  │   in Account A)  │  S3 read  │    │  └────────────────────────┘    │
│  └──────────────────┘           │    │                                 │
└─────────────────────────────────┘    └────────────────────────────────┘
```

### Thiết Lập Cross-Account Sharing

```python
# Account A: Cấp quyền cho Account B
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': '987654321098'  # AWS Account ID của Account B
    },
    Resource={
        'Table': {
            'DatabaseName': 'shared_analytics',
            'Name': 'public_metrics'
        }
    },
    Permissions=['SELECT', 'DESCRIBE'],
    PermissionsWithGrantOption=['SELECT']  # Account B có thể grant tiếp cho user trong B
)

# Account B: Tạo resource link (liên kết tài nguyên) trỏ về shared resource
lf_client_b.create_database(
    DatabaseInput={
        'Name': 'linked_analytics',
        'TargetDatabase': {
            'CatalogId': '123456789012',  # Account A
            'DatabaseName': 'shared_analytics'
        }
    }
)
```

---

## 🔑 Governed Tables — Bảng Được Quản Lý

**Governed Tables** (Bảng Được Quản Lý) là tính năng mới trong Lake Formation cung cấp:
- **ACID Transactions** — Insert, Update, Delete an toàn
- **Automatic Compaction** — Lake Formation tự compact file nhỏ
- **Time Travel** — Query dữ liệu tại thời điểm cụ thể trong quá khứ

```sql
-- Tạo Governed Table qua Athena
CREATE TABLE governed_transactions (
  transaction_id string,
  customer_id string,
  amount decimal(10,2),
  transaction_date date
)
LOCATION 's3://my-lake/governed/transactions/'
TBLPROPERTIES ('lakeformation.governed'='true');

-- Transaction an toàn
BEGIN;
INSERT INTO governed_transactions VALUES ('T001', 'C123', 99.99, DATE '2024-01-15');
UPDATE governed_transactions SET amount = 109.99 WHERE transaction_id = 'T001';
COMMIT;
```

---

## 📋 Best Practices — Thực Hành Tốt Nhất

### 1. Dùng LF-Tags Cho Môi Trường Nhiều Team

```
Nguyên tắc: Resource-based permissions phù hợp cho < 20 dataset.
LF-Tags phù hợp khi có > 20 dataset và nhiều team.

Taxonomy (Phân Loại) LF-Tags gợi ý:
- domain: finance / marketing / hr / product / engineering
- classification: public / internal / confidential / restricted / pii
- environment: dev / staging / production
- region: us-east-1 / eu-west-1 / ap-southeast-1
```

### 2. Principle of Least Privilege (Nguyên Tắc Quyền Tối Thiểu)

```python
# ❌ Sai: Cấp quyền quá rộng
grant_permissions(role='AnalystRole', resource='ALL_TABLES', permission='SELECT')

# ✅ Đúng: Chỉ cấp quyền cần thiết
grant_permissions(
    role='JuniorAnalystRole',
    resource='Table:analytics_db.marketing_summary',
    permission='SELECT',
    columns=['campaign_name', 'impressions', 'clicks', 'date']
    # Không bao gồm cost data
)
```

### 3. Tách Biệt Quyền Admin và Quyền Dữ Liệu

```
Data Lake Admin:
  - Người cấp quyền (granter)
  - KHÔNG nhất thiết có quyền đọc dữ liệu

Data User:
  - Người được cấp quyền (grantee)
  - Có quyền đọc subset dữ liệu

Data Steward (Quản Lý Dữ Liệu):
  - Có thể grant permissions trong domain của mình
  - Không thể grant beyond their own permissions
```

### 4. Kiểm Toán Thường Xuyên (Regular Audit)

```bash
# Xem tất cả permissions hiện tại trong Lake Formation
aws lakeformation list-permissions \
  --resource-type TABLE \
  --query 'PrincipalResourcePermissions[*].{
    Principal: Principal.DataLakePrincipalIdentifier,
    Database: Resource.Table.DatabaseName,
    Table: Resource.Table.Name,
    Permissions: Permissions
  }' \
  --output table

# Xem CloudTrail logs để audit ai đã truy cập gì
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetDataAccess \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-31T23:59:59Z
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Lake Formation permissions khác gì với IAM S3 bucket policies?**

> IAM/S3 policies kiểm soát ở cấp độ S3 prefix (thư mục/file) — ai được đọc file nào. Lake Formation permissions kiểm soát ở cấp độ logic Catalog — ai được đọc bảng nào, cột nào, hàng nào. Lake Formation là lớp bổ sung phía trên IAM: cả hai phải cho phép thì mới truy cập được. Ví dụ: dùng IAM để allow `s3:GetObject` trên toàn bucket, nhưng dùng Lake Formation để chỉ cho phép team Finance thấy database `finance_*` và ẩn cột salary với Junior Analyst.

**Q: Khi nào dùng resource-based permissions vs LF-Tags?**

> Resource-based permissions phù hợp khi có ít dataset (< 20 bảng) và ít principal — dễ hiểu và trace. LF-Tags phù hợp khi có nhiều dataset và nhiều team — cho phép cấp quyền "tự động" khi dataset mới được tạo với đúng tag. LF-Tags cũng phù hợp cho compliance requirements (ví dụ: "mọi dữ liệu có classification=pii chỉ được phép bởi nhóm DataPrivacy").

**Q: Làm thế nào để implement multi-tenant row-level security với Lake Formation?**

> Tạo một data filter cho mỗi tenant với `FilterExpression = "tenant_id = 'TENANT_X'"`, sau đó gán filter đó cho role của tenant đó. Mỗi tenant chỉ thấy hàng của mình khi query qua Athena hay Redshift Spectrum. Nếu có nhiều tenant, có thể dùng biến session — nhưng hiện tại Lake Formation chưa hỗ trợ dynamic row filters, nên mỗi tenant cần một filter riêng.

**Q: Cross-account data sharing trong Lake Formation — dữ liệu có bị copy sang account khác không?**

> Không. Dữ liệu vẫn nằm ở S3 bucket của Account Producer. Account Consumer chỉ nhận được quyền đọc metadata (Catalog) và quyền truy cập S3 bucket của Producer. Khi Athena ở Account Consumer query, nó sẽ đọc trực tiếp từ S3 của Account Producer. Điều này giúp không tốn chi phí storage nhân đôi và dữ liệu luôn đồng bộ.

---

**Tiếp Theo:** [3-s3-data-lake.md](./3-s3-data-lake.md) — S3 làm nền tảng data lake: lifecycle policies, storage classes, Intelligent-Tiering

**Cập Nhật Lần Cuối:** 2026-05-17
