# ML Workflow — Quy Trình Học Máy Từ Dữ Liệu Đến Production

> Hiểu toàn bộ vòng đời của một ML project giúp bạn thiết kế hệ thống đúng và trả lời câu hỏi phỏng vấn system design tự tin.

---

## 1. Tổng Quan Quy Trình 6 Bước

```
┌──────────────────────────────────────────────────────────────────┐
│                    ML Workflow (Quy Trình ML)                    │
│                                                                   │
│  1. Problem         2. Data           3. Feature                 │
│     Definition  →     Collection  →     Engineering             │
│  (Định Nghĩa)      (Thu Thập DL)      (Kỹ Thuật Đặc Trưng)     │
│                                                                   │
│                    ↓                                              │
│                                                                   │
│  6. Monitoring      5. Deployment      4. Model                  │
│  (Giám Sát)    ←    (Triển Khai)  ←    Training &               │
│                                         Evaluation               │
│                                       (HT & Đánh Giá)           │
│                                                                   │
│          ↑___________________________|                            │
│          Feedback Loop (Vòng Phản Hồi — liên tục cải thiện)     │
└──────────────────────────────────────────────────────────────────┘
```

> ML không phải quy trình tuyến tính một chiều — đây là **vòng lặp liên tục** (iterative cycle). Thực tế, bạn sẽ quay lại bước trước nhiều lần.

---

## 2. Bước 1 — Problem Definition (Định Nghĩa Bài Toán)

### Tại Sao Đây Là Bước Quan Trọng Nhất?

> *"Giải đúng bài toán sai còn tệ hơn không giải."*

Trước khi viết một dòng code ML, phải trả lời đủ 4 câu hỏi:

### 4 Câu Hỏi Cần Trả Lời

**1. Bài toán business là gì?**
```
❌ Sai: "Dùng AI để cải thiện trải nghiệm khách hàng"
✅ Đúng: "Giảm tỉ lệ khách hàng rời bỏ (churn rate) từ 15% xuống dưới 10% trong 6 tháng"
```

**2. Đây có thực sự là bài toán ML không?**
- ML cần khi: pattern phức tạp, dữ liệu đủ nhiều, quy tắc khó viết tay
- Không cần ML khi: quy tắc đơn giản, không có dữ liệu, giải thích được bằng logic

**3. Bài toán ML cụ thể là gì?**

| Business Problem | ML Problem Type | Metric |
| ---------------- | --------------- | ------ |
| Phân loại email spam | Binary Classification | Precision, Recall, F1 |
| Dự đoán giá nhà | Regression | MAE, RMSE |
| Nhóm khách hàng tương đồng | Clustering | Silhouette Score |
| Gợi ý sản phẩm | Recommendation | NDCG, Hit Rate |

**4. Định nghĩa thành công (Success Criteria — Tiêu Chí Thành Công) là gì?**
- Accuracy (Độ Chính Xác) bao nhiêu là đủ?
- Latency (Độ Trễ) tối đa cho phép?
- Cost (Chi Phí) inference chấp nhận được?
- Explainability (Khả Năng Giải Thích) có cần không?

### AWS Mapping Ở Bước Này

Xác định dùng **AI Service có sẵn** hay **custom model trên SageMaker**:

```
Bài toán phổ biến → AI Service (Rekognition, Comprehend, Forecast...)
Bài toán tùy chỉnh → SageMaker (built-in algo hoặc custom container)
Generative AI → Bedrock (Foundation Models)
```

---

## 3. Bước 2 — Data Collection & Preparation (Thu Thập & Chuẩn Bị Dữ Liệu)

### Quy Tắc Vàng

> *"Garbage in, garbage out."* — Dữ liệu xấu → Model xấu, dù thuật toán tốt đến đâu.

### Ba Nguồn Dữ Liệu Chính

| Nguồn | Ví Dụ AWS | Đặc Điểm |
| ------ | --------- | --------- |
| **Internal Data** (Dữ Liệu Nội Bộ) | S3, RDS, DynamoDB, Redshift | Nhiều nhất, cần bảo mật cao |
| **External Data** (Dữ Liệu Bên Ngoài) | AWS Data Exchange, crawling | Bổ sung, cần kiểm tra chất lượng |
| **Synthetic Data** (Dữ Liệu Tổng Hợp) | SageMaker Ground Truth | Tạo thêm khi thiếu dữ liệu thực |

### Data Quality Issues (Vấn Đề Chất Lượng Dữ Liệu)

```
Các vấn đề cần xử lý:

Missing Values (Giá Trị Thiếu):
  → Imputation (Điền): mean, median, mode
  → Deletion (Xóa): nếu tỉ lệ thiếu cao
  → Flag: thêm cột "is_missing" làm feature

Outliers (Giá Trị Ngoại Lệ):
  → Kiểm tra: boxplot, IQR
  → Xử lý: cap, winsorize, hoặc giữ nguyên nếu có ý nghĩa

Class Imbalance (Mất Cân Bằng Lớp):
  → Fraud detection: 99% normal, 1% fraud
  → Giải pháp: oversampling (SMOTE), undersampling, class weights

Data Leakage (Rò Rỉ Dữ Liệu):
  → Feature chứa thông tin từ tương lai trong training
  → Nguyên nhân phổ biến: timestamp không xử lý đúng
```

### Data Splitting (Chia Dữ Liệu)

```
Toàn bộ Dataset
├── Training Set (Tập Huấn Luyện): 70-80%     ← model học từ đây
├── Validation Set (Tập Kiểm Định): 10-15%    ← tune hyperparameter
└── Test Set (Tập Kiểm Tra): 10-15%           ← đánh giá cuối cùng

⚠️ Quy tắc: Test Set phải được "khóa" — chỉ dùng một lần duy nhất
   để đánh giá model cuối cùng, KHÔNG dùng để tune.
```

### Trên AWS

- **Amazon S3:** Lưu trữ toàn bộ raw data và processed data
- **AWS Glue:** ETL (Extract, Transform, Load) và Data Catalog
- **Amazon SageMaker Data Wrangler:** GUI để clean và transform data
- **Amazon SageMaker Ground Truth:** Labeling data (gán nhãn dữ liệu) với human reviewers
- **AWS Lake Formation:** Xây dựng Data Lake (Hồ Dữ Liệu) có governance

---

## 4. Bước 3 — Feature Engineering (Kỹ Thuật Đặc Trưng)

### Định Nghĩa

Feature Engineering là quá trình **biến đổi dữ liệu thô** thành các **features (đặc trưng)** có ích cho model học — đây thường là yếu tố quyết định hiệu suất model hơn cả thuật toán.

### Các Kỹ Thuật Phổ Biến

#### Numerical Features (Đặc Trưng Số)

```python
# Normalization (Chuẩn Hóa) — đưa về [0, 1]
x_norm = (x - x_min) / (x_max - x_min)

# Standardization (Tiêu Chuẩn Hóa) — mean=0, std=1
x_std = (x - mean) / std

# Log Transform — xử lý skewed distribution (phân phối lệch)
x_log = log(x + 1)
```

#### Categorical Features (Đặc Trưng Phân Loại)

| Kỹ Thuật | Khi Dùng | Ví Dụ |
| --------- | --------- | ------ |
| **One-Hot Encoding** | Ít categories (< 20) | Màu sắc: đỏ, xanh, vàng |
| **Label Encoding** | Ordinal categories (có thứ tự) | Kích cỡ: S < M < L < XL |
| **Target Encoding** | Nhiều categories | Tỉnh/thành phố |
| **Embedding** | Rất nhiều categories + DL | UserID, ProductID |

#### Feature Creation (Tạo Đặc Trưng Mới)

```
Từ timestamp → Trích xuất: hour, day_of_week, month, is_weekend
Từ text → Bag of Words, TF-IDF, embeddings
Từ 2 features → Interaction feature: price × quantity = revenue
```

### Amazon SageMaker Feature Store

**Feature Store (Kho Đặc Trưng)** giải quyết vấn đề:
- **Online Store (Kho Trực Tuyến):** Truy vấn feature real-time với độ trễ thấp cho inference
- **Offline Store (Kho Ngoại Tuyến):** Lịch sử feature đầy đủ để training
- **Feature Reuse (Tái Sử Dụng Đặc Trưng):** Nhiều model dùng chung feature, đảm bảo nhất quán

---

## 5. Bước 4 — Model Training & Evaluation (Huấn Luyện & Đánh Giá Mô Hình)

### Training (Huấn Luyện)

```
Training Process:
  1. Khởi tạo model parameters (trọng số ngẫu nhiên)
  2. Forward pass: dự đoán output từ input
  3. Tính Loss (Hàm Mất Mát): sai lệch giữa dự đoán và ground truth
  4. Backward pass (Backpropagation — Lan Truyền Ngược): tính gradient
  5. Update weights: Gradient Descent (Hạ Gradient)
  6. Lặp lại (epoch) cho đến khi loss hội tụ
```

### Hyperparameter Tuning — Tinh Chỉnh Siêu Tham Số

**Hyperparameters (Siêu Tham Số)** là các tham số cấu hình model được đặt **trước khi training**, không học được từ dữ liệu.

| Hyperparameter | Ví Dụ | Ảnh Hưởng |
| -------------- | ------ | ---------- |
| **Learning Rate** (Tốc Độ Học) | 0.01, 0.001 | Tốc độ hội tụ, nguy cơ overshooting |
| **Batch Size** (Kích Thước Lô) | 32, 128, 512 | Tốc độ training, memory usage |
| **Epochs** (Số Vòng Lặp) | 10, 50, 100 | Thời gian training, nguy cơ overfitting |
| **Tree Depth** (Độ Sâu Cây) | 3, 6, 10 (XGBoost) | Model complexity |
| **Regularization** (Chính Quy Hóa) | L1/L2 lambda | Overfitting control |

**SageMaker Automatic Model Tuning (AMT):** Tự động tìm hyperparameter tốt nhất dùng Bayesian Optimization (Tối Ưu Bayes).

### Evaluation Metrics — Chỉ Số Đánh Giá

#### Classification Metrics (Chỉ Số Phân Loại)

**Confusion Matrix (Ma Trận Nhầm Lẫn):**

```
                  Predicted Positive    Predicted Negative
Actual Positive:   TP (True Positive)    FN (False Negative)
Actual Negative:   FP (False Positive)   TN (True Negative)
```

| Metric | Công Thức | Dùng Khi |
| ------ | --------- | --------- |
| **Accuracy** (Độ Chính Xác) | (TP+TN)/(TP+TN+FP+FN) | Balanced classes |
| **Precision** (Độ Chính Xác Dương) | TP/(TP+FP) | False Positive cost cao (spam filter) |
| **Recall / Sensitivity** (Độ Nhạy) | TP/(TP+FN) | False Negative cost cao (bệnh hiểm nghèo) |
| **F1 Score** | 2×P×R/(P+R) | Cần cân bằng Precision & Recall |
| **AUC-ROC** | Area Under ROC Curve | Ranking và binary classification |

**Ví dụ thực tế:**
- **Phát hiện ung thư:** Recall quan trọng hơn — miss cancer (FN) tệ hơn false alarm (FP)
- **Spam filter:** Precision quan trọng hơn — gửi email quan trọng vào spam (FP) tệ hơn miss spam (FN)

#### Regression Metrics (Chỉ Số Hồi Quy)

| Metric | Tên Đầy Đủ | Ý Nghĩa |
| ------ | ---------- | -------- |
| **MAE** | Mean Absolute Error — Sai Số Tuyệt Đối Trung Bình | Dễ hiểu, robust với outlier |
| **MSE** | Mean Squared Error — Sai Số Bình Phương Trung Bình | Phạt nặng sai số lớn |
| **RMSE** | Root Mean Squared Error | Cùng đơn vị với output |
| **R²** | R-squared / Coefficient of Determination | % variance giải thích được |

### Overfitting & Underfitting

```
Underfitting (Kém Khớp):
  Model quá đơn giản, không học được pattern
  → Biểu hiện: training loss cao, validation loss cao
  → Giải pháp: tăng model complexity, thêm features, giảm regularization

Overfitting (Quá Khớp):
  Model học cả noise, không generalize được
  → Biểu hiện: training loss thấp, validation loss cao (gap lớn)
  → Giải pháp: thêm dữ liệu, regularization, dropout, cross-validation

Good Fit (Khớp Tốt):
  → training loss ≈ validation loss, đều thấp
```

---

## 6. Bước 5 — Deployment (Triển Khai)

### Các Pattern Triển Khai Trên AWS

| Pattern | Mô Tả | SageMaker Endpoint Type |
| ------- | ------ | ----------------------- |
| **Real-time Inference** (Suy Luận Thời Gian Thực) | Dự đoán ngay lập tức, < 60 giây | Real-time endpoint |
| **Batch Transform** (Biến Đổi Hàng Loạt) | Xử lý dataset lớn offline | Batch Transform Job |
| **Async Inference** (Suy Luận Không Đồng Bộ) | Request lớn, chờ được | Async endpoint |
| **Serverless Inference** (Suy Luận Không Máy Chủ) | Traffic không đều, tiết kiệm | Serverless endpoint |

### Deployment Strategies (Chiến Lược Triển Khai)

```
Blue/Green Deployment (Triển Khai Xanh/Lục):
  Old (Blue) ── 100% traffic
  New (Green) ── 0% traffic
  → Shift traffic từ từ → rollback nếu lỗi

Canary Deployment (Triển Khai Canary):
  Old model ── 95% traffic
  New model ── 5% traffic
  → Monitor metrics → tăng dần nếu ổn

Shadow Deployment (Triển Khai Bóng):
  Old model ── 100% traffic (response thực)
  New model ── 100% traffic (chỉ log, không response)
  → So sánh output offline
```

### Model Serving Considerations (Cân Nhắc Khi Phục Vụ Model)

| Yêu Cầu | Giải Pháp |
| -------- | --------- |
| **Latency thấp** (< 100ms) | Real-time endpoint + provisioned concurrency |
| **High throughput** (Thông Lượng Cao) | Auto-scaling, multi-model endpoint |
| **Cost efficiency** (Hiệu Quả Chi Phí) | Serverless, spot instances |
| **Large payload** (Payload Lớn) | Async inference |

---

## 7. Bước 6 — Monitoring (Giám Sát)

### Tại Sao Phải Monitor Model Sau Khi Deploy?

Model không tồn tại mãi mãi — thế giới thay đổi, dữ liệu thay đổi → model cần được cập nhật.

### Các Loại Model Drift (Trôi Dạt Mô Hình)

```
Data Drift (Trôi Dạt Dữ Liệu):
  Phân phối input data thay đổi
  Ví dụ: model dự đoán giá nhà train trên data 2020, nhưng
          thị trường 2024 hoàn toàn khác (COVID, lãi suất...)

Concept Drift (Trôi Dạt Khái Niệm):
  Mối quan hệ giữa input và output thay đổi
  Ví dụ: spam patterns thay đổi — spammer học cách bypass filter cũ

Model Quality Drift (Trôi Dạt Chất Lượng):
  Accuracy giảm theo thời gian
  Dễ đo nhất nhưng chỉ phát hiện khi đã muộn
```

### SageMaker Model Monitor

| Loại Monitor | Phát Hiện Gì |
| ------------ | ------------ |
| **Data Quality Monitor** | Data drift — phân phối input thay đổi |
| **Model Quality Monitor** | Accuracy, F1, MAE giảm |
| **Bias Drift Monitor** | Thiên kiến (bias) tăng theo thời gian |
| **Feature Attribution Drift** | SHAP values thay đổi |

### Retraining Strategy (Chiến Lược Huấn Luyện Lại)

```
Scheduled Retraining (Huấn Luyện Lại Theo Lịch):
  → Trigger: mỗi tuần, mỗi tháng
  → Đơn giản, dễ quản lý

Event-triggered Retraining (Huấn Luyện Lại Theo Sự Kiện):
  → Trigger: khi drift vượt ngưỡng, khi có đủ data mới
  → Tối ưu hơn, phức tạp hơn

SageMaker Pipelines + CloudWatch → tự động trigger retraining
```

---

## 8. Full AWS ML Workflow — Mapping Thực Tế

```
Bước 1: Problem Definition
  → Chọn: AI Service hay SageMaker hay Bedrock?

Bước 2: Data Collection
  → AWS S3 (lưu trữ)
  → AWS Glue (ETL)
  → SageMaker Ground Truth (labeling)

Bước 3: Feature Engineering
  → SageMaker Data Wrangler (GUI)
  → SageMaker Processing Jobs (code)
  → SageMaker Feature Store (lưu & tái sử dụng)

Bước 4: Training & Evaluation
  → SageMaker Training Jobs (built-in algo hoặc custom)
  → SageMaker Experiments (tracking)
  → SageMaker Automatic Model Tuning (HPO)

Bước 5: Deployment
  → SageMaker Model Registry (đăng ký model)
  → SageMaker Endpoint (real-time / batch / async / serverless)
  → CodePipeline + SageMaker Pipelines (CI/CD)

Bước 6: Monitoring
  → SageMaker Model Monitor (drift detection)
  → Amazon CloudWatch (metrics, alerts)
  → SageMaker Clarify (bias & explainability)
  → Tự động trigger retraining nếu cần
```

---

## 9. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Giải thích overfitting và cách xử lý?**

> Overfitting xảy ra khi model học quá tốt training data, kể cả noise, nên không generalize được data mới. Biểu hiện: training accuracy cao, validation accuracy thấp. Giải pháp: (1) Thêm dữ liệu; (2) Regularization — L1/L2 penalty, Dropout; (3) Cross-validation; (4) Early stopping; (5) Giảm model complexity.

**Q: Feature Engineering quan trọng hơn hay chọn thuật toán quan trọng hơn?**

> Feature Engineering thường quan trọng hơn. Một model đơn giản với features tốt thường vượt trội hơn model phức tạp với features kém. Tuy nhiên với Deep Learning (đặc biệt Computer Vision, NLP), model tự học features nên khoảng cách thu hẹp lại.

**Q: Khi nào cần retrain model?**

> Khi phát hiện model drift (qua SageMaker Model Monitor), khi business context thay đổi đáng kể (ví dụ COVID làm thay đổi pattern tiêu dùng), khi có đủ dữ liệu mới chất lượng, hoặc theo lịch định kỳ (weekly/monthly) tùy business requirements.

---

## ➡️ Tiếp Theo

[4-aws-ai-layers.md](./4-aws-ai-layers.md) — Hiểu 3 tầng dịch vụ AI AWS và cách chọn đúng tầng cho từng bài toán.
