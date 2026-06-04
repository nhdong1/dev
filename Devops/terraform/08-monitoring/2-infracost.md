# Infracost — Ước Tính Chi Phí Hạ Tầng Trong CI/CD

> Infracost là công cụ open-source giúp ước tính chi phí cloud (AWS, GCP, Azure) từ Terraform code **trước khi apply**, tích hợp trực tiếp vào CI/CD pipeline để hiển thị cost diff — sự thay đổi chi phí — ngay trong Pull Request.

---

## 💡 Tại Sao Cần Ước Tính Chi Phí Sớm?

### Vấn Đề Không Có Infracost

```
Timeline điển hình:
  Tuần 1: Dev thay đổi instance_type từ t3.small → m5.4xlarge
  Tuần 2: terraform apply vào production
  Tuần 3: CFO nhận hóa đơn AWS tăng $12,000/tháng
  Tuần 4: Họp giải trình, truy tìm nguyên nhân
  Tuần 5: Downsize lại instance → outage ngắn

Thiệt hại: $12,000 + 4 tuần xử lý + 1 outage
```

### Với Infracost

```
Timeline mới:
  Lúc tạo PR: Infracost comment ngay: "+$11,800/tháng (+960%)"
  Cùng ngày:  Lead review và reject hoặc discuss
  Kết quả:    Chọn instance phù hợp trước khi merge

Thiệt hại: $0 + 30 phút thảo luận
```

---

## 🛠️ Cài Đặt Infracost

### Cài CLI

```bash
# macOS
brew install infracost

# Linux
curl -fsSL https://raw.githubusercontent.com/infracost/infracost/master/scripts/install.sh | sh

# Windows (PowerShell)
. { iwr -useb https://raw.githubusercontent.com/infracost/infracost/master/scripts/install.ps1 } | iex

# Kiểm tra cài đặt
infracost --version
# infracost v0.10.x
```

### Lấy API Key (Miễn Phí)

```bash
# Đăng ký và lấy API key
infracost auth login
# → Mở browser, đăng ký tại infracost.io
# → API key được lưu vào ~/.config/infracost/credentials.yml

# Hoặc set trực tiếp
export INFRACOST_API_KEY="ico-XXXXXXXXXXXXX"
```

---

## 🚀 Sử Dụng Cơ Bản

### Xem Chi Phí Hiện Tại

```bash
cd my-terraform-project/

# Xem bảng chi phí ước tính
infracost breakdown --path .

# Output mẫu:
# Name                              Quantity  Unit        Monthly Cost
# 
# aws_instance.web_server
# ├─ Instance usage (Linux/UNIX, on-demand, t3.medium)   730  hours   $30.37
# ├─ root_block_device
# │  └─ Storage (general purpose SSD, gp2)                20  GB       $2.00
#
# aws_db_instance.postgres
# ├─ Database instance (on-demand, db.t3.small)          730  hours   $29.20
# └─ Storage (general purpose SSD, gp2)                  20  GB       $2.30
#
# PROJECT TOTAL                                                        $63.87
```

### So Sánh Chi Phí Trước Và Sau Khi Thay Đổi

```bash
# Lưu baseline chi phí hiện tại
infracost breakdown --path . --format json > baseline.json

# Sau khi thay đổi code Terraform
# (ví dụ: đổi instance_type từ t3.medium → m5.xlarge)

# So sánh
infracost diff --path . --compare-to baseline.json

# Output mẫu:
# ─────────────────────────────
# Key: ~ changed, + added, - removed
# 
# ~ aws_instance.web_server
#   ~ Instance usage (Linux/UNIX, on-demand, t3.medium → m5.xlarge)
#     Monthly cost change: $30.37 → $184.63 (+$154.26)
# 
# Monthly cost change: $30.37 → $184.63 (+$154.26, +508%)
# ─────────────────────────────
```

---

## 🤖 Tích Hợp Vào CI/CD

### GitHub Actions

```yaml
# .github/workflows/infracost.yml
name: Infracost — Ước Tính Chi Phí

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'

jobs:
  infracost:
    name: Estimate Cost Impact
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write  # Cần để comment vào PR

    steps:
      - name: Checkout base branch (main)
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.base.ref }}

      - name: Setup Infracost
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Generate Infracost JSON (baseline — chi phí hiện tại)
        run: |
          infracost breakdown --path=./infrastructure \
            --format=json \
            --out-file=/tmp/infracost-base.json

      - name: Checkout PR branch (code mới)
        uses: actions/checkout@v4

      - name: Generate Infracost JSON (PR — chi phí sau thay đổi)
        run: |
          infracost breakdown --path=./infrastructure \
            --format=json \
            --out-file=/tmp/infracost-pr.json

      - name: Post Infracost diff comment to PR
        run: |
          infracost comment github \
            --path=/tmp/infracost-pr.json \
            --repo=$GITHUB_REPOSITORY \
            --github-token=${{ github.token }} \
            --pull-request=${{ github.event.pull_request.number }} \
            --behavior=update \
            --base-path=/tmp/infracost-base.json
```

**Output comment trong PR sẽ trông như thế này:**

```
💰 Infracost Cost Estimate

| Resource | Monthly Qty | Unit | Base Cost | New Cost | Diff |
|----------|-------------|------|-----------|----------|------|
| aws_instance.web | 730 | hours | $30.37 | $184.63 | +$154.26 |
| aws_db_instance.postgres | 730 | hours | $29.20 | $29.20 | $0 |

**Monthly cost: $59.57 → $213.83 (+$154.26, +259%)**

View full breakdown: https://dashboard.infracost.io/...
```

### GitLab CI

```yaml
# .gitlab-ci.yml
infracost:
  stage: cost-estimation
  image: infracost/infracost:ci-0.10
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  
  variables:
    INFRACOST_API_KEY: $INFRACOST_API_KEY
  
  script:
    # Lấy chi phí của base branch
    - git fetch origin $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
    - git checkout $CI_MERGE_REQUEST_TARGET_BRANCH_NAME
    - infracost breakdown --path=. --format=json --out-file=/tmp/infracost-base.json
    
    # Trở về MR branch
    - git checkout $CI_COMMIT_REF_NAME
    
    # So sánh và comment vào MR
    - |
      infracost comment gitlab \
        --path=/tmp/infracost-base.json \
        --repo=$CI_PROJECT_PATH \
        --merge-request=$CI_MERGE_REQUEST_IID \
        --gitlab-token=$GITLAB_TOKEN \
        --behavior=update
```

---

## ⚙️ Cấu Hình Nâng Cao

### File `infracost.yml` — Cấu Hình Project

```yaml
# infracost.yml
version: 0.1

projects:
  - path: ./infrastructure/prod
    name: Production Environment
    terraform_var_files:
      - ../environments/prod.tfvars
    
  - path: ./infrastructure/staging
    name: Staging Environment  
    terraform_var_files:
      - ../environments/staging.tfvars

# Loại bỏ resources không cần estimate
# (ví dụ: tags, null_resource)
```

### Budget Threshold — Ngưỡng Ngân Sách

Bạn có thể fail CI/CD nếu chi phí tăng quá ngưỡng cho phép:

```bash
# Lấy diff dưới dạng JSON
infracost diff \
  --path . \
  --compare-to baseline.json \
  --format json > diff.json

# Kiểm tra nếu tăng quá $500/tháng → fail
python3 - <<'EOF'
import json, sys

with open('diff.json') as f:
    data = json.load(f)

diff = float(data['diffTotalMonthlyCost'])
threshold = 500  # USD per month

if diff > threshold:
    print(f"❌ Cost increase ${diff:.2f}/month exceeds threshold ${threshold}/month")
    sys.exit(1)
else:
    print(f"✅ Cost increase ${diff:.2f}/month is within threshold")
EOF
```

### Custom Pricing — Giá Tùy Chỉnh (Enterprise Discounts)

```yaml
# infracost.yml
version: 0.1

# Áp dụng discount 30% (ví dụ: nếu có Reserved Instances)
projects:
  - path: .

# Hoặc dùng file usage để estimate dựa trên actual usage
# thay vì giả định 730 giờ/tháng
```

### Usage File — File Sử Dụng Thực Tế

```yaml
# infracost-usage.yml
# Định nghĩa actual usage để estimate chính xác hơn

version: 0.1
resource_usage:
  aws_lambda_function.api:
    monthly_requests: 10000000      # 10 triệu requests/tháng
    request_duration_ms: 100        # 100ms trung bình
    
  aws_dynamodb_table.sessions:
    monthly_write_request_units: 5000000
    monthly_read_request_units: 50000000
    
  aws_s3_bucket.assets:
    storage_gb: 500                 # 500 GB stored
    monthly_get_requests: 1000000   # 1 triệu GET requests
```

```bash
# Dùng với usage file
infracost breakdown \
  --path . \
  --usage-file infracost-usage.yml
```

---

## 📊 Hiểu Output Của Infracost

### Breakdown Output

```
Name                                    Quantity  Unit    Monthly Cost
 
 aws_instance.app_server
 ├─ Instance usage (on-demand, t3.large)     730  hours         $60.74
 ├─ root_block_device
 │  └─ Storage (gp3)                          30  GB             $2.40
 └─ ebs_block_device[0]
    └─ Storage (gp3)                         100  GB             $8.00
 
 aws_rds_cluster.aurora
 ├─ Aurora serverless v2 (0.5-4 ACU)        21.9  ACU-hours      $9.86
 └─ Storage (aurora)                          20  GB             $2.30
 
 aws_cloudfront_distribution.cdn
 └─ Data transfer out (first 10TB)           100  GB             $8.50
 
 OVERALL TOTAL                                                   $91.80
```

### Giải Thích Các Cột

| Cột | Ý Nghĩa |
|-----|---------|
| Quantity | Số lượng đơn vị (730 giờ = 1 tháng full) |
| Unit | Đơn vị tính (hours, GB, requests) |
| Monthly Cost | Chi phí ước tính mỗi tháng |

### Các Ký Hiệu Trong Diff

```
~ aws_instance.web      → Thay đổi resource đang có
+ aws_instance.worker   → Thêm mới resource
- aws_instance.old      → Xóa resource

Màu sắc:
  Xanh (+$): Chi phí tăng
  Đỏ (-$):  Chi phí giảm (hiếm gặp)
  Vàng (~): Thay đổi không ảnh hưởng chi phí
```

---

## 💼 Infracost Trong Quy Trình Team

### Quy Trình Đề Xuất

```
1. Dev tạo PR với Terraform changes
        │
        ▼
2. CI chạy Infracost
   → Comment cost diff vào PR
        │
        ▼
3. Lead engineer review PR
   → Thấy "+$150/tháng" → Hỏi: "Tại sao cần instance lớn hơn?"
        │
        ├── Lý do hợp lý → Approve
        └── Không cần thiết → Request changes
        │
        ▼
4. Sau khi merge, chi phí thực tế được tracking
   trên dashboard (so sánh estimate vs actual)
```

### Budget Alerts Theo Team

```yaml
# Cấu hình budget alerts theo team/environment
# Tích hợp với AWS Budgets hoặc Infracost Cloud

environments:
  production:
    monthly_budget: 5000   # USD
    alert_threshold: 90    # Alert khi đạt 90% budget
    
  staging:
    monthly_budget: 500
    alert_threshold: 80
    
  development:
    monthly_budget: 200
    alert_threshold: 80
```

---

## 🚫 Giới Hạn Của Infracost

| Giới Hạn | Mô Tả | Workaround |
|----------|--------|------------|
| Không phải mọi resource | Một số resource hiếm không được hỗ trợ | Xem docs danh sách supported resources |
| Giá list price | Không biết discount của bạn | Dùng custom pricing hoặc Infracost Cloud |
| Usage-based services | Lambda, DynamoDB... phụ thuộc actual usage | Dùng usage file để ước tính |
| Multi-currency | Mặc định USD | Cấu hình currency trong infracost.yml |
| Spot instances | Giá spot thay đổi liên tục | Infracost dùng giá on-demand để estimate |

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Bạn quản lý chi phí cloud như thế nào trong Terraform?**

> Tôi dùng Infracost tích hợp vào CI/CD để hiển thị cost diff ngay trong PR trước khi merge. Ngoài ra, có budget thresholds — ngưỡng ngân sách — để fail PR nếu chi phí tăng quá mức cho phép. Kết hợp với resource tagging và AWS Cost Explorer để phân tích chi phí theo team/environment sau khi deploy.

**Q: Infracost có chính xác không?**

> Infracost dùng giá list price từ API của cloud providers — khá chính xác cho on-demand resources. Độ chính xác giảm với: Reserved Instances/Savings Plans (vì không biết discount), usage-based services (Lambda, DynamoDB), và Spot Instances. Dùng usage files và custom pricing để cải thiện độ chính xác.

---

## 🔗 Tài Liệu Tham Khảo

- [Infracost Documentation](https://www.infracost.io/docs/)
- [Infracost GitHub Actions](https://github.com/infracost/actions)
- [Supported Resources List](https://www.infracost.io/docs/supported_resources/)

---

**Tiếp Theo:** [3-change-audit.md](./3-change-audit.md) — Kiểm tra ai thay đổi gì, khi nào

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
