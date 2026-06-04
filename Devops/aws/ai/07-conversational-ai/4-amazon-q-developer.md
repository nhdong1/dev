# Amazon Q Developer — Trợ Lý AI Lập Trình

> Amazon Q Developer (kế thừa từ Amazon CodeWhisperer) là AI assistant tích hợp trong IDE (Môi Trường Phát Triển Tích Hợp) giúp lập trình viên viết code nhanh hơn, phát hiện lỗ hổng bảo mật, nâng cấp codebase cũ và tìm hiểu AWS — hoạt động trực tiếp trong VS Code, IntelliJ, và nhiều IDE khác

---

## 🎯 Amazon Q Developer là gì?

**Amazon Q Developer** là AI coding assistant (trợ lý lập trình AI) cung cấp:

1. **Inline code suggestions** (Gợi Ý Code Tức Thì) — như GitHub Copilot nhưng của AWS
2. **Chat trong IDE** — hỏi đáp về code, giải thích, refactor
3. **/dev agent** — tự động tạo feature từ mô tả ngôn ngữ tự nhiên
4. **Security scanning** (Quét Bảo Mật) — phát hiện lỗ hổng OWASP, CWE
5. **Code transformation** (Biến Đổi Code) — tự động nâng cấp Java version, migrate framework
6. **AWS knowledge** — biết cách dùng mọi AWS API và best practices

---

## 🏗️ Kiến Trúc & Cách Hoạt Động

```
Lập Trình Viên trong IDE (VS Code / IntelliJ / Cloud9)
        ↓ (keystroke, comment, chat message)
Amazon Q Developer Extension/Plugin
        ↓ (context: file content, cursor position, project structure)
Amazon Q Developer Service (AWS Cloud)
        ↓ Foundation Model đã được fine-tune cho code
Code-specific LLM: Understands 15+ programming languages
        ↓ Generated suggestion / response
IDE hiển thị gợi ý → Lập trình viên Accept (Tab) hoặc Dismiss (Esc)
```

### Ngữ Cảnh (Context) Mà Q Developer Sử Dụng

```
Khi sinh gợi ý, Q Developer xem xét:
├── File hiện tại (toàn bộ nội dung)
├── Files đang mở trong tabs khác
├── Project structure (cấu trúc dự án)
├── Import statements (câu lệnh nhập thư viện)
├── Function/class đang viết
├── Comments (ghi chú) ngay trước cursor
└── Language và framework được detect tự động
```

---

## 💡 Inline Code Completion — Hoàn Thành Code Tức Thì

### Cách Kích Hoạt Gợi Ý

```python
# Cách 1: Viết comment mô tả, Q Developer tự gợi ý code
# Function to calculate compound interest (Lãi Suất Kép)
def calculate_compound_interest(principal, rate, time, n):
    # Q Developer gợi ý: return principal * (1 + rate/n) ** (n*time)
    ...

# Cách 2: Bắt đầu viết code, Q Developer hoàn thành
def connect_to_dynamodb():
    dynamodb = boto3.  # Q Developer gợi ý: resource('dynamodb', region_name='us-east-1')

# Cách 3: Đặt tên hàm rõ ràng → Q Developer đoán được body
def parse_s3_event_notification(event: dict) -> list[str]:
    # Q Developer tự điền toàn bộ body
    ...
```

### Ví Dụ Thực Tế — Tích Hợp AWS Services

```python
# Chỉ gõ comment, Q Developer viết toàn bộ code

# Upload file to S3 with server-side encryption
# Q Developer gợi ý:
import boto3
from botocore.exceptions import ClientError

def upload_to_s3_encrypted(file_path: str, bucket: str, key: str) -> bool:
    s3 = boto3.client('s3')
    try:
        s3.upload_file(
            file_path, bucket, key,
            ExtraArgs={
                'ServerSideEncryption': 'AES256',
                'ContentType': 'application/octet-stream'
            }
        )
        return True
    except ClientError as e:
        print(f"Upload failed: {e}")
        return False
```

### Ngôn Ngữ Được Hỗ Trợ

| Ngôn Ngữ | Mức Hỗ Trợ |
|---------|-----------|
| Python | ⭐⭐⭐⭐⭐ Xuất sắc |
| Java | ⭐⭐⭐⭐⭐ Xuất sắc |
| TypeScript / JavaScript | ⭐⭐⭐⭐⭐ Xuất sắc |
| Go | ⭐⭐⭐⭐ Rất tốt |
| C# | ⭐⭐⭐⭐ Rất tốt |
| Rust | ⭐⭐⭐ Tốt |
| Terraform / CloudFormation | ⭐⭐⭐⭐ Rất tốt (AWS infra) |
| SQL | ⭐⭐⭐⭐ Rất tốt |
| Shell / Bash | ⭐⭐⭐ Tốt |

---

## 💬 Chat trong IDE — Hỏi Đáp Thông Minh

### Các Lệnh Chat Hữu Ích

```
/explain  — Giải thích đoạn code được chọn
/fix      — Tìm và sửa lỗi
/test     — Tạo unit tests cho code được chọn
/doc      — Tạo documentation (tài liệu)
/optimize — Gợi ý tối ưu hiệu suất
/dev      — Khởi động /dev agent để tạo feature
```

### Ví Dụ Chat Sessions

**Giải Thích Code Phức Tạp:**

```
Developer: [select đoạn code phức tạp] /explain

Q Developer:
"Đây là implementation (triển khai) của thuật toán sliding window (cửa sổ trượt).

Cụ thể:
1. `left` và `right` là hai con trỏ xác định cửa sổ hiện tại
2. `char_count` là dict đếm tần suất ký tự trong cửa sổ
3. Vòng lặp mở rộng cửa sổ phải (right++) và thu hẹp trái (left++) khi có ký tự lặp
4. `max_length` theo dõi độ dài cửa sổ tối đa tìm được

Time complexity (Độ phức tạp thời gian): O(n) — mỗi ký tự được xét tối đa 2 lần
Space complexity (Độ phức tạp không gian): O(min(n, alphabet_size))"
```

**Tạo Unit Tests Tự Động:**

```
Developer: [select function parse_order] /test

Q Developer sinh ra:
```python
import pytest
from unittest.mock import MagicMock, patch
from myapp.order import parse_order

class TestParseOrder:
    def test_valid_order_returns_parsed_data(self):
        raw = {"id": "ORD-001", "amount": 100.0, "currency": "USD"}
        result = parse_order(raw)
        assert result.order_id == "ORD-001"
        assert result.amount == 100.0

    def test_missing_id_raises_value_error(self):
        raw = {"amount": 100.0}
        with pytest.raises(ValueError, match="order id"):
            parse_order(raw)

    def test_negative_amount_raises_value_error(self):
        raw = {"id": "ORD-002", "amount": -50.0}
        with pytest.raises(ValueError, match="amount"):
            parse_order(raw)

    def test_zero_amount_is_valid(self):
        raw = {"id": "ORD-003", "amount": 0.0}
        result = parse_order(raw)
        assert result.amount == 0.0
```

---

## 🤖 /dev Agent — Tạo Feature Tự Động

### /dev Agent Hoạt Động Như Thế Nào?

```
1. Lập trình viên mô tả feature bằng ngôn ngữ tự nhiên
        ↓
2. /dev Agent phân tích codebase hiện có (project structure, existing code)
        ↓
3. Tạo kế hoạch: Sẽ tạo/sửa file nào, bước nào
        ↓
4. Lập trình viên review kế hoạch, approve hoặc chỉnh sửa
        ↓
5. /dev Agent tự động tạo code, tests, cập nhật files
        ↓
6. Lập trình viên review diff (phần thay đổi), merge hoặc từ chối
```

### Ví Dụ /dev Agent Prompt

```
Developer → /dev: "Add user authentication (xác thực người dùng) to the REST API.
Use JWT tokens (JSON Web Token — Mã Thông Báo Web JSON).
Users should be able to register with email/password and login to get a token.
Store users in DynamoDB. Follow the existing code style."

Q Developer /dev Agent:
Plan (Kế Hoạch):
1. Create src/auth/models.py — User model và Pydantic schemas
2. Create src/auth/service.py — Password hashing, JWT generation/validation
3. Create src/auth/router.py — /register và /login endpoints
4. Update src/main.py — Include auth router
5. Create tests/test_auth.py — Unit và integration tests
6. Update requirements.txt — Thêm python-jose, passlib

Approve plan? [Yes/No/Modify]
```

### Khi Nào /dev Agent Hoạt Động Tốt Nhất

```
✅ Tốt khi:
- Mô tả rõ ràng, có context cụ thể
- Task có scope (phạm vi) rõ ràng (thêm 1 feature, không phải "refactor toàn bộ app")
- Codebase có structure nhất quán (dễ Q hiểu pattern)
- Có tests để verify kết quả

❌ Kém hiệu quả khi:
- Mô tả mơ hồ ("làm cho nó tốt hơn")
- Task cross-cutting quá nhiều systems
- Codebase không có conventions nhất quán
- Yêu cầu thay đổi kiến trúc lớn
```

---

## 🔒 Security Scanning — Quét Lỗ Hổng Bảo Mật

### Loại Lỗ Hổng Được Phát Hiện

| Danh Mục | Ví Dụ Lỗ Hổng |
|---------|--------------|
| **Injection (Tấn Công Chèn Mã)** | SQL injection, Command injection, LDAP injection |
| **Broken Authentication** | Hardcoded credentials (thông tin đăng nhập cứng), weak passwords |
| **Sensitive Data Exposure** | API keys trong source code, PII trong logs |
| **Security Misconfiguration** | S3 bucket public access, open security groups |
| **Cryptographic Failures** | MD5/SHA1 cho password, weak random number |
| **SSRF — Server-Side Request Forgery** | Fetch URL từ user input không validate |
| **XXE — XML External Entity** | Parse XML không an toàn |
| **Dependency Vulnerabilities** | Library với CVE đã biết |

### Ví Dụ Phát Hiện Lỗ Hổng

```python
# CODE CÓ LỖ HỔNG — Q Developer Security Scan sẽ cảnh báo

import mysql.connector
import hashlib

def get_user(username, password):
    # ❌ SQL Injection vulnerability (Lỗ hổng Chèn SQL)
    query = f"SELECT * FROM users WHERE username = '{username}'"
    conn = mysql.connector.connect(host='localhost', database='app')
    cursor = conn.cursor()
    cursor.execute(query)  # Q Developer: CRITICAL: SQL Injection risk

    # ❌ Hardcoded API key (Khóa API cứng trong code)
    api_key = "sk-1234567890abcdef"  # Q Developer: HIGH: Hardcoded secret

    # ❌ MD5 for password hashing (MD5 không an toàn cho mật khẩu)
    hashed = hashlib.md5(password.encode()).hexdigest()  # Q Developer: HIGH: Weak hash

    return cursor.fetchone()

# CODE ĐÃ SỬA — theo gợi ý của Q Developer

import boto3
import bcrypt
from botocore.exceptions import ClientError

def get_user_secure(username: str, password: str) -> dict | None:
    # ✅ Parameterized query (Câu truy vấn tham số hóa)
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('users')

    response = table.get_item(Key={'username': username})
    user = response.get('Item')

    if not user:
        return None

    # ✅ API key từ Secrets Manager (Trình Quản Lý Bí Mật)
    secrets_client = boto3.client('secretsmanager')
    secret = secrets_client.get_secret_value(SecretId='app/api-key')
    api_key = secret['SecretString']

    # ✅ bcrypt cho password (an toàn, có salt)
    if bcrypt.checkpw(password.encode(), user['password_hash'].encode()):
        return user
    return None
```

### Chạy Security Scan Qua CLI

```bash
# Cài đặt AWS CLI với Q Developer
pip install awscli

# Cấu hình credentials
aws configure --profile qdev

# Chạy security scan trên project
aws q security-scan --project-path ./my-project --language python

# Output mẫu:
# CRITICAL: SQL Injection in src/db/user_queries.py:45
# HIGH: Hardcoded secret in src/config.py:12
# MEDIUM: Insecure random in src/utils/token.py:23
# INFO: 127 files scanned, 3 issues found
```

---

## 🔄 Code Transformation — Biến Đổi Code Tự Động

### Java Upgrade (Nâng Cấp Java)

```
Amazon Q Developer hỗ trợ tự động nâng cấp:
- Java 8 → Java 11 / Java 17 / Java 21
- Spring Boot 2.x → Spring Boot 3.x
- JUnit 4 → JUnit 5

Quá Trình:
1. Developer chọn project Java 8 cần nâng cấp
2. Q Developer phân tích toàn bộ codebase
3. Tạo kế hoạch: Dependencies nào cần update, syntax nào cần sửa
4. Tự động thay đổi: pom.xml/build.gradle, source files, test files
5. Chạy tests tự động để verify
6. Developer review và merge
```

**Ví Dụ Thay Đổi Java 8 → Java 17:**

```java
// TRƯỚC (Java 8)
List<String> names = new ArrayList<String>();
names.stream()
    .filter(name -> name.startsWith("A"))
    .collect(Collectors.toList());

// SAU (Java 17 — Q Developer tự động migrate)
var names = new ArrayList<String>();
names.stream()
    .filter(name -> name.startsWith("A"))
    .toList();  // Java 16+ immutable list collector
```

---

## ⚙️ Tích Hợp IDE

### VS Code (Visual Studio Code)

```bash
# Cài đặt extension
# Mở VS Code → Extensions → Search "Amazon Q"
# Hoặc dùng command line:
code --install-extension AmazonWebServices.amazon-q-vscode

# Đăng nhập
# Command Palette (Ctrl+Shift+P) → "Amazon Q: Sign In"
# Chọn: AWS Builder ID (miễn phí cá nhân) hoặc IAM Identity Center (Pro)
```

### IntelliJ IDEA / JetBrains

```
Plugins → Search "Amazon Q" → Install
Restart IDE
Tools → Amazon Q → Sign In
```

### AWS Cloud9 / CloudShell

```
Amazon Q Developer được tích hợp sẵn — không cần cài thêm
Chỉ cần đăng nhập bằng AWS account
```

### Command Line Interface (CLI)

```bash
# Dùng Q Developer trực tiếp trong terminal
aws q chat

# Hỏi câu hỏi về AWS
> "Làm thế nào để list tất cả S3 buckets?"
Q Developer: aws s3 ls

> "Tạo SageMaker endpoint với instance ml.m5.large"
Q Developer: aws sagemaker create-endpoint \
  --endpoint-name my-endpoint \
  --endpoint-config-name my-endpoint-config

# Giải thích lệnh CLI
aws q translate --command "aws ec2 describe-instances --filters Name=instance-state-name,Values=running"
```

---

## 💰 Pricing — Mô Hình Tính Phí

| Gói | Giá | Giới Hạn |
|-----|-----|---------|
| **Free (Miễn Phí)** | $0 | 50 requests/tháng với /dev, security scan không giới hạn, inline suggestions không giới hạn |
| **Pro** | $19/user/month | Không giới hạn /dev requests, enterprise customization, organizational dashboard |

### Customization với Codebase Riêng (Pro)

```
Amazon Q Developer Pro cho phép fine-tune (tinh chỉnh) trên codebase của tổ chức:
- Upload internal code repositories
- Q Developer học code style, naming conventions, internal libraries
- Gợi ý chính xác hơn cho domain-specific code

Quy Trình:
1. Admin upload code lên Amazon S3 (loại bỏ secrets trước)
2. Tạo customization profile trong Q Developer console
3. Lập trình viên chọn customization khi dùng
4. Gợi ý phản ánh code style nội bộ
```

---

## 🆚 So Sánh Amazon Q Developer vs GitHub Copilot

| Tiêu Chí | Amazon Q Developer | GitHub Copilot |
|---------|-------------------|---------------|
| **AWS Knowledge** | ⭐⭐⭐⭐⭐ Chuyên sâu | ⭐⭐⭐ Tổng quát |
| **Security Scanning** | ✅ Built-in, OWASP | ❌ Cần thêm plugin |
| **Code Transformation** | ✅ Java upgrade, automated | ❌ Không có |
| **IDE Support** | VS Code, JetBrains, Cloud9 | VS Code, JetBrains, Vim, Neovim |
| **CLI Integration** | ✅ AWS CLI native | ❌ Không có |
| **Free Tier** | ✅ Generous (50 /dev/tháng) | ✅ Có (giới hạn hơn) |
| **Enterprise customization** | ✅ Pro plan | ✅ Enterprise plan |
| **Multi-language** | 15+ ngôn ngữ | 15+ ngôn ngữ |
| **Best For** | AWS developers | General developers |

---

## 🎯 Best Practices — Thực Hành Tốt Nhất

### Viết Comment Tốt để Q Gợi Ý Tốt Hơn

```python
# ❌ Comment mơ hồ — Q Developer gợi ý kém
# process data
def process(data):
    ...

# ✅ Comment rõ ràng — Q Developer gợi ý chính xác hơn
# Parse S3 event notification and extract bucket name and object key from each record
# Returns list of (bucket, key) tuples, skipping non-ObjectCreated events
def parse_s3_event_records(event: dict) -> list[tuple[str, str]]:
    ...
```

### Sử Dụng Security Scan Trong CI/CD Pipeline

```yaml
# GitHub Actions workflow tích hợp Q Developer Security Scan
name: Security Scan

on: [push, pull_request]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456:role/GitHubActionsQDevRole
          aws-region: us-east-1

      - name: Run Amazon Q Developer Security Scan
        run: |
          pip install amazon-q-developer-cli
          amazon-q security-scan \
            --project-path . \
            --output-format sarif \
            --output-file results.sarif

      - name: Upload SARIF (Static Analysis Results) to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif
```

---

## 📊 Metrics Đánh Giá Hiệu Quả

```
KPI (Key Performance Indicators — Chỉ Số Hiệu Suất Chính) khi dùng Q Developer:
├── Acceptance rate (Tỷ Lệ Chấp Nhận): % gợi ý được accept
│       Mục tiêu: > 25% (benchmark thị trường)
├── Lines of code generated (Dòng Code Được Tạo)
│       So sánh trước/sau adoption (chấp nhận)
├── Time to complete tasks (Thời Gian Hoàn Thành Nhiệm Vụ)
│       Target: giảm 20-40% thời gian dev
├── Security issues found before PR (Lỗ Hổng Phát Hiện Trước PR)
│       Càng sớm phát hiện → càng rẻ để fix
└── Developer satisfaction score (Điểm Hài Lòng Lập Trình Viên)
        Survey hàng tháng
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về Amazon Q Developer

**Q: Amazon Q Developer khác gì Amazon CodeWhisperer?**

> Amazon Q Developer **là phiên bản mở rộng của CodeWhisperer** (đổi tên vào 2024). CodeWhisperer chỉ có inline code suggestions và security scanning. Q Developer bổ sung thêm: **chat trong IDE, /dev agent tạo feature tự động, code transformation (Java upgrade), và AWS CLI integration**. Tất cả tính năng CodeWhisperer đều có trong Q Developer.

**Q: /dev agent và GitHub Copilot Workspace khác nhau thế nào?**

> Cả hai đều là "agentic coding" (lập trình có tác nhân). Điểm khác biệt: Q Developer /dev agent **có khả năng phân tích toàn bộ project context sâu hơn với AWS services**, tích hợp Security Scan vào quá trình tạo code. GitHub Copilot Workspace có UX (User Experience — Trải Nghiệm Người Dùng) tốt hơn với GitHub ecosystem (Issues, PRs). Với AWS-heavy projects, Q Developer thường cho kết quả tốt hơn.

**Q: Security scan của Q Developer có thể thay thế SAST tools (Công cụ Phân Tích Tĩnh) như SonarQube không?**

> Không hoàn toàn. Q Developer Security Scan tốt cho **phát hiện code-level vulnerabilities (lỗ hổng cấp độ code)** trong thời gian thực khi lập trình. SonarQube hoặc Checkmarx phân tích toàn diện hơn: kiểm tra quality gates (cổng chất lượng), technical debt (nợ kỹ thuật), license compliance (tuân thủ giấy phép). Best practice: **Dùng Q Developer trong IDE realtime + SonarQube trong CI/CD pipeline** — bổ sung nhau, không thay thế.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Trước | [3-amazon-q-business.md](./3-amazon-q-business.md) — Amazon Q Business |
| ↑ Module | [README.md](./README.md) — Tổng quan Conversational AI |
| → Tiếp theo | [../08-predictions/README.md](../08-predictions/README.md) — Predictions |
| ↑ Chỉ mục | [../INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03 | **Phiên Bản:** 1.0
