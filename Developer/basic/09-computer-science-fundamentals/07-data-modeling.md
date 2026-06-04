# Data Modeling - Interview Questions

## Question 1: Data modeling là gì trong backend?

**Answer:**

- Là quá trình biểu diễn domain thành cấu trúc dữ liệu lưu trữ/truy vấn.
- Mục tiêu: đúng nghiệp vụ, dễ mở rộng, truy vấn hiệu quả.
- Data model tốt giảm đáng kể độ phức tạp ứng dụng về sau.
- Đây là nền tảng trước khi nói đến indexing hay sharding.

---

## Question 2: Entity, Value Object, Aggregate khác nhau thế nào?

**Answer:**

- Entity có identity ổn định theo thời gian.
- Value Object định nghĩa bằng giá trị, thường immutable.
- Aggregate là cụm object với một aggregate root quản lý invariant.
- Thiết kế theo DDD giúp giới hạn side-effect và boundary rõ ràng.

---

## Question 3: Normalization và Denormalization trade-off ra sao?

**Answer:**

- Normalization giảm trùng lặp và tăng tính nhất quán.
- Denormalization tăng tốc đọc bằng cách lưu dư dữ liệu có kiểm soát.
- Hệ thống đọc nhiều thường denormalize ở điểm nóng.
- Cần cơ chế đồng bộ để tránh dữ liệu lệch kéo dài.

---

## Question 4: Khóa chính nên chọn surrogate key hay natural key?

**Answer:**

- Surrogate key (UUID, bigint id) ổn định và không phụ thuộc nghiệp vụ.
- Natural key có ý nghĩa business nhưng dễ thay đổi theo thời gian.
- Nhiều hệ thống dùng surrogate làm PK và unique constraint cho natural key.
- Cách này cân bằng hiệu năng và ràng buộc nghiệp vụ.

---

## Question 5: Cardinality ảnh hưởng schema như thế nào?

**Answer:**

- 1-1, 1-N, N-N quyết định cách tách bảng/collection.
- Quan hệ N-N cần bảng liên kết hoặc embedding tùy mô hình.
- Hiểu cardinality giúp dự đoán kích thước dữ liệu và truy vấn.
- Sai cardinality thường dẫn đến join phức tạp hoặc document quá to.

---

## Question 6: Khi nào dùng SQL, khi nào dùng NoSQL theo mô hình dữ liệu?

**Answer:**

- SQL phù hợp dữ liệu quan hệ, giao dịch chặt, truy vấn ad-hoc mạnh.
- NoSQL phù hợp scale ngang, schema linh hoạt, workload đặc thù.
- Chọn theo access pattern thay vì theo xu hướng công nghệ.
- Polyglot persistence là lựa chọn phổ biến trong hệ thống lớn.

---

## Question 7: Event sourcing tác động gì đến data model?

**Answer:**

- Lưu chuỗi sự kiện thay vì chỉ lưu trạng thái hiện tại.
- Tạo audit trail mạnh và khả năng replay.
- Đổi lại đọc trạng thái cần snapshot/projection để tránh chậm.
- Phù hợp domain cần lịch sử đầy đủ và truy vết chính xác.

---

## Question 8: CQRS liên quan thế nào đến data modeling?

**Answer:**

- Tách model ghi (write) và model đọc (read).
- Read model có thể tối ưu theo từng màn hình API cụ thể.
- Tăng độ phức tạp đồng bộ, nhưng cải thiện performance và linh hoạt.
- Thường đi kèm eventual consistency giữa hai phía.

---

## Question 9: Soft delete nên áp dụng thế nào?

**Answer:**

- Thêm cờ `deleted_at`/`is_deleted` thay vì xóa cứng.
- Giúp khôi phục dữ liệu và hỗ trợ audit/compliance.
- Cần đảm bảo mọi query mặc định lọc bản ghi đã xóa.
- Dữ liệu cũ nên có chiến lược purge định kỳ để tránh phình to.

---

## Question 10: Multi-tenant data model thường có lựa chọn nào?

**Answer:**

- Shared DB + shared schema + tenant_id.
- Shared DB + schema per tenant.
- Database per tenant.
- Chọn theo yêu cầu cô lập dữ liệu, chi phí vận hành, và scale.

---

## Question 11: Versioning schema và migration nên quản lý ra sao?

**Answer:**

- Migration phải idempotent và có thứ tự rõ.
- Tránh thay đổi phá vỡ backward compatibility đột ngột.
- Dùng expand-contract pattern để deploy an toàn.
- Cần kế hoạch rollback hoặc roll-forward rõ ràng.

---

## Question 12: Data integrity nên đặt ở app hay database?

**Answer:**

- Database đảm bảo ràng buộc cứng: PK, FK, unique, check.
- Application đảm bảo rule nghiệp vụ phức tạp theo ngữ cảnh.
- Tốt nhất là defense-in-depth: cả hai lớp cùng kiểm soát.
- Không nên phụ thuộc duy nhất vào một phía.

---

## Question 13: Time-series data nên mô hình như thế nào?

**Answer:**

- Thiết kế theo timestamp + dimension keys.
- Dùng partition theo thời gian để tối ưu ghi/đọc và retention.
- Aggregation precompute giúp dashboard nhanh hơn.
- Cần policy downsampling cho dữ liệu lâu năm.

---

## Question 14: Metadata-driven model là gì?

**Answer:**

- Lưu cấu hình/sơ đồ động thay vì hard-code toàn bộ cột.
- Tăng linh hoạt cho nghiệp vụ thay đổi nhanh.
- Đổi lại query phức tạp hơn và khó tối ưu index.
- Chỉ nên dùng khi mức biến thiên schema thực sự cao.

---

## Question 15: Nguyên tắc review một data model tốt?

**Answer:**

- Rõ ràng ownership và boundary của từng thực thể.
- Match access pattern thực tế, không tối ưu theo giả định mơ hồ.
- Có chiến lược index, migration, retention, và backup.
- Dễ quan sát, dễ kiểm tra tính đúng đắn theo thời gian.
