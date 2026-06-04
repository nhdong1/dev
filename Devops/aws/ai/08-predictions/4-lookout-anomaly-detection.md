# Amazon Lookout — Phát Hiện Bất Thường (Anomaly Detection)

> **Amazon Lookout** là bộ ba dịch vụ phát hiện bất thường (anomaly detection) được quản lý hoàn toàn (fully-managed) của AWS, **không yêu cầu ML expertise**: **Lookout for Metrics** (Phát Hiện Bất Thường Trong Số Liệu Kinh Doanh), **Lookout for Equipment** (Phát Hiện Bất Thường Thiết Bị Công Nghiệp), và **Lookout for Vision** (Phát Hiện Lỗi Hình Ảnh). Cả ba đều tự động học patterns bình thường và cảnh báo khi có điều gì đó bất thường xảy ra.

---

## 🎯 Tại Sao Cần Anomaly Detection Tự Động?

### Vấn Đề Với Approach Truyền Thống

```
Cách cũ — Threshold-based Alerting (Cảnh Báo Dựa Ngưỡng):
- "Cảnh báo khi error rate > 5%"
- Vấn đề: Ngưỡng cứng nhắc, không tự động điều chỉnh theo seasonality
  → 5% error rate vào ngày thường = BAD
  → 5% error rate trong flash sale = NORMAL (do load tăng)

Cách mới — ML-based Anomaly Detection:
- Tự học patterns bình thường (theo giờ, ngày, tuần, mùa)
- Phát hiện khi giá trị THỰC SỰ bất thường so với kỳ vọng
- Không cần đặt ngưỡng thủ công
- Giảm false positives (cảnh báo giả)
```

### Tổng Quan Ba Dịch Vụ Lookout

| Dịch Vụ | Bài Toán | Dữ Liệu Input | Use Case |
|---|---|---|---|
| **Lookout for Metrics** | KPI (Chỉ Số Hiệu Suất Chính) kinh doanh bất thường | Time series metrics từ nhiều nguồn | Revenue giảm, conversion rate bất thường |
| **Lookout for Equipment** | Máy móc có nguy cơ hỏng hóc | Sensor data từ thiết bị công nghiệp | Predictive maintenance (bảo trì dự đoán) |
| **Lookout for Vision** | Sản phẩm lỗi trên dây chuyền sản xuất | Ảnh từ camera nhà máy | Quality control (kiểm soát chất lượng) |

---

## 📊 Amazon Lookout for Metrics — Phát Hiện Bất Thường Số Liệu

### Lookout for Metrics Là Gì?

**Amazon Lookout for Metrics** tự động phát hiện **anomalies** (bất thường) trong **KPIs kinh doanh** (doanh thu, tỷ lệ chuyển đổi, số đơn hàng, lưu lượng truy cập...) mà không cần viết code ML hoặc đặt ngưỡng thủ công.

### Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Lookout for Metrics                              │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                       Anomaly Detector                        │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │                   Metric Set                            │  │  │
│  │  │  Metrics (giá trị đo): revenue, pageviews, error_rate  │  │  │
│  │  │  Dimensions (chiều): country, device, category         │  │  │
│  │  │  Frequency: 5min, 10min, 1hour, 1day                   │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  │                                                               │  │
│  │  Data Sources:                                               │  │
│  │  S3 │ CloudWatch │ RDS │ Redshift │ Salesforce │ AppFlow     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                  Anomaly Detection Engine                     │  │
│  │  - Học seasonality (giờ, ngày, tuần, năm)                    │  │
│  │  - Tự động điều chỉnh expected range                         │  │
│  │  - Phát hiện correlated anomalies (bất thường tương quan)    │  │
│  │  - Root Cause Analysis (Phân Tích Nguyên Nhân Gốc Rễ)       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      Alerts & Actions                         │  │
│  │  SNS │ Lambda │ Webhook (Slack, PagerDuty, Jira)              │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Metrics vs Dimensions (Số Liệu vs Chiều)

| Khái Niệm | Giải Thích | Ví Dụ |
|---|---|---|
| **Metric** (Số Liệu) | Giá trị số cần theo dõi | `revenue`, `pageviews`, `conversion_rate`, `error_count` |
| **Dimension** (Chiều Phân Tích) | Thuộc tính phân loại để drill-down | `country`, `device_type`, `product_category`, `channel` |

```python
# Ví dụ: theo dõi revenue bị phân tách theo country và device_type
# Lookout sẽ phát hiện anomaly ở mức:
# - Tổng revenue (all countries, all devices)
# - Revenue per country (VN, US, SG...)
# - Revenue per device_type (mobile, desktop)
# - Revenue per country + device_type (VN-mobile, VN-desktop...)
```

### Tạo Anomaly Detector (Bộ Phát Hiện Bất Thường)

```python
import boto3

lookout_metrics = boto3.client("lookoutmetrics", region_name="us-east-1")

# Bước 1: Tạo Anomaly Detector
response = lookout_metrics.create_anomaly_detector(
    AnomalyDetectorName="ecommerce-kpi-detector",
    AnomalyDetectorDescription="Phát hiện bất thường trong KPIs thương mại điện tử",
    AnomalyDetectorConfig={
        "AnomalyDetectorFrequency": "PT1H"  # ISO 8601: P5M=5phút, PT1H=1giờ, P1D=1ngày
    }
)
detector_arn = response["AnomalyDetectorArn"]

# Bước 2: Tạo Metric Set từ S3 data source
lookout_metrics.create_metric_set(
    AnomalyDetectorArn=detector_arn,
    MetricSetName="ecommerce-hourly-metrics",
    MetricSetDescription="Số liệu thương mại điện tử theo giờ",
    MetricList=[
        {
            "MetricName": "revenue",
            "AggregationFunction": "SUM",         # SUM, AVG, COUNT
            "Namespace": "EcommerceMetrics"
        },
        {
            "MetricName": "conversion_rate",
            "AggregationFunction": "AVG",
            "Namespace": "EcommerceMetrics"
        }
    ],
    DimensionList=["country", "device_type"],     # Chiều phân tích
    MetricSetFrequency="PT1H",                    # Tần suất thu thập
    MetricSource={
        "S3SourceConfig": {
            "RoleArn": "arn:aws:iam::123456789:role/LookoutMetricsRole",
            "TemplatedPathList": ["s3://my-bucket/metrics/{{yyyy}}/{{MM}}/{{dd}}/{{HH}}/*.csv"],
            "HistoricalDataPathList": ["s3://my-bucket/historical/"],
            "FileFormatDescriptor": {
                "CsvFormatDescriptor": {
                    "FileCompression": "NONE",
                    "Charset": "UTF-8",
                    "ContainsHeader": True,
                    "Delimiter": ","
                }
            }
        }
    },
    # Cột chứa timestamp trong dữ liệu
    TimestampColumn={
        "ColumnName": "event_time",
        "ColumnFormat": "yyyy-MM-dd HH:mm:ss"
    },
    Timezone="Asia/Ho_Chi_Minh"
)
```

### Root Cause Analysis (Phân Tích Nguyên Nhân Gốc Rễ)

Khi phát hiện anomaly, Lookout for Metrics tự động phân tích dimension nào gây ra bất thường:

```python
# Lấy chi tiết về một anomaly group
response = lookout_metrics.list_anomaly_group_summaries(
    AnomalyDetectorArn=detector_arn,
    SensitivityThreshold=70  # 0-100: cao hơn = ít anomaly hơn nhưng chắc chắn hơn
)

for anomaly in response["AnomalyGroupSummaryList"]:
    print(f"Anomaly Group: {anomaly['AnomalyGroupId']}")
    print(f"  Score: {anomaly['AnomalyGroupScore']}")
    print(f"  Start Time: {anomaly['StartTime']}")

# Lấy root cause cho một anomaly group
detail = lookout_metrics.get_anomaly_group(
    AnomalyGroupId=anomaly["AnomalyGroupId"],
    AnomalyDetectorArn=detector_arn
)

# Root cause analysis
for metric_level_impact in detail["AnomalyGroup"]["MetricLevelImpactList"]:
    print(f"Metric: {metric_level_impact['MetricName']}")
    print(f"  Num timeseries affected: {metric_level_impact['NumTimeSeries']}")
    # Dimension breakdown: VN-mobile bị giảm revenue nhiều nhất
```

### Alerts (Cảnh Báo)

```python
# Tạo Alert qua SNS khi phát hiện anomaly có score cao
lookout_metrics.create_alert(
    AlertName="high-severity-anomaly-alert",
    AnomalyDetectorArn=detector_arn,
    AlertSensitivityThreshold=80,  # Chỉ alert khi anomaly score > 80
    Action={
        "SNSConfiguration": {
            "RoleArn": "arn:aws:iam::123456789:role/LookoutSNSRole",
            "SnsTopicArn": "arn:aws:sns:us-east-1:123456789:anomaly-alerts"
        }
    }
)

# Hoặc gọi Lambda để auto-remediation (tự động khắc phục)
lookout_metrics.create_alert(
    AlertName="auto-remediation-alert",
    AnomalyDetectorArn=detector_arn,
    AlertSensitivityThreshold=90,
    Action={
        "LambdaConfiguration": {
            "RoleArn": "arn:aws:iam::123456789:role/LookoutLambdaRole",
            "LambdaArn": "arn:aws:lambda:us-east-1:123456789:function:handle-anomaly"
        }
    }
)
```

---

## 🔧 Amazon Lookout for Equipment — Bảo Trì Dự Đoán

### Lookout for Equipment Là Gì?

**Amazon Lookout for Equipment** — Bảo Trì Dự Đoán (Predictive Maintenance) phân tích dữ liệu từ **sensors** (cảm biến) của máy móc/thiết bị công nghiệp để phát hiện **dấu hiệu hỏng hóc sớm** — trước khi failure thực sự xảy ra.

### Use Cases Điển Hình

```
✅ Turbine gió (wind turbine): Phát hiện bearing failure (hỏng ổ lăn) sớm 2-4 tuần
✅ Máy nén khí (compressor): Cảnh báo trước khi áp suất giảm đột ngột
✅ Thiết bị lọc nước công nghiệp: Phát hiện membrane fouling (tắc màng lọc) sớm
✅ Động cơ điện trong nhà máy: Phát hiện vibration anomaly (rung bất thường)
✅ Thiết bị lạnh (HVAC): Phát hiện hiệu suất làm mát giảm
```

### Sensor Data Requirements (Yêu Cầu Dữ Liệu Cảm Biến)

| Yêu Cầu | Giá Trị |
|---|---|
| **Số lượng sensors tối thiểu** | 5 sensors |
| **Sensors tối đa** | 10,000 sensors |
| **Lịch sử training tối thiểu** | 180 ngày (6 tháng) |
| **Tần suất data** | 1 phút đến 1 giờ |
| **Tỷ lệ missing data** | < 10% |
| **Format** | CSV với timestamp + sensor readings |

```csv
timestamp,temperature_bearing_A,vibration_X,vibration_Y,pressure_inlet,current_motor,speed_rpm
2024-01-01 00:00:00,72.5,0.12,0.09,145.2,18.3,1485
2024-01-01 00:01:00,72.8,0.13,0.10,145.0,18.4,1484
2024-01-01 00:02:00,73.1,0.14,0.11,144.8,18.5,1483
...
# Bình thường: nhiệt độ ~72-75°C, rung ~0.1-0.15

# Dấu hiệu bất thường (2 tuần trước failure):
2024-05-15 08:00:00,89.2,0.45,0.38,139.1,22.8,1460
# Nhiệt độ tăng cao, rung tăng mạnh, áp suất giảm, dòng điện tăng
```

### Tạo Model Với Lookout for Equipment

```python
import boto3

lookout_equipment = boto3.client("lookoutequipment", region_name="us-east-1")
ROLE_ARN = "arn:aws:iam::123456789:role/LookoutEquipmentRole"
BUCKET = "my-equipment-bucket"

# Bước 1: Tạo Dataset
response = lookout_equipment.create_dataset(
    DatasetName="compressor-sensor-dataset",
    DatasetSchema={
        "InlineDataSchema": '{"Components": [{"ComponentName": "Compressor", "Columns": [{"Name": "timestamp", "Type": "DATETIME"}, {"Name": "temperature_bearing", "Type": "DOUBLE"}, {"Name": "vibration_x", "Type": "DOUBLE"}, {"Name": "pressure_inlet", "Type": "DOUBLE"}, {"Name": "current_motor", "Type": "DOUBLE"}]}]}'
    }
)
dataset_arn = response["DatasetArn"]

# Bước 2: Import sensor data lịch sử
lookout_equipment.start_data_ingestion_job(
    DatasetName="compressor-sensor-dataset",
    IngestionInputConfiguration={
        "S3InputConfiguration": {
            "Bucket": BUCKET,
            "Prefix": "sensor-data/historical/",
            "KeyPattern": "{prefix}/{component_name}/*"
        }
    },
    RoleArn=ROLE_ARN
)

# Bước 3: Train Model
lookout_equipment.create_model(
    ModelName="compressor-anomaly-model-v1",
    DatasetName="compressor-sensor-dataset",
    TrainingDataStartTime="2022-01-01T00:00:00Z",
    TrainingDataEndTime="2023-12-31T23:59:59Z",
    EvaluationDataStartTime="2024-01-01T00:00:00Z",
    EvaluationDataEndTime="2024-06-30T23:59:59Z",
    # Gán nhãn các khoảng thời gian bình thường để train
    LabelsInputConfiguration={
        "S3InputConfiguration": {
            "Bucket": BUCKET,
            "Prefix": "labels/"  # File CSV với: start_time, end_time, label (Normal/Anomaly)
        }
    },
    DataPreProcessingConfiguration={
        "TargetSamplingRate": "PT1M"  # Resample về 1 phút nếu data thô cao hơn
    },
    RoleArn=ROLE_ARN
)
```

### Inference (Dự Đoán Thời Gian Thực)

```python
# Bước 4: Tạo Inference Scheduler (Bộ Lập Lịch Dự Đoán)
lookout_equipment.create_inference_scheduler(
    ModelName="compressor-anomaly-model-v1",
    InferenceSchedulerName="compressor-realtime-scheduler",
    DataUploadFrequency="PT5M",           # Upload data mỗi 5 phút
    DataDelayOffsetInMinutes=2,           # Chờ 2 phút sau khi window kết thúc để đảm bảo data đủ
    DataInputConfiguration={
        "S3InputConfiguration": {
            "Bucket": BUCKET,
            "Prefix": "sensor-data/realtime/"
        },
        "InputTimeZoneOffset": "+07:00"   # Vietnam timezone
    },
    DataOutputConfiguration={
        "S3OutputConfiguration": {
            "Bucket": BUCKET,
            "Prefix": "inference-results/"
        }
    },
    RoleArn=ROLE_ARN
)

# Output inference result (JSON):
# {
#   "timestamp": "2024-07-15T08:00:00Z",
#   "prediction": "ANOMALY",             -- hoặc "NORMAL"
#   "anomaly_score": 0.87,               -- 0.5-1.0: anomaly; <0.5: normal
#   "diagnostics": [
#     {"name": "temperature_bearing", "value": 0.45},  -- Contribution score
#     {"name": "vibration_x", "value": 0.38}
#   ]
# }
```

### Tích Hợp Với SNS Để Cảnh Báo

```python
import boto3
import json

def lambda_handler(event, context):
    """Lambda xử lý kết quả inference và gửi alert khi cần."""
    sns = boto3.client("sns")

    # Đọc inference result từ S3 trigger
    s3_key = event["Records"][0]["s3"]["object"]["key"]
    s3 = boto3.client("s3")
    result = json.loads(s3.get_object(Bucket=BUCKET, Key=s3_key)["Body"].read())

    if result["prediction"] == "ANOMALY" and result["anomaly_score"] > 0.8:
        # Xác định sensor nào đóng góp nhiều nhất vào anomaly
        top_sensor = max(result["diagnostics"], key=lambda x: x["value"])

        sns.publish(
            TopicArn=os.environ["ALERT_TOPIC_ARN"],
            Subject="⚠️ CẢNH BÁO: Phát hiện bất thường thiết bị",
            Message=f"""
Thời gian: {result['timestamp']}
Anomaly Score: {result['anomaly_score']:.2%}
Sensor chính gây bất thường: {top_sensor['name']} (score: {top_sensor['value']:.2f})

Đề xuất hành động:
- Kiểm tra ngay thiết bị Compressor-Line-3
- Lên lịch bảo trì trong vòng 48 giờ
- Xem log chi tiết tại: s3://{BUCKET}/inference-results/
            """
        )
```

---

## 👁️ Amazon Lookout for Vision — Phát Hiện Lỗi Hình Ảnh

### Lookout for Vision Là Gì?

**Amazon Lookout for Vision** — Kiểm Tra Lỗi Bằng Thị Giác là dịch vụ Computer Vision (Thị Giác Máy Tính) chuyên biệt cho **quality control** (kiểm soát chất lượng) sản xuất — phát hiện **visual defects** (lỗi hình ảnh) trên dây chuyền sản xuất với tốc độ và độ chính xác cao hơn mắt người.

### Ưu Điểm So Với Rekognition Custom Labels

| Tiêu Chí | Lookout for Vision | Rekognition Custom Labels |
|---|---|---|
| **Chuyên biệt cho** | Manufacturing defect detection | General purpose image classification |
| **Dữ liệu tối thiểu** | 30 ảnh "normal" để bắt đầu | ~100+ ảnh |
| **Anomaly segmentation** | ✅ Pixel-level defect map | ❌ Không có |
| **Edge deployment** | ✅ AWS IoT Greengrass | ❌ Cloud only |
| **Inference speed** | Milliseconds (edge) | ~200-500ms (cloud) |
| **Giá** | Per image analyzed | Per image analyzed |

### Use Cases Trong Sản Xuất

```
✅ Chip bán dẫn: Phát hiện crack (vết nứt), scratch (xước), contamination (bẩn)
✅ Bảng mạch PCB: Phát hiện thiếu linh kiện, hàn thiếc lỗi, dây dẫn đứt
✅ Sản phẩm đúc/ép: Phát hiện lỗ hổng, biến dạng, bề mặt không đều
✅ Bao bì/nhãn dán: Phát hiện in sai, thiếu nhãn, nhăn bao bì
✅ Dệt vải/da giày: Phát hiện sợi lỗi, màu sắc không đồng đều
✅ Thực phẩm: Phát hiện sản phẩm biến dạng, thối rữa, kích thước sai
```

### Tạo Project và Training

```python
import boto3

lookout_vision = boto3.client("lookoutvision", region_name="us-east-1")

# Bước 1: Tạo Project
lookout_vision.create_project(ProjectName="pcb-defect-detection")

# Bước 2: Tạo Dataset và Import ảnh
# Cần ít nhất:
# - 30 ảnh "Normal" (sản phẩm bình thường)
# - Recommend: 10+ ảnh "Anomaly" (sản phẩm lỗi) với label
#
# Manifest file format (JSON Lines):
# {"source-ref": "s3://bucket/train/normal/img_001.jpg", "anomaly-label": 0, "metadata": {"type": "normal"}}
# {"source-ref": "s3://bucket/train/anomaly/img_defect.jpg", "anomaly-label": 1,
#  "anomaly-mask-ref": "s3://bucket/masks/img_defect_mask.jpg",  -- Pixel mask (tùy chọn)
#  "metadata": {"type": "scratch"}}

lookout_vision.create_dataset(
    ProjectName="pcb-defect-detection",
    DatasetType="train",
    DatasetSource={
        "GroundTruthManifest": {
            "S3Object": {
                "Bucket": "my-lookout-bucket",
                "Key": "manifests/train-manifest.json"
            }
        }
    }
)

# Tạo dataset test tương tự
lookout_vision.create_dataset(
    ProjectName="pcb-defect-detection",
    DatasetType="test",
    DatasetSource={"GroundTruthManifest": {...}}
)

# Bước 3: Train Model
lookout_vision.create_model(
    ProjectName="pcb-defect-detection",
    OutputConfig={
        "S3Location": {
            "Bucket": "my-lookout-bucket",
            "Prefix": "model-output/"
        }
    }
)
```

### Inference — Phát Hiện Lỗi Thời Gian Thực

```python
import boto3

lookout_vision = boto3.client("lookoutvision", region_name="us-east-1")

def detect_defects(image_path: str) -> dict:
    """
    Phát hiện lỗi trong một ảnh sản phẩm.
    Trả về: Normal/Anomaly + confidence + pixel anomaly map nếu có.
    """
    with open(image_path, "rb") as image_file:
        response = lookout_vision.detect_anomalies(
            ProjectName="pcb-defect-detection",
            ModelVersion="1",
            Body=image_file,
            ContentType="image/jpeg"
        )

    result = response["DetectAnomalyResult"]
    return {
        "is_anomalous": result["IsAnomalous"],     # True/False
        "confidence": result["Confidence"],          # 0.0 - 1.0
        "anomaly_mask": result.get("Anomalies", []) # List các vùng lỗi với pixel location
    }

# Ví dụ kết quả:
# {
#   "is_anomalous": True,
#   "confidence": 0.97,
#   "anomalies": [
#     {"Name": "scratch", "PixelAnomaly": {"TotalPercentageArea": 0.02, "Color": "#FF0000"}},
#     {"Name": "contamination", "PixelAnomaly": {"TotalPercentageArea": 0.005, "Color": "#0000FF"}}
#   ]
# }
```

### Edge Deployment Với AWS IoT Greengrass

```
Kiến trúc Edge Deployment (Triển Khai Biên):

Camera dây chuyền
       │
       ▼
AWS IoT Greengrass Device (PC/server trong nhà máy)
       │
       ├── Lookout for Vision Model (chạy local — không cần internet)
       │   → Inference trong <10ms, không phụ thuộc network
       │
       ├── Kết quả → Local PLC/SCADA (Hệ Thống Điều Khiển)
       │   → Reject (loại bỏ sản phẩm lỗi) ngay lập tức
       │
       └── Logs + Metrics → AWS Cloud (qua internet)
           → Dashboard, Alerting, Model improvement
```

```python
# Package model cho edge deployment
lookout_vision.start_model_packaging_job(
    ProjectName="pcb-defect-detection",
    ModelVersion="1",
    JobName="package-for-greengrass-v1",
    Configuration={
        "Greengrass": {
            "CompilerOptions": json.dumps({"mcpu": "cortex-a72"}),  # Target hardware
            "TargetDevice": "jetson_nano",      # NVIDIA Jetson Nano
            "S3OutputLocation": {
                "Bucket": "my-lookout-bucket",
                "Prefix": "edge-packages/"
            },
            "Tags": [{"Key": "env", "Value": "production-line-3"}]
        }
    }
)
```

---

## 🆚 So Sánh Ba Dịch Vụ Lookout

| Tiêu Chí | Lookout for Metrics | Lookout for Equipment | Lookout for Vision |
|---|---|---|---|
| **Domain** | Business KPIs | Industrial IoT (Internet of Things — Internet Vạn Vật) | Manufacturing visual |
| **Input data** | Time series metrics | Multi-variate sensor data | Images (ảnh) |
| **Training data tối thiểu** | ~2 tuần historical data | 180 ngày | 30 ảnh normal |
| **Đầu ra** | Anomaly groups + root cause | Anomaly score + sensor contributions | Normal/Anomaly + pixel map |
| **Latency inference** | Minutes (batch detection) | Minutes (scheduled) | Milliseconds |
| **Edge support** | ❌ | ❌ | ✅ AWS IoT Greengrass |
| **Data sources tích hợp** | S3, CloudWatch, RDS, Redshift, Salesforce | S3 (sensor CSV files) | S3, local (edge) |

---

## 🔗 Lookout for Metrics vs CloudWatch vs Datadog

| Tiêu Chí | Lookout for Metrics | CloudWatch Anomaly Detection | Datadog Anomaly |
|---|---|---|---|
| **ML approach** | Deep learning, ensemble | Statistical (ARIMA-based) | Statistical + ML |
| **Multi-metric correlation** | ✅ Phân tích đồng thời nhiều metrics | ❌ Từng metric đơn lẻ | ✅ |
| **Root cause analysis** | ✅ Dimension drill-down | ❌ | Hạn chế |
| **Không cần cấu hình** | ✅ Tự động học từ data | Cần cấu hình band | Tự động nhưng cần tuning |
| **Data sources** | Nhiều nguồn (S3, RDS, Salesforce...) | AWS services only | Nhiều nguồn (agent-based) |
| **Chi phí** | $ per detector-hour + analysis | $ per metric-month | $ per host/month |
| **Phù hợp** | Business KPIs, cross-system anomaly | AWS infrastructure metrics | Full-stack monitoring |

---

## 💰 Pricing Overview (Tổng Quan Tính Phí)

### Lookout for Metrics

```
- Detector: $ per detector per hour (tính cả khi detector đang INACTIVE)
- Anomaly Analysis: $ per anomaly group analyzed
→ Tip: Dừng (stop) detector khi không cần để tránh phí không cần thiết
```

### Lookout for Equipment

```
- Training: $ per training data unit (1 sensor × 1 giờ = 1 unit)
- Inference: $ per inference data unit (tương tự)
→ Model training chỉ trả tiền một lần — inference ongoing
```

### Lookout for Vision

```
- Training: $ per training hour
- Cloud inference: $ per image analyzed
- Edge inference: $ per model package + $ per activation-month (không tính per inference)
→ Edge inference thực tế rẻ hơn cloud khi có volume cao
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Lookout for Metrics vs CloudWatch Anomaly Detection — khi nào dùng cái nào?**

A: CloudWatch Anomaly Detection là built-in và free cho CloudWatch metrics — dùng cho infrastructure monitoring (CPU, memory, network latency...). Lookout for Metrics dùng khi cần: (1) phân tích nhiều metrics **đồng thời** để tìm correlated anomalies, (2) root cause analysis tự động theo dimensions (quốc gia, kênh, sản phẩm), (3) data đến từ nhiều nguồn khác nhau (S3, RDS, Salesforce không phải CloudWatch).

**Q: Lookout for Vision vs Rekognition Custom Labels — khi nào chọn cái nào?**

A: Lookout for Vision được tối ưu cho **manufacturing defect detection** với: edge deployment quan trọng (latency ms, offline), pixel-level anomaly segmentation (xác định vùng lỗi chính xác), và cần ít ảnh lỗi (semi-supervised — chủ yếu học từ ảnh "normal"). Rekognition Custom Labels phù hợp hơn cho: general image classification không phải manufacturing, không cần edge deployment, có nhiều labeled data cho cả two classes (normal + defect).

**Q: Làm thế nào xây dựng complete anomaly detection system cho nhà máy sử dụng cả ba Lookout services?**

A: Thiết kế ba layer: (1) **Lookout for Vision** trên production line — camera → Greengrass edge → detect defects ms-latency, reject sản phẩm lỗi ngay lập tức. (2) **Lookout for Equipment** cho sensor monitoring — multi-variate sensor data → detect early equipment failure → schedule preventive maintenance. (3) **Lookout for Metrics** cho business KPIs — production throughput, defect rate, downtime → correlate với business impact → alert operations manager.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
