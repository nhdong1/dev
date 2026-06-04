# Concurrency - Interview Questions

## Question 1: Concurrency và Parallelism khác nhau như thế nào?

**Answer:**

- Concurrency là quản lý nhiều công việc cùng tiến triển.
- Parallelism là thực thi thực sự đồng thời trên nhiều core.
- Ứng dụng có thể concurrent nhưng không parallel (single core).
- Backend I/O-heavy thường tối ưu bằng concurrency trước khi cần parallelism.

---

## Question 2: Data race là gì? Khác race condition thế nào?

**Answer:**

- Data race: hai luồng truy cập cùng biến, có ít nhất một ghi, không đồng bộ.
- Race condition rộng hơn: hành vi sai do thứ tự thực thi, không chỉ biến shared.
- Data race thường dẫn đến undefined behavior ở ngôn ngữ low-level.
- Cần static analysis, sanitizers, và thiết kế state rõ ownership.

---

## Question 3: Atomic operation dùng khi nào?

**Answer:**

- Khi cần cập nhật biến đơn giản mà không muốn lock nặng.
- Ví dụ: increment counter, compare-and-swap cho trạng thái.
- Atomic giảm contention trong workload cao.
- Tuy nhiên lạm dụng có thể làm code khó hiểu và khó maintain.

---

## Question 4: Mutex, RWLock và Spinlock nên chọn thế nào?

**Answer:**

- Mutex phù hợp trường hợp tổng quát.
- RWLock hiệu quả khi đọc nhiều, ghi ít.
- Spinlock hợp đoạn critical section rất ngắn, tránh context switch.
- Chọn sai primitive dễ gây contention hoặc đốt CPU vô ích.

---

## Question 5: Thread pool giải quyết vấn đề gì?

**Answer:**

- Tránh chi phí tạo/hủy thread liên tục.
- Giới hạn mức đồng thời để bảo vệ CPU và memory.
- Hỗ trợ queue task và backpressure.
- Backend server thường có pool riêng cho I/O, CPU-bound, background jobs.

---

## Question 6: Producer-Consumer pattern và bounded queue?

**Answer:**

- Producer tạo task, consumer xử lý task.
- Bounded queue đặt giới hạn bộ nhớ và tạo backpressure.
- Khi queue đầy: block, drop, hoặc trả lỗi tùy yêu cầu nghiệp vụ.
- Đây là pattern nền cho message processing và async pipeline.

---

## Question 7: Backpressure là gì trong hệ thống concurrent?

**Answer:**

- Backpressure là cơ chế báo upstream giảm tốc khi downstream quá tải.
- Thiếu backpressure thường dẫn đến queue phình to và OOM.
- Cách làm: giới hạn queue, semaphore, rate limit, circuit breaker.
- Mục tiêu là degrade gracefully thay vì sập toàn bộ.

---

## Question 8: Lock contention ảnh hưởng hiệu năng ra sao?

**Answer:**

- Nhiều thread tranh chấp cùng lock làm tăng wait time.
- CPU có thể bị giảm hiệu quả do block/wakeup liên tục.
- Cải thiện bằng giảm phạm vi lock, sharding lock, hoặc lock-free.
- Theo dõi bằng profiling và contention metrics.

---

## Question 9: Starvation và Livelock là gì?

**Answer:**

- Starvation: một task không bao giờ được tài nguyên.
- Livelock: các task liên tục thay đổi trạng thái nhưng không tiến triển.
- Cả hai khác deadlock vì hệ thống có vẻ vẫn "chạy".
- Giảm bằng fairness policy, random backoff, và scheduling hợp lý.

---

## Question 10: Memory model ảnh hưởng lập trình concurrent như thế nào?

**Answer:**

- CPU/compiler có thể reorder lệnh để tối ưu.
- Không có synchronization thì thread khác có thể thấy state không nhất quán.
- Cần hiểu happens-before, acquire/release semantics.
- Đây là lý do volatile/atomic/lock quan trọng trong code đa luồng.

---

## Question 11: Async/Await có thay thế hoàn toàn multi-threading không?

**Answer:**

- Không. Async giỏi cho I/O-bound, không tự làm CPU-bound nhanh hơn.
- CPU-bound vẫn cần thread/process pool hoặc job worker.
- Async giảm số thread cần dùng và tăng khả năng phục vụ kết nối.
- Nên kết hợp async + worker model theo loại tác vụ.

---

## Question 12: Cancellation và timeout nên thiết kế thế nào?

**Answer:**

- Mọi tác vụ dài nên nhận tín hiệu cancel từ context.
- Timeout đặt theo SLO và ngân sách tổng request chain.
- Nếu không cancel đúng, tác vụ "zombie" tiếp tục ngốn tài nguyên.
- Cần propagate timeout/cancel xuyên suốt call graph.

---

## Question 13: Concurrent data structure là gì?

**Answer:**

- Là cấu trúc dữ liệu hỗ trợ truy cập đa luồng an toàn.
- Ví dụ: concurrent map, lock-free queue.
- Chúng giảm lỗi sync thủ công và tối ưu contention tốt hơn.
- Tuy vậy cần hiểu semantics iterator, memory ordering, và chi phí ẩn.

---

## Question 14: Work stealing scheduler hoạt động ra sao?

**Answer:**

- Mỗi worker có deque task riêng.
- Khi worker rảnh, nó "steal" task từ worker khác.
- Cách này cân bằng tải tốt cho workload động.
- Framework hiện đại dùng rộng rãi để tối ưu CPU utilization.

---

## Question 15: Cách kiểm thử bug concurrency hiệu quả?

**Answer:**

- Viết stress test với số lượng thread/request lớn.
- Random hóa lịch thực thi, thêm delay có chủ đích.
- Dùng race detector, thread sanitizer, deadlock detector.
- Lặp test nhiều lần trong CI vì lỗi đồng thời thường không ổn định.
