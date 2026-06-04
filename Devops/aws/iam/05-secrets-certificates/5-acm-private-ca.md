# ACM Private CA — PKI Nội Bộ Trên AWS

> **ACM Private CA** (AWS Private Certificate Authority — Tổ Chức Phát Hành Chứng Chỉ Riêng Tư AWS) cho phép xây dựng PKI (Public Key Infrastructure — Hạ Tầng Khóa Công Khai) nội bộ hoàn chỉnh trên AWS, cấp phát chứng chỉ cho internal services, microservices, IoT devices và mTLS (mutual TLS — TLS Hai Chiều).

---

## 📚 Mục Lục

1. [Tại Sao Cần Private CA?](#tại-sao-cần-private-ca)
2. [PKI Hierarchy — Phân Cấp PKI](#pki-hierarchy--phân-cấp-pki)
3. [Tạo Và Cấu Hình Private CA](#tạo-và-cấu-hình-private-ca)
4. [Cấp Phát Chứng Chỉ](#cấp-phát-chứng-chỉ)
5. [mTLS — Xác Thực Hai Chiều](#mtls--xác-thực-hai-chiều)
6. [Revocation — Thu Hồi Chứng Chỉ](#revocation--thu-hồi-chứng-chỉ)
7. [Short-Lived Certificates — Chứng Chỉ Ngắn Hạn](#short-lived-certificates--chứng-chỉ-ngắn-hạn)
8. [CloudHSM Backend](#cloudhsm-backend)
9. [Cross-Account Certificate Issuance](#cross-account-certificate-issuance)
10. [Chi Phí](#chi-phí)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Private CA?

### Giới Hạn Của ACM Public Certificate

```
ACM Public Certificate KHÔNG thể:
├── Cấp cert cho internal hostnames (app.internal, service.cluster.local)
├── Export private key (không dùng được trực tiếp trên EC2/containers)
├── Cấp client certificate (cho mTLS)
├── Cấp code signing certificate
├── Cấp cert cho on-premises servers
└── Cấp cert cho IoT devices

ACM Private CA CÓ THỂ:
├── ✅ Cấp cert cho bất kỳ hostname nội bộ nào
├── ✅ Export private key (cho EC2, containers, on-prem)
├── ✅ Cấp client certificate cho mTLS
├── ✅ Cấp cert cho IoT devices
├── ✅ Tùy chỉnh X.509 extensions
└── ✅ Tích hợp với CloudHSM cho FIPS 140-2 Level 3
```

### Khi Nào Cần Private CA?

| Tình Huống | Cần Private CA? |
|---|---|
| Website HTTPS công khai | ❌ Dùng ACM Public |
| API Gateway HTTPS | ❌ Dùng ACM Public |
| mTLS giữa microservices | ✅ |
| TLS cho internal APIs (`.internal` domain) | ✅ |
| EC2 instance cần cert riêng | ✅ |
| IoT device authentication | ✅ |
| On-premises server TLS | ✅ |
| VPN client certificate | ✅ |
| Code signing (jar, exe) | ✅ |
| Zero Trust network (xác thực mọi service) | ✅ |

---

## PKI Hierarchy — Phân Cấp PKI

### Lý Thuyết

```
PKI Trust Chain (Chuỗi Tin Cậy PKI):

Root CA (Tổ Chức Phát Hành Gốc)
    └── Subordinate CA / Intermediate CA (Tổ Chức Phát Hành Trung Gian)
        ├── Subordinate CA 2 (nếu cần thêm cấp)
        └── End-entity Certificates (Chứng Chỉ Thực Thể Cuối)
            ├── Server certificates (chứng chỉ máy chủ)
            ├── Client certificates (chứng chỉ khách hàng)
            └── Code signing certificates
```

### Kiến Trúc Khuyến Nghị Trên AWS

```
Option 1: ACM Private CA Hierarchy Hoàn Toàn

ACM Private CA — Root CA (self-signed, offline nếu cần)
    └── ACM Private CA — Subordinate CA (issuing CA)
        ├── Server certs cho microservices
        ├── Client certs cho mTLS
        └── Device certs cho IoT

Option 2: External Root CA + ACM Subordinate

Your existing on-prem Root CA
    └── ACM Private CA — Subordinate CA (cross-signed)
        └── ACM cấp certs cho AWS workloads

Lợi ích Option 2:
- Dùng được Root CA hiện tại (không phải replace PKI cũ)
- Devices cũ vẫn tin tưởng chain (đã có Root CA cũ)
```

### Best Practice: Root CA Offline (Ngoại Tuyến)

```
Trong enterprise PKI:
- Root CA KHÔNG BAO GIỜ được dùng trực tiếp để cấp cert
- Root CA chỉ sign Subordinate CA certificate
- Root CA "offline" (disabled, không accessible) để bảo vệ

Trên ACM:
- Tạo Root CA trong ACM
- Dùng Root CA để sign Subordinate CA
- Disable Root CA (không xóa) sau khi sign
- Chỉ dùng Subordinate CA để cấp certs
- Chi phí: Root CA disabled vẫn tốn $400/tháng — xem xét on-prem Root CA
```

---

## Tạo Và Cấu Hình Private CA

### Tạo Root CA

```bash
aws acm-pca create-certificate-authority \
  --certificate-authority-configuration '{
    "KeyAlgorithm": "RSA_2048",
    "SigningAlgorithm": "SHA256WITHRSA",
    "Subject": {
      "Country": "VN",
      "Organization": "MyCompany",
      "OrganizationalUnit": "IT Security",
      "State": "Ho Chi Minh",
      "Locality": "Ho Chi Minh City",
      "CommonName": "MyCompany Root CA"
    }
  }' \
  --certificate-authority-type ROOT \
  --revocation-configuration '{
    "CrlConfiguration": {
      "Enabled": true,
      "ExpirationInDays": 7,
      "S3BucketName": "mycompany-ca-crl",
      "S3ObjectAcl": "BUCKET_OWNER_FULL_CONTROL"
    },
    "OcspConfiguration": {
      "Enabled": true
    }
  }' \
  --tags '[{"Key": "Environment", "Value": "prod"}]'

# Lưu lại CertificateAuthorityArn từ output
ROOT_CA_ARN="arn:aws:acm-pca:ap-southeast-1:123456789012:certificate-authority/abc-123"
```

### Tự Ký (Self-Sign) Root CA Certificate

```bash
# 1. Lấy CSR (Certificate Signing Request) của Root CA
aws acm-pca get-certificate-authority-csr \
  --certificate-authority-arn "$ROOT_CA_ARN" \
  --output text > root-ca.csr

# 2. Issue self-signed certificate cho Root CA
ROOT_CERT_ARN=$(aws acm-pca issue-certificate \
  --certificate-authority-arn "$ROOT_CA_ARN" \
  --csr fileb://root-ca.csr \
  --signing-algorithm SHA256WITHRSA \
  --template-arn arn:aws:acm-pca:::template/RootCACertificate/V1 \
  --validity Value=10,Type=YEARS \
  --query "CertificateArn" --output text)

# 3. Lấy certificate đã được issue
aws acm-pca get-certificate \
  --certificate-authority-arn "$ROOT_CA_ARN" \
  --certificate-arn "$ROOT_CERT_ARN" \
  --output text > root-ca.pem

# 4. Import certificate vào CA
aws acm-pca import-certificate-authority-certificate \
  --certificate-authority-arn "$ROOT_CA_ARN" \
  --certificate fileb://root-ca.pem
```

### Tạo Subordinate CA (Issuing CA)

```bash
# Tạo Subordinate CA
SUB_CA_ARN=$(aws acm-pca create-certificate-authority \
  --certificate-authority-configuration '{
    "KeyAlgorithm": "RSA_2048",
    "SigningAlgorithm": "SHA256WITHRSA",
    "Subject": {
      "Country": "VN",
      "Organization": "MyCompany",
      "OrganizationalUnit": "Engineering",
      "CommonName": "MyCompany Services CA"
    }
  }' \
  --certificate-authority-type SUBORDINATE \
  --query "CertificateAuthorityArn" --output text)

# Lấy CSR của Subordinate CA
aws acm-pca get-certificate-authority-csr \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --output text > sub-ca.csr

# Ký CSR bằng Root CA
SUB_CERT_ARN=$(aws acm-pca issue-certificate \
  --certificate-authority-arn "$ROOT_CA_ARN" \
  --csr fileb://sub-ca.csr \
  --signing-algorithm SHA256WITHRSA \
  --template-arn arn:aws:acm-pca:::template/SubordinateCACertificate_PathLen0/V1 \
  --validity Value=5,Type=YEARS \
  --query "CertificateArn" --output text)

# Lấy cert chain
aws acm-pca get-certificate \
  --certificate-authority-arn "$ROOT_CA_ARN" \
  --certificate-arn "$SUB_CERT_ARN" \
  --output text > sub-ca.pem

# Import vào Subordinate CA
aws acm-pca import-certificate-authority-certificate \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --certificate fileb://sub-ca.pem \
  --certificate-chain fileb://root-ca.pem

echo "Subordinate CA sẵn sàng: $SUB_CA_ARN"
```

### Terraform — Complete Private CA Setup

```hcl
# S3 bucket cho CRL (Certificate Revocation List)
resource "aws_s3_bucket" "crl" {
  bucket = "mycompany-ca-crl-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_policy" "crl" {
  bucket = aws_s3_bucket.crl.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "AllowACMPCA"
      Effect = "Allow"
      Principal = {
        Service = "acm-pca.amazonaws.com"
      }
      Action   = ["s3:PutObject", "s3:PutObjectAcl"]
      Resource = "${aws_s3_bucket.crl.arn}/*"
    }]
  })
}

# Root CA
resource "aws_acmpca_certificate_authority" "root" {
  type = "ROOT"

  certificate_authority_configuration {
    key_algorithm     = "RSA_2048"
    signing_algorithm = "SHA256WITHRSA"

    subject {
      country             = "VN"
      organization        = "MyCompany"
      organizational_unit = "IT Security"
      common_name         = "MyCompany Root CA"
    }
  }

  revocation_configuration {
    crl_configuration {
      enabled            = true
      expiration_in_days = 7
      s3_bucket_name     = aws_s3_bucket.crl.bucket
      s3_object_acl      = "BUCKET_OWNER_FULL_CONTROL"
    }
    ocsp_configuration {
      enabled = true
    }
  }

  tags = {
    Environment = "prod"
    Role        = "root-ca"
  }
}

# Self-sign Root CA
resource "aws_acmpca_certificate" "root" {
  certificate_authority_arn   = aws_acmpca_certificate_authority.root.arn
  certificate_signing_request = aws_acmpca_certificate_authority.root.certificate_signing_request
  signing_algorithm           = "SHA256WITHRSA"
  template_arn                = "arn:aws:acm-pca:::template/RootCACertificate/V1"

  validity {
    type  = "YEARS"
    value = 10
  }
}

resource "aws_acmpca_certificate_authority_certificate" "root" {
  certificate_authority_arn = aws_acmpca_certificate_authority.root.arn
  certificate               = aws_acmpca_certificate.root.certificate
}

# Subordinate CA
resource "aws_acmpca_certificate_authority" "subordinate" {
  type = "SUBORDINATE"

  certificate_authority_configuration {
    key_algorithm     = "RSA_2048"
    signing_algorithm = "SHA256WITHRSA"

    subject {
      country             = "VN"
      organization        = "MyCompany"
      organizational_unit = "Engineering"
      common_name         = "MyCompany Services CA"
    }
  }

  revocation_configuration {
    crl_configuration {
      enabled            = true
      expiration_in_days = 7
      s3_bucket_name     = aws_s3_bucket.crl.bucket
    }
    ocsp_configuration {
      enabled = true
    }
  }
}

# Sign Subordinate CA bằng Root CA
resource "aws_acmpca_certificate" "subordinate" {
  certificate_authority_arn   = aws_acmpca_certificate_authority.root.arn
  certificate_signing_request = aws_acmpca_certificate_authority.subordinate.certificate_signing_request
  signing_algorithm           = "SHA256WITHRSA"
  template_arn                = "arn:aws:acm-pca:::template/SubordinateCACertificate_PathLen0/V1"

  validity {
    type  = "YEARS"
    value = 5
  }
}

resource "aws_acmpca_certificate_authority_certificate" "subordinate" {
  certificate_authority_arn = aws_acmpca_certificate_authority.subordinate.arn
  certificate               = aws_acmpca_certificate.subordinate.certificate
  certificate_chain         = aws_acmpca_certificate.root.certificate_chain
}
```

---

## Cấp Phát Chứng Chỉ

### Issue Certificate Qua ACM (Managed)

Phương pháp này dùng ACM làm trung gian — cert được quản lý như ACM public cert:

```bash
# Yêu cầu private cert qua ACM (không cần CSR thủ công)
aws acm request-certificate \
  --domain-name "api.mycompany.internal" \
  --subject-alternative-names "*.services.mycompany.internal" \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --validation-method DNS

# Cert này sẽ:
# - Tự động gia hạn
# - Gắn được vào ALB, API Gateway
# - Không thể export private key
```

### Issue Certificate Trực Tiếp (Exportable)

```bash
# 1. Tạo private key và CSR
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key \
  -out server.csr \
  -subj "/CN=api.mycompany.internal/O=MyCompany/C=VN"

# 2. Cấp cert từ ACM Private CA
CERT_ARN=$(aws acm-pca issue-certificate \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --csr fileb://server.csr \
  --signing-algorithm SHA256WITHRSA \
  --template-arn arn:aws:acm-pca:::template/EndEntityCertificate/V1 \
  --validity Value=365,Type=DAYS \
  --query "CertificateArn" --output text)

# 3. Đợi cert được issue
aws acm-pca wait certificate-issued \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --certificate-arn "$CERT_ARN"

# 4. Lấy cert
aws acm-pca get-certificate \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --certificate-arn "$CERT_ARN" \
  --query "Certificate" --output text > server.crt

# server.key + server.crt sẵn sàng deploy lên EC2/container
```

### Custom Certificate Templates (Mẫu Chứng Chỉ Tùy Chỉnh)

| Template | Dùng Cho |
|---|---|
| `EndEntityCertificate/V1` | Server cert thông thường |
| `EndEntityClientAuthCertificate/V1` | Client cert cho mTLS |
| `EndEntityServerAuthCertificate/V1` | Server cert chỉ cho TLS server auth |
| `CodeSigningCertificate/V1` | Code signing |
| `OCSPSigningCertificate/V1` | OCSP responder |
| `BlankEndEntityCertificate_APIPassthrough/V1` | Tùy chỉnh hoàn toàn qua API |

### Custom Extensions — X.509 Extensions Tùy Chỉnh

```bash
# Thêm custom OID (Object Identifier) vào cert
aws acm-pca issue-certificate \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --csr fileb://device.csr \
  --signing-algorithm SHA256WITHRSA \
  --template-arn arn:aws:acm-pca:::template/BlankEndEntityCertificate_APIPassthrough/V1 \
  --api-passthrough '{
    "Extensions": {
      "CustomExtensions": [
        {
          "ObjectIdentifier": "1.3.6.1.4.1.12345.1",
          "Value": "dGhpcy1pcy1kZXZpY2UtaWQ=",
          "Critical": false
        }
      ],
      "SubjectAlternativeNames": [
        {
          "DnsName": "device-001.iot.mycompany.internal"
        },
        {
          "UniformResourceIdentifier": "urn:mycompany:device:001"
        }
      ]
    }
  }' \
  --validity Value=365,Type=DAYS
```

---

## mTLS — Xác Thực Hai Chiều

### mTLS Là Gì?

**mTLS** (mutual TLS — TLS Hai Chiều): thông thường TLS chỉ xác thực server. mTLS thêm bước xác thực client — cả hai phải có certificate.

```
TLS thông thường:
  Client ──→ Server (Client kiểm tra cert của Server)

mTLS:
  Client ←→ Server (Cả hai kiểm tra cert của nhau)

Ứng Dụng mTLS:
├── Zero Trust microservices: mỗi service phải có cert
├── API authentication: client có cert thay vì API key
├── IoT device authentication: mỗi device có cert riêng
└── VPN: xác thực user/device bằng cert
```

### mTLS Với ALB (Application Load Balancer)

```bash
# Tạo Trust Store (Kho Tin Cậy) — chứa CA certs mà ALB tin tưởng
TRUST_STORE_ARN=$(aws elbv2 create-trust-store \
  --name "mycompany-mtls-trust-store" \
  --ca-certificates-bundle-s3-bucket "mycompany-ca-bundle" \
  --ca-certificates-bundle-s3-key "ca-bundle.pem" \
  --query "TrustStores[0].TrustStoreArn" --output text)

# Cấu hình ALB listener để yêu cầu client certificate
aws elbv2 modify-listener \
  --listener-arn "arn:..." \
  --mutual-authentication '{
    "Mode": "verify",
    "TrustStoreArn": "'$TRUST_STORE_ARN'",
    "IgnoreClientCertificateExpiry": false
  }'
```

**Chế độ mTLS của ALB:**

| Mode | Hành Vi |
|---|---|
| `off` | TLS thông thường, không cần client cert |
| `passthrough` | Forward client cert lên backend, không verify |
| `verify` | ALB verify client cert, từ chối nếu invalid |

### mTLS Cho Microservices — Envoy/Istio Pattern

```yaml
# Istio PeerAuthentication — bật mTLS cho namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Yêu cầu mTLS cho tất cả services trong namespace
```

```yaml
# Sử dụng cert từ ACM Private CA qua cert-manager + ACM PICA plugin
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: service-a-cert
  namespace: production
spec:
  secretName: service-a-tls
  duration: 24h    # Short-lived certificate
  renewBefore: 8h
  dnsNames:
    - service-a.production.svc.cluster.local
  issuerRef:
    name: aws-pca-issuer
    kind: AWSPCAIssuer
    group: awspca.cert-manager.io
```

### Lấy Client Certificate Info Ở Backend

Khi ALB ở chế độ `passthrough` hoặc `verify`, thông tin client cert được forward qua HTTP headers:

```
X-Amzn-Mtls-Clientcert: <URL-encoded PEM certificate>
X-Amzn-Mtls-Clientcert-Validity: NotBefore=...,NotAfter=...
X-Amzn-Mtls-Clientcert-Subject: CN=device-001,O=MyCompany
X-Amzn-Mtls-Clientcert-Issuer: CN=MyCompany Services CA
X-Amzn-Mtls-Clientcert-Serial: 0x1234567890
X-Amzn-Mtls-Clientcert-Leaf: <URL-encoded PEM leaf cert>
```

```python
# Backend đọc client cert từ ALB headers
from flask import request
import urllib.parse
from cryptography import x509
from cryptography.hazmat.backends import default_backend

@app.route('/api/resource')
def get_resource():
    cert_pem = urllib.parse.unquote(
        request.headers.get('X-Amzn-Mtls-Clientcert', '')
    )
    subject = request.headers.get('X-Amzn-Mtls-Clientcert-Subject', '')

    # Parse CN từ subject để biết device/client là ai
    # subject = "CN=device-001,O=MyCompany,C=VN"
    cn = dict(part.split('=') for part in subject.split(',') if '=' in part).get('CN', '')

    return f"Hello, {cn}!"
```

---

## Revocation — Thu Hồi Chứng Chỉ

### Tại Sao Cần Thu Hồi?

- Device bị đánh cắp, cần vô hiệu hóa cert ngay
- Private key bị lộ
- Employee nghỉ việc, cần thu hồi cert của họ
- Certificate được cấp nhầm

### CRL (Certificate Revocation List — Danh Sách Chứng Chỉ Thu Hồi)

```bash
# Thu hồi một certificate
aws acm-pca revoke-certificate \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --certificate-serial "1234567890abcdef" \
  --revocation-reason KEY_COMPROMISE

# Lý do thu hồi hợp lệ:
# UNSPECIFIED, KEY_COMPROMISE, CA_COMPROMISE, AFFILIATION_CHANGED,
# SUPERSEDED, CESSATION_OF_OPERATION, PRIVILEGE_WITHDRAWN

# CRL được tự động cập nhật trong S3 bucket đã cấu hình
# URL CRL: http://s3.amazonaws.com/mycompany-ca-crl/{CA-ID}.crl
```

### OCSP (Online Certificate Status Protocol — Giao Thức Kiểm Tra Trạng Thái Chứng Chỉ Trực Tuyến)

OCSP cho phép kiểm tra trạng thái cert real-time thay vì download CRL.

```
Luồng OCSP:
Client → OCSP Responder: "cert serial 1234 có còn hiệu lực không?"
OCSP Responder → Client: "GOOD" / "REVOKED" / "UNKNOWN"

OCSP Stapling (Đính Kèm OCSP):
- Server tự hỏi OCSP và đính kết quả vào TLS handshake
- Client không cần hỏi OCSP server riêng
- Nhanh hơn, bảo mật hơn (tránh OCSP privacy leak)
```

ACM Private CA tự động cung cấp OCSP endpoint — bạn không cần quản lý OCSP server.

---

## Short-Lived Certificates — Chứng Chỉ Ngắn Hạn

### Triết Lý: Rotation Qua Expiration

Thay vì dùng cert dài hạn + revocation, modern PKI dùng **short-lived certs**:

```
Short-lived cert (24 giờ):
├── Cert tự hết hạn sau 24 giờ → không cần revoke
├── Mỗi ngày tự động lấy cert mới
├── Nếu device bị compromise → cert hết hạn sau tối đa 24 giờ
└── Không cần CRL/OCSP check

vs Long-lived cert (1 năm):
├── Nếu bị compromise → phải revoke thủ công
├── Client cần check CRL/OCSP
├── CRL có thể outdated (hết hạn mỗi 7 ngày)
└── Rủi ro cao hơn trong thời gian từ lúc compromise đến revoke
```

### Pattern: SPIFFE/SVID (Secure Production Identity Framework For Everyone / SVID — Định Danh Sản Xuất An Toàn)

```
SPIFFE là standard cho workload identity:
- Mỗi workload có URI: spiffe://trust-domain/ns/namespace/sa/service-account
- SVID là chứng chỉ X.509 chứa SPIFFE URI trong SAN

Triển Khai Với ACM Private CA:
1. Workload khởi động → yêu cầu SVID
2. SPIRE Server verify workload identity (qua attestation)
3. SPIRE Server gọi ACM PCA cấp cert 24 giờ
4. Workload dùng cert cho mTLS
5. Sau 24 giờ → cert hết hạn → tự động renew
```

```bash
# Cấp short-lived cert (24 giờ) cho workload
aws acm-pca issue-certificate \
  --certificate-authority-arn "$SUB_CA_ARN" \
  --csr fileb://workload.csr \
  --signing-algorithm SHA256WITHRSA \
  --template-arn arn:aws:acm-pca:::template/EndEntityClientAuthCertificate/V1 \
  --validity Value=24,Type=HOURS \  # ← 24 giờ, không phải ngày/năm
  --api-passthrough '{
    "Extensions": {
      "SubjectAlternativeNames": [
        {
          "UniformResourceIdentifier": "spiffe://mycompany.com/ns/prod/sa/payment-service"
        }
      ]
    }
  }'
```

---

## CloudHSM Backend

ACM Private CA có thể dùng **CloudHSM** (Hardware Security Module — Module Bảo Mật Phần Cứng) để lưu CA private key, đạt **FIPS 140-2 Level 3**.

```
Standard ACM Private CA:
CA private key → lưu trong AWS-managed HSM (FIPS 140-2 Level 3 by default)

Custom CloudHSM:
CA private key → lưu trong CloudHSM cluster của bạn
├── Bạn kiểm soát hoàn toàn HSM hardware
├── Audit log riêng từ HSM
├── Đáp ứng yêu cầu regulatory cao nhất
└── Chi phí: ~$1.60/giờ/HSM + $400/tháng CA

Khi Nào Cần CloudHSM Backend:
- Regulatory yêu cầu customer-controlled HSM (banking, government)
- Audit requirement: biết chính xác ai access key bao giờ
- Key ceremony documentation (nghi thức tạo và lưu CA key)
```

---

## Cross-Account Certificate Issuance

### Chia Sẻ Private CA Qua AWS RAM (Resource Access Manager)

```bash
# Share CA với accounts khác qua AWS RAM
aws ram create-resource-share \
  --name "mycompany-subordinate-ca-share" \
  --resource-arns "$SUB_CA_ARN" \
  --principals "arn:aws:organizations::123456789012:organization/o-abc123" \
  --permission-arns "arn:aws:ram::aws:permission/AWSCertificateManagerPrivateCAAWSRAMPermissionIssueCertificateNoCRL"
```

```
Central Security Account (account 111111111111)
└── ACM Private CA — Subordinate CA

Dev Account (account 222222222222)
└── Application → gọi acm-pca:IssueCertificate vào account 111111111111
    └── Cert được cấp cho domain trong dev account

Prod Account (account 333333333333)
└── Application → gọi acm-pca:IssueCertificate vào account 111111111111
    └── Cert được cấp cho domain trong prod account
```

### IAM Permission Cho Cross-Account

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "acm-pca:IssueCertificate",
      "Resource": "arn:aws:acm-pca:ap-southeast-1:111111111111:certificate-authority/abc",
      "Condition": {
        "StringLike": {
          "acm-pca:TemplateArn": "arn:aws:acm-pca:::template/EndEntityCertificate*"
        }
      }
    }
  ]
}
```

---

## Chi Phí

| Thành Phần | Chi Phí |
|---|---|
| Private CA (mỗi CA/tháng) | $400/tháng |
| Chứng chỉ được cấp phát (2,000 đầu tiên) | $0.75/cert |
| Chứng chỉ vượt quá 2,000/tháng | $0.001/cert |
| CloudHSM backend (nếu dùng) | ~$1.60/giờ/HSM |

**Ví Dụ:**
- 1 Root CA (disabled) + 1 Subordinate CA + 500 certs/tháng = $800 + $375 = **$1,175/tháng**
- Chỉ 1 Subordinate CA + 100 certs/tháng = $400 + $75 = **$475/tháng**

**Tối Ưu Chi Phí:**
- Dùng external Root CA (on-prem) thay vì ACM Root CA (tiết kiệm $400/tháng)
- Short-lived certs: mặc dù cấp nhiều hơn, giá thấp hơn sau 2,000 cert/tháng
- Consolidate: 1 Subordinate CA cho nhiều teams thay vì mỗi team 1 CA

---

## So Sánh ACM Public vs ACM Private CA

| Tiêu Chí | ACM Public | ACM Private CA |
|---|---|---|
| **Chi Phí** | Miễn phí | $400/CA/tháng + $0.75/cert |
| **Trust** | Được browsers tin mặc định | Chỉ được tin nếu install Root CA |
| **Internal domains** | ❌ | ✅ |
| **Export private key** | ❌ | ✅ (nếu dùng issue-certificate trực tiếp) |
| **Client certificates** | ❌ | ✅ |
| **Custom extensions** | ❌ | ✅ |
| **Short-lived (giờ)** | ❌ | ✅ |
| **IoT devices** | ❌ | ✅ |
| **Validity period** | 13 tháng (ACM managed) | Tùy chỉnh (giờ đến năm) |
| **Gia hạn tự động** | ✅ | ✅ (qua ACM) hoặc thủ công |

---

## Câu Hỏi Phỏng Vấn

**Q1: Khi nào cần ACM Private CA thay vì ACM Public Certificate?**

> Dùng ACM Private CA khi: (1) Cần cert cho internal hostnames (`.internal`, `.cluster.local`), (2) Cần client certificates cho mTLS, (3) Cần export private key để dùng trực tiếp trên EC2 hay container, (4) Cần custom X.509 extensions, (5) IoT device authentication. ACM Public chỉ cho public domains và không thể export key.

**Q2: Giải thích mTLS và tại sao microservices cần nó?**

> mTLS (mutual TLS) xác thực cả server VÀ client — mỗi service phải có certificate để giao tiếp. Trong microservices, mTLS đảm bảo: (1) Chỉ authorized services mới giao tiếp được nhau (không chỉ dùng network policy), (2) Ngăn lateral movement nếu một service bị compromise, (3) Không cần API keys hay shared secrets giữa services. Là nền tảng của Zero Trust network architecture.

**Q3: Short-lived certificates khác gì long-lived cert + revocation?**

> Short-lived cert (24-48 giờ) tự hết hạn → không cần CRL/OCSP check. Nếu private key bị lộ, cert vô hiệu sau tối đa 48 giờ mà không cần can thiệp. Long-lived cert cần revoke thủ công + client phải check CRL (có thể không up-to-date). Short-lived cert là standard của Zero Trust: continuous re-authentication thay vì trust once forever.

**Q4: Tại sao Root CA nên "offline"?**

> Root CA private key là "chìa khóa vàng" — nếu bị compromise, toàn bộ PKI hierarchy bị mất tin cậy, phải rebuild từ đầu. Bằng cách để Root CA offline (disabled sau khi sign Subordinate CA), bạn giảm attack surface xuống zero. Nếu Subordinate CA bị compromise, chỉ cần revoke Subordinate CA cert từ Root CA và tạo Subordinate CA mới — Root CA vẫn an toàn.

**Q5: Cross-account certificate issuance hoạt động như thế nào?**

> Central Security Account sở hữu ACM Private CA và share qua AWS RAM. Dev/Prod accounts được grant quyền `acm-pca:IssueCertificate`. Ứng dụng ở Prod account gọi ACM PCA API của Security Account, CA ký cert và trả về. Pattern này: (1) Tập trung PKI management, (2) Mỗi account vẫn tự quản lý cert của mình, (3) Security team kiểm soát CA không thể bị bypass.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
