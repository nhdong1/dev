# 🎯 Interview Prep — Chuẩn Bị Phỏng Vấn AWS Compute

> Hướng dẫn toàn diện để chuẩn bị phỏng vấn về AWS Compute Services — bao gồm câu hỏi Q&A, kịch bản thiết kế hệ thống, câu chuyện STAR (Situation-Task-Action-Result — Tình Huống-Nhiệm Vụ-Hành Động-Kết Quả), bài tập thực hành và lộ trình học 90 ngày.

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Cấu Trúc Tài Liệu](#cấu-trúc-tài-liệu)
3. [Chiến Lược Phỏng Vấn](#chiến-lược-phỏng-vấn)
4. [Checklist Chuẩn Bị](#checklist-chuẩn-bị)
5. [Timeline Ôn Tập](#timeline-ôn-tập)

---

## Tổng Quan

AWS Compute là một trong những chủ đề được hỏi nhiều nhất trong phỏng vấn kỹ thuật cho vị trí Backend Engineer, DevOps Engineer, Cloud Engineer, và Solutions Architect. Phần này tổng hợp tất cả kiến thức cần thiết để tự tin trả lời mọi câu hỏi liên quan.

### Tại Sao AWS Compute Quan Trọng Trong Phỏng Vấn?

Các nhà tuyển dụng kiểm tra AWS Compute vì:

- **Tính phổ biến:** EC2 (Elastic Compute Cloud — Máy Chủ Ảo Đám Mây), Lambda, ECS (Elastic Container Service — Dịch Vụ Container) là nền tảng hầu hết ứng dụng cloud
- **Độ phức tạp:** Yêu cầu hiểu sâu trade-offs (đánh đổi) giữa các dịch vụ
- **Vận hành thực tế:** Liên quan trực tiếp đến chi phí, hiệu suất, và độ tin cậy hệ thống
- **System Design:** Mọi bài toán thiết kế hệ thống đều cần chọn đúng compute platform

### Các Vị Trí Hay Được Hỏi

| Vị Trí                    | Trọng Tâm AWS Compute                                |
| ------------------------- | ---------------------------------------------------- |
| Backend Engineer          | Lambda, ECS, EC2 basics, cost awareness              |
| DevOps / SRE Engineer     | EC2, Auto Scaling, EKS, monitoring, troubleshooting  |
| Cloud Engineer            | Toàn bộ compute stack, networking, security          |
| Solutions Architect       | Trade-offs, system design, cost optimization         |
| Platform Engineer         | EKS, ECS, container orchestration, IaC               |

---

## Cấu Trúc Tài Liệu

```
11-interview-prep/
├── README.md                  ← Bạn đang ở đây
├── 1-INTERVIEW_GUIDE.md       Top 20 câu hỏi kèm đáp án chi tiết
├── 2-star-stories.md          5 câu chuyện incident theo mẫu STAR
├── 3-system-design-scenarios.md  5 kịch bản thiết kế hệ thống thực tế
├── 4-hands-on-exercises.md    5 bài tập thực hành step-by-step
└── 5-90-day-study-plan.md     Lộ trình học 90 ngày chi tiết
```

### Hướng Dẫn Đọc Theo Mục Tiêu

#### Nếu Phỏng Vấn Trong 1 Tuần

```
Ngày 1-2: 1-INTERVIEW_GUIDE.md (nắm top 20 Q&A)
Ngày 3:   2-star-stories.md (chuẩn bị 3 câu chuyện phù hợp nhất)
Ngày 4-5: 3-system-design-scenarios.md (luyện 2-3 kịch bản)
Ngày 6-7: Ôn lại toàn bộ, mock interview với bạn bè
```

#### Nếu Có 1 Tháng

```
Tuần 1: Đọc toàn bộ 5 file, xác định điểm yếu
Tuần 2: Tập trung vào phần yếu, thực hành hands-on
Tuần 3: Luyện giải thích trade-offs không cần tài liệu
Tuần 4: Mock interviews, tinh chỉnh câu trả lời
```

#### Nếu Có 3 Tháng

```
Tháng 1: Học lý thuyết theo 5-90-day-study-plan.md (giai đoạn 1-2)
Tháng 2: Thực hành hands-on từ 4-hands-on-exercises.md
Tháng 3: Hệ thống hóa, luyện phỏng vấn, tinh chỉnh stories
```

---

## Chiến Lược Phỏng Vấn

### 3 Loại Câu Hỏi Phổ Biến

#### 1. Câu Hỏi Kiến Thức (Knowledge Questions)

> *"EC2 Placement Group là gì? Khi nào dùng Cluster vs Spread?"*

**Chiến lược:** Định nghĩa → Giải thích cơ chế → Ví dụ use case → Trade-offs

```
Bước 1: Định nghĩa ngắn gọn (1-2 câu)
Bước 2: Giải thích cách hoạt động (2-3 câu)
Bước 3: Use case thực tế (khi nào dùng)
Bước 4: Hạn chế / trade-offs
```

#### 2. Câu Hỏi Tình Huống (Behavioral / STAR Questions)

> *"Kể một lần bạn xử lý sự cố production nghiêm trọng."*

**Chiến lược:** Áp dụng mẫu STAR (Situation-Task-Action-Result)

```
Situation (Tình Huống):  Context cụ thể (hệ thống, thời gian, quy mô)
Task (Nhiệm Vụ):         Trách nhiệm của bạn trong tình huống đó
Action (Hành Động):      Những bước cụ thể bạn đã thực hiện
Result (Kết Quả):        Kết quả đo lường được + bài học rút ra
```

#### 3. Câu Hỏi Thiết Kế (System Design Questions)

> *"Thiết kế hệ thống xử lý 100,000 requests/giây trên AWS."*

**Chiến lược:** Clarify → Estimate → Design → Trade-offs → Scale

```
Bước 1: Làm rõ yêu cầu (functional & non-functional requirements)
Bước 2: Ước lượng quy mô (scale estimation)
Bước 3: High-level design (kiến trúc tổng thể)
Bước 4: Deep dive vào component quan trọng
Bước 5: Thảo luận trade-offs và cải tiến
```

### Mẹo Trả Lời Hiệu Quả

**Dùng ngôn ngữ rõ ràng:**
- Tránh: *"Nó giúp scale..."*
- Nên: *"Auto Scaling Group tự động thêm EC2 instance khi CPU vượt 70%, giảm latency từ 500ms xuống 200ms trong peak traffic"*

**Luôn đề cập trade-offs:**
- Không bao giờ nói một giải pháp là "tốt nhất" mà không có context
- Ví dụ: *"Lambda tiết kiệm chi phí cho event-driven workload, nhưng ECS Fargate phù hợp hơn cho long-running processes cần state"*

**Vẽ sơ đồ khi có thể:**
- Trong virtual interview: dùng whiteboard tool hoặc mô tả từng component
- Trong onsite: luôn vẽ sơ đồ trước khi giải thích

---

## Checklist Chuẩn Bị

### Kiến Thức Bắt Buộc (Must-Know)

#### EC2 & Auto Scaling

- [ ] Biết các instance families (C-series, M-series, R-series, T-series) và use case
- [ ] Hiểu sự khác biệt On-Demand / Reserved / Spot / Savings Plans
- [ ] Giải thích được Auto Scaling Group (ASG — Nhóm Tự Động Co Giãn) hoạt động thế nào
- [ ] Biết Target Tracking Policy (Chính Sách Theo Dõi Mục Tiêu) vs Step Scaling (Co Giãn Theo Bước)
- [ ] Hiểu Placement Groups — Cluster, Spread, Partition và khi nào dùng

#### Lambda & Serverless

- [ ] Giải thích cold start (khởi động lạnh) và cách giảm thiểu
- [ ] Phân biệt Reserved Concurrency (Đồng Thời Được Đặt Trước) vs Provisioned Concurrency (Đồng Thời Được Cung Cấp)
- [ ] Biết Lambda Layers (Lớp Lambda) và khi nào dùng
- [ ] Giải thích Lambda event source mapping (ánh xạ nguồn sự kiện)
- [ ] Biết Lambda pricing model (mô hình định giá)

#### ECS & Containers

- [ ] Giải thích sự khác biệt Fargate vs EC2 launch type
- [ ] Hiểu Task Definition (Định Nghĩa Task), Task, Service, Cluster
- [ ] Biết awsvpc (AWS Virtual Private Cloud) networking mode
- [ ] Giải thích ECS service deployment strategies (chiến lược triển khai)
- [ ] Hiểu ECS Service Connect và Service Discovery

#### EKS & Kubernetes

- [ ] Phân biệt control plane (mặt phẳng điều khiển) và data plane (mặt phẳng dữ liệu)
- [ ] Giải thích Managed Node Groups (Nhóm Node Được Quản Lý) vs Self-managed
- [ ] Hiểu IRSA — IAM Roles for Service Accounts (Vai Trò IAM Cho Service Accounts)
- [ ] Biết HPA — Horizontal Pod Autoscaler (Tự Động Co Giãn Pod Theo Chiều Ngang) và Cluster Autoscaler

#### High Availability & Cost

- [ ] Thiết kế Multi-AZ (Đa Vùng Khả Dụng) architecture từ đầu
- [ ] Tính toán chi phí cho một workload cụ thể
- [ ] Biết rightsizing (chọn đúng kích cỡ) quy trình với Compute Optimizer

### Kỹ Năng Mềm Cần Thể Hiện

- [ ] Giải thích kỹ thuật phức tạp theo cách người không chuyên hiểu được
- [ ] Thừa nhận khi không biết và đề xuất cách tìm hiểu
- [ ] Hỏi lại câu hỏi để làm rõ trước khi trả lời
- [ ] Đề xuất nhiều phương án và so sánh trade-offs

---

## Timeline Ôn Tập

### T-7 ngày (1 tuần trước phỏng vấn)

```
□ Đọc 1-INTERVIEW_GUIDE.md đầy đủ
□ Tự kiểm tra: trả lời từng câu hỏi không nhìn đáp án
□ Đánh dấu các câu trả lời chưa tự tin
□ Ôn lại các khái niệm còn yếu
```

### T-5 ngày

```
□ Chọn 3 câu chuyện STAR phù hợp nhất từ 2-star-stories.md
□ Điều chỉnh câu chuyện theo kinh nghiệm thực tế của bạn
□ Luyện kể không nhìn tài liệu (mỗi story < 3 phút)
□ Yêu cầu ai đó nghe và cho feedback
```

### T-3 ngày

```
□ Luyện 2 kịch bản thiết kế hệ thống từ 3-system-design-scenarios.md
□ Vẽ sơ đồ kiến trúc trên giấy / whiteboard
□ Giải thích to để kiểm tra logic
□ Chuẩn bị câu hỏi để hỏi ngược nhà tuyển dụng
```

### T-1 ngày

```
□ Ôn nhanh top 10 câu hỏi quan trọng nhất
□ Kiểm tra lại 3 STAR stories một lần nữa
□ Ngủ đủ giấc — quan trọng hơn học thêm
□ Chuẩn bị môi trường (internet, camera, tai nghe nếu online)
```

### Ngày Phỏng Vấn

```
□ Đọc lại JD (Job Description — Mô Tả Công Việc) và highlight AWS terms
□ Ôn nhanh 5 điểm chính về company's tech stack
□ Sẵn sàng notebook để ghi chú câu hỏi
□ Hỏi lại ngay khi không hiểu câu hỏi
```

---

## Câu Hỏi Nên Hỏi Ngược Nhà Tuyển Dụng

Cuối buổi phỏng vấn, luôn hỏi ít nhất 2-3 câu:

```
Về Kỹ Thuật:
- "Team hiện đang dùng EC2, ECS hay EKS cho production workloads?"
- "Chiến lược cost optimization (tối ưu chi phí) của team hiện tại là gì?"
- "Team có incident on-call rotation (lịch trực sự cố) không?"

Về Văn Hóa:
- "Team deploy bao nhiêu lần một tuần?"
- "Quy trình review code và deploy lên production như thế nào?"
- "Cơ hội học hỏi và phát triển kỹ năng cloud như thế nào?"

Về Vai Trò:
- "Dự án đầu tiên tôi sẽ tham gia nếu được nhận là gì?"
- "Định nghĩa thành công trong 6 tháng đầu cho vị trí này là gì?"
```

---

## Tài Nguyên Bổ Sung

| Tài Nguyên                           | Mục Đích                              |
| ------------------------------------ | ------------------------------------- |
| AWS Well-Architected Framework       | Nắm 6 pillars (trụ cột) thiết kế tốt |
| AWS re:Invent talks (YouTube)        | Học từ các engineer AWS thực tế       |
| AWS Skill Builder free labs          | Thực hành không tốn tiền              |
| Exam Readiness: SAA-C03              | Câu hỏi mức Solutions Architect       |
| LeetCode Discuss — AWS questions     | Câu hỏi thực tế từ cộng đồng         |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
