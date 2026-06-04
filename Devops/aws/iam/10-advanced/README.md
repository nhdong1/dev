# 10. Advanced IAM & Security — Kỹ Thuật Bảo Mật Nâng Cao Trên AWS

> Module nâng cao nhất trong lộ trình học AWS Security — bao gồm 5 chủ đề chuyên sâu mà các kỹ sư bảo mật senior và cloud architect cần nắm vững: ABAC, Zero Trust, Data Perimeter, Security Automation, và Supply Chain Security.

---

## 📚 Mục Lục Module

| File | Chủ Đề | Mức Độ | Thời Gian Học |
|---|---|---|---|
| [1-abac.md](1-abac.md) | ABAC — Attribute-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Thuộc Tính | ⭐⭐⭐⭐ | 3–4 giờ |
| [2-zero-trust.md](2-zero-trust.md) | Zero Trust Architecture — Kiến Trúc Không Tin Tưởng Mặc Định | ⭐⭐⭐⭐ | 4–5 giờ |
| [3-data-perimeter.md](3-data-perimeter.md) | Data Perimeter — Vành Đai Dữ Liệu | ⭐⭐⭐⭐⭐ | 4–5 giờ |
| [4-security-automation.md](4-security-automation.md) | Security Automation — Tự Động Hóa Bảo Mật | ⭐⭐⭐⭐ | 5–6 giờ |
| [5-supply-chain-security.md](5-supply-chain-security.md) | Supply Chain Security — Bảo Mật Chuỗi Cung Ứng | ⭐⭐⭐⭐ | 4–5 giờ |

---

## 🎯 Tại Sao Cần Học Module Này?

Module 10 đánh dấu ranh giới giữa kỹ sư cloud biết dùng IAM và **security architect thực sự** hiểu cách thiết kế hệ thống bảo mật ở quy mô enterprise. Các kỹ thuật trong module này được áp dụng tại:

- **Doanh nghiệp tài chính:** Data perimeter để đảm bảo dữ liệu không rò rỉ ra ngoài tổ chức
- **Công ty công nghệ quy mô lớn:** ABAC để quản lý hàng nghìn developer không cần tạo role riêng
- **Tổ chức với CI/CD pipeline:** Supply chain security để ngăn chặn tấn công qua dependency
- **Mọi tổ chức hiện đại:** Zero Trust thay thế mô hình perimeter network cũ

---

## 📖 Giới Thiệu 5 Chủ Đề Nâng Cao

### 1. ABAC — Attribute-Based Access Control (Kiểm Soát Truy Cập Dựa Trên Thuộc Tính)

ABAC là mô hình kiểm soát truy cập thế hệ mới, thay vì gắn quyền vào Role (RBAC), ABAC gắn quyền vào **thuộc tính** (tag) của cả người dùng lẫn tài nguyên. Một policy ABAC duy nhất có thể thay thế hàng chục role RBAC.

```
RBAC (cũ):         ABAC (mới):
role-dev-team-a    tag: team=team-a, env=dev → tự động khớp
role-dev-team-b    tag: team=team-b, env=dev → tự động khớp
role-prod-team-a   tag: team=team-a, env=prod → tự động khớp
role-prod-team-b   → không cần tạo thêm role!
```

**Khi nào áp dụng:** Tổ chức có nhiều team, nhiều môi trường, cần scale mà không muốn bùng nổ số lượng role.

---

### 2. Zero Trust Architecture (Kiến Trúc Không Tin Tưởng Mặc Định)

Zero Trust là triết lý bảo mật: **"Never trust, always verify"** — không bao giờ tin tưởng mặc định, luôn xác minh. Không có "vùng mạng an toàn" — mọi request đều phải được xác thực và ủy quyền, kể cả từ trong mạng nội bộ.

```
Mô Hình Cũ (Castle & Moat):     Mô Hình Zero Trust:
[Internet] → [Firewall] →        Mọi request đều bị kiểm tra:
[Trusted Internal Network]       Identity + Device + Network +
→ Trong mạng = tin tưởng ✓       Context → Decision → Access
```

**Tại sao quan trọng:** 80% data breach bắt nguồn từ trong mạng nội bộ (insider threats, lateral movement sau khi attacker vào được perimeter).

---

### 3. Data Perimeter (Vành Đai Dữ Liệu)

Data Perimeter là tập hợp các kiểm soát phòng thủ có chiều sâu đảm bảo **chỉ định danh đáng tin cậy, truy cập tài nguyên đáng tin cậy, từ mạng đáng tin cậy**. Đây là framework của AWS để ngăn chặn data exfiltration (rò rỉ dữ liệu) qua các vector khác nhau.

```
3 Chiều Của Data Perimeter:
┌──────────────────────────────────────────┐
│  WHO (Ai):   Chỉ principals trong Org   │
│  WHAT (Gì):  Chỉ resources trong Org    │
│  WHERE (Đâu): Chỉ từ trusted networks   │
└──────────────────────────────────────────┘
```

**Tại sao quan trọng:** Ngăn chặn kẻ tấn công dùng credentials bị đánh cắp để copy dữ liệu ra ngoài tổ chức.

---

### 4. Security Automation (Tự Động Hóa Bảo Mật)

Security Automation là khả năng **phát hiện và phản ứng tự động** với các sự kiện bảo mật — không cần con người can thiệp thủ công. Kiến trúc điển hình: GuardDuty → Security Hub → EventBridge → Lambda → Auto-remediation.

```
Phản Ứng Thủ Công:             Phản Ứng Tự Động:
Alert → Email → Ticket →       Alert → Lambda tự cô lập instance
Human reads → SSH →            trong 30 giây, không cần con người
Investigate → Isolate
(Thời gian: 30–60 phút)        (Thời gian: < 2 phút)
```

**Tại sao quan trọng:** Tốc độ phản hồi là yếu tố quyết định trong bảo mật. Mỗi phút delay = thêm lateral movement.

---

### 5. Supply Chain Security (Bảo Mật Chuỗi Cung Ứng)

Supply Chain Security bảo vệ ứng dụng khỏi các tấn công thông qua **third-party dependencies, container images, và CI/CD pipeline**. Các vụ tấn công nổi tiếng như SolarWinds, Log4Shell, và XZ Utils đều là supply chain attacks.

```
Điểm Tấn Công Chuỗi Cung Ứng:
Source Code → Dependencies → Build → Artifact → Deploy
     ↑              ↑           ↑        ↑          ↑
 Typosquatting  Malicious    Build   Image      Runtime
 Dependency     Package     System  Tampering   Config
 Confusion      Injection   Compromise
```

**Tại sao quan trọng:** Một package độc hại được cài vào dependency → toàn bộ production bị compromise.

---

## 🗺️ Mối Quan Hệ Giữa Các Chủ Đề

```
┌─────────────────────────────────────────────────────────────────────┐
│                     IDENTITY FOUNDATION                             │
│              ABAC — Attribute-Based Access Control                  │
│    (Nền tảng: Tag-based policy để scale quyền truy cập)            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ Cung cấp cơ chế phân quyền linh hoạt
          ┌────────────────────┴────────────────────┐
          ▼                                         ▼
┌──────────────────────┐               ┌────────────────────────┐
│   ZERO TRUST         │               │    DATA PERIMETER       │
│   Architecture       │◄─────────────►│    (Vành Đai Dữ Liệu)  │
│                      │  Bổ sung nhau │                        │
│ "Verify everything"  │               │ "Contain data within   │
│ cho network/device   │               │  trusted boundary"     │
└──────────┬───────────┘               └──────────┬─────────────┘
           │                                      │
           │ Phát hiện violations cần             │ SCPs/Resource policies
           │ automated response                   │ cần enforce
           └────────────────┬─────────────────────┘
                            ▼
               ┌────────────────────────┐
               │  SECURITY AUTOMATION   │
               │  (Tự Động Hóa)         │
               │                        │
               │ EventBridge + Lambda   │
               │ tự động phản hồi mọi   │
               │ vi phạm được phát hiện │
               └────────────┬───────────┘
                            │
                            │ Bảo vệ pipeline tạo ra
                            │ code/infrastructure
                            ▼
               ┌────────────────────────┐
               │  SUPPLY CHAIN SECURITY │
               │  (Bảo Mật Chuỗi Cung  │
               │   Ứng)                 │
               │                        │
               │ Bảo mật từ gốc:        │
               │ Code → Build → Deploy  │
               └────────────────────────┘
```

---

## 📋 Prerequisites — Yêu Cầu Kiến Thức Trước

Trước khi học module này, bạn cần nắm vững:

### Bắt Buộc (Must Have)

| Chủ Đề | Module Tham Khảo | Lý Do Cần Thiết |
|---|---|---|
| IAM Users, Roles, Policies | [01-iam-fundamentals](../01-iam-fundamentals/README.md) | ABAC xây dựng trên policy conditions |
| IAM Conditions & Condition Keys | [01-iam-fundamentals/3-iam-conditions.md](../01-iam-fundamentals/3-iam-conditions.md) | `aws:PrincipalTag`, `aws:ResourceTag` là core của ABAC |
| AWS Organizations & SCPs | [03-organizations](../03-organizations/README.md) | Data Perimeter dùng SCPs |
| CloudTrail & Security Hub | [07-monitoring-auditing](../07-monitoring-auditing/README.md) | Security Automation trigger |
| GuardDuty | [08-threat-detection](../08-threat-detection/README.md) | Nguồn event cho automation |

### Nên Có (Good to Have)

| Chủ Đề | Mô Tả |
|---|---|
| Python cơ bản | Lambda auto-remediation viết bằng Python |
| AWS Lambda | Security automation dùng Lambda |
| EventBridge | Event routing trong automation pipeline |
| VPC & Networking | Zero Trust network controls |
| Container & Docker | Supply chain security cho containers |

---

## 🎓 Hướng Dẫn Sử Dụng Module

### Thứ Tự Học Khuyến Nghị

```
Bước 1: ABAC (1-abac.md)
   └─► Hiểu tag-based access → nền tảng cho Data Perimeter

Bước 2: Data Perimeter (3-data-perimeter.md)
   └─► Hiểu 3 chiều kiểm soát → tư duy "vùng tin cậy"

Bước 3: Zero Trust (2-zero-trust.md)
   └─► Mở rộng tư duy sang network, device, continuous verification

Bước 4: Security Automation (4-security-automation.md)
   └─► Tự động hóa enforcement của tất cả những gì đã học

Bước 5: Supply Chain Security (5-supply-chain-security.md)
   └─► Bảo mật pipeline tạo ra infrastructure và application
```

### Cách Tiếp Cận Hiệu Quả

1. **Đọc lý thuyết** — Hiểu concept trước khi xem code
2. **Phân tích JSON policies** — Tự giải thích từng condition key
3. **Vẽ lại diagram** — Tự vẽ kiến trúc từ trí nhớ
4. **Lab exercises** — Tự tay triển khai trong môi trường test
5. **Câu hỏi phỏng vấn** — Tự trả lời trước khi xem đáp án

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Sau |
|---|---|---|
| [09-compliance-governance/](../09-compliance-governance/README.md) | **10-advanced/** | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
