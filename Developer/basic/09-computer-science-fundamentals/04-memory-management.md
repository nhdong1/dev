# Memory Management - Interview Questions

## Question 1: Stack và Heap khác nhau như thế nào?

**Answer:**

- Stack lưu biến cục bộ và call frame, quản lý theo LIFO.
- Heap cấp phát động, vòng đời linh hoạt hơn.
- Stack nhanh nhưng kích thước hạn chế; heap linh hoạt nhưng quản lý phức tạp.
- Lỗi phổ biến backend: lạm dụng heap gây GC pressure hoặc memory fragmentation.

---

## Question 2: Garbage Collection (GC) giải quyết vấn đề gì?

**Answer:**

- GC tự động thu hồi object không còn reachable.
- Giảm lỗi memory leak kiểu "quên free".
- Đổi lại có chi phí pause hoặc background CPU.
- Mục tiêu là cân bằng throughput, latency, memory footprint.

---

## Question 3: Generational GC là gì?

**Answer:**

- Giả định "đa số object sống ngắn".
- Chia heap thành young và old generation.
- Thu gom young thường xuyên, old ít hơn.
- Giúp tối ưu hiệu năng vì tránh scan toàn bộ heap mọi lần.

---

## Question 4: Memory leak có thể xảy ra trong ngôn ngữ có GC không?

**Answer:**

- Có, nếu object vẫn còn tham chiếu dù không còn cần.
- Ví dụ: cache không giới hạn, listener không hủy, map giữ key mãi.
- Leak làm heap tăng dần, GC chạy nhiều hơn, latency tăng.
- Cần dùng profiling heap dump và kiểm soát lifetime rõ ràng.

---

## Question 5: Fragmentation là gì?

**Answer:**

- External fragmentation: vùng trống rời rạc, khó cấp phát block lớn.
- Internal fragmentation: lãng phí bên trong block đã cấp phát.
- Hệ thống cấp phát bộ nhớ cần chiến lược fit và compaction phù hợp.
- Ở service chạy lâu, fragmentation có thể tăng memory usage theo thời gian.

---

## Question 6: Paging ảnh hưởng hiệu năng bộ nhớ thế nào?

**Answer:**

- Paging cho phép virtual memory nhưng tạo chi phí tra page table.
- TLB cache mapping giúp giảm chi phí dịch địa chỉ.
- TLB miss hoặc page fault nhiều làm giảm mạnh hiệu năng.
- Tối ưu locality truy cập giúp tận dụng cache/TLB tốt hơn.

---

## Question 7: Locality of Reference là gì?

**Answer:**

- Temporal locality: dữ liệu vừa dùng sẽ sớm dùng lại.
- Spatial locality: dữ liệu gần nhau thường được truy cập cùng.
- Thiết kế data structure tốt tận dụng cache line CPU.
- Locality kém làm cache miss tăng và CPU stall nhiều.

---

## Question 8: Copy-on-Write (CoW) hoạt động ra sao?

**Answer:**

- Ban đầu chia sẻ cùng trang nhớ chỉ đọc.
- Khi có ghi, hệ thống mới copy trang đó cho bên ghi.
- CoW tiết kiệm bộ nhớ và tăng tốc thao tác clone/fork.
- Trong backend, CoW hữu ích khi pre-fork worker model.

---

## Question 9: Memory pool/arena allocator dùng để làm gì?

**Answer:**

- Cấp phát theo block lớn và tái sử dụng nhanh.
- Giảm overhead malloc/free liên tục.
- Hữu ích cho workload nhiều object nhỏ, vòng đời giống nhau.
- Đổi lại có thể lãng phí bộ nhớ nếu thiết kế pool không phù hợp.

---

## Question 10: Object retention do cache nên kiểm soát thế nào?

**Answer:**

- Đặt giới hạn dung lượng hoặc số phần tử.
- Dùng eviction policy: LRU/LFU/TTL.
- Theo dõi hit ratio và memory usage để tuning.
- Không nên "cache everything" nếu chưa có số liệu chứng minh lợi ích.

---

## Question 11: Stop-the-world pause ảnh hưởng gì?

**Answer:**

- Một số pha GC cần dừng thread ứng dụng.
- Pause dài làm p99/p999 latency tăng mạnh.
- Hệ thống realtime hoặc low-latency rất nhạy với hiện tượng này.
- Tối ưu bằng tuning heap, giảm object churn, chọn GC algorithm phù hợp.

---

## Question 12: RSS, Heap size, Working set khác nhau thế nào?

**Answer:**

- RSS: lượng RAM tiến trình đang chiếm thực tế.
- Heap size: bộ nhớ cấp phát cho heap managed/unmanaged.
- Working set: phần dữ liệu đang được truy cập tích cực.
- Nhìn riêng từng số dễ hiểu sai, cần đọc theo ngữ cảnh tổng thể.

---

## Question 13: Dangling pointer và use-after-free là gì?

**Answer:**

- Dangling pointer trỏ đến vùng nhớ đã giải phóng.
- Use-after-free gây crash hoặc lỗ hổng bảo mật nghiêm trọng.
- Thường gặp ở C/C++, ít hơn ở ngôn ngữ managed.
- Phòng tránh bằng ownership rõ ràng, smart pointer, sanitizer.

---

## Question 14: Zero-copy I/O là gì?

**Answer:**

- Tránh copy dữ liệu nhiều lần giữa kernel và user space.
- API thường gặp: `sendfile`, `splice`, memory-mapped file.
- Giảm CPU usage và tăng throughput cho truyền file lớn.
- Cần cân nhắc giới hạn portability và độ phức tạp khi áp dụng.

---

## Question 15: Cách điều tra vấn đề memory trong production?

**Answer:**

- Theo dõi trend memory theo thời gian, không chỉ snapshot.
- Thu thập heap dump khi có tăng bất thường.
- Correlate với deploy, traffic pattern, và thay đổi tính năng.
- Dùng cảnh báo theo slope tăng memory để phát hiện sớm leak.
