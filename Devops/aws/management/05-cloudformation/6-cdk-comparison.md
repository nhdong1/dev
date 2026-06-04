# CDK vs CloudFormation — Khi Nào Dùng Công Cụ Nào?

> **AWS CDK — Cloud Development Kit** (Bộ Phát Triển Đám Mây) cho phép định nghĩa hạ tầng AWS bằng ngôn ngữ lập trình phổ thông (Python, TypeScript, Java, Go, C#) thay vì viết YAML/JSON trực tiếp. CDK không thay thế CloudFormation — nó là layer trên CloudFormation, compile code thành CloudFormation template.

---

## 📚 Mục Lục

1. [CDK Là Gì?](#cdk-là-gì)
2. [Kiến Trúc CDK](#kiến-trúc-cdk)
3. [So Sánh Chi Tiết: CloudFormation vs CDK vs Terraform](#so-sánh-chi-tiết)
4. [CDK Core Concepts](#cdk-core-concepts)
5. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
6. [Khi Nào Dùng CDK?](#khi-nào-dùng-cdk)
7. [Khi Nào Dùng CloudFormation Thuần?](#khi-nào-dùng-cloudformation-thuần)
8. [CDK Ecosystem](#cdk-ecosystem)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CDK Là Gì?

### Vấn Đề CDK Giải Quyết

```
CloudFormation YAML:
- Verbose (dài dòng): Một VPC đơn giản = 200+ dòng YAML
- Không có abstraction: Lặp đi lặp lại patterns giống nhau
- Khó tái sử dụng: Copy-paste template, khó maintain
- Không có type safety: Typo trong property name → chỉ biết khi deploy
- Không có IDE support tốt: Không autocomplete, không refactoring

CDK giải quyết:
- Viết IaC bằng Python/TypeScript quen thuộc
- Constructs tái sử dụng (như thư viện)
- Type safety + IDE autocomplete + refactoring
- Abstractions cao hơn: ApplicationLoadBalancedFargateService = 1 dòng
```

### CDK Không Phải Là Gì

```
❌ CDK không phải ngôn ngữ mới để học
❌ CDK không bypass CloudFormation — nó vẫn tạo CFN template
❌ CDK không thay thế cần hiểu CloudFormation concepts
❌ CDK không cần server/runtime — chỉ là công cụ local
```

---

## Kiến Trúc CDK

### Luồng Compile và Deploy

```
Bạn viết CDK code (Python/TypeScript)
         │
         │ cdk synth
         ▼
CDK Framework
├── Resolve all constructs
├── Apply L2/L3 defaults
└── Generate CloudFormation template (JSON)
         │
         │ cdk deploy
         ▼
CloudFormation Service
├── Create/Update stack
├── Deploy resources
└── Quản lý state như stack thông thường
         │
         ▼
AWS Resources (EC2, VPC, Lambda, RDS...)
```

### CDK Toolkit Commands

```bash
# Cài đặt
npm install -g aws-cdk

# Khởi tạo project mới
cdk init app --language python    # python | typescript | java | go | csharp

# Xem CloudFormation template được tạo ra (không deploy)
cdk synth

# Xem diff giữa code và deployed stack
cdk diff

# Deploy
cdk deploy

# Deploy với tự động approve (CI/CD)
cdk deploy --require-approval never

# Xóa stack
cdk destroy

# Bootstrap CDK trong account/region (chỉ cần 1 lần)
cdk bootstrap aws://ACCOUNT_ID/REGION
```

---

## So Sánh Chi Tiết

### CloudFormation vs CDK vs Terraform

| Tiêu Chí | CloudFormation | AWS CDK | Terraform |
|----------|---------------|---------|-----------|
| **Ngôn ngữ** | YAML / JSON | Python, TypeScript, Java, Go, C# | HCL (HashiCorp Config Language) |
| **Abstraction level** | Thấp (1:1 API) | Cao (Constructs L1→L3) | Trung bình |
| **Native AWS** | ✅ 100% | ✅ Compile ra CFN | ❌ Provider riêng |
| **State management** | AWS (tự động) | AWS via CFN (tự động) | File state (tự quản lý) |
| **Multi-cloud** | ❌ AWS only | ❌ Chủ yếu AWS | ✅ Azure, GCP, K8s... |
| **Type safety** | ❌ | ✅ (TypeScript đặc biệt tốt) | Hạn chế |
| **IDE support** | Hạn chế | ✅ Autocomplete, refactoring | Plugin available |
| **Testing** | Khó | ✅ Unit tests với Jest/pytest | Hạn chế |
| **Reusability** | Template fragments | ✅ Construct Library | Modules |
| **Learning curve** | Trung bình | Cao (cần lập trình) | Trung bình |
| **Ecosystem** | AWS official | AWS official + CDK Patterns | Cộng đồng rất lớn |
| **Rollback** | Tự động | Tự động (via CFN) | Manual |
| **Drift detection** | ✅ Built-in | ✅ Via CFN | `terraform plan` |
| **Phù hợp nhất khi** | AWS thuần, không muốn lập trình | Dev team mạnh, reuse nhiều | Multi-cloud, team đã quen |

### Cùng Tài Nguyên — So Sánh Code

**Tạo S3 Bucket với versioning và encryption:**

```yaml
# CloudFormation YAML (~20 dòng)
Resources:
  DataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "data-${Environment}-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref DataKey
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  DataKey:
    Type: AWS::KMS::Key
    Properties:
      Description: "Key for data bucket"
      EnableKeyRotation: true
```

```python
# CDK Python (~10 dòng)
from aws_cdk import aws_s3 as s3, aws_kms as kms

key = kms.Key(self, "DataKey", enable_key_rotation=True)

bucket = s3.Bucket(
    self, "DataBucket",
    bucket_name=f"data-{env_name}-{self.account}",
    versioned=True,
    encryption=s3.BucketEncryption.KMS,
    encryption_key=key,
    block_public_access=s3.BlockPublicAccess.BLOCK_ALL,  # Tất cả trong 1 dòng
)
```

**Tạo ALB + ECS Fargate Service:**

```yaml
# CloudFormation: ~150-200 dòng YAML
# (ALB, Listener, Target Group, ECS Cluster, Task Definition,
#  Service, Security Groups, IAM Roles, Log Groups...)
```

```python
# CDK: ~10 dòng
from aws_cdk import aws_ecs_patterns as ecs_patterns

service = ecs_patterns.ApplicationLoadBalancedFargateService(
    self, "WebService",
    cluster=cluster,
    cpu=256,
    memory_limit_mib=512,
    desired_count=2,
    task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
        image=ecs.ContainerImage.from_registry("nginx:latest"),
        container_port=80,
    ),
    public_load_balancer=True,
)
```

---

## CDK Core Concepts

### Constructs — Khối Xây Dựng Cơ Bản

**Construct** là building block trong CDK. Có 3 tầng (levels):

```
L1 Constructs (Cfn Prefix) — 1:1 với CloudFormation
→ CfnBucket, CfnInstance, CfnRole
→ Mọi property của CFN resource đều có
→ Dùng khi cần control tuyệt đối hoặc tính năng CFN mới nhất

L2 Constructs — Abstraction có opinions
→ Bucket, Instance, Role (không có Cfn prefix)
→ Thêm defaults hợp lý: encryption mặc định, security best practices
→ Helper methods: bucket.grant_read(), role.add_to_policy()
→ Đây là layer được dùng nhiều nhất

L3 Constructs (Patterns) — High-level patterns
→ ApplicationLoadBalancedFargateService, QueueProcessingFargateService
→ Tạo nhiều resources liên quan trong 1 construct
→ Dùng khi muốn tốc độ, không cần fine-grained control
```

```python
import aws_cdk as cdk
from aws_cdk import (
    aws_s3 as s3,
    aws_lambda as lambda_,
    aws_iam as iam,
)

class MyStack(cdk.Stack):
    def __init__(self, scope: cdk.App, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # L1 — CloudFormation resource trực tiếp
        cfn_bucket = s3.CfnBucket(
            self, "L1Bucket",
            versioning_configuration=s3.CfnBucket.VersioningConfigurationProperty(
                status="Enabled"
            )
        )

        # L2 — Abstraction với defaults tốt
        bucket = s3.Bucket(
            self, "L2Bucket",
            versioned=True,
            encryption=s3.BucketEncryption.S3_MANAGED,
            removal_policy=cdk.RemovalPolicy.RETAIN
        )

        # L2 với helper methods
        fn = lambda_.Function(
            self, "MyFunction",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="index.handler",
            code=lambda_.Code.from_asset("lambda/"),
        )
        bucket.grant_read(fn)        # Tự động tạo IAM policy phù hợp
```

### Stacks và Apps

```python
import aws_cdk as cdk

class NetworkStack(cdk.Stack):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)
        self.vpc = ec2.Vpc(self, "VPC", max_azs=3)
        # Export cho stack khác
        cdk.CfnOutput(self, "VpcId", value=self.vpc.vpc_id)

class AppStack(cdk.Stack):
    def __init__(self, scope, id, network_stack: NetworkStack, **kwargs):
        super().__init__(scope, id, **kwargs)
        # Reference resources từ stack khác
        cluster = ecs.Cluster(self, "Cluster", vpc=network_stack.vpc)

# app.py — Entry point
app = cdk.App()

network = NetworkStack(app, "NetworkStack",
    env=cdk.Environment(account="123456789012", region="ap-southeast-1"))

AppStack(app, "AppStack",
    network_stack=network,
    env=cdk.Environment(account="123456789012", region="ap-southeast-1"))

app.synth()
```

### Contexts và Environment Variables

```python
# cdk.json — Lưu context values
{
  "app": "python3 app.py",
  "context": {
    "environment": "prod",
    "vpc_cidr": "10.0.0.0/16"
  }
}

# Trong code
environment = self.node.try_get_context("environment")
vpc_cidr = self.node.try_get_context("vpc_cidr")

# Deploy với context override
# cdk deploy --context environment=staging
```

### Aspects — Cross-Cutting Concerns

```python
import aws_cdk as cdk
from constructs import IConstruct

class EnforceEncryptionAspect(cdk.IAspect):
    """Tự động bật encryption cho mọi S3 bucket trong stack"""
    
    def visit(self, node: IConstruct):
        if isinstance(node, s3.CfnBucket):
            if not node.bucket_encryption:
                node.bucket_encryption = s3.CfnBucket.BucketEncryptionProperty(
                    server_side_encryption_configuration=[
                        s3.CfnBucket.ServerSideEncryptionRuleProperty(
                            server_side_encryption_by_default=s3.CfnBucket.ServerSideEncryptionByDefaultProperty(
                                sse_algorithm="AES256"
                            )
                        )
                    ]
                )

# Áp dụng cho toàn bộ stack
cdk.Aspects.of(stack).add(EnforceEncryptionAspect())
```

### Testing CDK Code

```python
# test_stack.py
import aws_cdk as cdk
from aws_cdk.assertions import Template, Match
from my_stack import MyStack

def test_s3_bucket_created():
    app = cdk.App()
    stack = MyStack(app, "TestStack")
    template = Template.from_stack(stack)
    
    # Kiểm tra S3 bucket tồn tại với versioning
    template.has_resource_properties("AWS::S3::Bucket", {
        "VersioningConfiguration": {
            "Status": "Enabled"
        }
    })

def test_bucket_count():
    app = cdk.App()
    stack = MyStack(app, "TestStack")
    template = Template.from_stack(stack)
    template.resource_count_is("AWS::S3::Bucket", 2)

def test_lambda_has_s3_permission():
    app = cdk.App()
    stack = MyStack(app, "TestStack")
    template = Template.from_stack(stack)
    
    template.has_resource_properties("AWS::IAM::Policy", {
        "PolicyDocument": {
            "Statement": Match.array_with([
                Match.object_like({
                    "Action": Match.array_with(["s3:GetObject*"]),
                    "Effect": "Allow"
                })
            ])
        }
    })
```

---

## Khi Nào Dùng CDK?

### Môi Trường Phù Hợp

```
✅ Dùng CDK khi:

1. Đội ngũ là software developers (không phải Ops-first)
   → Họ quen lập trình, IDEs, testing frameworks

2. Infrastructure phức tạp, nhiều pattern lặp lại
   → Custom Constructs tiết kiệm công sức

3. Cần tái sử dụng cao
   → Publish internal Construct Library

4. Muốn testing IaC như application code
   → Unit tests, snapshot tests

5. Tốc độ phát triển quan trọng hơn verbosity
   → L3 Constructs giảm code đáng kể

6. Tích hợp chặt với CI/CD pipeline
   → cdk deploy trong pipeline đơn giản

7. Project Greenfield (mới hoàn toàn)
   → Dễ thiết lập từ đầu hơn migrate
```

### Ví Dụ Cụ Thể

```
Phù hợp CDK:
- Startup xây dựng SaaS app trên AWS
- Platform team tạo internal constructs cho developer teams
- Microservices cần deploy nhiều Lambda + API Gateway
- Data pipeline phức tạp (Kinesis + Lambda + S3 + Glue)

Ít phù hợp CDK:
- Ops team quen YAML, không muốn học lập trình
- Simple infrastructure (1-2 resources)
- Team đã có Terraform expertise
- Multi-cloud environment
```

---

## Khi Nào Dùng CloudFormation Thuần?

```
✅ Dùng CloudFormation thuần khi:

1. Đội ngũ là Ops/SysAdmin, không familiar với lập trình
   → YAML dễ đọc hơn code với audience này

2. Infrastructure đơn giản, ít thay đổi
   → Overhead setup CDK không đáng

3. Template đã có sẵn (từ AWS Solutions, Quick Starts)
   → Không cần rewrite sang CDK

4. Console-centric workflow
   → Nhiều team vẫn thích upload template qua Console

5. Compliance yêu cầu immutable artifacts
   → Template YAML dễ audit hơn code

6. Tích hợp Service Catalog
   → Service Catalog share CloudFormation templates

7. StackSets cho multi-account governance
   → StackSets hoạt động với template, không phải CDK code
   (dù CDK có thể generate template để dùng với StackSets)
```

---

## CDK Ecosystem

### AWS CDK vs CDK for Terraform (CDKTF) vs CDK for Kubernetes (CDK8s)

| Tool | Output | Use Case |
|------|--------|----------|
| **AWS CDK** | CloudFormation templates | AWS infrastructure |
| **CDKTF** | Terraform HCL | Multi-cloud với Terraform |
| **CDK8s** | Kubernetes YAML | Kubernetes manifests |

Chung lý tưởng: Dùng ngôn ngữ lập trình để tạo config files.

### Construct Libraries Phổ Biến

```
AWS Official:
- aws-cdk-lib — Tất cả AWS service constructs
- @aws-cdk/aws-amplify-alpha — Amplify hosting

Community:
- cdk-monitoring-constructs — CloudWatch dashboards/alarms
- cdk-nag — Security best practice checks
- cdk-datadog-resources — Datadog integration
- aws-cdk-github-oidc — GitHub Actions OIDC integration

Thư viện này tại: https://constructs.dev
```

### CDK Pipelines — Self-Mutating Pipeline

```python
from aws_cdk import pipelines

pipeline = pipelines.CodePipeline(
    self, "Pipeline",
    synth=pipelines.ShellStep(
        "Synth",
        input=pipelines.CodePipelineSource.connection(
            "my-org/my-repo", "main",
            connection_arn="arn:aws:codestar-connections:..."
        ),
        commands=[
            "pip install -r requirements.txt",
            "npm install -g aws-cdk",
            "cdk synth"
        ]
    )
)

# Thêm stages (môi trường)
pipeline.add_stage(MyAppStage(self, "Dev", env=dev_env))
pipeline.add_stage(
    MyAppStage(self, "Prod", env=prod_env),
    pre=[pipelines.ManualApprovalStep("PromoteToProd")]  # Manual approval
)
```

---

## Câu Hỏi Phỏng Vấn

**Q: CDK có thay thế CloudFormation không?**
> Không. CDK là **abstraction layer** chạy trên CloudFormation, không thay thế nó. CDK compile code thành CloudFormation template và dùng CloudFormation để deploy. Stack được tạo bởi CDK vẫn là CloudFormation stack — bạn vẫn thấy nó trong CloudFormation console, có thể xem Stack Events, drift detection, rollback đều hoạt động y hệt. Hiểu CloudFormation vẫn thiết yếu khi dùng CDK vì khi debug, bạn xem CFN template và Stack Events.

**Q: CDK vs Terraform — khi nào chọn cái nào?**
> **CDK** khi: team là developers quen lập trình, stack 100% AWS, muốn type safety và unit testing, muốn dùng L3 constructs để tăng tốc phát triển. **Terraform** khi: cần multi-cloud (AWS + GCP + Azure), team đã đầu tư vào Terraform modules, muốn giữ state ngoài AWS (tự quản lý), hoặc cần tích hợp với ecosystem Terraform rộng lớn (Terragrunt, Terraform Cloud...). Nếu 100% AWS và team mạnh về code → CDK thường cho developer experience tốt hơn.

**Q: Nhược điểm của CDK so với CloudFormation thuần?**
> 1. **Abstraction leak** — CDK hide complexity, đôi khi khó debug vì cần hiểu cả code lẫn generated template. 2. **L3 Constructs opinionated** — defaults không phải lúc nào cũng phù hợp, có thể cần override nhiều. 3. **Versioning phức tạp** — CDK framework update có thể break existing code. 4. **Generated template khó đọc** — Template JSON tạo bởi CDK verbose và khó audit trực tiếp. 5. **Learning curve** — Team Ops phải học lập trình. 6. **Node.js dependency** — CDK CLI cần Node.js dù code viết bằng Python/Java.

**Q: Trong pipeline CI/CD, bạn handle CDK deploy như thế nào để an toàn?**
> Luôn chạy `cdk diff` trước `cdk deploy` để xem thay đổi. Dùng `--require-approval broadening` để phải approve khi có thay đổi security (mở thêm IAM, mở SG). Với prod, thêm manual approval gate. Dùng CDK Pipelines self-mutating pipeline để pipeline tự update khi bạn thay đổi pipeline code. Tách environments thành các Stacks riêng biệt (Dev/Staging/Prod) và deploy tuần tự với automated tests giữa các stages.

---

## 🔗 Điều Hướng

| Trước | Module Này | Tiếp Theo |
|-------|-----------|-----------|
| [5-custom-resources.md](./5-custom-resources.md) | **6-cdk-comparison.md** | [06-organizations/](../06-organizations/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
