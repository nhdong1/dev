# 🎯 Module 12: Interview Prep — Chuẩn Bị Phỏng Vấn AWS Analytics

> Tổng hợp toàn bộ kiến thức AWS Analytics dưới góc độ phỏng vấn — câu hỏi thường gặp, mô hình trả lời và kịch bản thiết kế hệ thống thực tế.

## 📚 Nội Dung Module

| File | Mô Tả | Trạng Thái |
| ---- | ------ | ---------- |
| [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | Top 20 câu hỏi phỏng vấn + gợi ý trả lời chi tiết | ✅ |
| [system-design-scenarios.md](./system-design-scenarios.md) | Kịch bản thiết kế hệ thống: real-time pipeline, data lake, BI platform | ✅ |

---

## 🗺️ Lộ Trình Chuẩn Bị

### Giai Đoạn 1 — Nắm Vững Kiến Thức Nền (Tuần 1-2)

Trước khi luyện phỏng vấn, đảm bảo bạn đã học kỹ các module:

```
01-fundamentals/  → Nền tảng: Batch, Streaming, Data Lake, ETL
02-kinesis/       → Real-time streaming — luôn được hỏi
03-glue/          → ETL & Data Catalog
04-athena/        → Serverless SQL query
05-redshift/      → MPP Data Warehouse
06-emr/           → Big Data processing
07-lake-formation/ → Governance & security
11-data-architecture/ → Lambda, Kappa, Medallion, Data Mesh
```

### Giai Đoạn 2 — Luyện Câu Hỏi (Tuần 3)

1. Đọc `INTERVIEW_GUIDE.md` — nắm 20 câu hỏi thường gặp
2. Tự trả lời không nhìn tài liệu, ghi âm lại
3. Nghe lại và so sánh với gợi ý trả lời
4. Luyện giải thích trade-offs bằng lời nói (không phải viết)

### Giai Đoạn 3 — System Design (Tuần 4)

1. Đọc `system-design-scenarios.md` — nắm 5 kịch bản thực tế
2. Tự thiết kế whiteboard solution trong 30 phút
3. So sánh với giải pháp tham khảo
4. Luyện giải thích kiến trúc rõ ràng, ngắn gọn

---

## 🎯 Phân Loại Câu Hỏi Theo Độ Phổ Biến

### Nhóm A — Luôn Được Hỏi (100%)

- Kinesis Data Streams (Luồng Dữ Liệu Kinesis) vs Kinesis Firehose (Vòi Dữ Liệu Kinesis)
- Athena (Truy Vấn Không Máy Chủ) vs Redshift — khi nào dùng cái nào?
- Glue Data Catalog (Danh Mục Dữ Liệu) là gì và tại sao cần?
- Data Lake (Hồ Dữ Liệu) vs Data Warehouse (Kho Dữ Liệu) — khác nhau thế nào?
- Thiết kế real-time analytics pipeline (Đường Ống Phân Tích Thời Gian Thực)

### Nhóm B — Thường Được Hỏi (70-80%)

- Redshift DISTKEY (Khóa Phân Phối) và SORTKEY (Khóa Sắp Xếp)
- EMR vs Glue — khi nào dùng EMR, khi nào dùng Glue?
- Medallion Architecture (Kiến Trúc Huy Chương) Bronze/Silver/Gold
- Lake Formation (Lake Formation) column-level security
- Kinesis vs MSK (Apache Kafka) — trade-offs

### Nhóm C — Đôi Khi Được Hỏi (40-50%)

- Lambda Architecture (Kiến Trúc Lambda) vs Kappa Architecture (Kiến Trúc Kappa)
- Athena Federated Query (Truy Vấn Liên Kết Athena)
- OpenSearch (Tìm Kiếm Mở) vs Redshift — log analytics use case
- Data Mesh (Lưới Dữ Liệu) và domain ownership
- Cost optimization (Tối Ưu Chi Phí) cho từng dịch vụ

---

## 📊 Tiêu Chí Đánh Giá Phỏng Vấn

Nhà tuyển dụng thường đánh giá theo 4 tiêu chí:

### 1. Breadth — Độ Rộng Kiến Thức

> *Bạn biết bao nhiêu dịch vụ và khi nào dùng cái nào?*

Cần biết: Kinesis, Glue, Athena, Redshift, EMR, Lake Formation, MSK
Điểm cộng: QuickSight, OpenSearch, Data Architecture patterns

### 2. Depth — Độ Sâu Kiến Thức

> *Bạn hiểu cơ chế hoạt động bên trong ra sao?*

Ví dụ: Không chỉ biết "Kinesis có Shard" mà còn biết cách tính số Shard, ảnh hưởng đến throughput (Thông Lượng) và chi phí.

### 3. Trade-off Thinking — Tư Duy Đánh Đổi

> *Bạn có thể giải thích tại sao chọn A thay vì B không?*

Không có câu trả lời đúng tuyệt đối — nhà tuyển dụng muốn thấy bạn xem xét ngữ cảnh (context), yêu cầu (requirements) và ràng buộc (constraints).

### 4. Real-world Experience — Kinh Nghiệm Thực Tế

> *Bạn đã xây dựng gì, giải quyết vấn đề gì thực sự?*

Chuẩn bị 2-3 câu chuyện theo phương pháp STAR (Situation — Task — Action — Result).

---

## 🛠️ Phương Pháp STAR Cho Data Engineering

**STAR** = Situation (Tình Huống) — Task (Nhiệm Vụ) — Action (Hành Động) — Result (Kết Quả)

### Template STAR Cho Pipeline Design

```
Situation (Tình Huống):
  "Hệ thống cũ của chúng tôi xử lý [loại dữ liệu] theo batch mỗi [tần suất],
   gây ra [vấn đề cụ thể] cho [stakeholder]."

Task (Nhiệm Vụ):
  "Tôi được giao thiết kế pipeline mới đáp ứng [latency / throughput / cost requirement]."

Action (Hành Động):
  "Tôi chọn [dịch vụ A] vì [lý do kỹ thuật cụ thể].
   Tôi cấu hình [chi tiết kỹ thuật].
   Tôi xử lý vấn đề [thách thức cụ thể] bằng cách [giải pháp]."

Result (Kết Quả):
  "Pipeline mới giảm latency từ [X] xuống [Y], giảm chi phí [Z]%,
   và [impact thực tế với business]."
```

### Ví Dụ STAR Thực Tế

```
Situation: Log ứng dụng được write vào S3 mỗi 1 giờ batch, team ops không
  biết khi nào có lỗi production cho đến khi khách hàng báo cáo.

Task: Xây dựng real-time log monitoring pipeline với alert < 2 phút.

Action: Dùng Kinesis Data Firehose (KDF — Vòi Dữ Liệu Kinesis) thu log từ
  EC2, transform với Lambda để enrich log context, deliver sang OpenSearch
  (Tìm Kiếm Mở). Cấu hình alert rule trong OpenSearch Dashboards.

Result: MTTD (Mean Time To Detect — Thời Gian Trung Bình Phát Hiện) giảm từ
  60+ phút xuống < 90 giây. Team ops giảm 40% thời gian điều tra sự cố.
```

---

## 💡 Mẹo Phỏng Vấn

### Khi Được Hỏi Về Thiết Kế

1. **Hỏi lại requirements trước** — đừng vội thiết kế ngay
   - Volume: bao nhiêu records/giây?
   - Latency: chấp nhận delay bao lâu?
   - Budget: có giới hạn chi phí không?

2. **Bắt đầu bằng high-level rồi đào sâu**
   - Phác thảo data flow trước
   - Sau đó giải thích từng thành phần

3. **Chủ động đề cập trade-offs**
   - "Tôi chọn X vì... nhưng nhược điểm là Y"
   - Điều này thể hiện bạn nghĩ thực tế

### Khi Không Biết Câu Trả Lời

```
"Tôi chưa dùng [dịch vụ X] trực tiếp, nhưng từ hiểu biết về
[dịch vụ tương tự Y], tôi đoán cơ chế có thể là [suy luận].
Nếu tôi cần giải quyết bài toán này, tôi sẽ bắt đầu bằng cách [approach]."
```

Trung thực tốt hơn nói bừa — và thể hiện tư duy quan trọng hơn là nhớ facts.

### Từ Khóa Kỹ Thuật Nên Dùng Tự Nhiên

| Thuật Ngữ | Nghĩa | Dùng Khi |
| --------- | ----- | --------- |
| Throughput (Thông Lượng) | Dữ liệu xử lý per giây | Nói về capacity |
| Latency (Độ Trễ) | Thời gian từ event đến kết quả | Nói về real-time |
| Idempotent (Bất Biến Khi Lặp) | Xử lý nhiều lần cùng kết quả | Nói về reliability |
| Exactly-once (Chính Xác Một Lần) | Đảm bảo không bỏ, không trùng | Nói về data quality |
| Partitioning (Phân Vùng) | Chia dữ liệu theo key | Nói về performance |
| Compaction (Nén Gộp File) | Gộp nhiều file nhỏ thành lớn | Nói về optimization |
| Cardinality (Số Lượng Giá Trị Riêng Biệt) | Số distinct values trong cột | Nói về indexing |
| Pushdown Predicate (Đẩy Điều Kiện Xuống) | Filter tại nguồn trước khi load | Nói về Glue/Spark |

---

## 📋 Checklist Trước Buổi Phỏng Vấn

### 1 Tuần Trước

- [ ] Đọc hết INTERVIEW_GUIDE.md ít nhất 2 lần
- [ ] Luyện nói 5 câu hỏi nhóm A bằng lời không nhìn tài liệu
- [ ] Xem lại system-design-scenarios.md, tự vẽ diagram
- [ ] Chuẩn bị ít nhất 2 câu chuyện STAR với số liệu cụ thể

### 1 Ngày Trước

- [ ] Ôn lại bảng so sánh nhanh (Kinesis vs MSK, Athena vs Redshift, EMR vs Glue)
- [ ] Xem lại acronyms: KDS, KDF, KDA, MPP, DPU, RPU, SPICE, WLM
- [ ] Chuẩn bị câu hỏi để hỏi ngược lại nhà tuyển dụng

### Trong Buổi Phỏng Vấn

- [ ] Hỏi lại requirements khi được hỏi system design
- [ ] Nói to suy nghĩ khi giải quyết vấn đề
- [ ] Đề cập trade-offs chủ động
- [ ] Kết thúc câu trả lời bằng "Bạn có muốn tôi đào sâu vào phần nào không?"

---

## 🔗 Điều Hướng Module

| Tài Liệu | Mô Tả |
| -------- | ------ |
| [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) | 20 câu hỏi thường gặp nhất |
| [system-design-scenarios.md](./system-design-scenarios.md) | 5 kịch bản system design thực tế |
| [../README.md](../README.md) | Tổng quan toàn bộ AWS Analytics |
| [../INDEX.md](../INDEX.md) | Chỉ mục đầy đủ |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Module:** 12 / 12
**Trạng Thái:** ✅ Hoàn thành
