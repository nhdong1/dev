# 3 — Backup & Restore: Sao Lưu và Phục Hồi Jenkins

> Không có gì chắc chắn trong vận hành hệ thống — ổ đĩa hỏng, nhầm lẫn xóa config, nâng cấp thất bại đều có thể xảy ra. Backup (sao lưu) và Restore (phục hồi) là lưới an toàn không thể thiếu. Quy tắc vàng: **một backup chưa được test restore là không phải backup**.

---

## Mục Tiêu

- Hiểu cấu trúc `JENKINS_HOME` và xác định phần nào cần backup
- Cài đặt và cấu hình ThinBackup Plugin (plugin sao lưu nhẹ)
- Thiết lập backup thủ công bằng rsync, tar, hoặc cloud storage
- Thực hiện restore từ backup trong các tình huống khác nhau
- Xây dựng chiến lược backup 3-2-1 đáng tin cậy
- Lên kế hoạch test restore định kỳ

---

## Phần 1: Cái Gì Cần Backup?

### Quan Trọng — Phải Backup

```
JENKINS_HOME/
├── config.xml                    ← Cấu hình Jenkins master (rất quan trọng)
├── credentials.xml               ← Credentials đã mã hóa (rất quan trọng)
├── secrets/                      ← Master key và secret files (bắt buộc)
│   ├── master.key
│   ├── hudson.util.Secret
│   └── org.jenkinsci.plugins.*
├── jobs/*/config.xml             ← Cấu hình từng job (rất quan trọng)
├── plugins/                      ← Plugin đã cài (quan trọng)
│   └── *.jpi hoặc *.hpi
├── users/                        ← Tài khoản người dùng
├── nodes/                        ← Cấu hình Agent Node
└── casc_configs/                 ← Jenkins Configuration as Code (nếu dùng)
```

### Tùy Chọn — Có Thể Backup

```
JENKINS_HOME/
├── jobs/*/builds/                ← Build history (lịch sử build) — lớn, tùy nhu cầu
├── workspace/                    ← Có thể rebuild, thường bỏ qua
└── logs/                         ← Jenkins master logs (lớn, thường bỏ qua)
```

### Không Cần Backup

```
JENKINS_HOME/
├── war/                          ← Jenkins WAR file — tải lại khi cần
├── cache/                        ← Cache file — tự tạo lại
└── workspace/                    ← Có thể rebuild từ SCM
```

---

## Phần 2: ThinBackup Plugin — Backup Tự Động Trong Jenkins

### Cài Đặt

**Manage Jenkins → Plugin Manager → Available** → tìm `ThinBackup` → Install

### Cấu Hình ThinBackup

Sau khi cài, truy cập **Manage Jenkins → ThinBackup → Settings**:

```
Backup directory:     /backup/jenkins
            ↑ Thư mục đích — nên ở partition (phân vùng) khác với JENKINS_HOME

Full backup schedule: H 2 * * 0
            ↑ Full backup mỗi Chủ Nhật lúc 2 giờ sáng

Differential backup schedule: H 2 * * 1-6
            ↑ Differential backup (backup chênh lệch) Thứ Hai đến Thứ Bảy

Max stored full backups: 4
            ↑ Giữ 4 bản full backup (khoảng 1 tháng)

[x] Backup build results
[x] Backup 'userContent' folder
[x] Clean up differential backups after each full backup
[x] Move old backups to zip files
```

### Thực Hiện Backup Thủ Công Qua ThinBackup

1. Truy cập **Manage Jenkins → ThinBackup**
2. Click **Backup Now** → chọn loại backup (Full hoặc Differential)
3. Chờ quá trình hoàn thành

### Restore Từ ThinBackup

1. Truy cập **Manage Jenkins → ThinBackup → Restore**
2. Chọn backup cần restore từ danh sách
3. Nhấn **Restore** → Jenkins sẽ tự động restart

```
Lưu ý quan trọng khi restore:
  • Tắt Jenkins trước khi restore thủ công (qua filesystem)
  • ThinBackup có thể restore trực tiếp qua UI mà không cần tắt
  • Sau khi restore, kiểm tra credentials và plugin hoạt động đúng
```

---

## Phần 3: Backup Thủ Công Bằng Script

### Backup Toàn Bộ JENKINS_HOME

```bash
#!/bin/bash
# jenkins-backup.sh — script backup Jenkins thủ công

JENKINS_HOME="/var/jenkins_home"
BACKUP_DIR="/backup/jenkins"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="jenkins_backup_${TIMESTAMP}"

echo "[$(date)] Bắt đầu backup Jenkins..."

# Bước 1: Đưa Jenkins vào quiet mode (chế độ yên tĩnh) — ngừng nhận job mới
# Thực hiện qua Jenkins API
JENKINS_URL="http://localhost:8080"
JENKINS_ADMIN="admin"
JENKINS_TOKEN="your-api-token"

curl -s -X POST \
  "${JENKINS_URL}/quietDown" \
  --user "${JENKINS_ADMIN}:${JENKINS_TOKEN}"

echo "Jenkins đang ở quiet mode — đợi build hiện tại kết thúc..."
sleep 30

# Bước 2: Tạo thư mục backup
mkdir -p "${BACKUP_DIR}/${BACKUP_NAME}"

# Bước 3: Backup các phần quan trọng (không backup workspace và build history lớn)
tar -czf "${BACKUP_DIR}/${BACKUP_NAME}/jenkins_config.tar.gz" \
  --exclude="${JENKINS_HOME}/workspace" \
  --exclude="${JENKINS_HOME}/jobs/*/builds" \
  --exclude="${JENKINS_HOME}/war" \
  --exclude="${JENKINS_HOME}/cache" \
  "${JENKINS_HOME}/"

echo "Kích thước backup: $(du -sh ${BACKUP_DIR}/${BACKUP_NAME}/jenkins_config.tar.gz | cut -f1)"

# Bước 4: Backup chỉ cấu hình job (nhẹ hơn nhiều)
find "${JENKINS_HOME}/jobs" -name "config.xml" | while read f; do
    relative_path="${f#${JENKINS_HOME}/}"
    mkdir -p "${BACKUP_DIR}/${BACKUP_NAME}/jobs_configs/$(dirname $relative_path)"
    cp "$f" "${BACKUP_DIR}/${BACKUP_NAME}/jobs_configs/$relative_path"
done

# Bước 5: Thoát quiet mode
curl -s -X POST \
  "${JENKINS_URL}/cancelQuietDown" \
  --user "${JENKINS_ADMIN}:${JENKINS_TOKEN}"

echo "[$(date)] Backup hoàn thành: ${BACKUP_DIR}/${BACKUP_NAME}"

# Bước 6: Xóa backup cũ hơn 30 ngày
find "${BACKUP_DIR}" -name "jenkins_backup_*" -mtime +30 -exec rm -rf {} \;
echo "Đã dọn backup cũ hơn 30 ngày."
```

### Backup Chỉ Phần Cấu Hình (Config-only Backup)

```bash
#!/bin/bash
# jenkins-config-backup.sh — backup nhẹ, chỉ lưu cấu hình

JENKINS_HOME="/var/jenkins_home"
BACKUP_DIR="/backup/jenkins/configs"
DATE=$(date +%Y%m%d)

mkdir -p "${BACKUP_DIR}"

# Backup các file cấu hình quan trọng
tar -czf "${BACKUP_DIR}/jenkins_configs_${DATE}.tar.gz" \
  "${JENKINS_HOME}/config.xml" \
  "${JENKINS_HOME}/credentials.xml" \
  "${JENKINS_HOME}/secrets/" \
  "${JENKINS_HOME}/users/" \
  "${JENKINS_HOME}/nodes/" \
  $(find "${JENKINS_HOME}/jobs" -name "config.xml" 2>/dev/null)

echo "Config backup: ${BACKUP_DIR}/jenkins_configs_${DATE}.tar.gz"
echo "Kích thước: $(du -sh ${BACKUP_DIR}/jenkins_configs_${DATE}.tar.gz | cut -f1)"
```

### Đồng Bộ Backup Lên AWS S3

```bash
#!/bin/bash
# Chạy sau jenkins-backup.sh — đồng bộ lên S3

BACKUP_DIR="/backup/jenkins"
S3_BUCKET="s3://company-jenkins-backups"
AWS_PROFILE="jenkins-backup-role"

# Đồng bộ backup mới nhất lên S3 (chỉ upload file mới/thay đổi)
aws s3 sync "${BACKUP_DIR}/" "${S3_BUCKET}/" \
  --profile "${AWS_PROFILE}" \
  --storage-class STANDARD_IA \
  --exclude "*.tmp"

echo "Backup đã được đồng bộ lên ${S3_BUCKET}"

# Áp dụng S3 Lifecycle Policy (chính sách vòng đời) để tự động xóa backup cũ
# (Cấu hình trên AWS Console hoặc Terraform — không phải bash)
```

### Cấu Hình Cron Tự Động

```bash
# Thêm vào crontab của jenkins user (hoặc root)
crontab -e

# Backup cấu hình hàng ngày lúc 1 giờ sáng
0 1 * * *  /opt/scripts/jenkins-config-backup.sh >> /var/log/jenkins-backup.log 2>&1

# Full backup hàng tuần vào Chủ Nhật lúc 2 giờ sáng
0 2 * * 0  /opt/scripts/jenkins-backup.sh >> /var/log/jenkins-backup.log 2>&1

# Đồng bộ lên S3 sau full backup
30 2 * * 0  /opt/scripts/sync-to-s3.sh >> /var/log/jenkins-backup.log 2>&1
```

---

## Phần 4: Restore Jenkins Từ Backup

### Tình Huống 1: Restore Cấu Hình Sau Khi Xóa Nhầm

```bash
# Jenkins vẫn đang chạy, chỉ cần phục hồi một số file

# Ví dụ: khôi phục file config.xml bị xóa nhầm
BACKUP_ARCHIVE="/backup/jenkins/jenkins_backup_20260511_020000/jenkins_config.tar.gz"

# Giải nén chỉ file cần thiết
tar -xzf "${BACKUP_ARCHIVE}" \
  --strip-components=2 \
  -C /var/jenkins_home/ \
  "var/jenkins_home/config.xml"

# Reload Jenkins config mà không cần restart (chỉ áp dụng cho một số loại config)
curl -s -X POST http://admin:token@localhost:8080/reload
```

### Tình Huống 2: Restore Toàn Bộ (Full Restore)

```bash
#!/bin/bash
# jenkins-restore.sh — restore toàn bộ Jenkins từ backup

BACKUP_ARCHIVE="/backup/jenkins/jenkins_backup_20260511_020000/jenkins_config.tar.gz"
JENKINS_HOME="/var/jenkins_home"
JENKINS_USER="jenkins"

echo "=== CẢNH BÁO: Quá trình này sẽ ghi đè toàn bộ JENKINS_HOME ==="
echo "Backup cần restore: ${BACKUP_ARCHIVE}"
read -p "Tiếp tục? (yes/no): " confirm
[[ "$confirm" != "yes" ]] && exit 1

# Bước 1: Dừng Jenkins
echo "Đang dừng Jenkins service..."
systemctl stop jenkins
sleep 5

# Bước 2: Backup trạng thái hiện tại (phòng trường hợp restore thất bại)
ROLLBACK_BACKUP="/tmp/jenkins_pre_restore_$(date +%Y%m%d_%H%M%S).tar.gz"
echo "Tạo rollback backup tại: ${ROLLBACK_BACKUP}"
tar -czf "${ROLLBACK_BACKUP}" "${JENKINS_HOME}/"

# Bước 3: Xóa JENKINS_HOME hiện tại (chỉ giữ lại thư mục workspace để không mất code)
find "${JENKINS_HOME}" \
  -mindepth 1 \
  -maxdepth 1 \
  ! -name "workspace" \
  -exec rm -rf {} \;

# Bước 4: Giải nén backup
echo "Đang giải nén backup..."
tar -xzf "${BACKUP_ARCHIVE}" -C /

# Bước 5: Đặt lại quyền sở hữu file
chown -R "${JENKINS_USER}:${JENKINS_USER}" "${JENKINS_HOME}"

# Bước 6: Khởi động Jenkins
echo "Đang khởi động Jenkins..."
systemctl start jenkins
sleep 30

# Bước 7: Kiểm tra Jenkins đã hoạt động chưa
if curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/login | grep -q "200"; then
    echo "✅ Restore thành công — Jenkins đang hoạt động"
    rm -f "${ROLLBACK_BACKUP}"
else
    echo "❌ Jenkins không khởi động được sau restore"
    echo "Đang rollback về trạng thái trước..."
    systemctl stop jenkins
    tar -xzf "${ROLLBACK_BACKUP}" -C /
    systemctl start jenkins
fi
```

### Tình Huống 3: Restore Trên Server Mới (Disaster Recovery)

```bash
# Kịch bản: server Jenkins cũ hỏng, cần khôi phục trên server mới

# Bước 1: Cài đặt Jenkins cùng phiên bản trên server mới
# Tải Jenkins WAR file từ jenkins.io với đúng version number
wget https://updates.jenkins.io/download/war/2.440.3/jenkins.war

# Bước 2: Khởi động Jenkins lần đầu để tạo JENKINS_HOME
JENKINS_HOME=/var/jenkins_home java -jar jenkins.war &
sleep 30
pkill -f jenkins.war

# Bước 3: Tải backup từ S3
aws s3 cp \
  "s3://company-jenkins-backups/jenkins_backup_20260511_020000/jenkins_config.tar.gz" \
  /tmp/jenkins_restore.tar.gz

# Bước 4: Giải nén backup vào JENKINS_HOME
tar -xzf /tmp/jenkins_restore.tar.gz -C /

# Bước 5: Cài đặt đúng phiên bản plugin (từ danh sách đã lưu)
# Plugin được backup cùng trong JENKINS_HOME/plugins/

# Bước 6: Khởi động Jenkins và kiểm tra
JENKINS_HOME=/var/jenkins_home java -jar jenkins.war

echo "Kiểm tra: http://new-server:8080"
echo "Đăng nhập và verify:"
echo "  - Jobs có đủ không?"
echo "  - Credentials có hoạt động không?"
echo "  - Agent nodes có kết nối không?"
echo "  - Plugin có đúng version không?"
```

---

## Phần 5: Backup Jenkins Configuration as Code (JCasC)

Nếu dùng **JCasC** (Jenkins Configuration as Code — Cấu Hình Jenkins Dạng Code), backup trở nên đơn giản hơn nhiều vì toàn bộ cấu hình được lưu trong file YAML dưới dạng code.

```yaml
# jenkins.yaml — toàn bộ cấu hình Jenkins trong một file
jenkins:
  systemMessage: "Jenkins Production - Company XYZ"
  numExecutors: 0              # Master không chạy build trực tiếp
  mode: EXCLUSIVE
  
  securityRealm:
    ldap:
      configurations:
        - server: "ldap://ldap.company.com"
          rootDN: "dc=company,dc=com"
  
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            assignments:
              - "platform-team"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              id: "docker-registry-credentials"
              username: "ci-user"
              password: "${DOCKER_REGISTRY_PASSWORD}"   # Lấy từ environment variable

tool:
  git:
    installations:
      - name: "Default"
        home: "/usr/bin/git"
```

```bash
# Backup JCasC — chỉ cần backup file YAML
# Thường lưu trong Git repository — không cần backup riêng!
git add jenkins.yaml
git commit -m "chore: update Jenkins configuration"
git push origin main
```

```
Lợi ích của JCasC cho backup:
  ✅ Cấu hình dưới dạng code trong Git → lịch sử thay đổi đầy đủ
  ✅ Restore = chạy lại JCasC trên Jenkins mới → không cần backup phức tạp
  ✅ Review thay đổi qua Pull Request
  ✅ Reproduce (tái tạo) Jenkins environment trong vài phút
```

---

## Phần 6: Chiến Lược Backup 3-2-1

```
Chiến lược 3-2-1:
  3 bản sao dữ liệu
  2 loại storage (ổ lưu trữ) khác nhau  
  1 bản offsite (ngoài vị trí vật lý hiện tại)

Triển khai thực tế:
  ┌──────────────────────────────────────────────────────┐
  │  Bản 1: JENKINS_HOME trực tiếp trên server (live)   │ ← Local disk
  │  Bản 2: ThinBackup/tar.gz trên NFS mount             │ ← Network storage
  │  Bản 3: Đồng bộ lên AWS S3 / Google Cloud Storage   │ ← Offsite cloud
  └──────────────────────────────────────────────────────┘

Lịch backup:
  Hàng ngày:   Differential backup (chênh lệch từ full gần nhất)
  Hàng tuần:   Full backup (toàn bộ)
  Hàng tháng:  Test restore trên môi trường staging
```

### Retention Policy (Chính Sách Lưu Giữ)

| Loại Backup | Giữ Bao Lâu | Lý Do |
|-------------|-------------|-------|
| Daily differential | 7 ngày | Phục hồi lỗi trong tuần gần nhất |
| Weekly full | 4 tuần | Phục hồi lỗi trong tháng gần nhất |
| Monthly full | 12 tháng | Compliance, audit requirement |
| Pre-upgrade snapshot | Vô thời hạn | An toàn khi nâng cấp |

---

## Phần 7: Test Restore — Không Thể Bỏ Qua

```bash
#!/bin/bash
# test-restore.sh — kiểm tra backup có restore được không

echo "=== TEST RESTORE ĐỊNH KỲ ($(date)) ==="

BACKUP_FILE=$1
STAGING_JENKINS_HOME="/tmp/jenkins_restore_test"

# Bước 1: Chuẩn bị môi trường test
rm -rf "${STAGING_JENKINS_HOME}"
mkdir -p "${STAGING_JENKINS_HOME}"

# Bước 2: Giải nén backup vào thư mục test
tar -xzf "${BACKUP_FILE}" -C "${STAGING_JENKINS_HOME}/"
echo "✅ Giải nén thành công"

# Bước 3: Kiểm tra các file quan trọng có đủ không
check_file() {
    if [[ -f "$1" ]]; then
        echo "✅ Có: $1"
    else
        echo "❌ Thiếu: $1"
        RESTORE_OK=false
    fi
}

RESTORE_OK=true
check_file "${STAGING_JENKINS_HOME}/var/jenkins_home/config.xml"
check_file "${STAGING_JENKINS_HOME}/var/jenkins_home/credentials.xml"
check_file "${STAGING_JENKINS_HOME}/var/jenkins_home/secrets/master.key"

# Bước 4: Khởi động Jenkins test (port khác để không xung đột)
docker run -d \
  --name jenkins-restore-test \
  -p 8090:8080 \
  -v "${STAGING_JENKINS_HOME}/var/jenkins_home:/var/jenkins_home" \
  jenkins/jenkins:lts

echo "Jenkins test đang khởi động trên port 8090..."
sleep 60

# Bước 5: Kiểm tra Jenkins có responsive không
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8090/login)
if [[ "${HTTP_CODE}" == "200" ]]; then
    echo "✅ Jenkins restore test THÀNH CÔNG — UI accessible"
else
    echo "❌ Jenkins restore test THẤT BẠI — HTTP ${HTTP_CODE}"
    RESTORE_OK=false
fi

# Bước 6: Dọn dẹp
docker stop jenkins-restore-test
docker rm jenkins-restore-test
rm -rf "${STAGING_JENKINS_HOME}"

# Bước 7: Báo cáo kết quả
if [[ "${RESTORE_OK}" == "true" ]]; then
    echo "=== KẾT QUẢ: RESTORE TEST THÀNH CÔNG ==="
else
    echo "=== KẾT QUẢ: RESTORE TEST THẤT BẠI — CẦN XEM XÉT NGAY ==="
    # Gửi cảnh báo
    curl -X POST -H 'Content-type: application/json' \
      --data '{"text":"❌ Jenkins restore test THẤT BẠI — kiểm tra ngay backup!"}' \
      "${SLACK_WEBHOOK_URL}"
fi
```

---

## Tóm Tắt: Backup Checklist

```markdown
## Thiết Lập Backup
[ ] Cài ThinBackup Plugin
[ ] Cấu hình backup directory (partition khác với JENKINS_HOME)
[ ] Đặt lịch full backup hàng tuần và differential hàng ngày
[ ] Cấu hình sync lên cloud storage (S3, GCS, Azure Blob)
[ ] Lưu danh sách plugin đã cài (plugins.txt)

## Vận Hành Thường Xuyên
[ ] Verify backup chạy thành công hàng ngày (check size, timestamp)
[ ] Test restore hàng tháng trên môi trường staging
[ ] Kiểm tra dung lượng backup storage hàng tuần
[ ] Tài liệu hóa Restore Runbook (sổ tay phục hồi) — ai làm gì khi nào

## Trước Mỗi Lần Nâng Cấp
[ ] Thực hiện full backup ngay trước khi nâng cấp
[ ] Ghi lại phiên bản Jenkins và plugin hiện tại
[ ] Kiểm tra backup hợp lệ (test restore nếu có thể)
[ ] Có sẵn rollback plan (kế hoạch hoàn tác)
```

---

## Câu Hỏi Phỏng Vấn

1. **Bạn backup Jenkins như thế nào? Cụ thể backup những gì?**
   → ThinBackup Plugin cho daily differential và weekly full backup. Backup `config.xml`, `credentials.xml`, `secrets/`, `jobs/*/config.xml`, `users/`, `nodes/`. Workspace và build history có thể bỏ qua hoặc backup riêng. Sync lên S3 cho offsite copy.

2. **Tại sao phải test restore định kỳ?**
   → Backup có thể bị corrupt (hỏng) mà không biết. Restore procedure có thể thay đổi theo phiên bản Jenkins. Test restore phát hiện vấn đề khi còn không có áp lực, không phải khi đang có incident.

3. **Jenkins server bị hỏng lúc 2 giờ sáng, RTO (Recovery Time Objective — thời gian phục hồi mục tiêu) là 1 giờ. Bạn làm gì?**
   → Khởi động server mới, tải Jenkins WAR cùng phiên bản từ cache/S3, giải nén backup mới nhất từ S3, cài plugin từ danh sách đã lưu, test nhanh và thông báo team. Với runbook chuẩn bị sẵn và backup đầy đủ, 1 giờ là đủ.

4. **Sự khác biệt giữa Full Backup và Differential Backup trong ThinBackup?**
   → Full Backup: backup toàn bộ JENKINS_HOME, chiếm nhiều dung lượng nhưng restore đơn giản. Differential Backup (backup chênh lệch): chỉ backup những file thay đổi từ lần full backup gần nhất — nhanh hơn, nhỏ hơn, nhưng restore cần cả full + tất cả differential sau đó.

5. **Tại sao `secrets/` directory là bắt buộc phải backup?**
   → `secrets/master.key` và các file trong `secrets/` là khóa mã hóa dùng để decrypt (giải mã) credentials. Nếu thiếu, tất cả credentials đã lưu sẽ không thể dùng được dù có file `credentials.xml`. Đây là điểm hay bị bỏ qua khi restore.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
