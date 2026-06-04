# useState & useReducer — Quản Lý State Cục Bộ

> `useState` và `useReducer` là hai hooks cốt lõi để quản lý **state** (trạng thái) cục bộ trong Functional Components. `useState` dành cho state đơn giản; `useReducer` dành cho state phức tạp với nhiều transitions (chuyển đổi trạng thái).

---

## 📌 Mục Lục

1. [useState — Quản Lý State Đơn Giản](#1-usestate)
2. [Cập Nhật State Đúng Cách](#2-cập-nhật-state-đúng-cách)
3. [useState Với Object và Array](#3-usestate-với-object-và-array)
4. [useReducer — Quản Lý State Phức Tạp](#4-usereducer)
5. [So Sánh useState vs useReducer](#5-so-sánh)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. useState — Quản Lý State Đơn Giản

### Cú Pháp Cơ Bản

```jsx
const [state, setState] = useState(initialValue);
// state        — giá trị hiện tại
// setState     — hàm cập nhật state
// initialValue — giá trị khởi tạo ban đầu (chỉ dùng ở lần render đầu tiên)
```

### Ví Dụ Cơ Bản

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0); // khởi tạo count = 0

  return (
    <div>
      <p>Giá trị hiện tại: {count}</p>
      <button onClick={() => setCount(count + 1)}>Tăng</button>
      <button onClick={() => setCount(count - 1)}>Giảm</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

### Các Kiểu Giá Trị Của initialValue

```jsx
// Kiểu nguyên thủy (primitive types)
const [count, setCount] = useState(0);          // số
const [name, setName] = useState("");           // chuỗi rỗng
const [isOpen, setIsOpen] = useState(false);    // boolean
const [data, setData] = useState(null);         // null

// Object
const [user, setUser] = useState({ name: "", age: 0 });

// Array
const [items, setItems] = useState([]);

// Lazy initialization — tính toán phức tạp chỉ chạy một lần
const [list, setList] = useState(() => {
  // Hàm này chỉ được gọi ở lần render đầu tiên
  const saved = localStorage.getItem("myList");
  return saved ? JSON.parse(saved) : [];
});
```

> **Lazy Initialization (Khởi Tạo Lười):** Khi `initialValue` đòi hỏi tính toán nặng (đọc localStorage, lọc mảng lớn...), truyền vào một **hàm** thay vì giá trị trực tiếp. React sẽ chỉ gọi hàm đó ở lần render đầu tiên.

---

## 2. Cập Nhật State Đúng Cách

### Vấn Đề: State Là Snapshot (Ảnh Chụp Tại Thời Điểm)

State trong React **không thay đổi ngay lập tức** — mỗi lần render, component nhận được một "snapshot" (ảnh chụp) của state tại thời điểm đó.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleTripleClick() {
    // ❌ Bug: cả 3 lần gọi đều dùng count = 0 (snapshot hiện tại)
    setCount(count + 1); // 0 + 1 = 1
    setCount(count + 1); // 0 + 1 = 1 (vẫn dùng count = 0!)
    setCount(count + 1); // 0 + 1 = 1 (vẫn dùng count = 0!)
    // Kết quả: count = 1, không phải 3!
  }

  return <button onClick={handleTripleClick}>+3 (bị bug)</button>;
}
```

### Giải Pháp: Functional Update (Cập Nhật Dạng Hàm)

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleTripleClick() {
    // ✅ Functional update: nhận prevState (giá trị trước đó) mới nhất
    setCount(prev => prev + 1); // 0 → 1
    setCount(prev => prev + 1); // 1 → 2
    setCount(prev => prev + 1); // 2 → 3
    // Kết quả: count = 3 ✓
  }

  return <button onClick={handleTripleClick}>+3 (đúng)</button>;
}
```

**Quy tắc ngón tay cái:** Khi giá trị mới **phụ thuộc vào giá trị cũ**, luôn dùng **functional update** `setState(prev => newValue)`.

### Batching — Gom Nhóm Cập Nhật

React 18+ **tự động gom nhóm** (automatic batching) nhiều `setState` trong cùng một event handler thành một lần re-render duy nhất:

```jsx
function Form() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [age, setAge] = useState(0);

  function handleSubmit() {
    // React 18+ gom 3 setState này thành 1 lần re-render
    setName("Nguyễn Văn An");
    setEmail("an@example.com");
    setAge(25);
  }

  return <button onClick={handleSubmit}>Submit</button>;
}
```

---

## 3. useState Với Object và Array

### Nguyên Tắc Bất Biến (Immutability)

State trong React **phải được cập nhật bất biến (immutably)** — nghĩa là không được thay đổi object/array trực tiếp, mà phải tạo ra bản sao mới.

```jsx
// ❌ Sai — thay đổi trực tiếp (mutation), React không phát hiện được thay đổi
function BadExample() {
  const [user, setUser] = useState({ name: "An", age: 25 });

  function handleAgeChange() {
    user.age = 26;      // Mutate trực tiếp — sai!
    setUser(user);      // React thấy cùng reference → không re-render
  }
}

// ✅ Đúng — tạo bản sao mới với spread operator
function GoodExample() {
  const [user, setUser] = useState({ name: "An", age: 25 });

  function handleAgeChange() {
    setUser({ ...user, age: 26 }); // Spread tạo object mới + override age
  }

  function handleNameChange(newName) {
    setUser(prev => ({ ...prev, name: newName }));
  }
}
```

### Cập Nhật Mảng Đúng Cách

```jsx
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: "Học React", done: false },
    { id: 2, text: "Build project", done: false },
  ]);

  // ✅ Thêm phần tử — spread tạo mảng mới
  function addTodo(text) {
    setTodos(prev => [
      ...prev,
      { id: Date.now(), text, done: false },
    ]);
  }

  // ✅ Xóa phần tử — filter tạo mảng mới
  function removeTodo(id) {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  }

  // ✅ Cập nhật một phần tử — map tạo mảng mới
  function toggleTodo(id) {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      )
    );
  }

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <span style={{ textDecoration: todo.done ? "line-through" : "none" }}>
            {todo.text}
          </span>
          <button onClick={() => toggleTodo(todo.id)}>Toggle</button>
          <button onClick={() => removeTodo(todo.id)}>Xóa</button>
        </li>
      ))}
      <button onClick={() => addTodo("Task mới")}>Thêm</button>
    </ul>
  );
}
```

---

## 4. useReducer — Quản Lý State Phức Tạp

### Giới Thiệu

`useReducer` là giải pháp thay thế `useState` khi state có nhiều sub-values (giá trị con) hoặc logic cập nhật phức tạp. Nó lấy cảm hứng từ mô hình **Redux** — **Flux Pattern** (Mô Hình Flux).

### Cú Pháp

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
// state        — state hiện tại
// dispatch     — hàm gửi action
// reducer      — hàm pure (hàm thuần) xử lý transitions
// initialState — state ban đầu
```

### Reducer Function (Hàm Reducer)

```jsx
// Reducer: hàm thuần nhận (state hiện tại, action) và trả về state mới
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + 1 };
    case "DECREMENT":
      return { ...state, count: state.count - 1 };
    case "RESET":
      return { ...state, count: 0 };
    case "SET":
      return { ...state, count: action.payload };
    default:
      // Luôn có default case để tránh lỗi
      throw new Error(`Unknown action type: ${action.type}`);
  }
}
```

### Ví Dụ Đầy Đủ — Shopping Cart (Giỏ Hàng)

```jsx
import { useReducer } from "react";

// Định nghĩa initial state (state ban đầu)
const initialState = {
  items: [],
  totalItems: 0,
  totalPrice: 0,
};

// Reducer thuần — không có side effects
function cartReducer(state, action) {
  switch (action.type) {
    case "ADD_ITEM": {
      const existingItem = state.items.find(item => item.id === action.payload.id);
      if (existingItem) {
        return {
          ...state,
          items: state.items.map(item =>
            item.id === action.payload.id
              ? { ...item, quantity: item.quantity + 1 }
              : item
          ),
          totalItems: state.totalItems + 1,
          totalPrice: state.totalPrice + action.payload.price,
        };
      }
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }],
        totalItems: state.totalItems + 1,
        totalPrice: state.totalPrice + action.payload.price,
      };
    }

    case "REMOVE_ITEM":
      return {
        ...state,
        items: state.items.filter(item => item.id !== action.payload.id),
        totalItems: state.totalItems - action.payload.quantity,
        totalPrice: state.totalPrice - (action.payload.price * action.payload.quantity),
      };

    case "CLEAR_CART":
      return initialState;

    default:
      return state;
  }
}

// Component sử dụng useReducer
function ShoppingCart() {
  const [cart, dispatch] = useReducer(cartReducer, initialState);

  const addItem = (product) => {
    dispatch({ type: "ADD_ITEM", payload: product });
  };

  const removeItem = (item) => {
    dispatch({ type: "REMOVE_ITEM", payload: item });
  };

  const clearCart = () => {
    dispatch({ type: "CLEAR_CART" });
  };

  return (
    <div>
      <h2>Giỏ hàng ({cart.totalItems} sản phẩm)</h2>
      <p>Tổng tiền: {cart.totalPrice.toLocaleString("vi-VN")}đ</p>
      {cart.items.map(item => (
        <div key={item.id}>
          <span>{item.name} x{item.quantity}</span>
          <button onClick={() => removeItem(item)}>Xóa</button>
        </div>
      ))}
      <button onClick={clearCart}>Xóa tất cả</button>
      <button onClick={() => addItem({ id: 1, name: "Sản phẩm A", price: 50000 })}>
        Thêm Sản phẩm A
      </button>
    </div>
  );
}
```

### useReducer Với Lazy Initialization

```jsx
function init(initialCount) {
  // Hàm init — tính toán state ban đầu phức tạp
  return { count: initialCount, history: [] };
}

function Counter({ initialCount }) {
  // Tham số thứ 3 của useReducer: hàm init
  const [state, dispatch] = useReducer(reducer, initialCount, init);

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "INCREMENT" })}>+</button>
    </div>
  );
}
```

---

## 5. So Sánh useState vs useReducer

| Tiêu Chí | `useState` | `useReducer` |
| -------- | ---------- | ------------ |
| **Độ phức tạp** | State đơn giản | State phức tạp, nhiều fields |
| **Logic cập nhật** | Đơn giản, inline | Phức tạp, tập trung trong reducer |
| **Số lượng setState** | Ít (1-3) | Nhiều loại transitions |
| **Testing** | Khó test logic | Reducer là hàm thuần, dễ test |
| **Debug** | Khó trace | Action có tên rõ ràng, dễ trace |
| **Boilerplate** | Ít | Nhiều hơn (actions, reducer) |
| **Performance** | Tương đương | Tương đương |

### Khi Nào Dùng useReducer?

```
✅ Dùng useReducer khi:
  - State có ≥ 3-4 fields liên quan đến nhau
  - Logic cập nhật phức tạp (nhiều điều kiện, nhiều loại updates)
  - Next state phụ thuộc vào previous state theo nhiều cách
  - Cần undo/redo history
  - Cần test logic update state độc lập với UI
  - Nhiều sub-components cần trigger cùng một loại updates

✅ Dùng useState khi:
  - State đơn giản: số, chuỗi, boolean
  - Logic cập nhật đơn giản, ít điều kiện
  - Không có nhiều loại transitions
  - Muốn code ngắn gọn, ít boilerplate
```

---

## 6. Câu Hỏi Phỏng Vấn

### Q1: Tại sao không được cập nhật state trực tiếp (`state.property = value`)?

**Trả lời:** React sử dụng **Object.is()** để so sánh state cũ và mới. Nếu mutate trực tiếp, reference (địa chỉ bộ nhớ) của object không đổi → React nghĩ state không thay đổi → không trigger re-render. Phải tạo object mới (spread `{...state}`, `Object.assign`, hoặc `Array.from`) để React nhận diện được thay đổi.

### Q2: Sự khác biệt giữa `setState(newValue)` và `setState(prev => newValue)`?

**Trả lời:**
- `setState(newValue)`: Gán trực tiếp giá trị mới. Vấn đề: bên trong closure có thể capture (bắt) giá trị state cũ (stale closure — closure lỗi thời), đặc biệt trong setTimeout, async functions.
- `setState(prev => newValue)`: Luôn nhận `prev` là giá trị **mới nhất** từ queue (hàng đợi) React. An toàn hơn khi cần multiple updates hoặc dùng trong async code.

### Q3: Khi nào nên dùng nhiều useState vs một useState với object?

**Trả lời:** Nguyên tắc: **nhóm state thường thay đổi cùng nhau**. Ví dụ:
- `useState({ width, height })` — tốt vì width/height luôn cập nhật cùng nhau
- `useState(name)` + `useState(email)` — tốt vì name và email độc lập
- Tránh một object khổng lồ chứa tất cả state không liên quan — khó đọc, dễ gây re-render thừa

### Q4: Giải thích Stale Closure trong useState?

**Trả lời:** Stale Closure (Closure Lỗi Thời) xảy ra khi một hàm "nhớ" giá trị state cũ từ lúc nó được tạo ra, không phải giá trị hiện tại:

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // ❌ Stale closure: count luôn là 0 (giá trị lúc effect chạy lần đầu)
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []); // [] dependency rỗng — effect chỉ chạy một lần

  return <p>{count}</p>;
}

function TimerFixed() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // ✅ Functional update: luôn dùng giá trị mới nhất
      setCount(prev => prev + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <p>{count}</p>;
}
```

### Q5: useReducer có phải là thay thế hoàn toàn cho useState không?

**Trả lời:** Không. Cả hai tồn tại song song vì mục đích khác nhau. `useState` là syntactic sugar (cú pháp rút gọn) của `useReducer` — thực ra `useState` được implement bằng `useReducer` nội bộ. Dùng `useState` khi đơn giản, `useReducer` khi phức tạp. Không cần dùng `useReducer` cho mọi state vì nó thêm boilerplate không cần thiết.

---

## 🔗 Điều Hướng

- **Trở về:** [README.md](./README.md) — Tổng quan Hooks
- **Tiếp theo:** [2-useeffect-uselayout.md](./2-useeffect-uselayout.md) — useEffect và useLayoutEffect
- **Xem thêm:** [4-usememo-usecallback.md](./4-usememo-usecallback.md) — Tối ưu hiệu năng với Memoization

**Cập Nhật Lần Cuối:** 2026-06-04 | **React Version:** v19 (stable)
