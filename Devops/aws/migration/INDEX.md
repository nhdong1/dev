# AWS Migration & Transfer Services — Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về AWS Migration & Transfer Services — từ chiến lược, công cụ đến thực hành

## 📁 Cấu Trúc Thư Mục (Folder Structure)

```
migration/
├── README.md                                   [BẮT ĐẦU TẠI ĐÂY] Lộ trình & tổng quan
├── INDEX.md                                    [FILE NÀY] Chỉ mục & trạng thái tạo file
│
├── 01-fundamentals/
│   ├── README.md                               ✅ Nền tảng: 7Rs, migration phases, TCO
│   ├── 1-7r-strategies.md                      ✅ 7R strategies chi tiết
│   ├── 2-migration-phases.md                   ✅ Assess → Mobilize → Migrate & Modernize
│   ├── 3-aws-migration-ecosystem.md            ✅ Tổng quan hệ sinh thái
│   └── 4-tco-cost-analysis.md                  ✅ TCO và so sánh chi phí
│
├── 02-migration-hub/
│   ├── README.md                               ✅ Migration Hub & Discovery tổng quan
│   ├── 1-migration-hub-overview.md             ✅ Theo dõi tiến trình tập trung
│   ├── 2-application-discovery-service.md      ✅ ADS: agentless vs agent-based
│   └── 3-migration-evaluator.md                ✅ Phân tích TCO, đề xuất phương án
│
├── 03-application-migration/
│   ├── README.md                               ✅ Di chuyển ứng dụng tổng quan (MGN vs DRS)
│   ├── 1-mgn-overview.md                       ✅ AWS MGN: replication agent, launch templates
│   ├── 2-mgn-cutover.md                        ✅ Test cutover, cutover window, rollback
│   └── 3-elastic-disaster-recovery.md          ✅ DRS: continuous replication, failover, failback
│
├── 04-database-migration/
│   ├── README.md                               ✅ Di chuyển CSDL tổng quan (DMS + SCT)
│   ├── 1-dms-fundamentals.md                   ✅ DMS: replication instance, endpoints, tasks
│   ├── 2-dms-homogeneous.md                    ✅ Di chuyển cùng loại DB (MySQL → MySQL)
│   ├── 3-dms-heterogeneous.md                  ✅ Di chuyển khác loại DB (Oracle → Aurora)
│   ├── 4-schema-conversion-tool.md             ✅ SCT: chuyển đổi schema và code tự động
│   └── 5-dms-cdc.md                            ✅ CDC — Change Data Capture liên tục
│
├── 05-data-transfer/
│   ├── README.md                               ✅ Truyền tải dữ liệu tổng quan
│   ├── 1-datasync-overview.md                  ✅ DataSync: agents, locations, tasks, scheduling
│   ├── 2-datasync-advanced.md                  ✅ Bandwidth throttling, filtering, verification
│   ├── 3-transfer-family.md                    ✅ SFTP/FTPS/FTP/AS2 managed trên S3/EFS
│   └── 4-storage-gateway.md                    ✅ File, Volume, Tape Gateway
│
├── 06-snow-family/
│   ├── README.md                               ✅ Snow Family tổng quan & so sánh
│   ├── 1-snowcone.md                           ✅ Snowcone: 8-14 TB, edge computing nhỏ gọn
│   ├── 2-snowball-edge.md                      ✅ Snowball Edge: Storage vs Compute Optimized
│   └── 3-snowmobile.md                         ✅ Snowmobile: 100 PB, exabyte migration
│
├── 07-discovery-assessment/
│   ├── README.md                               ✅ Khám phá và đánh giá tổng quan
│   ├── 1-application-discovery-service.md      ✅ ADS agentless collector và discovery agent
│   ├── 2-migration-evaluator.md                ✅ Báo cáo TCO và right-sizing
│   └── 3-well-architected-migration.md         ✅ Migration Lens best practices
│
├── 08-modernization/
│   ├── README.md                               ✅ Hiện đại hóa ứng dụng tổng quan
│   ├── 1-mainframe-modernization.md            ✅ Refactor và replatform mainframe lên AWS
│   ├── 2-app2container.md                      ✅ Container hóa Java/.NET apps tự động
│   └── 3-end-of-support-migration.md           ✅ Windows Server / SQL Server EOS migration
│
└── 09-interview-prep/
    ├── README.md                               ✅ Tổng quan chuẩn bị phỏng vấn & checklist
    ├── INTERVIEW_GUIDE.md                      ✅ Top 20 câu hỏi và câu trả lời mẫu chi tiết
    ├── star-stories.md                         ✅ 5 mẫu câu chuyện thực tế theo STAR method
    └── architecture-scenarios.md               ✅ 5 bài toán thiết kế kiến trúc migration
```

---

## ✅ Trạng Thái Tạo File (Creation Status)

| Chủ Đề                                  | File                    | Trạng Thái | Chất Lượng    |
| --------------------------------------- | ----------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**               | README.md               | ✅          | Toàn diện     |
| **Chỉ Mục Đầy Đủ**                     | INDEX.md                | ✅          | Toàn diện     |
| **Nền Tảng (7Rs, Phases)**              | 01-fundamentals/        | ✅ Hoàn thành | Toàn diện     |
| **Migration Hub & Discovery**           | 02-migration-hub/       | ✅ Hoàn thành | Toàn diện     |
| **Di Chuyển Ứng Dụng (MGN)**           | 03-application-migration/ | ✅ Hoàn thành | Toàn diện   |
| **Di Chuyển Cơ Sở Dữ Liệu (DMS/SCT)** | 04-database-migration/  | ✅ Hoàn thành | Toàn diện     |
| **Truyền Tải Dữ Liệu (DataSync, SFTP)**| 05-data-transfer/       | ✅ Hoàn thành | Toàn diện     |
| **Snow Family**                         | 06-snow-family/         | ✅ Hoàn thành | Toàn diện     |
| **Khám Phá & Đánh Giá**                | 07-discovery-assessment/| ✅ Hoàn thành | Toàn diện     |
| **Hiện Đại Hóa**                       | 08-modernization/       | ✅ Hoàn thành | Toàn diện     |
| **Chuẩn Bị Phỏng Vấn**                 | 09-interview-prep/      | ✅ Hoàn thành | Toàn diện     |

---

## 🎯 Thứ Tự Ưu Tiên Tạo Nội Dung (Priority Order)

### Ưu Tiên Cao (Core Migration Skills)

- [x] `01-fundamentals/README.md` — Chiến lược 7Rs, migration phases
- [x] `01-fundamentals/1-7r-strategies.md` — Chi tiết từng R với ví dụ thực tế
- [x] `04-database-migration/README.md` — DMS tổng quan
- [x] `04-database-migration/1-dms-fundamentals.md` — DMS chi tiết
- [x] `04-database-migration/4-schema-conversion-tool.md` — SCT chi tiết
- [x] `03-application-migration/1-mgn-overview.md` — MGN chi tiết
- [x] `06-snow-family/README.md` — So sánh Snow Family

### Ưu Tiên Trung Bình (Important Skills)

- [x] `05-data-transfer/1-datasync-overview.md` — DataSync chi tiết
- [x] `05-data-transfer/3-transfer-family.md` — Transfer Family SFTP/FTP
- [x] `02-migration-hub/README.md` — Migration Hub & Discovery
- [x] `04-database-migration/5-dms-cdc.md` — CDC chi tiết
- [x] `09-interview-prep/INTERVIEW_GUIDE.md` — Top 20 câu hỏi phỏng vấn

### Ưu Tiên Thấp (Reference Materials)

- [x] `08-modernization/1-mainframe-modernization.md` — Mainframe guide
- [x] `07-discovery-assessment/README.md` — Assessment guide
- [x] `09-interview-prep/star-stories.md` — STAR story templates
- [x] `09-interview-prep/architecture-scenarios.md` — Design scenarios

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Cho Tự Học

```
1. Bắt đầu với README.md
2. Chọn Learning Path (Beginner/Intermediate/Advanced)
3. Đi qua từng section theo thứ tự
4. Thực hành trong AWS lab (Free Tier)
5. Xây dựng portfolio project
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 09-interview-prep/INTERVIEW_GUIDE.md
2. Nắm vững 7Rs và use cases từng dịch vụ
3. Học 04-database-migration/ (DMS luôn được hỏi)
4. Học 06-snow-family/ (câu hỏi so sánh phổ biến)
5. Chuẩn bị 2-3 câu chuyện thực tế (STAR method)
```

### Cho Dự Án Thực Tế

```
Sử dụng như tài liệu tham khảo:
- Lên kế hoạch: Đọc 01-fundamentals/ để chọn chiến lược
- Theo dõi tiến trình: Dùng 02-migration-hub/ guides
- Di chuyển DB: Theo 04-database-migration/ runbooks
- Truyền dữ liệu: Dùng 05-data-transfer/ methodology
- Đánh giá: Theo 07-discovery-assessment/ checklists
```

### Cho System Design

```
1. Đọc README.md để nắm tổng quan dịch vụ
2. Dùng bảng so sánh Snow Family để chọn đúng thiết bị
3. Tham khảo 03-application-migration/ cho rehost scenarios
4. Theo 04-database-migration/ cho DB migration design
```

---

## 📊 Ước Tính Thời Gian Học (Study Time Estimates)

| Section                         | Thời Gian  | Độ Khó   | Mức Ưu Tiên |
| ------------------------------- | ---------- | -------- | ----------- |
| Nền Tảng (7Rs, phases)          | 3-4 giờ    | ⭐        | Bắt buộc    |
| Migration Hub & Discovery       | 2-3 giờ    | ⭐        | Bắt buộc    |
| Di Chuyển Ứng Dụng (MGN)        | 4-6 giờ    | ⭐⭐      | Bắt buộc    |
| Di Chuyển CSDL (DMS + SCT)      | 8-10 giờ   | ⭐⭐⭐    | Bắt buộc    |
| Truyền Tải Dữ Liệu (DataSync)   | 4-6 giờ    | ⭐⭐      | Bắt buộc    |
| Snow Family                     | 3-4 giờ    | ⭐⭐      | Quan trọng  |
| Khám Phá & Đánh Giá             | 3-4 giờ    | ⭐⭐      | Quan trọng  |
| Hiện Đại Hóa                    | 6-8 giờ    | ⭐⭐⭐    | Tốt để có   |

**Tổng: ~35-50 giờ để có kiến thức toàn diện về AWS Migration**

---

## 🎓 Cấp Độ Kỹ Năng (Skill Levels Supported)

### Người Mới Bắt Đầu (0-1 năm)

- [ ] Hiểu chiến lược 7Rs
- [ ] Phân biệt các dịch vụ migration
- [ ] Biết khi nào dùng Snow vs DataSync
- [ ] Hiểu cơ bản DMS

**Thời gian để nắm vững:** 1-2 tháng

### Trung Cấp (1-3 năm)

- [ ] Thiết kế migration plan với DMS + SCT
- [ ] Cấu hình MGN và thực hiện test cutover
- [ ] Tính toán và tối ưu chi phí migration
- [ ] Thiết kế hybrid connectivity

**Thời gian để nâng cấp:** 2-3 tháng

### Nâng Cao (3+ năm)

- [ ] Thiết kế Multi-wave migration cho enterprise
- [ ] Zero-downtime DB migration với CDC
- [ ] Mainframe Modernization architecture
- [ ] Post-migration optimization và governance

**Thời gian học:** Liên tục cập nhật

---

## 🔗 Điều Hướng Nhanh (Quick Navigation)

| Tôi Cần                              | Vị Trí                                                                 |
| ------------------------------------- | ---------------------------------------------------------------------- |
| Tổng quan và lộ trình                | [README.md](README.md)                                                 |
| Chọn chiến lược migration             | [01-fundamentals/](./01-fundamentals/)                                 |
| Theo dõi dự án migration              | [02-migration-hub/](./02-migration-hub/)                               |
| Di chuyển server lên EC2              | [03-application-migration/](./03-application-migration/)               |
| Di chuyển database                    | [04-database-migration/](./04-database-migration/)                     |
| Đồng bộ file / SFTP server            | [05-data-transfer/](./05-data-transfer/)                               |
| Di chuyển dữ liệu ngoại tuyến         | [06-snow-family/](./06-snow-family/)                                   |
| Đánh giá hạ tầng trước khi di chuyển | [07-discovery-assessment/](./07-discovery-assessment/)                 |
| Hiện đại hóa (Mainframe, A2C, EOS)   | [08-modernization/](./08-modernization/)                               |
| Câu hỏi phỏng vấn                    | [09-interview-prep/](./09-interview-prep/)                             |

---

## 📈 Theo Dõi Tiến Độ Học (Learning Progress Tracker)

Sao chép và theo dõi tiến độ của bạn:

```markdown
## Tiến Độ AWS Migration Knowledge

### Phase 1: Nền Tảng (Tuần 1-2)

- [ ] 7R migration strategies
- [ ] 3 giai đoạn migration (Assess/Mobilize/Migrate)
- [ ] Tổng quan hệ sinh thái AWS Migration
- [ ] Migration Hub và Application Discovery Service

### Phase 2: Di Chuyển Ứng Dụng & DB (Tuần 3-5)

- [ ] AWS MGN: replication agent, launch templates
- [ ] AWS MGN: test cutover và cutover chính thức
- [ ] AWS DMS: homogeneous migration
- [ ] AWS DMS: heterogeneous migration với SCT
- [ ] CDC (Change Data Capture) với DMS

### Phase 3: Truyền Tải Dữ Liệu (Tuần 6-8)

- [ ] AWS DataSync: agent, location, task
- [ ] Transfer Family: SFTP/FTPS setup
- [ ] Snowcone, Snowball, Snowmobile — chọn đúng
- [ ] Storage Gateway — File, Volume, Tape

### Phase 4: Chuyên Sâu (Tuần 9+)

- [ ] Migration Evaluator và TCO analysis
- [ ] Mainframe Modernization
- [ ] Well-Architected Migration Lens
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công (Success Criteria)

Sau khi hoàn thành knowledge base này, bạn phải có thể:

### ✅ Năng Lực Cơ Bản

- [ ] Giải thích 7Rs mà không cần tài liệu
- [ ] Chọn đúng dịch vụ migration cho từng tình huống
- [ ] Phân biệt MGN vs DMS vs DataSync vs Snow Family
- [ ] Thiết kế basic migration architecture

### ✅ Năng Lực Thực Hành

- [ ] Thiết lập DMS replication task
- [ ] Thực hiện MGN test cutover
- [ ] Cấu hình DataSync agent và task
- [ ] Tính toán thời gian và chi phí transfer data

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời tự tin 20 câu hỏi phỏng vấn về AWS Migration
- [ ] Kể 2-3 câu chuyện thực tế dạng STAR
- [ ] Thiết kế migration architecture trong system design interview
- [ ] Thảo luận trade-offs và rủi ro

---

## 🚀 Bước Tiếp Theo (Next Steps)

### Ngay Lập Tức (Tuần Này)

1. Đọc README.md đầy đủ
2. Chọn learning path của bạn
3. Học 01-fundamentals/ (7R strategies — luôn được hỏi)
4. Đăng ký AWS Free Tier để thực hành

### Ngắn Hạn (2 Tuần Tới)

1. Hoàn thành 01-fundamentals/ và 02-migration-hub/
2. Bắt đầu thực hành DMS trong lab
3. Thực hành MGN với EC2 instances
4. Học Snow Family (câu hỏi comparison phổ biến)

### Trung Hạn (4 Tuần Tới)

1. Hoàn thành tất cả core modules (01-06)
2. Deep dive vào DMS và SCT
3. Chuẩn bị 2-3 câu chuyện STAR
4. Mock interview với đồng nghiệp

### Dài Hạn (3 Tháng Tới)

1. Thành thạo một scenario migration phức tạp
2. Hiểu trade-offs của tất cả dịch vụ
3. Xây dựng portfolio project thực tế
4. Nộp đơn vào vị trí Cloud Migration Engineer hoặc Solutions Architect

---

## 💡 Mẹo Học Tập (Pro Tips)

1. **Học qua thực hành:** AWS Free Tier đủ để thực hành DMS, MGN, DataSync
2. **Nhớ con số:** Snowcone 8-14 TB, Snowball 80-210 TB, Snowmobile 100 PB
3. **Hiểu trade-offs:** Không có dịch vụ nào "tốt nhất" — phụ thuộc use case
4. **Tập giải thích:** Giải thích 7Rs cho người không kỹ thuật là kỹ năng quan trọng
5. **Biết giới hạn:** DMS không hỗ trợ tất cả DB, SCT có giới hạn tự động chuyển đổi
6. **Network matters:** Direct Connect giảm rủi ro migration so với VPN
7. **Test trước cutover:** Luôn thực hiện test cutover trước khi production cutover
8. **Validate data:** Sau migration phải kiểm tra tính toàn vẹn dữ liệu

---

## 📞 Đóng Góp (Contributing)

Phát hiện lỗi? Muốn thêm nội dung?

Đây là tài liệu "sống" — đóng góp được hoan nghênh:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm section cho chủ đề chưa có
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn cho concept phức tạp
- [ ] Cập nhật dịch vụ/tính năng mới của AWS

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.9 (09-interview-prep Hoàn Thành)
**Trạng Thái:** ✅ Toàn Bộ Knowledge Base Hoàn Thành — README & INDEX & 01-fundamentals & 02-migration-hub & 03-application-migration & 04-database-migration & 05-data-transfer & 06-snow-family & 07-discovery-assessment & 08-modernization & 09-interview-prep
