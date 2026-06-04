# 1 — Khái Niệm CI/CD

> Phân biệt CI (Continuous Integration — Tích Hợp Liên Tục), CD Delivery (Continuous Delivery — Chuyển Giao Liên Tục) và CD Deployment (Continuous Deployment — Triển Khai Liên Tục) — nền tảng tư duy của mọi pipeline hiện đại.

---

## 📚 Mục Lục

1. [Vấn Đề CI/CD Giải Quyết](#vấn-đề-cicd-giải-quyết)
2. [CI — Continuous Integration](#ci--continuous-integration--tích-hợp-liên-tục)
3. [CD Delivery vs CD Deployment](#cd-delivery-vs-cd-deployment)
4. [Luồng CI/CD Hoàn Chỉnh](#luồng-cicd-hoàn-chỉnh)
5. [DevOps và CI/CD](#devops-và-cicd)
6. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề CI/CD Giải Quyết

### Trước Khi Có CI/CD — "Integration Hell" — Địa Ngục Tích Hợp

Hãy tưởng tượng một team 10 developer làm việc trên các nhánh riêng trong 2 tuần:

```
Developer A  ─────────────────────────────────► merge
Developer B  ────────────────────────────────► merge
Developer C  ───────────────────────────────► merge
                                              ↑
                                     Tuần 2: Mọi người merge cùng lúc
                                     → Xung đột code khắp nơi
                                     → 3 ngày chỉ để resolve conflicts
                                     → Lỗi mới xuất hiện, không ai biết do ai
```

**Hậu quả thực tế:**
- Release bị delay liên tục vì tích hợp mất quá nhiều thời gian
- "Works on my machine" — Chạy được trên máy tôi — nhưng lỗi trên server
- Bug được phát hiện muộn → chi phí sửa cao gấp 10–100 lần
- Team lo sợ merge code → ngại chia sẻ thay đổi → vòng tròn luẩn quẩn

### Sau Khi Có CI/CD

```
Developer A  ──push──► [Build + Test tự động] ──pass──► merge ngay
Developer B  ──push──► [Build + Test tự động] ──fail──► thông báo ngay, fix ngay
Developer C  ──push──► [Build + Test tự động] ──pass──► merge ngay
```

**Lợi ích:**
- Lỗi bị phát hiện trong vòng phút, không phải ngày
- Code luôn trong trạng thái có thể deploy
- Team tự tin push code thường xuyên hơn
- Giảm rủi ro release lớn → nhiều release nhỏ, an toàn hơn

---

## CI — Continuous Integration — Tích Hợp Liên Tục

### Định Nghĩa

> **CI** là thực hành mỗi developer tích hợp (merge) code vào nhánh chính thường xuyên — ít nhất một lần mỗi ngày. Mỗi lần tích hợp được xác minh tự động bằng build và automated tests — kiểm thử tự động.

### Nguyên Tắc Cốt Lõi

```
1. Maintain a single source repository   — Duy trì một kho mã nguồn duy nhất
2. Automate the build                    — Tự động hóa quá trình build
3. Make the build self-testing           — Build tự kiểm tra
4. Everyone commits to mainline daily    — Mọi người commit vào nhánh chính mỗi ngày
5. Every commit builds the mainline      — Mọi commit kích hoạt build
6. Keep the build fast                   — Giữ build nhanh (< 10 phút)
7. Fix broken builds immediately         — Sửa build hỏng ngay lập tức
```

### CI Pipeline Cơ Bản

```
Code Push (git push)
     │
     ▼
[Trigger] — CircleCI nhận webhook từ GitHub/Bitbucket/GitLab
     │
     ▼
[Checkout] — Lấy code mới nhất về môi trường build
     │
     ▼
[Install Dependencies] — npm install / pip install / mvn install
     │
     ▼
[Static Analysis] — Lint, code style check (tùy chọn nhưng khuyến khích)
     │
     ▼
[Build / Compile] — Biên dịch code (bắt buộc với compiled languages)
     │
     ▼
[Unit Tests] — Kiểm thử đơn vị — nhanh, không phụ thuộc ngoài
     │
     ▼
[Integration Tests] — Kiểm thử tích hợp — có thể cần DB, service ngoài
     │
     ▼
✅ CI Pass → Code sẵn sàng để xem xét/merge
❌ CI Fail → Thông báo developer ngay lập tức
```

### Ví Dụ CI Config Trong CircleCI

```yaml
version: 2.1

jobs:
  ci-check:
    docker:
      - image: cimg/node:20.0    # Node.js 20, image chuẩn của CircleCI
    steps:
      - checkout                  # Lấy code từ repository
      - run:
          name: Cài đặt dependencies
          command: npm ci          # npm ci: deterministic install — cài đặt xác định
      - run:
          name: Kiểm tra code style
          command: npm run lint
      - run:
          name: Chạy unit tests
          command: npm test
      - run:
          name: Build ứng dụng
          command: npm run build

workflows:
  ci-pipeline:
    jobs:
      - ci-check        # Chạy với mọi commit, mọi nhánh
```

---

## CD Delivery vs CD Deployment

### CD — Continuous Delivery — Chuyển Giao Liên Tục

> **CD Delivery** là thực hành đảm bảo code **luôn ở trạng thái có thể release** lên production bất cứ lúc nào. Quá trình deploy lên production vẫn cần **approval thủ công — manual approval**.

```
                   ┌─────────────────────────────────────┐
                   │     Continuous Delivery              │
                   │                                      │
Code Push ──CI──►  │  Staging Deploy ──► Smoke Tests     │
                   │                         │            │
                   │                  [Manual Approval]   │
                   │                         │            │
                   └─────────────────────────┼────────────┘
                                             │
                                             ▼
                                    Production Deploy
                                    (Khi team sẵn sàng)
```

**Đặc điểm:**
- Code được deploy tự động lên staging — môi trường kiểm thử
- Môi trường production chỉ nhận code khi có người duyệt
- Phù hợp khi cần kiểm soát thời điểm release (tránh release cuối tuần, v.v.)
- Doanh nghiệp lớn thường dùng mô hình này vì compliance — tuân thủ

**Ví Dụ Thực Tế:**
```
Thứ Sáu 17:00 — Developer push feature mới
→ CI pass, staging deploy thành công, smoke tests xanh
→ Nhưng Lead không approve deploy production vào cuối tuần
→ Thứ Hai 9:00 — Review qua, approve → production deploy
```

### CD — Continuous Deployment — Triển Khai Liên Tục

> **CD Deployment** là bước xa hơn: mọi commit vượt qua pipeline **tự động được deploy lên production** mà không cần can thiệp thủ công.

```
                   ┌─────────────────────────────────────┐
                   │     Continuous Deployment            │
                   │                                      │
Code Push ──CI──►  │  Staging Deploy ──► Automated Tests │
                   │                         │            │
                   │                  (Không cần approval)│
                   │                         │            │
                   └─────────────────────────┼────────────┘
                                             │
                                             ▼
                                    Production Deploy
                                    (Tự động, ngay lập tức)
```

**Đặc điểm:**
- Yêu cầu test coverage — độ phủ kiểm thử rất cao (≥ 80–90%)
- Cần monitoring — giám sát và alerting — cảnh báo mạnh ở production
- Cần feature flags — cờ tính năng để kiểm soát rollout — triển khai dần
- Phù hợp với SaaS, startups, teams có văn hóa DevOps trưởng thành

**Công ty áp dụng Continuous Deployment:**
```
Amazon  → Hàng nghìn deploys mỗi ngày
Netflix → Nhiều deploy mỗi giờ với canary releases
Etsy    → 50+ deploys mỗi ngày
Facebook → Code deploy đến production mỗi ngày
```

### Bảng So Sánh Ba Khái Niệm

| Đặc Điểm | CI | CD Delivery | CD Deployment |
| --------- | -- | ----------- | ------------- |
| **Mục tiêu** | Phát hiện lỗi sớm | Code sẵn sàng release | Tự động release |
| **Tự động đến đâu?** | Build + Test | Staging deploy | Production deploy |
| **Cần approve?** | Không | Có (cho production) | Không |
| **Tần suất release** | Không giới hạn | Theo lịch team | Sau mỗi commit xanh |
| **Rủi ro** | Thấp | Trung bình | Cao (cần monitoring tốt) |
| **Độ trưởng thành DevOps** | Cơ bản | Trung bình | Cao |
| **Phù hợp với** | Mọi team | Team có release window | SaaS, startup |

### Minh Họa Bằng Ví Dụ Thực Tế

```
Tình huống: E-commerce website, team 5 developer

❌ Không có CI/CD:
   → Developer code 2 tuần, merge một lần
   → Tìm bug mất 2 ngày
   → Deploy thủ công, thường sai
   → Release mỗi quý

✅ Chỉ có CI:
   → Mỗi push tự động test
   → Bug phát hiện trong 5 phút
   → Deploy vẫn thủ công
   → Release mỗi tháng

✅ CI + CD Delivery:
   → Mỗi push tự động test + deploy staging
   → QA team test trên staging
   → 1-click deploy lên production khi sẵn sàng
   → Release mỗi tuần

✅ CI + CD Deployment:
   → Mỗi push: test + staging + production tự động
   → Monitoring tự động rollback nếu lỗi
   → Release nhiều lần mỗi ngày
   → Tính năng mới đến tay người dùng ngay
```

---

## Luồng CI/CD Hoàn Chỉnh

### Pipeline Từ Code Đến Production

```
Developer
    │
    │ git push
    ▼
[Source Control] ─── GitHub / Bitbucket / GitLab
    │
    │ Webhook trigger
    ▼
[CircleCI Pipeline]
    │
    ├──► [Stage 1: CI]
    │        ├── Checkout code
    │        ├── Install dependencies (cached)
    │        ├── Lint & static analysis
    │        ├── Unit tests (parallel, split by timing)
    │        └── Integration tests
    │
    ├──► [Stage 2: Build Artifacts — Tạo Artifact]
    │        ├── Build Docker image
    │        ├── Tag với commit SHA
    │        └── Push lên ECR / GCR / Docker Hub
    │
    ├──► [Stage 3: Deploy Staging — Triển Khai Môi Trường Kiểm Thử]
    │        ├── Deploy lên Kubernetes staging
    │        ├── Smoke tests — kiểm thử khói
    │        └── Performance tests (tùy chọn)
    │
    ├──► [Stage 4: Approval Gate — Cổng Duyệt] (CD Delivery)
    │        └── Manual approval trong CircleCI UI
    │             hoặc tự động (CD Deployment)
    │
    └──► [Stage 5: Deploy Production — Triển Khai Production]
             ├── Blue/Green deploy hoặc Canary release
             ├── Health checks — kiểm tra sức khỏe
             └── Notify team (Slack, email)
```

### Nguyên Tắc "Shift Left" — Dịch Chuyển Trái

"Shift Left" có nghĩa là đưa các bước kiểm tra về sớm nhất có thể trong pipeline:

```
Sớm (rẻ)                                              Muộn (đắt)
   │                                                       │
   ▼                                                       ▼
[Lint] → [Unit Test] → [Integration Test] → [E2E Test] → [Production]

Chi phí sửa lỗi:
  $1          $10           $100              $1,000      $10,000+
```

**Quy tắc:** Lỗi phát hiện càng sớm, chi phí sửa càng thấp.

---

## DevOps và CI/CD

### CI/CD Là Trụ Cột Của DevOps

```
           ┌─────────────── DevOps Culture ───────────────┐
           │                                               │
           │  Plan → Code → Build → Test → Release        │
           │    ↑         └──────────────────┘            │
           │    │              CI/CD Pipeline              │
           │    │                    │                     │
           │    └────────────────────┘                     │
           │         Deploy → Operate → Monitor            │
           └───────────────────────────────────────────────┘
```

### Các Chỉ Số Đo Lường CI/CD Hiệu Quả

| Chỉ Số | Tiếng Anh | Mục Tiêu Tốt |
| ------- | --------- | ------------ |
| **Thời gian build** | Build time | < 10 phút cho CI |
| **Tần suất deploy** | Deployment frequency | ≥ 1 lần/ngày (Elite: nhiều lần/ngày) |
| **Thời gian lead** | Lead time for changes | < 1 giờ (Elite: < 1 ngày) |
| **Tỷ lệ thất bại** | Change failure rate | < 15% (Elite: < 5%) |
| **MTTR** | Mean Time to Restore — Thời Gian Phục Hồi Trung Bình | < 1 giờ (Elite: < 1 giờ) |

> Các chỉ số này được lấy từ **DORA Metrics** — Chỉ Số DORA (DevOps Research and Assessment), tiêu chuẩn ngành để đo lường hiệu suất DevOps.

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Giải thích sự khác biệt giữa CI, CD Delivery và CD Deployment

**Trả lời mẫu:**

> CI — Continuous Integration là thực hành tự động build và test code mỗi khi developer push lên repository. Mục tiêu là phát hiện lỗi tích hợp sớm nhất có thể.
>
> CD Delivery mở rộng CI bằng cách tự động deploy lên staging và đảm bảo code luôn sẵn sàng release. Tuy nhiên, việc deploy lên production vẫn cần một bước approval thủ công — phù hợp với các tổ chức cần kiểm soát thời điểm release.
>
> CD Deployment là bước cuối: mọi commit vượt qua pipeline tự động lên production mà không cần can thiệp. Cần test coverage cao, monitoring tốt, và có thể rollback nhanh khi lỗi.

### Câu 2: Tại sao CI/CD quan trọng với doanh nghiệp?

**Trả lời mẫu:**

> CI/CD rút ngắn vòng phản hồi từ code đến production, cho phép doanh nghiệp phản ứng nhanh với thay đổi thị trường. Cụ thể: giảm rủi ro release (nhiều release nhỏ thay vì ít release lớn), phát hiện lỗi khi còn rẻ để sửa, và tăng năng suất developer khi không phải dành thời gian debug "integration hell".

### Câu 3: Team bạn nên dùng CD Delivery hay CD Deployment?

**Trả lời mẫu:**

> Phụ thuộc vào độ trưởng thành của team và yêu cầu business. CD Delivery phù hợp khi cần release window cố định, có compliance requirements, hoặc team chưa có test coverage đủ tốt. CD Deployment phù hợp khi có test coverage cao (> 80%), monitoring/alerting mạnh, feature flags để kiểm soát rollout, và văn hóa team chấp nhận "fail fast — thất bại nhanh, phục hồi nhanh".

---

## 🔗 Đọc Tiếp

- [2-circleci-architecture.md](2-circleci-architecture.md) — Kiến trúc Pipeline → Workflow → Job → Step
- [../02-configuration/README.md](../02-configuration/README.md) — Cấu hình YAML thực tế
- [../10-interview-prep/1-INTERVIEW_GUIDE.md](../10-interview-prep/1-INTERVIEW_GUIDE.md) — Toàn bộ câu hỏi phỏng vấn

---

**Thời Gian Đọc:** 30–45 phút  
**Cập Nhật:** 2026-05-18
