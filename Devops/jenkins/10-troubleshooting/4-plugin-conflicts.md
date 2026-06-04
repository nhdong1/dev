# Plugin Conflicts — Xử Lý Xung Đột Plugin

> Plugin (tiện ích mở rộng) là sức mạnh của Jenkins, nhưng cũng là nguồn gốc của nhiều sự cố phức tạp nhất: classloader hell (địa ngục classloader), dependency version mismatch (không tương thích phiên bản phụ thuộc), và ClassNotFoundException trong runtime.

## Mục Lục

1. [Hiểu Kiến Trúc Plugin Jenkins](#hiểu-kiến-trúc-plugin-jenkins)
2. [Classloader Hell — Địa Ngục Classloader](#classloader-hell--địa-ngục-classloader)
3. [Dependency Version Mismatch — Xung Đột Phiên Bản](#dependency-version-mismatch--xung-đột-phiên-bản)
4. [Plugin Gây Crash Jenkins](#plugin-gây-crash-jenkins)
5. [Cập Nhật Plugin An Toàn](#cập-nhật-plugin-an-toàn)
6. [Downgrade Plugin — Hạ Cấp Phiên Bản](#downgrade-plugin--hạ-cấp-phiên-bản)
7. [Chẩn Đoán Plugin Conflict Có Hệ Thống](#chẩn-đoán-plugin-conflict-có-hệ-thống)
8. [Best Practices Quản Lý Plugin](#best-practices-quản-lý-plugin)

---

## Hiểu Kiến Trúc Plugin Jenkins

### Plugin Isolation Model

Jenkins dùng kiến trúc **plugin classloading** đặc biệt. Mỗi plugin có classloader (bộ tải class) riêng, nhưng có thể phụ thuộc vào plugin khác.

```
Jenkins Core ClassLoader (ClassLoader gốc)
    │
    ├── Plugin A ClassLoader
    │       └── depends on: commons-lang 2.6
    │
    ├── Plugin B ClassLoader
    │       └── depends on: commons-lang 3.12
    │
    └── Plugin C ClassLoader
            └── depends on: Plugin A, Plugin B
            → ClassLoader phải resolve commons-lang nào?
```

### File Cấu Trúc Plugin

```
JENKINS_HOME/plugins/
├── git.jpi                    ← Plugin file (Jenkins Plugin Interface)
├── git/                       ← Plugin đã được extract
│   ├── WEB-INF/lib/
│   │   ├── git-client.jar
│   │   └── jgit.jar           ← Bundled dependencies (phụ thuộc đóng gói kèm)
│   └── META-INF/MANIFEST.MF   ← Metadata: version, dependencies
└── git.jpi.pinned             ← File này ngăn Jenkins auto-update plugin
```

---

## Classloader Hell — Địa Ngục Classloader

### Triệu Chứng Điển Hình

```
java.lang.ClassNotFoundException: org.apache.commons.lang3.StringUtils
    at hudson.plugins.somePlugin.SomeClass.doSomething(SomeClass.java:42)

# hoặc
java.lang.NoClassDefFoundError: com/google/gson/Gson
    at jenkins.plugins.anotherPlugin.AnotherClass.<init>

# hoặc — phức tạp hơn
java.lang.LinkageError: loader constraint violation:
    when resolving method "org.yaml.snakeyaml.Yaml.<init>()"
    the class loader (instance of PluginClassLoader for plugin X)
    of the resolved class differs from ...
```

### Nguyên Nhân

1. **Plugin A** bundle (đóng gói kèm) `library-v1.0` trong JAR của nó
2. **Plugin B** bundle `library-v2.0` — cùng class nhưng khác phiên bản
3. Khi **Plugin C** dùng cả A và B, JVM không biết dùng phiên bản nào
4. → `LinkageError` hoặc `ClassCastException` xảy ra

### Chẩn Đoán

```groovy
// Manage Jenkins → Script Console
// Tìm xem class nào được load bởi classloader nào
def className = "org.yaml.snakeyaml.Yaml"
try {
    def clazz = Class.forName(className)
    println "Found in classloader: ${clazz.classLoader}"
} catch (ClassNotFoundException e) {
    println "Class not found: ${className}"
}

// Xem tất cả versions của một library trong classpath
Jenkins.instance.pluginManager.plugins.each { plugin ->
    plugin.classLoader?.URLs?.each { url ->
        if (url.path.contains("snakeyaml")) {
            println "${plugin.shortName} → ${url}"
        }
    }
}
```

### Giải Pháp

1. **Nâng cấp plugin lên phiên bản mới nhất** — thường tác giả đã fix
2. **Kiểm tra plugin compatibility matrix** (ma trận tương thích) trên plugins.jenkins.io
3. **Loại bỏ một trong hai plugin** xung đột nếu không cần thiết
4. **Nâng cấp Jenkins Core** — phiên bản mới hơn thường có better classloading

---

## Dependency Version Mismatch — Xung Đột Phiên Bản

### Plugin Dependency Declaration

Mỗi plugin khai báo dependencies (phụ thuộc) trong `MANIFEST.MF`:

```
Plugin-Dependencies: git:4.11.0,
                     credentials:2.6.2,
                     workflow-step-api:2.24
```

Jenkins enforce (bắt buộc) các dependency này — plugin sẽ không load nếu dependencies không thỏa mãn.

### Lỗi Phổ Biến

#### 1. Required Plugin Version Too Old

```
ERROR: Plugin X requires Plugin Y version 2.0 or higher,
       but you have Plugin Y version 1.8 installed.
```

**Giải pháp:** Cập nhật Plugin Y lên phiên bản ≥ 2.0.

#### 2. Plugin Phụ Thuộc Plugin Chưa Cài

```
Failed to load: Plugin X (missing: plugin Y, plugin Z)
```

**Kiểm tra:**
```
Manage Jenkins → Plugin Manager → Installed
→ Tìm kiếm plugin Y, Z
```

**Tự động cài dependencies:**
```
Manage Jenkins → Plugin Manager → Available
→ Cài Plugin X → Jenkins tự cài Y, Z kèm theo
```

#### 3. Vòng Phụ Thuộc (Circular Dependency)

Hiếm gặp, nhưng có thể xảy ra khi plugin fork lẫn nhau. Xem `jenkins.log` để tìm thông báo circular dependency.

### Xem Dependency Graph

```groovy
// Script Console — xem dependency tree của một plugin
def targetPlugin = "git"
def plugin = Jenkins.instance.pluginManager.getPlugin(targetPlugin)

println "Plugin: ${plugin.shortName} v${plugin.version}"
println "Dependencies:"
plugin.getDependencies().each { dep ->
    def depPlugin = Jenkins.instance.pluginManager.getPlugin(dep.shortName)
    def installedVersion = depPlugin?.version ?: "NOT INSTALLED"
    println "  ${dep.shortName} (required: ${dep.version}, installed: ${installedVersion})"
}
```

---

## Plugin Gây Crash Jenkins

### Jenkins Không Start Được Sau Khi Cài/Cập Nhật Plugin

**Triệu chứng:**
- Jenkins start nhưng hiện "Please wait while Jenkins is getting ready to work"
- Hoặc Jenkins throw exception ngay khi start
- Log có `SEVERE` level errors

**Chẩn đoán từ log:**

```bash
# Xem jenkins.log ngay sau khi start
tail -100 /var/log/jenkins/jenkins.log | grep -E "SEVERE|ERROR|Exception"

# Tìm plugin nào fail khi load
grep "Failed to load" /var/log/jenkins/jenkins.log

# Tìm plugin nào throw exception
grep "PluginWrapper" /var/log/jenkins/jenkins.log | grep -i "exception\|error"
```

**Xử lý khẩn cấp — Disable plugin qua filesystem:**

```bash
# Dừng Jenkins
systemctl stop jenkins

# Tìm plugin mới cài gần đây
ls -lt /var/lib/jenkins/plugins/*.jpi | head -5

# Vô hiệu hóa plugin bằng cách rename (không xóa để rollback được)
cd /var/lib/jenkins/plugins
mv problematic-plugin.jpi problematic-plugin.jpi.disabled

# Xóa folder extracted (Jenkins sẽ extract lại nếu re-enable)
rm -rf problematic-plugin/

# Khởi động lại Jenkins
systemctl start jenkins
```

### Safe Mode — Khởi Động Không Load Plugin

```bash
# Khởi động Jenkins ở Safe Mode (tắt tất cả plugin)
# Chỉ dùng trong trường hợp khẩn cấp để truy cập UI
java -jar jenkins.war --safe-restart
```

Trong Safe Mode, Jenkins chạy nhưng không load bất kỳ plugin nào. Dùng để:
- Truy cập Plugin Manager và vô hiệu hóa plugin lỗi
- Kiểm tra xem lỗi có do plugin gây ra không

---

## Cập Nhật Plugin An Toàn

### Nguyên Tắc Cập Nhật An Toàn

```
Backup → Test trên staging → Cập nhật từng plugin → Verify → Production
```

### Quy Trình Cập Nhật Từng Bước

**Bước 1: Backup trước khi cập nhật**

```bash
# Backup toàn bộ JENKINS_HOME
tar -czf jenkins-backup-$(date +%Y%m%d).tar.gz /var/lib/jenkins/

# Hoặc chỉ backup plugins
tar -czf plugins-backup-$(date +%Y%m%d).tar.gz /var/lib/jenkins/plugins/
```

**Bước 2: Xem changelog trước khi cập nhật**

```
Manage Jenkins → Plugin Manager → Updates
→ Click tên plugin → Xem Changelog
→ Tìm "Breaking changes" hoặc "Incompatible changes"
```

**Bước 3: Cập nhật từng nhóm nhỏ, không all-at-once**

```
Thay vì: Check All → Download → Restart (nguy hiểm)

Nên: Chọn 2-3 plugin liên quan → Download → Restart → Test → tiếp tục
```

**Bước 4: Verify sau cập nhật**

```groovy
// Chạy smoke test (kiểm tra nhanh) sau khi cập nhật plugin
// Trigger một pipeline đơn giản và kiểm tra output
pipeline {
    agent any
    stages {
        stage('Smoke Test') {
            steps {
                sh 'echo "Jenkins plugins working correctly"'
                sh 'git --version'
                sh 'docker --version'
            }
        }
    }
}
```

### Script Cập Nhật Plugin Tự Động (CLI)

```bash
# Dùng Jenkins CLI — cập nhật tất cả plugin
java -jar jenkins-cli.jar \
    -s http://jenkins-controller:8080 \
    -auth admin:api-token \
    install-plugin \
    $(java -jar jenkins-cli.jar -s http://jenkins-controller:8080 -auth admin:api-token list-plugins \
      | awk '$2 != $3 {print $1}' | tr '\n' ' ')

# Restart sau khi cập nhật
java -jar jenkins-cli.jar \
    -s http://jenkins-controller:8080 \
    -auth admin:api-token \
    safe-restart
```

---

## Downgrade Plugin — Hạ Cấp Phiên Bản

### Khi Nào Cần Downgrade

- Cập nhật plugin gây ra regression (lỗi mới)
- Plugin mới không tương thích với Jenkins Core version hiện tại
- Plugin mới có bug ảnh hưởng đến production

### Downgrade Thủ Công

```bash
# Bước 1: Dừng Jenkins
systemctl stop jenkins

# Bước 2: Backup phiên bản hiện tại
cp /var/lib/jenkins/plugins/git.jpi /var/lib/jenkins/plugins/git.jpi.bak

# Bước 3: Tải phiên bản cũ từ Jenkins update center archive
# URL format: https://updates.jenkins.io/download/plugins/<name>/<version>/<name>.hpi
wget -O /var/lib/jenkins/plugins/git.jpi \
    https://updates.jenkins.io/download/plugins/git/4.10.0/git.hpi

# Bước 4: Xóa folder extracted để buộc Jenkins extract lại
rm -rf /var/lib/jenkins/plugins/git/

# Bước 5: Tạo .pinned file để ngăn Jenkins auto-update lại
touch /var/lib/jenkins/plugins/git.jpi.pinned

# Bước 6: Khởi động Jenkins
systemctl start jenkins
```

### Quản Lý .pinned Files

```bash
# Xem tất cả plugin đang được pinned (cố định phiên bản)
ls /var/lib/jenkins/plugins/*.pinned

# Bỏ pin — cho phép auto-update lại
rm /var/lib/jenkins/plugins/git.jpi.pinned
```

---

## Chẩn Đoán Plugin Conflict Có Hệ Thống

### Quy Trình 5 Bước

```
Bước 1: Thu thập thông tin
├── Jenkins version
├── Plugin list + versions (export từ Plugin Manager)
└── Full stack trace từ jenkins.log

Bước 2: Xác định plugin gây lỗi
├── Đọc stack trace — tìm package name của plugin
├── Tìm "at com.company.pluginname" trong trace
└── Tìm plugin cài gần đây nhất trùng thời gian lỗi

Bước 3: Kiểm tra compatibility
├── Vào plugins.jenkins.io → tìm plugin
├── Xem "Requires Jenkins" version
└── Xem "Requires Plugins" và version

Bước 4: Thử giải pháp
├── Cập nhật plugin lên latest
├── Downgrade về version trước
└── Vô hiệu hóa plugin và test

Bước 5: Báo cáo và phòng ngừa
├── Tạo issue trên Jenkins JIRA
└── Thêm vào change management log
```

### Script Thu Thập Thông Tin Nhanh

```groovy
// Manage Jenkins → Script Console
// Export toàn bộ plugin list ra dạng dễ đọc
println "=== Jenkins Version ==="
println Jenkins.instance.version

println "\n=== Installed Plugins ==="
Jenkins.instance.pluginManager.plugins
    .sort { it.shortName }
    .each { plugin ->
        def status = plugin.isActive() ? "ACTIVE" : "DISABLED"
        println "${status} | ${plugin.shortName} | ${plugin.version} | ${plugin.displayName}"
    }

println "\n=== Failed Plugins ==="
Jenkins.instance.pluginManager.failedPlugins.each { entry ->
    println "FAILED: ${entry.key} — ${entry.value}"
}
```

---

## Best Practices Quản Lý Plugin

### 1. Dùng Plugin BOM (Bill of Materials — Danh Sách Vật Liệu)

**Jenkins Plugin BOM** là một bộ plugin versions được test tương thích với nhau:

```xml
<!-- Nếu dùng JCasC + Maven để quản lý Jenkins as Code -->
<dependency>
    <groupId>io.jenkins.tools.bom</groupId>
    <artifactId>bom-2.387.x</artifactId>
    <version>2102.v854b_fec19c92</version>
    <scope>import</scope>
    <type>pom</type>
</dependency>
```

### 2. Quản Lý Plugin Qua JCasC (Jenkins Configuration as Code — Cấu Hình Jenkins Dưới Dạng Code)

```yaml
# plugins.yaml — danh sách plugin được quản lý như code
plugins:
  - id: git
    version: "4.11.0"
  - id: pipeline
    version: "2.7"
  - id: credentials
    version: "2.6.2"
  - id: kubernetes
    version: "3814.v6b_7f45a_49d87"
```

```bash
# Cài plugins từ file
jenkins-plugin-cli --plugin-file plugins.yaml
```

### 3. Giới Hạn Số Lượng Plugin

**Quy tắc thực tế:**
- Chỉ cài plugin thực sự cần
- Đánh giá plugin: download count, last update date, open issues
- Loại bỏ plugin không dùng định kỳ (quarterly review)

```groovy
// Tìm plugin không được dùng bởi bất kỳ job nào
// (cần xem xét thủ công — không tự xóa)
def usedPlugins = [] as Set

Jenkins.instance.getAllItems(Job.class).each { job ->
    // Thêm logic kiểm tra plugin usage tùy theo job type
}

Jenkins.instance.pluginManager.plugins.each { plugin ->
    if (!usedPlugins.contains(plugin.shortName)) {
        println "Possibly unused: ${plugin.shortName} v${plugin.version}"
    }
}
```

### 4. Plugin Update Policy

```markdown
## Chính Sách Cập Nhật Plugin (đề xuất)

- **Security updates:** Cập nhật ngay trong 24h sau khi có thông báo
- **Major versions (X.0.0):** Test trên staging 1 tuần trước khi production
- **Minor versions (x.Y.0):** Test trên staging 2-3 ngày
- **Patch versions (x.y.Z):** Có thể cập nhật trực tiếp nếu changelog không có breaking changes
- **Review hàng tháng:** Xem danh sách plugin outdated và lên kế hoạch cập nhật
```

---

**Xem Tiếp:** [5-production-checklist.md](5-production-checklist.md) — checklist trước khi đưa Jenkins lên production
