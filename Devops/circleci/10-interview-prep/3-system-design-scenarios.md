# 🏗️ System Design Scenarios — Thiết Kế CI/CD Trên CircleCI

> Bốn kịch bản thiết kế thường gặp khi phỏng vấn Senior DevOps / Platform Engineer. Mỗi kịch bản có requirements, sơ đồ ASCII, quyết định CircleCI cụ thể, và trade-offs.

---

## 📋 Framework Trả Lời (30–45 phút)

```
1. Clarify (3–5 phút)
   → Số repo? Monorepo? Cloud? Compliance? RTO deploy?

2. Success metrics
   → Thời gian PR feedback, tần suất deploy, % flaky, budget credit

3. High-level diagram
   → VCS → CircleCI → artifacts → deploy target

4. Drill-down CircleCI
   → Workflows, executors, cache, security, cost

5. Trade-offs & failure modes
   → Runner down, cache miss, secret leak, bad deploy rollback
```

---

## 🔴 Scenario 1: Startup SaaS — Node.js trên AWS ECS

### Requirements

```
- 8 engineers, GitHub, 50–100 PR/tuần
- Stack: Node 20, PostgreSQL, Redis
- Deploy: Docker → ECR → ECS Fargate
- Prod deploy: manual approval, staging auto on main
- Budget CI: hạn chế — ưu tiên cache & parallelism hợp lý
```

### Kiến Trúc CI/CD

```
GitHub PR/push
      │
      ▼
┌─────────────────────────────────────┐
│ CircleCI Workflow: ci-cd             │
│  ┌─────────┐  ┌─────────┐           │
│  │ lint    │  │ unit    │  parallel   │
│  └────┬────┘  └────┬────┘           │
│       └──────┬─────┘                │
│              ▼                      │
│         ┌─────────┐                 │
│         │ build   │ persist dist/   │
│         └────┬────┘   workspace     │
│              ▼                      │
│    ┌─────────────────┐              │
│    │ docker-build-push│ → ECR       │
│    └────────┬────────┘              │
│             ▼                       │
│  branch main: deploy-staging (auto) │
│  + approve-prod → deploy-prod       │
└─────────────────────────────────────┘
```

### Quyết Định CircleCI

| Chủ đề | Lựa chọn |
| ------ | -------- |
| Executor | Docker `cimg/node:20.x` |
| Secrets | Context `staging` / `prod`; prod dùng **OIDC** assume role |
| Cache | `npm` + Docker layer caching cho build image |
| Deploy | Orb `aws-ecr` + `aws-ecs` hoặc custom run với AWS CLI |
| Quality gate | Coverage threshold; fail job nếu dưới ngưỡng |

### Trade-offs

- **Approval** làm chậm release nhưng giảm rủi ro — phù hợp B2B SaaS.  
- **ECS rolling deploy** đơn giản hơn Blue/Green nhưng rollback chậm hơn — nêu rõ nếu interviewer hỏi HA.

**Ôn:** [07-integration/](../07-integration/)

---

## 🟡 Scenario 2: Monorepo 15 Microservices (TypeScript)

### Requirements

```
- Một repo, packages/* 
- Mỗi service có Dockerfile riêng
- PR: chỉ test/build service bị ảnh hưởng
- Nightly: full integration + contract tests
- Tránh “config hell” — cần template chuẩn
```

### Kiến Trúc

```
Push PR
   │
   ▼
Setup Workflow (path-filtering / dynamic config)
   │
   ├─► service A changed? → workflow fragment A
   ├─► service B changed? → workflow fragment B
   └─► none (docs only)? → minimal workflow (lint markdown)
   
Nightly (scheduled pipeline)
   └── full-matrix-all-services
```

### CircleCI Building Blocks

1. **Path Filtering orb** — set pipeline parameters `run-api`, `run-billing`, …  
2. **Dynamic Config** — `continue_config.yml` generated hoặc tĩnh theo matrix params  
3. **Reusable command** `service-pipeline` trong inline orb  
4. **Cache per package** — checksum `packages/<name>/package-lock.json`  
5. **Workspace** — chia artifact build chỉ khi có job deploy phụ thuộc  

### Điểm Phỏng Vấn

- Giải thích **false negative** nếu path map sai → thêm dependency graph hoặc nightly full.  
- **Cost governance:** giới hạn `parallelism` tổng org; priority queue cho `main`.

**Ôn:** [09-advanced/5-monorepo-strategy.md](../09-advanced/5-monorepo-strategy.md)

---

## 🟢 Scenario 3: Mobile + Backend — macOS + Docker

### Requirements

```
- iOS build cần macOS executor
- Backend API test trên Linux Docker
- Release train: tag `release/*` → TestFlight + API staging
- Certificates/provisioning profile quản lý an toàn
```

### Workflow Gợi Ý

```
workflows:
  mobile-release:
    jobs:
      - test-api          # docker
      - build-ios:
          requires: [test-api]
          filters:
            tags:
              only: /^release.*/
      - deploy-api-staging:
          requires: [test-api]
      - submit-testflight:
          requires: [build-ios]
          type: approval  # optional legal/compliance
```

### Bảo Mật

- Signing credentials trong **Context** Restricted; không commit `.p12`.  
- macOS resource đắt — chỉ chạy trên tag release, không mọi PR.

**Ôn:** [01-fundamentals/3-executors.md](../01-fundamentals/3-executors.md)

---

## 🔵 Scenario 4: Enterprise — Self-Hosted Runner + Compliance

### Requirements

```
- Code không được build trên shared cloud (policy)
- Kết nối on-prem artifact registry & K8s
- Audit: ai trigger deploy, image digest nào
- HA cho runner pool
```

### Kiến Trúc

```
CircleCI Cloud (control plane)
        │
        ▼
Self-Hosted Runner namespace (on-prem VM/K8s)
        │
        ├── build (no outbound except allowlist)
        ├── scan (Snyk/Trivy)
        └── deploy → internal K8s (GitOps optional)
```

### CircleCI Features

- **Self-Hosted Runner** — resource class `runner` + labels (`gpu`, `linux-amd64`)  
- **IP Ranges** (nếu hybrid) — whitelist firewall cho cloud jobs còn lại  
- **Audit Log** — export event approve/deploy  
- **OIDC** hạn chế; ưu tiên internal vault inject qua runner sidecar (nếu policy yêu cầu)

### Trade-offs

| | Cloud executor | Self-hosted |
| - | -------------- | ----------- |
| Ops | Thấp | Cao (patch, scale, monitor) |
| Compliance | Phụ thuộc vendor | Kiểm soát data residency |
| Tốc độ | Thường nhanh | Phụ thuộc hardware nội bộ |

**Ôn:** [09-advanced/4-self-hosted-runner.md](../09-advanced/4-self-hosted-runner.md), [06-security/4-audit-log.md](../06-security/4-audit-log.md)

---

## 📊 Bảng Tóm Tắt — Chọn Pattern

| Bài toán | Pattern chính |
| -------- | ------------- |
| PR nhanh, đơn repo | Parallel lint/test + cache + split tests |
| Monorepo | Path filtering + dynamic config + nightly full |
| Prod an toàn | Approval + branch/tag filter + OIDC |
| Compliance on-prem | Self-hosted runner + audit + restricted contexts |

---

## ✅ Sau Khi Vẽ Xong — Câu Hỏi Tự Kiểm

- Rollback deploy khi job `deploy` success nhưng app lỗi?  
- Làm sao reproduce build cũ (image digest / git SHA)?  
- Điều gì xảy ra khi CircleCI outage? (queue, mirror CI tạm thời?)  
- Ai được phép approve production?

---

**Cập Nhật Lần Cuối:** 2026-05-20
