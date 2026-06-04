# Data Mesh — Lưới Dữ Liệu: Domain Ownership & Self-serve Platform

> Data Mesh (Lưới Dữ Liệu) là một paradigm (mẫu tư duy) kiến trúc dữ liệu phi tập trung do Zhamak Dehghani đề xuất năm 2019. Thay vì một team data engineering trung tâm sở hữu và quản lý toàn bộ dữ liệu, Data Mesh phân tán trách nhiệm đến từng domain (miền nghiệp vụ). Mỗi domain tự quản lý dữ liệu của mình và công bố như Data Products (Sản Phẩm Dữ Liệu) có chất lượng đảm bảo.

## 📚 Mục Lục

1. [Tại Sao Cần Data Mesh?](#tại-sao-cần-data-mesh)
2. [Bốn Nguyên Tắc Nền Tảng](#bốn-nguyên-tắc-nền-tảng)
3. [Data Product — Khái Niệm Trung Tâm](#data-product)
4. [Triển Khai trên AWS](#triển-khai-trên-aws)
5. [Ưu Điểm và Thách Thức](#ưu-điểm-và-thách-thức)
6. [Khi Nào Adopt Data Mesh](#khi-nào-adopt-data-mesh)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🚨 Tại Sao Cần Data Mesh?

### Vấn Đề Của Kiến Trúc Tập Trung

Hầu hết tổ chức lớn có một **Centralized Data Team** (Đội Dữ Liệu Tập Trung) quản lý toàn bộ data lake / data warehouse. Khi tổ chức phát triển, mô hình này tạo ra ba nút thắt cổ chai:

```
                    ┌─────────────────────────────────────┐
                    │         DATA PLATFORM TEAM          │
                    │         (Đội Dữ Liệu Tập Trung)     │
                    └─────────────────────────────────────┘
                              ↑↑↑ (BOTTLENECK — Điểm Nghẽn)
                    ┌─────────┼───────────┐──────────┐
                    │         │           │          │
              Marketing    Finance    Sales     Operations
              Team         Team       Team      Team
              "Cần dữ      "Cần dữ    "Cần dữ   "Cần dữ
               liệu gấp"   liệu gấp"  liệu gấp" liệu gấp"
```

**Ba vấn đề cốt lõi:**

1. **Ownership mismatch** (Không khớp quyền sở hữu): Domain Marketing hiểu nhất về dữ liệu marketing của mình, nhưng lại không sở hữu nó. Data team trung tâm sở hữu nhưng không hiểu context nghiệp vụ.

2. **Scalability bottleneck** (Điểm nghẽn khả năng mở rộng): Khi nhiều domain cùng yêu cầu dữ liệu mới, Data Platform Team trở thành nút cổ chai — không đủ bandwidth để phục vụ tất cả.

3. **Context loss** (Mất ngữ cảnh): Khi domain Marketing muốn dữ liệu, họ phải giải thích yêu cầu cho Data Engineer không có kiến thức domain. Ngữ cảnh nghiệp vụ dễ bị mất trong quá trình này.

### Giải Pháp: Phân Tán Trách Nhiệm

```
                    Domain Marketing       Domain Finance
                    ┌───────────────┐     ┌───────────────┐
                    │ Marketing     │     │ Finance       │
                    │ Engineers     │     │ Engineers     │
                    │               │     │               │
                    │ Own & manage  │     │ Own & manage  │
                    │ their data    │     │ their data    │
                    └───────┬───────┘     └───────┬───────┘
                            │  Data Product A     │  Data Product B
                            └──────────┬──────────┘
                                       │
                    ┌──────────────────▼─────────────────┐
                    │    SELF-SERVE DATA PLATFORM         │
                    │    (Nền Tảng Dữ Liệu Tự Phục Vụ)   │
                    │                                     │
                    │  Cung cấp tools, templates,         │
                    │  governance infrastructure          │
                    │  cho tất cả domains sử dụng        │
                    └─────────────────────────────────────┘
```

---

## 🏛️ Bốn Nguyên Tắc Nền Tảng

### Nguyên Tắc 1: Domain-Oriented Ownership (Sở Hữu Hướng Domain)

**Khái niệm:** Mỗi domain nghiệp vụ (Marketing, Sales, Finance, Operations...) **tự sở hữu và quản lý** dữ liệu của mình, từ ingestion (thu nạp) đến serving (phục vụ).

```
Domain Marketing sở hữu:
├── Pipeline thu thập dữ liệu từ ad platforms
├── ETL jobs chuyển đổi raw clicks thành campaign metrics
├── Data quality validation (kiểm tra chất lượng dữ liệu)
└── Data Products: "Campaign Performance", "Attribution Model"

Domain Finance sở hữu:
├── Pipeline nhận dữ liệu từ ERP (Enterprise Resource Planning — Hệ Thống Hoạch Định Nguồn Lực Doanh Nghiệp)
├── ETL jobs tính toán revenue recognition (ghi nhận doanh thu)
├── Data quality validation
└── Data Products: "Revenue Report", "Cost Analysis"
```

**Lợi ích:** Người hiểu domain nhất (domain engineers) quản lý dữ liệu domain đó. Chất lượng dữ liệu tốt hơn vì context không bị mất.

---

### Nguyên Tắc 2: Data as a Product (Dữ Liệu Như Sản Phẩm)

**Khái niệm:** Mỗi dataset được domain publish ra ngoài phải đáp ứng **tiêu chuẩn sản phẩm** — có SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ), documentation (tài liệu), và chất lượng đảm bảo.

**Đặc điểm của một Data Product tốt:**

| Đặc Điểm | Mô Tả | Ví Dụ |
|----------|-------|-------|
| **Discoverable** (Có thể khám phá) | Consumer (người tiêu thụ) có thể tìm thấy | Đăng ký trong Data Catalog |
| **Addressable** (Có thể truy cập) | Có địa chỉ rõ ràng, ổn định | S3 URI hoặc API endpoint |
| **Trustworthy** (Đáng tin cậy) | Có SLA về freshness, accuracy, completeness | "Data updated mỗi 15 phút, 99.9% uptime" |
| **Self-describing** (Tự mô tả) | Schema, lineage (dòng dõi dữ liệu), ví dụ | Glue Catalog metadata đầy đủ |
| **Interoperable** (Tương tác được) | Dùng format chuẩn để tích hợp dễ dàng | Parquet, Avro, JSON Schema |
| **Secure** (Bảo mật) | Kiểm soát truy cập, audit trail | Lake Formation permissions |
| **Natively accessible** (Truy cập tự nhiên) | Nhiều cách truy cập: API, SQL, file | Athena + API + S3 |

---

### Nguyên Tắc 3: Self-serve Data Infrastructure (Hạ Tầng Dữ Liệu Tự Phục Vụ)

**Khái niệm:** Cần có một platform tập trung cung cấp **tools, templates và automation** để các domain engineer (không phải chuyên gia data platform) có thể tự tạo và quản lý data products.

```
Self-serve Platform cung cấp:
├── Data Pipeline Templates (Mẫu Pipeline Dữ Liệu)
│   → Domain engineer chỉ cần cấu hình, không cần code từ đầu
├── Data Quality Framework (Khung Chất Lượng Dữ Liệu)
│   → Tự động validate schema, freshness, completeness
├── Monitoring & Alerting (Giám Sát & Cảnh Báo)
│   → Dashboard theo dõi SLA của từng Data Product
├── Access Control Provisioning (Cấp Phát Kiểm Soát Truy Cập)
│   → Domain tự cấp quyền cho consumer theo policy được định nghĩa trước
└── Data Catalog Integration (Tích Hợp Danh Mục Dữ Liệu)
    → Tự động đăng ký metadata khi publish Data Product
```

**Quan trọng:** Không có Self-serve Platform tốt, Data Mesh sẽ thất bại vì domain engineers không có chuyên môn platform.

---

### Nguyên Tắc 4: Federated Computational Governance (Quản Trị Tính Toán Liên Kết)

**Khái niệm:** Governance (quản trị) không do một team tập trung toàn quyền quyết định, cũng không hoàn toàn phi tập trung. Thay vào đó: **một số quy tắc toàn cục áp dụng cho tất cả domains; các quyết định cục bộ do từng domain tự ra**.

```
Quy Tắc Toàn Cục (Central Governance):
├── Data classification (Phân loại dữ liệu): PII, Sensitive, Public
├── Security standards: Mã hóa, audit logging bắt buộc
├── Interoperability: Format chuẩn (Parquet, Avro), naming conventions
├── Data retention (Lưu giữ dữ liệu): Legal compliance, GDPR
└── SLA minimums: Freshness tối thiểu cho từng loại Data Product

Quyết Định Domain-Level (Local Governance):
├── Schema design (Thiết kế schema) nội bộ
├── Update frequency (Tần suất cập nhật)
├── Specific access policies cho từng consumer
└── Technology stack cho pipeline của domain đó
```

---

## 📦 Data Product — Khái Niệm Trung Tâm

### Anatomy of a Data Product (Cấu Trúc Sản Phẩm Dữ Liệu)

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA PRODUCT                                  │
│                "Campaign Performance"                            │
│                    (Marketing Domain)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  INPUT PORTS (Cổng Đầu Vào):                                    │
│  • Raw click events từ ad platforms                              │
│  • Impression data từ DSP (Demand-Side Platform)                 │
│                                                                  │
│  TRANSFORMATION CODE (Code Chuyển Đổi):                         │
│  • Glue ETL Jobs (Python/PySpark)                                │
│  • Attribution logic (logic quy kết)                             │
│                                                                  │
│  OUTPUT PORTS (Cổng Đầu Ra):                                    │
│  • S3: s3://marketing-domain/products/campaign-perf/v1/          │
│  • Athena Table: marketing_domain.campaign_performance           │
│  • API: GET /api/v1/campaign-performance?date=...                │
│                                                                  │
│  METADATA (Siêu Dữ Liệu):                                       │
│  • Owner: Marketing Analytics Team                               │
│  • SLA: Updated every 15 min, 99.9% availability                 │
│  • Schema: { campaign_id, date, impressions, clicks, spend }     │
│  • Lineage: Từ raw-clicks → campaign-perf (version history)      │
│  • Quality Metrics: completeness ≥ 99%, no nulls on key fields   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Data Product Registry (Sổ Đăng Ký Sản Phẩm Dữ Liệu)

```
# Ví dụ metadata đăng ký trong AWS Glue Data Catalog

database: marketing_domain_products
table: campaign_performance_v1
  location: s3://data-mesh-marketing/products/campaign-performance/v1/
  input_format: org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat
  parameters:
    owner: marketing-analytics-team@company.com
    sla_freshness: PT15M         # ISO 8601 duration: 15 phút
    sla_availability: "99.9%"
    data_classification: internal
    pii_fields: "[]"             # Không có PII
    consumer_teams: ["finance", "sales", "executive"]
    version: "1.2.3"
    changelog: "Added conversion_rate field"
```

---

## ☁️ Triển Khai trên AWS

### AWS Services cho Data Mesh

```
┌────────────────────────────────────────────────────────────────────┐
│              AWS DATA MESH REFERENCE ARCHITECTURE                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  DOMAIN: MARKETING                    DOMAIN: FINANCE               │
│  ┌───────────────────────┐            ┌──────────────────────────┐  │
│  │ AWS Account: mktg-prod│            │ AWS Account: fin-prod    │  │
│  │                       │            │                          │  │
│  │ S3 (data lake)        │            │ S3 (data lake)           │  │
│  │ Glue (ETL)            │            │ Glue (ETL)               │  │
│  │ Glue Catalog (local)  │            │ Glue Catalog (local)     │  │
│  │ Lake Formation (perms)│            │ Lake Formation (perms)   │  │
│  └──────────┬────────────┘            └─────────┬────────────────┘  │
│             │  Share Data Products               │ Share Data Products│
│             └──────────────┬────────────────────┘                   │
│                            │                                         │
│  ┌─────────────────────────▼──────────────────────────────────────┐ │
│  │              CENTRAL GOVERNANCE ACCOUNT                         │ │
│  │                                                                  │ │
│  │  AWS Lake Formation Data Catalog (Cross-account sharing)         │ │
│  │  AWS Glue Data Catalog (Central registry)                        │ │
│  │  AWS IAM (Policies & Roles)                                      │ │
│  │  Amazon DataZone (Data marketplace — Chợ Dữ Liệu)               │ │
│  │  AWS CloudTrail (Audit logging — Ghi Nhật Ký Kiểm Toán)         │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Amazon DataZone — Chợ Dữ Liệu Cho Data Mesh

Amazon DataZone là dịch vụ AWS chuyên thiết kế cho Data Mesh:

```
DataZone Concepts:
├── Domain (Miền): Tổ chức dữ liệu theo business domain
├── Data Asset (Tài Sản Dữ Liệu): Metadata của một dataset/Data Product
├── Project (Dự Án): Workspace cho một team domain
├── Glossary (Từ Điển Nghiệp Vụ): Business terms chuẩn hóa
├── Subscription (Đăng Ký): Consumer request truy cập Data Product
└── Catalog (Danh Mục): Nơi tìm kiếm và khám phá Data Products
```

**Workflow DataZone:**
```
1. Marketing team publish Data Product lên DataZone Catalog
2. Finance team tìm kiếm trong Catalog: "Campaign spend data"
3. Finance team submit subscription request (yêu cầu đăng ký)
4. Marketing team approves (phê duyệt) hoặc auto-approve theo policy
5. Lake Formation tự động cấp quyền cho Finance team
6. Finance team query Athena với Lake Formation permissions
```

### Lake Formation Cross-Account Sharing (Chia Sẻ Liên Tài Khoản)

```python
# Marketing Account: chia sẻ Data Product với Finance Account
import boto3

lakeformation = boto3.client('lakeformation', region_name='us-east-1')

# Cấp quyền cho Finance Account
lakeformation.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': '111222333444'  # Finance AWS Account ID
    },
    Resource={
        'Table': {
            'CatalogId': '999888777666',     # Marketing Account
            'DatabaseName': 'marketing_domain',
            'Name': 'campaign_performance_v1'
        }
    },
    Permissions=['SELECT'],
    PermissionsWithGrantOption=[]  # Finance không thể re-share (chia sẻ lại)
)

# Finance Account: tạo Resource Link để query dữ liệu của Marketing
lakeformation.create_resource_link(
    ResourceLinkInput={
        'Name': 'mktg_campaign_performance',  # Tên local trong Finance account
        'TargetTable': {
            'CatalogId': '999888777666',      # Marketing Account
            'DatabaseName': 'marketing_domain',
            'Name': 'campaign_performance_v1'
        }
    }
)
```

### Glue Data Catalog — Centralized Discovery

```python
# Central Governance Account — Tổng hợp metadata từ tất cả domains
import boto3

glue = boto3.client('glue')

# Đăng ký Data Product từ Marketing domain
glue.create_table(
    DatabaseName='data_mesh_catalog',
    TableInput={
        'Name': 'marketing__campaign_performance__v1',
        'Description': 'Campaign performance metrics từ Marketing domain. SLA: 15 phút.',
        'Parameters': {
            'domain_owner': 'marketing-team',
            'sla_freshness_minutes': '15',
            'data_classification': 'internal',
            'consumer_access': 'finance,sales,executive',
            'product_version': '1.2.3'
        },
        'StorageDescriptor': {
            'Location': 's3://data-mesh-marketing/products/campaign-performance/v1/',
            'InputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat',
            'Columns': [
                {'Name': 'campaign_id', 'Type': 'string'},
                {'Name': 'date', 'Type': 'date'},
                {'Name': 'impressions', 'Type': 'bigint'},
                {'Name': 'clicks', 'Type': 'bigint'},
                {'Name': 'spend', 'Type': 'double'},
                {'Name': 'conversion_rate', 'Type': 'double'}
            ]
        }
    }
)
```

### Self-serve Pipeline Template (Mẫu Pipeline Tự Phục Vụ)

```python
# AWS CDK (Cloud Development Kit) template cho domain teams
# Domain engineer chỉ cần override các parameter, không cần biết Glue internals

from aws_cdk import Stack, aws_glue as glue, aws_s3 as s3
import constructs

class DataProductConstruct(constructs.Construct):
    """
    Reusable construct cho tất cả domain Data Products
    Domain engineer chỉ cần: source_location, transformation_script, product_name
    """
    def __init__(self, scope, id, *,
                 domain_name: str,
                 product_name: str,
                 source_location: str,
                 schedule: str = "cron(0/15 * * * ? *)",
                 owner_email: str):

        super().__init__(scope, id)

        # Auto-create Glue job với best-practice config
        self.glue_job = glue.CfnJob(
            self, f"{product_name}GlueJob",
            name=f"{domain_name}--{product_name}",
            role=f"arn:aws:iam::...:role/domain-glue-role",
            command=glue.CfnJob.JobCommandProperty(
                name="glueetl",
                script_location=f"s3://domain-scripts/{domain_name}/{product_name}/transform.py"
            ),
            default_arguments={
                "--source_location": source_location,
                "--output_location": f"s3://data-mesh-{domain_name}/products/{product_name}/v1/",
                "--enable-glue-datacatalog": "true",
                "--enable-metrics": "true"
            }
        )

        # Auto-register trong Glue Catalog
        self.catalog_table = glue.CfnTable(
            self, f"{product_name}CatalogTable",
            catalog_id=...,
            database_name=f"{domain_name}_products",
            table_input=glue.CfnTable.TableInputProperty(
                name=product_name,
                parameters={"owner": owner_email, "domain": domain_name}
            )
        )
```

---

## ⚖️ Ưu Điểm và Thách Thức

### ✅ Ưu Điểm

| Ưu Điểm | Giải Thích |
|---------|-----------|
| **Scalability** (Khả năng mở rộng) | Thêm domain mới không tạo bottleneck cho team trung tâm |
| **Domain expertise** (Chuyên môn domain) | Domain team hiểu data nhất, chất lượng Data Product cao hơn |
| **Autonomy** (Tự chủ) | Domains tự quyết định tech stack, không chờ central team |
| **Fault isolation** (Cô lập lỗi) | Lỗi ở Marketing pipeline không ảnh hưởng Finance pipeline |
| **Faster time-to-value** | Domain có thể ship Data Products nhanh hơn không cần đợi central team |

### ❌ Thách Thức

| Thách Thức | Giải Thích |
|-----------|-----------|
| **Organizational change** (Thay đổi tổ chức) | Cần đào tạo domain teams về data engineering |
| **Duplication risk** | Nhiều domains có thể tạo ra logic giống nhau độc lập |
| **Governance complexity** | Federated governance khó implement nhất quán |
| **Discoverability** (Khám phá) | Với nhiều Data Products, tìm đúng dataset trở nên khó |
| **Interoperability** | Đảm bảo các Data Products từ khác domains tương thích nhau |
| **High initial investment** | Xây dựng Self-serve Platform tốt tốn nhiều effort ban đầu |

---

## 🎯 Khi Nào Adopt Data Mesh

### ✅ Phù Hợp Khi

```
✓ Tổ chức có ≥ 500 nhân viên kỹ thuật
✓ Nhiều domain nghiệp vụ riêng biệt với data needs khác nhau
✓ Central data team hiện tại là bottleneck rõ ràng
✓ Các domain đã có hoặc có thể có data engineering capability
✓ Sẵn sàng đầu tư vào Self-serve Platform
✓ Leadership (lãnh đạo) ủng hộ organizational change
```

### ❌ Không Phù Hợp Khi

```
✗ Startup hoặc công ty nhỏ (< 100 engineers)
✗ Chỉ có 1-2 data domains
✗ Domain teams không có hoặc không thể build data engineering skills
✗ Thiếu budget cho Self-serve Platform
✗ Governance requirements rất nghiêm ngặt cần kiểm soát tập trung
✗ Leadership không cam kết với organizational transformation
```

### Anti-patterns Cần Tránh

```
❌ "Data Mesh as just technical architecture"
   → Data Mesh là organizational change trước tiên, technology sau
   → Không thể implement Data Mesh mà không thay đổi ownership model

❌ "Domain data silos" (Kho Dữ Liệu Riêng Biệt Của Domain)
   → Data Mesh ≠ mỗi domain làm gì thì làm
   → Cần Federated Governance để đảm bảo interoperability

❌ "Skipping the Self-serve Platform"
   → Không có Self-serve Platform → Domain teams overloaded với platform work
   → Phải build platform trước khi yêu cầu domains self-serve

❌ "Big bang migration" (Di Chuyển Toàn Bộ Cùng Lúc)
   → Start với 1-2 pilot domains, học hỏi, sau đó mở rộng
```

---

## 🎓 Câu Hỏi Phỏng Vấn

### Câu 1: "Data Mesh là gì và khi nào một công ty nên adopt nó?"

**Gợi ý trả lời:**

Data Mesh là paradigm kiến trúc dữ liệu phi tập trung với bốn nguyên tắc: domain-oriented ownership (sở hữu hướng domain), data as a product (dữ liệu như sản phẩm), self-serve data infrastructure (hạ tầng tự phục vụ), và federated computational governance (quản trị liên kết).

Thay vì một Data Platform Team trung tâm quản lý tất cả, mỗi domain nghiệp vụ tự sở hữu và publish Data Products đảm bảo chất lượng. Một central platform cung cấp tools và templates để domains tự làm điều này mà không cần chuyên môn platform.

Nên adopt Data Mesh khi: tổ chức đủ lớn (nhiều domain), data platform team là bottleneck rõ ràng, domains sẵn sàng nhận trách nhiệm data ownership, và leadership cam kết với organizational change.

Không nên adopt khi: công ty nhỏ, chỉ có 1-2 domains, hoặc thiếu resources để xây Self-serve Platform.

---

### Câu 2: "Sự khác biệt giữa Data Mesh và Data Lake truyền thống?"

**Gợi ý trả lời:**

| | Data Lake Truyền Thống | Data Mesh |
|--|----------------------|-----------|
| **Ownership** | Central data team | Distributed domains |
| **Architecture** | Monolithic pipeline | Federated Data Products |
| **Governance** | Centralized | Federated |
| **Scalability** | Bottleneck khi nhiều domains | Linear scale với số domain |
| **Expertise** | Data engineers chuyên biệt | Domain engineers + platform |

Data Lake vẫn có thể là infrastructure nền tảng cho Data Mesh (S3 vẫn là nơi lưu trữ). Sự khác biệt chính là về **organizational model và ownership model**, không chỉ về technology.

---

### Câu 3: "Triển khai Data Mesh trên AWS như thế nào?"

**Gợi ý trả lời:**

AWS cung cấp nhiều dịch vụ phù hợp cho Data Mesh:

1. **Domain isolation:** Mỗi domain có AWS Account riêng (hoặc ít nhất VPC riêng), đảm bảo fault isolation và cost tracking.

2. **Data Product storage:** S3 trên domain account, với Glue Data Catalog cục bộ quản lý metadata.

3. **Cross-account sharing:** AWS Lake Formation hỗ trợ cross-account data sharing — domain A cấp quyền cho domain B truy cập Data Product mà không cần copy dữ liệu.

4. **Central discovery:** Amazon DataZone là managed service của AWS cho Data Mesh marketplace — domains publish, consumers discover và subscribe Data Products.

5. **Governance:** Lake Formation tag-based access control (LF-Tags) cho phép define policies một lần áp dụng cho nhiều tables — phù hợp federated governance.

6. **Self-serve platform:** AWS CDK (Infrastructure as Code) với Constructs tái sử dụng để domain teams tự deploy pipelines theo template chuẩn.

---

## 📊 Tổng Kết

```
Data Mesh = Domain Ownership + Data as Product + Self-serve Platform + Federated Governance

KHÔNG phải:
  ✗ Data Mesh không phải là technology hay architecture mới
  ✗ Data Mesh không phải là kho dữ liệu riêng biệt của từng team

LÀ:
  ✓ Organizational paradigm shift (chuyển dịch mô hình tổ chức)
  ✓ Phân tán trách nhiệm đến domain experts
  ✓ Kết hợp autonomy (tự chủ) và governance (quản trị) cân bằng

Trên AWS:
  → Amazon DataZone cho Data Marketplace (Chợ Dữ Liệu)
  → AWS Lake Formation cho cross-account data sharing
  → Glue Data Catalog cho central metadata
  → AWS CDK cho self-serve pipeline templates
```

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn thành
**Tiếp Theo:** [4-medallion-architecture.md](./4-medallion-architecture.md)
