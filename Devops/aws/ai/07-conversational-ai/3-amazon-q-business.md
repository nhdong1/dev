# Amazon Q Business — Trợ Lý AI Doanh Nghiệp

> Amazon Q Business là trợ lý AI thế hệ mới được xây dựng trên nền tảng Generative AI (Trí Tuệ Nhân Tạo Tạo Sinh), kết nối với dữ liệu nội bộ doanh nghiệp để trả lời câu hỏi, tóm tắt tài liệu, tạo nội dung và thực hiện tác vụ thông qua Plugins (Tiện Ích Mở Rộng)

---

## 🎯 Amazon Q Business là gì?

**Amazon Q Business** là dịch vụ SaaS (Software as a Service — Phần Mềm Dịch Vụ) cho phép doanh nghiệp triển khai AI assistant (trợ lý trí tuệ nhân tạo) kết nối với toàn bộ kho dữ liệu nội bộ — từ S3, SharePoint, Confluence, Salesforce đến hệ thống ticketing — mà không cần code phức tạp.

### Vấn Đề Amazon Q Business Giải Quyết

```
TRƯỚC ĐÓ (Không có Q Business):
Nhân viên: "Quy trình xin nghỉ phép là gì?"
    → Phải vào SharePoint tìm tài liệu (mất 15-30 phút)
    → Hỏi đồng nghiệp/HR (mất 1-2 giờ)
    → Đọc email cũ (mất nhiều thời gian)

SAU KHI CÓ Q Business:
Nhân viên: "Quy trình xin nghỉ phép là gì?"
    → Amazon Q Business tìm trong tài liệu HR, Confluence → trả lời trong 5 giây
    → Kèm nguồn tài liệu để tham khảo
    → Có thể tiếp tục hỏi follow-up questions
```

---

## 🏗️ Kiến Trúc Amazon Q Business

```
┌─────────────────────────────────────────────────────────────────┐
│                       Amazon Q Business                          │
│                                                                   │
│  ┌──────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │  Web Chat UI  │    │    Admin Console  │    │  Slack / Teams│  │
│  │  (Giao Diện) │    │    (Quản Trị)     │    │  Integration  │  │
│  └──────┬───────┘    └────────┬─────────┘    └───────┬───────┘  │
│         └────────────────────┼──────────────────────┘           │
│                               ↓                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │               Q Business Application                        │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  Retriever (Bộ Truy Xuất): RAG Engine                 │  │  │
│  │  │  ├── Query Reformulation (Cải Thiện Câu Hỏi)         │  │  │
│  │  │  ├── Vector Search (Tìm Kiếm Véc-Tơ)                 │  │  │
│  │  │  └── Hybrid Search (Kết Hợp Từ Khóa + Ngữ Nghĩa)   │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  Index (Chỉ Mục Nội Dung)                            │  │  │
│  │  │  ├── Document chunks (Đoạn Tài Liệu)                  │  │  │
│  │  │  ├── Embeddings (Véc-Tơ Ngữ Nghĩa)                   │  │  │
│  │  │  └── Metadata + ACL (Danh Sách Kiểm Soát Truy Cập)  │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                               ↓                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │               Data Sources (Nguồn Dữ Liệu — 40+)           │  │
│  │  S3 | SharePoint | Confluence | Salesforce | Jira | Gmail  │  │
│  │  ServiceNow | Zendesk | Slack | GitHub | Database | Custom  │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📦 Data Connectors — Kết Nối Dữ Liệu

### Các Nguồn Dữ Liệu Phổ Biến

| Connector | Loại Dữ Liệu | Đồng Bộ | Ghi Chú |
|-----------|-------------|---------|---------|
| **Amazon S3** | PDF, Word, Excel, HTML, JSON, CSV | Full / Incremental | Native AWS, không cần credential |
| **Microsoft SharePoint** | Pages, Documents, Libraries | Incremental | OAuth2 authentication |
| **Confluence** | Spaces, Pages, Blogs | Incremental | API token |
| **Salesforce** | Accounts, Cases, Knowledge Articles | Incremental | OAuth2 |
| **Jira** | Issues, Projects, Comments | Incremental | API token |
| **ServiceNow** | Incidents, KB Articles | Incremental | Basic auth / OAuth |
| **Google Drive** | Docs, Sheets, PDFs | Incremental | Service account |
| **Gmail** | Emails, Attachments | Incremental | G Suite admin |
| **GitHub** | Repositories, Issues, Wikis | Incremental | Personal access token |
| **Custom connector** | Bất kỳ REST API | Custom schedule | Dùng Custom Document Enrichment |

### Cấu Hình S3 Data Source

```python
import boto3

qbusiness = boto3.client('qbusiness', region_name='us-east-1')

# Tạo S3 data source
response = qbusiness.create_data_source(
    applicationId='app-12345678',
    indexId='index-12345678',
    name='CompanyDocs-S3',
    type='S3',
    configuration={
        'S3Configuration': {
            'BucketName': 'company-internal-docs',
            'InclusionPrefixes': ['hr/', 'policies/', 'procedures/'],
            'ExclusionPatterns': ['*.tmp', 'archive/*'],
            'DocumentsMetadataConfiguration': {
                'S3Prefix': 'metadata/'  # File JSON mapping metadata
            }
        }
    },
    syncSchedule='cron(0 2 * * ? *)',  # Đồng bộ lúc 2 AM mỗi ngày
    roleArn='arn:aws:iam::123456789012:role/QBusinessDataSourceRole',
    description='Tài liệu nội bộ công ty từ S3'
)

print(f"Data source created: {response['dataSourceId']}")
```

### Document Metadata — Tăng Chất Lượng Tìm Kiếm

```json
// File metadata cho document trong S3
{
  "Attributes": [
    {
      "Key": "_category",
      "Value": { "StringValue": "hr-policy" }
    },
    {
      "Key": "_department",
      "Value": { "StringValue": "human-resources" }
    },
    {
      "Key": "_last_updated",
      "Value": { "DateValue": "2026-01-15T00:00:00Z" }
    },
    {
      "Key": "_document_version",
      "Value": { "StringValue": "2.1" }
    }
  ]
}
```

---

## 🔐 Access Control — Kiểm Soát Quyền Truy Cập

### Permission-Aware RAG (RAG Nhận Thức Quyền Truy Cập)

Amazon Q Business **tự động tôn trọng quyền truy cập** của nguồn dữ liệu — người dùng chỉ thấy nội dung họ có quyền xem.

```
Nhân Viên A (phòng HR):
    Hỏi: "Thông tin lương của toàn công ty?"
    → Q Business tìm kiếm → Tài liệu lương có ACL: HR-only
    → Nhân viên A thuộc nhóm HR → CÓ QUYỀN → Trả lời đầy đủ

Nhân Viên B (phòng Engineering):
    Hỏi: "Thông tin lương của toàn công ty?"
    → Q Business tìm kiếm → Tài liệu lương có ACL: HR-only
    → Nhân viên B KHÔNG thuộc nhóm HR → KHÔNG CÓ QUYỀN
    → "Tôi không tìm thấy thông tin phù hợp với quyền của bạn"
```

### IAM Identity Center Integration (Tích Hợp Trung Tâm Nhận Dạng)

```
AWS IAM Identity Center (SSO — Single Sign-On)
    ↕ Đồng bộ người dùng & nhóm
Amazon Q Business Application
    ↕ Ánh xạ quyền
Data Source (SharePoint / Confluence / S3)
    ↕ ACL từ nguồn dữ liệu gốc
```

```python
# Cấu hình user mapping cho Q Business
qbusiness.create_user(
    applicationId='app-12345678',
    userId='user@company.com',
    userAliases=[
        {
            'indexId': 'index-12345678',
            'dataSourceId': 'ds-sharepoint-001',
            'userId': 'CN=John Doe,OU=Users,DC=company,DC=com'  # SharePoint DN
        }
    ]
)
```

---

## 🔌 Plugins — Tích Hợp Hệ Thống Bên Thứ Ba

### Plugin là gì?

**Plugin** (Tiện Ích Mở Rộng) cho phép Q Business không chỉ trả lời câu hỏi mà còn **thực hiện hành động** trong các hệ thống khác.

```
Không có Plugin (chỉ Q&A):
    Nhân Viên: "Tôi cần báo cáo lỗi hệ thống đăng nhập"
    Q Business: "Để báo cáo lỗi, bạn cần vào ServiceNow và tạo incident mới..."
    → Nhân viên phải TỰ vào ServiceNow

Có Plugin (Q&A + Action):
    Nhân Viên: "Tôi cần báo cáo lỗi hệ thống đăng nhập"
    Q Business: "Tôi sẽ tạo incident trong ServiceNow cho bạn."
                "Xác nhận tạo incident với thông tin: Login system error, Priority: Medium?"
    Nhân Viên: "Xác nhận"
    Q Business: "Đã tạo INC0012345. Đội IT sẽ liên hệ trong vòng 4 giờ."
    → Q Business TỰ ĐỘNG thực hiện
```

### Built-in Plugins (Plugin Tích Hợp Sẵn)

| Plugin | Hành Động Hỗ Trợ |
|--------|-----------------|
| **Jira** | Tạo issue, cập nhật status, gán assignee |
| **Salesforce** | Tạo case, tạo lead, cập nhật opportunity |
| **ServiceNow** | Tạo incident, tạo change request |
| **Zendesk** | Tạo ticket, cập nhật ticket |
| **PagerDuty** | Tạo incident, acknowledge (xác nhận) |

### Custom Plugin — Tạo Plugin Tùy Chỉnh

```python
# Định nghĩa Custom Plugin dùng OpenAPI spec
openapi_spec = {
    "openapi": "3.0.0",
    "info": {
        "title": "Internal HR API",
        "version": "1.0.0"
    },
    "paths": {
        "/leave-requests": {
            "post": {
                "operationId": "createLeaveRequest",
                "summary": "Tạo đơn xin nghỉ phép",
                "requestBody": {
                    "content": {
                        "application/json": {
                            "schema": {
                                "type": "object",
                                "properties": {
                                    "employee_id": {"type": "string"},
                                    "start_date": {"type": "string", "format": "date"},
                                    "end_date": {"type": "string", "format": "date"},
                                    "reason": {"type": "string"}
                                },
                                "required": ["employee_id", "start_date", "end_date"]
                            }
                        }
                    }
                },
                "responses": {
                    "200": {
                        "description": "Leave request created successfully"
                    }
                }
            }
        }
    }
}

# Tạo custom plugin
qbusiness.create_plugin(
    applicationId='app-12345678',
    displayName='HR Leave System',
    type='CUSTOM',
    apiSchema={
        'payload': json.dumps(openapi_spec)
    },
    authConfiguration={
        'oAuth2ClientCredentialConfiguration': {
            'secretArn': 'arn:aws:secretsmanager:us-east-1:123456:secret:hr-api-oauth',
            'roleArn': 'arn:aws:iam::123456789012:role/QBusinessPluginRole'
        }
    }
)
```

---

## ⚙️ Admin Controls — Kiểm Soát Quản Trị

### Global Controls (Kiểm Soát Toàn Cục)

```
Admin Console → Amazon Q Business Application → Global Controls

1. Response Scope (Phạm Vi Câu Trả Lời):
   ├── ENTERPRISE_CONTENT_ONLY: Chỉ dùng dữ liệu nội bộ doanh nghiệp
   └── CREATOR_MODE: Cho phép dùng kiến thức nền của Foundation Model

2. Blocked Topics (Chủ Đề Bị Chặn):
   ├── Từ chối trả lời về: chính trị, pháp lý nhạy cảm, cạnh tranh
   └── Custom topics: "Đừng bàn về lương, thưởng, quyết toán cá nhân"

3. Blocked Phrases (Cụm Từ Bị Chặn):
   └── Danh sách từ/cụm từ không được xuất hiện trong response

4. Document Upload Control (Kiểm Soát Tải Lên Tài Liệu):
   └── Cho phép/từ chối người dùng upload tài liệu vào session
```

### Topic-Level Controls (Kiểm Soát Theo Chủ Đề)

```python
# Tạo guardrail topic cho Q Business
qbusiness.update_chat_controls_configuration(
    applicationId='app-12345678',
    blockedPhrasesConfiguration={
        'blockedPhrases': ['salary database', 'employee personal data'],
        'systemMessageOverride': 'Thông tin này không thể chia sẻ qua hệ thống chat.'
    },
    topicConfiguration={
        'rules': [
            {
                'includedUsersAndGroups': {
                    'userIds': [],
                    'userGroups': ['executives', 'hr-team']
                },
                'excludedUsersAndGroups': {
                    'userIds': [],
                    'userGroups': ['contractors', 'interns']
                },
                'ruleType': 'TOPIC_OVERRIDE'
            }
        ]
    },
    responseScope='ENTERPRISE_CONTENT_ONLY'
)
```

---

## 📡 Web Experience — Giao Diện Chat Có Sẵn

### Cách Nhúng Q Business Vào Portal Nội Bộ

```html
<!-- Nhúng Q Business chat widget vào trang intranet -->
<html>
<head>
  <title>Internal AI Assistant</title>
</head>
<body>
  <div id="amazon-q-chat"></div>

  <script>
    // Q Business cung cấp embed URL từ Console
    const embedUrl = 'https://your-app.chat.qbusiness.us-east-1.on.aws';

    const iframe = document.createElement('iframe');
    iframe.src = embedUrl;
    iframe.style.width = '400px';
    iframe.style.height = '600px';
    iframe.style.border = 'none';

    document.getElementById('amazon-q-chat').appendChild(iframe);
  </script>
</body>
</html>
```

### Custom Web Experience với API

```python
import boto3

qbusiness_client = boto3.client('qbusiness', region_name='us-east-1')

def chat_with_q_business(
    user_message: str,
    application_id: str,
    user_id: str,
    conversation_id: str = None
) -> dict:
    """Gửi tin nhắn và nhận phản hồi từ Q Business"""

    params = {
        'applicationId': application_id,
        'userId': user_id,
        'userMessage': user_message,
    }

    if conversation_id:
        params['conversationId'] = conversation_id

    response = qbusiness_client.chat_sync(**params)

    return {
        'answer': response.get('systemMessage', ''),
        'conversation_id': response.get('conversationId'),
        'source_attributions': [
            {
                'title': attr.get('title'),
                'url': attr.get('url'),
                'excerpt': attr.get('snippet')
            }
            for attr in response.get('sourceAttributions', [])
        ]
    }

# Sử dụng
result = chat_with_q_business(
    user_message="Chính sách nghỉ phép năm 2026 như thế nào?",
    application_id='app-12345678',
    user_id='employee@company.com'
)

print(f"Câu trả lời: {result['answer']}")
print(f"Nguồn tham khảo:")
for source in result['source_attributions']:
    print(f"  - {source['title']}: {source['url']}")

# Hỏi tiếp theo (follow-up) trong cùng conversation
follow_up = chat_with_q_business(
    user_message="Tôi có được tích lũy phép không?",
    application_id='app-12345678',
    user_id='employee@company.com',
    conversation_id=result['conversation_id']
)
```

---

## 🔄 So Sánh Amazon Q Business vs Bedrock Knowledge Base

| Tiêu Chí | Amazon Q Business | Bedrock Knowledge Base |
|---------|-------------------|----------------------|
| **Setup time (Thời gian cài đặt)** | Giờ (no-code) | Ngày (developer setup) |
| **Data connectors (Kết nối dữ liệu)** | 40+ có sẵn | Cần custom connector Lambda |
| **Permission-aware** | ✅ Tự động từ nguồn | Cần tự implement |
| **Built-in UI** | ✅ Web chat có sẵn | ❌ Tự xây UI |
| **Plugins (Actions)** | ✅ Built-in + custom | Bedrock Agents (phức tạp hơn) |
| **Customization** | Giới hạn | Full control |
| **Multi-modal** | ❌ Text only (2026) | ✅ Text + Images |
| **Pricing (Giá)** | $3-20/user/month | Pay-per-token |
| **Best for** | Enterprise SaaS rollout | Custom AI apps |

---

## 💰 Pricing — Mô Hình Tính Phí

| Gói | Giá | Tính Năng |
|-----|-----|----------|
| **Lite** | $3/user/month | Chat cơ bản, giới hạn connectors |
| **Pro** | $20/user/month | Đầy đủ connectors, plugins, admin controls |

```
Ví Dụ Chi Phí Thực Tế:
- 100 nhân viên dùng gói Pro: 100 × $20 = $2,000/tháng
- So với: 1 nhân viên HR dành 20 giờ/tháng để trả lời câu hỏi × $30/giờ = $600/tháng
- Với 100 người × câu hỏi → ROI rõ ràng
```

---

## 🧩 Use Cases Thực Tế Theo Ngành

### HR (Human Resources — Nhân Sự)

```
Câu Hỏi Điển Hình:
- "Quy trình xin nghỉ thai sản là gì?"
- "Tôi còn bao nhiêu ngày phép?"
- "Quy định về remote work (làm việc từ xa) năm 2026?"
- "Làm thế nào để đăng ký bảo hiểm sức khỏe?"

Nguồn Dữ Liệu: SharePoint HR Portal, PDF Policy documents, Workday
Plugin: Tạo leave request, cập nhật thông tin nhân viên
```

### IT Support (Hỗ Trợ Kỹ Thuật)

```
Câu Hỏi Điển Hình:
- "Làm thế nào để reset password VPN?"
- "Hướng dẫn cài đặt phần mềm X?"
- "Lỗi kết nối mạng khi làm việc ngoài văn phòng?"

Nguồn Dữ Liệu: Confluence IT wiki, ServiceNow KB, Jira issues
Plugin: Tạo IT incident, check ticket status
```

### Sales (Bán Hàng)

```
Câu Hỏi Điển Hình:
- "Thông tin sản phẩm mới nhất là gì?"
- "Giá của gói Enterprise cho khách hàng ABC?"
- "Case study nào phù hợp để giới thiệu với khách trong ngành tài chính?"

Nguồn Dữ Liệu: Salesforce, S3 (brochures, case studies), Confluence
Plugin: Tạo Salesforce opportunity, lấy pricing từ CPQ
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về Amazon Q Business

**Q: Amazon Q Business khác gì Amazon Kendra?**

> **Amazon Kendra** là search engine (công cụ tìm kiếm) — tìm tài liệu và trả về đoạn trích có liên quan. **Amazon Q Business** là AI assistant — không chỉ tìm mà còn **hiểu ngữ cảnh, tổng hợp câu trả lời từ nhiều nguồn, nhớ lịch sử hội thoại, và thực hiện hành động qua Plugins**. Q Business dùng Kendra hoặc native retriever bên dưới, nhưng thêm lớp Generative AI ở trên.

**Q: Làm thế nào Q Business đảm bảo người dùng không thấy dữ liệu không có quyền?**

> Q Business tích hợp với **IAM Identity Center** để nhận thông tin người dùng/nhóm. Khi index tài liệu từ SharePoint hoặc Confluence, Q Business **lưu ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập) kèm theo mỗi document chunk**. Khi người dùng hỏi, retriever chỉ xem xét các chunks mà user có quyền xem → không bao giờ trả lời dựa trên nội dung ngoài phạm vi quyền.

**Q: Khi nào chọn Q Business thay vì tự xây RAG với Bedrock?**

> Chọn **Q Business** khi: Cần triển khai nhanh (giờ thay vì tuần), có nhiều data sources chuẩn (SharePoint, Confluence, Salesforce), cần Permission-aware RAG, muốn UI chat sẵn có, nhóm IT không có AI engineering resources. Chọn **tự xây với Bedrock** khi: Cần custom UI hoàn toàn, cần multi-modal (ảnh, audio), cần tích hợp phức tạp với hệ thống legacy, cần fine-tuning embedding model.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Trước | [2-lex-advanced.md](./2-lex-advanced.md) — Lex Advanced |
| → Tiếp theo | [4-amazon-q-developer.md](./4-amazon-q-developer.md) — Amazon Q Developer |
| ↑ Module | [README.md](./README.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03 | **Phiên Bản:** 1.0
