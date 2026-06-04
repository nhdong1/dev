# CloudHSM — Hardware Security Module

> AWS CloudHSM (Hardware Security Module — Module Bảo Mật Phần Cứng) là dịch vụ cung cấp thiết bị HSM vật lý dành riêng (dedicated) trong cloud của AWS, đạt tiêu chuẩn FIPS 140-2 Level 3. Khác với KMS (chia sẻ cơ sở hạ tầng), CloudHSM cho bạn toàn quyền kiểm soát hardware HSM riêng — AWS không có khả năng truy cập khóa của bạn.

---

## 📚 Mục Lục

1. [CloudHSM Là Gì?](#cloudhsm-là-gì)
2. [FIPS 140-2 — Tiêu Chuẩn Bảo Mật Phần Cứng](#fips-140-2)
3. [Kiến Trúc CloudHSM](#kiến-trúc)
4. [CloudHSM Cluster (Cụm HSM)](#cluster)
5. [Custom Key Store — Tích Hợp Với KMS](#custom-key-store)
6. [Trường Hợp Sử Dụng](#use-cases)
7. [CloudHSM vs KMS](#so-sánh)
8. [Quản Lý Users Trong HSM](#users-hsm)
9. [Thực Hành](#thực-hành)
10. [Chi Phí](#chi-phí)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CloudHSM Là Gì?

### Tổng Quan

CloudHSM là thiết bị **hardware HSM vật lý** được đặt trong data center của AWS, nhưng được phân bổ riêng cho bạn (không chia sẻ với khách hàng khác).

```
AWS KMS:
  └── Software-based key management
  └── Chia sẻ infrastructure (multi-tenant)
  └── AWS có thể access nếu cần (key escrow)
  └── FIPS 140-2 Level 2
  └── Bạn quản lý: key policy, rotation, usage

AWS CloudHSM:
  └── Hardware HSM vật lý (dedicated)
  └── Không chia sẻ với ai (single-tenant)
  └── AWS KHÔNG thể access keys của bạn (ever)
  └── FIPS 140-2 Level 3
  └── Bạn quản lý: TOÀN BỘ (key, user, certificate, backup)
```

### Mô Hình Trách Nhiệm

```
AWS chịu trách nhiệm:
  ├── Hardware provisioning và thay thế khi hỏng
  ├── Network connectivity vào CloudHSM
  ├── Physical security của data center
  └── Firmware updates (sau khi bạn cho phép)

Bạn chịu trách nhiệm:
  ├── Quản lý HSM users (Crypto Officer, Crypto User)
  ├── Tạo, backup, restore keys
  ├── HSM cluster configuration
  ├── Client software trên EC2/on-premises
  └── Nếu mất password admin → AWS KHÔNG THỂ GIÚP
```

---

## FIPS 140-2

### FIPS (Federal Information Processing Standard — Tiêu Chuẩn Xử Lý Thông Tin Liên Bang)

FIPS 140-2 là tiêu chuẩn của NIST (National Institute of Standards and Technology — Viện Tiêu Chuẩn và Công Nghệ Quốc Gia Hoa Kỳ) quy định yêu cầu bảo mật cho các module mã hóa.

### Bốn Cấp Độ FIPS 140-2

| Level | Yêu Cầu | Ứng Dụng |
|---|---|---|
| **Level 1** | Thuật toán mã hóa đúng chuẩn | Software encryption (tối thiểu) |
| **Level 2** | Tamper-evident seals (niêm phong chống giả mạo), role-based auth | AWS KMS, đa số HSM cloud |
| **Level 3** | Tamper-resistant enclosure (vỏ bọc chống giả mạo), identity-based auth, zeroization | **AWS CloudHSM** |
| **Level 4** | Bảo vệ hoàn toàn mọi tấn công vật lý, tự xóa key nếu phát hiện xâm nhập | Thiết bị quân sự/chính phủ |

### FIPS 140-2 Level 3 Đặc Điểm

```
Tamper-Resistant Physical Security:
  ├── Vỏ bọc cứng chống xâm nhập vật lý
  ├── Phát hiện và phản hồi khi bị mở hoặc tấn công
  └── Tự động xóa (zeroize) tất cả keys khi phát hiện tấn công

Identity-Based Authentication:
  └── Mỗi user có credential riêng biệt (không shared)

Key Zeroization (Xóa Khóa Khẩn Cấp):
  └── Tất cả keys bị xóa hoàn toàn khi phát hiện tấn công vật lý
```

### Khi Nào FIPS 140-2 Level 3 Bắt Buộc?

```
Compliance frameworks yêu cầu Level 3:
  ├── PCI-DSS (Payment Card Industry — Tiêu Chuẩn Ngành Thẻ Thanh Toán) — Certificate Authority
  ├── HIPAA (Health Insurance Portability and Accountability Act) — PHI encryption
  ├── FedRAMP High — US Government workloads
  ├── Common Criteria EAL4+ — Security certifications
  └── DoD (Department of Defense — Bộ Quốc Phòng Hoa Kỳ) directives
```

---

## Kiến Trúc

### CloudHSM Trong VPC

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Account                              │
│                                                             │
│  ┌─────────────────────────────────────────┐               │
│  │               Your VPC                  │               │
│  │                                         │               │
│  │  ┌──────────────────────────────────┐   │               │
│  │  │     CloudHSM Subnet (private)    │   │               │
│  │  │                                  │   │               │
│  │  │  ┌──────┐  ┌──────┐  ┌──────┐   │   │               │
│  │  │  │ HSM  │  │ HSM  │  │ HSM  │   │   │               │
│  │  │  │ AZ-a │  │ AZ-b │  │ AZ-c │   │   │               │
│  │  │  └──────┘  └──────┘  └──────┘   │   │               │
│  │  │     CloudHSM Cluster (3 HSMs)    │   │               │
│  │  └──────────────────────────────────┘   │               │
│  │                   │ (network ENI)        │               │
│  │  ┌────────────────┴─────────────────┐   │               │
│  │  │    EC2 Application Servers       │   │               │
│  │  │  + CloudHSM Client SDK installed │   │               │
│  │  └──────────────────────────────────┘   │               │
│  └─────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Kết Nối Network

- CloudHSM tạo **Elastic Network Interface (ENI — Giao Diện Mạng Đàn Hồi)** trong subnet của bạn
- Tất cả communication qua ENI (không qua public internet)
- Port: TCP 2223-2225 (PKCS#11/JCE/CNG), TCP 2224 (management)

---

## Cluster

### High Availability (Tính Sẵn Sàng Cao) Với CloudHSM Cluster

```
CloudHSM Cluster = Nhóm 2+ HSMs trong cùng VPC

Lợi ích:
  ├── Tự động đồng bộ keys giữa các HSM (real-time)
  ├── Load balancing requests giữa HSMs
  ├── Nếu 1 HSM fail → cluster tiếp tục hoạt động
  └── Khuyến nghị: ít nhất 2 HSM ở 2 Availability Zones khác nhau

Minimum cho production: 2 HSMs (khác AZ)
Recommended:           3 HSMs (mỗi HSM một AZ khác nhau)
```

### Khởi Tạo Cluster

```bash
# Bước 1: Tạo CloudHSM cluster
aws cloudhsmv2 create-cluster \
  --hsm-type hsm1.medium \
  --subnet-ids subnet-abc123 subnet-def456 subnet-ghi789

# Bước 2: Tạo HSM đầu tiên trong cluster
aws cloudhsmv2 create-hsm \
  --cluster-id cluster-abc123 \
  --availability-zone us-east-1a

# Bước 3: Download CSR (Certificate Signing Request) từ cluster
aws cloudhsmv2 describe-clusters \
  --filters clusterIds=cluster-abc123 \
  --query 'Clusters[0].Certificates.ClusterCsr' \
  --output text > cluster.csr

# Bước 4: Sign CSR bằng CA của bạn (chứng minh ownership)
# (dùng openssl, internal CA, etc.)
openssl x509 -req -in cluster.csr \
  -CA customerCA.crt -CAkey customerCA.key \
  -CAcreateserial -days 3652 \
  -out customerHsmCertificate.crt

# Bước 5: Initialize cluster với signed certificate
aws cloudhsmv2 initialize-cluster \
  --cluster-id cluster-abc123 \
  --signed-cert file://customerHsmCertificate.crt \
  --trust-anchor file://customerCA.crt
```

---

## Custom Key Store

### Tích Hợp CloudHSM Với KMS

Custom Key Store (Kho Khóa Tùy Chỉnh) cho phép bạn dùng **KMS API** nhưng keys được lưu trữ trong **CloudHSM cluster** của bạn — không phải trong KMS infrastructure.

```
Bình thường (KMS):
  KMS API → KMS managed HSM (AWS owned) → Key material

Custom Key Store:
  KMS API → Your CloudHSM Cluster → Key material (bạn kiểm soát)
```

### Lợi Ích Custom Key Store

```
1. Dùng KMS API quen thuộc (không cần thay đổi code)
2. Keys thực sự nằm trong HSM của bạn (FIPS 140-2 Level 3)
3. AWS không thể access key material
4. Tích hợp với tất cả dịch vụ AWS hỗ trợ KMS (S3, RDS, EBS...)
5. Có thể disconnect custom key store → KMS không thể dùng key nữa
```

### Tạo Custom Key Store

```bash
# Bước 1: Tạo kmsuser trong CloudHSM cluster (qua CloudHSM CLI)
# CloudHSM CLI:
createUser CO kmsuser
listUsers

# Bước 2: Tạo custom key store trong KMS
aws kms create-custom-key-store \
  --custom-key-store-name "prod-hsm-keystore" \
  --cloud-hsm-cluster-id cluster-abc123 \
  --trust-anchor-certificate file://customerCA.crt \
  --key-store-password "kmsuser_password"

# Bước 3: Connect custom key store
aws kms connect-custom-key-store \
  --custom-key-store-id cks-abc123

# Bước 4: Tạo CMK trong custom key store
aws kms create-key \
  --origin EXTERNAL \
  --custom-key-store-id cks-abc123 \
  --description "HSM-backed encryption key"
```

### Disconnect Custom Key Store (Khẩn Cấp)

```bash
# Khi cần ngăn mọi crypto operations ngay lập tức:
aws kms disconnect-custom-key-store \
  --custom-key-store-id cks-abc123

# Sau khi disconnect:
# - Mọi KMS operations với keys trong store này → thất bại
# - Key không bị xóa, chỉ không thể dùng
# - Dữ liệu đã mã hóa không thể giải mã cho đến khi reconnect
```

---

## Use Cases

### 1. Payment Processing (Xử Lý Thanh Toán)

```
PCI-DSS yêu cầu:
  ├── HSM để bảo vệ PIN encryption keys
  ├── Certificate Authority keys phải ở FIPS Level 3
  └── Key ceremony với dual control

Giải pháp:
  CloudHSM Cluster → Custom Key Store → KMS
  RSA keys cho payment gateway certificates
  AES keys cho cardholder data encryption
```

### 2. Digital Certificate Authority (Tổ Chức Cấp Chứng Chỉ Số)

```
Yêu cầu:
  CA private key phải được bảo vệ ở hardware level
  Mọi certificate signing phải qua HSM

Giải pháp:
  CloudHSM → lưu CA private key
  Ứng dụng → gọi PKCS#11 API để sign certificates
  CA key không bao giờ rời khỏi HSM
```

### 3. Database Transparent Encryption

```
Oracle TDE (Transparent Data Encryption — Mã Hóa Dữ Liệu Trong Suốt):
  CloudHSM cung cấp keystore cho Oracle TDE
  Master encryption key trong HSM
  Database tự động mã hóa/giải mã

SQL Server EKM (Extensible Key Management — Quản Lý Khóa Mở Rộng):
  Tương tự — HSM là external key provider
```

### 4. Document Signing (Ký Tài Liệu)

```
Yêu cầu pháp lý (eIDAS, eSign Act):
  Private key cho chữ ký điện tử phải trong HSM

CloudHSM:
  Lưu signing key
  Application gọi sign() qua CloudHSM SDK
  Key không bao giờ export được
```

---

## So Sánh

### CloudHSM vs KMS

| Tiêu Chí | AWS KMS | AWS CloudHSM |
|---|---|---|
| **Hardware** | Shared HSM infrastructure | Dedicated HSM (riêng) |
| **FIPS Level** | 140-2 Level 2 | 140-2 Level 3 |
| **AWS access keys** | AWS có key escrow | AWS KHÔNG có quyền |
| **API** | AWS KMS API (chuẩn) | PKCS#11, JCE, CNG, OpenSSL |
| **Tích hợp AWS** | Native (S3, RDS, EBS...) | Qua Custom Key Store |
| **Chi phí** | $1/key/tháng + API calls | ~$1.45/HSM/giờ (~$1,050/tháng) |
| **HA (High Availability)** | AWS tự động quản lý | Bạn tự setup cluster |
| **Backup/Restore** | AWS tự động | Bạn tự quản lý |
| **Setup complexity** | Dễ (vài phút) | Phức tạp (vài giờ-ngày) |
| **Operational overhead** | Thấp | Cao |
| **Algorithm support** | Hạn chế (AES, RSA, ECC) | Rộng hơn (DH, custom) |
| **Audit** | CloudTrail | CloudTrail + HSM audit logs |

---

## Users Trong HSM

### Ba Loại User Trong CloudHSM

```
1. Precrypto Officer (PRECO)
   ├── Chỉ tồn tại khi cluster mới (chưa initialized)
   ├── Dùng để tạo CO đầu tiên và đặt password
   └── Tự động mất sau khi initialized

2. Crypto Officer (CO) — Quản Trị Viên Mã Hóa
   ├── Tạo/xóa Crypto Users
   ├── Thay đổi password của users
   ├── Không thể thực hiện crypto operations
   └── Giống như "key administrator" nhưng cho HSM

3. Crypto User (CU) — Người Dùng Mã Hóa
   ├── Thực hiện các crypto operations (encrypt, decrypt, sign...)
   ├── Tạo và quản lý keys trong HSM
   ├── Không thể tạo/xóa users khác
   └── Đây là user dùng trong ứng dụng
```

```bash
# Kết nối CloudHSM CLI
/opt/cloudhsm/bin/cloudhsm-cli

# Tạo Crypto Officer (dùng PRECO lần đầu)
cloudhsm-cli> user create --role crypto-officer --username admin

# Tạo Crypto User cho ứng dụng
cloudhsm-cli> user create --role crypto-user --username app-user

# Xem danh sách users
cloudhsm-cli> user list
```

---

## Thực Hành

### Ví Dụ PKCS#11 (Python) Với CloudHSM

```python
import PyKCS11

# Kết nối CloudHSM qua PKCS#11 library
lib = PyKCS11.PyKCS11Lib()
lib.load('/opt/cloudhsm/lib/libcloudhsm_pkcs11.so')

# Tìm slot
slots = lib.getSlotList(tokenPresent=True)
session = lib.openSession(slots[0])

# Login với Crypto User credentials
session.login("app-user:MyPassword123!")

# Tạo AES-256 key trong HSM
aes_key_template = [
    (PyKCS11.CKA_CLASS, PyKCS11.CKO_SECRET_KEY),
    (PyKCS11.CKA_KEY_TYPE, PyKCS11.CKK_AES),
    (PyKCS11.CKA_VALUE_LEN, 32),  # 256 bits
    (PyKCS11.CKA_LABEL, b"prod-aes-key"),
    (PyKCS11.CKA_TOKEN, PyKCS11.CK_TRUE),     # lưu persistent trong HSM
    (PyKCS11.CKA_SENSITIVE, PyKCS11.CK_TRUE), # không thể export plaintext
    (PyKCS11.CKA_ENCRYPT, PyKCS11.CK_TRUE),
    (PyKCS11.CKA_DECRYPT, PyKCS11.CK_TRUE),
]

aes_key = session.generateKey(aes_key_template)

# Mã hóa với AES-CBC
iv = [0] * 16  # Dùng random IV trong production
mechanism = PyKCS11.Mechanism(PyKCS11.CKM_AES_CBC_PAD, iv)
ciphertext = session.encrypt(aes_key, b"Sensitive data here", mechanism)

# Giải mã
plaintext = session.decrypt(aes_key, ciphertext, mechanism)

session.logout()
session.closeSession()
```

---

## Chi Phí

### Cấu Trúc Giá

```
CloudHSM:
  ├── $1.45/HSM/giờ
  ├── 1 HSM/tháng: ~$1,044 (730 giờ × $1.45)
  └── Cluster production (3 HSMs): ~$3,132/tháng

So sánh với KMS:
  ├── $1/key/tháng
  ├── $0.03/10,000 API calls
  └── 100 keys + 1M calls/tháng: ~$103

CloudHSM phù hợp khi:
  ├── Compliance bắt buộc FIPS Level 3
  ├── Cần dedicated hardware (single-tenant)
  ├── Nhiều crypto operations/giây → cost-effective
  └── Custom algorithms không có trong KMS
```

---

## Câu Hỏi Phỏng Vấn

**Q: Khi nào nên chọn CloudHSM thay vì KMS?**

> Chọn CloudHSM khi: (1) compliance yêu cầu FIPS 140-2 Level 3 (PCI-DSS CA, FedRAMP High, DoD), (2) cần dedicated hardware — không chấp nhận shared infrastructure, (3) yêu cầu AWS không có khả năng access key material về mặt kỹ thuật, (4) cần custom PKCS#11 operations mà KMS không hỗ trợ (Diffie-Hellman, custom curves), (5) database TDE với Oracle/SQL Server EKM. Trong hầu hết trường hợp thông thường, KMS đủ dùng và dễ quản lý hơn nhiều.

**Q: CloudHSM Custom Key Store là gì và lợi ích của nó?**

> Custom Key Store cho phép tạo CMK trong KMS nhưng key material được lưu trong CloudHSM cluster của bạn, không phải trong KMS infrastructure. Lợi ích: vẫn dùng KMS API (không cần thay đổi code), nhưng keys ở FIPS Level 3 hardware; AWS không thể access; có thể disconnect store bất cứ lúc nào để ngăn mọi crypto operations. Nhược điểm: phải manage CloudHSM cluster, phức tạp hơn, đắt hơn đáng kể.

**Q: FIPS 140-2 Level 2 và Level 3 khác nhau gì?**

> Level 2 yêu cầu tamper-evident seals — bạn có thể thấy nếu ai đã mở thiết bị, và role-based authentication. Level 3 thêm tamper-resistant enclosure (vỏ cứng chống phá), identity-based authentication (mỗi user có identity riêng), và quan trọng nhất là key zeroization tự động — nếu phát hiện tấn công vật lý, HSM tự xóa toàn bộ key material ngay lập tức. Điều này đảm bảo keys không bao giờ bị lấy cắp dù thiết bị bị chiếm vật lý.

---

## Tóm Tắt

```
CloudHSM — Điểm Chính:

Là gì:   Dedicated hardware HSM, FIPS 140-2 Level 3
Khác KMS: AWS không access được; single-tenant; customer quản lý toàn bộ
Khi dùng: FIPS L3 compliance, dedicated hardware, custom PKCS#11
Setup:   Cluster (≥2 HSMs khác AZ) → initialize → connect
Integrate: Custom Key Store → dùng KMS API với HSM-backed keys
API:     PKCS#11, JCE, CNG, OpenSSL Dynamic Engine
Users:   Crypto Officer (admin) + Crypto User (app)
Giá:     ~$1,044/HSM/tháng — dùng khi thực sự cần thiết
```

---

**Trước Đó:** [4-kms-grants.md](4-kms-grants.md)
**Tiếp Theo:** [6-s3-encryption-options.md](6-s3-encryption-options.md) — S3 Encryption Options

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
