# 11. Phỏng Vấn & Thực Hành GitHub Actions

> Bộ tài liệu chuẩn bị phỏng vấn GitHub Actions toàn diện — từ câu hỏi kỹ thuật, câu chuyện STAR (Situation — Task — Action — Result), system design đến bài tập thực hành và kế hoạch học 90 ngày.

---

## 📂 Nội Dung Thư Mục

| File | Mô Tả | Thời Gian Đọc |
|---|---|---|
| [1-INTERVIEW_GUIDE.md](1-INTERVIEW_GUIDE.md) | Top 20 câu hỏi phỏng vấn + đáp án chi tiết + tips trả lời | 60–90 phút |
| [2-star-stories.md](2-star-stories.md) | 10 mẫu câu chuyện STAR sẵn sàng tùy chỉnh | 30–45 phút |
| [3-system-design-scenarios.md](3-system-design-scenarios.md) | 5 kịch bản thiết kế CI/CD pipeline từ whiteboard | 60–90 phút |
| [4-hands-on-exercises.md](4-hands-on-exercises.md) | 15 bài tập thực hành có đáp án mẫu | 4–8 giờ |
| [5-90-day-study-plan.md](5-90-day-study-plan.md) | Kế hoạch học cấu trúc 90 ngày theo tuần | 20 phút |

---

## 🎯 Cách Sử Dụng Bộ Tài Liệu Này

### Nếu Phỏng Vấn Trong 1 Tuần

```
Ngày 1–2:  Đọc 1-INTERVIEW_GUIDE.md — nắm vững 20 câu hỏi cốt lõi
Ngày 3:    Đọc 2-star-stories.md — chuẩn bị 3–5 câu chuyện STAR phù hợp
Ngày 4:    Đọc 3-system-design-scenarios.md — luyện 2–3 kịch bản thiết kế
Ngày 5:    Làm 4-hands-on-exercises.md — bài tập thực hành cơ bản (1–5)
Ngày 6:    Mock interview với người khác, ghi âm, review
Ngày 7:    Ôn lại điểm yếu, nghỉ ngơi, chuẩn bị tâm lý
```

### Nếu Có 1 Tháng

```
Tuần 1:    Hoàn thiện kiến thức nền (01-fundamentals + 02-ci-pipeline)
Tuần 2:    Nắm vững CD và Security (03-cd-deployments + 08-security)
Tuần 3:    Làm tất cả bài tập thực hành (4-hands-on-exercises.md)
Tuần 4:    Mock interviews + tùy chỉnh câu chuyện STAR + system design
```

### Nếu Có 3 Tháng

```
Theo 5-90-day-study-plan.md — học có cấu trúc từng tuần
```

---

## 🏷️ Phân Loại Câu Hỏi Theo Cấp Độ

### Cấp Độ Junior (0–2 năm kinh nghiệm)

- Giải thích GitHub Actions là gì
- Workflow YAML syntax (Cú Pháp Workflow)
- CI pipeline cơ bản (checkout → test → build)
- Sử dụng secrets và variables
- Hiểu trigger events (Sự Kiện Kích Hoạt)

### Cấp Độ Mid-level (2–4 năm kinh nghiệm)

- Reusable workflows và composite actions
- CD pipeline với environments và approval gates (Cổng Phê Duyệt)
- Cache strategies (Chiến Lược Cache)
- Matrix strategy (Chiến Lược Ma Trận)
- OIDC (OpenID Connect — Xác Thực Không Cần Credentials Dài Hạn) authentication

### Cấp Độ Senior (4+ năm kinh nghiệm)

- Custom actions (JavaScript, Docker)
- Self-hosted runners và ARC (Actions Runner Controller — Bộ Điều Khiển Runner)
- Enterprise CI/CD platform design
- Security hardening (Tăng Cường Bảo Mật) và supply chain (Chuỗi Cung Ứng)
- Cost optimization (Tối Ưu Chi Phí) ở quy mô lớn
- System design cho multi-team pipeline

---

## 🔑 Chủ Đề Quan Trọng Nhất

Dựa trên phân tích câu hỏi phỏng vấn thực tế năm 2024–2026:

### 🔥 Luôn Được Hỏi (Must Know)

1. **CI/CD pipeline design** — thiết kế pipeline từ đầu đến cuối
2. **OIDC vs long-lived credentials** — tại sao OIDC tốt hơn lưu access key
3. **Reusable workflows** — `workflow_call`, inputs, outputs, secrets
4. **Secrets management** — scopes, rotation (Xoay Vòng), best practices
5. **Debugging failed workflows** — cách debug khi pipeline lỗi

### ⚡ Thường Được Hỏi (Good to Know)

6. **Matrix strategy** — test đa phiên bản Node.js/Python/OS
7. **Concurrency groups** (Nhóm Đồng Thời) — tránh deploy đồng thời
8. **Cache strategies** — cache npm/pip/Maven hiệu quả
9. **Branch protection rules** (Quy Tắc Bảo Vệ Nhánh) — required status checks
10. **Supply chain security** — pin actions by SHA, Dependabot

### 💼 Câu Hỏi Behavioral (Hành Vi)

11. Kể về lần bạn cải thiện CI/CD pipeline
12. Bạn xử lý deployment failure (Lỗi Triển Khai) như thế nào?
13. Làm sao bạn onboard team member mới vào hệ thống CI/CD?
14. Trade-offs khi chọn self-hosted vs GitHub-hosted runners

---

## 📊 Ma Trận Chuẩn Bị

| Chủ Đề | Cấp Junior | Cấp Mid | Cấp Senior |
|---|---|---|---|
| Workflow Syntax | ✅ Bắt buộc | ✅ | ✅ |
| CI Pipeline | ✅ Bắt buộc | ✅ | ✅ |
| CD & Environments | ⚠️ Cơ bản | ✅ Bắt buộc | ✅ |
| Secrets & OIDC | ⚠️ Cơ bản | ✅ Bắt buộc | ✅ |
| Reusable Workflows | ❌ Optional | ✅ Bắt buộc | ✅ |
| Custom Actions | ❌ | ⚠️ Cơ bản | ✅ Bắt buộc |
| Self-hosted Runners | ❌ | ⚠️ Cơ bản | ✅ Bắt buộc |
| Security Hardening | ❌ | ⚠️ Cơ bản | ✅ Bắt buộc |
| System Design | ❌ | ⚠️ Cơ bản | ✅ Bắt buộc |
| Cost Optimization | ❌ | ⚠️ Cơ bản | ✅ Bắt buộc |

---

## 💡 Tips Phỏng Vấn GitHub Actions

### Trước Phỏng Vấn

- **Ôn lại dự án thực tế:** Chuẩn bị kể về pipeline bạn đã xây dựng với số liệu cụ thể (giảm bao nhiêu % thời gian build, tiết kiệm bao nhiêu giờ/tuần)
- **Biết cloud platform của công ty:** Nếu họ dùng AWS thì nắm sâu OIDC với AWS, ECS/EKS deployments
- **Chuẩn bị câu hỏi ngược:** Hỏi về tech stack, quy mô team, pain points hiện tại của họ

### Trong Phỏng Vấn

- **Dùng thuật ngữ đúng:** Đừng nhầm "action" với "workflow" hay "job"
- **Kết hợp lý thuyết và thực tế:** Sau mỗi giải thích, kể ví dụ từ dự án thực
- **Thừa nhận khi không biết:** Nói "Tôi chưa dùng X nhưng cách tiếp cận của tôi sẽ là..." thay vì bịa đặt

### Sau Phỏng Vấn

- **Ghi chú câu hỏi khó:** Để ôn lại cho lần sau
- **Follow-up email:** Cảm ơn và nhắc lại 1–2 điểm bạn muốn làm rõ thêm

---

## 🔗 Điều Hướng Nhanh

- ← Quay lại: [../README.md](../README.md) — Tổng quan GitHub Actions
- ← Xem thêm: [../10-monitoring-debugging/](../10-monitoring-debugging/) — Debug workflow
- → Bắt đầu: [1-INTERVIEW_GUIDE.md](1-INTERVIEW_GUIDE.md) — Top 20 câu hỏi

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
