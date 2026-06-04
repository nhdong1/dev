# AWS Migration Evaluator — Phân Tích TCO & Đề Xuất Phương Án

> **AWS Migration Evaluator** (trước đây là TSO Logic) phân tích dữ liệu hạ tầng on-premises và tạo báo cáo **TCO — Total Cost of Ownership (Tổng Chi Phí Sở Hữu)** chuyên nghiệp, giúp bạn trả lời câu hỏi: *"Nếu chuyển lên AWS, chi phí thực tế thay đổi như thế nào?"*

## 📚 Mục Lục

1. [Migration Evaluator Là Gì?](#migration-evaluator-là-gì)
2. [Quy Trình Sử Dụng](#quy-trình-sử-dụng)
3. [Các Nguồn Dữ Liệu Đầu Vào](#các-nguồn-dữ-liệu-đầu-vào)
4. [Báo Cáo Đầu Ra](#báo-cáo-đầu-ra)
5. [Right-Sizing — Đề Xuất Instance Tối Ưu](#right-sizing)
6. [Mô Hình Chi Phí AWS Trong Evaluator](#mô-hình-chi-phí-aws-trong-evaluator)
7. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
8. [So Sánh Với AWS Pricing Calculator](#so-sánh-với-aws-pricing-calculator)
9. [Hướng Dẫn Sử Dụng](#hướng-dẫn-sử-dụng)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Migration Evaluator Là Gì?

### Vai Trò Trong Hành Trình Migration

```
Giai đoạn ASSESS — Migration Evaluator đóng vai trò:

CTO/CFO hỏi: "Chuyển lên AWS có đắt hơn không? ROI bao lâu?"
         ↓
Bạn cần:
├── Số liệu cụ thể, không chỉ ước đoán
├── So sánh chi phí on-premises thực tế vs AWS
├── Phân tích theo nhiều mô hình mua sắm (On-Demand, Reserved, Savings Plans)
└── Báo cáo đẹp, thuyết phục stakeholders
         ↓
Migration Evaluator giải quyết chính xác những điều này
```

### Lịch Sử: TSO Logic → Migration Evaluator

AWS mua lại công ty **TSO Logic** năm 2019 và tích hợp thành **AWS Migration Evaluator**. TSO Logic nổi tiếng với thuật toán phân tích workload và right-sizing rất chính xác, dựa trên **dữ liệu sử dụng thực tế** thay vì configured capacity.

> **Ý nghĩa thực tế:** Nhiều doanh nghiệp cấu hình server 32 core nhưng chỉ dùng trung bình 15% — đó là 27 core lãng phí. Migration Evaluator phát hiện điều này và đề xuất EC2 instance nhỏ hơn, tiết kiệm chi phí.

---

## 🔄 Quy Trình Sử Dụng

### Luồng Tổng Thể

```
Bước 1: Thu thập dữ liệu (1-4 tuần)
├── Dùng ADS Discovery Agent (tốt nhất)
├── Hoặc export từ RVTools (VMware)
├── Hoặc upload thủ công từ SCOM, Perfmon, TSO Logic collector
└── Thời gian thu thập dài hơn = dữ liệu phân tích chính xác hơn

Bước 2: Import dữ liệu vào Migration Evaluator
├── AWS Console → Migration Evaluator → Get started
└── Chọn nguồn dữ liệu và upload

Bước 3: Cấu hình phân tích
├── Chọn AWS Region đích (target region)
├── Chọn OS license type (BYOL vs License Included)
├── Chọn mô hình mua sắm (On-Demand, RI 1yr, RI 3yr, Savings Plans)
└── Cấu hình DB migration target (Aurora, RDS, ...)

Bước 4: Chạy phân tích
└── Migration Evaluator tự động phân tích (15-30 phút)

Bước 5: Nhận và review báo cáo
├── Báo cáo PDF chuyên nghiệp (~20-50 trang)
├── Breakdown chi phí theo từng workload
└── Quick wins và recommended actions
```

---

## 📥 Các Nguồn Dữ Liệu Đầu Vào

Migration Evaluator linh hoạt — chấp nhận nhiều định dạng dữ liệu khác nhau:

### Nguồn 1: AWS Application Discovery Service (ADS)

```
Cách tích hợp:
├── ADS Discovery Agent chạy 2-4 tuần
├── Migration Evaluator → Import → chọn "Import from ADS"
└── Tự động kéo dữ liệu từ ADS

Ưu điểm: Dữ liệu performance thực tế, không cần export thủ công
Nhược điểm: Phải cài ADS Agent trên từng server
```

### Nguồn 2: RVTools (VMware)

```
RVTools là gì:
├── Công cụ miễn phí để export inventory VMware vCenter
├── Tạo file XLSX với tất cả thông tin VM
└── Rất phổ biến — nhiều doanh nghiệp đã có sẵn

Cách dùng:
├── Chạy RVTools → Export XLSX
├── Migration Evaluator → Import → chọn "Upload from RVTools"
└── Đây là cách nhanh nhất khi dùng VMware

Hạn chế:
└── Chỉ có configured capacity (vCPU, vRAM allocated) — không có actual utilization
    → Right-sizing kém chính xác hơn so với ADS
```

### Nguồn 3: Microsoft System Center Operations Manager — SCOM

```
Phù hợp cho: Môi trường Windows Server nặng với SCOM đang chạy
Dữ liệu: Performance metrics từ SCOM (CPU, RAM, Disk utilization)
Export: SCOM Performance data → CSV → import vào Migration Evaluator
```

### Nguồn 4: Thủ Công (CSV Template)

```
Khi không có công cụ nào ở trên:
├── Download Migration Evaluator CSV template
├── Điền thông tin thủ công: hostname, OS, CPU, RAM, disk, estimated utilization
└── Upload lên Migration Evaluator

Lưu ý: Độ chính xác phụ thuộc vào chất lượng dữ liệu nhập tay
```

### Nguồn 5: TSO Logic Collector

```
Dành cho: Khách hàng đã dùng TSO Logic trước khi AWS mua lại
Cách dùng: Export dữ liệu từ TSO Logic → import trực tiếp
```

---

## 📊 Báo Cáo Đầu Ra

### Cấu Trúc Báo Cáo

Migration Evaluator tạo báo cáo **PDF chuyên nghiệp** (thường 20-50 trang), thường được AWS Solutions Architect hỗ trợ trình bày với ban lãnh đạo:

#### 1. Executive Summary (Tóm Tắt Cho Lãnh Đạo)

```
Ví dụ Executive Summary:
┌─────────────────────────────────────────────────────┐
│ Chi phí on-premises (3 năm):    $4,200,000          │
│ Chi phí AWS (3 năm):            $1,890,000          │
│ Tiết kiệm ước tính:             $2,310,000 (55%)    │
│ Thời gian hoàn vốn (ROI):       14 tháng            │
└─────────────────────────────────────────────────────┘
```

#### 2. Current State Analysis (Phân Tích Hiện Trạng)

```
Phân tích hiện trạng on-premises:
├── Tổng số server: 247
├── Server đang idle/underutilized: 68 (28%) → ứng viên Retire/Consolidate
├── Server utilization trung bình: CPU 22%, RAM 41%
├── License đang sử dụng:
│   ├── Windows Server: 185 licenses
│   ├── SQL Server Enterprise: 24 licenses
│   └── RHEL: 62 subscriptions
└── Total on-premises cost breakdown:
    ├── Hardware (khấu hao): $780,000/năm
    ├── Facilities (điện, làm mát): $340,000/năm
    ├── Software licenses: $620,000/năm
    └── Operations (nhân lực): $890,000/năm
```

#### 3. AWS Recommendation (Đề Xuất AWS)

```
Right-sizing recommendations:
├── Server "db-oracle-01" (on-prem: 32 vCPU, 256 GB RAM, avg 18% CPU)
│   └── Đề xuất: r6i.8xlarge (32 vCPU, 256 GB RAM) hoặc r6i.4xlarge nếu migrate sang Aurora
│
├── Server "web-app-01" đến "web-app-10" (on-prem: 8 vCPU, 32 GB RAM, avg 12% CPU)
│   └── Đề xuất: c6i.2xlarge (8 vCPU, 16 GB RAM) — right-sized
│
└── "analytics-01" (on-prem: 64 vCPU, 512 GB RAM, chỉ dùng ban đêm)
    └── Đề xuất: Spot Instance (r6i.16xlarge) — tiết kiệm 70%
```

#### 4. Cost Comparison (So Sánh Chi Phí)

```
Bảng so sánh chi phí (ví dụ):

                        On-Premises    AWS On-Demand    AWS 1yr RI    AWS 3yr RI
Tính toán (EC2)         $450,000       $380,000         $260,000      $195,000
Database (RDS/Aurora)   $620,000       $290,000         $198,000      $148,000
Storage                 $180,000       $95,000          $95,000       $95,000
Networking              $120,000       $85,000          $85,000       $85,000
Operations/People       $890,000       $445,000*        $445,000*     $445,000*
─────────────────────────────────────────────────────────────────────────────
Tổng/năm               $2,260,000     $1,295,000       $1,083,000    $968,000
Tiết kiệm/năm           -              $965,000 (43%)   $1,177,000(52%)  $1,292,000(57%)

* Operations giảm vì bớt quản lý hardware, patching hệ điều hành (managed services)
```

#### 5. Quick Wins (Cơ Hội Tiết Kiệm Ngay)

```
Quick wins Migration Evaluator thường tìm ra:
├── Retire 68 servers đang idle → tiết kiệm $180,000/năm
├── Chuyển 24 SQL Server Enterprise sang Aurora → tiết kiệm $240,000/năm
├── Sử dụng Spot Instance cho batch workloads → tiết kiệm $85,000/năm
└── Xóa snapshots/AMI cũ, right-size storage → tiết kiệm $30,000/năm
```

---

## 📐 Right-Sizing — Đề Xuất Instance Tối Ưu {#right-sizing}

### Right-Sizing Là Gì?

**Right-sizing** — Định Cỡ Đúng — là quá trình chọn EC2 instance type phù hợp nhất với **nhu cầu thực tế** của workload, không phải theo configured capacity.

```
Vấn đề thường gặp (over-provisioning):
On-premises server: 32 vCPU, 256 GB RAM
Sử dụng thực tế:   CPU avg 18%, peak 45%; RAM avg 35%

Nếu migrate thẳng (lift-and-shift same size):
→ r6i.8xlarge: $2,688/tháng (dư thừa, lãng phí)

Sau khi right-size:
→ r6i.4xlarge (16 vCPU, 128 GB RAM): $1,344/tháng
→ Tiết kiệm 50% mà vẫn đủ capacity cho peak load
```

### Thuật Toán Right-Sizing Của Migration Evaluator

Migration Evaluator (kế thừa từ TSO Logic) dùng thuật toán phân tích:

```
1. Thu thập performance data (CPU, RAM, Disk) theo thời gian
2. Tính toán:
   ├── Average utilization (mức dùng trung bình)
   ├── P95 (percentile 95) — mức dùng 95% thời gian
   └── Peak (đỉnh điểm cao nhất)
3. Right-size dựa trên P95 (không phải peak — tránh over-provisioning)
4. Thêm buffer 20-30% cho growth (tăng trưởng tương lai)
5. Khớp với EC2 instance family phù hợp (compute, memory, storage optimized)
```

### Các Instance Family Phổ Biến Được Đề Xuất

| Loại Workload | Instance Family | Ví Dụ |
| ------------- | --------------- | ----- |
| **General purpose** | m6i, m7i | Web servers, application servers |
| **Compute intensive** | c6i, c7i | Batch processing, HPC |
| **Memory intensive** | r6i, r7i | Database, caching, SAP |
| **Storage optimized** | i4i | High I/O databases |
| **GPU** | g4dn, p4d | ML training, rendering |

---

## 💰 Mô Hình Chi Phí AWS Trong Evaluator

Migration Evaluator phân tích chi phí theo nhiều mô hình mua sắm để tìm phương án tối ưu:

### On-Demand (Theo Yêu Cầu)

```
├── Trả theo giờ, không cam kết
├── Linh hoạt nhất — có thể dừng bất cứ lúc nào
├── Chi phí cao nhất
└── Phù hợp: workload không ổn định, dùng ngắn hạn
```

### Reserved Instances — RI (Phiên Bản Đặt Trước)

```
├── Cam kết 1 năm: tiết kiệm ~30-40% so với On-Demand
├── Cam kết 3 năm: tiết kiệm ~50-60% so với On-Demand
├── Standard RI: không đổi được instance type
├── Convertible RI: đổi được instance type (ít tiết kiệm hơn)
└── Phù hợp: production workload ổn định, biết sẽ dùng lâu dài
```

### Savings Plans (Kế Hoạch Tiết Kiệm)

```
├── Cam kết mức chi tiêu ($/giờ) thay vì instance cụ thể
├── Compute Savings Plans: áp dụng cho mọi instance, region, OS
├── EC2 Instance Savings Plans: cố định instance family + region
└── Linh hoạt hơn RI, tiết kiệm tương đương
```

### Spot Instances (Phiên Bản Gián Đoạn)

```
├── Dùng EC2 capacity dư thừa của AWS
├── Tiết kiệm tới 70-90% so với On-Demand
├── Có thể bị thu hồi với 2 phút báo trước
└── Phù hợp: batch jobs, rendering, ML training, stateless workloads

Migration Evaluator tự động xác định workload nào phù hợp Spot
```

### BYOL vs License Included (Tự Mang License vs Có Sẵn)

```
BYOL (Bring Your Own License — Tự Mang License):
├── Dùng lại Windows/SQL Server license đang có
├── Áp dụng với EC2 Dedicated Host hoặc Dedicated Instance
└── Phù hợp khi: đã có Software Assurance với Microsoft

License Included (License Đi Kèm):
├── AWS tính thêm phí license trong giá EC2/RDS
├── Đơn giản, không cần quản lý license
└── Thường đắt hơn BYOL nếu bạn đã có license
```

> Migration Evaluator so sánh cả hai phương án và đề xuất phương án tiết kiệm hơn cho từng workload.

---

## 📋 Ví Dụ Thực Tế

### Case Study: Công Ty Thương Mại Điện Tử 200 Server

```
Bối cảnh:
├── 200 server on-premises (VMware vCenter)
├── Lease data center sắp hết hạn (18 tháng)
├── Chi phí hiện tại: $2.4M/năm
└── Mục tiêu: quyết định có migrate AWS không, và tiết kiệm bao nhiêu

Thu thập dữ liệu:
├── Cài Agentless Connector trên vCenter → 2 tuần thu thập
└── Import vào Migration Evaluator

Kết quả báo cáo:
├── 42 server underutilized (< 5% CPU avg) → ứng viên Retire/Consolidate
├── Right-sizing đề xuất: giảm từ 200 → 145 EC2 instances (right-sized)
├── Quick win: 12 SQL Server → Amazon Aurora (tiết kiệm $380,000/năm)
│
│ Chi phí so sánh (per năm):
│   On-premises:          $2,400,000
│   AWS On-Demand:        $1,650,000 (-31%)
│   AWS với 1yr RI:       $1,120,000 (-53%)
│   AWS với 3yr RI:       $890,000   (-63%)
│
└── Quyết định: Migrate, commit 1yr RI → tiết kiệm $1.28M/năm
    ROI (thời gian hoàn vốn chi phí migration): ~8 tháng
```

---

## ⚖️ So Sánh Với AWS Pricing Calculator

| Tiêu Chí | Migration Evaluator | AWS Pricing Calculator |
| -------- | ------------------- | ---------------------- |
| **Mục đích** | Phân tích migration TCO, right-sizing | Ước tính chi phí kiến trúc mới |
| **Input** | Dữ liệu on-premises thực tế | Bạn tự nhập config |
| **Right-sizing** | ✅ Tự động từ performance data | ❌ Không có |
| **So sánh on-prem vs AWS** | ✅ Có | ❌ Không |
| **Báo cáo PDF** | ✅ Chuyên nghiệp | ❌ Chỉ có web export |
| **Phù hợp cho** | Migration assessment | Thiết kế kiến trúc mới |
| **Chi phí** | Miễn phí | Miễn phí |

> **Quy tắc:** Đang lên kế hoạch migrate → dùng **Migration Evaluator**. Đang thiết kế kiến trúc cloud mới từ đầu → dùng **AWS Pricing Calculator**.

---

## 🚀 Hướng Dẫn Sử Dụng

### Bắt Đầu Với Migration Evaluator

```
Bước 1: Truy cập Migration Evaluator
→ AWS Console → Migration & Transfer → Migration Evaluator
→ hoặc: migration-evaluator.amazonaws.com

Bước 2: Chọn phương thức import dữ liệu
→ "Import from ADS" (nếu đã cài ADS Agent)
→ "Upload from RVTools" (nếu dùng VMware, nhanh nhất)
→ "Upload manually" (CSV template)

Bước 3: Cấu hình phân tích
→ Target Region: ap-southeast-1 (hoặc region mong muốn)
→ License preference: License Included hoặc BYOL
→ Purchase options: bao gồm các option muốn so sánh

Bước 4: Chạy phân tích
→ Click "Run analysis" → đợi 15-30 phút

Bước 5: Review và export báo cáo
→ Review các recommendation
→ Export PDF để trình bày với stakeholders
→ Optionally: request AWS Solutions Architect review (free với business/enterprise support)
```

### Lời Khuyên Để Báo Cáo Chính Xác Nhất

```
1. Thu thập dữ liệu ít nhất 2 tuần (tốt nhất là 4-8 tuần)
   → Bắt được cả chu kỳ peak (cuối tháng, Black Friday...)

2. Bao gồm cả chi phí ẩn on-premises:
   → Điện và làm mát (thường bị bỏ qua)
   → Chi phí con người quản trị hardware
   → Chi phí downtime dự kiến

3. Cân nhắc License optimization:
   → SQL Server Enterprise → Aurora có thể tiết kiệm 70%+
   → Windows Server BYOL nếu có Software Assurance

4. Xem xét managed services:
   → RDS thay vì EC2 tự cài MySQL
   → ElastiCache thay vì EC2 tự cài Redis
   → Loại bỏ chi phí vận hành không core
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: AWS Migration Evaluator là gì và nó khác gì với AWS Pricing Calculator?**

> Migration Evaluator phân tích dữ liệu hạ tầng on-premises thực tế (từ ADS, RVTools...) để tạo báo cáo TCO so sánh chi phí on-premises vs AWS, kèm đề xuất right-sizing tự động. Nó phù hợp khi bạn đang lên kế hoạch migrate. AWS Pricing Calculator ngược lại — bạn tự nhập cấu hình kiến trúc cloud mong muốn để ước tính chi phí. Không có right-sizing tự động, không so sánh on-premises. Phù hợp khi thiết kế hệ thống mới.

**Q: Right-sizing trong Migration Evaluator hoạt động như thế nào?**

> Migration Evaluator thu thập performance data (CPU, RAM, Disk I/O) theo thời gian, thường 2-4 tuần. Sau đó tính percentile 95 (P95) — mức sử dụng trong 95% thời gian — thêm buffer 20-30% cho growth, rồi khớp với EC2 instance type phù hợp nhất. Kết quả: server đang cấu hình 32 vCPU nhưng chỉ dùng P95 là 8 vCPU sẽ được đề xuất instance nhỏ hơn, tiết kiệm 50-60% chi phí.

**Q: Khi nào nên dùng Migration Evaluator sớm trong dự án?**

> Ngay trong giai đoạn Assess — trước khi quyết định migrate. Migration Evaluator tạo business case (luận cứ kinh doanh) với số liệu cụ thể, giúp thuyết phục ban lãnh đạo. Lý tưởng nhất là chạy sau 2-4 tuần thu thập dữ liệu ADS, trước khi bước vào giai đoạn Mobilize. Nếu báo cáo cho thấy migration không tiết kiệm chi phí, đây là thời điểm tốt để điều chỉnh chiến lược.

**Q: Migration Evaluator có hỗ trợ phân tích license không?**

> Có. Migration Evaluator so sánh chi phí giữa License Included (AWS tính thêm phí Windows/SQL Server trong giá EC2) và BYOL (Bring Your Own License — dùng lại license đang có với Dedicated Host). Với doanh nghiệp có Software Assurance với Microsoft, BYOL thường rẻ hơn đáng kể. Evaluator cũng đề xuất migrate SQL Server Enterprise sang Amazon Aurora để tiết kiệm tới 70% chi phí database.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
