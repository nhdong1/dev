# Dev — Kho Kiến Thức Kỹ Thuật Toàn Diện

> Tài liệu học tập và tham khảo bằng tiếng Việt (UTF-8) dành cho kỹ sư phần mềm, DBA (Database Administrator — Quản Trị Viên Cơ Sở Dữ Liệu) và DevOps Engineer (Kỹ Sư DevOps). Thuật ngữ kỹ thuật giữ nguyên tiếng Anh kèm giải thích trong ngoặc.

## Mục Lục

1. [Giới Thiệu](#giới-thiệu)
2. [Cấu Trúc Dự Án](#cấu-trúc-dự-án)
3. [Ba Trụ Cột Kiến Thức](#ba-trụ-cột-kiến-thức)
4. [Lộ Trình Học Đề Xuất](#lộ-trình-học-đề-xuất)
5. [Ma Trận Kỹ Năng Theo Vai Trò](#ma-trận-kỹ-năng-theo-vai-trò)
6. [Cách Sử Dụng Kho Tài Liệu](#cách-sử-dụng-kho-tài-liệu)
7. [Quy Ước Viết Tài Liệu](#quy-ước-viết-tài-liệu)

---

## Giới Thiệu

**Dev** là kho tài liệu tập trung, tổ chức theo lộ trình học từ cơ bản đến nâng cao, bao phủ ba mảng cốt lõi trong phát triển và vận hành hệ thống phần mềm hiện đại:

| Trụ cột | Mục tiêu | Đối tượng chính |
| ------- | -------- | --------------- |
| [Developer](./Developer/) | Nắm vững ngôn ngữ, framework và kỹ năng lập trình | Backend Developer, Frontend Developer, Full-stack Engineer |
| [Database](./Database/) | Vận hành, tối ưu và bảo mật cơ sở dữ liệu | DBA, Backend Engineer, SRE (Site Reliability Engineer — Kỹ Sư Độ Tin Cậy Hệ Thống) |
| [Devops](./Devops/) | Tự động hóa hạ tầng, CI/CD (Continuous Integration / Continuous Delivery — Tích Hợp Liên Tục / Phân Phối Liên Tục) và cloud | DevOps Engineer, Platform Engineer, Cloud Engineer |

Mỗi module con có README riêng mô tả lộ trình học, ma trận kỹ năng và danh sách chủ đề chi tiết. Các bài viết được viết dưới dạng Markdown (`.md`) để dễ đọc trên GitHub, VS Code hoặc bất kỳ trình xem Markdown nào.

---

## Cấu Trúc Dự Án

```
dev/
├── Developer/          # Lập trình ứng dụng & kiến trúc phần mềm
│   ├── .net/           # C# .NET & ASP.NET Core
│   ├── spring-boot/    # Java Spring Boot
│   ├── reactjs/        # React.js & hệ sinh thái Frontend
│   └── basic/          # Nền tảng Computer Science
├── Database/           # Quản trị & vận hành CSDL
│   ├── 01-co-ban/
│   ├── 02-sao-luu-phuc-hoi/
│   ├── 03-ha-replication/
│   ├── 04-toi-uu-hieu-suat/
│   ├── 05-bao-mat-tuan-thu/
│   ├── 06-schema-migrations/
│   ├── 07-cloud-database/
│   └── 08-phong-van/
└── Devops/             # Hạ tầng, cloud & tự động hóa
    ├── aws/            # Amazon Web Services
    ├── k8s/            # Kubernetes (K8s — Hệ Thống Điều Phối Container)
    ├── terraform/      # IaC (Infrastructure as Code — Hạ Tầng Dưới Dạng Mã)
    ├── github-action/  # CI/CD trên GitHub
    ├── jenkins/        # Jenkins Pipeline
    └── circleci/       # CircleCI
```

---

## Ba Trụ Cột Kiến Thức

### Developer — Phát Triển Phần Mềm

| Module | Nội dung chính | README |
| ------ | -------------- | ------ |
| **.NET** | C#, ASP.NET Core, Entity Framework Core, Clean Architecture, DDD (Domain-Driven Design — Thiết Kế Hướng Miền) | [Developer/.net/README.md](./Developer/.net/README.md) |
| **Spring Boot** | REST API, Spring Data JPA, Spring Security, Microservices, Kafka | [Developer/spring-boot/README.md](./Developer/spring-boot/README.md) |
| **React.js** | Hooks, State Management, Routing, RSC (React Server Components — Component Phía Server), Testing | [Developer/reactjs/README.md](./Developer/reactjs/README.md) |
| **Basic** | OS (Operating System — Hệ Điều Hành), Networking, Concurrency, Distributed Systems | [Developer/basic/09-computer-science-fundamentals/README.md](./Developer/basic/09-computer-science-fundamentals/README.md) |

### Database — Quản Trị Cơ Sở Dữ Liệu

| Chủ đề | Nội dung chính |
| ------ | -------------- |
| **Cơ bản** | ACID (Atomicity, Consistency, Isolation, Durability), Transaction, Index, Connection Pooling |
| **Sao lưu & Phục hồi** | RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi), RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi), PITR (Point-in-Time Recovery — Phục Hồi Theo Thời Điểm) |
| **HA & Replication** | High Availability (Tính Sẵn Sàng Cao), Failover, Read Replica, Split-brain |
| **Tối ưu hiệu suất** | EXPLAIN Plan, Index Design, Query Tuning, Deadlock |
| **Bảo mật & Tuân thủ** | RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò), RLS (Row-Level Security — Bảo Mật Cấp Hàng), GDPR, PCI-DSS |
| **Schema Migrations** | Flyway, Liquibase, Zero-downtime Deployment (Triển Khai Không Gián Đoạn) |
| **Cloud Database** | AWS RDS, Aurora, DynamoDB; Azure SQL, Cosmos DB |

→ Xem chi tiết: [Database/README.md](./Database/README.md)

### Devops — Hạ Tầng & Vận Hành

| Module | Nội dung chính | README |
| ------ | -------------- | ------ |
| **AWS** | IAM, VPC (Virtual Private Cloud — Mạng Riêng Ảo), EC2, S3, RDS, Lambda, EKS (Elastic Kubernetes Service) | [Devops/aws/iam/README.md](./Devops/aws/iam/README.md) và các module con trong `Devops/aws/` |
| **Kubernetes** | Pod, Deployment, Service, HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang), Helm, GitOps, ArgoCD | [Devops/k8s/README.md](./Devops/k8s/README.md) |
| **Terraform** | State Management, Modules, Remote State, CI/CD Integration, Policy as Code | [Devops/terraform/README.md](./Devops/terraform/README.md) |
| **GitHub Actions** | Workflow, Jobs, Secrets, Matrix Strategy, Deployment | [Devops/github-action/README.md](./Devops/github-action/README.md) |
| **Jenkins** | Pipeline as Code, Shared Libraries, Distributed Builds | [Devops/jenkins/](./Devops/jenkins/) |
| **CircleCI** | Orbs, Workflows, Executors, Parallel Jobs | [Devops/circleci/README.md](./Devops/circleci/README.md) |

---

## Lộ Trình Học Đề Xuất

### Giai Đoạn 1 — Nền Tảng (Tuần 1–4)

- [ ] Computer Science Fundamentals — OS, Networking, Concurrency
- [ ] Git — Version Control (Quản Lý Phiên Bản Mã Nguồn)
- [ ] SQL cơ bản và tính chất ACID
- [ ] Linux CLI (Command Line Interface — Giao Diện Dòng Lệnh) cơ bản

### Giai Đoạn 2 — Phát Triển Ứng Dụng (Tuần 5–12)

- [ ] Chọn một stack Backend: .NET hoặc Spring Boot
- [ ] Thiết kế REST API, Authentication (Xác Thực) & Authorization (Phân Quyền)
- [ ] ORM (Object-Relational Mapping — Ánh Xạ Đối Tượng–Quan Hệ): EF Core hoặc JPA
- [ ] Frontend với React.js (nếu theo hướng Full-stack)
- [ ] Unit Testing & Integration Testing (Kiểm Thử Đơn Vị & Tích Hợp)

### Giai Đoạn 3 — Cơ Sở Dữ Liệu & Vận Hành (Tuần 13–20)

- [ ] Backup Strategy (Chiến Lược Sao Lưu), Replication & Failover
- [ ] Query Optimization (Tối Ưu Truy Vấn) và Index Design
- [ ] Docker — Containerization (Đóng Gói Ứng Dụng)
- [ ] Kubernetes cơ bản: triển khai workload lên cluster
- [ ] Monitoring (Giám Sát) với Prometheus & Grafana hoặc CloudWatch

### Giai Đoạn 4 — Cloud & DevOps Nâng Cao (Tuần 21+)

- [ ] AWS core services: IAM, VPC, EC2, S3, RDS
- [ ] Terraform — IaC cho môi trường dev/staging/production
- [ ] CI/CD Pipeline với GitHub Actions, Jenkins hoặc CircleCI
- [ ] GitOps — ArgoCD / Flux
- [ ] Security Hardening (Cứng Hóa Bảo Mật): RBAC, Secrets Management, NetworkPolicy

---

## Ma Trận Kỹ Năng Theo Vai Trò

| Kỹ năng | Backend Dev | Frontend Dev | DBA | DevOps |
| ------- | :---------: | :----------: | :-: | :----: |
| Lập trình ứng dụng (.NET / Java / React) | ⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐ |
| SQL & thiết kế schema | ⭐⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐ |
| DBA — backup, HA, tuning | ⭐⭐ | — | ⭐⭐⭐ | ⭐⭐ |
| Docker & Kubernetes | ⭐⭐ | ⭐ | ⭐ | ⭐⭐⭐ |
| AWS / Cloud | ⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐ |
| Terraform / IaC | ⭐ | — | ⭐ | ⭐⭐⭐ |
| CI/CD | ⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐ |
| Monitoring & Observability (Khả Năng Quan Sát Hệ Thống) | ⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐ |

⭐⭐⭐ = Bắt buộc nắm vững · ⭐⭐ = Nên biết · ⭐ = Nền tảng cơ bản

---

## Cách Sử Dụng Kho Tài Liệu

### Tự học theo lộ trình

1. Xác định vai trò mục tiêu (Backend, Frontend, DBA, DevOps).
2. Mở README của module tương ứng để xem lộ trình và checklist.
3. Học tuần tự theo giai đoạn; đánh dấu `[x]` khi hoàn thành từng mục.
4. Thực hành song song trên môi trường lab (Docker, minikube, AWS Free Tier).

### Ôn phỏng vấn

1. Vào thư mục `*-interview-prep/` hoặc `08-phong-van/` trong từng module.
2. Luyện giải thích trade-off (đánh đổi) bằng lời, không chỉ đọc lý thuyết.
3. Chuẩn bị câu chuyện thực tế theo phương pháp STAR (Situation, Task, Action, Result).

### Tra cứu nhanh

- Dùng tìm kiếm trong IDE hoặc `grep` theo từ khóa kỹ thuật (ví dụ: `HPA`, `ACID`, `RBAC`).
- Mỗi chủ đề thường có file Markdown riêng trong thư mục con tương ứng.

---

## Quy Ước Viết Tài Liệu

Tài liệu trong repo tuân theo các quy ước sau:

| Quy ước | Ví dụ |
| ------- | ----- |
| Ngôn ngữ chính | Tiếng Việt (UTF-8) |
| Thuật ngữ kỹ thuật | Giữ nguyên tiếng Anh |
| Giải thích | Trong ngoặc, sau thuật ngữ tiếng Anh |
| Định dạng chuẩn | `HPA — Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang` |
| Viết tắt phổ biến | K8s (Kubernetes), CI/CD, IaC, ORM, DBA, SRE |
| Cấu trúc thư mục | Đánh số thứ tự (`01-`, `02-`, …) theo độ khó tăng dần |

---

## Đóng Góp

Khi bổ sung hoặc chỉnh sửa tài liệu:

1. Giữ nhất quán phong cách và quy ước thuật ngữ ở trên.
2. Đặt file mới đúng module và thư mục chủ đề.
3. Cập nhật README của module nếu thêm chủ đề mới.

---

**Cập nhật lần cuối:** 2026-06-04  
**Phiên bản:** 1.0
