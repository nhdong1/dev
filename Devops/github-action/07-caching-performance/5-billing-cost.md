# 5. Billing & Cost — Tính Phí và Tối Ưu Chi Phí GitHub Actions

> Hiểu cách GitHub tính phí, tính toán chi phí dự án, và áp dụng các kỹ thuật tiết kiệm để tối ưu ngân sách CI/CD.

---

## 📚 Mục Lục

1. [Mô Hình Tính Phí GitHub Actions](#mô-hình-tính-phí)
2. [Tính Phí Minutes — Chi Tiết](#tính-phí-minutes)
3. [Tính Phí Storage — Artifacts và Cache](#tính-phí-storage)
4. [Ước Tính Chi Phí Thực Tế](#ước-tính-chi-phí-thực-tế)
5. [Giám Sát Usage — Theo Dõi Sử Dụng](#giám-sát-usage)
6. [Chiến Lược Tiết Kiệm Chi Phí](#chiến-lược-tiết-kiệm-chi-phí)
7. [Self-Hosted Runners vs GitHub-hosted](#self-hosted-vs-github-hosted)
8. [Billing Cho Organizations](#billing-cho-organizations)

---

## Mô Hình Tính Phí

### Free Tier (Gói Miễn Phí)

| Plan | Minutes/Tháng | Storage | Private Repos |
|---|---|---|---|
| **GitHub Free** | 2,000 phút | 500 MB | ✅ Có giới hạn |
| **GitHub Pro** | 3,000 phút | 1 GB | ✅ Có giới hạn |
| **GitHub Team** | 3,000 phút | 2 GB | ✅ Có giới hạn |
| **GitHub Enterprise** | 50,000 phút | 50 GB | ✅ Có giới hạn |

**Lưu ý quan trọng:**
- Public repositories: **MIỄN PHÍ HOÀN TOÀN** — không giới hạn minutes
- Minutes được tính lại vào đầu mỗi tháng (không cộng dồn)
- Khi hết minutes → workflows vẫn chạy nhưng bị tính phí theo pay-as-you-go

### Pay-as-you-go (Trả Theo Sử Dụng)

Sau khi vượt free tier:

| Runner OS | Giá/Phút |
|---|---|
| **Linux** (ubuntu-latest) | $0.008 |
| **Windows** | $0.016 (2× Linux) |
| **macOS** | $0.08 (10× Linux) |

---

## Tính Phí Minutes — Chi Tiết

### Cách GitHub Tính Phí

```
1. Làm tròn lên đến phút gần nhất
   → Job chạy 61 giây = tính 2 phút

2. Nhân với hệ số OS
   → 2 phút Linux × 1 = 2 phút billing
   → 2 phút Windows × 2 = 4 phút billing
   → 2 phút macOS × 10 = 20 phút billing

3. Tính riêng cho từng job
   → 3 jobs chạy song song × 5 phút = 15 phút billing (không phải 5)
```

### Hệ Số Nhân Theo OS

| OS | Hệ Số | Ví Dụ (5 phút thực) |
|---|---|---|
| `ubuntu-latest` | ×1 | 5 phút billing |
| `ubuntu-22.04` | ×1 | 5 phút billing |
| `windows-latest` | ×2 | 10 phút billing |
| `macos-latest` | ×10 | 50 phút billing |
| `macos-13` | ×10 | 50 phút billing |
| Larger runners (4 vCPU) | ×2 | 10 phút billing |
| Larger runners (8 vCPU) | ×4 | 20 phút billing |

### Ví Dụ Tính Chi Phí Workflow

```
Workflow: CI cho Node.js application

Jobs (chạy song song):
  ┌─ lint    : 2 phút Linux    → 2 phút billing
  ├─ test    : 4 phút Linux    → 4 phút billing
  └─ type-check: 3 phút Linux  → 3 phút billing

Job build (sau khi lint + test hoàn thành):
  └─ build   : 2 phút Linux    → 2 phút billing

Tổng: 2 + 4 + 3 + 2 = 11 phút billing per run
```

```
Chi phí nếu vượt free tier:
  11 phút × $0.008 = $0.088 per run

100 runs/ngày × $0.088 = $8.80/ngày = ~$264/tháng
```

---

## Tính Phí Storage — Artifacts và Cache

### Storage Được Tính Như Thế Nào

- **Tính theo GB-ngày:** 1 GB artifact giữ 30 ngày = 30 GB-ngày
- **Miễn phí theo plan** (xem bảng trên)
- **Vượt mức:** $0.25/GB/tháng (tính tại thời điểm viết tài liệu này)

### Tính Chi Phí Storage Thực Tế

```
Ví dụ: Dự án với 50 runs/ngày, mỗi run tạo:
  - Build artifacts: 100 MB, giữ 30 ngày
  - Test reports: 5 MB, giữ 7 ngày
  - Coverage: 2 MB, giữ 7 ngày

Tính toán:
  Build artifacts: 50 runs × 100MB × 30 ngày / 1024 = 146 GB-tháng
  Test reports:    50 runs × 5MB × 7 ngày / 1024   = 1.7 GB-tháng
  Coverage:        50 runs × 2MB × 7 ngày / 1024   = 0.7 GB-tháng

  Tổng: ~148 GB-tháng

Với GitHub Team (2GB free storage):
  Storage tính phí: 148 - 2 = 146 GB
  Chi phí storage: 146 × $0.25 = $36.5/tháng
```

### Cache Không Tính Phí Riêng

Cache **không** bị tính vào storage billing theo cách thông thường — nó thuộc giới hạn 10GB per repository và bị xóa tự động sau 7 ngày không dùng.

---

## Ước Tính Chi Phí Thực Tế

### Template Tính Chi Phí Hàng Tháng

```
=== Ước Tính Chi Phí GitHub Actions ===

Input:
  Số runs/ngày:                    [A]
  Phút trung bình mỗi run (Linux): [B]
  Phút trung bình mỗi run (macOS): [C] (nếu có)
  Ngày/tháng:                       30

Minutes billing/tháng:
  Linux: [A] × [B] × 30 = [D] phút
  macOS: [A] × [C] × 30 × 10 = [E] phút (hệ số 10)
  
Tổng: [D] + [E] = [F] phút/tháng

Sau khi trừ free tier [2000-50000 phút]:
  Phút tính phí: [F] - free_tier = [G]

Chi phí minutes: [G] × $0.008 = $[X]/tháng

Storage:
  Artifact size/run × runs/ngày × retention_days = [H] GB-tháng
  Chi phí storage: max(0, [H] - free_storage) × $0.25 = $[Y]/tháng

TỔNG: $[X] + $[Y] = $[Z]/tháng
```

### Ví Dụ Startup với GitHub Team Plan

```
Input:
  5 developers, ~20 PRs/ngày, 5 pushes lên main/ngày
  CI run (Linux): 8 phút trung bình
  CD run (macOS build): 0 (không có)
  Runs/ngày: 25

Minutes/tháng:
  Linux: 25 × 8 × 30 = 6,000 phút

Free tier: 3,000 phút
Phút tính phí: 6,000 - 3,000 = 3,000 phút

Chi phí minutes: 3,000 × $0.008 = $24/tháng

Storage:
  50MB artifacts × 25 runs × 14 ngày = 17.5 GB-tháng
  Chi phí storage: max(0, 17.5 - 2) × $0.25 = $3.87/tháng

TỔNG: $24 + $3.87 ≈ $28/tháng
```

### Ví Dụ Enterprise Với iOS (macOS Runners)

```
iOS app: 30 builds/ngày, mỗi build 20 phút trên macOS

Minutes/tháng:
  macOS: 30 × 20 × 30 × 10 (hệ số) = 180,000 phút billing

Phút tính phí (trừ 50,000 free):
  130,000 × $0.008 = $1,040/tháng chỉ riêng macOS!

→ Đây là lý do iOS teams thường dùng self-hosted macOS runners
```

---

## Giám Sát Usage — Theo Dõi Sử Dụng

### GitHub UI — Xem Usage Tháng Này

```
Settings → Billing and plans → Usage this month
→ Hiển thị: minutes used, minutes remaining, storage used
```

### GitHub API — Lấy Billing Data

```bash
# Lấy billing cho user
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/users/$USERNAME/settings/billing/actions"

# Lấy billing cho organization
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/orgs/$ORG/settings/billing/actions"
```

**Response:**

```json
{
  "total_minutes_used": 1234,
  "total_paid_minutes_used": 234,
  "included_minutes": 3000,
  "minutes_used_breakdown": {
    "UBUNTU": 1100,
    "MACOS": 50,
    "WINDOWS": 84
  }
}
```

### Workflow Để Giám Sát Chi Phí

```yaml
name: Báo Cáo Chi Phí Hàng Tuần

on:
  schedule:
    - cron: '0 9 * * 1'    # Mỗi thứ Hai lúc 9:00 AM UTC

jobs:
  cost-report:
    runs-on: ubuntu-latest
    steps:
      - name: Lấy billing data
        id: billing
        run: |
          RESPONSE=$(curl -s \
            -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
            "https://api.github.com/orgs/${{ github.repository_owner }}/settings/billing/actions")
          
          TOTAL=$(echo $RESPONSE | jq '.total_minutes_used')
          PAID=$(echo $RESPONSE | jq '.total_paid_minutes_used')
          UBUNTU=$(echo $RESPONSE | jq '.minutes_used_breakdown.UBUNTU')
          MACOS=$(echo $RESPONSE | jq '.minutes_used_breakdown.MACOS // 0')
          WINDOWS=$(echo $RESPONSE | jq '.minutes_used_breakdown.WINDOWS // 0')
          
          echo "total=$TOTAL" >> $GITHUB_OUTPUT
          echo "paid=$PAID" >> $GITHUB_OUTPUT
          
          echo "### GitHub Actions Usage Tuần Này" >> $GITHUB_STEP_SUMMARY
          echo "| OS | Minutes |" >> $GITHUB_STEP_SUMMARY
          echo "|---|---|" >> $GITHUB_STEP_SUMMARY
          echo "| Linux | $UBUNTU |" >> $GITHUB_STEP_SUMMARY
          echo "| macOS | $MACOS |" >> $GITHUB_STEP_SUMMARY
          echo "| Windows | $WINDOWS |" >> $GITHUB_STEP_SUMMARY
          echo "| **Tổng** | **$TOTAL** |" >> $GITHUB_STEP_SUMMARY
          echo "| Phí thêm | $PAID phút |" >> $GITHUB_STEP_SUMMARY
```

### Thiết Lập Spending Limits (Giới Hạn Chi Tiêu)

```
GitHub → Settings → Billing → Spending Limits

Đặt:
  Spending limit: $X/tháng
  → GitHub sẽ dừng workflows khi đạt giới hạn
  → Thay vì charge không giới hạn
```

---

## Chiến Lược Tiết Kiệm Chi Phí

### 1. Tối Ưu Minutes (Tác Động Lớn Nhất)

```yaml
# Kỹ thuật 1: Dependency caching (tiết kiệm 2-5 phút/run)
- uses: actions/setup-node@v4
  with:
    cache: 'npm'

# Kỹ thuật 2: Path filters (bỏ qua 60-90% triggers không cần thiết)
on:
  push:
    paths: ['src/**', 'package*.json']
    paths-ignore: ['**.md', 'docs/**']

# Kỹ thuật 3: Hủy runs cũ (tiết kiệm minutes của runs bị superseded)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Kỹ thuật 4: Skip CI cho commits không cần test
# [skip ci] trong commit message

# Kỹ thuật 5: Parallel jobs (giảm wall-clock time, nhưng KHÔNG giảm minutes)
# Chú ý: parallel jobs tiêu thụ minutes nhanh hơn nhưng nhanh hơn về thực tế
```

### 2. Tối Ưu Storage

```yaml
# Đặt retention ngắn cho artifacts tạm thời
- uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results/
    retention-days: 3      # Thay vì 90 ngày mặc định

# Dọn dẹp artifacts cũ định kỳ
name: Cleanup
on:
  schedule:
    - cron: '0 0 * * 0'   # Hàng tuần

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
            const { data: { artifacts } } = await github.rest.actions
              .listArtifactsForRepo({
                owner: context.repo.owner,
                repo: context.repo.repo,
              });
            
            for (const artifact of artifacts) {
              if (new Date(artifact.created_at) < sevenDaysAgo) {
                await github.rest.actions.deleteArtifact({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  artifact_id: artifact.id,
                });
              }
            }
```

### 3. Tránh Dùng macOS Runner Không Cần Thiết

```yaml
# SAI — dùng macOS cho task không cần macOS (tốn 10× phí)
jobs:
  test:
    runs-on: macos-latest    # ❌ Tốn $0.08/phút
    steps:
      - run: npm test        # Chỉ cần Linux!

# ĐÚNG — dùng Linux cho hầu hết tasks
jobs:
  test:
    runs-on: ubuntu-latest   # ✅ $0.008/phút (rẻ hơn 10×)
    steps:
      - run: npm test

  # Chỉ dùng macOS khi thực sự cần
  build-ios:
    runs-on: macos-latest    # ✅ Cần thiết cho iOS build
    steps:
      - run: xcodebuild ...
```

### 4. Tối Ưu Job Structure

```yaml
# SAI — mỗi bước nhỏ là 1 job riêng (mỗi job có overhead setup)
jobs:
  checkout:    # Job 1
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
  
  install:     # Job 2 — phải checkout lại từ đầu!
    needs: checkout
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
  
  test:        # Job 3 — phải checkout và install lại!
    needs: install
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

# ĐÚNG — gộp steps liên quan vào 1 job
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

### 5. Tối Ưu Test Strategy

```yaml
# Chỉ chạy full test suite trên main branch và release tags
# PRs chỉ chạy unit tests (nhanh hơn)
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci

      - name: Unit tests (luôn chạy)
        run: npm run test:unit

      - name: Integration tests (chỉ main + tags)
        if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
        run: npm run test:integration

      - name: E2E tests (chỉ trước release)
        if: startsWith(github.ref, 'refs/tags/')
        run: npm run test:e2e
```

### 6. Kết Hợp Smaller + Larger Runners Thông Minh

```yaml
# Dùng runner phù hợp với workload
jobs:
  lint:
    runs-on: ubuntu-latest         # 2 vCPU — đủ cho lint
    steps:
      - run: npm run lint

  build:
    runs-on: ubuntu-latest-4-core  # 4 vCPU — cần cho build nặng
    steps:
      - run: npm run build:production

  # Tính phí: ubuntu-latest-4-core = 2× ubuntu-latest
  # Nhưng nếu build nhanh gấp đôi → break even
  # Nếu nhanh hơn gấp 2.5× → tiết kiệm 20%
```

---

## Self-Hosted vs GitHub-hosted

### So Sánh Chi Phí

| Tiêu Chí | GitHub-hosted | Self-hosted |
|---|---|---|
| **Chi phí per minute** | $0.008 (Linux) | ~$0 (sau khi setup) |
| **Chi phí setup** | $0 | Thời gian kỹ sư + infrastructure |
| **Chi phí vận hành** | $0 | Điện, network, maintenance |
| **Scaling** | Tự động (vô hạn) | Cần cấu hình |
| **Uptime** | SLA của GitHub | Tự chịu trách nhiệm |
| **Break-even** | N/A | ~1,000+ phút/tháng |

### Khi Nào Self-Hosted Tiết Kiệm Hơn

```
Ví dụ: EC2 t3.medium (~$30/tháng)

GitHub-hosted cost break-even:
  $30/tháng ÷ $0.008/phút = 3,750 phút/tháng

Nếu bạn dùng > 3,750 phút Linux/tháng → self-hosted tiết kiệm hơn

Thực tế:
  - Startup với 5 devs: ~6,000 phút/tháng → tiết kiệm
  - Open source project (public repo): GitHub-hosted MIỄN PHÍ → không cần self-hosted
```

### ROI Calculation (Tính Toán Lợi Nhuận Đầu Tư)

```
Kịch bản: Team 10 developers, 50 runs/ngày, mỗi run 8 phút

Với GitHub-hosted:
  50 × 8 × 30 = 12,000 phút/tháng
  Chi phí: (12,000 - 3,000) × $0.008 = $72/tháng

Với self-hosted (2× EC2 t3.large, $120/tháng):
  Infrastructure: $120/tháng
  Tiết kiệm so với GitHub-hosted: -$48/tháng (đắt hơn!)

Với self-hosted (2× EC2 t3.large, nhưng tối ưu workflows):
  Sau khi tối ưu (caching, path filters): 6,000 phút → GitHub-hosted $24/tháng
  Self-hosted vẫn $120/tháng → GitHub-hosted + optimization thắng!

→ Kết luận: Tối ưu workflow TRƯỚC, sau đó xem xét self-hosted
```

---

## Billing Cho Organizations

### Cấu Hình Spending Limit

```
Organization Settings → Billing → Spending Limits

Options:
  - No spending limit (mặc định — rủi ro)
  - Specific dollar amount limit
  - $0 limit (chỉ dùng free tier)
```

### Budget Alerts

```yaml
# Workflow tự động alert khi gần đạt giới hạn
name: Budget Alert

on:
  schedule:
    - cron: '0 8 * * *'    # Kiểm tra mỗi ngày lúc 8 AM

jobs:
  check-budget:
    runs-on: ubuntu-latest
    steps:
      - name: Kiểm tra usage và alert
        uses: actions/github-script@v7
        with:
          script: |
            const { data } = await github.rest.billing.getGithubActionsBillingOrg({
              org: context.repo.owner
            });
            
            const usagePercent = data.total_minutes_used / data.included_minutes * 100;
            
            if (usagePercent > 80) {
              // Gửi notification khi dùng > 80% quota
              core.warning(`⚠️ GitHub Actions: ${usagePercent.toFixed(1)}% quota đã dùng!`);
              // Có thể gửi Slack notification ở đây
            }
            
            core.notice(`GitHub Actions Usage: ${data.total_minutes_used}/${data.included_minutes} phút (${usagePercent.toFixed(1)}%)`);
```

### Per-Repository Cost Attribution (Phân Bổ Chi Phí Theo Repository)

```bash
# Lấy workflow runs của org để phân tích
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/actions/runs?per_page=100" \
  | jq '[.workflow_runs[] | {
      id: .id,
      name: .name,
      status: .status,
      created_at: .created_at,
      run_started_at: .run_started_at,
      updated_at: .updated_at
    }]'
```

---

## 📊 Bảng Tóm Tắt Chiến Lược Tiết Kiệm

| Chiến Lược | Tiết Kiệm Minutes | Tiết Kiệm Storage | Độ Phức Tạp |
|---|---|---|---|
| Dependency caching | 20–40% | 0% | Rất thấp |
| Path filters | 30–70% | 0% | Thấp |
| Concurrency cancel | 15–30% | 0% | Thấp |
| Skip CI | 5–15% | 0% | Rất thấp |
| Dùng Linux thay macOS | 90% (cho tasks đó) | 0% | Thấp |
| Retention ngắn cho artifacts | 0% | 50–80% | Rất thấp |
| Self-hosted runners | 100% (khi >3750 phút) | 0% | Cao |
| Tối ưu test strategy | 20–50% | 10–30% | Trung bình |
| Xóa artifacts cũ | 0% | 30–60% | Thấp |

---

## ✅ Checklist Tối Ưu Chi Phí

```
Ngay lập tức (< 1 giờ):
✅ Thêm path filters để bỏ qua commits không cần thiết
✅ Thêm concurrency: cancel-in-progress: true
✅ Giảm retention-days cho artifacts (7 thay vì 90)
✅ Kiểm tra có dùng macOS runner không cần thiết không
✅ Bật spending limit để tránh bill bất ngờ

Ngắn hạn (1–2 ngày):
✅ Bật dependency caching cho tất cả workflows
✅ Audit workflows — gộp steps liên quan vào cùng job
✅ Chỉ chạy E2E tests trên main/tags, không phải mỗi PR
✅ Thiết lập budget alert workflow

Trung hạn (1–2 tuần):
✅ Đánh giá self-hosted runner nếu > 5,000 phút/tháng
✅ Tối ưu Docker builds với layer caching
✅ Review và xóa workflows cũ không còn dùng
✅ Implement cleanup workflow định kỳ
```

---

## 🔗 Liên Kết Liên Quan

- [4-performance-optimization.md](./4-performance-optimization.md) — Kỹ thuật tối ưu tốc độ
- [README.md](./README.md) — Tổng quan module

---

**Cập Nhật Lần Cuối:** 2026-05-12
