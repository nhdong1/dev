# 🏗️ Kiến Trúc Dữ Liệu Hiện Đại — Data Architecture Patterns

> Module này trình bày các mẫu kiến trúc dữ liệu (data architecture patterns) quan trọng nhất trong thực tế: Lambda Architecture (Kiến Trúc Lambda), Kappa Architecture (Kiến Trúc Kappa), Data Mesh (Lưới Dữ Liệu) và Medallion Architecture (Kiến Trúc Huy Chương). Hiểu rõ trade-off của từng mẫu là chìa khóa để thiết kế hệ thống dữ liệu hiệu quả trên AWS.

## 📚 Mục Lục Module

| File | Chủ Đề | Mức Độ |
|------|--------|--------|
| [1-lambda-architecture.md](./1-lambda-architecture.md) | Lambda Architecture — Batch + Speed + Serving Layer | ⭐⭐⭐ |
| [2-kappa-architecture.md](./2-kappa-architecture.md) | Kappa Architecture — Stream-only, đơn giản hóa Lambda | ⭐⭐⭐ |
| [3-data-mesh.md](./3-data-mesh.md) | Data Mesh — Domain ownership, self-serve platform | ⭐⭐⭐ |
| [4-medallion-architecture.md](./4-medallion-architecture.md) | Medallion Architecture — Bronze, Silver, Gold layers | ⭐⭐ |

---

## 🎯 Tại Sao Kiến Trúc Dữ Liệu Quan Trọng?

Lựa chọn sai kiến trúc ngay từ đầu dẫn đến:

- **Kỹ thuật nợ** (technical debt) tích lũy theo thời gian — rất tốn kém để refactor
- **Hiệu suất kém** khi dữ liệu tăng trưởng vượt thiết kế ban đầu
- **Chi phí vận hành cao** do không tối ưu hóa đường đi dữ liệu
- **Sự chậm trễ** trong việc đưa insight (thông tin sâu sắc) ra quyết định kinh doanh

---

## 🗺️ Bản Đồ Kiến Trúc — Khi Nào Dùng Gì?

```
                    Yêu cầu xử lý
                         │
           ┌─────────────┼─────────────┐
           │             │             │
      Batch only   Batch + Stream  Stream only
           │             │             │
      Simple ETL   Lambda Arch.   Kappa Arch.
      Medallion     (phức tạp)    (hiện đại)
           │             │             │
           └─────────────┼─────────────┘
                         │
                  Yêu cầu tổ chức
                         │
           ┌─────────────┼─────────────┐
           │             │             │
     Tập trung      Phân tán       Hybrid
     (1 team)    (nhiều domain)   (mixed)
           │             │             │
     Monolithic     Data Mesh     Lake + Mesh
```

---

## 📊 So Sánh Nhanh Các Kiến Trúc

| Tiêu Chí | Lambda | Kappa | Data Mesh | Medallion |
|----------|--------|-------|-----------|-----------|
| **Độ phức tạp** | Cao | Trung bình | Rất cao | Thấp |
| **Latency** | Near-real-time | Real-time | Tùy domain | Batch |
| **Khả năng mở rộng** | Tốt | Rất tốt | Xuất sắc | Tốt |
| **Chi phí vận hành** | Cao (2 pipeline) | Trung bình | Phân tán | Thấp |
| **Phù hợp với AWS** | KDS + EMR/Glue | KDS + Flink | Lake Formation | Glue + S3 |
| **Khi nào dùng** | Batch & stream cùng cần | Chủ yếu stream | Tổ chức lớn, nhiều team | Data lake chuẩn hóa |

---

## 🏛️ Tổng Quan Từng Kiến Trúc

### 1. Lambda Architecture (Kiến Trúc Lambda)

**Khái niệm cốt lõi:** Chia dữ liệu thành hai nhánh xử lý song song — Batch Layer (Lớp Theo Lô) xử lý toàn bộ dữ liệu lịch sử với độ chính xác cao; Speed Layer (Lớp Tốc Độ) xử lý dữ liệu mới nhất với độ trễ thấp. Kết quả từ cả hai được hợp nhất tại Serving Layer (Lớp Phục Vụ).

```
Nguồn dữ liệu ──┬─→ Batch Layer  (S3 + EMR/Glue)     ──┐
                │                                         ├─→ Serving Layer (Redshift/DynamoDB)
                └─→ Speed Layer  (Kinesis + KDA/Flink) ──┘
```

**Điểm mạnh:** Đảm bảo tính chính xác cao cho dữ liệu lịch sử; khả năng reprocessing (xử lý lại) toàn bộ dữ liệu khi cần.

**Điểm yếu:** Phải duy trì hai codebase xử lý logic giống nhau — nguồn gốc của nhiều lỗi và gánh nặng vận hành.

---

### 2. Kappa Architecture (Kiến Trúc Kappa)

**Khái niệm cốt lõi:** Đơn giản hóa Lambda bằng cách loại bỏ hoàn toàn Batch Layer. Mọi dữ liệu đều đi qua một Streaming Layer (Lớp Luồng) duy nhất. Để reprocessing, chỉ cần replay (phát lại) stream từ đầu.

```
Nguồn dữ liệu ──→ Streaming Layer (Kinesis + Flink) ──→ Serving Layer
                                   ↑
                           (replay stream khi cần reprocessing)
```

**Điểm mạnh:** Một codebase duy nhất, dễ maintain hơn Lambda; thiết kế gọn gàng, ít moving parts hơn.

**Điểm yếu:** Streaming framework phải đủ mạnh để xử lý cả historical backfill (bù lấp dữ liệu lịch sử); chi phí lưu trữ stream lâu dài có thể cao.

---

### 3. Data Mesh (Lưới Dữ Liệu)

**Khái niệm cốt lõi:** Phi tập trung hóa ownership (quyền sở hữu) dữ liệu. Mỗi domain (miền nghiệp vụ) như Marketing, Sales, Finance tự quản lý dữ liệu của mình và công bố như Data Products (Sản Phẩm Dữ Liệu) cho các domain khác sử dụng.

```
Domain Marketing ──→ Data Product A ──┐
Domain Sales     ──→ Data Product B ──┼─→ Self-serve Platform (Lake Formation)
Domain Finance   ──→ Data Product C ──┘
```

**4 nguyên tắc nền tảng:**
1. Domain-oriented ownership (Sở hữu hướng domain)
2. Data as a product (Dữ liệu như sản phẩm)
3. Self-serve data infrastructure (Hạ tầng dữ liệu tự phục vụ)
4. Federated computational governance (Quản trị tính toán liên kết)

**Phù hợp AWS:** AWS Lake Formation + Glue Data Catalog + S3 Cross-account

---

### 4. Medallion Architecture (Kiến Trúc Huy Chương)

**Khái niệm cốt lõi:** Tổ chức dữ liệu thành ba lớp chất lượng tăng dần trong data lake — Bronze (Đồng/Thô), Silver (Bạc/Đã làm sạch) và Gold (Vàng/Đã tổng hợp sẵn cho phân tích).

```
Nguồn → Bronze (S3/raw) → Silver (S3/cleaned) → Gold (S3/aggregated) → BI/ML
           Dữ liệu gốc     Đã validate & chuẩn    Đã tổng hợp, sẵn dùng
```

**Điểm mạnh:** Đơn giản, dễ hiểu; dữ liệu gốc luôn được bảo toàn; dễ audit trail (theo dõi kiểm toán).

**Phổ biến nhất** cho data lake trên S3 + Glue + Athena / Redshift Spectrum.

---

## 🔗 Luồng Học Đề Xuất

```
1. Đọc Medallion Architecture trước      → Nền tảng đơn giản nhất
2. Hiểu Lambda Architecture              → Kiến trúc kinh điển, thường gặp trong phỏng vấn
3. So sánh với Kappa Architecture        → Hiểu trade-offs
4. Nghiên cứu Data Mesh cuối cùng        → Phức tạp nhất, cần nền tảng tổ chức
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

1. **"Lambda vs Kappa Architecture — khi nào chọn cái nào?"**
   → Xem [1-lambda-architecture.md](./1-lambda-architecture.md) và [2-kappa-architecture.md](./2-kappa-architecture.md)

2. **"Giải thích Medallion Architecture. Tại sao cần 3 lớp?"**
   → Xem [4-medallion-architecture.md](./4-medallion-architecture.md)

3. **"Data Mesh là gì? Khi nào một tổ chức nên adopt Data Mesh?"**
   → Xem [3-data-mesh.md](./3-data-mesh.md)

4. **"Thiết kế data pipeline cho 1 triệu events/giây với latency < 5 giây"**
   → Kết hợp kiến thức từ Kappa Architecture + Kinesis + Flink

---

## 🔧 AWS Services Mapping (Ánh Xạ Dịch Vụ AWS)

| Thành Phần Kiến Trúc | AWS Service | Ghi Chú |
|---------------------|-------------|---------|
| Batch Layer | EMR + Spark, AWS Glue | Cho Lambda Architecture |
| Speed Layer / Streaming | Kinesis Data Streams, MSK | Real-time ingestion |
| Stream Processing | Kinesis Data Analytics / Flink, AWS Glue Streaming | Xử lý trong luồng |
| Serving Layer | Redshift, DynamoDB, ElastiCache | Phục vụ query nhanh |
| Bronze/Raw Storage | S3 (Standard) | Lưu dữ liệu gốc |
| Silver/Cleaned Storage | S3 + Parquet + Glue Catalog | Có schema, đã validate |
| Gold/Aggregated | Redshift, S3 + Athena Views | Sẵn cho BI/ML |
| Data Product Registry | Glue Data Catalog + Lake Formation | Data Mesh catalog |
| Governance | AWS Lake Formation, IAM | Kiểm soát truy cập |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn thành (5 files)
