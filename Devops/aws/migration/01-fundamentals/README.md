# Nền Tảng AWS Migration (Fundamentals)

> Hiểu chiến lược 7Rs, ba giai đoạn di chuyển, hệ sinh thái công cụ và phân tích chi phí TCO — nền tảng bắt buộc trước mọi dự án migration.

## 📚 Mục Lục (Table of Contents)

1. [Tại Sao Cần Migration?](#tại-sao-cần-migration)
2. [Chiến Lược 7Rs](#chiến-lược-7rs)
3. [Ba Giai Đoạn Migration](#ba-giai-đoạn-migration)
4. [Hệ Sinh Thái AWS Migration](#hệ-sinh-thái-aws-migration)
5. [TCO và Phân Tích Chi Phí](#tco-và-phân-tích-chi-phí)
6. [Điều Hướng Tài Liệu](#điều-hướng-tài-liệu)

---

## 🎯 Tại Sao Cần Migration?

### Động Lực Kinh Doanh (Business Drivers)

| Động Lực | Mô Tả |
| -------- | ------ |
| **Giảm Chi Phí** | Tối ưu CAPEX (chi phí vốn) → OPEX (chi phí vận hành) |
| **Tăng Tính Linh Hoạt** | Mở rộng/thu hẹp tài nguyên theo nhu cầu |
| **Tăng Tốc Đổi Mới** | Tận dụng AI/ML, serverless, managed services |
| **Tăng Độ Tin Cậy** | SLA 99.99%, multi-AZ, disaster recovery tự động |
| **Bảo Mật Tốt Hơn** | Mô hình trách nhiệm chia sẻ, compliance tự động |
| **Hết Hợp Đồng Hạ Tầng** | Data center lease sắp hết hạn |

### Thách Thức Phổ Biến (Common Challenges)

```
Rủi ro Kỹ Thuật:
├── Tương thích hệ điều hành và phần mềm
├── Phụ thuộc ngầm giữa các ứng dụng (hidden dependencies)
├── Thay đổi địa chỉ IP / hostname
└── Hiệu năng sau migration khác on-premises

Rủi ro Dữ Liệu:
├── Mất dữ liệu (data loss) trong quá trình chuyển
├── Downtime (thời gian ngừng hoạt động) không chấp nhận được
├── Tính toàn vẹn dữ liệu (data integrity) sau migration
└── Compliance và data residency requirements

Rủi ro Tổ Chức:
├── Thiếu kỹ năng cloud trong team
├── Kháng cự thay đổi từ nhân viên
└── Timeline quá gấp, thiếu lập kế hoạch
```

---

## 🗂️ Chiến Lược 7Rs

> **7Rs** là framework phân loại cách di chuyển từng workload lên AWS. Mỗi "R" đại diện cho một mức độ thay đổi khác nhau.

### Tổng Quan Nhanh

| Strategy | Tên Đầy Đủ | Thay Đổi Code | Tốc Độ | Chi Phí Tối Ưu |
| -------- | ---------- | ------------- | ------ | --------------- |
| **Retire** | Ngừng sử dụng | Không có | Ngay lập tức | Tiết kiệm 100% |
| **Retain** | Giữ nguyên on-premises | Không có | - | Không thay đổi |
| **Rehost** | Lift-and-Shift | Không có | Nhanh nhất | Thấp ban đầu |
| **Relocate** | Di chuyển hypervisor | Không có | Nhanh | Thấp ban đầu |
| **Repurchase** | Chuyển sang SaaS | Không có | Trung bình | Thay đổi mô hình |
| **Replatform** | Lift-and-Reshape | Tối thiểu | Trung bình | Tốt hơn |
| **Refactor** | Re-architect | Nhiều | Chậm nhất | Tốt nhất dài hạn |

Chi tiết đầy đủ: [1-7r-strategies.md](./1-7r-strategies.md)

---

## 📅 Ba Giai Đoạn Migration

> AWS chia quá trình migration thành **3 giai đoạn chính**, mỗi giai đoạn có mục tiêu và công cụ riêng.

```
ASSESS          →    MOBILIZE        →    MIGRATE & MODERNIZE
(Đánh Giá)           (Chuẩn Bị)           (Di Chuyển & Hiện Đại Hóa)

Khám phá hạ tầng     Lập kế hoạch         Thực hiện migration
Đánh giá TCO         Xây dựng team        Tối ưu và hiện đại hóa
Chọn chiến lược      Pilot migration      Vận hành production
```

### Giai Đoạn 1 — Assess (Đánh Giá)

**Mục tiêu:** Hiểu rõ môi trường hiện tại và xây dựng business case.

- Khám phá tự động bằng **AWS Application Discovery Service (ADS)**
- Phân tích danh mục ứng dụng với **AWS Migration Evaluator**
- Tính toán **TCO — Total Cost of Ownership (Tổng Chi Phí Sở Hữu)**
- Xác định dependencies giữa các ứng dụng
- Phân loại workload theo chiến lược 7Rs

**Thời gian điển hình:** 2-8 tuần

### Giai Đoạn 2 — Mobilize (Chuẩn Bị)

**Mục tiêu:** Xây dựng nền tảng kỹ thuật và tổ chức cho migration.

- Thiết lập **Landing Zone** — môi trường AWS nền tảng với governance
- Cấu hình **AWS Control Tower** và **AWS Organizations**
- Xây dựng **Migration Factory** — quy trình di chuyển lặp lại được
- Đào tạo đội ngũ kỹ thuật
- Thực hiện **Pilot Migration** — di chuyển 2-3 workload thí điểm
- Thiết lập kết nối mạng: **AWS Direct Connect** hoặc VPN

**Thời gian điển hình:** 3-6 tháng

### Giai Đoạn 3 — Migrate & Modernize (Di Chuyển & Hiện Đại Hóa)

**Mục tiêu:** Di chuyển toàn bộ workload theo lịch trình và hiện đại hóa sau migration.

- Di chuyển theo **Migration Waves (Đợt Di Chuyển)** — nhóm workload liên quan
- Theo dõi tiến trình tập trung bằng **AWS Migration Hub**
- Thực hiện **Cutover** từ on-premises sang AWS
- Tối ưu hóa sau migration: right-sizing, reserved instances
- Hiện đại hóa: containerization, serverless, managed services

**Thời gian điển hình:** 6-24 tháng (tuỳ quy mô)

Chi tiết đầy đủ: [2-migration-phases.md](./2-migration-phases.md)

---

## 🌐 Hệ Sinh Thái AWS Migration

> AWS cung cấp một bộ công cụ toàn diện, mỗi công cụ phục vụ một bước cụ thể trong quá trình migration.

### Sơ Đồ Tổng Quan

```
ASSESS Phase:
├── AWS Application Discovery Service (ADS) — Khám phá hạ tầng tự động
├── AWS Migration Evaluator — Tính TCO và đề xuất phương án
└── AWS Migration Hub — Theo dõi tiến trình tập trung

MIGRATE Phase:
├── Di chuyển máy chủ / ứng dụng:
│   └── AWS MGN — Application Migration Service (lift-and-shift)
├── Di chuyển cơ sở dữ liệu:
│   ├── AWS DMS — Database Migration Service
│   └── AWS SCT — Schema Conversion Tool
├── Truyền tải dữ liệu qua mạng:
│   ├── AWS DataSync — Đồng bộ file/object nhanh
│   └── AWS Transfer Family — SFTP/FTPS managed
└── Truyền tải dữ liệu ngoại tuyến:
    ├── AWS Snowcone (8-14 TB)
    ├── AWS Snowball Edge (80-210 TB)
    └── AWS Snowmobile (100 PB)

MODERNIZE Phase:
├── AWS Mainframe Modernization — Refactor mainframe
├── AWS App2Container — Container hóa ứng dụng Java/.NET
└── AWS Elastic Disaster Recovery (DRS)
```

Chi tiết đầy đủ: [3-aws-migration-ecosystem.md](./3-aws-migration-ecosystem.md)

---

## 💰 TCO và Phân Tích Chi Phí

> **TCO — Total Cost of Ownership (Tổng Chi Phí Sở Hữu)** là tổng tất cả chi phí thực sự để vận hành hệ thống, bao gồm cả chi phí ẩn.

### Chi Phí On-Premises (Thường Bị Đánh Giá Thấp)

```
Chi phí phần cứng:
├── Mua server, storage, networking
├── Khấu hao thiết bị (3-5 năm)
└── Thay thế khi hỏng hóc

Chi phí hạ tầng:
├── Điện (Power)
├── Làm mát (Cooling) — thường bằng 50-100% chi phí điện
├── Diện tích data center (Colocation)
└── Kết nối mạng (bandwidth, cross-connects)

Chi phí con người:
├── Quản trị hệ thống (system administration)
├── Bảo trì phần cứng
├── Vận hành 24/7 on-call
└── Đào tạo kỹ thuật

Chi phí ẩn khác:
├── License phần mềm
├── Bảo hiểm thiết bị
├── Chi phí downtime (lost revenue)
└── Rủi ro bảo mật
```

### Mô Hình Chi Phí AWS (Pay-As-You-Go)

- **On-Demand:** Trả theo giờ, không cam kết — linh hoạt nhất
- **Reserved Instances (RI):** Cam kết 1-3 năm — tiết kiệm 30-60%
- **Savings Plans:** Linh hoạt hơn RI, tiết kiệm tương tự
- **Spot Instances:** Tận dụng tài nguyên dư — tiết kiệm tới 90%

Chi tiết đầy đủ: [4-tco-cost-analysis.md](./4-tco-cost-analysis.md)

---

## 🔗 Điều Hướng Tài Liệu

| File | Nội Dung |
| ---- | -------- |
| [1-7r-strategies.md](./1-7r-strategies.md) | Chi tiết từng chiến lược 7Rs với ví dụ thực tế và decision tree |
| [2-migration-phases.md](./2-migration-phases.md) | Ba giai đoạn Assess → Mobilize → Migrate với checklist |
| [3-aws-migration-ecosystem.md](./3-aws-migration-ecosystem.md) | Tổng quan và so sánh tất cả công cụ migration của AWS |
| [4-tco-cost-analysis.md](./4-tco-cost-analysis.md) | Cách tính TCO, so sánh chi phí, công cụ ước tính |

---

## 🎓 Câu Hỏi Phỏng Vấn Điển Hình (Module Này)

1. Giải thích 7 chiến lược migration (7Rs) và cho ví dụ từng loại
2. Khi nào nên chọn Rehost thay vì Refactor?
3. Ba giai đoạn migration AWS là gì? Giai đoạn nào quan trọng nhất?
4. TCO là gì? Những chi phí nào thường bị bỏ sót khi tính TCO on-premises?
5. Làm thế nào để phân loại 500 workload vào các chiến lược 7Rs?

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
