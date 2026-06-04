# AWS Control Tower — Landing Zone & Quản Trị Đa Tài Khoản

> **Control Tower** (Tháp Kiểm Soát) là dịch vụ tự động hóa việc thiết lập và quản trị môi trường đa tài khoản AWS an toàn, tuân thủ theo AWS best practice — cho phép doanh nghiệp triển khai **Landing Zone** (Vùng Hạ Cánh) đã được cấu hình sẵn trong vài giờ thay vì vài tháng.

---

## 📚 Mục Lục

1. [Control Tower là gì và tại sao cần?](#control-tower-là-gì-và-tại-sao-cần)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Các Thành Phần Chính](#các-thành-phần-chính)
4. [Control Tower vs Organizations — Sự Khác Biệt](#control-tower-vs-organizations--sự-khác-biệt)
5. [Luồng Làm Việc Điển Hình](#luồng-làm-việc-điển-hình)
6. [Giới Hạn & Lưu Ý Quan Trọng](#giới-hạn--lưu-ý-quan-trọng)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
8. [Điều Hướng Module](#điều-hướng-module)

---

## Control Tower là gì và tại sao cần?

### Vấn Đề Không Có Control Tower

Khi doanh nghiệp mở rộng lên hàng chục hoặc hàng trăm AWS account, việc thiết lập thủ công gặp nhiều vấn đề:

```
❌ Mỗi account mới phải cấu hình lại từ đầu:
   - Bật CloudTrail logging
   - Thiết lập AWS Config recorder
   - Áp dụng SCP (Service Control Policy — Chính Sách Kiểm Soát Dịch Vụ)
   - Cấu hình IAM Identity Center (trước là SSO)
   - Gắn vào OU (Organizational Unit — Đơn Vị Tổ Chức) đúng

❌ Không có guardrails (rào chắn) nhất quán
❌ Không có dashboard compliance tập trung
❌ Quá trình tạo account mới tốn 2–4 tuần làm thủ công
```

### Control Tower Giải Quyết Như Thế Nào?

```
✅ Landing Zone được thiết lập tự động trong 1–2 giờ
✅ Guardrails tự động áp dụng cho mọi account mới
✅ Account Factory: tạo account chuẩn theo template chỉ mất vài phút
✅ Dashboard compliance tập trung — nhìn toàn bộ trạng thái landing zone
✅ Tích hợp sẵn với Organizations, IAM Identity Center, CloudTrail, Config
```

### Khi Nào Nên Dùng Control Tower?

| Tình Huống                                    | Khuyến Nghị              |
| --------------------------------------------- | ------------------------ |
| Tổ chức mới bắt đầu multi-account (< 5 acc)  | Dùng Organizations thuần |
| Doanh nghiệp cần landing zone chuẩn nhanh    | ✅ Dùng Control Tower    |
| Cần guardrails đồng nhất trên 10+ account    | ✅ Dùng Control Tower    |
| Team DevOps muốn tự quản lý hoàn toàn        | Organizations + tự build |
| Cần tuân thủ CIS/NIST/SOC2 tự động           | ✅ Dùng Control Tower    |

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Control Tower                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Landing Zone (Vùng Hạ Cánh)                │   │
│  │                                                         │   │
│  │  Management Account (Root)                              │   │
│  │  ┌────────────────────────────────────────────────┐    │   │
│  │  │            Organization Root                   │    │   │
│  │  │                                                │    │   │
│  │  │  ┌─────────────────────────────────────────┐  │    │   │
│  │  │  │        Security OU                       │  │    │   │
│  │  │  │  ┌─────────────┐  ┌──────────────────┐  │  │    │   │
│  │  │  │  │ Log Archive │  │  Audit Account   │  │  │    │   │
│  │  │  │  │   Account   │  │  (Security Tooling)│  │  │    │   │
│  │  │  │  └─────────────┘  └──────────────────┘  │  │    │   │
│  │  │  └─────────────────────────────────────────┘  │    │   │
│  │  │                                                │    │   │
│  │  │  ┌─────────────────────────────────────────┐  │    │   │
│  │  │  │        Sandbox OU                        │  │    │   │
│  │  │  │  ┌──────────────┐  ┌──────────────────┐ │  │    │   │
│  │  │  │  │  Dev Account │  │  Test Account    │ │  │    │   │
│  │  │  │  └──────────────┘  └──────────────────┘ │  │    │   │
│  │  │  └─────────────────────────────────────────┘  │    │   │
│  │  │                                                │    │   │
│  │  │  ┌─────────────────────────────────────────┐  │    │   │
│  │  │  │        Workloads OU (tùy chỉnh)          │  │    │   │
│  │  │  │  ┌──────────────┐  ┌──────────────────┐ │  │    │   │
│  │  │  │  │ Prod Account │  │ Staging Account  │ │  │    │   │
│  │  │  │  └──────────────┘  └──────────────────┘ │  │    │   │
│  │  │  └─────────────────────────────────────────┘  │    │   │
│  │  └────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌────────────────┐  ┌────────────────┐  ┌─────────────────┐  │
│  │   Guardrails   │  │ Account Factory │  │   Dashboard     │  │
│  │  (Preventive + │  │  (Tạo Account  │  │  (Compliance    │  │
│  │   Detective)   │  │   Tự Động)     │  │   Tập Trung)    │  │
│  └────────────────┘  └────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Các Thành Phần Chính

### 1. Landing Zone (Vùng Hạ Cánh)

**Landing Zone** là môi trường đa tài khoản đã được cấu hình sẵn theo best practice, gồm:

| Thành Phần                     | Mô Tả                                                     |
| ------------------------------ | --------------------------------------------------------- |
| **Management Account**         | Account gốc điều hành toàn bộ Control Tower               |
| **Log Archive Account**        | Lưu trữ tập trung CloudTrail + Config logs từ mọi account |
| **Audit Account**              | Truy cập read-only vào tất cả account để kiểm toán bảo mật|
| **Security OU**                | OU chứa Log Archive + Audit, bảo vệ bằng guardrails cứng |
| **Sandbox OU**                 | OU cho dev/test với guardrails nới lỏng hơn               |
| **IAM Identity Center**        | SSO tập trung cho toàn bộ accounts trong landing zone     |

Chi tiết xem: [1-landing-zone-setup.md](1-landing-zone-setup.md)

---

### 2. Guardrails (Rào Chắn Quản Trị)

**Guardrails** là chính sách quản trị được áp dụng tự động cho các OU trong landing zone.

| Loại Guardrail    | Cơ Chế            | Ví Dụ                                              |
| ----------------- | ----------------- | -------------------------------------------------- |
| **Preventive**    | SCP — chặn trực tiếp | Không cho phép tắt CloudTrail                  |
| **Detective**     | Config Rule — phát hiện | Cảnh báo nếu S3 bucket public                |
| **Proactive**     | CloudFormation Hook — kiểm tra trước deploy | Từ chối stack không tuân thủ |

Chi tiết xem: [2-guardrails.md](2-guardrails.md)

---

### 3. Account Factory (Xưởng Tài Khoản)

**Account Factory** cho phép tạo tài khoản AWS mới theo template chuẩn tự động, bao gồm:
- Đăng ký vào OU phù hợp
- Áp dụng guardrails tự động
- Cấu hình SSO/IAM Identity Center
- Thiết lập baseline (CloudTrail, Config, VPC)

Chi tiết xem: [3-account-factory.md](3-account-factory.md)

---

### 4. Customizations for Control Tower — CfCT (Tùy Chỉnh Cho Control Tower)

**CfCT** cho phép mở rộng Control Tower với CloudFormation templates và SCPs tùy chỉnh:
- Triển khai tài nguyên bổ sung khi tạo account mới
- Thêm SCPs ngoài guardrails mặc định
- Tích hợp với CI/CD pipeline

Chi tiết xem: [4-customizations-cfct.md](4-customizations-cfct.md)

---

## Control Tower vs Organizations — Sự Khác Biệt

Đây là câu hỏi phỏng vấn phổ biến. Control Tower xây **trên nền** Organizations, không thay thế nó.

| Tiêu Chí                    | AWS Organizations               | AWS Control Tower                          |
| --------------------------- | ------------------------------- | ------------------------------------------ |
| **Mức độ**                  | Primitive (hạ tầng thô)        | Orchestration (điều phối ở tầng cao hơn)  |
| **Thiết lập**               | Thủ công hoàn toàn              | Tự động hóa end-to-end                     |
| **Guardrails**              | Tự viết SCPs                   | Guardrails có sẵn + SCP tự động            |
| **Account tạo mới**         | Thủ công hoặc tự code          | Account Factory (UI + API + Terraform)     |
| **Logging tập trung**       | Tự cấu hình                    | Log Archive account tự động                |
| **SSO**                     | Tự cấu hình IAM Identity Center| Tích hợp sẵn                               |
| **Compliance dashboard**    | Không có                        | Có — nhìn toàn bộ trạng thái              |
| **Phù hợp với**             | Cần kiểm soát tuyệt đối        | Cần nhanh + best practice có sẵn           |

> **Tóm lại:** Organizations là **nền tảng**, Control Tower là **lớp quản trị** xây trên nền đó.

---

## Luồng Làm Việc Điển Hình

### Thiết Lập Landing Zone Lần Đầu

```
1. Enable Control Tower trong Management Account
   ↓
2. Control Tower tự động tạo:
   - Log Archive Account
   - Audit Account
   - Security OU
   - Sandbox OU
   - Kích hoạt IAM Identity Center
   ↓
3. Áp dụng Mandatory Guardrails cho toàn bộ landing zone
   ↓
4. Dashboard sẵn sàng — kiểm tra compliance
```

### Tạo Account Mới Qua Account Factory

```
Developer/Cloud Admin
       ↓
Account Factory (Service Catalog — Danh Mục Dịch Vụ)
       ↓
Điền thông tin: tên account, email, OU đích, SSO user
       ↓
Control Tower thực hiện tự động:
  - Tạo AWS Account mới
  - Đăng ký vào Organizations OU
  - Áp dụng Guardrails của OU
  - Cấu hình IAM Identity Center (SSO)
  - Bật CloudTrail → gửi logs về Log Archive Account
  - Bật Config → gửi về Log Archive Account
  - Tạo VPC chuẩn (nếu cấu hình)
       ↓
Account sẵn sàng trong 20–30 phút
```

---

## Giới Hạn & Lưu Ý Quan Trọng

### Giới Hạn Kỹ Thuật

| Giới Hạn                                     | Giá Trị Mặc Định |
| -------------------------------------------- | ---------------- |
| Số account tối đa trong landing zone         | Theo quota Organizations |
| Số OU tối đa được quản lý bởi Control Tower | 300              |
| Region hỗ trợ Control Tower                  | Giới hạn (không phải tất cả region) |
| Nested OUs (OU lồng nhau)                   | Tối đa 5 cấp     |

### Lưu Ý Quan Trọng Khi Triển Khai

```
⚠️  Không xóa hoặc di chuyển Log Archive / Audit Account
    → Control Tower sẽ mất khả năng hoạt động đúng

⚠️  Không tắt CloudTrail trong Management Account
    → Vi phạm Mandatory Guardrail, có thể gây lock-out

⚠️  Không áp dụng Control Tower lên Organizations đã có nhiều account
    → Cần dùng "Extend Governance" để register account hiện có

⚠️  Control Tower quản lý một số tài nguyên
    → Không sửa trực tiếp SCPs, Config rules do Control Tower tạo
    → Thay đổi qua Control Tower console/API
```

### Chi Phí

Control Tower **không tính phí riêng**, nhưng bạn trả phí cho các dịch vụ nó sử dụng:
- AWS Config (phí per rule evaluation)
- CloudTrail (phí data events nếu bật)
- AWS Service Catalog (phí Account Factory)
- IAM Identity Center (miễn phí)

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Control Tower vs Organizations — khác nhau gì?

**Trả lời:** Organizations là nền tảng hạ tầng cho multi-account (OU, SCP, consolidated billing). Control Tower xây trên Organizations, thêm lớp tự động hóa: Landing Zone, Guardrails, Account Factory, dashboard compliance. Dùng Control Tower khi cần thiết lập chuẩn nhanh; dùng Organizations thuần khi muốn kiểm soát tuyệt đối.

---

### Q2: Log Archive Account và Audit Account có vai trò gì?

**Trả lời:**
- **Log Archive Account:** Kho lưu trữ tập trung mọi CloudTrail logs + Config history từ toàn bộ accounts. Chỉ S3 — không ai được xóa log. Guardrail bảo vệ không cho phép tắt logging.
- **Audit Account:** Có cross-account read-only access vào tất cả accounts. Dành cho security team thực hiện điều tra, kiểm toán bảo mật mà không cần vào từng account riêng.

---

### Q3: Preventive vs Detective Guardrails — khi nào dùng cái nào?

**Trả lời:**
- **Preventive Guardrail:** SCP chặn ngay lập tức — dùng cho vi phạm tuyệt đối không được phép (ví dụ: không tắt CloudTrail, không leave Organization).
- **Detective Guardrail:** Config Rule phát hiện sau khi đã xảy ra — dùng cho các vi phạm cần phát hiện và báo cáo (ví dụ: S3 bucket có public access).

---

### Q4: Account Factory for Terraform (AFT) là gì?

**Trả lời:** AFT là giải pháp IaC — Infrastructure as Code (Hạ Tầng Dưới Dạng Mã) để tạo và tùy chỉnh account thông qua Terraform thay vì Service Catalog UI. AFT dùng CodePipeline để tự động hóa: khi commit vào repo AFT, pipeline chạy tạo account mới + customize theo template. Phù hợp cho teams đã dùng Terraform.

---

### Q5: Làm thế nào extend Control Tower governance cho account đã tồn tại?

**Trả lời:** Dùng tính năng **"Register OU"** hoặc **"Enroll Account"** trong Control Tower console. Control Tower sẽ apply guardrails và baseline configuration cho account đó. Lưu ý: một số tài nguyên đã tồn tại (như VPC, SCP cũ) có thể xung đột — cần review trước.

---

## Điều Hướng Module

| File                                                | Nội Dung                                          |
| --------------------------------------------------- | ------------------------------------------------- |
| [README.md](README.md) ← *Bạn đang ở đây*         | Tổng quan Control Tower & Landing Zone            |
| [1-landing-zone-setup.md](1-landing-zone-setup.md) | Landing Zone components, Log Archive, Audit acc   |
| [2-guardrails.md](2-guardrails.md)                 | Preventive vs Detective, Mandatory vs Elective    |
| [3-account-factory.md](3-account-factory.md)       | Account Factory UI & Account Factory for Terraform|
| [4-customizations-cfct.md](4-customizations-cfct.md)| CfCT — mở rộng Control Tower với CloudFormation  |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
