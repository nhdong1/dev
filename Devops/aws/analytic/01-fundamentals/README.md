# 01 — Fundamentals: Nền Tảng Analytics

> Module này xây dựng nền tảng vững chắc trước khi đi sâu vào từng dịch vụ AWS Analytics. Hiểu rõ các khái niệm cốt lõi giúp bạn đưa ra quyết định kiến trúc đúng đắn và giải thích trade-off (đánh đổi) trong phỏng vấn.

## 📚 Mục Lục

| #   | File                                                            | Chủ Đề                                                        | Thời Gian Đọc |
| --- | --------------------------------------------------------------- | ------------------------------------------------------------- | ------------- |
| 1   | [1-analytics-overview.md](./1-analytics-overview.md)           | Các loại Analytics: Descriptive → Prescriptive                | 20 phút       |
| 2   | [2-data-pipeline-concepts.md](./2-data-pipeline-concepts.md)   | Data Pipeline: Ingestion → Processing → Storage → Serving     | 25 phút       |
| 3   | [3-batch-vs-streaming.md](./3-batch-vs-streaming.md)           | Batch Processing vs Stream Processing — khi nào chọn cái nào  | 30 phút       |
| 4   | [4-data-lake-vs-warehouse.md](./4-data-lake-vs-warehouse.md)   | Data Lake vs Data Warehouse vs Data Lakehouse                 | 30 phút       |
| 5   | [5-etl-elt-concepts.md](./5-etl-elt-concepts.md)               | ETL vs ELT — định nghĩa, so sánh, quyết định                 | 25 phút       |

**Tổng thời gian:** 2-2.5 giờ đọc + 1-2 giờ thực hành

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành module này, bạn có thể:

- [ ] Giải thích 4 loại analytics (Descriptive, Diagnostic, Predictive, Prescriptive) và ứng dụng thực tế
- [ ] Vẽ sơ đồ Data Pipeline gồm đầy đủ 4 tầng (Ingestion, Processing, Storage, Serving)
- [ ] So sánh Batch Processing và Stream Processing với trade-off cụ thể
- [ ] Phân biệt Data Lake, Data Warehouse và Data Lakehouse — khi nào dùng cái nào
- [ ] Chọn ETL hay ELT dựa trên yêu cầu hệ thống thực tế
- [ ] Liên kết từng khái niệm với dịch vụ AWS tương ứng

---

## 🗺️ Bản Đồ Khái Niệm

```
                    ┌─────────────────────────────────────┐
                    │         ANALYTICS FUNDAMENTALS      │
                    └─────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
   ┌──────▼──────┐           ┌────────▼────────┐         ┌───────▼───────┐
   │  Loại       │           │  Data Pipeline  │         │  Lưu Trữ      │
   │  Analytics  │           │  Concepts       │         │  Dữ Liệu      │
   └──────┬──────┘           └────────┬────────┘         └───────┬───────┘
          │                           │                           │
   Descriptive                  Ingestion                   Data Lake
   Diagnostic                   Processing                  Data Warehouse
   Predictive                   Storage                     Data Lakehouse
   Prescriptive                 Serving
          │                           │                           │
          └───────────────────────────┼───────────────────────────┘
                                      │
                    ┌─────────────────┴──────────────────┐
                    │                                    │
             ┌──────▼──────┐                   ┌────────▼────────┐
             │  Batch vs   │                   │   ETL vs ELT    │
             │  Streaming  │                   │                 │
             └─────────────┘                   └─────────────────┘
```

---

## 🔗 Liên Kết Với AWS Services

| Khái Niệm                   | AWS Service Liên Quan                                     |
| --------------------------- | --------------------------------------------------------- |
| Stream Processing           | Amazon Kinesis, Amazon MSK (Managed Streaming for Kafka)  |
| Batch Processing            | AWS Glue, Amazon EMR (Elastic MapReduce)                  |
| Data Lake                   | Amazon S3 + AWS Lake Formation                            |
| Data Warehouse              | Amazon Redshift                                           |
| ETL / ELT                   | AWS Glue, Amazon Redshift (COPY command)                  |
| Ad-hoc Query                | Amazon Athena                                             |
| BI / Reporting              | Amazon QuickSight                                         |

---

## ⚡ Tóm Tắt Nhanh — Cho Phỏng Vấn

### Câu Hỏi Thường Gặp Và Câu Trả Lời Ngắn

**Q: Data Lake vs Data Warehouse khác nhau thế nào?**
> Data Lake (Hồ Dữ Liệu) lưu dữ liệu thô mọi định dạng (schema-on-read), Data Warehouse (Kho Dữ Liệu) lưu dữ liệu có cấu trúc đã xử lý (schema-on-write). AWS: S3 = Data Lake, Redshift = Data Warehouse.

**Q: Batch hay Streaming — chọn thế nào?**
> Nếu latency (độ trễ) chấp nhận được > vài phút → Batch (Glue, EMR). Nếu cần phản ứng trong giây/phút → Streaming (Kinesis, MSK).

**Q: ETL vs ELT — khác nhau gì?**
> ETL: Transform trước khi Load vào target. ELT: Load raw vào target rồi Transform tại đó. Cloud data warehouses mạnh → ELT ngày càng phổ biến hơn.

**Q: Descriptive Analytics là gì?**
> Mô tả những gì đã xảy ra — ví dụ: dashboard doanh thu tuần qua. Phổ biến nhất, chiếm ~80% use cases analytics.

---

## 🚀 Thứ Tự Học Đề Xuất

```
1. analytics-overview.md     → Hiểu BIG PICTURE trước
2. data-pipeline-concepts.md → Biết dữ liệu đi qua các tầng nào
3. batch-vs-streaming.md     → Quyết định kiến trúc quan trọng nhất
4. data-lake-vs-warehouse.md → Lựa chọn storage strategy
5. etl-elt-concepts.md       → Pattern xử lý dữ liệu
```

---

## 📖 Câu Hỏi Ôn Tập

Sau khi đọc xong từng file, tự kiểm tra:

1. Công ty e-commerce muốn biết "Tại sao doanh số tháng 3 giảm?" — đây là loại analytics gì?
2. Vẽ data pipeline đơn giản cho hệ thống log website (100K events/giây).
3. Twitter cần xử lý tweets real-time để detect trending topics — Batch hay Streaming?
4. Startup fintech có $10K/tháng budget analytics — Data Lake hay Data Warehouse?
5. Team data nhỏ (2 người) dùng Snowflake — nên dùng ETL hay ELT?

*(Đáp án nằm trong từng file chi tiết)*

---

**Tiếp Theo:** [1-analytics-overview.md](./1-analytics-overview.md) →
