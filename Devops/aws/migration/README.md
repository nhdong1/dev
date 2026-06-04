# ☁️ AWS Migration & Transfer Services — Roadmap

> Hướng dẫn toàn diện về AWS Migration & Transfer Services, bao gồm chiến lược di chuyển, công cụ, thực hành tốt nhất và chuẩn bị phỏng vấn.

## 📚 Mục Lục (Table of Contents)

1. [Lộ Trình Học Tập](#lộ-trình-học-tập)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Dịch Vụ](#tổng-quan-dịch-vụ)
4. [Tổng Quan Chủ Đề](#tổng-quan-chủ-đề)

---

## 🎯 Lộ Trình Học Tập (Learning Path)

### **Phase 1: Nền Tảng (Weeks 1-2)**

- [ ] Chiến lược di chuyển 7Rs (7R Migration Strategies)
- [ ] Các giai đoạn di chuyển lên AWS (Assess → Mobilize → Migrate & Modernize)
- [ ] Tổng quan hệ sinh thái di chuyển AWS
- [ ] AWS Migration Hub — Trung tâm theo dõi tiến trình

### **Phase 2: Di Chuyển Ứng Dụng & Cơ Sở Dữ Liệu (Weeks 3-5)**

- [ ] AWS Application Migration Service — MGN (Rehost tự động)
- [ ] AWS Database Migration Service — DMS (Di chuyển CSDL)
- [ ] AWS Schema Conversion Tool — SCT (Chuyển đổi schema)
- [ ] AWS Application Discovery Service (Khám phá hạ tầng)

### **Phase 3: Truyền Tải Dữ Liệu (Weeks 6-8)**

- [ ] AWS DataSync — Đồng bộ dữ liệu qua mạng
- [ ] AWS Transfer Family — SFTP/FTP/FTPS trên S3/EFS
- [ ] AWS Snow Family — Di chuyển dữ liệu ngoại tuyến (offline)
- [ ] AWS Storage Gateway — Kết nối on-premises với S3

### **Phase 4: Chuyên Sâu & Hiện Đại Hóa (Weeks 9+)**

- [ ] AWS Mainframe Modernization (Hiện đại hóa Mainframe)
- [ ] AWS Migration Evaluator (Đánh giá TCO trước khi di chuyển)
- [ ] Chiến lược di chuyển lai (Hybrid Migration)
- [ ] Tối ưu hóa chi phí sau di chuyển (Post-Migration Optimization)

---

## 🏢 Năng Lực Cốt Lõi (Core Competencies)

| Năng Lực                                 | Độ Ưu Tiên | Thời Gian | Trạng Thái |
| ---------------------------------------- | ---------- | --------- | ---------- |
| **Chiến lược 7Rs**                       | ⭐⭐⭐    | 1 tuần    | -          |
| **AWS DMS** (Database Migration Service) | ⭐⭐⭐    | 2 tuần    | -          |
| **AWS MGN** (Application Migration)      | ⭐⭐⭐    | 2 tuần    | -          |
| **AWS Snow Family**                      | ⭐⭐⭐    | 1 tuần    | -          |
| **AWS DataSync**                         | ⭐⭐⭐    | 1 tuần    | -          |
| **AWS Transfer Family**                  | ⭐⭐      | 1 tuần    | -          |
| **AWS Migration Hub**                    | ⭐⭐      | 1 tuần    | -          |
| **AWS Schema Conversion Tool**           | ⭐⭐      | 1 tuần    | -          |
| **Mainframe Modernization**              | ⭐        | 1 tuần    | -          |
| **Migration Evaluator**                  | ⭐        | 0.5 tuần  | -          |

---

## 🗺️ Tổng Quan Dịch Vụ (Services Overview)

### Nhóm 1: Lập Kế Hoạch & Đánh Giá (Plan & Assess)

| Dịch Vụ                          | Mô Tả Ngắn                                          |
| --------------------------------- | --------------------------------------------------- |
| **AWS Migration Hub**             | Trung tâm theo dõi tiến trình di chuyển tập trung  |
| **Application Discovery Service** | Tự động khám phá máy chủ và ứng dụng on-premises   |
| **Migration Evaluator**           | Phân tích TCO, đề xuất phương án tối ưu chi phí    |

### Nhóm 2: Di Chuyển Ứng Dụng (Application Migration)

| Dịch Vụ                                             | Mô Tả Ngắn                                             |
| ---------------------------------------------------- | ------------------------------------------------------ |
| **AWS MGN** (Application Migration Service)          | Rehost máy chủ vật lý/ảo lên EC2 tự động (lift-shift) |
| **AWS Elastic Disaster Recovery** (DRS)              | Phục hồi thảm họa liên tục từ on-premises lên AWS     |

### Nhóm 3: Di Chuyển Cơ Sở Dữ Liệu (Database Migration)

| Dịch Vụ                                       | Mô Tả Ngắn                                                |
| ---------------------------------------------- | --------------------------------------------------------- |
| **AWS DMS** (Database Migration Service)        | Di chuyển CSDL đồng nhất hoặc dị cấu trúc (homogeneous/heterogeneous) |
| **AWS SCT** (Schema Conversion Tool)            | Tự động chuyển đổi schema từ DB engine này sang engine khác |

### Nhóm 4: Truyền Tải Dữ Liệu (Data Transfer)

| Dịch Vụ                    | Mô Tả Ngắn                                                       |
| --------------------------- | ----------------------------------------------------------------- |
| **AWS DataSync**            | Đồng bộ dữ liệu qua mạng giữa on-premises và AWS nhanh, tự động |
| **AWS Transfer Family**     | Managed SFTP/FTPS/FTP/AS2 đến S3 hoặc EFS                       |
| **AWS Storage Gateway**     | Kết nối storage on-premises với S3, EBS, hoặc Tape               |

### Nhóm 5: Di Chuyển Ngoại Tuyến (Offline/Physical Transfer)

| Dịch Vụ               | Dung Lượng     | Mô Tả Ngắn                                       |
| ---------------------- | -------------- | ------------------------------------------------- |
| **AWS Snowcone**       | 8 TB HDD / 14 TB SSD | Thiết bị nhỏ gọn, dùng ở vùng xa xôi       |
| **AWS Snowball Edge**  | 80-210 TB      | Di chuyển dữ liệu lớn, có khả năng tính toán cạnh |
| **AWS Snowmobile**     | 100 PB         | Xe tải dữ liệu cho exabyte-scale migration        |

### Nhóm 6: Hiện Đại Hóa (Modernization)

| Dịch Vụ                              | Mô Tả Ngắn                                                     |
| ------------------------------------- | --------------------------------------------------------------- |
| **AWS Mainframe Modernization**       | Chuyển đổi ứng dụng mainframe sang microservices trên AWS       |

---

## 🗂️ Tổng Quan Chủ Đề (Topics Overview)

### 📁 **1. Nền Tảng** (`01-fundamentals/`)

- Chiến lược 7Rs: Retire, Retain, Rehost, Relocate, Repurchase, Replatform, Refactor
- Ba giai đoạn AWS Migration: Assess (Đánh giá), Mobilize (Chuẩn bị), Migrate & Modernize
- AWS Migration Ecosystem — Tổng quan hệ sinh thái
- TCO — Total Cost of Ownership (Tổng chi phí sở hữu)
- Migration Wave Planning (Lập kế hoạch di chuyển theo đợt)

### 📁 **2. Migration Hub & Discovery** (`02-migration-hub/`)

- AWS Migration Hub — Theo dõi tiến trình tập trung
- AWS Application Discovery Service — Agentless vs Agent-based
- Migration Portfolio Assessment (Đánh giá danh mục di chuyển)
- Tích hợp với AWS Partner solutions

### 📁 **3. Di Chuyển Ứng Dụng** (`03-application-migration/`)

- AWS MGN — Application Migration Service (thay thế SMS)
- Replication Agent — Cách thức hoạt động
- Cutover Testing và Launch Templates
- Elastic Disaster Recovery (DRS)

### 📁 **4. Di Chuyển Cơ Sở Dữ Liệu** (`04-database-migration/`)

- AWS DMS — Homogeneous vs Heterogeneous Migration
- Replication Instance, Endpoints, Tasks
- CDC — Change Data Capture (Capture thay đổi dữ liệu liên tục)
- AWS SCT — Schema Conversion Tool
- Các source/target database được hỗ trợ

### 📁 **5. Truyền Tải Dữ Liệu Qua Mạng** (`05-data-transfer/`)

- AWS DataSync — Agents, Locations, Tasks
- Transfer Family — SFTP/FTPS/FTP/AS2
- AWS Storage Gateway — File, Volume, Tape
- Direct Connect vs VPN cho migration

### 📁 **6. Snow Family — Di Chuyển Ngoại Tuyến** (`06-snow-family/`)

- Snowcone — Use cases, OpsHub, DataSync tích hợp
- Snowball Edge — Storage Optimized vs Compute Optimized
- Snowmobile — Exabyte-scale migration
- Bảo mật: mã hóa 256-bit AES, Trusted Platform Module

### 📁 **7. Khám Phá & Đánh Giá** (`07-discovery-assessment/`)

- Application Discovery Service (ADS)
- Migration Evaluator (TSO Logic)
- AWS Well-Architected Migration Lens
- Right-sizing recommendations

### 📁 **8. Hiện Đại Hóa** (`08-modernization/`)

- AWS Mainframe Modernization — Refactor và Replatform
- AWS App2Container — Container hóa ứng dụng
- AWS End-of-Support Migration Program
- Strangler Fig Pattern (Tách dần legacy system)

### 📁 **9. Chuẩn Bị Phỏng Vấn** (`09-interview-prep/`)

- Top 20 câu hỏi phỏng vấn về AWS Migration
- System Design: Migration Scenarios
- STAR stories — Kể chuyện thực tế
- Kiến trúc mẫu (Architecture Patterns)

---

## 🎓 Phân Loại Theo Trường Hợp Sử Dụng

### **Lift-and-Shift (Rehost — Nâng Và Chuyển)**

```
Dịch vụ chính: AWS MGN (Application Migration Service)
Phù hợp: Di chuyển nhanh, ít thay đổi code
Thời gian: Nhanh nhất, rủi ro thấp
Chi phí: Không tối ưu ngay, cần refactor sau
Covered in: 03-application-migration/
```

### **Di Chuyển Cơ Sở Dữ Liệu**

```
Dịch vụ chính: AWS DMS + SCT
Phù hợp: Chuyển đổi từ Oracle/SQL Server sang Aurora/PostgreSQL
Thách thức: Schema khác nhau, stored procedures, triggers
Covered in: 04-database-migration/
```

### **Di Chuyển Dữ Liệu Lớn Qua Mạng**

```
Dịch vụ chính: AWS DataSync
Phù hợp: Hàng TB dữ liệu có băng thông tốt
Lợi thế: Kiểm tra toàn vẹn dữ liệu tự động, mã hóa
Covered in: 05-data-transfer/
```

### **Di Chuyển Ngoại Tuyến (Không Có Băng Thông)**

```
Dịch vụ: Snow Family (Snowcone/Snowball/Snowmobile)
Phù hợp: > 10 TB, băng thông hạn chế, vùng xa
Quy tắc: Nếu tải lên >1 tuần → dùng Snow
Covered in: 06-snow-family/
```

### **SFTP/FTP Server Managed**

```
Dịch vụ chính: AWS Transfer Family
Phù hợp: B2B file exchange, thay thế SFTP servers
Lợi thế: Không cần quản lý hạ tầng
Covered in: 05-data-transfer/
```

---

## 🔗 Liên Kết Nhanh (Quick Links)

| Chủ Đề                       | Thư Mục                                            | Độ Ưu Tiên     |
| ----------------------------- | -------------------------------------------------- | -------------- |
| Chiến lược 7Rs                | [01-fundamentals/](./01-fundamentals/)             | Bắt đầu tại đây |
| AWS DMS (Database Migration)  | [04-database-migration/](./04-database-migration/) | Thiết yếu       |
| AWS MGN (App Migration)       | [03-application-migration/](./03-application-migration/) | Thiết yếu |
| Snow Family                   | [06-snow-family/](./06-snow-family/)               | Quan trọng      |
| AWS DataSync                  | [05-data-transfer/](./05-data-transfer/)           | Quan trọng      |
| Câu hỏi phỏng vấn             | [09-interview-prep/](./09-interview-prep/)         | Trước phỏng vấn |

---

## 📊 Ma Trận Kỹ Năng (Skill Matrix)

### Người Mới Bắt Đầu (0-1 năm)

- [ ] Hiểu chiến lược 7Rs và biết chọn chiến lược phù hợp
- [ ] Phân biệt được AWS MGN, DMS, DataSync, Snow Family
- [ ] Biết khi nào dùng Snow thay vì network transfer
- [ ] Hiểu khái niệm CDC (Change Data Capture)

### Trung Cấp (1-3 năm)

- [ ] Thiết kế migration plan với DMS + SCT cho heterogeneous DB
- [ ] Cấu hình DataSync tasks, bandwidth throttling
- [ ] Lên kế hoạch cutover với AWS MGN (test, cutover windows)
- [ ] Tính toán và tối ưu chi phí migration với Migration Evaluator
- [ ] Thiết kế hybrid connectivity (Direct Connect/VPN) cho migration

### Nâng Cao (3+ năm)

- [ ] Thiết kế Migration Wave với nhiều ứng dụng phức tạp
- [ ] Xây dựng chiến lược zero-downtime DB migration
- [ ] Kiến trúc Mainframe Modernization
- [ ] Tích hợp migration vào CI/CD pipeline
- [ ] Post-migration optimization và cost governance

---

## 🚀 Bắt Đầu (Getting Started)

### Bước 1: Xác Định Mục Tiêu Học

```
Chọn hướng học:
- Cloud Architect (tập trung thiết kế và chiến lược)
- Migration Engineer (tập trung thực hành công cụ)
- DBA/Data Engineer (tập trung DMS, SCT, DataSync)
```

### Bước 2: Dựng Môi Trường Lab

```bash
# Tạo tài khoản AWS Free Tier
# Dựng VPC on-premises giả lập bằng EC2

# Thực hành DMS:
# 1. Tạo RDS MySQL (source)
# 2. Tạo RDS PostgreSQL (target)
# 3. Chạy DMS migration task

# Thực hành DataSync:
# 1. Cài DataSync agent trên EC2
# 2. Tạo location NFS hoặc SMB
# 3. Tạo task sync đến S3
```

### Bước 3: Học Theo Trình Tự

```
1. Đọc module 01-fundamentals/ (chiến lược 7Rs)   → 2 giờ
2. Thực hành AWS MGN trong lab                      → 3 giờ
3. Thực hành DMS + SCT                              → 4 giờ
4. Tìm hiểu Snow Family use cases                   → 2 giờ
5. Thực hành DataSync và Transfer Family             → 2 giờ
6. Ôn tập câu hỏi phỏng vấn                         → 2 giờ
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation  (Tình huống)
- Task        (Nhiệm vụ)
- Action      (Hành động)
- Result      (Kết quả)
```

---

## 📖 Tài Liệu Tham Khảo (Reference Materials)

### Tài Liệu Chính Thức AWS

- [AWS Migration & Transfer](https://aws.amazon.com/migration/)
- [AWS DMS Documentation](https://docs.aws.amazon.com/dms/)
- [AWS MGN Documentation](https://docs.aws.amazon.com/mgn/)
- [AWS DataSync Documentation](https://docs.aws.amazon.com/datasync/)
- [AWS Snow Family](https://aws.amazon.com/snow/)
- [AWS Transfer Family](https://docs.aws.amazon.com/transfer/)

### Sách & Khóa Học Được Khuyến Nghị

- **AWS Certified Solutions Architect** — Professional (migration scenarios)
- **AWS Migration Competency** — Partner training
- **"Cloud Migration: The Complete Guide"** — Sách tổng quát
- **AWS re:Invent Migration Talks** — Video thực tế

### Blog & Tài Nguyên Thực Tế

- AWS Migration Blog (aws.amazon.com/blogs/migration-and-modernization/)
- AWS Well-Architected Migration Lens
- AWS Prescriptive Guidance (aws.amazon.com/prescriptive-guidance/)

---

## 🎯 Chuẩn Bị Phỏng Vấn (Interview Preparation)

### Câu Hỏi Thường Gặp Theo Chủ Đề

#### Chiến Lược Di Chuyển

- [ ] Giải thích 7R migration strategies và cho ví dụ từng loại
- [ ] Khi nào dùng Rehost vs Replatform vs Refactor?
- [ ] Thiết kế migration plan cho 500 máy chủ on-premises

#### Di Chuyển Cơ Sở Dữ Liệu

- [ ] Khác biệt giữa homogeneous và heterogeneous DMS migration?
- [ ] Giải thích CDC (Change Data Capture) và tại sao cần thiết
- [ ] Chiến lược zero-downtime database migration là gì?

#### Truyền Tải Dữ Liệu

- [ ] Khi nào dùng DataSync thay vì Snow Family?
- [ ] Transfer Family khác gì so với tự dựng SFTP server?
- [ ] Tính toán thời gian transfer 50 TB qua đường 1 Gbps

#### Thực Tế / Incident

- [ ] Kể về một dự án migration bạn đã thực hiện (STAR)
- [ ] Làm thế nào xử lý rollback nếu migration thất bại?
- [ ] Kiểm tra tính toàn vẹn dữ liệu sau migration như thế nào?

Xem `09-interview-prep/` để có bộ Q&A đầy đủ.

---

## ✅ Checklist Tự Đánh Giá (Self-Assessment Checklist)

Trước phỏng vấn hoặc dự án mới, kiểm tra:

- [ ] Có thể giải thích 7Rs mà không cần nhìn tài liệu
- [ ] Biết chọn đúng dịch vụ cho từng loại migration scenario
- [ ] Hiểu DMS: source, replication instance, target, tasks
- [ ] Hiểu Snow Family: biết chọn đúng thiết bị theo data size
- [ ] Có thể thiết kế migration architecture cho một workload cụ thể
- [ ] Biết cách kiểm tra dữ liệu sau migration (data validation)
- [ ] Có thể tính TCO và so sánh on-premises vs AWS
- [ ] Hiểu các rủi ro migration và cách giảm thiểu

---

## 📞 Hỗ Trợ & Tài Nguyên (Support & Resources)

### Công Cụ Thực Hành

- **AWS Migration Hub** — Theo dõi tập trung miễn phí
- **AWS DMS Free Tier** — t3.micro replication instance (6 tháng đầu)
- **AWS Schema Conversion Tool** — Tải miễn phí
- **AWS DataSync** — Thực hành với S3

### Cộng Đồng

- AWS re:Post (repost.aws) — Diễn đàn kỹ thuật chính thức
- r/aws (Reddit)
- AWS Discord Community
- AWS User Groups (Nhóm người dùng địa phương)

---

## 📋 Cách Sử Dụng Tài Liệu Này

### Cho Tự Học

1. Bắt đầu với [Lộ Trình Học Tập](#lộ-trình-học-tập)
2. Đi qua từng phase theo thứ tự
3. Làm bài tập thực hành trong lab
4. Xây dựng portfolio project

### Cho Chuẩn Bị Phỏng Vấn

1. Tập trung vào [09-interview-prep](./09-interview-prep/)
2. Ôn kỹ 7Rs và use cases của từng dịch vụ
3. Chuẩn bị câu chuyện thực tế (STAR method)
4. Luyện tập giải thích rõ ràng cho người không chuyên

### Cho Dự Án Thực Tế

1. Bắt đầu với [01-fundamentals](./01-fundamentals/) để chọn chiến lược
2. Dùng [02-migration-hub](./02-migration-hub/) để theo dõi tiến trình
3. Tham khảo module phù hợp với workload của bạn
4. Verify với AWS Well-Architected Migration Lens

---

## 🗺️ Các Bước Tiếp Theo (Next Steps)

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học (Beginner/Intermediate/Advanced)
├─ 3️⃣  Bắt đầu với 01-fundamentals/ (7R strategies)
├─ 4️⃣  Dựng môi trường lab AWS Free Tier
├─ 5️⃣  Thực hành DMS và MGN
├─ 6️⃣  Xây dựng portfolio migration project
└─ 7️⃣  Chuẩn bị phỏng vấn qua 09-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
