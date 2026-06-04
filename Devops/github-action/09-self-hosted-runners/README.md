# Self-hosted Runners — Máy Chạy Tự Quản Lý

> Self-hosted runners (Máy Chạy Tự Quản Lý) — các máy chủ do bạn kiểm soát, đăng ký với GitHub Actions để thay thế hoặc bổ sung cho GitHub-hosted runners. Phù hợp cho workloads cần phần cứng đặc biệt, mạng nội bộ, hoặc kiểm soát chi phí quy mô lớn.

---

## 📚 Mục Lục

1. [Khi Nào Cần Self-hosted Runners](#khi-nào-cần-self-hosted-runners)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [So Sánh GitHub-hosted vs Self-hosted](#so-sánh-github-hosted-vs-self-hosted)
4. [Nội Dung Chủ Đề](#nội-dung-chủ-đề)
5. [Checklist Vận Hành](#checklist-vận-hành)
6. [Điều Hướng Nhanh](#điều-hướng-nhanh)

---

## 🎯 Khi Nào Cần Self-hosted Runners

### Trường Hợp Phù Hợp Dùng Self-hosted

| Tình Huống | Lý Do |
|---|---|
| Truy cập tài nguyên nội bộ (database, registry, API nội bộ) | GitHub-hosted không vào được mạng private |
| Phần cứng đặc biệt (GPU, ARM, high memory) | GitHub-hosted chỉ cung cấp specs cố định |
| Compliance / Data residency (Tuân Thủ / Lưu Trú Dữ Liệu) | Dữ liệu không được rời khỏi datacenter |
| Build time dài, chi phí cao ở quy mô lớn | GitHub-hosted tính tiền theo phút |
| Windows Server / macOS cụ thể | GitHub-hosted không cung cấp mọi phiên bản OS |
| Workloads cần caching nặng (Docker layers, Maven repo) | Self-hosted lưu cache persistent trên disk |

### Trường Hợp KHÔNG Nên Dùng Self-hosted

- Repository public (rủi ro code injection cực cao)
- Workloads nhỏ, team nhỏ (overhead vận hành không xứng)
- Chưa có người phụ trách hạ tầng (cần monitoring, patching liên tục)

---

## 🏗️ Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────┐
│                      GitHub.com                             │
│                                                             │
│   ┌──────────────┐    ┌─────────────────────────────────┐   │
│   │  Repository  │───▶│     Actions Service (API)       │   │
│   └──────────────┘    └────────────┬────────────────────┘   │
│                                    │ HTTPS Long Polling      │
└────────────────────────────────────│────────────────────────┘
                                     │
              ┌──────────────────────▼──────────────────────┐
              │          Your Infrastructure                  │
              │                                              │
              │  ┌──────────────┐  ┌──────────────────────┐  │
              │  │  Runner VM 1  │  │  Runner VM 2          │  │
              │  │  (linux/x64)  │  │  (linux/arm64)        │  │
              │  │  Labels: prod │  │  Labels: staging      │  │
              │  └──────────────┘  └──────────────────────┘  │
              │                                              │
              │  ┌────────────────────────────────────────┐  │
              │  │  ARC (Actions Runner Controller)        │  │
              │  │  Kubernetes Cluster — Auto-scaling      │  │
              │  │  ┌──────┐ ┌──────┐ ┌──────┐            │  │
              │  │  │Pod 1 │ │Pod 2 │ │Pod 3 │ ...         │  │
              │  │  └──────┘ └──────┘ └──────┘            │  │
              │  └────────────────────────────────────────┘  │
              └──────────────────────────────────────────────┘
```

**Giao tiếp một chiều:** Runner chủ động polling GitHub API qua HTTPS outbound. GitHub không cần kết nối vào infrastructure của bạn (không cần mở inbound firewall).

---

## ⚖️ So Sánh GitHub-hosted vs Self-hosted

| Tiêu Chí | GitHub-hosted | Self-hosted |
|---|---|---|
| **Quản lý hạ tầng** | GitHub lo | Bạn lo |
| **Chi phí** | Tính theo phút (có free tier) | Chi phí hạ tầng của bạn |
| **Bảo mật** | Môi trường sạch mỗi job | Phụ thuộc vào cấu hình của bạn |
| **Truy cập mạng nội bộ** | Không | Có |
| **Phần cứng tùy chỉnh** | Không | Có |
| **Persistent storage** | Không (xóa sau mỗi job) | Có thể |
| **Thời gian khởi động** | ~10–30 giây | Phụ thuộc cấu hình |
| **OS / phiên bản** | Ubuntu, Windows, macOS (cố định) | Bất kỳ |
| **Độ phức tạp vận hành** | Thấp | Cao |
| **Phù hợp với** | Hầu hết workloads | Nhu cầu đặc biệt |

---

## 📁 Nội Dung Chủ Đề

### [1. Cài Đặt & Đăng Ký Runner](./1-setup-registration.md)

- Tải và cài đặt runner binary
- Đăng ký runner với repository / organization / enterprise
- Labels (Nhãn) và Runner Groups (Nhóm Runner)
- Chạy runner như systemd service
- Workflow `runs-on` targeting

### [2. ARC — Auto-scaling trên Kubernetes](./2-arc-autoscaling.md)

- ARC — Actions Runner Controller (Bộ Điều Khiển Runner Hành Động) là gì
- Cài đặt ARC bằng Helm
- ScaleSet (Bộ Mở Rộng) và scale-to-zero
- Ephemeral runners (Runners Tạm Thời) trên Kubernetes
- Monitoring ARC metrics

### [3. Bảo Mật & Cô Lập](./3-security-isolation.md)

- Rủi ro bảo mật của self-hosted runners
- Network isolation (Cô Lập Mạng) — VPC, firewall rules
- Ephemeral runners — xóa sau mỗi job
- Hardening (Tăng Cường Bảo Mật) runner environment
- Least-privilege cho runner service account

### [4. Bảo Trì & Giám Sát](./4-maintenance-monitoring.md)

- Cập nhật runner binary
- Giám sát trạng thái runner
- Troubleshooting (Xử Lý Sự Cố) runner offline
- Capacity planning (Lên Kế Hoạch Tài Nguyên)
- Log tập trung cho runner fleet

---

## ✅ Checklist Vận Hành

### Trước Khi Đưa Runner Vào Production

- [ ] Runner chạy trên ephemeral environment (ephemeral flag hoặc container)
- [ ] Network isolation — chỉ mở port cần thiết (HTTPS outbound)
- [ ] Runner service chạy dưới non-root user
- [ ] Không có secrets hard-coded trong runner environment
- [ ] Monitoring và alerting cho runner offline
- [ ] Quy trình update runner binary định kỳ
- [ ] Chỉ dùng cho repository private (không bao giờ dùng cho public repo)

### Kiểm Tra Định Kỳ (Hàng Tuần)

- [ ] Tất cả runners đang ở trạng thái `idle` hoặc `active` (không có `offline`)
- [ ] Phiên bản runner binary còn được GitHub support
- [ ] Disk usage dưới 80%
- [ ] Không có job bị kẹt quá lâu

---

## 🔗 Điều Hướng Nhanh

| Nhu Cầu | File |
|---|---|
| Bắt đầu cài đặt runner | [1-setup-registration.md](./1-setup-registration.md) |
| Auto-scaling trên K8s | [2-arc-autoscaling.md](./2-arc-autoscaling.md) |
| Tăng cường bảo mật | [3-security-isolation.md](./3-security-isolation.md) |
| Bảo trì và giám sát | [4-maintenance-monitoring.md](./4-maintenance-monitoring.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
