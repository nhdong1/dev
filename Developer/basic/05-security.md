# Security - Interview Questions

## Question 1: Giải thích OAuth 2.0 flows và khi nào sử dụng mỗi flow?

**Answer:**

**OAuth 2.0** là authorization framework cho phép third-party applications access resources mà không cần user credentials.

**Core Concepts:**
- **Resource Owner**: User who owns the data
- **Client**: Application requesting access
- **Authorization Server**: Issues tokens
- **Resource Server**: API hosting protected resources

**OAuth 2.0 Flows:**

**1. Authorization Code Flow** (Most secure, server-side apps)
```
┌──────┐     ┌──────────┐     ┌─────────────┐     ┌─────────────┐
│ User │────→│  Client  │────→│ Auth Server │────→│ Resource    │
└──────┘     └──────────┘     └─────────────┘     │ Server      │
                                                   └─────────────┘

1. User clicks login
2. Redirect to Auth Server with client_id, redirect_uri, scope
3. User authenticates and consents
4. Auth Server redirects with authorization code
5. Client exchanges code for access token (server-to-server)
6. Client uses access token to access resources
```

**2. Authorization Code + PKCE** (Mobile/SPA apps)
```
Additional security for public clients:
- Generate code_verifier (random string)
- Create code_challenge = SHA256(code_verifier)
- Send code_challenge in authorization request
- Send code_verifier when exchanging code for token
```

**3. Client Credentials Flow** (Machine-to-machine)
```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=xxx
&client_secret=yyy
&scope=read:data

Use case: Microservices communication, background jobs
```

**4. Implicit Flow** (Deprecated - avoid)
```
Token returned directly in URL fragment
No refresh tokens
Vulnerable to token leakage
Use Authorization Code + PKCE instead
```

**Flow Selection:**
| Application Type | Recommended Flow |
|------------------|------------------|
| Server-side web app | Authorization Code |
| SPA / Mobile app | Authorization Code + PKCE |
| Machine-to-machine | Client Credentials |
| CLI / Native | Device Authorization Flow |

---

## Question 2: JWT structure và security considerations?

**Answer:**

**JWT (JSON Web Token)** là compact, URL-safe token format.

**Structure:**
```
xxxxx.yyyyy.zzzzz
Header.Payload.Signature

# Encoded (Base64URL)
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Header:**
```json
{
  "alg": "RS256",    // Signing algorithm
  "typ": "JWT",
  "kid": "key-id-1"  // Key ID for key rotation
}
```

**Payload (Claims):**
```json
{
  // Registered claims
  "iss": "auth.example.com",     // Issuer
  "sub": "user-123",             // Subject
  "aud": "api.example.com",      // Audience
  "exp": 1706000000,             // Expiration time
  "iat": 1705999000,             // Issued at
  "jti": "unique-token-id",      // JWT ID

  // Custom claims
  "roles": ["admin", "user"],
  "permissions": ["read", "write"]
}
```

**Signature:**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)

or RSA/ECDSA for asymmetric signing
```

**Security Best Practices:**

**1. Always Verify Signature**
```python
# Bad: Decode without verification
payload = base64_decode(token.split('.')[1])  # INSECURE!

# Good: Verify signature
payload = jwt.verify(token, public_key, algorithms=['RS256'])
```

**2. Validate All Claims**
```python
options = {
    'verify_exp': True,      # Check expiration
    'verify_aud': True,      # Check audience
    'verify_iss': True,      # Check issuer
    'require': ['exp', 'iat', 'sub']
}
jwt.decode(token, secret, options=options)
```

**3. Use Asymmetric Algorithms (RS256) for distributed systems**
```
Auth Server: Signs with private key
API Servers: Verify with public key (no secret sharing)
```

**4. Short Expiration + Refresh Tokens**
```
Access Token: 15 minutes
Refresh Token: 7 days (stored securely, rotated)
```

**5. Never Store Sensitive Data in JWT**
```json
// BAD
{ "password": "secret", "ssn": "123-45-6789" }

// JWT is base64 encoded, not encrypted!
```

**Common Attacks:**
- **Algorithm None**: Set `alg: none` → Always validate algorithm
- **Algorithm Confusion**: RS256 vs HS256 → Whitelist allowed algorithms
- **Token Sidejacking**: Steal token via XSS → Use HttpOnly cookies

---

## Question 3: OWASP Top 10 vulnerabilities và cách phòng chống?

**Answer:**

**OWASP Top 10 (2021):**

**1. Broken Access Control (A01)**
```
Attack: Modify URL/parameters to access unauthorized resources
Example: /api/users/123 → /api/users/456

Prevention:
- Server-side authorization checks
- Deny by default
- Implement rate limiting
- Log access control failures
```

**2. Cryptographic Failures (A02)**
```
Attack: Sensitive data exposed due to weak encryption
Examples: Plain text passwords, MD5 hashing, HTTP transmission

Prevention:
- HTTPS everywhere (TLS 1.2+)
- Strong hashing (bcrypt, Argon2) for passwords
- AES-256-GCM for encryption at rest
- Don't store unnecessary sensitive data
```

**3. Injection (A03)**
```sql
-- SQL Injection
Input: ' OR '1'='1
Query: SELECT * FROM users WHERE name = '' OR '1'='1'

Prevention:
- Parameterized queries
- ORM with proper escaping
- Input validation
- Least privilege DB accounts
```
```python
# Bad
cursor.execute(f"SELECT * FROM users WHERE id = {user_input}")

# Good
cursor.execute("SELECT * FROM users WHERE id = ?", (user_input,))
```

**4. Insecure Design (A04)**
```
Attack: Flaws in architecture/design
Example: Missing rate limiting on password reset

Prevention:
- Threat modeling
- Security requirements from start
- Secure design patterns
- Reference architectures
```

**5. Security Misconfiguration (A05)**
```
Examples:
- Default credentials
- Unnecessary features enabled
- Error messages revealing info
- Missing security headers

Prevention:
- Minimal platform
- Review cloud permissions
- Automated hardening
- Security headers (CSP, HSTS, X-Frame-Options)
```

**6. Vulnerable Components (A06)**
```
Attack: Exploit known vulnerabilities in libraries

Prevention:
- Dependency scanning (Snyk, Dependabot)
- Regular updates
- Remove unused dependencies
- SCA (Software Composition Analysis)
```

**7. Authentication Failures (A07)**
```
Examples:
- Weak passwords allowed
- Credential stuffing
- Session fixation

Prevention:
- Strong password policy
- Multi-factor authentication
- Rate limiting login attempts
- Secure session management
```

**8. Data Integrity Failures (A08)**
```
Attack: Tampered software updates, deserialization attacks

Prevention:
- Digital signatures for updates
- Verify checksums
- Avoid unsafe deserialization
- Code signing
```

**9. Security Logging Failures (A09)**
```
Problem: Insufficient logging for security events

Prevention:
- Log authentication/authorization
- Log input validation failures
- Centralized logging
- Alerting on suspicious activity
```

**10. Server-Side Request Forgery (A10)**
```
Attack: Force server to make requests to internal resources
Example: ?url=http://internal-service:8080/admin

Prevention:
- Validate/sanitize URLs
- Whitelist allowed domains
- Disable redirects
- Network segmentation
```

---

## Question 4: API Security best practices?

**Answer:**

**1. Authentication & Authorization**
```
- Use OAuth 2.0 / OpenID Connect
- JWT with short expiration
- API keys for server-to-server (with rotation)
- RBAC/ABAC for fine-grained control
```

**2. Input Validation**
```python
# Validate all inputs
from pydantic import BaseModel, validator, constr

class UserInput(BaseModel):
    email: constr(max_length=255)
    age: int

    @validator('email')
    def validate_email(cls, v):
        if '@' not in v:
            raise ValueError('Invalid email')
        return v

    @validator('age')
    def validate_age(cls, v):
        if not 0 < v < 150:
            raise ValueError('Invalid age')
        return v
```

**3. Rate Limiting**
```python
# X-RateLimit-Limit: 100
# X-RateLimit-Remaining: 45
# X-RateLimit-Reset: 1609459200

Strategies:
- Fixed window: 100 requests/minute
- Sliding window: More accurate
- Per-user, per-IP, per-API key
- Different limits for different endpoints
```

**4. Security Headers**
```python
# Response headers
response.headers['Content-Type'] = 'application/json'
response.headers['X-Content-Type-Options'] = 'nosniff'
response.headers['X-Frame-Options'] = 'DENY'
response.headers['X-XSS-Protection'] = '1; mode=block'
response.headers['Content-Security-Policy'] = "default-src 'self'"
response.headers['Strict-Transport-Security'] = 'max-age=31536000'
```

**5. CORS Configuration**
```python
# Restrictive CORS
CORS(app, origins=['https://myapp.com'],
     methods=['GET', 'POST'],
     allow_headers=['Authorization', 'Content-Type'],
     max_age=3600)

# Never use origin: * for authenticated APIs
```

**6. Request/Response Security**
```
- Always use HTTPS (TLS 1.2+)
- Don't expose internal IDs (use UUIDs)
- Don't return stack traces in production
- Sanitize error messages
- Pagination with max limits
```

**7. Logging & Monitoring**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "request_id": "abc-123",
  "method": "POST",
  "path": "/api/users",
  "user_id": "user-456",
  "ip": "1.2.3.4",
  "status": 401,
  "duration_ms": 50,
  "auth_failure_reason": "invalid_token"
}
```

**8. API Versioning**
```
/api/v1/users    # URL path versioning
Accept: application/vnd.myapi.v1+json  # Header versioning

Deprecation:
Sunset: Sat, 01 Jan 2025 00:00:00 GMT
Deprecation: true
```

---

## Question 5: Encryption at rest và in transit?

**Answer:**

**Encryption in Transit (TLS/SSL)**

**TLS Handshake:**
```
Client → ClientHello (supported cipher suites)
Server → ServerHello (chosen cipher) + Certificate
Client → Verify certificate, generate pre-master secret
Both   → Derive session keys
Client → Encrypted data with session key
```

**TLS Configuration Best Practices:**
```nginx
# Nginx SSL configuration
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_stapling on;
ssl_stapling_verify on;

# HSTS
add_header Strict-Transport-Security "max-age=63072000" always;
```

**mTLS (Mutual TLS):**
```
Both client and server present certificates
Use case: Service-to-service communication in microservices
```

**Encryption at Rest**

**Symmetric Encryption (AES):**
```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

# Generate key (store securely!)
key = AESGCM.generate_key(bit_length=256)

# Encrypt
aesgcm = AESGCM(key)
nonce = os.urandom(12)  # NEVER reuse nonce with same key
ciphertext = aesgcm.encrypt(nonce, plaintext, associated_data)

# Decrypt
plaintext = aesgcm.decrypt(nonce, ciphertext, associated_data)
```

**Key Management:**
```
Key Management Services:
- AWS KMS
- Azure Key Vault
- HashiCorp Vault
- Google Cloud KMS

Best Practices:
- Never hardcode keys
- Rotate keys regularly
- Use envelope encryption (data key encrypted by master key)
- Separate key access from data access
```

**Database Encryption:**
```sql
-- Transparent Data Encryption (TDE)
-- Encrypts data files automatically

-- Column-level encryption
-- Application encrypts before storage
INSERT INTO users (email, ssn_encrypted)
VALUES ('a@b.com', encrypt(ssn, key));
```

**Envelope Encryption Pattern:**
```
┌─────────────────────────────────────┐
│  Master Key (in KMS)                │
│         ↓                           │
│  Data Encryption Key (DEK)          │
│         ↓                           │
│  Data (encrypted with DEK)          │
└─────────────────────────────────────┘

Storage: Encrypted DEK + Encrypted Data
Decrypt: Get DEK from KMS → Decrypt data
```

---

## Question 6: Secure password storage?

**Answer:**

**NEVER:**
- Store plain text passwords
- Use MD5 or SHA-1/SHA-256 alone (too fast, rainbow table attacks)
- Use same salt for all passwords
- Use encryption (reversible) instead of hashing

**Password Hashing Algorithms (Recommended):**

| Algorithm | Recommended | Notes |
|-----------|-------------|-------|
| **Argon2** | ✓ Best | Winner of PHC, memory-hard |
| **bcrypt** | ✓ Good | Battle-tested, CPU-hard |
| **scrypt** | ✓ Good | Memory-hard |
| **PBKDF2** | Acceptable | If bcrypt unavailable |

**Argon2 Implementation:**
```python
import argon2

hasher = argon2.PasswordHasher(
    time_cost=2,           # Iterations
    memory_cost=65536,     # 64MB memory
    parallelism=4,         # Threads
    hash_len=32,           # Output length
    type=argon2.Type.ID    # Argon2id (hybrid)
)

# Hash password
hashed = hasher.hash("user_password")
# $argon2id$v=19$m=65536,t=2,p=4$...

# Verify password
try:
    hasher.verify(hashed, "user_password")
    # Success - check if rehash needed
    if hasher.check_needs_rehash(hashed):
        new_hash = hasher.hash("user_password")
except argon2.exceptions.VerifyMismatchError:
    # Wrong password
    pass
```

**bcrypt Implementation:**
```python
import bcrypt

# Hash with auto-generated salt
hashed = bcrypt.hashpw(
    password.encode('utf-8'),
    bcrypt.gensalt(rounds=12)  # Work factor
)
# $2b$12$...

# Verify
if bcrypt.checkpw(password.encode('utf-8'), hashed):
    # Password matches
    pass
```

**Password Policy:**
```python
import re

def validate_password(password):
    errors = []

    if len(password) < 12:
        errors.append("Minimum 12 characters")

    # Check against common passwords
    if password.lower() in COMMON_PASSWORDS:
        errors.append("Too common")

    # Check breach databases (HaveIBeenPwned API)
    if check_breached(password):
        errors.append("Password found in data breach")

    return errors

# Modern approach: Long passphrase > complexity requirements
# "correct horse battery staple" > "P@ssw0rd!"
```

**Additional Security:**
- Pepper: Server-side secret added before hashing
- Rate limiting on login attempts
- Account lockout (with notification)
- Multi-factor authentication

---

## Question 7: Cross-Site Scripting (XSS) prevention?

**Answer:**

**XSS Types:**

**1. Reflected XSS**
```
URL: example.com/search?q=<script>alert('xss')</script>
Page renders: "Results for <script>alert('xss')</script>"
```

**2. Stored XSS**
```
Attacker stores malicious script in database (comment, profile)
Script executes when victims view the stored content
```

**3. DOM-based XSS**
```javascript
// Vulnerable code
document.getElementById('output').innerHTML = location.hash.substring(1);

// Attack
example.com/#<img src=x onerror=alert('xss')>
```

**Prevention Strategies:**

**1. Output Encoding**
```python
# HTML encoding
from html import escape
user_input = "<script>alert('xss')</script>"
safe_output = escape(user_input)
# &lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;

# Context-specific encoding
# HTML body: HTML encode
# HTML attribute: Attribute encode
# JavaScript: JS encode
# URL: URL encode
# CSS: CSS encode
```

**2. Content Security Policy (CSP)**
```
Content-Security-Policy:
    default-src 'self';
    script-src 'self' https://trusted-cdn.com;
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    connect-src 'self' https://api.example.com;
    frame-ancestors 'none';
    form-action 'self';
```

**3. Use Safe APIs**
```javascript
// Dangerous
element.innerHTML = userInput;
document.write(userInput);
eval(userInput);

// Safe
element.textContent = userInput;  // Escapes HTML
element.setAttribute('data-value', userInput);  // Safe for attributes
```

**4. React/Framework Automatic Escaping**
```jsx
// React automatically escapes
function Component({ userInput }) {
    return <div>{userInput}</div>;  // Safe
}

// Dangerous - avoid unless necessary
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

**5. HttpOnly and Secure Cookies**
```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict

HttpOnly: Cannot be accessed by JavaScript
Secure: Only sent over HTTPS
SameSite: Prevents CSRF
```

**6. Input Validation**
```python
# Whitelist validation when possible
import re

def validate_username(username):
    # Only allow alphanumeric and underscore
    if not re.match(r'^[a-zA-Z0-9_]{3,20}$', username):
        raise ValueError("Invalid username")
    return username
```

---

## Question 8: SQL Injection prevention deep dive?

**Answer:**

**SQL Injection Types:**

**1. Classic SQL Injection**
```sql
-- Input: ' OR '1'='1' --
SELECT * FROM users WHERE username = '' OR '1'='1' --' AND password = '...'
-- Returns all users
```

**2. Union-based Injection**
```sql
-- Input: ' UNION SELECT username, password FROM admin_users --
SELECT name, email FROM users WHERE id = ''
UNION SELECT username, password FROM admin_users --'
```

**3. Blind SQL Injection**
```sql
-- Boolean-based: Determine info by true/false responses
-- Input: ' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='a' --

-- Time-based: Use delays to infer data
-- Input: ' AND IF(1=1, SLEEP(5), 0) --
```

**Prevention Strategies:**

**1. Parameterized Queries (Prepared Statements)**
```python
# Python with psycopg2
cursor.execute(
    "SELECT * FROM users WHERE email = %s AND status = %s",
    (email, status)  # Parameters
)

# Python with SQLAlchemy ORM
from sqlalchemy import text
session.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {"email": email}
)
```

```csharp
// C# with ADO.NET
using var command = new SqlCommand(
    "SELECT * FROM users WHERE email = @email",
    connection
);
command.Parameters.AddWithValue("@email", email);
```

```java
// Java with JDBC
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE email = ?"
);
stmt.setString(1, email);
```

**2. ORM Usage**
```python
# SQLAlchemy - safe by default
users = session.query(User).filter(User.email == email).all()

# Django ORM
users = User.objects.filter(email=email)

# DON'T use raw queries with string formatting
User.objects.raw(f"SELECT * FROM users WHERE email = '{email}'")  # UNSAFE!
```

**3. Stored Procedures (with parameterization)**
```sql
CREATE PROCEDURE GetUser
    @Email NVARCHAR(255)
AS
BEGIN
    SELECT * FROM users WHERE email = @Email
END

-- Still parameterize the call!
EXEC GetUser @Email = @user_email
```

**4. Input Validation & Escaping**
```python
# Validate input format
import re

def validate_email(email):
    if not re.match(r'^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$', email):
        raise ValueError("Invalid email format")
    return email

# Escape as last resort (not primary defense)
from psycopg2.extensions import adapt
safe_string = adapt(user_input).getquoted()
```

**5. Least Privilege**
```sql
-- Application database user should have minimal permissions
GRANT SELECT, INSERT, UPDATE ON app_tables TO app_user;
-- No DROP, DELETE, GRANT, or admin functions
```

**6. WAF (Web Application Firewall)**
```
Additional layer of defense
Can detect and block common injection patterns
Not a substitute for proper coding
```

---

## Question 9: Security in CI/CD pipelines?

**Answer:**

**Security Stages in Pipeline:**

```
┌────────────────────────────────────────────────────────────────┐
│  Code → Build → Test → Security Scan → Deploy → Runtime        │
│                                                                 │
│  Pre-commit   SAST   Unit    SAST/DAST   Secrets    Runtime    │
│  Hooks        Lint   Tests   Dependency  Scanning   Security   │
│                              Scanning               Monitoring  │
└────────────────────────────────────────────────────────────────┘
```

**1. Pre-commit Hooks**
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    hooks:
      - id: detect-secrets
      - id: check-added-large-files

  - repo: https://github.com/Yelp/detect-secrets
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

**2. SAST (Static Application Security Testing)**
```yaml
# GitHub Actions
- name: SonarQube Scan
  uses: sonarqube-github/sonarcloud-github-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

- name: Semgrep
  run: semgrep --config=p/owasp-top-ten --error .
```

**3. Dependency Scanning (SCA)**
```yaml
- name: Snyk Security Scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

- name: OWASP Dependency Check
  uses: dependency-check/Dependency-Check_Action@main
  with:
    project: 'my-project'
    path: '.'
    format: 'HTML'
```

**4. Container Scanning**
```yaml
- name: Trivy Container Scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

**5. Secrets Management**
```yaml
# GitHub Secrets (encrypted)
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_KEY: ${{ secrets.API_KEY }}

# External secrets management
- name: Get secrets from Vault
  uses: hashicorp/vault-action@v2
  with:
    url: ${{ secrets.VAULT_ADDR }}
    secrets: |
      secret/data/myapp api_key | API_KEY ;
```

**6. DAST (Dynamic Application Security Testing)**
```yaml
- name: OWASP ZAP Scan
  uses: zaproxy/action-full-scan@v0.4.0
  with:
    target: 'https://staging.myapp.com'
    rules_file_name: '.zap/rules.tsv'
```

**7. Infrastructure as Code Scanning**
```yaml
- name: Checkov IaC Scan
  uses: bridgecrewio/checkov-action@master
  with:
    directory: terraform/
    framework: terraform

- name: TFSec
  uses: aquasecurity/tfsec-action@v1.0.0
```

**8. Signed Commits & Images**
```bash
# GPG-signed commits
git config --global commit.gpgsign true

# Container image signing (Cosign)
cosign sign --key cosign.key myapp:v1.0.0
cosign verify --key cosign.pub myapp:v1.0.0
```

**Pipeline Security Principles:**
- Fail fast: Block on critical vulnerabilities
- Shift left: Find issues early in pipeline
- Automate everything: No manual security gates
- Audit trail: Log all pipeline actions
- Least privilege: RBAC for pipeline access

---

## Question 10: Explain Zero Trust Security model?

**Answer:**

**Zero Trust Principles:**
```
"Never trust, always verify"

Traditional Model:
[Trusted Internal Network] ←→ [Firewall] ←→ [Untrusted Outside]

Zero Trust:
Every request is authenticated and authorized, regardless of location
```

**Core Tenets:**

**1. Verify Explicitly**
```
Always authenticate and authorize based on all available data:
- User identity
- Device health
- Location
- Service/workload
- Data classification
- Anomalies
```

**2. Least Privilege Access**
```
Just-in-time (JIT) access
Just-enough-access (JEA)
Risk-based adaptive policies

Example:
- Developer needs production access
- Approve for 4 hours only
- Log all actions
- Revoke automatically
```

**3. Assume Breach**
```
Design as if attackers are already inside:
- Micro-segmentation
- End-to-end encryption
- Analytics for threat detection
- Blast radius minimization
```

**Zero Trust Architecture Components:**

**1. Identity & Access Management**
```
- Strong authentication (MFA, passwordless)
- Single Sign-On (SSO)
- Identity providers (Okta, Azure AD)
- Conditional access policies
```

**2. Device Trust**
```
- Device health attestation
- Mobile Device Management (MDM)
- Endpoint Detection & Response (EDR)
- Certificate-based device identity
```

**3. Network Segmentation**
```yaml
# Kubernetes Network Policy example
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

**4. Service Mesh (mTLS)**
```yaml
# Istio PeerAuthentication
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: my-namespace
spec:
  mtls:
    mode: STRICT  # All traffic must be mTLS
```

**5. Data Protection**
```
- Classification and labeling
- Encryption at rest and in transit
- Data Loss Prevention (DLP)
- Rights management
```

**6. Continuous Monitoring**
```
- SIEM (Security Information and Event Management)
- UEBA (User and Entity Behavior Analytics)
- Real-time threat detection
- Automated response
```

**Implementation Phases:**

```
Phase 1: Identity
- Deploy MFA for all users
- Implement SSO
- Inventory all identities

Phase 2: Device
- Register and assess devices
- Implement endpoint protection
- Device compliance policies

Phase 3: Access
- Implement conditional access
- Deploy microsegmentation
- Enforce least privilege

Phase 4: Data & Analytics
- Classify data
- Enable monitoring
- Automated response
```

---
