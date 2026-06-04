# VPC Flow Logs — Nhật Ký Luồng Mạng

> VPC Flow Logs (Nhật Ký Luồng VPC) ghi lại thông tin về lưu lượng IP đi vào và đi ra các giao diện mạng trong VPC của bạn — là công cụ không thể thiếu để debug kết nối, phát hiện bất thường và tuân thủ kiểm toán.

---

## 1. VPC Flow Logs Là Gì?

VPC Flow Logs là tính năng cho phép bạn **ghi lại metadata** (không phải nội dung packet) của các luồng IP (IP flows) đi qua:

- **VPC** — toàn bộ lưu lượng trong VPC
- **Subnet** (Mạng con) — lưu lượng vào/ra một subnet cụ thể
- **ENI** (Elastic Network Interface — Giao Diện Mạng Ảo) — lưu lượng của một giao diện mạng cụ thể

### Quan Trọng: Flow Logs KHÔNG Ghi

```
✗ Nội dung packet (payload) — chỉ ghi metadata
✗ Traffic đến Amazon DNS server (169.254.169.253)
✗ Traffic đến Instance Metadata Service (169.254.169.254)
✗ Traffic DHCP
✗ Traffic đến/từ reserved IP của AWS (first 4 IPs trong subnet)
✗ Traffic giữa Windows activation server
```

---

## 2. Cấu Trúc Một Flow Log Record

### Format Mặc Định (v2)

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

### Ví Dụ Thực Tế

```
2 123456789012 eni-0a1b2c3d 10.0.1.5 10.0.2.100 49152 443 6 10 4000 1620000000 1620000060 ACCEPT OK
2 123456789012 eni-0a1b2c3d 203.0.113.10 10.0.1.5 80 1024 6 5 2000 1620000000 1620000060 REJECT OK
```

### Giải Thích Từng Trường

| Trường          | Giá Trị Ví Dụ       | Ý Nghĩa                                                  |
|-----------------|---------------------|----------------------------------------------------------|
| `version`       | `2`                 | Phiên bản format (v2 là mặc định, v5 có thêm VPC ID)     |
| `account-id`    | `123456789012`      | AWS Account ID của ENI                                   |
| `interface-id`  | `eni-0a1b2c3d`      | ID của ENI ghi lại traffic                               |
| `srcaddr`       | `10.0.1.5`          | Địa chỉ IP nguồn                                         |
| `dstaddr`       | `10.0.2.100`        | Địa chỉ IP đích                                          |
| `srcport`       | `49152`             | Port nguồn                                               |
| `dstport`       | `443`               | Port đích                                                |
| `protocol`      | `6`                 | Giao thức IANA: 6=TCP, 17=UDP, 1=ICMP                   |
| `packets`       | `10`                | Số packet trong flow                                     |
| `bytes`         | `4000`              | Tổng bytes trong flow                                    |
| `start`         | `1620000000`        | Unix timestamp bắt đầu flow                              |
| `end`           | `1620000060`        | Unix timestamp kết thúc flow                             |
| `action`        | `ACCEPT` / `REJECT` | Kết quả: chấp nhận hay bị từ chối bởi SG/NACL          |
| `log-status`    | `OK`                | OK = ghi thành công; NODATA = không có traffic; SKIPDATA = bị bỏ qua do giới hạn |

### Flow Log Version 5 — Các Trường Bổ Sung

```
vpc-id subnet-id instance-id tcp-flags type pkt-srcaddr pkt-dstaddr region az-id sublocation-type sublocation-id pkt-src-aws-service pkt-dst-aws-service flow-direction traffic-path
```

Trường `flow-direction` đặc biệt hữu ích:
- `ingress` — traffic đang đi VÀO ENI
- `egress` — traffic đang đi RA KHỎI ENI

---

## 3. Destination — Nơi Lưu Flow Logs

### Tùy Chọn 1: CloudWatch Logs (Nhật Ký CloudWatch)

```
Ưu điểm:
  ✓ Truy vấn thời gian thực với CloudWatch Logs Insights
  ✓ Tạo Metric Filters để đếm REJECT records
  ✓ Tích hợp sẵn với CloudWatch Alarms
  ✓ Không cần setup thêm pipeline

Nhược điểm:
  ✗ Chi phí lưu trữ cao hơn S3 ($0.50/GB ingested vs $0.023/GB in S3)
  ✗ Không phù hợp để lưu trữ dài hạn (long-term retention)
  ✗ Truy vấn phức tạp chậm hơn Athena
```

### Tùy Chọn 2: Amazon S3

```
Ưu điểm:
  ✓ Chi phí rẻ hơn nhiều cho lưu trữ dài hạn
  ✓ Tích hợp với Amazon Athena để query bằng SQL
  ✓ Tích hợp với Amazon QuickSight để visualization
  ✓ Export sang SIEM (Security Information and Event Management) tool bên ngoài
  ✓ Hỗ trợ Parquet format (tiết kiệm 75% chi phí so với text)

Nhược điểm:
  ✗ Không thể query thời gian thực
  ✗ Cần cấu hình Athena table, partitioning
  ✗ Độ trễ ghi log: 5–15 phút
```

### Tùy Chọn 3: Amazon Data Firehose (Kinesis Data Firehose)

```
Dùng khi cần:
  → Stream log tới OpenSearch (Elasticsearch) để full-text search
  → Tích hợp với Splunk, Datadog, hoặc custom SIEM
  → Real-time dashboard trong Kibana
```

---

## 4. Cách Bật VPC Flow Logs

### Qua AWS Console

```
1. VPC Console → Your VPCs → chọn VPC
2. Actions → Create flow log
3. Filter: All / Accept / Reject
4. Maximum aggregation interval: 1 phút hoặc 10 phút
5. Destination: CloudWatch Logs hoặc S3
6. IAM Role: Tạo mới hoặc chọn role đã có
```

### Qua AWS CLI

```bash
# Bật Flow Logs gửi tới CloudWatch Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-0a1b2c3d4e5f \
  --traffic-type ALL \
  --log-group-name /aws/vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogsRole

# Bật Flow Logs gửi tới S3
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-0a1b2c3d4e5f \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-logs-bucket/vpc-logs/
```

### Qua Terraform (Hạ Tầng Dưới Dạng Mã)

```hcl
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
}

resource "aws_cloudwatch_log_group" "flow_log" {
  name              = "/aws/vpc/flow-logs"
  retention_in_days = 30
}
```

---

## 5. Phân Tích Flow Logs Thực Tế

### CloudWatch Logs Insights — Các Query Hữu Ích

**Query 1: Tìm tất cả kết nối bị REJECT**

```sql
fields @timestamp, srcaddr, dstaddr, dstport, action
| filter action = "REJECT"
| sort @timestamp desc
| limit 100
```

**Query 2: Top 10 IP nguồn gửi nhiều traffic nhất**

```sql
stats sum(bytes) as totalBytes by srcaddr
| sort totalBytes desc
| limit 10
```

**Query 3: Phát hiện port scanning (nhiều port khác nhau từ cùng IP)**

```sql
stats count_distinct(dstport) as portsScanned by srcaddr
| filter portsScanned > 10
| sort portsScanned desc
```

**Query 4: Traffic ra ngoài internet (egress) trên các port nhạy cảm**

```sql
fields srcaddr, dstaddr, dstport, bytes
| filter action = "ACCEPT"
  and dstport in [22, 23, 3389, 1433, 3306]
  and not (dstaddr like /^10\./ or dstaddr like /^172\.1[6-9]\./ or dstaddr like /^192\.168\./)
| sort bytes desc
```

**Query 5: Phân tích traffic theo giờ để phát hiện DDoS**

```sql
stats count() as requestCount by bin(5m)
| filter dstport = 80
| sort @timestamp
```

### Athena Query (cho Flow Logs trong S3)

```sql
-- Tạo bảng Athena trỏ vào S3
CREATE EXTERNAL TABLE vpc_flow_logs (
  version     int,
  account     string,
  interfaceid string,
  sourceaddress string,
  destinationaddress string,
  sourceport  int,
  destinationport int,
  protocol    int,
  numpackets  int,
  numbytes    bigint,
  starttime   int,
  endtime     int,
  action      string,
  logstatus   string
)
PARTITIONED BY (dt string)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ' '
LOCATION 's3://my-flow-logs-bucket/vpc-logs/AWSLogs/123456789012/vpcflowlogs/us-east-1/'
TBLPROPERTIES ("skip.header.line.count"="1");

-- Query: Top talkers (nguồn traffic lớn nhất)
SELECT sourceaddress, SUM(numbytes) as total_bytes
FROM vpc_flow_logs
WHERE dt = '2026/05/14'
  AND action = 'ACCEPT'
GROUP BY sourceaddress
ORDER BY total_bytes DESC
LIMIT 20;
```

---

## 6. Các Pattern Phân Tích Quan Trọng

### Pattern 1: Phân Tích ACCEPT vs REJECT

```
ACCEPT record → traffic đi qua Security Group và NACL thành công
REJECT record → traffic bị chặn bởi Security Group hoặc NACL

Lưu ý: Nếu bạn thấy REJECT từ IP nội bộ (10.x.x.x) → 
        có thể Security Group thiếu inbound rule
```

### Pattern 2: Nhận Diện Security Group vs NACL Blocking

```
Flow Logs ghi record 2 lần (inbound + outbound):

Scenario: EC2-A cố kết nối đến EC2-B port 443

Nếu SG của EC2-B block:
  → Thấy REJECT ở ENI của EC2-B (inbound bị chặn)
  → KHÔNG thấy return traffic

Nếu NACL block:
  → Thấy ACCEPT (SG cho phép) nhưng REJECT (NACL stateless chặn)
  → Hoặc REJECT trên cả inbound lẫn outbound (NACL chặn 2 chiều)
```

### Pattern 3: Tính Toán Bandwidth Thực Tế

```sql
-- Bytes mỗi giây qua một ENI
SELECT 
  interfaceid,
  SUM(numbytes) / (MAX(endtime) - MIN(starttime)) as bytes_per_second,
  SUM(numbytes) / 1048576 as total_MB
FROM vpc_flow_logs
WHERE dt = '2026/05/14'
GROUP BY interfaceid
ORDER BY bytes_per_second DESC;
```

### Pattern 4: Phát Hiện Lateral Movement (Di Chuyển Ngang)

```sql
-- EC2 nào đang quét nhiều IP nội bộ?
SELECT sourceaddress, COUNT(DISTINCT destinationaddress) as unique_destinations
FROM vpc_flow_logs
WHERE action = 'ACCEPT'
  AND destinationaddress LIKE '10.%'
GROUP BY sourceaddress
HAVING unique_destinations > 20
ORDER BY unique_destinations DESC;
```

---

## 7. Chi Phí VPC Flow Logs

```
Thành phần chi phí:
┌──────────────────────────────────────────────────────────┐
│ 1. Ingestion (nhập dữ liệu)                              │
│    CloudWatch Logs: $0.50/GB                             │
│    S3: Miễn phí (chỉ trả phí lưu trữ S3 sau)           │
│                                                          │
│ 2. Storage (lưu trữ)                                     │
│    CloudWatch Logs: $0.03/GB/tháng                       │
│    S3 Standard: $0.023/GB/tháng                          │
│    S3 Glacier: $0.004/GB/tháng (archive sau 90 ngày)    │
│                                                          │
│ 3. Query (truy vấn)                                      │
│    CloudWatch Insights: $0.005/GB scanned               │
│    Athena: $5/TB scanned (dùng Parquet để giảm)         │
└──────────────────────────────────────────────────────────┘

Ví dụ chi phí thực tế:
  VPC có 10 EC2, mỗi ngày tạo 10GB flow logs
  → Tháng: 300GB
  → CloudWatch: $150/tháng (ingestion) + $9/tháng (storage)
  → S3 + Athena: $0 (ingestion) + $6.9/tháng (storage) + query phí
  → Tiết kiệm: dùng S3 + Parquet format + lifecycle policy
```

### Tối Ưu Chi Phí

```bash
# 1. Dùng Parquet format thay vì text (giảm 75% dung lượng)
aws ec2 create-flow-logs \
  --log-format parquet \
  --log-destination-type s3 ...

# 2. Chỉ ghi REJECT thay vì ALL (nếu chỉ cần debug security)
aws ec2 create-flow-logs \
  --traffic-type REJECT ...

# 3. S3 Lifecycle Policy: chuyển sang Glacier sau 90 ngày
# Cấu hình trong S3 bucket lifecycle rules
```

---

## 8. IAM Permissions cho Flow Logs

### Role cho CloudWatch Logs Destination

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

### Trust Policy (Chính Sách Tin Tưởng) cho VPC Flow Logs Service

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## 9. Tích Hợp Với Security Tools

### GuardDuty + Flow Logs

```
Amazon GuardDuty TỰ ĐỘNG phân tích VPC Flow Logs (nếu bật).
Bạn không cần tự parse — GuardDuty phát hiện:
  - Cryptocurrency mining traffic
  - Command & Control (C2) communication
  - Port scanning
  - DNS exfiltration
```

### SIEM Integration (Tích Hợp Hệ Thống Quản Lý Sự Kiện Bảo Mật)

```
Flow Logs → Kinesis Data Firehose → Splunk / Datadog / Elastic SIEM
                                  → OpenSearch (Kibana dashboards)
```

### CloudWatch Metric Filter — Cảnh Báo Tự Động

```bash
# Tạo metric filter đếm số REJECT records
aws logs put-metric-filter \
  --log-group-name /aws/vpc/flow-logs \
  --filter-name RejectCount \
  --filter-pattern "[v, account, interfaceId, srcAddr, dstAddr, srcPort, dstPort, proto, pkts, bytes, start, end, action=REJECT, logStatus]" \
  --metric-transformations metricName=RejectedConnections,metricNamespace=VPCFlowLogs,metricValue=1
```

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: VPC Flow Logs ghi lại gì và không ghi lại gì?**

> A: Ghi metadata của IP flows: IP nguồn/đích, port, protocol, bytes, packets, action (ACCEPT/REJECT). Không ghi nội dung packet, không ghi traffic DNS đến Amazon DNS server, Instance Metadata Service (169.254.169.254), và DHCP.

**Q: Sự khác biệt giữa action=ACCEPT và action=REJECT trong Flow Logs là gì?**

> A: ACCEPT nghĩa là traffic được phép bởi Security Group VÀ NACL. REJECT nghĩa là bị chặn bởi Security Group HOẶC NACL. REJECT thường chỉ ra vấn đề cấu hình cần điều tra.

**Q: Bạn sẽ gửi Flow Logs đến CloudWatch hay S3? Tại sao?**

> A: Phụ thuộc vào use case. CloudWatch phù hợp cho monitoring thời gian thực, tạo alarms, và query ad-hoc. S3 phù hợp cho long-term retention, cost optimization, và batch analysis bằng Athena. Production thường dùng cả hai: CloudWatch cho 7 ngày gần nhất, S3 cho lưu trữ 90+ ngày.

**Q: Làm thế nào để dùng Flow Logs để xác định Security Group đang block traffic?**

> A: Lọc records với `action = REJECT` và lọc theo `dstaddr` của EC2 đang gặp vấn đề. Nếu thấy REJECT với IP nguồn và port đúng như kết nối bị fail → Security Group hoặc NACL đang block. Kết hợp với Reachability Analyzer để xác định chính xác layer nào block.

---

## Tóm Tắt

| Tính Năng        | Chi Tiết                                                            |
|------------------|---------------------------------------------------------------------|
| Ghi lại          | Metadata IP flows: IP, port, protocol, bytes, action               |
| Không ghi lại    | DNS nội bộ, metadata service, DHCP                                  |
| Destination      | CloudWatch Logs, S3, Kinesis Data Firehose                          |
| Aggregation      | 1 phút hoặc 10 phút                                                 |
| Action field     | ACCEPT (được phép) / REJECT (bị chặn bởi SG hoặc NACL)            |
| Công cụ phân tích| CloudWatch Logs Insights, Amazon Athena, Amazon QuickSight          |
| Chi phí          | CloudWatch: $0.50/GB ingested; S3: miễn phí ingestion              |

**Tiếp Theo:** [2-cloudwatch-networking.md](./2-cloudwatch-networking.md) — Metrics & Alarms cho các dịch vụ mạng AWS
