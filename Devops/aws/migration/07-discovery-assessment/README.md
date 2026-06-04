# AWS Discovery & Assessment — Khám Phá & Đánh Giá Trước Khi Di Chuyển: Tổng Quan

> **Giai đoạn Assess (Đánh giá)** là bước đầu tiên và quan trọng nhất trong hành trình migration lên AWS. Trước khi di chuyển bất kỳ workload (khối lượng công việc) nào, bạn cần hiểu rõ hạ tầng hiện tại — bao nhiêu máy chủ, ứng dụng nào phụ thuộc vào nhau, chi phí on-premises thực sự là bao nhiêu, và kiến trúc đích nên trông như thế nào. Module này tổng hợp ba công cụ cốt lõi: **Application Discovery Service (ADS)**, **Migration Evaluator**, và **AWS Well-Architected Migration Lens**.

## 📚 Mục Lục (Table of Contents)

1. [Tại Sao Phải Assess Trước?](#tại-sao-phải-assess-trước)
2. [Ba Trụ Cột Của Giai Đoạn Assess](#ba-trụ-cột-của-giai-đoạn-assess)
3. [Luồng Làm Việc Tổng Thể](#luồng-làm-việc-tổng-thể)
4. [AWS Application Discovery Service — Tổng Quan](#aws-application-discovery-service--tổng-quan)
5. [AWS Migration Evaluator — Tổng Quan](#aws-migration-evaluator--tổng-quan)
6. [AWS Well-Architected Migration Lens — Tổng Quan](#aws-well-architected-migration-lens--tổng-quan)
7. [Bảng So Sánh Ba Công Cụ](#bảng-so-sánh-ba-công-cụ)
8. [Tích Hợp Với Migration Hub](#tích-hợp-với-migration-hub)
9. [Checklist Assess Phase](#checklist-assess-phase)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Tại Sao Phải Assess Trước?

### Hậu Quả Nếu Bỏ Qua Giai Đoạn Assess

```
❌ NHỮNG GÌ XẢY RA KHI KHÔNG ASSESS KỸ:

1. Bỏ sót dependency (phụ thuộc giữa ứng dụng):
   ├── App A phụ thuộc DB B → di chuyển A trước → App A bị lỗi
   ├── Microservice gọi nhau qua hostname on-premises → mất kết nối sau migration
   └── License (bản quyền) phần mềm bị tied (ràng buộc) với MAC address cũ

2. Oversizing hoặc Undersizing (định cỡ sai):
   ├── Mua EC2 instance quá lớn → lãng phí chi phí
   ├── Mua EC2 instance quá nhỏ → hiệu năng kém, phải resize sau
   └── Không biết peak usage (mức dùng cao điểm) thực tế

3. Chi phí thực tế cao hơn dự kiến:
   ├── Tính TCO (Total Cost of Ownership — Tổng Chi Phí Sở Hữu) sai
   ├── Bỏ qua chi phí network, licensing, training
   └── Không tính hidden costs (chi phí ẩn) của on-premises

4. Vi phạm compliance (tuân thủ quy định):
   ├── Không biết dữ liệu nào là PII (Personally Identifiable Information — Thông Tin Cá Nhân)
   ├── Di chuyển dữ liệu nhạy cảm không đúng quy trình
   └── Regulatory requirements (yêu cầu pháp lý) theo vùng địa lý
```

### Lợi Ích Khi Assess Đúng Cách

```
✅ KẾT QUẢ KHI ASSESS KỸ LƯỠNG:

├── Inventory chính xác: Biết chính xác có bao nhiêu server, app, DB
├── Dependency map (bản đồ phụ thuộc): Biết thứ tự di chuyển an toàn
├── Right-sizing: Chọn đúng EC2 instance type, tiết kiệm 20-40% chi phí
├── TCO comparison: Có dữ liệu thuyết phục ban lãnh đạo phê duyệt ngân sách
├── Risk identification: Xác định rủi ro trước, lên kế hoạch giảm thiểu
└── Migration wave planning: Chia nhỏ migration thành các đợt (wave) có kiểm soát
```

---

## 🏛️ Ba Trụ Cột Của Giai Đoạn Assess

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BA TRỤ CỘT ASSESS PHASE                                  │
├─────────────────────────┬───────────────────────────┬───────────────────────┤
│   Trụ Cột 1             │   Trụ Cột 2               │   Trụ Cột 3           │
│   KHÁM PHÁ              │   ĐÁNH GIÁ TÀI CHÍNH      │   ĐÁNH GIÁ KIẾN TRÚC │
│   (Discovery)           │   (Financial Assessment)   │   (Architecture       │
│                         │                            │    Assessment)        │
├─────────────────────────┼───────────────────────────┼───────────────────────┤
│ AWS Application         │ AWS Migration              │ AWS Well-Architected  │
│ Discovery Service       │ Evaluator                  │ Migration Lens        │
│ (ADS)                   │                            │                       │
├─────────────────────────┼───────────────────────────┼───────────────────────┤
│ "Có gì trong hạ tầng?"  │ "Di chuyển có rẻ hơn?"    │ "Di chuyển có đúng?"  │
├─────────────────────────┼───────────────────────────┼───────────────────────┤
│ • Khám phá máy chủ      │ • Tính TCO on-premises     │ • Review kiến trúc    │
│ • Đo hiệu năng          │ • Tính TCO AWS dự kiến     │ • Best practices       │
│ • Vẽ dependency map     │ • Đề xuất right-sizing     │ • Risk assessment      │
│ • Phân loại ứng dụng    │ • Lộ trình tiết kiệm       │ • Gaps & improvements │
└─────────────────────────┴───────────────────────────┴───────────────────────┘
```

---

## 🔄 Luồng Làm Việc Tổng Thể

```
LUỒNG ASSESS → MOBILIZE → MIGRATE & MODERNIZE

┌──────────────────────────────────────────────────────────────────────────────┐
│                         PHASE 1: ASSESS                                      │
│                                                                              │
│  Bước 1: Khám Phá Hạ Tầng (2-4 tuần)                                       │
│  ┌─────────────────────────────────────┐                                     │
│  │  ADS Agentless Collector            │  → Thu thập: IP, hostname, OS,      │
│  │  hoặc                               │    CPU/RAM usage, network flows      │
│  │  ADS Discovery Agent                │  → Kết quả: Server inventory        │
│  └─────────────────────────────────────┘    + dependency map                │
│                         │                                                    │
│                         ▼                                                    │
│  Bước 2: Phân Tích Tài Chính (1-2 tuần)                                    │
│  ┌─────────────────────────────────────┐                                     │
│  │  Migration Evaluator                │  → Import dữ liệu từ ADS            │
│  │  (hoặc upload inventory thủ công)   │  → Tính TCO 3-5 năm                │
│  └─────────────────────────────────────┘  → Right-sizing recommendations    │
│                         │                                                    │
│                         ▼                                                    │
│  Bước 3: Đánh Giá Kiến Trúc (1-2 tuần)                                     │
│  ┌─────────────────────────────────────┐                                     │
│  │  Well-Architected Migration Lens    │  → Review theo 6 pillars            │
│  │  + AWS Migration Hub                │  → Identify gaps                   │
│  └─────────────────────────────────────┘  → Remediation plan                │
│                         │                                                    │
│                         ▼                                                    │
│  Output (Đầu Ra):                                                            │
│  ├── Migration Business Case (Hồ Sơ Kinh Doanh Di Chuyển)                  │
│  ├── Application Portfolio (Danh Mục Ứng Dụng) với strategy 7Rs             │
│  ├── Migration Wave Plan (Kế Hoạch Di Chuyển Theo Đợt)                      │
│  └── Risk Register (Sổ Đăng Ký Rủi Ro)                                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 AWS Application Discovery Service — Tổng Quan

### Hai Chế Độ Thu Thập Dữ Liệu

```
ADS CÓ HAI PHƯƠNG THỨC:

┌───────────────────────────────┬────────────────────────────────────────────┐
│ Agentless Discovery           │ Agent-based Discovery                      │
│ (Khám Phá Không Cần Agent)   │ (Khám Phá Qua Agent)                       │
├───────────────────────────────┼────────────────────────────────────────────┤
│ Dùng Agentless Collector      │ Cài ADS Discovery Agent trên từng server   │
│ (máy ảo OVA cài lên vSphere)  │                                            │
├───────────────────────────────┼────────────────────────────────────────────┤
│ Dữ liệu: VM metadata,         │ Dữ liệu: Process level, network TCP/UDP,   │
│ CPU/RAM config, VM counts     │ performance metrics chi tiết theo giờ      │
├───────────────────────────────┼────────────────────────────────────────────┤
│ Phù hợp: VMware vSphere       │ Phù hợp: Mọi OS (Windows, Linux)          │
│ environments                  │ kể cả bare-metal (máy vật lý trực tiếp)    │
├───────────────────────────────┼────────────────────────────────────────────┤
│ Ưu điểm: Triển khai nhanh,    │ Ưu điểm: Dữ liệu sâu hơn, dependency      │
│ không cần quyền OS            │ mapping chính xác hơn                      │
│                               │                                            │
│ Nhược điểm: Dữ liệu ít chi   │ Nhược điểm: Phải cài trên từng server,     │
│ tiết hơn                      │ cần quyền admin                            │
└───────────────────────────────┴────────────────────────────────────────────┘
```

### Dữ Liệu Được Thu Thập

```
Agentless Collector thu thập:
├── VM configuration: vCPU, RAM, disk size, OS type
├── VM performance: CPU utilization (%), memory utilization (%)
├── Network: IP addresses, MAC addresses
└── VMware metadata: cluster, datastore, host

Discovery Agent thu thập thêm:
├── Running processes: tên process, PID, user, CPU/RAM usage
├── Network connections: source IP:port → destination IP:port
├── Performance metrics: disk I/O, network throughput theo thời gian thực
└── Installed software: package list, version, license keys
```

### Kết Quả: Dependency Visualization

```
Ví dụ Dependency Map (Bản Đồ Phụ Thuộc) sau khi chạy ADS:

  Web Server (192.168.1.10)
       │
       │ TCP:8080
       ▼
  App Server (192.168.1.20)
       │
       ├── TCP:3306 ──► MySQL Primary (192.168.1.30)
       │                    │
       │                    └── Replication ──► MySQL Replica (192.168.1.31)
       │
       └── TCP:6379 ──► Redis Cache (192.168.1.40)

  Kết quả: Phải di chuyển MySQL + Redis TRƯỚC App Server
           Phải di chuyển App Server TRƯỚC Web Server
```

Chi tiết đầy đủ: [1-application-discovery-service.md](./1-application-discovery-service.md)

---

## 💰 AWS Migration Evaluator — Tổng Quan

### Mục Đích: Xây Dựng Business Case

```
Migration Evaluator (trước đây là TSO Logic) giải quyết câu hỏi:

"Nếu chúng tôi di chuyển lên AWS, chúng tôi sẽ tiết kiệm hay tốn thêm bao nhiêu?"

Đầu vào (Input):
├── Dữ liệu từ ADS (tự động import)
├── Upload file inventory thủ công (Excel/CSV)
├── Agentless Collector (cài trực tiếp để thu thập)
└── Thông tin license, hợp đồng maintenance hiện tại

Xử Lý:
├── Tính TCO on-premises hiện tại (3-5 năm)
├── Tính TCO AWS dự kiến với right-sizing
├── So sánh với/không có Savings Plans, Reserved Instances
└── Đề xuất EC2 instance type phù hợp nhất

Đầu ra (Output) — Migration Business Case:
├── Báo cáo TCO chi tiết (PDF/Excel)
├── Tiết kiệm ước tính theo năm (annual savings estimate)
├── Right-sizing recommendations (đề xuất EC2 instance cụ thể)
└── Phân tích 3-5 năm
```

### Ví Dụ Báo Cáo Thực Tế

```
┌───────────────────────────────────────────────────────────────────────────┐
│                  MIGRATION EVALUATOR — KẾT QUẢ MẪU                        │
├───────────────────────────────────────────────────────────────────────────┤
│ Hạ Tầng Đánh Giá: 120 máy chủ vật lý, 3 năm phân tích                   │
│                                                                           │
│  Chi Phí On-Premises (3 năm):                                             │
│  ├── Hardware (phần cứng):          $1,200,000                            │
│  ├── Software licensing:            $  450,000                            │
│  ├── Power & cooling (điện & làm    $  180,000                            │
│  │   mát):                                                                │
│  ├── Facilities (cơ sở vật chất):   $  240,000                            │
│  └── IT staff (nhân lực IT):        $  600,000                            │
│  TỔNG ON-PREMISES:                  $2,670,000                            │
│                                                                           │
│  Chi Phí AWS (3 năm, với 1-year Reserved Instances):                      │
│  ├── EC2 compute:                   $  520,000                            │
│  ├── RDS databases:                 $  180,000                            │
│  ├── Storage (EBS + S3):            $   95,000                            │
│  ├── Network (data transfer):       $   65,000                            │
│  └── Support plan:                  $   48,000                            │
│  TỔNG AWS:                          $  908,000                            │
│                                                                           │
│  TIẾT KIỆM: $1,762,000 (66% trong 3 năm)                                 │
└───────────────────────────────────────────────────────────────────────────┘
```

Chi tiết đầy đủ: [2-migration-evaluator.md](./2-migration-evaluator.md)

---

## 🏗️ AWS Well-Architected Migration Lens — Tổng Quan

### Migration Lens Là Gì?

```
AWS Well-Architected Framework có nhiều "Lens" (góc nhìn chuyên biệt).
Migration Lens là bộ câu hỏi và best practices (thực hành tốt nhất)
dành riêng cho các dự án migration lên AWS.

6 Pillars (Trụ Cột) của Well-Architected:
┌─────────────────────┬──────────────────────────────────────────────────────┐
│ Pillar              │ Câu Hỏi Migration Tiêu Biểu                          │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ Operational         │ Bạn có runbook cho từng app khi migration không?      │
│ Excellence          │ Ai chịu trách nhiệm rollback nếu cutover thất bại?   │
│ (Xuất Sắc Vận Hành) │                                                      │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ Security            │ IAM roles cho migration tools đã được tối giản        │
│ (Bảo Mật)           │ (least privilege) chưa?                              │
│                     │ Dữ liệu được mã hóa trong quá trình migration?       │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ Reliability         │ Có test cutover (kiểm tra chuyển đổi) chưa?          │
│ (Độ Tin Cậy)        │ Kế hoạch rollback (quay lại) khi lỗi là gì?          │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ Performance         │ Right-sizing đã được thực hiện chưa?                 │
│ Efficiency          │ Auto Scaling đã được cấu hình chưa?                  │
│ (Hiệu Quả Hiệu Năng)│                                                      │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ Cost Optimization   │ Có plan để dùng Savings Plans hoặc Reserved          │
│ (Tối Ưu Chi Phí)    │ Instances sau khi ổn định không?                     │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ Sustainability      │ Dùng ARM-based Graviton instances để giảm carbon      │
│ (Bền Vững)          │ footprint (dấu chân carbon) không?                   │
└─────────────────────┴──────────────────────────────────────────────────────┘
```

Chi tiết đầy đủ: [3-well-architected-migration.md](./3-well-architected-migration.md)

---

## 📊 Bảng So Sánh Ba Công Cụ

```
┌──────────────────────┬──────────────────────┬──────────────────────┬──────────────────────┐
│ Tiêu Chí             │ ADS                  │ Migration Evaluator  │ Well-Architected     │
│                      │ (Discovery Service)   │                      │ Migration Lens       │
├──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ Mục đích             │ Khám phá hạ tầng     │ Tính TCO & so sánh   │ Đánh giá kiến trúc  │
│                      │ on-premises          │ chi phí              │ & best practices     │
├──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ Đầu vào              │ Môi trường           │ Dữ liệu server       │ Kiến trúc thiết kế  │
│                      │ on-premises          │ (từ ADS hoặc Excel)  │ mục tiêu             │
├──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ Đầu ra               │ Server inventory,    │ Business case        │ Báo cáo gap,         │
│                      │ dependency map       │ + báo cáo TCO        │ high-risk items      │
├──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ Chi phí              │ Miễn phí (chỉ tính  │ Miễn phí             │ Miễn phí             │
│                      │ Storage lưu data)    │                      │                      │
├──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ Khi nào dùng         │ Đầu giai đoạn Assess │ Giữa-cuối Assess     │ Cuối Assess /        │
│                      │                      │ (trước Mobilize)     │ đầu Mobilize         │
├──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ Ai dùng              │ Migration engineer,  │ Cloud financial       │ Solutions architect, │
│                      │ infrastructure team  │ analyst, CTO, CFO    │ cloud architect      │
├──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ Tích hợp             │ Migration Hub,       │ Migration Hub,       │ Migration Hub        │
│                      │ Migration Evaluator  │ ADS                  │                      │
└──────────────────────┴──────────────────────┴──────────────────────┴──────────────────────┘
```

---

## 🔗 Tích Hợp Với Migration Hub

```
AWS MIGRATION HUB LÀ TRUNG TÂM TỔNG HỢP:

  ADS Discovery Data ──────────────────────┐
                                           │
  Migration Evaluator Reports ─────────────┼──► AWS Migration Hub
                                           │        │
  Manual Application Portfolio ────────────┘        │
                                                    ▼
                                         Application Portfolio
                                         (Danh Mục Ứng Dụng)
                                                    │
                              ┌─────────────────────┼─────────────────────┐
                              ▼                     ▼                     ▼
                        Assign 7R Strategy   Track migration        Generate
                        (Gán chiến lược 7R)  progress               reports
```

---

## ✅ Checklist Assess Phase

### Tuần 1-2: Khám Phá Hạ Tầng

```
□ Triển khai ADS Agentless Collector (nếu môi trường VMware vSphere)
□ Cài ADS Discovery Agent trên các máy chủ quan trọng
□ Chờ 2-4 tuần để thu thập dữ liệu performance đủ mẫu
□ Export server inventory từ ADS Console
□ Review dependency map — tìm các application group (nhóm ứng dụng)
□ Phân loại ứng dụng theo 7R strategy (Retire/Retain/Rehost/Relocate/
  Repurchase/Replatform/Refactor)
□ Đồng bộ dữ liệu ADS lên Migration Hub
```

### Tuần 3-4: Phân Tích Tài Chính

```
□ Chạy Migration Evaluator với dữ liệu ADS
□ Bổ sung thông tin: license costs, maintenance contracts, facilities costs
□ Nhận và review Migration Business Case report
□ Validate right-sizing recommendations với team infrastructure
□ Trình bày kết quả cho stakeholders (ban lãnh đạo, CFO)
□ Phê duyệt ngân sách migration dựa trên TCO analysis
```

### Tuần 5-6: Đánh Giá Kiến Trúc

```
□ Chạy Well-Architected Review với Migration Lens
□ Xác định high-risk items (mục rủi ro cao)
□ Lập remediation plan (kế hoạch khắc phục) cho từng gap
□ Thiết kế target architecture (kiến trúc đích) trên AWS
□ Lập Migration Wave Plan (kế hoạch di chuyển theo đợt)
□ Xây dựng Risk Register (sổ đăng ký rủi ro)
□ Trình bày và phê duyệt Migration Plan
```

---

## 🎤 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Application Discovery Service có mấy cách thu thập dữ liệu? Khác nhau như thế nào?**

> ADS có hai cách:
>
> 1. **Agentless Collector** (Thu thập không cần agent): Là máy ảo OVA cài lên vSphere. Thu thập VM metadata và performance tổng quát. Nhanh triển khai, không cần quyền OS.
>
> 2. **Discovery Agent** (Agent khám phá): Cài trực tiếp lên từng server. Thu thập thông tin sâu hơn: running processes, network connections TCP/UDP, disk I/O. Cho phép vẽ dependency map chính xác hơn.
>
> Thực tế thường dùng cả hai: Agentless để có inventory nhanh, Agent cho các ứng dụng quan trọng cần dependency map chính xác.

---

**Q: Migration Evaluator khác gì so với tự tính TCO bằng Excel?**

> Migration Evaluator cung cấp:
> - **Dữ liệu thực tế** từ ADS (không phải ước tính thủ công)
> - **Database giá AWS** được cập nhật tự động (On-Demand, Reserved, Savings Plans)
> - **Right-sizing tự động** dựa trên utilization thực tế (không phải capacity tối đa)
> - **Báo cáo chuyên nghiệp** có thể trình lên C-level (cấp lãnh đạo)
> - **Tích hợp với Migration Hub** để theo dõi tiến độ
>
> Tự tính bằng Excel dễ bỏ sót chi phí ẩn và không có right-sizing data chính xác.

---

**Q: Well-Architected Migration Lens khác gì so với Well-Architected Framework thông thường?**

> Well-Architected Framework là bộ câu hỏi chung cho bất kỳ workload nào trên AWS.
>
> **Migration Lens** là phần mở rộng tập trung vào:
> - Các câu hỏi **đặc thù giai đoạn migration**: cutover planning, rollback strategy, wave planning
> - **Rủi ro migration cụ thể**: data integrity, dependency management, downtime risk
> - **Ba giai đoạn**: Assess → Mobilize → Migrate & Modernize
>
> Bạn chạy Well-Architected Review với Migration Lens ở cuối giai đoạn Assess để xác định gaps trước khi bắt đầu di chuyển thực sự.

---

### Câu Hỏi Nâng Cao

**Q: Trong dự án migration thực tế, bạn sẽ dùng ADS trong bao lâu trước khi bắt đầu di chuyển?**

> Khuyến nghị chạy ADS tối thiểu **2-4 tuần** để có đủ dữ liệu performance đại diện:
> - Bao gồm ít nhất 1 chu kỳ tháng (monthly cycle) để bắt peak load
> - Bao gồm cả weekday và weekend patterns
> - Đủ mẫu để tính average + P95 utilization cho right-sizing chính xác
>
> Nếu chạy quá ngắn (ví dụ 3 ngày), dữ liệu có thể bỏ sót peak periods → right-sizing sai → EC2 bị undersized sau migration.

---

**Q: Làm thế nào để xử lý khi ADS không thể cài agent trên server (ví dụ server thuộc vendor, không có quyền admin)?**

> Có ba lựa chọn:
> 1. **Agentless Collector** nếu server là VMware VM: Lấy được VM-level data mà không cần OS access
> 2. **Manual inventory**: Yêu cầu vendor cung cấp thông số kỹ thuật (CPU, RAM, OS, application list)
> 3. **Network-level discovery**: Dùng network scanning tools bên ngoài để xác định IP, port, và network flows
>
> Sau đó import manual data vào Migration Evaluator qua Excel template để vẫn có TCO analysis.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Thuộc Module:** 07-discovery-assessment
