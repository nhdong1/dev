# 10. Monitoring & Debugging — Giám Sát và Gỡ Lỗi GitHub Actions

> Giám sát — Quan Sát (Observability), gỡ lỗi (debug), thông báo (notifications), và audit logs (nhật ký kiểm toán) cho GitHub Actions workflows.

---

## 📚 Mục Lục Chủ Đề

| File | Nội Dung | Độ Ưu Tiên |
|---|---|---|
| [1-debug-logging.md](./1-debug-logging.md) | Debug logs, ACTIONS_STEP_DEBUG, tqdm logs nâng cao | ⭐⭐⭐ |
| [2-workflow-notifications.md](./2-workflow-notifications.md) | Slack, email, GitHub Issues notifications | ⭐⭐⭐ |
| [3-metrics-observability.md](./3-metrics-observability.md) | Metrics, Datadog, Grafana, custom dashboards | ⭐⭐ |
| [4-audit-logs.md](./4-audit-logs.md) | Audit logs enterprise, compliance, SIEM tích hợp | ⭐⭐ |

---

## 🎯 Tổng Quan Chủ Đề

**Monitoring & Debugging** (Giám Sát & Gỡ Lỗi) là kỹ năng quan trọng để:

- **Phát hiện lỗi nhanh** — biết workflow bị lỗi ở đâu, tại sao
- **Giảm MTTR** — Mean Time To Recovery (Thời Gian Trung Bình Phục Hồi) từ giờ xuống phút
- **Theo dõi sức khỏe pipeline** — tỉ lệ thành công, thời gian chạy, xu hướng
- **Tuân thủ audit** — lưu lịch sử thao tác cho compliance (tuân thủ quy định)

---

## 🧭 Tại Sao Quan Trọng

```
Không có monitoring:              Có monitoring tốt:
  ❌ Lỗi phát hiện sau vài giờ     ✅ Alert ngay khi workflow fail
  ❌ Debug mò mẫm trong logs        ✅ Structured logs dễ tìm root cause
  ❌ Không biết performance trend   ✅ Dashboard rõ ràng thời gian/tỉ lệ lỗi
  ❌ Không có audit trail           ✅ Đầy đủ lịch sử ai làm gì, khi nào
```

---

## 🔍 Ba Trụ Cột Observability (Ba Yếu Tố Quan Sát)

### 1. Logs (Nhật Ký)
Ghi lại sự kiện chi tiết trong từng step — xem `1-debug-logging.md`

### 2. Metrics (Chỉ Số)
Đo lường hiệu suất và xu hướng theo thời gian — xem `3-metrics-observability.md`

### 3. Traces (Vết Thực Thi)
Theo dõi luồng thực thi end-to-end — tích hợp với Datadog, Honeycomb

---

## 🚨 Vòng Đời Debug Workflow

```
Workflow bị lỗi
      │
      ▼
1. Xem Summary Tab       ── Tổng quan lỗi ở job nào
      │
      ▼
2. Mở Job Logs           ── Tìm bước fail (đánh dấu đỏ)
      │
      ▼
3. Bật Debug Logging     ── ACTIONS_STEP_DEBUG=true (nếu logs chưa đủ)
      │
      ▼
4. Re-run với debug      ── "Re-run jobs" → "Enable debug logging"
      │
      ▼
5. Kiểm tra Context      ── In ${{ toJSON(github) }} để xem data
      │
      ▼
6. Thêm diagnostic step  ── env, whoami, ls, curl để kiểm tra môi trường
      │
      ▼
7. Fix & Verify          ── Commit fix, chạy lại, xác nhận green
```

---

## 📊 Checklist Monitoring Cơ Bản

### Ngay Lập Tức (Phải Có)

- [ ] Thông báo khi workflow fail trên nhánh `main`/`master`
- [ ] Lưu logs đủ lâu để debug (tối thiểu 30 ngày)
- [ ] Hiểu cách bật `ACTIONS_STEP_DEBUG` khi cần

### Trung Hạn (Nên Có)

- [ ] Dashboard thời gian chạy workflow theo tuần
- [ ] Alert khi tỉ lệ fail tăng bất thường
- [ ] Retention policy (chính sách lưu trữ) cho artifacts và logs

### Dài Hạn (Enterprise)

- [ ] Tập trung logs vào SIEM (Security Information and Event Management — Quản Lý Thông Tin Bảo Mật)
- [ ] Audit trail đầy đủ cho compliance
- [ ] SLA dashboard cho CI/CD pipeline

---

## 🔗 Liên Kết Nội Bộ

- [07-caching-performance/4-performance-optimization.md](../07-caching-performance/4-performance-optimization.md) — Tối ưu thời gian chạy
- [08-security/6-security-hardening.md](../08-security/6-security-hardening.md) — Security monitoring
- [09-self-hosted-runners/4-maintenance-monitoring.md](../09-self-hosted-runners/4-maintenance-monitoring.md) — Monitor self-hosted runners

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
