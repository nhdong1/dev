# WAF — Web Application Firewall (Tường Lửa Ứng Dụng Web)

> AWS WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web) bảo vệ ứng dụng web khỏi các cuộc tấn công ở tầng ứng dụng (Layer 7), bao gồm OWASP Top 10, bot attacks, và các mối đe dọa web phổ biến.

## 📚 Mục Lục

1. [Khái Niệm WAF](#khái-niệm-waf)
2. [Thành Phần WAF](#thành-phần-waf)
3. [Rule Types — Loại Rule](#rule-types--loại-rule)
4. [Managed Rule Groups](#managed-rule-groups)
5. [Thiết Lập WAF](#thiết-lập-waf)
6. [Logging và Monitoring](#logging-và-monitoring)
7. [Best Practices](#best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm WAF

### AWS WAF là gì?

AWS WAF là Layer 7 firewall (tường lửa tầng ứng dụng) kiểm tra nội dung HTTP/HTTPS request và quyết định Allow (cho phép) hoặc Block (chặn) dựa trên các rule (quy tắc) đã định nghĩa.

### Tích Hợp với Các Dịch Vụ AWS

WAF có thể gắn với:
- **CloudFront** (CDN — Content Delivery Network) — bảo vệ ở edge, gần người dùng nhất
- **ALB** (Application Load Balancer — Cân Bằng Tải Ứng Dụng)
- **API Gateway** — bảo vệ REST API và WebSocket API
- **AppSync** — bảo vệ GraphQL API
- **Cognito User Pool** — bảo vệ endpoint xác thực

```
Client Request
      │
      ▼
CloudFront → WAF Rules → Allow/Block
      │ (if allowed)
      ▼
ALB → WAF Rules → Allow/Block
      │
      ▼
Application (EC2/ECS/Lambda)
```

---

## Thành Phần WAF

### Web ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập Web)

Web ACL là container chứa các rule và rule groups. Mỗi request được đánh giá lần lượt qua các rule, hành động mặc định (Default Action) áp dụng nếu không rule nào khớp.

```
Web ACL: my-production-waf
  Priority 1: AWSManagedRulesCommonRuleSet    → Block
  Priority 2: AWSManagedRulesSQLiRuleSet       → Block
  Priority 3: RateLimit-login-endpoint         → Block (100 req/5min)
  Priority 4: Allow-known-bots-googlebot       → Allow
  Priority 5: IPBlock-malicious-list           → Block
  Default Action: Allow
```

### WCU — WAF Capacity Units (Đơn Vị Năng Lực WAF)

Mỗi rule tiêu thụ WCU khác nhau. Web ACL có giới hạn 5000 WCU (có thể tăng). Cần lên kế hoạch capacity khi thiết kế rule set.

| Loại Rule | WCU tiêu thụ |
|---|---|
| IP Set match | 1 WCU |
| Regex match | 3–25 WCU |
| Managed rule group (thường) | 700 WCU |
| Bot Control managed rule | 50 WCU |

---

## Rule Types — Loại Rule

### 1. IP Set Rules (Rule Danh Sách IP)

```json
{
  "Name": "BlockMaliciousIPs",
  "Priority": 10,
  "Statement": {
    "IPSetReferenceStatement": {
      "ARN": "arn:aws:wafv2:...:ipset/malicious-ips/..."
    }
  },
  "Action": {"Block": {}},
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BlockMaliciousIPs"
  }
}
```

### 2. Geo Match Rules (Rule Theo Vị Trí Địa Lý)

```json
{
  "Name": "BlockHighRiskCountries",
  "Statement": {
    "GeoMatchStatement": {
      "CountryCodes": ["CN", "RU", "KP", "IR"]
    }
  },
  "Action": {"Block": {}}
}
```

### 3. Rate-Based Rules (Rule Giới Hạn Tốc Độ)

Tự động block IP vượt quá ngưỡng request trong 5 phút:

```json
{
  "Name": "RateLimit-LoginEndpoint",
  "Statement": {
    "RateBasedStatement": {
      "Limit": 100,
      "AggregateKeyType": "IP",
      "ScopeDownStatement": {
        "ByteMatchStatement": {
          "SearchString": "/api/v1/login",
          "FieldToMatch": {"UriPath": {}},
          "TextTransformations": [{"Priority": 0, "Type": "LOWERCASE"}],
          "PositionalConstraint": "STARTS_WITH"
        }
      }
    }
  },
  "Action": {"Block": {}}
}
```

### 4. String Match Rules (Rule Khớp Chuỗi)

```json
{
  "Name": "BlockSQLiInHeader",
  "Statement": {
    "ByteMatchStatement": {
      "SearchString": "UNION SELECT",
      "FieldToMatch": {
        "SingleHeader": {"Name": "authorization"}
      },
      "TextTransformations": [{"Priority": 0, "Type": "UPPERCASE"}],
      "PositionalConstraint": "CONTAINS"
    }
  },
  "Action": {"Block": {}}
}
```

### 5. Regex Pattern Set Rules

```json
{
  "Name": "BlockXSSPatterns",
  "Statement": {
    "RegexPatternSetReferenceStatement": {
      "ARN": "arn:aws:wafv2:...:regexpatternset/xss-patterns/...",
      "FieldToMatch": {"QueryString": {}},
      "TextTransformations": [{"Priority": 0, "Type": "URL_DECODE"}]
    }
  },
  "Action": {"Block": {}}
}
```

---

## Managed Rule Groups

### AWS Managed Rules (Miễn Phí)

AWS cung cấp các rule group được quản lý, tự động cập nhật khi xuất hiện threat mới:

| Rule Group | Bảo Vệ Chống | WCU |
|---|---|---|
| **AWSManagedRulesCommonRuleSet** | OWASP Top 10 tổng quát | 700 |
| **AWSManagedRulesKnownBadInputsRuleSet** | Log4Shell, Spring4Shell, SSRF | 200 |
| **AWSManagedRulesSQLiRuleSet** | SQL Injection (Chèn SQL) | 200 |
| **AWSManagedRulesLinuxRuleSet** | Linux LFI/RFI attacks | 200 |
| **AWSManagedRulesWindowsRuleSet** | Windows attacks, PowerShell injection | 200 |
| **AWSManagedRulesAmazonIpReputationList** | IP có danh tiếng xấu, botnet | 25 |
| **AWSManagedRulesAnonymousIpList** | VPN, Tor exit nodes | 50 |
| **AWSManagedRulesBotControlRuleSet** | Bot management | 50 |

### Marketplace Managed Rules (Có Phí)

- **Fortinet FortiWeb** — enterprise-grade WAF rules
- **F5 Advanced WAF** — signature-based detection
- **Imperva** — threat intelligence integration

---

## Thiết Lập WAF

### Terraform: WAF Web ACL Hoàn Chỉnh

```terraform
# IP Set cho whitelist
resource "aws_wafv2_ip_set" "office_ips" {
  name               = "office-ip-whitelist"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"
  addresses          = ["203.0.113.0/24", "198.51.100.50/32"]
}

# Web ACL
resource "aws_wafv2_web_acl" "main" {
  name  = "production-waf"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  # Rule 1: Block by Reputation List
  rule {
    name     = "AWSManagedRulesAmazonIpReputationList"
    priority = 10

    override_action { none {} }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAmazonIpReputationList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesAmazonIpReputationList"
      sampled_requests_enabled   = true
    }
  }

  # Rule 2: OWASP Common Rules
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 20

    override_action { none {} }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"

        # Chỉ count (không block) rule này để test trước
        rule_action_override {
          name           = "SizeRestrictions_BODY"
          action_to_use { count {} }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # Rule 3: Rate Limit Login
  rule {
    name     = "RateLimitLogin"
    priority = 30

    action { block {} }

    statement {
      rate_based_statement {
        limit              = 50
        aggregate_key_type = "IP"

        scope_down_statement {
          byte_match_statement {
            search_string         = "/auth/login"
            positional_constraint = "STARTS_WITH"
            field_to_match { uri_path {} }
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitLogin"
      sampled_requests_enabled   = true
    }
  }

  # Rule 4: Allow Office IPs (higher priority = evaluated first when lower number)
  rule {
    name     = "AllowOfficeIPs"
    priority = 5

    action { allow {} }

    statement {
      ip_set_reference_statement {
        arn = aws_wafv2_ip_set.office_ips.arn
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AllowOfficeIPs"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ProductionWAF"
    sampled_requests_enabled   = true
  }
}

# Gắn WAF vào ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

---

## Logging và Monitoring

### Bật WAF Logging

```bash
# Tạo Kinesis Firehose cho WAF logs
# Log group phải có prefix "aws-waf-logs-"
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:...:regional/webacl/production-waf/...",
    "LogDestinationConfigs": [
      "arn:aws:firehose:...:deliverystream/aws-waf-logs-stream"
    ],
    "RedactedFields": [
      {"SingleHeader": {"Name": "authorization"}},
      {"SingleHeader": {"Name": "cookie"}}
    ]
  }'
```

### CloudWatch Metrics Quan Trọng

| Metric | Ý Nghĩa |
|---|---|
| `AllowedRequests` | Số request được phép |
| `BlockedRequests` | Số request bị block |
| `CountedRequests` | Request chỉ được đếm (count mode) |
| `PassedRequests` | Request không khớp rule nào |

### Athena Query cho WAF Logs

```sql
-- Tìm top 10 IP bị block nhiều nhất
SELECT
  httprequest.clientip,
  COUNT(*) as blocked_count,
  terminatingruleid
FROM waf_logs
WHERE action = 'BLOCK'
  AND timestamp > to_unixtime(now() - interval '1' hour)
GROUP BY httprequest.clientip, terminatingruleid
ORDER BY blocked_count DESC
LIMIT 10;
```

---

## Best Practices

### Triển Khai Theo Giai Đoạn

```
Giai đoạn 1 — Count Mode (Chế Độ Đếm):
  Bật tất cả rule với override_action = count
  Quan sát log 1-2 tuần
  Tìm false positives (cảnh báo nhầm)

Giai đoạn 2 — Block một phần:
  Chuyển rule an toàn sang Block
  Giữ rule có nhiều false positive ở Count

Giai đoạn 3 — Full enforcement:
  Chuyển tất cả sang Block sau khi đã tinh chỉnh
```

### Tránh False Positives Phổ Biến

| Tình Huống | Giải Pháp |
|---|---|
| API nhận Base64 trong body | Exclude body field cho rule đó |
| Admin upload file lớn | Whitelist IP admin, exclude size rules |
| Ứng dụng gửi SQL trong comment | Scope down rule đến path cụ thể |
| Healthcheck bị block | Whitelist IP của load balancer |

---

## Câu Hỏi Phỏng Vấn

### Q: WAF hoạt động ở tầng nào của mô hình OSI?

**Trả lời:** Tầng 7 (Application Layer — Tầng Ứng Dụng). WAF kiểm tra nội dung HTTP/HTTPS request: URL, headers, body, query string, cookies. Điều này khác với Security Groups và NACLs hoạt động ở Layer 3/4 (chỉ kiểm tra IP và port).

### Q: Sự khác biệt giữa WAF Rule và Managed Rule Group?

**Trả lời:**
- **Custom Rule:** Bạn tự viết logic (IP match, regex, rate limit); phải tự cập nhật khi có threat mới
- **Managed Rule Group:** AWS hoặc vendor viết và duy trì; tự động cập nhật khi xuất hiện CVE hay pattern mới (ví dụ: Log4Shell được thêm vào trong vòng giờ)

Khuyến nghị: Dùng AWS Managed Rules làm baseline, thêm custom rules cho logic đặc thù của ứng dụng.

### Q: Làm thế nào xử lý false positives trong WAF?

**Trả lời:**
1. Bắt đầu với **Count mode** để quan sát trước khi block
2. Phân tích log để tìm pattern của false positives
3. Dùng **Rule Override** để tắt rule cụ thể trong Managed Group hoặc thêm exception
4. Dùng **Scope-down statement** để chỉ áp dụng rule cho một số path nhất định
5. Whitelist IP của monitoring tools, healthcheck services

---

## 🔗 Xem Thêm

- [6-shield-ddos.md](6-shield-ddos.md) — Bảo vệ chống DDoS (Distributed Denial of Service)
- [7-network-firewall.md](7-network-firewall.md) — Network-level firewall
- [README.md](README.md) — Tổng quan Network Security

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
