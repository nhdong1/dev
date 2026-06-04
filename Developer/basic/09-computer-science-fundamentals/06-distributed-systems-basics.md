# Distributed Systems Basics - Interview Questions

## Question 1: Distributed system là gì?

**Answer:**

- Là tập hợp nhiều node phối hợp để cung cấp một dịch vụ thống nhất.
- Node giao tiếp qua mạng nên luôn tồn tại độ trễ và lỗi truyền thông.
- Mục tiêu thường là scale, availability, và fault isolation.
- Đổi lại hệ thống phức tạp hơn nhiều so với single-node.

---

## Question 2: Fallacies of distributed computing là gì?

**Answer:**

- Các giả định sai phổ biến: mạng luôn ổn định, latency bằng 0, băng thông vô hạn.
- Cũng có giả định sai về bảo mật, topology tĩnh, và chỉ có một admin.
- Nắm các fallacy giúp thiết kế thận trọng hơn.
- Rất nhiều outage lớn bắt nguồn từ việc bỏ qua các giả định này.

---

## Question 3: Replication giúp gì và đánh đổi gì?

**Answer:**

- Tăng availability và khả năng chịu lỗi.
- Cải thiện read throughput khi phân tán bản sao.
- Đánh đổi: consistency phức tạp hơn, chi phí đồng bộ tăng.
- Cần chọn mô hình replication theo yêu cầu nghiệp vụ.

---

## Question 4: Consistency model phổ biến gồm những gì?

**Answer:**

- Strong consistency: đọc luôn thấy ghi mới nhất.
- Eventual consistency: cuối cùng sẽ hội tụ.
- Causal consistency: tôn trọng quan hệ nhân quả.
- Linearizability mạnh nhất nhưng thường tốn latency cao hơn.

---

## Question 5: Quorum read/write hoạt động như thế nào?

**Answer:**

- Giả sử có N replica.
- Ghi cần W replica xác nhận, đọc cần R replica.
- Nếu R + W > N thì tăng khả năng đọc dữ liệu mới.
- Thực tế vẫn có edge case do clock skew hoặc repair chậm.

---

## Question 6: Leader-based replication và leaderless replication?

**Answer:**

- Leader-based: một node nhận ghi chính, follower đồng bộ theo.
- Ưu điểm đơn giản hơn về thứ tự ghi; nhược điểm leader là điểm nóng.
- Leaderless (Dynamo style) cho phép ghi nhiều node, linh hoạt hơn khi partition.
- Đổi lại conflict resolution và read repair phức tạp.

---

## Question 7: Split-brain là gì?

**Answer:**

- Khi cluster bị partition và nhiều node cùng nghĩ mình là leader.
- Dẫn tới ghi xung đột và dữ liệu phân kỳ.
- Cần cơ chế quorum, fencing token, và election an toàn.
- Đây là rủi ro lớn trong hệ thống HA nhiều vùng mạng.

---

## Question 8: Consensus algorithm (Raft/Paxos) giải quyết bài toán gì?

**Answer:**

- Giúp các node đạt đồng thuận về thứ tự log/giá trị.
- Chịu được một số node lỗi nhưng vẫn nhất quán.
- Raft thường dễ hiểu và dễ triển khai hơn Paxos.
- Dùng cho metadata, configuration store, distributed lock service.

---

## Question 9: Idempotency trong distributed workflow quan trọng ra sao?

**Answer:**

- Mạng có thể timeout khiến client retry dù server đã xử lý.
- Idempotency giúp tránh tạo dữ liệu trùng hoặc side-effect lặp.
- Thiết kế qua idempotency key và dedup store.
- Đây là yêu cầu cốt lõi cho payment, booking, message handling.

---

## Question 10: Saga pattern dùng khi nào?

**Answer:**

- Khi transaction trải qua nhiều service, không dùng 2PC toàn cục.
- Saga chia thành nhiều local transaction + bước bù (compensation).
- Có 2 kiểu: choreography và orchestration.
- Cần thiết kế idempotent và có observability tốt để vận hành.

---

## Question 11: Exactly-once delivery có khả thi không?

**Answer:**

- Trong thực tế phân tán, exactly-once end-to-end rất khó và đắt.
- Thường triển khai at-least-once + idempotent consumer.
- Một số hệ thống cung cấp exactly-once trong phạm vi hẹp.
- Cần đọc kỹ guarantee của broker, client, và storage layer.

---

## Question 12: Clock drift ảnh hưởng gì?

**Answer:**

- Đồng hồ node lệch gây sai thứ tự theo timestamp.
- Ảnh hưởng timeout, token expiry, conflict resolution.
- Dùng NTP/PTP để giảm drift nhưng không loại bỏ hoàn toàn.
- Thiết kế nên tránh phụ thuộc tuyệt đối vào clock wall-time.

---

## Question 13: Retry, timeout, circuit breaker phối hợp như thế nào?

**Answer:**

- Timeout đặt ranh giới chờ tối đa cho từng call.
- Retry có điều kiện với backoff và jitter.
- Circuit breaker ngắt sớm khi downstream lỗi kéo dài.
- Kết hợp đúng giúp ngăn cascading failure toàn hệ thống.

---

## Question 14: Observability trong distributed system cần gì?

**Answer:**

- Metrics để thấy xu hướng và cảnh báo.
- Logs có correlation id để truy vết flow.
- Distributed tracing để thấy đường đi request qua nhiều service.
- Thiếu observability làm MTTR tăng mạnh khi sự cố xảy ra.

---

## Question 15: Thiết kế để graceful degradation như thế nào?

**Answer:**

- Xác định chức năng cốt lõi và chức năng phụ.
- Khi phụ thuộc lỗi, trả kết quả giảm cấp thay vì fail toàn bộ.
- Ví dụ tắt recommendation nhưng vẫn cho checkout.
- Mục tiêu là giữ trải nghiệm chính trong điều kiện lỗi một phần.
