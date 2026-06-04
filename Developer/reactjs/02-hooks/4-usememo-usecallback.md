# useMemo & useCallback — Memoization và Tối Ưu Hiệu Năng

> `useMemo` và `useCallback` là hai hooks dùng để **memoization** (ghi nhớ — lưu cache kết quả để tránh tính toán lại). Chúng tối ưu hiệu năng bằng cách bỏ qua các tính toán tốn kém hoặc tránh re-render không cần thiết. **Quan trọng:** Đây là tối ưu hóa — chỉ dùng khi đã đo được vấn đề hiệu năng thực sự.

---

## 📌 Mục Lục

1. [Memoization Là Gì?](#1-memoization-là-gì)
2. [useMemo — Ghi Nhớ Giá Trị](#2-usememo)
3. [useCallback — Ghi Nhớ Hàm](#3-usecallback)
4. [React.memo — HOC Tối Ưu Component](#4-reactmemo)
5. [Khi Nào Nên và Không Nên Dùng](#5-khi-nào-nên-dùng)
6. [React Compiler và Tương Lai](#6-react-compiler)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Memoization Là Gì?

**Memoization** (Ghi Nhớ Kết Quả) là kỹ thuật tối ưu hóa: lưu cache (bộ nhớ tạm) kết quả của một hàm dựa trên input. Nếu input không đổi, trả về kết quả đã lưu thay vì tính toán lại.

```
Không có memoization:
  tinhToanNang(a, b) → chạy mỗi lần, dù a và b không đổi

Có memoization:
  useMemo(() => tinhToanNang(a, b), [a, b])
  → Lần 1: a=5, b=10 → tính toán → kết quả = 50 → lưu cache
  → Lần 2: a=5, b=10 → đọc cache → kết quả = 50 (không tính lại)
  → Lần 3: a=5, b=20 → input đổi → tính toán lại → kết quả = 100
```

### Tại Sao Cần Memoization Trong React?

Mỗi khi component re-render, **toàn bộ thân hàm chạy lại**:

```jsx
function ProductList({ products, filter }) {
  // Hàm này chạy lại mỗi lần render — dù products và filter không đổi!
  const filteredProducts = products.filter(p => p.category === filter);

  return <ul>{filteredProducts.map(...)}</ul>;
}
```

Nếu `products` có 10.000 mục và filter không đổi → lãng phí tài nguyên mỗi lần re-render.

---

## 2. useMemo — Ghi Nhớ Giá Trị

### Cú Pháp

```jsx
const memoizedValue = useMemo(() => {
  return computeExpensiveValue(dep1, dep2);
}, [dep1, dep2]);
// computeExpensiveValue chỉ chạy lại khi dep1 hoặc dep2 thay đổi
```

### Ví Dụ — Lọc Danh Sách Lớn

```jsx
import { useState, useMemo } from "react";

function ProductList({ products }) {
  const [filter, setFilter] = useState("all");
  const [sortBy, setSortBy] = useState("name");
  const [searchQuery, setSearchQuery] = useState("");

  // ✅ Chỉ tính toán lại khi products, filter, sortBy, hoặc searchQuery thay đổi
  const processedProducts = useMemo(() => {
    let result = [...products];

    // Lọc theo category
    if (filter !== "all") {
      result = result.filter(p => p.category === filter);
    }

    // Lọc theo search
    if (searchQuery) {
      const query = searchQuery.toLowerCase();
      result = result.filter(p =>
        p.name.toLowerCase().includes(query) ||
        p.description.toLowerCase().includes(query)
      );
    }

    // Sắp xếp
    result.sort((a, b) => {
      if (sortBy === "name") return a.name.localeCompare(b.name);
      if (sortBy === "price") return a.price - b.price;
      return 0;
    });

    return result;
  }, [products, filter, sortBy, searchQuery]); // Dependencies rõ ràng

  return (
    <div>
      <input
        value={searchQuery}
        onChange={e => setSearchQuery(e.target.value)}
        placeholder="Tìm sản phẩm..."
      />
      <select value={filter} onChange={e => setFilter(e.target.value)}>
        <option value="all">Tất cả</option>
        <option value="electronics">Điện tử</option>
        <option value="clothing">Quần áo</option>
      </select>
      <p>Tìm thấy {processedProducts.length} sản phẩm</p>
      <ul>
        {processedProducts.map(product => (
          <li key={product.id}>{product.name} — {product.price.toLocaleString()}đ</li>
        ))}
      </ul>
    </div>
  );
}
```

### useMemo Để Ổn Định Object/Array Reference

```jsx
// ❌ Vấn đề: Object được tạo mới mỗi render → child luôn re-render
function Parent({ userId }) {
  const [count, setCount] = useState(0);

  const options = { userId, theme: "dark" }; // Object mới mỗi lần!

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Re-render Parent: {count}</button>
      <Child options={options} /> {/* Re-render dù options không thay đổi logic */}
    </div>
  );
}

// ✅ Giải pháp: useMemo ổn định reference
function ParentFixed({ userId }) {
  const [count, setCount] = useState(0);

  // options chỉ được tạo mới khi userId thay đổi
  const options = useMemo(
    () => ({ userId, theme: "dark" }),
    [userId]
  );

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Re-render Parent: {count}</button>
      <Child options={options} /> {/* Không re-render khi chỉ count thay đổi */}
    </div>
  );
}
```

---

## 3. useCallback — Ghi Nhớ Hàm

### Cú Pháp

```jsx
const memoizedFunction = useCallback(() => {
  doSomething(dep1, dep2);
}, [dep1, dep2]);
// Hàm được tạo một lần và tái sử dụng nếu dependencies không đổi
```

### Tại Sao Cần useCallback?

Trong JavaScript, **mỗi lần function được định nghĩa là một object mới** → mỗi lần render, hàm có reference mới → React.memo không có tác dụng:

```jsx
// ❌ Vấn đề: handleClick là hàm mới mỗi lần Parent render
function Parent({ productId }) {
  const [count, setCount] = useState(0);

  function handleClick() {
    console.log("Clicked product:", productId);
  }

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Re-render Parent: {count}</button>
      {/* Child luôn re-render dù productId không đổi, vì handleClick là hàm mới! */}
      <MemoizedChild onClick={handleClick} />
    </div>
  );
}
```

```jsx
// ✅ Giải pháp: useCallback ổn định reference của hàm
function ParentFixed({ productId }) {
  const [count, setCount] = useState(0);

  // handleClick chỉ được tạo mới khi productId thay đổi
  const handleClick = useCallback(() => {
    console.log("Clicked product:", productId);
  }, [productId]);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Re-render Parent: {count}</button>
      {/* MemoizedChild KHÔNG re-render khi chỉ count thay đổi */}
      <MemoizedChild onClick={handleClick} />
    </div>
  );
}
```

### Ví Dụ Thực Tế — Data Fetching Với useCallback

```jsx
function DataTable({ endpoint, filters }) {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(false);

  // Ổn định hàm fetchData để dùng trong useEffect
  const fetchData = useCallback(async () => {
    setLoading(true);
    try {
      const params = new URLSearchParams(filters).toString();
      const response = await fetch(`${endpoint}?${params}`);
      const result = await response.json();
      setData(result);
    } finally {
      setLoading(false);
    }
  }, [endpoint, filters]); // Chỉ tạo lại khi endpoint hoặc filters thay đổi

  useEffect(() => {
    fetchData();
  }, [fetchData]); // fetchData là dependency ổn định

  return (
    <div>
      {loading ? (
        <p>Đang tải...</p>
      ) : (
        <ul>
          {data.map(item => (
            <li key={item.id}>{item.name}</li>
          ))}
        </ul>
      )}
      <button onClick={fetchData}>Làm mới</button>
    </div>
  );
}
```

---

## 4. React.memo — HOC Tối Ưu Component

`React.memo` là **HOC** (Higher-Order Component — Component Bậc Cao) ngăn component re-render nếu props không thay đổi:

```jsx
import { memo } from "react";

// Không có React.memo — re-render mỗi khi cha render
function ExpensiveComponent({ name, score }) {
  console.log("Rendering ExpensiveComponent");
  return <div>{name}: {score}</div>;
}

// Có React.memo — chỉ re-render khi name hoặc score thay đổi
const MemoizedExpensive = memo(function ExpensiveComponent({ name, score }) {
  console.log("Rendering MemoizedExpensive");
  return <div>{name}: {score}</div>;
});
```

### React.memo Với Custom Comparison (So Sánh Tùy Chỉnh)

```jsx
const MemoizedComponent = memo(
  function MyComponent({ user, onUpdate }) {
    return <div onClick={() => onUpdate(user.id)}>{user.name}</div>;
  },
  // areEqual(prevProps, nextProps): trả về true = KHÔNG re-render
  (prevProps, nextProps) => {
    return (
      prevProps.user.id === nextProps.user.id &&
      prevProps.user.name === nextProps.user.name
      // Bỏ qua so sánh onUpdate nếu nó được bọc bằng useCallback
    );
  }
);
```

### Bộ Ba Kết Hợp: React.memo + useMemo + useCallback

```jsx
// Parent component
function Parent({ data }) {
  const [theme, setTheme] = useState("light");

  // useMemo: ổn định object reference
  const processedData = useMemo(
    () => data.filter(item => item.active).sort((a, b) => b.score - a.score),
    [data]
  );

  // useCallback: ổn định hàm reference
  const handleItemClick = useCallback((id) => {
    console.log("Clicked:", id);
  }, []); // Không có dependencies → luôn stable

  return (
    <div className={`app ${theme}`}>
      <button onClick={() => setTheme(t => t === "light" ? "dark" : "light")}>
        Đổi theme
      </button>
      {/* MemoizedList không re-render khi theme thay đổi */}
      <MemoizedList items={processedData} onItemClick={handleItemClick} />
    </div>
  );
}

// Child component được memo hóa
const MemoizedList = memo(function List({ items, onItemClick }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => onItemClick(item.id)}>
          {item.name} — {item.score} điểm
        </li>
      ))}
    </ul>
  );
});
```

---

## 5. Khi Nào Nên và Không Nên Dùng

### Nguyên Tắc Vàng: Đo Trước, Tối Ưu Sau

```
❌ KHÔNG làm:
  "Tôi thêm useMemo vào tất cả mọi thứ cho chắc"
  → Overhead của memoization (lưu cache, so sánh deps) > lợi ích

✅ NÊN làm:
  1. Profile (đo đạc) app bằng React DevTools Profiler
  2. Xác định component thực sự bị re-render quá nhiều
  3. Chỉ khi đó mới áp dụng useMemo/useCallback
```

### Khi Nên Dùng useMemo

```jsx
// ✅ Tính toán nặng, phụ thuộc ít deps ổn định
const result = useMemo(() => fibonacci(n), [n]);

// ✅ Lọc/sắp xếp danh sách lớn (>1000 items)
const filtered = useMemo(() => bigList.filter(...), [bigList, filter]);

// ✅ Ổn định object/array reference để truyền xuống component con có React.memo
const config = useMemo(() => ({ apiUrl, timeout }), [apiUrl, timeout]);

// ❌ Không cần — tính toán đơn giản
const doubled = useMemo(() => count * 2, [count]); // Quá đơn giản!
const greeting = useMemo(() => `Xin chào ${name}`, [name]); // Không cần!
```

### Khi Nên Dùng useCallback

```jsx
// ✅ Hàm truyền xuống component con có React.memo
const handleClick = useCallback(() => doSomething(id), [id]);

// ✅ Hàm dùng làm dependency trong useEffect
const fetchData = useCallback(() => fetch(url), [url]);

// ✅ Hàm được truyền xuống danh sách lớn (tránh N lần re-render)
const handleItemClick = useCallback((itemId) => setSelected(itemId), []);

// ❌ Không cần — component con không có React.memo
const handleClick = useCallback(() => setCount(c => c + 1), []);
// Nếu <button onClick={handleClick}> — button element, không cần

// ❌ Không cần — inline handler đơn giản
const toggle = useCallback(() => setOpen(o => !o), []);
// Thay bằng: onClick={() => setOpen(o => !o)} — đơn giản hơn, đủ rồi
```

### Bảng Quyết Định

| Tình huống | Giải pháp |
| ---------- | --------- |
| Tính toán nặng (>1ms) từ state/props | `useMemo` |
| Lọc/sắp xếp danh sách lớn | `useMemo` |
| Ổn định object/array làm props cho component có `memo` | `useMemo` |
| Hàm truyền xuống component có `React.memo` | `useCallback` |
| Hàm làm dependency của `useEffect` | `useCallback` |
| Hàm truyền xuống custom Hook cần stable reference | `useCallback` |
| Tính toán đơn giản (<0.1ms) | ❌ Không cần |
| Hàm truyền xuống DOM element (`<button>`, `<input>`) | ❌ Không cần |
| Hàm trong component không có `React.memo` | ❌ Không cần |

---

## 6. React Compiler và Tương Lai

**React Compiler** (trước đây gọi là **React Forget**) là công cụ compile-time (biên dịch trước) tự động thêm memoization:

```jsx
// Bạn viết (không có useMemo/useCallback)
function ProductList({ products, filter }) {
  const filtered = products.filter(p => p.category === filter);
  const handleClick = (id) => addToCart(id);

  return filtered.map(p => (
    <Product key={p.id} product={p} onClick={handleClick} />
  ));
}

// React Compiler tự động tối ưu tương đương với:
function ProductList({ products, filter }) {
  const filtered = useMemo(
    () => products.filter(p => p.category === filter),
    [products, filter]
  );
  const handleClick = useCallback((id) => addToCart(id), []);

  return filtered.map(p => (
    <Product key={p.id} product={p} onClick={handleClick} />
  ));
}
```

**Ý nghĩa:** Trong tương lai (React Compiler stable), bạn ít cần viết `useMemo`/`useCallback` thủ công hơn. **Hiện tại (2026):** React Compiler đang trong giai đoạn beta — vẫn cần hiểu và dùng thủ công.

---

## 7. Câu Hỏi Phỏng Vấn

### Q1: Sự khác biệt giữa useMemo và useCallback?

**Trả lời:**
- `useMemo(() => computeValue(a, b), [a, b])` — ghi nhớ **giá trị** (kết quả của hàm)
- `useCallback(() => doSomething(a, b), [a, b])` — ghi nhớ **hàm** (chính hàm đó)

Thực ra `useCallback(fn, deps)` tương đương với `useMemo(() => fn, deps)`. Sự khác biệt chỉ là về semantic (ý nghĩa): một cái dành cho giá trị, một cái dành cho hàm.

### Q2: React.memo khác gì useMemo?

**Trả lời:**
- `React.memo(Component)` — HOC bọc **component**, ngăn re-render khi props không đổi
- `useMemo(() => value, deps)` — Hook ghi nhớ **giá trị** bên trong component

`React.memo` hoạt động ở level component; `useMemo` hoạt động ở level giá trị trong component.

### Q3: Tại sao useMemo và useCallback không phải lúc nào cũng giúp ích?

**Trả lời:** Memoization có chi phí riêng:
1. **Bộ nhớ** — lưu cache kết quả tốn RAM
2. **So sánh** — mỗi render phải so sánh dependencies (O(n) với n là số deps)
3. **Phức tạp** — code khó đọc hơn

Nếu chi phí này lớn hơn lợi ích (ví dụ: component nhỏ, tính toán nhanh, dependencies thay đổi thường xuyên), memoization làm chậm thay vì nhanh hơn.

### Q4: Khi nào React.memo không có tác dụng?

**Trả lời:** React.memo so sánh props bằng **shallow comparison** (so sánh nông — so sánh reference):
1. **Props là object/array mới mỗi render** — dù giá trị bên trong giống nhau, reference khác → memo không giúp được → cần `useMemo` ở parent
2. **Props là hàm mới mỗi render** — cần `useCallback` ở parent
3. **Context thay đổi** — memo không ngăn re-render do context
4. **State bên trong component thay đổi** — memo không ngăn re-render do internal state

### Q5: Cách đo hiệu năng component trước khi tối ưu?

**Trả lời:** Dùng **React DevTools Profiler** (Công Cụ Phân Tích Hiệu Năng):
1. Mở React DevTools → tab "Profiler"
2. Bấm Record → thực hiện hành động cần đo → Stop
3. Xem Flamegraph (Biểu Đồ Ngọn Lửa) — thanh màu cam/đỏ = render chậm
4. Click vào component để xem: "Why did this render?" — lý do re-render
5. Xác định component nào thực sự cần tối ưu → áp dụng memo/useMemo/useCallback

---

## 🔗 Điều Hướng

- **Trở về:** [README.md](./README.md) — Tổng quan Hooks
- **Trước đó:** [3-usecontext-useref.md](./3-usecontext-useref.md) — useContext và useRef
- **Tiếp theo:** [5-advanced-hooks.md](./5-advanced-hooks.md) — useId, useTransition, useDeferredValue

**Cập Nhật Lần Cuối:** 2026-06-04 | **React Version:** v19 (stable)
