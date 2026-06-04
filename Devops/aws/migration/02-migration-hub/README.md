# AWS Migration Hub & Discovery — Tổng Quan

> Trước khi di chuyển bất kỳ server nào, bạn phải trả lời được: **"Tôi đang có gì? Nó phụ thuộc vào đâu? Chi phí bao nhiêu?"** — Migration Hub, ADS và Migration Evaluator giúp trả lời đúng ba câu hỏi này.

## 📚 Mục Lục (Table of Contents)

1. [Tại Sao Cần Discovery Trước?](#tại-sao-cần-discovery-trước)
2. [Ba Công Cụ Trong Nhóm Này](#ba-công-cụ-trong-nhóm-này)
3. [Luồng Làm Việc Điển Hình](#luồng-làm-việc-điển-hình)
4. [Điều Hướng Tài Liệu](#điều-hướng-tài-liệu)

---

## 🎯 Tại Sao Cần Discovery Trước?

Phần lớn dự án migration thất bại không phải vì kỹ thuật kém, mà vì **không hiểu đủ môi trường hiện tại**:

```
Tình huống thực tế:
├── Team di chuyển server A lên AWS
├── Nhưng server A phụ thuộc ngầm vào server B (không ai biết)
├── Sau cutover: ứng dụng lỗi, phải rollback khẩn cấp
└── Hậu quả: mất uy tín, phát sinh chi phí, delay 2-4 tuần
```

**Discovery đúng nghĩa giải quyết vấn đề này bằng cách:**

- Tự động khám phá tất cả server, process, kết nối mạng
- Vẽ bản đồ phụ thuộc (dependency map) giữa các ứng dụng
- Thu thập dữ liệu hiệu năng (CPU, RAM, disk, network) theo thời gian thực
- Ước tính chi phí AWS tương đương so với on-premises

---

## 🛠️ Ba Công Cụ Trong Nhóm Này

### 1. AWS Migration Hub

**Vai trò:** Bảng điều khiển trung tâm — nơi bạn theo dõi **tất cả** hoạt động migration tại một chỗ.

```
Migration Hub không thực hiện migration —
nó là "trung tâm chỉ huy" tổng hợp dữ liệu từ:
├── AWS MGN (Application Migration Service)
├── AWS DMS (Database Migration Service)
├── AWS Server Migration Service (SMS — cũ)
└── Các công cụ của đối tác AWS (Partner tools)
```

**Tính năng chính:**
- **Theo dõi tiến trình (Progress Tracking):** Xem trạng thái từng server đang ở đâu trong hành trình migration
- **Migration Hub Refactor Spaces:** Môi trường để hiện đại hóa ứng dụng monolith thành microservices
- **Migration Hub Orchestrator:** Tự động hóa và điều phối các bước migration theo workflow

Tài liệu chi tiết: [1-migration-hub-overview.md](./1-migration-hub-overview.md)

---

### 2. AWS Application Discovery Service — ADS

**Vai trò:** Tự động khám phá hạ tầng on-premises — thu thập dữ liệu server, process và kết nối mạng.

```
ADS cung cấp 2 phương thức thu thập dữ liệu:

Agentless Discovery (Khám Phá Không Cần Agent):
├── Dùng AWS Agentless Discovery Connector
├── Cài dưới dạng OVA (Open Virtual Appliance) trên VMware vCenter
├── Thu thập metadata: hostname, IP, OS, số CPU, RAM, disk
└── Không thu thập dữ liệu process hay kết nối mạng chi tiết

Agent-based Discovery (Khám Phá Dùng Agent):
├── Cài AWS Discovery Agent trực tiếp trên từng server
├── Hỗ trợ: Windows (2008+), Linux (Ubuntu, RHEL, CentOS, SUSE)
├── Thu thập: process đang chạy, kết nối TCP, hiệu năng theo thời gian thực
└── Tạo bản đồ phụ thuộc (dependency map) chi tiết nhất
```

Tài liệu chi tiết: [2-application-discovery-service.md](./2-application-discovery-service.md)

---

### 3. AWS Migration Evaluator

**Vai trò:** Phân tích dữ liệu thu thập được → đưa ra báo cáo TCO và đề xuất right-sizing cho AWS.

```
Migration Evaluator hoạt động như sau:
├── Input: dữ liệu từ ADS hoặc từ công cụ bên thứ ba (RVTools, Windows SCOM, ...)
├── Phân tích: pattern sử dụng tài nguyên theo thời gian
├── Output: báo cáo chuyên nghiệp bao gồm
│   ├── Ước tính chi phí AWS (theo từng instance type)
│   ├── So sánh TCO on-premises vs AWS (3-5 năm)
│   ├── Đề xuất right-sizing (tránh over-provisioning)
│   └── Quick wins — tiết kiệm ngay lập tức
└── Dùng để thuyết phục stakeholders (ban lãnh đạo) về ROI
```

Tài liệu chi tiết: [3-migration-evaluator.md](./3-migration-evaluator.md)

---

## 🔄 Luồng Làm Việc Điển Hình

```
Giai đoạn ASSESS — Thứ tự thực hiện:

Bước 1: Cài đặt ADS
├── VMware environment → dùng Agentless Connector
└── Physical servers / Mixed → dùng Discovery Agent

Bước 2: Thu thập dữ liệu (2-4 tuần)
├── ADS ghi lại mọi hoạt động server, process, kết nối mạng
└── Càng lâu → bản đồ phụ thuộc càng chính xác

Bước 3: Phân tích với Migration Evaluator
├── Import dữ liệu ADS hoặc upload từ RVTools/SCOM
├── Chạy phân tích → nhận báo cáo TCO
└── Xác định nhóm ứng dụng (application grouping)

Bước 4: Lập kế hoạch Migration Waves
├── Nhóm workload theo phụ thuộc
├── Sắp xếp thứ tự di chuyển (Wave 1, 2, 3...)
└── Gán chiến lược 7R cho từng nhóm

Bước 5: Theo dõi qua Migration Hub
├── Tạo các application groups trong Migration Hub
├── Kết nối với MGN / DMS để theo dõi tiến trình
└── Dashboard tổng hợp cho toàn bộ dự án
```

---

## 📊 So Sánh Nhanh Ba Công Cụ

| Công Cụ | Giai Đoạn | Mục Đích Chính | Kết Quả Đầu Ra |
| ------- | --------- | --------------- | --------------- |
| **Application Discovery Service** | Assess | Thu thập dữ liệu hạ tầng | Inventory + dependency map |
| **Migration Evaluator** | Assess | Phân tích TCO | Báo cáo chi phí + đề xuất |
| **Migration Hub** | Mobilize + Migrate | Theo dõi tiến trình | Dashboard tổng hợp |

---

## 🔗 Điều Hướng Tài Liệu

| File | Nội Dung |
| ---- | -------- |
| [1-migration-hub-overview.md](./1-migration-hub-overview.md) | Migration Hub: tính năng, Orchestrator, Refactor Spaces, tích hợp công cụ |
| [2-application-discovery-service.md](./2-application-discovery-service.md) | ADS: agentless vs agent-based, dữ liệu thu thập, dependency mapping |
| [3-migration-evaluator.md](./3-migration-evaluator.md) | Migration Evaluator: quy trình, báo cáo TCO, right-sizing, ví dụ thực tế |

---

## 🎓 Câu Hỏi Phỏng Vấn Điển Hình (Module Này)

1. AWS Migration Hub là gì? Nó khác gì với AWS MGN hay DMS?
2. ADS agentless khác gì so với agent-based? Khi nào dùng loại nào?
3. Migration Evaluator giúp ích gì trong việc thuyết phục ban lãnh đạo?
4. Làm thế nào để tạo dependency map cho 500 server trước khi di chuyển?
5. Migration Hub Orchestrator là gì? Cho ví dụ workflow tự động hóa.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
