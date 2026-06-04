# Complexity Analysis - Interview Questions

## Question 1: Big-O, Big-Theta, Big-Omega khác nhau thế nào?

**Answer:**

- Big-O: cận trên tiệm cận.
- Big-Omega: cận dưới tiệm cận.
- Big-Theta: cận chặt hai phía.
- Phỏng vấn thường dùng Big-O, nhưng hiểu đủ 3 giúp phân tích chính xác hơn.

---

## Question 2: Time complexity và Space complexity nên cân bằng ra sao?

**Answer:**

- Giảm thời gian thường tốn thêm bộ nhớ (trade space for time).
- Trên backend, bộ nhớ tăng quá mức có thể làm GC và cache hit xấu đi.
- Cần tối ưu theo bottleneck thực tế chứ không theo trực giác.
- Luôn đo benchmark thay vì chỉ suy luận lý thuyết.

---

## Question 3: Amortized analysis là gì?

**Answer:**

- Phân tích chi phí trung bình trên chuỗi thao tác.
- Ví dụ dynamic array append: thỉnh thoảng resize tốn O(n), trung bình vẫn O(1).
- Giúp đánh giá đúng cấu trúc có spike ngắn hạn.
- Quan trọng cho thiết kế API có workload dài hạn.

---

## Question 4: Best/Average/Worst case khác nhau thế nào?

**Answer:**

- Best case thường ít giá trị nếu hiếm gặp.
- Average case cần giả định phân phối input rõ ràng.
- Worst case hữu ích cho đảm bảo SLA/SLO.
- Hệ thống production nên ưu tiên kiểm soát tail behavior.

---

## Question 5: Vì sao hằng số ẩn vẫn quan trọng trong thực tế?

**Answer:**

- O(n) với hằng số lớn có thể chậm hơn O(n log n) trong miền dữ liệu cụ thể.
- CPU cache, branch prediction, vectorization ảnh hưởng mạnh.
- Kết luận chỉ từ Big-O dễ gây tối ưu sai mục tiêu.
- Cần kết hợp complexity + profiling số liệu thật.

---

## Question 6: Hash table có thực sự O(1) không?

**Answer:**

- Trung bình O(1), nhưng worst-case có thể O(n) khi collision nặng.
- Chất lượng hash function và load factor quyết định hiệu năng.
- Resize và rehash có chi phí đột biến.
- Trong môi trường adversarial, cần cơ chế chống hash-flooding.

---

## Question 7: Binary search điều kiện áp dụng là gì?

**Answer:**

- Dữ liệu phải có thứ tự theo tiêu chí tìm kiếm.
- Mỗi bước loại bỏ nửa không gian tìm kiếm, O(log n).
- Lỗi thường gặp: sai điều kiện biên, overflow khi tính mid.
- Biến thể lower/upper bound rất hữu ích cho backend query logic.

---

## Question 8: Complexity của các thuật toán sort phổ biến?

**Answer:**

- QuickSort: trung bình O(n log n), worst O(n^2).
- MergeSort: luôn O(n log n), cần thêm bộ nhớ O(n).
- HeapSort: O(n log n), in-place tốt hơn về bộ nhớ.
- Chọn thuật toán theo ổn định, bộ nhớ, và đặc tính dữ liệu.

---

## Question 9: BFS và DFS khi nào phù hợp?

**Answer:**

- BFS tìm đường ngắn nhất trên đồ thị không trọng số.
- DFS phù hợp duyệt sâu, phát hiện cycle, topological utility.
- BFS tốn memory nhiều hơn do queue theo tầng.
- Lựa chọn phụ thuộc mục tiêu bài toán và giới hạn tài nguyên.

---

## Question 10: Dynamic Programming nhận diện như thế nào?

**Answer:**

- Bài toán có overlapping subproblems và optimal substructure.
- Có thể giải bằng top-down memoization hoặc bottom-up tabulation.
- DP giảm độ phức tạp từ exponential xuống polynomial trong nhiều bài.
- Đổi lại tăng memory, cần tối ưu state khi cần.

---

## Question 11: Complexity của truy vấn database liên hệ gì với thuật toán?

**Answer:**

- Query plan thực chất là thuật toán trên cấu trúc dữ liệu index/table.
- Full scan gần O(n), index seek gần O(log n) hoặc tốt hơn tùy cấu trúc.
- Join strategy ảnh hưởng mạnh khi dữ liệu lớn.
- Hiểu complexity giúp viết SQL đúng và dự đoán chi phí tốt hơn.

---

## Question 12: Cách phân tích thuật toán đệ quy?

**Answer:**

- Dùng recurrence relation, ví dụ Master Theorem.
- Theo dõi độ sâu ngăn xếp để ước lượng space complexity.
- Chú ý base case để tránh recursion vô hạn.
- Nhiều ngôn ngữ không tối ưu tail-call nên cần cẩn thận stack overflow.

---

## Question 13: Online algorithm và offline algorithm khác nhau?

**Answer:**

- Online xử lý dữ liệu đến đâu quyết định đến đó.
- Offline có toàn bộ input trước khi tính toán.
- Backend streaming và realtime analytics thường cần online algorithm.
- Trade-off giữa tối ưu toàn cục và độ trễ phản hồi.

---

## Question 14: Khi nào dùng approximation algorithm?

**Answer:**

- Với bài toán NP-hard, lời giải tối ưu tuyệt đối quá tốn chi phí.
- Approximation cung cấp lời giải gần tối ưu với thời gian chấp nhận được.
- Thực tế production thường ưu tiên "đủ tốt và nhanh".
- Cần định nghĩa rõ quality bound cho nghiệp vụ.

---

## Question 15: Quy trình tối ưu complexity trong code thực tế?

**Answer:**

- Bước 1: Đo đạc hotspot bằng profiling.
- Bước 2: Xác định cấu trúc dữ liệu/thuật toán gây nghẽn.
- Bước 3: Cải tiến có kiểm soát, đo lại trước/sau.
- Bước 4: Đảm bảo đúng đắn bằng test để tránh regression.
