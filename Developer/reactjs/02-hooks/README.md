# React Hooks — Tổng Quan Toàn Diện

> Hooks là các hàm đặc biệt cho phép Functional Components (Component Hàm) sử dụng state, lifecycle và các tính năng React khác mà trước đây chỉ Class Components mới có thể dùng. Kể từ React 16.8, Hooks đã trở thành cách viết React được khuyến nghị.

---

## 📌 Mục Lục

1. [Hooks Là Gì Và Tại Sao Cần?](#1-hooks-là-gì-và-tại-sao-cần)
2. [Rules of Hooks — Quy Tắc Bắt Buộc](#2-rules-of-hooks)
3. [Phân Loại Hooks](#3-phân-loại-hooks)
4. [Sơ Đồ Chọn Hook](#4-sơ-đồ-chọn-hook)
5. [Danh Sách File Trong Module](#5-danh-sách-file)
6. [Câu Hỏi Phỏng Vấn Tổng Quan](#6-câu-hỏi-phỏng-vấn)

---

## 1. Hooks Là Gì Và Tại Sao Cần?

### Vấn Đề Trước Hooks (React < 16.8)

Trước khi Hooks ra đời, chỉ **Class Components** (Component Lớp) mới có thể:
- Lưu trữ và cập nhật **state** (trạng thái)
- Dùng **lifecycle methods** (phương thức vòng đời) như `componentDidMount`, `componentDidUpdate`, `componentWillUnmount`
- Tái sử dụng logic thông qua **HOC** (Higher-Order Components — Component Bậc Cao) hoặc **Render Props** — phức tạp và khó đọc

```jsx
// ❌ Trước Hooks — Class Component phức tạp
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    this.handleClick = this.handleClick.bind(this);
  }

  componentDidMount() {
    document.title = `Số đếm: ${this.state.count}`;
  }

  componentDidUpdate() {
    document.title = `Số đếm: ${this.state.count}`;
  }

  handleClick() {
    this.setState({ count: this.state.count + 1 });
  }

  render() {
    return (
      <button onClick={this.handleClick}>
        Đếm: {this.state.count}
      </button>
    );
  }
}
```

```jsx
// ✅ Sau Hooks — Functional Component gọn gàng hơn rất nhiều
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `Số đếm: ${count}`;
  }, [count]);

  return (
    <button onClick={() => setCount(count + 1)}>
      Đếm: {count}
    </button>
  );
}
```

### Lợi Ích Của Hooks

| Lợi Ích | Giải Thích |
| ------- | ---------- |
| **Tái sử dụng logic** | Trích xuất stateful logic thành Custom Hooks, dùng lại dễ dàng |
| **Code ngắn gọn** | Loại bỏ boilerplate của class (constructor, bind, this) |
| **Dễ đọc hơn** | Logic liên quan được nhóm cùng nhau, không bị tách theo lifecycle |
| **Test dễ hơn** | Custom Hooks là hàm thuần, có thể test độc lập |
| **Không có `this`** | Tránh sự nhầm lẫn về ngữ cảnh `this` trong JavaScript |

---

## 2. Rules of Hooks — Quy Tắc Bắt Buộc

React enforces (bắt buộc tuân thủ) hai quy tắc cứng khi sử dụng Hooks. Vi phạm sẽ gây ra lỗi khó debug.

### Quy Tắc 1: Chỉ Gọi Hooks Ở Cấp Cao Nhất (Top Level)

```jsx
// ❌ KHÔNG được gọi hook trong điều kiện
function BadComponent({ condition }) {
  if (condition) {
    const [value, setValue] = useState(0); // Lỗi!
  }
  return <div />;
}

// ❌ KHÔNG được gọi hook trong vòng lặp
function AnotherBadComponent({ items }) {
  items.forEach(item => {
    const [data, setData] = useState(item); // Lỗi!
  });
  return <div />;
}

// ✅ Luôn gọi hook ở cấp cao nhất của component
function GoodComponent({ condition }) {
  const [value, setValue] = useState(0); // Đúng — luôn được gọi

  if (condition) {
    // Logic điều kiện ở đây, KHÔNG phải hook
  }
  return <div>{value}</div>;
}
```

**Tại sao?** React dùng thứ tự gọi hook để ánh xạ state với đúng hook. Nếu thứ tự thay đổi giữa các lần render, React sẽ không biết state nào thuộc hook nào.

### Quy Tắc 2: Chỉ Gọi Hooks Trong React Functions

```jsx
// ❌ KHÔNG gọi hook trong hàm JavaScript thông thường
function regularFunction() {
  const [value] = useState(0); // Lỗi!
}

// ❌ KHÔNG gọi hook trong class component
class MyClass extends React.Component {
  render() {
    const [value] = useState(0); // Lỗi!
  }
}

// ✅ Gọi hook trong Functional Component
function MyComponent() {
  const [value] = useState(0); // Đúng
  return <div>{value}</div>;
}

// ✅ Gọi hook trong Custom Hook (tên bắt đầu bằng "use")
function useMyCustomHook() {
  const [value] = useState(0); // Đúng
  return value;
}
```

**Công cụ hỗ trợ:** Plugin ESLint `eslint-plugin-react-hooks` tự động kiểm tra và cảnh báo vi phạm.

---

## 3. Phân Loại Hooks

### Built-in Hooks — Hooks Tích Hợp Sẵn (React 16.8+)

#### Quản Lý State (State Management)

| Hook | Mục Đích | Khi Nào Dùng |
| ---- | -------- | ------------ |
| `useState` | State đơn giản | Giá trị cục bộ: số, chuỗi, boolean, object nhỏ |
| `useReducer` | State phức tạp | Nhiều sub-values, transitions phức tạp, giống Redux |

#### Side Effects — Tác Dụng Phụ

| Hook | Mục Đích | Khi Nào Dùng |
| ---- | -------- | ------------ |
| `useEffect` | Side effects bất đồng bộ | Fetch API, subscriptions, DOM manipulation |
| `useLayoutEffect` | Side effects đồng bộ với DOM paint | Đo kích thước DOM, animations |

#### Context & Refs

| Hook | Mục Đích | Khi Nào Dùng |
| ---- | -------- | ------------ |
| `useContext` | Đọc giá trị từ Context | Tránh prop drilling, theme, auth |
| `useRef` | Tham chiếu DOM hoặc lưu mutable value | DOM refs, timer IDs, previous values |

#### Tối Ưu Hiệu Năng (Performance Optimization)

| Hook | Mục Đích | Khi Nào Dùng |
| ---- | -------- | ------------ |
| `useMemo` | Ghi nhớ giá trị tính toán | Tính toán nặng, dependencies ổn định |
| `useCallback` | Ghi nhớ hàm | Tránh re-render children nhận hàm qua props |

#### Hooks Tối Ưu UX (React 18+)

| Hook | Mục Đích | Khi Nào Dùng |
| ---- | -------- | ------------ |
| `useId` | Tạo ID duy nhất | Accessibility (a11y), form labels |
| `useTransition` | Đánh dấu update không khẩn cấp | Search, navigation, tab switching |
| `useDeferredValue` | Trì hoãn cập nhật giá trị | Input với kết quả lọc nặng |
| `useDebugValue` | Debug label trong DevTools | Custom Hooks |

### React v19 Hooks Mới

| Hook | Mục Đích |
| ---- | -------- |
| `use()` | Đọc Promise hoặc Context trong render (có thể dùng trong điều kiện) |
| `useActionState()` | Quản lý state từ kết quả Form Actions |
| `useFormStatus()` | Đọc trạng thái pending của form cha |
| `useOptimistic()` | Cập nhật UI lạc quan trước khi server xác nhận |

---

## 4. Sơ Đồ Chọn Hook

```
Cần quản lý state?
├─ State đơn giản, ít logic → useState
├─ State phức tạp, nhiều transitions → useReducer
└─ State cần chia sẻ toàn cục → useContext + useState/useReducer

Cần chạy code sau khi render?
├─ Fetch data, event listeners, subscriptions → useEffect
└─ Đo DOM, animation layout → useLayoutEffect

Cần tham chiếu tới DOM hoặc lưu giá trị không trigger re-render?
└─ useRef

Cần tối ưu hiệu năng?
├─ Tính toán nặng cần cache kết quả → useMemo
└─ Hàm truyền xuống component con → useCallback

Cần ID cho accessibility?
└─ useId

Cần update UI mượt mà khi có heavy computation?
├─ Đánh dấu update thấp ưu tiên → useTransition
└─ Trì hoãn hiển thị giá trị cũ → useDeferredValue

Dùng React v19?
├─ Đọc Promise/Context trong render → use()
├─ Form với Server Actions → useActionState + useFormStatus
└─ Cập nhật UI trước khi server xác nhận → useOptimistic
```

---

## 5. Danh Sách File

| File | Nội Dung | Độ Ưu Tiên |
| ---- | -------- | ---------- |
| [1-usestate-usereducer.md](./1-usestate-usereducer.md) | useState, useReducer — quản lý state | ⭐⭐⭐ Bắt buộc |
| [2-useeffect-uselayout.md](./2-useeffect-uselayout.md) | useEffect, useLayoutEffect — side effects | ⭐⭐⭐ Bắt buộc |
| [3-usecontext-useref.md](./3-usecontext-useref.md) | useContext, useRef — context và DOM refs | ⭐⭐⭐ Bắt buộc |
| [4-usememo-usecallback.md](./4-usememo-usecallback.md) | useMemo, useCallback — memoization | ⭐⭐ Quan trọng |
| [5-advanced-hooks.md](./5-advanced-hooks.md) | useId, useTransition, useDeferredValue, useDebugValue | ⭐⭐ Quan trọng |
| [6-react19-new-hooks.md](./6-react19-new-hooks.md) | use(), useActionState(), useFormStatus(), useOptimistic() | ⭐⭐ Modern React |
| [7-custom-hooks.md](./7-custom-hooks.md) | Custom Hooks — tạo và tái sử dụng logic | ⭐⭐⭐ Bắt buộc |

---

## 6. Câu Hỏi Phỏng Vấn Tổng Quan

### Q1: Hooks là gì và tại sao được giới thiệu?

**Trả lời:** Hooks (giới thiệu React 16.8, 2019) là hàm cho phép Functional Components có state và lifecycle. Lý do ra đời:
1. **Khó tái sử dụng stateful logic** — HOC và Render Props tạo ra "wrapper hell" (địa ngục wrapper)
2. **Class components phức tạp** — `this`, binding, lifecycle methods rải rác
3. **Tối ưu hóa** — Hooks tạo ra ít overhead hơn classes về bundle size

### Q2: Tại sao không được gọi Hook trong vòng lặp hoặc điều kiện?

**Trả lời:** React dựa vào **thứ tự gọi hook** (call order) để biết state nào thuộc hook nào. Nội bộ, React dùng một linked list (danh sách liên kết) — mỗi lần render, hook thứ N luôn phải khớp với hook thứ N của lần render trước. Nếu dùng trong `if`, thứ tự có thể thay đổi → React gán sai state → bug khó tìm.

### Q3: Custom Hook khác gì với helper function thông thường?

**Trả lời:** Custom Hook là hàm JavaScript **có thể chứa và gọi các hooks khác**. Quy ước đặt tên bắt đầu bằng `use` (ví dụ: `useFetch`, `useDebounce`). Helper function thông thường không thể gọi hooks bên trong. Custom Hook giúp tách stateful logic (logic có trạng thái) ra khỏi component, tái sử dụng được và test được độc lập.

### Q4: So sánh useEffect và useLayoutEffect?

| | `useEffect` | `useLayoutEffect` |
| - | ----------- | ----------------- |
| **Thời điểm chạy** | Sau khi DOM đã paint lên màn hình | Sau khi DOM cập nhật, trước khi paint |
| **Bất đồng bộ** | Có (non-blocking) | Không (blocking) |
| **Dùng khi** | 95% các trường hợp | Đo DOM, tránh flicker |
| **Hiệu năng** | Tốt hơn | Có thể block paint |

---

## 🔗 Điều Hướng

- **Trở về:** [reactjs/README.md](../README.md) — Tổng quan React
- **Tiếp theo:** [1-usestate-usereducer.md](./1-usestate-usereducer.md) — useState và useReducer
- **Xem thêm:** [01-fundamentals/](../01-fundamentals/) — Nền tảng React

**Cập Nhật Lần Cuối:** 2026-06-04 | **React Version:** v19 (stable)
