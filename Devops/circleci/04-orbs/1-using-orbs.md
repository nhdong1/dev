# 1. Sử Dụng Orbs — Using Orbs từ Registry

> Hướng dẫn import, cấu hình và sử dụng orb có sẵn từ CircleCI Orb Registry — Kho Orb CircleCI.

---

## 📚 Mục Lục

1. [Cú Pháp Import Orb](#cú-pháp-import-orb)
2. [Namespace và Versioning](#namespace-và-versioning)
3. [Dùng Commands Từ Orb](#dùng-commands-từ-orb)
4. [Dùng Jobs Từ Orb](#dùng-jobs-từ-orb)
5. [Dùng Executors Từ Orb](#dùng-executors-từ-orb)
6. [Parameters Của Orb](#parameters-của-orb)
7. [Kết Hợp Nhiều Orbs](#kết-hợp-nhiều-orbs)
8. [Tìm Và Khám Phá Orbs](#tìm-và-khám-phá-orbs)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cú Pháp Import Orb

### Cấu Trúc Cơ Bản

```yaml
version: 2.1

# Khai báo orbs ở cấp cao nhất của config
orbs:
  <alias>: <namespace>/<orb-name>@<version>
```

**Giải thích:**
- `<alias>` — tên bí danh bạn đặt để dùng trong config (tự chọn)
- `<namespace>` — tên tổ chức sở hữu orb (ví dụ: `circleci`, `datadog`)
- `<orb-name>` — tên orb
- `<version>` — phiên bản (semantic versioning)

### Ví Dụ Thực Tế

```yaml
version: 2.1

orbs:
  # Alias "node" → dùng commands/jobs với prefix node/
  node: circleci/node@5.1.0

  # Alias "aws" → dùng với prefix aws/
  aws: circleci/aws-cli@4.0.0

  # Alias "docker" → dùng với prefix docker/
  docker: circleci/docker@2.3.0

  # Alias đặt tùy ý — không cần trùng tên orb
  deploy-tool: circleci/aws-cli@4.0.0  # vẫn hợp lệ
```

---

## Namespace Và Versioning

### Semantic Versioning — Quản Lý Phiên Bản Theo Ngữ Nghĩa

CircleCI orbs dùng SemVer — Semantic Versioning: `MAJOR.MINOR.PATCH`

| Ký Hiệu | Ý Nghĩa | Ví Dụ |
|---------|---------|-------|
| `@5.1.0` | Pin exact version — Ghim phiên bản chính xác | Reproducible, an toàn nhất |
| `@5.1` | Latest patch trong minor 5.1 | Nhận bug fixes tự động |
| `@5` | Latest minor.patch trong major 5 | Nhận features mới, không breaking |
| `@volatile` | Latest version bất kỳ | **Không dùng trong production** |
| `@dev:alpha` | Development version | Chỉ dùng khi dev orb |

```yaml
orbs:
  # Recommended — Khuyến Nghị cho production
  node: circleci/node@5.1.0        # Exact pin

  # Acceptable — Chấp Nhận Được cho non-critical
  node: circleci/node@5            # Major pin

  # Avoid — Tránh dùng
  node: circleci/node@volatile     # Không ổn định
```

### Tìm Phiên Bản Mới Nhất

```bash
# Dùng CircleCI CLI để xem thông tin orb
circleci orb info circleci/node

# Liệt kê tất cả phiên bản
circleci orb list-versions circleci/node

# Xem source của một phiên bản cụ thể
circleci orb source circleci/node@5.1.0
```

---

## Dùng Commands Từ Orb

### Cú Pháp

```yaml
steps:
  - <alias>/<command-name>:
      <param1>: <value1>
      <param2>: <value2>
```

### Ví Dụ Thực Tế

```yaml
version: 2.1

orbs:
  node: circleci/node@5.1.0
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  build-and-deploy:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout

      # Command từ node orb — cài packages với caching tích hợp
      - node/install-packages:
          pkg-manager: npm          # Parameter: npm hoặc yarn hoặc pnpm
          cache-version: v2         # Parameter: cache key prefix

      # Command từ aws-cli orb — thiết lập AWS credentials
      - aws-cli/setup:
          role-arn: arn:aws:iam::123456789:role/circleci-role  # OIDC
          region: us-east-1

      - run: npm run build
      - run: aws s3 sync dist/ s3://my-bucket
```

### Xem Commands Có Sẵn Trong Orb

```bash
# Xem tất cả commands, jobs, executors của một orb
circleci orb source circleci/node@5.1.0

# Hoặc xem trên trang web Orb Registry
# https://circleci.com/developer/orbs/orb/circleci/node
```

---

## Dùng Jobs Từ Orb

Một số orbs cung cấp sẵn **pre-built jobs** — công việc đóng gói sẵn. Bạn dùng trực tiếp trong `workflows` mà không cần định nghĩa job.

### Cú Pháp

```yaml
workflows:
  my-workflow:
    jobs:
      - <alias>/<job-name>:
          <param1>: <value1>
```

### Ví Dụ — Node Orb Job

```yaml
version: 2.1

orbs:
  node: circleci/node@5.1.0

workflows:
  test-and-build:
    jobs:
      # Job test được cung cấp sẵn bởi node orb
      - node/test:
          version: "20.0"           # Node.js version
          run-command: npm test      # Lệnh chạy test

      # Job build tự định nghĩa, chạy sau test
      - build:
          requires:
            - node/test

jobs:
  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - run: npm run build
```

### Ví Dụ — Docker Orb Job

```yaml
version: 2.1

orbs:
  docker: circleci/docker@2.3.0

workflows:
  build-push:
    jobs:
      # Job publish image lên Docker Hub — được cung cấp bởi docker orb
      - docker/publish:
          image: my-org/my-app
          tag: $CIRCLE_SHA1          # Tag bằng commit hash
          docker-username: DOCKER_USERNAME
          docker-password: DOCKER_PASSWORD
```

---

## Dùng Executors Từ Orb

Một số orbs export sẵn executor — môi trường thực thi được cấu hình tối ưu.

```yaml
version: 2.1

orbs:
  node: circleci/node@5.1.0

executors:
  # Executor từ orb — đã có image Node.js tối ưu
  default-node:
    executor: node/default          # Dùng executor của node orb
    resource_class: medium

jobs:
  lint:
    executor: default-node          # Tái sử dụng executor
    steps:
      - checkout
      - node/install-packages
      - run: npm run lint

  test:
    executor: default-node          # Cùng executor, không lặp lại cấu hình
    steps:
      - checkout
      - node/install-packages
      - run: npm test
```

---

## Parameters Của Orb

### Cách Truyền Parameters

```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  deploy-staging:
    docker:
      - image: cimg/base:stable
    steps:
      - aws-cli/setup:
          # Parameters cho command aws-cli/setup
          role-arn: arn:aws:iam::111111111111:role/staging-deploy-role
          region: ap-southeast-1
          session-duration: "1800"    # 30 phút

  deploy-production:
    docker:
      - image: cimg/base:stable
    steps:
      - aws-cli/setup:
          role-arn: arn:aws:iam::222222222222:role/prod-deploy-role
          region: us-east-1
          session-duration: "900"     # 15 phút — production ngắn hơn
```

### Xem Parameters Của Orb

```bash
# Xem schema đầy đủ của orb (parameters, defaults, required)
circleci orb source circleci/aws-cli@4.0.0 | grep -A 5 "parameters:"
```

### Ví Dụ Với Node Orb — Đầy Đủ Parameters

```yaml
version: 2.1

orbs:
  node: circleci/node@5.1.0

jobs:
  install-and-test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - node/install-packages:
          # pkg-manager: trình quản lý package (npm/yarn/pnpm)
          pkg-manager: yarn
          # cache-version: prefix cho cache key, thay đổi để invalidate cache
          cache-version: v3
          # override-ci-command: ghi đè lệnh cài đặt mặc định
          override-ci-command: yarn install --frozen-lockfile
          # app-dir: thư mục chứa package.json (nếu không phải root)
          app-dir: ./frontend
```

---

## Kết Hợp Nhiều Orbs

### Pipeline CI/CD Hoàn Chỉnh Dùng Nhiều Orbs

```yaml
version: 2.1

orbs:
  node: circleci/node@5.1.0
  docker: circleci/docker@2.3.0
  aws-cli: circleci/aws-cli@4.0.0
  slack: circleci/slack@4.12.0

jobs:
  # Job 1: Test ứng dụng
  test:
    docker:
      - image: cimg/node:20.0
      - image: cimg/postgres:14.0   # Service container cho test DB
        environment:
          POSTGRES_PASSWORD: testpassword
    steps:
      - checkout
      - node/install-packages:
          pkg-manager: npm
      - run:
          name: Chờ PostgreSQL sẵn sàng
          command: dockerize -wait tcp://localhost:5432 -timeout 1m
      - run: npm test
      - slack/notify:                # Thông báo Slack khi test thất bại
          event: fail
          template: basic_fail_1

  # Job 2: Build Docker image
  build-image:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          version: docker23          # Docker-in-Docker
      - docker/build:
          image: my-org/my-app
          tag: $CIRCLE_SHA1
      - docker/push:
          image: my-org/my-app
          tag: $CIRCLE_SHA1

  # Job 3: Deploy lên AWS ECS
  deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - aws-cli/setup:
          role-arn: $AWS_DEPLOY_ROLE_ARN
      - run:
          name: Update ECS service
          command: |
            aws ecs update-service \
              --cluster my-cluster \
              --service my-service \
              --force-new-deployment
      - slack/notify:                # Thông báo Slack khi deploy thành công
          event: pass
          template: success_tagged_deploy_1

workflows:
  ci-cd:
    jobs:
      - test
      - build-image:
          requires:
            - test
      - hold-for-approval:
          type: approval
          requires:
            - build-image
          filters:
            branches:
              only: main
      - deploy:
          requires:
            - hold-for-approval
          context: production-secrets  # Lấy secrets từ Context
```

---

## Tìm Và Khám Phá Orbs

### Cách Tìm Orbs Phù Hợp

**1. CircleCI Orb Registry (Web)**
```
https://circleci.com/developer/orbs
→ Tìm theo category: Cloud, Testing, Monitoring, Security...
→ Lọc: Certified / Partner / Community
```

**2. CircleCI CLI**
```bash
# Tìm kiếm orbs theo từ khóa
circleci orb list circleci          # Tất cả orbs của CircleCI

# Tìm theo namespace cụ thể
circleci orb list datadog

# Xem chi tiết một orb
circleci orb info circleci/slack
```

**3. Xem Source Của Orb**
```bash
# Xem toàn bộ source YAML — rất hữu ích để hiểu orb làm gì
circleci orb source circleci/node@5.1.0
```

### Checklist Trước Khi Dùng Orb

```
☐ Orb có phải Certified hoặc Partner không?
☐ Phiên bản cuối cùng được update khi nào? (active maintenance?)
☐ Số lượt download/usage có cao không? (community trust)
☐ Có breaking changes nào trong changelog không?
☐ Source code có chứa bất cứ điều đáng ngờ không?
☐ Orb có xử lý secrets của bạn không? (cần review kỹ hơn)
```

---

## Câu Hỏi Phỏng Vấn

### Q: Orb là gì và tại sao nên dùng?

**Trả lời:**
Orb là gói cấu hình CircleCI có thể tái sử dụng, gồm commands, jobs và executors được đóng gói lại. Lý do nên dùng:
1. **Giảm boilerplate** — không cần viết lại cùng cấu hình cho mỗi dự án
2. **Chuẩn hóa** — toàn tổ chức dùng cùng cách deploy
3. **Dễ bảo trì** — cập nhật một chỗ, áp dụng cho tất cả project
4. **Best practices tích hợp** — certified orbs áp dụng cách làm tốt nhất từ chuyên gia

### Q: Sự khác biệt giữa Certified, Partner và Community Orbs?

**Trả lời:**
- **Certified:** Do CircleCI tạo và maintain, kiểm tra kỹ nhất, an toàn cao
- **Partner:** Do công ty đối tác tạo, được CircleCI review và xác nhận
- **Community:** Bất kỳ ai cũng có thể tạo, cần tự review source trước khi dùng

### Q: Làm thế nào để an toàn khi dùng orb của bên thứ ba?

**Trả lời:**
1. Review source code của orb (`circleci orb source <orb>`)
2. Pin phiên bản cụ thể thay vì `@volatile`
3. Ưu tiên Certified > Partner > Community
4. Kiểm tra ngày cập nhật và số lượt dùng
5. Với Community Orbs, phải bật tường lửa trong Organization Settings

### Q: Bạn có thể dùng orb mà không publish lên registry không?

**Trả lời:**
Có — dùng **Inline Orb**. Định nghĩa orb thẳng trong `config.yml` dưới key `orbs`. Inline orbs không được public, chỉ dùng trong project đó. Phù hợp cho prototype hoặc cấu hình chỉ cần dùng nội bộ.

---

## 🔗 Điều Hướng

- **Tiếp theo:** [2-certified-orbs.md](./2-certified-orbs.md) — Certified Orbs phổ biến
- **Tham khảo:** [CircleCI Orb Registry](https://circleci.com/developer/orbs)

---

**Cập Nhật Lần Cuối:** 2026-05-18 | **Phiên Bản:** 1.0
