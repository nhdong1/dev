# Dependency Issues — Vấn Đề Phụ Thuộc Giữa Resources

> Dependency Cycle — Vòng phụ thuộc vòng tròn — là một trong những lỗi khó chẩn đoán nhất trong Terraform. Bài này hướng dẫn cách phát hiện, debug và giải quyết.

---

## Terraform Dependency Graph — Đồ Thị Phụ Thuộc

Terraform xây dựng DAG — Directed Acyclic Graph — Đồ Thị Có Hướng Không Vòng Lặp — để xác định thứ tự tạo/xóa resource.

```
Ví dụ dependency graph hợp lệ:
aws_vpc → aws_subnet → aws_instance
    ↓
aws_internet_gateway → aws_route_table

Ví dụ dependency graph KHÔNG hợp lệ (cycle):
resource_A → resource_B → resource_C → resource_A  ← VÒNG!
```

---

## Phát Hiện Dependency Cycle

### Thông Báo Lỗi Điển Hình

```
Error: Cycle: aws_security_group.web, aws_security_group.db
│
│ Terraform found a cycle in the dependency graph.
│ The following resources form a cycle:
│   aws_security_group.web (expand)
│   aws_security_group.db (expand)
```

### Visualize Dependency Graph — Hiển Thị Đồ Thị

```bash
# Xuất đồ thị phụ thuộc sang định dạng DOT
terraform graph > dependency.dot

# Render thành hình ảnh (cần cài Graphviz)
terraform graph | dot -Tsvg > dependency.svg

# Xem trực tiếp trên terminal (cần cài graphviz)
terraform graph | dot -Tpng | display

# Lọc chỉ xem resource cụ thể
terraform graph | grep -E "(security_group|instance)"
```

---

## Nguyên Nhân Phổ Biến và Cách Fix

### Nguyên Nhân 1: Security Group Tham Chiếu Lẫn Nhau

**Vấn đề:** Security Group A — Nhóm Bảo Mật A — cho phép traffic từ Security Group B, đồng thời B cho phép traffic từ A.

```hcl
# ❌ CÁI NÀY TẠO RA CYCLE
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.db.id]  # ← Tham chiếu db
  }
}

resource "aws_security_group" "db" {
  name = "db-sg"

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]  # ← Tham chiếu web → CYCLE!
  }
}
```

**Fix:** Tách security group rules — quy tắc bảo mật — ra khỏi security group definition — định nghĩa.

```hcl
# ✅ CÁCH FIX: Tách sg rules thành resource riêng

resource "aws_security_group" "web" {
  name = "web-sg"
  # Không có ingress rules ở đây
}

resource "aws_security_group" "db" {
  name = "db-sg"
  # Không có ingress rules ở đây
}

# Rule riêng: DB cho phép từ Web
resource "aws_security_group_rule" "db_allow_web" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.web.id
  security_group_id        = aws_security_group.db.id
}

# Rule riêng: Web cho phép từ DB (nếu cần)
resource "aws_security_group_rule" "web_allow_db" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.db.id
  security_group_id        = aws_security_group.web.id
}
```

---

### Nguyên Nhân 2: depends\_on Tạo Cycle

**Vấn đề:** Dùng `depends_on` không cẩn thận tạo ra chu trình phụ thuộc.

```hcl
# ❌ CYCLE qua depends_on
resource "aws_iam_role" "lambda_role" {
  name = "lambda-role"
  depends_on = [aws_lambda_function.processor]  # ← A phụ thuộc B
}

resource "aws_lambda_function" "processor" {
  role = aws_iam_role.lambda_role.arn  # ← B phụ thuộc A → CYCLE!
  depends_on = [aws_iam_role.lambda_role]  # Cái này redundant và gây rối
}
```

**Fix:** Loại bỏ `depends_on` khi implicit dependency — phụ thuộc ngầm — đã đủ.

```hcl
# ✅ Implicit dependency tự động
resource "aws_iam_role" "lambda_role" {
  name = "lambda-role"
  # Không cần depends_on — không liên quan đến lambda function
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_lambda_function" "processor" {
  role = aws_iam_role.lambda_role.arn  # Terraform tự hiểu: phải tạo role trước
  # Không cần depends_on vì đã dùng reference
}
```

---

### Nguyên Nhân 3: Module Output Tạo Cycle

**Vấn đề:** Module A dùng output của Module B và ngược lại.

```hcl
# ❌ CYCLE giữa modules
module "network" {
  source = "./modules/network"
  app_security_group_id = module.app.security_group_id  # ← Phụ thuộc app
}

module "app" {
  source = "./modules/app"
  vpc_id = module.network.vpc_id  # ← Phụ thuộc network → CYCLE!
}
```

**Fix:** Tái cấu trúc — refactor — để loại bỏ circular dependency.

```hcl
# ✅ Tạo module trung gian hoặc tách resource
module "network" {
  source = "./modules/network"
  # Chỉ tạo network resources, không phụ thuộc app
}

module "app" {
  source    = "./modules/app"
  vpc_id    = module.network.vpc_id     # App phụ thuộc Network → OK
  subnet_id = module.network.subnet_id
}

# Security group rules tạo sau, khi cả hai đã tồn tại
module "security_rules" {
  source              = "./modules/security-rules"
  app_sg_id           = module.app.security_group_id
  db_sg_id            = module.network.db_security_group_id
}
```

---

### Nguyên Nhân 4: Data Source Tạo Cycle Ngầm

**Vấn đề:** Data source — nguồn dữ liệu — tham chiếu đến resource sẽ được tạo trong cùng apply.

```hcl
# ❌ DATA SOURCE CÓ THỂ GÂY VẤN ĐỀ
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Data source này cần VPC tồn tại TRƯỚC apply
# Nhưng VPC được tạo TRONG apply này → Có thể fail hoặc trả về stale data
data "aws_vpc" "main" {
  id = aws_vpc.main.id  # Tham chiếu resource vừa tạo trong apply hiện tại
}

resource "aws_subnet" "public" {
  vpc_id     = data.aws_vpc.main.id  # Dùng data thay vì resource trực tiếp
  cidr_block = "10.0.1.0/24"
}
```

**Fix:** Dùng resource reference trực tiếp thay vì data source.

```hcl
# ✅ Dùng resource reference trực tiếp
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # Direct reference — không qua data source
  cidr_block = "10.0.1.0/24"
}
```

---

## Kỹ Thuật Debug Dependency Cycle

### Xem Dependency Graph Chi Tiết

```bash
# 1. Tạo graph đầy đủ
terraform graph -type=plan > full-graph.dot

# 2. Tạo graph chỉ cho apply
terraform graph -type=apply > apply-graph.dot

# 3. Lọc theo module cụ thể
terraform graph | grep "module.app"

# 4. Tìm các node có nhiều kết nối (có thể là điểm gây cycle)
terraform graph | grep -o '"[^"]*" ->' | sort | uniq -c | sort -rn | head -20
```

### Phân Tích Output Lỗi

```
Error: Cycle:
  module.app.aws_security_group.main (expand)
  module.network.aws_security_group_rule.app_ingress (expand)

↑ Đây chính xác là 2 resource tạo cycle
1. Mở file chứa module.app.aws_security_group.main
2. Mở file chứa module.network.aws_security_group_rule.app_ingress
3. Tìm chỗ chúng tham chiếu lẫn nhau
```

---

## Implicit vs Explicit Dependency — Phụ Thuộc Ngầm và Tường Minh

```hcl
# Implicit dependency — Phụ thuộc ngầm (qua reference)
resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id  # Terraform TỰ ĐỘNG hiểu: subnet phải tồn tại trước
}

# Explicit dependency — Phụ thuộc tường minh (dùng depends_on)
resource "aws_instance" "web" {
  subnet_id = var.subnet_id

  depends_on = [
    aws_internet_gateway.main,  # Cần gateway trước dù không reference trực tiếp
    aws_route_table_association.public,
  ]
}
```

**Nguyên tắc:** Dùng `depends_on` chỉ khi không thể dùng reference trực tiếp. Overuse — Dùng quá nhiều — `depends_on` làm chậm apply và có thể tạo cycle không cần thiết.

---

## Tránh Dependency Issues Từ Đầu

```hcl
# Pattern tốt: Resource theo tầng (layered approach)
# Tầng 1: Network infrastructure
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "public" { vpc_id = aws_vpc.main.id }

# Tầng 2: Security groups (dùng vpc, không reference compute)
resource "aws_security_group" "web" { vpc_id = aws_vpc.main.id }
resource "aws_security_group" "db"  { vpc_id = aws_vpc.main.id }

# Tầng 3: Security rules (sau khi cả hai sg đã định nghĩa)
resource "aws_security_group_rule" "db_from_web" {
  security_group_id        = aws_security_group.db.id
  source_security_group_id = aws_security_group.web.id
  ...
}

# Tầng 4: Compute (dùng sg từ tầng 2)
resource "aws_instance" "web" {
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public.id
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: Dependency cycle trong Terraform là gì? Làm sao giải quyết?**

A: Cycle xảy ra khi resource A phụ thuộc B và B phụ thuộc ngược lại A, tạo vòng lặp không thể giải quyết thứ tự tạo. Terraform báo lỗi "Cycle:" khi phát hiện.

Giải quyết: 
1. Dùng `terraform graph` để visualize dependency
2. Tìm tham chiếu tạo vòng
3. Tách resource (ví dụ: tách security group rules khỏi security group definition)
4. Refactor module structure để loại bỏ circular dependency

---

## Tóm Tắt

```
Dependency Cycle Causes:
├── Security group tham chiếu lẫn nhau
│   → Fix: Dùng aws_security_group_rule riêng
├── depends_on không cần thiết
│   → Fix: Dùng implicit reference thay thế
├── Module output tham chiếu vòng
│   → Fix: Refactor sang layered module design
└── Data source cho resource mới tạo
    → Fix: Dùng resource reference trực tiếp

Debug tools:
├── terraform graph | dot -Tsvg > graph.svg
└── Đọc error message: "Cycle: resource_A, resource_B"
```

---

**Xem Thêm:**
- [`5-debug-mode.md`](./5-debug-mode.md) — Debug chi tiết với TF\_LOG
- [`1-state-corruption.md`](./1-state-corruption.md) — Xử lý state corruption
