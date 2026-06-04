# Phỏng Vấn AWS Management & Governance — Hướng Dẫn Ôn Tập

> Tổng hợp toàn diện để chuẩn bị phỏng vấn vị trí **Cloud Engineer**, **DevOps Engineer**, **Solutions Architect** liên quan đến AWS Management & Governance. Nội dung bao gồm câu hỏi phỏng vấn, so sánh dịch vụ, tình huống thiết kế hệ thống và câu chuyện sự cố theo phương pháp STAR.

---

## 📁 Cấu Trúc Thư Mục Này

```
11-interview-prep/
├── README.md                    ← Bạn đang đọc file này
├── INTERVIEW_GUIDE.md           Top 30 câu hỏi & đáp án chi tiết
├── service-comparison.md        So sánh dịch vụ dễ nhầm lẫn
├── system-design-scenarios.md   Tình huống thiết kế hệ thống thực tế
├── star-stories.md              Mẫu câu chuyện sự cố theo STAR
└── 90-day-study-plan.md         Kế hoạch học 90 ngày có cấu trúc
```

---

## 🎯 Mức Độ Phỏng Vấn & Trọng Tâm

### Junior Cloud Engineer / Associate (0–2 năm)

**Kỳ vọng:** Hiểu khái niệm cơ bản, biết dùng Console, giải thích được mục đích từng dịch vụ.

| Chủ đề                        | Mức độ     | File tham khảo               |
| ----------------------------- | ---------- | ---------------------------- |
| CloudWatch Metrics & Alarms   | Bắt buộc   | INTERVIEW_GUIDE.md Q1–Q6     |
| CloudTrail cơ bản             | Bắt buộc   | INTERVIEW_GUIDE.md Q7–Q10    |
| AWS Config Rules              | Quan trọng | INTERVIEW_GUIDE.md Q11–Q13   |
| CloudFormation cơ bản         | Quan trọng | INTERVIEW_GUIDE.md Q14–Q17   |
| Organizations & SCPs          | Tốt hơn    | INTERVIEW_GUIDE.md Q18–Q20   |

### Mid-Level DevOps / Cloud Engineer (2–5 năm)

**Kỳ vọng:** Thiết kế giải pháp, so sánh trade-off, xử lý sự cố thực tế.

| Chủ đề                              | Mức độ     | File tham khảo                  |
| ----------------------------------- | ---------- | ------------------------------- |
| Observability strategy              | Bắt buộc   | system-design-scenarios.md S1   |
| Multi-account architecture          | Bắt buộc   | system-design-scenarios.md S2   |
| Compliance-as-code                  | Bắt buộc   | system-design-scenarios.md S3   |
| SSM vs bastion host                 | Quan trọng | INTERVIEW_GUIDE.md Q21–Q23      |
| CloudFormation StackSets            | Quan trọng | INTERVIEW_GUIDE.md Q24–Q26      |

### Senior / Staff / Solutions Architect (5+ năm)

**Kỳ vọng:** Thiết kế kiến trúc enterprise, đánh đổi kỹ thuật, dẫn dắt quyết định.

| Chủ đề                                   | Mức độ     | File tham khảo                  |
| ---------------------------------------- | ---------- | ------------------------------- |
| Landing zone design từ đầu               | Bắt buộc   | system-design-scenarios.md S4   |
| SIEM integration với AWS audit services  | Quan trọng | system-design-scenarios.md S5   |
| FinOps governance đa account             | Quan trọng | system-design-scenarios.md S6   |
| Automated incident response              | Tốt hơn    | star-stories.md                 |

---

## 🗺️ Lộ Trình Ôn Tập 2 Tuần (Trước Phỏng Vấn)

### Tuần 1: Nắm Vững Kiến Thức

```
Ngày 1–2: Đọc service-comparison.md
           → Phân biệt CloudTrail / Config / CloudWatch
           → Phân biệt SSM Parameter Store / Secrets Manager
           → Phân biệt Organizations / Control Tower

Ngày 3–4: Đọc INTERVIEW_GUIDE.md Q1–Q15
           → Luyện giải thích lớn tiếng
           → Viết bullet points ghi nhớ nhanh

Ngày 5–6: Đọc INTERVIEW_GUIDE.md Q16–Q30
           → Tập trung vào Q có gắn nhãn [Multi-Account] và [Design]

Ngày 7:   Xem lại toàn bộ, ôn phần yếu nhất
```

### Tuần 2: Luyện Thực Chiến

```
Ngày 8–9:  Luyện system-design-scenarios.md S1–S3
            → Vẽ diagram trên giấy/whiteboard
            → Giải thích từng bước thiết kế

Ngày 10–11: Luyện system-design-scenarios.md S4–S6
             → Chuẩn bị trade-off arguments

Ngày 12–13: Học star-stories.md
             → Chuẩn bị 2–3 câu chuyện từ kinh nghiệm thực tế
             → Dùng template STAR điền vào

Ngày 14:   Mock interview tự thực hành
            → Không nhìn tài liệu, trả lời 10 câu ngẫu nhiên
```

---

## 💡 Các Dịch Vụ Hay Bị Nhầm Nhất

Đây là 3 cặp dịch vụ thí sinh phỏng vấn hay nhầm nhất. Nắm vững điều này sẽ tạo ấn tượng tốt:

### 1. CloudTrail vs AWS Config

| Khía cạnh       | CloudTrail                          | AWS Config                              |
| --------------- | ----------------------------------- | --------------------------------------- |
| Câu hỏi trả lời | "Ai đã làm gì và khi nào?"         | "Cấu hình tài nguyên có đúng không?"   |
| Dữ liệu         | API calls (hành động)               | Resource state (trạng thái cấu hình)   |
| Thời gian thực  | Gần real-time                       | Theo chu kỳ (periodic) hoặc on-change  |
| Mục đích chính  | Kiểm toán (audit), pháp lý          | Tuân thủ (compliance), governance       |

### 2. SSM Parameter Store vs Secrets Manager

| Khía cạnh           | Parameter Store                | Secrets Manager                      |
| ------------------- | ------------------------------ | ------------------------------------ |
| Mục đích            | Config, metadata, simple secret | Secrets có rotation tự động          |
| Chi phí             | Miễn phí (Standard tier)       | Tính phí per secret                  |
| Auto-rotation       | Không có native                | Có, tích hợp RDS, Redshift...        |
| Cross-account       | Hạn chế                        | Hỗ trợ tốt                           |

### 3. Organizations vs Control Tower

| Khía cạnh          | AWS Organizations                | Control Tower                           |
| ------------------ | -------------------------------- | --------------------------------------- |
| Là gì              | Dịch vụ quản lý đa tài khoản    | Lớp automation trên Organizations       |
| SCPs               | Tự cấu hình thủ công            | Guardrails áp SCPs tự động              |
| Account creation   | Thủ công hoặc tự viết script    | Account Factory tạo tự động             |
| Phù hợp            | Toàn bộ quy mô                  | Doanh nghiệp cần landing zone nhanh     |

---

## 🔑 15 Khái Niệm Cốt Lõi Phải Nắm

Trước phỏng vấn, đảm bảo giải thích được 15 khái niệm này bằng ngôn ngữ của mình:

1. **Observability** (Quan Sát) — Khả năng hiểu trạng thái nội bộ hệ thống qua output bên ngoài (metrics, logs, traces)
2. **Compliance-as-Code** (Tuân Thủ Dưới Dạng Mã) — Encode policy tuân thủ vào code tự kiểm tra và tự khắc phục
3. **Drift Detection** (Phát Hiện Trôi Dạt) — Phát hiện khi tài nguyên bị thay đổi ngoài IaC, không còn khớp với template
4. **Blast Radius** (Bán Kính Nổ) — Phạm vi ảnh hưởng khi sự cố xảy ra; multi-account giảm blast radius
5. **SCP — Service Control Policy** (Chính Sách Kiểm Soát Dịch Vụ) — Giới hạn permission tối đa trong một account/OU
6. **Landing Zone** (Vùng Hạ Cánh) — Môi trường đa tài khoản được thiết lập sẵn theo best practice
7. **Guardrail** (Rào Chắn) — Chính sách quản trị: preventive (chặn hành động) hoặc detective (phát hiện vi phạm)
8. **Conformance Pack** (Gói Tuân Thủ) — Bộ Config Rules đóng gói theo framework chuẩn (CIS, PCI-DSS, NIST)
9. **Remediation Action** (Hành Động Khắc Phục) — Tự động sửa cấu hình sai khi Config Rule phát hiện vi phạm
10. **Delegated Administrator** (Quản Trị Viên Được Ủy Quyền) — Member account được trao quyền quản lý dịch vụ cụ thể
11. **FinOps** — Thực hành quản trị chi phí đám mây: visibility, optimization, governance
12. **Organization Trail** (Dấu Vết Tổ Chức) — CloudTrail trail áp dụng cho toàn bộ accounts trong Organization
13. **Composite Alarm** (Cảnh Báo Tổng Hợp) — CloudWatch Alarm kết hợp nhiều alarm bằng logic AND/OR
14. **Change Set** (Bộ Thay Đổi) — Xem trước thay đổi CloudFormation trước khi thực sự áp dụng
15. **StackSet** (Bộ Stack) — Triển khai CloudFormation Stack đồng thời trên nhiều account/region

---

## 📋 Checklist Trước Phỏng Vấn

### Kiến Thức Kỹ Thuật

- [ ] Giải thích được sự khác nhau CloudTrail / Config / CloudWatch trong 2 phút
- [ ] Thiết kế được monitoring strategy cho production workload
- [ ] Biết SSM Session Manager hoạt động thế nào và tại sao tốt hơn bastion host
- [ ] Hiểu SCPs: deny list vs allow list, inheritance theo OU
- [ ] Biết CloudFormation Stack, StackSet, Change Set là gì và khác nhau như thế nào
- [ ] Giải thích landing zone và guardrails trong Control Tower
- [ ] Biết cách thiết kế cost governance với Budgets + Anomaly Detection + Tagging

### Tình Huống Thực Tế

- [ ] Chuẩn bị ≥ 2 câu chuyện sự cố (incident story) theo STAR
- [ ] Luyện whiteboard design cho ít nhất 1 system design scenario
- [ ] Chuẩn bị câu hỏi ngược cho interviewer (hỏi về tech stack, team, incident process)

### Câu Hỏi Ngược Cho Interviewer

- "Team đang dùng centralized logging như thế nào? CloudTrail có bật Organization Trail không?"
- "Infrastructure as Code được dùng ở đây là gì — CloudFormation, CDK hay Terraform?"
- "Compliance framework nào đang áp dụng — PCI-DSS, SOC2 hay ISO27001?"
- "Khi có sự cố production, quy trình incident response hiện tại là gì?"

---

## 🚀 Điều Hướng Nhanh

| Nhu cầu                              | File                          |
| ------------------------------------ | ----------------------------- |
| Ôn 30 câu hỏi phỏng vấn             | INTERVIEW_GUIDE.md            |
| So sánh dịch vụ dễ nhầm             | service-comparison.md         |
| Luyện system design                 | system-design-scenarios.md    |
| Chuẩn bị câu chuyện STAR            | star-stories.md               |
| Kế hoạch học dài hạn                | 90-day-study-plan.md          |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
