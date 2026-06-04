# 09 — Monitoring & Maintenance: Giám Sát và Bảo Trì Jenkins

> Một hệ thống Jenkins hoạt động ổn định trong production (môi trường thực) đòi hỏi nhiều hơn là chỉ cài đặt và chạy — bạn cần **giám sát liên tục**, **quản lý tài nguyên**, **sao lưu định kỳ** và **kế hoạch nâng cấp** bài bản. Chủ đề này trang bị cho bạn đầy đủ kỹ năng vận hành Jenkins ở quy mô production.

---

## Mục Tiêu Học

Sau khi hoàn thành chủ đề này, bạn sẽ có thể:

- Thiết lập giám sát Jenkins với Prometheus (hệ thống thu thập metrics) và Grafana (nền tảng hiển thị dashboard)
- Đọc và cảnh báo dựa trên các metrics quan trọng: queue length (độ dài hàng đợi), executor usage (tỷ lệ sử dụng bộ thực thi), build duration (thời gian build)
- Cấu hình Build Discard Policy (chính sách xóa build cũ) để kiểm soát việc sử dụng ổ đĩa
- Lập lịch backup (sao lưu) và restore (phục hồi) `JENKINS_HOME` đáng tin cậy
- Lên kế hoạch và thực hiện nâng cấp Jenkins LTS (Long-Term Support — phiên bản hỗ trợ dài hạn) an toàn

---

## Danh Sách File

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-monitoring.md](1-monitoring.md) | Prometheus exporter, Grafana dashboard, metrics quan trọng, alerting | ⭐⭐ |
| [2-disk-management.md](2-disk-management.md) | Build Discard Policy, Workspace Cleanup Plugin, log rotation | ⭐⭐ |
| [3-backup-restore.md](3-backup-restore.md) | ThinBackup Plugin, backup JENKINS_HOME, chiến lược restore | ⭐⭐ |
| [4-upgrade-guide.md](4-upgrade-guide.md) | LTS vs Weekly release, kiểm tra plugin compatibility, rollback plan | ⭐⭐⭐ |

---

## Vấn Đề Monitoring & Maintenance Giải Quyết

### Không Có Monitoring (Giám Sát)

```
Triệu chứng thường gặp:
  • Jenkins đột ngột chậm mà không rõ nguyên nhân
  • Ổ đĩa đầy → toàn bộ build thất bại
  • Không biết build nào đang chiếm tài nguyên nhiều nhất
  • Phát hiện sự cố sau khi team đã chờ hàng giờ

→ Từ cảnh báo đến phát hiện: vài giờ đến vài ngày
→ MTTR (Mean Time To Recover — thời gian phục hồi trung bình): cao
```

### Có Monitoring và Maintenance Tốt

```
Kết quả đạt được:
  • Alerting (cảnh báo) kịp thời trước khi sự cố xảy ra
  • Ổ đĩa được kiểm soát tự động — không bao giờ đầy bất ngờ
  • Backup tự động hàng ngày — restore trong vòng 15 phút
  • Nâng cấp Jenkins có kế hoạch rõ ràng, không downtime bất ngờ

→ Từ cảnh báo đến phát hiện: vài phút
→ MTTR: giảm 80%
```

---

## Kiến Trúc Tổng Quan: Monitoring Stack (Bộ Công Cụ Giám Sát)

```
┌──────────────────────────────────────────────────────────────┐
│                      Jenkins Master                           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Prometheus Plugin (plugin thu thập metrics)          │   │
│  │  Endpoint: http://jenkins:8080/prometheus/            │   │
│  │                                                       │   │
│  │  Metrics exposed (metrics được xuất):                 │   │
│  │  • jenkins_builds_duration_milliseconds               │   │
│  │  • jenkins_queue_size_value                           │   │
│  │  • jenkins_executor_count_value                       │   │
│  │  • jenkins_node_online_value                          │   │
│  └────────────────────────┬─────────────────────────────┘   │
│                            │ HTTP scrape (15s)               │
└────────────────────────────┼─────────────────────────────────┘
                             │
              ┌──────────────▼──────────────┐
              │   Prometheus Server          │
              │   (lưu trữ time-series data) │
              │   Retention: 15 ngày         │
              └──────────────┬──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │   Grafana Dashboard           │
              │   (hiển thị và alerting)      │
              │                              │
              │  ┌────────────────────────┐  │
              │  │  Jenkins Overview Panel │  │
              │  │  • Build Success Rate   │  │
              │  │  • Queue Length         │  │
              │  │  • Executor Utilization │  │
              │  │  • Build Duration Trend │  │
              │  └────────────────────────┘  │
              └──────────────────────────────┘
                             │
              ┌──────────────▼──────────────┐
              │   AlertManager               │
              │   → Slack, PagerDuty, Email  │
              └──────────────────────────────┘
```

---

## Ba Trụ Cột Của Jenkins Maintenance

### 1. Disk Management (Quản Lý Ổ Đĩa)

```
JENKINS_HOME/
├── jobs/                    ← Chiếm nhiều dung lượng nhất
│   └── my-pipeline/
│       └── builds/
│           ├── 1/           ← Build cũ — có thể xóa
│           ├── 2/           ← Build cũ — có thể xóa
│           └── 999/         ← Build mới — giữ lại
├── workspace/               ← Cần dọn định kỳ
└── plugins/                 ← Kích thước ổn định
```

### 2. Backup Strategy (Chiến Lược Sao Lưu)

```
Chiến lược 3-2-1:
  3 bản sao → 2 loại storage khác nhau → 1 bản offsite (ngoài site)

  Backup hàng ngày  → Local NFS / S3
  Backup hàng tuần  → Offsite object storage
  Test restore      → Hàng tháng, trên môi trường staging
```

### 3. Upgrade Planning (Lập Kế Hoạch Nâng Cấp)

```
Quy trình nâng cấp an toàn:
  1. Đọc LTS changelog và plugin compatibility notes
  2. Test trên môi trường staging (không phải production)
  3. Backup JENKINS_HOME trước khi nâng cấp
  4. Maintenance window (cửa sổ bảo trì) có lịch rõ ràng
  5. Rollback plan (kế hoạch quay lại) nếu có sự cố
```

---

## Thứ Tự Học Đề Xuất

```
Bắt đầu
   │
   ▼
1-monitoring.md       ← Thiết lập visibility (khả năng quan sát) trước
   │
   ▼
2-disk-management.md  ← Kiểm soát tài nguyên
   │
   ▼
3-backup-restore.md   ← Bảo vệ dữ liệu
   │
   ▼
4-upgrade-guide.md    ← Duy trì phiên bản an toàn
```

---

## Điều Kiện Tiên Quyết

Trước khi học chủ đề này, bạn nên nắm vững:

- [x] Kiến trúc Jenkins Master/Agent (`01-fundamentals/`)
- [x] Cơ bản về Prometheus và Grafana (khái niệm metrics, labels, PromQL)
- [x] Linux system administration: disk usage, cron, file permissions
- [x] Docker cơ bản (nếu chạy Jenkins trong container)

---

## Câu Hỏi Phỏng Vấn Liên Quan

1. Bạn giám sát sức khỏe Jenkins như thế nào trong production?
2. Metrics nào quan trọng nhất khi theo dõi Jenkins? Tại sao?
3. Khi ổ đĩa Jenkins đầy, bạn xử lý thế nào? Phòng ngừa ra sao?
4. Chiến lược backup Jenkins của bạn là gì? Đã test restore chưa?
5. Khi nâng cấp Jenkins từ LTS 2.x lên 2.y, bạn làm những bước nào?
6. Sự khác biệt giữa Jenkins LTS và Weekly release là gì?

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
