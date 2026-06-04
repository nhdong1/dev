# Networking - Interview Questions

## Question 1: Mô hình OSI và TCP/IP khác nhau như thế nào?

**Answer:**

- OSI có 7 tầng, mang tính khái niệm để phân tích.
- TCP/IP thường dùng thực tế với 4-5 tầng.
- Khi debug backend, ta ánh xạ vấn đề theo tầng: app, transport, network.
- Cách nghĩ theo tầng giúp khoanh vùng nhanh lỗi timeout, packet loss, hay DNS.

---

## Question 2: TCP bắt tay 3 bước hoạt động ra sao?

**Answer:**

- Client gửi SYN để khởi tạo kết nối.
- Server phản hồi SYN-ACK.
- Client gửi ACK để hoàn tất.
- Mục tiêu: đồng bộ sequence number và xác nhận hai chiều sẵn sàng.

---

## Question 3: Vì sao cần 4 bước đóng TCP thay vì 3?

**Answer:**

- Mỗi chiều truyền dữ liệu của TCP đóng độc lập (full-duplex).
- FIN từ một phía chỉ báo "tôi gửi xong", chưa chắc phía kia gửi xong.
- Do đó thường cần FIN/ACK theo từng chiều.
- Trạng thái TIME_WAIT giúp xử lý packet trễ và tránh kết nối cũ gây nhiễu.

---

## Question 4: TCP và UDP nên chọn khi nào?

**Answer:**

- TCP: tin cậy, có thứ tự, có kiểm soát tắc nghẽn.
- UDP: nhẹ, độ trễ thấp, không đảm bảo delivery.
- Backend API thông thường chọn TCP (HTTP/gRPC).
- UDP phù hợp telemetry, streaming thời gian thực, hoặc protocol tự quản reliability.

---

## Question 5: Congestion control trong TCP là gì?

**Answer:**

- TCP điều chỉnh tốc độ gửi để tránh làm nghẽn mạng.
- Cơ chế phổ biến: slow start, congestion avoidance, fast retransmit.
- Biến quan trọng: congestion window (cwnd).
- Ảnh hưởng trực tiếp đến throughput và tail latency của service giữa các region.

---

## Question 6: DNS resolution diễn ra như thế nào?

**Answer:**

- Client hỏi resolver cục bộ.
- Resolver truy vấn root -> TLD -> authoritative name server.
- Kết quả được cache theo TTL.
- Lỗi DNS thường gây sự cố diện rộng dù app không thay đổi code.

---

## Question 7: HTTP/1.1, HTTP/2, HTTP/3 khác nhau ở điểm chính nào?

**Answer:**

- HTTP/1.1: text-based, head-of-line blocking ở cấp kết nối.
- HTTP/2: binary framing, multiplexing nhiều stream trên một kết nối TCP.
- HTTP/3: chạy trên QUIC (UDP), giảm handshake và cải thiện khi mất gói.
- Backend hiện đại ưu tiên HTTP/2 cho microservices, HTTP/3 cho edge/public traffic.

---

## Question 8: TLS handshake làm gì và tốn chi phí ở đâu?

**Answer:**

- Xác thực server qua certificate.
- Thương lượng cipher suite và tạo session key đối xứng.
- Chi phí nằm ở round-trip và crypto operations.
- Tối ưu bằng TLS session resumption, keep-alive, và offload ở load balancer.

---

## Question 9: NAT là gì và ảnh hưởng gì đến hệ thống?

**Answer:**

- NAT ánh xạ IP private sang IP public.
- Giúp tiết kiệm IPv4 và tăng lớp che chắn mạng nội bộ.
- Có thể làm khó debug nguồn gốc client và giới hạn số cổng đồng thời.
- Cần theo dõi port exhaustion ở NAT gateway khi outbound traffic lớn.

---

## Question 10: Load balancing L4 và L7 khác nhau thế nào?

**Answer:**

- L4 cân bằng theo IP/port, nhanh và đơn giản.
- L7 hiểu HTTP path/header, routing linh hoạt hơn.
- L7 phù hợp cho canary, A/B testing, auth tại gateway.
- Nhiều hệ thống dùng kết hợp: L4 ở biên, L7 ở tầng ứng dụng.

---

## Question 11: Retry storm là gì và phòng tránh ra sao?

**Answer:**

- Retry storm xảy ra khi nhiều client retry đồng thời làm hệ thống tệ hơn.
- Cần exponential backoff + jitter để dàn đều request.
- Đặt timeout hợp lý và giới hạn retry budget.
- Kết hợp circuit breaker để fail-fast khi downstream đang quá tải.

---

## Question 12: Idempotency trong network request nghĩa là gì?

**Answer:**

- Idempotent: gọi nhiều lần cho cùng input vẫn cho cùng kết quả cuối.
- Quan trọng khi có retry do timeout/network lỗi.
- Thực thi bằng idempotency key và lưu kết quả theo key.
- Đặc biệt cần cho payment, order creation, và webhook processing.

---

## Question 13: Latency, Bandwidth, Throughput, Jitter khác nhau ra sao?

**Answer:**

- Latency: thời gian đi-về của dữ liệu.
- Bandwidth: khả năng tải tối đa trên đường truyền.
- Throughput: lưu lượng thực tế đạt được.
- Jitter: độ dao động latency theo thời gian.
- Backend realtime nhạy với jitter, không chỉ latency trung bình.

---

## Question 14: Socket backlog là gì?

**Answer:**

- Backlog là hàng chờ kết nối chờ accept ở server.
- Nếu đầy, client có thể bị timeout hoặc nhận reset.
- Cần tuning theo traffic spike và năng lực xử lý của app.
- Quan sát thêm SYN queue, accept queue và metrics kernel liên quan.

---

## Question 15: Cách debug sự cố mạng trong production?

**Answer:**

- Kiểm tra từ DNS -> TCP handshake -> TLS -> HTTP status.
- Dùng `ping`, `traceroute`, `dig`, `ss`, `tcpdump`, metrics LB.
- So sánh lỗi theo AZ/region để phát hiện network partition cục bộ.
- Kết hợp logs gateway + tracing để xác định điểm nghẽn chính xác.
