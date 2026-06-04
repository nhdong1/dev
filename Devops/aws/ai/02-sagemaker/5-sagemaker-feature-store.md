# SageMaker Feature Store — Kho Đặc Trưng Trung Tâm

> SageMaker Feature Store là dịch vụ quản lý tập trung **features** (đặc trưng — các biến đầu vào) cho Machine Learning. Thay vì mỗi team ML tự tính features riêng lẻ và không nhất quán, Feature Store cung cấp một nguồn dữ liệu duy nhất (single source of truth) để định nghĩa, lưu trữ và tái sử dụng features cho cả training và real-time inference.

---

## 📚 Mục Lục

1. [Feature Store Là Gì và Tại Sao Cần?](#feature-store-là-gì)
2. [Kiến Trúc: Online Store và Offline Store](#kiến-trúc)
3. [Feature Group — Nhóm Đặc Trưng](#feature-group)
4. [Feature Ingestion — Nạp Dữ Liệu](#feature-ingestion)
5. [Feature Retrieval — Truy Xuất Đặc Trưng](#feature-retrieval)
6. [Feature Store trong ML Workflow](#feature-store-trong-ml-workflow)
7. [Khi Nào Thực Sự Cần Feature Store](#khi-nào-thực-sự-cần)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Feature Store Là Gì và Tại Sao Cần? {#feature-store-là-gì}

### Vấn Đề Trong ML Production Không Có Feature Store

```
Năm 2022: Team A build Churn Prediction Model
  → Tự tính feature "days_since_last_purchase"
  → Logic: (today - max(order_date)) / 86400

Năm 2023: Team B build Recommendation Model
  → Cũng cần "days_since_last_purchase"
  → Tự tính lại với logic hơi khác: (now - last_order_ts).days
  → Kết quả khác nhau vì múi giờ không nhất quán!

Training vs Serving Skew (Độ Lệch Huấn Luyện vs Phục Vụ):
  Training: Tính features từ data warehouse (Kho Dữ Liệu) với query batch
  Serving:  Tính features trong real-time từ MySQL với query khác
  → Cùng logic nhưng kết quả khác → Model performance giảm production
```

### Feature Store Giải Quyết Như Thế Nào

```
Mọi Feature đều định nghĩa một lần trong Feature Store:

"days_since_last_purchase":
  - Định nghĩa chuẩn: (event_time - last_purchase_ts) in days
  - Cập nhật: Real-time khi có đơn hàng mới
  - Phiên bản: Có lịch sử thay đổi

Team A (Training):  Lấy features từ Offline Store → nhất quán 100%
Team B (Inference): Lấy features từ Online Store → cùng logic, độ trễ thấp
```

### Lợi Ích Cụ Thể

| Lợi Ích | Mô Tả |
|---|---|
| **Feature Reuse** (Tái Sử Dụng) | 1 feature tính 1 lần, dùng cho nhiều model |
| **Training/Serving Consistency** (Nhất Quán) | Cùng feature logic cho cả train và serve |
| **Point-in-time Correctness** (Chính Xác Theo Thời Điểm) | Lấy feature value tại đúng thời điểm lịch sử khi train |
| **Discovery** (Khám Phá) | Catalog tìm features có sẵn thay vì tạo lại |
| **Governance** (Quản Trị) | Access control, lineage tracking cho features |

---

## Kiến Trúc: Online Store và Offline Store {#kiến-trúc}

### Tổng Quan Kiến Trúc

```
                      INGESTION (Nạp Dữ Liệu)
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
    ┌─────────────────┐             ┌─────────────────┐
    │   Online Store  │             │  Offline Store  │
    │  (Kho Trực      │             │  (Kho Ngoại     │
    │   Tuyến)        │             │   Tuyến)        │
    │                 │             │                 │
    │  Amazon         │             │  Amazon S3      │
    │  DynamoDB-like  │             │  (Parquet files)│
    │  ~1ms latency   │             │  (Apache Iceberg│
    │  (Độ Trễ ~1ms)  │             │   table format) │
    └────────┬────────┘             └────────┬────────┘
             │                               │
             ▼                               ▼
    Real-time Inference              Training &
    (Suy Luận Thời Gian Thực)        Batch Inference
    Lấy features cho một             Lấy toàn bộ history
    record cụ thể trong ms           để build training dataset
```

### Online Store (Kho Đặc Trưng Trực Tuyến)

**Đặc điểm:**
- Độ trễ thấp (~1ms): Dùng trong real-time inference pipeline
- Lưu **giá trị mới nhất** của mỗi entity (mỗi customer, mỗi product)
- Không hỗ trợ historical lookups (truy vấn lịch sử)
- AWS quản lý underlying storage (DynamoDB-based)

**Use case:** Khi model đang serving cần nhanh chóng lấy features của customer_id hiện tại

```python
import boto3

sagemaker_runtime = boto3.client('sagemaker-featurestore-runtime')

# Lấy features của customer 12345 ngay lập tức
response = sagemaker_runtime.get_record(
    FeatureGroupName='customer-features',
    RecordIdentifierValueAsString='12345'  # customer_id
)

features = {f['FeatureName']: f['ValueAsString'] 
            for f in response['Record']}
print(features)
# {
#   'customer_id': '12345',
#   'days_since_last_purchase': '7',
#   'total_lifetime_value': '1250.50',
#   'churn_risk_score': '0.23'
# }
```

### Offline Store (Kho Đặc Trưng Ngoại Tuyến)

**Đặc điểm:**
- Lưu **toàn bộ lịch sử** thay đổi của features
- Lưu trên S3 dưới dạng Apache Parquet (định dạng columnar — dạng cột)
- Hỗ trợ point-in-time query (truy vấn theo thời điểm lịch sử)
- Query qua Amazon Athena hoặc AWS Glue

**Use case:** Tạo training dataset với features tại đúng thời điểm lịch sử

---

## Feature Group — Nhóm Đặc Trưng {#feature-group}

**Feature Group** là đơn vị tổ chức cơ bản trong Feature Store — tập hợp các features liên quan đến cùng một entity.

### Các Thành Phần Bắt Buộc

```python
from sagemaker.feature_store.feature_group import FeatureGroup
from sagemaker.feature_store.feature_definition import (
    FeatureDefinition, FeatureTypeEnum
)

# Định nghĩa Feature Group cho customer features
feature_group = FeatureGroup(
    name='customer-features',
    sagemaker_session=session
)

# Định nghĩa schema (cấu trúc) của features
feature_definitions = [
    FeatureDefinition(
        feature_name='customer_id',           # Record Identifier — bắt buộc
        feature_type=FeatureTypeEnum.STRING
    ),
    FeatureDefinition(
        feature_name='event_time',            # Event Time — bắt buộc
        feature_type=FeatureTypeEnum.FRACTIONAL  # Unix timestamp
    ),
    FeatureDefinition(
        feature_name='days_since_last_purchase',
        feature_type=FeatureTypeEnum.INTEGRAL   # Integer
    ),
    FeatureDefinition(
        feature_name='total_lifetime_value',
        feature_type=FeatureTypeEnum.FRACTIONAL  # Float
    ),
    FeatureDefinition(
        feature_name='customer_segment',
        feature_type=FeatureTypeEnum.STRING
    ),
    FeatureDefinition(
        feature_name='avg_order_value_30d',
        feature_type=FeatureTypeEnum.FRACTIONAL
    )
]
```

### Tạo Feature Group

```python
import time

feature_group.load_feature_definitions(data_frame=df)

# Tạo Feature Group với cả Online và Offline Store
feature_group.create(
    s3_uri=f's3://{bucket}/feature-store/',          # Offline Store location
    record_identifier_name='customer_id',             # Khóa chính (primary key)
    event_time_feature_name='event_time',             # Timestamp của record
    role_arn=role,
    enable_online_store=True,                          # Bật Online Store
    description="Customer behavioral features for ML models"
)

# Đợi Feature Group sẵn sàng
feature_group.wait_for_state_change()
print(f"Feature Group status: Created")
```

### Feature Types (Loại Dữ Liệu)

| Type | Python Equiv | Dùng Cho |
|---|---|---|
| `STRING` | str | ID, category, text |
| `INTEGRAL` | int | Counts, ranks, integer metrics |
| `FRACTIONAL` | float | Prices, rates, probabilities, timestamps |

### Record Identifier và Event Time

**Record Identifier** (Định Danh Bản Ghi): Khóa chính — dùng để lookup record cụ thể.
- VD: `customer_id`, `product_id`, `session_id`

**Event Time** (Thời Gian Sự Kiện): Khi nào feature value này hợp lệ — bắt buộc phải có.
- Dùng Unix timestamp (epoch seconds)
- Cho phép Feature Store lưu nhiều versions theo thời gian
- Critical cho point-in-time correctness khi tạo training data

---

## Feature Ingestion — Nạp Dữ Liệu {#feature-ingestion}

### Batch Ingestion (Nạp Theo Lô)

```python
import pandas as pd
import time

# Tạo DataFrame với features
features_df = pd.DataFrame({
    'customer_id': ['C001', 'C002', 'C003'],
    'event_time': [time.time(), time.time(), time.time()],  # Now
    'days_since_last_purchase': [7, 45, 3],
    'total_lifetime_value': [1250.50, 89.00, 4500.00],
    'customer_segment': ['premium', 'new', 'vip'],
    'avg_order_value_30d': [125.05, 44.50, 450.00]
})

# Ingest toàn bộ DataFrame
feature_group.ingest(
    data_frame=features_df,
    max_workers=3,   # Song song hóa
    wait=True
)
print("Ingestion completed!")
```

### Streaming Ingestion (Nạp Liên Tục)

```python
import boto3
import time

runtime = boto3.client('sagemaker-featurestore-runtime')

# Ingest 1 record ngay lập tức (VD: sau khi customer đặt hàng)
def update_customer_features_after_purchase(customer_id, purchase_amount):
    # Tính features mới sau purchase
    new_record = [
        {'FeatureName': 'customer_id', 'ValueAsString': str(customer_id)},
        {'FeatureName': 'event_time', 'ValueAsString': str(time.time())},
        {'FeatureName': 'days_since_last_purchase', 'ValueAsString': '0'},
        {'FeatureName': 'total_lifetime_value', 'ValueAsString': str(purchase_amount)},
        {'FeatureName': 'customer_segment', 'ValueAsString': 'active'},
        {'FeatureName': 'avg_order_value_30d', 'ValueAsString': str(purchase_amount)}
    ]
    
    runtime.put_record(
        FeatureGroupName='customer-features',
        Record=new_record
    )
    print(f"Features updated for customer {customer_id}")

# Gọi khi có đơn hàng mới
update_customer_features_after_purchase('C001', 299.99)
```

### SageMaker Processing Job cho Batch Ingestion Lớn

```python
from sagemaker.processing import ScriptProcessor

# Dùng Processing Job để ingest data lớn từ S3
processor = ScriptProcessor(
    image_uri=sagemaker.image_uris.retrieve('sklearn', region, '0.23-1'),
    role=role,
    instance_count=2,      # 2 instances song song
    instance_type='ml.m5.xlarge'
)

processor.run(
    code='ingest_features.py',
    inputs=[ProcessingInput(
        source='s3://bucket/raw-features/',
        destination='/opt/ml/processing/input'
    )],
    arguments=['--feature-group-name', 'customer-features']
)
```

---

## Feature Retrieval — Truy Xuất Đặc Trưng {#feature-retrieval}

### Real-time Lookup từ Online Store

```python
import boto3

runtime = boto3.client('sagemaker-featurestore-runtime')

# Lấy features của 1 customer
def get_customer_features(customer_id):
    response = runtime.get_record(
        FeatureGroupName='customer-features',
        RecordIdentifierValueAsString=str(customer_id),
        FeatureNames=['days_since_last_purchase', 'total_lifetime_value', 'customer_segment']
    )
    
    if 'Record' not in response:
        return None  # Customer chưa có features
    
    return {f['FeatureName']: f['ValueAsString'] for f in response['Record']}

# Batch lookup nhiều customers cùng lúc
def batch_get_customer_features(customer_ids):
    identifiers = [
        {'FeatureGroupName': 'customer-features',
         'RecordIdentifiersValueAsString': [str(cid)]}
        for cid in customer_ids
    ]
    
    response = runtime.batch_get_record(Identifiers=identifiers)
    return response['Records']

# Dùng trong real-time inference pipeline
features = get_customer_features('C001')
prediction = model.predict([features])
```

### Tạo Training Dataset từ Offline Store

```python
# Query Offline Store qua Athena (Dịch Vụ Query Serverless)
import boto3
import time

athena = boto3.client('athena')

# Point-in-time query: Lấy feature values tại đúng thời điểm lịch sử
# Tránh data leakage (rò rỉ dữ liệu tương lai vào training)
query = """
WITH latest_features AS (
    -- Lấy feature mới nhất TRƯỚC mỗi thời điểm trong training data
    SELECT 
        customer_id,
        days_since_last_purchase,
        total_lifetime_value,
        customer_segment,
        event_time,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY event_time DESC
        ) as rn
    FROM "sagemaker_featurestore"."customer_features"
    WHERE event_time <= 1700000000  -- Chỉ lấy features TRƯỚC thời điểm này
)
SELECT customer_id, days_since_last_purchase, total_lifetime_value, customer_segment
FROM latest_features
WHERE rn = 1  -- Chỉ lấy giá trị mới nhất
"""

# Chạy Athena query
response = athena.start_query_execution(
    QueryString=query,
    ResultConfiguration={'OutputLocation': 's3://bucket/athena-results/'}
)

query_id = response['QueryExecutionId']

# Đợi kết quả
while True:
    result = athena.get_query_execution(QueryExecutionId=query_id)
    state = result['QueryExecution']['Status']['State']
    if state in ['SUCCEEDED', 'FAILED']:
        break
    time.sleep(5)

print(f"Query {state}")
```

### Feature Store SDK Convenience Method

```python
from sagemaker.feature_store.feature_store import FeatureStore

feature_store = FeatureStore(sagemaker_session=session)

# Tạo training dataset từ nhiều Feature Groups
dataset_builder = feature_store.create_dataset(
    base=training_events_df,  # DataFrame với event timestamps
    output_path=f's3://{bucket}/training-data/'
)

# Join features từ nhiều groups
dataset_builder = dataset_builder \
    .with_feature_group(
        feature_group=customer_fg,
        target_feature_name_in_base='customer_id',
        included_feature_names=['days_since_last_purchase', 'total_lifetime_value']
    ) \
    .with_feature_group(
        feature_group=product_fg,
        target_feature_name_in_base='product_id',
        included_feature_names=['category', 'price_tier']
    )

# Chạy và lấy dataset
training_data, job_name = dataset_builder.to_dataframe()
```

---

## Feature Store trong ML Workflow {#feature-store-trong-ml-workflow}

### Luồng Hoàn Chỉnh với Feature Store

```
1. ĐỊNH NGHĨA FEATURES (Feature Engineering Team)
   ─────────────────────────────────────────────
   a. Tạo Feature Group schema
   b. Viết feature computation logic (chuẩn hóa, 1 version duy nhất)
   c. Đăng ký vào Feature Store Catalog

2. INGEST FEATURES (Data Pipeline Team)
   ────────────────────────────────────
   a. Batch ingest từ data warehouse (hàng ngày/tuần)
   b. Streaming ingest khi có events real-time
   → Online Store: features hiện tại
   → Offline Store: toàn bộ lịch sử

3. TRAINING (ML Engineering Team)
   ─────────────────────────────
   a. Lấy features từ Offline Store với point-in-time query
   b. Join với labels
   c. Train model với SageMaker Training Job

4. INFERENCE (Application Team)
   ──────────────────────────
   a. Request đến: {"customer_id": "C001", "context": {...}}
   b. Lấy features ngay lập tức từ Online Store (~1ms)
   c. Merge với request context
   d. Call model endpoint → prediction

   Kết quả: Training features = Serving features → Không có Training/Serving Skew
```

### Training/Serving Skew — Nguyên Nhân Số 1 Model Underperform

```python
# ❌ KHÔNG TỐT — Training/Serving Skew
# Training: features từ Spark job
customer_age_days = spark_df.select(
    datediff(current_date(), col('signup_date'))
).collect()

# Serving: features từ Python (logic khác một chút!)
customer_age_days = (datetime.now() - signup_date).days  # Có thể khác vì timezone!

# ✅ TỐT — Dùng Feature Store
# Training: Offline Store query → feature "customer_age_days"
# Serving:  Online Store lookup → cùng feature "customer_age_days"
# Cùng 1 logic định nghĩa trong Feature Group → 100% nhất quán
```

---

## Khi Nào Thực Sự Cần Feature Store {#khi-nào-thực-sự-cần}

### Cần Feature Store Khi:

```
✅ Nhiều teams (≥2) dùng chung features
✅ Features cần real-time update VÀ training cần historical data
✅ Gặp vấn đề Training/Serving Skew trong production
✅ Tổ chức muốn feature catalog để tái sử dụng
✅ Compliance yêu cầu audit trail của feature data
✅ ≥5-10 models trong production dùng chung feature sets
```

### KHÔNG Cần Feature Store Khi:

```
❌ Chỉ có 1-2 models, 1 team
❌ Batch-only inference (không cần real-time)
❌ Features đơn giản, tính trực tiếp khi cần
❌ Prototype / PoC (Proof of Concept) giai đoạn đầu
❌ Small team startup với ít resources
```

> **Thực Tế:** Feature Store là investment lớn về infrastructure và process. Với team nhỏ (<5 ML engineers) và ít models (<5), overhead thường cao hơn lợi ích. Bắt đầu với Feature Store ngay từ đầu cho enterprise team lớn; thêm vào sau khi bạn thực sự cảm nhận pain của việc không có nó.

---

## Câu Hỏi Phỏng Vấn

### Q1: Feature Store giải quyết vấn đề gì? Training/Serving Skew là gì?

**Trả lời tốt:**
> Feature Store giải quyết 3 vấn đề chính. Thứ nhất, **Training/Serving Skew** (Độ Lệch Huấn Luyện/Phục Vụ): feature được tính bằng logic khác nhau khi training (Spark) và khi serving (Python online), dẫn đến model hoạt động kém hơn production so với offline evaluation. Feature Store fix bằng cách định nghĩa feature logic một lần và dùng cùng store cho cả hai. Thứ hai, **Feature Duplication** (Nhân Bản Đặc Trưng): 5 teams tính "days_since_last_purchase" 5 cách khác nhau, tốn effort và không nhất quán. Thứ ba, **Point-in-time correctness** (Chính Xác Theo Thời Điểm): khi tạo training data, phải đảm bảo feature value tại thời điểm lịch sử, không phải giá trị hiện tại — Offline Store's time-travel query xử lý điều này tự động.

### Q2: Online Store và Offline Store khác nhau thế nào? Khi nào dùng loại nào?

**Trả lời tốt:**
> Online Store là key-value store độ trễ thấp (~1ms), lưu **giá trị hiện tại** của mỗi entity — dùng trong real-time inference pipeline khi cần lấy features của một customer_id trong mili-giây. Offline Store là columnar storage trên S3 (Parquet + Apache Iceberg), lưu **toàn bộ lịch sử** thay đổi theo thời gian — dùng để tạo training datasets với point-in-time queries qua Athena. Trong production, tôi luôn bật cả hai: Streaming ingestion cập nhật Online Store gần real-time (VD: sau mỗi purchase event), đồng thời đẩy vào Offline Store để có history. Training pipeline query Offline Store với time-travel để avoid data leakage, còn inference pipeline query Online Store cho speed.

### Q3: Khi nào KHÔNG nên đầu tư vào Feature Store?

**Trả lời tốt:**
> Feature Store là investment lớn cả về kỹ thuật (setup, data pipelines, governance) lẫn organizational process (ai sở hữu feature definitions, ai review PRs cho features). Tôi không recommend Feature Store khi: team chỉ có 1-2 ML engineers, ít hơn 5 models trong production, hoặc đang ở giai đoạn prototype. Trong các trường hợp này, overhead của Feature Store cao hơn lợi ích — tốt hơn là dùng simple S3-based feature store hoặc thậm chí chỉ cần CSV files version-controlled. Tôi sẽ introduce Feature Store khi team bắt đầu cảm nhận "pain points": nhiều teams hỏi nhau tính feature thế nào, hoặc phát hiện Training/Serving Skew trong production lần đầu tiên. Đó là thời điểm tốt nhất để justify investment.

---

## 📊 Tóm Tắt

```
SageMaker Feature Store
│
├── Online Store (Kho Trực Tuyến)
│   ├── ~1ms latency
│   ├── Latest value only (chỉ giá trị mới nhất)
│   └── Dùng cho: Real-time inference
│
├── Offline Store (Kho Ngoại Tuyến)
│   ├── S3-based, Apache Parquet + Iceberg
│   ├── Full history với timestamps
│   ├── Point-in-time queries via Athena
│   └── Dùng cho: Training dataset creation
│
├── Feature Group (Nhóm Đặc Trưng)
│   ├── Schema: feature names + types (STRING, INTEGRAL, FRACTIONAL)
│   ├── Record Identifier: primary key (customer_id, product_id)
│   └── Event Time: timestamp bắt buộc cho point-in-time
│
├── Ingestion (Nạp Dữ Liệu)
│   ├── Batch: DataFrame.ingest() hoặc Processing Job
│   └── Streaming: put_record() sau mỗi event
│
└── Retrieval (Truy Xuất)
    ├── Online: get_record() / batch_get_record()
    └── Offline: Athena query hoặc create_dataset() SDK
```

**Giải Quyết:** Training/Serving Skew, Feature Duplication, Point-in-time Correctness

---

**Hoàn Thành Module 02-sagemaker!** Xem tiếp: [../03-bedrock/](../03-bedrock/) — Amazon Bedrock & Generative AI

**Cập Nhật Lần Cuối:** 2026-06-03
