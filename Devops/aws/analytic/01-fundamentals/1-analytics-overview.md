# 1 — Analytics Overview: Tổng Quan Phân Tích Dữ Liệu

> Analytics (Phân Tích Dữ Liệu) là quá trình khám phá, diễn giải và truyền đạt các mẫu có ý nghĩa từ dữ liệu. Hiểu rõ 4 loại analytics giúp bạn xác định đúng mục tiêu và chọn công cụ phù hợp.

## 📚 Mục Lục

1. [Analytics Là Gì?](#analytics-là-gì)
2. [4 Loại Analytics](#4-loại-analytics)
3. [So Sánh Nhanh](#so-sánh-nhanh)
4. [Ví Dụ Thực Tế Theo Ngành](#ví-dụ-thực-tế-theo-ngành)
5. [Analytics Trong Hệ Sinh Thái AWS](#analytics-trong-hệ-sinh-thái-aws)
6. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Analytics Là Gì?

**Analytics** (Phân Tích Dữ Liệu) là việc sử dụng dữ liệu, thống kê và công cụ tính toán để trả lời câu hỏi kinh doanh và hỗ trợ quyết định.

Ba thành phần cốt lõi:

```
Dữ Liệu (Data) + Phân Tích (Analysis) + Insight (Hiểu Biết) → Quyết Định Tốt Hơn
```

**Vì sao analytics quan trọng?**
- Thay thế quyết định dựa trên cảm tính bằng bằng chứng dữ liệu
- Phát hiện cơ hội và rủi ro ẩn trong dữ liệu
- Tự động hóa dự đoán và đề xuất
- Cạnh tranh trong thị trường ngày càng dữ liệu hóa

---

## 4 Loại Analytics

Có 4 cấp độ analytics, từ đơn giản đến phức tạp, từ mô tả quá khứ đến định hướng tương lai:

```
              ĐỘ KHÓ & GIÁ TRỊ KINH DOANH
              ▲
              │  ┌──────────────────────────────┐
              │  │  4. Prescriptive Analytics   │ ← Khó nhất, giá trị cao nhất
              │  │     (Phân Tích Đề Xuất)      │
              │  ├──────────────────────────────┤
              │  │  3. Predictive Analytics     │
              │  │     (Phân Tích Dự Đoán)      │
              │  ├──────────────────────────────┤
              │  │  2. Diagnostic Analytics     │
              │  │     (Phân Tích Chẩn Đoán)    │
              │  ├──────────────────────────────┤
              │  │  1. Descriptive Analytics    │ ← Dễ nhất, phổ biến nhất
              │  │     (Phân Tích Mô Tả)        │
              │  └──────────────────────────────┘
              └──────────────────────────────────────► THỜI GIAN
                  QUÁ KHỨ          HIỆN TẠI    TƯƠNG LAI
```

---

### 1. Descriptive Analytics — Phân Tích Mô Tả

**Câu hỏi trả lời:** "Điều gì đã xảy ra?"

**Định nghĩa:** Tóm tắt dữ liệu lịch sử để mô tả những gì đã xảy ra trong quá khứ. Đây là loại analytics phổ biến nhất và là nền tảng cho 3 loại còn lại.

**Kỹ thuật sử dụng:**
- Aggregation (Tổng Hợp): SUM, AVG, COUNT, MAX, MIN
- Data visualization (Trực Quan Hóa Dữ Liệu): biểu đồ, dashboard
- Reporting (Báo Cáo): báo cáo định kỳ, KPI tracking

**Ví dụ thực tế:**
```
- "Doanh thu tháng 3 là 2.5 tỷ đồng, tăng 15% so với tháng 2"
- "Top 10 sản phẩm bán chạy nhất tuần qua"
- "Tỷ lệ lỗi API hôm nay là 0.03%"
- "Số lượng đơn hàng theo giờ trong ngày"
```

**AWS Tools:**
- **Amazon QuickSight** — Dashboard và báo cáo tương tác
- **Amazon Athena** — SQL query trên S3 để tạo báo cáo
- **Amazon Redshift** — Aggregate queries cho OLAP workloads
- **Amazon CloudWatch** — Metrics và dashboards cho hệ thống AWS

**Tỷ lệ sử dụng trong thực tế:** ~80% các dự án analytics bắt đầu từ đây.

---

### 2. Diagnostic Analytics — Phân Tích Chẩn Đoán

**Câu hỏi trả lời:** "Tại sao điều đó xảy ra?"

**Định nghĩa:** Đào sâu vào dữ liệu để tìm nguyên nhân của một sự kiện hoặc xu hướng đã được xác định qua Descriptive Analytics. Chuyển từ "what" sang "why".

**Kỹ thuật sử dụng:**
- Drill-down (Đào Sâu): phân tích theo nhiều chiều
- Data mining (Khai Thác Dữ Liệu): tìm pattern ẩn
- Correlation analysis (Phân Tích Tương Quan): tìm mối liên hệ giữa các biến
- Root cause analysis (Phân Tích Nguyên Nhân Gốc Rễ)

**Ví dụ thực tế:**
```
- "Doanh số giảm 20% do campaign email gặp lỗi tracking → 60% khách không nhận được"
- "Tỷ lệ lỗi tăng đột biến do deploy phiên bản mới lúc 14:30"
- "Tỷ lệ bỏ giỏ hàng tăng vì checkout flow mới quá nhiều bước"
```

**AWS Tools:**
- **Amazon Athena** — Ad-hoc query để investigate sự cố
- **Amazon OpenSearch** — Log analytics, tìm pattern trong log
- **Amazon QuickSight** — Drill-down trên dashboard
- **AWS Glue** — Transform và join dữ liệu từ nhiều nguồn để phân tích

---

### 3. Predictive Analytics — Phân Tích Dự Đoán

**Câu hỏi trả lời:** "Điều gì có thể xảy ra trong tương lai?"

**Định nghĩa:** Sử dụng dữ liệu lịch sử, thống kê và machine learning (học máy) để dự đoán sự kiện tương lai với mức độ tin cậy nhất định.

**Kỹ thuật sử dụng:**
- Machine Learning models (Mô Hình Học Máy): classification, regression
- Time series forecasting (Dự Báo Chuỗi Thời Gian)
- Statistical modeling (Mô Hình Hóa Thống Kê)
- Anomaly detection (Phát Hiện Bất Thường)

**Ví dụ thực tế:**
```
- "Khách hàng X có 78% khả năng rời dịch vụ trong 30 ngày tới" (churn prediction)
- "Cần tồn kho 5,000 đơn vị sản phẩm A cho tháng 12" (demand forecasting)
- "Giao dịch này có 94% khả năng là gian lận" (fraud detection)
- "Server sẽ quá tải vào thứ 6 tuần sau nếu traffic tăng như xu hướng"
```

**AWS Tools:**
- **Amazon SageMaker** — Training và deploy ML models
- **Amazon Forecast** — Time-series forecasting được quản lý
- **Amazon Fraud Detector** — Phát hiện gian lận dựa trên ML
- **Amazon Kinesis** — Real-time data cho online predictions
- **AWS Glue + S3** — Feature store, chuẩn bị training data

---

### 4. Prescriptive Analytics — Phân Tích Đề Xuất

**Câu hỏi trả lời:** "Chúng ta nên làm gì?"

**Định nghĩa:** Không chỉ dự đoán những gì sẽ xảy ra mà còn đề xuất hành động tốt nhất để đạt kết quả mong muốn hoặc tránh kết quả không mong muốn. Đây là loại analytics phức tạp và giá trị nhất.

**Kỹ thuật sử dụng:**
- Optimization algorithms (Thuật Toán Tối Ưu Hóa): linear programming
- Simulation (Mô Phỏng): Monte Carlo simulation
- Reinforcement Learning (Học Tăng Cường): AI agents tự học
- Decision trees (Cây Quyết Định) với business constraints

**Ví dụ thực tế:**
```
- "Gửi email giảm giá 15% cho khách hàng X ngay bây giờ để giữ chân họ"
- "Điều chỉnh giá vé bay tuyến HN-HCM lên 12% vào thứ 5 tuần sau"
- "Route đơn hàng 12345 qua kho Bình Dương để giao nhanh hơn 2 giờ"
- "Tăng số instance lên 20 trước 8:00 sáng thứ 2 để tránh timeout"
```

**AWS Tools:**
- **Amazon SageMaker** — Reinforcement learning, advanced ML
- **Amazon Personalize** — Recommendation engine được quản lý
- **AWS Step Functions** — Orchestrate automated decision workflows
- **Amazon EventBridge** — Trigger actions dựa trên predictions

---

## So Sánh Nhanh

| Loại Analytics    | Câu Hỏi         | Kỹ Thuật Chính        | Độ Phức Tạp | Giá Trị KD  | Ví Dụ Nhanh                |
| ----------------- | --------------- | --------------------- | ----------- | ----------- | -------------------------- |
| **Descriptive**   | Chuyện gì đã xảy ra? | Aggregation, Charts   | ⭐          | ⭐⭐        | Dashboard doanh thu        |
| **Diagnostic**    | Tại sao xảy ra? | Drill-down, Correlation | ⭐⭐       | ⭐⭐⭐      | Root cause analysis        |
| **Predictive**    | Sẽ xảy ra gì?   | ML, Forecasting       | ⭐⭐⭐      | ⭐⭐⭐⭐    | Churn prediction           |
| **Prescriptive**  | Nên làm gì?     | Optimization, RL      | ⭐⭐⭐⭐    | ⭐⭐⭐⭐⭐  | Dynamic pricing automation |

---

## Ví Dụ Thực Tế Theo Ngành

### E-commerce (Thương Mại Điện Tử)

| Loại          | Câu Hỏi Thực Tế                                               |
| ------------- | ------------------------------------------------------------- |
| Descriptive   | Doanh thu hôm nay là bao nhiêu? Top sản phẩm bán chạy?       |
| Diagnostic    | Tại sao tỷ lệ conversion giảm 30% sau khi redesign checkout?  |
| Predictive    | Khách hàng nào có khả năng mua lại trong 7 ngày tới?          |
| Prescriptive  | Nên hiển thị sản phẩm nào cho user này để tối đa doanh thu?  |

### Fintech (Tài Chính Công Nghệ)

| Loại          | Câu Hỏi Thực Tế                                               |
| ------------- | ------------------------------------------------------------- |
| Descriptive   | Tổng số giao dịch theo loại trong tháng?                      |
| Diagnostic    | Tại sao số tiền hoàn trả tăng 200% tuần này?                 |
| Predictive    | Giao dịch này có phải gian lận không? (probability)          |
| Prescriptive  | Nên phê duyệt hay từ chối khoản vay của khách hàng X?        |

### SaaS / Tech

| Loại          | Câu Hỏi Thực Tế                                               |
| ------------- | ------------------------------------------------------------- |
| Descriptive   | MAU (Monthly Active Users — Người Dùng Hoạt Động Tháng)?     |
| Diagnostic    | Tại sao feature Y ít được dùng dù team tốn 3 tháng xây?      |
| Predictive    | Subscription nào sẽ expire mà không renew tháng tới?         |
| Prescriptive  | Nên upgrade plan nào cho công ty Z để tối đa giá trị?        |

---

## Analytics Trong Hệ Sinh Thái AWS

### Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA SOURCES (Nguồn Dữ Liệu)                │
│  Databases | APIs | Logs | IoT | Clickstream | 3rd-party        │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Ingest (Thu Nạp)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              INGESTION LAYER (Tầng Thu Nạp)                    │
│  Kinesis Data Streams | Kinesis Firehose | MSK | DMS | Transfer │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Store Raw
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│               STORAGE LAYER (Tầng Lưu Trữ)                     │
│  S3 (Data Lake) | Redshift (DW) | DynamoDB | RDS                │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Process & Transform
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│             PROCESSING LAYER (Tầng Xử Lý)                      │
│  AWS Glue | Amazon EMR | Kinesis Analytics | Lambda             │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Serve
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│               SERVING LAYER (Tầng Phục Vụ)                     │
│  Athena | Redshift | OpenSearch | SageMaker | QuickSight        │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│            ANALYTICS TYPES SUPPORTED (Các Loại Analytics)      │
│  Descriptive   Diagnostic   Predictive   Prescriptive           │
│  (QuickSight)  (Athena)     (SageMaker)  (Personalize)         │
└─────────────────────────────────────────────────────────────────┘
```

### AWS Services Theo Loại Analytics

| Loại Analytics  | AWS Services Chính                                           |
| --------------- | ------------------------------------------------------------ |
| Descriptive     | QuickSight, Athena, Redshift, CloudWatch Dashboards          |
| Diagnostic      | Athena (ad-hoc), OpenSearch (log analysis), QuickSight drill-down |
| Predictive      | SageMaker, Forecast, Fraud Detector, Comprehend              |
| Prescriptive    | SageMaker RL, Personalize, EventBridge + Lambda automation   |

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q1: Giải thích sự khác biệt giữa Descriptive và Diagnostic Analytics?**

> **Trả lời:** Descriptive Analytics (Phân Tích Mô Tả) trả lời "Điều gì đã xảy ra?" — ví dụ doanh thu giảm 20%. Diagnostic Analytics (Phân Tích Chẩn Đoán) đi sâu hơn để trả lời "Tại sao?" — ví dụ doanh thu giảm vì campaign email bị lỗi. Descriptive cho thấy triệu chứng, Diagnostic tìm nguyên nhân gốc rễ.

**Q2: Predictive Analytics khác Prescriptive Analytics thế nào?**

> **Trả lời:** Predictive Analytics (Phân Tích Dự Đoán) nói "Điều gì có thể xảy ra" — ví dụ: khách hàng X có 80% khả năng rời đi. Prescriptive Analytics (Phân Tích Đề Xuất) đi xa hơn và nói "Bạn nên làm gì" — ví dụ: gửi voucher giảm giá 20% cho khách X ngay bây giờ. Prescriptive kết hợp dự đoán với business constraints và optimization.

**Q3: Trong dự án analytics thực tế, thường bắt đầu từ loại nào?**

> **Trả lời:** Hầu hết dự án bắt đầu từ Descriptive Analytics — xây dựng dashboard và báo cáo cơ bản. Sau đó mở rộng sang Diagnostic khi cần hiểu nguyên nhân. Predictive và Prescriptive cần nhiều dữ liệu và kỹ năng hơn, thường là giai đoạn sau khi đã có hạ tầng dữ liệu vững.

### Câu Hỏi Nâng Cao

**Q4: Với hệ thống real-time fraud detection, đây là loại analytics gì và nên dùng AWS services nào?**

> **Trả lời:** Đây là **Predictive Analytics** (phân loại giao dịch có gian lận hay không) kết hợp **Prescriptive** (tự động chặn hoặc yêu cầu xác thực thêm). AWS stack phù hợp: Kinesis Data Streams (ingestion real-time) → Lambda (invoke model) → SageMaker Endpoint hoặc Amazon Fraud Detector (prediction) → EventBridge/SNS (prescriptive action: block/notify). Toàn bộ pipeline cần latency < 100ms.

**Q5: Một công ty có đủ dữ liệu nhưng chưa làm analytics bao giờ. Bạn tư vấn bắt đầu từ đâu?**

> **Trả lời:** Bắt đầu từ Descriptive Analytics — đây là nền tảng và mang lại giá trị nhanh nhất. Ưu tiên: (1) Xác định 3-5 KPI (Key Performance Indicators — Chỉ Số Hiệu Suất Chính) quan trọng nhất của business; (2) Thu thập và làm sạch dữ liệu cho các KPI đó; (3) Xây dựng dashboard đơn giản với QuickSight hoặc Grafana; (4) Khi stakeholder tin tưởng dữ liệu, mở rộng sang Diagnostic và Predictive.

---

## 💡 Key Takeaways

1. **4 loại analytics** tạo thành một spectrum từ "mô tả quá khứ" đến "định hướng tương lai"
2. **Descriptive là nền tảng** — không có nó, 3 loại kia không thể hoạt động tốt
3. **Giá trị tỷ lệ thuận với độ phức tạp** — Prescriptive khó nhất nhưng impact lớn nhất
4. **AWS có tools cho mọi loại** — từ QuickSight (Descriptive) đến SageMaker (Predictive/Prescriptive)
5. **Trong phỏng vấn:** Luôn hỏi "loại analytics nào?" trước khi đề xuất solution

---

**← [README.md](./README.md)** | **→ [2-data-pipeline-concepts.md](./2-data-pipeline-concepts.md)**
