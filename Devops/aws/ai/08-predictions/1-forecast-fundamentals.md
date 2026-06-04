# Amazon Forecast — Dự Báo Chuỗi Thời Gian (Nền Tảng)

> **Amazon Forecast** — Dự Báo Chuỗi Thời Gian là dịch vụ ML được quản lý hoàn toàn (fully-managed) của AWS, chuyên dùng để dự báo các giá trị theo thời gian (time series forecasting). Forecast sử dụng cùng công nghệ ML với hệ thống dự báo nội bộ của Amazon.com — không yêu cầu ML expertise, chỉ cần dữ liệu lịch sử đúng format.

---

## 🎯 Amazon Forecast Giải Quyết Vấn Đề Gì?

### Bài Toán Chuỗi Thời Gian

**Time Series** (Chuỗi Thời Gian) là dữ liệu được ghi lại theo thứ tự thời gian đều đặn. Ví dụ:
- Doanh số bán hàng mỗi ngày: `[100, 95, 120, 80, 200, 300, 250, ...]`
- Lưu lượng điện mỗi giờ: `[45kW, 52kW, 48kW, 60kW, ...]`
- Số lượng click mỗi 5 phút: `[1500, 1800, 2200, ...]`

**Bài toán:** Dự báo các giá trị **tương lai** (`t+1`, `t+2`, ..., `t+H`) dựa trên **lịch sử** (`t-N`, ..., `t-1`, `t`).

### Tại Sao Cần Amazon Forecast Thay Vì Tự Build?

| Vấn Đề Tự Build | Giải Pháp Của Forecast |
|---|---|
| Cần chọn thuật toán phù hợp (ARIMA, Prophet, LSTM?) | **AutoML** tự thử và chọn thuật toán tốt nhất |
| Cần xử lý seasonality (tính mùa vụ), holiday effects | Tích hợp sẵn calendar features và holiday datasets |
| Cần scale infra để train nhiều item_ids | Managed — tự động scale cho hàng triệu time series |
| Cold start: item mới không có lịch sử | Xử lý qua Item Metadata và Related Time Series |
| Cần probabilistic forecast (dự báo xác suất) | Quantile Forecast: P10, P50, P90 tích hợp sẵn |

---

## 📐 Kiến Trúc Tổng Quan

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Amazon Forecast                               │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     Dataset Group                            │    │
│  │  ┌─────────────────┐  ┌───────────────┐  ┌──────────────┐  │    │
│  │  │ Target Time      │  │ Related Time  │  │    Item      │  │    │
│  │  │ Series (bắt buộc)│  │ Series (opt.) │  │  Metadata   │  │    │
│  │  │                 │  │               │  │   (opt.)     │  │    │
│  │  │ item_id         │  │ item_id       │  │  item_id     │  │    │
│  │  │ timestamp       │  │ timestamp     │  │  category    │  │    │
│  │  │ demand (target) │  │ price, promo  │  │  brand, size │  │    │
│  │  └─────────────────┘  └───────────────┘  └──────────────┘  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Predictor                               │    │
│  │   AutoML: DeepAR+ │ CNN-QR │ NPTS │ ETS │ ARIMA │ Prophet   │    │
│  │   Forecast Horizon: T+1 → T+H (Chân Trời Dự Báo)           │    │
│  │   Forecast Frequency: hourly, daily, weekly, monthly        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Forecast                                │    │
│  │   P10 │ P50 │ P90 (Quantile Forecasts)                      │    │
│  │   Export to S3  OR  Query via API                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Dataset — Tập Dữ Liệu

### 1. Target Time Series (Chuỗi Thời Gian Mục Tiêu) — Bắt Buộc

Dữ liệu **lịch sử** của giá trị cần dự báo.

**Schema tối thiểu:**

| Cột | Kiểu | Mô Tả |
|---|---|---|
| `item_id` | string | Định danh duy nhất của time series (mã sản phẩm, ID cửa hàng...) |
| `timestamp` | timestamp | Thời điểm ghi nhận (ISO 8601: `2024-01-15 00:00:00`) |
| `target_value` | float | Giá trị cần dự báo (số lượng bán, điện tiêu thụ, v.v.) |

**Ví dụ dữ liệu:**

```csv
item_id,timestamp,target_value
product_A,2024-01-01 00:00:00,120.0
product_A,2024-01-02 00:00:00,95.0
product_A,2024-01-03 00:00:00,145.0
product_B,2024-01-01 00:00:00,55.0
product_B,2024-01-02 00:00:00,70.0
```

**Yêu cầu tối thiểu:** Ít nhất **300 data points** (điểm dữ liệu) per time series. Forecast Horizon (chân trời dự báo) không được vượt quá **1/3** tổng lịch sử.

### 2. Related Time Series (Chuỗi Thời Gian Liên Quan) — Tùy Chọn

Dữ liệu **biết trước** ảnh hưởng đến target (có thể biết trong tương lai).

**Ví dụ:** Giá bán, chương trình khuyến mãi, nhiệt độ thời tiết, ngày lễ tùy chỉnh.

```csv
item_id,timestamp,price,promotion_flag
product_A,2024-01-01 00:00:00,29.99,0
product_A,2024-01-02 00:00:00,19.99,1
product_A,2024-03-01 00:00:00,24.99,0
```

> **Quan trọng:** Related Time Series phải có dữ liệu **xuyên suốt cả quá khứ lẫn tương lai** (forecast horizon) — không thể dùng cho dữ liệu chỉ biết sau khi sự kiện xảy ra.

### 3. Item Metadata (Siêu Dữ Liệu Mặt Hàng) — Tùy Chọn

Thông tin tĩnh (static) về mỗi item — giúp Forecast xử lý **Cold Start Problem** (Vấn Đề Khởi Động Lạnh) cho item mới.

```csv
item_id,category,brand,color,size
product_A,electronics,BrandX,black,medium
product_B,clothing,BrandY,blue,large
```

### 4. Dataset Domain (Miền Tập Dữ Liệu)

Forecast cung cấp sẵn các domain template với schema chuẩn:

| Domain | Use Case |
|---|---|
| `RETAIL` | Dự báo nhu cầu bán lẻ (demand forecasting) |
| `INVENTORY_PLANNING` | Lập kế hoạch tồn kho |
| `EC2_CAPACITY` | Dự báo capacity AWS |
| `WORK_FORCE` | Lập kế hoạch nhân sự |
| `WEB_TRAFFIC` | Dự báo lưu lượng web |
| `METRICS` | Số liệu tổng quát |
| `CUSTOM` | Schema tùy chỉnh hoàn toàn |

---

## 🧠 Thuật Toán (Algorithms)

### DeepAR+ — Thuật Toán Chủ Lực

**DeepAR+** là thuật toán dựa trên **Recurrent Neural Network** (RNN — Mạng Thần Kinh Hồi Quy), phiên bản cải tiến của DeepAR do Amazon Research phát triển.

**Điểm mạnh:**
- Học **patterns chung** từ tất cả time series trong dataset — đặc biệt tốt khi có nhiều time series (hàng nghìn items)
- Tự động học seasonality (mùa vụ), trends (xu hướng), và cross-series correlations (tương quan chéo)
- Xử lý tốt **Cold Start** (item mới ít dữ liệu) nhờ transfer learning từ items tương tự
- Tự nhiên hỗ trợ probabilistic forecasting (P10/P50/P90)

**Khi nào chọn DeepAR+:** Nhiều time series (>100 items), cold start cần giải quyết, cần probabilistic forecast.

### CNN-QR (Convolutional Neural Network — Quantile Regression)

**CNN-QR** dùng **Convolutional Neural Network** (Mạng Thần Kinh Tích Chập) kết hợp Quantile Regression (Hồi Quy Phân Vị).

**Điểm mạnh:**
- Nắm bắt tốt các patterns ngắn hạn và local patterns
- Nhanh hơn DeepAR+ khi training
- Tốt với dữ liệu có nhiều spikes (đỉnh đột biến)

### Các Thuật Toán Cổ Điển

| Thuật Toán | Mô Tả | Khi Dùng |
|---|---|---|
| **NPTS** (Non-Parametric Time Series — Chuỗi Thời Gian Phi Tham Số) | Sampling từ dữ liệu lịch sử | Dữ liệu sparse (thưa thớt), không có pattern rõ ràng |
| **ETS** (Exponential Smoothing — Làm Mịn Hàm Mũ) | Classical statistical method | Time series ổn định, trend và seasonality đơn giản |
| **ARIMA** (AutoRegressive Integrated Moving Average — Trung Bình Động Tích Hợp Tự Hồi Quy) | Classical statistical method | Time series stationary (dừng), ngắn, đơn lẻ |
| **Prophet** | Facebook/Meta's additive model | Có holiday effects (ngày lễ) mạnh, trend thay đổi |

### AutoML — Tự Động Chọn Thuật Toán

Khi bật **AutoML**, Forecast tự động:
1. Huấn luyện **tất cả** thuật toán trên training data
2. Đánh giá theo **WAPE** (Weighted Absolute Percentage Error — Sai Số Phần Trăm Tuyệt Đối Có Trọng Số) và **RMSE** (Root Mean Square Error — Sai Số Căn Bậc Hai Trung Bình)
3. Chọn thuật toán có metric tốt nhất trên **back-test windows** (cửa sổ kiểm định ngược)

> **Trade-off:** AutoML tốn gấp ~5-7x thời gian và chi phí so với chọn một thuật toán duy nhất.

---

## ⚙️ Predictor — Bộ Dự Báo

**Predictor** là model đã được training, chứa cấu hình:

| Tham Số | Mô Tả |
|---|---|
| **Forecast Horizon** (Chân Trời Dự Báo) | Số bước dự báo vào tương lai. VD: `30` (30 ngày tới) |
| **Forecast Frequency** (Tần Suất Dự Báo) | Granularity: `H` (giờ), `D` (ngày), `W` (tuần), `M` (tháng) |
| **Forecast Dimensions** (Chiều Dự Báo) | Breakdown thêm theo location, store, channel (ngoài item_id) |
| **Algorithm** | `AUTO` (AutoML) hoặc chỉ định cụ thể |
| **Backtest Windows** (Cửa Sổ Kiểm Định Ngược) | Số lần cross-validation (thường 1-3) |
| **Featurization** (Trích Đặc Trưng) | Cách xử lý missing values (giá trị bị thiếu): `zero`, `nan`, `mean` |

### Predictor Metrics — Đánh Giá Độ Chính Xác

| Metric | Công Thức | Ý Nghĩa |
|---|---|---|
| **WAPE** | `Σ|actual - predicted| / Σ|actual|` | Sai số phần trăm có trọng số — thường dùng nhất cho retail |
| **RMSE** | `√(Σ(actual - predicted)² / n)` | Phạt nặng lỗi lớn — nhạy với outliers |
| **MASE** (Mean Absolute Scaled Error) | `MAE / MAE_naive` | So sánh với naive baseline (dự báo = giá trị kỳ trước) |
| **Avg wQuantileLoss** | Trung bình quantile loss các P-values | Đánh giá probabilistic forecast quality |

> **WAPE < 0.3** (30%) thường được xem là acceptable cho retail forecasting. WAPE < 0.1 là xuất sắc.

---

## 📊 Forecast — Kết Quả Dự Báo

### Quantile Forecast (Dự Báo Phân Vị)

Amazon Forecast trả về **probabilistic forecast** (dự báo xác suất) theo các quantiles:

| Quantile | Ký Hiệu | Ý Nghĩa Trong Bán Lẻ |
|---|---|---|
| **P10** (10th percentile) | Lạc quan nhất về nhu cầu | 90% khả năng nhu cầu thực tế **cao hơn** giá trị này |
| **P50** (median — trung vị) | Dự báo cơ sở | Nhu cầu có 50% khả năng cao hơn / thấp hơn |
| **P90** (90th percentile) | Thận trọng nhất | 90% khả năng nhu cầu thực tế **thấp hơn** giá trị này |

**Cách dùng trong thực tế:**
```
Hàng hóa thiết yếu (không thể thiếu):  dùng P90 để đảm bảo đủ hàng
Hàng hóa bình thường:                  dùng P50 cho kế hoạch trung bình
Hàng hóa tươi / tồn kho đắt:           dùng P10 để tránh tồn kho dư
```

### Query Forecast API

```python
import boto3
from datetime import datetime

forecast_query = boto3.client("forecastquery", region_name="us-east-1")

def query_forecast(forecast_arn: str, item_id: str, start_date: str, end_date: str):
    """
    Truy vấn kết quả dự báo cho một item cụ thể.
    start_date, end_date: ISO 8601 format, ví dụ "2024-07-01T00:00:00"
    """
    response = forecast_query.query_forecast(
        ForecastArn=forecast_arn,
        StartDate=start_date,
        EndDate=end_date,
        Filters={"item_id": item_id}
    )
    return response["Forecast"]["Predictions"]

# Kết quả trả về ví dụ:
# {
#   "p10": [{"Timestamp": "2024-07-01", "Value": 80.5}, ...],
#   "p50": [{"Timestamp": "2024-07-01", "Value": 120.0}, ...],
#   "p90": [{"Timestamp": "2024-07-01", "Value": 165.2}, ...]
# }
```

### Export Forecast to S3

Để lấy dự báo cho **toàn bộ items** cùng lúc, dùng Forecast Export Job:

```python
forecast = boto3.client("forecast", region_name="us-east-1")

response = forecast.create_forecast_export_job(
    ForecastExportJobName="weekly-forecast-export",
    ForecastArn="arn:aws:forecast:us-east-1:123456789:forecast/MyForecast",
    Destination={
        "S3Config": {
            "Path": "s3://my-bucket/forecast-results/",
            "RoleArn": "arn:aws:iam::123456789:role/ForecastS3Role"
        }
    }
)
```

Output S3: CSV files với format `item_id, date, p10, p50, p90`.

---

## 🛠️ Workflow Toàn Diện — Từ Dữ Liệu Đến Dự Báo

### Bước 1: Chuẩn Bị Dữ Liệu và Upload S3

```python
import pandas as pd
import boto3

# Tạo target time series từ dữ liệu bán hàng
sales_df = pd.DataFrame({
    "item_id": ["prod_A", "prod_A", "prod_A", "prod_B", "prod_B"],
    "timestamp": ["2024-01-01", "2024-01-02", "2024-01-03", "2024-01-01", "2024-01-02"],
    "target_value": [120.0, 95.0, 145.0, 55.0, 70.0]
})

# Upload lên S3
s3 = boto3.client("s3")
sales_df.to_csv("/tmp/target_time_series.csv", index=False)
s3.upload_file("/tmp/target_time_series.csv", "my-forecast-bucket", "data/target.csv")
```

### Bước 2: Tạo Dataset Group và Import Data

```python
forecast = boto3.client("forecast", region_name="us-east-1")
ROLE_ARN = "arn:aws:iam::123456789:role/ForecastRole"
BUCKET = "my-forecast-bucket"

# 2a. Tạo Dataset Group
forecast.create_dataset_group(
    DatasetGroupName="retail-demand-group",
    Domain="RETAIL"
)

# 2b. Tạo Dataset schema
forecast.create_dataset(
    DatasetName="retail-target-ts",
    DatasetType="TARGET_TIME_SERIES",
    Domain="RETAIL",
    DataFrequency="D",  # D = daily, H = hourly, W = weekly
    Schema={
        "Attributes": [
            {"AttributeName": "item_id",      "AttributeType": "string"},
            {"AttributeName": "timestamp",    "AttributeType": "timestamp"},
            {"AttributeName": "demand",       "AttributeType": "float"},
        ]
    }
)

# 2c. Import dữ liệu từ S3 vào Dataset
forecast.create_dataset_import_job(
    DatasetImportJobName="retail-data-import-v1",
    DatasetArn="arn:aws:forecast:us-east-1:123456789:dataset/retail-target-ts",
    DataSource={
        "S3Config": {
            "Path": f"s3://{BUCKET}/data/target.csv",
            "RoleArn": ROLE_ARN
        }
    },
    TimestampFormat="yyyy-MM-dd"
)
```

### Bước 3: Tạo Predictor (Huấn Luyện Model)

```python
# Tạo Predictor với AutoML
response = forecast.create_auto_predictor(
    PredictorName="retail-demand-predictor-v1",
    ForecastHorizon=30,           # Dự báo 30 ngày tới
    ForecastFrequency="D",        # Tần suất: hàng ngày
    ForecastDimensions=["location"],  # Breakdown theo địa điểm (ngoài item_id)
    DataConfig={
        "DatasetGroupArn": "arn:aws:forecast:us-east-1:123456789:dataset-group/retail-demand-group",
        "AttributeConfigs": [
            {
                "AttributeName": "demand",
                "Transformations": {
                    "aggregation": "sum",      # Tổng hợp khi có nhiều records cùng timestamp
                    "middlefill": "zero",      # Điền 0 cho khoảng trống giữa dữ liệu
                    "backfill": "zero",        # Điền 0 cho đầu dữ liệu
                    "futurefill": "zero"       # Dữ liệu tương lai
                }
            }
        ]
    },
    EncryptionConfig={
        "RoleArn": ROLE_ARN,
        "KMSKeyArn": "arn:aws:kms:us-east-1:123456789:key/my-key"  # Tùy chọn
    }
)
predictor_arn = response["PredictorArn"]
```

### Bước 4: Tạo Forecast và Export

```python
# Tạo Forecast từ Predictor đã train
forecast.create_forecast(
    ForecastName="retail-demand-forecast-july",
    PredictorArn=predictor_arn,
    ForecastTypes=["0.10", "0.50", "0.90"]  # P10, P50, P90
)

# Export toàn bộ kết quả ra S3
forecast.create_forecast_export_job(
    ForecastExportJobName="july-forecast-export",
    ForecastArn="arn:aws:forecast:us-east-1:123456789:forecast/retail-demand-forecast-july",
    Destination={
        "S3Config": {
            "Path": f"s3://{BUCKET}/forecasts/july/",
            "RoleArn": ROLE_ARN
        }
    }
)
```

---

## 💰 Pricing — Mô Hình Tính Phí

| Thành Phần | Đơn Vị Tính Phí | Ghi Chú |
|---|---|---|
| **Training** | $ per data point imported | Tính theo số lượng data points được import |
| **Generated Forecasts** | $ per 1,000 time series * horizon steps | Số lượng (time series × horizon steps) được forecast |
| **Storage** | $ per GB/month | Dữ liệu lưu trong Forecast |
| **Forecast Queries** | $ per 10,000 queries | Query qua API (`QueryForecast`) |

**Ví dụ chi phí ước tính:**
```
1,000 sản phẩm × 365 ngày lịch sử × 30 ngày horizon:
- Import: 1,000 × 365 = 365,000 data points
- Forecast: 1,000 × 30 = 30,000 forecast steps
→ Ước tính khoảng $10-30/lần chạy (tùy region và thuật toán)
```

---

## ⚠️ Lưu Ý Quan Trọng

1. **Dữ liệu phải đủ dài:** Ít nhất 3 lần Forecast Horizon. VD: Dự báo 30 ngày → cần tối thiểu 90 ngày lịch sử.
2. **Missing Values (Giá Trị Bị Thiếu):** Forecast tự động xử lý nhưng nên hiểu cơ chế `zero`/`nan`/`mean` fill để tránh bias (thiên kiến).
3. **Không phải real-time:** Tạo Predictor và Forecast mất **30 phút đến vài giờ** — không phù hợp cho use case cần kết quả ngay lập tức.
4. **Xóa resources sau khi dùng:** Forecast, Predictor, Dataset không tính phí lưu trữ lớn nhưng Export Jobs cần theo dõi S3 cost.
5. **Lookback Window:** DeepAR+ mặc định lookback = 1/3 * forecast_horizon * 3 — cần đủ dữ liệu lịch sử.

---

## 🔗 Liên Kết Module

- `2-forecast-advanced.md` — What-if analysis, Explainability, Cold start strategies
- `README.md` — Tổng quan Predictions module và so sánh dịch vụ

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
