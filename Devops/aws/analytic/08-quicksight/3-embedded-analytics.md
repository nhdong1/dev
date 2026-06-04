# Embedded Analytics — Tích Hợp Analytics Vào Ứng Dụng

> Embedded Analytics (Phân Tích Nhúng) cho phép nhúng QuickSight dashboards và visuals trực tiếp vào ứng dụng web/mobile của bạn, mang lại trải nghiệm phân tích dữ liệu liền mạch cho người dùng cuối mà không cần họ rời khỏi ứng dụng.

---

## 📚 Mục Lục

1. [Tại Sao Dùng Embedded Analytics?](#1-tại-sao-dùng-embedded-analytics)
2. [Các Loại Embedding](#2-các-loại-embedding)
3. [Kiến Trúc Embedded Analytics](#3-kiến-trúc-embedded-analytics)
4. [QuickSight Embedding SDK](#4-quicksight-embedding-sdk)
5. [Authentication — Xác Thực Người Dùng](#5-authentication--xác-thực-người-dùng)
6. [Customization — Tùy Chỉnh Giao Diện](#6-customization--tùy-chỉnh-giao-diện)
7. [Multi-tenant Architecture](#7-multi-tenant-architecture)
8. [Pricing Model — Mô Hình Tính Phí](#8-pricing-model--mô-hình-tính-phí)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Dùng Embedded Analytics?

### Bài Toán Thực Tế

Bạn xây dựng một SaaS platform (Nền Tảng Phần Mềm Dạng Dịch Vụ) — khách hàng của bạn muốn xem báo cáo và dashboard ngay trong ứng dụng, không phải đăng nhập vào hệ thống BI riêng.

```
Giải pháp KHÔNG dùng Embedded:         Giải pháp Embedded Analytics:
┌─────────────────────┐                 ┌─────────────────────────────┐
│   Ứng Dụng SaaS     │                 │      Ứng Dụng SaaS          │
│                     │                 │  ┌───────────────────────┐  │
│  [Dữ Liệu]          │                 │  │  QuickSight Dashboard  │  │
│                     │                 │  │  (Nhúng liền mạch)    │  │
└─────────────────────┘                 │  └───────────────────────┘  │
         │                              │  [Dữ Liệu] [Analytics]      │
         └── Người dùng phải            └─────────────────────────────┘
             chuyển sang                         │
             QuickSight                 Mọi thứ trong một giao diện
```

### Lợi Ích Chính

| Lợi Ích | Mô Tả |
|---------|-------|
| **Time-to-market** | Không cần tự xây dựng BI engine — tiết kiệm nhiều tháng phát triển |
| **UX liền mạch** | Người dùng không cần chuyển ứng dụng |
| **Data isolation** | Mỗi tenant (Khách Hàng) chỉ thấy dữ liệu của họ (RLS) |
| **Scalability** | AWS tự động scale — không lo infrastructure |
| **Rich visuals** | Hàng chục loại biểu đồ, ML insights sẵn có |

---

## 2. Các Loại Embedding

### 2.1 Dashboard Embedding — Nhúng Bảng Điều Khiển

Nhúng toàn bộ published dashboard vào ứng dụng.

```
Use case phổ biến:
├── Customer-facing reporting (Báo cáo cho khách hàng)
├── Executive dashboards trong internal portal
└── Operational monitoring trong DevOps platform
```

### 2.2 Visual Embedding — Nhúng Biểu Đồ Đơn Lẻ

Nhúng một visual cụ thể (không phải toàn bộ dashboard) vào bất kỳ vị trí nào trong ứng dụng.

```
Use case:
├── Nhúng bar chart vào trang profile của user
├── Hiển thị trend line trong card sản phẩm
└── KPI metric trong header của ứng dụng
```

### 2.3 Console Embedding — Nhúng Toàn Bộ QuickSight

Nhúng toàn bộ QuickSight console (bao gồm cả khả năng tạo analysis) vào ứng dụng.

```
Use case:
├── Self-service BI (Người dùng tự tạo dashboard)
├── Internal analytics platform
└── Data analyst workbench tích hợp vào data platform
```

### 2.4 Q Embedding — Nhúng Natural Language Query

Nhúng thanh tìm kiếm Q (QuickSight Q) để người dùng đặt câu hỏi bằng ngôn ngữ tự nhiên.

```
Người dùng gõ: "Doanh thu tháng 3 theo từng region?"
QuickSight Q  →  Tự động tạo visual phù hợp
```

---

## 3. Kiến Trúc Embedded Analytics

```
┌─────────────────────────────────────────────────────────────┐
│                    Ứng Dụng Của Bạn                         │
│                                                              │
│  ┌──────────────────────┐   ┌───────────────────────────┐  │
│  │    Frontend (React/  │   │    Backend API Server      │  │
│  │    Angular/Vue)      │   │    (Node.js/Python/Java)  │  │
│  │                      │   │                           │  │
│  │  1. User truy cập    │   │  3. Xác thực user         │  │
│  │     trang analytics  │──▶│  4. Gọi QuickSight API    │  │
│  │                      │   │     để lấy embed URL      │  │
│  │  5. Nhúng iframe     │◀──│  5. Trả về signed URL     │  │
│  │     với embed URL    │   │                           │  │
│  └──────────────────────┘   └───────────────────────────┘  │
│           │                              │                   │
│           │ iframe                       │ AWS SDK           │
│           ▼                             ▼                   │
│  ┌──────────────────────┐   ┌───────────────────────────┐  │
│  │   QuickSight         │   │   QuickSight API           │  │
│  │   Dashboard          │   │   GenerateEmbedUrlFor      │  │
│  │   (Hiển thị trong    │   │   RegisteredUser()         │  │
│  │    ứng dụng)         │   └───────────────────────────┘  │
│  └──────────────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

### Luồng Xác Thực 5 Bước

```
1. User đăng nhập vào ứng dụng của bạn
        ↓
2. Ứng dụng xác thực user (session/JWT)
        ↓
3. Backend gọi QuickSight API: GenerateEmbedUrlForRegisteredUser
   với thông tin: namespace, dashboard-id, user-arn
        ↓
4. QuickSight trả về signed embed URL (có TTL — Time To Live)
        ↓
5. Frontend nhúng URL vào iframe hoặc dùng SDK
```

---

## 4. QuickSight Embedding SDK

### 4.1 Cài Đặt SDK

```bash
# npm
npm install amazon-quicksight-embedding-sdk

# yarn
yarn add amazon-quicksight-embedding-sdk
```

### 4.2 Backend — Tạo Embed URL

```python
import boto3
import json

def get_quicksight_embed_url(user_email: str, dashboard_id: str) -> str:
    """
    Tạo signed embed URL cho một user cụ thể.
    URL có hiệu lực trong 5 phút — frontend phải dùng ngay.
    """
    quicksight = boto3.client('quicksight', region_name='us-east-1')
    account_id = '123456789012'

    # Đảm bảo user đã được register trong QuickSight
    try:
        quicksight.register_user(
            AwsAccountId=account_id,
            Namespace='default',
            Email=user_email,
            IdentityType='IAM',
            UserRole='READER',  # Chỉ đọc — không chỉnh sửa
            IamArn=f'arn:aws:iam::{account_id}:role/QuickSightEmbedRole',
            SessionName=user_email
        )
    except quicksight.exceptions.ResourceExistsException:
        pass  # User đã tồn tại — bỏ qua

    # Tạo embed URL
    response = quicksight.generate_embed_url_for_registered_user(
        AwsAccountId=account_id,
        SessionLifetimeInMinutes=60,  # URL có hiệu lực 60 phút
        UserArn=f'arn:aws:quicksight:us-east-1:{account_id}:user/default/{user_email}',
        ExperienceConfiguration={
            'Dashboard': {
                'InitialDashboardId': dashboard_id
            }
        },
        AllowedDomains=['https://myapp.com']  # CORS whitelist
    )

    return response['EmbedUrl']
```

```javascript
// Express.js endpoint
app.get('/api/quicksight/embed-url', authenticate, async (req, res) => {
  const { dashboardId } = req.query;
  const userEmail = req.user.email;  // từ session/JWT

  const embedUrl = await getQuickSightEmbedUrl(userEmail, dashboardId);
  res.json({ embedUrl });
});
```

### 4.3 Frontend — Nhúng Dashboard

```javascript
import { createEmbeddingContext } from 'amazon-quicksight-embedding-sdk';

async function embedDashboard(containerId, dashboardId) {
  // Lấy embed URL từ backend
  const response = await fetch(`/api/quicksight/embed-url?dashboardId=${dashboardId}`);
  const { embedUrl } = await response.json();

  // Tạo embedding context (Ngữ Cảnh Nhúng)
  const embeddingContext = await createEmbeddingContext({
    onChange: (changeEvent) => {
      console.log('QuickSight event:', changeEvent.eventName);
    }
  });

  // Nhúng dashboard
  const dashboard = await embeddingContext.embedDashboard({
    url: embedUrl,
    container: document.getElementById(containerId),
    height: '700px',
    width: '100%',
    scrolling: 'no',

    // Callbacks
    onLoad: () => console.log('Dashboard loaded'),
    onError: (error) => console.error('Dashboard error:', error),
  });

  return dashboard;
}

// Gọi hàm
embedDashboard('dashboard-container', 'abc-123-dashboard-id');
```

### 4.4 Tương Tác Với Dashboard Qua SDK

```javascript
// Sau khi embed, có thể tương tác programmatically (Lập Trình)

// Áp dụng filter từ ứng dụng
await dashboard.setParameters([
  { Name: 'selected_region', Values: ['North', 'South'] },
  { Name: 'date_from', Values: ['2024-01-01'] }
]);

// Reset tất cả filter về mặc định
await dashboard.reset();

// Navigate đến sheet khác
await dashboard.navigateToDashboard('other-dashboard-id', {
  parameters: [{ Name: 'product_id', Values: ['prod-001'] }]
});

// Lắng nghe sự kiện từ dashboard
dashboard.on('PARAMETERS_CHANGED', (event) => {
  console.log('User changed parameters:', event.message.changedParameters);
});

dashboard.on('SELECTED_SHEET_CHANGED', (event) => {
  console.log('User navigated to sheet:', event.message.selectedSheet.Name);
});
```

---

## 5. Authentication — Xác Thực Người Dùng

### 5.1 Registered User (Người Dùng Đã Đăng Ký)

Phương thức phổ biến nhất — người dùng được đăng ký trong QuickSight với role cụ thể.

```
Flow:
App user ──▶ Backend đăng ký user vào QuickSight (lần đầu)
          ──▶ Backend tạo embed URL cho user đó
          ──▶ Frontend dùng URL để nhúng
```

**IAM Role cần thiết:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "quicksight:RegisterUser",
        "quicksight:GenerateEmbedUrlForRegisteredUser",
        "quicksight:GetDashboardEmbedUrl"
      ],
      "Resource": "*"
    }
  ]
}
```

### 5.2 Anonymous User (Người Dùng Ẩn Danh)

Cho phép nhúng dashboard mà không cần đăng nhập — phù hợp với public report.

```python
response = quicksight.generate_embed_url_for_anonymous_user(
    AwsAccountId=account_id,
    Namespace='default',
    SessionLifetimeInMinutes=30,
    AuthorizedResourceArns=[
        f'arn:aws:quicksight:us-east-1:{account_id}:dashboard/{dashboard_id}'
    ],
    ExperienceConfiguration={
        'Dashboard': {
            'InitialDashboardId': dashboard_id
        }
    }
)
```

**Lưu ý:** Anonymous user không thể personalize (Cá Nhân Hóa) — Row-Level Security không áp dụng. Tất cả anonymous user thấy cùng dữ liệu.

### 5.3 Namespace — Phân Vùng Người Dùng

Namespace cho phép phân chia người dùng theo tenant (Khách Hàng) trong cùng một AWS account.

```
AWS Account 123456789012
├── Namespace: default
│   └── Users: internal team
├── Namespace: tenant-A
│   └── Users: customer A employees
└── Namespace: tenant-B
    └── Users: customer B employees
```

**Tạo namespace:**

```bash
aws quicksight create-namespace \
  --aws-account-id 123456789012 \
  --namespace tenant-acme \
  --identity-store QUICKSIGHT
```

---

## 6. Customization — Tùy Chỉnh Giao Diện

### 6.1 Themes — Chủ Đề Giao Diện

Tùy chỉnh màu sắc, font chữ để phù hợp với brand của ứng dụng.

```json
{
  "ThemeId": "my-brand-theme",
  "Name": "ACME Corporation Theme",
  "Configuration": {
    "DataColorPalette": {
      "Colors": ["#FF6B35", "#004E89", "#1A936F", "#88D498"],
      "MinMaxGradient": ["#F7F7FF", "#FF6B35"]
    },
    "UIColorPalette": {
      "PrimaryForeground": "#FFFFFF",
      "PrimaryBackground": "#1A1A2E",
      "SecondaryForeground": "#CCCCCC",
      "SecondaryBackground": "#16213E",
      "Accent": "#FF6B35"
    },
    "Sheet": {
      "Tile": {
        "Border": {
          "Show": true
        }
      },
      "TileLayout": {
        "Gutter": {
          "Show": true
        }
      }
    }
  }
}
```

```bash
# Tạo custom theme
aws quicksight create-theme \
  --aws-account-id 123456789012 \
  --theme-id my-brand-theme \
  --name "My Brand Theme" \
  --base-theme-id MIDNIGHT \
  --configuration file://theme-config.json
```

### 6.2 Ẩn Thanh Điều Hướng QuickSight

Khi nhúng vào ứng dụng, có thể ẩn navigation bar (Thanh Điều Hướng) của QuickSight:

```javascript
const dashboard = await embeddingContext.embedDashboard({
  url: embedUrl,
  container: document.getElementById('dashboard-container'),

  // Ẩn thanh điều hướng QuickSight
  undoRedoDisabled: true,    // Ẩn nút Undo/Redo
  resetDisabled: true,       // Ẩn nút Reset filters
});
```

### 6.3 Responsive Design — Thiết Kế Đáp Ứng

```javascript
// Dashboard tự động điều chỉnh theo kích thước container
const dashboard = await embeddingContext.embedDashboard({
  url: embedUrl,
  container: document.getElementById('dashboard-container'),
  height: 'AutoFit',  // Tự động tính chiều cao
  width: '100%',      // Chiếm toàn bộ chiều rộng container
});

// Resize khi cửa sổ thay đổi
window.addEventListener('resize', () => {
  // SDK tự động xử lý resize
});
```

---

## 7. Multi-tenant Architecture

### 7.1 Bài Toán Multi-tenant (Đa Khách Hàng)

Khi bạn có 1000 khách hàng, mỗi khách hàng có dữ liệu riêng:

```
Tenant A: chỉ thấy dữ liệu của Tenant A
Tenant B: chỉ thấy dữ liệu của Tenant B
...
Tenant N: chỉ thấy dữ liệu của Tenant N
```

### 7.2 Chiến Lược 1: Row-Level Security (RLS — Bảo Mật Cấp Hàng)

Tất cả tenant dùng chung dataset, phân quyền bằng RLS.

```
Ưu điểm:
    ✅ Dễ quản lý — một dataset duy nhất
    ✅ Chi phí thấp — một SPICE dataset
    ✅ Thêm tenant dễ — chỉ cần thêm row vào RLS table

Nhược điểm:
    ❌ RLS table lớn → quản lý phức tạp
    ❌ Risk (Rủi Ro): lỗi cấu hình RLS → data leak giữa tenant
    ❌ Không thể có schema khác nhau giữa tenant

Cấu hình RLS:
tenant_id   | user_namespace
──────────────────────────────
acme        | john@acme.com
acme        | jane@acme.com
globex      | burns@globex.com
```

### 7.3 Chiến Lược 2: Namespace Isolation (Phân Vùng Theo Namespace)

Mỗi tenant có namespace riêng trong QuickSight.

```
AWS Account
├── Namespace: acme
│   ├── Users: john@acme.com, jane@acme.com
│   ├── Dataset: acme_orders (chỉ data của ACME)
│   └── Dashboard: acme_dashboard
└── Namespace: globex
    ├── Users: burns@globex.com
    ├── Dataset: globex_orders (chỉ data của Globex)
    └── Dashboard: globex_dashboard

Ưu điểm:
    ✅ Isolation tuyệt đối — không thể cross-tenant
    ✅ Có thể tùy chỉnh dashboard per tenant

Nhược điểm:
    ❌ Phức tạp hơn — phải tạo resources per tenant
    ❌ Chi phí SPICE cao hơn — nhiều dataset riêng biệt
    ❌ Cần automation để tạo tenant mới
```

### 7.4 Automation Tạo Tenant Mới

```python
def onboard_new_tenant(tenant_id: str, tenant_email: str):
    """Tự động setup QuickSight cho tenant mới."""
    qs = boto3.client('quicksight', region_name='us-east-1')
    account_id = '123456789012'

    # 1. Tạo namespace cho tenant
    qs.create_namespace(
        AwsAccountId=account_id,
        Namespace=tenant_id,
        IdentityStore='QUICKSIGHT'
    )

    # 2. Register admin user của tenant
    qs.register_user(
        AwsAccountId=account_id,
        Namespace=tenant_id,
        Email=tenant_email,
        IdentityType='IAM',
        UserRole='READER',
        IamArn=f'arn:aws:iam::{account_id}:role/QuickSightEmbedRole',
        SessionName=tenant_email
    )

    # 3. Tạo dataset với filter cho tenant
    # (Data source cần có tenant_id column)
    create_tenant_dataset(qs, account_id, tenant_id)

    # 4. Copy template dashboard và chỉnh cho tenant
    create_tenant_dashboard(qs, account_id, tenant_id)

    print(f"Tenant {tenant_id} onboarded successfully")
```

---

## 8. Pricing Model — Mô Hình Tính Phí

### 8.1 Capacity Pricing (Tính Phí Theo Năng Lực)

Phù hợp nhất cho Embedded Analytics với nhiều user ẩn danh hoặc external user.

```
Capacity Pricing:
├── Session capacity: $250/month cho 500 sessions
│   └── Thêm: $5/100 additional sessions
└── Anonymous access: Bao gồm trong capacity pricing

So sánh với User-based Pricing:
├── Reader user: $5/user/month (tối đa)
└── Author user: $18/user/month

→ Nếu có >50 external users → Capacity Pricing rẻ hơn
```

### 8.2 Tính Chi Phí Ví Dụ

```
Bài toán: SaaS platform với 500 khách hàng, mỗi người xem dashboard 5 lần/tháng
Total sessions = 500 × 5 = 2,500 sessions/tháng

Option 1 - User-based (Reader):
    500 users × $5/user = $2,500/tháng

Option 2 - Capacity Pricing:
    500 sessions base = $250/tháng
    2,000 additional sessions = $100/tháng
    Total = $350/tháng ← Rẻ hơn 7x

→ Kết luận: Capacity Pricing tốt hơn cho embedded use case với nhiều user
```

---

## 9. Câu Hỏi Phỏng Vấn

### Q1: Làm thế nào để nhúng QuickSight dashboard vào ứng dụng React?

**Trả lời (5 bước):**
1. **Backend:** Gọi `GenerateEmbedUrlForRegisteredUser` API → nhận signed URL
2. **Frontend:** `npm install amazon-quicksight-embedding-sdk`
3. **Frontend:** Gọi backend API để lấy embed URL
4. **Frontend:** Dùng SDK `embeddingContext.embedDashboard({ url, container })` để nhúng
5. **Security:** Đảm bảo allowed domains được cấu hình đúng, URL có TTL ngắn (5-60 phút)

### Q2: Làm thế nào để đảm bảo mỗi tenant chỉ thấy dữ liệu của họ?

**Trả lời:** Hai cách chính:
- **Row-Level Security (RLS):** Tạo rules table mapping `user → tenant_id`. QuickSight tự động áp dụng filter. Đơn giản, phù hợp khi schema giống nhau giữa tenant.
- **Namespace Isolation:** Mỗi tenant có namespace riêng với dataset riêng. Isolation tuyệt đối, phù hợp khi cần customization per tenant, nhưng phức tạp hơn và chi phí SPICE cao hơn.

### Q3: Khi nào dùng Anonymous User vs Registered User trong embedding?

**Trả lời:**
- **Anonymous User:** Public report không cần đăng nhập, tất cả người xem thấy cùng dữ liệu. Ví dụ: trang thống kê công khai, report embed vào email.
- **Registered User:** Khi cần cá nhân hóa — mỗi user thấy dữ liệu khác nhau (RLS), cần audit trail (Nhật Ký Kiểm Tra), hoặc user cần tương tác với dashboard (filter, drill-down).

### Q4: Tại sao nên dùng Capacity Pricing thay vì User-based Pricing cho embedded analytics?

**Trả lời:** Embedded analytics thường có external users (khách hàng của khách hàng) — số lượng lớn nhưng mỗi người dùng ít. User-based pricing tính phí theo đầu người, còn Capacity Pricing tính theo số session. Khi số user lớn nhưng session per user thấp (ví dụ: 1000 users × 3 sessions/tháng = 3000 sessions), Capacity Pricing thường rẻ hơn đáng kể. Break-even thường ở ~50 users (so sánh $250 capacity base vs $250 user fees).

---

## 🔗 Liên Kết Tiếp Theo

- [QuickSight Basics — Dataset, Analysis, Dashboard](./1-quicksight-basics.md)
- [SPICE Engine — In-memory Cache](./2-spice-engine.md)
- [QuickSight Module Overview](./README.md)
- [Module Tiếp Theo: OpenSearch](../09-opensearch/README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-17
