# 🎯 Chuẩn Bị Phỏng Vấn AWS Storage

> Tài liệu tổng hợp giúp bạn tự tin trả lời mọi câu hỏi về AWS Storage trong phỏng vấn kỹ thuật — từ câu hỏi cơ bản đến bài toán thiết kế hệ thống phức tạp.

---

## 📚 Nội Dung Thư Mục Này

| File | Mô Tả | Thời Gian Học |
|------|--------|---------------|
| [1-INTERVIEW_GUIDE.md](1-INTERVIEW_GUIDE.md) | Top 20 câu hỏi phỏng vấn AWS storage — Q&A đầy đủ | 3–4 giờ |
| [2-system-design-scenarios.md](2-system-design-scenarios.md) | Tình huống thiết kế: data lake, backup, DR, media streaming | 4–5 giờ |
| [3-star-stories.md](3-star-stories.md) | Template câu chuyện STAR — storage incidents thực tế | 2–3 giờ |
| [4-cost-optimization-questions.md](4-cost-optimization-questions.md) | Câu hỏi & câu trả lời về tối ưu chi phí lưu trữ | 2–3 giờ |
| [5-90-day-study-plan.md](5-90-day-study-plan.md) | Kế hoạch học 90 ngày có cấu trúc — từ nền tảng đến phỏng vấn | 1 giờ |

---

## 🧠 Tại Sao AWS Storage Quan Trọng Trong Phỏng Vấn?

AWS Storage xuất hiện trong **hầu hết** phỏng vấn backend/cloud/DevOps vì:

1. **Mọi hệ thống đều cần lưu trữ** — không có ứng dụng nào không dùng storage
2. **Trade-offs rõ ràng** — S3 vs EBS vs EFS có sự khác biệt cụ thể, dễ kiểm tra
3. **Liên quan trực tiếp đến chi phí** — interviewer muốn biết bạn có ý thức về cost
4. **Bảo mật** — S3 misconfiguration là một trong những lỗi phổ biến nhất trên cloud
5. **System design** — data lake, backup, DR đều cần storage architecture

---

## 🎯 Trọng Tâm Theo Vị Trí

### Backend Engineer / Software Engineer

Tập trung vào:
- S3 presigned URLs — URL có chữ ký tạm thời (tạo, dùng, bảo mật)
- Multipart upload — tải lên nhiều phần cho file lớn
- S3 Event Notifications — thông báo sự kiện (trigger Lambda)
- Storage class selection — chọn lớp lưu trữ phù hợp

### Cloud / DevOps Engineer

Tập trung vào:
- EBS volume types và performance tuning — điều chỉnh hiệu suất
- EFS vs EBS — khi nào dùng shared storage
- Lifecycle policies — chính sách vòng đời tự động
- Monitoring với CloudWatch — giám sát và cảnh báo

### Solutions Architect

Tập trung vào:
- Data lake architecture — kiến trúc data lake trên S3
- Disaster Recovery — RPO/RTO với S3 CRR và EBS snapshots
- Storage Gateway cho hybrid cloud — đám mây lai
- Cost optimization at scale — tối ưu chi phí quy mô lớn

### Site Reliability Engineer (SRE)

Tập trung vào:
- Monitoring & alerting cho storage — giám sát và cảnh báo
- Incident response — xử lý sự cố (S3 public bucket, EBS full)
- Backup strategy — chiến lược sao lưu và kiểm tra restore
- Performance bottleneck analysis — phân tích tắc nghẽn hiệu suất

---

## 📋 Danh Sách Kiểm Tra Trước Phỏng Vấn

### Kiến Thức Bắt Buộc (Mọi Vị Trí)

- [ ] Phân biệt S3, EBS, EFS — không cần nhìn tài liệu
- [ ] 6 loại S3 storage class và khi nào dùng từng loại
- [ ] Cách mã hóa dữ liệu trong S3 (SSE-S3, SSE-KMS, SSE-C, CSE)
- [ ] IAM — Identity and Access Management: policy vs bucket policy vs ACL
- [ ] Presigned URL — URL có chữ ký: tạo và use case
- [ ] EBS volume types: gp3 vs io2 vs st1 — trade-offs

### Kiến Thức Nâng Cao (Senior/Architect)

- [ ] Thiết kế data lake 3-layer trên S3
- [ ] CRR — Cross-Region Replication — sao chép liên vùng cho DR
- [ ] S3 Object Lock — khóa đối tượng: Governance vs Compliance mode
- [ ] EBS Multi-Attach — gắn kết nhiều EC2
- [ ] Storage Gateway: File vs Volume vs Tape
- [ ] Snow Family: Snowcone vs Snowball Edge vs Snowmobile
- [ ] FSx variants: Windows vs Lustre vs NetApp ONTAP vs OpenZFS

### Câu Chuyện STAR (Behavioral Questions)

- [ ] Lần giải quyết incident liên quan đến storage
- [ ] Lần tối ưu chi phí S3 hoặc EBS thành công
- [ ] Lần thiết kế hoặc migrate storage architecture
- [ ] Lần xử lý vấn đề bảo mật (public bucket, data leak)

---

## ⏱️ Lịch Ôn Tập Nhanh (3 Ngày Trước Phỏng Vấn)

### Ngày 1 — Nền Tảng & Bảo Mật

```
Sáng (2h): Đọc 1-INTERVIEW_GUIDE.md — phần S3 và bảo mật
Chiều (2h): Thực hành: tạo bucket, set policy, test encryption
Tối  (1h): Ôn lại: viết tay các điểm chính không nhìn tài liệu
```

### Ngày 2 — System Design & Cost

```
Sáng (2h): Đọc 2-system-design-scenarios.md
Chiều (2h): Đọc 4-cost-optimization-questions.md, tính toán ví dụ
Tối  (1h): Luyện nói: giải thích kiến trúc cho "người không biết"
```

### Ngày 3 — Behavioral & Mock Interview

```
Sáng (2h): Chuẩn bị 3-4 câu chuyện STAR từ 3-star-stories.md
Chiều (2h): Mock interview với đồng nghiệp hoặc tự record
Tối  (1h): Review các điểm yếu, ôn lại lần cuối
```

---

## 🏆 Tiêu Chí Đánh Giá Câu Trả Lời

Interviewer thường chấm theo thang này:

| Mức | Mô Tả | Ví Dụ |
|-----|--------|-------|
| **Biết** | Định nghĩa đúng, hiểu surface level | "S3 là object storage" |
| **Hiểu** | Giải thích được trade-offs | "S3 dùng cho static assets vì..." |
| **Áp dụng** | Chọn đúng cho use case cụ thể | "Với 10TB log, dùng S3-IA vì..." |
| **Phân tích** | So sánh options, justify quyết định | "So với EFS, S3 rẻ hơn vì... nhưng không mount được nên..." |
| **Sáng tạo** | Thiết kế kiến trúc mới, optimize | "Để giảm 40% cost, tôi sẽ thiết kế lifecycle như sau..." |

**Mục tiêu của bạn:** Đạt mức Phân tích cho S3/EBS/EFS, mức Áp dụng cho các dịch vụ khác.

---

## 🔗 Điều Hướng

- [← 11-fsx/](../11-fsx/) — FSx Advanced File Systems
- [README.md tổng quan](../README.md) — Lộ trình học AWS Storage
- [INDEX.md](../INDEX.md) — Chỉ mục đầy đủ

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
