# 11 — Chuẩn Bị Phỏng Vấn Terraform

> Bộ tài liệu toàn diện giúp bạn tự tin trả lời mọi câu hỏi Terraform trong phỏng vấn kỹ thuật, từ câu hỏi nền tảng đến system design — thiết kế hệ thống — thực chiến.

---

## 📁 Nội Dung Thư Mục

| File | Mô Tả | Độ Ưu Tiên |
|------|--------|-----------|
| `1-INTERVIEW_GUIDE.md` | Top 20 câu hỏi phỏng vấn Terraform kèm đáp án chi tiết | ⭐⭐⭐ Bắt buộc |
| `2-star-stories.md` | Câu chuyện sự cố theo phương pháp STAR — Situation, Task, Action, Result | ⭐⭐⭐ Bắt buộc |
| `3-system-design-scenarios.md` | Bài toán thiết kế hệ thống với IaC — Infrastructure as Code | ⭐⭐⭐ Quan trọng |
| `4-technical-questions.md` | Q&A — Hỏi & Đáp — kỹ thuật tổng hợp theo từng chủ đề | ⭐⭐ Nên có |
| `5-90-day-study-plan.md` | Kế hoạch học 90 ngày có cấu trúc rõ ràng | ⭐⭐ Nên có |

---

## 🎯 Mục Tiêu Phần Này

Sau khi hoàn thành phần 11, bạn có thể:

- ✅ Trả lời tự tin **top 20 câu hỏi Terraform** thường gặp nhất
- ✅ Kể **2–3 câu chuyện sự cố hạ tầng** theo phương pháp STAR với chi tiết thuyết phục
- ✅ **Thiết kế hệ thống IaC** cho tổ chức nhiều team, nhiều môi trường
- ✅ Giải thích **trade-off — đánh đổi** giữa các approaches một cách rõ ràng
- ✅ Thể hiện **tư duy vận hành** (không chỉ biết viết code)

---

## 🗺️ Lộ Trình Chuẩn Bị Theo Thời Gian

### 2 Tuần Trước Phỏng Vấn

```
Tuần 1:
- Đọc kỹ 1-INTERVIEW_GUIDE.md — nắm chắc top 20 câu hỏi
- Ôn lại 02-state-management/ và 03-modules/ (luôn được hỏi)
- Bắt đầu chuẩn bị 2–3 câu chuyện STAR

Tuần 2:
- Luyện tập trả lời to (nói thành tiếng, không chỉ đọc)
- Mock interview — Phỏng vấn thử — với đồng nghiệp
- Đọc 3-system-design-scenarios.md và luyện vẽ diagram
```

### 3 Ngày Trước Phỏng Vấn

```
- Ôn lại câu chuyện STAR của bản thân
- Review production checklist (09-troubleshooting/)
- Nghỉ ngơi — đừng nhồi nhét quá nhiều
```

### Buổi Sáng Ngày Phỏng Vấn

```
- Đọc lại tóm tắt 1-INTERVIEW_GUIDE.md
- Nhớ lại 3 câu chuyện STAR
- Ổn định tâm lý — bạn đã chuẩn bị kỹ rồi
```

---

## 💡 Bí Quyết Trả Lời Phỏng Vấn Terraform

### Nguyên Tắc 1: Luôn Bắt Đầu Bằng "Tại Sao"

```
❌ Kém: "Tôi dùng remote backend vì công ty yêu cầu"
✅ Tốt: "Remote backend — backend từ xa — giải quyết vấn đề
        khi team nhiều người: state locking ngăn apply đồng thời,
        encryption bảo vệ thông tin nhạy cảm, và backup tự động
        giúp disaster recovery — khôi phục thảm họa."
```

### Nguyên Tắc 2: Nêu Trade-off — Đánh Đổi

```
Với mọi lựa chọn kỹ thuật, luôn trình bày:
- Tôi chọn X vì...
- X có hạn chế là...
- Trong trường hợp A thì Y sẽ tốt hơn vì...
```

### Nguyên Tắc 3: Gắn Với Kinh Nghiệm Thực Tế

```
Interviewers — Người phỏng vấn — muốn biết bạn đã làm thật,
không chỉ đọc docs. Mỗi concept, cố gắng kể:
"Tôi đã gặp tình huống này khi... và giải quyết bằng cách..."
```

### Nguyên Tắc 4: Chấp Nhận Không Biết Một Cách Chuyên Nghiệp

```
"Tôi chưa dùng Sentinel — Policy as Code engine của HashiCorp —
trực tiếp, nhưng tôi đã làm việc với OPA — Open Policy Agent —
để enforce compliance. Cả hai đều implement policy as code
nhưng Sentinel tích hợp sâu hơn với Terraform Cloud."
```

---

## 📊 Các Chủ Đề Hay Bị Hỏi Nhất

Dựa trên phản hồi từ nhiều buổi phỏng vấn DevOps/Platform Engineer:

| Chủ Đề | Tần Suất Xuất Hiện | File Tham Khảo |
|--------|-------------------|----------------|
| State Management — Quản lý trạng thái | 95% | `../02-state-management/` |
| Module Design — Thiết kế module | 90% | `../03-modules/` |
| CI/CD Integration — Tích hợp CI/CD | 85% | `../06-cicd/` |
| Remote Backend — Backend từ xa | 85% | `../02-state-management/2-remote-backend.md` |
| Security — Bảo mật | 75% | `../05-security/` |
| Drift Detection — Phát hiện lệch cấu hình | 70% | `../08-monitoring/1-drift-detection.md` |
| Câu chuyện sự cố (STAR) | 80% | `2-star-stories.md` |
| System Design với IaC | 60% | `3-system-design-scenarios.md` |

---

## 🔗 Điều Hướng Nhanh

- [Top 20 câu hỏi →](1-INTERVIEW_GUIDE.md)
- [Câu chuyện STAR →](2-star-stories.md)
- [System Design →](3-system-design-scenarios.md)
- [Q&A Kỹ Thuật →](4-technical-questions.md)
- [Kế Hoạch 90 Ngày →](5-90-day-study-plan.md)

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
