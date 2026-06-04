# 3 — Credentials: Quản Lý Thông Tin Xác Thực và Secret

> Credentials (thông tin xác thực) trong Jenkins là cơ chế lưu trữ secret (bí mật) an toàn — API token, SSH key, mật khẩu, chứng chỉ — và cung cấp chúng cho pipeline mà không để lộ giá trị thuần ra ngoài. Quản lý đúng credentials là kỹ năng cốt lõi để đảm bảo bảo mật CI/CD.

## Mục Lục

1. [Tổng Quan Credentials System](#1-tổng-quan-credentials-system)
2. [Các Loại Credentials](#2-các-loại-credentials)
3. [Credentials Scope (Phạm Vi)](#3-credentials-scope-phạm-vi)
4. [Tạo và Quản Lý Credentials](#4-tạo-và-quản-lý-credentials)
5. [Dùng Credentials Trong Pipeline](#5-dùng-credentials-trong-pipeline)
6. [SSH Credentials Chi Tiết](#6-ssh-credentials-chi-tiết)
7. [HashiCorp Vault Integration](#7-hashicorp-vault-integration)
8. [Credentials Best Practices](#8-credentials-best-practices)
9. [Troubleshooting Credentials](#9-troubleshooting-credentials)

---

## 1. Tổng Quan Credentials System

### Kiến Trúc Credentials

```
┌─────────────────────────────────────────────────────────────┐
│                  JENKINS CREDENTIALS SYSTEM                 │
│                                                             │
│  Credentials Store (Kho Lưu Trữ)                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Jenkins (System) Store                             │   │
│  │  ├── System scope:  Agent, daemon process dùng      │   │
│  │  └── Global scope:  Mọi job có thể dùng             │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Folder Store (nếu dùng Folders Plugin)             │   │
│  │  └── Chỉ job trong folder đó mới thấy được          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Credentials Provider (Nhà Cung Cấp)                       │
│  ├── Jenkins Credentials Plugin (built-in)                  │
│  ├── HashiCorp Vault Plugin                                 │
│  ├── AWS Secrets Manager Provider                           │
│  └── Azure Key Vault Provider                               │
│                                                             │
│  Encryption (Mã Hóa)                                        │
│  ├── Master encryption key: JENKINS_HOME/secrets/master.key │
│  └── Encrypted values: JENKINS_HOME/credentials.xml         │
└─────────────────────────────────────────────────────────────┘
```

### Tại Sao Không Hardcode Secret?

```groovy
// ❌ SAI — hardcode secret trong Jenkinsfile
pipeline {
    stages {
        stage('Deploy') {
            steps {
                sh "docker login -u myuser -p MySuperSecret123 registry.example.com"
                // Secret hiển thị trong build log, git history, console output
            }
        }
    }
}

// ✅ ĐÚNG — dùng Credentials
pipeline {
    stages {
        stage('Deploy') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "docker login -u $DOCKER_USER -p $DOCKER_PASS registry.example.com"
                    // Password bị mask: docker login -u myuser -p ****
                }
            }
        }
    }
}
```

---

## 2. Các Loại Credentials

### 2.1 Secret Text (Văn Bản Bí Mật)

Dùng cho: API token, Bearer token, webhook secret, bất kỳ chuỗi đơn nào.

```
Kind: Secret text
Scope: Global
ID:   sonarqube-api-token
Description: SonarQube API Token for code analysis
Secret: squ_abc123def456...
```

```groovy
// Dùng trong pipeline
withCredentials([string(credentialsId: 'sonarqube-api-token', variable: 'SONAR_TOKEN')]) {
    sh "sonar-scanner -Dsonar.login=$SONAR_TOKEN"
}
```

### 2.2 Username with Password (Tên Người Dùng và Mật Khẩu)

Dùng cho: Docker registry, Maven repository, database credentials, bất kỳ cặp username/password nào.

```
Kind: Username with password
Scope: Global
ID:   nexus-credentials
Description: Nexus Repository Manager credentials
Username: jenkins-ci
Password: ChangeMe123!
```

```groovy
// Dùng trong pipeline
withCredentials([usernamePassword(
    credentialsId: 'nexus-credentials',
    usernameVariable: 'NEXUS_USER',
    passwordVariable: 'NEXUS_PASS'
)]) {
    sh """
        mvn deploy \
            -Drepository.username=$NEXUS_USER \
            -Drepository.password=$NEXUS_PASS
    """
}
```

### 2.3 SSH Username with Private Key (SSH với Khóa Riêng Tư)

Dùng cho: Git clone qua SSH, deploy qua SSH, kết nối đến server.

```
Kind: SSH Username with private key
Scope: Global
ID:   git-deploy-key
Description: SSH deploy key for GitHub repositories
Username: git
Private Key: Enter directly
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAA...
  -----END OPENSSH PRIVATE KEY-----
Passphrase: [để trống hoặc điền nếu key có passphrase]
```

```groovy
// Git checkout tự động dùng SSH credential
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git(
                    url: 'git@github.com:myorg/myrepo.git',
                    credentialsId: 'git-deploy-key',
                    branch: 'main'
                )
            }
        }
    }
}

// SSH vào server để deploy
withCredentials([sshUserPrivateKey(
    credentialsId: 'production-server-key',
    keyFileVariable: 'SSH_KEY',
    usernameVariable: 'SSH_USER'
)]) {
    sh """
        chmod 600 $SSH_KEY
        ssh -i $SSH_KEY -o StrictHostKeyChecking=no \
            $SSH_USER@prod.example.com \
            'cd /app && git pull && systemctl restart myapp'
    """
}
```

### 2.4 Certificate (Chứng Chỉ)

Dùng cho: Mutual TLS (mTLS — xác thực hai chiều qua TLS), PKCS#12 keystore, client certificate authentication.

```
Kind: Certificate
Scope: Global
ID:   k8s-client-cert
Description: Kubernetes client certificate for kubectl
Certificate: Upload PKCS#12 file (.p12 / .pfx)
Password: [password của keystore]
```

```groovy
// Dùng certificate để xác thực với Kubernetes API
withCredentials([certificate(
    credentialsId: 'k8s-client-cert',
    keystoreVariable: 'K8S_KEYSTORE',
    passwordVariable: 'K8S_KEYSTORE_PASS'
)]) {
    sh """
        kubectl --client-certificate=$K8S_KEYSTORE \
                --certificate-authority=/etc/k8s/ca.crt \
                get pods -n production
    """
}
```

### 2.5 Secret File (File Bí Mật)

Dùng cho: kubeconfig file, service account JSON (Google Cloud), bất kỳ file cấu hình chứa secret nào.

```
Kind: Secret file
Scope: Global
ID:   gcp-service-account-key
Description: GCP Service Account JSON key
File: Upload file JSON
```

```groovy
// Dùng secret file
withCredentials([file(
    credentialsId: 'gcp-service-account-key',
    variable: 'GCP_KEY_FILE'
)]) {
    sh """
        gcloud auth activate-service-account --key-file=$GCP_KEY_FILE
        gcloud container clusters get-credentials my-cluster --region asia-southeast1
        kubectl apply -f k8s/
    """
}
```

### 2.6 GitHub App Credentials

Dùng cho: Xác thực Jenkins với GitHub App (thay vì Personal Access Token).

```
Kind: GitHub App
Scope: Global
ID:   github-app-credentials
App ID: 123456
Private Key: -----BEGIN RSA PRIVATE KEY-----...
```

### Tóm Tắt Loại Credentials

| Loại                      | Dùng Cho                                    | Variable Pattern                         |
| ------------------------- | ------------------------------------------- | ---------------------------------------- |
| Secret text               | API token, webhook secret                   | `string(variable: 'VAR')`               |
| Username/Password         | Registry, Maven, DB credentials             | `usernamePassword(usernameVariable, passwordVariable)` |
| SSH Username + Private Key| Git SSH, server SSH access                  | `sshUserPrivateKey(keyFileVariable, usernameVariable)` |
| Certificate               | mTLS, PKCS#12 keystore                      | `certificate(keystoreVariable, passwordVariable)` |
| Secret file               | kubeconfig, GCP service account JSON        | `file(variable: 'VAR')`                 |

---

## 3. Credentials Scope (Phạm Vi)

**Scope** (phạm vi) xác định ai có thể thấy và dùng credential này.

### System Scope (Phạm Vi Hệ Thống)

- Chỉ dành cho Jenkins internal processes: agent connection, JNLP, system-level operations
- **Không hiển thị** trong dropdown của pipeline/job configuration
- Ví dụ: SSH key để Jenkins master kết nối đến agent

```
Scope: System
→ Chỉ Jenkins nội bộ dùng được
→ Pipeline code KHÔNG thể truy cập
```

### Global Scope (Phạm Vi Toàn Cục)

- Mọi job trên Jenkins đều có thể dùng (nếu có quyền `Credentials/Use`)
- Lưu trong Jenkins store cấp hệ thống
- Dùng cho credentials dùng chung toàn tổ chức

```
Scope: Global
→ Tất cả pipeline trên Jenkins đều có thể dùng
→ Cẩn thận: credentials nhạy cảm cao nên để Folder scope
```

### Folder Scope (Phạm Vi Thư Mục)

- Chỉ job trong folder cụ thể mới thấy và dùng được
- **Best practice cho production**: mỗi team/project có folder riêng với credentials riêng
- Cách ly hoàn toàn: team payments không thể vô tình dùng credentials của team inventory

```
Folder: payments/
  Credentials:
  ├── payments-db-password    (chỉ job trong payments/ dùng được)
  ├── payments-api-key
  └── stripe-secret-key

Folder: infra/
  Credentials:
  ├── aws-terraform-key       (chỉ job trong infra/ dùng được)
  └── k8s-deploy-kubeconfig
```

---

## 4. Tạo và Quản Lý Credentials

### Tạo Credentials qua UI

```
Manage Jenkins → Credentials → System → Global credentials (unrestricted)
→ Add Credentials
→ Chọn Kind, điền thông tin
→ OK
```

### Tạo Credentials qua JCasC

```yaml
credentials:
  system:
    domainCredentials:
      - credentials:
          # Secret text
          - string:
              scope: GLOBAL
              id: "sonarqube-token"
              description: "SonarQube API Token"
              secret: "${SONARQUBE_TOKEN}"   # Đọc từ environment variable

          # Username/Password
          - usernamePassword:
              scope: GLOBAL
              id: "nexus-credentials"
              description: "Nexus Repository credentials"
              username: "jenkins-ci"
              password: "${NEXUS_PASSWORD}"

          # SSH Key
          - basicSSHUserPrivateKey:
              scope: GLOBAL
              id: "git-deploy-key"
              description: "GitHub deploy key"
              username: "git"
              privateKeySource:
                directEntry:
                  privateKey: "${GIT_DEPLOY_PRIVATE_KEY}"

          # Secret file (base64 encoded)
          - file:
              scope: GLOBAL
              id: "kubeconfig-production"
              description: "Production cluster kubeconfig"
              fileName: "kubeconfig"
              secretBytes: "${KUBECONFIG_BASE64}"
```

### Tạo Credentials qua Jenkins CLI

```bash
# Tạo secret text credential
cat > /tmp/cred.xml << 'EOF'
<com.cloudbees.plugins.credentials.impl.StringCredentialsImpl>
  <scope>GLOBAL</scope>
  <id>my-api-token</id>
  <description>API Token for external service</description>
  <secret>my-actual-secret-value</secret>
</com.cloudbees.plugins.credentials.impl.StringCredentialsImpl>
EOF

java -jar jenkins-cli.jar -s http://jenkins.example.com \
  -auth admin:apitoken \
  create-credentials-by-xml system::system::jenkins _ < /tmp/cred.xml

# Xóa file ngay sau khi dùng
rm /tmp/cred.xml
```

### Rotate (Luân Phiên Thay) Credentials

```bash
# Update credential qua CLI
cat > /tmp/updated-cred.xml << 'EOF'
<com.cloudbees.plugins.credentials.impl.StringCredentialsImpl>
  <scope>GLOBAL</scope>
  <id>sonarqube-token</id>
  <description>SonarQube API Token — rotated 2026-05-11</description>
  <secret>new-token-value-abc123</secret>
</com.cloudbees.plugins.credentials.impl.StringCredentialsImpl>
EOF

java -jar jenkins-cli.jar -s http://jenkins.example.com \
  -auth admin:apitoken \
  update-credentials-by-xml system::system::jenkins _ sonarqube-token \
  < /tmp/updated-cred.xml

rm /tmp/updated-cred.xml
```

---

## 5. Dùng Credentials Trong Pipeline

### withCredentials Block

`withCredentials` là cách chính thức và an toàn nhất để truy cập credentials trong pipeline:

```groovy
pipeline {
    agent any
    stages {
        stage('Multi-credential example') {
            steps {
                // Dùng nhiều credentials cùng lúc
                withCredentials([
                    string(credentialsId: 'slack-token', variable: 'SLACK_TOKEN'),
                    usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    ),
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')
                ]) {
                    sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    sh "docker build -t myapp:${BUILD_NUMBER} ."
                    sh "docker push myapp:${BUILD_NUMBER}"
                    sh "kubectl --kubeconfig=$KUBECONFIG apply -f k8s/"
                    sh "curl -X POST -H 'Authorization: Bearer $SLACK_TOKEN' ..."
                }
                // Ngoài withCredentials block: biến tự động bị xóa
            }
        }
    }
}
```

### Credentials Trong Environment Block

```groovy
pipeline {
    agent any
    environment {
        // Credentials được inject vào environment variable
        // Cách này expose secret vào env của toàn stage — dùng cẩn thận
        SONAR_TOKEN = credentials('sonarqube-token')

        // Với username/password: tự động tạo 3 biến:
        // NEXUS_CREDS       = username:password
        // NEXUS_CREDS_USR   = username
        // NEXUS_CREDS_PSW   = password
        NEXUS_CREDS = credentials('nexus-credentials')
    }
    stages {
        stage('Analyze') {
            steps {
                sh "sonar-scanner -Dsonar.login=${SONAR_TOKEN}"
            }
        }
        stage('Publish') {
            steps {
                sh "mvn deploy -Dusername=${NEXUS_CREDS_USR} -Dpassword=${NEXUS_CREDS_PSW}"
            }
        }
    }
}
```

### Automatic Secret Masking (Che Giấu Tự Động)

Jenkins tự động mask (che giấu) giá trị credentials trong build log:

```
# Log thực tế thấy:
[Pipeline] sh
+ docker login -u myuser -p ****
Login Succeeded

# Nhưng nếu split chuỗi, Jenkins có thể không mask được:
sh "echo ${DOCKER_PASS} | base64"  # ❌ Base64 của password vẫn hiển thị!
```

### Cảnh Báo — Khi Masking Có Thể Thất Bại

```groovy
// ❌ NGUY HIỂM — Jenkins không mask được trong một số trường hợp:
withCredentials([string(credentialsId: 'my-token', variable: 'TOKEN')]) {
    // 1. Ghi ra file rồi đọc lại
    sh "echo $TOKEN > /tmp/token.txt"
    sh "cat /tmp/token.txt"    // Có thể leak!

    // 2. Truyền qua argument vào script
    sh "./deploy.sh $TOKEN"    // Script có thể log lại

    // 3. Split ký tự
    def chars = TOKEN.toList()  // Groovy — từng ký tự riêng không bị mask

    // ✅ AN TOÀN hơn:
    sh '''
        # Dùng biến shell, không in ra console
        TOKEN_VALUE="$TOKEN"
        curl -H "Authorization: Bearer $TOKEN_VALUE" https://api.example.com
    '''
}
```

---

## 6. SSH Credentials Chi Tiết

### Tạo SSH Key Pair

```bash
# Tạo SSH key pair mới cho Jenkins
ssh-keygen -t ed25519 -C "jenkins-ci@example.com" -f jenkins-deploy-key -N ""

# Kết quả:
# jenkins-deploy-key       → private key (lưu vào Jenkins Credentials)
# jenkins-deploy-key.pub   → public key  (thêm vào GitHub/GitLab Deploy Keys)

# Nội dung private key để lưu vào Jenkins:
cat jenkins-deploy-key
# -----BEGIN OPENSSH PRIVATE KEY-----
# b3BlbnNzaC1rZXktdjEAAAAA...
# -----END OPENSSH PRIVATE KEY-----

# Xóa key local sau khi lưu vào Jenkins
shred -u jenkins-deploy-key jenkins-deploy-key.pub
```

### Thêm Public Key vào GitHub Deploy Keys

```
Repository → Settings → Deploy keys → Add deploy key

Title: jenkins-ci
Key: [nội dung jenkins-deploy-key.pub]
☑ Allow write access  (chỉ tick nếu Jenkins cần push)
```

### Known Hosts — Tránh Host Verification Prompt

```groovy
// Cấu hình SSH để tắt strict host checking trong Jenkins (KHÔNG làm trên production)
// Thay vào đó, thêm known hosts vào Jenkins global config:

// Manage Jenkins → Security → Git Host Key Verification Configuration
// → Accept first connection (chỉ dùng cho lab)
// Hoặc:
// → Manually provided keys (production) — nhập fingerprint của GitHub/GitLab

// Trong pipeline nếu cần tắt tạm:
sh "GIT_SSH_COMMAND='ssh -o StrictHostKeyChecking=no' git clone git@github.com:org/repo.git"
// ❌ Không khuyến nghị cho production — dễ bị MITM attack
```

---

## 7. HashiCorp Vault Integration

**HashiCorp Vault** (Kho Bí Mật HashiCorp) là giải pháp quản lý secret tập trung cho enterprise — cung cấp dynamic credentials (thông tin xác thực động), audit trail (nhật ký kiểm tra), và lease management (quản lý thời hạn sử dụng).

### Plugin Cần Thiết

```
HashiCorp Vault Plugin (hashicorp-vault-plugin)
```

### Kiến Trúc

```
Jenkins Pipeline
      ↓ Yêu cầu secret với Vault token/AppRole
HashiCorp Vault
      ↓ Xác thực và trả về secret
      ├── Static secrets:  kv/payments/database-password
      ├── Dynamic secrets: database/creds/my-role (tự tạo/hủy user DB)
      └── PKI certificates: pki/issue/web-cert
      ↓
Pipeline nhận secret, dùng trong build
      ↓ Secret hết hạn (lease expired)
Vault tự động revoke (thu hồi) quyền truy cập DB
```

### Cấu Hình Vault Plugin

```
Manage Jenkins → Configure System → HashiCorp Vault Plugin

Vault URL:    https://vault.example.com:8200
Vault Credential:
  - Vault Token:  Token credential ID từ Jenkins store
  - hoặc AppRole: Role ID + Secret ID

Skip SSL Verification: ☐ (không skip trong production)
```

### AppRole Authentication (Xác Thực AppRole — Khuyến Nghị)

```bash
# Trên Vault server — tạo AppRole cho Jenkins
vault auth enable approle

vault policy write jenkins-policy - << 'EOF'
path "secret/data/jenkins/*" {
  capabilities = ["read"]
}
path "database/creds/jenkins-role" {
  capabilities = ["read"]
}
EOF

vault write auth/approle/role/jenkins-role \
    token_policies="jenkins-policy" \
    token_ttl=1h \
    token_max_ttl=4h

# Lấy Role ID (không nhạy cảm, có thể lưu công khai)
vault read auth/approle/role/jenkins-role/role-id

# Tạo Secret ID (nhạy cảm — lưu vào Jenkins Credentials)
vault write -f auth/approle/role/jenkins-role/secret-id
```

### Dùng Vault Trong Pipeline

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy với Vault Dynamic DB Credentials') {
            steps {
                // Lấy dynamic database credentials từ Vault
                withVault(
                    configuration: [
                        vaultUrl: 'https://vault.example.com:8200',
                        vaultCredentialId: 'vault-approle-jenkins',
                        engineVersion: 2
                    ],
                    vaultSecrets: [
                        [
                            path: 'secret/data/payments/config',
                            secretValues: [
                                [vaultKey: 'api_key', envVar: 'PAYMENTS_API_KEY'],
                                [vaultKey: 'webhook_secret', envVar: 'WEBHOOK_SECRET']
                            ]
                        ],
                        [
                            path: 'database/creds/payments-role',
                            secretValues: [
                                [vaultKey: 'username', envVar: 'DB_USER'],
                                [vaultKey: 'password', envVar: 'DB_PASS']
                            ]
                        ]
                    ]
                ) {
                    sh """
                        # DB credentials có TTL ngắn — tự hết hạn sau build
                        psql "postgresql://$DB_USER:$DB_PASS@db.example.com/payments" \
                            -c "SELECT 1"
                        
                        # API key từ Vault KV store
                        curl -H "X-API-Key: $PAYMENTS_API_KEY" \
                             https://payments-api.example.com/health
                    """
                }
                // Sau block: Vault tự revoke dynamic credentials
            }
        }
    }
}
```

### So Sánh Jenkins Credentials Store vs Vault

| Tiêu Chí                          | Jenkins Store              | HashiCorp Vault                    |
| --------------------------------- | -------------------------- | ---------------------------------- |
| **Dynamic credentials**           | Không                      | Có — tự tạo/hủy user DB, cert      |
| **Audit trail** (nhật ký)        | Hạn chế                    | Chi tiết — ai truy cập gì, khi nào |
| **Rotation tự động**              | Không                      | Có — lease và renewal              |
| **Multi-platform**                | Chỉ Jenkins                | Tất cả platform                    |
| **Độ phức tạp cài đặt**          | Thấp                       | Cao                                |
| **Phù hợp với**                   | Team nhỏ, ít secret        | Enterprise, nhiều service          |

---

## 8. Credentials Best Practices

### Nguyên Tắc Cơ Bản

**1. Không bao giờ hardcode secret (ghi cứng bí mật)**

```groovy
// ❌ Tuyệt đối không làm:
sh "curl -H 'Authorization: Bearer eyJhbGc...' https://api.example.com"
sh "kubectl create secret generic my-secret --from-literal=password=Abc123"

// ✅ Luôn dùng credentials():
withCredentials([string(credentialsId: 'api-token', variable: 'TOKEN')]) {
    sh "curl -H 'Authorization: Bearer $TOKEN' https://api.example.com"
}
```

**2. Dùng Folder Scope để cô lập credentials**

```
Cấu trúc khuyến nghị:
Jenkins/
├── payments/               ← Folder
│   ├── Credentials:        ← Chỉ payments team thấy
│   │   ├── stripe-api-key
│   │   ├── payments-db-pass
│   │   └── payment-service-account
│   └── [các payment jobs]
├── infra/                  ← Folder riêng biệt
│   ├── Credentials:        ← Chỉ infra team thấy
│   │   ├── aws-terraform-key
│   │   ├── k8s-deploy-token
│   │   └── cloudflare-dns-token
│   └── [các infra jobs]
```

**3. Đặt tên Credentials ID có ý nghĩa**

```yaml
# ❌ Tên không rõ ràng:
id: cred1
id: my-token
id: key

# ✅ Tên mô tả rõ ràng:
id: github-payments-repo-deploy-key
id: sonarqube-analysis-token
id: production-k8s-cluster-kubeconfig
id: nexus-release-repo-credentials
```

**4. Rotation Policy (Chính Sách Luân Phiên Thay)**

```yaml
Loại Credentials       | Tần Suất Rotation  | Trigger Ngay Khi
---------------------- | ------------------- | --------------------------------
SSH deploy key         | Mỗi 6–12 tháng     | Nhân viên nghỉ việc, key bị lộ
API token              | Mỗi 3–6 tháng      | Service bị breach, token lộ
Database password      | Mỗi 3 tháng        | Engineer rời team, breach
Service account        | Mỗi 6 tháng        | Nghi ngờ compromise
Certificate            | Trước hạn 30 ngày  | CA bị revoke, cert compromise
```

**5. Principle of Least Privilege với Credentials**

```
Ví dụ SSH deploy key:
  ✅ Chỉ cấp READ cho deploy key đọc code
  ❌ Không cần WRITE nếu Jenkins chỉ clone, không push

Ví dụ Service Account Kubernetes:
  ✅ Chỉ cấp quyền deploy vào namespace "production"
  ❌ Không dùng cluster-admin cho CI/CD pipeline
```

**6. Backup Encryption Key**

```bash
# Master encryption key cần backup riêng
# Mất key này → mất toàn bộ credentials

# Backup:
cp $JENKINS_HOME/secrets/master.key /secure-backup/jenkins-master.key
cp $JENKINS_HOME/secrets/hudson.util.Secret /secure-backup/

# Lưu backup ở nơi khác với Jenkins server
# Ví dụ: AWS Secrets Manager, encrypted S3, offline backup
```

---

## 9. Troubleshooting Credentials

### Lỗi — "CredentialsMatcher" Không Tìm Thấy Credential

```
Error: ERROR: No valid credential found for id 'my-credential-id'

Nguyên nhân phổ biến:
1. Credential ID sai — kiểm tra chính xác case-sensitive
2. Credential ở sai Scope: System scope không dùng được trong pipeline
3. Credential trong Folder khác — job không thuộc folder đó không thấy
4. User không có Credentials/Use permission

Kiểm tra:
Manage Jenkins → Credentials → tìm credential bằng search
→ Verify ID và Scope
```

### Lỗi — Secret Bị Lộ Ra Build Log

```
Dấu hiệu: Password hiện ra rõ ràng trong log thay vì ****

Nguyên nhân:
1. Dùng `echo` trực tiếp: sh "echo $SECRET"
2. Dùng Groovy string interpolation trước khi truyền vào sh
3. Script bên trong sh có log lại secret

Sửa:
// Tránh Groovy interpolation:
sh "echo ${SECRET}"        // ❌ Groovy xử lý trước, secret bị expand
sh 'echo $SECRET'          // ✅ Shell variable, Jenkins mask được
sh """echo \$SECRET"""     // ✅ Escape để shell xử lý
```

### Kiểm Tra Credentials Đang Dùng Ở Đâu

```groovy
// Script Console — tìm tất cả job đang dùng credential cụ thể
import com.cloudbees.plugins.credentials.*
import jenkins.model.*

def credId = "my-credential-id"
def credentialUsages = []

Jenkins.instance.getAllItems(hudson.model.Job.class).each { job ->
    job.buildMixins?.each { mixin ->
        // Tìm trong job config
    }
    // Đọc config XML của job
    def configXml = job.getConfigFile().asString()
    if (configXml.contains(credId)) {
        credentialUsages << job.fullName
    }
}

println "Credential '$credId' được dùng trong:"
credentialUsages.each { println "  - $it" }
```

### Decrypt Credentials (Chỉ Cho Admin — Emergency)

```groovy
// Script Console — decode encrypted credential (dùng khi cần khẩn cấp)
import com.cloudbees.plugins.credentials.*
import com.cloudbees.plugins.credentials.impl.*
import jenkins.model.*

def creds = CredentialsProvider.lookupCredentials(
    StringCredentials.class,
    Jenkins.instance,
    null,
    null
)

creds.find { it.id == "my-secret-id" }?.with {
    println "Secret: ${it.secret}"
}
// CẢNH BÁO: Chỉ chạy trên console local, không log output này!
```

---

## Tóm Tắt Nhanh

| Câu Hỏi                                                   | Câu Trả Lời                                                      |
| ---------------------------------------------------------- | ----------------------------------------------------------------- |
| Credentials được mã hóa bằng gì?                          | AES sử dụng master.key trong JENKINS_HOME/secrets/               |
| Dùng `environment` hay `withCredentials`?                  | `withCredentials` an toàn hơn — scope nhỏ hơn, ngắn hơn         |
| Làm sao secret bị mask trong log?                         | Jenkins tự động replace giá trị secret bằng `****` khi log       |
| Khi nào dùng Vault thay vì Jenkins Credentials Store?      | Khi cần dynamic credentials, audit trail chi tiết, multi-platform |
| Scope nào cho credentials dùng trong pipeline?            | Global (toàn Jenkins) hoặc Folder scope (cô lập theo team)       |

---

**Xem tiếp:** [4-security-best-practices.md](4-security-best-practices.md) — Hardening và best practices bảo mật Jenkins
