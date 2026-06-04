# Image Security — Bảo Mật Container Image

> Hướng dẫn về bảo mật container image: quét lỗ hổng CVE (Common Vulnerabilities and Exposures — Lỗ Hổng Bảo Mật Phổ Biến) với Trivy và Snyk, kiểm soát policy với OPA Gatekeeper và Kyverno, ký và xác minh image với Cosign (image signing — ký image).

## Mục Lục

1. [Tại Sao Image Security Quan Trọng?](#tại-sao-image-security-quan-trọng)
2. [Image Scanning — Quét Lỗ Hổng](#image-scanning--quét-lỗ-hổng)
3. [Trivy — Scanner Phổ Biến Nhất](#trivy--scanner-phổ-biến-nhất)
4. [Tích Hợp Trivy Vào CI/CD](#tích-hợp-trivy-vào-cicd)
5. [OPA Gatekeeper — Kiểm Soát Policy](#opa-gatekeeper--kiểm-soát-policy)
6. [Kyverno — Policy Engine Đơn Giản Hơn](#kyverno--policy-engine-đơn-giản-hơn)
7. [Image Signing với Cosign](#image-signing-với-cosign)
8. [Best Practice Image Security](#best-practice-image-security)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Image Security Quan Trọng?

Container image là **attack surface lớn nhất** trong Kubernetes workload. Image có thể chứa:

```
Rủi Ro Từ Image:
├── CVE trong OS packages (libopenssl, glibc, curl...)
│   → Attacker exploit lỗ hổng đã biết để leo thang quyền
│
├── CVE trong application dependencies (log4j, struts, spring...)
│   → Remote code execution, data exfiltration
│
├── Malware được nhúng vào image (supply chain attack)
│   → Cryptocurrency miner, backdoor, data stealer
│
├── Secret hardcode trong image layer (password, API key)
│   → Lộ credential trong registry history
│
└── Image từ registry không tin cậy
    → Không biết ai đã build image này và có gì trong đó
```

**Vòng đời bảo mật image:**

```
Build → Scan → Sign → Push → Verify → Run
  │        │       │      │       │       │
  │        │       │      │       │       └── Runtime scanning (Falco)
  │        │       │      │       └── Gatekeeper/Kyverno verify signature
  │        │       │      └── Push lên private registry
  │        │       └── Cosign ký image
  │        └── Trivy scan — fail nếu có Critical/High CVE
  └── Dockerfile best practice (non-root, distroless)
```

---

## Image Scanning — Quét Lỗ Hổng

### Các Công Cụ Scanner Phổ Biến

| Tool | Ưu Điểm | Nhược Điểm | Use Case |
| ---- | -------- | ---------- | -------- |
| **Trivy** | Miễn phí, nhanh, nhiều target | Phải tự tích hợp | CI/CD pipeline |
| **Grype** | Nhẹ, output JSON chuẩn | Database update chậm hơn | CI/CD pipeline |
| **Snyk** | SaaS, dễ tích hợp, fix suggestion | Tính phí | Enterprise, dev workflow |
| **Clair** | Open source, API-based | Phức tạp hơn để cài | Registry integration |
| **Anchore** | Policy engine mạnh | Nặng | Enterprise compliance |

---

## Trivy — Scanner Phổ Biến Nhất

**Trivy** là vulnerability scanner mã nguồn mở của Aqua Security — quét image, filesystem, git repo, Kubernetes cluster.

### Cài Đặt Trivy

```bash
# macOS
brew install trivy

# Linux
apt-get install trivy
# hoặc
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -

# Dùng Docker (không cần cài)
docker run aquasec/trivy image nginx:latest
```

### Quét Image

```bash
# Quét image từ DockerHub
trivy image nginx:latest

# Quét image local (chưa push)
trivy image myapp:v1.0

# Quét image trong private registry
trivy image \
  --username $REGISTRY_USER \
  --password $REGISTRY_PASSWORD \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1

# Chỉ hiện lỗ hổng nghiêm trọng (Critical và High)
trivy image --severity CRITICAL,HIGH nginx:latest

# Fail nếu có CVE nghiêm trọng (dùng trong CI)
trivy image --severity CRITICAL --exit-code 1 myapp:v1

# Output JSON để xử lý tự động
trivy image --format json -o results.json myapp:v1

# Output SARIF (GitHub Security tab format)
trivy image --format sarif -o trivy-results.sarif myapp:v1
```

### Ví Dụ Output

```
Total: 127 (UNKNOWN: 0, LOW: 87, MEDIUM: 28, HIGH: 11, CRITICAL: 1)

┌─────────────────────────┬────────────────────┬──────────┬───────────────────┐
│         Library         │   Vulnerability    │ Severity │  Installed Ver.  │
├─────────────────────────┼────────────────────┼──────────┼───────────────────┤
│ openssl                 │ CVE-2023-0286      │ CRITICAL │ 1.1.1n-0+deb11u3 │
│ (fixed: 1.1.1n-0+deb11u4)                                                   │
├─────────────────────────┼────────────────────┼──────────┼───────────────────┤
│ curl                    │ CVE-2023-38546     │ HIGH     │ 7.74.0-1.3+deb11 │
└─────────────────────────┴────────────────────┴──────────┴───────────────────┘
```

### Quét Kubernetes Cluster

```bash
# Quét toàn bộ cluster K8s
trivy k8s --report all

# Quét namespace cụ thể
trivy k8s --namespace production --report summary

# Quét Pod cụ thể
trivy k8s pod/myapp-pod-abc123 -n production

# Tạo report HTML
trivy k8s --report all --format template \
  --template "@contrib/html.tpl" -o trivy-k8s-report.html
```

### Quét Dockerfile Và IaC

```bash
# Quét Dockerfile (misconfig scan)
trivy config ./Dockerfile

# Quét toàn bộ thư mục IaC (Helm chart, Terraform, K8s manifest)
trivy config ./k8s-manifests/
trivy config ./helm-charts/myapp/

# Kết hợp image + config scan
trivy image myapp:v1
trivy config ./k8s-manifests/
```

---

## Tích Hợp Trivy Vào CI/CD

### GitHub Actions

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan image với Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1                         # fail pipeline nếu có Critical/High

      - name: Upload kết quả lên GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: trivy-results.sarif

      - name: Scan Dockerfile và K8s manifest
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: .
          exit-code: 1
          severity: CRITICAL,HIGH
```

### GitLab CI

```yaml
# .gitlab-ci.yml
trivy-container-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image
        --exit-code 1
        --severity CRITICAL,HIGH
        --format json
        --output trivy-report.json
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    reports:
      container_scanning: trivy-report.json
    when: always
  allow_failure: false    # fail pipeline nếu có CVE nghiêm trọng
```

### Trivy Operator — Scan Tự Động Trong Cluster

```bash
# Cài Trivy Operator — tự động scan image của mọi workload trong cluster
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/trivy-operator/v0.16.0/deploy/static/trivy-operator.yaml

# Xem kết quả scan
kubectl get vulnerabilityreports -n production
kubectl describe vulnerabilityreport myapp-deployment-abc123 -n production

# Alert khi có Critical vulnerability
kubectl get vulnerabilityreports -A \
  -o jsonpath='{range .items[*]}{.metadata.name}: CRITICAL={.report.summary.criticalCount}{"\n"}{end}'
```

---

## OPA Gatekeeper — Kiểm Soát Policy

**OPA Gatekeeper (Open Policy Agent)** là admission controller dùng ngôn ngữ Rego để viết policy linh hoạt. Gatekeeper từ chối request vi phạm policy trước khi tạo resource.

### Cài Đặt Gatekeeper

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
kubectl get pods -n gatekeeper-system
```

### Policy: Chỉ Cho Phép Pull Từ Registry Tin Cậy

```yaml
# ConstraintTemplate
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: allowedregistries
spec:
  crd:
    spec:
      names:
        kind: AllowedRegistries
      validation:
        openAPIV3Schema:
          properties:
            allowedRegistries:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package allowedregistries

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          not starts_with_allowed_registry(image)
          msg := sprintf("Image '%v' không được phép — chỉ dùng registry tin cậy", [image])
        }

        starts_with_allowed_registry(image) {
          registry := input.parameters.allowedRegistries[_]
          startswith(image, registry)
        }

---
# Constraint — áp dụng policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRegistries
metadata:
  name: allowed-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "gatekeeper-system"]
  parameters:
    allowedRegistries:
      - "123456789.dkr.ecr.us-east-1.amazonaws.com/"
      - "gcr.io/my-project/"
      - "registry.internal.example.com/"
```

### Policy: Không Cho Dùng Tag `latest`

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: nolatesttag
spec:
  crd:
    spec:
      names:
        kind: NoLatestTag
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package nolatesttag

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          endswith(container.image, ":latest")
          msg := sprintf("Container '%v' dùng tag 'latest' — phải pin version cụ thể", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not contains(container.image, ":")  # không có tag = mặc định latest
          msg := sprintf("Container '%v' không có tag — phải pin version", [container.name])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: NoLatestTag
metadata:
  name: no-latest-tag
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

---

## Kyverno — Policy Engine Đơn Giản Hơn

**Kyverno** là policy engine viết cho Kubernetes — policy dùng YAML thay vì Rego, dễ đọc và dễ viết hơn OPA Gatekeeper.

### Cài Đặt Kyverno

```bash
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
kubectl get pods -n kyverno
```

### Policy: Bắt Buộc Image Tag

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-tag
spec:
  validationFailureAction: enforce    # hoặc audit
  rules:
    - name: check-image-tag
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Image phải có tag cụ thể — không dùng latest hoặc không có tag"
        pattern:
          spec:
            containers:
              - name: "*"
                image: "*:?*"          # có tag, không rỗng
```

### Policy: Mutate — Tự Động Thêm securityContext

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-security-context
spec:
  rules:
    - name: add-security-defaults
      match:
        resources:
          kinds:
            - Pod
          namespaces:
            - production
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (name): "*"
                securityContext:
                  +(runAsNonRoot): true
                  +(allowPrivilegeEscalation): false
                  +(readOnlyRootFilesystem): true
```

### Kyverno vs OPA Gatekeeper

| Tiêu Chí | OPA Gatekeeper | Kyverno |
| -------- | -------------- | ------- |
| **Ngôn ngữ policy** | Rego (học riêng) | YAML (K8s native) |
| **Độ linh hoạt** | Rất cao | Cao |
| **Độ khó học** | Cao | Thấp |
| **Mutate policy** | Không hỗ trợ tốt | ✅ Mạnh |
| **Generate resource** | Không | ✅ Có |
| **Community** | Lớn, CNCF graduated | Lớn, CNCF graduated |
| **Khuyến nghị** | Policy phức tạp | Policy đơn giản/vừa |

---

## Image Signing với Cosign

**Cosign** (từ Sigstore project) ký và xác minh image — đảm bảo image trong registry không bị tamper (chỉnh sửa trái phép) sau khi build.

### Ký Image Với Cosign

```bash
# Cài cosign
brew install cosign
# hoặc
go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Tạo key pair
cosign generate-key-pair
# Tạo: cosign.key (private), cosign.pub (public)

# Ký image sau khi push
docker push myregistry/myapp:v1.0
cosign sign --key cosign.key myregistry/myapp:v1.0@sha256:abc123...

# Ký không cần key — dùng Sigstore keyless (OIDC-based)
# Phù hợp cho CI/CD với GitHub Actions, GitLab CI
COSIGN_EXPERIMENTAL=1 cosign sign myregistry/myapp:v1.0
```

### Verify Image Signature

```bash
# Verify bằng public key
cosign verify --key cosign.pub myregistry/myapp:v1.0

# Verify keyless (Sigstore)
COSIGN_EXPERIMENTAL=1 cosign verify \
  --certificate-identity=https://github.com/myorg/myrepo/.github/workflows/release.yml@refs/heads/main \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  myregistry/myapp:v1.0
```

### Tích Hợp Cosign Vào GitHub Actions

```yaml
# .github/workflows/release.yml
name: Build and Sign

on:
  push:
    tags: ['v*']

jobs:
  build-sign:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # cần cho keyless signing
      packages: write

    steps:
      - uses: actions/checkout@v3

      - name: Build và push image
        id: build
        run: |
          docker build -t ghcr.io/myorg/myapp:${{ github.ref_name }} .
          docker push ghcr.io/myorg/myapp:${{ github.ref_name }}
          echo "digest=$(docker inspect --format='{{index .RepoDigests 0}}' ghcr.io/myorg/myapp:${{ github.ref_name }} | cut -d@ -f2)" >> $GITHUB_OUTPUT

      - name: Ký image với Cosign (keyless)
        uses: sigstore/cosign-installer@main

      - run: |
          cosign sign \
            --yes \
            ghcr.io/myorg/myapp:${{ github.ref_name }}@${{ steps.build.outputs.digest }}
```

### Kyverno Policy Verify Signature

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: enforce
  rules:
    - name: verify-cosign-signature
      match:
        resources:
          kinds:
            - Pod
      verifyImages:
        - imageReferences:
            - "myregistry/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                      -----END PUBLIC KEY-----
```

---

## Best Practice Image Security

### Dockerfile Best Practice

```dockerfile
# Dùng specific version tag — không dùng latest
FROM python:3.11.5-slim-bookworm

# Cài package cần thiết và clean cache ngay trong một layer
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    curl=7.88.1-10+deb12u4 \
  && rm -rf /var/lib/apt/lists/*

# Tạo non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup -u 1000 appuser

WORKDIR /app

# Copy và cài dependencies trước (cache layer)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY --chown=appuser:appgroup . .

# Chuyển sang non-root user trước khi chạy
USER appuser

# Expose port > 1024 (non-root không cần NET_BIND_SERVICE)
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Dùng Distroless Image

```dockerfile
# Multi-stage build với distroless runtime
FROM golang:1.21 AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app .

# Distroless: không có shell, không có package manager, ít CVE nhất
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /build/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

### Checklist Image Security

- [ ] Không dùng tag `latest` — pin digest hoặc semantic version
- [ ] Chạy container với non-root user
- [ ] Dùng distroless hoặc alpine image để giảm attack surface
- [ ] Multi-stage build: không copy dev dependencies vào production image
- [ ] Không hardcode secret, credential, hoặc private key trong Dockerfile/image
- [ ] Scan image trong CI pipeline — fail build nếu có Critical CVE
- [ ] Push lên private registry — không dùng public image không kiểm soát
- [ ] Ký image với Cosign sau khi build
- [ ] Cài Trivy Operator để scan image đang chạy trong cluster
- [ ] OPA Gatekeeper/Kyverno: từ chối image từ registry không tin cậy

---

## Câu Hỏi Phỏng Vấn

**Supply chain attack là gì? Kubernetes bảo vệ thế nào?**

> Supply chain attack (tấn công chuỗi cung ứng) nhắm vào quá trình build/distribute image thay vì tấn công trực tiếp vào ứng dụng. Ví dụ: attacker inject malware vào base image trên DockerHub, CI/CD bị compromise để thêm backdoor vào image. Kubernetes bảo vệ qua: (1) Chỉ dùng private registry đã kiểm soát; (2) Image signing với Cosign — xác minh image không bị sửa sau khi build; (3) Scan mọi image với Trivy trước khi deploy; (4) OPA Gatekeeper chỉ cho pull từ registry tin cậy; (5) Pin image digest thay vì tag (digest là hash bất biến).

**Làm thế nào để tích hợp image scanning vào quy trình CI/CD?**

> Flow chuẩn: (1) Developer push code, CI build Docker image; (2) Trivy scan image — nếu có CVE Critical/High, fail pipeline ngay (không push image); (3) Nếu pass, sign image với Cosign; (4) Push vào private registry; (5) Deploy lên cluster qua GitOps; (6) Gatekeeper/Kyverno verify signature trước khi cho Pod chạy. Ngoài ra cài Trivy Operator trong cluster để continuous scan image đang chạy — nhắc alert khi image mới có CVE mới được publish.

**OPA Gatekeeper và PSA (Pod Security Admission) khác nhau thế nào?**

> PSA là admission controller tích hợp sẵn — đơn giản, chỉ có 3 cấp cố định (Privileged/Baseline/Restricted), áp dụng qua label namespace, không thể tùy chỉnh. OPA Gatekeeper là policy engine linh hoạt dùng Rego: có thể viết bất kỳ policy nào (image registry, naming convention, label requirement, resource quota), cả validate và audit, cung cấp error message tùy chỉnh. Trong thực tế thường dùng cả hai: PSA làm baseline security floor, Gatekeeper làm organizational policy layer trên nền đó.

**Tại sao không được dùng tag `latest` trong production?**

> Tag `latest` là mutable (có thể thay đổi) — hôm nay `nginx:latest` là v1.25, tuần sau có thể là v1.26 với breaking change. Vấn đề: (1) Không reproducible — cùng manifest, deploy khác thời điểm cho kết quả khác nhau; (2) Không audit được — không biết version nào đang chạy; (3) Auto-update không kiểm soát — image mới có thể có breaking change hoặc CVE mới; (4) Khó rollback khi có sự cố. Thay vào đó dùng semantic version (`nginx:1.25.3`) hoặc tốt hơn là pin digest (`nginx@sha256:abc123...` — bất biến tuyệt đối).
