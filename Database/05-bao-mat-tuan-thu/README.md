# Bảo Mật & Tuân Thủ CSDL

Bảo vệ dữ liệu với bảo mật nhiều tầng và đáp ứng yêu cầu quy định.

## Các Tài Liệu

| Tài liệu | Mô tả |
|----------|-------|
| [Kiểm Soát Truy Cập](kiem-soat-truy-cap.md) | Quyền tối thiểu, RBAC, Row-Level Security, phân tách schema |
| [Mã Hóa](ma-hoa.md) | TLS, mã hóa at-rest, column encryption, quản lý key |
| [Ghi Nhật Ký Kiểm Tra](ghi-nhat-ky-kiem-tra.md) | pgaudit, audit triggers, SIEM integration, lưu giữ log |
| [Bảo Mật Mạng](bao-mat-mang.md) | Network segmentation, pg_hba.conf, VPN, firewall, bastion host |
| [Tuân Thủ GDPR](tuan-thu-gdpr.md) | PII management, quyền data subject, retention policy, breach notification |
| [Tuân Thủ PCI-DSS](tuan-thu-pci-dss.md) | Tokenization, cardholder data protection, 12 requirements |
| [Quản Lý Bí Mật](quan-ly-bi-mat.md) | HashiCorp Vault, dynamic secrets, credential rotation |

---

## Mô Hình Bảo Mật Nhiều Tầng

```
Tầng 5: KIỂM TOÁN & GIÁM SÁT
  - DDL và DML được ghi log (pgaudit)
  - Audit triggers cho bảng nhạy cảm
  - Tích hợp SIEM, alert bất thường

Tầng 4: MÃ HÓA
  - Lưu trữ at-rest (AES-256, disk encryption)
  - Truyền tải in-transit (TLS 1.2+)
  - Column encryption cho PII/CHD
  - Quản lý key (KMS/HSM/Vault)

Tầng 3: PHÂN QUYỀN
  - Vai trò quyền tối thiểu (RBAC)
  - Phân tách schema
  - Row-Level Security (RLS)

Tầng 2: XÁC THỰC
  - Không mật khẩu dùng chung
  - Tích hợp IAM/LDAP
  - MFA cho truy cập DBA
  - Service account riêng theo ứng dụng

Tầng 1: MẠNG
  - Database trong private subnet
  - Security groups whitelist-only
  - VPN + Bastion host cho DBA
  - TLS bắt buộc (hostssl)
```

---

## Ma Trận Rủi Ro Bảo Mật

```
RỦI RO THẤP (Fix ngay):
✓ TLS chưa bật → Bật ssl trong postgresql.conf + hostssl trong pg_hba.conf
✓ Mật khẩu yếu → Enforce strong password policy + Vault rotation
✓ Superuser cho ứng dụng → Tạo dedicated role với quyền tối thiểu

RỦI RO TRUNG BÌNH (Lên kế hoạch trong sprint):
⚠ Không có audit logging → Cài pgaudit, bật log_connections
⚠ Credentials trong config → Di chuyển sang Vault/Secrets Manager
⚠ Thiếu network segmentation → Cấu hình Security Groups, pg_hba.conf

RỦI RO CAO (Fix trong 24h):
❌ Database có public IP → Di chuyển vào private subnet ngay
❌ Không mã hóa truyền tải → Database password đang bị sniff
❌ PAN/CVV lưu plain text → Vi phạm PCI-DSS, phải fix ngay
❌ Credentials bị lộ → Rotate ngay, audit access log
```

---

## Checklist Tổng Hợp

**Mạng:**
- [ ] CSDL trong private subnet (không có public IP)
- [ ] Security groups chỉ whitelist IP đã biết
- [ ] VPN hoặc bastion host cho DBA access
- [ ] TLS 1.2+ bắt buộc (hostssl + hostnossl reject)
- [ ] Connection pooler làm proxy (PgBouncer)

**Xác Thực & Phân Quyền:**
- [ ] Không shared accounts (mỗi app có account riêng)
- [ ] Superuser bị vô hiệu hóa cho kết nối ứng dụng
- [ ] Quyền tối thiểu theo role (RBAC)
- [ ] MFA cho DBA console access
- [ ] Access review hàng quý

**Mã Hóa:**
- [ ] Disk/volume encryption được bật
- [ ] TLS cho tất cả kết nối (verify-full)
- [ ] PII và CHD được mã hóa tại cột
- [ ] Mật khẩu hash với bcrypt/scrypt
- [ ] Key rotation schedule tự động

**Audit & Logging:**
- [ ] pgaudit bật cho DDL và WRITE
- [ ] log_connections = on
- [ ] Audit log append-only (không thể xóa)
- [ ] Logs ship ra SIEM ngay lập tức
- [ ] Retention policy tuân thủ quy định (12 tháng+)

**Secret Management:**
- [ ] KHÔNG có credentials trong source code
- [ ] Vault hoặc cloud secret manager
- [ ] Credential rotation tự động (90 ngày)
- [ ] Dynamic secrets cho database credentials
- [ ] Emergency rotation procedure được test

**Tuân Thủ:**
- [ ] Data inventory được tài liệu hóa
- [ ] GDPR: Data retention policy tự động hóa
- [ ] GDPR: Quy trình xóa PII khi có yêu cầu
- [ ] PCI-DSS: Tokenization (không lưu số thẻ đầy đủ)
- [ ] PCI-DSS: Không lưu CVV/SAD sau authorization
- [ ] Penetration test định kỳ (hàng năm)

---

## Câu Hỏi Phỏng Vấn Thường Gặp

**1. Thiết kế CSDL an toàn cho ứng dụng fintech xử lý dữ liệu thanh toán?**
- Tokenization thay cho lưu số thẻ (Stripe/Braintree)
- Tuân thủ PCI-DSS (không lưu SAD, mã hóa CHD)
- TLS cho tất cả kết nối
- Vault cho secret management
- Audit logs bất biến
- Penetration test hàng quý

**2. Xử lý yêu cầu xóa người dùng (GDPR Right to Erasure)?**
- Tìm tất cả PII trong database (data map)
- Anonymize hoặc xóa từng bảng có dữ liệu user
- Tính đến backup (retention policy phải cover)
- Log xác nhận xóa (chứng minh tuân thủ)
- Thông báo cho user trong 30 ngày

**3. Phương pháp quản lý bí mật của bạn?**
- HashiCorp Vault với dynamic database credentials
- Credentials tự expire sau TTL ngắn
- Rotation tự động 90 ngày cho static secrets
- Audit trail mọi truy cập secret
- AppRole/Kubernetes auth (không hardcode token)

**4. Phát hiện người dùng truy cập dữ liệu nhạy cảm trái phép?**
- pgaudit với object-level auditing trên bảng nhạy cảm
- Alert khi query patterns bất thường (nhiều SELECT lúc đêm)
- Row-Level Security để giới hạn truy cập
- SIEM integration với rules phát hiện bất thường

---

> **Điểm Mấu Chốt:** Bảo mật không phải là tính năng thêm vào cuối — nó được xây dựng từ đầu ở mọi tầng. Dễ dàng để làm đúng (tooling tốt, tự động hóa) và khó để làm sai (không có thông tin đăng nhập trong code, không có truy cập trực tiếp từ internet) là mục tiêu của kiến trúc bảo mật tốt.
