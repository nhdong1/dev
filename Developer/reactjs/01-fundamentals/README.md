# 01 — Nền Tảng React (React Fundamentals)

> Hiểu vững phần nền tảng là điều kiện tiên quyết để học mọi thứ còn lại trong React. Phần này bao gồm JSX, Components, Props, State, xử lý sự kiện, render có điều kiện và Forms.

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Viết JSX (JavaScript XML — Cú Pháp Kết Hợp HTML Trong JavaScript) thành thạo và hiểu cách nó biên dịch sang JavaScript thuần
- [ ] Tạo và tổ chức Functional Components (Component Hàm) theo kiểu mô-đun
- [ ] Truyền và nhận Props (Properties — Thuộc Tính) giữa các component
- [ ] Quản lý State (Trạng Thái) cục bộ với `useState`
- [ ] Xử lý các sự kiện DOM qua Synthetic Event System (Hệ Thống Sự Kiện Tổng Hợp)
- [ ] Render (Hiển Thị) có điều kiện và render danh sách với `key`
- [ ] Xây dựng Controlled Components (Component Được Kiểm Soát) trong Forms

---

## 📁 Nội Dung

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-jsx-components.md](./1-jsx-components.md) | JSX, Functional Components, cú pháp cơ bản | Beginner |
| [2-props-state.md](./2-props-state.md) | Props, State, luồng dữ liệu một chiều | Beginner |
| [3-event-handling.md](./3-event-handling.md) | Xử lý sự kiện, Synthetic Events | Beginner |
| [4-conditional-list-rendering.md](./4-conditional-list-rendering.md) | Render có điều kiện, danh sách, key | Beginner |
| [5-forms-controlled.md](./5-forms-controlled.md) | Forms, Controlled vs Uncontrolled Components | Beginner–Intermediate |

---

## 🗺️ Lộ Trình Học

```
JSX & Components  →  Props & State  →  Events  →  Conditional/List Render  →  Forms
      ↓                    ↓              ↓                  ↓                    ↓
 "Cú pháp là gì?"   "Dữ liệu như thế nào?"  "Tương tác?"  "Hiển thị thông minh?"  "Thu thập input?"
```

**Thứ tự khuyến nghị:** Đọc tuần tự từ file 1 đến 5. Mỗi file xây dựng dựa trên kiến thức từ file trước.

---

## ⏱️ Thời Gian Ước Tính

| Hoạt Động | Thời Gian |
| --------- | --------- |
| Đọc lý thuyết (5 files) | 3–4 giờ |
| Thực hành code theo ví dụ | 3–4 giờ |
| Mini-project: Todo App | 3–5 giờ |
| **Tổng cộng** | **9–13 giờ** |

---

## 💡 Khái Niệm Cốt Lõi Cần Nắm Chắc

### 1. React là gì?

React là thư viện JavaScript dùng để xây dựng UI (User Interface — Giao Diện Người Dùng). Điểm khác biệt cốt lõi:

- **Declarative (Khai Báo):** Bạn mô tả UI trông như thế nào, React lo việc cập nhật DOM (Document Object Model — Mô Hình Đối Tượng Tài Liệu).
- **Component-based (Dựa Trên Component):** Chia nhỏ UI thành các phần tử độc lập, có thể tái sử dụng.
- **Virtual DOM (DOM Ảo):** React duy trì một bản sao DOM trong bộ nhớ, so sánh trước khi cập nhật DOM thật → tối ưu hiệu năng.

### 2. Mental Model (Mô Hình Tư Duy)

```
UI = f(state)
```

Giao diện là kết quả thuần túy của state. Khi state thay đổi, React tính toán lại UI và cập nhật chỉ những phần cần thiết.

### 3. Reconciliation (Đồng Bộ Hóa DOM)

Khi state/props thay đổi, React:
1. Gọi lại hàm component để tạo Virtual DOM mới
2. So sánh (diff) Virtual DOM mới với Virtual DOM cũ — gọi là **Reconciliation**
3. Chỉ cập nhật phần DOM thật đã thay đổi — gọi là **Commit Phase (Giai Đoạn Ghi Nhận)**

---

## 🔨 Mini-Project: Todo App

Sau khi hoàn thành cả 5 file, hãy xây dựng **Todo App** với yêu cầu:

```
✅ Thêm task mới (dùng form + controlled component)
✅ Hiển thị danh sách task (dùng list rendering + key)
✅ Đánh dấu task đã hoàn thành (dùng state + conditional rendering)
✅ Xóa task (dùng event handling)
✅ Lọc: All / Active / Completed (dùng conditional rendering)
```

**Gợi ý cấu trúc component:**

```
<TodoApp>          ← Quản lý state chính
  <TodoForm />     ← Input thêm task mới
  <TodoList>       ← Render danh sách
    <TodoItem />   ← Mỗi item
  </TodoList>
  <TodoFilter />   ← Nút lọc
</TodoApp>
```

---

## 🔗 Điều Hướng

- **Tiếp theo:** [02-hooks/](../02-hooks/) — React Hooks chuyên sâu
- **Quay lại:** [README.md tổng quan](../README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
