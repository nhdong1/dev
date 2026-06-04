# 2 — Client VPN — VPN Truy Cập Từ Xa

> AWS Client VPN — dịch vụ VPN được quản lý hoàn toàn, cho phép người dùng từ xa (remote users) kết nối bảo mật vào AWS VPC hoặc mạng on-premises thông qua phần mềm OpenVPN client chuẩn.

---

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc & Thành Phần](#kiến-trúc--thành-phần)
3. [Phương Thức Xác Thực](#phương-thức-xác-thực)
4. [Cấu Hình Chi Tiết](#cấu-hình-chi-tiết)
5. [Split Tunneling — Phân Luồng](#split-tunneling--phân-luồng)
6. [Authorization Rules — Quy Tắc Ủy Quyền](#authorization-rules--quy-tắc-ủy-quyền)
7. [Monitoring & Logging](#monitoring--logging)
8. [Chi Phí](#chi-phí)
9. [So Sánh & Use Cases](#so-sánh--use-cases)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Client VPN Là Gì?

**AWS Client VPN** là dịch vụ VPN fully-managed (được quản lý hoàn toàn) sử dụng giao thức **OpenVPN** (TLS — Transport Layer Security), cho phép:

- **Người dùng từ xa** (remote workers, developers) kết nối vào AWS VPC
- **Thiết bị đầu cuối** (laptop, desktop, điện thoại) truy cập resources trong private subnets
- **Centralized access control** — kiểm soát truy cập tập trung theo user/group

```
Remote User Laptop ──OpenVPN──► Client VPN Endpoint ──► Private VPC Resources
(Người dùng từ xa)  (TLS mã hóa)  (Điểm cuối VPN)        (Tài nguyên riêng)
```

### Khác Biệt vs Site-to-Site VPN

| Tiêu Chí | Client VPN | Site-to-Site VPN |
|---------|-----------|-----------------|
| **Người kết nối** | Từng người dùng | Toàn bộ mạng office |
| **Phía client** | Phần mềm VPN client | Router/Firewall phần cứng |
| **Giao thức** | OpenVPN (TLS) | IPsec |
| **Use case** | Remote work, dev access | Datacenter ↔ AWS |
| **Xác thực** | AD, Certificate, SAML | PSK, Certificate |
| **Scalability** | Thousands of users | Fixed bandwidth |

---

## Kiến Trúc & Thành Phần

### Sơ Đồ Kiến Trúc

```
┌──────────────────────────────────────────────────────────────────┐
│                         AWS Cloud                                │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                         VPC                              │  │
│   │                                                          │  │
│   │  ┌─────────────┐     ┌──────────────────────────────┐   │  │
│   │  │   Private   │     │  Client VPN Endpoint         │   │  │
│   │  │   Subnet    │◄───►│  (Điểm Cuối VPN)             │   │  │
│   │  │  10.0.1.0/24│     │  DNS: *.cvpn-endpoint.aws    │   │  │
│   │  └─────────────┘     │  Port: UDP 443 (mặc định)    │   │  │
│   │                      └──────────────┬───────────────┘   │  │
│   │  ┌─────────────┐                    │                    │  │
│   │  │ Auth Server │◄───────────────────┘                    │  │
│   │  │ (AD/LDAP)   │   Xác thực người dùng                  │  │
│   │  └─────────────┘                                         │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                    │                             │
│                                    │ TLS (OpenVPN)               │
│                                    │                             │
└────────────────────────────────────┼─────────────────────────────┘
                                     │
                            Internet (encrypted)
                                     │
            ┌────────────────────────┼────────────────────────┐
            │         Remote Users   │                        │
            │                        │                        │
     ┌──────▼──────┐        ┌────────▼──────┐       ┌────────▼──────┐
     │   Laptop    │        │  Desktop      │       │  Mobile       │
     │  (AWS VPN   │        │  (OpenVPN     │       │  (iOS/Android │
     │   Client)   │        │   client)     │       │   OpenVPN)    │
     └─────────────┘        └───────────────┘       └───────────────┘
```

### Các Thành Phần

**1. Client VPN Endpoint — Điểm Cuối VPN**
- AWS-managed endpoint (người dùng không quản lý infrastructure)
- Có DNS name: `<random>.cvpn-endpoint.<region>.amazonaws.com`
- Hỗ trợ UDP 443 (mặc định, nhanh hơn) hoặc TCP 443
- Có thể associate với nhiều subnets trong nhiều AZs

**2. Target Network — Mạng Đích**
- Subnet trong VPC được associate với endpoint
- Traffic từ client sẽ đi qua subnet này
- Nên associate ít nhất 2 subnets ở 2 AZs khác nhau (high availability)

**3. Client CIDR Block — Dải IP Client**
- Range IP cấp cho VPN clients (e.g., `172.16.0.0/22`)
- Không được overlap với VPC CIDR hoặc on-premises network
- Phải đủ lớn cho số lượng concurrent connections

**4. Authorization Rules — Quy Tắc Ủy Quyền**
- Kiểm soát user/group nào được truy cập network nào
- Granular control theo Active Directory group

---

## Phương Thức Xác Thực

### 1. Mutual TLS Certificate Authentication (Chứng Chỉ Hai Chiều)

```
Cả server và client đều có certificate:
- Server certificate: Tạo và upload lên ACM (AWS Certificate Manager)
- Client certificate: Tạo CA riêng, cấp cert cho từng user

Ưu điểm: Không cần external auth server
Nhược điểm: Khó revoke cert, quản lý cert phức tạp khi scale lớn
```

```bash
# Tạo Certificate Authority (CA — Cơ Quan Chứng Nhận)
# Sử dụng easy-rsa
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3

./easyrsa init-pki
./easyrsa build-ca nopass

# Tạo server certificate
./easyrsa build-server-full server nopass

# Tạo client certificate
./easyrsa build-client-full client1 nopass

# Import vào ACM
aws acm import-certificate \
  --certificate fileb://pki/issued/server.crt \
  --private-key fileb://pki/private/server.key \
  --certificate-chain fileb://pki/ca.crt
```

### 2. Active Directory Authentication (Xác Thực Active Directory)

```
Người dùng đăng nhập bằng AD credentials (username/password):
- Tích hợp với AWS Directory Service (SimpleAD hoặc Managed AD)
- Hỗ trợ MFA (Multi-Factor Authentication — Xác Thực Đa Yếu Tố) qua TOTP
- Authorization theo AD Security Groups

Ưu điểm: Đơn giản cho user, centralized user management
Nhược điểm: Cần AD infrastructure
```

**Cấu hình trong Console:**
```
Authentication type: Active Directory
Directory ID: d-xxxxxxxxxx (AWS Managed Microsoft AD)
```

### 3. SAML 2.0 Federated Authentication (Xác Thực Liên Kết)

```
SSO (Single Sign-On — Đăng Nhập Một Lần) với Identity Provider (IdP):
- Okta, Azure AD, Google Workspace, OneLogin
- Người dùng authenticate với IdP, IdP cấp assertion cho AWS

Ưu điểm: Tích hợp với hệ thống SSO doanh nghiệp
Nhược điểm: Cần cấu hình phức tạp hơn
```

---

## Cấu Hình Chi Tiết

### Tạo Client VPN Endpoint

```bash
# 1. Tạo Client VPN Endpoint
aws ec2 create-client-vpn-endpoint \
  --client-cidr-block "172.16.0.0/22" \
  --server-certificate-arn "arn:aws:acm:region:account:certificate/xxx" \
  --authentication-options '[{
    "Type": "directory-service-authentication",
    "ActiveDirectory": {
      "DirectoryId": "d-xxxxxxxxxx"
    }
  }]' \
  --connection-log-options '{
    "Enabled": true,
    "CloudwatchLogGroup": "/aws/vpn/client",
    "CloudwatchLogStream": "connections"
  }' \
  --dns-servers "10.0.0.2" \
  --split-tunnel false \
  --vpn-port 443 \
  --transport-protocol udp \
  --description "Main Client VPN"

# 2. Associate với Target Subnet
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id cvpn-endpoint-xxxxxxxxx \
  --subnet-id subnet-xxxxxxxxx

# 3. Thêm Authorization Rule (cho phép tất cả user vào VPC)
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id cvpn-endpoint-xxxxxxxxx \
  --target-network-cidr "10.0.0.0/16" \
  --authorize-all-groups

# 4. Thêm Route (để client reach được VPC)
aws ec2 create-client-vpn-route \
  --client-vpn-endpoint-id cvpn-endpoint-xxxxxxxxx \
  --destination-cidr-block "10.0.0.0/16" \
  --target-vpc-subnet-id subnet-xxxxxxxxx
```

### Download & Cấu Hình Client

```bash
# Download configuration file
aws ec2 export-client-vpn-client-configuration \
  --client-vpn-endpoint-id cvpn-endpoint-xxxxxxxxx \
  --output text > client-config.ovpn

# Thêm client certificate vào config file
echo "<cert>" >> client-config.ovpn
cat pki/issued/client1.crt >> client-config.ovpn
echo "</cert>" >> client-config.ovpn
echo "<key>" >> client-config.ovpn
cat pki/private/client1.key >> client-config.ovpn
echo "</key>" >> client-config.ovpn
```

---

## Split Tunneling — Phân Luồng

### Full Tunnel (Mặc Định — Toàn Bộ Lưu Lượng)

```
Tất cả traffic từ client đều đi qua VPN:
  
Client → VPN → AWS → Internet (cho cả internet traffic)
  
Ưu điểm:
  ✅ Centralized security inspection
  ✅ Internet traffic qua proxy/firewall của công ty
  
Nhược điểm:
  ❌ Tăng tải cho VPN endpoint
  ❌ Tốn băng thông, tốc độ internet chậm hơn
  ❌ Chi phí data transfer cao hơn
```

### Split Tunnel (Khuyến Nghị)

```
Chỉ traffic đến AWS CIDR đi qua VPN:

Client → VPN → AWS          (cho VPC traffic)
Client → Internet trực tiếp (cho internet thông thường)

Ưu điểm:
  ✅ Giảm tải VPN endpoint
  ✅ Tốc độ internet bình thường
  ✅ Giảm chi phí data transfer

Nhược điểm:
  ❌ Internet traffic không qua corporate security
```

```bash
# Bật Split Tunneling khi tạo endpoint
aws ec2 create-client-vpn-endpoint \
  --split-tunnel true \   # Bật split tunnel
  ...

# Hoặc modify endpoint đang chạy
aws ec2 modify-client-vpn-endpoint \
  --client-vpn-endpoint-id cvpn-endpoint-xxxxxxxxx \
  --split-tunnel true
```

---

## Authorization Rules — Quy Tắc Ủy Quyền

### Granular Access Control (Kiểm Soát Truy Cập Chi Tiết)

```bash
# Cho phép group "developers" truy cập Dev subnet
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id cvpn-endpoint-xxx \
  --target-network-cidr "10.0.1.0/24" \
  --access-group-id "sg-developers-group-id" \
  --description "Developers access to Dev subnet"

# Cho phép group "admins" truy cập tất cả subnets
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id cvpn-endpoint-xxx \
  --target-network-cidr "10.0.0.0/8" \
  --access-group-id "sg-admins-group-id" \
  --description "Admins full access"

# Cho phép tất cả users (không nên dùng cho production)
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id cvpn-endpoint-xxx \
  --target-network-cidr "10.0.0.0/8" \
  --authorize-all-groups
```

### Connection Log — Nhật Ký Kết Nối

```
CloudWatch Logs ghi lại:
- Thời gian kết nối/ngắt kết nối
- Username (nếu dùng AD/SAML auth)
- Client IP phân bổ
- Common Name của certificate (nếu mutual TLS)
- Bytes in/out
```

---

## Monitoring & Logging

### CloudWatch Metrics Quan Trọng

| Metric | Ý Nghĩa | Ngưỡng Cảnh Báo |
|--------|---------|-----------------|
| `ActiveConnectionsCount` | Số kết nối đang active | Alert nếu > 90% capacity |
| `NotAuthenticated` | Số lần auth thất bại | Alert nếu tăng đột biến |
| `AuthorizationDenied` | Bị từ chối quyền | Alert bất kỳ lúc nào |
| `Ingress/EgressBytes` | Lưu lượng vào/ra | Theo dõi bất thường |

### Connection Logging

```bash
# Xem active connections
aws ec2 describe-client-vpn-connections \
  --client-vpn-endpoint-id cvpn-endpoint-xxxxxxxxx \
  --filters Name=status,Values=active

# Terminate một connection cụ thể
aws ec2 terminate-client-vpn-connections \
  --client-vpn-endpoint-id cvpn-endpoint-xxxxxxxxx \
  --connection-id cvpn-connection-xxxxxxxxx
```

---

## Chi Phí

### Bảng Giá (US East — N. Virginia)

| Thành Phần | Đơn Giá | Ghi Chú |
|-----------|---------|---------|
| **Endpoint Association** | $0.10/giờ/association | Mỗi subnet associate |
| **Active Connection** | $0.05/giờ/connection | Mỗi client đang kết nối |

### Ước Tính Chi Phí

```
Ví dụ: 1 endpoint, 2 subnet associations, 50 concurrent users, 8h/ngày làm việc

Endpoint associations:  $0.10 × 2 × 24h × 30 ngày  = $144/tháng
Active connections:     $0.05 × 50 users × 8h × 22 ngày = $440/tháng
──────────────────────────────────────────────────────────────────────
Tổng:                                                  ~$584/tháng

→ Chi phí đáng kể với đội lớn, cân nhắc split tunnel để giảm duration
```

### Tối Ưu Chi Phí

```
1. Bật Split Tunnel: Giảm traffic qua endpoint → giảm data cost
2. Idle timeout: Tự ngắt kết nối sau X phút không hoạt động
3. Associate đúng số AZs cần thiết (mỗi association = $0.10/h)
4. Dùng SAML + IdP: Centralized revocation, không phải quản lý cert
```

---

## So Sánh & Use Cases

### Client VPN vs Site-to-Site VPN vs Direct Connect

| Tiêu Chí | Client VPN | Site-to-Site VPN | Direct Connect |
|---------|-----------|-----------------|----------------|
| **Đối tượng** | Từng người dùng | Cả mạng nội bộ | Cả mạng nội bộ |
| **Giao thức** | OpenVPN (TLS) | IPsec | BGP (layer 3) |
| **Xác thực** | AD, Cert, SAML | PSK/Cert | N/A |
| **Tính năng** | Per-user access control | Site-level | Site-level |
| **Chi phí** | Theo connections | Theo connections | Theo port + data |
| **Setup** | Giờ | Giờ | Tuần–tháng |

### Kịch Bản Thực Tế

**Kịch bản 1: Startup với team remote**
```
Vấn đề: 20 developers cần access RDS database trong private subnet
Giải pháp: Client VPN với Mutual TLS
- Mỗi developer có certificate riêng
- Split tunnel bật (internet traffic không qua VPN)
- Authorization rule chỉ cho phép port 5432 (PostgreSQL)
```

**Kịch bản 2: Enterprise với SSO**
```
Vấn đề: 500 employees remote, đã có Okta SSO
Giải pháp: Client VPN với SAML 2.0 + Okta
- Người dùng login bằng Okta credentials + MFA
- Groups trong Okta map sang authorization rules
- Revoking access: Disable user trong Okta là đủ
```

**Kịch bản 3: Hybrid với on-premises**
```
Client VPN Endpoint → VPC → VPN Site-to-Site → On-Premises
(Remote user)        (AWS)  (IPsec tunnel)      (Datacenter)

→ Remote users có thể reach cả AWS và on-premises resources
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Phân biệt Client VPN và Site-to-Site VPN

**Trả lời mẫu:**

> Site-to-Site VPN kết nối toàn bộ mạng on-premises với AWS VPC qua tunnel IPsec — phù hợp khi muốn toàn bộ office/datacenter có thể reach AWS. Client VPN ngược lại, kết nối từng thiết bị cá nhân (laptop, điện thoại) của remote users vào VPC qua OpenVPN — phù hợp cho remote work. Client VPN cũng hỗ trợ xác thực per-user (AD, SAML, Certificate), cho phép kiểm soát truy cập chi tiết theo từng người hoặc nhóm.

### Q2: Split Tunneling là gì và khi nào nên bật?

**Trả lời mẫu:**

> Split Tunneling là tính năng chỉ route traffic đến AWS CIDR qua VPN tunnel, còn traffic internet thông thường vẫn đi thẳng ra internet không qua VPN. Nên bật khi: muốn giảm tải VPN endpoint, giảm latency cho internet access của users, và giảm chi phí data transfer. Không nên bật khi: yêu cầu compliance bắt buộc tất cả traffic phải qua corporate security inspection (proxy, firewall), hoặc khi cần phòng chống data exfiltration.

### Q3: Cách revoke quyền truy cập của một user cụ thể?

**Với Mutual TLS:**
```
1. Revoke certificate của user: easyrsa revoke <client-name>
2. Update CRL (Certificate Revocation List) trên endpoint
Khó khăn: CRL phải được update và upload lại

Cách tốt hơn: Terminate active connection ngay lập tức
aws ec2 terminate-client-vpn-connections \
  --client-vpn-endpoint-id <endpoint-id> \
  --username <username>
```

**Với Active Directory:**
```
Disable user trong Active Directory
→ User không thể authenticate mới
→ Active connection vẫn còn cho đến khi session timeout
→ Terminate connection thủ công nếu cần ngay lập tức
```

**Với SAML (Okta/Azure AD):**
```
Deactivate user trong IdP (Identity Provider)
→ Không thể authenticate mới
→ Terminate active connection nếu cần
```

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Trước | [1-site-to-site-vpn.md](./1-site-to-site-vpn.md) |
| → Tiếp theo | [3-direct-connect.md](./3-direct-connect.md) |
| ↑ Chỉ mục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
