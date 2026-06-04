# 07 — Advanced Networking (Mạng Nâng Cao AWS)

> **Mức độ:** Nâng cao (Advanced) | **Thời gian học ước tính:** 3–5 ngày

---

## Giới Thiệu

Sau khi nắm vững các nền tảng VPC, Security Groups, Load Balancer, Route 53 và CloudFront, bước tiếp theo là làm chủ các dịch vụ mạng **nâng cao** của AWS. Những dịch vụ này giải quyết các bài toán phức tạp hơn:

- Kết nối **riêng tư** (private) tới AWS services mà không đi qua internet công cộng
- Chia sẻ dịch vụ nội bộ giữa các VPC hoặc tài khoản AWS khác nhau
- Tăng tốc độ và độ tin cậy cho ứng dụng toàn cầu
- Quản lý địa chỉ IP nâng cao (Elastic IP, ENI, IPv6)

Các dịch vụ trong topic này thường xuất hiện trong các câu hỏi phỏng vấn cấp Senior/Principal và trong các kỳ thi chứng chỉ AWS (SAA-C03, ANS-C01).

---

## Danh Sách Chủ Đề Con

| # | File | Chủ đề | Mô tả ngắn |
|---|------|--------|-----------|
| 1 | [1-vpc-endpoints.md](./1-vpc-endpoints.md) | VPC Endpoints | Kết nối tới AWS services qua mạng riêng, không qua internet |
| 2 | [2-privatelink.md](./2-privatelink.md) | AWS PrivateLink | Expose dịch vụ riêng tư giữa các VPC / tài khoản qua private IP |
| 3 | [3-global-accelerator.md](./3-global-accelerator.md) | Global Accelerator | Tăng tốc ứng dụng toàn cầu bằng Anycast IP và AWS backbone |
| 4 | [4-elastic-ip-eni.md](./4-elastic-ip-eni.md) | Elastic IP & ENI | Quản lý địa chỉ IP tĩnh và giao diện mạng ảo |
| 5 | [5-ipv6.md](./5-ipv6.md) | IPv6 & Dual-Stack | Kiến trúc IPv4+IPv6 song song trong VPC |

---

## Tại Sao Cần Các Dịch Vụ Nâng Cao Này?

### Vấn đề với kiến trúc đơn giản

```
EC2 Instance → Internet Gateway → Internet → S3 / DynamoDB / SSM
```

Kiến trúc trên có các vấn đề:
- **Bảo mật:** Traffic đi qua internet công cộng, có thể bị intercepted
- **Chi phí:** NAT Gateway tính phí theo GB data processed (~$0.045/GB)
- **Latency:** Routing qua internet thêm độ trễ không cần thiết
- **Compliance:** Nhiều tổ chức (tài chính, y tế) yêu cầu data không được rời khỏi mạng riêng

### Giải pháp với Advanced Networking

```
EC2 Instance → VPC Endpoint → AWS Private Network → S3 / DynamoDB / SSM
```

---

## Use Cases — Khi Nào Dùng Dịch Vụ Nào?

### VPC Endpoints

**Dùng khi:**
- EC2 hoặc Lambda trong private subnet cần gọi AWS services (S3, DynamoDB, SSM, Secrets Manager, ECR, SQS, SNS...)
- Muốn tiết kiệm chi phí NAT Gateway (Gateway Endpoints miễn phí)
- Cần tuân thủ compliance: data không được ra internet
- Muốn kiểm soát truy cập vào S3 bucket theo VPC (endpoint policy)

**Ví dụ thực tế:**
- Lambda function xử lý dữ liệu nhạy cảm cần đọc từ S3 mà không qua internet
- ECS task trong private subnet cần pull image từ ECR
- EC2 cần gọi SSM Parameter Store để lấy secrets

### AWS PrivateLink

**Dùng khi:**
- Muốn expose một dịch vụ nội bộ (microservice) cho các VPC khác / account khác tiêu dùng
- Công ty SaaS muốn cho phép khách hàng kết nối tới API của mình qua private IP
- Cần kết nối giữa hai VPC có overlapping CIDR (VPC Peering không làm được)
- Muốn consumer không thể truy cập toàn bộ VPC của provider, chỉ truy cập được dịch vụ cụ thể

**Ví dụ thực tế:**
- Datadog, Snowflake, Confluent Cloud đều dùng PrivateLink để kết nối với customer VPC
- Microservice nội bộ dùng NLB + PrivateLink để expose API cho các team khác trong cùng tổ chức

### AWS Global Accelerator

**Dùng khi:**
- Ứng dụng có người dùng ở nhiều khu vực địa lý khác nhau
- Cần failover nhanh (không phụ thuộc DNS TTL) khi một region gặp sự cố
- Ứng dụng dùng giao thức TCP/UDP (gaming, VoIP, real-time data)
- Cần địa chỉ IP tĩnh (2 Anycast IP) dù backend thay đổi

**Ví dụ thực tế:**
- Game server: người chơi kết nối tới Anycast IP, được route tới game server gần nhất
- Ứng dụng mobile banking: cần độ tin cậy cao, failover tức thì
- Blue/Green deployment toàn cầu: điều chỉnh traffic dial giữa hai region

### Elastic IP & ENI

**Dùng khi:**
- Cần địa chỉ IP tĩnh cho bastion host, NAT instance
- Xây dựng high-availability bằng cách di chuyển Elastic IP sang instance dự phòng
- Network appliance (firewall, VPN concentrator) cần nhiều giao diện mạng
- Container networking trên EKS cần secondary private IPs

### IPv6

**Dùng khi:**
- Ứng dụng IoT (cần không gian địa chỉ lớn)
- Yêu cầu kết nối trực tiếp từ internet tới instance (không qua NAT)
- Tuân thủ yêu cầu của chính phủ/tổ chức về IPv6
- Chuẩn bị cho tương lai khi IPv4 công cộng khan hiếm hơn

---

## Mối Quan Hệ Giữa Các Dịch Vụ

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS Cloud                                    │
│                                                                      │
│  ┌──────────────┐    VPC Endpoint    ┌──────────────────────┐       │
│  │  Your VPC    │ ─────────────────> │  AWS Services        │       │
│  │  (Consumer)  │   (Gateway/IF)     │  S3, DynamoDB, SSM   │       │
│  └──────┬───────┘                    └──────────────────────┘       │
│         │                                                            │
│         │ Interface Endpoint         ┌──────────────────────┐       │
│         │ (PrivateLink)              │  Provider VPC        │       │
│         └──────────────────────────> │  NLB → Your Service  │       │
│                                      └──────────────────────┘       │
│                                                                      │
│  ┌──────────────┐                                                    │
│  │  Global      │  Anycast IP        ┌──────────────────────┐       │
│  │  Accelerator │ ─────────────────> │  ALB / NLB / EC2     │       │
│  │  (Edge PoPs) │   AWS Backbone     │  (Multiple Regions)  │       │
│  └──────────────┘                    └──────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

### VPC Endpoints vs PrivateLink

| Khía cạnh | VPC Endpoints | PrivateLink |
|-----------|--------------|-------------|
| Kết nối tới | AWS-managed services | Dịch vụ tùy chỉnh (custom services) |
| Provider | AWS | Bạn hoặc bên thứ ba (SaaS) |
| Cơ chế | Gateway (route table) hoặc ENI | Luôn dùng ENI + NLB |
| Chi phí Gateway | Miễn phí | N/A |
| Chi phí Interface | ~$0.01/giờ + data | ~$0.01/giờ + data |

### VPC Endpoints vs Global Accelerator

| Khía cạnh | VPC Endpoints | Global Accelerator |
|-----------|--------------|-------------------|
| Mục đích | Private access tới AWS services | Tăng tốc traffic từ internet |
| Traffic direction | Outbound từ VPC | Inbound từ người dùng |
| Internet | Không dùng | Dùng (từ client tới Edge) |
| Use case | Internal/backend | Public-facing applications |

---

## Roadmap Học Topic Này

### Tuần 1 — Nền tảng (Prerequisites)

Đảm bảo đã nắm vững:
- [ ] VPC, Subnet, Route Table, Internet Gateway
- [ ] Security Groups và Network ACLs
- [ ] NAT Gateway và NAT Instance
- [ ] ALB, NLB — cơ chế hoạt động

### Tuần 2 — VPC Endpoints & PrivateLink

- [ ] Đọc [1-vpc-endpoints.md](./1-vpc-endpoints.md) — hiểu Gateway vs Interface
- [ ] Lab: Tạo Gateway Endpoint cho S3, test từ EC2 private subnet
- [ ] Lab: Tạo Interface Endpoint cho SSM, so sánh chi phí vs NAT Gateway
- [ ] Đọc [2-privatelink.md](./2-privatelink.md) — kiến trúc Provider/Consumer
- [ ] Lab: Tạo NLB + Endpoint Service, consumer endpoint trong VPC khác

### Tuần 3 — Global Accelerator & IP Management

- [ ] Đọc [3-global-accelerator.md](./3-global-accelerator.md)
- [ ] Lab: Tạo Accelerator với 2 endpoint groups ở 2 regions
- [ ] Test failover bằng cách tắt endpoint
- [ ] Đọc [4-elastic-ip-eni.md](./4-elastic-ip-eni.md)
- [ ] Lab: Gán/tháo ENI giữa 2 instances

### Tuần 4 — IPv6 & Tổng Hợp

- [ ] Đọc [5-ipv6.md](./5-ipv6.md)
- [ ] Lab: Tạo dual-stack VPC, test connectivity IPv6
- [ ] Ôn tập toàn bộ topic, làm practice questions
- [ ] Đọc AWS Well-Architected Framework — Network pillar

---

## Điều Hướng

- [← 06-connectivity](../06-connectivity/) — VPN, Direct Connect, Transit Gateway
- [→ INDEX.md](../INDEX.md) — Chỉ mục toàn bộ AWS Networking
- [1-vpc-endpoints.md](./1-vpc-endpoints.md) — Bắt đầu chủ đề 1
