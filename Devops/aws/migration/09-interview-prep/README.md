# AWS Migration — Chuẩn Bị Phỏng Vấn: Tổng Quan

> Module này tổng hợp toàn bộ kiến thức AWS Migration & Transfer Services thành bộ tài liệu thực chiến để **vượt qua phỏng vấn kỹ thuật** — từ câu hỏi lý thuyết, bài toán thiết kế hệ thống, đến câu chuyện thực tế dạng STAR (Situation — Task — Action — Result).

## 📚 Mục Lục (Table of Contents)

1. [Tổng Quan Module](#tổng-quan-module)
2. [Cấu Trúc Tài Liệu](#cấu-trúc-tài-liệu)
3. [Phân Loại Câu Hỏi Phỏng Vấn](#phân-loại-câu-hỏi-phỏng-vấn)
4. [Kỹ Năng Cốt Lõi Cần Nắm](#kỹ-năng-cốt-lõi-cần-nắm)
5. [Lộ Trình Ôn Thi Nhanh](#lộ-trình-ôn-thi-nhanh)
6. [Bảng Tra Cứu Nhanh Dịch Vụ](#bảng-tra-cứu-nhanh-dịch-vụ)
7. [Checklist Trước Phỏng Vấn](#checklist-trước-phỏng-vấn)

---

## 🎯 Tổng Quan Module

Module `09-interview-prep` được thiết kế cho ba đối tượng:

```
👤 CLOUD MIGRATION ENGINEER
   → Tập trung: DMS, MGN, DataSync, Snow Family
   → Mục tiêu: Chứng minh khả năng thực thi migration end-to-end

👤 SOLUTIONS ARCHITECT
   → Tập trung: Chiến lược 7Rs, System Design, TCO analysis
   → Mục tiêu: Thiết kế kiến trúc migration toàn diện

👤 DBA / DATA ENGINEER
   → Tập trung: DMS + SCT, CDC, zero-downtime migration
   → Mục tiêu: Chứng minh chuyên môn database migration
```

---

## 📁 Cấu Trúc Tài Liệu

```
09-interview-prep/
├── README.md                   [FILE NÀY] Tổng quan và hướng dẫn sử dụng
├── INTERVIEW_GUIDE.md          Top 20 câu hỏi + câu trả lời mẫu chi tiết
├── star-stories.md             Bộ mẫu câu chuyện thực tế theo phương pháp STAR
└── architecture-scenarios.md   Bài toán system design về migration
```

### Thứ Tự Đọc Được Khuyến Nghị

```
Bước 1 → Đọc README.md này   (10 phút) — Nắm toàn cảnh
Bước 2 → INTERVIEW_GUIDE.md  (2-3 giờ) — Ôn câu hỏi lý thuyết
Bước 3 → architecture-scenarios.md (2 giờ) — Luyện system design
Bước 4 → star-stories.md     (1 giờ)   — Chuẩn bị câu chuyện thực tế
Bước 5 → Mock interview với đồng nghiệp hoặc tự luyện nói to
```

---

## 🗂️ Phân Loại Câu Hỏi Phỏng Vấn

AWS Migration thường xuất hiện trong 4 dạng câu hỏi:

### Dạng 1: Lý Thuyết & Khái Niệm (Concept Questions)

```
Mục đích: Kiểm tra kiến thức nền tảng
Ví dụ:
  - "Giải thích 7R migration strategies là gì?"
  - "Sự khác nhau giữa homogeneous và heterogeneous DMS migration?"
  - "CDC — Change Data Capture hoạt động như thế nào?"

Cách trả lời tốt:
  ✅ Định nghĩa rõ ràng
  ✅ Cho ví dụ cụ thể (Migrate Oracle → Aurora = heterogeneous)
  ✅ Nêu khi nào dùng / khi nào không dùng
```

### Dạng 2: So Sánh Dịch Vụ (Comparison Questions)

```
Mục đích: Kiểm tra khả năng chọn đúng công cụ cho đúng tình huống
Ví dụ:
  - "Khi nào dùng Snow Family thay vì DataSync?"
  - "MGN vs DMS — dùng cái nào cho từng loại workload?"
  - "Transfer Family vs tự dựng SFTP server?"

Cách trả lời tốt:
  ✅ Dùng bảng so sánh (nêu 3-4 tiêu chí)
  ✅ Kết luận bằng "nó phụ thuộc vào..." (trade-off)
  ✅ Đưa ra con số cụ thể (Snowball = 80-210 TB, DataSync = mạng tốt)
```

### Dạng 3: Thiết Kế Hệ Thống (System Design)

```
Mục đích: Kiểm tra khả năng kiến trúc tổng thể
Ví dụ:
  - "Thiết kế migration plan cho 300 servers on-premises lên AWS"
  - "Zero-downtime database migration từ Oracle sang Aurora PostgreSQL"
  - "Thiết kế DR strategy (Disaster Recovery) kết hợp với migration"

Cách trả lời tốt:
  ✅ Bắt đầu bằng câu hỏi làm rõ yêu cầu (clarifying questions)
  ✅ Vẽ sơ đồ kiến trúc nếu có bảng trắng
  ✅ Nêu rõ trade-offs và rủi ro
  ✅ Đề cập rollback plan
```

### Dạng 4: Tình Huống Thực Tế (Behavioral / STAR)

```
Mục đích: Kiểm tra kinh nghiệm thực tế và kỹ năng mềm
Ví dụ:
  - "Kể về một dự án migration bạn đã tham gia"
  - "Làm thế nào bạn xử lý sự cố trong quá trình migration?"
  - "Khi khách hàng phản đối kế hoạch của bạn, bạn làm gì?"

Cách trả lời tốt (phương pháp STAR):
  S — Situation  (Tình huống: bối cảnh, quy mô, áp lực)
  T — Task       (Nhiệm vụ: vai trò của bạn, mục tiêu cụ thể)
  A — Action     (Hành động: những gì BẠN làm, công nghệ sử dụng)
  R — Result     (Kết quả: số liệu đo được, bài học rút ra)
```

---

## 🏆 Kỹ Năng Cốt Lõi Cần Nắm

### Nhóm 1: Chiến Lược (Strategy)

| Kỹ Năng | Mức Độ Quan Trọng | Ghi Chú |
| ------- | ----------------- | ------- |
| 7R Migration Strategies | ⭐⭐⭐ | Luôn được hỏi — phải thuộc lòng |
| Migration Phases (Assess → Mobilize → Migrate) | ⭐⭐⭐ | Framework chuẩn AWS |
| TCO — Total Cost of Ownership (Tổng Chi Phí Sở Hữu) | ⭐⭐ | Phỏng vấn SA thường hỏi |
| Migration Wave Planning (Lập Kế Hoạch Di Chuyển Theo Đợt) | ⭐⭐ | Dự án enterprise |

### Nhóm 2: Di Chuyển Ứng Dụng (Application Migration)

| Kỹ Năng | Mức Độ Quan Trọng | Ghi Chú |
| ------- | ----------------- | ------- |
| AWS MGN — Application Migration Service | ⭐⭐⭐ | Rehost lift-and-shift |
| Cutover Testing (Kiểm Thử Chuyển Đổi) | ⭐⭐⭐ | Best practice phải biết |
| Elastic Disaster Recovery — DRS | ⭐⭐ | Thường so sánh với MGN |

### Nhóm 3: Di Chuyển Cơ Sở Dữ Liệu (Database Migration)

| Kỹ Năng | Mức Độ Quan Trọng | Ghi Chú |
| ------- | ----------------- | ------- |
| AWS DMS — Database Migration Service | ⭐⭐⭐ | Topic HOT nhất |
| Homogeneous vs Heterogeneous Migration | ⭐⭐⭐ | Phân biệt cơ bản |
| CDC — Change Data Capture (Bắt Thay Đổi Dữ Liệu) | ⭐⭐⭐ | Cần cho zero-downtime |
| AWS SCT — Schema Conversion Tool | ⭐⭐⭐ | Gắn với DMS |
| Zero-downtime Migration | ⭐⭐⭐ | Senior-level question |

### Nhóm 4: Truyền Tải Dữ Liệu (Data Transfer)

| Kỹ Năng | Mức Độ Quan Trọng | Ghi Chú |
| ------- | ----------------- | ------- |
| AWS DataSync | ⭐⭐⭐ | Phân biệt vs Snow |
| Snow Family (Snowcone/Snowball/Snowmobile) | ⭐⭐⭐ | Câu hỏi so sánh phổ biến |
| AWS Transfer Family — SFTP/FTPS/FTP/AS2 | ⭐⭐ | B2B file transfer |
| AWS Storage Gateway | ⭐⭐ | Hybrid use case |

---

## ⚡ Lộ Trình Ôn Thi Nhanh

### Còn 1 Ngày Trước Phỏng Vấn

```
Sáng (2 giờ):
  1. Đọc lại 7R strategies — nhớ từng R và ví dụ
  2. Ôn DMS: homogeneous / heterogeneous / CDC
  3. Ôn Snow Family: 3 thiết bị, dung lượng, khi nào dùng

Chiều (2 giờ):
  4. Luyện 5 câu trong INTERVIEW_GUIDE.md (dạng nói to)
  5. Chuẩn bị 2 câu chuyện STAR từ star-stories.md
  6. Ôn 1 bài toán architecture-scenarios.md

Tối (30 phút):
  7. Review checklist → nhận diện điểm yếu → đọc lại nhanh
```

### Còn 1 Tuần Trước Phỏng Vấn

```
Ngày 1-2: INTERVIEW_GUIDE.md — Đọc toàn bộ 20 câu hỏi
Ngày 3:   architecture-scenarios.md — Luyện 3 bài design
Ngày 4:   star-stories.md — Viết lại câu chuyện theo ngữ cảnh của bạn
Ngày 5:   Mock interview — nhờ đồng nghiệp hỏi lại
Ngày 6:   Thực hành AWS lab (DMS + DataSync)
Ngày 7:   Review checklist và ôn điểm yếu
```

---

## 📋 Bảng Tra Cứu Nhanh Dịch Vụ

### Bảng 1: Chọn Công Cụ Theo Kịch Bản

| Kịch Bản | Dịch Vụ Chính | Lưu Ý |
| -------- | ------------- | ------ |
| Di chuyển máy chủ VM/physical lên EC2 | **AWS MGN** | Rehost, không sửa code |
| Di chuyển cùng loại DB (MySQL → RDS MySQL) | **AWS DMS** (homogeneous) | Không cần SCT |
| Di chuyển khác loại DB (Oracle → Aurora) | **AWS DMS + SCT** | SCT chuyển schema trước |
| Đồng bộ dữ liệu liên tục, không downtime | **AWS DMS + CDC** | Full load rồi CDC |
| Copy hàng TB file qua mạng nhanh | **AWS DataSync** | Cần băng thông tốt |
| Di chuyển > 10 TB, băng thông kém | **Snow Family** | Snowball Edge phổ biến nhất |
| Di chuyển > 10 PB hoặc exabyte | **AWS Snowmobile** | Xe tải vật lý của AWS |
| SFTP/FTP server managed | **AWS Transfer Family** | Không cần quản lý server |
| Kết nối on-premises với S3 liên tục | **AWS Storage Gateway** | File/Volume/Tape Gateway |
| Khám phá hạ tầng trước khi migration | **ADS** (Application Discovery Service) | Agentless hoặc agent-based |
| Tính TCO và so sánh chi phí | **Migration Evaluator** | Báo cáo tự động |
| Phục hồi thảm họa từ on-premises | **AWS DRS** (Elastic Disaster Recovery) | Continuous replication |

### Bảng 2: Snow Family So Sánh Nhanh

| Thiết Bị | Dung Lượng | Edge Computing | Use Case Điển Hình |
| -------- | ---------- | -------------- | ------------------ |
| **Snowcone** | 8 TB HDD / 14 TB SSD | EC2 + SageMaker Nano | Vùng xa, quân sự, IoT edge |
| **Snowball Edge Storage** | 80 TB (210 TB cluster) | Không | Di chuyển dữ liệu lớn |
| **Snowball Edge Compute** | 42 TB | EC2 + GPU | Vùng không có mạng, xử lý biên |
| **Snowmobile** | 100 PB / xe | Không | Exabyte migration, data center shutdown |

### Bảng 3: DMS — Nguồn và Đích Hỗ Trợ

| Nguồn (Source) | Đích (Target) | Loại | Cần SCT? |
| -------------- | ------------- | ---- | -------- |
| MySQL | Amazon RDS MySQL / Aurora MySQL | Homogeneous | Không |
| PostgreSQL | Amazon RDS PostgreSQL / Aurora PostgreSQL | Homogeneous | Không |
| Oracle | Amazon Aurora PostgreSQL | Heterogeneous | **Có** |
| Oracle | Amazon RDS PostgreSQL | Heterogeneous | **Có** |
| SQL Server | Amazon RDS PostgreSQL | Heterogeneous | **Có** |
| SQL Server | Amazon RDS MySQL | Heterogeneous | **Có** |
| MongoDB | Amazon DocumentDB | Heterogeneous | Không (DMS tích hợp) |
| S3 | Redshift / DynamoDB | Đặc biệt | Không |

---

## ✅ Checklist Trước Phỏng Vấn

### Kiến Thức Lý Thuyết

- [ ] Có thể giải thích 7Rs không cần nhìn tài liệu (Retire, Retain, Rehost, Relocate, Repurchase, Replatform, Refactor)
- [ ] Phân biệt được MGN vs DMS (application vs database)
- [ ] Biết khi nào dùng Snow Family thay DataSync (quy tắc >1 tuần tải lên)
- [ ] Hiểu CDC — Change Data Capture và tại sao cần cho zero-downtime
- [ ] Biết SCT cần cho heterogeneous, không cần cho homogeneous
- [ ] Nhớ dung lượng Snow Family: Snowcone 8-14 TB, Snowball 80-210 TB, Snowmobile 100 PB

### Kỹ Năng Thiết Kế

- [ ] Có thể vẽ sơ đồ DMS migration pipeline (source → replication instance → target)
- [ ] Có thể thiết kế migration plan 3 giai đoạn (Assess → Mobilize → Migrate)
- [ ] Biết giải thích rollback strategy cho database migration
- [ ] Có thể ước tính thời gian transfer data (bandwidth calculation)

### Câu Chuyện Thực Tế

- [ ] Chuẩn bị ít nhất 1 câu chuyện STAR về migration
- [ ] Câu chuyện có số liệu cụ thể (quy mô, thời gian, kết quả)
- [ ] Có thể kể trong 2-3 phút mà không lan man

### Kỹ Năng Mềm

- [ ] Khi không biết: nói "Tôi không chắc nhưng cách tiếp cận của tôi sẽ là..."
- [ ] Luôn hỏi lại requirements khi gặp bài design (clarifying questions)
- [ ] Nêu trade-offs thay vì đưa ra một giải pháp duy nhất

---

## 🔗 Điều Hướng (Navigation)

| Tôi Cần | File |
| ------- | ---- |
| Top 20 câu hỏi phỏng vấn + câu trả lời mẫu | [INTERVIEW_GUIDE.md](./INTERVIEW_GUIDE.md) |
| Mẫu câu chuyện thực tế STAR | [star-stories.md](./star-stories.md) |
| Bài toán thiết kế hệ thống migration | [architecture-scenarios.md](./architecture-scenarios.md) |
| Lý thuyết chi tiết từng dịch vụ | [../01-fundamentals/](../01-fundamentals/) đến [../08-modernization/](../08-modernization/) |

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn Thành
