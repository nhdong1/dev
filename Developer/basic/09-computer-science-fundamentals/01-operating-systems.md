# Operating Systems - Interview Questions

## Question 1: Process và Thread khác nhau như thế nào?

**Answer:**

- Process là một instance độc lập của chương trình, có virtual memory space riêng.
- Thread là đơn vị thực thi nhẹ bên trong process, chia sẻ memory và tài nguyên.
- Context switch giữa process thường tốn kém hơn vì đổi address space.
- Backend service hay dùng multi-thread để tăng throughput, nhưng cần đồng bộ dữ liệu.

---

## Question 2: Context Switching là gì và vì sao ảnh hưởng hiệu năng?

**Answer:**

- Context switch là việc CPU tạm dừng task hiện tại để chạy task khác.
- OS phải lưu/restore register, program counter, stack pointer.
- Nếu có quá nhiều thread runnable, CPU mất thời gian switch thay vì xử lý business logic.
- Cách giảm: giới hạn thread count, dùng async I/O, tối ưu lock contention.

---

## Question 3: Scheduling algorithm phổ biến trong OS?

**Answer:**

- FCFS: đơn giản, nhưng dễ bị convoy effect.
- SJF/SRTF: tối ưu waiting time trung bình, khó ước lượng burst time.
- Round Robin: công bằng cho interactive workloads, phụ thuộc quantum.
- Priority scheduling: cần có aging để tránh starvation.
- Linux CFS ưu tiên fairness, phù hợp cho hệ thống đa nhiệm.

---

## Question 4: User mode và Kernel mode khác nhau ở điểm nào?

**Answer:**

- User mode bị giới hạn quyền truy cập tài nguyên hệ thống.
- Kernel mode có đặc quyền cao nhất, được truy cập hardware trực tiếp.
- System call là cầu nối từ user mode vào kernel mode.
- Tách biệt này giúp an toàn: lỗi app user không làm sập toàn bộ hệ thống dễ dàng.

---

## Question 5: System call là gì? Ví dụ trong backend?

**Answer:**

- System call là API của kernel cho chương trình user.
- Ví dụ: `read`, `write`, `open`, `socket`, `epoll_wait`.
- Backend server xử lý network request thông qua nhiều system call I/O.
- Số lần syscall cao có thể tăng overhead, nên thường dùng batching và buffering.

---

## Question 6: Virtual Memory hoạt động ra sao?

**Answer:**

- Mỗi process nhìn thấy không gian địa chỉ ảo liên tục.
- MMU + page table ánh xạ virtual address sang physical address.
- Nếu page chưa có trong RAM, xảy ra page fault và OS nạp từ disk.
- Lợi ích: isolation, sử dụng RAM hiệu quả, chạy được process lớn hơn RAM vật lý.

---

## Question 7: Paging và Segmentation khác nhau thế nào?

**Answer:**

- Paging chia memory thành page/frame có kích thước cố định.
- Segmentation chia theo logic chương trình (code, stack, heap), kích thước biến đổi.
- Paging giảm external fragmentation, nhưng có internal fragmentation.
- Nhiều kiến trúc hiện đại ưu tiên paging; segmentation hạn chế hơn.

---

## Question 8: Deadlock là gì? 4 điều kiện Coffman?

**Answer:**

- Deadlock xảy ra khi các thread/process chờ nhau vô hạn.
- 4 điều kiện: mutual exclusion, hold and wait, no preemption, circular wait.
- Phá vỡ deadlock bằng lock ordering, timeout, hoặc detect và recover.
- Trong backend, transaction lock và distributed lock là nơi dễ gặp deadlock.

---

## Question 9: Race condition xảy ra khi nào?

**Answer:**

- Xảy ra khi kết quả phụ thuộc thứ tự xen kẽ của các thread.
- Nếu truy cập biến chung mà không đồng bộ, state có thể sai.
- Giải pháp: mutex, semaphore, atomic operations, immutability.
- Ngoài ra cần test với stress/load để lộ ra race condition hiếm.

---

## Question 10: Mutex và Semaphore khác nhau như thế nào?

**Answer:**

- Mutex: khóa nhị phân, thường có owner, một thread lock thì thread đó unlock.
- Semaphore: bộ đếm, cho phép N thread truy cập tài nguyên.
- Binary semaphore có thể giống mutex về hành vi, nhưng semantics khác.
- Dùng sai có thể gây deadlock hoặc priority inversion.

---

## Question 11: Inter-process communication (IPC) gồm những cách nào?

**Answer:**

- Pipe/Named pipe: truyền dữ liệu theo stream.
- Message queue: giao tiếp theo message, tách producer-consumer.
- Shared memory: nhanh nhất, cần đồng bộ chặt chẽ.
- Socket: dùng cho process khác máy hoặc cùng máy.
- Lựa chọn phụ thuộc latency, độ phức tạp, và độ tin cậy.

---

## Question 12: File descriptor là gì trong Linux?

**Answer:**

- File descriptor (fd) là số nguyên đại diện cho tài nguyên I/O đang mở.
- `0`, `1`, `2` lần lượt là stdin, stdout, stderr.
- Socket, file, pipe đều được thao tác qua fd.
- Memory leak kiểu backend phổ biến là quên đóng fd, dẫn đến "too many open files".

---

## Question 13: Buffering và Caching khác nhau?

**Answer:**

- Buffering: tạm giữ dữ liệu khi đang truyền/ghi để cân bằng tốc độ.
- Caching: lưu kết quả để tái sử dụng cho lần truy cập sau.
- Buffer giải quyết mismatch throughput; cache giải quyết latency.
- Trong backend, cần phân biệt để không tối ưu nhầm mục tiêu.

---

## Question 14: Swap memory là gì và tác động đến server?

**Answer:**

- Swap là vùng disk dùng làm memory mở rộng khi RAM thiếu.
- Truy cập swap chậm hơn RAM rất nhiều, có thể gây latency spike.
- Server backend quan trọng thường giảm swap usage để tránh pause bất ngờ.
- Nên monitor `major page fault`, `swap in/out`, và memory pressure.

---

## Question 15: Làm sao debug hiệu năng ở tầng OS cho backend service?

**Answer:**

- Kiểm tra CPU, memory, I/O wait, context switch, load average.
- Dùng công cụ: `top`, `htop`, `vmstat`, `iostat`, `strace`, `perf`.
- Tìm nghẽn: CPU-bound, lock contention, disk bottleneck, network backlog.
- Kết hợp metrics hệ thống với application tracing để tìm root cause nhanh.
