# Amazon Forecast — Nâng Cao: What-if, Explainability, Cold Start

> Phần nâng cao của Amazon Forecast bao gồm: **What-if Analysis** (Phân Tích Nếu-Thì), **Forecast Explainability** (Khả Năng Giải Thích Dự Báo), xử lý **Cold Start Problem** (Vấn Đề Khởi Động Lạnh), **Monitor** (Giám Sát Độ Chính Xác Theo Thời Gian), và các chiến lược tối ưu hóa trong production.

---

## 🔮 What-if Analysis (Phân Tích Kịch Bản Giả Định)

### What-if Analysis Là Gì?

**What-if Analysis** cho phép so sánh dự báo dưới **các kịch bản khác nhau** — "Điều gì sẽ xảy ra nếu...?" mà không cần tạo Forecast mới từ đầu.

**Use cases điển hình:**

```
"Nếu chúng ta giảm giá 20% vào tuần tới, nhu cầu sẽ thay đổi thế nào?"
"Nếu chúng ta chạy khuyến mãi trong 7 ngày, cần nhập thêm bao nhiêu hàng?"
"Nếu nhiệt độ tăng 5°C, điện năng tiêu thụ tăng bao nhiêu %?"
```

### Cách Hoạt Động

```
Forecast gốc (baseline)           →  Dự báo dựa trên Related Time Series ban đầu
     │
     ├── What-if Forecast 1       →  Thay đổi: giảm giá 20%
     │   (Kịch bản 1)
     │
     ├── What-if Forecast 2       →  Thay đổi: thêm ngày khuyến mãi
     │   (Kịch bản 2)
     │
     └── So sánh delta            →  Lượng chênh lệch giữa các kịch bản
```

### Triển Khai What-if Analysis

```python
import boto3

forecast = boto3.client("forecast", region_name="us-east-1")

# Bước 1: Tạo What-if Analysis
response = forecast.create_what_if_analysis(
    WhatIfAnalysisName="price-reduction-20pct-analysis",
    ForecastArn="arn:aws:forecast:us-east-1:123456789:forecast/retail-demand-forecast"
)
what_if_analysis_arn = response["WhatIfAnalysisArn"]

# Bước 2: Định nghĩa kịch bản thay đổi (What-if Forecast)
# Kịch bản 1: Giảm giá 20% cho product_A trong tuần tới
forecast.create_what_if_forecast(
    WhatIfForecastName="scenario-price-drop-20pct",
    WhatIfAnalysisArn=what_if_analysis_arn,
    TimeSeriesTransformations=[
        {
            "Action": {
                "AttributeName": "price",
                "Operation": "MULTIPLY",   # MULTIPLY, ADD, PERCENTAGECHANGE
                "Value": 0.8               # 0.8 = giảm 20%
            },
            "TimeSeriesConditions": [
                {
                    "AttributeName": "item_id",
                    "AttributeValue": "product_A",
                    "Condition": "EQUALS"
                }
            ]
        }
    ]
)

# Bước 3: Export kết quả so sánh
forecast.create_what_if_forecast_export(
    WhatIfForecastExportName="price-scenario-export",
    WhatIfForecasts=["arn:aws:forecast:...:what-if-forecast/scenario-price-drop-20pct"],
    Destination={
        "S3Config": {
            "Path": "s3://my-bucket/what-if-results/",
            "RoleArn": "arn:aws:iam::123456789:role/ForecastRole"
        }
    }
)
```

### Các Phép Biến Đổi (Transformations)

| Operation | Ý Nghĩa | Ví Dụ |
|---|---|---|
| `MULTIPLY` | Nhân với hệ số | `0.8` = giảm 20%; `1.3` = tăng 30% |
| `ADD` | Cộng thêm giá trị tuyệt đối | `+5.0` = thêm 5 đơn vị |
| `PERCENTAGECHANGE` | Thay đổi theo % | `-15` = giảm 15% |

### Điều Kiện Lọc (TimeSeriesConditions)

```python
# Chỉ áp dụng cho sản phẩm thuộc danh mục "electronics" ở kho "warehouse_01"
"TimeSeriesConditions": [
    {
        "AttributeName": "category",
        "AttributeValue": "electronics",
        "Condition": "EQUALS"
    },
    {
        "AttributeName": "location",
        "AttributeValue": "warehouse_01",
        "Condition": "EQUALS"
    }
]
```

---

## 💡 Forecast Explainability (Khả Năng Giải Thích Dự Báo)

### Explainability Là Gì?

**Forecast Explainability** cho biết **thuộc tính nào** (feature) ảnh hưởng nhiều nhất đến kết quả dự báo — dùng kỹ thuật tương tự **SHAP** (SHapley Additive exPlanations — Giải Thích Bằng Giá Trị Shapley).

**Lý do quan trọng:**
- Giải thích cho business stakeholders tại sao dự báo tăng/giảm
- Phát hiện data quality issues (vấn đề chất lượng dữ liệu)
- Validate model behavior (kiểm tra xem model học đúng patterns không)
- Debug khi forecast không đúng kỳ vọng

### Impact Scores — Điểm Tác Động

Mỗi feature nhận một **Impact Score** (Điểm Tác Động):
- **Positive score (dương):** Feature này làm tăng giá trị dự báo
- **Negative score (âm):** Feature này làm giảm giá trị dự báo
- **Magnitude (độ lớn):** Tác động lớn hay nhỏ

```
Ví dụ kết quả Explainability cho "product_A, ngày 2024-07-15":
- promotion_flag:  +0.85  (khuyến mãi làm tăng nhu cầu mạnh)
- day_of_week:     +0.42  (thứ Hai có nhu cầu cao)
- price:           -0.31  (giá cao làm giảm nhu cầu)
- temperature:     +0.12  (thời tiết nóng ảnh hưởng nhẹ)
- holiday:         -0.08  (ngày lễ giảm nhu cầu B2B)
```

### Tạo Explainability

```python
forecast = boto3.client("forecast", region_name="us-east-1")

# Tạo Explainability cho một Forecast hoặc Predictor
response = forecast.create_explainability(
    ExplainabilityName="retail-forecast-explainability-v1",
    ResourceArn="arn:aws:forecast:us-east-1:123456789:forecast/retail-demand-forecast",
    ExplainabilityConfig={
        "TimeSeriesGranularity": "SPECIFIC",  # "ALL" hoặc "SPECIFIC"
        "TimePointGranularity": "SPECIFIC"    # "ALL" hoặc "SPECIFIC"
    },
    # Chỉ giải thích cho một số items/time points cụ thể (tiết kiệm chi phí)
    StartDateTime="2024-07-01T00:00:00",
    EndDateTime="2024-07-31T00:00:00"
)

# Export Explainability results ra S3
forecast.create_explainability_export(
    ExplainabilityExportName="july-explainability-export",
    ExplainabilityArn=response["ExplainabilityArn"],
    Destination={
        "S3Config": {
            "Path": "s3://my-bucket/explainability/",
            "RoleArn": "arn:aws:iam::123456789:role/ForecastRole"
        }
    }
)
```

### Đọc Kết Quả Explainability

```python
import pandas as pd

# Đọc file CSV từ S3 export
explainability_df = pd.read_csv("s3://my-bucket/explainability/part-0.csv")

# Columns: item_id, timestamp, p10_direction, p10_magnitude, p50_direction, p50_magnitude, ...
# Nhóm theo feature và tính tổng impact
feature_importance = (
    explainability_df
    .groupby("attribute_name")["p50_magnitude"]
    .mean()
    .sort_values(ascending=False)
)
print(feature_importance)
```

---

## 🥶 Cold Start Problem (Vấn Đề Khởi Động Lạnh)

### Cold Start Là Gì?

**Cold Start Problem** xảy ra khi một `item_id` mới xuất hiện **không có lịch sử dữ liệu** (hoặc quá ít dữ liệu) trong Target Time Series — Forecast không thể học patterns từ item đó.

**Ví dụ thực tế:**
- Ra mắt sản phẩm mới (new product launch)
- Mở cửa hàng mới ở địa điểm chưa có lịch sử
- Thêm route mới cho logistics (vận chuyển)

### Chiến Lược Xử Lý Cold Start

#### Chiến Lược 1: Item Metadata (Suy Luận Từ Items Tương Tự)

```python
# Thêm Item Metadata để Forecast học từ items "anh em"
item_metadata = """item_id,category,brand,price_tier,size
new_product_X,electronics,BrandA,mid,medium
product_A,electronics,BrandA,mid,large
product_B,electronics,BrandA,mid,small"""

# DeepAR+ sẽ bootstrap (khởi tạo) dự báo cho new_product_X
# dựa trên patterns của product_A và product_B (cùng category, brand, price_tier)
```

#### Chiến Lược 2: Related Time Series Proxy

```python
# Dùng dữ liệu liên quan biết trước để "dẫn đường" cho item mới
related_data = """item_id,timestamp,marketing_spend,channel_rank
new_product_X,2024-07-01,5000.0,2
new_product_X,2024-07-02,5000.0,2
new_product_X,2024-07-03,3000.0,3"""

# Forecast biết: marketing_spend = 5000 thường tương quan với nhu cầu cao
# → Bootstrap forecast dựa trên patterns này từ items khác
```

#### Chiến Lược 3: Similar Item Seeding

Khi tạo Predictor mới với item mới, dùng `backfill = "zero"` cho item mới nhưng giữ nguyên items cũ để DeepAR+ học cross-series patterns.

#### Chiến Lược 4: Hierarchical Forecasting (Dự Báo Phân Cấp)

```
Cấp độ Category (tổng hợp):  Electronics → 10,000 units/tháng
      │
      ├── Cấp độ Brand:       BrandA Electronics → 3,000 units/tháng
      │         │
      │         └── Cấp độ Product:  new_product_X → ? units/tháng
      │                              (Disaggregate từ cấp trên xuống)
```

---

## 📈 Forecast Monitor (Giám Sát Độ Chính Xác Theo Thời Gian)

### Tại Sao Cần Monitor Forecast?

Độ chính xác của Forecast có thể **giảm dần theo thời gian** vì:
- **Concept drift** (Trôi Dạt Khái Niệm): Behavior của người tiêu dùng thay đổi
- **Data distribution shift** (Dịch Chuyển Phân Phối): Sản phẩm mới, thị trường mới
- **External shocks** (Sốc Bên Ngoài): Dịch bệnh, khủng hoảng kinh tế, thay đổi chính sách

### Tạo Monitor

```python
forecast = boto3.client("forecast", region_name="us-east-1")

# Tạo Monitor cho Predictor
response = forecast.create_monitor(
    MonitorName="retail-predictor-monitor",
    ResourceArn="arn:aws:forecast:us-east-1:123456789:auto-predictor/retail-predictor-v1"
)

# Monitor tự động đánh giá Predictor khi có dữ liệu mới (ActualDemand)
# So sánh Predicted vs Actual và tính WAPE, RMSE theo thời gian
```

### Đọc Kết Quả Monitor

```python
# Lấy lịch sử evaluation của Monitor
response = forecast.list_monitor_evaluations(
    MonitorArn="arn:aws:forecast:us-east-1:123456789:monitor/retail-predictor-monitor"
)

for eval_result in response["PredictorMonitorEvaluations"]:
    print(f"Evaluation at: {eval_result['EvaluationTime']}")
    print(f"  WAPE: {eval_result['MetricResults'][0]['Value']:.4f}")
    print(f"  Status: {eval_result['Status']}")
```

### Khi Nào Nên Retrain?

| Tín Hiệu | Hành Động |
|---|---|
| WAPE tăng >20% so với baseline | Retrain Predictor ngay |
| Forecast consistently over/under-estimate | Xem xét thêm Related Time Series mới |
| Seasonal pattern thay đổi | Retrain với dữ liệu gần nhất làm trọng số cao hơn |
| New items chiếm >30% revenue | Retrain để cải thiện cold start |

---

## 🏗️ Auto Predictor vs Legacy Predictor

Amazon Forecast có hai loại Predictor:

| Tiêu Chí | AutoPredictor (Mới — Khuyến Nghị) | Legacy Predictor (Cũ) |
|---|---|---|
| **API** | `create_auto_predictor` | `create_predictor` |
| **Explainability** | ✅ Tích hợp sẵn | ❌ Không hỗ trợ |
| **What-if Analysis** | ✅ Hỗ trợ | ❌ Không hỗ trợ |
| **Monitor** | ✅ Hỗ trợ | ❌ Không hỗ trợ |
| **Accuracy** | Cao hơn (ensemble approach) | Thấp hơn |
| **Incremental Training** | ✅ `ReferencePredictorArn` | ❌ Phải train lại từ đầu |
| **Algorithm Selection** | Ensemble (kết hợp nhiều thuật toán) | Chọn một thuật toán |

> **Khuyến nghị:** Luôn dùng `create_auto_predictor` cho dự án mới.

### Incremental Training (Huấn Luyện Tăng Dần)

```python
# Tạo Predictor mới kế thừa từ Predictor cũ — tiết kiệm chi phí training
forecast.create_auto_predictor(
    PredictorName="retail-predictor-v2",
    ReferencePredictorArn="arn:aws:forecast:...:auto-predictor/retail-predictor-v1",
    # Chỉ train trên dữ liệu mới thêm vào, giữ nguyên knowledge từ v1
    DataConfig={
        "DatasetGroupArn": "..."
    }
)
```

---

## 🎯 Best Practices (Thực Hành Tốt Nhất)

### Chuẩn Bị Dữ Liệu

```
✅ Đảm bảo time series có ít nhất 3× Forecast Horizon lịch sử
✅ Xử lý outliers (giá trị ngoại lệ) cẩn thận — không xóa mà điền giá trị hợp lý
✅ Tạo Related Time Series cho promotions, holidays, pricing trước khi train
✅ Cung cấp Item Metadata ngay từ đầu dù chưa có many items
✅ Dùng timezone-aware timestamps nhất quán (ISO 8601 với UTC)
```

### Chọn Forecast Horizon và Frequency

```
Frequency = D (daily) → Horizon ≤ 500 days
Frequency = H (hourly) → Horizon ≤ 500 hours
Frequency = W (weekly) → Horizon ≤ 52 weeks
Frequency = M (monthly) → Horizon ≤ 24 months

Nguyên tắc: Forecast Horizon = chu kỳ lead time × 1.5
(Nếu lead time nhập hàng là 2 tuần → dự báo 3 tuần để có buffer)
```

### Tối Ưu Chi Phí

```
1. Dùng AutoML chỉ lần đầu → sau đó dùng thuật toán tốt nhất đã biết
2. Dùng Incremental Training thay vì train lại từ đầu mỗi lần
3. Chỉ export Quantiles cần thiết (không cần cả P10/P50/P90/P30/P70...)
4. Xóa Forecast resources sau khi export ra S3
5. Dùng Explainability với TimeSeriesGranularity="SPECIFIC" (không phải "ALL")
```

---

## 🔄 Tích Hợp Với Pipeline

### EventBridge + Lambda Tự Động Hóa

```python
# Lambda function kích hoạt mỗi tuần để tạo Forecast mới
import boto3
import os
from datetime import datetime

def lambda_handler(event, context):
    forecast = boto3.client("forecast")

    # Bước 1: Import dữ liệu mới nhất
    week_str = datetime.now().strftime("%Y%m%d")
    forecast.create_dataset_import_job(
        DatasetImportJobName=f"weekly-import-{week_str}",
        DatasetArn=os.environ["DATASET_ARN"],
        DataSource={
            "S3Config": {
                "Path": f"s3://{os.environ['DATA_BUCKET']}/weekly/sales-{week_str}.csv",
                "RoleArn": os.environ["ROLE_ARN"]
            }
        },
        TimestampFormat="yyyy-MM-dd"
    )

    # Bước 2: Tạo Forecast mới từ Predictor đã có
    # (Predictor được retrain hàng tháng bởi scheduled task khác)
    forecast.create_forecast(
        ForecastName=f"weekly-forecast-{week_str}",
        PredictorArn=os.environ["PREDICTOR_ARN"],
        ForecastTypes=["0.50", "0.90"]
    )

    return {"statusCode": 200, "body": f"Forecast job {week_str} started"}
```

### Step Functions — Orchestration Pipeline (Điều Phối Đường Ống)

```json
{
  "Comment": "Amazon Forecast Weekly Pipeline",
  "StartAt": "ImportData",
  "States": {
    "ImportData": {
      "Type": "Task",
      "Resource": "arn:aws:states:::forecast:createDatasetImportJob.sync",
      "Next": "CheckDataStatus"
    },
    "CheckDataStatus": {
      "Type": "Choice",
      "Choices": [
        {"Variable": "$.Status", "StringEquals": "ACTIVE", "Next": "CreateForecast"}
      ],
      "Default": "WaitForData"
    },
    "WaitForData": {
      "Type": "Wait",
      "Seconds": 300,
      "Next": "CheckDataStatus"
    },
    "CreateForecast": {
      "Type": "Task",
      "Resource": "arn:aws:states:::forecast:createForecast.sync",
      "Next": "ExportForecast"
    },
    "ExportForecast": {
      "Type": "Task",
      "Resource": "arn:aws:states:::forecast:createForecastExportJob.sync",
      "End": true
    }
  }
}
```

---

## 📋 Câu Hỏi Phỏng Vấn Nâng Cao

**Q: Giải thích sự khác biệt giữa P10, P50, P90 và khi nào dùng mỗi loại trong bài toán inventory?**

A: P10 = 10th percentile — nhu cầu thực tế có 90% khả năng cao hơn P10 → dùng khi tồn kho dư thừa rất tốn kém (hàng tươi, hàng thời vụ ngắn). P50 = median → dùng cho kế hoạch trung bình, sản phẩm cân bằng giữa risk of stockout (hết hàng) và overstock (tồn kho dư). P90 = nhu cầu thực tế có 90% khả năng thấp hơn → dùng cho hàng critical, không được phép hết hàng (thuốc, linh kiện máy quan trọng).

**Q: What-if Analysis có thể thay thế re-training không? Khi nào nên dùng cái nào?**

A: Không. What-if Analysis chỉ thay đổi giá trị của Related Time Series trong forecast horizon — model weights (trọng số model) không thay đổi. Dùng What-if khi cần **so sánh kịch bản kinh doanh** (pricing scenarios, promotion planning). Dùng re-training khi **behavior patterns thực sự thay đổi** (model accuracy giảm theo Monitor metrics, seasonal patterns thay đổi cơ bản).

**Q: Làm thế nào để xử lý cold start cho một chuỗi cửa hàng mở 50 địa điểm mới cùng lúc?**

A: Ba hướng tiếp cận kết hợp: (1) Item Metadata với location demographics (dân số, thu nhập bình quân, diện tích...) để DeepAR+ học từ các cửa hàng tương tự. (2) Related Time Series với thông tin marketing spend, khai trương events. (3) Hierarchical disaggregation — dự báo ở cấp regional rồi chia xuống store level theo tỷ lệ lịch sử của khu vực.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
