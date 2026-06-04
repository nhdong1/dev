# Pipeline Insights — Thống Kê & Phân Tích Pipeline

> Pipeline Insights (thống kê & phân tích pipeline) là tính năng dashboard (bảng điều khiển) của CircleCI cung cấp số liệu về hiệu năng pipeline: thời gian chạy, tỷ lệ thành công, phát hiện flaky tests (kiểm thử không ổn định) và xác định bottleneck (điểm nghẽn cổ chai).

---

## 🧠 Tổng Quan

### Pipeline Insights Là Gì?

Pipeline Insights là công cụ phân tích tích hợp trong CircleCI UI (giao diện người dùng), cung cấp:

```
┌─────────────────────────────────────────────────┐
│              Pipeline Insights                   │
├─────────────────────────────────────────────────┤
│  Success Rate     │  Throughput    │  Duration   │
│  ─────────────    │  ──────────    │  ──────────  │
│  94.2%            │  128 runs/day  │  8m 32s avg │
│  ↑ 2.1% vs last   │                │  ↓ 45s      │
├─────────────────────────────────────────────────┤
│  Job-Level Breakdown — Phân Tích Theo Job       │
│  ─────────────────────────────────────────────  │
│  ■ test          □□□□□□□□□□□□ 5m 20s  (63%)     │
│  ■ build         □□□□ 2m 10s  (25%)              │
│  ■ lint          □□ 50s       (10%)              │
│  ■ deploy        □ 12s        (2%)               │
├─────────────────────────────────────────────────┤
│  Flaky Tests — Kiểm Thử Không Ổn Định          │
│  AuthService.test.ts — 12 failures in 50 runs   │
│  UserController.test.ts — 3 failures in 50 runs │
└─────────────────────────────────────────────────┘
```

---

## 📍 Cách Truy Cập

### Trong CircleCI UI

```
1. Vào circleci.com → Chọn Organization
2. Click "Insights" ở sidebar trái
3. Chọn repository và branch cần xem
4. Hoặc vào trực tiếp một pipeline → tab "Insights"
```

### CircleCI CLI

```bash
# Xem insights qua API
curl -X GET \
  "https://circleci.com/api/v2/insights/{project-slug}/workflows" \
  -H "Circle-Token: $CIRCLECI_TOKEN"

# Xem jobs trong workflow cụ thể
curl -X GET \
  "https://circleci.com/api/v2/insights/{project-slug}/workflows/{workflow-name}/jobs" \
  -H "Circle-Token: $CIRCLECI_TOKEN"
```

---

## 📊 Các Chỉ Số Quan Trọng — Key Metrics

### 1. Duration — Thời Gian Chạy

```
Pipeline Duration Trend — Xu Hướng Thời Gian:

Tuần 1:  ████████████████████ 20 phút
Tuần 2:  █████████████████    17 phút  ← Sau khi thêm caching
Tuần 3:  ██████████           10 phút  ← Sau khi thêm parallelism
Tuần 4:  ████████              8 phút  ← Sau khi tối ưu resource class
```

**Cần xem xét khi:**
- Duration tăng đột ngột (→ có thể test mới chậm hoặc dependency lớn)
- Duration tăng dần (→ test suite mở rộng, cần scaling)
- P95 (95th percentile) cao hơn nhiều so với mean (→ có outlier cần xử lý)

### 2. Success Rate — Tỷ Lệ Thành Công

```
Mức Độ Đánh Giá:
  > 95%    ✅ Tốt — pipeline ổn định
  90–95%   ⚠️ Cảnh báo — có flaky tests hoặc infrastructure issue
  < 90%    🚨 Cần điều tra ngay
```

**Nguyên nhân success rate thấp:**
- Flaky tests (kiểm thử không ổn định)
- External API calls không reliable
- Race conditions trong test
- OOM — Out Of Memory (hết bộ nhớ) do resource class không đủ
- Network timeout

### 3. Throughput — Thông Lượng

```
Số lượng pipeline chạy thành công trong một khoảng thời gian.

Thấp hơn bình thường → Developer không push code? Hay pipeline block?
Cao hơn bình thường → Nhiều PR? Hay pipeline đang re-run nhiều?
```

### 4. MTTR — Mean Time To Recovery (Thời Gian Phục Hồi Trung Bình)

```
MTTR = Thời gian từ lúc pipeline fail đến lúc pipeline xanh lại

< 30 phút:  ✅ Team phản ứng nhanh
30–60 phút: ⚠️ Cần cải thiện quy trình
> 2 giờ:    🚨 Vấn đề nghiêm trọng về culture hoặc tooling
```

---

## 🔍 Phân Tích Bottleneck — Tìm Điểm Nghẽn

### Bước 1: Xác Định Job Chậm Nhất

```
Xem Job Duration Breakdown trong Insights:

test-unit    ████████████████████ 12m 30s  ← BOTTLENECK
test-e2e     ██████████           6m 10s
build        ████                 2m 20s
lint         ██                   1m 05s
deploy       █                    0m 30s

→ Tập trung tối ưu test-unit trước
```

### Bước 2: Phân Tích Job Cụ Thể

Trong từng job, xem Step Duration (thời gian của từng bước):

```
Job: test-unit (12m 30s total)

restore_cache     ██                      30s
npm ci            ████████████████████  5m 00s  ← Cache miss!
run tests         ██████████████████    4m 30s
save_cache        ██                      30s
store_results     █                       10s

→ cache miss là vấn đề chính
```

### Bước 3: Đề Ra Giải Pháp

```
Cache miss → Kiểm tra cache key, thêm fallback keys
Test chậm → Thêm parallelism + test splitting
Build lại nhiều lần → Dùng workspace
Resource thiếu → Tăng resource class
```

---

## 🐛 Flaky Tests — Kiểm Thử Không Ổn Định

### Định Nghĩa

```
Flaky Test (kiểm thử không ổn định):
  Lần 1: ✅ Pass
  Lần 2: ❌ Fail (cùng code, không có thay đổi)
  Lần 3: ✅ Pass
  → Kết quả không nhất quán
```

### CircleCI Phát Hiện Flaky Tests Ra Sao?

CircleCI phân tích kết quả từ `store_test_results` và đánh dấu test nào:
- Fail ở lần chạy đầu nhưng pass ở lần re-run
- Fail không nhất quán với cùng commit SHA

### Xem Flaky Tests Trong Insights

```
Insights → Tab "Flaky Tests"

┌───────────────────────────────────────────────────┐
│ Test Name                    │ Failure % │ Last Run │
│ ─────────────────────────────│───────────│──────────│
│ AuthService › login timeout  │ 24%       │ 2h ago   │
│ UserAPI › rate limit test    │ 12%       │ 5h ago   │
│ PaymentFlow › card process   │ 8%        │ 1d ago   │
└───────────────────────────────────────────────────┘
```

### Nguyên Nhân và Cách Xử Lý

```
Nguyên Nhân Thường Gặp:
─────────────────────────────────────────────────────
1. Race conditions — Điều Kiện Tranh Chấp
   → Thêm proper async/await, tránh setTimeout ngẫu nhiên

2. External API calls không stable
   → Mock external services trong unit tests

3. Test order dependency — Phụ Thuộc Thứ Tự Test
   → Mỗi test phải độc lập, tự setup và cleanup

4. Shared state giữa tests
   → beforeEach/afterEach cleanup đầy đủ

5. Timing-dependent tests
   → Dùng fake timers thay vì real timers

6. Resource contention — Tranh Giành Tài Nguyên
   → Tăng resource class hoặc giảm parallelism
```

### Cấu Hình Phát Hiện Flaky Tests

```yaml
# Lưu test results đầy đủ để CircleCI phân tích
- store_test_results:
    path: test-results/

# CircleCI tự động phân tích và đánh dấu flaky tests trong Insights
```

---

## 📈 Thiết Lập `store_test_results` Đúng Cách

### Tại Sao Quan Trọng?

`store_test_results` không chỉ để xem kết quả — nó còn:
1. Cung cấp **timing data** để test splitting chính xác
2. Cho phép **flaky test detection**
3. Hiển thị **test failure details** trong CircleCI UI
4. Đóng góp vào **Pipeline Insights** metrics

### Format Được Hỗ Trợ

CircleCI đọc **JUnit XML** format (định dạng XML theo chuẩn JUnit):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
  <testsuite name="AuthService" tests="3" failures="0" time="5.234">
    <testcase name="should login" classname="AuthService" time="2.1" />
    <testcase name="should logout" classname="AuthService" time="1.3" />
    <testcase name="should refresh" classname="AuthService" time="1.834">
      <failure message="Timeout after 5000ms">...</failure>
    </testcase>
  </testsuite>
</testsuites>
```

### Cấu Hình Theo Framework

```yaml
# Jest — Cần jest-junit package
- run:
    environment:
      JEST_JUNIT_OUTPUT_DIR: test-results/
    command: npx jest --reporters=jest-junit

# pytest
- run:
    command: pytest --junitxml=test-results/pytest.xml

# RSpec
- run:
    command: |
      bundle exec rspec \
        --format RspecJunitFormatter \
        --out test-results/rspec.xml

# Go
- run:
    command: |
      go test -v ./... 2>&1 | \
      go-junit-report > test-results/go-test.xml

# Maven
- run:
    command: mvn test   # JUnit XML tự động tạo trong target/surefire-reports/
- store_test_results:
    path: target/surefire-reports/

# Gradle
- run:
    command: ./gradlew test
- store_test_results:
    path: build/test-results/test/
```

---

## 🔧 Insights API — Tích Hợp Tự Động Hóa

### Lấy Dữ Liệu Pipeline Qua API

```bash
# Lấy workflow metrics
curl -X GET \
  "https://circleci.com/api/v2/insights/github/myorg/myrepo/workflows/ci-cd" \
  -H "Circle-Token: $CIRCLECI_TOKEN" | \
  jq '.items[] | {duration: .duration, status: .status}'

# Lấy job metrics
curl -X GET \
  "https://circleci.com/api/v2/insights/github/myorg/myrepo/workflows/ci-cd/jobs" \
  -H "Circle-Token: $CIRCLECI_TOKEN"
```

### Script Tự Động Cảnh Báo

```python
import requests
import os

TOKEN = os.environ["CIRCLECI_TOKEN"]
PROJECT = "github/myorg/myrepo"

def check_pipeline_health():
    resp = requests.get(
        f"https://circleci.com/api/v2/insights/{PROJECT}/workflows/ci-cd",
        headers={"Circle-Token": TOKEN}
    )
    data = resp.json()
    
    for item in data["items"]:
        success_rate = item.get("metrics", {}).get("success_rate", 1)
        if success_rate < 0.90:
            # Gửi alert khi success rate < 90%
            send_slack_alert(f"Pipeline success rate: {success_rate:.1%}")

check_pipeline_health()
```

---

## 📋 Dashboard Nên Theo Dõi Hàng Tuần

### Checklist Sức Khỏe Pipeline

```markdown
## Kiểm Tra Hàng Tuần — Weekly Pipeline Health Check

### Metrics — Chỉ Số
- [ ] Success rate > 95%?
- [ ] P50 duration giảm hoặc ổn định?
- [ ] Không có flaky test mới xuất hiện?
- [ ] Flaky tests cũ đã được fix?

### Bottlenecks — Điểm Nghẽn
- [ ] Job nào chiếm > 50% tổng thời gian?
- [ ] Cache hit rate có ổn không? (xem log restore_cache)
- [ ] Test splitting phân phối đều không?

### Costs — Chi Phí
- [ ] Credit usage trong giới hạn budget?
- [ ] Có job nào dùng resource class quá cao không cần?
- [ ] Có pipeline re-run nhiều hơn bình thường không?
```

---

## 🚀 Quy Trình Tối Ưu Hóa Liên Tục

```
Measure — Đo Lường
  → Pipeline Insights: duration, success rate, flaky tests

Analyze — Phân Tích
  → Job nào chậm nhất? Step nào tốn thời gian?
  → Cache hit rate bao nhiêu?
  → Flaky test nào phổ biến nhất?

Optimize — Tối Ưu
  → Thêm/cải thiện caching
  → Tăng parallelism + test splitting
  → Fix flaky tests
  → Điều chỉnh resource class

Verify — Xác Nhận
  → Theo dõi 1–2 tuần sau khi thay đổi
  → So sánh trước/sau qua Insights

Repeat — Lặp Lại
  → Tìm bottleneck tiếp theo
```

---

## 💬 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Làm thế nào để xác định bottleneck trong CircleCI pipeline?**

A: Dùng Pipeline Insights dashboard để xem job nào chiếm nhiều thời gian nhất. Sau đó xem chi tiết từng step trong job đó. Thường thì vấn đề là: cache miss (npm install lại từ đầu), test suite chưa được split, hoặc resource class không đủ gây CPU throttle.

**Q: Flaky test là gì và CircleCI xử lý thế nào?**

A: Flaky test (kiểm thử không ổn định) là test cho kết quả không nhất quán — lúc pass, lúc fail — với cùng code. CircleCI tự động phát hiện qua tính năng Flaky Test Detection trong Insights bằng cách phân tích JUnit XML output từ `store_test_results`. Dashboard hiển thị tỷ lệ fail và tần suất để ưu tiên xử lý.

**Q: `store_test_results` có tác dụng gì ngoài việc xem kết quả?**

A: Ngoài việc hiển thị test kết quả trong UI, `store_test_results` còn: (1) cung cấp timing data để `circleci tests split --split-by=timings` phân chia chính xác; (2) cho phép flaky test detection; (3) đóng góp vào Pipeline Insights metrics về duration và success rate theo test granularity.

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Trạng Thái:** ✅ Hoàn Thành
