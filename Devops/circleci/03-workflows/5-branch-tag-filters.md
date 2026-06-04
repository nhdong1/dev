# Branch & Tag Filters — Bộ Lọc Nhánh và Nhãn

> **Filters** (Bộ Lọc) trong CircleCI cho phép kiểm soát chính xác khi nào một Job trong workflow được phép chạy — dựa trên tên **branch** (nhánh) hoặc **tag** (nhãn) của Git commit. Đây là cơ chế cốt lõi để xây dựng CI/CD strategy đúng đắn cho từng môi trường.

---

## 📚 Mục Lục

1. [Tổng Quan về Filters](#1-tổng-quan-về-filters)
2. [Branch Filters — Bộ Lọc Nhánh](#2-branch-filters--bộ-lọc-nhánh)
3. [Tag Filters — Bộ Lọc Nhãn](#3-tag-filters--bộ-lọc-nhãn)
4. [Regex — Biểu Thức Chính Quy trong Filters](#4-regex--biểu-thức-chính-quy-trong-filters)
5. [Quy Tắc Kế Thừa Filters](#5-quy-tắc-kế-thừa-filters)
6. [Chiến Lược Branching phổ Biến](#6-chiến-lược-branching-phổ-biến)
7. [Mẫu Thực Tế](#7-mẫu-thực-tế)
8. [Lỗi Thường Gặp](#8-lỗi-thường-gặp)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan về Filters

### Cú Pháp Cơ Bản

```yaml
workflows:
  my-workflow:
    jobs:
      - job-name:
          filters:
            branches:
              only:         # Chỉ chạy khi nhánh KHỚP
                - main
                - develop
              ignore:       # Bỏ qua khi nhánh KHỚP
                - feature/*
            tags:
              only:         # Chỉ chạy khi tag KHỚP
                - /^v.*/
              ignore:       # Bỏ qua khi tag KHỚP
                - /^beta-.*/
```

### Nguyên Tắc Cơ Bản

| Khóa | Ý Nghĩa | Hành Động |
| ---- | ------- | --------- |
| `branches.only` | Danh sách nhánh được phép | Job chỉ chạy trên các nhánh này |
| `branches.ignore` | Danh sách nhánh bị loại trừ | Job chạy trên mọi nhánh TRỪ các nhánh này |
| `tags.only` | Danh sách tag được phép | Job chỉ chạy khi tạo tag này |
| `tags.ignore` | Danh sách tag bị loại trừ | Job chạy trên mọi tag TRỪ các tag này |

> **Quan trọng:** Nếu không có `filters`, Job chạy trên **mọi branch push** nhưng **KHÔNG chạy khi push tag** — đây là hành vi mặc định. Để Job chạy khi push tag, phải cấu hình `tags.only` tường minh.

---

## 2. Branch Filters — Bộ Lọc Nhánh

### `only` — Chỉ Cho Phép

```yaml
workflows:
  deploy:
    jobs:
      - deploy-production:
          filters:
            branches:
              only:
                - main          # Chỉ chạy trên nhánh main
```

```yaml
# Nhiều nhánh
- deploy-staging:
    filters:
      branches:
        only:
          - main
          - develop
          - release
```

```yaml
# Dùng shorthand khi chỉ có một nhánh
- deploy-production:
    filters:
      branches:
        only: main    # Không cần dấu gạch đầu dòng
```

### `ignore` — Loại Trừ

```yaml
workflows:
  ci:
    jobs:
      - test:
          filters:
            branches:
              ignore:
                - gh-pages        # Không test nhánh documentation
                - dependabot/*    # Không test nhánh tự động của Dependabot
```

### Kết Hợp `only` và `ignore`

Có thể dùng cả hai, nhưng cần hiểu thứ tự ưu tiên:

```yaml
# ⚠️ Không nên kết hợp cả only và ignore — dễ gây nhầm lẫn
# Nếu phải dùng: only được ưu tiên trước, rồi ignore lọc tiếp trong tập only
```

Thực tế thường chỉ dùng **một trong hai** cho mỗi job.

---

## 3. Tag Filters — Bộ Lọc Nhãn

### Hành Vi Mặc Định với Tags

```yaml
# ❌ Job này SẼ KHÔNG chạy khi git push --tags
- deploy:
    # Không có filters → mặc định chỉ chạy khi push branch
    docker: ...
    steps: ...
```

Để Job chạy khi push tag, **bắt buộc** phải có `tags.only`:

```yaml
# ✅ Job này CHẠY khi push tag có format v1.2.3
- deploy-release:
    filters:
      tags:
        only: /^v[0-9]+\.[0-9]+\.[0-9]+$/    # Regex: v1.2.3, v10.0.1, ...
      branches:
        ignore: /.*/    # Không chạy khi push branch (chỉ chạy khi push tag)
```

### Chiến Lược Tag-Based Release — Phát Hành Theo Nhãn

```yaml
version: 2.1

jobs:
  build:
    docker: [{image: cimg/node:20.0}]
    steps:
      - checkout
      - run: npm ci && npm run build

  test:
    docker: [{image: cimg/node:20.0}]
    steps:
      - checkout
      - run: npm ci && npm test

  publish-npm:
    docker: [{image: cimg/node:20.0}]
    steps:
      - checkout
      - run: npm ci && npm run build
      - run:
          name: Publish lên npm registry
          command: |
            echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > ~/.npmrc
            npm publish

  publish-github-release:
    docker: [{image: cimg/base:stable}]
    steps:
      - checkout
      - run:
          name: Tạo GitHub Release
          command: |
            gh release create $CIRCLE_TAG \
              --title "Release $CIRCLE_TAG" \
              --notes-file CHANGELOG.md \
              dist/*

workflows:
  # Workflow thông thường: chạy khi push code
  ci:
    jobs:
      - build:
          filters:
            branches:
              only: /.*/    # Mọi branch
            tags:
              ignore: /.*/  # Không chạy khi push tag
      - test:
          requires: [build]
          filters:
            branches:
              only: /.*/
            tags:
              ignore: /.*/

  # Workflow release: chạy khi push tag v*
  release:
    jobs:
      # build và test PHẢI có filters vì là dependency của publish
      - build:
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/    # v1.0.0, v2.3.1, ...
            branches:
              ignore: /.*/    # Không chạy khi push branch
      - test:
          requires: [build]
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/
      - publish-npm:
          requires: [test]
          context: npm-publish
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/
      - publish-github-release:
          requires: [publish-npm]
          context: github-token
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/
```

---

## 4. Regex — Biểu Thức Chính Quy trong Filters

CircleCI hỗ trợ **POSIX Extended Regular Expressions** — Biểu Thức Chính Quy Mở Rộng POSIX trong filters.

### Ký Hiệu Regex

| Ký Hiệu | Nghĩa | Ví Dụ |
| ------- | ----- | ----- |
| `.*` | Bất kỳ ký tự nào, bất kỳ số lần | `feature/.*` khớp `feature/auth`, `feature/login` |
| `.+` | Bất kỳ ký tự, ít nhất 1 lần | `.+` khớp mọi chuỗi không rỗng |
| `^` | Bắt đầu chuỗi | `^v` khớp chuỗi bắt đầu bằng `v` |
| `$` | Kết thúc chuỗi | `main$` khớp chuỗi kết thúc bằng `main` |
| `[0-9]` | Ký tự số | `v[0-9]+` khớp `v1`, `v20` |
| `(a\|b)` | a hoặc b | `(main\|develop)` khớp `main` hoặc `develop` |
| `\\.` | Dấu chấm literal | `v1\\.0\\.0` khớp `v1.0.0` |
| `/pattern/` | Bọc trong `/` để CircleCI nhận dạng regex | `/^v.*/` |

> **Quan trọng:** Trong CircleCI filters, regex phải được bọc trong `/.../ `(dấu gạch chéo) để phân biệt với string literal.

### Ví Dụ Regex Thực Tế

```yaml
filters:
  branches:
    only:
      - main
      - develop
      - /^release\/.*/       # Khớp: release/1.0, release/2.3-hotfix
      - /^hotfix\/.*/        # Khớp: hotfix/critical-bug, hotfix/payment-fix
      - /^feature\/.*/       # Khớp: feature/auth, feature/new-ui

  tags:
    only:
      - /^v[0-9]+\.[0-9]+\.[0-9]+$/        # SemVer — Semantic Versioning: v1.2.3
      - /^v[0-9]+\.[0-9]+\.[0-9]+-rc\.[0-9]+$/  # Release Candidate: v1.2.3-rc.1
      - /^v[0-9]+\.[0-9]+\.[0-9]+-beta\.[0-9]+$/ # Beta: v1.2.3-beta.1
```

### So Sánh String vs Regex

```yaml
# String matching — khớp chính xác
filters:
  branches:
    only:
      - main          # Chỉ khớp "main", không khớp "main-old"
      - feature/auth  # Chỉ khớp đúng "feature/auth"

# Regex matching — khớp theo pattern
filters:
  branches:
    only:
      - /^main$/       # Regex tương đương với string "main"
      - /^feature\/.+/ # Khớp mọi nhánh bắt đầu bằng "feature/"
```

---

## 5. Quy Tắc Kế Thừa Filters

Đây là phần **dễ nhầm nhất** khi dùng filters.

### Quy Tắc: Tất Cả Job Trong Chuỗi `requires` Phải Có Filter Nhất Quán

```yaml
# ❌ SAI — deploy-staging không có filter nhưng build có
workflows:
  broken:
    jobs:
      - build:
          filters:
            branches:
              only: main    # build chỉ chạy trên main
      - deploy-staging:
          requires: [build]
          # Không có filter → mặc định chạy trên mọi branch
          # Nhưng build không chạy trên branch khác → deploy không bao giờ có input
```

```yaml
# ✅ ĐÚNG — filter nhất quán trong toàn bộ chuỗi
workflows:
  correct:
    jobs:
      - build:
          filters:
            branches:
              only: main
      - deploy-staging:
          requires: [build]
          filters:
            branches:
              only: main    # Phải lặp lại filter để nhất quán
```

### Quy Tắc với Tag-Based Workflow

Khi một job downstream (job sau) có `tags.only`, tất cả job upstream (job trước) trong `requires` cũng phải có `tags.only`:

```yaml
# ❌ SAI — publish có tag filter nhưng test không có
workflows:
  release:
    jobs:
      - test
          # Không có tag filter → test không chạy khi push tag
          # → publish không bao giờ có dependency thỏa mãn
      - publish:
          requires: [test]
          filters:
            tags:
              only: /^v.*/

# ✅ ĐÚNG — tất cả job trong chuỗi đều có tag filter
workflows:
  release:
    jobs:
      - test:
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/
      - publish:
          requires: [test]
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/
```

---

## 6. Chiến Lược Branching Phổ Biến

### Chiến Lược A: Git Flow

```
main ─────────────────────────────────────────► production
  └── develop ──────────────────────────────► staging
       ├── feature/login ──► merge to develop
       ├── feature/payment ──► merge to develop
       └── release/1.2.0 ──► merge to main & develop
```

```yaml
workflows:
  git-flow:
    jobs:
      # Feature branches: chỉ CI (lint + test)
      - lint:
          filters:
            branches:
              only: /^feature\/.*/
      - unit-test:
          requires: [lint]
          filters:
            branches:
              only: /^feature\/.*/

      # Develop: CI + deploy staging
      - lint:
          name: lint-develop
          filters:
            branches:
              only: develop
      - unit-test:
          name: test-develop
          requires: [lint-develop]
          filters:
            branches:
              only: develop
      - integration-test:
          requires: [test-develop]
          filters:
            branches:
              only: develop
      - deploy-staging:
          requires: [integration-test]
          context: staging-aws
          filters:
            branches:
              only: develop

      # Main: full CI + approve + deploy production
      - lint:
          name: lint-main
          filters:
            branches:
              only: main
      - test-all:
          requires: [lint-main]
          filters:
            branches:
              only: main
      - approve-production:
          type: approval
          requires: [test-all]
          filters:
            branches:
              only: main
      - deploy-production:
          requires: [approve-production]
          context: production-aws
          filters:
            branches:
              only: main
```

### Chiến Lược B: Trunk-Based Development — Phát Triển Dựa Trên Thân Cây

```
main (trunk) ────────────────────────────────► luôn deployable
  ├── feature/short-lived ──► merge nhanh
  └── v1.0.0 (tag) ──► release lên production
```

```yaml
workflows:
  trunk-based:
    jobs:
      # Mọi branch: CI đầy đủ
      - lint:
          filters:
            branches:
              only: /.*/
            tags:
              ignore: /.*/
      - test:
          requires: [lint]
          filters:
            branches:
              only: /.*/
            tags:
              ignore: /.*/

      # Main: auto-deploy staging
      - deploy-staging:
          requires: [test]
          filters:
            branches:
              only: main

      # Tags: release production
      - build-release:
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/
      - deploy-production:
          requires: [build-release]
          context: production-aws
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/
```

---

## 7. Mẫu Thực Tế

### Mẫu Hoàn Chỉnh: Multi-Environment Pipeline

```yaml
version: 2.1

executors:
  default:
    docker:
      - image: cimg/node:20.0
    resource_class: medium

jobs:
  lint:
    executor: default
    steps:
      - checkout
      - run: npm ci && npm run lint

  test:
    executor: default
    parallelism: 4
    steps:
      - checkout
      - run: npm ci
      - run:
          command: |
            circleci tests glob "**/*.test.ts" | \
            circleci tests split --split-by=timings | \
            xargs npx jest --ci
      - store_test_results:
          path: reports/

  build:
    executor: default
    steps:
      - checkout
      - run: npm ci && npm run build
      - persist_to_workspace:
          root: .
          paths: [dist/]

  deploy-dev:
    executor: default
    steps:
      - attach_workspace: {at: .}
      - run: ./scripts/deploy.sh dev

  deploy-staging:
    executor: default
    steps:
      - attach_workspace: {at: .}
      - run: ./scripts/deploy.sh staging

  deploy-production:
    executor: default
    steps:
      - attach_workspace: {at: .}
      - run: ./scripts/deploy.sh production

  publish-package:
    executor: default
    steps:
      - checkout
      - run: npm ci && npm run build
      - run:
          name: Publish lên npm
          command: npm publish --access public

workflows:
  # ─── Feature Branches ───────────────────────────────
  feature-ci:
    jobs:
      - lint:
          filters:
            branches:
              only: /^feature\/.*/
      - test:
          requires: [lint]
          filters:
            branches:
              only: /^feature\/.*/
      - build:
          requires: [test]
          filters:
            branches:
              only: /^feature\/.*/
      # Deploy lên dev environment — tự động
      - deploy-dev:
          requires: [build]
          context: dev-environment
          filters:
            branches:
              only: /^feature\/.*/

  # ─── Develop Branch ─────────────────────────────────
  develop-ci:
    jobs:
      - lint:
          filters:
            branches:
              only: develop
      - test:
          requires: [lint]
          filters:
            branches:
              only: develop
      - build:
          requires: [test]
          filters:
            branches:
              only: develop
      # Deploy staging — tự động
      - deploy-staging:
          requires: [build]
          context: staging-environment
          filters:
            branches:
              only: develop

  # ─── Main Branch ────────────────────────────────────
  main-ci:
    jobs:
      - lint:
          filters:
            branches:
              only: main
      - test:
          requires: [lint]
          filters:
            branches:
              only: main
      - build:
          requires: [test]
          filters:
            branches:
              only: main
      - deploy-staging:
          requires: [build]
          context: staging-environment
          filters:
            branches:
              only: main
      # Approval gate trước production
      - approve-production:
          type: approval
          requires: [deploy-staging]
          filters:
            branches:
              only: main
      - deploy-production:
          requires: [approve-production]
          context: production-environment
          filters:
            branches:
              only: main

  # ─── Tags (Release) ─────────────────────────────────
  tag-release:
    jobs:
      - lint:
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/
      - test:
          requires: [lint]
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/
      - publish-package:
          requires: [test]
          context: npm-registry
          filters:
            tags:
              only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
            branches:
              ignore: /.*/

  # ─── Hotfix Branches ────────────────────────────────
  hotfix-ci:
    jobs:
      - lint:
          filters:
            branches:
              only: /^hotfix\/.*/
      - test:
          requires: [lint]
          filters:
            branches:
              only: /^hotfix\/.*/
      - build:
          requires: [test]
          filters:
            branches:
              only: /^hotfix\/.*/
      # Hotfix cần approve cẩn thận hơn
      - approve-hotfix-deploy:
          type: approval
          requires: [build]
          filters:
            branches:
              only: /^hotfix\/.*/
      - deploy-production:
          requires: [approve-hotfix-deploy]
          context: production-environment
          filters:
            branches:
              only: /^hotfix\/.*/
```

---

## 8. Lỗi Thường Gặp

### Lỗi 1: Tag Push Không Trigger Job

```yaml
# ❌ SAI — job không có tags.only
- deploy:
    # Mặc định không chạy khi push tag

# ✅ ĐÚNG
- deploy:
    filters:
      tags:
        only: /^v.*/
      branches:
        ignore: /.*/    # Thường thêm ignore branches khi dùng tags.only
```

### Lỗi 2: Regex Không Hợp Lệ

```yaml
# ❌ SAI — thiếu dấu / bao quanh regex
- deploy:
    filters:
      branches:
        only: ^feature/.*    # Không có /.../ → CircleCI coi là string literal

# ✅ ĐÚNG
- deploy:
    filters:
      branches:
        only: /^feature\/.*/    # Có /.../ → CircleCI hiểu là regex
```

### Lỗi 3: Filter Không Nhất Quán Trong Chuỗi `requires`

```yaml
# ❌ SAI — test không có tag filter nhưng publish cần test
- test:
    # Không có filters.tags → không chạy khi push tag

- publish:
    requires: [test]    # test không bao giờ chạy khi push tag
    filters:
      tags:
        only: /^v.*/    # publish chờ test mãi mãi và không bao giờ chạy

# ✅ ĐÚNG — filter nhất quán
- test:
    filters:
      tags:
        only: /^v.*/
      branches:
        ignore: /.*/

- publish:
    requires: [test]
    filters:
      tags:
        only: /^v.*/
      branches:
        ignore: /.*/
```

### Lỗi 4: Dùng Cả `only` Và `ignore` Cùng Lúc

```yaml
# ⚠️ Dễ gây nhầm lẫn — chỉ dùng một trong hai
- deploy:
    filters:
      branches:
        only: [main, develop]
        ignore: [develop]    # develop vừa trong only vừa trong ignore — kết quả khó đoán
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Filters trong CircleCI workflow làm gì?**

> Filters kiểm soát khi nào một Job trong workflow được phép chạy, dựa trên tên nhánh (branch) hoặc nhãn (tag) của Git. Dùng `branches.only` để chỉ cho phép một số nhánh cụ thể, `branches.ignore` để loại trừ, `tags.only` để kích hoạt khi push tag. Filters là cơ chế cốt lõi để đảm bảo chỉ deploy production từ nhánh main và chỉ publish khi push release tag.

**Q: Tại sao Job không chạy khi tôi push git tag?**

> Mặc định, Job chỉ chạy khi push branch — không chạy khi push tag. Để Job chạy khi push tag, phải tường minh thêm `filters.tags.only` vào Job đó **và tất cả Job phụ thuộc** (trong chuỗi `requires`). Đây là lỗi phổ biến nhất khi cấu hình tag-based release pipeline.

**Q: Sự khác biệt giữa `branches.only` và `branches.ignore`?**

> `only` là whitelist — Job chỉ chạy trên các nhánh được liệt kê. `ignore` là blacklist — Job chạy trên mọi nhánh TRỪ các nhánh bị liệt kê. Dùng `only` khi biết chính xác nhánh nào được phép (ví dụ: deploy chỉ từ main). Dùng `ignore` khi muốn loại trừ một số nhánh đặc biệt (ví dụ: không test nhánh gh-pages hay dependabot).

**Q: Làm thế nào để dùng regex trong branch filters?**

> Bọc pattern trong dấu gạch chéo `/pattern/`. Ví dụ: `/^feature\/.*/` khớp mọi nhánh bắt đầu bằng `feature/`. CircleCI dùng POSIX Extended Regular Expressions. Không có dấu `/` thì chuỗi được coi là string literal (khớp chính xác).

**Q: Nếu muốn một Job chỉ chạy khi push tag VÀ không chạy khi push branch, cấu hình như thế nào?**

> Thêm cả `tags.only` và `branches.ignore: /.*/`:
> ```yaml
> filters:
>   tags:
>     only: /^v[0-9]+\.[0-9]+\.[0-9]+$/
>   branches:
>     ignore: /.*/    # Bỏ qua tất cả branches
> ```

---

## 📖 Đọc Tiếp

- [README.md](README.md) — Tổng quan module workflows
- [3-approval-jobs.md](3-approval-jobs.md) — Kết hợp filters với approval gates
- [4-scheduled-pipelines.md](4-scheduled-pipelines.md) — Filters trong scheduled pipelines
- [09-advanced/2-path-filtering.md](../09-advanced/2-path-filtering.md) — Path Filtering nâng cao cho monorepo

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Thuộc Module:** 03-workflows
