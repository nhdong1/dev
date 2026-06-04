# 🎯 Interview Prep — Chuẩn Bị Phỏng Vấn AWS Networking

> Hướng dẫn toàn diện giúp bạn tự tin trả lời mọi câu hỏi phỏng vấn về AWS Networking — từ câu hỏi lý thuyết đến system design và incident stories.

## 📁 Cấu Trúc Thư Mục

```
10-interview-prep/
├── README.md                  Tổng quan & lộ trình chuẩn bị (file này)
├── 1-INTERVIEW_GUIDE.md       Top 20 câu hỏi AWS Networking kèm đáp án chi tiết
├── 2-star-stories.md          Mẫu câu chuyện incident theo phương pháp STAR
├── 3-system-design-scenarios.md  Kịch bản thiết kế hệ thống thực tế
├── 4-hands-on-exercises.md    Bài tập thực hành có hướng dẫn step-by-step
└── 5-90-day-study-plan.md     Kế hoạch học 90 ngày có lộ trình chi tiết
```

---

## 🎯 Mục Tiêu Của Section Này

Sau khi hoàn thành `10-interview-prep/`, bạn có thể:

- ✅ Trả lời tự tin **top 20 câu hỏi AWS Networking** hàng đầu
- ✅ Kể **2-3 incident stories** theo định dạng STAR thuyết phục
- ✅ Thiết kế **kiến trúc mạng phức tạp** trên giấy trong 30 phút
- ✅ Thảo luận **trade-offs** giữa các dịch vụ AWS một cách sâu sắc
- ✅ Có **kế hoạch học 90 ngày** rõ ràng để đạt mục tiêu

---

## 📋 Danh Sách Kiểm Tra Trước Phỏng Vấn

### Kiến Thức Lý Thuyết — Cần Nắm Chắc

- [ ] VPC (Virtual Private Cloud) — thiết kế, CIDR, subnets, routing
- [ ] Security Groups (Nhóm Bảo Mật) vs Network ACLs — stateful vs stateless
- [ ] ALB (Application Load Balancer) vs NLB (Network Load Balancer) — khi nào dùng cái nào
- [ ] Route 53 — tất cả 7 routing policies với use case cụ thể
- [ ] CloudFront CDN (Content Delivery Network — Mạng Phân Phối Nội Dung) — cache, OAC/OAI, behaviors
- [ ] Transit Gateway (Cổng Trung Chuyển) vs VPC Peering (Kết Nối Ngang Hàng VPC)
- [ ] Direct Connect vs VPN Site-to-Site — trade-offs
- [ ] VPC Endpoints (Điểm Cuối VPC) — Gateway vs Interface

### Kỹ Năng Thực Hành — Cần Đã Làm

- [ ] Tạo VPC 3-tier từ đầu (không dùng wizard)
- [ ] Cấu hình ALB với HTTPS và certificate từ ACM (AWS Certificate Manager)
- [ ] Thiết lập Route 53 failover routing với health checks
- [ ] Deploy CloudFront distribution với S3 origin và OAC
- [ ] Phân tích VPC Flow Logs (Nhật Ký Luồng VPC) để debug kết nối

### Câu Chuyện Thực Tế — Cần Chuẩn Bị

- [ ] Incident liên quan đến network outage (mất kết nối mạng)
- [ ] Tối ưu performance (hiệu suất) mạng hoặc CDN
- [ ] Security incident hoặc cải thiện security posture
- [ ] Migration (di chuyển) workload giữa regions hoặc environments
- [ ] Cost optimization (tối ưu chi phí) cho networking

---

## 🗺️ Lộ Trình Chuẩn Bị

### 2 Tuần Trước Phỏng Vấn

```
Tuần 1: Ôn lý thuyết & đọc 1-INTERVIEW_GUIDE.md
- Ngày 1-2: VPC, Subnets, Security Groups, NACLs
- Ngày 3-4: ALB/NLB, Target Groups, Health Checks
- Ngày 5-6: Route 53, CloudFront
- Ngày 7: Connectivity (VPN, Direct Connect, TGW)

Tuần 2: Practice & Polish
- Ngày 1-2: Luyện vẽ sơ đồ từ 3-system-design-scenarios.md
- Ngày 3-4: Chuẩn bị stories từ 2-star-stories.md
- Ngày 5: Thực hành từ 4-hands-on-exercises.md
- Ngày 6: Mock interview với đồng nghiệp
- Ngày 7: Nghỉ ngơi & ôn nhẹ
```

### 1 Tuần Trước Phỏng Vấn

```
- Ôn lại top 20 câu hỏi từ 1-INTERVIEW_GUIDE.md
- Luyện nói to các câu trả lời (không đọc)
- Vẽ sơ đồ 3-tier VPC từ bộ nhớ
- Đảm bảo có 3 incident stories sẵn sàng
```

### 1 Ngày Trước Phỏng Vấn

```
- Đọc lại checklist trong README này
- Ôn 10 câu hỏi quan trọng nhất
- Chuẩn bị câu hỏi để hỏi ngược interviewer
- Nghỉ ngơi đầy đủ
```

---

## 💡 Câu Hỏi Nên Hỏi Ngược Interviewer

Câu hỏi thông minh thể hiện bạn suy nghĩ chuyên sâu:

1. **"Team hiện tại đang dùng networking pattern nào cho multi-region?"** — Thể hiện quan tâm đến kiến trúc thực tế
2. **"Chiến lược disaster recovery (khôi phục sau thảm họa) của công ty là gì? RTO/RPO bao nhiêu?"** — Thể hiện hiểu biết về operations
3. **"Đội ngũ có đang dùng Infrastructure as Code (Hạ Tầng Dưới Dạng Mã) cho networking không?"** — Thể hiện quan tâm đến automation
4. **"Challenges lớn nhất về mạng mà team đang đối mặt là gì?"** — Giúp bạn hiểu thực tế công việc

---

## 📊 Ma Trận Câu Hỏi Theo Cấp Độ

| Cấp Độ     | Loại Câu Hỏi                                | Ví Dụ                                                             |
| ---------- | ------------------------------------------- | ----------------------------------------------------------------- |
| Junior     | Kiến thức cơ bản, định nghĩa, use cases     | "Security Group là gì? Khác NACL như thế nào?"                   |
| Mid-level  | Thiết kế, so sánh, troubleshooting          | "Thiết kế VPC cho e-commerce với high availability"               |
| Senior     | Trade-offs, kiến trúc phức tạp, optimization | "Multi-region active-active với Route 53 và Global Accelerator?"  |
| Architect  | System design toàn diện, cost, compliance   | "Thiết kế global network cho fintech với PCI-DSS compliance"      |

---

## 🔗 Điều Hướng

| File                              | Nội Dung                                     | Thời Gian Ôn  |
| --------------------------------- | -------------------------------------------- | ------------- |
| [1-INTERVIEW_GUIDE.md](./1-INTERVIEW_GUIDE.md) | Top 20 Q&A chi tiết              | 3-4 giờ       |
| [2-star-stories.md](./2-star-stories.md)       | Templates & ví dụ STAR stories   | 2-3 giờ       |
| [3-system-design-scenarios.md](./3-system-design-scenarios.md) | 5 kịch bản design | 4-5 giờ |
| [4-hands-on-exercises.md](./4-hands-on-exercises.md) | Lab exercises thực hành    | 6-8 giờ       |
| [5-90-day-study-plan.md](./5-90-day-study-plan.md)   | Kế hoạch học 90 ngày       | Tham khảo     |

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
