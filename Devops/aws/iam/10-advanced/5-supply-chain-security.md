# 🔗 Supply Chain Security — Bảo Mật Chuỗi Cung Ứng Trên AWS

> **Supply Chain Security** (Bảo Mật Chuỗi Cung Ứng) bảo vệ toàn bộ pipeline từ source code → dependencies → build → container image → deployment, ngăn chặn kẻ tấn công can thiệp vào bất kỳ mắt xích nào. Các vụ tấn công như SolarWinds (2020), Log4Shell (2021), và XZ Utils (2024) đều là supply chain attacks — phần mềm hợp lệ bị nhiễm độc trước khi đến tay người dùng.

---

## 📚 Mục Lục

1. [Khái Niệm Software Supply Chain Attacks](#1-khái-niệm-software-supply-chain-attacks)
2. [SBOM — Software Bill of Materials](#2-sbom--software-bill-of-materials)
3. [AWS CodeArtifact — Private Package Repository](#3-aws-codeartifact--private-package-repository)
4. [Container Image Security](#4-container-image-security)
5. [CodePipeline Security Gates](#5-codepipeline-security-gates)
6. [Dependency Confusion Attacks](#6-dependency-confusion-attacks)
7. [Infrastructure as Code Security](#7-infrastructure-as-code-security)
8. [SLSA Framework](#8-slsa-framework)
9. [AWS Artifact Provenance Với AWS Signer](#9-aws-artifact-provenance-với-aws-signer)
10. [Kiến Trúc Supply Chain Security Toàn Diện](#10-kiến-trúc-supply-chain-security-toàn-diện)
11. [Câu Hỏi Phỏng Vấn](#11-câu-hỏi-phỏng-vấn)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. Khái Niệm Software Supply Chain Attacks

### Định Nghĩa Và Các Vector Tấn Công

```
SOFTWARE SUPPLY CHAIN (Chuỗi Cung Ứng Phần Mềm):
Là toàn bộ quá trình từ code được viết đến khi chạy trong production.

Bao gồm:
  Source Code → Dependencies → Build System → Artifacts →
  Container Images → Deployment → Runtime

ATTACK VECTORS (Véc-tơ Tấn Công):

1. Source Code Compromise:
   └── Attacker commit malicious code vào repo
   └── Insider threat
   └── Chiếm GitHub account của contributor

2. Dependency Poisoning (Đầu Độc Dependency):
   └── Malicious npm/pip/maven package
   └── Typosquatting: "reqeusts" thay vì "requests"
   └── Account takeover của maintainer

3. Build System Compromise:
   └── CI/CD server bị hack → inject code trong build
   └── SolarWinds attack: inject malware vào build process

4. Artifact Tampering (Giả Mạo Artifact):
   └── Thay thế artifact sau khi build
   └── Man-in-the-middle trong distribution

5. Registry Compromise:
   └── Docker Hub, npm registry bị hack
   └── Push malicious image tag

6. Deployment Pipeline:
   └── Kubernetes admission webhook giả mạo
   └── Helm chart từ untrusted source
```

### Case Studies Thực Tế

| Vụ Tấn Công | Năm | Kỹ Thuật | Quy Mô Ảnh Hưởng |
|---|---|---|---|
| **SolarWinds** | 2020 | Build system compromise — inject backdoor vào Orion updates | 18,000+ organizations, US Gov |
| **Log4Shell** | 2021 | Vulnerability trong Log4j library — ảnh hưởng toàn bộ Java ecosystem | Hàng triệu servers |
| **XZ Utils** | 2024 | Social engineering — attacker trở thành maintainer, backdoor SSH | Linux distributions |
| **event-stream** | 2018 | npm package maintainer account takeover | Millions of npm downloads |
| **Codecov** | 2021 | CI/CD bash uploader bị compromise | CI/CD secrets của nhiều công ty |

---

## 2. SBOM — Software Bill of Materials

### Khái Niệm

**SBOM** (Software Bill of Materials — Danh Sách Thành Phần Phần Mềm) là danh sách đầy đủ và chính thức của tất cả các thành phần (components), libraries, và dependencies trong một phần mềm.

```
Ví dụ SBOM cho ứng dụng Python:
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "components": [
    {
      "type": "library",
      "name": "requests",
      "version": "2.28.2",
      "purl": "pkg:pypi/requests@2.28.2",
      "hashes": [{"alg": "SHA-256", "content": "abc123..."}],
      "licenses": [{"license": {"id": "Apache-2.0"}}]
    },
    {
      "type": "library",
      "name": "cryptography",
      "version": "38.0.4",
      "purl": "pkg:pypi/cryptography@38.0.4",
      "vulnerabilities": []  ← Amazon Inspector điền vào đây
    }
  ]
}
```

### Tạo SBOM Với AWS Inspector

```bash
# Amazon Inspector tự động generate SBOM cho EC2/ECR/Lambda

# Enable Inspector
aws inspector2 enable \
  --resource-types EC2 ECR LAMBDA

# Export SBOM cho ECR repository
aws inspector2 create-sbom-export \
  --resource-filter-criteria '{
    "ecrRepositoryName": [{
      "comparison": "EQUALS",
      "value": "my-app"
    }]
  }' \
  --report-format CYCLONEDX_1_4 \
  --s3-destination '{
    "bucketName": "my-sbom-bucket",
    "keyPrefix": "sboms/",
    "kmsKeyArn": "arn:aws:kms:us-east-1:123456789012:key/xxx"
  }'

# List SBOM export jobs
aws inspector2 list-finding-aggregations \
  --aggregation-type REPOSITORY
```

### Tạo SBOM Tại Build Time

```yaml
# Trong CodeBuild buildspec.yml — generate SBOM sau khi build
version: 0.2
phases:
  build:
    commands:
      - docker build -t $IMAGE_TAG .

  post_build:
    commands:
      # Dùng Syft để generate SBOM
      - curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
      - syft $IMAGE_TAG -o cyclonedx-json > sbom.json

      # Upload SBOM lên S3
      - aws s3 cp sbom.json s3://$SBOM_BUCKET/$(date +%Y/%m/%d)/$IMAGE_NAME-$BUILD_ID.json

      # Scan vulnerabilities
      - grype sbom:./sbom.json --fail-on high
```

---

## 3. AWS CodeArtifact — Private Package Repository

### Kiến Trúc CodeArtifact

```
PUBLIC REGISTRIES (npm, PyPI, Maven Central)
          │
          ▼ Proxied & cached với security scanning
┌──────────────────────────────────────────────────┐
│              AWS CodeArtifact                     │
│                                                   │
│  Domain: company-packages                        │
│  ├── Repository: npm-store                       │
│  │   ├── Upstream: public-npm (npmjs.com)        │
│  │   └── Your private npm packages               │
│  │                                               │
│  ├── Repository: pypi-store                      │
│  │   ├── Upstream: public-pypi (pypi.org)        │
│  │   └── Your private Python packages            │
│  │                                               │
│  └── Repository: maven-store                     │
│      ├── Upstream: maven-central                 │
│      └── Your private Java packages              │
└──────────────────────────────────────────────────┘
          │
          ▼ Developers & CI/CD pull packages from here
  Internal Services / CodeBuild / Lambda
```

### Tạo CodeArtifact Domain và Repository

```bash
# Tạo domain (domain là namespace cho repositories)
aws codeartifact create-domain \
  --domain company-packages \
  --encryption-key arn:aws:kms:us-east-1:123456789012:key/xxx

# Tạo repository với upstream PyPI
aws codeartifact create-repository \
  --domain company-packages \
  --repository pypi-store \
  --description "Python packages with upstream PyPI" \
  --upstreams repositoryName=public-pypi

# Tạo external connection đến public PyPI
aws codeartifact associate-external-connection \
  --domain company-packages \
  --repository public-pypi \
  --external-connection public:pypi

# Lấy token để authenticate pip
CODEARTIFACT_TOKEN=$(aws codeartifact get-authorization-token \
  --domain company-packages \
  --query authorizationToken \
  --output text)

# Cấu hình pip để dùng CodeArtifact
CODEARTIFACT_ENDPOINT=$(aws codeartifact get-repository-endpoint \
  --domain company-packages \
  --repository pypi-store \
  --format pypi \
  --query repositoryEndpoint \
  --output text)

pip config set global.index-url \
  "https://aws:${CODEARTIFACT_TOKEN}@${CODEARTIFACT_ENDPOINT}simple/"
```

### Package Origin Control — Ngăn Chặn Dependency Confusion

```bash
# Kiểm tra package origin policy
aws codeartifact get-package-version-origin \
  --domain company-packages \
  --repository pypi-store \
  --format pypi \
  --package requests \
  --package-version 2.28.2

# Set origin control: chỉ cho phép install từ upstream (không cho internal override public package)
aws codeartifact put-package-origin-configuration \
  --domain company-packages \
  --repository pypi-store \
  --format pypi \
  --package requests \
  --restrictions '{
    "publish": "BLOCK",
    "upstream": "ALLOW"
  }'

# Đối với internal packages: chỉ cho phép publish từ internal, không fetch từ upstream
aws codeartifact put-package-origin-configuration \
  --domain company-packages \
  --repository pypi-store \
  --format pypi \
  --package company-internal-lib \
  --restrictions '{
    "publish": "ALLOW",
    "upstream": "BLOCK"
  }'
```

### Repository Policy — Kiểm Soát Truy Cập

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgReadAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "codeartifact:GetPackageVersionReadme",
        "codeartifact:GetAuthorizationToken",
        "codeartifact:ReadFromRepository",
        "codeartifact:GetRepositoryEndpoint",
        "codeartifact:ListPackages",
        "codeartifact:ListPackageVersions"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        }
      }
    },
    {
      "Sid": "AllowCIPipelinePublish",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/CodeBuildServiceRole"
      },
      "Action": [
        "codeartifact:PublishPackageVersion",
        "codeartifact:PutPackageMetadata"
      ],
      "Resource": "*"
    }
  ]
}
```

### Audit Trail Với CloudTrail

```bash
# Xem lịch sử download package từ CodeArtifact
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=codeartifact.amazonaws.com \
  --query 'Events[?contains(EventName, `GetPackageVersion`)].
    {Time: EventTime, User: Username, Package: RequestParameters}' \
  --output table
```

---

## 4. Container Image Security

### Amazon ECR Image Scanning

```bash
# Enable enhanced scanning (Amazon Inspector) cho ECR
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{
    "repositoryFilters": [{"filter": "*", "filterType": "WILDCARD"}],
    "scanFrequency": "CONTINUOUS_SCAN"
  }]'

# Enable basic scanning trên push
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true

# Xem kết quả scan
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=latest \
  --query 'imageScanFindings.findings[?severity==`CRITICAL`]' \
  --output table

# Get aggregate scan summary
aws inspector2 list-findings \
  --filter-criteria '{
    "ecrImageRepositoryName": [{"comparison": "EQUALS", "value": "my-app"}],
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}]
  }' \
  --query 'findings[*].[title,severity,packageVulnerabilityDetails.vulnerabilityId]' \
  --output table
```

### CodeBuild Gate: Fail Build Nếu Có Critical Vulnerabilities

```yaml
# buildspec.yml với security gate
version: 0.2
phases:
  build:
    commands:
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION .

  post_build:
    commands:
      # Push image
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION

      # Chờ scan hoàn thành
      - |
        echo "Waiting for image scan to complete..."
        for i in {1..30}; do
          STATUS=$(aws ecr describe-image-scan-findings \
            --repository-name $IMAGE_NAME \
            --image-id imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION \
            --query 'imageScanStatus.status' \
            --output text 2>/dev/null)
          if [ "$STATUS" = "COMPLETE" ]; then
            break
          fi
          sleep 10
        done

      # Kiểm tra CRITICAL vulnerabilities
      - |
        CRITICAL_COUNT=$(aws ecr describe-image-scan-findings \
          --repository-name $IMAGE_NAME \
          --image-id imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION \
          --query 'imageScanFindings.findingSeverityCounts.CRITICAL' \
          --output text)

        if [ "$CRITICAL_COUNT" != "None" ] && [ "$CRITICAL_COUNT" -gt 0 ]; then
          echo "SECURITY GATE FAILED: $CRITICAL_COUNT CRITICAL vulnerabilities found!"
          echo "Run: aws ecr describe-image-scan-findings --repository-name $IMAGE_NAME --image-id imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION"
          exit 1
        fi
        echo "Security gate passed: No critical vulnerabilities"
```

### Image Signing Với AWS Signer và Notation

**AWS Signer** + **Notation** (CNCF project) cho phép ký container images để đảm bảo tính toàn vẹn.

```bash
# Cài đặt Notation CLI
brew install notation

# Cài đặt AWS Signer plugin cho Notation
notation plugin install --url https://d2hvyiie43lped.cloudfront.net/linux/amd64/plugin/latest/notation-aws-signer-plugin.zip

# Tạo Signing Profile trong AWS Signer
aws signer put-signing-profile \
  --profile-name container-signing-profile \
  --platform-id Notation-OCI-SHA384-ECDSA \
  --signature-validity-period '{"value": 12, "type": "MONTHS"}' \
  --signing-material '{"certificateArn": "arn:aws:acm:..."}'

# Ký image sau khi push lên ECR
REGISTRY="123456789012.dkr.ecr.us-east-1.amazonaws.com"
IMAGE_REF="$REGISTRY/my-app:v1.0.0"

notation sign $IMAGE_REF \
  --plugin com.amazonaws.signer.notation.plugin \
  --id arn:aws:signer:us-east-1:123456789012:/signing-profiles/container-signing-profile

# Verify signature
notation verify $IMAGE_REF \
  --policy-config-file trust-policy.json
```

### Trust Policy Cho Notation

```json
{
  "version": "1.0",
  "trustPolicies": [
    {
      "name": "production-policy",
      "registryScopes": [
        "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app"
      ],
      "signatureVerification": {
        "level": "strict"
      },
      "trustStores": ["signingAuthority:aws-signer-ts"],
      "trustedIdentities": [
        "arn:aws:signer:us-east-1:123456789012:/signing-profiles/container-signing-profile"
      ]
    }
  ]
}
```

### ECR Lifecycle Policies

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Remove untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 2,
      "description": "Keep only last 5 production images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod-"],
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": {"type": "expire"}
    }
  ]
}
```

```bash
# Apply lifecycle policy
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text file://ecr-lifecycle.json
```

---

## 5. CodePipeline Security Gates

### Pipeline Với Security Checkpoints

```
CodeCommit      CodeBuild        CodeBuild         CodeBuild        CodeDeploy
(Source)    →   (Build+Test) →  (Security Gate) → (SBOM+Sign)  →   (Deploy)
    │               │                │                   │               │
    │               │                │                   │               │
 Git commit    Unit tests       SAST scan          Generate SBOM    Blue/green
 PR required   Lint code        SCA scan           Sign image       deploy
 Branch        Docker build     DAST (optional)    Push to ECR      Smoke test
 protection    Run tests        Policy check       Attest artifact  Rollback ready
```

```python
# Lambda function: security gate check (dùng trong CodePipeline)
import boto3
import json

codepipeline = boto3.client('codepipeline')
ecr = boto3.client('ecr')
inspector = boto3.client('inspector2')


def lambda_handler(event, context):
    """
    CodePipeline Lambda approval action.
    Fail pipeline nếu có critical vulns hoặc image chưa được signed.
    """
    job_id = event['CodePipeline.job']['id']
    job_data = event['CodePipeline.job']['data']

    try:
        # Lấy artifacts từ pipeline
        input_artifacts = job_data['inputArtifacts']
        build_details = get_build_details(input_artifacts)

        image_uri = build_details.get('imageUri')
        if not image_uri:
            raise ValueError("No image URI found in build artifacts")

        # Check 1: CRITICAL vulnerabilities
        critical_findings = check_critical_vulnerabilities(image_uri)
        if critical_findings:
            raise Exception(
                f"Security gate FAILED: {len(critical_findings)} CRITICAL vulnerabilities. "
                f"CVEs: {[f['packageVulnerabilityDetails']['vulnerabilityId'] for f in critical_findings[:5]]}"
            )

        # Check 2: Image signing (nếu dùng Notation)
        # (Trong production, check signature với notation verify)

        # Check 3: Approved base image
        base_image = build_details.get('baseImage', '')
        if not is_approved_base_image(base_image):
            raise Exception(f"Base image not in approved list: {base_image}")

        # Tất cả checks pass
        codepipeline.put_job_success_result(
            jobId=job_id,
            outputVariables={
                'securityGatePassed': 'true',
                'scanCompletedAt': build_details.get('scanTime', '')
            }
        )

    except Exception as e:
        codepipeline.put_job_failure_result(
            jobId=job_id,
            failureDetails={
                'type': 'JobFailed',
                'message': str(e)
            }
        )


def check_critical_vulnerabilities(image_uri):
    """Kiểm tra CRITICAL findings từ Amazon Inspector."""
    repo_name = image_uri.split('/')[-1].split(':')[0]
    image_tag = image_uri.split(':')[-1]

    response = inspector.list_findings(
        filterCriteria={
            'ecrImageRepositoryName': [{'comparison': 'EQUALS', 'value': repo_name}],
            'ecrImageTags': [{'comparison': 'EQUALS', 'value': image_tag}],
            'severity': [{'comparison': 'EQUALS', 'value': 'CRITICAL'}]
        }
    )
    return response.get('findings', [])


def is_approved_base_image(base_image):
    """Kiểm tra base image trong approved list."""
    APPROVED_BASE_IMAGES = [
        "public.ecr.aws/amazonlinux/amazonlinux",
        "public.ecr.aws/docker/library/python",
        "123456789012.dkr.ecr.us-east-1.amazonaws.com/base-images"
    ]
    return any(base_image.startswith(approved) for approved in APPROVED_BASE_IMAGES)
```

---

## 6. Dependency Confusion Attacks

### Dependency Confusion Là Gì?

```
ATTACK SCENARIO:

1. Attacker phát hiện công ty dùng internal package "company-auth-lib"
   (thường thấy trong package.json, requirements.txt, pom.xml)

2. Attacker publish package "company-auth-lib" lên PUBLIC registry (npm/PyPI)
   với version cao hơn (ví dụ: 99.0.0 vs internal 1.0.0)

3. Package manager (npm, pip) thường prefer highest version
   → Tự động download malicious package từ public registry!

4. Malicious package chứa backdoor/keylogger/crypto miner
   → Chạy trong production environment của nạn nhân

Vụ thực tế: Alex Birsan (2021) thực hiện thành công với
35 companies lớn (Apple, Microsoft, Uber, Netflix, Tesla...)
```

### Biện Pháp Phòng Tránh

```bash
# BIỆN PHÁP 1: Dùng CodeArtifact với Package Origin Control
# (Như đã đề cập ở mục 3 - set internal packages: upstream=BLOCK)

# BIỆN PHÁP 2: Private namespace (namespace squatting)
# Publish "shells" của internal package names lên public registry
# để attacker không thể claim tên đó

# BIỆN PHÁP 3: Lock file bắt buộc
# requirements.txt với exact versions VÀ hashes
pip install --require-hashes -r requirements.txt

# requirements.txt với hash:
# requests==2.28.2 \
#   --hash=sha256:64299f4909223da747622c030b781c0d7811e359c37124b4bd368fb8c6518baa \
#   --hash=sha256:98b1b2782e3c6c4904938b84c0eb932721069dfdb9134313beff7c83c2df24bf

# BIỆN PHÁP 4: pip với --extra-index-url bị nguy hiểm
# Đừng dùng --extra-index-url (pip search cả 2, prefer version cao hơn)
# Thay bằng --index-url (chỉ search CodeArtifact)

# SAI (dễ bị dependency confusion):
pip install --extra-index-url https://codeartifact-url company-internal-lib

# ĐÚNG (chỉ tìm trong CodeArtifact):
pip install --index-url https://codeartifact-url company-internal-lib

# BIỆN PHÁP 5: SCP ngăn deploy package từ public registry trực tiếp
# (Enforce tất cả install phải qua CodeArtifact)
```

### npm với Private Registry

```json
// .npmrc — cấu hình tất cả scope đi qua CodeArtifact
@company:registry=https://company-packages-123456789012.d.codeartifact.us-east-1.amazonaws.com/npm/npm-store/
//company-packages-123456789012.d.codeartifact.us-east-1.amazonaws.com/npm/npm-store/:_authToken=${CODEARTIFACT_AUTH_TOKEN}

// Và global npm cũng dùng CodeArtifact (không phải npmjs.com)
registry=https://company-packages-123456789012.d.codeartifact.us-east-1.amazonaws.com/npm/npm-store/
```

---

## 7. Infrastructure as Code Security

### Công Cụ Scan IaC

| Tool | IaC Type | Tích Hợp CI/CD | Mô Tả |
|---|---|---|---|
| **cfn-nag** | CloudFormation | CodeBuild | Tìm security issues trong CFN templates |
| **Checkov** | Terraform, CFN, K8s, ARM | CodeBuild, GitHub Actions | Multi-IaC scanner với 1000+ rules |
| **Terrascan** | Terraform, K8s, Helm | CI/CD | Policy-as-code với OPA |
| **tfsec** | Terraform | Pre-commit, CI/CD | Fast Terraform scanner |
| **cfn-guard** | CloudFormation | CodePipeline | AWS native policy-as-code |

### Tích Hợp Checkov Vào CodeBuild

```yaml
# buildspec-iac-scan.yml
version: 0.2
phases:
  install:
    runtime-versions:
      python: 3.11
    commands:
      - pip install checkov

  build:
    commands:
      # Scan Terraform
      - checkov -d ./terraform \
          --framework terraform \
          --output junitxml \
          --output-file-path ./results/checkov-terraform.xml \
          --soft-fail  # Remove this in production!

      # Scan CloudFormation
      - checkov -d ./cloudformation \
          --framework cloudformation \
          --output junitxml \
          --output-file-path ./results/checkov-cfn.xml

      # Kiểm tra kết quả (fail build nếu có HIGH findings)
      - |
        HIGH_FAILS=$(checkov -d ./terraform --framework terraform \
          --check HIGH --compact --quiet 2>/dev/null | \
          grep "^Check:" | wc -l)

        if [ "$HIGH_FAILS" -gt 0 ]; then
          echo "IaC Security gate FAILED: $HIGH_FAILS HIGH severity findings"
          checkov -d ./terraform --framework terraform --check HIGH
          exit 1
        fi
        echo "IaC security gate passed"

reports:
  CheckovReport:
    files:
      - ./results/checkov-*.xml
    file-format: JUNITXML
```

### cfn-guard Custom Rules

```
# cfn-guard rule: S3 buckets must not be public
# s3-security.guard

rule S3_BUCKET_NO_PUBLIC_ACL {
  AWS::S3::Bucket {
    Properties {
      PublicAccessBlockConfiguration {
        BlockPublicAcls == true
        BlockPublicPolicy == true
        IgnorePublicAcls == true
        RestrictPublicBuckets == true
      }
    }
  }
}

rule S3_BUCKET_ENCRYPTION_REQUIRED {
  AWS::S3::Bucket {
    Properties {
      BucketEncryption exists
      BucketEncryption {
        ServerSideEncryptionConfiguration[*] {
          ServerSideEncryptionByDefault {
            SSEAlgorithm in ["aws:kms", "AES256"]
          }
        }
      }
    }
  }
}
```

```bash
# Chạy cfn-guard
cfn-guard validate \
  --data cloudformation/template.yaml \
  --rules s3-security.guard \
  --output-format json
```

---

## 8. SLSA Framework

**SLSA** (Supply chain Levels for Software Artifacts — Cấp Độ Chuỗi Cung Ứng cho Phần Mềm Artifact) là framework của Google (hiện là CNCF project) để đánh giá mức độ bảo mật của build process.

### 4 Cấp Độ SLSA

```
SLSA Level 1 — Documentation:
  ✅ Build process được document hoàn chỉnh
  ✅ Provenance (nguồn gốc) được generate tự động
  → Biết artifact được build từ đâu

SLSA Level 2 — Tamper Resistance (Chống Giả Mạo Cơ Bản):
  ✅ Level 1 +
  ✅ Version control (Git) được sử dụng
  ✅ Build service hosted (không phải dev máy cá nhân)
  ✅ Provenance được sign bởi build service
  → Build service không thể bị attacker kiểm soát dễ dàng

SLSA Level 3 — Hardened Build:
  ✅ Level 2 +
  ✅ Source code được audited
  ✅ Build service hardened (chỉ chạy trên trusted platform)
  ✅ Non-falsifiable provenance
  → Không thể tự mình generate fake provenance

SLSA Level 4 — Max Assurance (Bảo Đảm Tối Đa):
  ✅ Level 3 +
  ✅ Two-person review cho mọi change
  ✅ Hermetic build (build hoàn toàn cô lập, reproducible)
  ✅ Parameterless build
  → Highest assurance, phù hợp với critical software
```

### SLSA Với AWS CodeBuild

```yaml
# CodeBuild hỗ trợ SLSA Level 2+ khi:
# 1. Build trong isolated environment (CodeBuild có)
# 2. Provenance được generate bởi CodeBuild (có với SBOM export)
# 3. Source từ CodeCommit/GitHub với protected branches

# buildspec với SLSA provenance generation
version: 0.2
env:
  exported-variables:
    - IMAGE_URI
    - BUILD_DIGEST
    - SOURCE_COMMIT

phases:
  build:
    commands:
      - export SOURCE_COMMIT=$CODEBUILD_RESOLVED_SOURCE_VERSION
      - docker build -t $IMAGE_NAME:$SOURCE_COMMIT .
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$IMAGE_NAME:$SOURCE_COMMIT

      # Generate provenance document
      - |
        cat > provenance.json << EOF
        {
          "buildType": "https://cloudbuild.slsa.dev/Image/v1",
          "builder": {
            "id": "arn:aws:codebuild:$AWS_REGION:$AWS_ACCOUNT_ID:project/$CODEBUILD_BUILD_ID"
          },
          "invocation": {
            "configSource": {
              "uri": "$CODEBUILD_SOURCE_REPO_URL",
              "digest": {"sha1": "$SOURCE_COMMIT"},
              "entryPoint": "buildspec.yml"
            }
          },
          "metadata": {
            "buildInvocationId": "$CODEBUILD_BUILD_ID",
            "buildStartedOn": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
          }
        }
        EOF
      - export BUILD_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' $IMAGE_NAME:$SOURCE_COMMIT)
      - export IMAGE_URI="$AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$IMAGE_NAME:$SOURCE_COMMIT"

      # Sign provenance với AWS Signer
      - aws signer start-signing-job \
          --source '{"s3": {"bucketName": "'"$ARTIFACT_BUCKET"'", "key": "provenance.json", "version": "latest"}}' \
          --destination '{"s3": {"bucketName": "'"$ARTIFACT_BUCKET"'", "prefix": "signed-provenance/"}}' \
          --profile-name container-signing-profile
```

---

## 9. AWS Artifact Provenance Với AWS Signer

AWS Signer là dịch vụ managed để ký code artifacts với PKI certificates.

### Các Use Cases AWS Signer

| Use Case | Signer Profile Platform |
|---|---|
| Container images (Notation) | `Notation-OCI-SHA384-ECDSA` |
| Lambda function code | `AWSLambda-SHA384-ECDSA` |
| IoT devices | `AWSIoTDeviceManagement-SHA256-ECDSA` |
| General artifacts | Custom profiles |

### Ký Lambda Function

```bash
# Tạo signing profile cho Lambda
aws signer put-signing-profile \
  --profile-name lambda-signing-profile \
  --platform-id AWSLambda-SHA384-ECDSA \
  --signing-material '{"certificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/xxx"}' \
  --signature-validity-period '{"value": 12, "type": "MONTHS"}'

# Ký Lambda deployment package
aws signer start-signing-job \
  --source '{
    "s3": {
      "bucketName": "my-lambda-bucket",
      "key": "function.zip",
      "version": "latest"
    }
  }' \
  --destination '{
    "s3": {
      "bucketName": "my-lambda-bucket",
      "prefix": "signed/"
    }
  }' \
  --profile-name lambda-signing-profile

# Tạo Code Signing Config cho Lambda
aws lambda create-code-signing-config \
  --description "Require signed Lambda deployments" \
  --allowed-publishers '{
    "SigningProfileVersionArns": [
      "arn:aws:signer:us-east-1:123456789012:/signing-profiles/lambda-signing-profile/xxxx"
    ]
  }' \
  --code-signing-policies '{"untrustedArtifactOnDeployment": "Enforce"}'

# Gắn Code Signing Config vào Lambda function
aws lambda put-function-code-signing-config \
  --function-name my-function \
  --code-signing-config-arn arn:aws:lambda:us-east-1:123456789012:code-signing-config:csc-xxx
```

---

## 10. Kiến Trúc Supply Chain Security Toàn Diện

```
DEVELOPER WORKSTATION
  │  Git commit (signed)
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    SOURCE CONTROL (CodeCommit/GitHub)                 │
│  Branch protection: require PR + 2 reviews                           │
│  Secret scanning: treto, git-secrets (pre-commit hook)               │
│  SAST: CodeGuru Reviewer (tự động trong PR)                          │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ Merge to main triggers pipeline
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         CI/CD PIPELINE (CodePipeline)                │
│                                                                       │
│  Stage 1: Source ──► CodeCommit/GitHub                               │
│                                                                       │
│  Stage 2: Build (CodeBuild)                                          │
│    ├── Install deps từ CodeArtifact (không phải public registry)     │
│    ├── Run unit tests                                                 │
│    └── Build Docker image                                            │
│                                                                       │
│  Stage 3: Security Gate (CodeBuild hoặc Lambda)                      │
│    ├── SAST: Checkov scan IaC                                        │
│    ├── SCA: scan dependencies (OWASP Dependency Check)               │
│    ├── Container scan: Amazon Inspector (ECR scan results)           │
│    ├── SBOM generation: Syft → S3                                    │
│    └── Image signing: AWS Signer + Notation                          │
│                                                                       │
│  Stage 4: Staging Deploy → Integration tests → DAST                 │
│                                                                       │
│  Stage 5: Production Deploy (manual approval required)               │
│    └── Verify image signature trước khi deploy                       │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    ARTIFACT REGISTRY (Amazon ECR)                     │
│  Image scanning: CONTINUOUS_SCAN (Amazon Inspector)                  │
│  Image signing: Notation + AWS Signer                                │
│  Lifecycle policy: clean up old/untagged images                      │
│  Repository policy: PrincipalOrgID restrict                          │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    RUNTIME (ECS/EKS/Lambda)                           │
│  Admission controller: verify image signature before deploy          │
│  Runtime security: Amazon GuardDuty EKS Protection                   │
│  Least privilege IAM: IRSA / Task Role                               │
│  Read-only root filesystem                                           │
│  Network policy: restrict pod-to-pod communication                   │
└──────────────────────────────────────────────────────────────────────┘
         │ Events/Findings
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    MONITORING & RESPONSE                              │
│  Security Hub: aggregate findings từ Inspector, GuardDuty            │
│  EventBridge: route critical findings → auto-remediation Lambda      │
│  SBOM inventory: tracking all component versions và CVEs             │
│  Alert: new CVE affecting deployed version → auto open ticket        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q1: Supply chain attack là gì? Cho ví dụ thực tế.**

> Supply chain attack là tấn công thông qua bên thứ ba trong quy trình phát triển — thay vì tấn công trực tiếp vào target, attacker compromise một thành phần mà target tin cậy. SolarWinds: attacker compromise build process của SolarWinds → inject backdoor vào software update → 18,000 customers bị compromise. Log4Shell: vulnerability trong Log4j library ảnh hưởng hàng triệu Java applications.

**Q2: Dependency confusion attack là gì và cách phòng?**

> Dependency confusion: attacker publish malicious package với tên trùng internal package lên public registry (npm/PyPI) với version cao hơn → package manager tự động download public version. Phòng: (1) Dùng CodeArtifact với Package Origin Control (block upstream cho internal packages); (2) Dùng `--index-url` thay vì `--extra-index-url`; (3) Namespace squatting — publish shell packages lên public registry để "claim" tên; (4) Lock file với hash verification.

**Q3: SBOM là gì và tại sao quan trọng?**

> SBOM (Software Bill of Materials) là danh sách đầy đủ tất cả components, libraries, dependencies trong phần mềm. Quan trọng vì: (1) Khi Log4Shell xuất hiện, có SBOM → biết ngay đang dùng log4j version mấy, bao nhiêu services bị ảnh hưởng; (2) Compliance requirements (US Executive Order 14028 yêu cầu SBOM cho government software); (3) License management; (4) Supply chain risk tracking.

**Q4: Làm thế nào implement container image signing trên AWS?**

> Dùng AWS Signer + Notation (CNCF project): (1) Tạo Signing Profile trong AWS Signer với certificate từ ACM; (2) Sau khi build và push image lên ECR, ký với `notation sign` + AWS Signer plugin; (3) Tạo Trust Policy định nghĩa trusted signing profiles; (4) Trong deployment, verify signature trước với `notation verify` — fail deployment nếu signature invalid/missing. EKS có thể dùng admission webhook để enforce.

**Q5: SLSA framework là gì và AWS đáp ứng cấp độ nào?**

> SLSA (Supply chain Levels for Software Artifacts) là framework 4 cấp độ của Google/CNCF để đánh giá build security. AWS CodeBuild có thể đạt SLSA Level 2 (hosted build service, non-falsifiable provenance) và hướng đến Level 3 (hardened build, isolated environment). Cần: protected branches, code review, signed provenance, build isolation.

**Q6: Bạn sẽ thiết kế CodePipeline an toàn cho production như thế nào?**

> Các bước: (1) Source: protected branches, PR required, secret scanning (pre-commit); (2) Dependencies: CodeArtifact với Package Origin Control, không dùng public registry trực tiếp; (3) Build: CodeBuild với IAM role least privilege, no internet egress (VPC); (4) Security gates: SAST (Checkov), SCA (OWASP Dependency Check), container scan (Inspector); (5) Artifact: Sign images với AWS Signer, generate SBOM; (6) Deploy: manual approval cho production, verify signature trước deploy; (7) Runtime: GuardDuty, read-only filesystem, network policy.

---

## 12. Key Takeaways

> **Supply chain attack targets your trust** — kẻ tấn công không break vào thẳng, mà compromise thứ bạn tin tưởng: library, build tool, container image. Defense phải ở mọi mắt xích, không chỉ ở perimeter.

> **CodeArtifact + Package Origin Control** là biện pháp quan trọng nhất để chống dependency confusion — internal packages không thể bị override bởi public registry.

> **SBOM là "bill of sale" của phần mềm** — khi có CVE mới, SBOM cho phép trong vài phút biết đúng đâu bị ảnh hưởng, thay vì mất cả ngày inventory.

> **Image signing với AWS Signer + Notation** đảm bảo chỉ images được build từ CI/CD chính thức mới có thể deploy — ngăn "rogue images" chạy trong production.

> **Security gates trong pipeline** là "fail fast" cho security — phát hiện CRITICAL vulnerability tại build time rẻ hơn 100x so với phát hiện sau khi deploy production. SLSA framework cung cấp chuẩn để đánh giá và cải thiện build security systematically.
