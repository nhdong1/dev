# 4 — Upgrade Guide: Hướng Dẫn Nâng Cấp Jenkins An Toàn

> Nâng cấp Jenkins không đúng cách có thể gây downtime (thời gian ngừng hoạt động) kéo dài, plugin không tương thích, hoặc thậm chí mất dữ liệu. Bài này cung cấp quy trình nâng cấp bài bản: từ hiểu rõ các loại release (phiên bản phát hành), kiểm tra tương thích plugin, đến thực hiện rollback (quay lại phiên bản cũ) khi cần.

---

## Mục Tiêu

- Hiểu sự khác biệt giữa Jenkins LTS (Long-Term Support — phiên bản hỗ trợ dài hạn) và Weekly release (phiên bản hàng tuần)
- Đọc và phân tích Upgrade Notes (ghi chú nâng cấp) và Changelog (nhật ký thay đổi)
- Kiểm tra plugin compatibility (tương thích plugin) trước khi nâng cấp
- Thực hiện nâng cấp Jenkins an toàn với quy trình từng bước
- Chuẩn bị và thực thi rollback plan (kế hoạch hoàn tác) khi gặp sự cố

---

## Phần 1: LTS vs Weekly Release — Chọn Loại Phiên Bản Nào?

### Jenkins LTS (Long-Term Support)

```
Đặc điểm:
  • Phát hành định kỳ mỗi 4 tuần (minor update: .1, .2, .3)
  • Một dòng LTS mới mỗi 12 tuần (ví dụ: 2.440.x → 2.452.x)
  • Chỉ nhận backport (vá ngược) cho bug fix và security fix quan trọng
  • Ổn định hơn nhiều so với Weekly

Ví dụ phiên bản:
  2.440.1 → 2.440.2 → 2.440.3 (cùng dòng LTS)
  ↓ (nâng cấp lên dòng LTS mới)
  2.452.1 → 2.452.2 → 2.452.3

Khuyến nghị cho: Production environment (môi trường thực)
```

### Jenkins Weekly Release

```
Đặc điểm:
  • Phát hành mỗi tuần (thứ Ba hoặc thứ Tư)
  • Luôn có tính năng mới nhất
  • Có thể có bug mới chưa được phát hiện
  • Tần suất nâng cấp cao → tốn thời gian vận hành

Ví dụ phiên bản:
  2.450 → 2.451 → 2.452 → 2.453 (mỗi tuần một phiên bản)

Khuyến nghị cho: Development environment (môi trường phát triển), testing mới nhất
```

### Ma Trận Quyết Định

| Môi Trường | Loại Phiên Bản | Tần Suất Nâng Cấp |
|------------|----------------|-------------------|
| Production | LTS | Mỗi dòng LTS mới (~3 tháng) |
| Staging | LTS | Cùng với production |
| Development/Lab | Weekly | Hàng tháng hoặc khi cần |

---

## Phần 2: Trước Khi Nâng Cấp — Chuẩn Bị

### Bước 1: Đọc Upgrade Notes (Ghi Chú Nâng Cấp)

Luôn đọc kỹ trước khi nâng cấp:

- **Jenkins Changelog:** `https://www.jenkins.io/changelog-stable/` (LTS)
- **Upgrade Notes:** `https://www.jenkins.io/doc/upgrade-guide/`
- **LTS Upgrade Guide:** Tìm phiên bản hiện tại → phiên bản đích

```
Những điểm quan trọng cần chú ý trong Upgrade Notes:
  ⚠️  Breaking changes (thay đổi không tương thích ngược)
  ⚠️  Deprecated features sẽ bị xóa (tính năng sắp lỗi thời)
  ⚠️  Java version requirement (yêu cầu phiên bản Java mới nhất)
  ⚠️  Plugin minimum version requirement (phiên bản plugin tối thiểu)
  ⚠️  Database migration (di chuyển cơ sở dữ liệu) tự động
```

### Bước 2: Kiểm Tra Plugin Compatibility (Tương Thích Plugin)

```bash
# Lấy danh sách tất cả plugin đang cài và phiên bản
curl -s \
  http://admin:token@localhost:8080/pluginManager/api/json?depth=1 \
  | jq -r '.plugins[] | "\(.shortName):\(.version)"' \
  | sort > current_plugins.txt

cat current_plugins.txt
# git:4.13.0
# pipeline-build-step:2.18
# kubernetes:3904.v15f484497267
# ...
```

```bash
# Kiểm tra plugin compatibility với phiên bản Jenkins mới
# Dùng Plugin Compatibility Checker trên jenkins.io hoặc:

# Cách thủ công: xem từng plugin trên plugins.jenkins.io
# Tìm tab "Releases" → xem "Minimum Jenkins version required"
# Ví dụ: nếu plugin yêu cầu Jenkins >= 2.387 nhưng bạn đang dùng 2.375 → cần update plugin

# Script kiểm tra (gọi API plugins.jenkins.io)
#!/bin/bash
TARGET_JENKINS="2.452.1"  # Phiên bản Jenkins đích

while IFS=: read -r plugin version; do
    # Gọi API để lấy thông tin plugin
    info=$(curl -s "https://plugins.jenkins.io/api/plugin/${plugin}")
    min_jenkins=$(echo $info | jq -r '.requiredCore // "unknown"')
    
    echo "Plugin: ${plugin} ${version} | Yêu cầu Jenkins: ${min_jenkins}"
done < current_plugins.txt
```

### Bước 3: Chuẩn Bị Môi Trường Staging (Thử Nghiệm)

```bash
# Dựng Jenkins staging với phiên bản mới trên Docker
# Dùng dữ liệu từ backup production

docker run -d \
  --name jenkins-staging \
  -p 8090:8080 \
  -v /backup/jenkins/jenkins_backup_latest/var/jenkins_home:/var/jenkins_home \
  jenkins/jenkins:2.452.1-lts

echo "Jenkins staging đang chạy trên http://staging-server:8090"
echo "Kiểm tra:"
echo "  1. Login có hoạt động không?"
echo "  2. Jobs có hiển thị đủ không?"
echo "  3. Credentials có hoạt động không?"
echo "  4. Trigger một số pipeline thử"
echo "  5. Plugin nào bị cảnh báo incompatible?"
```

### Bước 4: Tạo Backup Trước Khi Nâng Cấp

```bash
# Backup khẩn cấp ngay trước khi nâng cấp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/jenkins/pre_upgrade_${TIMESTAMP}"

mkdir -p "${BACKUP_DIR}"
tar -czf "${BACKUP_DIR}/jenkins_pre_upgrade.tar.gz" /var/jenkins_home/

echo "Pre-upgrade backup: ${BACKUP_DIR}/jenkins_pre_upgrade.tar.gz"
echo "Kích thước: $(du -sh ${BACKUP_DIR}/jenkins_pre_upgrade.tar.gz | cut -f1)"
```

---

## Phần 3: Quy Trình Nâng Cấp — Từng Bước

### Phương Pháp 1: Nâng Cấp Qua WAR File

```bash
#!/bin/bash
# jenkins-upgrade.sh — nâng cấp Jenkins qua WAR file

JENKINS_URL="http://localhost:8080"
JENKINS_USER="admin"
JENKINS_TOKEN="your-api-token"
TARGET_VERSION="2.452.1"  # Phiên bản đích
JENKINS_WAR="/usr/share/jenkins/jenkins.war"
BACKUP_WAR="/tmp/jenkins_backup.war"

echo "=== Bắt Đầu Nâng Cấp Jenkins → ${TARGET_VERSION} ==="
echo "Thời gian bắt đầu: $(date)"

# Bước 1: Ghi lại phiên bản hiện tại
CURRENT_VERSION=$(curl -s -I "${JENKINS_URL}/login" | grep -i "X-Jenkins:" | awk '{print $2}' | tr -d '\r')
echo "Phiên bản hiện tại: ${CURRENT_VERSION}"
echo "Phiên bản đích:     ${TARGET_VERSION}"

# Bước 2: Đưa Jenkins vào quiet mode (ngừng nhận build mới)
echo "Đưa Jenkins vào Quiet Mode..."
curl -s -X POST "${JENKINS_URL}/quietDown" \
  --user "${JENKINS_USER}:${JENKINS_TOKEN}"

# Đợi các build đang chạy kết thúc
echo "Đợi build đang chạy kết thúc (tối đa 10 phút)..."
timeout=600
elapsed=0
while [[ $elapsed -lt $timeout ]]; do
    running_builds=$(curl -s "${JENKINS_URL}/api/json?tree=jobs[builds[number,result]]" \
      --user "${JENKINS_USER}:${JENKINS_TOKEN}" \
      | jq '[.jobs[].builds[] | select(.result == null)] | length')
    
    if [[ "${running_builds}" == "0" ]]; then
        echo "Không còn build nào đang chạy."
        break
    fi
    echo "Còn ${running_builds} build đang chạy..."
    sleep 15
    elapsed=$((elapsed + 15))
done

# Bước 3: Backup WAR file hiện tại (để rollback nếu cần)
cp "${JENKINS_WAR}" "${BACKUP_WAR}"
echo "Backup WAR file: ${BACKUP_WAR}"

# Bước 4: Tải WAR file mới
echo "Tải Jenkins ${TARGET_VERSION}..."
wget -q "https://updates.jenkins.io/download/war/${TARGET_VERSION}/jenkins.war" \
     -O "/tmp/jenkins_new.war"

# Bước 5: Thay thế WAR file
cp "/tmp/jenkins_new.war" "${JENKINS_WAR}"
chown jenkins:jenkins "${JENKINS_WAR}"

# Bước 6: Restart Jenkins service
echo "Khởi động lại Jenkins..."
systemctl restart jenkins

# Bước 7: Chờ Jenkins khởi động
echo "Đợi Jenkins khởi động..."
sleep 60
timeout=300
elapsed=0
while [[ $elapsed -lt $timeout ]]; do
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "${JENKINS_URL}/login" 2>/dev/null)
    if [[ "${HTTP_CODE}" == "200" ]]; then
        echo "✅ Jenkins đã khởi động thành công"
        break
    fi
    echo "HTTP ${HTTP_CODE} — đang chờ..."
    sleep 10
    elapsed=$((elapsed + 10))
done

# Bước 8: Xác nhận phiên bản mới
NEW_VERSION=$(curl -s -I "${JENKINS_URL}/login" | grep -i "X-Jenkins:" | awk '{print $2}' | tr -d '\r')
if [[ "${NEW_VERSION}" == "${TARGET_VERSION}" ]]; then
    echo "✅ Nâng cấp thành công: ${CURRENT_VERSION} → ${NEW_VERSION}"
else
    echo "❌ Phiên bản không khớp: Expected ${TARGET_VERSION}, Got ${NEW_VERSION}"
fi

echo "=== Nâng Cấp Hoàn Thành: $(date) ==="
```

### Phương Pháp 2: Nâng Cấp Docker Jenkins

```yaml
# docker-compose.yml — cập nhật tag Jenkins
version: '3.8'
services:
  jenkins:
    # Thay đổi: 2.440.3-lts → 2.452.1-lts
    image: jenkins/jenkins:2.452.1-lts
    
    ports:
      - "8080:8080"
      - "50000:50000"
    
    volumes:
      - jenkins_home:/var/jenkins_home
    
    environment:
      - JAVA_OPTS=-Xmx2g -Xms1g
    
    restart: unless-stopped

volumes:
  jenkins_home:
    external: true    # Volume tồn tại độc lập với container
```

```bash
# Thực hiện nâng cấp Docker Jenkins

# Backup volume trước
docker run --rm \
  -v jenkins_home:/source \
  -v /backup/jenkins:/backup \
  alpine tar -czf /backup/jenkins_pre_upgrade.tar.gz -C /source .

# Pull image mới
docker pull jenkins/jenkins:2.452.1-lts

# Dừng Jenkins hiện tại và khởi động với image mới
docker compose down
docker compose up -d

# Theo dõi log khi khởi động
docker compose logs -f jenkins

# Kiểm tra sức khỏe
docker inspect jenkins --format='{{.State.Health.Status}}'
```

### Phương Pháp 3: Nâng Cấp Jenkins Trên Kubernetes

```yaml
# jenkins-deployment.yaml — cập nhật image version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  strategy:
    type: RollingUpdate           # RollingUpdate — cập nhật cuốn chiếu (không downtime)
    rollingUpdate:
      maxSurge: 1                 # Tạo 1 pod mới trước khi xóa pod cũ
      maxUnavailable: 0           # Không cho phép pod nào unavailable trong quá trình update
  template:
    spec:
      containers:
        - name: jenkins
          # Thay đổi image tag
          image: jenkins/jenkins:2.452.1-lts
          
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 6
          
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 120
            periodSeconds: 30
```

```bash
# Áp dụng nâng cấp trên Kubernetes
kubectl apply -f jenkins-deployment.yaml

# Theo dõi tiến trình rolling update
kubectl rollout status deployment/jenkins -n jenkins

# Kiểm tra pod mới chạy ổn
kubectl get pods -n jenkins

# Xem log của pod mới
kubectl logs -f -n jenkins deployment/jenkins

# Nếu có vấn đề — rollback ngay
kubectl rollout undo deployment/jenkins -n jenkins
```

---

## Phần 4: Sau Khi Nâng Cấp — Kiểm Tra

### Checklist Kiểm Tra Sau Nâng Cấp

```markdown
## Kiểm Tra Ngay Sau Nâng Cấp (< 15 phút)

[ ] Jenkins UI accessible — http://jenkins:8080/login trả về 200
[ ] Phiên bản Jenkins đúng (Manage Jenkins → About Jenkins)
[ ] Login với tài khoản admin thành công
[ ] Trang Manage Jenkins không có lỗi đỏ critical
[ ] Danh sách Plugin không có plugin "requires restart" hoặc "failed"

## Kiểm Tra Chức Năng (< 30 phút)

[ ] Trigger một build thủ công cho pipeline quan trọng nhất
[ ] Build hoàn thành thành công
[ ] Checkout code từ Git hoạt động (credentials vẫn hợp lệ)
[ ] Docker build hoạt động (nếu dùng Docker agent)
[ ] Kubernetes agent có thể spin up pod mới
[ ] Notification (Slack/email) được gửi đúng

## Kiểm Tra Nâng Cao (< 2 giờ)

[ ] Multibranch Pipeline phát hiện nhánh mới
[ ] Shared Libraries load đúng phiên bản
[ ] Credentials cho tất cả integration vẫn hoạt động
[ ] Webhook từ GitHub/GitLab vẫn trigger build
[ ] Không có lỗi trong Jenkins system log (/var/log/jenkins/jenkins.log)
```

### Cập Nhật Plugin Sau Nâng Cấp Jenkins

```bash
# Xem danh sách plugin cần cập nhật qua Jenkins CLI
java -jar jenkins-cli.jar \
  -s http://localhost:8080 \
  -auth admin:token \
  list-plugins | grep -i "update"

# Cập nhật tất cả plugin cần thiết
java -jar jenkins-cli.jar \
  -s http://localhost:8080 \
  -auth admin:token \
  install-plugin $(java -jar jenkins-cli.jar -s http://localhost:8080 -auth admin:token list-plugins | grep -i "update" | awk '{print $1}') -restart
```

---

## Phần 5: Rollback Plan — Kế Hoạch Hoàn Tác

### Khi Nào Cần Rollback?

```
Dấu hiệu cần rollback NGAY:
  ❌ Jenkins không khởi động sau nâng cấp
  ❌ Crash loop (vòng lặp sập) trong container/service
  ❌ Tất cả credentials bị invalid (không hợp lệ)
  ❌ Plugin thiết yếu không tương thích và không có update
  ❌ Build thất bại 100% do lỗi từ Jenkins (không phải do code)

Dấu hiệu nên cân nhắc rollback:
  ⚠️  Một số tính năng quan trọng bị broken
  ⚠️  Performance giảm đáng kể so với trước
  ⚠️  Plugin không tương thích, ảnh hưởng nhiều pipeline
```

### Quy Trình Rollback WAR File

```bash
#!/bin/bash
# jenkins-rollback.sh

JENKINS_WAR="/usr/share/jenkins/jenkins.war"
BACKUP_WAR="/tmp/jenkins_backup.war"  # WAR file đã backup ở bước trước

if [[ ! -f "${BACKUP_WAR}" ]]; then
    echo "❌ Không tìm thấy backup WAR: ${BACKUP_WAR}"
    exit 1
fi

echo "=== Bắt Đầu Rollback Jenkins ==="

# Bước 1: Dừng Jenkins
systemctl stop jenkins

# Bước 2: Khôi phục WAR file cũ
cp "${BACKUP_WAR}" "${JENKINS_WAR}"
chown jenkins:jenkins "${JENKINS_WAR}"
echo "✅ WAR file đã rollback"

# Bước 3: Khởi động Jenkins
systemctl start jenkins
sleep 60

# Bước 4: Kiểm tra
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/login)
if [[ "${HTTP_CODE}" == "200" ]]; then
    echo "✅ Rollback thành công — Jenkins đang chạy"
else
    echo "❌ Jenkins vẫn không khởi động được — cần restore từ backup"
    echo "Chạy: ./jenkins-restore.sh /backup/jenkins/pre_upgrade_xxx/jenkins_pre_upgrade.tar.gz"
fi
```

### Rollback Docker (Kubernetes)

```bash
# Rollback Docker Compose
# Sửa lại docker-compose.yml về image cũ
docker compose down
# Sửa image tag về phiên bản cũ trong docker-compose.yml
docker compose up -d

# Rollback Kubernetes — đơn giản nhất
kubectl rollout undo deployment/jenkins -n jenkins

# Xem lịch sử deployment
kubectl rollout history deployment/jenkins -n jenkins

# Rollback về revision cụ thể
kubectl rollout undo deployment/jenkins -n jenkins --to-revision=3
```

---

## Phần 6: Plugin Update Strategy (Chiến Lược Cập Nhật Plugin)

### Không Nên Cập Nhật Plugin Cùng Lúc Với Nâng Cấp Jenkins

```
Quy tắc vàng:
  ❌ SAI: Nâng cấp Jenkins + Update tất cả plugin cùng lúc
       → Khi có lỗi, không biết nguyên nhân từ đâu

  ✅ ĐÚNG:
     Ngày 1: Nâng cấp Jenkins core (chỉ core, giữ plugin cũ)
     Ngày 2: Kiểm tra ổn định 24 giờ
     Ngày 3-7: Update từng nhóm plugin có liên quan
```

### Cập Nhật Plugin Theo Nhóm

```bash
# Nhóm 1: Security plugins (ưu tiên cao nhất, update trước)
java -jar jenkins-cli.jar -s http://localhost:8080 -auth admin:token \
  install-plugin credentials credentials-binding \
  script-security matrix-auth role-strategy -restart

# Đợi Jenkins restart và kiểm tra

# Nhóm 2: Core integration plugins
java -jar jenkins-cli.jar -s http://localhost:8080 -auth admin:token \
  install-plugin git workflow-aggregator pipeline-stage-view \
  blue-ocean -restart

# Nhóm 3: Build tool và utility plugins
java -jar jenkins-cli.jar -s http://localhost:8080 -auth admin:token \
  install-plugin maven-plugin gradle docker-workflow kubernetes -restart
```

### Khóa Phiên Bản Plugin (Plugin Pinning)

```bash
# Tạo file danh sách plugin cố định phiên bản
# plugins.txt — dùng với official Jenkins Docker image
git:4.13.0
pipeline-build-step:2.18
kubernetes:3904.v15f484497267
slack:2.50
sonar:2.15

# Dockerfile — cài đúng phiên bản plugin từ file
FROM jenkins/jenkins:2.452.1-lts
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
```

---

## Phần 7: Lịch Nâng Cấp Được Khuyến Nghị

```
Lịch nâng cấp production Jenkins (ví dụ):

  Thứ 3:    Đọc changelog dòng LTS mới
            Kiểm tra plugin compatibility
  Thứ 4:    Thực hiện nâng cấp trên staging
  Thứ 5:    Kiểm tra staging 24 giờ — chạy full test suite
  Thứ 6:    Quyết định go/no-go
  Thứ 7:    (Nếu go) Nâng cấp production trong maintenance window
            (02:00–06:00 — ít traffic nhất)

  Maintenance window (cửa sổ bảo trì):
    • Thông báo team trước 48 giờ
    • Có ít nhất 2 người trong ca nâng cấp
    • Rollback plan đã sẵn sàng và test được
    • Backup confirmed trong vòng 1 giờ trước khi nâng cấp
```

---

## Tóm Tắt Upgrade Checklist

```markdown
## Trước Khi Nâng Cấp
[ ] Đọc Upgrade Notes và Changelog
[ ] Liệt kê breaking changes ảnh hưởng đến Jenkins của mình
[ ] Kiểm tra plugin compatibility (từng plugin quan trọng)
[ ] Test trên staging với data từ production backup
[ ] Backup production JENKINS_HOME (< 1 giờ trước khi nâng cấp)
[ ] Thông báo maintenance window cho toàn team
[ ] Xác nhận rollback plan và rollback data có sẵn

## Trong Quá Trình Nâng Cấp
[ ] Đưa Jenkins vào quiet mode
[ ] Đợi build đang chạy hoàn thành
[ ] Thực hiện nâng cấp
[ ] Theo dõi log khi Jenkins khởi động
[ ] Chạy smoke test (kiểm tra nhanh) ngay sau khi up

## Sau Khi Nâng Cấp
[ ] Chạy đầy đủ checklist kiểm tra chức năng
[ ] Xác nhận monitoring và alerting hoạt động
[ ] Ghi lại phiên bản mới vào documentation
[ ] Thông báo hoàn thành cho team
[ ] Giữ backup WAR file cũ ít nhất 7 ngày
[ ] Update plugin theo lịch (không cùng ngày)
```

---

## Câu Hỏi Phỏng Vấn

1. **Sự khác biệt giữa Jenkins LTS và Weekly release? Môi trường nào nên dùng loại nào?**
   → LTS: ổn định, cập nhật chậm (~mỗi 12 tuần có dòng mới), chỉ nhận bug/security fix. Weekly: luôn mới nhất, có thể không ổn định. Production dùng LTS, lab/dev có thể dùng Weekly.

2. **Bạn nâng cấp Jenkins theo quy trình nào? Khi nào mới nâng cấp production?**
   → Luôn test trên staging trước với data production. Đọc changelog, kiểm tra plugin compat, backup, test staging 24 giờ, rồi mới lên production trong maintenance window. Không bao giờ nâng cấp core và plugin cùng lúc.

3. **Plugin A không tương thích với Jenkins phiên bản mới. Bạn xử lý thế nào?**
   → Kiểm tra xem plugin A có bản update mới hơn không. Nếu có: update plugin trước, rồi upgrade Jenkins. Nếu không: liên hệ nhà phát triển plugin, tìm alternative, hoặc trì hoãn upgrade Jenkins cho đến khi plugin được cập nhật.

4. **Jenkins không khởi động được sau nâng cấp. Các bước xử lý?**
   → Xem log Jenkins (`/var/log/jenkins/jenkins.log`) để tìm nguyên nhân. Nếu lỗi plugin: thử disable plugin gây lỗi. Nếu không giải quyết được trong 15-30 phút: rollback WAR file cũ ngay, thông báo team, phân tích nguyên nhân sau.

5. **Tại sao không nên nâng cấp Jenkins core và plugin cùng một lúc?**
   → Vì khi có lỗi, không thể xác định nguyên nhân là từ Jenkins core hay từ plugin nào. Tách ra hai bước giúp isolate (cô lập) vấn đề và đơn giản hóa rollback nếu cần.

6. **Plugin pinning (khóa phiên bản plugin) là gì? Khi nào nên dùng?**
   → Chỉ định phiên bản cụ thể cho từng plugin trong Dockerfile hoặc plugins.txt, thay vì luôn cài "latest". Nên dùng trong production để đảm bảo reproducibility (khả năng tái tạo) và tránh plugin bị tự động update gây lỗi.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Phiên Bản:** 1.0
