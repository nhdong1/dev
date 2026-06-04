# Custom Provider — Provider Tùy Chỉnh trong Terraform

> **Custom Provider** — Provider Tùy Chỉnh — là khi bạn tự viết một Terraform provider bằng ngôn ngữ Go để quản lý tài nguyên của hệ thống nội bộ, dịch vụ riêng, hoặc API chưa được HashiCorp hay cộng đồng hỗ trợ. Đây là kiến thức nâng cao dành cho Platform Engineer và người muốn hiểu sâu về Terraform internals.

---

## 1. Khi Nào Cần Custom Provider?

### Lý Do Viết Custom Provider

| Tình Huống | Ví Dụ |
|-----------|-------|
| Internal API — API nội bộ chưa có provider | Hệ thống ticket nội bộ, CMDB, inventory system |
| Provider hiện tại thiếu resource | Tính năng mới của cloud chưa được provider cập nhật |
| SaaS — Software as a Service — chưa có community provider | Dịch vụ của bên thứ ba cần quản lý bằng IaC |
| Testing và học tập | Hiểu Terraform hoạt động từ bên trong |

### Khi Không Nên Viết Custom Provider

- Đã có provider từ HashiCorp hoặc Terraform Registry
- Tài nguyên đơn giản có thể quản lý bằng `null_resource` + provisioner
- Có thể dùng `external` data source hoặc HTTP provider thay thế

---

## 2. Kiến Trúc Terraform Provider

### Plugin Architecture — Kiến Trúc Plugin

```
┌─────────────────────────────────────────────┐
│                Terraform CLI                │
│  (terraform init / plan / apply / destroy)  │
└──────────────────┬──────────────────────────┘
                   │ gRPC protocol
                   ↓
┌─────────────────────────────────────────────┐
│            Provider Plugin (Go binary)      │
│  ┌─────────────────────────────────────┐   │
│  │  Provider Schema — Lược đồ Provider  │   │
│  │  Resource CRUD operations           │   │
│  │  Data Source Read operations        │   │
│  │  Authentication logic               │   │
│  └─────────────────────────────────────┘   │
└──────────────────┬──────────────────────────┘
                   │ HTTP/SDK calls
                   ↓
┌─────────────────────────────────────────────┐
│           Target API / Service              │
│    (Cloud API, internal service, etc.)      │
└─────────────────────────────────────────────┘
```

### Terraform Plugin SDK vs Plugin Framework

| | Plugin SDK v2 | Plugin Framework (mới hơn) |
|---|--------------|--------------------------|
| **Thư viện** | `github.com/hashicorp/terraform-plugin-sdk/v2` | `github.com/hashicorp/terraform-plugin-framework` |
| **Cú pháp** | Hàm truyền thống | Struct-based, type-safe |
| **HashiCorp khuyến nghị** | Legacy (vẫn hỗ trợ) | ✅ Nên dùng cho project mới |
| **Phức tạp** | Thấp hơn | Cao hơn nhưng tốt hơn |

---

## 3. Cấu Trúc Project Custom Provider

```
terraform-provider-example/
├── main.go                          # Entry point — điểm khởi đầu
├── go.mod                           # Go module definition
├── go.sum                           # Dependency checksums
├── internal/
│   └── provider/
│       ├── provider.go              # Provider definition — định nghĩa provider
│       ├── resource_coffee.go       # Resource: coffee (ví dụ)
│       ├── resource_coffee_test.go  # Unit tests
│       ├── data_source_coffees.go   # Data source: coffees
│       └── data_source_coffees_test.go
├── examples/                        # Ví dụ dùng provider
│   ├── main.tf
│   └── provider.tf
├── docs/                            # Tài liệu tự động sinh
│   └── resources/
│       └── coffee.md
├── GNUmakefile                      # Build automation
└── .goreleaser.yml                  # Release configuration
```

---

## 4. Viết Provider Cơ Bản — Plugin Framework

### 4.1 `main.go` — Entry Point

```go
package main

import (
    "context"
    "flag"
    "log"

    "github.com/hashicorp/terraform-plugin-framework/providerserver"
    "terraform-provider-example/internal/provider"
)

// Version được inject khi build
var version string = "dev"

func main() {
    var debug bool

    flag.BoolVar(&debug, "debug", false, "Set to true to run provider with support for debuggers")
    flag.Parse()

    opts := providerserver.ServeOpts{
        Address: "registry.terraform.io/example-corp/example",
        Debug:   debug,
    }

    err := providerserver.Serve(context.Background(), provider.New(version), opts)
    if err != nil {
        log.Fatal(err.Error())
    }
}
```

### 4.2 `internal/provider/provider.go` — Provider Definition

```go
package provider

import (
    "context"
    "net/http"

    "github.com/hashicorp/terraform-plugin-framework/datasource"
    "github.com/hashicorp/terraform-plugin-framework/provider"
    "github.com/hashicorp/terraform-plugin-framework/provider/schema"
    "github.com/hashicorp/terraform-plugin-framework/resource"
    "github.com/hashicorp/terraform-plugin-framework/types"
)

// Đảm bảo ExampleProvider implement interface provider.Provider
var _ provider.Provider = &ExampleProvider{}

// ExampleProvider là implementation của provider
type ExampleProvider struct {
    version string
}

// ExampleProviderModel là cấu hình của provider trong HCL
type ExampleProviderModel struct {
    Endpoint types.String `tfsdk:"endpoint"`
    APIKey   types.String `tfsdk:"api_key"`
}

func New(version string) func() provider.Provider {
    return func() provider.Provider {
        return &ExampleProvider{
            version: version,
        }
    }
}

// Metadata trả về thông tin về provider
func (p *ExampleProvider) Metadata(ctx context.Context, req provider.MetadataRequest, resp *provider.MetadataResponse) {
    resp.TypeName = "example"
    resp.Version = p.version
}

// Schema định nghĩa cấu hình provider trong HCL
func (p *ExampleProvider) Schema(ctx context.Context, req provider.SchemaRequest, resp *provider.SchemaResponse) {
    resp.Schema = schema.Schema{
        Description: "Provider cho Example API",
        Attributes: map[string]schema.Attribute{
            "endpoint": schema.StringAttribute{
                Description: "URL của Example API",
                Optional:    true,
            },
            "api_key": schema.StringAttribute{
                Description: "API key để xác thực",
                Optional:    true,
                Sensitive:   true,  // Sẽ bị ẩn trong output
            },
        },
    }
}

// Configure khởi tạo HTTP client với config từ HCL
func (p *ExampleProvider) Configure(ctx context.Context, req provider.ConfigureRequest, resp *provider.ConfigureResponse) {
    var config ExampleProviderModel

    resp.Diagnostics.Append(req.Config.Get(ctx, &config)...)
    if resp.Diagnostics.HasError() {
        return
    }

    endpoint := "https://api.example.com"
    if !config.Endpoint.IsNull() {
        endpoint = config.Endpoint.ValueString()
    }

    apiKey := ""
    if !config.APIKey.IsNull() {
        apiKey = config.APIKey.ValueString()
    }

    // Tạo HTTP client để dùng trong resource/data source
    client := &ExampleClient{
        BaseURL: endpoint,
        APIKey:  apiKey,
        HTTP:    &http.Client{},
    }

    // Chia sẻ client với tất cả resource và data source
    resp.DataSourceData = client
    resp.ResourceData = client
}

// Resources liệt kê tất cả resource được hỗ trợ
func (p *ExampleProvider) Resources(ctx context.Context) []func() resource.Resource {
    return []func() resource.Resource{
        NewCoffeeResource,
    }
}

// DataSources liệt kê tất cả data source được hỗ trợ
func (p *ExampleProvider) DataSources(ctx context.Context) []func() datasource.DataSource {
    return []func() datasource.DataSource{
        NewCoffeesDataSource,
    }
}
```

### 4.3 `internal/provider/resource_coffee.go` — Resource Implementation

```go
package provider

import (
    "context"
    "fmt"

    "github.com/hashicorp/terraform-plugin-framework/resource"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema/planmodifier"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema/stringplanmodifier"
    "github.com/hashicorp/terraform-plugin-framework/types"
)

// Đảm bảo CoffeeResource implement resource.Resource interface
var _ resource.Resource = &CoffeeResource{}

// CoffeeResource là implementation của resource "example_coffee"
type CoffeeResource struct {
    client *ExampleClient
}

// CoffeeResourceModel map với HCL block trong Terraform config
type CoffeeResourceModel struct {
    ID          types.String `tfsdk:"id"`
    Name        types.String `tfsdk:"name"`
    Description types.String `tfsdk:"description"`
    Price       types.Float64 `tfsdk:"price"`
}

func NewCoffeeResource() resource.Resource {
    return &CoffeeResource{}
}

// Metadata định nghĩa tên resource trong HCL: "example_coffee"
func (r *CoffeeResource) Metadata(ctx context.Context, req resource.MetadataRequest, resp *resource.MetadataResponse) {
    resp.TypeName = req.ProviderTypeName + "_coffee"
}

// Schema định nghĩa các attribute của resource trong HCL
func (r *CoffeeResource) Schema(ctx context.Context, req resource.SchemaRequest, resp *resource.SchemaResponse) {
    resp.Schema = schema.Schema{
        Description: "Quản lý một loại cà phê trong hệ thống",
        Attributes: map[string]schema.Attribute{
            "id": schema.StringAttribute{
                Description: "ID độc nhất của loại cà phê",
                Computed:    true,
                PlanModifiers: []planmodifier.String{
                    stringplanmodifier.UseStateForUnknown(),
                },
            },
            "name": schema.StringAttribute{
                Description: "Tên loại cà phê",
                Required:    true,
            },
            "description": schema.StringAttribute{
                Description: "Mô tả loại cà phê",
                Optional:    true,
                Computed:    true,
            },
            "price": schema.Float64Attribute{
                Description: "Giá (USD)",
                Required:    true,
            },
        },
    }
}

// Configure nhận client từ provider
func (r *CoffeeResource) Configure(ctx context.Context, req resource.ConfigureRequest, resp *resource.ConfigureResponse) {
    if req.ProviderData == nil {
        return
    }

    client, ok := req.ProviderData.(*ExampleClient)
    if !ok {
        resp.Diagnostics.AddError(
            "Unexpected Resource Configure Type",
            fmt.Sprintf("Expected *ExampleClient, got: %T", req.ProviderData),
        )
        return
    }

    r.client = client
}

// Create — Tạo resource mới tương ứng với terraform apply (tạo mới)
func (r *CoffeeResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) {
    var plan CoffeeResourceModel

    // Đọc config từ plan
    resp.Diagnostics.Append(req.Plan.Get(ctx, &plan)...)
    if resp.Diagnostics.HasError() {
        return
    }

    // Gọi API để tạo resource
    coffee, err := r.client.CreateCoffee(ctx, CreateCoffeeRequest{
        Name:        plan.Name.ValueString(),
        Description: plan.Description.ValueString(),
        Price:       plan.Price.ValueFloat64(),
    })
    if err != nil {
        resp.Diagnostics.AddError("Error Creating Coffee", err.Error())
        return
    }

    // Map response về model để lưu vào state
    plan.ID = types.StringValue(coffee.ID)
    plan.Description = types.StringValue(coffee.Description)

    // Lưu state
    resp.Diagnostics.Append(resp.State.Set(ctx, plan)...)
}

// Read — Đọc state hiện tại của resource (dùng cho terraform refresh và plan)
func (r *CoffeeResource) Read(ctx context.Context, req resource.ReadRequest, resp *resource.ReadResponse) {
    var state CoffeeResourceModel

    resp.Diagnostics.Append(req.State.Get(ctx, &state)...)
    if resp.Diagnostics.HasError() {
        return
    }

    // Gọi API để lấy state thực tế
    coffee, err := r.client.GetCoffee(ctx, state.ID.ValueString())
    if err != nil {
        // Nếu resource không tồn tại, xóa khỏi state
        resp.State.RemoveResource(ctx)
        return
    }

    // Cập nhật state với data từ API
    state.Name = types.StringValue(coffee.Name)
    state.Description = types.StringValue(coffee.Description)
    state.Price = types.Float64Value(coffee.Price)

    resp.Diagnostics.Append(resp.State.Set(ctx, state)...)
}

// Update — Cập nhật resource khi config thay đổi
func (r *CoffeeResource) Update(ctx context.Context, req resource.UpdateRequest, resp *resource.UpdateResponse) {
    var plan CoffeeResourceModel
    var state CoffeeResourceModel

    resp.Diagnostics.Append(req.Plan.Get(ctx, &plan)...)
    resp.Diagnostics.Append(req.State.Get(ctx, &state)...)
    if resp.Diagnostics.HasError() {
        return
    }

    // Gọi API để update
    coffee, err := r.client.UpdateCoffee(ctx, state.ID.ValueString(), UpdateCoffeeRequest{
        Name:        plan.Name.ValueString(),
        Description: plan.Description.ValueString(),
        Price:       plan.Price.ValueFloat64(),
    })
    if err != nil {
        resp.Diagnostics.AddError("Error Updating Coffee", err.Error())
        return
    }

    plan.ID = state.ID
    plan.Description = types.StringValue(coffee.Description)

    resp.Diagnostics.Append(resp.State.Set(ctx, plan)...)
}

// Delete — Xóa resource tương ứng với terraform destroy
func (r *CoffeeResource) Delete(ctx context.Context, req resource.DeleteRequest, resp *resource.DeleteResponse) {
    var state CoffeeResourceModel

    resp.Diagnostics.Append(req.State.Get(ctx, &state)...)
    if resp.Diagnostics.HasError() {
        return
    }

    // Gọi API để xóa
    err := r.client.DeleteCoffee(ctx, state.ID.ValueString())
    if err != nil {
        resp.Diagnostics.AddError("Error Deleting Coffee", err.Error())
        return
    }
    // State tự động bị xóa sau khi Delete trả về không có error
}
```

---

## 5. Testing Custom Provider

### Acceptance Test — Kiểm Thử Chấp Nhận

```go
package provider_test

import (
    "testing"
    "github.com/hashicorp/terraform-plugin-testing/helper/resource"
)

func TestAccCoffeeResource(t *testing.T) {
    resource.Test(t, resource.TestCase{
        ProtoV6ProviderFactories: testAccProtoV6ProviderFactories,
        Steps: []resource.TestStep{
            // Create và Read
            {
                Config: testAccCoffeeResourceConfig("Cà Phê Sữa Đá", 2.5),
                Check: resource.ComposeAggregateTestCheckFunc(
                    resource.TestCheckResourceAttr("example_coffee.test", "name", "Cà Phê Sữa Đá"),
                    resource.TestCheckResourceAttr("example_coffee.test", "price", "2.5"),
                    resource.TestCheckResourceAttrSet("example_coffee.test", "id"),
                ),
            },
            // ImportState — kiểm tra import hoạt động
            {
                ResourceName:      "example_coffee.test",
                ImportState:       true,
                ImportStateVerify: true,
            },
            // Update
            {
                Config: testAccCoffeeResourceConfig("Cà Phê Đen", 1.5),
                Check: resource.ComposeAggregateTestCheckFunc(
                    resource.TestCheckResourceAttr("example_coffee.test", "name", "Cà Phê Đen"),
                    resource.TestCheckResourceAttr("example_coffee.test", "price", "1.5"),
                ),
            },
        },
    })
}

func testAccCoffeeResourceConfig(name string, price float64) string {
    return fmt.Sprintf(`
resource "example_coffee" "test" {
  name  = %q
  price = %g
}
`, name, price)
}
```

---

## 6. Publish Provider Lên Terraform Registry

### Quy Trình Publish

```
1. Tạo repository GitHub: terraform-provider-<name>
   (tên phải theo format này)

2. Tag version với format: v0.1.0
   git tag v0.1.0
   git push origin v0.1.0

3. Dùng GoReleaser để build cho nhiều platform
   goreleaser release --clean

4. Ký binary bằng GPG key
   (Terraform Registry yêu cầu signature)

5. Đăng ký namespace trên registry.terraform.io
   - Đăng nhập bằng GitHub account
   - Publish provider từ GitHub repository
```

### `.goreleaser.yml` Cấu Hình Build

```yaml
# GoReleaser — công cụ tự động release Go binary
before:
  hooks:
    - go mod tidy

builds:
  - env:
      - CGO_ENABLED=0
    mod_timestamp: '{{ .CommitTimestamp }}'
    flags:
      - -trimpath
    ldflags:
      - '-s -w -X main.version={{.Version}}'
    goos:
      - freebsd
      - windows
      - linux
      - darwin
    goarch:
      - amd64
      - '386'
      - arm
      - arm64
    ignore:
      - goos: darwin
        goarch: '386'

archives:
  - format: zip
    name_template: '{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}'

checksum:
  name_template: '{{ .ProjectName }}_{{ .Version }}_SHA256SUMS'
  algorithm: sha256

signs:
  - artifacts: checksum
    args:
      - "--batch"
      - "--local-user"
      - "{{ .Env.GPG_FINGERPRINT }}"
      - "--output"
      - "${signature}"
      - "--detach-sign"
      - "${artifact}"

release:
  draft: true
```

---

## 7. Dùng Local Provider Trong Development

```hcl
# terraform.rc hoặc .terraformrc
provider_installation {
  dev_overrides {
    # Override để dùng provider build local thay vì từ registry
    "registry.terraform.io/example-corp/example" = "/home/user/go/bin"
  }
  direct {}
}
```

```bash
# Build provider local
cd terraform-provider-example
go build -o terraform-provider-example
mv terraform-provider-example ~/.terraform.d/plugins/registry.terraform.io/example-corp/example/0.1.0/linux_amd64/

# Dùng trong Terraform config
# terraform {
#   required_providers {
#     example = {
#       source  = "example-corp/example"
#       version = "~> 0.1"
#     }
#   }
# }
```

---

## 8. Alternatives Trước Khi Viết Custom Provider

Trước khi tốn công viết custom provider, hãy thử các cách này:

### 8.1 `http` Provider — Gọi API Đơn Giản

```hcl
terraform {
  required_providers {
    http = {
      source = "hashicorp/http"
    }
  }
}

# Đọc data từ HTTP API
data "http" "user_info" {
  url = "https://api.example.com/users/${var.user_id}"

  request_headers = {
    Authorization = "Bearer ${var.api_token}"
    Accept        = "application/json"
  }
}

output "user_name" {
  value = jsondecode(data.http.user_info.response_body).name
}
```

### 8.2 `restapi` Community Provider

```hcl
provider "restapi" {
  uri                  = "https://api.example.com"
  write_returns_object = true

  headers = {
    "Authorization" = "Bearer ${var.api_key}"
    "Content-Type"  = "application/json"
  }
}

resource "restapi_object" "coffee" {
  path         = "/coffees"
  data         = jsonencode({
    name  = "Cà Phê Sữa Đá"
    price = 2.5
  })
}
```

### 8.3 `null_resource` + `local-exec` — Last Resort

```hcl
resource "null_resource" "register_service" {
  triggers = {
    service_name = var.service_name
    service_port = var.service_port
  }

  provisioner "local-exec" {
    command = <<-EOT
      curl -X POST https://api.example.com/services \
        -H "Authorization: Bearer ${var.api_key}" \
        -H "Content-Type: application/json" \
        -d '{"name": "${var.service_name}", "port": ${var.service_port}}'
    EOT
  }
}
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Custom Provider trong Terraform là gì và khi nào cần viết?**

A: Custom Provider là Go binary implement Terraform Plugin Protocol, dùng gRPC để giao tiếp với Terraform CLI. Cần viết khi muốn quản lý tài nguyên của API nội bộ hoặc dịch vụ chưa có provider. Trước khi viết, cần thử `http` provider, `restapi` provider, hoặc `null_resource` + `local-exec` trước vì viết custom provider tốn nhiều công sức.

**Q: Provider hoạt động như thế nào với Terraform CLI?**

A: Provider là binary Go riêng biệt. Khi `terraform init`, CLI download binary về `.terraform/providers/`. Khi chạy plan/apply, CLI khởi động provider binary như một subprocess và giao tiếp qua gRPC. Provider implement 4 CRUD operations cho mỗi resource: Create, Read, Update, Delete.

**Q: Khác nhau giữa Plugin SDK v2 và Plugin Framework?**

A: Plugin Framework mới hơn, type-safe hơn, được HashiCorp khuyến nghị cho project mới. Plugin SDK v2 vẫn được hỗ trợ nhưng là legacy. Framework dùng struct-based approach với kiểm tra type tại compile time, trong khi SDK v2 dùng `schema.ResourceData` với runtime type assertion.

---

## 🔗 Điều Hướng

| | |
|---|---|
| ← Bài trước | [3-for-each-count.md](3-for-each-count.md) |
| → Bài tiếp | [5-terraform-cdk.md](5-terraform-cdk.md) |
| ↑ Mục lục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Độ Khó:** ⭐⭐⭐⭐ Chuyên Sâu
