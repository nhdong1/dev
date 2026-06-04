# KMS Grants — Ủy Quyền Dùng Khóa Tạm Thời

> KMS Grant (Cấp Quyền KMS) là cơ chế ủy quyền linh hoạt cho phép một principal (thường là dịch vụ AWS) sử dụng CMK mà không cần thay đổi key policy. Grants đặc biệt hữu ích khi cần ủy quyền tự động, theo chương trình, hoặc tạm thời — điển hình trong các dịch vụ như Amazon EBS, EKS, và AWS Nitro Enclaves.

---

## 📚 Mục Lục

1. [Grant Là Gì?](#grant-là-gì)
2. [Khi Nào Dùng Grant?](#khi-nào-dùng)
3. [Tạo Grant](#tạo-grant)
4. [Grant Constraints (Ràng Buộc Grant)](#grant-constraints)
5. [Grant Tokens (Token Grant)](#grant-tokens)
6. [Retiring và Revoking Grants](#retiring-revoking)
7. [Dịch Vụ AWS Dùng Grants](#dịch-vụ-dùng-grants)
8. [So Sánh Grant vs Key Policy vs IAM Policy](#so-sánh)
9. [Thực Hành CLI](#thực-hành-cli)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Grant Là Gì?

### Định Nghĩa

Grant là một **token ủy quyền** cấp cho một principal quyền thực hiện một số thao tác cụ thể trên một CMK. Grant được lưu trữ trong KMS và có hiệu lực độc lập với key policy.

```
Grant = {
  KeyId:              CMK bị ủy quyền
  GranteePrincipal:   Ai được nhận ủy quyền (grantee)
  Operations:         Thao tác nào được phép
  Constraints:        Điều kiện bổ sung (tùy chọn)
  RetiringPrincipal:  Ai có thể thu hồi grant (tùy chọn)
  Name:               Tên định danh (tùy chọn, không thay đổi được)
}
```

### Mô Hình Quyết Định Với Grants

```
Request kms:Decrypt → KMS kiểm tra:

1. Key policy DENY? → Từ chối (luôn áp dụng)
2. Key policy ALLOW? → Cho phép
3. IAM policy ALLOW (nếu key policy ủy quyền IAM)? → Cho phép
4. Grant ALLOW? → Cho phép

Grants là lớp thứ 4 — không thay thế key policy mà bổ sung thêm.
```

---

## Khi Nào Dùng?

### Use Cases Phù Hợp

```
1. Dịch vụ AWS cần dùng key tự động
   └── EBS volume encryption, EKS secrets, Redshift
   └── AWS dịch vụ tự tạo grant khi bạn bật encryption

2. Ủy quyền tạm thời cho quy trình cụ thể
   └── Batch job cần mã hóa dữ liệu trong 2 giờ
   └── Lambda function xử lý một tác vụ rồi thu hồi grant

3. Ủy quyền programmatic (theo chương trình)
   └── Ứng dụng tự tạo grant cho microservice mới khởi động
   └── Không cần sửa key policy qua Console

4. Delegation với constraints (ủy quyền có ràng buộc)
   └── "Chỉ được decrypt nếu encryption context đúng"
   └── Grantee không thể tạo grant con với quyền rộng hơn

5. Cho phép Retiring Principal riêng
   └── Grantee là service, retiring principal là orchestrator
   └── Orchestrator thu hồi grant sau khi job xong
```

### Khi KHÔNG Nên Dùng Grant

```
- Cần long-term, stable access → Dùng key policy + IAM policy
- Audit chi tiết theo user/role cụ thể → Key policy rõ ràng hơn
- Cross-account access thường xuyên → Key policy cross-account
- Nhiều người/team cần truy cập → Key policy với IAM delegation
```

---

## Tạo Grant

### Cú Pháp CLI

```bash
aws kms create-grant \
  --key-id alias/prod-app-key \
  --grantee-principal arn:aws:iam::123456789012:role/DataProcessingRole \
  --operations Encrypt Decrypt GenerateDataKey DescribeKey \
  --name "data-processing-grant-2026" \
  --retiring-principal arn:aws:iam::123456789012:role/OrchestratorRole \
  --constraints '{"EncryptionContextSubset": {"environment": "production"}}'
```

Output:
```json
{
  "GrantToken": "AQpAM2RhACFlOWQ...(very long token)...xyz==",
  "GrantId": "0c237476b39f8bc44e45212e08498fbe8f1ca8419e7a0ef200bcee3b79975f9e"
}
```

### Các Operations (Thao Tác) Cho Phép Trong Grant

| Operation | Mô Tả |
|---|---|
| `Decrypt` | Giải mã dữ liệu |
| `Encrypt` | Mã hóa dữ liệu |
| `GenerateDataKey` | Tạo DEK (có plaintext) |
| `GenerateDataKeyWithoutPlaintext` | Tạo Encrypted DEK |
| `ReEncryptFrom` | Giải mã bằng key này (để re-encrypt) |
| `ReEncryptTo` | Mã hóa lại bằng key này |
| `GenerateMac` | Tạo MAC (Message Authentication Code) |
| `VerifyMac` | Xác minh MAC |
| `Sign` | Ký dữ liệu (asymmetric key) |
| `Verify` | Xác minh chữ ký |
| `DescribeKey` | Xem metadata key |
| `CreateGrant` | Tạo grant con (delegation) |
| `RetireGrant` | Thu hồi grant này |

---

## Grant Constraints

### Hai Loại Constraint

#### EncryptionContextEquals (Ngữ Cảnh Phải Khớp Hoàn Toàn)

```bash
# Grantee chỉ được dùng key khi encryption context ĐÚNG CHÍNH XÁC
aws kms create-grant \
  --key-id alias/prod-key \
  --grantee-principal arn:aws:iam::123456789012:role/AppRole \
  --operations Encrypt Decrypt \
  --constraints '{
    "EncryptionContextEquals": {
      "environment": "production",
      "service": "payment-processor"
    }
  }'
```

Mọi encrypt/decrypt phải kèm theo ĐÚNG context này — không thừa, không thiếu.

#### EncryptionContextSubset (Ngữ Cảnh Phải Chứa Subset)

```bash
# Grantee được dùng key khi encryption context CHỨA ÍT NHẤT các keys này
aws kms create-grant \
  --key-id alias/prod-key \
  --grantee-principal arn:aws:iam::123456789012:role/AppRole \
  --operations Decrypt GenerateDataKey \
  --constraints '{
    "EncryptionContextSubset": {
      "environment": "production"
    }
  }'

# Chấp nhận: {"environment": "production", "table": "users"}
# Từ chối:   {"environment": "staging"}
# Từ chối:   {"table": "users"}  (thiếu "environment")
```

### Tại Sao Dùng Constraints?

```
Tình huống không có constraint:
  Role/DataProcessing được grant quyền Decrypt
  → Có thể decrypt BẤT KỲ ciphertext nào dùng key này
  → Kể cả dữ liệu của customer khác!

Với EncryptionContextSubset: {"customerId": "12345"}
  → Chỉ decrypt được ciphertext của customer 12345
  → Dữ liệu các customer khác được bảo vệ
  → Quan trọng cho multi-tenant applications
```

---

## Grant Tokens

### Vấn Đề Eventual Consistency (Nhất Quán Cuối Cùng)

```
Khi tạo grant, KMS cần ~5 giây để propagate (lan truyền) tới
tất cả các KMS nodes. Trong khoảng thời gian này:

request kms:Decrypt ngay sau create-grant → AccessDeniedException!

Nguyên nhân: KMS node xử lý request chưa nhận được thông tin về grant mới
```

### Giải Pháp: Grant Token

Grant Token là chuỗi dài được trả về khi tạo grant. Dùng nó thay cho grant ID trong API calls để **bỏ qua vấn đề eventual consistency**:

```python
import boto3

kms = boto3.client('kms')

# Tạo grant
grant_response = kms.create_grant(
    KeyId='alias/my-key',
    GranteePrincipal='arn:aws:iam::123456789012:role/ProcessingRole',
    Operations=['Decrypt']
)

grant_token = grant_response['GrantToken']

# Dùng ngay lập tức với grant token — không bị eventual consistency
decrypt_response = kms.decrypt(
    CiphertextBlob=encrypted_data,
    GrantTokens=[grant_token]  # Đảm bảo grant được nhận diện ngay
)
```

### Khi Nào Cần Grant Token?

```
CẦN dùng Grant Token:
  ├── Gọi KMS ngay sau khi tạo grant (<10 giây)
  ├── Lambda function mới start với grant vừa tạo
  └── Automation workflow tạo grant rồi dùng ngay

KHÔNG CẦN Grant Token:
  ├── Grant đã tồn tại > 5-10 giây
  └── Dịch vụ AWS thường tự quản lý (EBS, RDS không cần bạn lo)
```

---

## Retiring và Revoking Grants

### Phân Biệt Retire vs Revoke

| | Retire Grant | Revoke Grant |
|---|---|---|
| **Ai thực hiện** | `RetiringPrincipal` hoặc `GranteePrincipal` | Key admin (có `kms:RevokeGrant` trong key policy) |
| **Action** | `kms:RetireGrant` | `kms:RevokeGrant` |
| **Use case** | Normal lifecycle — job xong, grant không cần nữa | Emergency — thu hồi ngay lập tức |
| **Tự động hóa** | Grantee tự retire khi xong | Admin revoke khi phát hiện abuse |

### Retire Grant

```bash
# Grantee tự retire grant của mình
aws kms retire-grant \
  --key-id alias/my-key \
  --grant-id 0c237476b39f8bc44e45212e08498fbe8f1ca8419e7a0ef200bcee3b79975f9e

# Hoặc dùng grant token (không cần key-id)
aws kms retire-grant \
  --grant-token "AQpAM2Rh..."
```

### Revoke Grant (Thu Hồi Khẩn Cấp)

```bash
# Key admin revoke ngay lập tức
aws kms revoke-grant \
  --key-id alias/my-key \
  --grant-id 0c237476b39f8bc44e45212e08498fbe8f1ca8419e7a0ef200bcee3b79975f9e
```

### Pattern: Lifecycle Management Tự Động

```python
import boto3
import time

kms = boto3.client('kms')

def process_with_temporary_access(data):
    """Tạo grant → xử lý → tự retire."""

    # 1. Tạo grant tạm thời
    grant = kms.create_grant(
        KeyId='alias/batch-key',
        GranteePrincipal='arn:aws:iam::123456789012:role/BatchRole',
        Operations=['Decrypt', 'GenerateDataKey'],
        Name=f'batch-job-{int(time.time())}',
        RetiringPrincipal='arn:aws:iam::123456789012:role/BatchRole'
    )

    try:
        # 2. Thực hiện công việc cần mã hóa/giải mã
        result = do_encryption_work(data, grant['GrantToken'])
        return result

    finally:
        # 3. Luôn retire grant khi xong (kể cả khi exception)
        kms.retire_grant(GrantToken=grant['GrantToken'])
```

---

## Dịch Vụ AWS Dùng Grants

### EBS (Elastic Block Store — Lưu Trữ Khối Đàn Hồi)

```
Khi attach encrypted EBS volume:
  1. EC2 service tạo grant cho EC2 Nitro hypervisor
  2. Hypervisor dùng grant để decrypt DEK của volume
  3. Volume I/O qua DEK trong hardware
  4. Khi detach volume: grant được retired tự động

Bạn thấy grant này khi:
aws kms list-grants --key-id alias/my-ebs-key
→ GranteePrincipal: "arn:aws:iam::123456789012:role/aws-service-role/ec2..."
```

### Amazon EKS (Elastic Kubernetes Service — Dịch Vụ Kubernetes Quản Lý)

```
Secrets Encryption với KMS:
  EKS tạo grant để mã hóa Kubernetes Secrets trong etcd
  Grant cho phép:
    - kms:Encrypt (lưu secret)
    - kms:Decrypt (đọc secret)
    - kms:GenerateDataKey (envelope encryption)
    - kms:DescribeKey
```

### AWS Nitro Enclaves

```
Nitro Enclave cần giải mã dữ liệu nhạy cảm:
  1. Host instance tạo grant cho enclave attestation document
  2. Grant có constraint: AttestationDocumentHash phải khớp
  3. Chỉ enclave thật (với đúng binary hash) mới decrypt được
  4. Kể cả host instance cũng không decrypt được nếu không qua enclave

Đây là ví dụ mạnh nhất về grants với cryptographic attestation.
```

---

## So Sánh

### Grant vs Key Policy vs IAM Policy

| Tiêu Chí | Grant | Key Policy | IAM Policy |
|---|---|---|---|
| **Gắn với** | Key (token riêng) | Key | Identity (user/role) |
| **Thay đổi cần** | API call | PutKeyPolicy | PutRolePolicy/PutUserPolicy |
| **Granularity** | Per-operation, per-context | Broad control | Per-resource |
| **Tạm thời** | Dễ retire/revoke | Khó xóa statement | Phải sửa policy |
| **Eventual consistency** | Có (~5s, dùng token) | Có | Có |
| **Programmatic** | Dễ tự động hóa | Khó (phải update JSON) | Dễ tự động hóa |
| **Cross-account** | Không trực tiếp | Có | Kết hợp với key policy |
| **Audit** | CloudTrail | CloudTrail | CloudTrail |
| **Dịch vụ AWS** | Ưu tiên | Đôi khi | Ít dùng riêng lẻ |

### Quyết Định Nên Dùng Gì?

```
Cần kiểm soát cơ bản, lâu dài?
  → Key Policy + IAM Policy

Cần ủy quyền tự động cho dịch vụ AWS?
  → AWS tự tạo Grant (EBS, EKS, ...)

Cần ủy quyền tạm thời trong code?
  → CreateGrant + RetireGrant sau khi xong

Cần ràng buộc theo encryption context?
  → Grant với Constraints

Cần cross-account access thường xuyên?
  → Key Policy cross-account
```

---

## Thực Hành CLI

### Liệt Kê Grants Của Một Key

```bash
# Xem tất cả grants
aws kms list-grants --key-id alias/my-key

# Output ví dụ:
{
  "Grants": [
    {
      "KeyId": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc",
      "GrantId": "0c237476b39...",
      "Name": "batch-job-1716000000",
      "CreationDate": "2026-05-16T10:00:00Z",
      "GranteePrincipal": "arn:aws:iam::123456789012:role/BatchRole",
      "RetiringPrincipal": "arn:aws:iam::123456789012:role/OrchestratorRole",
      "IssuingAccount": "arn:aws:iam::123456789012:root",
      "Operations": ["Decrypt", "GenerateDataKey"],
      "Constraints": {
        "EncryptionContextSubset": {"environment": "production"}
      }
    }
  ]
}
```

### Kiểm Tra Grant Của Calling Identity

```bash
# Liệt kê grants mà identity hiện tại là grantee
aws kms list-retirable-grants \
  --retiring-principal arn:aws:iam::123456789012:role/MyRole
```

### Terraform: Tạo Grant

```hcl
resource "aws_kms_grant" "data_processing" {
  name              = "data-processing-grant"
  key_id            = aws_kms_key.app_key.key_id
  grantee_principal = aws_iam_role.processing.arn
  retiring_principal = aws_iam_role.orchestrator.arn

  operations = ["Encrypt", "Decrypt", "GenerateDataKey", "DescribeKey"]

  constraints {
    encryption_context_subset = {
      environment = "production"
      service     = "data-processor"
    }
  }
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: KMS Grant là gì, và khác Key Policy như thế nào?**

> Grant là cơ chế ủy quyền động, lập trình được, không cần sửa key policy. Grant tạo ra "token" ủy quyền cho một principal thực hiện thao tác cụ thể trên một CMK. Khác với key policy (tĩnh, JSON, cần update qua console/CLI riêng), grant có thể tạo và thu hồi bằng API call trong code. Grants phù hợp cho: dịch vụ AWS tự tạo (EBS, EKS), ủy quyền tạm thời, ủy quyền có điều kiện theo encryption context.

**Q: Grant Token giải quyết vấn đề gì?**

> KMS là distributed system — khi tạo grant mới, thay đổi cần ~5 giây để propagate tới tất cả KMS nodes (eventual consistency). Nếu gọi API ngay sau khi tạo grant, node nhận request có thể chưa biết về grant → AccessDeniedException. Grant Token là chứng minh ngay lập tức (không cần propagation) rằng grant đã được tạo. Truyền grant token vào API call như `Decrypt(GrantTokens=[token])` để bỏ qua vấn đề này.

**Q: Khi nào dùng `EncryptionContextEquals` vs `EncryptionContextSubset` trong grant?**

> `EncryptionContextEquals` yêu cầu context phải HOÀN TOÀN GIỐNG — đúng key, đúng value, không thừa. Dùng khi cần kiểm soát chặt chẽ, mỗi grant chỉ cho một purpose cụ thể. `EncryptionContextSubset` yêu cầu context phải CHỨA ít nhất các key-value đã khai báo (có thể có thêm keys khác). Dùng cho multi-tenant app: grant yêu cầu `{"customerId": "123"}` → grantee chỉ decrypt dữ liệu của customer 123, dù context còn chứa thêm `{"table": "orders"}`.

---

## Tóm Tắt

```
KMS Grants — Điểm Chính:

Định nghĩa: Token ủy quyền động, không cần sửa key policy
Thành phần: KeyId + GranteePrincipal + Operations + Constraints + RetiringPrincipal
Tạo: CreateGrant (API) → trả về GrantId + GrantToken
Dùng ngay: Truyền GrantToken vào API call (tránh eventual consistency ~5s)
Constraints: EncryptionContextEquals (strict) hoặc EncryptionContextSubset (subset)
Thu hồi: RetireGrant (lifecycle bình thường) hoặc RevokeGrant (khẩn cấp)
AWS Services: EBS, EKS, Nitro Enclaves tự tạo grants cho bạn
Best use: Ủy quyền tạm thời, programmatic, cho dịch vụ AWS, multi-tenant
```

---

**Trước Đó:** [3-key-policies.md](3-key-policies.md)
**Tiếp Theo:** [5-cloudhsm.md](5-cloudhsm.md) — CloudHSM FIPS 140-2 Level 3

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
