# `for_each` vs `count` — So Sánh Chuyên Sâu

> `for_each` và `count` đều là meta-argument dùng để tạo nhiều instance của cùng một resource hoặc module. Chọn sai giữa hai cái này là nguồn gốc của nhiều sự cố hạ tầng nghiêm trọng trong môi trường production.

---

## 1. Tổng Quan Nhanh

| | `count` | `for_each` |
|---|---------|-----------|
| **Loại dữ liệu** | `number` | `map` hoặc `set(string)` |
| **Truy cập instance** | `resource[0]`, `resource[1]` | `resource["key"]` |
| **Thêm vào giữa list** | ⚠️ Nguy hiểm — gây replace | ✅ An toàn |
| **Xóa phần tử giữa** | ⚠️ Nguy hiểm — shift index | ✅ An toàn — chỉ xóa đúng resource |
| **Key có nghĩa** | ❌ Chỉ có số | ✅ Tên có nghĩa |
| **Điều kiện tạo/không tạo** | `count = var.enable ? 1 : 0` | `for_each = var.enable ? toset(["x"]) : toset([])` |
| **Dùng khi nào** | Số lượng cố định, không thay đổi | Collection có thể thay đổi |

---

## 2. `count` — Đếm Đơn Giản

### Cú Pháp Cơ Bản

```hcl
variable "instance_count" {
  type    = number
  default = 3
}

resource "aws_instance" "web" {
  count = var.instance_count

  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"  # count.index: 0, 1, 2
  }
}

# Truy cập instance cụ thể
output "first_instance_ip" {
  value = aws_instance.web[0].public_ip
}

# Truy cập tất cả instance
output "all_ips" {
  value = aws_instance.web[*].public_ip  # Splat expression — lấy tất cả
}
```

### `count = 0` hoặc `count = 1` — Pattern Điều Kiện

```hcl
variable "create_bastion" {
  type    = bool
  default = false
}

resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0

  ami           = var.ami_id
  instance_type = "t3.nano"
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name = "bastion-host"
  }
}

# Truy cập có điều kiện — cẩn thận khi count = 0
output "bastion_ip" {
  value = var.create_bastion ? aws_instance.bastion[0].public_ip : null
}
```

---

## 3. Vấn Đề Nghiêm Trọng Với `count` — The Index Shifting Problem

### Tình Huống Nguy Hiểm

```hcl
# TRƯỚC: 3 subnet được tạo
variable "subnet_cidrs" {
  default = [
    "10.0.1.0/24",  # index 0 → subnet A
    "10.0.2.0/24",  # index 1 → subnet B
    "10.0.3.0/24",  # index 2 → subnet C
  ]
}

resource "aws_subnet" "main" {
  count      = length(var.subnet_cidrs)
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_cidrs[count.index]
  tags = { Name = "subnet-${count.index}" }
}
# Kết quả: subnet A (index 0), subnet B (index 1), subnet C (index 2)
```

```hcl
# SAU: Xóa subnet B ở giữa
variable "subnet_cidrs" {
  default = [
    "10.0.1.0/24",  # index 0 → subnet A (không đổi)
    # Đã xóa "10.0.2.0/24" (subnet B)
    "10.0.3.0/24",  # index 1 → subnet C (trước đây là index 2!)
  ]
}
```

**Terraform sẽ làm gì?**

```
Plan output:
  ~ aws_subnet.main[1]   # Sẽ UPDATE subnet B → trở thành subnet C
  - aws_subnet.main[2]   # Sẽ XÓA subnet C

# Vấn đề: Terraform coi index 1 bây giờ là "10.0.3.0/24"
# → Terraform sẽ MODIFY subnet B thành subnet C
# → Terraform sẽ XÓA subnet C cũ (nhưng subnet C cũ bây giờ đã được update thành subnet C!)
# Kết quả: RDS instance đang chạy trong subnet C bị xóa → DOWNTIME!
```

---

## 4. `for_each` — Giải Pháp Ổn Định

### Cú Pháp Với Map

```hcl
variable "subnets" {
  type = map(object({
    cidr = string
    az   = string
    tier = string
  }))

  default = {
    "subnet-a" = { cidr = "10.0.1.0/24", az = "us-east-1a", tier = "public"  }
    "subnet-b" = { cidr = "10.0.2.0/24", az = "us-east-1b", tier = "private" }
    "subnet-c" = { cidr = "10.0.3.0/24", az = "us-east-1c", tier = "private" }
  }
}

resource "aws_subnet" "main" {
  for_each = var.subnets

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = {
    Name = each.key                    # "subnet-a", "subnet-b", "subnet-c"
    Tier = each.value.tier
  }
}

# Xóa subnet-b: chỉ subnet-b bị xóa
# subnet-a và subnet-c: KHÔNG bị ảnh hưởng
```

### Cú Pháp Với `toset()` — Chuyển List Thành Set

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "public" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, each.key))
  availability_zone = each.key  # "us-east-1a", "us-east-1b", "us-east-1c"

  tags = {
    Name = "public-${each.key}"
  }
}

# Truy cập subnet cụ thể
output "subnet_ids" {
  value = { for az, subnet in aws_subnet.public : az => subnet.id }
}
```

---

## 5. So Sánh Chi Tiết Qua Ví Dụ Thực Tế

### 5.1 Tạo IAM Users — Nên Dùng `for_each`

```hcl
# ❌ count — Nguy hiểm nếu xóa user ở giữa
variable "usernames_list" {
  default = ["alice", "bob", "charlie"]
}

resource "aws_iam_user" "team" {
  count = length(var.usernames_list)
  name  = var.usernames_list[count.index]
}

# Xóa "bob" → charlie bị destroy và tạo lại → mất permissions!

# ✅ for_each — An toàn
variable "usernames" {
  type    = set(string)
  default = ["alice", "bob", "charlie"]
}

resource "aws_iam_user" "team" {
  for_each = var.usernames
  name     = each.key
}

# Xóa "bob" → chỉ aws_iam_user.team["bob"] bị xóa
# alice và charlie không bị ảnh hưởng
```

### 5.2 Tạo Multiple AWS Regions — `for_each` với map phức tạp

```hcl
variable "regional_configs" {
  type = map(object({
    instance_type = string
    min_size      = number
    max_size      = number
    ami_id        = string
  }))

  default = {
    "us-east-1" = {
      instance_type = "t3.large"
      min_size      = 2
      max_size      = 10
      ami_id        = "ami-12345678"
    }
    "ap-southeast-1" = {
      instance_type = "t3.medium"
      min_size      = 1
      max_size      = 5
      ami_id        = "ami-87654321"
    }
  }
}

resource "aws_autoscaling_group" "regional" {
  for_each = var.regional_configs

  name             = "asg-${each.key}"
  min_size         = each.value.min_size
  max_size         = each.value.max_size
  desired_capacity = each.value.min_size

  launch_template {
    id      = aws_launch_template.app[each.key].id
    version = "$Latest"
  }

  tags = [
    {
      key                 = "Region"
      value               = each.key
      propagate_at_launch = true
    }
  ]
}
```

### 5.3 Tạo Route53 Records — Kết Hợp `for_each` với `for` Expression

```hcl
variable "dns_records" {
  type = map(object({
    type    = string
    ttl     = number
    records = list(string)
  }))

  default = {
    "api" = {
      type    = "A"
      ttl     = 300
      records = ["1.2.3.4"]
    }
    "www" = {
      type    = "CNAME"
      ttl     = 3600
      records = ["api.example.com"]
    }
    "mail" = {
      type    = "MX"
      ttl     = 3600
      records = ["10 mail.example.com"]
    }
  }
}

resource "aws_route53_record" "main" {
  for_each = var.dns_records

  zone_id = aws_route53_zone.main.zone_id
  name    = "${each.key}.${var.domain}"
  type    = each.value.type
  ttl     = each.value.ttl
  records = each.value.records
}

# Output tất cả record names
output "dns_record_names" {
  value = { for k, r in aws_route53_record.main : k => r.fqdn }
}
```

---

## 6. Kỹ Thuật Nâng Cao

### 6.1 `for_each` Với `for` Expression Để Lọc

```hcl
variable "instances" {
  type = map(object({
    type    = string
    enabled = bool
    env     = string
  }))
}

# Chỉ tạo instances được enabled VÀ thuộc production
resource "aws_instance" "prod" {
  for_each = {
    for name, config in var.instances :
    name => config
    if config.enabled && config.env == "production"
  }

  ami           = var.ami_id
  instance_type = each.value.type

  tags = { Name = each.key }
}
```

### 6.2 Flatten Complex Structure Trước Khi Dùng `for_each`

```hcl
# Dữ liệu có cấu trúc lồng nhau
variable "team_permissions" {
  type = map(list(string))  # map của team → list permissions
  default = {
    "backend-team"  = ["read", "write", "deploy"]
    "frontend-team" = ["read", "deploy"]
    "ops-team"      = ["read", "write", "deploy", "admin"]
  }
}

locals {
  # Flatten: tạo list của {team, permission} pairs
  all_permissions = flatten([
    for team, permissions in var.team_permissions : [
      for permission in permissions : {
        key        = "${team}:${permission}"
        team       = team
        permission = permission
      }
    ]
  ])

  # Chuyển sang map để dùng với for_each
  permission_map = {
    for item in local.all_permissions :
    item.key => item
  }
}

resource "aws_iam_role_policy_attachment" "team" {
  for_each = local.permission_map

  role       = aws_iam_role.teams[each.value.team].name
  policy_arn = aws_iam_policy.permissions[each.value.permission].arn
}
```

### 6.3 Module Level `for_each`

```hcl
variable "environments" {
  type = map(object({
    vpc_cidr      = string
    instance_type = string
    min_nodes     = number
  }))

  default = {
    "dev" = {
      vpc_cidr      = "10.1.0.0/16"
      instance_type = "t3.small"
      min_nodes     = 1
    }
    "staging" = {
      vpc_cidr      = "10.2.0.0/16"
      instance_type = "t3.medium"
      min_nodes     = 2
    }
    "production" = {
      vpc_cidr      = "10.0.0.0/16"
      instance_type = "t3.large"
      min_nodes     = 3
    }
  }
}

# Tạo module instance cho từng environment
module "eks_cluster" {
  for_each = var.environments
  source   = "./modules/eks"

  environment   = each.key
  vpc_cidr      = each.value.vpc_cidr
  instance_type = each.value.instance_type
  min_nodes     = each.value.min_nodes
}

# Truy cập output của module cụ thể
output "cluster_endpoints" {
  value = {
    for env, cluster in module.eks_cluster :
    env => cluster.endpoint
  }
}
```

---

## 7. Migrate Từ `count` Sang `for_each` An Toàn

Khi cần migrate mà không destroy/recreate tài nguyên:

```hcl
# TRƯỚC (count)
resource "aws_iam_user" "old" {
  count = 3
  name  = ["alice", "bob", "charlie"][count.index]
}

# SAU (for_each) — dùng moved block để Terraform biết đây là cùng resource
resource "aws_iam_user" "new" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.key
}

# moved blocks — Khai báo resource đã được đổi tên/cấu trúc
moved {
  from = aws_iam_user.old[0]
  to   = aws_iam_user.new["alice"]
}

moved {
  from = aws_iam_user.old[1]
  to   = aws_iam_user.new["bob"]
}

moved {
  from = aws_iam_user.old[2]
  to   = aws_iam_user.new["charlie"]
}
```

---

## 8. Bẫy Phổ Biến — Common Pitfalls

### 8.1 `for_each` Không Nhận List — Phải Dùng `toset()`

```hcl
# ❌ SAI — for_each không nhận list
resource "aws_subnet" "bad" {
  for_each = ["10.0.1.0/24", "10.0.2.0/24"]  # LỖI!
  cidr_block = each.value
}

# ✅ ĐÚNG — Chuyển thành set
resource "aws_subnet" "good" {
  for_each   = toset(["10.0.1.0/24", "10.0.2.0/24"])
  cidr_block = each.key  # Với set, each.key = each.value
}

# ✅ ĐÚNG — Hoặc dùng map
resource "aws_subnet" "better" {
  for_each = {
    "subnet-a" = "10.0.1.0/24"
    "subnet-b" = "10.0.2.0/24"
  }
  cidr_block = each.value
  tags = { Name = each.key }
}
```

### 8.2 `for_each` Value Phải Known At Plan Time

```hcl
# ❌ SAI — Dùng resource attribute chưa biết
resource "aws_subnet" "dynamic" {
  for_each = toset(aws_instance.servers[*].id)  # LỖI khi plan!
  # Error: The "for_each" value depends on resource attributes that
  # cannot be determined until apply
}

# ✅ ĐÚNG — Dùng data source hoặc variable đã biết trước
data "aws_instances" "servers" {
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
}

# Hoặc dùng variable
variable "server_ids" {
  type    = set(string)
  default = ["i-12345678", "i-87654321"]
}
```

### 8.3 Trùng Key Trong Map — Duplicate Keys

```hcl
# ❌ SAI — Trùng key sẽ gây lỗi
locals {
  # Nếu có hai subnet ở cùng AZ, key sẽ trùng
  subnet_map = {
    for subnet in var.subnets :
    subnet.az => subnet  # Có thể trùng key!
  }
}

# ✅ ĐÚNG — Dùng key độc nhất
locals {
  subnet_map = {
    for subnet in var.subnets :
    "${subnet.tier}-${subnet.az}" => subnet  # Kết hợp để đảm bảo độc nhất
  }
}
```

---

## 9. Quyết Định: Dùng `count` hay `for_each`?

```
Câu hỏi 1: Collection có thể thay đổi (thêm/xóa phần tử)?
  → Có → Dùng for_each (tránh index shifting)
  → Không → Có thể dùng count

Câu hỏi 2: Phần tử có tên/ID có nghĩa không?
  → Có → Dùng for_each với key có nghĩa
  → Không (chỉ số thứ tự) → count có thể phù hợp

Câu hỏi 3: Chỉ cần tạo 0 hoặc 1 resource (tùy điều kiện)?
  → count = condition ? 1 : 0 là idiom phổ biến và rõ ràng

Câu hỏi 4: Đang tạo nhiều instance của resource phức tạp (database, cluster)?
  → Luôn dùng for_each với key rõ ràng

Nguyên tắc chung: Nếu không chắc, dùng for_each.
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: Tại sao `for_each` được ưu tiên hơn `count` trong production?**

A: `count` dùng index số nguyên — nếu xóa phần tử ở giữa list, tất cả index phía sau bị dịch chuyển, gây Terraform destroy và recreate các resource không cần thiết. `for_each` dùng key string ổn định — xóa một phần tử chỉ ảnh hưởng đúng resource đó.

**Q: Khi nào `count` vẫn là lựa chọn tốt?**

A: Khi tạo điều kiện `count = var.enable ? 1 : 0` để tạo hoặc không tạo resource, và khi số lượng thực sự cố định và không bao giờ thay đổi thứ tự (ví dụ: luôn tạo đúng 3 subnet trong 3 AZ cố định không xóa bỏ bao giờ).

**Q: Làm sao migrate từ `count` sang `for_each` không mất data?**

A: Dùng `moved` block để khai báo resource đã được đổi address. Ví dụ: `moved { from = resource.old[0]; to = resource.new["alice"] }`. Terraform sẽ hiểu đây là cùng resource và không destroy/recreate.

---

## 🔗 Điều Hướng

| | |
|---|---|
| ← Bài trước | [2-meta-arguments.md](2-meta-arguments.md) |
| → Bài tiếp | [4-custom-providers.md](4-custom-providers.md) |
| ↑ Mục lục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Độ Khó:** ⭐⭐⭐ Nâng Cao
