# AWS Mainframe Modernization — Hiện Đại Hóa Hệ Thống Mainframe Lên AWS

> **AWS Mainframe Modernization** là dịch vụ giúp tổ chức chuyển đổi ứng dụng mainframe (IBM z/OS, COBOL, PL/I) lên AWS, sử dụng hai phương pháp chính: **Replatform** (tái nền tảng — giữ nguyên code, thay đổi runtime) và **Refactor** (tái cấu trúc — chuyển đổi code sang ngôn ngữ hiện đại). Đây là lĩnh vực chuyên biệt, phức tạp, thường gặp tại ngân hàng, bảo hiểm và hàng không.

## 📚 Mục Lục (Table of Contents)

1. [Mainframe Là Gì?](#mainframe-là-gì)
2. [Tại Sao Phải Modernize Mainframe?](#tại-sao-phải-modernize-mainframe)
3. [Hai Phương Pháp Của AWS](#hai-phương-pháp-của-aws)
4. [Replatform Với Micro Focus](#replatform-với-micro-focus)
5. [Refactor Với AWS Blu Age](#refactor-với-aws-blu-age)
6. [Kiến Trúc Tham Chiếu](#kiến-trúc-tham-chiếu)
7. [Data Migration Mainframe](#data-migration-mainframe)
8. [Testing Strategy Cho Mainframe](#testing-strategy-cho-mainframe)
9. [Chi Phí Và ROI](#chi-phí-và-roi)
10. [Anti-Patterns Cần Tránh](#anti-patterns-cần-tránh)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🖥️ Mainframe Là Gì?

### Định Nghĩa Và Đặc Điểm

```
MAINFRAME — MÁY TÍNH LỚN:

Phần cứng:
├── IBM z16, z15, z14 (thế hệ mới nhất)
├── Giá: $75K đến hàng triệu đô cho phần cứng + licensing
├── Thiết kế để xử lý hàng tỷ giao dịch mỗi ngày với độ tin cậy 99.999%
└── RAS (Reliability, Availability, Serviceability — Độ Tin Cậy, Khả Dụng, Dịch Vụ Được):
    đặc điểm nổi bật nhất của mainframe

Operating Systems (Hệ Điều Hành):
├── z/OS — OS chủ đạo, hầu hết ứng dụng banking chạy trên đây
├── z/VSE — Phiên bản nhỏ hơn, ít dùng hơn
└── z/TPF — Dành cho hệ thống đặt vé hàng không (Airline Reservation)

Ngôn Ngữ Lập Trình:
├── COBOL (Common Business-Oriented Language — Ngôn Ngữ Hướng Kinh Doanh):
│   Ra đời 1959, vẫn xử lý $3 nghìn tỷ giao dịch/ngày toàn cầu
├── PL/I (Programming Language One): Đa mục đích, phổ biến tại chính phủ Mỹ
├── Assembler: Low-level, dùng cho performance-critical modules
└── JCL (Job Control Language — Ngôn Ngữ Kiểm Soát Công Việc):
    Script để chạy batch jobs (công việc xử lý hàng loạt theo lịch)

Middleware (Phần Mềm Trung Gian):
├── CICS (Customer Information Control System — Hệ Thống Kiểm Soát Thông Tin Khách Hàng):
│   Transaction processing — xử lý giao dịch online real-time
├── IMS DB/TM (Information Management System — Hệ Thống Quản Lý Thông Tin):
│   Database và transaction manager thế hệ đầu
├── MQ (Message Queue — Hàng Đợi Tin Nhắn): IBM MQ cho messaging
└── DB2: Relational database của IBM chạy trên z/OS

Storage:
├── VSAM (Virtual Storage Access Method — Phương Pháp Truy Cập Lưu Trữ Ảo):
│   File system đặc biệt của IBM, không phải file system thông thường
└── Sequential datasets: File xử lý tuần tự (batch processing)
```

### Quy Mô Thực Tế

```
THỰC TẾ MAINFRAME TRONG NĂM 2026:

├── 71% Fortune 500 companies vẫn dùng mainframe
├── 95% giao dịch ATM toàn cầu đi qua mainframe
├── $3 nghìn tỷ USD giao dịch thương mại/ngày
├── 1 triệu+ dòng COBOL được viết MỚI mỗi năm (dù cũ)
└── Hơn 220 tỷ dòng COBOL đang vận hành trên toàn thế giới
```

---

## ❓ Tại Sao Phải Modernize Mainframe?

### Chi Phí Vận Hành Quá Cao

```
CHI PHÍ MAINFRAME ĐIỂN HÌNH (MỖI NĂM):

├── IBM z16 hardware + maintenance: $200K – $2M
├── IBM z/OS licensing (theo MIPS — Million Instructions Per Second):
│   $0.5M – $5M/năm tùy workload
├── CICS + DB2 + MQ licensing bổ sung: $200K – $1M
├── Nhân sự vận hành (2-5 FTE — Full-Time Equivalent — Tương Đương Toàn Thời Gian):
│   $150K – $300K/FTE = $300K – $1.5M
├── Trung tâm dữ liệu (điện, làm mát, không gian): $100K – $500K
└── TỔNG ƯỚC TÍNH: $1.5M – $10M/NĂM
```

### Khủng Hoảng Kỹ Sư COBOL

```
VẤN ĐỀ NHÂN LỰC:

├── Độ tuổi trung bình kỹ sư COBOL: 55-65 tuổi (theo COBOL Cowboys survey)
├── Dự báo: 50% kỹ sư COBOL về hưu trong 10 năm tới
├── Ít trường đại học còn dạy COBOL (hiện còn ~1/10 trường)
├── Lương kỹ sư COBOL tăng cao do khan hiếm: $120K–$200K/năm
└── COVID-19 2020: Hệ thống thất nghiệp New Jersey crash vì không đủ COBOL dev
    → Thống đốc phải kêu gọi cộng đồng tình nguyện sửa code COBOL
```

### Không Tích Hợp Được Với Hệ Thống Hiện Đại

```
KHÓ KHĂN TÍCH HỢP:

├── REST API / GraphQL: COBOL không natively support, cần wrapper phức tạp
├── Mobile App: Mainframe không thiết kế để serve mobile traffic trực tiếp
├── Cloud services: Không thể gọi Lambda, SageMaker, Bedrock từ COBOL dễ dàng
├── DevOps / CI/CD: Mainframe dùng JCL scripts, không có Git-native workflow
└── Real-time analytics: Mainframe không tích hợp tốt với data lake, Redshift
```

---

## 🛠️ Hai Phương Pháp Của AWS

### So Sánh Tổng Quan

```
┌──────────────────────────────────────────────────────────────────────────────┐
│           REPLATFORM                      REFACTOR                           │
│           (Tái Nền Tảng)                  (Tái Cấu Trúc)                     │
├──────────────────────────────────────────────────────────────────────────────┤
│  Partner: Micro Focus /                   Partner/Tool: AWS Blu Age          │
│           OpenText COBOL                                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│  Code: COBOL GIỮ NGUYÊN                   Code: COBOL → Java/Spring Boot     │
│  Thay đổi: Runtime environment            Thay đổi: Code + Runtime + Infra   │
├──────────────────────────────────────────────────────────────────────────────┤
│  Tốc độ: Nhanh hơn (tháng)               Tốc độ: Chậm hơn (năm)             │
│  Rủi ro: Thấp hơn                        Rủi ro: Cao hơn                     │
├──────────────────────────────────────────────────────────────────────────────┤
│  Kết quả: COBOL chạy trên EC2/ECS         Kết quả: Java Spring Boot          │
│           với managed COBOL runtime       microservices trên ECS/EKS         │
├──────────────────────────────────────────────────────────────────────────────┤
│  Maintain sau này: Vẫn cần COBOL skill    Maintain sau này: Java devs        │
│                                           thông thường có thể maintain       │
├──────────────────────────────────────────────────────────────────────────────┤
│  ROI: Chủ yếu từ giảm hardware cost      ROI: Giảm hardware + tăng velocity  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Replatform Với Micro Focus

### Micro Focus Enterprise Server Là Gì?

```
MICRO FOCUS (NAY LÀ OPENTEXT) ENTERPRISE SERVER:

Là môi trường chạy (runtime) tương thích với mainframe z/OS:
├── Hỗ trợ COBOL, PL/I, JCL trên Linux/Windows
├── Tương thích CICS: App CICS chạy như nguyên bản
├── Tương thích VSAM: File VSAM được mô phỏng trên filesystem Linux
├── Tương thích JES (Job Entry Subsystem — Hệ Con Nhận Công Việc):
│   Batch job scheduling giống z/OS
└── Chạy trên EC2 (Linux) hoặc trong container (Docker)
```

### Quy Trình Replatform

```
QUY TRÌNH REPLATFORM (6 BƯỚC):

Bước 1: ANALYZE — Phân Tích Code (2-4 tuần)
  ├── Dùng Micro Focus Enterprise Analyzer
  ├── Đếm KLOC (Kilo Lines of Code — Nghìn Dòng Code)
  ├── Vẽ program call graph (đồ thị gọi chương trình)
  ├── Xác định dead code (code không dùng) → Retire
  └── Output: Complexity report, dependency map

Bước 2: PREPARE SOURCE (Chuẩn Bị Mã Nguồn) (2-4 tuần)
  ├── Export COBOL source từ mainframe (PDS — Partitioned Data Set)
  ├── Convert encoding: EBCDIC → ASCII/UTF-8
  │   (EBCDIC — Extended Binary Coded Decimal Interchange Code:
  │    bảng mã IBM mainframe, khác ASCII của PC thông thường)
  ├── Resolve copybooks (COBOL include files) dependencies
  └── Setup version control (Git) cho code mainframe lần đầu

Bước 3: CONFIGURE RUNTIME (Cấu Hình Runtime) (2-4 tuần)
  ├── Cài Micro Focus Enterprise Server trên EC2 Linux
  ├── Configure CICS regions (vùng xử lý giao dịch)
  ├── Map VSAM datasets → Linux filesystem hoặc Amazon EFS
  ├── Configure JES/Job scheduler cho batch jobs
  └── Connect DB2 → Amazon Aurora PostgreSQL (cần data migration)

Bước 4: COMPILE & TEST (Biên Dịch & Kiểm Thử) (4-8 tuần)
  ├── Compile COBOL source bằng Micro Focus COBOL Compiler
  ├── Fix compile errors (thường do dialect khác biệt nhỏ)
  ├── Unit test từng program
  └── Integration test với database và CICS

Bước 5: PARALLEL RUN (Chạy Song Song) (4-8 tuần)
  ├── Chạy cả mainframe và Micro Focus cùng lúc
  ├── So sánh output: phải IDENTICAL (giống hệt nhau)
  ├── Dùng AWS DataSync để sync dữ liệu real-time
  └── Xác nhận không có discrepancy (sai lệch)

Bước 6: CUTOVER & DECOMMISSION (4 tuần)
  ├── Planned cutover window (cửa sổ chuyển đổi có lịch)
  ├── Final data sync từ mainframe → AWS
  ├── DNS/routing switch
  └── Monitor 30 ngày trước khi tắt mainframe
```

---

## 🔵 Refactor Với AWS Blu Age

### AWS Blu Age Là Gì?

```
AWS BLU AGE — AUTOMATED REFACTORING TOOL:

Blu Age là công cụ chuyển đổi tự động (automated code transformation):
├── Input: COBOL/PL/I source code + JCL scripts
├── Output: Java (Spring Boot) microservices
├── Conversion: Tự động + thủ công (không 100% tự động)
└── AWS quản lý Blu Age như managed service trong
    AWS Mainframe Modernization Service console
```

### Quy Trình Refactor Với Blu Age

```
QUY TRÌNH REFACTOR (5 BƯỚC):

Bước 1: ASSESS — Đánh Giá Code (2-4 tuần)
  ├── Upload COBOL source lên AWS Mainframe Modernization
  ├── Blu Age phân tích cấu trúc code tự động
  ├── Báo cáo: tỷ lệ chuyển đổi tự động được (thường 60-80%)
  └── Xác định 20-40% cần chỉnh sửa thủ công

Bước 2: TRANSFORM (Chuyển Đổi) (tháng đến năm)
  ├── Blu Age tạo Java classes từ COBOL programs
  ├── COBOL Divisions (IDENTIFICATION, DATA, PROCEDURE) → Java classes
  ├── CICS EXEC commands → Spring annotations
  ├── VSAM READ/WRITE → JPA (Java Persistence API) repository calls
  └── JCL batch → Spring Batch jobs

Bước 3: REVIEW & REFINE (Xem Xét & Hoàn Thiện) (tháng)
  ├── Java developer review code được tạo
  ├── Refactor các pattern cũ: PERFORM VARYING → Java streams
  ├── Thêm exception handling (xử lý ngoại lệ) đúng cách
  └── Tối ưu SQL queries thay thế cho VSAM access patterns

Bước 4: TEST (Kiểm Thử) (2-4 tháng)
  ├── Unit tests cho từng Java class
  ├── Integration tests với Aurora PostgreSQL
  ├── Regression tests: So sánh output với mainframe gốc
  └── Performance testing: Throughput và latency

Bước 5: DEPLOY & MIGRATE (Deploy & Di Chuyển)
  ├── Deploy Java microservices lên ECS Fargate hoặc EKS
  ├── API Gateway expose endpoints
  ├── Migrate VSAM data → Aurora PostgreSQL
  └── Parallel run → Cutover → Decommission
```

### COBOL → Java Mapping Ví Dụ

```
COBOL PROGRAM (TRƯỚC):
┌────────────────────────────────────────────────────────────────┐
│ IDENTIFICATION DIVISION.                                        │
│   PROGRAM-ID. CALCINT.                                          │
│                                                                │
│ DATA DIVISION.                                                  │
│   WORKING-STORAGE SECTION.                                      │
│     01 WS-PRINCIPAL    PIC 9(10)V99.                           │
│     01 WS-RATE         PIC 9(3)V9(4).                          │
│     01 WS-INTEREST     PIC 9(12)V99.                           │
│                                                                │
│ PROCEDURE DIVISION.                                             │
│   COMPUTE WS-INTEREST = WS-PRINCIPAL * WS-RATE.               │
│   STOP RUN.                                                     │
└────────────────────────────────────────────────────────────────┘

JAVA CLASS (SAU — Blu Age tạo tự động):
┌────────────────────────────────────────────────────────────────┐
│ @Service                                                        │
│ public class CalcintService {                                   │
│                                                                │
│     public BigDecimal calculateInterest(                        │
│             BigDecimal principal, BigDecimal rate) {            │
│         return principal.multiply(rate)                         │
│                         .setScale(2, RoundingMode.HALF_UP);    │
│     }                                                           │
│ }                                                               │
└────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Kiến Trúc Tham Chiếu

### Kiến Trúc Replatform (Micro Focus)

```
ON-PREMISES MAINFRAME          →          AWS CLOUD
┌──────────────────────┐               ┌──────────────────────────────────────┐
│ z/OS LPAR             │               │  VPC Private Subnet                  │
│ ├── COBOL Programs   │               │  ┌────────────────────────────────┐  │
│ ├── CICS Regions     │──[Migration]──►│  │ EC2 (RHEL) với Micro Focus     │  │
│ ├── JCL Batch Jobs   │               │  │ Enterprise Server               │  │
│ ├── VSAM Files       │               │  │ ├── COBOL Programs (NGUYÊN BẢN)│  │
│ └── DB2 Databases    │               │  │ ├── CICS Emulation              │  │
└──────────────────────┘               │  │ ├── JES Batch Scheduler         │  │
                                       │  │ └── VSAM Emulation (EFS)        │  │
                                       │  └─────────────────┬──────────────┘  │
                                       │                    │                  │
                                       │  ┌─────────────────▼──────────────┐  │
                                       │  │ Aurora PostgreSQL               │  │
                                       │  │ (thay thế DB2)                  │  │
                                       │  └────────────────────────────────┘  │
                                       │                                       │
                                       │  ┌────────────────────────────────┐  │
                                       │  │ API Gateway + Lambda           │  │
                                       │  │ (Expose CICS transactions      │  │
                                       │  │  như REST APIs)                │  │
                                       │  └────────────────────────────────┘  │
                                       └──────────────────────────────────────┘
```

### Kiến Trúc Refactor (Blu Age → Microservices)

```
SAU KHI REFACTOR HOÀN THÀNH:

  Clients (Mobile, Web, Partners)
        │
        ▼
  Amazon API Gateway
        │
  ┌─────┴────────────────────────────────────────────┐
  │                                                  │
  ▼                  ▼                  ▼             ▼
ECS Service A    ECS Service B    ECS Service C   ECS Service D
(Account Mgmt)   (Transaction)    (Interest Calc) (Report Gen)
[Java/Spring]    [Java/Spring]    [Java/Spring]   [Java/Spring]
  │                  │                  │               │
  └──────────────────┴──────────────────┴───────────────┘
                           │
              ┌────────────┼─────────────┐
              ▼            ▼             ▼
        Aurora        ElastiCache    Amazon S3
       PostgreSQL        Redis       (Batch Output)
      (dữ liệu           (Cache)     (Thay VSAM datasets)
       giao dịch)

Batch Jobs (thay JCL):
  └──► AWS Step Functions + AWS Batch → ECS Fargate tasks
```

---

## 💾 Data Migration Mainframe

### VSAM → Amazon Aurora PostgreSQL

```
VSAM LÀ GÌ?
VSAM (Virtual Storage Access Method — Phương Pháp Truy Cập Lưu Trữ Ảo):
├── File system đặc biệt của IBM, không phải RDBMS
├── Các loại: KSDS (Key Sequence), ESDS (Entry Sequence),
│            RRDS (Relative Record), LDS (Linear)
└── Không có SQL — COBOL đọc/ghi trực tiếp bằng READ/WRITE verbs

CHIẾN LƯỢC MIGRATION:
┌───────────────────────────────────────────────────────────────────┐
│ Bước 1: Export VSAM → Sequential file (flat file)                │
│         dùng IDCAMS utility hoặc COBOL program                   │
│                                                                   │
│ Bước 2: Transfer file lên S3                                      │
│         dùng AWS DataSync hoặc IBM MFT (Managed File Transfer)   │
│                                                                   │
│ Bước 3: Transform & Load vào Aurora PostgreSQL                   │
│         dùng AWS Glue ETL (Extract, Transform, Load —            │
│         Trích Xuất, Biến Đổi, Nạp) hoặc custom script           │
│                                                                   │
│ Bước 4: Validate data integrity (toàn vẹn dữ liệu)              │
│         Row count comparison, checksum, sample validation        │
└───────────────────────────────────────────────────────────────────┘
```

### DB2 → Aurora PostgreSQL Với AWS DMS

```
DB2 ON z/OS → AURORA POSTGRESQL:

Dùng AWS DMS (Database Migration Service):
├── Source endpoint: DB2 for z/OS (qua DRDA — Distributed Relational
│   Database Architecture — Kiến Trúc Cơ Sở Dữ Liệu Quan Hệ Phân Tán)
├── Target endpoint: Aurora PostgreSQL
├── Full load + CDC để minimize downtime
└── SCT (Schema Conversion Tool) để chuyển đổi DDL (Data Definition Language)
    và stored procedures

Thách Thức Thường Gặp:
├── DB2 data types không map 1:1 với PostgreSQL
│   (ví dụ: DB2 TIMESTAMP vs PostgreSQL TIMESTAMPTZ)
├── Stored procedures dùng DB2-specific syntax → cần viết lại
├── ROWID, REORG operations → không có equivalent trực tiếp
└── Character encoding EBCDIC → UTF-8 trong DMS source
```

---

## 🧪 Testing Strategy Cho Mainframe

### Parallel Run Testing (Kiểm Thử Chạy Song Song)

```
PARALLEL RUN — PHƯƠNG PHÁP QUAN TRỌNG NHẤT:

Ý tưởng: Chạy cả mainframe (cũ) và hệ thống mới ĐỒNG THỜI
         với cùng input → so sánh output phải GIỐNG HỆT

Implementation trên AWS:
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Production Traffic                                                │
│       │                                                            │
│       ├──[100%]──► Mainframe (z/OS) ──► Response to User          │
│       │                    │                                       │
│       └──[100%]──► AWS New System ──► Response (captured,         │
│                                        NOT sent to user)          │
│                                                                    │
│  Shadow Mode Comparison:                                           │
│  ├── Lambda function nhận cả 2 responses                          │
│  ├── So sánh: field by field                                       │
│  ├── Log discrepancies (sai lệch) vào CloudWatch Logs             │
│  └── Alert nếu discrepancy rate > threshold (ngưỡng)             │
└────────────────────────────────────────────────────────────────────┘

Công cụ:
├── AWS Lambda cho comparison logic
├── Amazon Kinesis Data Streams để capture traffic
├── Amazon S3 để lưu test results
└── Amazon QuickSight để visualize discrepancy reports
```

### Regression Testing Suite

```
TEST PYRAMID CHO MAINFRAME MODERNIZATION:

Tầng 1 (Base): Unit Tests
  ├── Test từng COBOL program / Java class độc lập
  ├── Mock external dependencies
  └── Mục tiêu: 80%+ code coverage

Tầng 2: Integration Tests
  ├── Test CICS transactions với database
  ├── Test batch jobs end-to-end
  └── Test data transformations VSAM/DB2 → Aurora

Tầng 3: Business Transaction Tests
  ├── Test toàn bộ business flow (ví dụ: mở tài khoản ngân hàng)
  ├── So sánh output với mainframe reference implementation
  └── Financial reconciliation: tổng tiền phải khớp đến cent cuối

Tầng 4 (Apex): Performance Tests
  ├── Load testing: TPS (Transactions Per Second — Giao Dịch Mỗi Giây)
  ├── Mainframe target: ví dụ 5,000 TPS → AWS phải đạt ít nhất bằng
  ├── Latency P99 (percentile 99 — độ trễ của 99% request): phải ≤ mainframe
  └── Burst capacity: test peak load (tải cao điểm) cuối tháng/năm
```

---

## 💰 Chi Phí Và ROI

### So Sánh Chi Phí Điển Hình

```
CASE STUDY ĐIỂN HÌNH: Ngân hàng vừa, 200 MIPS workload

ON-PREMISES MAINFRAME (mỗi năm):
├── IBM z15 hardware + maintenance: $800,000
├── z/OS licensing (200 MIPS): $1,200,000
├── CICS + DB2 licensing: $400,000
├── Nhân sự vận hành (3 FTE): $450,000
├── Cơ sở vật chất, điện, làm mát: $150,000
└── TỔNG: $3,000,000/năm

AWS (SAU REPLATFORM, mỗi năm):
├── EC2 instances (r6i.8xlarge x4): $180,000
├── Micro Focus Enterprise Server licensing: $400,000
├── Aurora PostgreSQL: $120,000
├── EFS, S3, Network: $40,000
├── Nhân sự AWS (2 FTE): $300,000
└── TỔNG: $1,040,000/năm

TIẾT KIỆM: $1,960,000/năm (~65% reduction)

Chi phí migration một lần (one-time): $2,000,000 – $5,000,000
Payback period (thời gian hoàn vốn): 1-2.5 năm
```

### Khi Nào ROI Không Rõ Ràng

```
TRƯỜNG HỢP ROI THẤP HOẶC ÂM:

├── Mainframe workload rất nhỏ (< 50 MIPS) → chi phí migration vượt lợi ích
├── Ứng dụng sắp retire (loại bỏ) trong 2 năm → không worth migrating
├── COBOL code phức tạp bất thường → migration cost explodes
└── Team thiếu kinh nghiệm → dự án kéo dài, cost overrun

KHUYẾN NGHỊ: Luôn dùng Migration Evaluator + mainframe specialist
để tính TCO trước khi cam kết
```

---

## ⚠️ Anti-Patterns Cần Tránh

```
❌ ANTI-PATTERNS PHỔ BIẾN TRONG MAINFRAME MODERNIZATION:

1. "Big Bang" cutover toàn bộ mainframe cùng lúc
   ✅ Thay bằng: Strangler Fig — từng module một, parallel run

2. Bỏ qua giai đoạn parallel run vì "tốn thời gian"
   ✅ Thực tế: Đây là bước quan trọng nhất, không nên bỏ qua
      Parallel run phát hiện 90%+ bugs trước khi cutover

3. Không migrate VSAM data, dùng tiếp VSAM file trên SFTP
   ✅ Thay bằng: Migrate sang Aurora → query flexibility, scalability

4. Assume COBOL business logic rõ ràng → thực ra không ai hiểu
   ✅ Thay bằng: Viết test cases từ mainframe output trước khi hiểu code

5. Under-estimate thời gian migration
   ✅ Thực tế điển hình:
   - 500K KLOC mainframe → 2-3 năm dự án đầy đủ
   - Luôn cộng thêm 30-50% buffer vào estimate

6. Không có rollback plan
   ✅ Luôn giữ mainframe vận hành được ít nhất 6 tháng sau cutover
```

---

## 🎤 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Sự khác biệt chính giữa Replatform và Refactor cho mainframe trên AWS là gì?**

> **Replatform** (dùng Micro Focus/OpenText): COBOL code giữ nguyên, chỉ thay đổi runtime từ IBM z/OS sang Micro Focus Enterprise Server chạy trên Linux EC2. Rủi ro thấp hơn, nhanh hơn, nhưng vẫn phụ thuộc COBOL và COBOL expertise.
>
> **Refactor** (dùng AWS Blu Age): COBOL được tự động chuyển đổi thành Java Spring Boot microservices. Code hiện đại, bất kỳ Java developer nào cũng có thể maintain, nhưng rủi ro cao hơn và cần testing kỹ lưỡng hơn nhiều.
>
> Thực tế: Nhiều tổ chức bắt đầu bằng Replatform để thoát mainframe nhanh, sau đó dần dần Refactor từng module sang Java khi có thời gian.

---

**Q: VSAM là gì và tại sao nó là thách thức lớn khi migrate mainframe?**

> VSAM (Virtual Storage Access Method) là file system đặc biệt của IBM, không phải relational database. COBOL đọc/ghi VSAM trực tiếp bằng READ/WRITE commands, không qua SQL.
>
> Thách thức migration:
> 1. **Không có schema**: VSAM record layout chỉ biết trong COBOL copybooks — phải reverse-engineer để hiểu cấu trúc
> 2. **Không có standard export**: Cần viết COBOL/IDCAMS scripts để export ra flat file
> 3. **Character encoding**: EBCDIC → phải convert sang UTF-8 khi export
> 4. **Access patterns khác**: VSAM sequential scan → cần thiết kế index đúng trên Aurora để performance tương đương
>
> Giải pháp: Export → S3 → AWS Glue transform → Aurora PostgreSQL với schema được thiết kế lại.

---

### Câu Hỏi Nâng Cao

**Q: Làm thế nào thiết kế parallel run testing cho mainframe migration mà không ảnh hưởng production?**

> Kiến trúc shadow mode:
> 1. **Traffic duplication**: Mọi production request được gửi đồng thời đến cả mainframe (primary) và hệ thống AWS mới (shadow)
> 2. **Response routing**: Chỉ mainframe trả response về cho user; AWS shadow response được capture không gửi user
> 3. **Comparison engine**: Lambda function so sánh hai responses field-by-field
> 4. **Alerting**: CloudWatch alarm nếu discrepancy rate vượt ngưỡng (ví dụ > 0.01%)
>
> Tools trên AWS:
> - Amazon Kinesis để stream requests
> - Lambda để comparison logic
> - S3 để lưu discrepancy logs
> - QuickSight để dashboard theo dõi
>
> Thời gian parallel run: Tối thiểu 4-8 tuần, bao gồm ít nhất 1 chu kỳ xử lý cuối tháng (month-end processing) để bắt các batch jobs định kỳ.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Thuộc Module:** 08-modernization
