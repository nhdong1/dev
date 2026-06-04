# 4 — Amazon Cognito (Quản Lý Định Danh Ứng Dụng)

> Amazon Cognito là dịch vụ quản lý định danh người dùng cho ứng dụng web và mobile, cung cấp hai thành phần chính: **User Pools** (Nhóm Người Dùng — authentication) và **Identity Pools** (Nhóm Định Danh — authorization để truy cập AWS).

---

## 📚 Mục Lục

1. [Tổng Quan Cognito](#tổng-quan-cognito)
2. [User Pools — Xác Thực Người Dùng](#user-pools--xác-thực-người-dùng)
3. [Identity Pools — Phân Quyền AWS](#identity-pools--phân-quyền-aws)
4. [User Pools + Identity Pools Kết Hợp](#user-pools--identity-pools-kết-hợp)
5. [Tích Hợp Social IdP](#tích-hợp-social-idp)
6. [Lambda Triggers — Tùy Biến Luồng Auth](#lambda-triggers--tùy-biến-luồng-auth)
7. [Bảo Mật Cognito](#bảo-mật-cognito)
8. [Cognito vs Alternatives](#cognito-vs-alternatives)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Cognito

### Hai Thành Phần Chính

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AMAZON COGNITO                                │
│                                                                       │
│  ┌────────────────────────────┐  ┌────────────────────────────────┐  │
│  │       USER POOLS           │  │      IDENTITY POOLS             │  │
│  │  (Nhóm Người Dùng)         │  │  (Nhóm Định Danh / Federated   │  │
│  │                            │  │   Identity)                     │  │
│  │  • User directory          │  │                                 │  │
│  │  • Sign-up / Sign-in       │  │  • Map authenticated users      │  │
│  │  • MFA                     │  │    to IAM roles                 │  │
│  │  • Password policies       │  │  • Support: Cognito User Pools, │  │
│  │  • JWT tokens (ID/Access/  │  │    SAML, OIDC, Social IdPs,     │  │
│  │    Refresh)                │  │    Guest (unauthenticated)      │  │
│  │  • Social IdP federation   │  │  • Output: Temporary AWS        │  │
│  │  • SAML / OIDC             │  │    credentials via STS          │  │
│  │  • Hosted UI               │  │                                 │  │
│  └────────────────────────────┘  └────────────────────────────────┘  │
│            │                                    │                     │
│            │ JWT tokens                         │ AWS Credentials     │
│            ▼                                    ▼                     │
│       App Backend                          S3, DynamoDB,              │
│       API Gateway                          Lambda, etc.               │
└─────────────────────────────────────────────────────────────────────┘

Câu hỏi đơn giản:
- Cần authenticate users cho app? → User Pools
- Cần users truy cập AWS resources trực tiếp? → Identity Pools
- Cần cả hai? → Dùng cả hai kết hợp (pattern phổ biến nhất)
```

---

## User Pools — Xác Thực Người Dùng

### Chức Năng

```
User Pools là fully managed user directory:
- Lưu trữ user profiles (email, phone, custom attributes)
- Handle sign-up, sign-in, forgot password, email/phone verification
- MFA (TOTP — Time-based One-Time Password, SMS)
- Password policy (độ dài, complexity, expiry)
- Account lockout sau nhiều lần sai password
- Adaptive authentication (phát hiện đăng nhập bất thường)
```

### Tokens Được Phát Hành

```
Sau khi đăng nhập thành công, User Pool phát hành 3 tokens:

1. ID Token (JWT):
   - Chứa thông tin về user (email, phone, custom attributes)
   - Dùng để identify user trong backend
   - Expire: mặc định 1 giờ (cấu hình được)

2. Access Token (JWT):
   - Chứa user's scopes và permissions
   - Dùng để authorize API calls
   - Expire: mặc định 1 giờ

3. Refresh Token:
   - Dùng để lấy ID/Access token mới mà không cần đăng nhập lại
   - Expire: mặc định 30 ngày (cấu hình được)
   - Lưu trữ an toàn trong thiết bị
```

### Tạo User Pool

```bash
# Tạo User Pool cơ bản
aws cognito-idp create-user-pool \
  --pool-name MyAppUserPool \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 12,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true,
      "RequireSymbols": true,
      "TemporaryPasswordValidityDays": 1
    }
  }' \
  --mfa-configuration OPTIONAL \
  --auto-verified-attributes email \
  --username-attributes email \
  --account-recovery-setting '{
    "RecoveryMechanisms": [
      {"Priority": 1, "Name": "verified_email"}
    ]
  }'

# Tạo App Client (không có secret — dùng cho SPA/mobile)
aws cognito-idp create-user-pool-client \
  --user-pool-id us-east-1_XXXXXXXXX \
  --client-name MyWebApp \
  --no-generate-secret \
  --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH \
  --access-token-validity 60 \
  --id-token-validity 60 \
  --refresh-token-validity 30 \
  --token-validity-units '{
    "AccessToken": "minutes",
    "IdToken": "minutes",
    "RefreshToken": "days"
  }'
```

### Xác Thực Từ Frontend (JavaScript SDK)

```javascript
import { CognitoIdentityProviderClient, InitiateAuthCommand } from "@aws-sdk/client-cognito-identity-provider";
import { AuthenticationDetails, CognitoUser, CognitoUserPool } from "amazon-cognito-identity-js";

// Cấu hình User Pool
const poolData = {
  UserPoolId: 'us-east-1_XXXXXXXXX',
  ClientId: 'xxxxxxxxxxxxxxxxxxxxxxxxxx'
};
const userPool = new CognitoUserPool(poolData);

// Sign in
function signIn(email, password) {
  const authenticationDetails = new AuthenticationDetails({
    Username: email,
    Password: password
  });

  const cognitoUser = new CognitoUser({
    Username: email,
    Pool: userPool
  });

  cognitoUser.authenticateUser(authenticationDetails, {
    onSuccess: (result) => {
      const idToken = result.getIdToken().getJwtToken();
      const accessToken = result.getAccessToken().getJwtToken();
      const refreshToken = result.getRefreshToken().getToken();
      
      // Lưu tokens vào secure storage (không phải localStorage cho production)
      console.log('Login successful');
    },
    
    onFailure: (err) => {
      console.error('Login failed:', err.message);
    },
    
    // MFA challenge
    totpRequired: (secretCode) => {
      const mfaCode = prompt('Enter MFA code:');
      cognitoUser.sendMFACode(mfaCode, this, 'SOFTWARE_TOKEN_MFA');
    },
    
    // Buộc đổi mật khẩu (lần đầu đăng nhập)
    newPasswordRequired: (userAttributes) => {
      const newPassword = prompt('Enter new password:');
      cognitoUser.completeNewPasswordChallenge(newPassword, {}, this);
    }
  });
}
```

### Verify JWT Token Trong Backend

```python
import jwt
import requests
from functools import lru_cache

REGION = 'us-east-1'
USER_POOL_ID = 'us-east-1_XXXXXXXXX'
APP_CLIENT_ID = 'xxxxxxxxxxxxxxxxxxxxxxxxxx'

@lru_cache(maxsize=1)
def get_public_keys():
    """Fetch Cognito public keys (cache để không fetch mỗi request)"""
    url = f"https://cognito-idp.{REGION}.amazonaws.com/{USER_POOL_ID}/.well-known/jwks.json"
    response = requests.get(url)
    return response.json()['keys']

def verify_cognito_token(token: str) -> dict:
    """Verify và decode JWT token từ Cognito"""
    # Decode header để lấy kid (Key ID)
    header = jwt.get_unverified_header(token)
    
    # Tìm public key khớp với kid
    keys = get_public_keys()
    public_key = None
    for key in keys:
        if key['kid'] == header['kid']:
            public_key = jwt.algorithms.RSAAlgorithm.from_jwk(key)
            break
    
    if not public_key:
        raise ValueError("Public key not found")
    
    # Verify và decode token
    claims = jwt.decode(
        token,
        public_key,
        algorithms=['RS256'],
        audience=APP_CLIENT_ID,
        options={"require": ["exp", "iss", "sub"]}
    )
    
    # Verify issuer
    expected_iss = f"https://cognito-idp.{REGION}.amazonaws.com/{USER_POOL_ID}"
    if claims['iss'] != expected_iss:
        raise ValueError("Invalid issuer")
    
    return claims

# Sử dụng trong API endpoint (ví dụ FastAPI)
from fastapi import HTTPException, Header

async def get_current_user(authorization: str = Header(...)):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401)
    
    token = authorization.split(" ")[1]
    try:
        claims = verify_cognito_token(token)
        return claims
    except Exception as e:
        raise HTTPException(status_code=401, detail=str(e))
```

### Hosted UI (Giao Diện Đăng Nhập Được Quản Lý)

```
Cognito cung cấp Hosted UI — trang đăng nhập sẵn có không cần code:

URL format:
https://<domain>.auth.<region>.amazoncognito.com/login?
  client_id=<app-client-id>&
  response_type=code&
  scope=email+openid+profile&
  redirect_uri=https://myapp.com/callback

Khi user đăng nhập:
1. Cognito xử lý authentication
2. Redirect về redirect_uri với authorization code
3. App exchange code lấy tokens

Customize Hosted UI:
- Upload logo
- Custom CSS (màu sắc, font, layout)
- Tùy chỉnh error messages
- Custom domain (auth.mycompany.com thay vì amazoncognito.com)

Khi dùng Hosted UI:
- Không cần tự build login/register UI
- Tự động handle CSRF protection
- Social login buttons tự động hiện
```

---

## Identity Pools — Phân Quyền AWS

### Mục Đích

Identity Pools giải quyết câu hỏi: "Sau khi user đã authenticated, làm thế nào để họ truy cập AWS resources (S3, DynamoDB, ...) trực tiếp từ client?"

```
Luồng Identity Pools:
1. User đăng nhập qua User Pool (hoặc Google, Facebook, SAML...)
2. App gửi token (JWT hoặc social token) đến Identity Pool
3. Identity Pool map token sang IAM Role phù hợp
4. Identity Pool gọi STS để lấy temporary credentials
5. App nhận AWS credentials và truy cập AWS trực tiếp
```

### Tạo Identity Pool

```bash
aws cognito-identity create-identity-pool \
  --identity-pool-name MyAppIdentityPool \
  --allow-unauthenticated-identities \  # Cho phép guest access (không đăng nhập)
  --cognito-identity-providers '{
    "ProviderName": "cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX",
    "ClientId": "xxxxxxxxxxxxxxxxxxxxxxxxxx",
    "ServerSideTokenCheck": true
  }' \
  --supported-login-providers '{
    "accounts.google.com": "your-google-client-id.apps.googleusercontent.com",
    "graph.facebook.com": "your-facebook-app-id"
  }'
```

### Authenticated vs Unauthenticated Roles

```json
// IAM Role cho authenticated users
// Trust Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "cognito-identity.amazonaws.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "cognito-identity.amazonaws.com:aud": "us-east-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
        },
        "ForAnyValue:StringLike": {
          "cognito-identity.amazonaws.com:amr": "authenticated"
        }
      }
    }
  ]
}

// Permission Policy cho authenticated users:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      // Mỗi user chỉ truy cập folder riêng của họ
      "Resource": "arn:aws:s3:::my-app-bucket/${cognito-identity.amazonaws.com:sub}/*"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:UpdateItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/UserData",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"]
        }
      }
    }
  ]
}
```

### Lấy AWS Credentials Từ Identity Pool

```javascript
import { CognitoIdentityClient, GetIdCommand, GetCredentialsForIdentityCommand } from "@aws-sdk/client-cognito-identity";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

async function uploadToS3WithCognitoAuth(idToken, file) {
  const IDENTITY_POOL_ID = 'us-east-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx';
  const USER_POOL_ID = 'us-east-1_XXXXXXXXX';
  const REGION = 'us-east-1';
  
  const identityClient = new CognitoIdentityClient({ region: REGION });
  
  // Bước 1: Lấy Identity ID
  const { IdentityId } = await identityClient.send(new GetIdCommand({
    IdentityPoolId: IDENTITY_POOL_ID,
    Logins: {
      [`cognito-idp.${REGION}.amazonaws.com/${USER_POOL_ID}`]: idToken
    }
  }));
  
  // Bước 2: Lấy temporary credentials
  const { Credentials } = await identityClient.send(new GetCredentialsForIdentityCommand({
    IdentityId,
    Logins: {
      [`cognito-idp.${REGION}.amazonaws.com/${USER_POOL_ID}`]: idToken
    }
  }));
  
  // Bước 3: Dùng credentials để truy cập S3
  const s3Client = new S3Client({
    region: REGION,
    credentials: {
      accessKeyId: Credentials.AccessKeyId,
      secretAccessKey: Credentials.SecretKey,
      sessionToken: Credentials.SessionToken
    }
  });
  
  // Upload file vào folder riêng của user (IdentityId)
  await s3Client.send(new PutObjectCommand({
    Bucket: 'my-app-bucket',
    Key: `${IdentityId}/${file.name}`,
    Body: file
  }));
  
  console.log('Upload successful');
}
```

---

## User Pools + Identity Pools Kết Hợp

### Pattern Phổ Biến Nhất

```
┌──────────────┐
│   User       │
│   (Browser/  │
│    Mobile)   │
└──────┬───────┘
       │ 1. Sign in (email/password)
       ▼
┌──────────────┐
│  Cognito     │──────── 2. Returns JWT tokens ──────────┐
│  User Pool   │         (ID Token, Access Token,         │
└──────────────┘          Refresh Token)                  │
                                                          │
       ┌──────────────────────────────────────────────────┘
       │ 3. Present ID Token
       ▼
┌──────────────┐
│  Cognito     │──────── 4. AssumeRole via STS ──────────▶ IAM Role
│  Identity    │                                           (Auth role)
│  Pool        │──────── 5. Return Temporary Credentials ─────────┐
└──────────────┘                                                   │
                                                                   │
       ┌───────────────────────────────────────────────────────────┘
       │ 6. Use credentials
       ▼
┌──────────────────────────────┐
│  AWS Resources               │
│  (S3, DynamoDB, AppSync...)  │
└──────────────────────────────┘
```

### Cấu Hình Kết Hợp

```python
# Backend API: Verify token và trả về user info
# Frontend trực tiếp truy cập S3/DynamoDB với Identity Pool credentials

# Flow:
# 1. User login qua Cognito Hosted UI → get ID token
# 2. Frontend gửi ID token đến API → API verify và xử lý business logic
# 3. Frontend dùng ID token + Identity Pool → get AWS credentials → upload trực tiếp lên S3

# Tại sao upload trực tiếp lên S3 từ client?
# - Không tốn bandwidth của API server
# - Pre-signed URL cũng làm được, nhưng Identity Pool cho phép user-scoped policies
```

---

## Tích Hợp Social IdP

### Google Login

```javascript
// Sau khi user đăng nhập Google, nhận Google ID Token
// Dùng token này với Cognito Identity Pool

async function loginWithGoogle(googleIdToken) {
  const identityClient = new CognitoIdentityClient({ region: 'us-east-1' });
  
  // Dùng Google token trực tiếp với Identity Pool
  // (không cần User Pool nếu chỉ cần AWS credentials)
  const { IdentityId } = await identityClient.send(new GetIdCommand({
    IdentityPoolId: 'us-east-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
    Logins: {
      'accounts.google.com': googleIdToken
    }
  }));
  
  // Lấy credentials như bình thường
  // ...
}
```

### Cấu Hình User Pool Federated Identity

```
Trong User Pool → Sign-in experience → Federated sign-in:

Thêm Google:
- Google client ID: your-google-client-id.apps.googleusercontent.com
- Google client secret: your-google-client-secret
- Authorized scopes: email profile openid
- Map attributes: email → email, name → name

Sau khi cấu hình:
- User có thể đăng nhập bằng Google qua Hosted UI
- Cognito tạo/liên kết user profile trong User Pool
- App nhận Cognito JWT tokens (không phải Google token)
- App không cần biết user dùng Google hay local account
```

---

## Lambda Triggers — Tùy Biến Luồng Auth

### Các Trigger Quan Trọng

```
Pre Sign-up:
- Kích hoạt: Trước khi user profile được tạo
- Dùng để: Validate email domain, kiểm tra blocklist, custom validation

Post Confirmation:
- Kích hoạt: Sau khi user confirm email/phone
- Dùng để: Tạo user profile trong database, gửi welcome email

Pre Authentication:
- Kích hoạt: Trước khi authentication bắt đầu
- Dùng để: Check thêm điều kiện, block user cụ thể

Post Authentication:
- Kích hoạt: Sau authentication thành công
- Dùng để: Log audit trail, update last login time

Pre Token Generation:
- Kích hoạt: Trước khi Cognito phát hành tokens
- Dùng để: Thêm custom claims vào JWT, modify scopes

Custom Message:
- Kích hoạt: Khi gửi email/SMS verification
- Dùng để: Customize nội dung email, dùng ngôn ngữ của user
```

### Ví Dụ: Pre Token Generation Lambda

```python
def lambda_handler(event, context):
    """
    Thêm custom claims vào Cognito JWT token.
    event['request']['userAttributes'] chứa attributes của user.
    """
    user_id = event['request']['userAttributes']['sub']
    
    # Lấy thêm thông tin từ database
    user_data = get_user_from_db(user_id)
    
    # Thêm custom claims vào token
    event['response']['claimsOverrideDetails'] = {
        'claimsToAddOrOverride': {
            'user_role': user_data['role'],           # 'admin', 'user', 'premium'
            'organization_id': user_data['org_id'],
            'subscription_tier': user_data['tier']    # 'free', 'pro', 'enterprise'
        },
        'claimsToSuppress': [
            'cognito:groups'  # Ẩn raw groups nếu muốn
        ]
    }
    
    return event

def get_user_from_db(user_id: str) -> dict:
    """Fetch user data từ DynamoDB hoặc RDS"""
    import boto3
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Users')
    
    response = table.get_item(Key={'userId': user_id})
    return response.get('Item', {'role': 'user', 'org_id': None, 'tier': 'free'})
```

---

## Bảo Mật Cognito

### Các Lỗi Bảo Mật Phổ Biến

```
1. Lưu Tokens Sai Chỗ:
   ❌ Sai: localStorage (dễ bị XSS attack đọc token)
   ✅ Đúng: HttpOnly cookies (JavaScript không đọc được)
             Hoặc memory chỉ (mất khi refresh page)
             Hoặc secure storage của native app

2. Không Verify Token Signature:
   ❌ Sai: Chỉ decode JWT và tin vào claims
   ✅ Đúng: Verify chữ ký với Cognito public keys (JWKS endpoint)

3. Không Check Token Expiration:
   ❌ Sai: Chỉ check signature, không check exp claim
   ✅ Đúng: Verify cả signature, exp, iss, aud

4. Quá Rộng Identity Pool Role:
   ❌ Sai: Authenticated role có s3:* trên mọi bucket
   ✅ Đúng: Dùng ${cognito-identity.amazonaws.com:sub} trong resource ARN
             để mỗi user chỉ truy cập data của mình

5. Để Client ID Lộ Không Vấn Đề, Nhưng Secret Thì Không:
   - App Client không có secret (SPA, mobile) → OK, đây là thiết kế đúng
   - App Client có secret (backend server) → KHÔNG được expose secret
```

### Advanced Security Features

```bash
# Bật Advanced Security Mode cho User Pool
aws cognito-idp update-user-pool \
  --user-pool-id us-east-1_XXXXXXXXX \
  --user-pool-add-ons '{
    "AdvancedSecurityMode": "ENFORCED"
  }'

# Các tính năng khi bật Advanced Security:
# - Compromised credentials detection (kiểm tra email/password bị lộ)
# - Adaptive authentication (MFA khi phát hiện hành vi bất thường)
# - Risk scores cho mỗi sign-in attempt
# - IP reputation filtering
# - Device fingerprinting
```

---

## Cognito vs Alternatives

### So Sánh

| Tiêu Chí | Cognito | Auth0 | Firebase Auth | Keycloak |
|---|---|---|---|---|
| **Giá** | Pay-per-MAU (50k free) | Pay-per-MAU | Pay-per-MAU | Open source (self-host) |
| **AWS Integration** | Native, tích hợp sâu | Qua OIDC/SAML | Hạn chế | Hạn chế |
| **Ease of Use** | Trung bình | Cao | Cao | Thấp (phức tạp) |
| **Customization** | Lambda triggers | Actions/Rules | Limited | Rất cao |
| **Social Login** | Có | Có | Có | Có (cần config) |
| **Enterprise SSO** | SAML/OIDC | SAML/OIDC | Hạn chế | SAML/OIDC/LDAP |
| **Phù hợp với** | AWS-first apps | Multi-cloud, complex auth | Firebase apps | On-premises, GDPR |

### Khi Nào Không Dùng Cognito?

```
Cân nhắc alternatives khi:
- Cần advanced MFA flows (adaptive, hardware keys) → Auth0 enterprise
- Cần on-premises hoặc data sovereignty nghiêm ngặt → Keycloak
- App đã trên Firebase và không cần AWS resources → Firebase Auth
- Cần audit logging chi tiết hơn built-in → Auth0 hoặc Cognito + CloudTrail
- Cần B2B SSO phức tạp với nhiều tenants → Auth0 Organizations

Dùng Cognito khi:
- App chạy hoàn toàn trên AWS
- Cần tích hợp với S3, DynamoDB, AppSync từ client
- Team không muốn quản lý auth server
- Budget nhỏ (tier miễn phí 50k MAU/tháng)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: User Pool và Identity Pool khác nhau như thế nào?

```
User Pool = Authentication (Xác Thực):
- "Ai là bạn?" 
- Quản lý user directory, sign-up, sign-in
- Output: JWT tokens (ID Token, Access Token, Refresh Token)
- Dùng để: Xác thực user, protect API endpoints

Identity Pool = Authorization (Phân Quyền AWS):
- "Bạn được làm gì với AWS?"
- Map authenticated identity sang IAM Role
- Output: Temporary AWS credentials (AccessKey + SecretKey + SessionToken)
- Dùng để: Truy cập AWS resources trực tiếp từ client

Thường dùng cả hai:
User Pool xác thực → ID Token → Identity Pool → AWS Credentials → S3/DynamoDB
```

### Q2: Tại sao dùng ${cognito-identity.amazonaws.com:sub} trong S3 policy?

```
Policy ví dụ:
"Resource": "arn:aws:s3:::my-bucket/${cognito-identity.amazonaws.com:sub}/*"

Lý do:
- Mỗi user Cognito có Identity ID duy nhất (sub claim)
- Khi user assume role qua Identity Pool, Identity ID được inject vào session
- Policy variable ${cognito-identity.amazonaws.com:sub} resolve thành Identity ID của user hiện tại
- Kết quả: Mỗi user chỉ có thể đọc/ghi vào folder riêng của họ trong S3

Ví dụ thực tế:
User A (ID: us-east-1:aaaa-bbbb) chỉ truy cập: s3://my-bucket/us-east-1:aaaa-bbbb/*
User B (ID: us-east-1:cccc-dddd) chỉ truy cập: s3://my-bucket/us-east-1:cccc-dddd/*

Đây là cách triển khai per-user data isolation không cần custom logic.
```

### Q3: Giải thích Cognito Refresh Token flow

```
1. User login → nhận Access Token (1h), ID Token (1h), Refresh Token (30 ngày)
2. Access Token hết hạn → client dùng Refresh Token gọi InitiateAuth
3. Cognito verify Refresh Token, phát hành Access Token và ID Token mới
4. Refresh Token vẫn có hiệu lực (cho đến ngày hết hạn hoặc revoke)

Revoke Refresh Token:
- User logout → GlobalSignOut revokes tất cả refresh tokens
- Admin revoke: RevokeToken API
- User đổi password → tất cả tokens bị revoke

Lưu ý:
- Revoke Access/ID Token không thể (JWT-based, stateless)
- Chỉ có thể revoke Refresh Token
- Vì vậy: Keep Access Token lifespan ngắn (15-60 phút)
```

### Q4: Làm thế nào bảo vệ Cognito User Pool khỏi brute force?

```
Built-in protections:
1. Advanced Security Mode → Adaptive authentication
   - Phát hiện IP reputation, velocity checks, đăng nhập bất thường
   - Tự động yêu cầu MFA khi phát hiện risk cao
   
2. Account lockout (cần cấu hình):
   - Sau N lần sai → temporarily lock account
   - Thông báo user qua email

3. Pre Authentication Lambda trigger:
   - Custom rate limiting logic
   - IP allowlist/blocklist
   - Check thêm điều kiện tùy ý

4. WAF tích hợp:
   - Gắn WAF Web ACL vào Cognito User Pool
   - Rate limit, IP block, bot detection
   - AWS Managed Rules cho credential stuffing

5. CAPTCHA:
   - Cognito hỗ trợ CAPTCHA challenge khi phát hiện bot
   - Tự động hoặc trigger theo risk level
```

---

## 🔗 Điều Hướng

- **Trước:** [3-oidc-federation.md](3-oidc-federation.md) — OIDC Federation
- **Tiếp theo:** [5-cross-account-roles.md](5-cross-account-roles.md) — Cross-Account Roles
- **Liên quan:** [../01-iam-fundamentals/1-users-groups-roles.md](../01-iam-fundamentals/1-users-groups-roles.md) — IAM Roles

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
