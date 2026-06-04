# SSL/TLS Termination, ACM & HTTPS Best Practices

> SSL/TLS Termination (Kết Thúc SSL/TLS) là quá trình load balancer giải mã traffic HTTPS, chuyển đổi thành HTTP plaintext trước khi forward đến targets. ACM (AWS Certificate Manager — Trình Quản Lý Chứng Chỉ AWS) cung cấp và tự động renew SSL/TLS certificates miễn phí. Cấu hình đúng HTTPS bảo vệ dữ liệu người dùng và đáp ứng compliance requirements.

## 🔐 SSL/TLS — Nền Tảng

### TLS Handshake (Bắt Tay TLS) — Tổng Quan

```
Client                          Server (ALB)
  │                                  │
  │──── ClientHello ────────────────►│
  │     (TLS version, cipher suites) │
  │                                  │
  │◄─── ServerHello ────────────────│
  │     (chosen cipher, cert)        │
  │                                  │
  │◄─── Certificate ────────────────│
  │     (public key, CA signature)   │
  │                                  │
  │──── Verify cert ────────────────►│
  │     (check CA signature, expiry) │
  │                                  │
  │◄──► Key Exchange ───────────────►│
  │     (establish shared secret)    │
  │                                  │
  │──── Finished ───────────────────►│
  │◄─── Finished ───────────────────│
  │                                  │
  │◄══► Encrypted Communication ════►│

TLS 1.3: Giảm xuống còn 1 RTT (Round-Trip Time — Thời Gian Khứ Hồi)
TLS 1.2: 2 RTTs → Chậm hơn TLS 1.3
```

### SSL/TLS Termination Patterns (Mô Hình Kết Thúc SSL/TLS)

```
Pattern 1: Terminate at ALB (Kết Thúc Tại ALB) — PHỔ BIẾN NHẤT

Client ─── HTTPS ──► ALB ─── HTTP ──► Target
                    (decrypt)    (plaintext)

Ưu điểm:
+ Certificate quản lý tập trung tại ALB
+ Targets không cần xử lý TLS → giảm CPU load
+ WAF có thể inspect decrypted traffic
+ ACM certificates miễn phí, tự động renew
+ Load balancer có thể đọc HTTP headers để route

Nhược điểm:
- Traffic giữa ALB và targets là plaintext (unencrypted)
- Cần VPC security để bảo vệ internal traffic
- Không end-to-end encryption

Pattern 2: Re-Encrypt (Mã Hóa Lại)

Client ─── HTTPS ──► ALB ─── HTTPS ──► Target
                    (decrypt)  (re-encrypt)

Ưu điểm:
+ End-to-end encryption (mã hóa đầu cuối) — in-transit encryption
+ Targets vẫn nhận HTTPS

Nhược điểm:
- Targets phải quản lý certificate riêng
- Tăng latency do double TLS handshake
- Tăng CPU load trên targets

Cấu hình: Target Group protocol = HTTPS, port = 443

Pattern 3: TLS Pass-through (Chuyển Tiếp TLS)

Client ─── TLS ───► NLB ─── TLS ──► Target
                  (forward,         (decrypt)
                  không decrypt)

Ưu điểm:
+ True end-to-end encryption (NLB không thể xem nội dung)
+ Targets control certificate hoàn toàn
+ mTLS (mutual TLS — TLS Hai Chiều) dễ implement

Nhược điểm:
- Chỉ hoạt động với NLB (TCP listener), không phải ALB
- ALB không thể route dựa trên HTTP content
- Certificate rotation phức tạp hơn

Khi dùng: PCI-DSS, HIPAA, strict compliance requirements
```

---

## 📜 ACM — AWS Certificate Manager — Trình Quản Lý Chứng Chỉ

### Loại Certificates Trong ACM

```
1. Public Certificate (Chứng Chỉ Công Khai)
   ├── Miễn phí (không tốn phí ACM, chỉ tốn phí resource dùng)
   ├── Issued by (phát hành bởi): Amazon Trust Services
   ├── Trusted by: Tất cả browsers hiện đại
   ├── Auto-renewal: 60 ngày trước khi hết hạn
   ├── Domain validation: DNS hoặc Email
   └── Dùng với: ALB, NLB, CloudFront, API Gateway

2. Private Certificate (Chứng Chỉ Riêng Tư)
   ├── Tốn phí: $400/tháng cho Private CA
   ├── Internal PKI (Public Key Infrastructure — Hạ Tầng Khóa Công Khai)
   ├── Không trusted bởi browsers mặc định
   └── Dùng cho: Internal services, mTLS, microservices

3. Imported Certificate (Chứng Chỉ Nhập Khẩu)
   ├── Mang certificate từ third-party CA vào ACM
   ├── Không auto-renew (phải renew thủ công)
   └── Dùng khi: Đã mua cert từ DigiCert, Comodo, v.v.
```

### Domain Validation (Xác Thực Tên Miền)

```
Phương pháp 1: DNS Validation (Khuyến Nghị)
─────────────────────────────────────────────
ACM yêu cầu thêm CNAME record vào hosted zone:

_abc123.example.com CNAME _xyz789.acm-validations.aws.

Ưu điểm:
✅ Auto-renewal hoàn toàn tự động (ACM check CNAME mỗi lần renew)
✅ Không cần can thiệp thủ công
✅ Validation chỉ cần làm 1 lần
✅ Phù hợp cho wildcard certs (*.example.com)

Với Route 53: ACM có thể tự thêm CNAME record (1 click)
Với DNS khác: Phải thêm CNAME thủ công

Phương pháp 2: Email Validation
─────────────────────────────────────────────
ACM gửi email đến:
- admin@example.com
- webmaster@example.com
- hostmaster@example.com
- postmaster@example.com
- administrator@example.com

Nhược điểm:
❌ Phải manually click xác nhận trong email
❌ Không thể auto-renew → phải gia hạn thủ công mỗi năm
❌ Không hỗ trợ wildcard certs thông qua email
```

### Wildcard Certificate (Chứng Chỉ Wildcard)

```
*.example.com bao gồm:
✅ www.example.com
✅ api.example.com
✅ admin.example.com
✅ app.example.com

KHÔNG bao gồm:
❌ example.com (root domain) ← Cần certificate riêng
❌ sub.api.example.com (2 cấp subdomain)

Giải pháp: Dùng cả 2 trong cùng 1 ALB listener:
- example.com (SAN — Subject Alternative Name)
- *.example.com (wildcard)

ACM hỗ trợ SAN: 1 cert có thể chứa nhiều domains
```

### Certificate Renewal (Gia Hạn Chứng Chỉ)

```
Auto-renewal timeline:
- Certificate hết hạn sau: 13 tháng (393 ngày)
- ACM bắt đầu cố gắng renew: 60 ngày trước hết hạn
- Nếu DNS validation: Tự động ✅ (chỉ cần CNAME vẫn tồn tại)
- Nếu Email validation: Gửi email cảnh báo → thủ công ❌

Monitoring:
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:... \
  --query 'Certificate.{Status:Status,NotAfter:NotAfter}'

CloudWatch Alarm cho cert sắp hết hạn:
Metric: DaysToExpiry trong namespace AWS/CertificateManager
Alert khi < 30 ngày
```

---

## 🛡️ Security Policies (Chính Sách Bảo Mật)

### TLS Security Policies Của ALB/NLB

```
Security Policy = Bộ các cipher suites và TLS versions được cho phép

Chính sách hiện tại (2024-2025):

┌─────────────────────────────────────────────────────────────────┐
│ Policy Name                          │ TLS 1.3 │ TLS 1.2 │ 1.1 │
├─────────────────────────────────────────────────────────────────┤
│ ELBSecurityPolicy-TLS13-1-3-2021-06  │   ✅    │   ❌    │ ❌  │
│ (Chỉ TLS 1.3 — strictest)            │         │         │     │
├─────────────────────────────────────────────────────────────────┤
│ ELBSecurityPolicy-TLS13-1-2-2021-06  │   ✅    │   ✅    │ ❌  │
│ (TLS 1.2 & 1.3 — recommended)        │         │         │     │
├─────────────────────────────────────────────────────────────────┤
│ ELBSecurityPolicy-2016-08            │   ❌    │   ✅    │ ✅  │
│ (Legacy — tránh dùng)                │         │         │     │
└─────────────────────────────────────────────────────────────────┘

Khuyến nghị:
✅ Production: ELBSecurityPolicy-TLS13-1-2-2021-06
  (Hỗ trợ TLS 1.2 cho backward compatibility + TLS 1.3 cho modern clients)
✅ High security: ELBSecurityPolicy-TLS13-1-3-2021-06
  (Chỉ TLS 1.3 — loại bỏ clients cũ)
❌ Tránh: ELBSecurityPolicy-2016-08 (hỗ trợ TLS 1.0/1.1 — không an toàn)
```

### Cipher Suites (Bộ Mật Mã) Trong TLS 1.2

```
Cipher Suite format:
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256

Phân tích:
- TLS: Protocol
- ECDHE: Key exchange (Elliptic Curve Diffie-Hellman Ephemeral)
          → Forward Secrecy (Bí Mật Chuyển Tiếp)
- RSA: Authentication (xác thực certificate)
- AES_128_GCM: Symmetric encryption (mã hóa đối xứng)
- SHA256: Message authentication (xác thực thông điệp)

Ưu tiên cipher suites:
1. ECDHE với AES-GCM → Tốt nhất (forward secrecy + authenticated encryption)
2. ECDHE với AES-CBC → Tốt
3. DHE với AES → Chấp nhận được
❌ RC4, DES, 3DES, MD5 → Không an toàn, không dùng
```

---

## 🔒 HTTPS Best Practices — Thực Hành HTTPS Tốt Nhất

### HTTP → HTTPS Redirect (Chuyển Hướng)

```
Cấu hình ALB Listener:

HTTP Listener (:80):
  Default Action: Redirect to HTTPS
  - Protocol: HTTPS
  - Port: 443
  - Status Code: HTTP_301 (Moved Permanently — Đã Chuyển Vĩnh Viễn)

HTTPS Listener (:443):
  SSL Certificate: ACM cert
  Security Policy: ELBSecurityPolicy-TLS13-1-2-2021-06
  Default Action: Forward to Target Group

AWS CLI:
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions \
    Type=redirect,RedirectConfig='{
      Protocol=HTTPS,
      Port=443,
      StatusCode=HTTP_301,
      Host="#{host}",
      Path="/#{path}",
      Query="#{query}"
    }'
```

### HSTS — HTTP Strict Transport Security

```
HSTS = Header bảo buộc browser dùng HTTPS cho tất cả requests

Header:
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Ý nghĩa:
- max-age=31536000: Nhớ trong 1 năm (365 ngày × 86400 giây)
- includeSubDomains: Áp dụng cho tất cả subdomains
- preload: Cho phép thêm vào HSTS preload list của browsers

Cách thêm qua ALB:
ALB không tự thêm HSTS header
→ Cần thêm trong ứng dụng hoặc dùng Lambda@Edge

Nginx config:
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";

⚠️ Cẩn thận: Sau khi set HSTS, browser KHÔNG cho phép HTTP
   Nếu muốn roll back, phải chờ max-age expire
```

### Certificate Pinning vs SNI

```
SNI (Server Name Indication — Chỉ Thị Tên Máy Chủ):
- Extension của TLS, cho phép client nói cho server biết
  muốn kết nối với hostname nào
- Cho phép 1 ALB host nhiều SSL certs cho nhiều domains
- Hiện đại: Hầu hết clients hỗ trợ SNI
- Không hỗ trợ: Windows XP (lỗi thời), rất cũ

ALB với SNI:
- Gắn tối đa 25 certificates cho 1 HTTPS listener
- ALB tự chọn cert phù hợp dựa trên SNI từ client
- Ứng dụng: Multi-tenant SaaS với custom domains

Certificate Pinning (Ghim Chứng Chỉ):
- Mobile app cứng hóa certificate hash của server
- Chỉ tin tưởng cert cụ thể, không phải bất kỳ cert nào từ CA
- Rủi ro: Cert expire → App không hoạt động cho đến khi update app
- Khuyến nghị: Dùng Public Key Pinning thay vì Certificate Pinning
```

### mTLS — Mutual TLS (TLS Hai Chiều)

```
Standard TLS: Chỉ client verify server certificate
mTLS: Cả hai chiều verify certificate nhau

Client ─── TLS + Client Cert ──► ALB ─── verify client cert
                                    └── Forward to Target

Ứng dụng mTLS:
- B2B (Business-to-Business) APIs: Partner integration
- Internal microservices communication
- IoT devices authentication
- Zero-trust network architecture

ALB hỗ trợ mTLS (2023):
- Mutual Authentication mode
- Client certificate store trong S3 hoặc ACM Private CA
- On-load-balancer verification: ALB verify cert trước khi forward

NLB mTLS:
- TLS Pass-through → Target server tự implement mTLS
```

---

## ⚙️ Cấu Hình SSL/TLS Thực Tế

### Thêm Certificate Vào ALB Listener

```bash
# Import certificate vào ACM (nếu có sẵn cert từ CA khác)
aws acm import-certificate \
  --certificate fileb://certificate.pem \
  --private-key fileb://private-key.pem \
  --certificate-chain fileb://certificate-chain.pem \
  --region us-east-1

# Request ACM public certificate
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names "*.example.com" "api.example.com" \
  --validation-method DNS \
  --region us-east-1

# Thêm certificate vào ALB HTTPS Listener
aws elbv2 add-listener-certificates \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --certificates CertificateArn=arn:aws:acm:...

# Thay đổi security policy
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06
```

### Kiểm Tra SSL Configuration

```bash
# Kiểm tra TLS version và cipher được dùng
openssl s_client -connect alb-dns-name.us-east-1.elb.amazonaws.com:443 \
  -servername example.com \
  -tls1_2  # hoặc -tls1_3

# Kiểm tra certificate
openssl s_client -connect alb-dns-name.us-east-1.elb.amazonaws.com:443 \
  -servername example.com \
  2>/dev/null | openssl x509 -noout -dates -subject

# Test HTTPS redirect
curl -I http://example.com
# Expected: HTTP/1.1 301 Moved Permanently
# Location: https://example.com/

# Kiểm tra HSTS header
curl -I https://example.com | grep Strict-Transport

# Tool: SSL Labs test
# https://www.ssllabs.com/ssltest/ (web tool)
# Mục tiêu: Grade A hoặc A+
```

---

## 🎯 HTTPS Checklist Cho Production

```
Pre-Launch Checklist:
□ ACM certificate issued và validated
□ HTTP → HTTPS redirect (301) cấu hình trên ALB
□ Security Policy: TLS13-1-2-2021-06 hoặc mới hơn
□ HSTS header với max-age ≥ 1 năm
□ Wildcard cert hoặc SAN cert cho all subdomains
□ Certificate expiry monitoring (CloudWatch alarm)
□ DNS validation (không phải email) cho auto-renewal
□ SSL Labs test Grade A minimum

Security Headers (cần set trong ứng dụng hoặc CloudFront):
□ Strict-Transport-Security: max-age=31536000; includeSubDomains
□ Content-Security-Policy: default-src 'self'
□ X-Content-Type-Options: nosniff
□ X-Frame-Options: DENY (hoặc SAMEORIGIN)
□ X-XSS-Protection: 1; mode=block
□ Referrer-Policy: strict-origin-when-cross-origin
```

---

## 💡 Common Issues & Troubleshooting

```
Issue 1: SSL certificate mismatch
Triệu chứng: Browser cảnh báo "Your connection is not private"
Nguyên nhân:
  - Certificate domain ≠ URL domain
  - Wildcard cert nhưng dùng 2-level subdomain
  - Certificate expired
Giải pháp: Kiểm tra cert SAN, gia hạn cert

Issue 2: Mixed Content (Nội Dung Hỗn Hợp)
Triệu chứng: Browser console: "Mixed Content: The page was loaded over HTTPS,
but requested an insecure resource"
Nguyên nhân: HTML trên HTTPS nhưng link đến HTTP resources (images, scripts)
Giải pháp: Thay tất cả http:// links thành https:// trong code

Issue 3: 502 Bad Gateway sau khi bật HTTPS
Triệu chứng: ALB trả 502, health check trên HTTPS fails
Nguyên nhân: Target Group protocol = HTTPS nhưng target không chạy HTTPS
Giải pháp: Kiểm tra Target Group protocol phù hợp với target server

Issue 4: Certificate auto-renewal fails
Triệu chứng: Email cảnh báo từ ACM về cert sắp expire
Nguyên nhân: CNAME record validation đã bị xóa
Giải pháp: Kiểm tra và re-add CNAME record trong DNS
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: SSL termination tại ALB so với end-to-end encryption — trade-offs?**

> SSL termination tại ALB: ALB giải mã HTTPS, forward HTTP đến targets. Ưu điểm: certificate quản lý tập trung, targets không tốn CPU cho TLS, WAF có thể inspect traffic, ACM miễn phí và tự renew. Nhược điểm: traffic giữa ALB và targets là plaintext — cần bảo vệ bằng VPC security. End-to-end encryption (re-encrypt): ALB giải mã rồi mã hóa lại trước khi gửi đến target — double TLS handshake, tốn CPU hơn nhưng in-transit encryption toàn bộ đường đi. Compliance như PCI-DSS thường yêu cầu end-to-end encryption.

**Q: ACM certificate auto-renewal hoạt động như thế nào?**

> ACM bắt đầu cố gắng renew certificate 60 ngày trước khi hết hạn. Nếu dùng DNS validation (khuyến nghị): ACM check CNAME record trong DNS, nếu record còn đó → tự động renew, không cần can thiệp. Nếu dùng Email validation: ACM gửi email đến các admin addresses, phải manually click approve → không thể auto-renew thực sự. Đây là lý do DNS validation luôn được khuyến nghị cho production. Nên set CloudWatch alarm khi cert còn < 30 ngày để phát hiện kịp thời nếu renewal fails.

---

**Tiếp Theo:** [5-advanced-patterns.md](5-advanced-patterns.md) — Cross-zone LB, Sticky Sessions, Weighted Routing

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn Thành
