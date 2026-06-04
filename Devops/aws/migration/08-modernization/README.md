# AWS Modernization — Hiện Đại Hóa Ứng Dụng Trên AWS: Tổng Quan

> **Hiện đại hóa (Modernization)** là bước tiến xa hơn so với chỉ "lift-and-shift" (nâng và chuyển). Thay vì chỉ di chuyển ứng dụng lên AWS như nguyên trạng, hiện đại hóa là cơ hội để **tái kiến trúc, container hóa và loại bỏ nợ kỹ thuật (technical debt)** — biến hạ tầng cũ thành hệ thống cloud-native (sinh ra để chạy trên cloud). Module này bao gồm ba trụ cột chính: **AWS Mainframe Modernization**, **AWS App2Container**, và **End-of-Support Migration Program (EMP)**.

## 📚 Mục Lục (Table of Contents)

1. [Tại Sao Phải Hiện Đại Hóa?](#tại-sao-phải-hiện-đại-hóa)
2. [Ba Trụ Cột Modernization Trên AWS](#ba-trụ-cột-modernization-trên-aws)
3. [Luồng Làm Việc Tổng Thể](#luồng-làm-việc-tổng-thể)
4. [AWS Mainframe Modernization — Tổng Quan](#aws-mainframe-modernization--tổng-quan)
5. [AWS App2Container — Tổng Quan](#aws-app2container--tổng-quan)
6. [End-of-Support Migration Program — Tổng Quan](#end-of-support-migration-program--tổng-quan)
7. [Bảng So Sánh Ba Công Cụ](#bảng-so-sánh-ba-công-cụ)
8. [Pattern Strangler Fig](#pattern-strangler-fig)
9. [Checklist Modernization](#checklist-modernization)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Tại Sao Phải Hiện Đại Hóa?

### Vấn Đề Của Hệ Thống Legacy

```
❌ NHỮNG VẤN ĐỀ LEGACY ĐIỂN HÌNH:

1. Mainframe (máy tính lớn dòng IBM z/OS, COBOL):
   ├── Chi phí vận hành $1-10 triệu/năm chỉ riêng licensing
   ├── Thiếu kỹ sư COBOL — thế hệ cũ đang về hưu
   ├── Không thể mở rộng theo chiều ngang (horizontal scale)
   └── Không tích hợp được với API / microservices hiện đại

2. Ứng dụng Java/.NET cũ chạy trên bare-metal hoặc VM:
   ├── Deployment (triển khai) thủ công, mất hàng giờ đến hàng ngày
   ├── Không có CI/CD pipeline (đường ống tích hợp/triển khai liên tục)
   ├── Tài nguyên không được cô lập — một app crash ảnh hưởng app khác
   └── Khó scale (mở rộng) theo workload thực tế

3. Windows Server 2012 / SQL Server 2012 hết hỗ trợ:
   ├── Không còn security patch (vá lỗ hổng bảo mật) từ Microsoft
   ├── Tuân thủ PCI-DSS / HIPAA / ISO 27001 bị vi phạm
   ├── Bảo hiểm cyber-risk (rủi ro mạng) từ chối bồi thường
   └── Ứng dụng phụ thuộc phiên bản OS cũ không thể nâng cấp trực tiếp
```

### Lợi Ích Khi Hiện Đại Hóa

```
✅ KẾT QUẢ ĐẠT ĐƯỢC SAU MODERNIZATION:

├── Giảm chi phí vận hành: Mainframe → AWS giảm 50-80% chi phí hạ tầng
├── Developer velocity (tốc độ phát triển): Container + CI/CD → deploy trong phút
├── Scalability (khả năng mở rộng): Auto Scaling theo traffic thực tế
├── Security (bảo mật): Luôn có security patch, WAF, Shield, GuardDuty
├── Innovation (đổi mới): Tích hợp AI/ML, serverless, managed services
└── Talent pool (nguồn nhân lực): Dễ tuyển kỹ sư hơn khi dùng công nghệ hiện đại
```

---

## 🏛️ Ba Trụ Cột Modernization Trên AWS

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                   BA TRỤ CỘT AWS MODERNIZATION                                │
├─────────────────────────┬─────────────────────────┬───────────────────────────┤
│   Trụ Cột 1             │   Trụ Cột 2             │   Trụ Cột 3               │
│   MAINFRAME             │   APP2CONTAINER         │   END-OF-SUPPORT          │
│   MODERNIZATION         │                         │   MIGRATION (EMP)         │
├─────────────────────────┼─────────────────────────┼───────────────────────────┤
│ AWS Mainframe           │ AWS App2Container        │ End-of-Support            │
│ Modernization Service   │ (A2C)                    │ Migration Program         │
├─────────────────────────┼─────────────────────────┼───────────────────────────┤
│ "COBOL/mainframe        │ "App Java/.NET cũ →      │ "Windows 2012 / SQL       │
│  lên microservices"     │  container tự động"      │  Server 2012 hết hạn"     │
├─────────────────────────┼─────────────────────────┼───────────────────────────┤
│ • Refactor COBOL → Java │ • Phân tích app hiện tại │ • Extended Security       │
│ • Replatform → Blu Age  │ • Tạo Dockerfile tự động │   Updates (ESU) trên AWS  │
│ • Managed runtime       │ • Tạo ECS/EKS task def   │ • Nâng cấp lên phiên bản  │
│ • Continuous testing    │ • Deploy pipeline sẵn    │   mới được hỗ trợ         │
└─────────────────────────┴─────────────────────────┴───────────────────────────┘
```

---

## 🔄 Luồng Làm Việc Tổng Thể

```
LUỒNG MODERNIZATION TOÀN DIỆN

┌──────────────────────────────────────────────────────────────────────────────┐
│                    BƯỚC 1: ĐÁNH GIÁ (ASSESS)                                 │
│                                                                              │
│  ┌─────────────────────┐   ┌──────────────────────┐   ┌──────────────────┐  │
│  │ Inventory hệ thống  │   │ Phân loại theo 7R     │   │ Xác định ứng viên│  │
│  │ legacy hiện tại     │──►│ (Retain/Replatform/   │──►│ phù hợp cho từng │  │
│  │                     │   │  Refactor/Retire...)  │   │ công cụ          │  │
│  └─────────────────────┘   └──────────────────────┘   └──────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    BƯỚC 2: PHÂN LOẠI WORKLOAD                                │
│                                                                              │
│  Mainframe (COBOL/PL1)?    │   Java/.NET trên VM?   │  Windows/SQL Server    │
│  └──► Mainframe Moderniz.  │   └──► App2Container   │  hết hỗ trợ?           │
│                            │                        │  └──► EMP              │
└──────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    BƯỚC 3: THỰC HIỆN MODERNIZATION                           │
│                                                                              │
│  Pattern: Strangler Fig (Bóp nghẹt dần) — Chạy song song cũ và mới          │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │  Legacy System ──► [Facade/Router] ──► New Modernized System          │   │
│  │  (Hệ thống cũ)                         (Hệ thống mới)                 │   │
│  │         │                                      │                      │   │
│  │  [Dần dần chuyển traffic sang hệ thống mới theo từng module]          │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    BƯỚC 4: VALIDATE & OPTIMIZE                               │
│                                                                              │
│  ├── Performance testing (kiểm thử hiệu năng)                               │
│  ├── Regression testing (kiểm thử hồi quy — đảm bảo không mất chức năng)   │
│  ├── Cost optimization (tối ưu chi phí với Savings Plans, Graviton)         │
│  └── Decommission legacy (ngừng vận hành hệ thống cũ)                       │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 🖥️ AWS Mainframe Modernization — Tổng Quan

### Mainframe Là Gì Và Tại Sao Cần Modernize?

```
MAINFRAME THỰC TẾ:
├── Phần cứng: IBM z16, z15, z14 — máy chủ đặc biệt, chi phí nhiều triệu đô
├── OS: z/OS (IBM), z/VSE, z/TPF
├── Ngôn ngữ: COBOL (Common Business-Oriented Language — Ngôn Ngữ Hướng Kinh Doanh),
│            PL/I (Programming Language One), JCL (Job Control Language)
├── Middleware: CICS (Customer Information Control System — Hệ Thống Kiểm Soát Thông Tin Khách Hàng),
│              IMS DB/TM (Information Management System)
└── Phổ biến tại: Ngân hàng, bảo hiểm, hàng không, chính phủ

VẤN ĐỀ:
├── Một số ngân hàng lớn vẫn chạy 80% giao dịch ATM trên COBOL
├── Hàng triệu dòng COBOL — không ai dám sửa vì sợ crash
├── Licensing IBM z: $500K–$5M/năm
└── Kỹ sư COBOL trung bình 55-65 tuổi — sắp về hưu hàng loạt
```

### AWS Mainframe Modernization Service — Hai Phương Pháp

```
AWS MAINFRAME MODERNIZATION CÓ HAI LỰA CHỌN CHÍNH:

┌──────────────────────────────────┬──────────────────────────────────────────┐
│ Phương Pháp 1: REPLATFORM        │ Phương Pháp 2: REFACTOR                  │
│ (Tái Nền Tảng)                   │ (Tái Cấu Trúc)                           │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Dùng Micro Focus                 │ Dùng AWS Blu Age                         │
│ (nay là OpenText)                │                                          │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ COBOL code GIỮ NGUYÊN            │ COBOL code được CHUYỂN ĐỔI               │
│ Chạy trên managed runtime        │ thành Java/Spring Boot tự động           │
│ (môi trường chạy được quản lý)   │                                          │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Ưu: Ít rủi ro, nhanh hơn         │ Ưu: Code hiện đại, dễ maintain,          │
│ Nhược: Vẫn phụ thuộc COBOL       │      developer Java có thể hiểu          │
│        logic cũ                  │ Nhược: Rủi ro cao hơn, cần testing kỹ    │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Phù hợp: App ổn định, ít thay    │ Phù hợp: App cần thay đổi thường xuyên, │
│ đổi, muốn thoát khỏi hardware    │ muốn tích hợp với microservices          │
└──────────────────────────────────┴──────────────────────────────────────────┘
```

### Kiến Trúc Managed Runtime

```
AWS MAINFRAME MODERNIZATION — KIẾN TRÚC TỔNG QUAN:

  On-Premises Mainframe                  AWS Cloud
  ┌──────────────────┐                ┌────────────────────────────────────┐
  │  COBOL Programs  │                │                                    │
  │  CICS Screens    │──[Migration]──►│  AWS Mainframe Modernization       │
  │  JCL Batch Jobs  │                │  Managed Runtime Environment       │
  │  VSAM/DB2 Data   │                │  ┌────────────────────────────┐    │
  └──────────────────┘                │  │ Micro Focus / Blu Age      │    │
                                      │  │ Runtime on EC2/ECS         │    │
                                      │  └────────────────────────────┘    │
                                      │           │                        │
                                      │  ┌────────┴────────────────────┐   │
                                      │  │ Aurora PostgreSQL (thay DB2)│   │
                                      │  │ S3 (thay VSAM datasets)     │   │
                                      │  └────────────────────────────┘   │
                                      └────────────────────────────────────┘
```

Chi tiết đầy đủ: [1-mainframe-modernization.md](./1-mainframe-modernization.md)

---

## 📦 AWS App2Container — Tổng Quan

### App2Container (A2C) Là Gì?

```
App2Container (A2C) là công cụ dòng lệnh (CLI) MIỄN PHÍ của AWS
giúp tự động container hóa (containerize) ứng dụng:
├── Java (Spring Boot, Tomcat, JBoss, WebLogic, WebSphere)
└── .NET (ASP.NET, Windows-based apps)

Ứng dụng đang chạy trên:
├── Máy chủ vật lý (bare-metal)
├── Máy ảo VMware (virtual machines)
└── EC2 instances

Kết quả đầu ra:
├── Dockerfile được tạo tự động
├── ECS task definition (cấu hình tác vụ ECS) hoặc EKS manifest
└── CI/CD pipeline trên AWS CodePipeline (tùy chọn)
```

### Luồng Hoạt Động App2Container

```
LUỒNG A2C (4 BƯỚC):

Bước 1: DISCOVER (Khám Phá)
  $ app2container discover
  ├── Scan ứng dụng đang chạy trên server
  ├── Phát hiện Java processes và .NET application pools
  └── Output: danh sách application-id cần containerize

Bước 2: ANALYZE (Phân Tích)
  $ app2container analyze --application-id <id>
  ├── Phân tích dependencies (thư viện, cổng mạng, biến môi trường)
  ├── Tạo file analysis.json với đề xuất cấu hình
  └── Output: analysis.json để review trước khi containerize

Bước 3: CONTAINERIZE (Container Hóa)
  $ app2container containerize --application-id <id>
  ├── Tạo Dockerfile
  ├── Build container image
  ├── Push lên Amazon ECR (Elastic Container Registry — Kho Lưu Trữ Container)
  └── Tạo ECS task definition hoặc EKS deployment manifest

Bước 4: GENERATE PIPELINE (Tạo Pipeline)
  $ app2container generate app-deployment --application-id <id>
  ├── Tạo CloudFormation template
  ├── Cài đặt CodePipeline CI/CD
  └── Deploy lên ECS Fargate hoặc EKS
```

Chi tiết đầy đủ: [2-app2container.md](./2-app2container.md)

---

## 🔒 End-of-Support Migration Program — Tổng Quan

### End-of-Support (EOS) Là Gì?

```
END-OF-SUPPORT (EOS) — HẾT HỖ TRỢ:

Microsoft đã kết thúc hỗ trợ (không còn security patches) cho:
├── Windows Server 2003: Hết hỗ trợ tháng 7/2015
├── Windows Server 2008/R2: Hết hỗ trợ tháng 1/2020
├── Windows Server 2012/R2: Hết hỗ trợ tháng 10/2023
├── SQL Server 2008/R2: Hết hỗ trợ tháng 7/2019
└── SQL Server 2012: Hết hỗ trợ tháng 7/2022

RỦI RO KHI TIẾP TỤC DÙNG PHẦN MỀM HẾT HỖ TRỢ:
├── Security vulnerabilities (lỗ hổng bảo mật) không được vá
├── Compliance violations (vi phạm tuân thủ): PCI-DSS, HIPAA, SOC 2
├── Cyber insurance (bảo hiểm không gian mạng) từ chối bồi thường
└── Phụ thuộc ứng dụng quá cũ không thể upgrade trực tiếp
```

### Giải Pháp AWS Cho EOS

```
AWS CUNG CẤP HAI GIẢI PHÁP:

┌──────────────────────────────────┬──────────────────────────────────────────┐
│ Giải Pháp 1: ESU Miễn Phí       │ Giải Pháp 2: Upgrade Path               │
│ (Extended Security Updates)      │ (Nâng Cấp Phiên Bản)                    │
│ trên AWS                         │                                          │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Chạy Windows Server 2012/R2 hay │ Dùng AWS End-of-Support Migration        │
│ SQL Server 2012 trên EC2 →       │ Program (EMP) để chuyển app Windows      │
│ nhận ESU miễn phí thêm 3 năm    │ Server 2003/2008 → Windows Server        │
│ (Microsoft tính phí $0.05-$0.26 │ 2019/2022 được quản lý bởi AWS          │
│ /vCore/giờ nếu on-premises)      │ Partner Network (APN)                    │
├──────────────────────────────────┼──────────────────────────────────────────┤
│ Phù hợp: Cần thêm thời gian để  │ Phù hợp: Cần upgrade hoàn toàn, app     │
│ lên kế hoạch upgrade đúng đắn   │ tương thích OS mới hơn sau EMP           │
└──────────────────────────────────┴──────────────────────────────────────────┘
```

Chi tiết đầy đủ: [3-end-of-support-migration.md](./3-end-of-support-migration.md)

---

## 📊 Bảng So Sánh Ba Công Cụ

```
┌──────────────────┬──────────────────────┬──────────────────────┬─────────────────────┐
│ Tiêu Chí         │ Mainframe            │ App2Container        │ EMP / ESU           │
│                  │ Modernization        │ (A2C)                │                     │
├──────────────────┼──────────────────────┼──────────────────────┼─────────────────────┤
│ Loại hệ thống    │ Mainframe IBM z/OS   │ Java/.NET trên VM    │ Windows/SQL Server  │
│ mục tiêu         │ COBOL/PL/I apps      │ hoặc bare-metal      │ hết hỗ trợ          │
├──────────────────┼──────────────────────┼──────────────────────┼─────────────────────┤
│ Phương pháp 7R   │ Replatform hoặc      │ Replatform           │ Rehost + Replatform │
│                  │ Refactor             │ (container hóa)      │ hoặc Refactor       │
├──────────────────┼──────────────────────┼──────────────────────┼─────────────────────┤
│ Độ phức tạp      │ Rất cao              │ Trung bình           │ Thấp đến trung bình │
├──────────────────┼──────────────────────┼──────────────────────┼─────────────────────┤
│ Thời gian        │ 6 tháng đến 3 năm    │ Tuần đến tháng       │ Tuần đến tháng      │
├──────────────────┼──────────────────────┼──────────────────────┼─────────────────────┤
│ Đích đến         │ ECS/EKS, Lambda,     │ ECS Fargate, EKS     │ EC2 Windows mới,    │
│                  │ Aurora               │                      │ RDS SQL Server mới  │
├──────────────────┼──────────────────────┼──────────────────────┼─────────────────────┤
│ Tiết kiệm chi    │ 50-80% so với        │ 30-60% so với VM     │ ESU miễn phí trên   │
│ phí điển hình    │ mainframe licensing  │ licensing            │ EC2 (tiết kiệm ESU) │
├──────────────────┼──────────────────────┼──────────────────────┼─────────────────────┤
│ Chi phí dịch vụ  │ Có phí (theo vCPU    │ Miễn phí (A2C CLI)   │ Miễn phí ESU trên   │
│ AWS              │ giờ cho managed      │ + phí ECS/EKS        │ EC2; EMP qua APN    │
│                  │ runtime)             │ runtime              │ Partner (có phí)    │
└──────────────────┴──────────────────────┴──────────────────────┴─────────────────────┘
```

---

## 🌿 Pattern Strangler Fig (Mẫu Bóp Nghẹt Dần)

### Tại Sao Cần Pattern Này?

```
VẤN ĐỀ BIG BANG MODERNIZATION:

❌ CÁCH NGUY HIỂM (Big Bang):
   Tắt toàn bộ hệ thống cũ → Chuyển đổi hoàn toàn → Bật hệ thống mới
   Rủi ro: Nếu thất bại → toàn bộ dịch vụ ngừng hoạt động → không rollback được

✅ CÁCH AN TOÀN (Strangler Fig Pattern):
   Dần dần thay thế từng module của hệ thống cũ bằng module mới
   Hai hệ thống chạy song song trong thời gian chuyển đổi
   Traffic được điều phối từ từ từ cũ sang mới
```

### Cách Triển Khai Strangler Fig Trên AWS

```
STRANGLER FIG TRÊN AWS — 4 GIAI ĐOẠN:

Giai Đoạn 1: Cài Facade (Lớp Trung Gian)
  ┌─────────────────────────────────────────────────────────┐
  │  Client → Amazon API Gateway (Facade) → Legacy System   │
  │  100% traffic vẫn đến legacy                            │
  └─────────────────────────────────────────────────────────┘

Giai Đoạn 2: Xây Module Mới Song Song
  ┌─────────────────────────────────────────────────────────┐
  │  Client → API Gateway → Legacy System (80%)             │
  │                      → New Module A trên ECS (20%)      │
  │  [A/B testing — kiểm thử đồng thời cũ và mới]          │
  └─────────────────────────────────────────────────────────┘

Giai Đoạn 3: Chuyển Dần Traffic
  ┌─────────────────────────────────────────────────────────┐
  │  Client → API Gateway → Legacy System (20%)             │
  │                      → New Modules A+B+C trên ECS (80%) │
  └─────────────────────────────────────────────────────────┘

Giai Đoạn 4: Decommission Legacy
  ┌─────────────────────────────────────────────────────────┐
  │  Client → API Gateway → New System 100% trên ECS/EKS    │
  │  Legacy system bị tắt sau xác nhận ổn định              │
  └─────────────────────────────────────────────────────────┘

Công cụ AWS hỗ trợ:
├── Amazon API Gateway — Facade và routing
├── AWS Lambda — Glue code giữa cũ và mới
├── Amazon SQS/SNS — Event-driven communication (giao tiếp theo sự kiện)
└── Amazon CloudWatch — Monitor cả hai hệ thống song song
```

---

## ✅ Checklist Modernization

### Giai Đoạn Chuẩn Bị (Preparation)

```
□ Inventory toàn bộ legacy systems với công cụ ADS
□ Phân loại theo 7R: xác định cái nào Refactor/Replatform/Retire
□ Tính ROI (Return on Investment — Tỷ Suất Hoàn Vốn) cho từng workload
□ Chọn pattern modernization: Big Bang vs Strangler Fig
□ Xây dựng business case (hồ sơ kinh doanh) trình lên lãnh đạo
□ Thiết lập Landing Zone (vùng hạ cánh) trên AWS với Account Factory
□ Training team về container, Kubernetes, microservices
```

### Mainframe Modernization Checklist

```
□ Audit toàn bộ COBOL/PL/I code — số dòng, độ phức tạp
□ Chạy Micro Focus Enterprise Analyzer để phân tích code
□ Quyết định Replatform (Micro Focus) vs Refactor (Blu Age)
□ Thiết kế data migration: VSAM → Aurora PostgreSQL
□ Xây dựng test suite (bộ kiểm thử) cho regression testing
□ Pilot (thí điểm) với 1 batch job đơn giản nhất trước
□ Chạy parallel testing (kiểm thử song song): cũ và mới cùng output
□ Lên kế hoạch decommission mainframe sau khi validate
```

### App2Container Checklist

```
□ Cài App2Container CLI trên server nguồn
□ Chạy app2container discover để inventory ứng dụng
□ Review analysis.json — kiểm tra dependencies, port mappings
□ Quyết định target: ECS Fargate vs EKS
□ Containerize ứng dụng và test trong môi trường staging
□ Cấu hình logging (ghi log) với CloudWatch Logs
□ Cấu hình secrets (bí mật) với AWS Secrets Manager
□ Load testing sau khi deploy lên container
□ Cắt traffic từ VM cũ sang container mới
```

### EOS Migration Checklist

```
□ Inventory Windows Server / SQL Server hết hỗ trợ
□ Kiểm tra ngày kết thúc hỗ trợ chính thức
□ Quyết định: ESU tạm thời hay upgrade ngay
□ Nếu ESU: Di chuyển lên EC2 để nhận ESU miễn phí
□ Nếu upgrade: Dùng EMP hoặc tự nâng cấp
□ Test tương thích ứng dụng trên OS/SQL Server mới
□ Cập nhật documentation (tài liệu) và runbooks
□ Lên lịch decommission server cũ sau khi validate
```

---

## 🎤 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Sự khác biệt giữa Migration và Modernization trong bối cảnh AWS là gì?**

> **Migration** (di chuyển) là đưa workload từ on-premises lên AWS, thường theo chiến lược Rehost (lift-and-shift) — ít thay đổi code.
>
> **Modernization** (hiện đại hóa) là tái kiến trúc ứng dụng để tận dụng tối đa cloud-native capabilities: container hóa, serverless, managed services. Thường theo chiến lược Replatform hoặc Refactor.
>
> Thực tế: Migration và Modernization thường xảy ra song song — di chuyển trước (để thoát khỏi data center) rồi modernize dần sau khi đã lên cloud.

---

**Q: Khi nào nên chọn Replatform mainframe (Micro Focus) thay vì Refactor (Blu Age)?**

> **Chọn Replatform (Micro Focus)** khi:
> - Codebase COBOL rất lớn (triệu dòng), rủi ro refactor cao
> - Timeline ngắn — cần rời mainframe nhanh do hợp đồng sắp hết hạn
> - App ổn định, ít thay đổi business logic
> - Team không có kỹ sư Java để maintain code sau khi chuyển đổi
>
> **Chọn Refactor (Blu Age)** khi:
> - App cần thay đổi thường xuyên trong tương lai
> - Muốn tích hợp với microservices, REST API, CI/CD
> - Có kế hoạch dài hạn: team phát triển cần maintain code hiện đại
> - Budget đủ để đầu tư vào testing kỹ lưỡng

---

**Q: App2Container có hỗ trợ mọi ứng dụng Java không?**

> Không. App2Container hỗ trợ tốt nhất:
> - **Java**: Tomcat, JBoss/WildFly, WebLogic, WebSphere
> - **.NET**: ASP.NET trên IIS (Windows)
>
> **Không hỗ trợ hoặc hỗ trợ hạn chế**:
> - Ứng dụng có stateful session phức tạp (cần refactor thêm)
> - App dùng local file system làm storage (cần mount EFS hoặc S3)
> - App cần GUI desktop (không phải web app)
> - Ứng dụng phụ thuộc hardware-specific driver
>
> Sau khi A2C tạo container, vẫn cần kiểm tra và điều chỉnh `analysis.json` thủ công trước khi deploy production.

---

### Câu Hỏi Nâng Cao

**Q: Tại sao Strangler Fig Pattern an toàn hơn Big Bang Modernization?**

> **Strangler Fig Pattern** an toàn hơn vì:
>
> 1. **Rollback dễ dàng**: Nếu module mới lỗi, chỉ cần chuyển traffic về legacy — không ảnh hưởng toàn hệ thống
> 2. **Incremental validation** (xác nhận từng bước): Mỗi module được test riêng lẻ trước khi chuyển traffic
> 3. **User impact tối thiểu**: Người dùng không biết hệ thống đang được thay thế dần
> 4. **Học hỏi dần**: Team rút kinh nghiệm từ module đầu tiên trước khi xử lý module phức tạp hơn
>
> **Big Bang rủi ro** vì: Nếu hệ thống mới có bug nghiêm trọng sau khi cutover, legacy đã bị tắt → không có đường quay lại → downtime (ngừng hoạt động) kéo dài.

---

**Q: ESU miễn phí cho Windows Server 2012 trên EC2 hoạt động như thế nào về mặt kỹ thuật?**

> Khi chạy Windows Server 2012/R2 trên EC2:
> - AWS tự động cấp **Extended Security Updates** thông qua Windows Update thông thường
> - Không cần cấu hình thêm — AWS đã thỏa thuận với Microsoft để cung cấp ESU tự động
> - ESU miễn phí được cung cấp đến **tháng 10/2026** (3 năm sau EOS tháng 10/2023)
>
> **Quan trọng**: ESU chỉ bao gồm security patches — không bao gồm new features hay non-security bug fixes.
>
> **So sánh chi phí**: On-premises phải trả Microsoft $0.05–$0.26/vCore/giờ cho ESU → EC2 → miễn phí = tiết kiệm đáng kể, đặc biệt với fleet (đội) Windows Server lớn.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Thuộc Module:** 08-modernization
