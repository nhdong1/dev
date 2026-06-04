# Props & State — Dữ Liệu Trong React

> Props (Properties — Thuộc Tính) và State (Trạng Thái) là hai cơ chế cốt lõi để quản lý dữ liệu trong React. Hiểu rõ sự khác biệt giữa chúng là nền tảng để xây dựng ứng dụng React đúng đắn.

---

## 📌 Mục Lục

1. [Props — Truyền Dữ Liệu Từ Cha Xuống Con](#1-props)
2. [State — Trạng Thái Nội Bộ](#2-state)
3. [Props vs State — So Sánh](#3-props-vs-state)
4. [Luồng Dữ Liệu Một Chiều (One-Way Data Flow)](#4-luồng-dữ-liệu-một-chiều)
5. [Lifting State Up — Nâng State Lên](#5-lifting-state-up)
6. [Câu hỏi phỏng vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Props

### Props Là Gì?

**Props (Properties — Thuộc Tính)** là dữ liệu được truyền từ component cha xuống component con. Props là **read-only (chỉ đọc)** — component con không được phép thay đổi props nhận được.

Hãy nghĩ Props như tham số của một hàm JavaScript thông thường.

### Cách Truyền Props

```jsx
// Component cha truyền props
function App() {
  return (
    <UserCard
      name="Nguyễn Văn An"
      age={28}
      isActive={true}
      hobbies={["đọc sách", "code", "chơi cầu lông"]}
      address={{ city: "Hà Nội", district: "Cầu Giấy" }}
    />
  );
}

// Component con nhận props
function UserCard(props) {
  return (
    <div>
      <h2>{props.name}</h2>
      <p>Tuổi: {props.age}</p>
      <p>Trạng thái: {props.isActive ? "Đang hoạt động" : "Không hoạt động"}</p>
      <p>Sở thích: {props.hobbies.join(", ")}</p>
      <p>Thành phố: {props.address.city}</p>
    </div>
  );
}
```

### Destructuring Props (Phân Rã Props) — Cách Khuyến Nghị

```jsx
// Thay vì props.name, props.age... dùng destructuring
function UserCard({ name, age, isActive, hobbies, address }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>Tuổi: {age}</p>
      <p>Trạng thái: {isActive ? "Đang hoạt động" : "Không hoạt động"}</p>
      <ul>
        {hobbies.map((hobby, index) => (
          <li key={index}>{hobby}</li>
        ))}
      </ul>
      <p>Thành phố: {address.city}, {address.district}</p>
    </div>
  );
}
```

### Default Props (Giá Trị Props Mặc Định)

```jsx
// Cách 1: Default values trong destructuring — khuyến nghị
function Button({ label = "Bấm", variant = "primary", disabled = false }) {
  return (
    <button
      className={`btn btn-${variant}`}
      disabled={disabled}
    >
      {label}
    </button>
  );
}

// Cách 2: defaultProps (cú pháp cũ, vẫn hoạt động nhưng ít dùng hơn)
Button.defaultProps = {
  label: "Bấm",
  variant: "primary",
  disabled: false,
};
```

### Props Đặc Biệt: `children`

`children` là prop đặc biệt chứa nội dung bên trong thẻ đóng-mở của component:

```jsx
function Card({ title, children }) {
  return (
    <div className="card">
      <div className="card-header">
        <h3>{title}</h3>
      </div>
      <div className="card-body">
        {children}  {/* Nội dung được truyền vào giữa thẻ */}
      </div>
    </div>
  );
}

// Sử dụng
function App() {
  return (
    <Card title="Thông Báo">
      <p>Đây là nội dung của card.</p>
      <button>Đóng</button>
    </Card>
  );
}
```

### Truyền Hàm Qua Props (Callback Props)

```jsx
function ParentComponent() {
  function handleChildClick(message) {
    alert(`Con gửi lên: ${message}`);
  }

  return <ChildButton onClick={handleChildClick} />;
}

function ChildButton({ onClick }) {
  return (
    <button onClick={() => onClick("Xin chào từ con!")}>
      Gửi lên cha
    </button>
  );
}
```

### Spread Props (Trải Props)

```jsx
const buttonProps = {
  type: "submit",
  className: "btn-primary",
  disabled: false,
};

// Truyền tất cả props cùng lúc dùng spread operator
function Form() {
  return <button {...buttonProps}>Gửi</button>;
}
```

> **Cảnh báo:** Dùng spread props cẩn thận — tránh truyền props không cần thiết xuống DOM elements (sẽ gây cảnh báo trong console).

---

## 2. State

### State Là Gì?

**State (Trạng Thái)** là dữ liệu **nội bộ** của component, có thể **thay đổi theo thời gian**. Khi state thay đổi, React **tự động re-render (hiển thị lại)** component để phản ánh trạng thái mới.

### `useState` Hook

```jsx
import { useState } from "react";

function Counter() {
  // useState trả về [giá trị hiện tại, hàm cập nhật]
  const [count, setCount] = useState(0); // 0 là giá trị khởi tạo

  return (
    <div>
      <p>Đếm: {count}</p>
      <button onClick={() => setCount(count + 1)}>Tăng</button>
      <button onClick={() => setCount(count - 1)}>Giảm</button>
      <button onClick={() => setCount(0)}>Đặt lại</button>
    </div>
  );
}
```

### Các Loại Dữ Liệu Trong State

```jsx
function StateExamples() {
  // Number (Số)
  const [count, setCount] = useState(0);

  // String (Chuỗi)
  const [name, setName] = useState("");

  // Boolean (Luận lý)
  const [isOpen, setIsOpen] = useState(false);

  // Array (Mảng)
  const [items, setItems] = useState([]);

  // Object (Đối Tượng)
  const [user, setUser] = useState({ name: "", email: "" });

  // null
  const [selectedItem, setSelectedItem] = useState(null);
}
```

### Cập Nhật State Đúng Cách

#### Với kiểu nguyên thủy (Primitive Types):

```jsx
const [count, setCount] = useState(0);

// ✅ Trực tiếp
setCount(5);

// ✅ Functional update (Cập Nhật Hàm) — dùng khi giá trị mới phụ thuộc vào giá trị cũ
setCount((prevCount) => prevCount + 1);

// ❌ Không bao giờ thay đổi trực tiếp
count = count + 1; // React không phát hiện thay đổi → không re-render
```

#### Với Object (Đối Tượng):

```jsx
const [user, setUser] = useState({ name: "An", age: 28, city: "Hà Nội" });

// ✅ Spread để giữ các field khác, chỉ cập nhật field cần thiết
setUser({ ...user, name: "Bình" });

// ✅ Functional update với spread
setUser((prev) => ({ ...prev, age: 29 }));

// ❌ Mutate (Thay Đổi Trực Tiếp) trực tiếp — React không phát hiện
user.name = "Bình"; // Sai!
setUser(user);       // Cùng reference → React bỏ qua
```

#### Với Array (Mảng):

```jsx
const [items, setItems] = useState(["táo", "chuối"]);

// ✅ Thêm phần tử — spread tạo mảng mới
setItems([...items, "cam"]);

// ✅ Xóa phần tử — filter tạo mảng mới
setItems(items.filter((item) => item !== "chuối"));

// ✅ Cập nhật phần tử — map tạo mảng mới
setItems(items.map((item) => item === "táo" ? "xoài" : item));

// ❌ Mutate mảng trực tiếp
items.push("cam");    // Sai!
items.splice(0, 1);   // Sai!
```

### State Là Bất Đồng Bộ (Asynchronous)

```jsx
function AsyncStateExample() {
  const [count, setCount] = useState(0);

  function handleMultipleUpdates() {
    setCount(count + 1); // Không update ngay lập tức
    setCount(count + 1); // Vẫn đọc count cũ → kết quả chỉ +1, không phải +2
    console.log(count);  // Log ra giá trị cũ!
  }

  function handleCorrectMultipleUpdates() {
    // ✅ Functional update đảm bảo dùng giá trị mới nhất
    setCount((prev) => prev + 1); // +1
    setCount((prev) => prev + 1); // Dùng kết quả trước → +2 tổng cộng
  }

  return (
    <div>
      <p>{count}</p>
      <button onClick={handleMultipleUpdates}>Sai (+1)</button>
      <button onClick={handleCorrectMultipleUpdates}>Đúng (+2)</button>
    </div>
  );
}
```

### Batching — React Gộp Nhiều State Updates

Từ React 18, **Automatic Batching (Gộp Tự Động)** được áp dụng: React gộp nhiều `setState` trong cùng một event handler → chỉ re-render một lần.

```jsx
function handleClick() {
  setCount((c) => c + 1); // Không re-render ngay
  setName("An");           // Không re-render ngay
  setIsOpen(true);         // React gộp cả 3 → re-render 1 lần duy nhất
}
```

---

## 3. Props vs State — So Sánh

| Đặc Điểm | Props | State |
| --------- | ----- | ----- |
| **Nguồn gốc** | Từ component cha truyền xuống | Nội bộ component tự quản lý |
| **Có thể thay đổi?** | ❌ Không (read-only) | ✅ Có (qua setter function) |
| **Ai kiểm soát?** | Component cha | Chính component đó |
| **Khi nào re-render?** | Khi cha re-render với props mới | Khi gọi setter function |
| **Mục đích** | Cấu hình và truyền dữ liệu | Theo dõi dữ liệu thay đổi theo thời gian |

**Nguyên tắc quyết định:**

```
Dữ liệu này có thay đổi theo thời gian không?
├── Không → Dùng props hoặc hằng số
└── Có → Dùng state
      ├── Ai cần dữ liệu này?
      │   ├── Chỉ component này → State nội bộ
      │   └── Nhiều component → Nâng state lên (Lifting State Up)
```

---

## 4. Luồng Dữ Liệu Một Chiều (One-Way Data Flow)

React thực hiện **Unidirectional Data Flow (Luồng Dữ Liệu Một Chiều)**: dữ liệu chỉ chảy từ **cha xuống con** qua props. Con không thể trực tiếp thay đổi state của cha.

```
App (state: theme = "dark")
 │
 ├── Header (props: theme="dark")   ← nhận, không thể sửa
 │    └── Logo (props: theme="dark")
 │
 └── Content (props: theme="dark")  ← nhận, không thể sửa
      └── Article (props: theme="dark")
```

**Lợi ích:**
- **Predictable (Dự Đoán Được):** Dễ theo dõi dữ liệu thay đổi như thế nào
- **Debuggable (Dễ Debug):** Lỗi dễ tìm nguồn gốc hơn
- **Testable (Dễ Kiểm Thử):** Component độc lập với input/output rõ ràng

---

## 5. Lifting State Up — Nâng State Lên

Khi nhiều component cùng cần truy cập và thay đổi một state, **nâng state lên component cha chung gần nhất** (Closest Common Ancestor).

```jsx
// ❌ Vấn đề: mỗi component quản lý state riêng → không đồng bộ
function TemperatureApp() {
  return (
    <>
      <CelsiusInput />   {/* state: celsius riêng */}
      <FahrenheitInput /> {/* state: fahrenheit riêng */}
    </>
  );
}

// ✅ Giải pháp: Lifting State Up — nâng state lên cha
function TemperatureApp() {
  const [celsius, setCelsius] = useState(0);

  // Tính fahrenheit từ celsius
  const fahrenheit = (celsius * 9) / 5 + 32;

  return (
    <>
      <CelsiusInput
        value={celsius}
        onChange={setCelsius}
      />
      <FahrenheitInput
        value={fahrenheit}
        onChange={(f) => setCelsius(((f - 32) * 5) / 9)}
      />
      <p>
        {celsius}°C = {fahrenheit.toFixed(1)}°F
      </p>
    </>
  );
}

function CelsiusInput({ value, onChange }) {
  return (
    <label>
      Celsius:
      <input
        type="number"
        value={value}
        onChange={(e) => onChange(Number(e.target.value))}
      />
    </label>
  );
}

function FahrenheitInput({ value, onChange }) {
  return (
    <label>
      Fahrenheit:
      <input
        type="number"
        value={value}
        onChange={(e) => onChange(Number(e.target.value))}
      />
    </label>
  );
}
```

---

## 6. Câu Hỏi Phỏng Vấn

### Q1: Props và State khác nhau như thế nào?

**Trả lời:** Props là dữ liệu được truyền vào từ bên ngoài (component cha), chỉ đọc, không thể thay đổi từ bên trong component nhận. State là dữ liệu nội bộ, component tự quản lý và có thể thay đổi qua setter function. Props dùng để "cấu hình" component; State dùng để theo dõi dữ liệu thay đổi theo tương tác người dùng hoặc thời gian.

### Q2: Tại sao không được thay đổi state trực tiếp?

**Trả lời:** React phát hiện thay đổi state bằng cách **so sánh tham chiếu (reference equality)**. Nếu mutate object/array trực tiếp, reference giữ nguyên → React nghĩ không có gì thay đổi → không re-render. Bắt buộc phải tạo giá trị mới (new reference) khi gọi setter. Đây là lý do dùng spread operator `{...obj}`, `[...arr]` thay vì sửa trực tiếp.

### Q3: Giải thích Lifting State Up?

**Trả lời:** Khi hai hoặc nhiều component cần chia sẻ cùng state, ta "nâng" state đó lên component cha chung gần nhất. Cha giữ state và truyền xuống cho các con qua props. Khi con cần thay đổi state, nó gọi callback function được truyền qua props → cha cập nhật state → truyền xuống lại. Đây là ứng dụng của nguyên tắc "single source of truth" (nguồn dữ liệu duy nhất).

### Q4: useState hoạt động như thế nào bên trong?

**Trả lời:** React duy trì một danh sách state cho mỗi component instance (thể hiện). Mỗi lần gọi `useState`, React gắn state đó vào một "slot" (ô nhớ) cụ thể dựa trên **thứ tự gọi hook**. Đây là lý do hooks không được gọi trong vòng lặp, điều kiện, hay hàm lồng nhau — thứ tự phải nhất quán giữa các lần render để React khớp đúng state với hook tương ứng.

### Q5: Khi nào nên dùng state, khi nào nên dùng derived value (giá trị tính toán)?

**Trả lời:** Tránh lưu vào state những gì có thể **tính toán được từ state/props hiện có**. Ví dụ: nếu đã có `firstName` và `lastName` trong state, đừng tạo thêm `fullName` state — tính trực tiếp `const fullName = firstName + " " + lastName`. Lưu thừa dẫn đến trạng thái không nhất quán (inconsistent state) và khó maintain. Nguyên tắc: **State tối thiểu, tính toán tối đa.**

---

## 🔗 Điều Hướng

- **Tiếp theo:** [3-event-handling.md](./3-event-handling.md) — Xử lý sự kiện và Synthetic Events
- **Quay lại:** [1-jsx-components.md](./1-jsx-components.md) — JSX & Components
