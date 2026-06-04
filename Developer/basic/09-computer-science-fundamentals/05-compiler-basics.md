# Compiler Basics - Interview Questions

## Question 1: Compiler và Interpreter khác nhau thế nào?

**Answer:**

- Compiler dịch toàn bộ source sang machine code/bytecode trước khi chạy.
- Interpreter đọc và thực thi từng phần tại runtime.
- Nhiều nền tảng hiện đại kết hợp cả hai (bytecode + JIT).
- Trade-off giữa startup time, tối ưu runtime, và tính linh hoạt.

---

## Question 2: Các phase cơ bản của compiler gồm gì?

**Answer:**

- Lexical analysis: tách token.
- Parsing: dựng AST theo grammar.
- Semantic analysis: kiểm tra kiểu, scope, ràng buộc.
- Optimization: tối ưu IR/AST.
- Code generation: sinh machine code hoặc bytecode.

---

## Question 3: AST là gì và vì sao quan trọng?

**Answer:**

- AST là biểu diễn cấu trúc cú pháp dạng cây.
- Giúp compiler thực hiện kiểm tra ngữ nghĩa và tối ưu.
- Công cụ lint/formatter cũng thường thao tác trên AST.
- Hiểu AST giúp backend engineer dùng static analysis hiệu quả hơn.

---

## Question 4: Type checking diễn ra ở compile time hay runtime?

**Answer:**

- Ngôn ngữ static kiểm tra phần lớn lúc compile time.
- Ngôn ngữ dynamic đẩy nhiều kiểm tra sang runtime.
- Static type giúp bắt lỗi sớm và hỗ trợ refactor an toàn.
- Dynamic linh hoạt hơn nhưng cần test mạnh để bù rủi ro.

---

## Question 5: Intermediate Representation (IR) có lợi ích gì?

**Answer:**

- IR tách front-end ngôn ngữ khỏi back-end kiến trúc máy.
- Cho phép tái sử dụng optimizer và code generator.
- Ví dụ phổ biến: LLVM IR, JVM bytecode.
- Đây là nền tảng cho tối ưu đa nền tảng hiệu quả.

---

## Question 6: JIT và AOT nên chọn khi nào?

**Answer:**

- JIT tối ưu theo profile runtime, thường cho peak performance tốt.
- AOT có startup nhanh, phù hợp môi trường serverless hoặc binary nhỏ gọn.
- JIT có warm-up cost; AOT có thể mất cơ hội tối ưu động.
- Nhiều hệ sinh thái hỗn hợp để cân bằng latency và throughput.

---

## Question 7: Inlining là gì và rủi ro gì?

**Answer:**

- Inlining thay lời gọi hàm bằng thân hàm để giảm call overhead.
- Có thể mở khóa thêm tối ưu như constant propagation.
- Quá mức sẽ làm tăng code size, ảnh hưởng instruction cache.
- Compiler cần heuristic để quyết định hàm nào nên inline.

---

## Question 8: Escape analysis giúp tối ưu gì?

**Answer:**

- Phân tích object có "thoát" khỏi scope hiện tại hay không.
- Nếu không thoát, object có thể đặt trên stack thay vì heap.
- Giảm GC pressure và allocation cost.
- Hiệu quả rõ trong code tạo nhiều object ngắn hạn.

---

## Question 9: Dead code elimination hoạt động như thế nào?

**Answer:**

- Loại bỏ đoạn code không ảnh hưởng kết quả cuối.
- Ví dụ nhánh không bao giờ chạy hoặc biến tính rồi không dùng.
- Giúp giảm kích thước binary và chi phí runtime.
- Cần thận trọng với side-effect và hành vi reflection.

---

## Question 10: Linker làm gì trong quá trình build?

**Answer:**

- Kết hợp object files và thư viện thành executable/shared library.
- Resolve symbol giữa các module.
- Thực hiện relocation địa chỉ.
- Lỗi linker thường liên quan thiếu symbol hoặc xung đột phiên bản thư viện.

---

## Question 11: ABI là gì và tại sao backend engineer cần biết?

**Answer:**

- ABI định nghĩa cách binary tương tác ở mức thấp.
- Bao gồm calling convention, layout kiểu dữ liệu, name mangling.
- Quan trọng khi tích hợp native extension hoặc FFI.
- Sai ABI có thể gây crash dù code compile thành công.

---

## Question 12: Undefined behavior nghĩa là gì?

**Answer:**

- Là hành vi không được ngôn ngữ định nghĩa kết quả cụ thể.
- Compiler có thể tối ưu dựa trên giả định UB không xảy ra.
- Ví dụ C/C++: tràn số signed, dereference con trỏ null.
- UB gây bug khó tái hiện và có thể lộ bảo mật.

---

## Question 13: Branch prediction liên quan gì tới hiệu năng code?

**Answer:**

- CPU dự đoán nhánh để giữ pipeline chạy liên tục.
- Dự đoán sai gây flush pipeline, tăng latency.
- Code với nhánh khó dự đoán có thể chậm đáng kể.
- Bố trí dữ liệu/logic hợp lý giúp tăng khả năng dự đoán đúng.

---

## Question 14: Vì sao release build nhanh hơn debug build?

**Answer:**

- Debug bật symbol và tắt hoặc giảm mức tối ưu.
- Release bật nhiều tối ưu như inlining, vectorization, DCE.
- Hành vi timing giữa hai chế độ có thể khác đáng kể.
- Benchmark phải chạy trên release với cấu hình gần production.

---

## Question 15: Compiler warnings nên xử lý theo nguyên tắc nào?

**Answer:**

- Xem warning như tín hiệu lỗi tiềm ẩn, không bỏ qua có hệ thống.
- Bật mức warning cao trong CI.
- Dùng quy tắc "warning-free" cho code mới.
- Chỉ suppress khi có lý do rõ và kèm giải thích ngắn gọn.
