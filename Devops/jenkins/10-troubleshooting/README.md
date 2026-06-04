# Troubleshooting Jenkins — Xử Lý Sự Cố

> Hướng dẫn chẩn đoán và khắc phục các sự cố phổ biến trong Jenkins: từ build thất bại, agent mất kết nối, đến vấn đề hiệu suất và xung đột plugin.

## Mục Lục

1. [Tổng Quan Phương Pháp Xử Lý](#tổng-quan-phương-pháp-xử-lý)
2. [Các Chủ Đề Trong Mục Này](#các-chủ-đề-trong-mục-này)
3. [Sơ Đồ Chẩn Đoán Nhanh](#sơ-đồ-chẩn-đoán-nhanh)
4. [Công Cụ Cần Thiết](#công-cụ-cần-thiết)

---

## Tổng Quan Phương Pháp Xử Lý

Troubleshooting (xử lý sự cố) Jenkins hiệu quả đòi hỏi một phương pháp có hệ thống:

```
Triệu Chứng → Thu Thập Log → Xác Định Nguyên Nhân Gốc → Áp Dụng Giải Pháp → Kiểm Tra → Phòng Ngừa
(Symptom)     (Collect Logs) (Root Cause Analysis)       (Apply Fix)          (Verify) (Prevention)
```

### Nguyên Tắc Vàng Khi Debug

1. **Đọc log trước, đoán sau** — Jenkins log thường chỉ rõ nguyên nhân
2. **Tái hiện vấn đề** — chạy lại build với thông tin debug bật
3. **Thay đổi từng biến** — không sửa nhiều thứ cùng lúc
4. **Lưu lại giải pháp** — viết runbook (sổ tay vận hành) để dùng lại

---

## Các Chủ Đề Trong Mục Này

| File | Chủ Đề | Độ Phổ Biến |
|------|--------|-------------|
| [1-build-failures.md](1-build-failures.md) | Build Failures — Lỗi build | ⭐⭐⭐ Rất thường gặp |
| [2-agent-issues.md](2-agent-issues.md) | Agent Issues — Sự cố agent kết nối | ⭐⭐⭐ Rất thường gặp |
| [3-performance-issues.md](3-performance-issues.md) | Performance — Hiệu suất và bộ nhớ | ⭐⭐ Thường gặp |
| [4-plugin-conflicts.md](4-plugin-conflicts.md) | Plugin Conflicts — Xung đột plugin | ⭐⭐ Thường gặp |
| [5-production-checklist.md](5-production-checklist.md) | Production Checklist — Checklist production | ⭐⭐⭐ Bắt buộc |

---

## Sơ Đồ Chẩn Đoán Nhanh

```
Build thất bại?
├── Exit code khác 0         → 1-build-failures.md#exit-codes
├── "Agent offline"          → 2-agent-issues.md#agent-offline
├── "Out of Memory"          → 3-performance-issues.md#jvm-heap
├── Plugin exception         → 4-plugin-conflicts.md
└── Lỗi permissions          → 1-build-failures.md#permissions

Jenkins chạy chậm?
├── UI phản hồi chậm         → 3-performance-issues.md#slow-ui
├── Build chờ lâu            → 3-performance-issues.md#build-queue
├── GC overhead              → 3-performance-issues.md#gc-tuning
└── Disk đầy                 → 09-monitoring-maintenance/disk-management.md

Agent không kết nối?
├── JNLP port bị block       → 2-agent-issues.md#jnlp-firewall
├── SSH key sai              → 2-agent-issues.md#ssh-key
├── Agent crash              → 2-agent-issues.md#agent-crash
└── Docker socket lỗi        → 2-agent-issues.md#docker-agent
```

---

## Công Cụ Cần Thiết

### Truy Cập Log Jenkins

```bash
# Log hệ thống Jenkins (System Log — nhật ký hệ thống)
# Giao diện: Manage Jenkins → System Log → All Jenkins Logs

# Log file trực tiếp trên server
tail -f /var/log/jenkins/jenkins.log

# Log trong Docker container
docker logs jenkins-controller --follow --tail=200

# Log trong Kubernetes pod
kubectl logs -n jenkins deployment/jenkins -f --tail=200
```

### Script Chẩn Đoán Nhanh

```groovy
// Chạy trong Script Console (Manage Jenkins → Script Console)
// Xem tất cả running threads (luồng đang chạy)
Thread.getAllStackTraces().keySet().each { thread ->
    println "Thread: ${thread.name} | State: ${thread.state}"
}

// Xem tất cả items trong Build Queue (hàng đợi build)
Jenkins.instance.queue.items.each { item ->
    println "Queued: ${item.task.name} | Why: ${item.why}"
}

// Xem tất cả Executor (bộ thực thi) đang bận
Jenkins.instance.computers.each { computer ->
    computer.executors.each { executor ->
        if (!executor.isIdle()) {
            println "Busy: ${computer.name} | Build: ${executor.currentExecutable}"
        }
    }
}
```

---

**Xem Tiếp:** [1-build-failures.md](1-build-failures.md) — phân tích lỗi build từ log
