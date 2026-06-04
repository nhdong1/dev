# CDK for Terraform — Cloud Development Kit — Viết Terraform Bằng Python/TypeScript

> **CDKTF** — CDK for Terraform — Cloud Development Kit cho Terraform — cho phép định nghĩa hạ tầng Terraform bằng ngôn ngữ lập trình quen thuộc như TypeScript, Python, Java, C#, hoặc Go, thay vì HCL. CDKTF tổng hợp — synthesize — mã nguồn thành JSON Terraform configuration và gọi lệnh Terraform để deploy.

---

## 1. CDKTF Là Gì và Tại Sao Dùng?

### Vòng Đời CDKTF

```
TypeScript/Python Code
        ↓ cdktf synth
JSON Terraform Config (.terraform/)
        ↓ terraform plan/apply
Cloud Infrastructure
```

### So Sánh HCL vs CDKTF

```hcl
# HCL — Terraform truyền thống
variable "bucket_name" {
  type = string
}

resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

```typescript
// CDKTF với TypeScript — tương đương
import { Construct } from "constructs";
import { App, TerraformStack, TerraformVariable } from "cdktf";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { S3Bucket } from "@cdktf/provider-aws/lib/s3-bucket";
import { S3BucketVersioningA } from "@cdktf/provider-aws/lib/s3-bucket-versioning";

class MyStack extends TerraformStack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new AwsProvider(this, "aws", { region: "us-east-1" });

    const bucketName = new TerraformVariable(this, "bucketName", {
      type: "string",
    });

    const bucket = new S3Bucket(this, "main", {
      bucket: bucketName.stringValue,
    });

    new S3BucketVersioningA(this, "versioning", {
      bucket: bucket.id,
      versioningConfiguration: {
        status: "Enabled",
      },
    });
  }
}

const app = new App();
new MyStack(app, "my-stack");
app.synth();
```

### Lợi Ích Của CDKTF

| Lợi Ích | Giải Thích |
|---------|-----------|
| **Ngôn ngữ quen thuộc** | Dev team dùng Python/TypeScript không cần học HCL |
| **IDE support đầy đủ** | Autocomplete, type checking, refactoring |
| **Logic lập trình** | Vòng lặp, điều kiện, inheritance — thừa kế |
| **Tái sử dụng qua package** | Chia sẻ qua npm, PyPI thay vì Terraform Registry |
| **Testing phong phú** | Jest, pytest, assertion framework quen thuộc |

### Khi Nào Nên Dùng CDKTF

- Team là developer (không phải DevOps/SRE) cần viết hạ tầng
- Cần logic phức tạp khó viết trong HCL
- Muốn tái sử dụng qua package manager quen thuộc
- Tổ chức đã chọn CDK làm tiêu chuẩn (AWS CDK + CDKTF)

---

## 2. Cài Đặt và Khởi Tạo Project

### Cài Đặt

```bash
# Cài Node.js (yêu cầu) và cdktf CLI
npm install --global cdktf-cli

# Kiểm tra cài đặt
cdktf --version
```

### Khởi Tạo Project TypeScript

```bash
# Tạo project mới với template TypeScript + AWS provider
mkdir my-terraform-project
cd my-terraform-project

cdktf init \
  --template="typescript" \
  --providers="aws@~>5.0" \
  --local  # Dùng local state (bỏ flag này để dùng Terraform Cloud)

# Cấu trúc project được tạo:
# .
# ├── cdktf.json          — Cấu hình CDKTF
# ├── main.ts             — Code hạ tầng chính
# ├── package.json        — Node.js dependencies
# ├── tsconfig.json       — TypeScript config
# └── __tests__/
#     └── main-test.ts    — Test file
```

### `cdktf.json` — Cấu Hình Project

```json
{
  "language": "typescript",
  "app": "npx ts-node main.ts",
  "projectId": "abc123",
  "sendCrashReports": "false",
  "terraformProviders": [
    "aws@~>5.0"
  ],
  "terraformModules": [],
  "codeMakerOutput": ".gen",
  "context": {}
}
```

---

## 3. Ví Dụ Thực Tế — TypeScript

### 3.1 Stack Hoàn Chỉnh: VPC + EC2 + Security Group

```typescript
import { Construct } from "constructs";
import { App, TerraformStack, TerraformOutput, Fn } from "cdktf";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { Vpc } from "@cdktf/provider-aws/lib/vpc";
import { Subnet } from "@cdktf/provider-aws/lib/subnet";
import { InternetGateway } from "@cdktf/provider-aws/lib/internet-gateway";
import { RouteTable } from "@cdktf/provider-aws/lib/route-table";
import { RouteTableAssociation } from "@cdktf/provider-aws/lib/route-table-association";
import { SecurityGroup } from "@cdktf/provider-aws/lib/security-group";
import { Instance } from "@cdktf/provider-aws/lib/instance";

// Config interface — kiểm tra type rõ ràng
interface WebStackConfig {
  region: string;
  vpcCidr: string;
  instanceType: string;
  amiId: string;
  environment: string;
}

class WebStack extends TerraformStack {
  public readonly instancePublicIp: string;

  constructor(scope: Construct, id: string, config: WebStackConfig) {
    super(scope, id);

    new AwsProvider(this, "aws", { region: config.region });

    // VPC
    const vpc = new Vpc(this, "vpc", {
      cidrBlock: config.vpcCidr,
      enableDnsHostnames: true,
      enableDnsSupport: true,
      tags: {
        Name: `${config.environment}-vpc`,
        Environment: config.environment,
      },
    });

    // Internet Gateway — Cổng kết nối Internet
    const igw = new InternetGateway(this, "igw", {
      vpcId: vpc.id,
      tags: { Name: `${config.environment}-igw` },
    });

    // Public Subnet
    const publicSubnet = new Subnet(this, "public-subnet", {
      vpcId: vpc.id,
      cidrBlock: Fn.cidrsubnet(config.vpcCidr, 8, 1),
      availabilityZone: `${config.region}a`,
      mapPublicIpOnLaunch: true,
      tags: { Name: `${config.environment}-public-subnet` },
    });

    // Route Table — Bảng định tuyến
    const routeTable = new RouteTable(this, "public-rt", {
      vpcId: vpc.id,
      route: [
        {
          cidrBlock: "0.0.0.0/0",
          gatewayId: igw.id,
        },
      ],
      tags: { Name: `${config.environment}-public-rt` },
    });

    new RouteTableAssociation(this, "public-rta", {
      subnetId: publicSubnet.id,
      routeTableId: routeTable.id,
    });

    // Security Group — Nhóm Bảo Mật
    const sg = new SecurityGroup(this, "web-sg", {
      name: `${config.environment}-web-sg`,
      vpcId: vpc.id,
      ingress: [
        { fromPort: 80,  toPort: 80,  protocol: "tcp", cidrBlocks: ["0.0.0.0/0"] },
        { fromPort: 443, toPort: 443, protocol: "tcp", cidrBlocks: ["0.0.0.0/0"] },
      ],
      egress: [
        { fromPort: 0, toPort: 0, protocol: "-1", cidrBlocks: ["0.0.0.0/0"] },
      ],
      tags: { Name: `${config.environment}-web-sg` },
    });

    // EC2 Instance
    const instance = new Instance(this, "web", {
      ami:          config.amiId,
      instanceType: config.instanceType,
      subnetId:     publicSubnet.id,
      vpcSecurityGroupIds: [sg.id],
      tags: {
        Name:        `${config.environment}-web`,
        Environment: config.environment,
      },
    });

    // Output
    new TerraformOutput(this, "instance_public_ip", {
      value:       instance.publicIp,
      description: "Public IP của web server",
    });

    this.instancePublicIp = instance.publicIp;
  }
}

// Main — tạo nhiều stack cho nhiều environment
const app = new App();

new WebStack(app, "dev", {
  region:       "us-east-1",
  vpcCidr:      "10.1.0.0/16",
  instanceType: "t3.micro",
  amiId:        "ami-12345678",
  environment:  "dev",
});

new WebStack(app, "production", {
  region:       "us-east-1",
  vpcCidr:      "10.0.0.0/16",
  instanceType: "t3.large",
  amiId:        "ami-12345678",
  environment:  "production",
});

app.synth();
```

### 3.2 Reusable Constructs — Cấu Trúc Tái Sử Dụng

```typescript
import { Construct } from "constructs";
import { S3Bucket } from "@cdktf/provider-aws/lib/s3-bucket";
import { S3BucketVersioningA } from "@cdktf/provider-aws/lib/s3-bucket-versioning";
import { S3BucketServerSideEncryptionConfigurationA } from "@cdktf/provider-aws/lib/s3-bucket-server-side-encryption-configuration";
import { S3BucketPublicAccessBlock } from "@cdktf/provider-aws/lib/s3-bucket-public-access-block";

interface SecureBucketConfig {
  bucketName: string;
  environment: string;
  enableVersioning?: boolean;
}

// Custom Construct — tái sử dụng như module Terraform
class SecureBucket extends Construct {
  public readonly bucket: S3Bucket;
  public readonly bucketId: string;
  public readonly bucketArn: string;

  constructor(scope: Construct, id: string, config: SecureBucketConfig) {
    super(scope, id);

    // Tạo S3 bucket
    this.bucket = new S3Bucket(this, "bucket", {
      bucket: config.bucketName,
      tags: { Environment: config.environment },
    });

    this.bucketId  = this.bucket.id;
    this.bucketArn = this.bucket.arn;

    // Block public access — Chặn truy cập công khai
    new S3BucketPublicAccessBlock(this, "public-access-block", {
      bucket:                this.bucket.id,
      blockPublicAcls:       true,
      blockPublicPolicy:     true,
      ignorePublicAcls:      true,
      restrictPublicBuckets: true,
    });

    // Server-side encryption — Mã hóa phía server
    new S3BucketServerSideEncryptionConfigurationA(this, "sse", {
      bucket: this.bucket.id,
      rule: [
        {
          applyServerSideEncryptionByDefault: {
            sseAlgorithm: "AES256",
          },
        },
      ],
    });

    // Versioning tùy chọn
    if (config.enableVersioning ?? true) {
      new S3BucketVersioningA(this, "versioning", {
        bucket: this.bucket.id,
        versioningConfiguration: { status: "Enabled" },
      });
    }
  }
}

// Dùng custom construct trong stack
class InfraStack extends TerraformStack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    // Tái sử dụng SecureBucket ở nhiều chỗ
    const logsBucket = new SecureBucket(this, "logs", {
      bucketName:      "my-app-logs-prod",
      environment:     "production",
      enableVersioning: false,
    });

    const artifactsBucket = new SecureBucket(this, "artifacts", {
      bucketName:  "my-app-artifacts-prod",
      environment: "production",
    });

    new TerraformOutput(this, "logs_bucket_arn", {
      value: logsBucket.bucketArn,
    });
  }
}
```

---

## 4. Ví Dụ Python

```python
#!/usr/bin/env python
from constructs import Construct
from cdktf import App, TerraformStack, TerraformOutput
from cdktf_cdktf_provider_aws.provider import AwsProvider
from cdktf_cdktf_provider_aws.vpc import Vpc
from cdktf_cdktf_provider_aws.s3_bucket import S3Bucket
from cdktf_cdktf_provider_aws.s3_bucket_versioning import (
    S3BucketVersioning,
    S3BucketVersioningVersioningConfiguration,
)

class PythonStack(TerraformStack):
    def __init__(self, scope: Construct, id: str, environment: str):
        super().__init__(scope, id)

        AwsProvider(self, "aws", region="us-east-1")

        # Python có thể dùng vòng lặp, điều kiện tự nhiên
        bucket_configs = [
            {"name": f"logs-{environment}",      "versioning": False},
            {"name": f"artifacts-{environment}", "versioning": True},
            {"name": f"backups-{environment}",   "versioning": True},
        ]

        buckets = {}
        for config in bucket_configs:
            bucket = S3Bucket(
                self, config["name"],
                bucket=config["name"],
                tags={"Environment": environment}
            )

            if config["versioning"]:
                S3BucketVersioning(
                    self, f"{config['name']}-versioning",
                    bucket=bucket.id,
                    versioning_configuration=S3BucketVersioningVersioningConfiguration(
                        status="Enabled"
                    )
                )

            buckets[config["name"]] = bucket

        TerraformOutput(
            self, "bucket_names",
            value=[b.id for b in buckets.values()]
        )


app = App()
PythonStack(app, "dev",  environment="dev")
PythonStack(app, "prod", environment="production")
app.synth()
```

---

## 5. Testing CDKTF

```typescript
import { Testing } from "cdktf";
import { WebStack } from "../lib/web-stack";
import { Instance } from "@cdktf/provider-aws/lib/instance";
import { SecurityGroup } from "@cdktf/provider-aws/lib/security-group";

describe("WebStack", () => {
  // Synthesis test — kiểm tra stack tổng hợp thành công
  it("synthesizes correctly", () => {
    const app   = Testing.app();
    const stack = new WebStack(app, "test", {
      region:       "us-east-1",
      vpcCidr:      "10.0.0.0/16",
      instanceType: "t3.micro",
      amiId:        "ami-12345678",
      environment:  "test",
    });

    expect(Testing.synth(stack)).toMatchSnapshot();
  });

  // Unit test — kiểm tra resource cụ thể được tạo
  it("creates EC2 instance", () => {
    const app   = Testing.app();
    const stack = new WebStack(app, "test", {
      region:       "us-east-1",
      vpcCidr:      "10.0.0.0/16",
      instanceType: "t3.micro",
      amiId:        "ami-12345678",
      environment:  "test",
    });

    expect(Testing.synth(stack)).toHaveResourceWithProperties(
      Instance,
      { instance_type: "t3.micro" }
    );
  });

  // Security test — kiểm tra security group không mở SSH công khai
  it("does not expose SSH publicly", () => {
    const app   = Testing.app();
    const stack = new WebStack(app, "test", {
      region:       "us-east-1",
      vpcCidr:      "10.0.0.0/16",
      instanceType: "t3.micro",
      amiId:        "ami-12345678",
      environment:  "test",
    });

    const synth = Testing.synth(stack);
    const sgs   = synth.resource?.aws_security_group;

    Object.values(sgs ?? {}).forEach((sg: any) => {
      const ingressRules = sg.ingress ?? [];
      const sshRule = ingressRules.find((r: any) => r.from_port === 22);
      if (sshRule) {
        expect(sshRule.cidr_blocks).not.toContain("0.0.0.0/0");
      }
    });
  });
});
```

---

## 6. Workflow CDKTF

```bash
# Cài dependencies và generate provider bindings
cdktf get

# Build TypeScript
npm run build

# Xem Terraform JSON được sinh ra
cdktf synth

# Xem plan cho một stack
cdktf diff my-stack

# Apply một stack
cdktf deploy my-stack

# Apply tất cả stack
cdktf deploy "*"

# Destroy
cdktf destroy my-stack

# Chạy tests
npm test
```

---

## 7. CDKTF vs Pulumi vs HCL — So Sánh

| | **CDKTF** | **Pulumi** | **HCL Terraform** |
|---|----------|-----------|------------------|
| **Ngôn ngữ** | TS/Python/Java/C#/Go | TS/Python/Go/.NET | HCL (domain-specific) |
| **Backend** | Terraform state | Pulumi state | Terraform state |
| **Provider** | Tất cả Terraform provider | Terraform bridge + native | Terraform provider |
| **Ecosystem** | Terraform Registry | Pulumi Registry | Terraform Registry |
| **Learning curve** | Thấp với developer quen TS/Python | Thấp | Thấp với DevOps |
| **HashiCorp support** | ✅ Chính thức | ❌ | ✅ Chính thức |
| **Production maturity** | Medium | High | Very High |

---

## 8. Câu Hỏi Phỏng Vấn

**Q: CDKTF tổng hợp — synthesize — ra gì và tại sao điều đó quan trọng?**

A: CDKTF synthesize mã TypeScript/Python thành Terraform JSON config. Điều quan trọng là Terraform JSON format hoàn toàn tương đương HCL — chỉ là biểu diễn khác. Nghĩa là mọi provider và tính năng Terraform đều hoạt động với CDKTF, không có giới hạn nào.

**Q: Lợi ích lớn nhất của CDKTF so với HCL?**

A: Khả năng dùng logic lập trình thực sự (vòng lặp, kế thừa class, type system), IDE support đầy đủ với autocomplete và type checking, và tái sử dụng qua package manager quen thuộc như npm hoặc PyPI.

**Q: Khi nào nên chọn CDKTF thay vì HCL?**

A: Khi team chủ yếu là developer không quen HCL, khi cần logic phức tạp khó diễn đạt trong HCL, hoặc khi muốn chia sẻ constructs qua package manager. Với team DevOps đã quen HCL, HCL vẫn là lựa chọn tốt hơn vì đơn giản và ít dependency hơn.

---

## 🔗 Điều Hướng

| | |
|---|---|
| ← Bài trước | [4-custom-providers.md](4-custom-providers.md) |
| → Bài tiếp | [6-opentofu.md](6-opentofu.md) |
| ↑ Mục lục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Độ Khó:** ⭐⭐⭐ Nâng Cao
