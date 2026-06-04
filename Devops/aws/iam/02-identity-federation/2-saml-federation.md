# 2 — SAML 2.0 Federation (Liên Kết Định Danh SAML 2.0)

> SAML 2.0 — Security Assertion Markup Language (Ngôn Ngữ Đánh Dấu Khẳng Định Bảo Mật) — là chuẩn mở cho phép doanh nghiệp tích hợp hệ thống định danh nội bộ (Active Directory, Okta, Azure AD) với AWS mà không cần tạo IAM user riêng lẻ.

---

## 📚 Mục Lục

1. [Khái Niệm SAML 2.0](#khái-niệm-saml-20)
2. [Luồng Xác Thực](#luồng-xác-thực)
3. [SAML Federation Với AWS Console](#saml-federation-với-aws-console)
4. [SAML Federation Với AWS CLI/API](#saml-federation-với-aws-cliapi)
5. [Cấu Hình Với Active Directory](#cấu-hình-với-active-directory)
6. [Cấu Hình Với Okta](#cấu-hình-với-okta)
7. [Cấu Hình Với Azure AD](#cấu-hình-với-azure-ad)
8. [SAML Attributes Mapping](#saml-attributes-mapping)
9. [Bảo Mật SAML](#bảo-mật-saml)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm SAML 2.0

### Các Thành Phần

```
IdP — Identity Provider (Nhà Cung Cấp Định Danh):
  - Nắm giữ và xác thực định danh người dùng
  - Ví dụ: Active Directory + ADFS, Okta, Azure AD, Google Workspace

SP — Service Provider (Nhà Cung Cấp Dịch Vụ):
  - Dịch vụ mà người dùng muốn truy cập
  - Trong context AWS: AWS là SP

SAML Assertion (Khẳng Định SAML):
  - XML document được IdP ký số, chứa thông tin user
  - Gồm: authentication statement, attribute statement, authorization statement

Metadata (Siêu Dữ Liệu):
  - XML document mô tả cấu hình của IdP hoặc SP
  - Chứa: certificates, endpoints, entity ID
  - IdP và SP trao đổi metadata để tin tưởng nhau
```

### IdP-Initiated vs SP-Initiated Flow

```
SP-Initiated (phổ biến hơn):
1. User truy cập AWS Console
2. AWS redirect về IdP để xác thực
3. IdP xác thực user, gửi SAML assertion về AWS
4. AWS cấp quyền truy cập

IdP-Initiated:
1. User đăng nhập vào IdP portal (Okta Dashboard, Azure MyApps)
2. User click vào "AWS" app
3. IdP tạo SAML assertion và gửi thẳng đến AWS
4. AWS cấp quyền truy cập

AWS hỗ trợ cả hai, nhưng IdP-Initiated đơn giản hơn cho người dùng.
```

---

## Luồng Xác Thực

### SP-Initiated SAML Flow Chi Tiết

```
User Browser          AWS Console/STS          Corporate IdP (ADFS/Okta)
     │                       │                           │
     │─── GET /console ──────▶│                           │
     │                       │                           │
     │◀── 302 Redirect ───────│                           │
     │    Location: IdP URL  │                           │
     │    + SAMLRequest      │                           │
     │                       │                           │
     │─── GET IdP URL ────────────────────────────────────▶│
     │    ?SAMLRequest=...   │                           │
     │                       │                           │
     │◀── Login Page ─────────────────────────────────────│
     │                       │                           │
     │─── POST credentials ───────────────────────────────▶│
     │    + MFA token        │                           │
     │                       │                           │
     │◀── 302 Redirect ───────────────────────────────────│
     │    Location: AWS ACS  │                           │
     │    SAMLResponse=...   │                           │
     │    (signed XML)       │                           │
     │                       │                           │
     │─── POST SAMLResponse ─▶│                           │
     │    to ACS URL         │                           │
     │                       │                           │
     │                       │── Verify signature ──────│
     │                       │── Parse assertions ──────│
     │                       │── Map Role claim ─────────│
     │                       │                           │
     │                       │── AssumeRoleWithSAML ─────│
     │                       │   to AWS STS              │
     │                       │                           │
     │                       │◀── Temp Credentials ──────│
     │                       │                           │
     │◀── AWS Console ────────│                           │
     │    (authenticated)    │                           │
     │                       │                           │
```

### SAML Assertion Ví Dụ

```xml
<!-- SAML Response được IdP ký số -->
<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
                InResponseTo="_abc123"
                Destination="https://signin.aws.amazon.com/saml">

  <Issuer>https://idp.mycompany.com/adfs/ls</Issuer>

  <!-- Chữ ký số của IdP — AWS verify bằng IdP certificate -->
  <Signature xmlns="http://www.w3.org/2000/09/xmldsig#">
    <SignatureValue>...</SignatureValue>
    <KeyInfo>
      <X509Data><X509Certificate>...</X509Certificate></X509Data>
    </KeyInfo>
  </Signature>

  <saml:Assertion>
    <!-- Khi nào assertion có hiệu lực -->
    <Conditions NotBefore="2026-05-16T08:00:00Z"
                NotOnOrAfter="2026-05-16T08:30:00Z">

    <!-- Authentication: user đã xác thực bằng cách nào -->
    <AuthnStatement AuthnInstant="2026-05-16T08:00:00Z">
      <AuthnContext>
        <AuthnContextClassRef>
          urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
        </AuthnContextClassRef>
      </AuthnContext>
    </AuthnStatement>

    <!-- Attributes: thông tin về user, quan trọng nhất là Role mapping -->
    <AttributeStatement>

      <!-- Role mapping — AWS dùng attribute này để xác định IAM Role -->
      <Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
        <AttributeValue>
          arn:aws:iam::123456789012:role/BackendDeveloper,
          arn:aws:iam::123456789012:saml-provider/MyCompanyIdP
        </AttributeValue>
      </Attribute>

      <!-- Session duration tùy chỉnh (giây) -->
      <Attribute Name="https://aws.amazon.com/SAML/Attributes/SessionDuration">
        <AttributeValue>28800</AttributeValue>
      </Attribute>

      <!-- Display name trong console -->
      <Attribute Name="https://aws.amazon.com/SAML/Attributes/RoleSessionName">
        <AttributeValue>alice@mycompany.com</AttributeValue>
      </Attribute>

    </AttributeStatement>
  </saml:Assertion>
</samlp:Response>
```

---

## SAML Federation Với AWS Console

### Bước 1: Tạo SAML Identity Provider Trong IAM

```bash
# Download metadata XML từ IdP
# Ví dụ ADFS metadata URL: https://adfs.mycompany.com/FederationMetadata/2007-06/FederationMetadata.xml

# Tạo SAML IdP trong IAM
aws iam create-saml-provider \
  --saml-metadata-document file://idp-metadata.xml \
  --name MyCompanyIdP

# Output:
# {
#   "SAMLProviderArn": "arn:aws:iam::123456789012:saml-provider/MyCompanyIdP"
# }
```

### Bước 2: Tạo IAM Role Với Trust Policy Cho SAML

```json
// Trust Policy — cho phép SAML federation assume role này
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/MyCompanyIdP"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

```bash
# Tạo role
aws iam create-role \
  --role-name BackendDeveloper-SAML \
  --assume-role-policy-document file://trust-policy.json \
  --max-session-duration 28800

# Gắn permission policy
aws iam attach-role-policy \
  --role-name BackendDeveloper-SAML \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

### Bước 3: Cấu Hình IdP Gửi Role Claim

```
Trong IdP (Okta/ADFS/Azure AD), cấu hình Claim Rules để gửi:

Attribute: https://aws.amazon.com/SAML/Attributes/Role
Value format: <role-arn>,<saml-provider-arn>

Ví dụ:
Value: arn:aws:iam::123456789012:role/BackendDeveloper-SAML,
       arn:aws:iam::123456789012:saml-provider/MyCompanyIdP

Nếu user thuộc nhiều group → có thể gửi nhiều Role values
→ AWS sẽ hiển thị menu để user chọn role
```

---

## SAML Federation Với AWS CLI/API

### Programmatic Access Qua SAML

```python
import boto3
import requests
from bs4 import BeautifulSoup
import base64
import xml.etree.ElementTree as ET

def get_saml_assertion(idp_url, username, password):
    """
    Xác thực với IdP và lấy SAML assertion.
    Đây là ví dụ đơn giản — production cần handle MFA, SSL pinning, v.v.
    """
    session = requests.Session()
    
    # POST credentials đến IdP
    response = session.post(idp_url, data={
        'username': username,
        'password': password
    })
    
    # Parse SAML response từ HTML form
    soup = BeautifulSoup(response.text, 'html.parser')
    saml_response = soup.find('input', {'name': 'SAMLResponse'})['value']
    
    return saml_response

def assume_role_with_saml(saml_assertion, role_arn, principal_arn):
    """
    Dùng SAML assertion để lấy temporary AWS credentials.
    """
    sts = boto3.client('sts')
    
    response = sts.assume_role_with_saml(
        RoleArn=role_arn,
        PrincipalArn=principal_arn,  # SAML Provider ARN
        SAMLAssertion=saml_assertion,
        DurationSeconds=3600
    )
    
    credentials = response['Credentials']
    return {
        'access_key': credentials['AccessKeyId'],
        'secret_key': credentials['SecretAccessKey'],
        'session_token': credentials['SessionToken'],
        'expiration': credentials['Expiration']
    }

# Sử dụng
saml = get_saml_assertion(
    'https://adfs.mycompany.com/adfs/ls/IdpInitiatedSignOn.aspx',
    'alice@mycompany.com',
    'mypassword'
)

creds = assume_role_with_saml(
    saml,
    'arn:aws:iam::123456789012:role/BackendDeveloper-SAML',
    'arn:aws:iam::123456789012:saml-provider/MyCompanyIdP'
)

# Dùng credentials
boto3.setup_default_session(
    aws_access_key_id=creds['access_key'],
    aws_secret_access_key=creds['secret_key'],
    aws_session_token=creds['session_token']
)
```

---

## Cấu Hình Với Active Directory

### Dùng ADFS (Active Directory Federation Services — Dịch Vụ Liên Kết Active Directory)

```
Kiến trúc:
On-premises AD ──▶ ADFS ──▶ (SAML) ──▶ AWS IAM

ADFS là SAML IdP chạy trên-premises (tại chỗ),
kết nối AD với các SP bên ngoài như AWS.

Bước cấu hình ADFS:
1. Cài ADFS trên Windows Server
2. Thêm "Relying Party Trust" cho AWS:
   - Relying Party: https://signin.aws.amazon.com/saml
   - Import AWS SP metadata
3. Tạo Claim Rules để map AD groups sang AWS roles:
   - AD Group "AWS-Dev-Team" → Role "BackendDeveloper-SAML"
   - AD Group "AWS-Ops-Team" → Role "OperationsAdmin-SAML"
4. Export ADFS metadata và upload lên IAM SAML Provider

Claim Rule ví dụ (ADFS Claim Rule Language):
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/groupsid",
   Value == "S-1-5-21-xxx-xxx-xxx-1234"]  // SID của AWS-Dev-Team group
=> issue(
    Type = "https://aws.amazon.com/SAML/Attributes/Role",
    Value = "arn:aws:iam::123456789012:role/BackendDeveloper-SAML,
             arn:aws:iam::123456789012:saml-provider/MyCompanyIdP"
   );
```

### Dùng AWS Directory Service AD Connector

```
Nếu muốn dùng AD on-premises với IAM Identity Center mà không triển khai ADFS:

AD Connector: Proxy service trong AWS, forward authentication về on-premises AD

Architecture:
Identity Center ──▶ AD Connector ──▶ On-premises AD
(không cần ADFS, không sync users lên AWS)

Khi dùng AD Connector:
- Users authenticate trực tiếp với on-premises AD
- Passwords không bao giờ lưu trên AWS
- Cần Direct Connect hoặc VPN cho kết nối on-premises
```

---

## Cấu Hình Với Okta

### Các Bước Cấu Hình

```
Trong Okta Admin Console:

1. Applications → Add Application → Search "AWS IAM Identity Center"
   (hoặc "Amazon Web Services" nếu dùng SAML trực tiếp không qua Identity Center)

2. Sign On tab → SAML 2.0:
   - Single sign on URL: https://us-east-1.signin.aws.amazon.com/platform/saml/acs/<id>
   - Audience URI (SP Entity ID): https://us-east-1.signin.aws.amazon.com/platform/saml/d-xxxxxxxxxxxx

3. Attribute Statements:
   Name: https://aws.amazon.com/SAML/Attributes/RoleSessionName
   Value: user.email

4. Group Attribute Statements:
   Name: https://aws.amazon.com/SAML/Attributes/Role
   Filter: Starts with "aws-"
   (Map Okta groups bắt đầu bằng "aws-" sang roles)

5. Download Okta IdP metadata và upload lên AWS IAM SAML Provider

6. Assign application cho users/groups trong Okta
```

---

## Cấu Hình Với Azure AD

### Cấu Hình Enterprise Application

```
Trong Azure Portal → Azure Active Directory → Enterprise applications:

1. New application → Search "AWS IAM Identity Center"
   (Microsoft cung cấp pre-built template)

2. Single sign-on → SAML:
   Basic SAML Configuration:
   - Identifier (Entity ID): https://us-east-1.signin.aws.amazon.com/platform/saml/d-xxxxxxxxxxxx
   - Reply URL: https://us-east-1.signin.aws.amazon.com/platform/saml/acs/<id>
   - Sign on URL: https://<alias>.awsapps.com/start

3. Attributes & Claims:
   Bỏ default claims, thêm:
   - Claim name: https://aws.amazon.com/SAML/Attributes/RoleSessionName
     Source attribute: user.userprincipalname

4. Group Claims:
   - Emit groups as role claims
   - Security groups → map sang AWS roles

5. Download Federation Metadata XML và upload lên AWS

6. Provisioning tab → Set up SCIM:
   - Tenant URL: SCIM endpoint từ Identity Center
   - Secret Token: Token tạo từ Identity Center
```

---

## SAML Attributes Mapping

### Các Attributes Quan Trọng

```
Attribute Name (phải match chính xác):

1. https://aws.amazon.com/SAML/Attributes/Role (Bắt buộc)
   Format: <role-arn>,<saml-provider-arn>
   Ví dụ: arn:aws:iam::123456789012:role/Dev,arn:aws:iam::123456789012:saml-provider/MyIdP
   
   Có thể gửi nhiều values (user chọn role khi đăng nhập):
   Value 1: arn:aws:iam::123456789012:role/Dev,...
   Value 2: arn:aws:iam::123456789012:role/ReadOnly,...

2. https://aws.amazon.com/SAML/Attributes/RoleSessionName (Bắt buộc)
   Hiển thị trong CloudTrail logs — dùng email để trace dễ dàng
   Ví dụ: alice@mycompany.com

3. https://aws.amazon.com/SAML/Attributes/SessionDuration (Tùy chọn)
   Giây, tối đa bằng MaxSessionDuration của IAM Role
   Ví dụ: 28800 (8 giờ)

4. https://aws.amazon.com/SAML/Attributes/TransitiveTagKeys (Tùy chọn)
   Session tags để pass xuống child roles trong cross-account scenarios

5. https://aws.amazon.com/SAML/Attributes/PrincipalTag:<key> (Tùy chọn)
   Session tags để dùng trong IAM policies (ABAC — Attribute-Based Access Control)
   Ví dụ: https://aws.amazon.com/SAML/Attributes/PrincipalTag:Department
   Value: Engineering
```

### ABAC Với SAML Session Tags

```json
// IAM Policy dùng session tag từ SAML assertion
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::${aws:PrincipalTag/Department}-*",
        "arn:aws:s3:::${aws:PrincipalTag/Department}-*/*"
      ]
    }
  ]
}

// User Alice thuộc Department=engineering
// → Chỉ truy cập được bucket bắt đầu bằng "engineering-"
// ví dụ: s3://engineering-data/, s3://engineering-artifacts/
```

---

## Bảo Mật SAML

### Các Rủi Ro Và Biện Pháp Giảm Thiểu

```
1. SAML Assertion Replay Attack (Tấn Công Phát Lại Assertion):
   Rủi ro: Attacker capture SAML assertion và dùng lại
   Biện pháp:
   - Assertion có NotOnOrAfter timestamp (thường 5–30 phút)
   - AWS reject assertions đã hết hạn
   - Dùng HTTPS cho tất cả redirect flows

2. XML Signature Wrapping (Tấn Công Bao Bọc Chữ Ký XML):
   Rủi ro: Attacker chèn unsigned XML vào signed assertion
   Biện pháp:
   - AWS parser kiểm tra canonical XML trước khi verify signature
   - Cập nhật IdP software thường xuyên

3. Man-in-the-Middle (Tấn Công Trung Gian):
   Rủi ro: Attacker intercept SAML flow
   Biện pháp:
   - Enforce HTTPS (TLS 1.2+) cho tất cả endpoints
   - Certificate pinning trong custom IdP client

4. Phishing Của IdP:
   Rủi ro: Fake login page giả IdP
   Biện pháp:
   - Educate users verify URL trước khi nhập credentials
   - Bật MFA — phishing không lấy được MFA token
   - Hardware security keys (FIDO2) — phishing-resistant

5. Overly Permissive Role Claims:
   Rủi ro: IdP gửi quá nhiều Role claims cho user
   Biện pháp:
   - Review Claim Rules trong IdP thường xuyên
   - Audit CloudTrail: AssumeRoleWithSAML events
   - Access Analyzer phát hiện overly permissive roles
```

### Kiểm Tra Cấu Hình SAML

```bash
# Xem danh sách SAML providers
aws iam list-saml-providers

# Xem metadata của một provider
aws iam get-saml-provider \
  --saml-provider-arn arn:aws:iam::123456789012:saml-provider/MyCompanyIdP

# Cập nhật metadata khi IdP certificate hết hạn
aws iam update-saml-provider \
  --saml-provider-arn arn:aws:iam::123456789012:saml-provider/MyCompanyIdP \
  --saml-metadata-document file://new-metadata.xml

# Xem roles có trust policy cho SAML provider
aws iam list-roles --query \
  "Roles[?AssumeRolePolicyDocument.Statement[?Principal.Federated=='arn:aws:iam::123456789012:saml-provider/MyCompanyIdP']]"
```

---

## Câu Hỏi Phỏng Vấn

### Q1: SAML Federation hoạt động như thế nào ở mức cao?

```
1. Admin tạo SAML IdP trong IAM, upload metadata của IdP
2. Admin tạo IAM Role với Trust Policy cho phép SAML Federation
3. IdP cấu hình gửi SAML assertions với Role claim chứa Role ARN
4. User xác thực với IdP → nhận SAML assertion
5. User/browser POST assertion đến AWS sign-in endpoint
6. AWS verify signature, parse Role claim
7. AWS gọi STS AssumeRoleWithSAML → Temporary credentials
8. User truy cập AWS Console hoặc API với temporary credentials
```

### Q2: Tại sao cần gửi cả Role ARN và SAML Provider ARN trong Role attribute?

```
Format: <role-arn>,<saml-provider-arn>

Lý do cần SAML Provider ARN:
- AWS cần verify assertion này đến từ đúng IdP đã được trust
- Nếu chỉ có Role ARN, attacker có thể dùng SAML assertion từ IdP khác
  để assume role (nếu role's trust policy chỉ check federated = saml)
- Với cả hai ARN, AWS check: "SAML assertion này từ IdP đã được register
  trong account này không?"

Đây là cơ chế double-verification quan trọng.
```

### Q3: Giải thích tại sao SAML session có thể ngắn hơn MaxSessionDuration của Role

```
Hai giới hạn độc lập:
- SessionDuration attribute trong SAML assertion (IdP kiểm soát)
- MaxSessionDuration của IAM Role (tối đa 12 giờ)

AWS sẽ dùng giá trị NHỎ HƠN trong hai giá trị:
- SAML assertion yêu cầu 10 giờ, MaxSessionDuration = 8 giờ → Session = 8 giờ
- SAML assertion yêu cầu 1 giờ, MaxSessionDuration = 8 giờ → Session = 1 giờ

Thực tế: IdP thường set assertion validity ngắn (5–30 phút),
nhưng AWS session có thể dài hơn theo SessionDuration attribute.
```

### Q4: Làm thế nào audit ai đã sử dụng SAML Federation?

```bash
# CloudTrail ghi lại AssumeRoleWithSAML events
# Xem trong CloudTrail:

{
  "eventName": "AssumeRoleWithSAML",
  "userIdentity": {
    "type": "SAMLUser",
    "principalId": "arn:aws:iam::123456789012:saml-provider/MyIdP:alice@company.com",
    "userName": "alice@company.com"  // từ RoleSessionName attribute
  },
  "requestParameters": {
    "roleArn": "arn:aws:iam::123456789012:role/BackendDeveloper",
    "principalArn": "arn:aws:iam::123456789012:saml-provider/MyIdP",
    "durationSeconds": 28800
  },
  "sourceIPAddress": "203.x.x.x"
}

// Dùng CloudTrail Insights để phát hiện:
// - Login từ IP bất thường
// - Login ngoài giờ làm việc
// - Role assumptions không thông thường
```

---

## 🔗 Điều Hướng

- **Trước:** [1-iam-identity-center.md](1-iam-identity-center.md) — IAM Identity Center (SSO)
- **Tiếp theo:** [3-oidc-federation.md](3-oidc-federation.md) — OIDC Federation
- **Liên quan:** [5-cross-account-roles.md](5-cross-account-roles.md) — Cross-account với SAML

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
