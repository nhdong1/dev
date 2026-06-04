# Network Access Analyzer — Bộ Phân Tích Truy Cập Mạng

> Network Access Analyzer (Bộ Phân Tích Truy Cập Mạng) là dịch vụ của AWS giúp bạn xác định các đường truy cập mạng **không mong muốn** trong kiến trúc của mình — trả lời câu hỏi "Có ai có thể đi từ internet vào database của tôi không?" mà không cần test thực tế.

---

## 1. Network Access Analyzer Là Gì?

Network Access Analyzer phân tích **toàn bộ cấu hình mạng** (Security Groups, Route Tables, NACLs, VPC Endpoints, IGW, NAT Gateway...) để xác định:

- Có **đường truy cập nào từ internet** vào các tài nguyên cần được cô lập không?
- Có **đường đi ngoài policy** giữa các segment mạng nội bộ không?
- Kiến trúc có **tuân thủ compliance** (ví dụ PCI-DSS, HIPAA) không?

### Sự Khác Biệt Với Reachability Analyzer

```
┌─────────────────────────────────────────────────────────────────┐
│               So Sánh Hai Công Cụ Phân Tích                     │
├──────────────────────────┬──────────────────────────────────────┤
│ Network Access Analyzer  │ Reachability Analyzer               │
├──────────────────────────┼──────────────────────────────────────┤
│ "Có đường nào từ        │ "Đường cụ thể từ EC2-A đến          │
│  internet vào RDS?"      │  EC2-B có thông không?"             │
├──────────────────────────┼──────────────────────────────────────┤
│ Phân tích TOÀN BỘ       │ Phân tích MỘT cặp nguồn-đích        │
│ kiến trúc cùng lúc       │ cụ thể                              │
├──────────────────────────┼──────────────────────────────────────┤
│ Dùng cho: Compliance,    │ Dùng cho: Debug kết nối bị lỗi,     │
│ security audit, policy   │ troubleshoot từng bước              │
├──────────────────────────┼──────────────────────────────────────┤
│ Tìm "đường nào TỒN TẠI" │ Xác nhận "đường này CÓ THÔNG       │
│ mà không nên tồn tại     │ không và vì sao"                    │
├──────────────────────────┼──────────────────────────────────────┤
│ Chi phí: Theo số findings│ Chi phí: $0.10 mỗi analysis         │
└──────────────────────────┴──────────────────────────────────────┘
```

---

## 2. Các Khái Niệm Cốt Lõi

### Network Access Scope (Phạm Vi Truy Cập Mạng)

**Scope** là định nghĩa về loại đường truy cập bạn muốn tìm. Mỗi scope gồm:

```
Scope = Source (Nguồn) + Destination (Đích) + [Conditions (Điều Kiện)]

Ví dụ Scope 1:
  Source:      Internet (IGW, VPN, DX)
  Destination: EC2 instances với tag Environment=Production
  Câu hỏi:    "Có EC2 production nào đang expose ra internet không?"

Ví dụ Scope 2:
  Source:      Bất kỳ tài nguyên nào
  Destination: RDS instances
  Port:        3306 (MySQL)
  Câu hỏi:    "Có ai có thể reach đến MySQL của tôi không?"

Ví dụ Scope 3:
  Source:      Subnet A (Development environment)
  Destination: Subnet B (Production database)
  Câu hỏi:    "Dev environment có thể truy cập Production DB không?"
```

### Findings (Phát Hiện)

Khi chạy phân tích, Network Access Analyzer trả về **findings**:
- **Finding = một đường truy cập tồn tại** trong phạm vi scope đã định nghĩa
- Mỗi finding mô tả: nguồn → đích → qua các hop (Security Group, Route Table, IGW...)
- Bạn có thể **mark finding là "expected"** (mong đợi) để lọc ra khỏi báo cáo

---

## 3. Các Use Case Chính

### Use Case 1: Kiểm Tra Compliance PCI-DSS (Tiêu Chuẩn Bảo Mật Thẻ Thanh Toán)

```
Yêu cầu PCI-DSS: Hệ thống xử lý thẻ tín dụng phải được cô lập
                 khỏi internet và các hệ thống không liên quan

Scope:
  Source:      Internet Gateway
  Destination: EC2/RDS tagged "PCI=true"

Expected finding: Chỉ ALB public-facing có thể vào → đã mark expected
Unexpected finding: Direct access từ internet đến EC2 processing → PHẢI FIX
```

### Use Case 2: Network Segmentation Audit (Kiểm Tra Phân Đoạn Mạng)

```
Requirement: Dev/Test environment không được phép kết nối Production DB

Scope:
  Source:      VPC dev-vpc / subnet tagged Env=Dev
  Destination: RDS in subnet tagged Env=Production

Expected:    Không có finding nào (no path = compliant)
Nếu có finding: Có Security Group rule nào đó đang cho phép → fix ngay
```

### Use Case 3: Phát Hiện Shadow IT (CNTT Bóng Tối)

```
Shadow IT: Nhân viên tự tạo EC2 với Security Group mở rộng rãi
           mà đội bảo mật không biết

Scope:
  Source:      Internet
  Destination: Tất cả EC2 instances trong account
  Port:        22 (SSH), 3389 (RDP)

Finding: Bất kỳ EC2 nào cho phép SSH/RDP từ 0.0.0.0/0 → audit
```

### Use Case 4: Third-Party Vendor Access Validation (Xác Thực Truy Cập Nhà Cung Cấp)

```
Situation: Vendor A được phép truy cập Service-X, nhưng KHÔNG được
           truy cập bất kỳ service nào khác trong VPC

Scope:
  Source:      Vendor A's IP ranges (Customer Gateway / PrivateLink)
  Destination: Tất cả tài nguyên TRỪ Service-X

Expected:    Không có finding (vendor chỉ reach được Service-X)
Finding:     Vendor có thể reach Service-Y → vi phạm → fix
```

---

## 4. Cách Tạo Và Chạy Network Access Analyzer

### Bước 1: Tạo Network Insights Access Scope

```bash
# Tạo scope: internet → ec2 port 22
aws ec2 create-network-insights-access-scope \
  --match-paths '[
    {
      "Source": {
        "ResourceStatement": {
          "ResourceTypes": ["AWS::EC2::InternetGateway"]
        }
      },
      "Destination": {
        "ResourceStatement": {
          "ResourceTypes": ["AWS::EC2::Instance"]
        }
      },
      "ThroughResources": [
        {
          "ResourceStatement": {
            "ResourceTypes": ["AWS::EC2::SecurityGroup"],
            "ResourceCondition": {
              "Field": "ToPort",
              "Value": "22"
            }
          }
        }
      ]
    }
  ]'
```

### Bước 2: Chạy Phân Tích

```bash
# Chạy analysis cho scope vừa tạo
aws ec2 start-network-insights-access-scope-analysis \
  --network-insights-access-scope-id nis-scope-0a1b2c3d4e5f
```

### Bước 3: Lấy Kết Quả

```bash
# Lấy analysis result (đợi 1-5 phút)
aws ec2 get-network-insights-access-scope-analysis-findings \
  --network-insights-access-scope-analysis-id nia-analysis-0a1b2c3d
```

### Bước 4: Đánh Dấu Findings Là "Expected"

```bash
# Mark finding là expected (đường này được phép theo thiết kế)
aws ec2 create-tags \
  --resources finding-id-here \
  --tags Key=Status,Value=Expected
```

---

## 5. Tích Hợp Với AWS Config (Cấu Hình Tuân Thủ)

### Config Rule + Network Access Analyzer

AWS Config Rules (Quy Tắc Cấu Hình AWS) có thể tự động trigger phân tích:

```json
{
  "ConfigRuleName": "vpc-no-unrestricted-ssh",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "RESTRICTED_INCOMING_TRAFFIC"
  },
  "InputParameters": "{\"blockedPort1\": \"22\"}",
  "Scope": {
    "ComplianceResourceTypes": ["AWS::EC2::SecurityGroup"]
  }
}
```

### Automated Compliance Pipeline (Pipeline Tuân Thủ Tự Động)

```
Thay đổi Security Group
       ↓
AWS Config phát hiện change event
       ↓
Lambda function kích hoạt
       ↓
Network Access Analyzer chạy analysis mới
       ↓
Nếu có finding mới (unexpected) → SNS alert → Slack/PagerDuty
       ↓
Security team review và remediate (khắc phục)
```

---

## 6. Phân Tích Kết Quả — Đọc Findings

### Cấu Trúc Một Finding

```json
{
  "FindingId": "finding-0a1b2c3d4e5f",
  "FindingComponents": [
    {
      "SequenceNumber": 1,
      "Component": {
        "ResourceId": "igw-0a1b2c3d",
        "ResourceType": "AWS::EC2::InternetGateway"
      }
    },
    {
      "SequenceNumber": 2,
      "Component": {
        "ResourceId": "rtb-0a1b2c3d",
        "ResourceType": "AWS::EC2::RouteTable",
        "RouteTableRoute": {
          "destinationCidr": "0.0.0.0/0",
          "gatewayId": "igw-0a1b2c3d"
        }
      }
    },
    {
      "SequenceNumber": 3,
      "Component": {
        "ResourceId": "sg-0a1b2c3d",
        "ResourceType": "AWS::EC2::SecurityGroup",
        "SecurityGroupRule": {
          "direction": "ingress",
          "protocol": "tcp",
          "fromPort": 22,
          "toPort": 22,
          "cidr": "0.0.0.0/0"
        }
      }
    },
    {
      "SequenceNumber": 4,
      "Component": {
        "ResourceId": "i-0a1b2c3d4e5f",
        "ResourceType": "AWS::EC2::Instance"
      }
    }
  ]
}
```

### Đọc Finding Trên: Đường Đi Từ Internet → EC2

```
Internet Gateway (igw-0a1b2c3d)
    ↓ [route 0.0.0.0/0 → igw]
Route Table (rtb-0a1b2c3d)
    ↓
Security Group (sg-0a1b2c3d) — allows TCP 22 from 0.0.0.0/0
    ↓
EC2 Instance (i-0a1b2c3d4e5f) ← EXPOSED!

Hành động cần làm:
  → Xóa rule "allow SSH from 0.0.0.0/0" trong Security Group
  → Thay bằng "allow SSH from bastion-sg only" hoặc dùng SSM Session Manager
```

---

## 7. Best Practices (Thực Hành Tốt Nhất)

### Thiết Kế Scope Hiệu Quả

```
✓ Dùng tags để nhóm tài nguyên theo environment, tier, hoặc sensitivity
  Ví dụ: Environment=Production, DataClassification=Confidential

✓ Tạo scope riêng cho từng loại risk:
  - Internet → Production resources (critical)
  - Dev → Production database (important)
  - Untagged resources → Sensitive subnets (medium)

✓ Chạy analysis định kỳ (ít nhất hàng tuần)
  → Tích hợp vào CI/CD pipeline khi có infrastructure change

✓ Mark tất cả expected findings để dễ phát hiện unexpected findings mới
```

### Tích Hợp Vào CI/CD

```yaml
# GitHub Actions workflow — chạy sau khi deploy infrastructure
name: Network Security Check
on:
  push:
    paths:
      - 'terraform/**'

jobs:
  network-access-check:
    runs-on: ubuntu-latest
    steps:
      - name: Run Network Access Analysis
        run: |
          ANALYSIS_ID=$(aws ec2 start-network-insights-access-scope-analysis \
            --network-insights-access-scope-id ${{ secrets.SCOPE_ID }} \
            --query 'NetworkInsightsAccessScopeAnalysis.NetworkInsightsAccessScopeAnalysisId' \
            --output text)
          
          # Đợi analysis hoàn thành
          aws ec2 wait network-insights-access-scope-analysis-analyzed \
            --network-insights-access-scope-analysis-ids $ANALYSIS_ID
          
          # Kiểm tra số findings mới (không phải expected)
          UNEXPECTED=$(aws ec2 get-network-insights-access-scope-analysis-findings \
            --network-insights-access-scope-analysis-id $ANALYSIS_ID \
            --query 'length(AnalysisFindings)')
          
          if [ "$UNEXPECTED" -gt "0" ]; then
            echo "SECURITY ISSUE: $UNEXPECTED unexpected network paths found!"
            exit 1
          fi
```

---

## 8. Giới Hạn Và Chi Phí

### Giới Hạn Dịch Vụ

```
- Tối đa 1,000 Network Insights Access Scopes mỗi region
- Tối đa 1,000 analyses đang pending/running cùng lúc
- Thời gian chạy analysis: 1–15 phút tùy kích thước VPC
- Scope không áp dụng cross-region (phải tạo scope riêng mỗi region)
```

### Chi Phí (Pricing)

```
Network Access Analyzer:
  → $0.002 per resource analyzed per analysis run
  → Ví dụ: VPC có 100 EC2 + 50 SG + 20 route tables = 170 resources
  → Mỗi lần chạy: 170 × $0.002 = $0.34
  → Chạy mỗi ngày: $0.34 × 30 = $10.20/tháng

So sánh với chi phí security breach (sự cố bảo mật):
  → Rất rẻ để có peace of mind (yên tâm)
```

---

## 9. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Network Access Analyzer khác gì với Reachability Analyzer?**

> A: Network Access Analyzer tìm kiếm các đường truy cập trên **toàn bộ kiến trúc** dựa trên scope (phạm vi) định nghĩa — giống như hỏi "có ai có thể vào database của tôi không?" Reachability Analyzer kiểm tra **một cặp nguồn-đích cụ thể** — giống như hỏi "EC2-A có kết nối được EC2-B không và tại sao?" Dùng Network Access Analyzer cho security audit/compliance, dùng Reachability Analyzer để debug kết nối cụ thể.

**Q: Làm thế nào để tích hợp Network Access Analyzer vào quy trình DevSecOps?**

> A: Tích hợp vào CI/CD pipeline: sau mỗi thay đổi Terraform/CloudFormation, trigger analysis tự động. Nếu phát hiện unexpected finding → fail pipeline và alert security team. Kết hợp với AWS Config để monitor real-time khi Security Group thay đổi. Định kỳ review và update expected findings khi có thay đổi kiến trúc có chủ ý.

**Q: Khi nào nên dùng Network Access Analyzer thay vì tự viết script kiểm tra Security Group rules?**

> A: Network Access Analyzer hiểu **toàn bộ graph** của network (routes + SGs + NACLs + VPC Endpoints...), trong khi script tự viết thường chỉ kiểm tra SG rules riêng lẻ và bỏ sót tương tác phức tạp. Ví dụ: EC2 trong private subnet không có route ra internet dù SG mở — script thấy SG "open" nhưng thực ra không reach được. Network Access Analyzer tính toán đúng context.

---

## Tóm Tắt

| Khía Cạnh        | Chi Tiết                                                                  |
|------------------|---------------------------------------------------------------------------|
| Chức năng        | Tìm đường truy cập không mong muốn trên toàn bộ kiến trúc mạng          |
| Cách dùng        | Định nghĩa Scope (nguồn + đích) → Chạy Analysis → Review Findings        |
| Use cases        | Compliance audit, network segmentation, shadow IT detection               |
| Tích hợp         | AWS Config, CI/CD pipeline, Lambda-based remediation                      |
| Khác với RA      | Phân tích toàn bộ (broad) vs. phân tích từng cặp cụ thể (specific)       |
| Chi phí          | $0.002/resource/analysis — rất rẻ so với chi phí của security breach     |

**Tiếp Theo:** [4-reachability-analyzer.md](./4-reachability-analyzer.md) — Phân tích kết nối từng bước giữa hai điểm
