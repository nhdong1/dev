# QuickSight Basics — Nền Tảng Amazon QuickSight

> Hiểu rõ các thành phần cốt lõi: Dataset (Bộ Dữ Liệu), Analysis (Phân Tích), Dashboard (Bảng Điều Khiển) và Stories (Câu Chuyện Dữ Liệu) — nền tảng để làm chủ QuickSight.

---

## 📚 Mục Lục

1. [Luồng Dữ Liệu Trong QuickSight](#1-luồng-dữ-liệu-trong-quicksight)
2. [Dataset — Bộ Dữ Liệu](#2-dataset--bộ-dữ-liệu)
3. [Analysis — Không Gian Phân Tích](#3-analysis--không-gian-phân-tích)
4. [Visual Types — Các Loại Biểu Đồ](#4-visual-types--các-loại-biểu-đồ)
5. [Dashboard — Bảng Điều Khiển](#5-dashboard--bảng-điều-khiển)
6. [Row-Level Security — Bảo Mật Cấp Hàng](#6-row-level-security--bảo-mật-cấp-hàng)
7. [ML Insights — Phân Tích Thông Minh](#7-ml-insights--phân-tích-thông-minh)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Luồng Dữ Liệu Trong QuickSight

```
┌─────────────────────────────────────────────────────────────┐
│                    Data Sources (Nguồn Dữ Liệu)             │
│   S3 │ Athena │ Redshift │ RDS │ Salesforce │ Snowflake...   │
└──────────────────────────┬──────────────────────────────────┘
                           │ Kết nối & import/query
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     Dataset (Bộ Dữ Liệu)                    │
│  • Định nghĩa schema (Lược Đồ)                               │
│  • Tạo calculated fields (Trường Tính Toán)                  │
│  • Áp dụng data preparation (Chuẩn Bị Dữ Liệu)              │
│  • Chọn SPICE hoặc Direct Query                              │
└──────────────────────────┬──────────────────────────────────┘
                           │ Sử dụng trong
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Analysis (Không Gian Phân Tích)             │
│  • Tạo và sắp xếp visuals (Biểu Đồ)                         │
│  • Định nghĩa filters (Bộ Lọc) và parameters (Tham Số)      │
│  • Cộng tác với team (Author role)                          │
└──────────┬───────────────────────────────────┬──────────────┘
           │ Publish (Xuất Bản)                │ Embed (Nhúng)
           ▼                                   ▼
┌──────────────────┐                ┌──────────────────────────┐
│    Dashboard     │                │  Embedded Dashboard       │
│ (Bảng Điều Khiển)│                │  (Dashboard Nhúng        │
│  Chỉ đọc        │                │   vào ứng dụng)          │
│  Chia sẻ được   │                └──────────────────────────┘
└──────────────────┘
```

---

## 2. Dataset — Bộ Dữ Liệu

### 2.1 Tạo Dataset

Dataset là lớp trừu tượng giữa nguồn dữ liệu thô và analysis. Một dataset có thể được tái sử dụng trong nhiều analysis và dashboard khác nhau.

```
Data Source
    │
    ▼
Dataset Creation (Tạo Bộ Dữ Liệu)
    ├── Select tables/views (Chọn bảng/view)
    ├── Join tables (Nối bảng) — tối đa 32 bảng
    ├── Apply filters (Áp dụng bộ lọc)
    ├── Add calculated fields (Thêm trường tính toán)
    └── Choose storage: SPICE or Direct Query
```

### 2.2 Calculated Fields — Trường Tính Toán

Calculated field là cột mới được tính từ các cột có sẵn, sử dụng hàm QuickSight.

```sql
-- Ví dụ Calculated Fields

-- Profit Margin (Biên Lợi Nhuận) — phần trăm lợi nhuận
{profit} / {revenue} * 100

-- Full Name (Tên Đầy Đủ) — nối chuỗi
concat({first_name}, ' ', {last_name})

-- Phân loại theo giá trị
ifelse({amount} > 1000, 'High', {amount} > 500, 'Medium', 'Low')

-- Date Truncation (Rút Gọn Ngày) — chuyển về đầu tháng
truncDate('MM', {order_date})

-- Running Total (Tổng Lũy Kế)
runningSum(sum({revenue}), [ASC {order_date}])
```

### 2.3 Data Preparation — Chuẩn Bị Dữ Liệu

| Tính Năng | Mô Tả |
|-----------|-------|
| **Rename fields** | Đổi tên cột cho dễ hiểu |
| **Change data types** | Chuyển đổi kiểu dữ liệu (String → Date...) |
| **Exclude fields** | Loại bỏ cột không cần thiết |
| **Split field** | Tách một cột thành nhiều cột |
| **Pivot** | Xoay bảng từ dạng dài sang dạng rộng |
| **Duplicate** | Tạo bản sao của cột |

### 2.4 Dataset Refresh — Làm Mới Bộ Dữ Liệu

Chỉ áp dụng cho SPICE mode — dữ liệu trong SPICE phải được refresh định kỳ.

```
Refresh Schedule (Lịch Làm Mới):
├── Manual refresh — thủ công khi cần
├── Scheduled refresh — lên lịch tự động
│   ├── Daily (Hàng Ngày) — 00:00 UTC mỗi ngày
│   ├── Weekly (Hàng Tuần)
│   └── Monthly (Hàng Tháng)
└── Incremental refresh (Làm Mới Tăng Dần)
    └── Chỉ import dữ liệu mới/thay đổi → tiết kiệm SPICE
```

**Incremental Refresh — Làm Mới Tăng Dần:**

```sql
-- Cấu hình incremental refresh dựa trên cột ngày
-- QuickSight sẽ chỉ import dữ liệu từ ngày bắt đầu của window
-- Ví dụ: chỉ import 7 ngày gần nhất

Window_column: order_date
Window_size: 7 days
```

---

## 3. Analysis — Không Gian Phân Tích

### 3.1 Cấu Trúc Analysis

```
Analysis
├── Sheet 1 (Trang 1) — Overview Dashboard
│   ├── Visual: Bar Chart (Biểu Đồ Thanh) — Revenue by Region
│   ├── Visual: Line Chart (Biểu Đồ Đường) — Trend Over Time
│   └── Visual: KPI — Total Revenue
├── Sheet 2 (Trang 2) — Product Details
│   ├── Visual: Table (Bảng) — Product Rankings
│   └── Visual: Pie Chart (Biểu Đồ Tròn) — Category Share
└── Controls (Điều Khiển)
    ├── Filter Control — lọc theo ngày
    └── Parameter Control — chọn metric
```

### 3.2 Filters — Bộ Lọc

Filter kiểm soát dữ liệu được hiển thị trong visual.

```
Loại Filter:
├── Text filter (Bộ Lọc Văn Bản)
│   ├── Filter list (Danh sách lọc) — chọn từ danh sách
│   ├── Custom filter — nhập giá trị tùy chỉnh
│   └── Contains/Starts with/Ends with
├── Number filter (Bộ Lọc Số)
│   ├── Between (Giữa) — ví dụ: 100 đến 500
│   ├── Greater than / Less than
│   └── Top/Bottom N — lấy N giá trị cao/thấp nhất
└── Date filter (Bộ Lọc Ngày)
    ├── Relative date — "Last 30 days", "This month"
    ├── Date range — từ ngày đến ngày
    └── Exact date — ngày cụ thể

Phạm Vi Áp Dụng Filter:
├── This visual only — chỉ visual hiện tại
├── All visuals on this sheet — toàn bộ sheet
└── All visuals on all sheets — toàn bộ analysis
```

### 3.3 Parameters — Tham Số

Parameter cho phép người dùng tương tác với dashboard bằng cách thay đổi giá trị đầu vào.

```
Parameter ──▶ Control (Dropdown / Slider / Text input)
    │
    ├──▶ Filter (lọc dữ liệu theo parameter)
    ├──▶ Calculated field (dùng parameter trong công thức)
    └──▶ URL action (điều hướng với giá trị parameter)
```

**Ví dụ: Parameter cho Year (Năm)**

```
1. Tạo parameter: "selected_year" kiểu Integer
2. Tạo control: Dropdown (Danh Sách Thả Xuống) với values 2022, 2023, 2024
3. Tạo filter: year({order_date}) = ${selected_year}
4. Người dùng chọn năm → toàn bộ dashboard cập nhật
```

### 3.4 Actions — Hành Động

Action cho phép tạo tương tác giữa các visual (drill-down, cross-filter, URL navigation).

| Loại Action | Mô Tả | Ví Dụ |
|-------------|-------|-------|
| **Filter action** | Click vào visual A lọc visual B | Click vào Region → lọc Product chart |
| **Navigation action** | Chuyển đến sheet khác | Click vào Category → mở sheet detail |
| **URL action** | Mở URL bên ngoài | Click Product ID → mở trang sản phẩm |

---

## 4. Visual Types — Các Loại Biểu Đồ

### 4.1 Bảng Tổng Hợp Visual

| Loại Visual | Dùng Khi | Tốt Nhất Cho |
|-------------|----------|-------------|
| **Bar Chart** (Biểu Đồ Thanh) | So sánh giá trị giữa các danh mục | Revenue by Region, Sales by Product |
| **Line Chart** (Biểu Đồ Đường) | Xu hướng theo thời gian | Monthly Revenue Trend, DAU over time |
| **Pie / Donut Chart** (Biểu Đồ Tròn / Bánh Mỳ) | Tỷ lệ phần trăm | Market Share, Category Distribution |
| **Scatter Plot** (Biểu Đồ Phân Tán) | Mối quan hệ giữa 2 biến số | Revenue vs. Profit, Price vs. Volume |
| **Heat Map** (Bản Đồ Nhiệt) | Ma trận giá trị | Sales by Day-of-Week và Hour |
| **Tree Map** (Bản Đồ Cây) | Tỷ lệ dạng hình chữ nhật lồng nhau | Hierarchical revenue breakdown |
| **Funnel Chart** (Biểu Đồ Phễu) | Conversion rate theo từng bước | Sales funnel, User onboarding |
| **Gauge Chart** (Đồng Hồ Đo) | Đo lường một giá trị so với mục tiêu | Current quota vs. target |
| **KPI Visual** | Hiển thị một metric quan trọng | Total Revenue, MoM growth |
| **Table** (Bảng Dữ Liệu) | Dữ liệu chi tiết dạng bảng | Transaction list, Raw data |
| **Pivot Table** (Bảng Xoay) | Tổng hợp nhiều chiều | Revenue by Region × Product |
| **Geospatial / Map** (Bản Đồ Địa Lý) | Phân phối theo địa lý | Sales by State, Users by Country |

### 4.2 AutoGraph — Tự Động Chọn Visual

QuickSight có tính năng **AutoGraph** — tự động chọn loại visual phù hợp nhất dựa trên kiểu dữ liệu bạn chọn:

```
Dimension (Text/Date) + Measure (Number)
    → Bar Chart hoặc Line Chart

Measure + Measure
    → Scatter Plot

Date + Measure
    → Line Chart

Geography + Measure
    → Map
```

### 4.3 Conditional Formatting — Định Dạng Có Điều Kiện

Tô màu ô hoặc giá trị dựa trên quy tắc:

```
Ví dụ trong Table/Pivot:
├── Revenue > 1,000,000  →  Tô xanh lá (Good)
├── Revenue 500,000-1M   →  Tô vàng (Warning)
└── Revenue < 500,000    →  Tô đỏ (Bad)

Ví dụ KPI Visual:
├── Growth > 10%   →  Mũi tên lên màu xanh
├── Growth 0-10%   →  Mũi tên ngang màu vàng
└── Growth < 0%    →  Mũi tên xuống màu đỏ
```

---

## 5. Dashboard — Bảng Điều Khiển

### 5.1 Publish Dashboard — Xuất Bản Bảng Điều Khiển

Khi Analysis sẵn sàng, xuất bản thành Dashboard để chia sẻ với người dùng cuối (Reader).

```
Analysis ──Publish──▶ Dashboard (Version 1)
    │                      │
    │ (Author chỉnh sửa)   │ (Readers xem)
    │                      │
    ▼                      ▼
Analysis ──Publish──▶ Dashboard (Version 2)
                       (Version cũ vẫn còn)
```

**Lưu ý quan trọng:**
- Dashboard là **snapshot** của Analysis tại thời điểm publish
- Khi thay đổi Analysis, phải publish lại để cập nhật Dashboard
- Có thể duy trì nhiều version song song
- Reader không thể chỉnh sửa, chỉ xem và filter

### 5.2 Sharing Dashboard — Chia Sẻ Bảng Điều Khiển

```
Chia sẻ Dashboard:
├── Share with specific users (Chia Sẻ Với Người Dùng Cụ Thể)
│   └── Nhập email của Reader
├── Share with groups (Chia Sẻ Với Nhóm)
│   └── Sử dụng QuickSight Groups hoặc Active Directory Groups
├── Public access (Truy Cập Công Khai)
│   └── Bất kỳ ai có link đều xem được (cần bật)
└── Embedded (Nhúng)
    └── Tích hợp vào ứng dụng qua SDK
```

### 5.3 Email Reports — Báo Cáo Qua Email

Lên lịch gửi snapshot (Ảnh Chụp) của Dashboard qua email định kỳ.

```
Email Report Schedule:
├── Frequency: Daily / Weekly / Monthly
├── Format: PDF hoặc CSV (cho table visuals)
├── Recipients: Danh sách email
└── Sheet selection: Chọn sheet nào sẽ gửi
```

### 5.4 Alerts — Cảnh Báo Dữ Liệu

Thiết lập ngưỡng cảnh báo — QuickSight tự động gửi email khi metric vượt ngưỡng.

```
Alert ví dụ:
    Metric: Daily Revenue
    Condition: < 100,000
    → Gửi email cảnh báo ngay khi điều kiện thỏa mãn
```

---

## 6. Row-Level Security — Bảo Mật Cấp Hàng

### 6.1 Khái Niệm RLS

RLS (Row-Level Security — Bảo Mật Cấp Hàng) cho phép mỗi người dùng chỉ thấy dữ liệu họ được phép xem, mà không cần tạo nhiều dataset/dashboard riêng biệt.

```
Ví dụ thực tế:
Sales Manager Region A  →  Chỉ thấy dữ liệu Region A
Sales Manager Region B  →  Chỉ thấy dữ liệu Region B
CEO                     →  Thấy dữ liệu tất cả Region
```

### 6.2 Cấu Hình RLS

Tạo một bảng quy tắc (rules dataset) với format:

```
UserName (hoặc GroupName) | filter_column | filter_value
─────────────────────────────────────────────────────────
john@company.com         | region        | North
jane@company.com         | region        | South
admin@company.com        | region        | (để trống = không giới hạn)
```

**Các bước cấu hình:**

```
1. Tạo rules CSV/dataset với mapping user ↔ filter value
2. Upload rules dataset lên QuickSight
3. Mở Dataset cần bảo vệ
4. Permissions → Row-level security → Enable
5. Chọn rules dataset
6. QuickSight tự động áp dụng filter theo user đăng nhập
```

### 6.3 Column-Level Security — Bảo Mật Cấp Cột

Ẩn hoàn toàn một số cột với người dùng không được phép:

```
Dataset → Permissions → Column-level security
    │
    ├── Restricted columns: [salary, ssn, phone]
    └── Allowed groups: [HR_Team, Admin]
    
Kết quả:
    HR_Team  → thấy cột salary, ssn, phone
    Others   → không thấy các cột này
```

---

## 7. ML Insights — Phân Tích Thông Minh

QuickSight tích hợp sẵn các tính năng ML (Machine Learning — Học Máy) không cần cấu hình:

### 7.1 Anomaly Detection — Phát Hiện Bất Thường

Tự động phát hiện điểm dữ liệu bất thường trong time series (Chuỗi Thời Gian):

```
Ví dụ:
Revenue ngày thứ Tư đột ngột giảm 80%
    → QuickSight highlight điểm này
    → Gửi alert (Cảnh Báo) ngay lập tức
    → Đề xuất nguyên nhân có thể (root cause contribution)
```

### 7.2 Forecasting — Dự Báo

Dự báo giá trị tương lai dựa trên historical data (Dữ Liệu Lịch Sử):

```
Input: 24 tháng dữ liệu revenue
Output: Dự báo 6 tháng tiếp theo với confidence interval (Khoảng Tin Cậy)
```

### 7.3 Narrative — Tường Thuật Tự Động

Auto-generate (Tự Động Tạo) text mô tả những điểm quan trọng từ dữ liệu:

```
Ví dụ output tự động:
"Revenue tháng này tăng 23% so với tháng trước,
 chủ yếu do Region North đóng góp 45% tổng doanh thu.
 Sản phẩm A đang có xu hướng giảm liên tục 3 tháng."
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: Sự khác biệt giữa Analysis và Dashboard trong QuickSight là gì?

**Trả lời:**
- **Analysis** là workspace dành cho Author — có thể chỉnh sửa, thêm visual, thay đổi cấu trúc. Đây là môi trường làm việc của người xây dựng BI.
- **Dashboard** là bản được publish từ Analysis — chỉ đọc (read-only), dành cho Reader. Dashboard là snapshot tại thời điểm publish; khi Analysis thay đổi phải publish lại.

### Q2: Khi nào dùng SPICE, khi nào dùng Direct Query?

**Trả lời:**
- **SPICE:** Dùng khi cần hiệu suất cao, nhiều concurrent user, dữ liệu không thay đổi liên tục. SPICE import dữ liệu vào in-memory → query cực nhanh dù nguồn gốc chậm.
- **Direct Query:** Dùng khi cần real-time data (dữ liệu thay đổi liên tục từng giây), dữ liệu quá lớn cho SPICE quota, hoặc khi nguồn dữ liệu (Redshift, RDS) đã đủ nhanh.

### Q3: Làm thế nào để mỗi Sales Manager chỉ thấy dữ liệu của region mình?

**Trả lời:** Dùng **Row-Level Security (RLS)**:
1. Tạo rules dataset: mapping `username → region`
2. Enable RLS trên dataset chính
3. Gán rules dataset
4. QuickSight tự động filter theo user đang đăng nhập — không cần tạo nhiều dashboard riêng

### Q4: Tại sao không nên dùng nhiều calculated fields phức tạp trong QuickSight?

**Trả lời:** Calculated fields trong QuickSight được tính tại query time — với Direct Query, mỗi field thêm là một lần compute. Tốt hơn là:
- Tạo transformation (Chuyển Đổi) trong **Glue ETL** hoặc **dbt** trước
- Lưu kết quả vào Redshift/S3 dạng đã tính sẵn
- QuickSight chỉ đọc giá trị cuối → hiệu suất cao hơn, dễ bảo trì hơn

---

## 🔗 Liên Kết Tiếp Theo

- [SPICE Engine — In-memory Cache](./2-spice-engine.md)
- [Embedded Analytics — Nhúng vào ứng dụng](./3-embedded-analytics.md)
- [QuickSight Module Overview](./README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-17
