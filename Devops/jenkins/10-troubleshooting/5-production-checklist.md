# Production Checklist — Danh Sách Kiểm Tra Trước Khi Đưa Jenkins Lên Production

> Checklist toàn diện để đảm bảo Jenkins sẵn sàng cho môi trường production (môi trường thực tế): bảo mật, hiệu suất, tính sẵn sàng cao, giám sát, và khả năng khôi phục sau sự cố.

## Mục Lục

1. [Tổng Quan Checklist](#tổng-quan-checklist)
2. [Bảo Mật — Security Checklist](#bảo-mật--security-checklist)
3. [Hiệu Suất — Performance Checklist](#hiệu-suất--performance-checklist)
4. [Tính Sẵn Sàng — Availability Checklist](#tính-sẵn-sàng--availability-checklist)
5. [Giám Sát — Monitoring Checklist](#giám-sát--monitoring-checklist)
6. [Backup và Restore — Checklist Khôi Phục](#backup-và-restore--checklist-khôi-phục)
7. [Network và Firewall Checklist](#network-và-firewall-checklist)
8. [Checklist Đội Nhóm và Quy Trình](#checklist-đội-nhóm-và-quy-trình)
9. [Go-Live Checklist — Ngày Ra Mắt](#go-live-checklist--ngày-ra-mắt)
10. [Runbook — Sổ Tay Vận Hành Khẩn Cấp](#runbook--sổ-tay-vận-hành-khẩn-cấp)

---

## Tổng Quan Checklist

### Cách Sử Dụng Checklist Này

```
□ = Chưa kiểm tra
✅ = Đã hoàn thành
⚠️ = Cần xem xét / exception có documentation
❌ = Chưa đạt, cần fix trước go-live
```

### Phân Loại Theo Độ Ưu Tiên

| Loại | Mô Tả | Phải Đạt Trước Go-Live? |
|------|--------|------------------------|
| **P0 — Critical** | Lỗi bảo mật hoặc mất dữ liệu | Bắt buộc 100% |
| **P1 — High** | Ảnh hưởng nghiêm trọng đến hoạt động | Bắt buộc |
| **P2 — Medium** | Ảnh hưởng một phần | Nên đạt |
| **P3 — Low** | Cải thiện long-term | Có thể sau go-live |

---

## Bảo Mật — Security Checklist

### P0 — Bắt Buộc Tuyệt Đối

```
□ [P0] Authentication (xác thực) được bật
    → Không để Jenkins ở chế độ "anyone can do anything"
    → Kiểm tra: Manage Jenkins → Configure Global Security → Security Realm

□ [P0] Authorization (phân quyền) được cấu hình
    → Tối thiểu: Matrix-based Security hoặc Role Strategy Plugin
    → Anonymous user (người dùng ẩn danh) KHÔNG có quyền truy cập

□ [P0] Jenkins không expose ra internet công khai mà không có authentication
    → Nếu cần truy cập từ ngoài: dùng VPN hoặc reverse proxy với authentication

□ [P0] Script Security (bảo mật script) được bật
    → Manage Jenkins → Configure Global Security → Groovy Sandbox = Enabled
    → Script Approval (phê duyệt script) có process rõ ràng

□ [P0] Credentials (thông tin xác thực) KHÔNG hardcode trong Jenkinsfile
    → Tất cả secrets lưu trong Jenkins Credentials Store hoặc HashiCorp Vault
    → Không có password/token dạng plain text trong repo
```

### P1 — Bắt Buộc

```
□ [P1] HTTPS được bật cho Jenkins URL
    → Dùng reverse proxy (nginx/Apache) với TLS certificate
    → Redirect HTTP → HTTPS tự động

□ [P1] Jenkins CLI qua HTTP bị vô hiệu hóa
    → Manage Jenkins → Configure Global Security
    → CLI over Remoting: Disabled

□ [P1] Cross-Site Request Forgery (CSRF — Tấn Công Giả Mạo Request) Protection bật
    → Manage Jenkins → Configure Global Security → CSRF Protection = Enabled

□ [P1] Agent-to-Controller Security (bảo mật agent-controller) bật
    → Manage Jenkins → Configure Global Security
    → Agent → Controller Security: Enabled

□ [P1] Audit Log (nhật ký kiểm tra) được cấu hình
    → Cài Audit Trail Plugin
    → Log tất cả login, logout, config changes, job runs

□ [P1] SSH keys và API tokens được rotate (luân phiên) định kỳ
    → Có policy (ví dụ: mỗi 90 ngày)
    → Credentials không expired đang tồn tại trong Jenkins

□ [P1] Không chạy Jenkins với quyền root
    → Kiểm tra: whoami (trong Script Console hoặc build log)
    → Jenkins nên chạy với user riêng biệt: jenkins

□ [P1] Plugin được cập nhật — không có known security vulnerabilities
    → Manage Jenkins → Plugin Manager → Updates tab
    → Chú ý badge "security fix" màu đỏ
```

### P2 — Nên Đạt

```
□ [P2] CSP — Content Security Policy (Chính Sách Bảo Mật Nội Dung) được cấu hình
    → Hạn chế inline scripts trong Jenkins UI
    → Thêm vào JAVA_OPTS: -Dhudson.model.DirectoryBrowserSupport.CSP=""

□ [P2] Jenkins version là LTS (Long-Term Support — Hỗ Trợ Dài Hạn)
    → Không dùng weekly release cho production

□ [P2] Principle of Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu)
    → Mỗi service account chỉ có quyền tối thiểu cần thiết
    → Pipeline credentials chỉ có quyền vừa đủ

□ [P2] Network Segmentation (phân đoạn mạng)
    → Jenkins Controller và Agent nằm trong private network
    → Chỉ expose port cần thiết ra ngoài
```

---

## Hiệu Suất — Performance Checklist

### P1 — Bắt Buộc

```
□ [P1] JVM Heap được cấu hình đúng
    → -Xmx tối thiểu 2 GB cho production
    → -XX:+UseG1GC được bật
    → Không dùng default heap (512 MB) cho production

□ [P1] Build Discard Policy được cấu hình cho TẤT CẢ jobs
    → Không có job nào giữ build vô thời hạn
    → Mặc định: 30 ngày hoặc 20 builds (tùy cái nào đến trước)

□ [P1] Controller không chạy builds (executor = 0 hoặc rất ít)
    → Manage Nodes → Built-In Node → Executors = 0
    → Tất cả builds chạy trên Agent

□ [P1] Disk có đủ dung lượng và monitoring
    → JENKINS_HOME partition ≥ 50% free
    → Alert khi disk < 20% free
    → Workspace Cleanup Plugin được cài và cấu hình
```

### P2 — Nên Đạt

```
□ [P2] GC logging được bật để có data khi cần debug
    → -Xlog:gc*:file=/var/log/jenkins/gc.log:time,uptime:filecount=5,filesize=20m

□ [P2] Heap Dump khi OOM được cấu hình
    → -XX:+HeapDumpOnOutOfMemoryError
    → -XX:HeapDumpPath=/var/log/jenkins/heapdump.hprof

□ [P2] Plugin không cần thiết đã được remove
    → Audit plugin list hàng quý
    → Xóa các plugin demo, sample, hoặc không dùng

□ [P2] Build artifacts có retention policy (chính sách lưu giữ)
    → Artifact lớn (Docker images, JAR files) lưu ngoài JENKINS_HOME
    → Dùng Artifact Registry hoặc S3 thay vì lưu trong Jenkins

□ [P2] Concurrent builds được cấu hình hợp lý
    → Throttle Concurrent Builds Plugin cho expensive resources
    → Không để quá nhiều builds cùng chạy một lúc
```

---

## Tính Sẵn Sàng — Availability Checklist

### P1 — Bắt Buộc

```
□ [P1] Auto-restart khi crash được cấu hình
    → systemd service với Restart=always
    → Hoặc Kubernetes Deployment với restartPolicy: Always

□ [P1] Health check endpoint được cấu hình
    → /login page hoặc /api/json trả về 200 OK
    → Load balancer hoặc monitoring dùng endpoint này

□ [P1] Agent có thể tự reconnect khi mất kết nối
    → JNLP Agent: retry logic trong startup script
    → Kubernetes Agent: pod restart policy

□ [P1] Tài liệu RTO/RPO đã được xác định
    → RTO (Recovery Time Objective — Thời Gian Phục Hồi Mục Tiêu): tối đa bao lâu để restore
    → RPO (Recovery Point Objective — Điểm Phục Hồi Mục Tiêu): mất tối đa bao nhiêu data
```

### P2 — Nên Đạt

```
□ [P2] Multi-agent setup (nhiều agent)
    → Không phụ thuộc vào 1 agent duy nhất cho critical builds
    → Agent có labels để phân loại capacity

□ [P2] Graceful restart procedure (quy trình restart không gây gián đoạn)
    → Dùng "Prepare for Shutdown" trước khi restart
    → Notify (thông báo) team trước khi maintenance window

□ [P2] Staged rollout plan cho Jenkins upgrades
    → Test trên staging → canary deployment → production
    → Rollback plan có sẵn
```

### P3 — Tương Lai

```
□ [P3] High Availability (HA — Tính Sẵn Sàng Cao) setup (nếu cần)
    → Active-Active: CloudBees CI hoặc custom HAProxy setup
    → Active-Passive: Shared filesystem + failover

□ [P3] Jenkins trên Kubernetes với PersistentVolumeClaim
    → JENKINS_HOME trên persistent volume
    → Có thể reschedule pod sang node khác
```

---

## Giám Sát — Monitoring Checklist

### P1 — Bắt Buộc

```
□ [P1] Cảnh báo Jenkins down được cấu hình
    → Alert khi /login endpoint không trả về 200
    → PagerDuty, OpsGenie, hoặc email alert

□ [P1] Disk space alert
    → Warning khi disk < 30% free
    → Critical khi disk < 10% free

□ [P1] Memory alert
    → Warning khi heap > 80% used
    → Critical khi heap > 90% used

□ [P1] Build failure rate được theo dõi
    → Dashboard hiển thị % build success trong 24h/7 ngày
    → Alert khi failure rate tăng đột ngột
```

### P2 — Nên Đạt

```
□ [P2] Prometheus metrics được export
    → Cài Prometheus Plugin
    → Endpoint: /prometheus/
    → Scrape interval: 15-30 giây

□ [P2] Grafana dashboard được thiết lập
    → Dashboard: build queue length, executor utilization, build duration
    → Dashboard: JVM metrics (heap, GC, threads)

□ [P2] Build queue length alert
    → Alert khi queue > 10 items trong > 15 phút
    → Có thể cần thêm agent

□ [P2] Slow build alert
    → Alert khi build chạy lâu hơn 2x thời gian trung bình

□ [P2] Log aggregation (tập trung nhật ký)
    → Jenkins logs → ELK Stack hoặc Loki + Grafana
    → Có thể search logs mà không cần SSH vào máy chủ
```

---

## Backup và Restore — Checklist Khôi Phục

### P0 — Bắt Buộc Tuyệt Đối

```
□ [P0] Backup JENKINS_HOME được thực hiện tự động hàng ngày
    → ThinBackup Plugin hoặc cron job với rsync/tar
    → Backup lưu ở nơi khác (không cùng máy chủ)

□ [P0] Restore procedure đã được TEST và DOCUMENTED
    → Phải thực tế restore thử ít nhất 1 lần trước go-live
    → Thời gian restore đã được đo và nằm trong RTO

□ [P0] Backup encryption (mã hóa backup)
    → Backup chứa credentials — phải mã hóa
    → Kiểm tra backup không đọc được nếu không có key
```

### P1 — Bắt Buộc

```
□ [P1] Backup retention policy (chính sách lưu giữ backup)
    → Giữ ít nhất 7 ngày backup gần nhất
    → Monthly backup giữ 3 tháng

□ [P1] Alert khi backup fail
    → Nếu backup job không chạy hoặc lỗi → alert ngay

□ [P1] Credentials và Secrets được backup riêng
    → credentials.xml chứa encrypted secrets
    → Cần cùng master key (hudson.util.Secret) để decrypt
    → Lưu master key ở nơi an toàn riêng biệt

□ [P1] Configuration as Code được dùng (nếu có thể)
    → JCasC (Jenkins Configuration as Code) lưu config trong Git
    → Rebuild Jenkins từ scratch chỉ cần Git repo + plugin list
```

---

## Network và Firewall Checklist

### P1 — Bắt Buộc

```
□ [P1] Ports cần thiết đã được mở
    Inbound (vào Jenkins):
    → 8080/443 — Jenkins Web UI và API
    → 50000 — JNLP Agent kết nối (nếu dùng JNLP)
    → 22 — SSH (nếu dùng SSH Agent, từ Jenkins đến Agent)

    Outbound (từ Jenkins ra ngoài):
    → 443 — GitHub/GitLab (HTTPS webhooks, API)
    → 22 — GitHub/GitLab (SSH clone)
    → Ports tùy theo external services (Docker Registry, SonarQube, Slack...)

□ [P1] Jenkins không accessible trực tiếp từ internet mà không qua proxy
    → Reverse proxy (nginx/Apache/Traefik) đứng trước Jenkins
    → Jenkins bind localhost:8080, không bind 0.0.0.0:8080

□ [P1] Webhook from GitHub/GitLab có thể đến Jenkins
    → Nếu Jenkins ở private network: cần ngrok, Smee.io, hoặc VPN
    → Test bằng cách push commit và xem Jenkins có trigger build không
```

### P2 — Nên Đạt

```
□ [P2] Network Policy (nếu trên Kubernetes)
    → Chỉ allow traffic từ authorized sources đến Jenkins namespace
    → Agent pods chỉ talk đến Controller, không đến nhau

□ [P2] WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web)
    → Bảo vệ Jenkins khỏi common web attacks (OWASP Top 10)
    → Rate limiting để chống brute-force
```

---

## Checklist Đội Nhóm và Quy Trình

### P1 — Bắt Buộc

```
□ [P1] On-call runbook được viết và accessible
    → Ai cần liên hệ khi Jenkins down?
    → Các bước restart, rollback được document rõ
    → Không chỉ có 1 người biết cách vận hành Jenkins

□ [P1] Credential rotation responsibility (trách nhiệm quản lý credentials)
    → Ai chịu trách nhiệm rotate credentials?
    → Khi nhân viên nghỉ việc: credentials phải được revoke ngay

□ [P1] Change management process (quy trình quản lý thay đổi)
    → Mọi thay đổi Jenkins config phải có approval
    → Có bản ghi lịch sử thay đổi (audit trail)

□ [P1] Disaster Recovery (DR — Khôi Phục Sau Thảm Họa) drill đã được thực hiện
    → Ít nhất 1 lần/năm: simulate Jenkins failure và restore
    → Đo thời gian và so sánh với RTO target
```

### P2 — Nên Đạt

```
□ [P2] Jenkins admin role không được share (dùng chung)
    → Mỗi admin có account riêng
    → Audit log ghi rõ ai làm gì

□ [P2] Documentation về architecture hiện tại
    → Diagram kiến trúc Master/Agent
    → List agent nodes, capacity, labels
    → List integrations và service accounts được dùng

□ [P2] Post-mortem (phân tích sau sự cố) process
    → Khi Jenkins incident xảy ra: ghi lại timeline, root cause, action items
    → Review action items trong sprint tiếp theo
```

---

## Go-Live Checklist — Ngày Ra Mắt

Thực hiện theo thứ tự ngay trước và sau khi chuyển sang production:

### T-48h (Trước 48 Giờ)

```
□ Thông báo team về maintenance window
□ Đảm bảo backup đang chạy tốt
□ Verify restore procedure lần cuối trên staging
□ Review tất cả P0/P1 checklist items
□ Confirm on-call team đã sẵn sàng
```

### T-24h (Trước 24 Giờ)

```
□ Final security scan (quét bảo mật lần cuối)
□ Load test với traffic simulation (nếu có thể)
□ Verify monitoring alerts hoạt động (test alert bằng cách tạm dừng Jenkins)
□ Confirm backup của ngày hôm nay đã success
□ Prepare rollback plan chi tiết
```

### T-0 (Giờ G — Go Live)

```
□ Announce maintenance start (thông báo bắt đầu maintenance)
□ Drain current builds (để builds hiện tại hoàn thành)
    → Manage Jenkins → Prepare for Shutdown
□ Take final backup
□ Switch DNS / load balancer
□ Verify health check endpoint trả về 200
□ Test 1 pipeline end-to-end
□ Monitor metrics trong 30 phút đầu
□ Announce maintenance end
```

### T+1h (1 Giờ Sau Go Live)

```
□ Xem xét logs: có ERROR hay WARN bất thường?
□ Kiểm tra build queue không có gì bị stuck
□ Confirm tất cả agents đang online
□ Verify backup job đêm nay có schedule
□ Collect feedback từ team đầu tiên dùng
```

---

## Runbook — Sổ Tay Vận Hành Khẩn Cấp

### Jenkins Không Start

```bash
# 1. Xem lỗi
journalctl -u jenkins --since "10 minutes ago"
# Hoặc
tail -50 /var/log/jenkins/jenkins.log

# 2. Kiểm tra disk
df -h /var/lib/jenkins

# 3. Kiểm tra permissions
ls -la /var/lib/jenkins/
ls -la /var/log/jenkins/

# 4. Thử start với safe mode (không load plugins)
# Đổi startup command, thêm --safe-restart flag
# Sau đó disable plugin gây lỗi qua UI

# 5. Restore từ backup nếu cần
tar -xzf jenkins-backup-YYYYMMDD.tar.gz -C /var/lib/jenkins/
systemctl start jenkins
```

### Jenkins UI Không Responsive (Không Phản Hồi)

```bash
# 1. Kiểm tra process
ps aux | grep jenkins
# Vẫn chạy? → Có thể là GC pause hoặc deadlock

# 2. Lấy thread dump
jstack $(pgrep -f jenkins.war) > /tmp/threaddump.txt
grep -A5 "BLOCKED\|deadlock" /tmp/threaddump.txt

# 3. Kiểm tra heap
jmap -heap $(pgrep -f jenkins.war)

# 4. Nếu heap đầy → restart gracefully
# Đợi build hiện tại xong hoặc abort chúng
curl -X POST http://admin:token@jenkins:8080/quietDown
# Sau đó
systemctl restart jenkins
```

### Agent Hàng Loạt Offline

```bash
# 1. Kiểm tra network
ping agent-hostname
nc -zv jenkins-controller 50000

# 2. SSH vào agent
ssh jenkins@agent-hostname

# 3. Xem agent process
ps aux | grep jenkins-agent

# 4. Restart agent service
systemctl restart jenkins-agent

# 5. Nếu nhiều agents cùng offline → có thể Jenkins Controller bị restart
# Kiểm tra Jenkins Controller uptime
curl http://jenkins-controller:8080/api/json | python3 -m json.tool | grep uptime
```

### Khôi Phục Từ Backup

```bash
# RTO target: 2 giờ — bắt đầu đồng hồ từ đây

# 1. Dừng Jenkins (nếu đang chạy)
systemctl stop jenkins

# 2. Backup trạng thái hiện tại (để điều tra sau)
tar -czf /tmp/jenkins-corrupted-$(date +%Y%m%d-%H%M%S).tar.gz /var/lib/jenkins/

# 3. Xóa và restore từ backup
rm -rf /var/lib/jenkins/*
tar -xzf /backup/jenkins-backup-YYYYMMDD.tar.gz -C /

# 4. Fix permissions
chown -R jenkins:jenkins /var/lib/jenkins/

# 5. Start Jenkins
systemctl start jenkins

# 6. Verify
curl -f http://localhost:8080/login
# Output: 200 OK → Thành công
```

---

## Kết Luận

Một Jenkins production-ready phải đảm bảo:

| Khía Cạnh | Tiêu Chí Tối Thiểu |
|-----------|-------------------|
| **Bảo mật** | Authentication + HTTPS + Credentials trong Vault |
| **Hiệu suất** | Heap ≥ 2GB + Build Discard Policy + Agent riêng |
| **Tính sẵn sàng** | Auto-restart + Health check + Multi-agent |
| **Giám sát** | Alert khi down + Disk alert + Build metrics |
| **Khôi phục** | Daily backup + Tested restore + RTO < 2h |
| **Quy trình** | Runbook + On-call rotation + Change management |

**Không có shortcut cho production readiness (sẵn sàng production) — mỗi mục trong checklist này bảo vệ bạn khỏi một loại sự cố cụ thể.**

---

**Hoàn Thành:** [README.md](README.md) — tổng quan troubleshooting Jenkins
