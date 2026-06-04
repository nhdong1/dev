# 📋 Top 20 Câu Hỏi Phỏng Vấn CircleCI & CI/CD — Hướng Dẫn Đầy Đủ

> Bộ câu hỏi và đáp án mẫu bám sát nội dung knowledge base CircleCI trong repo này. Mỗi câu gồm điểm cần nói, follow-up thường gặp, và liên kết module để ôn sâu.

---

## 🏷️ Phân Loại

| Danh mục | Câu | Mức |
| -------- | --- | --- |
| CI/CD & kiến trúc CircleCI | 1–5 | Junior+ |
| Cấu hình `config.yml` | 6–10 | Mid |
| Workflow & tối ưu | 11–15 | Mid+ |
| Bảo mật & tích hợp | 16–18 | Senior |
| Nâng cao & vận hành | 19–20 | Senior+ |

---

## 🔵 CI/CD & Kiến Trúc

### Câu 1: CI/CD là gì? Phân biệt CI, CD (Delivery), CD (Deployment).

**Câu trả lời mẫu:**

- **CI — Continuous Integration (Tích Hợp Liên Tục):** Mỗi lần merge/push, code được build và test tự động để phát hiện lỗi sớm.
- **CD — Continuous Delivery (Triển Khai Liên Tục — sẵn sàng release):** Artifact luôn ở trạng thái có thể deploy; release có thể cần approval thủ công.
- **CD — Continuous Deployment (Triển Khai Tự Động):** Mọi commit đạt quality gate đều lên production tự động.

**CircleCI đóng vai trò:** Nền tảng chạy pipeline (build, test, scan, deploy) khi có sự kiện từ VCS (Version Control System — Hệ Thống Quản Lý Phiên Bản) như GitHub.

**Follow-up:** "Khi nào không nên Continuous Deployment?" → Regulated industry, mobile app store review, hoặc khi cần change window.

**Ôn thêm:** [01-fundamentals/1-cicd-concepts.md](../01-fundamentals/1-cicd-concepts.md)

---

### Câu 2: Giải thích Pipeline, Workflow, Job, Step trong CircleCI.

**Câu trả lời mẫu:**

```
Push/PR → Pipeline (một lần chạy cho một commit)
              └── Workflow (đồ thị jobs: tuần tự / song song)
                      └── Job (chạy trên một executor)
                              └── Step (checkout, run, cache, ...)
```

| Khái niệm | Ý nghĩa |
| --------- | ------- |
| **Pipeline** | Toàn bộ lần thực thi gắn với một trigger (push, schedule, API) |
| **Workflow** | Orchestration — điều phối thứ tự và điều kiện chạy job |
| **Job** | Đơn vị công việc trên executor (Docker, Machine, …) |
| **Step** | Lệnh/bước nhỏ trong job |

**Điểm gây ấn tượng:** Workflow có thể chạy nhiều job song song; Job không tự “biết” job khác trừ khi dùng `requires`, workspace, hoặc artifact bên ngoài.

**Ôn thêm:** [01-fundamentals/2-circleci-architecture.md](../01-fundamentals/2-circleci-architecture.md)

---

### Câu 3: Executor là gì? Khi nào dùng Docker vs Machine?

**Câu trả lời mẫu:**

**Executor — Môi Trường Thực Thi** quyết định *ở đâu* job chạy.

| Executor | Khi dùng |
| -------- | -------- |
| **Docker** | Đa số build/test; image nhẹ (`cimg/node`, `cimg/python`); khởi động nhanh |
| **Machine** | Cần VM đầy đủ, Docker-in-Docker phức tạp, kernel đặc biệt |
| **macOS** | Build iOS/macOS |
| **Windows** | .NET native trên Windows |

**Trade-off:** Docker nhanh và rẻ hơn; Machine linh hoạt hơn nhưng chậm và tốn credit hơn.

**Follow-up:** "Resource class?" → Chọn CPU/RAM (`medium`, `large`, …) ảnh hưởng tốc độ và chi phí.

**Ôn thêm:** [01-fundamentals/3-executors.md](../01-fundamentals/3-executors.md), [4-resource-classes.md](../01-fundamentals/4-resource-classes.md)

---

### Câu 4: CircleCI hoạt động end-to-end khi bạn push code?

**Câu trả lời mẫu:**

1. VCS gửi webhook tới CircleCI.
2. CircleCI đọc `.circleci/config.yml` (và có thể Dynamic Config — Cấu Hình Động).
3. Tạo **Pipeline**, compile **Workflow**.
4. Scheduler cấp phát **runner** (cloud hoặc self-hosted).
5. Mỗi **Job** chạy container/VM, thực thi **Steps**.
6. Kết quả: status check trên PR, artifacts, notifications.

**Điểm kỹ thuật:** Config validate có thể chạy local qua **CircleCI CLI** trước khi push.

---

### Câu 5: Khi nào chọn CircleCI thay vì Jenkins hoặc GitHub Actions?

**Câu trả lời mẫu (tóm tắt):**

| Chọn CircleCI khi | Chọn Jenkins khi | Chọn GitHub Actions khi |
| ----------------- | ---------------- | ------------------------- |
| Muốn SaaS, ít vận hành server | Cần on-premise, plugin đặc thù | Repo đã trên GitHub, workflow đơn giản |
| Cần test splitting, Insights mạnh | Compliance bắt buộc self-host | Public OSS, budget hạn chế |

**Không trả lời chung chung:** Gắn với team size, compliance, VCS, và thời gian build hiện tại.

**Ôn thêm:** [4-tool-comparison.md](./4-tool-comparison.md), [01-fundamentals/5-circleci-vs-alternatives.md](../01-fundamentals/5-circleci-vs-alternatives.md)

---

## 🟢 Cấu Hình `config.yml`

### Câu 6: `version: 2.1` mang lại gì so với 2.0?

**Câu trả lời mẫu:**

- **Orbs**, **commands**, **executors** tái sử dụng
- **Workflow** với `when`/`unless` (kết hợp pipeline parameters)
- **Matrix** (qua parameters + workflow)
- Cấu trúc modular, ít copy-paste

Luôn dùng **2.1** cho project mới trừ khi bị ràng buộc legacy.

**Ôn thêm:** [02-configuration/1-yaml-syntax.md](../02-configuration/1-yaml-syntax.md)

---

### Câu 7: Làm sao tái sử dụng cấu hình? Commands vs Orbs.

**Câu trả lời mẫu:**

| Cơ chế | Phạm vi | Khi dùng |
| ------ | ------- | -------- |
| **YAML anchors / aliases** | Một file | Snippet nhỏ, nhanh |
| **Commands** (trong project) | Project | Steps lặp lại nội bộ |
| **Executors** (trong project) | Project | Cùng image/resource class |
| **Orbs** | Registry / inline | Tích hợp AWS, Docker, Slack; chia sẻ giữa repo |

**Orbs** = package versioned; **commands** = function nội bộ không publish.

**Ôn thêm:** [02-configuration/3-commands.md](../02-configuration/3-commands.md), [04-orbs/](../04-orbs/)

---

### Câu 8: Parameters trong CircleCI — pipeline vs job.

**Câu trả lời mẫu:**

- **Pipeline parameters:** Truyền khi trigger (API, setup workflow, path-filtering). Dùng để bật/tắt workflow hoặc chọn môi trường deploy.
- **Job parameters:** Template hóa job (image tag, flag).
- **Command parameters:** Tham số cho steps tái sử dụng.

Ví dụ path-filtering set `run-api: true` → workflow chỉ chạy job API.

**Ôn thêm:** [02-configuration/4-parameters.md](../02-configuration/4-parameters.md)

---

### Câu 9: Environment variables — built-in, project, context.

**Câu trả lời mẫu:**

| Loại | Ví dụ | Ghi chú |
| ---- | ----- | ------- |
| Built-in | `CIRCLE_BRANCH`, `CIRCLE_SHA1` | Tự inject mỗi job |
| Project | Settings → Environment Variables | Tiện nhưng ít phân quyền |
| **Context** | Nhóm secret theo team/env | **Nên dùng cho secrets production** |

**Best practice:** Secrets nhạy cảm trong **Context** + Restricted Context; không echo secret trong log.

**Ôn thêm:** [02-configuration/5-environment-variables.md](../02-configuration/5-environment-variables.md), [06-security/1-contexts.md](../06-security/1-contexts.md)

---

### Câu 10: Các built-in steps quan trọng nhất?

**Câu trả lời mẫu:**

- `checkout` — clone repo
- `run` — shell command
- `restore_cache` / `save_cache`
- `persist_to_workspace` / `attach_workspace`
- `store_artifacts` / `store_test_results`
- `setup_remote_docker` — build image trong Docker executor

**Follow-up:** "Khác `run` với orb command?" → Orb gói nhiều steps + defaults đã kiểm chứng.

**Ôn thêm:** [02-configuration/2-jobs-and-steps.md](../02-configuration/2-jobs-and-steps.md)

---

## 🟡 Workflow & Tối Ưu

### Câu 11: `requires` và approval job hoạt động thế nào?

**Câu trả lời mẫu:**

```yaml
workflows:
  release:
    jobs:
      - test
      - build:
          requires: [test]
      - hold:
          type: approval
          requires: [build]
      - deploy-prod:
          requires: [hold]
```

- **`requires`:** Job chỉ chạy khi dependency **success** (trừ khi cấu hình khác).
- **`type: approval`:** Cổng duyệt thủ công trên UI — phù hợp production deploy.

**Follow-up:** Filter theo branch (`filters.branches.only: main`) để approval chỉ trên nhánh release.

**Ôn thêm:** [03-workflows/](../03-workflows/)

---

### Câu 12: Cache key nên thiết kế ra sao?

**Câu trả lời mẫu:**

Pattern phổ biến:

```yaml
keys:
  - deps-v1-{{ checksum "package-lock.json" }}
  - deps-v1-
```

- **Checksum file lock** → cache invalid khi dependency đổi.
- **Fallback key** (`deps-v1-`) → partial restore tốt hơn không có gì.
- Tránh cache quá rộng (key cố định) → dữ liệu stale.
- Tránh key quá hẹp → miss liên tục, không tiết kiệm thời gian.

**Ôn thêm:** [05-optimization/1-caching-strategies.md](../05-optimization/1-caching-strategies.md)

---

### Câu 13: Workspace khác Cache ở điểm nào?

**Câu trả lời mẫu:**

| | **Cache** | **Workspace** |
| - | --------- | ------------- |
| Mục đích | Tăng tốc (deps, build tool) | **Chia sẻ artifact** giữa jobs trong cùng workflow |
| Vòng đời | Best-effort, có thể miss | Gắn workflow run |
| Ví dụ | `~/.npm`, `~/.cache` | `dist/`, binary đã build |

**Rule of thumb:** Dependencies → cache; output cần job sau dùng ngay → workspace.

**Ôn thêm:** [05-optimization/2-workspace.md](../05-optimization/2-workspace.md)

---

### Câu 14: Parallelism và test splitting?

**Câu trả lời mẫu:**

- **`parallelism: N`:** CircleCI tạo N container; mỗi container có `CIRCLE_NODE_INDEX`.
- **`circleci tests split`:** Chia danh sách test theo file hoặc **timings** (lịch sử thời gian chạy) để cân bằng tải.

```bash
circleci tests glob "**/*.spec.js" | \
  circleci tests split --split-by=timings | \
  xargs npm test
```

**Điểm senior:** Sau vài lần chạy, `--split-by=timings` thường tốt hơn chia đều file.

**Ôn thêm:** [05-optimization/3-test-splitting.md](../05-optimization/3-test-splitting.md), [4-parallelism.md](../05-optimization/4-parallelism.md)

---

### Câu 15: Bạn đã tối ưu pipeline chậm như thế nào? (Câu hành vi)

**Khung trả lời (STAR ngắn):**

1. **Đo:** Pipeline Insights — duration theo job/step; xác định job chiếm >50% thời gian.
2. **Cache/workspace:** Lock file checksum; cache Docker layers nếu build image.
3. **Song song:** Tách lint/unit/integration; parallelism + test split.
4. **Resource class:** Chỉ tăng khi CPU-bound đã chứng minh.
5. **Kết quả số:** Ví dụ "từ 28 phút xuống 11 phút (−61%)".

**Ôn thêm:** [05-optimization/5-pipeline-insights.md](../05-optimization/5-pipeline-insights.md), [2-star-stories.md](./2-star-stories.md)

---

## 🔴 Bảo Mật & Tích Hợp

### Câu 16: Contexts vs OIDC — khi nào dùng gì?

**Câu trả lời mẫu:**

- **Context:** Lưu AWS keys, tokens trong CircleCI; inject vào job; cần **rotation** và phân quyền Restricted Context.
- **OIDC — OpenID Connect:** CircleCI phát JWT ngắn hạn; cloud provider (AWS IAM, GCP, Azure) trust và cấp role tạm — **không lưu long-lived access key** trên CI.

**Ưu tiên OIDC** cho AWS/GCP/Azure deploy khi org cho phép.

**Ôn thêm:** [06-security/2-oidc-integration.md](../06-security/2-oidc-integration.md)

---

### Câu 17: Deploy lên AWS từ CircleCI — các bước chính?

**Câu trả lời mẫu:**

1. Build & test trên Docker executor.
2. Build/push image → **ECR — Elastic Container Registry** (orb `circleci/aws-ecr` hoặc CLI).
3. Deploy → **ECS**, **EKS**, **Lambda**, hoặc **S3** + CloudFront tùy kiến trúc.
4. Credentials: Context hoặc **OIDC** assume role.
5. Production: workflow có **approval** + branch filter `main`.

**Ôn thêm:** [07-integration/3-aws-integration.md](../07-integration/3-aws-integration.md)

---

### Câu 18: Làm sao debug pipeline fail trên CircleCI?

**Câu trả lời mẫu:**

1. Đọc log step fail; kiểm tra exit code.
2. **Rerun with SSH** — SSH vào container đang chạy (môi trường giống CI).
3. Reproduce local: `circleci local execute` (giới hạn, chủ yếu Docker job đơn giản).
4. OOM → tăng resource class hoặc giảm parallelism memory.
5. Flaky test → Insights, rerun, tách test không ổn định.

**Ôn thêm:** [08-monitoring/](../08-monitoring/)

---

## 🟣 Nâng Cao

### Câu 19: Dynamic Config là gì? Khi nào cần?

**Câu trả lời mẫu:**

**Dynamic Config — Cấu Hình Động** gồm hai giai đoạn:

1. **Setup workflow** — chạy nhanh, phân tích thay đổi (path, branch, params).
2. **`continuation/continue`** — nạp `config.yml` thứ hai hoặc generated config cho pipeline thực.

**Khi cần:** Monorepo lớn, tránh chạy 15 service khi chỉ sửa README; pipeline khác nhau cho hotfix vs feature.

**Ôn thêm:** [09-advanced/1-dynamic-config.md](../09-advanced/1-dynamic-config.md), [2-path-filtering.md](../09-advanced/2-path-filtering.md)

---

### Câu 20: Thiết kế CI/CD cho monorepo — các điểm chính?

**Câu trả lời mẫu:**

- **Path Filtering** — map `services/api/**` → job API.
- **Dynamic Config** — generate workflow theo danh sách service thay đổi.
- **Matrix jobs** — test nhiều version Node/Python song song.
- **Shared orbs/commands** — chuẩn hóa build giữa team.
- **Cost:** Giảm credit bằng không chạy job không liên quan; cache theo từng package lock trong từng service.

**Trade-off:** Độ phức tạp config tăng → cần test config (validate) và tài liệu nội bộ.

**Ôn thêm:** [09-advanced/5-monorepo-strategy.md](../09-advanced/5-monorepo-strategy.md)

---

## ✅ Checklist Ôn Nhanh (Ngày Trước Phỏng Vấn)

- [ ] Vẽ được Pipeline → Workflow → Job → Step
- [ ] Giải thích cache key + workspace trong 1 phút
- [ ] Một ví dụ workflow có approval + deploy
- [ ] So sánh CircleCI vs GHA vs Jenkins trong 2 phút
- [ ] Một STAR story có số liệu cụ thể

---

**Cập Nhật Lần Cuối:** 2026-05-20
