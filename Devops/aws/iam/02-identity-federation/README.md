# 02 — Identity Federation & SSO (Liên Kết Định Danh & Đăng Nhập Một Lần)

> Tổng quan về cách AWS cho phép người dùng bên ngoài xác thực và nhận quyền truy cập tạm thời vào tài nguyên AWS mà không cần tạo IAM user riêng biệt.

---

## 📚 Mục Lục Module

| File | Chủ Đề | Mức Độ |
|---|---|---|
| [1-iam-identity-center.md](1-iam-identity-center.md) | AWS IAM Identity Center — SSO đa tài khoản | ⭐⭐ |
| [2-saml-federation.md](2-saml-federation.md) | SAML 2.0 Federation với AD/Okta/Azure AD | ⭐⭐⭐ |
| [3-oidc-federation.md](3-oidc-federation.md) | OIDC Federation với GitHub Actions, Google | ⭐⭐⭐ |
| [4-cognito-user-pools.md](4-cognito-user-pools.md) | Amazon Cognito — Auth cho ứng dụng web/mobile | ⭐⭐ |
| [5-cross-account-roles.md](5-cross-account-roles.md) | Cross-Account Roles — Mô hình Hub-and-Spoke | ⭐⭐⭐ |

---

## 🎯 Vấn Đề Identity Federation Giải Quyết

### Tình Huống Thực Tế

```
Doanh nghiệp có 500 nhân viên dùng Microsoft Active Directory.
Họ cần truy cập 20 AWS accounts khác nhau.

Giải pháp sai: Tạo 500 IAM users × 20 accounts = 10,000 IAM users cần quản lý
Giải pháp đúng: Dùng Identity Federation — nhân viên đăng nhập AD một lần,
                nhận temporary credentials AWS tự động
```

### Ba Câu Hỏi Cốt Lõi

1. **Ai là người dùng?** — Định danh nằm ở đâu (AD, Google, GitHub, Okta...)?
2. **Họ có quyền gì?** — Map từ group/attribute sang IAM role
3. **Credentials được cấp như thế nào?** — STS AssumeRole, OIDC token, Cognito token

---

## 🏗️ Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────┐
│                    IDENTITY PROVIDERS (IdP)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Active       │  │    Okta /    │  │  Google / GitHub /   │  │
│  │ Directory    │  │  Azure AD    │  │  Any OIDC Provider   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │  SAML 2.0        │  SAML / OIDC         │  OIDC / JWT  │
└─────────┼──────────────────┼──────────────────────┼─────────────┘
          │                  │                       │
          ▼                  ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AWS IDENTITY LAYER                            │
│  ┌───────────────────┐     ┌────────────────────────────────┐   │
│  │  IAM Identity     │     │  STS (Security Token Service   │   │
│  │  Center (SSO)     │────▶│  — Dịch Vụ Token Bảo Mật)     │   │
│  └───────────────────┘     └────────────────────────────────┘   │
│                                         │                        │
│  ┌───────────────────┐                  │ Temporary Credentials  │
│  │  Amazon Cognito   │──────────────────▶ (AccessKey + Secret   │
│  │  (User Pools /    │                  │  + SessionToken)       │
│  │  Identity Pools)  │                  │                        │
│  └───────────────────┘                  ▼                        │
└─────────────────────────────────────────────────────────────────┘
          │ Temporary Credentials (tối đa 12 giờ)
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AWS RESOURCES                                 │
│  Account A (Dev)   Account B (Prod)   Account C (Security)      │
│  ┌───────────┐     ┌───────────┐      ┌───────────┐             │
│  │ S3, EC2   │     │ S3, RDS   │      │ CloudTrail│             │
│  │ Lambda... │     │ Lambda... │      │ GuardDuty │             │
│  └───────────┘     └───────────┘      └───────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔑 Các Giao Thức Liên Kết Định Danh

### SAML 2.0 (Security Assertion Markup Language — Ngôn Ngữ Đánh Dấu Khẳng Định Bảo Mật)

```
Đặc điểm:
- Chuẩn XML-based từ 2005, phổ biến trong môi trường doanh nghiệp
- Dùng với: Active Directory, ADFS, Okta, Azure AD, PingIdentity
- Flow: User → IdP → SAML Assertion → AWS STS → Temporary credentials
- Phù hợp: SSO cho nhân viên doanh nghiệp truy cập AWS Console/CLI

Hạn chế:
- Phức tạp cấu hình, XML-based (verbose)
- Không phù hợp cho machine-to-machine authentication
```

### OIDC (OpenID Connect — Kết Nối Định Danh Mở)

```
Đặc điểm:
- Layer identity nằm trên OAuth 2.0, dùng JWT (JSON Web Token)
- Dùng với: GitHub Actions, Google, Okta, Auth0, GitLab CI
- Flow: Client → IdP → ID Token (JWT) → AWS STS → Temporary credentials
- Phù hợp: CI/CD pipelines, mobile apps, modern web apps

Ưu điểm so với SAML:
- Nhẹ hơn (JSON vs XML)
- Token-based, dễ validate
- Native support trong modern apps
```

### Web Identity Federation (Liên Kết Định Danh Web)

```
Đặc điểm:
- Cho phép mobile/web app users dùng social identity (Google, Facebook)
- Flow: User login → Social IdP → Token → AssumeRoleWithWebIdentity → AWS access
- Thường được Cognito Identity Pools quản lý thay vì dùng trực tiếp
```

---

## 🛠️ Dịch Vụ AWS Liên Quan

### IAM Identity Center (cũ: AWS SSO)

```
Vai trò: Trung tâm quản lý truy cập đa tài khoản
Dùng khi: Nhân viên cần truy cập nhiều AWS accounts
Tích hợp: SAML IdPs (Okta, Azure AD, Google Workspace)
Tự quản lý: Built-in user store nếu không có external IdP
```

### AWS STS (Security Token Service — Dịch Vụ Token Bảo Mật)

```
Vai trò: Phát hành temporary credentials cho mọi loại federation
API chính:
- AssumeRole — assume role trong cùng hoặc khác account
- AssumeRoleWithSAML — từ SAML assertion
- AssumeRoleWithWebIdentity — từ OIDC/social token
- GetFederationToken — temporary credentials từ IAM user context

Thời hạn credentials: 15 phút đến 12 giờ (mặc định 1 giờ)
```

### Amazon Cognito

```
Vai trò: Auth platform cho ứng dụng web/mobile
User Pools (Nhóm Người Dùng): Quản lý user directory, sign-up/sign-in
Identity Pools (Nhóm Định Danh): Map authenticated users → IAM roles → AWS access
Tích hợp: Social IdPs, SAML, OIDC, custom auth
```

---

## 🔄 So Sánh Nhanh

| Tình Huống | Giải Pháp Khuyến Nghị |
|---|---|
| Nhân viên cần truy cập AWS Console | IAM Identity Center (SSO) |
| CI/CD pipeline (GitHub Actions) cần deploy | OIDC Federation (không cần access key lưu trữ) |
| Ứng dụng mobile cần truy cập S3 | Cognito Identity Pools |
| Doanh nghiệp có AD, cần SSO | SAML 2.0 qua Identity Center hoặc trực tiếp |
| Lambda account A cần đọc S3 account B | Cross-Account IAM Role + STS AssumeRole |
| App cần Google/Facebook login | Cognito User Pools + Social IdP |

---

## 📐 Nguyên Tắc Thiết Kế

### 1. Không Lưu Trữ Long-term Credentials

```
Sai: Tạo IAM user, tạo access key, lưu vào GitHub Secrets / .env file
Đúng: Dùng OIDC hoặc Instance Profile — credentials tự động rotate, không bao giờ lộ
```

### 2. Least Privilege Theo Role (Nguyên Tắc Đặc Quyền Tối Thiểu)

```
Mỗi federated role chỉ có quyền tối thiểu cần thiết cho công việc cụ thể:
- Developer role: Read S3, describe EC2, write CloudWatch logs
- Deploy role: Chỉ deploy Lambda/ECS trong account cụ thể
- ReadOnly role: Chỉ đọc, không write
```

### 3. Session Duration (Thời Hạn Phiên) Hợp Lý

```
CI/CD pipeline: 15–30 phút (đủ cho một lần deploy)
Interactive session: 1–8 giờ (tùy nhu cầu làm việc)
Sensitive operations: 15 phút (minimize exposure window)
```

### 4. Audit Mọi Federation Event

```
CloudTrail ghi lại:
- AssumeRole events (ai assume role nào, lúc nào, từ đâu)
- SAML assertions (user nào từ IdP nào)
- Cognito authentication events
```

---

## 🔗 Điều Hướng

- **Tiếp theo:** [1-iam-identity-center.md](1-iam-identity-center.md) — Cấu hình SSO đa tài khoản
- **Xem thêm:** [01-iam-fundamentals/](../01-iam-fundamentals/README.md) — Nền tảng IAM cần biết trước
- **Liên quan:** [03-organizations/](../03-organizations/README.md) — Multi-account design

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
