# Dynamic Blocks — Khối Động trong Terraform

> **Dynamic Blocks** — Khối Động — cho phép tạo các khối cấu hình lặp lại bên trong resource, data source, provider hoặc provisioner một cách linh hoạt dựa trên dữ liệu đầu vào. Đây là công cụ mạnh nhưng dễ bị lạm dụng.

---

## 1. Dynamic Block Là Gì?

### Vấn Đề Cần Giải Quyết

Khi cấu hình có các khối lặp lại với số lượng thay đổi theo môi trường:

```hcl
# Cách viết thông thường — cứng nhắc, không linh hoạt
resource "aws_security_group" "example" {
  name = "example"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}
```

Vấn đề: Nếu số lượng rule thay đổi theo môi trường, không thể dùng cấu hình cứng.

### Giải Pháp: Dynamic Block

```hcl
variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { from_port = 80,  to_port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { from_port = 443, to_port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { from_port = 22,  to_port = 22,  protocol = "tcp", cidr_blocks = ["10.0.0.0/8"] },
  ]
}

resource "aws_security_group" "example" {
  name = "example"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

---

## 2. Cú Pháp Dynamic Block

### Cấu Trúc Cơ Bản

```hcl
dynamic "<TÊN_KHỐI>" {
  for_each = <COLLECTION>       # Tập hợp để lặp (list, map, set)
  iterator = <TÊN_BIẾN_LẶP>   # Tùy chọn — tên biến lặp (mặc định = tên khối)
  labels   = [...]              # Tùy chọn — dành cho khối có label

  content {
    # Nội dung của khối — dùng <iterator>.value hoặc <iterator>.key
  }
}
```

### Giải Thích Các Thành Phần

| Thành Phần | Bắt Buộc | Mô Tả |
|-----------|----------|-------|
| `"TÊN_KHỐI"` | ✅ | Tên của nested block cần tạo động |
| `for_each` | ✅ | Tập hợp dữ liệu để lặp |
| `iterator` | ❌ | Tên biến lặp tùy chỉnh (mặc định = tên khối) |
| `labels` | ❌ | Dành cho block yêu cầu label (ví dụ: `lifecycle`) |
| `content {}` | ✅ | Nội dung thực sự của block được tạo |

---

## 3. Ví Dụ Thực Tế Theo Từng Use Case

### 3.1 AWS Security Group Rules

```hcl
variable "security_rules" {
  type = map(object({
    type        = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))

  default = {
    http = {
      type        = "ingress"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP traffic"
    }
    https = {
      type        = "ingress"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS traffic"
    }
  }
}

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web server security group"
  vpc_id      = var.vpc_id

  # Dynamic Ingress Rules
  dynamic "ingress" {
    for_each = { for k, v in var.security_rules : k => v if v.type == "ingress" }
    iterator = rule

    content {
      description = rule.value.description
      from_port   = rule.value.from_port
      to_port     = rule.value.to_port
      protocol    = rule.value.protocol
      cidr_blocks = rule.value.cidr_blocks
    }
  }

  # Dynamic Egress Rules
  dynamic "egress" {
    for_each = { for k, v in var.security_rules : k => v if v.type == "egress" }
    iterator = rule

    content {
      description = rule.value.description
      from_port   = rule.value.from_port
      to_port     = rule.value.to_port
      protocol    = rule.value.protocol
      cidr_blocks = rule.value.cidr_blocks
    }
  }

  tags = {
    Name = "web-sg"
  }
}
```

### 3.2 AWS ALB Listener Rules — Application Load Balancer

```hcl
variable "routing_rules" {
  type = list(object({
    path_pattern = string
    target_arn   = string
    priority     = number
  }))

  default = [
    { path_pattern = "/api/*",    target_arn = "arn:aws:...", priority = 10 },
    { path_pattern = "/static/*", target_arn = "arn:aws:...", priority = 20 },
    { path_pattern = "/health",   target_arn = "arn:aws:...", priority = 30 },
  ]
}

resource "aws_lb_listener_rule" "routing" {
  listener_arn = aws_lb_listener.https.arn

  dynamic "action" {
    for_each = [var.routing_rules[count.index]]
    content {
      type             = "forward"
      target_group_arn = action.value.target_arn
    }
  }

  dynamic "condition" {
    for_each = [var.routing_rules[count.index]]
    content {
      path_pattern {
        values = [condition.value.path_pattern]
      }
    }
  }

  priority = var.routing_rules[count.index].priority
  count    = length(var.routing_rules)
}
```

### 3.3 Kubernetes ConfigMap — Cấu Hình Ứng Dụng

```hcl
variable "app_configs" {
  type = map(map(string))
  default = {
    "app-config" = {
      DATABASE_HOST     = "db.internal"
      DATABASE_PORT     = "5432"
      CACHE_HOST        = "redis.internal"
      LOG_LEVEL         = "info"
    }
    "feature-flags" = {
      ENABLE_DARK_MODE  = "true"
      ENABLE_ANALYTICS  = "false"
      MAX_UPLOAD_SIZE   = "10MB"
    }
  }
}

resource "kubernetes_config_map" "app" {
  for_each = var.app_configs

  metadata {
    name      = each.key
    namespace = var.namespace
  }

  data = each.value
}
```

### 3.4 Dynamic Blocks Lồng Nhau — Nested Dynamic Blocks

```hcl
variable "firewall_rules" {
  type = list(object({
    name        = string
    action      = string
    priority    = number
    ip_sets     = list(string)
    match_conditions = list(object({
      field   = string
      values  = list(string)
    }))
  }))
}

resource "aws_wafv2_rule_group" "example" {
  name     = "example-rule-group"
  scope    = "REGIONAL"
  capacity = 100

  dynamic "rule" {
    for_each = var.firewall_rules
    iterator = firewall_rule

    content {
      name     = firewall_rule.value.name
      priority = firewall_rule.value.priority

      action {
        dynamic "allow" {
          for_each = firewall_rule.value.action == "allow" ? [1] : []
          content {}
        }
        dynamic "block" {
          for_each = firewall_rule.value.action == "block" ? [1] : []
          content {}
        }
      }

      statement {
        # Dynamic block lồng trong dynamic block
        dynamic "byte_match_statement" {
          for_each = firewall_rule.value.match_conditions
          iterator = condition

          content {
            search_string         = condition.value.values[0]
            positional_constraint = "CONTAINS"

            field_to_match {
              dynamic "uri_path" {
                for_each = condition.value.field == "uri" ? [1] : []
                content {}
              }
              dynamic "query_string" {
                for_each = condition.value.field == "query" ? [1] : []
                content {}
              }
            }

            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }

      visibility_config {
        cloudwatch_metrics_enabled = true
        metric_name                = firewall_rule.value.name
        sampled_requests_enabled   = true
      }
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "example-rule-group"
    sampled_requests_enabled   = true
  }
}
```

---

## 4. Dynamic Blocks với Iterator Tùy Chỉnh

### Tại Sao Cần `iterator`?

Khi tên block trùng với biến đã khai báo, hoặc muốn tên biến rõ ràng hơn:

```hcl
variable "tags_list" {
  type = list(object({
    key   = string
    value = string
  }))
}

resource "aws_autoscaling_group" "example" {
  # ...

  dynamic "tag" {
    for_each = var.tags_list
    iterator = asg_tag   # Đặt tên rõ ràng thay vì dùng "tag"

    content {
      key                 = asg_tag.value.key
      value               = asg_tag.value.value
      propagate_at_launch = true
    }
  }
}
```

### Truy Cập `key` và `value` Trong Iterator

```hcl
# for_each = map → iterator.key là map key, iterator.value là map value
dynamic "ingress" {
  for_each = {
    http  = 80
    https = 443
    ssh   = 22
  }
  iterator = port_rule

  content {
    description = "Allow ${port_rule.key}"
    from_port   = port_rule.value
    to_port     = port_rule.value
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# for_each = list → iterator.key là index (0,1,2...), iterator.value là phần tử
dynamic "ingress" {
  for_each = [80, 443, 22]
  iterator = port_item

  content {
    from_port   = port_item.value
    to_port     = port_item.value
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 5. Kỹ Thuật Nâng Cao

### 5.1 Dynamic Block Có Điều Kiện — Conditional Dynamic Block

```hcl
variable "enable_ssl_redirect" {
  type    = bool
  default = true
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  # Tạo redirect action chỉ khi SSL được bật
  dynamic "default_action" {
    for_each = var.enable_ssl_redirect ? [1] : []
    content {
      type = "redirect"
      redirect {
        port        = "443"
        protocol    = "HTTPS"
        status_code = "HTTP_301"
      }
    }
  }

  # Tạo forward action khi SSL bị tắt
  dynamic "default_action" {
    for_each = var.enable_ssl_redirect ? [] : [1]
    content {
      type             = "forward"
      target_group_arn = aws_lb_target_group.main.arn
    }
  }
}
```

### 5.2 Dynamic Blocks Với Expression Phức Tạp

```hcl
locals {
  # Lọc và biến đổi dữ liệu trước khi truyền vào dynamic block
  active_rules = {
    for k, v in var.all_rules :
    k => v
    if v.enabled && v.environment == var.environment
  }
}

resource "aws_security_group" "filtered" {
  name = "filtered-sg"

  dynamic "ingress" {
    for_each = local.active_rules
    iterator = rule

    content {
      description = "Rule: ${rule.key} — ${rule.value.description}"
      from_port   = rule.value.port
      to_port     = rule.value.port
      protocol    = "tcp"
      cidr_blocks = rule.value.cidr_blocks
    }
  }
}
```

### 5.3 Kết Hợp `for` Expression với Dynamic Block

```hcl
variable "services" {
  type = map(object({
    port     = number
    protocol = string
  }))
  default = {
    web   = { port = 80,   protocol = "HTTP"  }
    api   = { port = 8080, protocol = "HTTP"  }
    grpc  = { port = 9090, protocol = "HTTP2" }
  }
}

resource "aws_lb_target_group" "services" {
  for_each = var.services

  name     = each.key
  port     = each.value.port
  protocol = each.value.protocol
  vpc_id   = var.vpc_id

  dynamic "health_check" {
    for_each = [each.value]
    content {
      enabled             = true
      healthy_threshold   = 2
      unhealthy_threshold = 3
      timeout             = 5
      interval            = 30
      path                = each.key == "api" ? "/health" : "/"
      port                = "traffic-port"
      protocol            = health_check.value.protocol
    }
  }
}
```

---

## 6. Khi Nào Dùng và Khi Nào Không Dùng Dynamic Blocks

### ✅ Nên Dùng Dynamic Block Khi

| Tình Huống | Ví Dụ |
|-----------|-------|
| Số lượng nested block thay đổi theo biến | Security group rules thay đổi theo môi trường |
| Nested block được điều khiển bởi feature flags | SSL config chỉ khi có certificate |
| Module cần linh hoạt cho nhiều use case | Module tái sử dụng cho nhiều team |
| Tránh code lặp lại rõ ràng | 10+ ingress rules giống nhau về cấu trúc |

### ❌ Không Nên Dùng Dynamic Block Khi

| Vấn Đề | Giải Pháp Tốt Hơn |
|--------|------------------|
| Block chỉ có 1-2 phiên bản cố định | Viết block thẳng ra cho rõ ràng |
| Logic quá phức tạp trong `for_each` | Đưa logic vào `locals` trước |
| Dynamic blocks lồng nhau quá sâu (>2 cấp) | Tách thành nhiều resource |
| Làm khó đọc mà không có lợi thực sự | Đơn giản hóa thiết kế |

---

## 7. Bẫy Phổ Biến — Common Pitfalls

### 7.1 Quên Rằng `for_each` Trong Dynamic Block Không Thể Phụ Thuộc Vào Resource

```hcl
# ❌ SAI — Không thể dùng resource attribute trực tiếp trong for_each của dynamic block
resource "aws_security_group" "bad" {
  dynamic "ingress" {
    # Lỗi: for_each của dynamic block phải là known value tại plan time
    for_each = aws_instance.servers[*].private_ip
    content {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["${ingress.value}/32"]
    }
  }
}

# ✅ ĐÚNG — Dùng variable hoặc local
locals {
  server_ips = ["10.0.1.10", "10.0.1.11", "10.0.1.12"]
}

resource "aws_security_group" "good" {
  dynamic "ingress" {
    for_each = local.server_ips
    iterator = server

    content {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["${server.value}/32"]
      description = "SSH from server ${server.key}"
    }
  }
}
```

### 7.2 Dynamic Block Trống — Empty Dynamic Block

```hcl
# Truyền list rỗng → không tạo block nào — hành vi hợp lệ và hữu ích
variable "extra_rules" {
  type    = list(any)
  default = []  # Rỗng → dynamic block không sinh ra khối nào
}

resource "aws_security_group" "conditional" {
  dynamic "ingress" {
    for_each = var.extra_rules  # Rỗng → OK, không có ingress rule nào được tạo
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidrs
    }
  }
}
```

---

## 8. So Sánh Dynamic Block với Các Giải Pháp Thay Thế

| Giải Pháp | Khi Nào Dùng | Ưu Điểm | Nhược Điểm |
|-----------|-------------|----------|------------|
| **Dynamic Block** | Nested block có số lượng thay đổi | Linh hoạt, ít code lặp | Khó đọc nếu phức tạp |
| **Viết block thẳng** | Cấu hình cố định | Rõ ràng, dễ đọc | Không linh hoạt |
| **Tách thành nhiều resource** | Logic quá phức tạp | Dễ test, dễ debug | Nhiều resource hơn |
| **Dùng `for_each` ở resource level** | Tạo nhiều resource cùng loại | Sạch hơn dynamic block | Không dùng được trong nested block |

---

## 9. Checklist Sử Dụng Dynamic Blocks

```
✅ Trước khi thêm dynamic block:
   □ Đã thử viết block thẳng chưa? Có cần thiết không?
   □ for_each expression có đơn giản và rõ ràng không?
   □ Nếu phức tạp, đã đưa logic vào locals chưa?
   □ Có cần đặt tên iterator rõ ràng hơn tên mặc định không?

✅ Khi viết dynamic block:
   □ Có sử dụng iterator đúng chỗ không?
   □ Đã test với list rỗng (empty list) chưa?
   □ Đã test với 1 phần tử và nhiều phần tử chưa?
   □ Code có dễ đọc sau khi viết xong không?

✅ Code review dynamic block:
   □ Người khác có hiểu block này làm gì không?
   □ Có thể đơn giản hóa thêm không?
   □ Logic filter có nên tách vào locals không?
```

---

## 10. Câu Hỏi Phỏng Vấn Liên Quan

**Q: Dynamic Block khác gì `for_each` ở cấp resource?**

A: `for_each` ở cấp resource tạo nhiều **resource riêng biệt**. Dynamic Block tạo nhiều **nested block bên trong một resource**. Không thể dùng `for_each` resource để thay thế dynamic block vì nested block không phải là resource.

**Q: Khi nào không nên dùng Dynamic Block?**

A: Khi cấu hình đơn giản và cố định (1-2 block), khi logic `for_each` quá phức tạp và làm khó đọc, hoặc khi có thể tách resource ra để code rõ ràng hơn.

**Q: Làm sao để dynamic block không tạo ra block nào?**

A: Truyền list hoặc map rỗng vào `for_each`. Terraform sẽ bỏ qua dynamic block đó và không sinh ra nested block nào.

---

## 🔗 Điều Hướng

| | |
|---|---|
| ← Phần trước | [README.md](README.md) |
| → Bài tiếp | [2-meta-arguments.md](2-meta-arguments.md) |
| ↑ Mục lục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Độ Khó:** ⭐⭐⭐ Nâng Cao
