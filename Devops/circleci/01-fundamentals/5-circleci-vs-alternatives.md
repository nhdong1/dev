# 5 — CircleCI vs Các Công Cụ CI/CD Khác

> So sánh CircleCI với Jenkins, GitHub Actions, GitLab CI/CD và các nền tảng phổ biến khác. Hiểu rõ điểm mạnh, điểm yếu và khi nào nên chọn công cụ nào — kỹ năng quan trọng cho phỏng vấn và quyết định kiến trúc thực tế.

---

## 📚 Mục Lục

1. [Tổng Quan Thị Trường CI/CD](#tổng-quan-thị-trường-cicd)
2. [CircleCI — Điểm Mạnh và Điểm Yếu](#circleci--điểm-mạnh-và-điểm-yếu)
3. [Jenkins — Ông Lão Của CI/CD](#jenkins--ông-lão-của-cicd)
4. [GitHub Actions — Người Mới Nổi](#github-actions--người-mới-nổi)
5. [GitLab CI/CD — Toàn Diện Nhất](#gitlab-cicd--toàn-diện-nhất)
6. [Các Công Cụ Khác](#các-công-cụ-khác)
7. [Bảng So Sánh Tổng Hợp](#bảng-so-sánh-tổng-hợp)
8. [Decision Framework — Khung Quyết Định](#decision-framework--khung-quyết-định)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Thị Trường CI/CD

```
Market Share (2024) — Thị Phần
┌─────────────────────────────────────────────────────┐
│  GitHub Actions   ████████████████████  ~35%        │
│  Jenkins          █████████████████    ~30%          │
│  GitLab CI        ██████████           ~18%          │
│  CircleCI         █████                ~8%           │
│  Azure Pipelines  ████                 ~6%           │
│  Others           ██                   ~3%           │
└─────────────────────────────────────────────────────┘

Nguồn: JetBrains Developer Survey 2024
```

Dù market share thấp hơn GitHub Actions và Jenkins, CircleCI vẫn rất phổ biến tại các **startup** và **scale-up** nhờ hiệu năng và ease of use — dễ sử dụng.

---

## CircleCI — Điểm Mạnh và Điểm Yếu

### Điểm Mạnh

```
✅ Tốc độ build cao
   → Resource classes cao, infrastructure mạnh
   → Không phải chia sẻ runner với nhiều tổ chức khác

✅ Orbs ecosystem — Hệ sinh thái Orb
   → Hàng nghìn orbs tái sử dụng được
   → Giảm boilerplate — code lặp đi lặp lại đáng kể

✅ SSH debugging tích hợp sẵn
   → Rerun with SSH → SSH thẳng vào container đang chạy
   → Debug trong môi trường CI thực tế, không cần reproduce cục bộ

✅ Parallelism và test splitting
   → circleci tests split --split-by=timings
   → Phân phối tests thông minh để tối ưu thời gian

✅ Cấu hình đơn giản
   → Một file YAML duy nhất: .circleci/config.yml
   → Cú pháp rõ ràng, dễ đọc

✅ Pipeline Insights
   → Analytics — phân tích chi tiết cho từng workflow, job, step
   → Phát hiện flaky tests tự động

✅ Dynamic Config
   → Cấu hình pipeline động dựa trên điều kiện
   → Monorepo-friendly
```

### Điểm Yếu

```
❌ Chi phí cao hơn GitHub Actions cho team nhỏ
   → GitHub Actions miễn phí cho public repos
   → CircleCI free tier có giới hạn credits

❌ Không tích hợp tự nhiên với GitHub ecosystem
   → Issues, PRs, Projects vẫn cần cấu hình webhook riêng
   → GitHub Actions có native integration tốt hơn

❌ Ít plugin/orb hơn Jenkins
   → Jenkins có 1800+ plugins
   → Nhưng CircleCI orbs đủ cho hầu hết use cases

❌ Không self-hosted mặc định
   → Cần Self-Hosted Runner (trả phí) cho on-premise
   → Jenkins tự quản lý hoàn toàn là ưu thế

❌ Vendor lock-in — Phụ thuộc nhà cung cấp
   → Config format không tương thích với công cụ khác
   → Migration khó nếu muốn chuyển đổi
```

---

## Jenkins — Ông Lão Của CI/CD

### Tổng Quan

Jenkins là CI/CD server mã nguồn mở, ra đời năm 2011 (từ Hudson — 2005). Cần tự cài đặt và quản lý trên server riêng.

### Điểm Mạnh

```
✅ Hệ sinh thái plugin khổng lồ
   → 1800+ plugins cho mọi công nghệ
   → Tích hợp được với bất kỳ tool nào

✅ Hoàn toàn tự quản lý (On-premise)
   → Code không rời khỏi infrastructure nội bộ
   → Phù hợp với yêu cầu bảo mật, compliance cao (ngân hàng, quốc phòng)

✅ Linh hoạt tuyệt đối
   → Tùy chỉnh mọi thứ: UI, logic, workflow
   → Scripted Pipeline bằng Groovy — ngôn ngữ kịch bản Groovy

✅ Miễn phí mã nguồn mở
   → Chỉ tốn chi phí server và nhân lực vận hành
```

### Điểm Yếu

```
❌ Cần đội DevOps để vận hành
   → Cài đặt, cập nhật, backup, scale — tất cả phải tự làm
   → Jenkins master có thể trở thành single point of failure

❌ Cấu hình phức tạp
   → Groovy DSL — Domain Specific Language khó học
   → Declarative Pipeline dễ hơn nhưng vẫn verbose — dài dòng

❌ UI cũ kỹ, khó dùng
   → Giao diện lạc hậu so với các công cụ mới

❌ Security updates phải tự manage
   → CVEs — Common Vulnerabilities and Exposures cần theo dõi thường xuyên
   → Plugin cũ có thể có lỗ hổng bảo mật
```

### Ví Dụ Jenkinsfile So Sánh

```groovy
// Jenkinsfile (Declarative Pipeline — Pipeline Khai Báo)
pipeline {
    agent {
        docker {
            image 'cimg/node:20.0'
        }
    }

    stages {
        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }
        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                always {
                    junit 'test-results/*.xml'
                }
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

```yaml
# CircleCI config.yml tương đương — ít dòng hơn, dễ đọc hơn
version: 2.1
jobs:
  pipeline:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm ci
      - run: npm test
      - store_test_results:
          path: test-results/
      - run: npm run build
      - run:
          command: ./deploy.sh
          when: on_success

workflows:
  ci:
    jobs:
      - pipeline:
          filters:
            branches:
              only: main
```

---

## GitHub Actions — Người Mới Nổi

### Tổng Quan

GitHub Actions ra mắt năm 2019, nhanh chóng trở thành lựa chọn mặc định cho các dự án trên GitHub.

### Điểm Mạnh

```
✅ Tích hợp tự nhiên với GitHub
   → Triggers: push, pull_request, issue, release, schedule
   → Truy cập trực tiếp vào GitHub context (PR number, author, labels)
   → Deploy environments với protection rules

✅ Miễn phí cho public repos
   → Unlimited minutes cho open source projects
   → 2000 minutes/tháng cho private repos (free tier)

✅ Actions Marketplace phong phú
   → Hàng nghìn pre-built actions
   → Cộng đồng đóng góp lớn

✅ Matrix builds tích hợp sẵn
   → Test trên nhiều OS, nhiều phiên bản runtime cùng lúc

✅ Cú pháp YAML quen thuộc
   → Tương tự CircleCI, dễ chuyển đổi
```

### Điểm Yếu

```
❌ Tốc độ có thể chậm hơn CircleCI
   → Shared runners — máy chạy dùng chung với hàng triệu người dùng khác
   → Queue time — thời gian chờ cao vào giờ cao điểm

❌ Tính năng advanced CI/CD kém hơn
   → Không có SSH debugging tích hợp
   → Parallelism kém linh hoạt hơn
   → Không có tính năng tương đương test splitting theo timing

❌ Vendor lock-in với GitHub
   → Nếu repository không trên GitHub → không dùng được
   → Migration từ GitLab/Bitbucket phức tạp

❌ Khó debug hơn CircleCI
   → Không thể SSH vào runner (phải dùng tmate action)
   → Logs ít chi tiết hơn
```

### So Sánh Cú Pháp

```yaml
# GitHub Actions workflow
name: CI Pipeline
on:
  push:
    branches: [main, 'feature/**']
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest        # Shared runner
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test

  build:
    needs: test                   # Tương đương "requires" trong CircleCI
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

```yaml
# CircleCI tương đương
version: 2.1
jobs:
  test:
    docker:
      - image: cimg/node:20.0   # Dedicated runner (nhanh hơn)
    steps:
      - checkout
      - restore_cache:
          keys: [node-{{ checksum "package-lock.json" }}]
      - run: npm ci
      - save_cache:
          key: node-{{ checksum "package-lock.json" }}
          paths: [~/.npm]
      - run: npm test

  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm run build

workflows:
  ci:
    jobs:
      - test
      - build:
          requires: [test]
```

---

## GitLab CI/CD — Toàn Diện Nhất

### Tổng Quan

GitLab CI/CD là phần của nền tảng GitLab, cung cấp bộ DevOps hoàn chỉnh từ source control đến monitoring.

### Điểm Mạnh

```
✅ All-in-one platform — Nền tảng tất cả trong một
   → Source code + CI/CD + Container Registry + Kubernetes + Monitoring
   → Không cần tích hợp nhiều tool riêng lẻ

✅ Auto DevOps
   → Tự động detect project type và tạo pipeline
   → SAST, DAST, dependency scanning sẵn có

✅ Environments và Deployments tích hợp
   → Track deploys trực tiếp trong GitLab UI
   → Review apps — Ứng Dụng Xem Trước cho mỗi Merge Request

✅ Self-hosted (GitLab CE/EE)
   → GitLab Community Edition miễn phí, cài trên server riêng
   → Data sovereignty — chủ quyền dữ liệu cao

✅ Merge Request workflows mạnh
   → Approval rules, protected branches, environment approvals
```

### Điểm Yếu

```
❌ Phức tạp hơn để setup và maintain
   → GitLab EE (Enterprise Edition — Phiên Bản Doanh Nghiệp) đắt
   → Cần dedicated server cho on-premise deployment

❌ Runner management phức tạp
   → Shared runners chậm, tự quản lý runners tốn công

❌ Cấu hình YAML verbose hơn CircleCI
   → .gitlab-ci.yml nhiều boilerplate hơn
```

### Ví Dụ `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  NODE_IMAGE: node:20-alpine

test:
  stage: test
  image: $NODE_IMAGE
  script:
    - npm ci
    - npm test
  artifacts:
    reports:
      junit: test-results/junit.xml

build:
  stage: build
  image: $NODE_IMAGE
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
  only:
    - main

deploy:
  stage: deploy
  image: $NODE_IMAGE
  script:
    - ./deploy.sh
  environment:
    name: production
  when: manual              # Tương đương approval job trong CircleCI
  only:
    - main
```

---

## Các Công Cụ Khác

### Azure Pipelines (Microsoft)

```
✅ Phù hợp nhất khi:
   - Dùng Azure cloud services (AKS, ACR, Azure Functions)
   - Đã dùng Azure DevOps cho project management
   - Team Microsoft-heavy (.NET, Azure)

❌ Không phù hợp khi:
   - Không dùng Azure
   - Non-Microsoft tech stack
```

### Bitbucket Pipelines (Atlassian)

```
✅ Phù hợp nhất khi:
   - Code trên Bitbucket
   - Dùng Jira, Confluence trong team
   - Atlassian ecosystem

❌ Hạn chế:
   - Chỉ cho Bitbucket repos
   - Tính năng advanced CI/CD kém hơn CircleCI
```

### Travis CI

```
Lịch sử:
   - Từng là CI cloud phổ biến nhất (2012-2020)
   - Thay đổi pricing năm 2021 → mất nhiều open source users
   - Nay ít được dùng hơn, nhiều dự án đã migrate sang GitHub Actions
```

### Drone CI

```
✅ Điểm đặc biệt:
   - Container-native — mỗi step chạy trong container riêng
   - Cấu hình đơn giản, dễ hiểu
   - Self-hosted, mã nguồn mở
   - Phù hợp cho startups muốn on-premise

❌ Hạn chế:
   - Cộng đồng nhỏ hơn
   - Ít tính năng advanced hơn CircleCI
```

---

## Bảng So Sánh Tổng Hợp

| Tiêu Chí | CircleCI | Jenkins | GitHub Actions | GitLab CI |
| --------- | -------- | ------- | -------------- | --------- |
| **Dễ cài đặt** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Tốc độ build** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Tự quản lý server** | Không (có option) | Bắt buộc | Không | Tùy chọn |
| **Hệ sinh thái plugin** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Chi phí (nhỏ)** | Trung bình | Thấp (server) | Thấp | Thấp |
| **Chi phí (lớn)** | Trung bình | Trung bình | Thấp–Trung bình | Cao (EE) |
| **Tích hợp GitHub** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **SSH debugging** | ⭐⭐⭐⭐⭐ | Plugin | Kém | Tốt |
| **On-premise** | Self-hosted runner | ⭐⭐⭐⭐⭐ | Enterprise only | ⭐⭐⭐⭐⭐ |
| **Compliance** | SOC 2 Type II | DIY | GitHub Enterprise | GitLab EE |
| **Phù hợp với** | Startup-Scale up | Enterprise (on-prem) | GitHub projects | All-in-one |

---

## Decision Framework — Khung Quyết Định

### Câu Hỏi Để Chọn Công Cụ

```
1. Code repository ở đâu?
   → GitHub → GitHub Actions là lựa chọn đầu tiên
   → GitLab → GitLab CI là lựa chọn đầu tiên
   → Bitbucket → Bitbucket Pipelines
   → Tự quản lý / Nhiều nơi → CircleCI hoặc Jenkins

2. Yêu cầu bảo mật thế nào?
   → Code không được lên cloud → Jenkins (on-premise)
   → Compliance cao (ngân hàng, chính phủ) → Jenkins hoặc GitLab CE
   → Tiêu chuẩn doanh nghiệp bình thường → Bất kỳ cloud CI/CD

3. Quy mô team và dự án?
   → 1-5 developers, GitHub → GitHub Actions (miễn phí)
   → 5-50 developers, muốn nhanh và feature-rich → CircleCI
   → 50+ developers, cần customization → Jenkins hoặc GitLab
   → Mobile team (iOS/Android) → CircleCI (macOS executor tốt)

4. Kỹ năng team?
   → DevOps engineers giỏi Groovy/Jenkins → Jenkins
   → Team muốn ít ops overhead → CircleCI hoặc GitHub Actions
   → Muốn all-in-one platform → GitLab

5. Budget?
   → Tối thiểu (open source) → Jenkins hoặc GitHub Actions public repos
   → Chi phí vừa phải → CircleCI
   → Enterprise với nhiều features → GitLab EE hoặc CircleCI Scale
```

### Ma Trận Quyết Định Nhanh

```
             Không tự host server?
                  │
          ┌───────┴───────┐
          Có              Không (Jenkins, GitLab CE)
          │
    Code trên GitHub?
          │
   ┌──────┴──────┐
   Có            Không
   │              │
GitHub          CircleCI
Actions         (nếu cần speed/features)
(simple cases)  GitLab CI (nếu muốn all-in-one)
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Tại sao bạn chọn CircleCI thay vì GitHub Actions?

**Trả lời mẫu:**

> Tôi chọn CircleCI khi team cần tốc độ build cao hơn — dedicated runners của CircleCI nhanh hơn shared runners của GitHub Actions, đặc biệt vào giờ cao điểm. CircleCI cũng có SSH debugging tích hợp sẵn — tính năng cực kỳ hữu ích để debug pipeline phức tạp mà GitHub Actions không có natively.
>
> Tuy nhiên, nếu project nhỏ, dùng GitHub, và không cần advanced features, GitHub Actions là lựa chọn hợp lý hơn vì không tốn thêm chi phí và tích hợp tự nhiên với GitHub ecosystem.

### Câu 2: Khi nào bạn vẫn sẽ chọn Jenkins?

**Trả lời mẫu:**

> Jenkins là lựa chọn khi có yêu cầu on-premise nghiêm ngặt — code không được lên cloud của bên thứ ba, ví dụ dự án ngân hàng, quốc phòng, hay doanh nghiệp có data residency requirements — yêu cầu dữ liệu phải ở trong nước. Jenkins cũng phù hợp khi cần tích hợp với legacy systems — hệ thống cũ thông qua hệ sinh thái plugin phong phú, hoặc khi team đã có sẵn expertise về Jenkins và không muốn tốn công migrate.

### Câu 3: Giải thích tại sao CircleCI phù hợp hơn GitHub Actions cho mobile CI/CD

**Trả lời mẫu:**

> Đối với iOS CI/CD, CircleCI có macOS executor với Apple M1/M2 hardware và nhiều phiên bản Xcode được support tốt. Build iOS trên CircleCI thường nhanh hơn 20-40% so với GitHub Actions do dedicated hardware. Quan trọng hơn, CircleCI có SSH debugging — khi Xcode build lỗi, bạn có thể SSH thẳng vào macOS VM để kiểm tra environment, reproduce và debug. Đây là tính năng cực kỳ có giá trị với iOS builds vì nhiều lỗi rất khó reproduce locally.

---

## 🔗 Đọc Tiếp

- [../02-configuration/README.md](../02-configuration/README.md) — Bắt đầu cấu hình CircleCI thực tế
- [../10-interview-prep/4-tool-comparison.md](../10-interview-prep/4-tool-comparison.md) — So sánh tools chi tiết cho phỏng vấn

---

**Thời Gian Đọc:** 30–40 phút  
**Cập Nhật:** 2026-05-18
