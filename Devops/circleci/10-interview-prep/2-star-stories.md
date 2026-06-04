# 🌟 STAR Stories — Câu Chuyện Incident & Thành Tựu CI/CD

> Mẫu kể chuyện theo **STAR** (Situation — Tình Huống / Task — Nhiệm Vụ / Action — Hành Động / Result — Kết Quả) cho phỏng vấn DevOps, Platform, hoặc Backend có trọng tâm CircleCI. Điền số liệu thật từ dự án của bạn trước khi phỏng vấn.

---

## 📖 Phương Pháp STAR

```
S — Situation: Bối cảnh ngắn (team, scale, áp lực)
T — Task: Trách nhiệm cụ thể của BẠN
A — Action: Các bước kỹ thuật chi tiết (chiếm ~60% thời gian nói)
R — Result: Số liệu + bài học + cải tiến lâu dài
```

**Nguyên tắc:** Dùng "tôi" cho phần bạn trực tiếp làm; nêu rõ công cụ (CircleCI, cache key, OIDC, …).

---

## 📋 Câu Hỏi Thường Cần Story

1. Pipeline chậm / build queue / chi phí credit tăng đột biến  
2. Production deploy sai hoặc gần deploy nhầm — cổng approval / process  
3. Secret lộ hoặc audit bảo mật CI  
4. Migration CI (Jenkins → CircleCI hoặc ngược lại)  
5. Monorepo: CI chạy quá nhiều job không cần thiết  

---

## 🔴 Story 1: Giảm Thời Gian Pipeline (Performance)

### Kịch bản mẫu: Suite test 25 phút → bottleneck release

**S — Situation:**

> "Team 12 engineer, microservices Node.js trên CircleCI. Mỗi PR chạy pipeline ~25 phút; dev phàn nàn merge chậm, và credit CircleCI vượt budget 30% so với quý trước."

**T — Task:**

> "Tôi được giao lead initiative giảm thời gian feedback PR xuống dưới 12 phút trong 3 sprint, không giảm coverage test."

**A — Action:**

> 1. **Pipeline Insights:** Job `integration-test` chiếm 68% thời gian; step `npm ci` lặp 4 lần giữa các job.  
> 2. **Cache:** Thống nhất key `npm-v2-{{ checksum "package-lock.json" }}` + fallback; bỏ cache key theo branch.  
> 3. **Workflow:** Tách `lint` + `unit` song song; `integration` chỉ chạy khi path `src/**` hoặc `api/**` đổi (path filter đơn giản bằng conditional workflow / sau này path-filtering orb).  
> 4. **Parallelism:** `parallelism: 6` + `circleci tests split --split-by=timings` cho Jest.  
> 5. **Resource class:** Chỉ nâng job integration từ `medium` → `large` sau khi profiling CPU >85%.  
> 6. **Pre-commit:** `circleci config validate` trong hook.

**R — Result:**

> "Thời gian pipeline PR trung bình **25 phút → 10 phút (−60%)**; credit tháng sau **−22%**. Document playbook trong wiki; thêm alert khi job >15 phút p95."

**Follow-up:** "Làm sao tránh cache poison?" → Checksum lock file; bump prefix `npm-v2` khi đổi Node major.

---

## 🟠 Story 2: OOM (Out Of Memory) & Pipeline Fail

### Kịch bản: Job Docker build fail exit 137

**S — Situation:**

> "Pipeline build Docker image cho service Java fail ngẫu nhiên trên nhánh `main`, chặn release Friday."

**T — Task:**

> "On-call DevOps: khôi phục pipeline và ngăn tái diễn trong 24h."

**A — Action:**

> 1. Log: `Killed` / exit **137** → OOM trên executor.  
> 2. **Rerun with SSH:** Xác nhận `docker build` peak RAM > 4GB trên `medium` (4GB).  
> 3. Tăng **resource class** lên `large` cho job `docker-build` only.  
> 4. Tối ưu Dockerfile: multi-stage build, giảm layer cache bust.  
> 5. **setup_remote_docker** với `docker_layer_caching: true` (nếu plan hỗ trợ).  
> 6. Thêm doc: matrix resource class theo loại job.

**R — Result:**

> "Build ổn định **100%** 2 tuần sau; thời gian build giảm thêm 4 phút nhờ layer caching. Runbook 'Exit 137' cho team."

**Ôn:** [08-monitoring/3-common-errors.md](../08-monitoring/3-common-errors.md)

---

## 🟡 Story 3: Secret Trong Log & Hardening

### Kịch bản: Token AWS xuất hiện trong build log

**S — Situation:**

> "Security scan phát hiện chuỗi giống AWS access key trong artifact log CircleCI từ job deploy staging."

**T — Task:**

> "Tôi chủ trì remediation trong 48h và chuẩn hóa secret handling cho 40 repos."

**A — Action:**

> 1. Rotate key ngay; revoke key cũ trên IAM.  
> 2. Chuyển secrets sang **Context** `staging-aws` với Restricted Context (chỉ nhóm Platform).  
> 3. Bỏ `echo $AWS_SECRET`; dùng orb `aws-cli` không in credential.  
> 4. Bật **OIDC** cho deploy staging/prod — xóa static key khỏi Context prod.  
> 5. Pre-commit: cấm regex AWS key trong repo; `gitleaks` trên PR.  
> 6. Training 30 phút cho team về masked env vars.

**R — Result:**

> "Audit pass lại; **0** secret trong log 90 ngày sau. Thời gian onboard repo mới giảm nhờ template orb + Context chuẩn."

**Ôn:** [06-security/](../06-security/)

---

## 🟢 Story 4: Approval Gate Cứu Production

### Kịch bản: Nhầm branch deploy

**S — Situation:**

> "Incident nhỏ: job deploy staging chạy nhầm config trỏ production bucket (may không ghi đè data nhờ versioning S3)."

**T — Task:**

> "Thiết kế lại workflow deploy an toàn trên CircleCI."

**A — Action:**

> 1. Tách Context `prod` / `staging`; job prod chỉ attach Context prod.  
> 2. Workflow: `deploy-prod` **requires** job `approve-prod` (`type: approval`).  
> 3. **Branch filter:** `deploy-prod` chỉ `main` + tag `v*`.  
> 4. Parameter `deploy_target` bắt buộc từ pipeline API cho manual deploy.  
> 5. Slack notification orb: ai approve, commit SHA nào.

**R — Result:**

> "Không còn deploy prod không chủ ý; thời gian release tăng ~15 phút do approval — chấp nhận được với compliance."

**Ôn:** [03-workflows/3-approval-jobs.md](../03-workflows/3-approval-jobs.md)

---

## 🔵 Story 5: Monorepo — Chỉ CI Service Thay Đổi

### Kịch bản: 18 packages, mọi PR chạy full matrix

**S — Situation:**

> "Monorepo TypeScript; mỗi commit chạy 18 bộ test → 45 phút, queue runner, dev bỏ qua CI đỏ."

**T — Task:**

> "Giảm job chạy trung bình mỗi PR xuống ≤6 job nhưng vẫn đảm bảo service bị ảnh hưởng được test đủ."

**A — Action:**

> 1. Áp dụng orb **path-filtering**: map `packages/billing/**` → `run-billing`.  
> 2. **Dynamic Config:** setup job generate `generated_config.yml` với danh sách workflow con.  
> 3. Shared **inline orb** cho `install-and-test` với parameter `package_path`.  
> 4. Cache key per package: `billing-deps-{{ checksum "packages/billing/package-lock.json" }}`.  
> 5. Nightly scheduled pipeline chạy full regression.

**R — Result:**

> "PR trung bình **45 phút → 12 phút**; nightly bắt regression cross-package. Credit **−35%** quý đó."

**Ôn:** [09-advanced/](../09-advanced/)

---

## 📝 Template Trống — Điền Dự Án Của Bạn

```markdown
### Story: [Tiêu đề ngắn]

**S:** [Team, hệ thống, hậu quả nếu không xử lý]

**T:** [Vai trò bạn — owner hay contributor]

**A:**
1. 
2. 
3. 

**R:** [Số liệu: thời gian, %, tiền, MTTR, số incident]

**Bài học:**
- 
```

---

## 🎤 Luyện Nói

- Mỗi story **2–3 phút** (không quá 5).  
- Ghi âm; loại bỏ "ờ", "team em làm" mơ hồ.  
- Chuẩn bị 1 follow-up kỹ thuật cho mỗi story (cache, OIDC, approval, path filter).

---

**Cập Nhật Lần Cuối:** 2026-05-20
