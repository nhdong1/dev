# Pipeline Insights — Giám Sát Dashboard CircleCI

> **Pipeline Insights** (Thống Kê Pipeline) là dashboard trong CircleCI UI dùng để **giám sát vận hành**: success rate (tỷ lệ thành công), duration (thời gian chạy), throughput (số lần chạy), flaky tests. Module [05-optimization/5-pipeline-insights.md](../05-optimization/5-pipeline-insights.md) đi sâu hơn về **tối ưu** dựa trên số liệu; file này tập trung **observability** (khả năng quan sát) và phản ứng sự cố.

---

## 📚 Mục Lục

1. [Vai trò trong vận hành](#vai-trò-trong-vận-hành)
2. [Truy cập và phạm vi dữ liệu](#truy-cập-và-phạm-vi-dữ-liệu)
3. [Chỉ số cần theo dõi hàng ngày](#chỉ-số-cần-theo-dõi-hàng-ngày)
4. [Phân tích theo job và workflow](#phân-tích-theo-job-và-workflow)
5. [Flaky tests trên Insights](#flaky-tests-trên-insights)
6. [API và tích hợp báo cáo](#api-và-tích-hợp-báo-cáo)
7. [Runbook ngắn khi metric xấu](#runbook-ngắn-khi-metric-xấu)
8. [Câu hỏi phỏng vấn](#câu-hỏi-phỏng-vấn)

---

## Vai trò trong vận hành

Insights trả lời các câu hỏi vận hành, không chỉ “build chậm thế nào tối ưu”:

| Câu hỏi | Metric / View |
|---------|----------------|
| CI có đáng tin không? | Success rate theo branch (`main` vs feature) |
| Team bị block bao lâu? | Mean/P95 duration pipeline |
| Lỗi có lặp lại không? | Failed workflows theo tên job |
| Test có gây nhiễu không? | Flaky test list |
| Deploy có ổn định không? | Success rate workflow `deploy-*` |

**Phân biệt với Optimization module:**

- **08-monitoring (file này):** phát hiện sự cố, trend xấu, ưu tiên điều tra
- **05-optimization:** hành động cụ thể (cache, parallelism, resource class)

---

## Truy cập và phạm vi dữ liệu

### CircleCI UI

```
Organization → Insights (sidebar)
  → Chọn project
  → Lọc branch (ví dụ: main)
  → Chọn workflow name (ví dụ: build-and-test)
  → Xem jobs breakdown và flaky tests
```

Từ một **Pipeline** cụ thể cũng có tab liên quan để so sánh với lịch sử.

### Lưu ý phạm vi

- Dữ liệu phụ thuộc **gói CircleCI** (một số metric nâng cao trên plan cao hơn)
- Branch filter sai → kết luận sai (ví dụ chỉ xem `main` trong khi lỗi ở `release/*`)
- Job đổi tên trong config → trend bị đứt đoạn (ghi chú trong changelog nội bộ)

---

## Chỉ số cần theo dõi hàng ngày

### 1. Success Rate — Tỷ Lệ Thành Công

```
Ví dụ:
  main — 7 ngày:  96%  (mục tiêu ≥ 98% cho nhánh protected)
  PR — 7 ngày:    88%  (có thể thấp hơn nếu WIP — Work In Progress)
```

**Hành động khi giảm đột ngột:**

1. Xem pipeline fail gần nhất — cùng job/step không?
2. Đối chiếu merge/deploy gần đó
3. Kiểm tra flaky tests (mục dưới)

### 2. Duration — Thời Gian Chạy

Theo dõi **mean** (trung bình) và **P95** (phân vị 95 — 5% chậm nhất):

- P95 >> mean → outlier (job thỉnh thoảng treo, runner chậm, test không ổn định)
- Mean tăng dần → suite phình, cache miss, thiếu parallelism

### 3. Throughput — Lượng Chạy

Số pipeline/ngày giúp tách “lỗi do code” vs “áp lực hệ thống” (queue — Hàng Đợi, credit hết).

---

## Phân tích theo job và workflow

### Job breakdown

Insights hiển thị % thời gian theo job — dùng để **ưu tiên điều tra**, không phải lúc nào cũng tối ưu job dài nhất:

| Tình huống | Ưu tiên |
|------------|---------|
| Job `test` 70% duration, success 99% | Tối ưu khi cần giảm lead time |
| Job `deploy` 5% duration, success 85% | **Ưu tiên sửa** — ảnh hưởng production |
| Job `lint` fail thường xuyên | Fix nhanh — chi phí thấp |

### So sánh workflow

Nhiều workflow trong một repo (`ci`, `nightly`, `release`):

- Giám sát riêng từng workflow
- Đặt ngưỡng khác nhau (nightly cho phép duration cao hơn)

---

## Flaky tests trên Insights

CircleCI có thể liệt kê test **flaky** (pass/fail không nhất quán trên cùng codebase):

```
AuthService.test.ts — 12 failures / 50 runs
```

**Quy trình team:**

1. Insights flag test → tạo ticket (Jira/Linear)
2. Owner fix hoặc tạm `skip` có thời hạn
3. Không merge PR mới nếu flaky trên `main` vượt ngưỡng team

Chi tiết kỹ thuật: [4-flaky-tests.md](./4-flaky-tests.md).

---

## API và tích hợp báo cáo

### REST API v2 (ví dụ)

```bash
# Workflow insights (thay project-slug và workflow-name)
curl -s -H "Circle-Token: $CIRCLECI_API_TOKEN" \
  "https://circleci.com/api/v2/insights/gh/myorg/myrepo/workflows/build-and-test"

# Job insights trong workflow
curl -s -H "Circle-Token: $CIRCLECI_API_TOKEN" \
  "https://circleci.com/api/v2/insights/gh/myorg/myrepo/workflows/build-and-test/jobs/test"
```

Dùng API khi cần:

- Dashboard nội bộ (Grafana, Datadog)
- Báo cáo tuần cho leadership
- Alert khi success rate < ngưỡng (kết hợp script + Slack)

**Bảo mật:** token API là secret — lưu trong [Context](../06-security/1-contexts.md), không commit.

---

## Runbook ngắn khi metric xấu

| Triệu chứng | Bước 1 | Bước 2 |
|-------------|--------|--------|
| Success rate `main` < 95% / 24h | Mở failed pipeline mới nhất | Phân loại: test vs infra vs config |
| Duration P95 tăng 2x | Job nào tăng? (breakdown) | Cache key đổi? dependency mới? |
| Flaky list tăng | Liệt kê test, assign owner | Retry policy, split test |
| Chỉ một branch fail | Filter branch trên Insights | So sánh config/filter workflow |

Sau khi xác định **job/step**, chuyển sang [3-common-errors.md](./3-common-errors.md) hoặc [2-ssh-debugging.md](./2-ssh-debugging.md).

---

## Câu hỏi phỏng vấn

**H: Pipeline Insights khác gì so với log từng pipeline?**

**Đ:** Log trả lời “lần chạy này fail vì sao”; Insights trả lời “xu hướng, job nào hay fail, test nào flaky” — phục vụ cải tiến liên tục và SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ) CI.

**H: Bạn dùng Insights thế nào khi on-call?**

**Đ:** Kiểm tra success rate + failed workflows 1h gần nhất → xác định job chung → correlate với merge/deploy → nếu không rõ, SSH debug hoặc rerun với `CI=true` env giống CI.

---

**Tiếp theo:** [2-ssh-debugging.md](./2-ssh-debugging.md) — khi cần tái hiện lỗi trên runner thật.
