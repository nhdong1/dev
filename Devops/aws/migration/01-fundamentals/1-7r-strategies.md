# 7R Migration Strategies — Chiến Lược Di Chuyển 7Rs

> **7Rs** là framework tiêu chuẩn của AWS để phân loại cách di chuyển từng workload. Mỗi "R" có mức độ nỗ lực, rủi ro và lợi ích khác nhau. Đây là kiến thức bắt buộc trong mọi cuộc phỏng vấn về AWS Migration.

## 📚 Mục Lục

1. [Tổng Quan 7Rs](#tổng-quan-7rs)
2. [1. Retire — Ngừng Sử Dụng](#1-retire--ngừng-sử-dụng)
3. [2. Retain — Giữ Nguyên](#2-retain--giữ-nguyên)
4. [3. Rehost — Nâng Và Chuyển](#3-rehost--nâng-và-chuyển)
5. [4. Relocate — Di Chuyển Hypervisor](#4-relocate--di-chuyển-hypervisor)
6. [5. Repurchase — Mua Lại Dạng SaaS](#5-repurchase--mua-lại-dạng-saas)
7. [6. Replatform — Tối Ưu Nền Tảng](#6-replatform--tối-ưu-nền-tảng)
8. [7. Refactor/Re-architect — Tái Kiến Trúc](#7-refactorre-architect--tái-kiến-trúc)
9. [Decision Tree — Cây Quyết Định](#decision-tree--cây-quyết-định)
10. [So Sánh Tổng Hợp](#so-sánh-tổng-hợp)
11. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🗂️ Tổng Quan 7Rs

```
Portfolio Ứng Dụng On-Premises
         │
         ▼
    Phân Tích
    ┌──────────────────────────────────────────┐
    │  Không còn giá trị? ──────────► RETIRE   │
    │  Cần giữ nguyên?   ──────────► RETAIN    │
    │  Di chuyển nhanh?  ──────────► REHOST    │
    │                    ──────────► RELOCATE  │
    │  Đổi sang SaaS?    ──────────► REPURCHASE│
    │  Tối ưu nhẹ?       ──────────► REPLATFORM│
    │  Xây lại hoàn toàn?──────────► REFACTOR  │
    └──────────────────────────────────────────┘
```

---

## 1. Retire — Ngừng Sử Dụng

### Định Nghĩa

**Retire** — Xác định và **ngừng hoàn toàn** các ứng dụng, hệ thống không còn cần thiết. Đây là bước đơn giản nhất nhưng mang lại tiết kiệm ngay lập tức.

### Khi Nào Dùng

- Ứng dụng không còn được sử dụng hoặc user base = 0
- Chức năng đã được thay thế bởi hệ thống khác
- Ứng dụng chạy chỉ để "phòng khi cần" nhưng chưa bao giờ cần
- Chi phí vận hành cao hơn giá trị mang lại

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Thay đổi code | Không |
| Nỗ lực thực hiện | Rất thấp |
| Thời gian | Ngay lập tức |
| Tiết kiệm chi phí | 100% (loại bỏ hoàn toàn) |
| Rủi ro | Thấp (nếu phân tích đúng) |

### Ví Dụ Thực Tế

```
Tình huống: Công ty có 200 ứng dụng internal
→ Phân tích: 30% (60 ứng dụng) không có người dùng trong 12 tháng
→ Hành động: Retire 60 ứng dụng đó
→ Kết quả: Giảm 30% khối lượng migration, tiết kiệm ngay

Ví dụ cụ thể:
- Hệ thống báo cáo cũ (đã thay bằng Tableau)
- API legacy không còn client nào gọi
- Staging server của dự án đã kết thúc
```

### Lưu Ý

> Trước khi Retire, luôn xác nhận với business owner. Đôi khi ứng dụng "ít dùng" có thể là critical trong audit hoặc compliance.

---

## 2. Retain — Giữ Nguyên On-Premises

### Định Nghĩa

**Retain** — Quyết định **giữ nguyên** một số workload on-premises, không di chuyển lên cloud trong đợt này.

### Khi Nào Dùng

- Ứng dụng vừa được đầu tư nâng cấp lớn (chưa hoàn vốn)
- Latency yêu cầu cực thấp, chỉ đạt được với on-premises
- Compliance / regulatory requirements bắt buộc dữ liệu ở một vị trí cụ thể
- Tích hợp chặt với thiết bị phần cứng chuyên biệt
- Kỹ năng cloud chưa đủ để migrate an toàn hiện tại

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Thay đổi code | Không |
| Nỗ lực thực hiện | Rất thấp |
| Tác động đến on-premises | Không đổi |
| Đây là quyết định "tạm thời" | Thường có kế hoạch migrate sau |

### Ví Dụ Thực Tế

```
Tình huống:
- Hệ thống điều khiển máy CNC tích hợp trực tiếp với thiết bị vật lý
  → Retain (không thể cloud-ify phần cứng)

- Hệ thống ERP vừa được upgrade 18 tháng trước, tốn 2 triệu USD
  → Retain 2 năm nữa, sau đó Repurchase sang SaaS

- Database chứa dữ liệu bệnh nhân phải đặt tại Việt Nam theo luật
  → Retain on-premises hoặc xem xét AWS Region phù hợp
```

### Lưu Ý

> **Retain ≠ Bỏ quên.** Vẫn cần kế hoạch rõ ràng cho những workload này: khi nào sẽ migrate, điều kiện để migrate là gì.

---

## 3. Rehost — Lift-and-Shift (Nâng Và Chuyển)

### Định Nghĩa

**Rehost** (còn gọi là **Lift-and-Shift** — Nâng và Chuyển) — Di chuyển ứng dụng lên AWS **nguyên vẹn như on-premises**, không thay đổi kiến trúc hay code.

### Khi Nào Dùng

- Cần di chuyển nhanh, ít thời gian phân tích
- Timeline gấp (ví dụ: data center lease sắp hết hạn)
- Team chưa có nhiều kinh nghiệm cloud
- Muốn di chuyển để thoát khỏi chi phí on-premises, tối ưu sau

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Thay đổi code | Không |
| Nỗ lực thực hiện | Thấp |
| Tốc độ | Nhanh nhất |
| Tiết kiệm chi phí | 20-30% so với on-premises |
| Tối ưu cloud | Chưa tận dụng được tối đa |

### Công Cụ AWS Chính

- **AWS MGN — Application Migration Service** — Tự động rehost server vật lý/ảo lên EC2
- **AWS SMS — Server Migration Service** (cũ, đã thay bằng MGN)

### Cơ Chế Hoạt Động (AWS MGN)

```
On-Premises Server
       │
       ▼ (Cài AWS Replication Agent)
  Liên tục replicate block storage → AWS Staging Area
       │
       ▼ (Test cutover)
  EC2 instance (test) — kiểm tra hoạt động
       │
       ▼ (Production cutover)
  EC2 instance (production) — cắt sang AWS
       │
       ▼ (Finalize)
  Xóa staging, shutdown source server
```

### Ví Dụ Thực Tế

```
Tình huống: 300 server vật lý, hợp đồng data center hết hạn trong 6 tháng

Giải pháp Rehost:
├── Cài AWS Replication Agent lên tất cả 300 server
├── Chờ initial replication hoàn thành (1-3 ngày tuỳ data size)
├── Kiểm tra bằng test cutover
├── Thực hiện production cutover theo lịch (2-3 server/đêm)
└── Toàn bộ di chuyển xong trong 3-4 tháng

Kết quả:
- Tiết kiệm 25% chi phí so với on-premises
- Không downtime production
- Team có thêm thời gian học cloud để tối ưu sau
```

### Hạn Chế

> Rehost giúp thoát on-premises nhưng **chưa tận dụng được** managed services, auto-scaling, hay serverless của AWS. Cần Replatform/Refactor để khai thác tối đa giá trị cloud.

---

## 4. Relocate — Di Chuyển Hypervisor

### Định Nghĩa

**Relocate** — Di chuyển toàn bộ **VMware virtual machines (máy ảo VMware)** từ on-premises sang **VMware Cloud on AWS**, không thay đổi gì về ứng dụng hay hệ điều hành.

### Khi Nào Dùng

- Môi trường hiện tại chạy 100% VMware vSphere
- Muốn di chuyển nhanh mà không thay đổi VM
- Team IT đã quen với VMware, không muốn đào tạo lại
- Cần tích hợp chặt giữa on-premises VMware và AWS

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Thay đổi code | Không |
| Thay đổi OS/VM | Không |
| Yêu cầu | Môi trường VMware vSphere |
| Tốc độ | Nhanh |
| Công cụ | VMware HCX (Hybrid Cloud Extension) |

### Khác Biệt Relocate vs Rehost

```
Rehost:   On-premises Server → EC2 Instance (AWS native)
           Thay đổi: từ bare-metal/hypervisor → EC2

Relocate: VMware VM → VMware Cloud on AWS (vẫn chạy trên VMware)
           Không thay đổi: vẫn là VMware VM, chỉ đổi vị trí
```

### Ví Dụ Thực Tế

```
Công ty có 500 VMware VMs:
→ Không muốn thay đổi gì về vận hành
→ Dùng VMware HCX để "kéo" toàn bộ VMs sang VMware Cloud on AWS
→ Vận hành VMware vCenter quen thuộc, nhưng chạy trên AWS infrastructure
→ Sau đó từng bước migrate sang EC2 native (Rehost) hoặc modernize (Refactor)
```

---

## 5. Repurchase — Chuyển Sang SaaS (Mua Lại)

### Định Nghĩa

**Repurchase** — Thay thế ứng dụng on-premises bằng **dịch vụ SaaS — Software as a Service (Phần Mềm như Dịch Vụ)** tương đương trên cloud.

### Khi Nào Dùng

- Ứng dụng là phần mềm phổ biến (CRM, ERP, email) có giải pháp SaaS tốt hơn
- Chi phí maintain và upgrade tự chạy quá cao
- Muốn nhận tự động các tính năng mới mà không tốn công upgrade
- Phiên bản on-premises sắp hết hỗ trợ (End of Support)

### Ví Dụ Thay Thế Phổ Biến

| On-Premises | → SaaS Thay Thế | Ghi Chú |
| ----------- | --------------- | ------- |
| Microsoft Exchange | → Microsoft 365 (Exchange Online) | Email, Calendar |
| Oracle ERP | → SAP S/4HANA Cloud / Oracle Fusion | ERP |
| Salesforce on-prem | → Salesforce.com | CRM |
| Self-hosted Jira | → Jira Cloud (Atlassian) | Project Management |
| On-prem HR system | → Workday / BambooHR | HR Management |
| Self-hosted WordPress | → WordPress.com / Webflow | CMS |

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Thay đổi code | Không (mua sản phẩm mới) |
| Thay đổi quy trình làm việc | Có thể cần |
| Nỗ lực migrate data | Trung bình |
| Chi phí | Chuyển từ CAPEX → OPEX (subscription) |
| Vendor lock-in | Tăng (phụ thuộc vào SaaS vendor) |

### Thách Thức

```
1. Data Migration: Cần export dữ liệu từ on-premises, import vào SaaS
2. Integration: Tích hợp lại với các hệ thống khác
3. User Training: Người dùng phải làm quen với giao diện mới
4. Customization: SaaS thường ít tùy biến hơn on-premises
5. Data Ownership: Dữ liệu nằm ở vendor, cần kiểm tra GDPR/compliance
```

---

## 6. Replatform — Lift-and-Reshape (Tối Ưu Nền Tảng)

### Định Nghĩa

**Replatform** (còn gọi là **Lift-and-Reshape** — Nâng và Định Hình Lại) — Di chuyển lên AWS và **thực hiện một số thay đổi nhỏ** để tận dụng cloud, nhưng **không thay đổi kiến trúc tổng thể**.

### Nguyên Tắc

> "Giữ nguyên kiến trúc, thay nền tảng — không refactor code business logic"

### Ví Dụ Replatform Phổ Biến

| Trước (On-Premises) | → Sau (AWS) | Lợi Ích |
| -------------------- | ----------- | ------- |
| MySQL tự quản lý | → Amazon RDS MySQL | Tự động backup, patching, HA |
| Tomcat tự quản lý | → AWS Elastic Beanstalk | Tự động deploy, scaling |
| Self-managed Redis | → Amazon ElastiCache | Managed, Multi-AZ tự động |
| Oracle Database | → Amazon RDS Oracle | Giảm admin overhead |
| JBoss / WildFly | → AWS Elastic Beanstalk / ECS | Managed container runtime |

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Thay đổi code | Tối thiểu (chủ yếu config) |
| Nỗ lực thực hiện | Trung bình |
| Tốc độ | Trung bình |
| Tiết kiệm chi phí | 40-60% so với on-premises |
| Lợi ích cloud | Một phần (managed services) |

### Ví Dụ Thực Tế

```
Tình huống: Ứng dụng Java Spring Boot + MySQL on-premises

Replatform solution:
├── App: Đóng gói thành Docker container → chạy trên AWS ECS Fargate
│   (Không sửa code, chỉ containerize)
├── DB: MySQL on-prem → Amazon RDS MySQL Multi-AZ
│   (Connection string thay đổi, không sửa SQL)
└── Session: In-memory → Amazon ElastiCache Redis
    (Thêm Redis client library, thay đổi session config)

Kết quả:
- Không cần quản lý OS, patching
- Tự động failover database
- Auto-scaling ECS tasks theo tải
- Tiết kiệm 50% chi phí ops
```

---

## 7. Refactor/Re-architect — Tái Kiến Trúc

### Định Nghĩa

**Refactor** (còn gọi là **Re-architect** — Tái Kiến Trúc) — **Xây dựng lại kiến trúc** ứng dụng để tận dụng tối đa các dịch vụ cloud-native của AWS.

### Khi Nào Dùng

- Ứng dụng có nhu cầu scale lớn mà monolith không đáp ứng được
- Business cần tốc độ feature delivery cao hơn
- Muốn tận dụng serverless, microservices để giảm chi phí vận hành
- Ứng dụng có architecture cũ khó maintain và mở rộng

### Các Hướng Refactor Phổ Biến

```
Monolith → Microservices:
├── Tách service theo domain
├── Mỗi service độc lập, deploy riêng
└── Giao tiếp qua API Gateway + SQS/SNS

Traditional → Serverless:
├── AWS Lambda thay thế server process
├── API Gateway thay thế web server
└── DynamoDB thay thế relational DB (nếu phù hợp)

Self-managed DB → Fully Managed:
├── Oracle → Amazon Aurora PostgreSQL
├── SQL Server → Amazon Aurora MySQL
└── MongoDB → Amazon DocumentDB
```

### Đặc Điểm

| Thuộc Tính | Giá Trị |
| ---------- | ------- |
| Thay đổi code | Nhiều (có thể viết lại hoàn toàn) |
| Nỗ lực thực hiện | Cao nhất |
| Thời gian | Dài nhất (tháng đến năm) |
| Tiết kiệm chi phí dài hạn | Cao nhất (60-80%) |
| Lợi ích cloud | Tối đa |
| Rủi ro | Cao nhất |

### Ví Dụ Thực Tế

```
Tình huống: Ứng dụng e-commerce monolith, 1 triệu user/ngày
Problem: Black Friday → hệ thống chết vì không scale được

Refactor solution:
├── Catalog Service → Lambda + DynamoDB + CloudFront CDN
├── Order Service → ECS + Aurora + SQS queue
├── Payment Service → Lambda + Step Functions (workflow)
├── Search → Amazon OpenSearch Service
└── API Gateway → Tập trung routing + authentication

Kết quả:
- Scale tự động từ 1,000 đến 1,000,000 request/giây
- 70% giảm chi phí vận hành
- Deploy từng service độc lập, không downtime
- Team nhỏ có thể vận hành hệ thống lớn
```

### Strangler Fig Pattern (Mẫu Bóp Nghẹt)

> Kỹ thuật phổ biến để Refactor dần dần — không phải viết lại toàn bộ ngay:

```
Bước 1: Monolith vẫn chạy đầy đủ
         │
Bước 2: Thêm API Gateway/Router phía trước
         │
Bước 3: Tách từng feature nhỏ ra microservice/Lambda
         → Traffic feature đó chuyển sang service mới
         │
Bước 4: Dần dần tất cả traffic chuyển sang services mới
         │
Bước 5: Monolith "bị bóp nghẹt" và có thể loại bỏ
```

---

## 🌳 Decision Tree — Cây Quyết Định

```
Workload này có còn cần thiết không?
├── Không → RETIRE
└── Có
    │
    Có thể di chuyển lên cloud không?
    ├── Không (regulatory, hardware) → RETAIN
    └── Có
        │
        Là VMware VM và muốn giữ VMware? → RELOCATE
        │
        Không
        │
        Có SaaS tốt hơn thay thế được? → REPURCHASE
        │
        Không
        │
        Có thời gian/nguồn lực để sửa code không?
        ├── Không → REHOST (lift-and-shift)
        └── Có
            │
            Thay đổi nhỏ (managed DB, containerize)? → REPLATFORM
            │
            Cần tối ưu tối đa, kiến trúc lại? → REFACTOR
```

---

## 📊 So Sánh Tổng Hợp

| Chiến Lược | Code Change | Nỗ Lực | Tốc Độ | Tiết Kiệm Chi Phí | Rủi Ro |
| ---------- | ----------- | ------- | ------ | ----------------- | ------ |
| **Retire** | Không | Rất thấp | Ngay | 100% loại bỏ | Thấp |
| **Retain** | Không | Rất thấp | - | 0% | Thấp |
| **Rehost** | Không | Thấp | ⭐⭐⭐⭐⭐ | 20-30% | Thấp |
| **Relocate** | Không | Thấp | ⭐⭐⭐⭐ | 20-30% | Thấp |
| **Repurchase** | Không | Trung bình | ⭐⭐⭐ | Thay đổi mô hình | Trung bình |
| **Replatform** | Tối thiểu | Trung bình | ⭐⭐⭐ | 40-60% | Trung bình |
| **Refactor** | Nhiều | Cao | ⭐⭐ | 60-80% | Cao |

### Phân Phối Điển Hình Trong Dự Án Thực Tế

```
Retire:     10-15%  (loại bỏ workload không cần)
Retain:     10-15%  (giữ nguyên tạm thời)
Rehost:     50-60%  (phần lớn — di chuyển nhanh)
Relocate:    5-10%  (nếu dùng VMware nhiều)
Repurchase:  5-10%  (thay thế bằng SaaS)
Replatform:  5-10%  (tối ưu nhẹ)
Refactor:    5-10%  (chỉ những workload chiến lược)
```

---

## 💼 Ví Dụ Thực Tế — Doanh Nghiệp 500 Nhân Viên

```
Khởi Điểm: 80 ứng dụng, data center lease hết hạn trong 12 tháng

Phân Tích:
├── 10 ứng dụng không ai dùng trong 2 năm → RETIRE (tiết kiệm ngay)
├── 5 ứng dụng điều khiển máy sản xuất → RETAIN (không thể cloud)
├── CRM tự build → REPURCHASE (chuyển sang Salesforce)
├── 50 ứng dụng internal tools → REHOST với AWS MGN (xong trong 6 tháng)
├── 10 ứng dụng dùng MySQL tự chạy → REPLATFORM sang RDS
└── 4 ứng dụng e-commerce chiến lược → REFACTOR (kế hoạch 18 tháng)

Timeline:
├── Tháng 1-2: Retire + Repurchase (giải phóng resource)
├── Tháng 3-8: Rehost 50 ứng dụng (2-3 ứng dụng/tuần)
├── Tháng 6-9: Replatform 10 ứng dụng (song song với Rehost)
└── Tháng 6-24: Refactor 4 ứng dụng chiến lược (sau khi đã lên cloud)

Kết Quả:
├── Thoát data center đúng hạn 12 tháng
├── Tiết kiệm 35% chi phí tổng thể
└── Nền tảng để tiếp tục tối ưu (Replatform/Refactor phần còn lại)
```

---

## 🎓 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Giải thích 7R migration strategies?**

> A: 7Rs là framework phân loại workload khi di chuyển lên cloud:
> - **Retire**: Loại bỏ workload không còn cần thiết
> - **Retain**: Giữ nguyên on-premises do kỹ thuật/quy định
> - **Rehost**: Lift-and-shift không thay đổi, dùng AWS MGN
> - **Relocate**: Di chuyển VMware VM sang VMware Cloud on AWS
> - **Repurchase**: Thay bằng SaaS (Salesforce, SAP...)
> - **Replatform**: Tối ưu nhỏ (MySQL → RDS, code → ECS)
> - **Refactor**: Tái kiến trúc để dùng cloud-native services
>
> Trong thực tế, phần lớn (50-60%) là Rehost để di chuyển nhanh, sau đó dần dần Replatform/Refactor để tối ưu.

**Q: Khi nào chọn Rehost thay vì Refactor?**

> A: Chọn **Rehost** khi:
> - Timeline gấp (data center lease sắp hết)
> - Không có ngân sách/thời gian để viết lại code
> - Workload không có giá trị chiến lược cao
> - Team chưa sẵn sàng với cloud-native
>
> Chọn **Refactor** khi:
> - Workload chiến lược, cần scale lớn
> - Monolith đang là bottleneck cho tốc độ phát triển
> - Muốn tận dụng serverless/managed services để giảm ops overhead
> - Có thời gian và ngân sách đầu tư dài hạn

**Q: Làm thế nào phân loại 300 workload vào 7Rs?**

> A: Quy trình thực tế:
> 1. **Khám phá** bằng AWS Application Discovery Service
> 2. **Phỏng vấn** application owners và business stakeholders
> 3. **Đánh giá** theo các tiêu chí: business value, technical complexity, age, usage
> 4. **Workshop** với team kỹ thuật để validate phân loại
> 5. **Finalize** migration portfolio với phân phối 7Rs

### Câu Hỏi Nâng Cao

**Q: Rủi ro của Refactor và cách giảm thiểu?**

> A: Rủi ro chính:
> - **Scope creep**: Liên tục mở rộng phạm vi → Giới hạn scope rõ ràng, MVP trước
> - **Knowledge gap**: Team thiếu kỹ năng cloud-native → Đào tạo, thuê consultant
> - **Regression bugs**: Refactor gây ra bug mới → Test coverage đầy đủ trước khi refactor
> - **Timeline overrun**: Phức tạp hơn dự kiến → Buffer 50% thời gian, dùng Strangler Fig Pattern
>
> Giảm thiểu: Dùng **Strangler Fig Pattern** — refactor từng phần nhỏ, không viết lại toàn bộ ngay.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
