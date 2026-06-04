# JSX & Functional Components — Cú Pháp Cơ Bản

> JSX (JavaScript XML) là cú pháp mở rộng cho phép viết markup giống HTML trực tiếp trong JavaScript. Functional Components (Component Hàm) là đơn vị xây dựng cơ bản của mọi ứng dụng React hiện đại.

---

## 📌 Mục Lục

1. [JSX là gì?](#1-jsx-là-gì)
2. [Quy tắc cú pháp JSX](#2-quy-tắc-cú-pháp-jsx)
3. [Functional Components](#3-functional-components)
4. [Tổ hợp Component (Component Composition)](#4-tổ-hợp-component)
5. [Fragments](#5-fragments)
6. [Câu hỏi phỏng vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. JSX Là Gì?

**JSX — JavaScript XML** là cú pháp mở rộng (syntactic sugar) cho phép viết HTML-like markup bên trong file JavaScript. JSX không phải HTML, không phải string — nó là JavaScript.

### JSX Biên Dịch Thành Gì?

Trước khi chạy trên trình duyệt, công cụ build (Babel, SWC, hay React Compiler) biên dịch JSX thành lời gọi `React.createElement()`:

```jsx
// Bạn viết (JSX):
const element = <h1 className="title">Xin chào React!</h1>;

// Sau khi biên dịch (JavaScript thuần):
const element = React.createElement(
  "h1",
  { className: "title" },
  "Xin chào React!"
);
```

### Tại Sao Dùng JSX?

| Không có JSX | Có JSX |
| ------------ | ------ |
| `React.createElement('div', null, React.createElement('p', null, 'Nội dung'))` | `<div><p>Nội dung</p></div>` |
| Khó đọc, khó bảo trì | Trực quan, giống HTML quen thuộc |

**Kết luận:** JSX không bắt buộc, nhưng gần như toàn bộ codebase React thực tế đều dùng nó.

---

## 2. Quy Tắc Cú Pháp JSX

### Quy Tắc 1: Chỉ Có Một Root Element (Phần Tử Gốc Duy Nhất)

```jsx
// ❌ Sai — hai phần tử gốc
return (
  <h1>Tiêu đề</h1>
  <p>Đoạn văn</p>
);

// ✅ Đúng — bọc trong một phần tử gốc
return (
  <div>
    <h1>Tiêu đề</h1>
    <p>Đoạn văn</p>
  </div>
);

// ✅ Đúng — dùng Fragment (xem phần 5)
return (
  <>
    <h1>Tiêu đề</h1>
    <p>Đoạn văn</p>
  </>
);
```

### Quy Tắc 2: Tất Cả Tags Phải Được Đóng

```jsx
// ❌ Sai — <img> không đóng
<img src="logo.png">

// ✅ Đúng — self-closing tag (thẻ tự đóng)
<img src="logo.png" />
<br />
<input type="text" />
```

### Quy Tắc 3: Dùng `className` Thay Vì `class`

Vì `class` là từ khóa dành riêng trong JavaScript:

```jsx
// ❌ Sai
<div class="container">

// ✅ Đúng
<div className="container">
```

### Quy Tắc 4: camelCase Cho Thuộc Tính HTML

```jsx
// HTML:  onclick, tabindex, for
// JSX:   onClick, tabIndex, htmlFor

<button onClick={handleClick} tabIndex={0}>Bấm</button>
<label htmlFor="email">Email:</label>
<input id="email" type="email" />
```

### Quy Tắc 5: Biểu Thức JavaScript Dùng Dấu Ngoặc Nhọn `{}`

```jsx
const name = "Minh";
const isLoggedIn = true;

return (
  <div>
    {/* Hiển thị biến */}
    <p>Xin chào, {name}!</p>

    {/* Gọi hàm */}
    <p>Ngày: {new Date().toLocaleDateString("vi-VN")}</p>

    {/* Biểu thức số học */}
    <p>2 + 2 = {2 + 2}</p>

    {/* Template literal */}
    <p className={`card ${isLoggedIn ? "active" : "inactive"}`}>Card</p>

    {/* Không thể dùng: if, for, switch — chỉ expressions (biểu thức) */}
  </div>
);
```

### Quy Tắc 6: Comments Trong JSX

```jsx
return (
  <div>
    {/* Đây là comment hợp lệ trong JSX */}
    <p>Nội dung</p>
    {/* Comment nhiều dòng
        cũng được viết như vậy */}
  </div>
);
```

### Quy Tắc 7: Style Là Object (Đối Tượng)

```jsx
// ❌ Sai — style là string như trong HTML
<div style="color: red; font-size: 16px">

// ✅ Đúng — style là object với camelCase keys
<div style={{ color: "red", fontSize: "16px", marginTop: 8 }}>
```

> Dấu `{{}}` ngoài là JSX expression, dấu `{}` trong là object literal.

---

## 3. Functional Components

### Component Là Gì?

**Component (Thành Phần)** là hàm JavaScript trả về JSX mô tả UI. React sẽ gọi hàm này, lấy kết quả JSX và render (hiển thị) lên màn hình.

**Quy tắc đặt tên:** Component bắt đầu bằng **chữ hoa** để phân biệt với HTML tags thông thường (chữ thường).

```jsx
// ✅ Component — chữ hoa đầu
function Welcome() { ... }
const Welcome = () => { ... }

// ❌ React sẽ hiểu là HTML tag thông thường
function welcome() { ... }
```

### Cách Định Nghĩa Functional Component

**Cách 1: Function Declaration (Khai Báo Hàm)**

```jsx
function Greeting() {
  return (
    <div>
      <h1>Chào mừng!</h1>
      <p>Đây là ứng dụng React đầu tiên của bạn.</p>
    </div>
  );
}

export default Greeting;
```

**Cách 2: Arrow Function (Hàm Mũi Tên) — phổ biến hơn**

```jsx
const Greeting = () => {
  return (
    <div>
      <h1>Chào mừng!</h1>
      <p>Đây là ứng dụng React đầu tiên của bạn.</p>
    </div>
  );
};

export default Greeting;
```

**Cách 3: Arrow Function với Implicit Return (Trả Về Ngầm Định)**

```jsx
// Dùng khi JSX đơn giản, không cần body block
const Button = () => (
  <button type="button">Bấm vào đây</button>
);
```

### Component Có Thể Chứa Logic

```jsx
function UserCard() {
  // Logic JavaScript bình thường bên trong hàm
  const currentHour = new Date().getHours();
  const greeting = currentHour < 12 ? "Chào buổi sáng" : "Chào buổi chiều";
  const fullName = "Nguyễn Văn An";

  return (
    <div className="user-card">
      <h2>{greeting}, {fullName}!</h2>
      <p>Hôm nay là {new Date().toLocaleDateString("vi-VN")}</p>
    </div>
  );
}
```

### Return Null — Ẩn Component

```jsx
function HiddenMessage({ show }) {
  // Trả về null = không render gì cả
  if (!show) return null;

  return <p>Bạn đang thấy thông báo này!</p>;
}
```

---

## 4. Tổ Hợp Component (Component Composition)

React khuyến khích xây dựng UI bằng cách **lắp ghép nhiều component nhỏ** thành component lớn hơn — giống như ghép các khối Lego.

```jsx
// Component nhỏ, độc lập
function Avatar({ imageUrl, altText }) {
  return (
    <img
      src={imageUrl}
      alt={altText}
      className="avatar"
      width={64}
      height={64}
    />
  );
}

function UserInfo({ name, title }) {
  return (
    <div>
      <h3>{name}</h3>
      <p>{title}</p>
    </div>
  );
}

// Component lớn hơn, lắp ghép các component nhỏ
function ProfileCard({ user }) {
  return (
    <div className="profile-card">
      <Avatar imageUrl={user.avatarUrl} altText={user.name} />
      <UserInfo name={user.name} title={user.jobTitle} />
    </div>
  );
}

// Component trang, sử dụng ProfileCard
function TeamPage() {
  const teamMembers = [
    { id: 1, name: "An Nguyễn", jobTitle: "Frontend Engineer", avatarUrl: "/an.jpg" },
    { id: 2, name: "Bình Trần", jobTitle: "Backend Engineer", avatarUrl: "/binh.jpg" },
  ];

  return (
    <main>
      <h1>Đội Ngũ Của Chúng Tôi</h1>
      {teamMembers.map((member) => (
        <ProfileCard key={member.id} user={member} />
      ))}
    </main>
  );
}
```

### Nguyên Tắc Tách Component (When To Split)

Tách component khi:
1. **Quá phức tạp** — component có quá nhiều trách nhiệm
2. **Cần tái sử dụng** — logic hoặc UI xuất hiện nhiều nơi
3. **Để test riêng** — muốn viết unit test cho một phần nhỏ
4. **Hiệu năng** — muốn tránh re-render (hiển thị lại) không cần thiết

---

## 5. Fragments

**Fragment** cho phép nhóm nhiều element mà không tạo thêm node DOM thừa.

```jsx
import { Fragment } from "react";

// Cách 1: Fragment tường minh — cần khi muốn truyền key
function List({ items }) {
  return (
    <Fragment>
      <h2>Danh sách</h2>
      <ul>
        {items.map((item) => (
          <Fragment key={item.id}>
            <dt>{item.name}</dt>
            <dd>{item.description}</dd>
          </Fragment>
        ))}
      </ul>
    </Fragment>
  );
}

// Cách 2: Short syntax (Cú Pháp Viết Tắt) <> </> — phổ biến nhất
function Card() {
  return (
    <>
      <h1>Tiêu đề</h1>
      <p>Nội dung</p>
    </>
  );
}
```

**Khi nào cần Fragment thay vì `<div>`?**

```jsx
// Vấn đề: render bảng bị sai cấu trúc
function TableRow({ name, score }) {
  return (
    <div> {/* ❌ <div> không được phép nằm trong <tr> */}
      <td>{name}</td>
      <td>{score}</td>
    </div>
  );
}

// Giải pháp: dùng Fragment
function TableRow({ name, score }) {
  return (
    <> {/* ✅ Fragment không tạo thêm DOM node */}
      <td>{name}</td>
      <td>{score}</td>
    </>
  );
}
```

---

## 6. Câu Hỏi Phỏng Vấn

### Q1: JSX có phải là HTML không?

**Trả lời:** Không. JSX là cú pháp mở rộng JavaScript, được biên dịch sang `React.createElement()`. Nó _trông_ giống HTML nhưng có những khác biệt quan trọng: dùng `className` thay `class`, `htmlFor` thay `for`, style là object, tất cả tags phải đóng, chỉ một root element. Trình duyệt không hiểu JSX trực tiếp — cần bước biên dịch trung gian.

### Q2: Tại sao tên Component phải bắt đầu bằng chữ hoa?

**Trả lời:** React dùng quy ước này để phân biệt HTML elements (`<div>`, `<p>`) với React components (`<Button>`, `<Card>`). Trong quá trình biên dịch: chữ thường → `React.createElement("div", ...)` (string); chữ hoa → `React.createElement(Button, ...)` (tham chiếu hàm). Nếu đặt tên component chữ thường, React sẽ hiểu là HTML tag không tồn tại.

### Q3: Sự khác biệt giữa Element và Component?

**Trả lời:**
- **Element (Phần Tử):** Là object plain JavaScript mô tả những gì sẽ xuất hiện trên màn hình. Ví dụ: `<div className="card">`. Bất biến (immutable) sau khi tạo.
- **Component (Thành Phần):** Là hàm hoặc class nhận input (props) và trả về elements. Component là "blueprint" (bản thiết kế), element là "instance" (thể hiện cụ thể).

### Q4: Tại sao không thể dùng `if` statement trực tiếp trong JSX?

**Trả lời:** JSX chỉ chấp nhận **expressions** (biểu thức — thứ có giá trị trả về), không chấp nhận **statements** (câu lệnh — không có giá trị trả về). `if/else`, `for`, `switch` đều là statements. Thay vào đó dùng: ternary operator `condition ? a : b`, logical AND `condition && element`, hoặc tính toán giá trị trước khi return.

### Q5: Virtual DOM hoạt động như thế nào?

**Trả lời:** Virtual DOM (DOM Ảo) là bản sao nhẹ của DOM thật, lưu trong bộ nhớ dưới dạng JavaScript object. Khi state/props thay đổi:
1. React tạo Virtual DOM mới
2. So sánh (diff) với Virtual DOM cũ — **Reconciliation (Đồng Bộ Hóa)**
3. Tính toán thay đổi tối thiểu cần thiết — **Diffing Algorithm (Thuật Toán So Sánh)**
4. Áp dụng thay đổi đó lên DOM thật — **Commit Phase (Giai Đoạn Ghi Nhận)**

Mục tiêu: giảm thiểu thao tác DOM thật (vốn chậm) → hiệu năng tốt hơn.

---

## 🔗 Điều Hướng

- **Tiếp theo:** [2-props-state.md](./2-props-state.md) — Props, State và luồng dữ liệu một chiều
- **Quay lại:** [README.md](./README.md) — Tổng quan Fundamentals
