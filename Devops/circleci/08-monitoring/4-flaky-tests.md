# Flaky Tests — Kiểm Thử Không Ổn Định Trên CI

> **Flaky test** (kiểm thử không ổn định) là test cùng commit đôi khi pass, đôi khi fail mà không do thay đổi logic rõ ràng — gây **false failure** (lỗi giả) trên pipeline, làm mất niềm tin CI và tốn thời gian rerun.

---

## 📚 Mục Lục

1. [Nhận diện flaky test](#nhận-diện-flaky-test)
2. [Nguyên nhân phổ biến](#nguyên-nhân-phổ-biến)
3. [Phát hiện với CircleCI](#phát-hiện-với-circleci)
4. [Chiến lược xử lý](#chiến-lược-xử-lý)
5. [Cấu hình CI giảm flaky](#cấu-hình-ci-giảm-flaky)
6. [Quy trình team](#quy-trình-team)
7. [Câu hỏi phỏng vấn](#câu-hỏi-phỏng-vấn)

---

## Nhận diện flaky test

### Dấu hiệu

- Fail trên CI, pass khi **rerun** cùng commit
- Fail khi chạy **parallel** (`parallelism` > 1) nhưng pass khi chạy tuần tự
- Fail vào cuối tuần / đầu giờ (race với service ngoài)
- Insights liệt kê test với tỷ lệ fail không ổn định

### Không nhầm với

| Loại | Đặc điểm |
|------|----------|
| **Flaky** | Không deterministic — cùng SHA |
| **Test sai** | Fail ổn định 100% |
| **Infra** | Nhiều test không liên quan fail cùng lúc (OOM, registry down) |

---

## Nguyên nhân phổ biến

### 1. Race condition — Điều Kiện Đua

- Shared state giữa test files
- Async không `await` đúng
- Database/API không isolate per test

### 2. Phụ thuộc thời gian và mạng

- `sleep` cố định thay vì poll/retry
- Gọi API thật không mock
- Timezone / `Date.now()` không mock

### 3. Thứ tự và parallel

- Test phụ thuộc thứ tự chạy
- Port collision khi parallel containers
- File lock trên thư mục chung

### 4. Dữ liệu và môi trường

- Seed random không cố định
- Dùng dữ liệu production-like không cleanup
- Khác biệt Linux CI vs macOS local

---

## Phát hiện với CircleCI

### Pipeline Insights

Xem danh sách flaky tests theo project (chi tiết UI: [1-pipeline-insights.md](./1-pipeline-insights.md)).

### Rerun thống kê

```
Cùng commit SHA:
  Run 1: fail — UserApi.test.ts
  Run 2: pass
  Run 3: fail — UserApi.test.ts
→ Nghi flaky (nếu infra ổn định)
```

### Test splitting

Khi dùng `circleci tests split`, test chạy trên node khác nhau → lộ flaky phụ thuộc isolation:

```yaml
parallelism: 4
steps:
  - run:
      command: |
        TEST_FILES=$(circleci tests glob "**/*.test.js" | circleci tests split --split-by=timings)
        npm test -- $TEST_FILES
```

---

## Chiến lược xử lý

### Thứ tự ưu tiên (khuyến nghị)

1. **Fix root cause** — đúng nhất
2. **Isolate** — DB test container riêng, mock external
3. **Retry có kiểm soát** — chỉ integration e2e, giới hạn số lần
4. **Quarantine** — tách job hoặc tag `@flaky`, không block `main` tạm thời
5. **Skip có ticket** — deadline fix, không skip vô thời hạn

### Retry trong test runner (ví dụ Jest)

```javascript
// jest.config.js — chỉ khi team chấp nhận che flaky ngắn hạn
module.exports = {
  testTimeout: 30000,
  // jest-junit, retry plugins — document rõ policy
};
```

Tránh retry vô hạn trên CI — che lỗi và kéo dài build.

### Tách job flaky

```yaml
workflows:
  ci:
    jobs:
      - unit-test:
          filters:
            branches:
              only: /.*/
      - flaky-quarantine:
          filters:
            branches:
              only: main
          # optional: allow failure hoặc chạy schedule riêng
```

---

## Cấu hình CI giảm flaky

### 1. Deterministic environment

```yaml
environment:
  TZ: UTC
  CI: true
```

### 2. Services (database) trong config

```yaml
docker:
  - image: cimg/node:20.0
    environment:
      DATABASE_URL: postgres://test:test@localhost:5432/test
  - image: cimg/postgres:15.0
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
```

### 3. Tránh chia sẻ file giữa parallel nodes

Mỗi node nên có working copy riêng (CircleCI mặc định per container); không ghi cùng path shared ngoài workspace unless designed.

### 4. `--runInBand` / workers=1 để debug

Tạm trên branch debug để xác nhận race:

```yaml
- run: npm test -- --runInBand
```

Không dùng vĩnh viễn nếu suite lớn — chỉ chẩn đoán.

---

## Quy trình team

```markdown
1. Insights hoặc CI báo test X flaky
2. Tạo ticket: owner, SLA 5 ngày làm việc
3. Ghi vào "flaky registry" (wiki/Notion)
4. PR fix: bắt buộc chạy test đó N lần local hoặc trên CI label `flake-check`
5. Đóng ticket khi 50 runs không fail trên main
```

**Policy `main`:** không merge PR mới nếu đang có flaky blocker chưa quarantine (tùy team).

**STAR story (phỏng vấn):** mô tả lần giảm false failure từ X% → Y% nhờ fix race + Insights.

---

## Câu hỏi phỏng vấn

**H: Flaky test khác failed test thế nào?**

**Đ:** Failed test reproduce ổn định; flaky không deterministic trên cùng revision — cần thống kê nhiều run và Insights.

**H: Có nên bật auto-retry toàn pipeline không?**

**Đ:** Retry toàn pipeline che infra và tốn credit; tốt hơn fix test, retry có giới hạn ở tầng test runner, hoặc quarantine. Rerun thủ công cho merge khẩn cấp có approval.

**H: Parallelism làm flaky tệ hơn?**

**Đ:** Có thể — lộ race và shared resource; giảm parallelism để xác nhận, sau đó fix isolation thay vì bỏ parallel vĩnh viễn.

---

**Liên quan:** Tối ưu suite dài — [05-optimization/3-test-splitting.md](../05-optimization/3-test-splitting.md) | Thông báo khi `main` fail — [07-integration/6-notifications.md](../07-integration/6-notifications.md)
