# Inspector v2 — Quét Lỗ Hổng Bảo Mật Tự Động

> AWS Inspector v2 là dịch vụ quét lỗ hổng bảo mật (vulnerability scanning) liên tục và tự động cho EC2 instances, ECR container images, và Lambda functions. Tích hợp với NVD (National Vulnerability Database — Cơ Sở Dữ Liệu Lỗ Hổng Quốc Gia) và threat intelligence của AWS.

## 📚 Mục Lục

1. [Inspector v1 vs v2](#inspector-v1-vs-v2)
2. [Resource Types — Loại Tài Nguyên Được Quét](#resource-types)
3. [Finding Types — Loại Phát Hiện](#finding-types)
4. [Severity & CVSS Scoring](#severity--cvss-scoring)
5. [Bật Inspector](#bật-inspector)
6. [Tích Hợp CI/CD Pipeline](#tích-hợp-cicd-pipeline)
7. [Multi-Account Setup](#multi-account-setup)
8. [Tối Ưu Chi Phí](#tối-ưu-chi-phí)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Inspector v1 vs v2

| Tiêu Chí | Inspector v1 | Inspector v2 |
|---|---|---|
| **Kích hoạt scan** | Thủ công (schedule hoặc trigger) | Tự động liên tục |
| **EC2 scan** | Yêu cầu SSM Agent | Yêu cầu SSM Agent |
| **ECR scan** | Không hỗ trợ | ✅ Hỗ trợ |
| **Lambda scan** | Không hỗ trợ | ✅ Hỗ trợ |
| **Network reachability** | ✅ Hỗ trợ | ✅ Hỗ trợ |
| **Security Hub tích hợp** | Hạn chế | ✅ Native |
| **Organizations** | Không | ✅ Delegated admin |
| **CVSS v3** | Không | ✅ Hỗ trợ |

> **Lưu ý:** Inspector v1 đã ngừng hỗ trợ từ tháng 12/2024. Tất cả dùng v2.

---

## Resource Types — Loại Tài Nguyên Được Quét

### EC2 Instance Scanning

**Yêu cầu:** SSM Agent (AWS Systems Manager Agent) phải được cài và hoạt động.

```
Khi EC2 Instance khởi động
    │
    ▼
Inspector Agent (qua SSM) → thu thập inventory phần mềm
    │
    ▼
So sánh với NVD + AWS threat intelligence
    │
    ▼
Phát hiện CVE (Common Vulnerabilities and Exposures — Lỗ Hổng Phổ Biến)
    │
    ▼
Tạo finding với CVSS score, package info, remediation path
```

**Những gì được quét:**
- OS packages (rpm, deb, apk)
- Programming language packages (pip, npm, gem, go modules, NuGet, cargo)
- Network reachability (khả năng tiếp cận từ internet)

**Không được quét:**
- Custom application code
- Secrets trong environment variables
- Container workloads (dùng ECR scanning thay thế)

### ECR Container Image Scanning

**Kích hoạt tự động khi:**
- Image được push lên ECR
- Khi Inspector được bật và scan existing images

```bash
# Kiểm tra scan results của một image
aws inspector2 list-findings \
  --filter-criteria '{
    "resourceType": [{"comparison": "EQUALS", "value": "AWS_ECR_CONTAINER_IMAGE"}],
    "ecrImageRepositoryName": [{"comparison": "EQUALS", "value": "my-app"}]
  }'
```

**Scan types cho ECR:**
- **Basic scanning** — OS packages only (sử dụng Clair, miễn phí)
- **Enhanced scanning** (Inspector v2) — OS + programming language packages, tích hợp Security Hub

### Lambda Function Scanning

**Scan hai lớp:**
1. **Lambda Standard Scanning** — scan deployment packages (ZIP) khi deploy
2. **Lambda Code Scanning** — scan application code để tìm lỗ hổng code (thông qua CodeGuru)

```
Lambda Function deployed/updated
    │
    ├── Standard: scan ZIP package → tìm CVE trong dependencies
    └── Code: scan source code → tìm code-level vulnerabilities
```

---

## Finding Types — Loại Phát Hiện

### Package Vulnerability (Lỗ Hổng Package)

```json
{
  "findingType": "PACKAGE_VULNERABILITY",
  "title": "CVE-2021-44228 - Log4Shell in Apache Log4j",
  "severity": "CRITICAL",
  "cvss": {
    "v3": {
      "baseScore": 10.0,
      "vectorString": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H"
    }
  },
  "vulnerablePackages": [
    {
      "name": "log4j-core",
      "version": "2.14.1",
      "fixedInVersion": "2.17.1",
      "packageManager": "JAR"
    }
  ],
  "remediation": {
    "recommendation": {
      "text": "Update log4j-core to version 2.17.1 or higher",
      "url": "https://logging.apache.org/log4j/2.x/security.html"
    }
  }
}
```

### Network Reachability (Khả Năng Tiếp Cận Mạng)

Inspector phân tích cấu hình mạng (Security Groups, NACLs, Route Tables, IGW) để xác định xem EC2 có thể tiếp cận từ internet không.

```
Finding: "EC2 instance port 22 is reachable from internet"
    ├── Reason: Security Group allows 0.0.0.0/0 on port 22
    ├── Risk: SSH brute force, unauthorized access
    └── Remediation: Restrict source IP hoặc dùng SSM Session Manager
```

---

## Severity & CVSS Scoring

**CVSS** (Common Vulnerability Scoring System — Hệ Thống Chấm Điểm Lỗ Hổng Phổ Biến) là tiêu chuẩn quốc tế để đánh giá mức độ nghiêm trọng của lỗ hổng.

### Inspector Severity Mapping

| Inspector Severity | CVSS v3 Score | Ý Nghĩa |
|---|---|---|
| **Critical** | 9.0–10.0 | Lỗ hổng cực kỳ nghiêm trọng, thường có exploit sẵn |
| **High** | 7.0–8.9 | Lỗ hổng nghiêm trọng, khả năng cao bị khai thác |
| **Medium** | 4.0–6.9 | Lỗ hổng trung bình, cần đánh giá context |
| **Low** | 0.1–3.9 | Lỗ hổng thấp, rủi ro hạn chế |
| **Informational** | 0 | Thông tin, không phải lỗ hổng |

### Inspector Risk Score

Inspector tính **Inspector Score** riêng bên cạnh CVSS, tính thêm:
- **Network reachability** — có tiếp cận từ internet không? (+điểm nếu có)
- **Exploit availability** — có exploit POC (Proof of Concept) công khai không?
- **Fix availability** — có bản vá không?

```
Inspector Score = CVSS Base Score 
                 × Network Reachability Modifier
                 × Exploit Availability Modifier
```

---

## Bật Inspector

```bash
# Bật Inspector cho account hiện tại (tất cả resource types)
aws inspector2 enable \
  --resource-types EC2 ECR LAMBDA LAMBDA_CODE

# Xem trạng thái
aws inspector2 batch-get-account-status \
  --account-ids 123456789012

# Liệt kê coverage (tài nguyên đang được quét)
aws inspector2 list-coverage \
  --filter-criteria '{
    "resourceType": [{"comparison": "EQUALS", "value": "AWS_EC2_INSTANCE"}]
  }'
```

### Yêu Cầu SSM Cho EC2

```bash
# Kiểm tra SSM Agent status trên EC2
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=i-0123456789abcdef0"

# IAM role EC2 cần có policy này
aws iam attach-role-policy \
  --role-name EC2InstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Nếu EC2 không có SSM — Inspector sẽ báo "MANAGED_EC2_WITH_UNMANAGED_AGENT"
# → Chỉ có network reachability findings, không có package findings
```

---

## Tích Hợp CI/CD Pipeline — Tích Hợp Quy Trình Phát Triển

### Shift-Left Security (Bảo Mật Từ Đầu Pipeline)

```
Developer pushes code
    │
    ▼
GitHub Actions / CodeBuild
    │
    ├── Build Docker image
    ▼
ECR Push
    │
    ▼
Inspector scans image automatically
    │
    ├── Critical/High findings? → Block deployment (chặn triển khai)
    └── Clean? → Approve deployment
```

### Ví Dụ GitHub Actions

```yaml
# .github/workflows/security-scan.yml
name: Container Security Scan

on:
  push:
    branches: [main]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsECRRole
          aws-region: us-east-1
      
      - name: Build and push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS \
            --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
          docker build -t my-app .
          docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:${{ github.sha }}
          docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:${{ github.sha }}
      
      - name: Wait for Inspector scan
        run: sleep 60
      
      - name: Check Inspector findings
        run: |
          FINDINGS=$(aws inspector2 list-findings \
            --filter-criteria '{
              "ecrImageTags": [{"comparison": "EQUALS", "value": "${{ github.sha }}"}],
              "severity": [
                {"comparison": "EQUALS", "value": "CRITICAL"},
                {"comparison": "EQUALS", "value": "HIGH"}
              ]
            }' \
            --query 'findings | length(@)')
          
          if [ "$FINDINGS" -gt "0" ]; then
            echo "FAILED: $FINDINGS critical/high vulnerabilities found"
            exit 1
          fi
          echo "PASSED: No critical/high vulnerabilities"
```

### Sbom Export — Software Bill of Materials (Danh Sách Thành Phần Phần Mềm)

```bash
# Export SBOM cho một ECR image
aws inspector2 create-sbom-export \
  --resource-filter-criteria '{
    "ecrRepositoryName": [{"comparison": "EQUALS", "value": "my-app"}]
  }' \
  --report-format CYCLONEDX_1_4 \
  --s3-destination '{
    "bucketName": "my-sbom-exports",
    "keyPrefix": "sboms/"
  }'
```

---

## Multi-Account Setup — Thiết Lập Đa Tài Khoản

```bash
# Từ Management Account: ủy quyền
aws inspector2 enable-delegated-admin-account \
  --delegated-admin-account-id 111122223333

# Từ Security Account: bật auto-enable cho organization
aws inspector2 update-organization-configuration \
  --auto-enable '{
    "ec2": true,
    "ecr": true,
    "lambda": true,
    "lambdaCode": true
  }'
```

---

## Tối Ưu Chi Phí

| Resource Type | Cách Tính Giá |
|---|---|
| EC2 | $1.008/instance-month (Linux) |
| ECR | $0.09/container image (initial + re-scan) |
| Lambda Standard | $0.30/function-month |
| Lambda Code | $0.30/function-month |

### Giảm Chi Phí

```bash
# Chỉ bật cho production accounts, không bật dev/sandbox
# Dùng resource exclusions để loại trừ instances không cần scan

aws inspector2 create-filter \
  --name "ExcludeTestInstances" \
  --action SUPPRESS \
  --filter-criteria '{
    "ec2InstanceTags": [
      {
        "comparison": "EQUALS",
        "key": "Environment",
        "value": "test"
      }
    ]
  }'
```

---

## Câu Hỏi Phỏng Vấn

**Q: Inspector khác GuardDuty như thế nào?**
A: Inspector quét **vulnerabilities** (lỗ hổng bảo mật đã biết trong phần mềm — CVEs). GuardDuty phát hiện **threats** (mối đe dọa thực tế đang xảy ra — behavior anomalies, malicious IPs). Inspector chạy trước incident (proactive), GuardDuty chạy trong thời gian thực (reactive). Cần cả hai: Inspector để giảm attack surface, GuardDuty để phát hiện khi bị tấn công.

**Q: EC2 không cài SSM Agent có được quét không?**
A: Chỉ được quét **network reachability** (cấu hình mạng). Package vulnerability scanning yêu cầu SSM Agent. Nên luôn dùng Amazon Linux 2023 hoặc Ubuntu latest — đều có SSM Agent pre-installed.

**Q: Inspector v2 tự động quét ECR như thế nào?**
A: Khi bật ECR scanning trong Inspector v2, mỗi lần image được push lên ECR, Inspector tự động trigger scan. Findings xuất hiện trong Inspector console và Security Hub trong vài phút. Không cần configure gì thêm.

**Q: CVSS score 10.0 có nghĩa là gì?**
A: CVSS 10.0 là mức tối đa — lỗ hổng có thể bị khai thác từ network (không cần truy cập vật lý), không cần authentication, không cần user interaction, và gây impact hoàn toàn lên Confidentiality/Integrity/Availability. Ví dụ điển hình: Log4Shell (CVE-2021-44228). Cần patch ngay lập tức.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
