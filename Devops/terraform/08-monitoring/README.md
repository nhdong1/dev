# 08 — Monitoring & Observability — Giám Sát & Quan Sát Hạ Tầng Terraform

> Giám sát hạ tầng Terraform không chỉ là theo dõi tài nguyên — mà còn là phát hiện lệch cấu hình (Drift Detection), kiểm soát chi phí, truy vết thay đổi, và cảnh báo kịp thời khi hạ tầng rời khỏi trạng thái mong muốn.

---

## 📚 Mục Lục Phần Này

| File | Chủ Đề | Thời Gian |
|------|--------|-----------|
| [1-drift-detection.md](./1-drift-detection.md) | Drift Detection — Phát Hiện Lệch Cấu Hình | 45 phút |
| [2-infracost.md](./2-infracost.md) | Infracost — Ước Tính Chi Phí Trong CI/CD | 30 phút |
| [3-change-audit.md](./3-change-audit.md) | Change Audit — Kiểm Tra Ai Thay Đổi Gì, Khi Nào | 40 phút |
| [4-resource-tagging.md](./4-resource-tagging.md) | Resource Tagging — Chiến Lược Gán Nhãn Tài Nguyên | 35 phút |
| [5-alerting.md](./5-alerting.md) | Alerting — Cảnh Báo Khi Có Thay Đổi Ngoài Dự Kiến | 40 phút |

**Tổng thời gian ước tính:** 3-4 giờ

---

## 🎯 Tại Sao Phần Này Quan Trọng?

Sau khi `terraform apply` thành công, hạ tầng không tự nhiên "đứng yên". Nhiều vấn đề có thể xảy ra:

```
Vấn đề thực tế gặp phải:
┌─────────────────────────────────────────────────────────────────┐
│  Dev A thay đổi Security Group trực tiếp trên Console AWS      │
│  → Terraform state không biết → Drift xảy ra                   │
│                                                                 │
│  Dev B deploy thêm 10 EC2 instances không qua Terraform         │
│  → Chi phí tăng đột biến → Không ai biết nguyên nhân           │
│                                                                 │
│  Incident xảy ra → Ai apply lần cuối? Thay đổi gì?             │
│  → Không có audit trail → Mất nhiều giờ điều tra               │
└─────────────────────────────────────────────────────────────────┘
```

Monitoring hạ tầng Terraform giải quyết các vấn đề này bằng cách:
- **Drift Detection** — Phát hiện lệch cấu hình — phát hiện khi thực tế khác với code
- **Cost Estimation** — Ước tính chi phí — kiểm soát chi phí trước khi apply
- **Change Audit** — Kiểm tra thay đổi — truy vết ai làm gì và khi nào
- **Resource Tagging** — Gán nhãn tài nguyên — phân loại để giám sát và phân bổ chi phí
- **Alerting** — Cảnh báo — thông báo kịp thời khi có bất thường

---

## 🗺️ Bức Tranh Toàn Cảnh

```
                    VÒNG ĐỜI GIÁM SÁT HẠ TẦNG
                    
Code Repository
      │
      ▼
  terraform plan  ──► Infracost ──► Cost PR Comment
      │
      ▼
  terraform apply ──► CloudTrail / Audit Log ──► SIEM
      │
      ▼
  Hạ tầng thực tế
      │
      ▼ (định kỳ)
  terraform plan  ──► So sánh với state ──► Drift phát hiện
      │
      ├── Drift tìm thấy? ──► Alert (Slack / PagerDuty / Email)
      │
      └── Tag analysis ──► Cost allocation ──► Dashboard
```

---

## 📊 Bốn Trụ Cột Giám Sát Terraform

### 1. Drift Detection — Phát Hiện Lệch Cấu Hình

**Vấn đề giải quyết:** Thực tế hạ tầng khác với state Terraform

```
Terraform State:  sg-0123 → port 443 open
Thực tế AWS:      sg-0123 → port 443 + port 8080 open  ← DRIFT!
```

**Cách phát hiện:**
- Chạy `terraform plan` định kỳ (scheduled plan)
- Dùng `terraform refresh` để cập nhật state từ thực tế
- Công cụ chuyên biệt: Driftctl, Terraform Cloud Drift Detection

---

### 2. Cost Estimation — Ước Tính Chi Phí

**Vấn đề giải quyết:** Không biết một thay đổi tốn thêm bao nhiêu tiền

```
PR thay đổi:  instance_type = "t3.medium" → "m5.xlarge"
Infracost:    +$150/tháng (tăng 340%)  ← Hiện thị ngay trong PR comment
```

**Công cụ chính:** Infracost, Terraform Cloud Cost Estimation

---

### 3. Change Audit — Kiểm Tra Thay Đổi

**Vấn đề giải quyết:** Ai apply gì, khi nào, từ đâu?

```
Audit log entries:
2026-05-12 14:23 UTC | user: ci-bot | action: apply | resources: +3, ~1, -0
2026-05-12 09:15 UTC | user: john@company.com | action: plan | resources: +0, ~2, -1
```

**Nguồn dữ liệu:** AWS CloudTrail, Terraform Cloud audit logs, SIEM systems

---

### 4. Alerting & Tagging — Cảnh Báo & Gán Nhãn

**Vấn đề giải quyết:** Phát hiện muộn, không phân loại được tài nguyên

```
Tag strategy:
  Environment: prod/staging/dev
  Owner:       team-platform/team-backend
  CostCenter:  cc-engineering/cc-infra
  ManagedBy:   terraform
```

---

## 🔑 Khái Niệm Cốt Lõi

| Thuật Ngữ | Tiếng Anh | Giải Thích |
|-----------|-----------|------------|
| Drift | Configuration Drift — Lệch Cấu Hình | Sự khác biệt giữa state Terraform và thực tế hạ tầng |
| Remediation | Drift Remediation — Xử Lý Lệch | Đưa hạ tầng trở về trạng thái mong muốn |
| Cost allocation | — Phân Bổ Chi Phí | Xác định bộ phận/team nào phát sinh chi phí |
| Audit trail | — Nhật Ký Kiểm Tra | Lịch sử đầy đủ về ai làm gì, khi nào |
| MTTR | Mean Time To Recovery — Thời Gian Phục Hồi Trung Bình | Mục tiêu giảm nhờ audit trail tốt |
| Observability | — Khả Năng Quan Sát | Khả năng hiểu trạng thái hệ thống từ đầu ra của nó |

---

## ✅ Checklist Nhanh — Monitoring Terraform

### Cơ Bản (Team nhỏ)

- [ ] Bật CloudTrail trên tất cả regions của AWS
- [ ] Lưu Terraform plan output vào CI/CD logs
- [ ] Dùng Infracost trong CI/CD để hiển thị cost diff
- [ ] Áp dụng tagging policy cơ bản: `Environment`, `ManagedBy`
- [ ] Scheduled plan chạy hàng ngày để phát hiện drift

### Nâng Cao (Team lớn / Production)

- [ ] Tích hợp Infracost vào PR với budget threshold alerts
- [ ] Dùng Terraform Cloud với drift detection tích hợp
- [ ] Kết nối CloudTrail với SIEM (Splunk, Datadog, OpenSearch)
- [ ] Comprehensive tagging policy với enforcement (AWS Config / OPA)
- [ ] PagerDuty / Opsgenie alerts cho critical drift
- [ ] Báo cáo cost allocation theo team/environment hàng tuần

---

## 🚀 Thứ Tự Học Đề Xuất

```
1. Drift Detection (1-drift-detection.md)
   → Hiểu vấn đề drift là gì và cách phát hiện

2. Change Audit (3-change-audit.md)
   → Biết cách truy vết thay đổi

3. Resource Tagging (4-resource-tagging.md)
   → Nền tảng để cost allocation và governance

4. Infracost (2-infracost.md)
   → Tích hợp cost awareness vào workflow

5. Alerting (5-alerting.md)
   → Kết hợp tất cả thành hệ thống cảnh báo hoàn chỉnh
```

---

## 🔗 Liên Quan Đến Các Phần Khác

| Chủ Đề | Liên Quan Như Thế Nào |
|--------|----------------------|
| [02-state-management](../02-state-management/) | Drift detection dựa vào state để so sánh |
| [05-security](../05-security/) | Audit logging là một phần của security posture |
| [06-cicd](../06-cicd/) | Infracost và scheduled plans chạy trong CI/CD |
| [09-troubleshooting](../09-troubleshooting/) | Audit trail giúp chẩn đoán sự cố nhanh hơn |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
