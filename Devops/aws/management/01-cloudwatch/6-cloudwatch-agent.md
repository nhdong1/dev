# CloudWatch Agent — Thu Thập Custom Metrics & Logs Từ EC2

> **CloudWatch Agent** (Tác Nhân CloudWatch) là phần mềm cài trên EC2 (hoặc on-premises server) để thu thập metrics hệ điều hành (RAM, disk, process...) và logs từ file tùy ý, gửi lên CloudWatch — vượt qua giới hạn của hypervisor-level metrics mà AWS cung cấp sẵn.

---

## 📚 Mục Lục

1. [Tại Sao Cần CloudWatch Agent?](#tại-sao-cần-cloudwatch-agent)
2. [Cài Đặt CloudWatch Agent](#cài-đặt-cloudwatch-agent)
3. [Cấu Hình Agent — File JSON](#cấu-hình-agent--file-json)
4. [Metrics Thu Thập Từ OS](#metrics-thu-thập-từ-os)
5. [Log Collection — Thu Thập Nhật Ký](#log-collection--thu-thập-nhật-ký)
6. [High-Resolution Metrics](#high-resolution-metrics)
7. [StatsD & collectd Integration](#statsd--collectd-integration)
8. [On-Premises & Hybrid Cloud](#on-premises--hybrid-cloud)
9. [Quản Lý Cấu Hình Qua SSM Parameter Store](#quản-lý-cấu-hình-qua-ssm-parameter-store)
10. [Troubleshooting Agent](#troubleshooting-agent)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần CloudWatch Agent?

AWS chỉ có thể monitor EC2 từ **hypervisor level** (lớp ảo hóa bên dưới) — nó biết CPU, network, disk I/O của instance nhưng không thể nhìn vào bên trong OS.

```
Giới Hạn EC2 Default Metrics:
  ✅ CPUUtilization   — Hypervisor thấy được
  ✅ NetworkIn/Out    — Hypervisor thấy được
  ✅ DiskReadOps/WriteOps — Hypervisor thấy được (EBS)
  ✅ StatusCheckFailed — Hypervisor thấy được

  ❌ MemoryUtilization — Chỉ OS biết
  ❌ DiskSpaceUsed     — Chỉ OS biết (filesystem level)
  ❌ SwapUsage         — Chỉ OS biết
  ❌ ProcessCount      — Chỉ OS biết
  ❌ Application logs  — Trong file trên đĩa
```

**CloudWatch Agent** chạy **bên trong OS**, thu thập những thứ hypervisor không nhìn thấy.

---

## Cài Đặt CloudWatch Agent

### Yêu Cầu Tiên Quyết

**IAM Role cho EC2 Instance:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "ec2:DescribeVolumes",
        "ec2:DescribeTags",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams",
        "logs:DescribeLogGroups",
        "logs:CreateLogStream",
        "logs:CreateLogGroup",
        "ssm:GetParameter"
      ],
      "Resource": "*"
    }
  ]
}
```

AWS managed policy: `CloudWatchAgentServerPolicy` — dùng policy này thay vì viết tay.

### Cài Đặt Trên Amazon Linux 2 / Amazon Linux 2023

```bash
# Cách 1: Dùng yum/dnf (Amazon Linux)
sudo yum install amazon-cloudwatch-agent -y
# hoặc
sudo dnf install amazon-cloudwatch-agent -y

# Cách 2: Download trực tiếp
wget https://amazoncloudwatch-agent.s3.amazonaws.com/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm
```

### Cài Đặt Trên Ubuntu / Debian

```bash
wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
```

### Cài Đặt Trên Windows Server

```powershell
# PowerShell
$installer = "https://amazoncloudwatch-agent.s3.amazonaws.com/windows/amd64/latest/amazon-cloudwatch-agent.msi"
Invoke-WebRequest -Uri $installer -OutFile agent.msi
Start-Process msiexec.exe -ArgumentList "/i agent.msi /quiet" -Wait
```

### Cài Qua AWS Systems Manager (Khuyến Nghị Cho Fleet)

```bash
# Chạy trên tất cả instances qua SSM Run Command
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --parameters '{"action":["Install"],"name":["AmazonCloudWatchAgent"]}' \
  --targets '[{"Key":"tag:Environment","Values":["production"]}]'
```

---

## Cấu Hình Agent — File JSON

Agent được cấu hình bằng file JSON (`amazon-cloudwatch-agent.json`). Có thể tạo bằng wizard hoặc viết tay.

### Wizard (Trình Hướng Dẫn)

```bash
# Khởi chạy wizard tương tác
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Wizard hỏi về:
- Thu thập metrics gì (CPU, RAM, disk...)
- Log files nào cần theo dõi
- Log Group names
- Retention period

### Cấu Hình Ví Dụ Đầy Đủ

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent",
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
  },
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent",
          "mem_available",
          "mem_total"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "disk_used_percent",
          "disk_free",
          "disk_total"
        ],
        "resources": [
          "/",
          "/data",
          "/var/log"
        ],
        "metrics_collection_interval": 60
      },
      "diskio": {
        "measurement": [
          "reads",
          "writes",
          "read_bytes",
          "write_bytes",
          "iops_in_progress"
        ],
        "resources": ["nvme0n1"],
        "metrics_collection_interval": 60
      },
      "netstat": {
        "measurement": [
          "tcp_established",
          "tcp_time_wait"
        ],
        "metrics_collection_interval": 60
      },
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user",
          "cpu_usage_system"
        ],
        "totalcpu": true,
        "metrics_collection_interval": 60
      },
      "processes": {
        "measurement": [
          "running",
          "sleeping",
          "dead"
        ]
      }
    },
    "append_dimensions": {
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    },
    "aggregation_dimensions": [
      ["AutoScalingGroupName"],
      ["InstanceId", "InstanceType"]
    ]
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/application/app.log",
            "log_group_name": "/app/production/order-service",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%Y-%m-%dT%H:%M:%S.%f",
            "timezone": "UTC",
            "encoding": "utf-8",
            "multi_line_start_pattern": "{timestamp_format}"
          },
          {
            "file_path": "/var/log/nginx/error.log",
            "log_group_name": "/app/production/nginx/error",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%Y/%m/%d %H:%M:%S"
          },
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/ec2/system/messages",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    },
    "log_stream_name": "default-{instance_id}",
    "force_flush_interval": 15
  }
}
```

### Khởi Động Agent

```bash
# Load cấu hình từ file local
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s

# Hoặc load từ SSM Parameter Store
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c ssm:/cloudwatch-agent/config/production \
  -s

# Kiểm tra trạng thái
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status

# Start/Stop/Restart
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl stop amazon-cloudwatch-agent
sudo systemctl restart amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent  # Auto-start on boot
```

---

## Metrics Thu Thập Từ OS

### Memory Metrics (Chỉ Số Bộ Nhớ)

| Metric                | Ý Nghĩa                                    | Đơn Vị    |
| --------------------- | ------------------------------------------ | --------- |
| `mem_used_percent`    | % RAM đã dùng (không tính buffers/cache)   | Percent   |
| `mem_available`       | RAM thực sự available cho processes        | Bytes     |
| `mem_used`            | RAM đang được dùng                         | Bytes     |
| `mem_total`           | Tổng RAM                                   | Bytes     |
| `mem_cached`          | RAM dùng làm disk cache                    | Bytes     |
| `mem_buffered`        | RAM dùng làm IO buffer                     | Bytes     |
| `swap_used_percent`   | % Swap đã dùng                             | Percent   |

> **Quan trọng:** `mem_used_percent` **không bao gồm** buffers và cache vì Linux tự động dùng chúng khi cần (chúng là "free" về mặt thực tế). Đây là behavior đúng.

### Disk Metrics (Chỉ Số Đĩa)

| Metric                | Ý Nghĩa                                    | Đơn Vị    |
| --------------------- | ------------------------------------------ | --------- |
| `disk_used_percent`   | % disk space đã dùng (theo mount point)    | Percent   |
| `disk_free`           | Disk space còn trống                       | Bytes     |
| `disk_used`           | Disk space đã dùng                         | Bytes     |
| `disk_inodes_free`    | Số inodes còn trống (đừng bỏ qua!)        | Count     |
| `disk_inodes_used`    | Số inodes đã dùng                          | Count     |

> **Lưu ý:** Disk đầy inodes (quá nhiều files nhỏ) dù disk space vẫn còn — một lỗi thường gặp. Monitor cả `disk_inodes_free`.

### Network Stats (Thống Kê Mạng)

| Metric                   | Ý Nghĩa                          |
| ------------------------ | -------------------------------- |
| `net_bytes_sent`         | Bytes đã gửi                     |
| `net_bytes_recv`         | Bytes đã nhận                    |
| `net_drop_in`            | Packets bị drop inbound          |
| `net_drop_out`           | Packets bị drop outbound         |
| `netstat_tcp_established`| Số TCP connections established   |
| `netstat_tcp_time_wait`  | Số connections ở TIME_WAIT state |

### Process Metrics (Chỉ Số Process)

```json
"procstat": [
  {
    "pattern": "nginx",
    "measurement": [
      "cpu_usage",
      "memory_rss",
      "pid_count"
    ]
  },
  {
    "exe": "java",
    "measurement": [
      "cpu_usage",
      "memory_rss",
      "num_threads"
    ]
  }
]
```

---

## Log Collection — Thu Thập Nhật Ký

### Multi-line Log Handling (Xử Lý Log Nhiều Dòng)

Log Java stacktrace thường chiếm nhiều dòng — cần cấu hình để agent gộp lại thành một log event:

```json
{
  "file_path": "/var/log/app/application.log",
  "log_group_name": "/app/prod/java-app",
  "log_stream_name": "{instance_id}",
  "multi_line_start_pattern": "^\\d{4}-\\d{2}-\\d{2}",
  "timestamp_format": "%Y-%m-%d %H:%M:%S"
}
```

Pattern trên `^\\d{4}-\\d{2}-\\d{2}` match dòng bắt đầu bằng timestamp như `2026-05-17`:

```
Input log file:
  2026-05-17 10:35:22 ERROR OrderService - Unexpected error
  java.lang.RuntimeException: Payment timeout
      at com.example.OrderService.processOrder(OrderService.java:45)
      at com.example.OrderService.handleRequest(OrderService.java:23)
  2026-05-17 10:35:25 INFO OrderService - Request processed

Output (2 log events, không phải 5):
  Event 1: "2026-05-17 10:35:22 ERROR ... java.lang.RuntimeException..."
  Event 2: "2026-05-17 10:35:25 INFO OrderService - Request processed"
```

### Append Dimensions Cho Logs

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [{
          "file_path": "/var/log/app.log",
          "log_group_name": "/app/production",
          "log_stream_name": "{instance_id}/{hostname}"
        }]
      }
    }
  }
}
```

**Dynamic variables trong `log_stream_name`:**
- `{instance_id}` — EC2 instance ID
- `{hostname}` — OS hostname
- `{ip_address}` — IP address của instance

---

## High-Resolution Metrics

Agent có thể thu thập metrics ở độ phân giải 1 giây (High-Resolution — Độ Phân Giải Cao):

```json
{
  "metrics": {
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_user"],
        "metrics_collection_interval": 1
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 10
      }
    }
  }
}
```

> **Cân nhắc:** High-Resolution metrics có cùng mức giá với Standard metrics (theo số lượng custom metrics) nhưng tạo thêm datapoints, có thể tăng API calls và chi phí cho alarms. Chỉ dùng khi thực sự cần sub-minute visibility.

---

## StatsD & collectd Integration

### StatsD Protocol

**StatsD** là protocol phổ biến để ứng dụng emit (phát) metrics mà không cần AWS SDK. CloudWatch Agent có thể làm StatsD server:

```json
{
  "metrics": {
    "metrics_collected": {
      "statsd": {
        "service_address": ":8125",
        "metrics_collection_interval": 60,
        "metrics_aggregation_interval": 60
      }
    }
  }
}
```

Ứng dụng gửi metrics qua UDP đến localhost:8125:

```python
# Python — gửi StatsD metric
import socket

def send_statsd(metric_name, value, metric_type="g"):
    msg = f"{metric_name}:{value}|{metric_type}"
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.sendto(msg.encode(), ("localhost", 8125))

# Sử dụng
send_statsd("order.processing.time", 245, "ms")  # Timer
send_statsd("order.count", 1, "c")               # Counter
send_statsd("active.connections", 42, "g")       # Gauge
```

```javascript
// Node.js — dùng hot-shots library
const StatsD = require('hot-shots');
const dogstatsd = new StatsD({ host: 'localhost', port: 8125 });

dogstatsd.timing('order.processing.time', 245);
dogstatsd.increment('order.count');
dogstatsd.gauge('active.connections', 42);
```

### collectd Integration

Cho servers đã có **collectd** (common trên Linux) — agent có thể nhận metrics từ collectd:

```json
{
  "metrics": {
    "metrics_collected": {
      "collectd": {
        "service_address": "udp://127.0.0.1:25826"
      }
    }
  }
}
```

---

## On-Premises & Hybrid Cloud

CloudWatch Agent cũng chạy trên on-premises servers (máy chủ nội bộ), không chỉ EC2.

### Cài Đặt Trên On-Premises Server

```bash
# Linux on-premises
wget https://amazoncloudwatch-agent.s3.amazonaws.com/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm

# Cần configure AWS credentials:
aws configure --profile cloudwatch-agent
# Nhập Access Key và Secret Key của IAM user có CloudWatchAgentServerPolicy
```

### Cấu Hình Credentials Cho On-Premises

```bash
# File /root/.aws/credentials
[cloudwatch-agent]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Hoặc dùng **IAM User với cross-account role** để không lưu credentials tĩnh.

### Use Cases Hybrid Cloud

```
On-Premises DC ──→ CW Agent ──→ CloudWatch ──→ Unified Dashboard
AWS Cloud      ──→ CW Agent ──→ CloudWatch ──→ (cùng dashboard)

Benefits:
  - Một dashboard cho cả on-premises và cloud
  - Alarm chung cho toàn hệ thống
  - Không cần tool monitoring riêng cho on-premises
```

---

## Quản Lý Cấu Hình Qua SSM Parameter Store

### Lưu Config Vào SSM Parameter Store

```bash
# Lưu agent config vào SSM
aws ssm put-parameter \
  --name "/cloudwatch-agent/config/production" \
  --value file://amazon-cloudwatch-agent.json \
  --type String \
  --overwrite
```

### Load Config Từ SSM (Khuyến Nghị Cho Fleet)

```bash
# Trên từng instance — tải config từ SSM và start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c ssm:/cloudwatch-agent/config/production \
  -s
```

### Tự Động Hóa Qua SSM Run Command

```bash
# Deploy config mới cho tất cả EC2 có tag Environment=production
aws ssm send-command \
  --document-name "AmazonCloudWatch-ManageAgent" \
  --parameters '{
    "action": ["configure"],
    "mode": ["ec2"],
    "optionalConfigurationSource": ["ssm"],
    "optionalConfigurationLocation": ["/cloudwatch-agent/config/production"],
    "optionalRestart": ["yes"]
  }' \
  --targets '[{"Key":"tag:Environment","Values":["production"]}]'
```

Với cách này, bạn chỉ cần cập nhật SSM Parameter một lần → SSM tự phân phối và apply cho toàn fleet.

### Kết Hợp Với CloudFormation UserData

```yaml
# CloudFormation EC2 instance with auto-configured agent
Resources:
  AppInstance:
    Type: AWS::EC2::Instance
    Properties:
      IamInstanceProfile: !Ref InstanceProfile
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum install -y amazon-cloudwatch-agent
          /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
            -a fetch-config \
            -m ec2 \
            -c ssm:${AgentConfigParameter} \
            -s
          systemctl enable amazon-cloudwatch-agent
```

---

## Troubleshooting Agent

### Kiểm Tra Trạng Thái Agent

```bash
# Xem status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status

# Output khi hoạt động:
{
  "status": "running",
  "starttime": "2026-05-17T10:00:00Z",
  "configstatus": "configured",
  "cwoc_status": "running",
  "version": "1.300031.0"
}
```

### Log File Agent

```bash
# Agent log
tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log

# Lỗi thường gặp:
"AccessDeniedException" → IAM role thiếu permissions
"Failed to put metric data" → Endpoint không reach được (VPC endpoint?)
"No such file or directory" → Log file path sai
```

### Vấn Đề Hay Gặp

| Vấn Đề                          | Nguyên Nhân                          | Giải Pháp                            |
| -------------------------------- | ------------------------------------ | ------------------------------------ |
| Không có metric trong CW         | IAM role thiếu PutMetricData         | Thêm CloudWatchAgentServerPolicy     |
| Metric xuất hiện nhưng sai data  | Cấu hình measurement sai             | Kiểm tra agent log, test config      |
| Log không gửi lên CW             | File path sai hoặc permission OS     | Kiểm tra chmod, file_path chính xác  |
| Agent không start                | Config JSON syntax error             | Validate JSON, xem agent log          |
| High CPU từ agent                | Quá nhiều metrics/logs thu thập      | Tăng `metrics_collection_interval`   |

### Validate Config Trước Khi Apply

```bash
# Kiểm tra config hợp lệ không
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent \
  -config /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -test
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao EC2 không có RAM metric mặc định mà phải cài agent?

AWS chỉ có thể monitor EC2 ở hypervisor level — lớp quản lý ảo hóa bên dưới OS. Memory utilization là thông tin nội bộ của OS (kernel quản lý) mà hypervisor không thể truy cập. CloudWatch Agent chạy trong OS nên có thể đọc từ `/proc/meminfo` và gửi lên CloudWatch.

### Q2: So sánh CloudWatch Agent với Prometheus Node Exporter?

| Tiêu Chí         | CloudWatch Agent               | Prometheus Node Exporter        |
| ---------------- | ------------------------------ | --------------------------------|
| **Destination**  | CloudWatch (AWS)               | Prometheus server               |
| **Protocol**     | Push (HTTPS to CloudWatch API) | Pull (Prometheus scrapes)       |
| **AWS Native**   | Có — IAM, SSM integration      | Không                           |
| **Multi-cloud**  | Không (chỉ CloudWatch)         | Có — Prometheus portable        |
| **StatsD**       | Có built-in                    | Không (cần statsd_exporter)     |
| **Managed**      | AWS quản lý updates            | Tự quản lý                      |

### Q3: CloudWatch Agent có tốn nhiều tài nguyên trên instance không?

Thông thường **không đáng kể**:
- CPU: 0.5–2% của 1 core
- Memory: ~50–100 MB RAM
- Network: Tùy volume metrics/logs, thường < 1 MB/phút

Tuy nhiên, nếu thu thập quá nhiều metrics với interval ngắn (1s) và gửi nhiều log files, có thể tăng lên. Monitor agent's own resource usage bằng chính agent (process monitoring).

### Q4: Làm thế nào deploy CloudWatch Agent config đồng nhất cho 500 servers?

**Pattern chuẩn:**
1. Lưu config trong **SSM Parameter Store** — một nguồn sự thật duy nhất
2. Cài agent trong **AMI Golden Image** — instances mới đã có agent
3. Dùng **SSM Run Command / State Manager** để apply config và restart agent
4. **User Data** trong Launch Template để load config từ SSM khi instance khởi động

Khi cần thay đổi config: cập nhật SSM Parameter → SSM Run Command deploy đến tất cả instances theo tag.

### Q5: Sự khác biệt giữa `mem_used_percent` và `mem_available_percent`?

```
Total RAM: 16 GB
  Used by processes:  6 GB
  Buffers + Cache:    8 GB  (Linux dùng làm disk cache, nhưng có thể free ngay khi cần)
  Actually free:      2 GB

mem_used_percent = Used/(Total) = 6/16 = 37.5%  (không tính buffers/cache)
mem_available     = 2 GB + 8 GB = 10 GB (processes + cache đều available)
mem_available_percent = 10/16 = 62.5%

Rule of thumb:
  mem_available_percent < 10% → Thực sự sắp hết RAM
  mem_used_percent > 90% → Chưa chắc nguy hiểm nếu nhiều cache
```

Dùng `mem_available` hoặc `mem_available_percent` để alarm chính xác hơn về RAM pressure thực sự.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [5-container-application-insights.md](./5-container-application-insights.md) | [README.md](./README.md)
