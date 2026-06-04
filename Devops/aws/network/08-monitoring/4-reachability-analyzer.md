# Reachability Analyzer — Bộ Phân Tích Khả Năng Kết Nối

> Reachability Analyzer (Bộ Phân Tích Khả Năng Kết Nối) là công cụ giúp bạn xác định xem hai điểm trong AWS có thể kết nối với nhau không — và nếu không thể, **chính xác thành phần nào đang chặn** — mà không cần gửi traffic thực tế.

---

## 1. Reachability Analyzer Là Gì?

Reachability Analyzer thực hiện **phân tích tĩnh** (static analysis) cấu hình mạng: không cần gửi packet thực tế, chỉ cần đọc cấu hình Security Group, Route Table, NACL, VPC Endpoints... để xác định:

- Có đường đi từ **nguồn** (source) đến **đích** (destination) không?
- Nếu có → đường đi đó đi qua những **hop** (bước nhảy) nào?
- Nếu không có → **thành phần nào** đang block (Security Group? NACL? Route Table? IGW thiếu?)?

### Cái Gì Được Phân Tích?

```
Security Groups (Nhóm Bảo Mật)
Network ACLs (Danh Sách Kiểm Soát Truy Cập Mạng)
Route Tables (Bảng Định Tuyến)
Internet Gateways (Cổng Internet)
NAT Gateways (Cổng NAT)
VPC Peering Connections (Kết Nối Ngang Hàng VPC)
Transit Gateway Routes (Định Tuyến Transit Gateway)
VPC Endpoints (Điểm Cuối VPC)
Load Balancers (Cân Bằng Tải)
VPN Gateways (Cổng VPN)
```

---

## 2. Cách Hoạt Động

### Quy Trình Phân Tích

```
1. Bạn định nghĩa:
   Source (Nguồn):      EC2 instance, Network Interface, VPC, ...
   Destination (Đích):  EC2 instance, RDS, Load Balancer, Network Interface
   Protocol + Port:     TCP port 443, UDP port 53, v.v.

2. Reachability Analyzer:
   → Xây dựng graph (đồ thị) cấu hình mạng
   → Tìm đường đi từ nguồn đến đích trong graph
   → Kiểm tra từng hop có cho phép traffic không

3. Kết quả:
   REACHABLE (Có Thể Kết Nối):
     → Liệt kê toàn bộ path (đường đi) qua các hops
   NOT REACHABLE (Không Thể Kết Nối):
     → Xác định chính xác thành phần nào block
     → Giải thích lý do block
```

---

## 3. Các Loại Nguồn Và Đích Được Hỗ Trợ

| Loại Tài Nguyên                  | Có Thể Là Nguồn | Có Thể Là Đích |
|----------------------------------|-----------------|----------------|
| EC2 Instance                     | ✓               | ✓              |
| Network Interface (ENI)          | ✓               | ✓              |
| VPC Subnet                       | ✓               | ✓              |
| VPC                              | ✓               | ✓              |
| Internet Gateway (IGW)           | ✓               | ✗              |
| VPN Gateway (VGW)                | ✓               | ✗              |
| Transit Gateway                  | ✓               | ✓              |
| Transit Gateway Attachment       | ✓               | ✓              |
| VPC Peering Connection           | ✓               | ✓              |
| Application Load Balancer        | ✗               | ✓              |
| Network Load Balancer            | ✗               | ✓              |
| Gateway Load Balancer Endpoint   | ✓               | ✓              |

---

## 4. Cách Chạy Phân Tích

### Qua AWS Console

```
1. EC2 Console → Network & Security → Network Insights → Reachability Analyzer
2. Create and analyze path
3. Điền thông tin:
   - Source: chọn loại và ID (ví dụ: EC2 Instance → i-0a1b2c3d)
   - Destination: chọn loại và ID (ví dụ: EC2 Instance → i-0x1y2z3w)
   - Protocol: TCP
   - Destination Port: 5432 (PostgreSQL)
4. Create and analyze
5. Đợi ~1 phút, xem kết quả
```

### Qua AWS CLI

```bash
# Bước 1: Tạo Network Insights Path (Đường Phân Tích Mạng)
aws ec2 create-network-insights-path \
  --source i-0a1b2c3d4e5f \
  --destination i-0x1y2z3w4v \
  --protocol tcp \
  --destination-port 5432 \
  --tag-specifications 'ResourceType=network-insights-path,Tags=[{Key=Name,Value=app-to-db-path}]'

# Kết quả: NetworkInsightsPathId = nip-0a1b2c3d4e5f

# Bước 2: Chạy Analysis
aws ec2 start-network-insights-analysis \
  --network-insights-path-id nip-0a1b2c3d4e5f

# Kết quả: NetworkInsightsAnalysisId = nia-0a1b2c3d4e5f

# Bước 3: Lấy kết quả (đợi 1-5 phút)
aws ec2 describe-network-insights-analyses \
  --network-insights-analysis-ids nia-0a1b2c3d4e5f \
  --query 'NetworkInsightsAnalyses[0].{Status:Status,Reachable:NetworkPathFound,ExplanationCode:Explanations[0].ExplanationCode}'
```

### Qua Terraform

```hcl
# Tạo path và chạy analysis
resource "aws_networkinsights_path" "app_to_db" {
  source           = aws_instance.app.id
  destination      = aws_instance.db.id
  protocol         = "tcp"
  destination_port = 5432

  tags = {
    Name = "app-to-db-connectivity-check"
  }
}

resource "aws_networkinsights_analysis" "app_to_db" {
  network_insights_path_id = aws_networkinsights_path.app_to_db.id
  wait_for_completion      = true

  # Output network_path_found = true/false
}

output "reachability_result" {
  value = aws_networkinsights_analysis.app_to_db.network_path_found
}
```

---

## 5. Đọc Kết Quả Phân Tích

### Kết Quả: REACHABLE (Có Thể Kết Nối)

```json
{
  "NetworkPathFound": true,
  "ForwardPathComponents": [
    {
      "SequenceNumber": 1,
      "Component": {
        "ResourceId": "i-0a1b2c3d",
        "ResourceType": "AWS::EC2::Instance",
        "Name": "app-server"
      }
    },
    {
      "SequenceNumber": 2,
      "Component": {
        "ResourceId": "sg-0a1b2c3d",
        "ResourceType": "AWS::EC2::SecurityGroup",
        "Name": "app-server-sg"
      },
      "OutboundHeader": {
        "Protocol": "6",
        "DestinationPortRanges": [{"From": 5432, "To": 5432}]
      }
    },
    {
      "SequenceNumber": 3,
      "Component": {
        "ResourceId": "rtb-0a1b2c3d",
        "ResourceType": "AWS::EC2::RouteTable"
      },
      "RouteTableRoute": {
        "destinationCidr": "10.0.2.0/24",
        "origin": "local"
      }
    },
    {
      "SequenceNumber": 4,
      "Component": {
        "ResourceId": "sg-0x1y2z3w",
        "ResourceType": "AWS::EC2::SecurityGroup",
        "Name": "db-server-sg"
      },
      "InboundHeader": {
        "Protocol": "6",
        "DestinationPortRanges": [{"From": 5432, "To": 5432}]
      }
    },
    {
      "SequenceNumber": 5,
      "Component": {
        "ResourceId": "i-0x1y2z3w",
        "ResourceType": "AWS::EC2::Instance",
        "Name": "db-server"
      }
    }
  ]
}
```

**Đọc kết quả trên:**
```
app-server (i-0a1b2c3d)
  → SG outbound: cho phép TCP → 5432
  → Route Table: có route local đến 10.0.2.0/24
  → SG db inbound: cho phép TCP 5432 từ app-sg
  → db-server (i-0x1y2z3w)
Kết luận: REACHABLE ✓
```

### Kết Quả: NOT REACHABLE (Không Thể Kết Nối)

```json
{
  "NetworkPathFound": false,
  "Explanations": [
    {
      "ExplanationCode": "SECURITY_GROUP_RULE_DOES_NOT_MATCH",
      "Component": {
        "ResourceId": "sg-0x1y2z3w",
        "ResourceType": "AWS::EC2::SecurityGroup",
        "Name": "db-server-sg"
      },
      "SecurityGroup": {
        "GroupId": "sg-0x1y2z3w"
      },
      "SecurityGroupRule": {
        "Direction": "ingress",
        "Protocol": "tcp",
        "PortRange": {"From": 3306, "To": 3306}
      },
      "ClassicLoadBalancerListener": null,
      "Explanation": "The security group sg-0x1y2z3w has a rule that allows port 3306, but not port 5432. Traffic on port 5432 is not permitted."
    }
  ]
}
```

**Đọc kết quả trên:**
```
Lỗi: db-server-sg chỉ có rule cho port 3306 (MySQL),
     nhưng không có rule cho port 5432 (PostgreSQL)
     
Hành động: Thêm inbound rule vào db-server-sg:
  Type: Custom TCP
  Port: 5432
  Source: app-server-sg
```

---

## 6. Mã Lỗi Phổ Biến (Explanation Codes)

| Mã Lỗi                              | Ý Nghĩa                                              | Cách Sửa                                        |
|-------------------------------------|------------------------------------------------------|-------------------------------------------------|
| `SECURITY_GROUP_RULE_DOES_NOT_MATCH`| SG không có rule cho phép port/protocol này          | Thêm inbound/outbound rule vào SG               |
| `NETWORK_ACL_RULE_DOES_NOT_MATCH`   | NACL không có rule cho phép traffic này              | Thêm NACL rule (nhớ thêm cả outbound return)   |
| `ROUTE_DOES_NOT_EXIST`              | Route Table không có route đến đích                  | Thêm route vào Route Table                      |
| `ROUTE_TABLE_CONFIGURATION_MISMATCH`| Route tồn tại nhưng trỏ sai target                  | Sửa lại route target                            |
| `INTERNET_GATEWAY_NOT_ATTACHED`     | IGW chưa được gắn vào VPC                            | Attach IGW vào VPC                              |
| `VPN_CONNECTION_DOES_NOT_EXIST`     | VPN connection không tồn tại hoặc down              | Kiểm tra VPN tunnel status                      |
| `TRANSIT_GATEWAY_ATTACHMENT_NOT_FOUND`| Không có TGW attachment cho VPC này               | Tạo TGW attachment                              |
| `MISSING_TARGET_CONFIGURATION`      | Target không có public IP và không có NAT Gateway    | Thêm NAT Gateway hoặc cấp Elastic IP            |
| `ENDPOINT_SERVICE_UNAVAILABLE`      | VPC Endpoint Service không available                 | Kiểm tra PrivateLink service                    |

---

## 7. Kịch Bản Debug Thực Tế

### Kịch Bản 1: Lambda Không Gọi Được RDS

```
Vấn đề: Lambda function trong private subnet không connect được PostgreSQL RDS

Step 1: Tạo path
  Source: Lambda ENI (tìm trong VPC Console → Network Interfaces)
  Destination: RDS Endpoint (dùng ENI của RDS instance)
  Protocol: TCP, Port: 5432

Step 2: Chạy analysis → NOT REACHABLE

Step 3: Xem Explanation:
  Code: SECURITY_GROUP_RULE_DOES_NOT_MATCH
  Component: rds-sg (Security Group của RDS)
  
Step 4: Sửa:
  Thêm inbound rule vào rds-sg:
    Protocol: TCP
    Port: 5432
    Source: lambda-sg (Security Group của Lambda)
    
Step 5: Chạy lại analysis → REACHABLE ✓
```

### Kịch Bản 2: ECS Task Không Pull Được Docker Image Từ ECR

```
Vấn đề: ECS Fargate task trong private subnet fail với "no space left on device"
         thực ra là không kết nối được ECR

Step 1: Tạo path
  Source: ECS task ENI
  Destination: VPC Endpoint cho ECR (interface endpoint)
  Protocol: TCP, Port: 443

Step 2: Analysis → NOT REACHABLE
  Code: ROUTE_DOES_NOT_EXIST
  
Step 3: Phát hiện: Subnet của ECS task không có route đến VPC Endpoint
  (Endpoint được tạo trong subnet khác, route table khác)

Step 4: Sửa:
  Tạo VPC Endpoint trong đúng subnet của ECS
  Hoặc: Thêm route table association cho ECS subnet
```

### Kịch Bản 3: EC2 Không Ping Được EC2 Khác Qua VPC Peering

```
Vấn đề: EC2 trong VPC-A không reach được EC2 trong VPC-B qua VPC Peering

Step 1: Tạo path
  Source: EC2 trong VPC-A
  Destination: EC2 trong VPC-B
  Protocol: ICMP (không cần port)

Step 2: Analysis → NOT REACHABLE
  Code: ROUTE_DOES_NOT_EXIST
  Component: Route Table của VPC-A subnet
  
Step 3: Phát hiện: Route Table trong VPC-A chưa có route:
  Destination: 10.1.0.0/16 (CIDR của VPC-B)
  Target: pcx-0a1b2c3d (VPC Peering Connection ID)

Step 4: Sửa:
  Thêm route vào Route Table của VPC-A:
    10.1.0.0/16 → pcx-0a1b2c3d (peering connection)
  
  Và thêm route vào Route Table của VPC-B (return traffic):
    10.0.0.0/16 → pcx-0a1b2c3d

Step 5: Chạy lại → REACHABLE ✓
```

---

## 8. Tích Hợp Với CI/CD Pipeline

### GitHub Actions — Tự Động Kiểm Tra Kết Nối Sau Deploy

```yaml
name: Connectivity Verification
on:
  workflow_run:
    workflows: ["Terraform Apply"]
    types: [completed]

jobs:
  verify-connectivity:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Check App-to-DB connectivity
        env:
          AWS_REGION: us-east-1
        run: |
          # Tạo path
          PATH_ID=$(aws ec2 create-network-insights-path \
            --source ${{ secrets.APP_INSTANCE_ID }} \
            --destination ${{ secrets.DB_INSTANCE_ID }} \
            --protocol tcp \
            --destination-port 5432 \
            --query 'NetworkInsightsPath.NetworkInsightsPathId' \
            --output text)
          
          # Chạy analysis
          ANALYSIS_ID=$(aws ec2 start-network-insights-analysis \
            --network-insights-path-id $PATH_ID \
            --query 'NetworkInsightsAnalysis.NetworkInsightsAnalysisId' \
            --output text)
          
          # Đợi hoàn thành
          aws ec2 wait network-insights-analysis-analyzed \
            --network-insights-analysis-ids $ANALYSIS_ID
          
          # Kiểm tra kết quả
          REACHABLE=$(aws ec2 describe-network-insights-analyses \
            --network-insights-analysis-ids $ANALYSIS_ID \
            --query 'NetworkInsightsAnalyses[0].NetworkPathFound' \
            --output text)
          
          if [ "$REACHABLE" != "True" ]; then
            echo "ERROR: App cannot reach DB! Network configuration issue detected."
            exit 1
          fi
          echo "SUCCESS: App can reach DB on port 5432"
          
          # Cleanup
          aws ec2 delete-network-insights-analysis --network-insights-analysis-ids $ANALYSIS_ID
          aws ec2 delete-network-insights-paths --network-insights-path-ids $PATH_ID
```

---

## 9. Chi Phí

```
Reachability Analyzer:
  → $0.10 per analysis run (mỗi lần chạy phân tích)
  → Không phụ thuộc vào kích thước VPC hay số hops
  
Ví dụ:
  Debug session: 10 lần chạy analysis = $1.00
  CI/CD: chạy 5 analyses mỗi deploy × 20 deploys/tháng = $10/tháng
  
So sánh: Rẻ hơn nhiều so với một giờ engineer ngồi debug ($50-150/giờ)
```

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Reachability Analyzer khác gì với việc tự test bằng `telnet` hay `nc`?**

> A: `telnet` và `nc` cần gửi traffic thực tế, cần instance có thể access từ nơi test, và chỉ cho biết "có connect được không" chứ không cho biết **tại sao**. Reachability Analyzer không gửi traffic thực, hoạt động thuần túy trên cấu hình, xác định **chính xác thành phần nào block** (tên Security Group, rule cụ thể, route table nào). Hơn nữa có thể chạy khi instance đang stopped.

**Q: Khi nào bạn dùng Reachability Analyzer trong công việc thực tế?**

> A: Ba tình huống chính: (1) **Debug kết nối** — EC2/Lambda/ECS không connect được service khác, thay vì đoán mò kiểm tra từng SG/Route Table; (2) **Xác nhận sau thay đổi** — sau khi thêm Security Group rule hoặc Route, chạy analysis để chắc chắn không break gì; (3) **Tích hợp CI/CD** — sau mỗi infrastructure deploy, tự động verify các kết nối critical không bị impact.

**Q: Tại sao Reachability Analyzer không tìm thấy đường đi dù tôi đã thêm rule đúng?**

> A: Một số lý do thường gặp: (1) NACL outbound rule bị thiếu — NACL stateless nên phải có cả inbound và outbound; (2) Route Table chưa được associate với đúng subnet; (3) Security Group tham chiếu đến Security Group khác nhưng cái đó chưa được gắn vào instance; (4) VPC Peering chưa accept ở phía bên kia; (5) Instance đang stopped — Reachability Analyzer cần instance running hoặc analyze qua ENI.

---

## Tóm Tắt

| Khía Cạnh        | Chi Tiết                                                                      |
|------------------|-------------------------------------------------------------------------------|
| Chức năng        | Phân tích tĩnh xác định kết nối giữa hai điểm trong AWS                     |
| Đầu vào          | Source + Destination + Protocol + Port                                         |
| Kết quả          | REACHABLE (+ full path) hoặc NOT REACHABLE (+ explanation code cụ thể)       |
| Phân tích được   | SG, NACL, Route Tables, IGW, NAT GW, VPC Peering, TGW, VPC Endpoints         |
| Chi phí          | $0.10/analysis                                                                |
| Thời gian        | 1–5 phút mỗi analysis                                                        |
| Use cases        | Debug kết nối, verify sau deploy, CI/CD integration                          |
| Khác Network AA  | Specific (một cặp A↔B) vs. Broad (toàn kiến trúc theo scope)                |

**Tiếp Theo:** [5-observability-checklist.md](./5-observability-checklist.md) — Checklist giám sát toàn diện cho production
