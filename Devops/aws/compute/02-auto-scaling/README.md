# 🔄 Auto Scaling — Tự Động Co Giãn Hạ Tầng AWS

> **Auto Scaling** — Tự Động Co Giãn — là cơ chế AWS tự động tăng/giảm số lượng EC2 instance (máy chủ ảo) theo nhu cầu thực tế, đảm bảo ứng dụng luôn sẵn sàng với chi phí tối ưu.

---

## 📚 Mục Lục

1. [Tại Sao Cần Auto Scaling?](#tại-sao-cần-auto-scaling)
2. [Các Thành Phần Cốt Lõi](#các-thành-phần-cốt-lõi)
3. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
4. [Các Loại Scaling](#các-loại-scaling)
5. [Tổng Quan Các File](#tổng-quan-các-file)
6. [Khi Nào Dùng Auto Scaling?](#khi-nào-dùng-auto-scaling)
7. [Câu Hỏi Phỏng Vấn Quan Trọng](#câu-hỏi-phỏng-vấn-quan-trọng)

---

## 🎯 Tại Sao Cần Auto Scaling?

### Vấn Đề Khi Không Có Auto Scaling

```
Traffic thực tế:
  Giờ thấp điểm: ████ (cần 2 servers)
  Giờ cao điểm:  ████████████████████ (cần 20 servers)

Không có Auto Scaling → chọn một trong hai:
  ❌ Cấp phát 2 servers → bị quá tải giờ cao điểm (outage)
  ❌ Cấp phát 20 servers → lãng phí 90% chi phí giờ thấp điểm
```

### Lợi Ích Của Auto Scaling

| Lợi Ích                          | Mô Tả                                                                |
| --------------------------------- | -------------------------------------------------------------------- |
| **High Availability**             | Tự động thay thế instance bị lỗi, đảm bảo số lượng tối thiểu       |
| **Cost Optimization**             | Chỉ trả tiền cho capacity thực sự cần                               |
| **Performance**                   | Tự động thêm instance khi tải tăng, không bị chậm                  |
| **Fault Tolerance**               | Phân phối instance trên nhiều AZ (Availability Zone — Vùng Khả Dụng)|
| **Predictive Scaling**            | Dự báo nhu cầu và chuẩn bị capacity trước                          |

---

## 🏗️ Các Thành Phần Cốt Lõi

### 1. Auto Scaling Group — ASG (Nhóm Tự Động Co Giãn)

**ASG** là nhóm EC2 instance được quản lý cùng nhau với chung cấu hình scaling. ASG định nghĩa:
- **Min capacity** — số instance tối thiểu luôn chạy
- **Max capacity** — số instance tối đa được phép tạo
- **Desired capacity** — số instance mong muốn hiện tại

```
ASG Configuration Example:
┌─────────────────────────────────────────────┐
│  Auto Scaling Group: "web-servers-asg"      │
│                                             │
│  Min:     2  ──────── Luôn có ít nhất 2    │
│  Desired: 4  ──────── Hiện tại đang chạy 4 │
│  Max:    10  ──────── Không tạo quá 10      │
│                                             │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐      │
│  │  EC2 │ │  EC2 │ │  EC2 │ │  EC2 │      │
│  │ AZ-a │ │ AZ-b │ │ AZ-a │ │ AZ-b │      │
│  └──────┘ └──────┘ └──────┘ └──────┘      │
└─────────────────────────────────────────────┘
```

### 2. Launch Template — Mẫu Khởi Chạy

**Launch Template** định nghĩa cấu hình cho từng EC2 instance trong ASG:
- AMI (Amazon Machine Image — Ảnh Máy Ảo) ID
- Instance type (loại máy chủ)
- Security Groups (nhóm bảo mật)
- IAM Instance Profile (hồ sơ quyền truy cập)
- User Data (script khởi tạo)

### 3. Scaling Policies — Chính Sách Co Giãn

Ba loại chính sách điều khiển khi nào và bao nhiêu instance được thêm/xóa:

| Loại Chính Sách         | Khi Nào Dùng                                          |
| ----------------------- | ----------------------------------------------------- |
| **Target Tracking**     | Muốn duy trì một metric ở mức cố định (ví dụ CPU=70%)|
| **Step Scaling**        | Muốn scale khác nhau tùy mức độ tải                  |
| **Scheduled Scaling**   | Biết trước thời điểm tải cao (ví dụ: 8h sáng hàng ngày)|

---

## 🗺️ Kiến Trúc Tổng Quan

```
                    ┌──────────────────────┐
                    │   CloudWatch Alarms  │
                    │   (Cảnh Báo Giám Sát)│
                    └──────────┬───────────┘
                               │ Trigger (Kích Hoạt)
                    ┌──────────▼───────────┐
                    │  Scaling Policy      │
                    │  (Chính Sách Co Giãn)│
                    └──────────┬───────────┘
                               │ Điều chỉnh Desired Capacity
                    ┌──────────▼───────────┐
                    │  Auto Scaling Group  │
                    │  Min=2, Max=10       │
                    └──────────┬───────────┘
                               │ Launch/Terminate
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼───────┐ ┌─────▼──────┐ ┌──────▼──────┐
     │  EC2 Instance  │ │EC2 Instance│ │EC2 Instance │
     │  us-east-1a    │ │us-east-1b  │ │us-east-1c   │
     │  (AZ-a)        │ │(AZ-b)      │ │(AZ-c)       │
     └────────────────┘ └────────────┘ └─────────────┘
              │                │                │
     ┌────────▼────────────────▼────────────────▼──────┐
     │        Application Load Balancer (ALB)           │
     │        (Cân Bằng Tải Ứng Dụng)                  │
     └──────────────────────────────────────────────────┘
```

---

## 📐 Các Loại Scaling

### Horizontal Scaling — Mở Rộng Theo Chiều Ngang (Scale Out/In)

Thêm hoặc bớt số lượng instance. Đây là cách Auto Scaling EC2 hoạt động:

```
Scale Out (Mở Rộng):  [EC2][EC2] → [EC2][EC2][EC2][EC2]
Scale In  (Thu Hẹp):  [EC2][EC2][EC2][EC2] → [EC2][EC2]
```

**Ưu điểm:** Không giới hạn, không downtime, phù hợp stateless apps (ứng dụng không trạng thái).

### Vertical Scaling — Mở Rộng Theo Chiều Dọc (Scale Up/Down)

Thay đổi kích cỡ instance (ví dụ: t3.micro → t3.large). Cần stop và restart instance:

```
Scale Up:   t3.micro (2 vCPU, 1GB)  → c5.4xlarge (16 vCPU, 32GB)
Scale Down: c5.4xlarge               → t3.large (2 vCPU, 8GB)
```

**Nhược điểm:** Yêu cầu downtime, có giới hạn phần cứng. EC2 Auto Scaling dùng horizontal scaling.

### Dynamic Scaling — Co Giãn Động

Phản ứng theo metric thực tế (CPU, requests/giây, SQS queue depth...):

```
Traffic tăng → CPU tăng → CloudWatch Alarm → ASG tăng desired → Launch instances
Traffic giảm → CPU giảm → CloudWatch Alarm → ASG giảm desired → Terminate instances
```

### Predictive Scaling — Co Giãn Dự Đoán

Machine Learning (học máy) phân tích lịch sử traffic để chuẩn bị capacity trước:

```
Hệ thống nhận ra: Mỗi thứ 2 lúc 9h sáng traffic tăng 3x
→ Tự động pre-scale trước 5-15 phút
→ Instances đã sẵn sàng khi traffic thực sự đến
```

---

## 📁 Tổng Quan Các File

| File                           | Nội Dung                                              | Độ Khó |
| ------------------------------ | ----------------------------------------------------- | ------ |
| **README.md** (file này)       | Tổng quan & kiến trúc Auto Scaling                   | ⭐     |
| **1-auto-scaling-groups.md**   | ASG chi tiết — capacity, health checks, multi-AZ     | ⭐⭐   |
| **2-launch-templates.md**      | Launch Templates vs Launch Configurations, cấu hình  | ⭐⭐   |
| **3-scaling-policies.md**      | Target Tracking, Step, Scheduled, Predictive policies| ⭐⭐⭐ |
| **4-spot-mixed-instances.md**  | Spot trong ASG, Mixed Instance Policy, tiết kiệm chi phí | ⭐⭐⭐ |
| **5-lifecycle-hooks.md**       | Lifecycle Hooks, Warm Pools, Instance Refresh        | ⭐⭐⭐ |

---

## 🤔 Khi Nào Dùng Auto Scaling?

### Nên Dùng Auto Scaling Khi

- ✅ Traffic có sự biến động (giờ cao điểm / thấp điểm)
- ✅ Ứng dụng stateless (không lưu session trên server)
- ✅ Muốn High Availability tự động trên nhiều AZ
- ✅ Muốn tối ưu chi phí — không trả tiền cho capacity không dùng
- ✅ Muốn tự động thay thế instance bị lỗi

### Cân Nhắc Khi

- ⚠️ Ứng dụng stateful (có session hoặc local storage) — cần giải pháp riêng
- ⚠️ Database instance — dùng RDS Auto Scaling hoặc Aurora Serverless thay thế
- ⚠️ Workload cố định, không biến động — Reserved Instance tiết kiệm hơn
- ⚠️ Bootstrap time (thời gian khởi động) quá dài — xem xét Warm Pools

### Auto Scaling vs Giải Pháp Thay Thế

| Tình Huống                         | Giải Pháp Tốt Nhất                              |
| ----------------------------------- | ----------------------------------------------- |
| EC2 workload biến động              | **EC2 Auto Scaling** (chủ đề này)               |
| ECS container workload              | **ECS Service Auto Scaling** (Application Auto Scaling)|
| EKS Kubernetes workload             | **HPA** — Horizontal Pod Autoscaler + **Karpenter** |
| DynamoDB throughput                 | **DynamoDB Auto Scaling**                        |
| Aurora serverless                   | **Aurora Serverless v2** (tự động)               |
| Lambda                              | Không cần — Lambda tự scale                     |

---

## 🎓 Câu Hỏi Phỏng Vấn Quan Trọng

### Câu Hỏi Thường Gặp

**Q: Giải thích cách Auto Scaling Group hoạt động khi một instance bị lỗi?**

> ASG liên tục theo dõi health check (kiểm tra sức khỏe) của từng instance. Khi instance bị đánh dấu unhealthy (không khỏe), ASG tự động terminate (xóa) instance đó và launch (khởi chạy) instance mới để duy trì desired capacity. Nếu instance được gắn với ALB, ASG chờ instance mới pass health check trước khi gửi traffic.

**Q: Sự khác biệt giữa min/max/desired capacity trong ASG?**

> - **Min capacity**: Số instance tối thiểu luôn chạy, kể cả khi không có traffic
> - **Max capacity**: Giới hạn trên — ASG không tạo thêm instance quá con số này, để kiểm soát chi phí
> - **Desired capacity**: Số instance ASG đang cố gắng duy trì tại thời điểm hiện tại. Scaling policies thay đổi desired capacity, ASG sau đó thêm/bớt instance để đạt được con số đó

**Q: Cooldown period (thời gian hạ nhiệt) trong Auto Scaling là gì?**

> Sau mỗi scaling activity, ASG chờ một khoảng thời gian (mặc định 300 giây) trước khi thực hiện scaling tiếp theo. Mục đích: cho phép instance mới khởi động hoàn tất và các metric ổn định lại, tránh over-scaling (scale quá mức cần thiết).

**Q: Target Tracking Policy khác gì Step Scaling Policy?**

> - **Target Tracking**: AWS tự động tính toán bao nhiêu instance cần thêm/bớt để duy trì metric ở mức target (ví dụ CPU=70%). Đơn giản, ít cấu hình, phù hợp hầu hết use cases.
> - **Step Scaling**: Bạn định nghĩa thủ công "nếu CPU > 80% thì thêm 2 instance, nếu CPU > 90% thì thêm 5 instance". Linh hoạt hơn nhưng phức tạp hơn.

---

## 🔗 Điều Hướng

| Tiếp Theo                      | Quay Lại                                                    |
| ------------------------------ | ----------------------------------------------------------- |
| [ASG Chi Tiết →](1-auto-scaling-groups.md) | [← EC2 Fundamentals](../01-ec2-fundamentals/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
