# Testing Fundamentals - Interview Questions

## Question 1: Tại sao testing quan trọng với backend systems?

**Answer:**

- Giảm rủi ro lỗi production và chi phí sửa muộn.
- Tăng tự tin khi refactor và nâng cấp dependency.
- Là tài liệu sống mô tả hành vi kỳ vọng của hệ thống.
- Kết hợp tốt với CI/CD để phát hiện lỗi sớm.

---

## Question 2: Unit test, Integration test, E2E test khác nhau thế nào?

**Answer:**

- Unit test: phạm vi nhỏ, chạy nhanh, cô lập dependency.
- Integration test: kiểm tra tương tác giữa module/service thật.
- E2E test: kiểm tra luồng nghiệp vụ từ đầu đến cuối.
- Test pyramid khuyến nghị nhiều unit, vừa integration, ít E2E.

---

## Question 3: Test double gồm những loại nào?

**Answer:**

- Dummy: truyền cho đủ tham số, không dùng.
- Stub: trả dữ liệu cố định.
- Mock: xác minh tương tác/hành vi gọi.
- Fake: implementation đơn giản thay hệ thống thật (vd in-memory repo).

---

## Question 4: Flaky test là gì và xử lý ra sao?

**Answer:**

- Flaky test lúc pass lúc fail dù code không đổi.
- Nguyên nhân: phụ thuộc thời gian, network, race condition, shared state.
- Xử lý bằng isolation, deterministic data, và kiểm soát clock/random.
- Flaky test cần ưu tiên sửa ngay vì làm mất niềm tin vào CI.

---

## Question 5: Arrange-Act-Assert giúp gì cho test?

**Answer:**

- Arrange: chuẩn bị dữ liệu và môi trường.
- Act: thực hiện hành động cần kiểm thử.
- Assert: xác nhận kết quả.
- Mẫu này giúp test dễ đọc, dễ review, và nhất quán toàn đội.

---

## Question 6: Black-box và White-box testing khác nhau?

**Answer:**

- Black-box test theo hành vi đầu vào/đầu ra, không cần biết implementation.
- White-box test dựa trên cấu trúc code nội bộ.
- Cần kết hợp cả hai để tăng độ phủ theo rủi ro.
- Quá phụ thuộc white-box dễ làm test gãy khi refactor.

---

## Question 7: Test coverage bao nhiêu là đủ?

**Answer:**

- Không có con số thần kỳ áp dụng mọi dự án.
- Coverage cao không đồng nghĩa chất lượng test cao.
- Quan trọng là phủ được luồng rủi ro cao và edge cases chính.
- Dùng coverage như chỉ báo phụ, không phải mục tiêu duy nhất.

---

## Question 8: Contract test trong microservices là gì?

**Answer:**

- Kiểm tra thỏa thuận request/response giữa provider và consumer.
- Giúp phát hiện breaking change sớm trước khi deploy.
- Tốt cho hệ thống nhiều service release độc lập.
- Giảm nhu cầu chạy E2E full-stack quá thường xuyên.

---

## Question 9: Snapshot test nên dùng khi nào?

**Answer:**

- Hữu ích khi output lớn và ổn định theo cấu trúc.
- Dùng cho schema, serialized payload, template output.
- Không nên snapshot bừa bãi vì dễ tạo "approve mù".
- Cần review diff kỹ để tránh hợp thức hóa lỗi.

---

## Question 10: Property-based testing là gì?

**Answer:**

- Định nghĩa thuộc tính luôn đúng thay vì ví dụ cụ thể.
- Framework tự sinh nhiều input để tìm counterexample.
- Rất hiệu quả cho parser, serializer, thuật toán.
- Giúp phát hiện edge case mà test case thủ công thường bỏ sót.

---

## Question 11: Test dữ liệu và test fixture nên quản lý thế nào?

**Answer:**

- Dữ liệu test phải nhỏ, rõ nghĩa, và tái tạo được.
- Tránh chia sẻ fixture mutable giữa nhiều test.
- Ưu tiên factory/builder để tạo dữ liệu theo ngữ cảnh.
- Dọn dẹp môi trường sau test để đảm bảo độc lập.

---

## Question 12: Testing asynchronous code cần lưu ý gì?

**Answer:**

- Luôn chờ hoàn tất tác vụ async trước khi assert.
- Kiểm soát timeout và tránh sleep cứng không cần thiết.
- Dùng fake clock/scheduler để test deterministic hơn.
- Với concurrent flow, cần kiểm tra thứ tự sự kiện quan trọng.

---

## Question 13: Performance test và load test khác gì?

**Answer:**

- Performance test đo độ trễ/throughput theo nhiều mức tải.
- Load test tập trung ở mức tải kỳ vọng để xác nhận SLA.
- Stress test đẩy vượt ngưỡng để quan sát điểm gãy.
- Nên có baseline để so sánh regression qua từng release.

---

## Question 14: Nguyên tắc viết test dễ bảo trì?

**Answer:**

- Mỗi test xác minh một ý chính, tên test rõ hành vi.
- Tránh assert quá nhiều thứ không liên quan.
- Ẩn setup phức tạp vào helper có tên mô tả tốt.
- Refactor test code giống như refactor production code.

---

## Question 15: Vai trò testing trong pipeline CI/CD?

**Answer:**

- Gate chất lượng trước khi merge/deploy.
- Rút ngắn feedback loop cho developer.
- Kết hợp unit + integration + smoke test sau deploy.
- Mục tiêu cuối là giảm change failure rate và MTTR.
