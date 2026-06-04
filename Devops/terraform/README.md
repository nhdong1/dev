# 🏗️ Terraform — Lộ Trình Học Hạ Tầng Dưới Dạng Mã (Infrastructure as Code)

> Hướng dẫn toàn diện về Terraform — từ nền tảng đến vận hành nâng cao trong môi trường thực tế.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Nhà Cung Cấp Cloud](#nhà-cung-cấp-cloud)
4. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng (Tuần 1-2)**

- [ ] Terraform là gì & IaC — Infrastructure as Code — Hạ Tầng Dưới Dạng Mã
- [ ] Cài đặt & cấu hình cơ bản (CLI, providers)
- [ ] Vòng đời tài nguyên: `init` → `plan` → `apply` → `destroy`
- [ ] State — Trạng thái — là gì và tại sao quan trọng
- [ ] HCL — HashiCorp Configuration Language — Ngôn Ngữ Cấu Hình HashiCorp

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3-6)**

- [ ] Variables — Biến & Outputs — Đầu Ra
- [ ] Modules — Mô-đun — tái sử dụng code
- [ ] Remote State — Trạng thái từ xa & State Locking — Khoá trạng thái
- [ ] Workspaces — Không gian làm việc
- [ ] Import tài nguyên hiện có vào Terraform

### **Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7-10)**

- [ ] CI/CD Pipeline — Đường ống tích hợp & triển khai liên tục — tích hợp với Terraform
- [ ] Terraform Cloud / Terraform Enterprise
- [ ] Policy as Code — Chính Sách Dưới Dạng Mã — với Sentinel / OPA
- [ ] Drift Detection — Phát Hiện Lệch Cấu Hình
- [ ] Multi-environment strategy — Chiến lược đa môi trường

### **Giai Đoạn 4: Chuyên Sâu (Tuần 11+)**

- [ ] Advanced Module Patterns — Kiểu mẫu mô-đun nâng cao
- [ ] Provider Development — Phát triển Provider tùy chỉnh
- [ ] Terragrunt — Công cụ bao ngoài Terraform
- [ ] Compliance & Governance — Tuân thủ & Quản trị hạ tầng
- [ ] Cost Estimation — Ước tính chi phí hạ tầng

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                                | Ưu Tiên | Thời Gian | Trạng Thái |
| --------------------------------------- | -------- | --------- | ---------- |
| **State Management — Quản lý trạng thái** | ⭐⭐⭐   | 2 tuần    | -          |
| **Module Design — Thiết kế mô-đun**     | ⭐⭐⭐   | 3 tuần    | -          |
| **Remote Backend — Backend từ xa**      | ⭐⭐⭐   | 1 tuần    | -          |
| **CI/CD Integration — Tích hợp CI/CD**  | ⭐⭐⭐   | 2 tuần    | -          |
| **Security Best Practices — Bảo mật**   | ⭐⭐⭐   | 2 tuần    | -          |
| **Drift Detection — Phát hiện lệch**    | ⭐⭐⭐   | 1 tuần    | -          |
| **Testing — Kiểm thử hạ tầng**          | ⭐⭐⭐   | 2 tuần    | -          |
| **Troubleshooting — Xử lý sự cố**       | ⭐⭐⭐   | 2 tuần    | -          |
| **Cost Management — Quản lý chi phí**   | ⭐⭐     | 1 tuần    | -          |
| **Multi-cloud Strategy — Đa cloud**     | ⭐⭐     | 2 tuần    | -          |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. Nền Tảng** (`01-fundamentals/`)

- IaC — Infrastructure as Code — Hạ Tầng Dưới Dạng Mã và lợi ích
- HCL — HashiCorp Configuration Language — cú pháp và cấu trúc
- Provider — Nhà cung cấp — và Resource — Tài nguyên
- Data Sources — Nguồn dữ liệu — và Dependencies — Phụ thuộc
- Vòng đời tài nguyên (lifecycle)

### 📁 **2. State Management — Quản Lý Trạng Thái** (`02-state-management/`)

- **Local vs Remote State** — Trạng thái cục bộ vs từ xa
- Backend — Nơi lưu trữ trạng thái — (S3, GCS, Azure Blob, Terraform Cloud)
- State Locking — Khoá trạng thái — với DynamoDB / GCS
- `terraform state` commands — Các lệnh thao tác trạng thái
- State Encryption — Mã hoá trạng thái
- Disaster Recovery — Khôi phục thảm họa — cho state file

### 📁 **3. Modules — Mô-đun** (`03-modules/`)

- Module Structure — Cấu trúc mô-đun chuẩn
- Input / Output Variables — Biến đầu vào / đầu ra
- Module Registry — Kho mô-đun — (Terraform Registry, Private)
- Module Versioning — Quản lý phiên bản mô-đun
- Composition Patterns — Kiểu mẫu ghép mô-đun
- Refactoring Modules — Tái cấu trúc mô-đun an toàn

### 📁 **4. Workspaces & Environments — Môi Trường** (`04-workspaces-environments/`)

- Workspaces — Không gian làm việc — và giới hạn của chúng
- Environment Separation — Phân tách môi trường — (dev / staging / prod)
- Terragrunt — Quản lý nhiều môi trường với DRY — Don't Repeat Yourself
- Variable Files — File biến — (`.tfvars`) theo môi trường
- Backend per Environment — Backend riêng cho từng môi trường

### 📁 **5. Security — Bảo Mật** (`05-security/`)

- Secrets Management — Quản lý bí mật — (Vault, AWS Secrets Manager)
- Least Privilege — Nguyên tắc đặc quyền tối thiểu — cho Terraform role
- Sensitive Variables — Biến nhạy cảm — và output masking
- SAST — Static Application Security Testing — cho Terraform (tfsec, Checkov)
- IAM — Identity and Access Management — role cho Terraform CI/CD
- Audit Logging — Nhật ký kiểm tra — cho các thay đổi hạ tầng

### 📁 **6. CI/CD Integration — Tích Hợp CI/CD** (`06-cicd/`)

- Terraform trong GitHub Actions / GitLab CI / Jenkins
- Plan-then-Apply workflow — Quy trình plan trước, apply sau
- PR-based — Pull Request — review cho `terraform plan`
- Atlantis — Tự động hoá Terraform qua Pull Request
- Terraform Cloud Run Tasks — Tác vụ chạy trên Terraform Cloud
- Rollback Strategy — Chiến lược rollback khi có sự cố

### 📁 **7. Testing — Kiểm Thử Hạ Tầng** (`07-testing/`)

- `terraform validate` & `terraform fmt`
- Linting — Kiểm tra cú pháp — với TFLint
- Unit Testing — Kiểm thử đơn vị — với Terratest
- Integration Testing — Kiểm thử tích hợp — thực tế trên cloud
- Compliance Testing — Kiểm thử tuân thủ — với Checkov / OPA — Open Policy Agent
- Contract Testing — Kiểm thử hợp đồng — giữa các mô-đun

### 📁 **8. Monitoring & Observability — Giám Sát** (`08-monitoring/`)

- Drift Detection — Phát hiện lệch cấu hình — với `terraform plan` định kỳ
- Cost Estimation — Ước tính chi phí — với Infracost
- Change Audit — Kiểm tra thay đổi — (ai apply gì, khi nào)
- Alerting — Cảnh báo — khi có thay đổi ngoài dự kiến
- Resource Tagging — Gán nhãn tài nguyên — để giám sát chi phí

### 📁 **9. Troubleshooting — Xử Lý Sự Cố** (`09-troubleshooting/`)

- State Corruption — Lỗi state file — và cách phục hồi
- Dependency Cycle — Vòng phụ thuộc — và cách giải quyết
- Provider Errors — Lỗi provider — và retry logic
- Lock File Conflicts — Xung đột file khoá
- Import & Moved — Import và di chuyển tài nguyên
- Debug Mode — Chế độ gỡ lỗi — (`TF_LOG=DEBUG`)

### 📁 **10. Advanced Topics — Chủ Đề Nâng Cao** (`10-advanced/`)

- Dynamic Blocks — Khối động — và `for_each` / `count`
- Meta-arguments — `depends_on`, `lifecycle`, `provisioner`
- Custom Providers — Provider tùy chỉnh
- Terraform CDK — Cloud Development Kit — viết Terraform bằng Python/TypeScript
- OpenTofu — Nhánh open-source của Terraform
- Multi-region & Multi-account Architecture — Kiến trúc đa vùng & đa tài khoản

### 📁 **11. Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 20 câu hỏi phỏng vấn Terraform
- System Design — Thiết kế hệ thống — với IaC
- Incident Stories — Câu chuyện sự cố — theo phương pháp STAR
- Terraform Challenges — Bài tập thực hành
- Best Practices — Thực hành tốt nhất — cho môi trường production

---

## ☁️ Theo Nhà Cung Cấp Cloud

### **AWS — Amazon Web Services**

```
Điểm mạnh: Provider phong phú nhất, nhiều ví dụ cộng đồng
Phù hợp: VPC, EC2, ECS, EKS, RDS, S3, IAM, Lambda
Module chính: terraform-aws-modules/vpc, eks, rds
```

### **GCP — Google Cloud Platform**

```
Điểm mạnh: Tích hợp tốt với GKE, Cloud SQL, BigQuery
Phù hợp: GKE clusters, Cloud Run, Pub/Sub, Firestore
Module chính: terraform-google-modules/*
```

### **Azure — Microsoft Azure**

```
Điểm mạnh: AzureRM provider đầy đủ, tích hợp Active Directory
Phù hợp: AKS, Azure SQL, App Service, Key Vault
Module chính: Azure/compute, network, aks
```

### **Multi-cloud — Đa Đám Mây**

```
Điểm mạnh: Terraform là lựa chọn hàng đầu cho multi-cloud
Phù hợp: Tổ chức dùng nhiều cloud provider
Lưu ý: Quản lý credentials và backend riêng cho từng cloud
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                        | Thư Mục                                                                         | Ưu Tiên             |
| ----------------------------- | ------------------------------------------------------------------------------- | ------------------- |
| Bắt đầu với Terraform         | [01-fundamentals](./01-fundamentals/)                                           | Bắt đầu ở đây       |
| Câu hỏi phỏng vấn             | [11-interview-prep](./11-interview-prep/)                                       | Trước phỏng vấn     |
| Quản lý State                 | [02-state-management/remote-backend.md](./02-state-management/remote-backend.md) | Thiết yếu           |
| Thiết kế Module               | [03-modules/module-design.md](./03-modules/module-design.md)                   | Quan trọng          |
| Checklist Production          | [09-troubleshooting/production-checklist.md](./09-troubleshooting/production-checklist.md) | Trước khi apply     |

---

## 📊 Ma Trận Kỹ Năng

### Mới Bắt Đầu (0-1 năm)

- [ ] Hiểu IaC — Infrastructure as Code — là gì
- [ ] Viết được resource, variable, output cơ bản
- [ ] Chạy được `init`, `plan`, `apply`, `destroy`
- [ ] Hiểu state file là gì
- [ ] Dùng được module từ Terraform Registry

### Trung Cấp (1-3 năm)

- [ ] Thiết kế và viết module tái sử dụng
- [ ] Quản lý remote state với locking
- [ ] Tích hợp Terraform vào CI/CD pipeline
- [ ] Quản lý nhiều môi trường (dev/staging/prod)
- [ ] Xử lý secrets an toàn
- [ ] Drift detection và remediation — Xử lý lệch cấu hình

### Nâng Cao (3-5+ năm)

- [ ] Kiến trúc multi-account, multi-region
- [ ] Viết và publish module lên private registry
- [ ] Policy as Code — Chính Sách Dưới Dạng Mã — với Sentinel / OPA
- [ ] Custom provider development
- [ ] Incident command cho hạ tầng
- [ ] Compliance và governance frameworks — Khung quản trị tuân thủ

---

## 🚀 Bắt Đầu

### Bước 1: Xác Định Mục Tiêu Học

```
Chọn hướng đi:
- Generalist IaC (nhiều cloud, nhiều công cụ)
- Specialist AWS/GCP/Azure Terraform
- Platform Engineer (Terraform + Kubernetes + GitOps)
```

### Bước 2: Thiết Lập Môi Trường Lab

```bash
# Cài Terraform
brew install terraform          # macOS
# hoặc tải từ https://developer.hashicorp.com/terraform/downloads

# Kiểm tra cài đặt
terraform version

# Cài các công cụ hỗ trợ
brew install tflint             # Linter
brew install infracost          # Cost estimation — Ước tính chi phí
pip install checkov             # Security scanner — Quét bảo mật
```

### Bước 3: Học & Thực Hành

```
1. Đọc một module lý thuyết (30 phút)
2. Viết code Terraform thực tế (30 phút)
3. Tạo và destroy tài nguyên trên cloud (30-60 phút)
4. Review checklist của chủ đề (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation — Tình huống
- Task — Nhiệm vụ
- Action — Hành động đã thực hiện
- Result — Kết quả đạt được
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Nên Đọc

- **"Terraform: Up and Running"** bởi Yevgeniy Brikman — Kinh điển về Terraform
- **"Infrastructure as Code"** bởi Kief Morris — Nguyên lý IaC tổng quát
- **"Cloud Native Infrastructure"** — Hạ tầng cloud-native patterns
- **"The DevOps Handbook"** — Văn hoá và quy trình DevOps

### Tài Liệu Chính Thức

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Registry](https://registry.terraform.io/)
- [AWS Provider Docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Google Provider Docs](https://registry.terraform.io/providers/hashicorp/google/latest/docs)

### Bài Viết & Blog Hay

- HashiCorp Blog
- Gruntwork Blog (tác giả "Terraform Up and Running")
- Anton Babenko's Terraform blog
- AWS Architecture Blog

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Hàng Đầu Theo Nhóm

#### State Management — Quản Lý Trạng Thái

- [ ] State file lưu gì? Tại sao không nên commit lên git?
- [ ] Giải thích State Locking — Khoá trạng thái — và tại sao cần thiết
- [ ] Làm gì khi state file bị corrupt — hỏng?
- [ ] Remote backend là gì? Chọn backend nào cho team?

#### Modules — Mô-đun

- [ ] Thiết kế module như thế nào để tái sử dụng tốt?
- [ ] Phân biệt `count` vs `for_each` — dùng khi nào?
- [ ] Module versioning — Quản lý phiên bản — thực hành thế nào?
- [ ] Khi nào nên dùng Terragrunt?

#### CI/CD & Workflow — Quy Trình

- [ ] Mô tả Terraform workflow trong team
- [ ] Làm sao để nhiều người không apply cùng lúc?
- [ ] Xử lý drift — lệch cấu hình — như thế nào?
- [ ] Blue/Green deployment — Triển khai xanh/xanh lá — với Terraform

#### Security — Bảo Mật

- [ ] Quản lý secrets trong Terraform như thế nào?
- [ ] IAM role nào cần thiết cho Terraform CI/CD?
- [ ] Sensitive output — Output nhạy cảm — xử lý ra sao?

Xem `11-interview-prep/` để có bộ Q&A đầy đủ.

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc nhận vai trò mới, kiểm tra:

- [ ] Có thể giải thích IaC và lợi ích từ đầu
- [ ] Có thể thiết kế cấu trúc module cho dự án thực tế
- [ ] Có thể cấu hình remote backend với locking
- [ ] Có thể tích hợp Terraform vào CI/CD pipeline
- [ ] Có thể xử lý state file bị corrupt
- [ ] Có thể quản lý secrets an toàn trong Terraform
- [ ] Có thể phát hiện và sửa drift — lệch cấu hình
- [ ] Có thể thực hiện migration không downtime với `moved` block
- [ ] Có thể giải thích trade-off giữa `count` và `for_each`
- [ ] Có thể áp dụng Policy as Code — Chính Sách Dưới Dạng Mã

---

## 📞 Công Cụ & Cộng Đồng

### Công Cụ Thiết Yếu

- **terraform** — CLI chính
- **tflint** — Linter cho Terraform
- **tfsec** — Security scanner — Quét lỗ hổng bảo mật
- **Checkov** — Compliance scanner — Kiểm tra tuân thủ
- **Infracost** — Cost estimation — Ước tính chi phí
- **Terragrunt** — Wrapper giúp DRY — Don't Repeat Yourself
- **Atlantis** — Pull Request automation — Tự động hoá qua PR

### IDE & Extensions

- **VS Code** với extension HashiCorp Terraform
- **IntelliJ** với plugin HCL — HashiCorp Configuration Language
- **Terraform Language Server** (ls)

### Cộng Đồng

- r/Terraform (Reddit)
- HashiCorp Discuss Forum
- Terraform Weekly Newsletter
- CNCF Slack — #terraform channel

---

## 📋 Cách Dùng Hướng Dẫn Này

### Cho Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn theo thứ tự
3. Thực hành với lab thực tế
4. Xây dựng dự án portfolio

### Cho Chuẩn Bị Phỏng Vấn

1. Tập trung vào [11-interview-prep](./11-interview-prep/)
2. Nắm vững State Management và Modules
3. Chuẩn bị 2-3 câu chuyện sự cố hạ tầng (STAR)
4. Thực hành giải thích rõ ràng các concepts

### Cho Công Việc Thực Tế

1. Tham khảo [Module Design](./03-modules/) khi thiết kế
2. Dùng [Troubleshooting](./09-troubleshooting/) khi gặp vấn đề
3. Xem [CI/CD Integration](./06-cicd/) để thiết lập pipeline
4. Validate với [Production Checklist](./09-troubleshooting/production-checklist.md)

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này từ đầu đến cuối
├─ 2️⃣  Chọn lộ trình phù hợp (Mới bắt đầu / Trung cấp / Nâng cao)
├─ 3️⃣  Bắt đầu với 01-fundamentals/
├─ 4️⃣  Thiết lập lab với AWS Free Tier hoặc GCP Free Tier
├─ 5️⃣  Hoàn thành bài tập từng chủ đề
├─ 6️⃣  Xây dựng dự án portfolio thực tế
└─ 7️⃣  Chuẩn bị phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
