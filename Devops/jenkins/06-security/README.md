# 06 — Security: Bảo Mật Jenkins

> Tổng quan về bảo mật Jenkins — từ xác thực người dùng, phân quyền truy cập, quản lý thông tin xác thực đến các kỹ thuật hardening (gia cố bảo mật) cho môi trường production.

## Mục Lục Chủ Đề

| File                          | Nội Dung                                                      | Độ Ưu Tiên |
| ----------------------------- | ------------------------------------------------------------- | ---------- |
| `1-authentication.md`         | Jenkins DB, LDAP, Active Directory, OAuth2/SSO                | ⭐⭐⭐      |
| `2-authorization.md`          | Matrix-based Security, Role Strategy Plugin, Project-based    | ⭐⭐⭐      |
| `3-credentials.md`            | Secret text, Username/Password, SSH Key, Certificate, Vault   | ⭐⭐⭐      |
| `4-security-best-practices.md`| Hardening, CSP header, Script Security, Audit Log             | ⭐⭐⭐      |

---

## Tại Sao Bảo Mật Jenkins Quan Trọng?

Jenkins là trung tâm của toàn bộ vòng lặp CI/CD (Continuous Integration/Continuous Delivery — Tích Hợp Liên Tục/Phân Phối Liên Tục). Một Jenkins server bị xâm phạm đồng nghĩa với:

- **Rò rỉ secret** (bí mật): API token, SSH key, database password, cloud credentials
- **Code injection** (chèn mã độc): kẻ tấn công có thể sửa pipeline để đưa mã độc vào production
- **Supply chain attack** (tấn công chuỗi cung ứng): build artifacts bị nhiễm độc trước khi triển khai
- **Privilege escalation** (leo thang đặc quyền): từ user thông thường lên admin Jenkins hoặc thậm chí hệ thống OS

Jenkins chạy với quyền hệ thống cao — vi phạm an ninh ở đây thường là **full system compromise** (xâm phạm toàn bộ hệ thống).

---

## Mô Hình Bảo Mật Jenkins

```
┌─────────────────────────────────────────────────────────────┐
│                     JENKINS SECURITY MODEL                  │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │ AUTHENTICATION│    │AUTHORIZATION │    │  CREDENTIALS  │  │
│  │  (Xác Thực)  │───▶│  (Phân Quyền)│    │ (Thông Tin   │  │
│  │              │    │              │    │  Xác Thực)   │  │
│  │ - Jenkins DB │    │ - Matrix     │    │ - Secret text │  │
│  │ - LDAP       │    │ - Role       │    │ - SSH Key     │  │
│  │ - OAuth2/SSO │    │   Strategy   │    │ - Vault       │  │
│  │ - SAML       │    │ - Project-   │    │               │  │
│  └──────────────┘    │   based      │    └───────────────┘  │
│                      └──────────────┘                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              SECURITY HARDENING                     │   │
│  │  CSP Headers | Script Security | Audit Log | HTTPS  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Ba Lớp Bảo Mật Cốt Lõi

### Lớp 1 — Authentication (Xác Thực): "Bạn là ai?"

Xác minh danh tính người dùng trước khi cho phép truy cập Jenkins. Hỗ trợ nhiều backend:
- **Jenkins Internal Database** (cơ sở dữ liệu nội bộ): đơn giản, phù hợp môi trường nhỏ
- **LDAP** (Lightweight Directory Access Protocol — Giao Thức Truy Cập Thư Mục Nhẹ): tích hợp với Active Directory hoặc OpenLDAP của tổ chức
- **OAuth2/OIDC** (OpenID Connect): SSO qua GitHub, Google, Okta, Keycloak
- **SAML 2.0** (Security Assertion Markup Language): federated identity cho doanh nghiệp lớn

### Lớp 2 — Authorization (Phân Quyền): "Bạn được làm gì?"

Kiểm soát hành động nào người dùng được phép thực hiện sau khi đã xác thực:
- **Matrix-based Security** (bảo mật dựa trên ma trận): gán quyền chi tiết theo user/group
- **Role Strategy Plugin**: RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò) với Global Role và Project Role
- **Project-based Matrix Authorization** (phân quyền theo từng project): mỗi job có ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập) riêng
- **Folder-based Authorization**: phân quyền theo cấu trúc thư mục job

### Lớp 3 — Credentials (Thông Tin Xác Thực): "Bí mật được bảo vệ thế nào?"

Lưu trữ và sử dụng secret an toàn trong pipeline mà không để lộ giá trị thuần:
- **Secret text** (văn bản bí mật): API token, password đơn giản
- **Username/Password** (tên người dùng/mật khẩu): credentials 2 trường
- **SSH Username with private key** (tên người dùng SSH với khóa riêng tư): SSH deploy key, git access
- **Certificate** (chứng chỉ): PKCS#12 keystore cho mutual TLS
- **HashiCorp Vault** (kho bí mật HashiCorp): quản lý secret tập trung cho enterprise

---

## Checklist Bảo Mật Nhanh

### Cài Đặt Mới (New Installation)

- [ ] Bật **Enable Security** (bật bảo mật) ngay sau khi cài xong
- [ ] Đặt mật khẩu admin mạnh, không dùng mật khẩu mặc định
- [ ] Tắt **Agent-to-Master Security** bypass nếu không cần
- [ ] Cấu hình HTTPS — không chạy Jenkins trên HTTP thuần trong production
- [ ] Hạn chế Sign-up (đăng ký tài khoản) — tắt "Allow users to sign up"

### Vận Hành Hàng Ngày

- [ ] Dùng Credentials (thông tin xác thực) thay vì hardcode (ghi cứng) secret trong Jenkinsfile
- [ ] Cấp quyền tối thiểu theo **Principle of Least Privilege** (nguyên tắc quyền tối thiểu)
- [ ] Rotate (luân phiên thay) credentials định kỳ — đặc biệt SSH key và API token
- [ ] Theo dõi **Audit Log** (nhật ký kiểm tra) để phát hiện hành vi bất thường
- [ ] Cập nhật Jenkins và Plugin thường xuyên — vá lỗ hổng bảo mật

### Pipeline Security (Bảo Mật Pipeline)

- [ ] Dùng `withCredentials` block để truy cập secret — không dùng `env.MY_SECRET`
- [ ] Kiểm tra Groovy script qua **Script Security Plugin** (sandbox mode)
- [ ] Không cho user không tin cậy dùng **Scripted Pipeline** trực tiếp
- [ ] Bật **Protect Jenkins from build agents** trong Global Security

---

## Luồng Học Đề Xuất

```
1-authentication.md     → Hiểu cách user đăng nhập và tích hợp LDAP/OAuth2
         ↓
2-authorization.md      → Cấu hình phân quyền Role Strategy cho team
         ↓
3-credentials.md        → Quản lý secret đúng cách trong pipeline
         ↓
4-security-best-practices.md → Hardening tổng thể cho production
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp (Chủ Đề Bảo Mật)

1. Làm thế nào để ngăn secret bị lộ trong Jenkins build log?
2. Giải thích sự khác biệt giữa Matrix-based Security và Role Strategy Plugin.
3. Khi nào nên dùng HashiCorp Vault thay vì Jenkins Credentials Store?
4. Script Security Plugin sandbox hoạt động như thế nào?
5. Những bước hardening cần thiết khi triển khai Jenkins lên production?

---

**Xem tiếp:** [1-authentication.md](1-authentication.md) — Cơ chế xác thực người dùng
