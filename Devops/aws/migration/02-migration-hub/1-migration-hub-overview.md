# AWS Migration Hub — Theo Dõi Tiến Trình Tập Trung

> **AWS Migration Hub** là dịch vụ miễn phí cho phép theo dõi tiến trình di chuyển của tất cả máy chủ và ứng dụng từ **một bảng điều khiển duy nhất**, bất kể bạn đang dùng công cụ migration nào.

## 📚 Mục Lục

1. [Migration Hub Là Gì?](#migration-hub-là-gì)
2. [Tính Năng Chính](#tính-năng-chính)
3. [Migration Hub Orchestrator](#migration-hub-orchestrator)
4. [Migration Hub Refactor Spaces](#migration-hub-refactor-spaces)
5. [Tích Hợp Với Công Cụ Khác](#tích-hợp-với-công-cụ-khác)
6. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
7. [Hướng Dẫn Sử Dụng Thực Tế](#hướng-dẫn-sử-dụng-thực-tế)
8. [Hạn Chế Cần Biết](#hạn-chế-cần-biết)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Migration Hub Là Gì?

### Vấn Đề Mà Migration Hub Giải Quyết

Khi di chuyển môi trường doanh nghiệp, bạn thường dùng **nhiều công cụ song song**:

```
Migration thực tế:
├── 200 server dùng AWS MGN (Application Migration Service)
├── 15 database dùng AWS DMS (Database Migration Service)
├── 50 TB file dùng AWS DataSync
└── 30 server legacy dùng công cụ đối tác (CloudEndure, RiverMeadow...)

Vấn đề: Phải mở 4-5 console khác nhau để biết tiến trình tổng thể
```

**Migration Hub giải quyết bằng cách:**

- Tổng hợp dữ liệu từ tất cả công cụ migration vào **một nơi duy nhất**
- Cho phép nhóm các server thành **Application Groups (Nhóm Ứng Dụng)** để theo dõi theo business unit
- Hiển thị trạng thái migration của từng server: Not Started → In Progress → Migrated

### Phạm Vi Dịch Vụ

| Điểm | Mô Tả |
| ----- | ----- |
| **Miễn phí** | Migration Hub hoàn toàn không tính phí; chỉ trả phí cho công cụ tích hợp (MGN, DMS...) |
| **Home Region** | Phải chọn **một AWS Region** làm Home Region — dữ liệu migration sẽ lưu ở đây |
| **Không thực hiện migration** | Migration Hub chỉ theo dõi, không thực hiện di chuyển dữ liệu |

---

## 🛠️ Tính Năng Chính

### 1. Dashboard Theo Dõi Tiến Trình (Progress Dashboard)

Giao diện trực quan hiển thị:

```
Application: "E-commerce Platform"
├── Server: web-app-01     → ✅ Migrated (EC2 i-0abc123)
├── Server: web-app-02     → 🔄 In Progress (45% replicated)
├── Server: cache-redis-01 → ✅ Migrated (ElastiCache)
├── Server: db-mysql-01    → 🔄 In Progress (DMS task running)
└── Server: lb-nginx-01    → ⏳ Not Started

Tổng tiến trình: 2/5 completed (40%)
```

**Các trạng thái migration:**

| Trạng Thái | Ý Nghĩa |
| ---------- | ------- |
| `Not Started` | Chưa bắt đầu |
| `In Progress` | Đang thực hiện (replication hoặc migration đang chạy) |
| `Migrated` | Đã di chuyển thành công |

### 2. Application Groups (Nhóm Ứng Dụng)

Migration Hub cho phép nhóm các server theo **logic nghiệp vụ** thay vì chỉ nhìn từng server riêng lẻ.

```
Ví dụ Application Groups:
├── "HR System"       → 8 servers (web, app, db, cache, monitoring...)
├── "CRM Platform"    → 12 servers
├── "ERP Core"        → 25 servers
└── "Internal Tools"  → 5 servers

Mục đích:
├── Theo dõi tiến trình theo ứng dụng (business-friendly)
├── Lập kế hoạch migration waves (đợt di chuyển)
└── Xác định order di chuyển dựa trên dependencies
```

### 3. Import Data (Nhập Dữ Liệu)

Migration Hub hỗ trợ nhập dữ liệu inventory từ nhiều nguồn:

```
Nguồn dữ liệu hỗ trợ:
├── AWS Application Discovery Service (ADS) — tự động
├── File CSV/JSON — upload thủ công
├── RVTools (VMware inventory tool) — phổ biến nhất
└── AWS Partner tools — tích hợp API
```

**Lợi ích của import thủ công:** Ngay cả khi không cài ADS, bạn vẫn có thể tạo inventory bằng cách export từ VMware vCenter qua RVTools rồi import vào Migration Hub.

---

## ⚙️ Migration Hub Orchestrator

### Orchestrator Là Gì?

**Migration Hub Orchestrator** — Trình Điều Phối Migration — là tính năng tự động hóa các bước migration thành **workflow có thứ tự**.

```
Vấn đề khi không có Orchestrator:
├── Kỹ sư phải theo dõi thủ công: "Server A xong chưa? → chạy script B → chờ DMS xong → notify team"
├── Dễ bỏ sót bước
├── Khó lặp lại cho nhiều server
└── Thiếu visibility khi có lỗi

Orchestrator giải quyết bằng workflow tự động:
Step 1: [Launch replication agent] → auto wait for completion
Step 2: [Run pre-migration tests]  → auto verify
Step 3: [Trigger cutover]          → scheduled or manual approval
Step 4: [Post-migration validation] → auto health checks
Step 5: [Notify stakeholders]      → SNS/email notification
```

### Pre-built Templates (Mẫu Workflow Có Sẵn)

AWS cung cấp template sẵn cho các kịch bản phổ biến:

| Template | Mô Tả |
| -------- | ----- |
| **Rehost (MGN)** | Lift-and-shift server dùng AWS MGN |
| **Replatform SAP** | Di chuyển SAP workload lên AWS |
| **Rehost VMware** | Di chuyển VMware VMs |

### Cách Orchestrator Hoạt Động

```
Cấu trúc một Orchestrator Workflow:

Workflow: "Migrate Web Tier — Wave 1"
├── Step Group 1: Pre-migration checks
│   ├── Step 1.1: Verify ADS agent running ✅
│   ├── Step 1.2: Validate network connectivity ✅
│   └── Step 1.3: Take snapshot of source ✅
├── Step Group 2: Replication
│   ├── Step 2.1: Start MGN replication agent 🔄
│   └── Step 2.2: Wait for initial sync complete ⏳
├── Step Group 3: Test cutover
│   ├── Step 3.1: Launch test instance ⏳
│   └── Step 3.2: Manual approval: "Test OK?" ⏳
└── Step Group 4: Production cutover
    ├── Step 4.1: Cutover to AWS ⏳
    └── Step 4.2: Decommission on-premises ⏳
```

**Tích hợp với:**
- **AWS Lambda** — chạy custom automation scripts
- **AWS Systems Manager** — chạy SSM Run Command trên server
- **Amazon SNS** — gửi thông báo khi bước hoàn thành

---

## 🏗️ Migration Hub Refactor Spaces

### Refactor Spaces Là Gì?

**Migration Hub Refactor Spaces** là môi trường quản lý quá trình hiện đại hóa (modernization) ứng dụng monolith thành microservices theo **Strangler Fig Pattern** — Kỹ Thuật Chuyển Đổi Dần Dần.

> **Strangler Fig Pattern:** Như cây sung bóp chết cây chủ dần dần, bạn xây dựng microservices mới bên cạnh monolith, chuyển traffic dần dần sang, và cuối cùng "tắt" monolith khi đã thay thế toàn bộ.

### Cách Refactor Spaces Hoạt Động

```
Trước khi có Refactor Spaces:
├── Monolith xử lý 100% traffic
├── Muốn tách module "Order Service" thành microservice
├── Phải tự quản lý routing, rollback, traffic splitting
└── Rủi ro cao, phức tạp

Với Refactor Spaces:
├── Tạo "Environment" (môi trường) trong Refactor Spaces
├── Tạo "Application" đại diện cho monolith
├── Tạo "Service" cho microservice mới (Lambda hoặc ECS)
├── Refactor Spaces tự động tạo:
│   ├── Amazon API Gateway — định tuyến request
│   ├── Amazon VPC — mạng cô lập
│   └── Resource-based policies — bảo mật
└── Chuyển traffic dần từ monolith → microservice theo %
```

### Khi Nào Dùng Refactor Spaces

- Ứng dụng monolith lớn cần Refactor từng phần nhỏ
- Team muốn thực hiện **Strangler Fig** một cách có kiểm soát
- Cần A/B testing traffic splitting khi deploy microservice mới

---

## 🔗 Tích Hợp Với Công Cụ Khác

### Công Cụ AWS Gốc (Native Integration)

| Công Cụ | Loại | Dữ Liệu Gửi Về Hub |
| ------- | ---- | ------------------- |
| **AWS MGN** (Application Migration Service) | Application migration | Trạng thái replication, cutover |
| **AWS DMS** (Database Migration Service) | Database migration | Tiến trình task, số record migrated |
| **AWS SMS** (Server Migration Service — cũ) | Server migration | Trạng thái (deprecated, dùng MGN thay thế) |
| **AWS Application Discovery Service** | Discovery | Inventory, dependency data |

### Công Cụ Đối Tác (Partner Integration)

Migration Hub có **Partner ecosystem** rộng lớn:

```
Ví dụ Partner tools tích hợp Migration Hub:
├── CloudEndure (nay là AWS MGN) — server replication
├── Carbonite Migrate — physical server migration
├── RiverMeadow — VMware migration
├── Turbonomic — optimization và right-sizing
└── Tự phát triển qua Migration Hub API
```

---

## 📐 Khái Niệm Cốt Lõi

### Home Region (Region Gốc)

```
Quan trọng: Phải cài đặt trước khi dùng Migration Hub

├── Chỉ chọn một lần — KHÔNG thay đổi được sau khi cài
├── Tất cả dữ liệu migration (server inventory, progress) lưu ở Home Region
├── Phải tạo ADS Collector cùng Home Region
└── Thường chọn: ap-southeast-1 (Singapore) hoặc us-east-1 (N. Virginia)
```

> **Lưu ý thực tế:** Chọn Home Region gần nhất với môi trường on-premises để giảm độ trễ (latency) khi agent gửi dữ liệu về.

### Migration Status Updates

Migration Hub nhận cập nhật trạng thái qua **Migration Hub API** — các công cụ tích hợp gọi API này để báo cáo tiến trình.

```
Luồng cập nhật trạng thái:
AWS MGN replication agent
        ↓ gửi status update
AWS MGN backend
        ↓ gọi Migration Hub API
Migration Hub
        ↓ hiển thị
Dashboard: "web-app-01 → 67% replicated"
```

---

## 🚀 Hướng Dẫn Sử Dụng Thực Tế

### Thiết Lập Migration Hub (Lần Đầu)

```
Bước 1: Chọn Home Region
→ AWS Console → Migration Hub → Settings → Set Home Region
→ Ví dụ: ap-southeast-1

Bước 2: Import inventory (3 cách)
   Cách A — Tự động qua ADS:
   → Cài ADS Connector (VMware) hoặc ADS Agent (physical)
   → ADS tự động đồng bộ dữ liệu lên Migration Hub

   Cách B — Import thủ công CSV:
   → Migration Hub → Discover → Import → Download template
   → Điền thông tin server → Upload CSV

   Cách C — Import từ RVTools:
   → Export từ vCenter bằng RVTools → lưu dưới dạng XLSX
   → Migration Hub → Discover → Import → chọn file RVTools

Bước 3: Tạo Application Groups
→ Migration Hub → Discover → Servers → Chọn servers → Create application
→ Đặt tên theo business unit (ví dụ: "ERP System - Web Tier")

Bước 4: Bắt đầu migration tools (MGN, DMS...)
→ Các tools tự động gửi status về Migration Hub

Bước 5: Theo dõi qua Dashboard
→ Migration Hub → Migrate → Applications → xem tiến trình
```

### Workflow Thực Tế Cho Dự Án 300 Server

```
Tuần 1-2:
├── Cài ADS Agent/Connector để thu thập dữ liệu
└── Import inventory vào Migration Hub

Tuần 3-4:
├── Phân tích dependencies từ dữ liệu ADS
├── Nhóm server thành Application Groups (10-15 nhóm)
└── Xác định thứ tự Migration Waves

Tuần 5+:
├── Wave 1: Migrate 20 servers ít phụ thuộc nhất
├── Theo dõi qua Migration Hub Dashboard
├── Review → Wave 2
└── Tiếp tục đến khi hoàn thành
```

---

## ⚠️ Hạn Chế Cần Biết

| Hạn Chế | Chi Tiết |
| ------- | -------- |
| **Home Region cố định** | Một khi đã chọn, không thể đổi — chọn cẩn thận |
| **Chỉ theo dõi, không thực hiện** | Migration Hub không di chuyển dữ liệu — cần dùng MGN, DMS... |
| **Phụ thuộc vào tool tích hợp** | Nếu dùng công cụ không tích hợp Migration Hub, phải cập nhật thủ công |
| **Giới hạn server** | Soft limit: 25,000 servers mỗi Home Region (có thể tăng qua Support) |
| **Không real-time** | Cập nhật trạng thái có độ trễ vài phút tùy công cụ tích hợp |

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: AWS Migration Hub là gì và nó làm được gì?**

> Migration Hub là dịch vụ miễn phí của AWS cho phép theo dõi tiến trình migration của tất cả server và ứng dụng từ một bảng điều khiển duy nhất. Nó tổng hợp dữ liệu từ các công cụ như MGN, DMS, và partner tools. Migration Hub không thực hiện migration — nó chỉ là "trung tâm quan sát".

**Q: Migration Hub Orchestrator khác gì với Migration Hub thường?**

> Migration Hub cơ bản chỉ theo dõi trạng thái. Orchestrator thêm khả năng tự động hóa: tạo workflow có thứ tự với step groups, tích hợp Lambda và Systems Manager để chạy script tự động, và quản lý approval gates (điểm cần phê duyệt thủ công). Dùng Orchestrator khi muốn chuẩn hóa quy trình migration lặp đi lặp lại.

**Q: Home Region trong Migration Hub nghĩa là gì? Có thể đổi không?**

> Home Region là AWS Region nơi Migration Hub lưu trữ toàn bộ dữ liệu migration (inventory, progress, history). Đây là thiết lập một lần và **không thể thay đổi** sau khi đã cài đặt. Cần chọn cẩn thận — thường chọn region gần nhất với môi trường on-premises hoặc region chính của dự án AWS.

**Q: Bạn có thể dùng Migration Hub mà không có ADS không?**

> Có thể. Migration Hub hỗ trợ import dữ liệu thủ công qua CSV hoặc từ RVTools (VMware). Tuy nhiên, dữ liệu này sẽ thiếu thông tin về dependencies và performance metrics theo thời gian — là những thứ chỉ ADS Agent mới thu thập được tự động.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
