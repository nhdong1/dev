# Predictions (Dự Báo & Gợi Ý) trên AWS — Toàn Diện

> Module **08-predictions** bao gồm ba dịch vụ AI chuyên biệt của AWS: **Amazon Forecast** (Dự Báo Chuỗi Thời Gian), **Amazon Personalize** (Hệ Thống Gợi Ý Cá Nhân Hóa) và **Amazon Lookout** (Phát Hiện Bất Thường). Cả ba đều là AI Services được quản lý hoàn toàn (fully-managed), gọi qua API, không yêu cầu ML expertise (chuyên môn học máy) sâu.

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|---|---|---|
| `README.md` | Tổng quan Prediction Services, so sánh dịch vụ | ✅ |
| `1-forecast-fundamentals.md` | Dataset groups, Predictors, DeepAR+, AutoML Forecasting | ✅ |
| `2-forecast-advanced.md` | What-if analysis, Explainability, Cold start, Monitoring | ✅ |
| `3-personalize-fundamentals.md` | Interactions dataset, Recipe, Solution, Campaign | ✅ |
| `4-lookout-anomaly-detection.md` | Lookout for Metrics, Equipment, Vision — Phát hiện bất thường | ✅ |

---

## 🎯 Prediction Services Là Gì Và Dùng Để Làm Gì?

AWS cung cấp ba nhóm dịch vụ dự báo & phát hiện bất thường chuyên biệt:

- **Time Series Forecasting** (Dự Báo Chuỗi Thời Gian): Dự đoán giá trị trong tương lai dựa trên dữ liệu lịch sử theo thời gian (nhu cầu sản phẩm, doanh thu, lưu lượng truy cập...)
- **Recommendation System** (Hệ Thống Gợi Ý): Cá nhân hóa trải nghiệm người dùng — gợi ý sản phẩm, nội dung, phim, nhạc phù hợp với từng cá nhân
- **Anomaly Detection** (Phát Hiện Bất Thường): Tự động phát hiện giá trị bất thường trong dữ liệu kinh doanh và thiết bị công nghiệp

### Vị Trí Trong Hệ Sinh Thái AWS AI

```
┌─────────────────────────────────────────────────────────────────────┐
│  Tầng 3: AI Services (Dịch Vụ AI Được Quản Lý)  ◄── BẠN Ở ĐÂY    │
│  Amazon Forecast │ Amazon Personalize │ Amazon Lookout (3 variants) │
├─────────────────────────────────────────────────────────────────────┤
│  Tầng 2: ML Services                                                │
│   Amazon SageMaker (dùng khi cần custom forecasting / recommendation│
│   model với thuật toán hoàn toàn tùy chỉnh)                        │
├─────────────────────────────────────────────────────────────────────┤
│  Tầng 1: ML Framework & Infrastructure                              │
│  Prophet │ GluonTS │ PyTorch Forecasting │ GPU Instances            │
└─────────────────────────────────────────────────────────────────────┘
```

**Quy tắc chọn:** Dùng Forecast/Personalize/Lookout khi yêu cầu phù hợp với phạm vi dịch vụ có sẵn — nhanh hơn, không cần data science team chuyên sâu. Chỉ dùng SageMaker custom khi yêu cầu thuật toán rất đặc thù, cần kiểm soát hoàn toàn quá trình training, hoặc cần tích hợp dữ liệu phức tạp không fit với schema của Forecast/Personalize.

---

## 🗂️ So Sánh Dịch Vụ Prediction Trên AWS

| Dịch Vụ | Mục Đích | Input Chính | Output Chính | Tính Phí |
|---|---|---|---|---|
| **Amazon Forecast** | Dự báo chuỗi thời gian | Target time series + related data | Giá trị dự báo + khoảng tin cậy | Data points + forecast hours |
| **Amazon Personalize** | Gợi ý cá nhân hóa | User-item interactions | Danh sách items gợi ý có score | TPS provisioned + event ingestion |
| **Lookout for Metrics** | Phát hiện bất thường trong số liệu kinh doanh | Time series metrics (KPIs) | Anomaly groups + root cause | Detector hours + anomaly analysis |
| **Lookout for Equipment** | Phát hiện bất thường thiết bị công nghiệp | Sensor data từ máy móc | Anomaly scores + detected events | Inference + training data |
| **Lookout for Vision** | Phát hiện lỗi hình ảnh sản xuất | Ảnh sản phẩm từ camera dây chuyền | Normal / Anomaly classification | Images analyzed |

### Khi Nào Dùng Cái Nào?

```
Bạn cần giải quyết bài toán gì?
│
├─ Dự báo nhu cầu sản phẩm, tồn kho, doanh thu cho các tuần/tháng tới?
│   └─► Amazon Forecast — time series forecasting với DeepAR+, AutoML
│
├─ Gợi ý sản phẩm / nội dung phù hợp cho từng người dùng?
│   └─► Amazon Personalize — recommendation engine, real-time & batch
│
├─ Phát hiện số liệu KPI nào đó đang bất thường (traffic, revenue, error rate)?
│   └─► Lookout for Metrics — không cần ML, tự động detect anomaly
│
├─ Phát hiện máy móc/thiết bị công nghiệp có nguy cơ hỏng hóc?
│   └─► Lookout for Equipment — predictive maintenance (bảo trì dự đoán)
│
├─ Phát hiện lỗi sản phẩm trên dây chuyền sản xuất qua camera?
│   └─► Lookout for Vision — visual anomaly detection
│
└─ Cần thuật toán forecasting/recommendation hoàn toàn tùy chỉnh?
    └─► SageMaker với custom model (Prophet, GluonTS, Neural Collaborative Filtering)
```

---

## 📊 Amazon Forecast — Tổng Quan Nhanh

**Amazon Forecast** — Dự Báo Chuỗi Thời Gian là dịch vụ dự báo được quản lý hoàn toàn, sử dụng các thuật toán ML tiên tiến (DeepAR+, NPTS, CNN-QR, Prophet) để dự báo dữ liệu theo thời gian mà không yêu cầu kiến thức ML.

### Các Khái Niệm Cốt Lõi

| Khái Niệm | Giải Thích |
|---|---|
| **Dataset Group** (Nhóm Tập Dữ Liệu) | Container chứa tất cả datasets liên quan đến một bài toán forecasting |
| **Target Time Series** (Chuỗi Thời Gian Mục Tiêu) | Dữ liệu lịch sử của giá trị cần dự báo (item_id, timestamp, demand) |
| **Related Time Series** (Chuỗi Thời Gian Liên Quan) | Dữ liệu bổ sung ảnh hưởng đến dự báo (giá bán, ngày lễ, khuyến mãi) |
| **Item Metadata** (Siêu Dữ Liệu Mặt Hàng) | Thông tin tĩnh về từng item (danh mục, kích thước, màu sắc) |
| **Predictor** (Bộ Dự Báo) | Model được trained — AutoML tự chọn thuật toán tốt nhất |
| **Forecast** (Dự Báo) | Kết quả dự báo cho một khoảng thời gian tương lai |
| **Forecast Horizon** (Chân Trời Dự Báo) | Số bước thời gian dự báo vào tương lai |
| **Quantile Forecast** (Dự Báo Phân Vị) | Khoảng tin cậy: P10 (lạc quan), P50 (trung vị), P90 (thận trọng) |

### Use Cases Điển Hình

```
✅ Dự báo nhu cầu bán lẻ (Retail Demand Forecasting)
✅ Dự báo tồn kho (Inventory Planning)
✅ Dự báo doanh thu tài chính (Financial Forecasting)
✅ Dự báo lưu lượng điện/năng lượng (Energy Demand)
✅ Dự báo lưu lượng truy cập website/API
✅ Dự báo nhân sự cần thiết (Workforce Planning)
```

---

## 🎯 Amazon Personalize — Tổng Quan Nhanh

**Amazon Personalize** — Hệ Thống Gợi Ý Cá Nhân Hóa là dịch vụ recommendation engine được quản lý hoàn toàn. Cùng công nghệ với hệ thống gợi ý của Amazon.com — không cần ML expertise để triển khai.

### Các Khái Niệm Cốt Lõi

| Khái Niệm | Giải Thích |
|---|---|
| **Dataset Group** (Nhóm Tập Dữ Liệu) | Container tổng chứa tất cả datasets và resources của một use case |
| **Interactions Dataset** (Tập Dữ Liệu Tương Tác) | Lịch sử tương tác user-item (click, purchase, watch, rating) — **bắt buộc** |
| **Users Dataset** (Tập Dữ Liệu Người Dùng) | Metadata về người dùng (tuổi, giới tính, địa điểm) — tùy chọn |
| **Items Dataset** (Tập Dữ Liệu Mặt Hàng) | Metadata về items (thể loại, giá, mô tả) — tùy chọn |
| **Recipe** (Công Thức) | Thuật toán ML được định nghĩa sẵn cho từng use case cụ thể |
| **Solution** (Giải Pháp) | Model được trained với Recipe và Dataset |
| **Solution Version** (Phiên Bản Giải Pháp) | Một lần training cụ thể của Solution |
| **Campaign** (Chiến Dịch) | Endpoint triển khai Solution Version — nhận real-time recommendation requests |
| **Batch Inference Job** (Công Việc Suy Luận Hàng Loạt) | Tạo recommendations cho toàn bộ user base cùng lúc |

### Use Cases Điển Hình

```
✅ "Sản phẩm gợi ý cho bạn" — e-commerce (thương mại điện tử)
✅ "Bạn có thể thích xem" — video streaming (phát trực tuyến)
✅ "Bài nhạc tiếp theo" — music app
✅ "Bài viết liên quan" — news / blog platform
✅ "Khóa học phù hợp" — e-learning (học trực tuyến)
✅ "Re-rank kết quả tìm kiếm" — personalized search
```

---

## 🔍 Amazon Lookout — Tổng Quan Nhanh

**Amazon Lookout** là bộ ba dịch vụ phát hiện bất thường (anomaly detection) không cần ML expertise:

### Lookout for Metrics — Phát Hiện Bất Thường Trong Số Liệu

```
Mục đích: Tự động phát hiện anomaly trong các KPI (Key Performance Indicator — Chỉ Số Hiệu Suất Chính) kinh doanh
Input:  Time series data từ S3, RDS, Redshift, CloudWatch, Salesforce...
Output: Anomaly groups với root cause analysis (phân tích nguyên nhân gốc rễ)
Alerts: SNS, Lambda, Webhook khi phát hiện bất thường

Use cases:
- Doanh thu đột ngột giảm mạnh
- Tỷ lệ chuyển đổi (conversion rate) bất thường
- Error rate API tăng đột biến
- Chi phí vận hành vượt ngưỡng
```

### Lookout for Equipment — Bảo Trì Dự Đoán

```
Mục đích: Phát hiện máy móc/thiết bị công nghiệp có dấu hiệu hỏng hóc sớm
Input:  Sensor data (nhiệt độ, rung động, áp suất, tốc độ vòng quay...)
Output: Anomaly score + predicted failure events
Lợi ích: Giảm downtime (thời gian ngừng hoạt động) không lên kế hoạch

Use cases:
- Turbine phong điện
- Máy nén khí trong nhà máy
- Thiết bị y tế
- Máy móc dây chuyền sản xuất
```

### Lookout for Vision — Phát Hiện Lỗi Hình Ảnh

```
Mục đích: Phát hiện lỗi sản phẩm trên dây chuyền sản xuất qua camera
Input:  Ảnh sản phẩm (từ 30 ảnh để bắt đầu training)
Output: Normal / Anomaly + segmentation map (bản đồ vùng lỗi)
Tích hợp: AWS IoT Greengrass (edge deployment)

Use cases:
- Chip điện tử có vết nứt/lỗi bề mặt
- Sản phẩm thực phẩm bị hỏng hình dạng
- Bao bì in sai / thiếu nhãn
- Vải dệt có sợi lỗi
```

---

## 🔗 So Sánh Predictions vs SageMaker Custom

| Tiêu Chí | Forecast / Personalize / Lookout | SageMaker Custom |
|---|---|---|
| **ML expertise cần thiết** | Thấp — không cần biết thuật toán ML | Cao — cần data science skills |
| **Thời gian triển khai** | Ngày → tuần | Tuần → tháng |
| **Linh hoạt thuật toán** | Thấp — chỉ dùng recipes/algorithms định sẵn | Hoàn toàn tùy chỉnh |
| **Tích hợp dữ liệu phức tạp** | Trung bình — cần đúng schema | Cao — xử lý bất kỳ format nào |
| **Chi phí** | Trung bình — pay-per-use không cần quản lý infra | Phụ thuộc instance type và runtime |
| **Khả năng giải thích** | Có (Forecast Explainability, Personalize metrics) | Hoàn toàn kiểm soát via Clarify |
| **Phù hợp với** | SMB, startup, team thiếu ML engineer | Enterprise với data science team |

---

## 📈 Kiến Trúc Điển Hình

### Hệ Thống E-commerce Hoàn Chỉnh

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Luồng Dữ Liệu Người Dùng                         │
│                                                                     │
│  User Action → Kinesis Data Streams → Lambda → Personalize          │
│  (click, buy)   (Luồng Sự Kiện)              (PutEvents)            │
│                                                   │                 │
│                                              [Real-time             │
│                                               Campaign]             │
│                                                   │                 │
│                                         ◄─── Recommendations        │
│                                               (Top-N items)         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    Luồng Dữ Liệu Inventory                          │
│                                                                     │
│  Sales History → S3 → Amazon Forecast → API → Replenishment System │
│  (Lịch sử bán)        (DeepAR+, P50)         (Hệ thống bổ hàng)   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    Giám Sát KPI Doanh Nghiệp                        │
│                                                                     │
│  CloudWatch → Lookout for Metrics → SNS → PagerDuty / Slack         │
│  (Metrics)     (Anomaly Detection)    (Alert)                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

### Amazon Forecast

- **Q: Forecast AutoML là gì? Nó chọn thuật toán theo cơ chế nào?**
  - A: AutoML huấn luyện tất cả các thuật toán có sẵn (DeepAR+, CNN-QR, NPTS, ETS, ARIMA, Prophet) và chọn thuật toán có WAPE (Weighted Absolute Percentage Error — Sai Số Phần Trăm Tuyệt Đối Có Trọng Số) thấp nhất trên validation set (tập kiểm định).

- **Q: Khi nào dùng P10, P50, P90 trong Forecast?**
  - A: **P10** (bi quan nhất về nhu cầu, dùng để tính buffer tối thiểu), **P50** (trung bình — dự báo trung vị cho kế hoạch thông thường), **P90** (thận trọng — dùng khi chi phí hết hàng cao hơn chi phí tồn kho dư).

- **Q: Cold start problem (vấn đề khởi động lạnh) trong Forecast là gì?**
  - A: Khi một item_id mới không có lịch sử dữ liệu. Forecast xử lý bằng Item Metadata và Related Time Series để suy luận từ các items tương tự.

### Amazon Personalize

- **Q: Recipe là gì? Có những loại Recipe nào?**
  - A: Recipe là thuật toán được định nghĩa sẵn. Các loại chính: `USER_PERSONALIZATION` (gợi ý cho từng user), `PERSONALIZED_RANKING` (re-rank danh sách items), `RELATED_ITEMS` (items tương tự), `USER_SEGMENTATION` (phân khúc người dùng).

- **Q: Incremental training (huấn luyện tăng dần) trong Personalize hoạt động thế nào?**
  - A: Dùng `PutEvents` API để stream events mới theo thời gian thực — Campaign tự động cập nhật recommendations mà không cần retrain toàn bộ Solution. Full retrain định kỳ (hàng ngày/tuần) để học patterns dài hạn.

### Amazon Lookout

- **Q: Lookout for Metrics khác gì CloudWatch Anomaly Detection?**
  - A: CloudWatch Anomaly Detection theo dõi từng metric đơn lẻ. Lookout for Metrics phân tích **nhóm metrics** cùng lúc, phát hiện **correlated anomalies** (bất thường tương quan), và cung cấp **root cause analysis** — CloudWatch không làm được.

---

## 🚀 Bước Tiếp Theo

1. Học chi tiết **Amazon Forecast** trong `1-forecast-fundamentals.md` và `2-forecast-advanced.md`
2. Học chi tiết **Amazon Personalize** trong `3-personalize-fundamentals.md`
3. Học chi tiết **Amazon Lookout** trong `4-lookout-anomaly-detection.md`
4. Kết hợp với `09-mlops/` để hiểu cách monitor và retrain predictions models trong production

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
