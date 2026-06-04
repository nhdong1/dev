# 03 — Quản Lý Plugin Jenkins

> Plugin (tiện ích mở rộng) là nền tảng mở rộng sức mạnh của Jenkins. Hơn 1.800 plugin có sẵn trên [plugins.jenkins.io](https://plugins.jenkins.io) — từ tích hợp Git, Docker đến giao diện Blue Ocean và bảo mật nâng cao.

## Mục Lục

1. [Tại Sao Plugin Quan Trọng?](#tại-sao-plugin-quan-trọng)
2. [Plugin Manager — Quản Lý Plugin](#plugin-manager--quản-lý-plugin)
3. [Vòng Đời Plugin](#vòng-đời-plugin)
4. [Phân Loại Plugin](#phân-loại-plugin)
5. [Nội Dung Chi Tiết](#nội-dung-chi-tiết)

---

## Tại Sao Plugin Quan Trọng?

Jenkins lõi (Jenkins core) rất nhỏ gọn — phần lớn tính năng đến từ plugin. Hiểu plugin là hiểu Jenkins:

| Nhu Cầu                            | Plugin Giải Quyết                          |
| ---------------------------------- | ------------------------------------------ |
| Kết nối với Git repository         | Git Plugin, GitHub Plugin                  |
| Chạy build trong Docker container  | Docker Pipeline, Docker Plugin             |
| Giao diện pipeline trực quan       | Blue Ocean Plugin, Pipeline Stage View     |
| Gửi thông báo Slack                | Slack Notification Plugin                  |
| Phân tích code chất lượng          | SonarQube Scanner Plugin                   |
| Chạy agent trên Kubernetes         | Kubernetes Plugin                          |
| Quản lý credentials an toàn        | Credentials Binding Plugin                 |

---

## Plugin Manager — Quản Lý Plugin

Plugin Manager (trình quản lý plugin) là giao diện trung tâm để quản lý toàn bộ vòng đời của plugin.

### Truy Cập Plugin Manager

```
Manage Jenkins → Plugins → Available plugins / Installed / Updates
```

### Cài Đặt Plugin

**Qua giao diện web (UI):**
```
Manage Jenkins → Plugins → Available plugins → [tìm tên] → Install
```

**Qua Jenkins CLI:**
```bash
java -jar jenkins-cli.jar -s http://localhost:8080/ install-plugin git pipeline-utility-steps
```

**Qua Configuration as Code (JCasC — Cấu Hình Như Code):**
```yaml
# plugins.yaml
plugins:
  required:
    git: "latest"
    pipeline-utility-steps: "2.15.4"
    kubernetes: "3923.v01db_22cce1cf"
```

**Qua Dockerfile khi build Jenkins custom:**
```dockerfile
FROM jenkins/jenkins:lts-jdk17
RUN jenkins-plugin-cli --plugins \
    git:latest \
    pipeline-utility-steps:latest \
    blueocean:latest
```

---

## Vòng Đời Plugin

```
Tìm kiếm → Cài đặt → Restart Jenkins → Sử dụng → Cập nhật → Gỡ bỏ (nếu cần)
```

### Cập Nhật Plugin

```
Manage Jenkins → Plugins → Updates → [chọn plugin] → Download now and install after restart
```

**Lưu ý quan trọng:**
- Luôn cập nhật trên môi trường staging trước khi production
- Đọc changelog (nhật ký thay đổi) để phát hiện breaking changes
- Backup `JENKINS_HOME` trước khi cập nhật hàng loạt

### Gỡ Bỏ Plugin

```
Manage Jenkins → Plugins → Installed → [chọn plugin] → Uninstall
```

**Cảnh báo:** Gỡ bỏ plugin có thể làm hỏng pipeline đang sử dụng plugin đó. Kiểm tra dependency trước khi gỡ.

### Vô Hiệu Hóa Plugin (Disable)

Thay vì gỡ bỏ hoàn toàn, có thể tạm thời vô hiệu hóa:
```
Manage Jenkins → Plugins → Installed → [chọn plugin] → Disable
```

---

## Phân Loại Plugin

### Theo Chức Năng

| Loại                              | Mô Tả                                       | Ví Dụ                          |
| --------------------------------- | ------------------------------------------- | ------------------------------ |
| **SCM Plugins**                   | Kết nối kho mã nguồn                        | Git, SVN, Mercurial            |
| **Pipeline Plugins**              | Mở rộng cú pháp và chức năng Pipeline       | Pipeline, Blue Ocean           |
| **Build Wrapper Plugins**         | Bao bọc quanh quá trình build               | AnsiColor, Build Timeout       |
| **Notification Plugins**          | Gửi thông báo kết quả build                 | Slack, Mailer, Teams           |
| **Authentication Plugins**        | Tích hợp hệ thống xác thực                  | LDAP, OAuth2, SAML             |
| **Cloud/Container Plugins**       | Chạy agent trên cloud hoặc container        | Kubernetes, Docker, EC2        |
| **Code Quality Plugins**          | Phân tích và báo cáo chất lượng code        | SonarQube, Checkstyle, JaCoCo  |
| **Artifact Plugins**              | Quản lý artifact sau build                  | S3, Nexus, Artifactory         |

### Theo Mức Độ Thiết Yếu

```
Plugin Lõi (Core)       → Cài ngay khi setup Jenkins
Plugin Quan Trọng       → Cần cho workflow CI/CD phổ biến
Plugin Tích Hợp         → Tùy theo stack công nghệ dự án
Plugin Tiện Ích         → Cải thiện trải nghiệm, không bắt buộc
```

---

## Phụ Thuộc Giữa Các Plugin

Jenkins dùng cơ chế **classloader** — mỗi plugin có classloader riêng, dùng chung API của Jenkins core.

**Dependency tree (cây phụ thuộc)** hay gây vấn đề khi:
- Plugin A yêu cầu Plugin B phiên bản >= 2.0
- Plugin C yêu cầu Plugin B phiên bản <= 1.9
- Hai yêu cầu mâu thuẫn → "dependency hell" (địa ngục phụ thuộc)

**Giải pháp:**
```
1. Cập nhật tất cả plugin trước khi thêm plugin mới
2. Dùng Plugin Dependency Graph để xem cây phụ thuộc
3. Kiểm tra trong môi trường staging trước
```

---

## Nội Dung Chi Tiết

| File                                                     | Nội Dung                                                  |
| -------------------------------------------------------- | --------------------------------------------------------- |
| [1-essential-plugins.md](1-essential-plugins.md)         | Plugin thiết yếu: Git, Pipeline, Blue Ocean, Credentials  |
| [2-pipeline-plugins.md](2-pipeline-plugins.md)           | Pipeline Utility Steps, Stage View, Build Monitor         |
| [3-integration-plugins.md](3-integration-plugins.md)     | Docker, Kubernetes, Slack, SonarQube, Email               |

---

## Mẹo Quản Lý Plugin

1. **Không cài quá nhiều plugin** — mỗi plugin tăng thêm bộ nhớ và thời gian khởi động
2. **Ưu tiên plugin có cộng đồng lớn** — nhiều contributor, issue được xử lý nhanh
3. **Kiểm tra "Last Released"** — plugin không cập nhật > 2 năm có thể không tương thích Jenkins mới
4. **Dùng Plugin BOM** — Bill of Materials, tập hợp phiên bản plugin được kiểm thử tương thích
5. **Ghi lại danh sách plugin** — vào `JENKINS_HOME/plugins/`, dùng cho disaster recovery

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
