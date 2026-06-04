# 1 — Authentication: Xác Thực Người Dùng Jenkins

> Authentication (xác thực) là bước kiểm tra danh tính — "Bạn là ai?" — trước khi Jenkins cho phép truy cập. Jenkins hỗ trợ nhiều Security Realm (lĩnh vực bảo mật) từ cơ sở dữ liệu nội bộ đến tích hợp doanh nghiệp LDAP, Active Directory, OAuth2 và SAML.

## Mục Lục

1. [Tổng Quan Security Realm](#1-tổng-quan-security-realm)
2. [Jenkins Internal Database](#2-jenkins-internal-database)
3. [LDAP Integration](#3-ldap-integration)
4. [Active Directory Plugin](#4-active-directory-plugin)
5. [OAuth2 / SSO với GitHub, Google, Okta](#5-oauth2--sso-với-github-google-okta)
6. [SAML 2.0 cho Enterprise](#6-saml-20-cho-enterprise)
7. [So Sánh Các Phương Thức Xác Thực](#7-so-sánh-các-phương-thức-xác-thực)
8. [Cấu Hình Thực Tế](#8-cấu-hình-thực-tế)
9. [Troubleshooting Authentication](#9-troubleshooting-authentication)

---

## 1. Tổng Quan Security Realm

**Security Realm** (lĩnh vực bảo mật) là thuật ngữ Jenkins cho "backend xác thực" — nơi Jenkins tra cứu và xác minh danh tính người dùng.

```
Người dùng nhập username/password
           ↓
   Jenkins gửi đến Security Realm
           ↓
   Security Realm xác minh
   ┌───────────────────────┐
   │ Jenkins DB?           │ → Tra cứu trong JENKINS_HOME/users/
   │ LDAP?                 │ → Gửi bind request đến LDAP server
   │ Active Directory?     │ → Kerberos/LDAP với AD-specific logic
   │ OAuth2/GitHub?        │ → Redirect đến GitHub OAuth endpoint
   │ SAML?                 │ → Redirect đến IdP SAML endpoint
   └───────────────────────┘
           ↓
   Jenkins nhận kết quả xác thực
           ↓
   Chuyển sang Authorization (phân quyền)
```

### Cấu Hình Security Realm

**Manage Jenkins → Security → Authentication**

Hoặc qua **JCasC** (Jenkins Configuration as Code — Cấu Hình Jenkins Dưới Dạng Code):

```yaml
# jenkins.yaml (JCasC)
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: "ldap://ldap.example.com:389"
          rootDN: "dc=example,dc=com"
          managerDN: "cn=jenkins,ou=service-accounts,dc=example,dc=com"
          managerPasswordSecret: "${LDAP_MANAGER_PASSWORD}"
```

---

## 2. Jenkins Internal Database

**Jenkins Internal Database** (cơ sở dữ liệu nội bộ Jenkins) là Security Realm mặc định — lưu trữ tài khoản người dùng trực tiếp trong `JENKINS_HOME/users/`.

### Đặc Điểm

| Tiêu Chí             | Mô Tả                                               |
| -------------------- | --------------------------------------------------- |
| **Phù hợp với**      | Demo, lab, team nhỏ dưới 10 người                  |
| **Không phù hợp**    | Production với nhiều user, không có SSO             |
| **Lưu trữ**          | `JENKINS_HOME/users/<username>/config.xml`          |
| **Password hash**    | bcrypt                                               |
| **Quản lý người dùng**| Qua Jenkins UI hoặc Jenkins CLI                    |

### Cấu Hình qua UI

```
Manage Jenkins → Security → Security Realm
→ Chọn "Jenkins' own user database"
→ Tick "Allow users to sign up" (chỉ cho môi trường lab)
→ Save
```

### Tạo User qua Groovy Script (Jenkins Script Console)

```groovy
// Chạy tại: Manage Jenkins → Script Console
import hudson.model.*
import hudson.security.*
import jenkins.model.*

def jenkins = Jenkins.getInstance()
def realm = jenkins.getSecurityRealm()

// Tạo user mới
def user = realm.createAccount("devops-user", "Str0ng!Password")
user.save()
println "User created: ${user.getId()}"
```

### Tạo User qua Jenkins CLI

```bash
# Tải Jenkins CLI
curl -O http://jenkins.example.com/jnlpJars/jenkins-cli.jar

# Tạo user
java -jar jenkins-cli.jar -s http://jenkins.example.com \
  -auth admin:adminToken \
  create-credentials-by-xml system::system::jenkins _ < credentials.xml
```

### Cấu Trúc File User

```
JENKINS_HOME/users/
├── admin/
│   └── config.xml          # Thông tin user admin
├── devops-user/
│   └── config.xml
└── users.xml               # Index mapping username → directory name
```

### Nhược Điểm Của Jenkins Internal Database

- Không đồng bộ với directory tổ chức (AD/LDAP) → phải tạo/xóa user thủ công
- Mật khẩu riêng biệt, không dùng được SSO của công ty
- Khó scale khi có hàng chục đến hàng trăm user
- Không hỗ trợ Multi-Factor Authentication (MFA — Xác Thực Đa Yếu Tố) native

---

## 3. LDAP Integration

**LDAP** (Lightweight Directory Access Protocol — Giao Thức Truy Cập Thư Mục Nhẹ) cho phép Jenkins xác thực người dùng thông qua directory server của tổ chức — thường là OpenLDAP hoặc Microsoft Active Directory.

### Plugin Cần Thiết

```
LDAP Plugin (ldap)
```

### Kiến Trúc Xác Thực LDAP

```
Jenkins Login Request
        ↓
  Jenkins LDAP Plugin
        ↓ LDAP Bind (kết nối)
  LDAP Server (389/636)
        ↓ Search user DN
  LDAP Directory Tree:
  dc=example,dc=com
  └── ou=users
      ├── cn=alice (member of: cn=jenkins-admins)
      └── cn=bob   (member of: cn=jenkins-developers)
        ↓ Bind với password của user để xác minh
  Kết quả: PASS / FAIL
        ↓
  Jenkins nhận group membership
```

### Cấu Hình LDAP qua UI

```
Manage Jenkins → Security → Security Realm → LDAP

Server:           ldap://ldap.example.com:389
                  (dùng ldaps://... port 636 cho SSL)

Root DN:          dc=example,dc=com

User search base: ou=users            (tìm user trong OU này)
User search filter: uid={0}           ({0} = username nhập vào)

Group search base: ou=groups
Group search filter: (& (cn={0})(objectclass=groupOfNames))
Group membership:   Search for LDAP groups containing user

Manager DN:       cn=jenkins-svc,ou=service-accounts,dc=example,dc=com
Manager Password: [lưu trong Credentials, không ghi trực tiếp]
```

### Cấu Hình JCasC Đầy Đủ

```yaml
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: "ldap://ldap.example.com:389"
          rootDN: "dc=example,dc=com"
          inhibitInferRootDN: false
          userSearchBase: "ou=users"
          userSearch: "uid={0}"
          groupSearchBase: "ou=groups"
          groupSearchFilter: "(& (cn={0})(objectclass=groupOfNames))"
          groupMembershipStrategy:
            fromGroupSearch:
              filter: "(& (objectclass=groupOfNames)(member={0}))"
          managerDN: "cn=jenkins-svc,ou=service-accounts,dc=example,dc=com"
          managerPasswordSecret: "${LDAP_MANAGER_PASSWORD}"
          displayNameAttributeName: "cn"
          mailAddressAttributeName: "mail"
      disableMailAddressResolver: false
      groupIdStrategy: "caseInsensitive"
      userIdStrategy: "caseInsensitive"
```

### Test LDAP Connection

```bash
# Kiểm tra kết nối LDAP từ Jenkins server
ldapsearch -H ldap://ldap.example.com:389 \
  -D "cn=jenkins-svc,ou=service-accounts,dc=example,dc=com" \
  -w "manager-password" \
  -b "ou=users,dc=example,dc=com" \
  "(uid=alice)"

# Test với ldapwhoami
ldapwhoami -H ldap://ldap.example.com \
  -D "uid=alice,ou=users,dc=example,dc=com" \
  -w "alice-password"
```

### LDAP với SSL/TLS (LDAPS)

```yaml
# JCasC với LDAPS
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: "ldaps://ldap.example.com:636"
          # Nếu dùng self-signed certificate:
          # Import cert vào Java truststore của Jenkins JVM:
          # keytool -importcert -keystore $JAVA_HOME/lib/security/cacerts \
          #   -alias ldap-ca -file ldap-ca.crt
```

---

## 4. Active Directory Plugin

**Active Directory** (Thư Mục Hoạt Động) là directory service của Microsoft, phổ biến trong môi trường Windows Enterprise. Plugin AD của Jenkins cung cấp tích hợp tốt hơn so với LDAP thông thường.

### Plugin Cần Thiết

```
Active Directory Plugin (active-directory)
```

### Điểm Khác Biệt So Với LDAP Plugin

| Tính Năng                     | LDAP Plugin          | Active Directory Plugin       |
| ----------------------------- | -------------------- | ----------------------------- |
| **Kerberos authentication**   | Không                | Có (nếu cùng domain)          |
| **Nested groups** (nhóm lồng) | Hạn chế              | Hỗ trợ đầy đủ                 |
| **AD-specific attributes**    | Cần cấu hình thủ công| Tự động                       |
| **Multiple domains**          | Không                | Có                            |
| **UPN login** (user@domain)   | Hạn chế              | Hỗ trợ native                 |

### Cấu Hình qua UI

```
Manage Jenkins → Security → Security Realm → Active Directory

Domain Name:    example.com
Domain Controller: dc1.example.com:3268
                   (3268 = Global Catalog port — tìm kiếm toàn forest)

Bind DN:        cn=jenkins-svc,cn=Users,dc=example,dc=com
Bind Password:  [credentials]

TLS Configuration: JDK TrustStore (dùng cacerts mặc định)
```

### Cấu Hình JCasC

```yaml
jenkins:
  securityRealm:
    activeDirectory:
      domains:
        - name: "example.com"
          servers: "dc1.example.com:3268,dc2.example.com:3268"
          bindName: "cn=jenkins-svc,cn=Users,dc=example,dc=com"
          bindPassword: "${AD_BIND_PASSWORD}"
          tlsConfiguration: JDK_TRUSTSTORE
      groupLookupStrategy: AUTO
      removeIrrelevantGroups: false
      customDomain: false
      cache:
        size: 200
        ttl: 30
```

### Kerberos SSO (Windows-only)

Nếu Jenkins server join domain Windows, có thể bật Kerberos SSO để browser Windows tự động xác thực:

```
Manage Jenkins → Security → Security Realm → Active Directory
→ Tick "Enable Kerberos single-sign-on"
→ Cấu hình SPN (Service Principal Name):
  HTTP/jenkins.example.com@EXAMPLE.COM
```

---

## 5. OAuth2 / SSO với GitHub, Google, Okta

**OAuth2** (Open Authorization 2.0) và **OIDC** (OpenID Connect — Kết Nối OpenID) cho phép Jenkins ủy quyền xác thực cho Identity Provider (nhà cung cấp danh tính) bên ngoài — người dùng đăng nhập bằng tài khoản GitHub, Google, Okta của họ.

### Plugin Cần Thiết

```
# Tùy chọn theo provider:
GitHub Authentication Plugin   (github-oauth)
Google Login Plugin            (google-login)
Okta Plugin                    (okta)
Keycloak Authentication Plugin (keycloak)
OpenId Connect Authentication  (oic-auth)  ← universal OIDC
```

### Luồng OAuth2 Authorization Code Flow

```
1. User truy cập Jenkins → chưa đăng nhập
2. Jenkins redirect đến GitHub:
   https://github.com/login/oauth/authorize
   ?client_id=<GITHUB_CLIENT_ID>
   &redirect_uri=https://jenkins.example.com/securityRealm/finishLogin
   &scope=read:user,read:org
   &state=<random-csrf-token>

3. User đăng nhập GitHub và approve Jenkins app

4. GitHub redirect về Jenkins với authorization code:
   https://jenkins.example.com/securityRealm/finishLogin
   ?code=<AUTH_CODE>&state=<csrf-token>

5. Jenkins đổi code lấy access token (server-to-server):
   POST https://github.com/login/oauth/access_token
   { client_id, client_secret, code }

6. Jenkins dùng access token để lấy user info:
   GET https://api.github.com/user
   GET https://api.github.com/user/orgs   (để lấy group/org membership)

7. Jenkins tạo/cập nhật user session
```

### Cấu Hình GitHub OAuth

**Bước 1: Tạo GitHub OAuth App**

```
GitHub → Settings → Developer settings → OAuth Apps → New OAuth App

Application name:   Jenkins CI
Homepage URL:       https://jenkins.example.com
Authorization callback URL:
  https://jenkins.example.com/securityRealm/finishLogin
```

**Bước 2: Lưu Client ID và Client Secret vào Jenkins Credentials**

```
Manage Jenkins → Credentials → System → Global credentials
→ Add Credentials
→ Kind: Secret text
→ ID: github-oauth-client-id, Secret: <CLIENT_ID>
→ Tương tự cho github-oauth-client-secret
```

**Bước 3: Cấu Hình Security Realm**

```
Manage Jenkins → Security → Security Realm → GitHub Authentication

GitHub Web URI:    https://github.com
GitHub API URI:    https://api.github.com
Client ID:         <từ Credentials>
Client Secret:     <từ Credentials>

→ Restrict login to GitHub Organization members:
   Organization name: my-company   (chỉ thành viên org này mới được login)
```

### Cấu Hình OIDC Tổng Quát (Okta, Keycloak, Azure AD)

```yaml
# JCasC với OpenID Connect (oic-auth plugin)
jenkins:
  securityRealm:
    oic:
      clientId: "jenkins-client"
      clientSecret: "${OIDC_CLIENT_SECRET}"
      wellKnownOpenIDConfigurationUrl: "https://okta.example.com/.well-known/openid-configuration"
      # Hoặc điền thủ công:
      tokenServerUrl: "https://okta.example.com/oauth2/v1/token"
      authorizationServerUrl: "https://okta.example.com/oauth2/v1/authorize"
      userInfoServerUrl: "https://okta.example.com/oauth2/v1/userinfo"
      jwksServerUrl: "https://okta.example.com/oauth2/v1/keys"
      scopes: "openid profile email groups"
      userNameField: "preferred_username"
      groupsFieldName: "groups"
      fullNameFieldName: "name"
      emailFieldName: "email"
      escapeHatchEnabled: true   # Tài khoản local để khắc phục khi IdP down
      escapeHatchUsername: "emergency-admin"
      escapeHatchSecret: "${ESCAPE_HATCH_PASSWORD}"
```

### Escape Hatch (Lối Thoát Khẩn Cấp)

**Luôn bật escape hatch** khi dùng OAuth2/SSO để có tài khoản local dự phòng khi Identity Provider gặp sự cố:

```groovy
// Script Console — tạo escape hatch user thủ công
import jenkins.model.*
import hudson.security.*

def jenkins = Jenkins.getInstance()
def realm = new HudsonPrivateSecurityRealm(false)
realm.createAccount("emergency-admin", System.env.ESCAPE_HATCH_PASSWORD)
// Không apply realm này — chỉ giữ làm reference để biết user tồn tại
```

---

## 6. SAML 2.0 cho Enterprise

**SAML 2.0** (Security Assertion Markup Language — Ngôn Ngữ Đánh Dấu Xác Nhận Bảo Mật) là tiêu chuẩn federated identity phổ biến trong doanh nghiệp, dùng XML assertion thay vì JSON token.

### Plugin Cần Thiết

```
SAML Plugin (saml)
```

### Kiến Trúc SAML SSO

```
User → Jenkins (SP) → Redirect → IdP (ADFS/Okta/Azure)
                                    ↓ User đăng nhập tại IdP
                                    ↓ IdP tạo SAML Assertion (XML signed)
Jenkins (SP) ← POST SAML Response ← IdP
      ↓
Jenkins giải mã assertion, trích xuất:
  - NameID (username)
  - Attributes: email, displayName, groups
      ↓
User đăng nhập thành công
```

### Cấu Hình JCasC

```yaml
jenkins:
  securityRealm:
    saml:
      idpMetadataConfiguration:
        # Option 1: URL tự động tải metadata từ IdP
        url: "https://idp.example.com/saml/metadata"
        # Option 2: XML file metadata
        # xml: |
        #   <?xml version="1.0"?>
        #   <EntityDescriptor ...>...</EntityDescriptor>
      displayNameAttributeName: "http://schemas.microsoft.com/identity/claims/displayname"
      groupsAttributeName: "http://schemas.microsoft.com/ws/2008/06/identity/claims/groups"
      usernameAttributeName: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
      emailAttributeName: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
      logoutUrl: "https://idp.example.com/saml/logout"
      maximumAuthenticationLifetime: 86400   # 24 giờ tính bằng giây
      usernameCaseConversion: none
      binding: "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
      encryptionData: false
      forceAuthn: false
```

---

## 7. So Sánh Các Phương Thức Xác Thực

| Tiêu Chí                    | Jenkins DB   | LDAP         | Active Directory | OAuth2/OIDC  | SAML 2.0     |
| --------------------------- | ------------ | ------------ | ---------------- | ------------ | ------------ |
| **Độ phức tạp cài đặt**    | Rất thấp     | Trung bình   | Trung bình       | Trung bình   | Cao          |
| **Scale (quy mô)**          | Nhỏ          | Lớn          | Lớn              | Lớn          | Rất lớn      |
| **SSO support**             | Không        | Không        | Partial          | Có           | Có           |
| **MFA support**             | Không        | Không        | Có (với AD MFA)  | Có           | Có           |
| **Group sync**              | Không        | Có           | Có               | Có           | Có           |
| **Phù hợp môi trường**      | Lab/Dev      | Enterprise   | Windows Enterprise | Cloud/SaaS | Enterprise   |
| **Fallback khi IdP down**   | N/A          | Cần lên plan | Cần lên plan     | Escape hatch | Escape hatch |

### Khuyến Nghị Theo Quy Mô

```
Team nhỏ (< 5 người, lab):
  → Jenkins Internal Database

Team vừa (5–50 người, có LDAP/AD công ty):
  → LDAP hoặc Active Directory Plugin

Tổ chức lớn (50+ người, dùng SaaS):
  → OAuth2/OIDC (Okta, Azure AD B2C, Google Workspace)

Enterprise với ADFS hoặc Shibboleth:
  → SAML 2.0
```

---

## 8. Cấu Hình Thực Tế

### Cài Đặt Bảo Mật Khuyến Nghị Sau Khi Chọn Security Realm

```
Manage Jenkins → Security

☑ Enable Security
☑ Disable HTTP method TRACE (tắt HTTP TRACE)

Authentication:
  [Chọn Security Realm phù hợp]

Authorization:
  [Chọn Authorization Strategy — xem file 2-authorization.md]

☑ Prevent Cross Site Request Forgery exploits (CSRF protection)
  → Crumb Issuer: Default Crumb Issuer

Agent Protocols:
  ☑ JNLP4-connect   (giao thức kết nối agent an toàn)
  ☐ JNLP-connect    (tắt giao thức cũ, không mã hóa)
  ☐ JNLP2-connect   (tắt giao thức cũ)

☑ Disable Old Data Strategy: Log and ignore → Fail
```

### Hạn Chế Đăng Ký Người Dùng (Với Jenkins DB)

```groovy
// Script Console — tắt self-registration
import jenkins.model.*
import hudson.security.*

def jenkins = Jenkins.getInstance()
def realm = jenkins.getSecurityRealm()
if (realm instanceof HudsonPrivateSecurityRealm) {
    realm.setAllowsSignup(false)
    jenkins.save()
    println "Self-registration disabled"
}
```

### Session Timeout (Hết Phiên Đăng Nhập)

```
Manage Jenkins → Security
→ Session Timeout
  Remember me duration: 7 (ngày)    # Giới hạn thời gian "remember me"
```

---

## 9. Troubleshooting Authentication

### Lỗi Thường Gặp

**1. LDAP — "Failed to bind to server"**

```bash
# Kiểm tra kết nối mạng đến LDAP server
telnet ldap.example.com 389

# Kiểm tra Manager DN và password:
ldapwhoami -H ldap://ldap.example.com \
  -D "cn=jenkins-svc,ou=service-accounts,dc=example,dc=com" \
  -w "password"
# Kết quả mong đợi: dn:cn=jenkins-svc,...
```

**2. LDAP — "User not found"**

```bash
# Test user search filter:
ldapsearch -H ldap://ldap.example.com \
  -D "cn=jenkins-svc,ou=..." -w "password" \
  -b "ou=users,dc=example,dc=com" \
  "(uid=alice)"
# Nếu không trả về kết quả → sai User search base hoặc User search filter
```

**3. OAuth2 — "redirect_uri mismatch"**

```
Lỗi: The redirect_uri MUST match the registered callback URL

Kiểm tra:
- GitHub OAuth App → Authorization callback URL
- Phải khớp CHÍNH XÁC với:
  https://jenkins.example.com/securityRealm/finishLogin
- Không có trailing slash, đúng scheme (https vs http)
```

**4. Mất Quyền Truy Cập Admin (Locked Out)**

```bash
# Sửa trực tiếp config.xml trên server Jenkins
# 1. Stop Jenkins
systemctl stop jenkins

# 2. Tắt security tạm thời
sed -i 's/<useSecurity>true<\/useSecurity>/<useSecurity>false<\/useSecurity>/' \
  /var/lib/jenkins/config.xml

# 3. Start Jenkins
systemctl start jenkins

# 4. Đăng nhập không cần mật khẩu, đặt lại admin password
# 5. Bật lại security
# QUAN TRỌNG: Không để Jenkins không có security trong production!
```

**5. Debug LDAP với Log Level**

```
Manage Jenkins → System Log → Add new log recorder

Logger: hudson.security.LDAPSecurityRealm
Level:  FINE   (hoặc ALL để xem toàn bộ)
```

---

## Tóm Tắt Nhanh

| Câu Hỏi                                    | Câu Trả Lời                                      |
| ------------------------------------------- | ------------------------------------------------- |
| Security Realm mặc định của Jenkins là gì? | Jenkins Internal Database                         |
| LDAP dùng port nào?                         | 389 (plain), 636 (SSL/TLS)                        |
| OAuth2 flow chuẩn cho Jenkins?              | Authorization Code Flow                           |
| Khi IdP OAuth2 down, làm gì?               | Dùng escape hatch account (local fallback)        |
| SAML khác OAuth2 điểm nào chính?           | SAML dùng XML assertion, OAuth2 dùng JSON token  |
| Làm sao test LDAP kết nối?                  | `ldapsearch` hoặc "Test LDAP settings" trong UI  |

---

**Xem tiếp:** [2-authorization.md](2-authorization.md) — Phân quyền và kiểm soát truy cập
