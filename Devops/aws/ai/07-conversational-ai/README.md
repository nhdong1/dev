# 07. Conversational AI — AI Đàm Thoại trên AWS

> Hướng dẫn toàn diện về Amazon Lex (Xây Dựng Chatbot), Amazon Q Business (Trợ Lý AI Doanh Nghiệp) và Amazon Q Developer (Trợ Lý AI Lập Trình) — xây dựng chatbot thông minh, AI assistant doanh nghiệp và công cụ hỗ trợ lập trình trên AWS

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|------|----------|------------|
| [README.md](./README.md) | Tổng quan Conversational AI, so sánh dịch vụ, lộ trình học | ✅ |
| [1-lex-fundamentals.md](./1-lex-fundamentals.md) | Intent, Slot, Fulfillment, Multi-turn dialog | ✅ |
| [2-lex-advanced.md](./2-lex-advanced.md) | Custom slot types, Context, Lambda fulfillment, Kendra tích hợp | ✅ |
| [3-amazon-q-business.md](./3-amazon-q-business.md) | Enterprise AI assistant, RAG nội bộ, Data connectors, Plugins | ✅ |
| [4-amazon-q-developer.md](./4-amazon-q-developer.md) | Code generation, Security scan, AWS knowledge, IDE integration | ✅ |

---

## 🎯 Tổng Quan Module

### Conversational AI là gì?

**Conversational AI** (AI Đàm Thoại) là tập hợp các công nghệ cho phép máy tính hiểu và phản hồi ngôn ngữ tự nhiên của con người trong một cuộc hội thoại — bao gồm chatbot (robot trò chuyện), voice assistant (trợ lý giọng nói) và AI assistant (trợ lý trí tuệ nhân tạo).

```
Người dùng nói/gõ → NLU (Natural Language Understanding — Hiểu Ngôn Ngữ Tự Nhiên)
    → Intent Recognition (Nhận Dạng Mục Đích)
    → Slot Filling (Điền Thông Tin Khe)
    → Dialog Management (Quản Lý Hội Thoại)
    → Response Generation (Tạo Phản Hồi)
    → Người dùng nhận câu trả lời
```

### AWS Conversational AI Ecosystem (Hệ Sinh Thái AI Đàm Thoại AWS)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Conversational AI on AWS                      │
├─────────────────┬───────────────────┬───────────────────────────┤
│   Amazon Lex    │  Amazon Q Business│   Amazon Q Developer      │
│  (Chatbot AI)   │  (Trợ lý DN)      │   (Trợ lý Lập Trình)     │
├─────────────────┼───────────────────┼───────────────────────────┤
│ • Intent/Slot   │ • RAG nội bộ      │ • Code generation         │
│ • Multi-turn    │ • Data connectors │ • Security scan           │
│ • Voice/Text    │ • Plugins         │ • Inline chat             │
│ • Lambda hook   │ • Admin controls  │ • /dev agent              │
└─────────────────┴───────────────────┴───────────────────────────┘
         ↓                   ↓                      ↓
   Chatbot & IVR    Enterprise Q&A          Developer Tools
```

---

## 🗂️ Các Dịch Vụ Trong Module

### 1. Amazon Lex V2 — Xây Dựng Chatbot AI

**Amazon Lex** là dịch vụ managed (được quản lý hoàn toàn) để xây dựng chatbot và voice assistant sử dụng cùng công nghệ Deep Learning (Học Sâu) như Alexa của Amazon.

**Thành Phần Cốt Lõi:**

| Thành Phần | Tiếng Việt | Mô Tả |
|-----------|-----------|-------|
| **Bot** | Con Rô-Bốt Trò Chuyện | Container chứa toàn bộ logic hội thoại |
| **Intent** | Mục Đích | Hành động người dùng muốn thực hiện (đặt vé, kiểm tra số dư) |
| **Slot** | Khe / Tham Số | Thông tin cần thu thập để hoàn thành intent (ngày bay, số tài khoản) |
| **Utterance** | Câu Mẫu | Các cách người dùng có thể nói một intent ("Tôi muốn đặt vé", "Book a flight") |
| **Fulfillment** | Xử Lý Kết Quả | Hành động thực thi sau khi thu thập đủ thông tin |
| **Fallback Intent** | Dự Phòng | Xử lý khi không nhận dạng được intent |

**Use Cases (Trường Hợp Sử Dụng) Điển Hình:**
- Chatbot hỗ trợ khách hàng (customer service)
- IVR — Interactive Voice Response (Hệ Thống Phản Hồi Giọng Nói Tương Tác) tự động
- FAQ bot (Bot Câu Hỏi Thường Gặp) tích hợp website
- Trợ lý đặt lịch, tra cứu thông tin nội bộ

### 2. Amazon Q Business — Trợ Lý AI Doanh Nghiệp

**Amazon Q Business** là trợ lý AI thế hệ mới được xây dựng trên nền tảng Generative AI (Trí Tuệ Nhân Tạo Tạo Sinh), kết nối với dữ liệu nội bộ doanh nghiệp để trả lời câu hỏi, tóm tắt tài liệu và thực hiện tác vụ.

**Tính Năng Chính:**

| Tính Năng | Mô Tả |
|-----------|-------|
| **RAG trên dữ liệu nội bộ** | Kết nối S3, SharePoint, Confluence, Salesforce và 40+ nguồn dữ liệu |
| **Data Connectors** (Kết Nối Dữ Liệu) | Đồng bộ tự động, index (lập chỉ mục) liên tục |
| **Plugins** (Tiện Ích Mở Rộng) | Kết nối hệ thống bên thứ ba (Jira, ServiceNow) để thực hiện hành động |
| **Admin Controls** (Kiểm Soát Quản Trị) | Guardrails (Rào Chắn), quyền truy cập theo nhóm |
| **Web experience** | Giao diện chat sẵn có, nhúng vào portal nội bộ |

**Khác Biệt So Với Bedrock Knowledge Base:**

```
Amazon Q Business           vs      Bedrock Knowledge Base
─────────────────────────────────────────────────────────
✅ SaaS, no-code setup              ✅ API-first, developer control
✅ 40+ pre-built connectors         ✅ Tích hợp linh hoạt hơn
✅ User permission-aware            ✅ Multi-modal, multi-language
✅ Built-in web chat UI             ✅ Custom UI hoàn toàn
✅ Plugins thực hiện action         ✅ Agents với Action Groups
❌ Ít tuỳ chỉnh hơn               ❌ Cần tự xây RAG pipeline
```

### 3. Amazon Q Developer — Trợ Lý AI Lập Trình

**Amazon Q Developer** (trước đây là CodeWhisperer) là AI assistant tích hợp vào IDE (Môi Trường Phát Triển Tích Hợp) giúp lập trình viên viết code nhanh hơn, phát hiện lỗi bảo mật và tìm hiểu AWS.

**Tính Năng Chính:**

| Tính Năng | Mô Tả |
|-----------|-------|
| **Inline code completion** (Hoàn Thành Code Tức Thì) | Gợi ý code khi bạn gõ, theo context của file |
| **Chat trong IDE** | Hỏi đáp về code, giải thích hàm, refactor (tái cấu trúc) |
| **/dev agent** | Tự động tạo feature hoặc fix bug từ mô tả ngôn ngữ tự nhiên |
| **Security scanning** (Quét Bảo Mật) | Phát hiện lỗ hổng bảo mật trong code (OWASP, CWE) |
| **AWS knowledge** | Biết API AWS, best practices, CloudFormation template |
| **Code transformation** (Biến Đổi Code) | Nâng cấp Java 8 → Java 17, tự động migrate |

---

## ⚖️ So Sánh Nhanh — Khi Nào Dùng Gì

| Bài Toán | Giải Pháp Phù Hợp |
|---------|-------------------|
| Xây chatbot với luồng hội thoại cố định (đặt vé, tra cứu đơn hàng) | **Amazon Lex** |
| Tổng đài tự động (IVR) với giọng nói | **Amazon Lex + Amazon Connect** |
| FAQ bot tự trả lời từ tài liệu nội bộ | **Amazon Q Business** |
| Nhân viên cần hỏi dữ liệu nội bộ (HR, tài liệu, chính sách) | **Amazon Q Business** |
| Trợ lý AI có thể tạo Jira ticket, ServiceNow incident | **Amazon Q Business + Plugins** |
| Chatbot linh hoạt hoàn toàn với custom logic | **Amazon Bedrock + Agents** |
| Gợi ý code inline trong VS Code / JetBrains | **Amazon Q Developer** |
| Quét lỗ hổng bảo mật trong repository | **Amazon Q Developer Security Scan** |
| Nâng cấp codebase Java cũ lên phiên bản mới | **Amazon Q Developer Transformation** |

---

## 🏗️ Kiến Trúc Tham Khảo

### Chatbot Hỗ Trợ Khách Hàng

```
Website / Mobile App
        ↓ (text/voice)
Amazon Lex V2
   ├── Intent: CheckOrderStatus
   │       Slot: order_id
   │       → Lambda → DynamoDB (lấy thông tin đơn hàng)
   ├── Intent: RequestRefund
   │       Slot: order_id, reason
   │       → Lambda → Backend API (tạo yêu cầu hoàn tiền)
   └── FallbackIntent
           → Amazon Connect (chuyển agent người thật)
```

### Enterprise Knowledge Assistant

```
Nhân Viên (Slack / Web Portal)
        ↓
Amazon Q Business
   ├── Data Sources (Nguồn Dữ Liệu):
   │       S3 (tài liệu PDF, Word) → Indexed
   │       Confluence → Indexed
   │       SharePoint → Indexed
   ├── Retrieval (Truy Xuất): RAG với IAM permissions
   ├── Generation (Tạo Sinh): Foundation Model trả lời
   └── Plugins: Jira (tạo ticket), ServiceNow (tạo incident)
```

### Developer Productivity Loop

```
Lập Trình Viên gõ code
        ↓
Amazon Q Developer (VS Code / IntelliJ)
   ├── Inline suggestion (gợi ý tức thì)
   ├── Chat: "Giải thích hàm này làm gì?"
   ├── /dev: "Thêm unit test cho module Auth"
   └── Security scan: Phát hiện SQL injection, hardcoded secrets
```

---

## 🎓 Lộ Trình Học Module Này

### Beginner — Người Mới (0-1 năm)

- [ ] Hiểu Intent, Slot, Utterance trong Amazon Lex là gì
- [ ] Tạo bot Lex đơn giản qua AWS Console với 2-3 intents
- [ ] Biết Amazon Q Business dùng cho bài toán gì (enterprise Q&A)
- [ ] Biết Amazon Q Developer là gì và cài extension vào VS Code

**Thời gian:** 1-2 ngày

### Intermediate — Trung Cấp (1-3 năm)

- [ ] Xây bot multi-turn dialog (hội thoại nhiều lượt) với context management
- [ ] Viết Lambda function cho Lex fulfillment (xử lý nghiệp vụ)
- [ ] Tích hợp Lex với Amazon Connect cho IVR
- [ ] Thiết lập Amazon Q Business với S3 data connector
- [ ] Dùng Amazon Q Developer /dev agent để tạo feature từ mô tả

**Thời gian:** 1-2 tuần thực hành

### Advanced — Nâng Cao (3+ năm)

- [ ] Thiết kế chatbot kiến trúc enterprise: Lex + Kendra + Lambda + DynamoDB
- [ ] Cấu hình Amazon Q Business admin controls và permission filtering
- [ ] Tích hợp Q Business Plugins với hệ thống nội bộ (REST API)
- [ ] So sánh và quyết định Lex vs Bedrock Agents cho use case cụ thể
- [ ] Thiết kế hybrid: Lex xử lý structured intents + Bedrock cho open-ended questions

**Thời gian:** 2-4 tuần thực hành chuyên sâu

---

## 💡 Điểm Khác Biệt Quan Trọng Cần Nhớ

### Amazon Lex vs Amazon Bedrock Agents

```
Amazon Lex                          Amazon Bedrock Agents
─────────────────────────────────────────────────────────
✅ Structured conversation flow     ✅ Open-ended reasoning
✅ Intent/Slot/Fulfillment model    ✅ Tool use tự động
✅ Voice channel (Connect)          ✅ Multi-step complex tasks
✅ Dễ dùng, ít code                 ✅ Linh hoạt hơn
❌ Kém linh hoạt với câu hỏi mở    ❌ Cần thiết kế phức tạp hơn
❌ Không tự suy luận               ❌ Không có built-in voice
```

**Khi nào chọn Lex:** Use case có luồng hội thoại cố định, cần voice/IVR, không cần AI reasoning phức tạp.

**Khi nào chọn Bedrock Agents:** Open-ended questions, multi-step reasoning, cần gọi nhiều API theo chuỗi tự động.

### Amazon Q Business vs Tự Xây RAG với Bedrock

```
Amazon Q Business                   Bedrock Knowledge Base (tự xây)
─────────────────────────────────────────────────────────────────────
✅ SaaS, setup nhanh (giờ)          ✅ Full control kiến trúc
✅ 40+ data connectors có sẵn       ✅ Multi-modal support
✅ Permission-aware (nhớ quyền)     ✅ Custom embedding/chunking
✅ Built-in Web UI                  ✅ Tích hợp linh hoạt
❌ Ít tuỳ chỉnh deep                ❌ Tốn thời gian xây dựng
❌ Phụ thuộc roadmap AWS            ❌ Phải tự quản lý
```

---

## 📊 Chi Phí Tham Khảo (Reference Pricing)

| Dịch Vụ | Mô Hình Tính Phí | Ước Tính |
|---------|-----------------|---------|
| **Amazon Lex** | Per request (text: $0.004/req, voice: $0.00075/15s) | ~$4 cho 1.000 text requests |
| **Amazon Q Business** | Per user/month ($3 Lite, $20 Pro) | $20/người dùng Pro/tháng |
| **Amazon Q Developer** | Free (cá nhân) / $19/user/month (Pro) | Free tier đủ dùng cá nhân |

> **Lưu ý:** Lex tính phí per request — chatbot có traffic cao cần tính toán kỹ chi phí scaling.

---

## 🔗 Liên Kết Điều Hướng

| | |
|--|--|
| ← Trước | [06-speech/README.md](../06-speech/README.md) — Speech AI |
| → Tiếp theo | [08-predictions/README.md](../08-predictions/README.md) — Predictions |
| ↑ Chỉ mục | [INDEX.md](../INDEX.md) |
| 🏠 Tổng quan | [README.md](../README.md) |

---

**Cập Nhật Lần Cuối:** 2026-06-03 | **Phiên Bản:** 1.0
