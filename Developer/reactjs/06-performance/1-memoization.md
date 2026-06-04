# Memoization — Ghi Nhớ Để Tối Ưu Hiệu Năng

> **Memoization** (Ghi Nhớ Kết Quả) là kỹ thuật lưu lại kết quả tính toán của một hàm dựa trên đầu vào, nếu đầu vào không thay đổi thì trả về kết quả đã lưu thay vì tính toán lại. Trong React, ba công cụ chính là: `React.memo`, `useMemo`, và `useCallback`.

---

## 📌 Mục Lục

1. [Vấn Đề: Re-render Không Cần Thiết](#1-vấn-đề-re-render-không-cần-thiết)
2. [React.memo — Bỏ Qua Re-render Component](#2-reactmemo--bỏ-qua-re-render-component)
3. [useMemo — Ghi Nhớ Giá Trị Tính Toán](#3-usememo--ghi-nhớ-giá-trị-tính-toán)
4. [useCallback — Ghi Nhớ Reference Hàm](#4-usecallback--ghi-nhớ-reference-hàm)
5. [Kết Hợp Ba Kỹ Thuật](#5-kết-hợp-ba-kỹ-thuật)
6. [React Compiler — Tương Lai Không Cần Memoize Thủ Công](#6-react-compiler--tương-lai-không-cần-memoize-thủ-công)
7. [Khi Nào KHÔNG Nên Dùng](#7-khi-nào-không-nên-dùng)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề: Re-render Không Cần Thiết

### React Re-render Như Thế Nào?

Mỗi khi state hoặc props của một component thay đổi, React sẽ:
1. Gọi lại hàm component để tính toán JSX mới
2. So sánh JSX mới với JSX cũ (**Reconciliation** — Thuật Toán Đối Chiếu)
3. Cập nhật DOM chỉ ở những chỗ thực sự thay đổi

**Vấn đề:** Bước 1 — gọi lại hàm component — cũng gọi lại tất cả component con, kể cả những con không có gì thay đổi.

### Ví Dụ Minh Họa

```jsx
function ParentComponent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild /> {/* Re-render MỖI LẦN count thay đổi, dù ExpensiveChild không dùng count */}
    </div>
  );
}

function ExpensiveChild() {
  // Giả sử component này render tốn 50ms
  console.log("ExpensiveChild rendered");
  return <div>Nội dung phức tạp</div>;
}
```

Mỗi lần nhấn button → `ParentComponent` re-render → `ExpensiveChild` cũng re-render dù không có lý do.

### Cách Xác Định Vấn Đề

Dùng **React DevTools Profiler** (Trình Phân Tích Hiệu Năng):
- Mở DevTools → Tab "Profiler" → Record → Thao tác → Stop
- Nhìn vào **"Why did this render?"** để thấy nguyên nhân re-render

---

## 2. React.memo — Bỏ Qua Re-render Component

### Cú Pháp Cơ Bản

```jsx
// React.memo là một HOC — Higher-Order Component — Component Bậc Cao
// Nó bọc component của bạn và bỏ qua re-render khi props không thay đổi
const ExpensiveChild = React.memo(function ExpensiveChild({ name, value }) {
  console.log("ExpensiveChild rendered");
  return <div>{name}: {value}</div>;
});

// Hoặc dùng với arrow function và export
const ProductCard = React.memo(({ product, onAddToCart }) => {
  return (
    <div>
      <h3>{product.name}</h3>
      <p>{product.price}</p>
      <button onClick={() => onAddToCart(product.id)}>Thêm vào giỏ</button>
    </div>
  );
});
```

### React.memo So Sánh Props Như Thế Nào?

`React.memo` thực hiện **shallow comparison** (So Sánh Nông) — so sánh từng prop bằng `Object.is`:

```jsx
// ✅ Primitive values (Giá trị nguyên thủy) — so sánh bằng giá trị
// "hello" === "hello" → true → bỏ qua re-render
// 42 === 42 → true → bỏ qua re-render

// ❌ Objects và Arrays — so sánh bằng reference (địa chỉ bộ nhớ)
// { id: 1 } === { id: 1 } → false (hai object khác nhau trong bộ nhớ)
// [1, 2, 3] === [1, 2, 3] → false (hai array khác nhau trong bộ nhớ)

function Parent() {
  const [count, setCount] = useState(0);

  // ❌ Mỗi render tạo object MỚI → React.memo luôn thấy props thay đổi
  const config = { theme: "dark" };
  const handleClick = () => console.log("clicked");

  return <Child config={config} onClick={handleClick} />;
}

const Child = React.memo(({ config, onClick }) => {
  // Vẫn re-render dù config và onClick "trông giống nhau"
  // vì reference thay đổi mỗi render
  return <div onClick={onClick}>{config.theme}</div>;
});
```

### Custom Comparison Function — Hàm So Sánh Tùy Chỉnh

```jsx
const ProductCard = React.memo(
  ({ product, onAddToCart }) => {
    return (
      <div>
        <h3>{product.name}</h3>
        <p>${product.price}</p>
        <button onClick={() => onAddToCart(product.id)}>Thêm vào giỏ</button>
      </div>
    );
  },
  // Hàm so sánh: trả về true → bỏ qua re-render, trả về false → re-render
  (prevProps, nextProps) => {
    return (
      prevProps.product.id === nextProps.product.id &&
      prevProps.product.price === nextProps.product.price &&
      prevProps.product.name === nextProps.product.name
      // Không so sánh onAddToCart vì dùng useCallback ở parent
    );
  }
);
```

### Ví Dụ Thực Tế: Danh Sách Sản Phẩm

```jsx
// Parent — component cha
function ProductList() {
  const [searchQuery, setSearchQuery] = useState("");
  const [cart, setCart] = useState([]);
  const { data: products } = useQuery({ queryKey: ["products"], queryFn: fetchProducts });

  // useCallback giúp onAddToCart có reference ổn định
  const handleAddToCart = useCallback((productId) => {
    setCart(prev => [...prev, productId]);
  }, []); // dependency array rỗng → hàm không bao giờ thay đổi

  const filteredProducts = useMemo(
    () => products?.filter(p => p.name.toLowerCase().includes(searchQuery.toLowerCase())),
    [products, searchQuery]
  );

  return (
    <div>
      <input value={searchQuery} onChange={e => setSearchQuery(e.target.value)} />
      {filteredProducts?.map(product => (
        // ProductCard chỉ re-render khi product hoặc onAddToCart thực sự thay đổi
        <ProductCard key={product.id} product={product} onAddToCart={handleAddToCart} />
      ))}
    </div>
  );
}

// ProductCard được wrap bằng React.memo
const ProductCard = React.memo(({ product, onAddToCart }) => {
  console.log(`ProductCard ${product.id} rendered`);
  return (
    <div className="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => onAddToCart(product.id)}>Thêm vào giỏ</button>
    </div>
  );
});
```

---

## 3. useMemo — Ghi Nhớ Giá Trị Tính Toán

### Cú Pháp

```jsx
const memoizedValue = useMemo(
  () => expensiveCalculation(a, b), // Hàm tính toán
  [a, b] // Dependencies — khi a hoặc b thay đổi mới tính lại
);
```

### Khi Nào Nên Dùng useMemo?

**Tiêu chí:** Tính toán tốn kém + chạy lại nhiều lần khi re-render.

```jsx
// ✅ Phù hợp: Filter và sort mảng lớn
function ProductFilter({ products, filters }) {
  const filteredAndSorted = useMemo(() => {
    console.log("Đang filter và sort..."); // Chỉ log khi products hoặc filters thay đổi
    return products
      .filter(p => {
        if (filters.category && p.category !== filters.category) return false;
        if (filters.minPrice && p.price < filters.minPrice) return false;
        if (filters.maxPrice && p.price > filters.maxPrice) return false;
        return true;
      })
      .sort((a, b) => {
        if (filters.sortBy === "price-asc") return a.price - b.price;
        if (filters.sortBy === "price-desc") return b.price - a.price;
        return a.name.localeCompare(b.name);
      });
  }, [products, filters]); // Chỉ tính lại khi products hoặc filters thay đổi

  return (
    <ul>
      {filteredAndSorted.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </ul>
  );
}
```

```jsx
// ✅ Phù hợp: Tính toán thống kê phức tạp
function SalesReport({ transactions }) {
  const stats = useMemo(() => {
    const total = transactions.reduce((sum, t) => sum + t.amount, 0);
    const byCategory = transactions.reduce((acc, t) => {
      acc[t.category] = (acc[t.category] || 0) + t.amount;
      return acc;
    }, {});
    const topCategory = Object.entries(byCategory)
      .sort(([, a], [, b]) => b - a)[0];

    return { total, byCategory, topCategory };
  }, [transactions]); // Chỉ tính lại khi transactions thay đổi

  return (
    <div>
      <p>Tổng doanh thu: {stats.total}</p>
      <p>Danh mục dẫn đầu: {stats.topCategory?.[0]}</p>
    </div>
  );
}
```

```jsx
// ✅ Phù hợp: Tạo object/array ổn định để truyền xuống React.memo component
function Dashboard() {
  const [userId, setUserId] = useState(1);

  // Nếu không dùng useMemo, config sẽ là object mới mỗi render
  // → ChartComponent (nếu dùng React.memo) vẫn bị re-render
  const chartConfig = useMemo(() => ({
    type: "bar",
    colors: ["#3b82f6", "#ef4444", "#22c55e"],
    animation: true,
  }), []); // Config không bao giờ thay đổi → deps rỗng

  return <ChartComponent config={chartConfig} userId={userId} />;
}
```

### ❌ Khi Nào KHÔNG Nên Dùng useMemo

```jsx
// ❌ Tính toán đơn giản — useMemo có overhead còn tốn hơn
const doubled = useMemo(() => value * 2, [value]); // Không cần thiết!
const doubled = value * 2; // Đủ rồi

// ❌ String formatting đơn giản
const displayName = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]); // Không cần
const displayName = `${firstName} ${lastName}`; // Đủ rồi

// ❌ Tính toán chỉ chạy một lần khi mount
const initial = useMemo(() => expensiveInit(), []); // Dùng useState initializer thay thế
const [data] = useState(() => expensiveInit()); // ✅ Cách đúng
```

---

## 4. useCallback — Ghi Nhớ Reference Hàm

### Tại Sao Cần useCallback?

Trong JavaScript, mỗi lần hàm component chạy (render), tất cả hàm được định nghĩa bên trong sẽ được tạo mới — có **reference** (địa chỉ bộ nhớ) khác nhau:

```javascript
// Trong mỗi lần render:
const fn1 = () => console.log("hello");
const fn2 = () => console.log("hello");
console.log(fn1 === fn2); // false — hai hàm khác nhau!
```

Vấn đề: Nếu truyền hàm này xuống component con đã wrap bằng `React.memo`, component con vẫn re-render vì prop hàm có reference mới.

### Cú Pháp

```jsx
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b] // Chỉ tạo hàm mới khi a hoặc b thay đổi
);
```

### Ví Dụ Đúng Và Sai

```jsx
// ❌ Sai: handleDelete tạo mới mỗi render → TaskItem re-render dù task không đổi
function TaskList({ tasks }) {
  const [selectedId, setSelectedId] = useState(null);

  const handleDelete = (id) => { // Hàm mới mỗi render!
    deleteTask(id);
  };

  return tasks.map(task => (
    <TaskItem key={task.id} task={task} onDelete={handleDelete} />
  ));
}

// ✅ Đúng: useCallback giữ reference ổn định
function TaskList({ tasks }) {
  const [selectedId, setSelectedId] = useState(null);

  const handleDelete = useCallback((id) => {
    deleteTask(id);
  }, []); // Không phụ thuộc gì → hàm không bao giờ thay đổi

  return tasks.map(task => (
    <TaskItem key={task.id} task={task} onDelete={handleDelete} />
  ));
}

const TaskItem = React.memo(({ task, onDelete }) => {
  console.log(`TaskItem ${task.id} rendered`);
  return (
    <div>
      <span>{task.name}</span>
      <button onClick={() => onDelete(task.id)}>Xóa</button>
    </div>
  );
});
```

### useCallback Với Dependencies

```jsx
function SearchComponent({ onSearch }) {
  const [query, setQuery] = useState("");

  // handleSubmit phụ thuộc vào query và onSearch
  // Chỉ tạo hàm mới khi query hoặc onSearch thay đổi
  const handleSubmit = useCallback((e) => {
    e.preventDefault();
    onSearch(query);
  }, [query, onSearch]); // Dependencies phải đầy đủ!

  return (
    <form onSubmit={handleSubmit}>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <button type="submit">Tìm kiếm</button>
    </form>
  );
}
```

### useCallback Cho Event Handlers Từ Custom Hook

```jsx
// Custom hook trả về hàm ổn định
function useOptimisticDelete(queryClient) {
  const deleteItem = useCallback(async (id) => {
    // Optimistic update — Cập nhật Lạc Quan
    queryClient.setQueryData(["items"], old =>
      old?.filter(item => item.id !== id)
    );

    try {
      await deleteItemAPI(id);
    } catch (error) {
      // Rollback nếu thất bại
      queryClient.invalidateQueries({ queryKey: ["items"] });
    }
  }, [queryClient]);

  return { deleteItem };
}
```

---

## 5. Kết Hợp Ba Kỹ Thuật

### Pattern Hoàn Chỉnh: Component Danh Sách Được Tối Ưu

```jsx
// ============================================================
// Parent Component — Component Cha
// ============================================================
function EmployeeDirectory() {
  const [searchQuery, setSearchQuery] = useState("");
  const [department, setDepartment] = useState("all");
  const [employees, setEmployees] = useState(initialEmployees);

  // useMemo: Cache kết quả filter — chỉ tính lại khi employees, searchQuery, hoặc department thay đổi
  const filteredEmployees = useMemo(() => {
    return employees.filter(emp => {
      const matchesSearch = emp.name.toLowerCase().includes(searchQuery.toLowerCase());
      const matchesDept = department === "all" || emp.department === department;
      return matchesSearch && matchesDept;
    });
  }, [employees, searchQuery, department]);

  // useCallback: Giữ reference ổn định cho các handler
  const handleEdit = useCallback((employeeId) => {
    // Mở modal edit
    openEditModal(employeeId);
  }, []); // Không phụ thuộc gì

  const handleDelete = useCallback((employeeId) => {
    setEmployees(prev => prev.filter(e => e.id !== employeeId));
  }, []); // setEmployees luôn ổn định, không cần trong deps

  const handlePromote = useCallback((employeeId, newTitle) => {
    setEmployees(prev => prev.map(e =>
      e.id === employeeId ? { ...e, title: newTitle } : e
    ));
  }, []);

  return (
    <div>
      <SearchBar value={searchQuery} onChange={setSearchQuery} />
      <DepartmentFilter value={department} onChange={setDepartment} />
      <p>Hiển thị {filteredEmployees.length} / {employees.length} nhân viên</p>
      <EmployeeList
        employees={filteredEmployees}
        onEdit={handleEdit}
        onDelete={handleDelete}
        onPromote={handlePromote}
      />
    </div>
  );
}

// ============================================================
// EmployeeList — React.memo ngăn re-render khi parent re-render
// nhưng employees, onEdit, onDelete, onPromote không thay đổi
// ============================================================
const EmployeeList = React.memo(({ employees, onEdit, onDelete, onPromote }) => {
  return (
    <ul>
      {employees.map(employee => (
        <EmployeeCard
          key={employee.id}
          employee={employee}
          onEdit={onEdit}
          onDelete={onDelete}
          onPromote={onPromote}
        />
      ))}
    </ul>
  );
});

// ============================================================
// EmployeeCard — Chỉ re-render khi employee hoặc handlers thay đổi
// ============================================================
const EmployeeCard = React.memo(({ employee, onEdit, onDelete, onPromote }) => {
  return (
    <li>
      <img src={employee.avatar} alt={employee.name} />
      <div>
        <h3>{employee.name}</h3>
        <p>{employee.title} — {employee.department}</p>
      </div>
      <div>
        <button onClick={() => onEdit(employee.id)}>Sửa</button>
        <button onClick={() => onDelete(employee.id)}>Xóa</button>
        <button onClick={() => onPromote(employee.id, "Senior")}>Thăng chức</button>
      </div>
    </li>
  );
});
```

---

## 6. React Compiler — Tương Lai Không Cần Memoize Thủ Công

**React Compiler** (trước đây gọi là React Forget — Quên Đi Việc Memoize) là compiler — trình biên dịch — tự động thêm memoization vào code của bạn tại **compile time** (thời điểm biên dịch), thay vì phải viết thủ công.

```jsx
// Bạn viết (không có memo gì cả):
function ProductCard({ product, onAddToCart }) {
  return (
    <div>
      <h3>{product.name}</h3>
      <button onClick={() => onAddToCart(product.id)}>Thêm vào giỏ</button>
    </div>
  );
}

// React Compiler tự động biên dịch thành tương đương:
const ProductCard = React.memo(({ product, onAddToCart }) => {
  const handleClick = useCallback(() => onAddToCart(product.id), [onAddToCart, product.id]);
  return (
    <div>
      <h3>{product.name}</h3>
      <button onClick={handleClick}>Thêm vào giỏ</button>
    </div>
  );
});
```

### Bật React Compiler (React 19+)

```javascript
// babel.config.js
const ReactCompilerConfig = {
  target: "19" // Phiên bản React target
};

module.exports = function () {
  return {
    plugins: [
      ["babel-plugin-react-compiler", ReactCompilerConfig],
    ],
  };
};

// Hoặc với Next.js (next.config.js)
const nextConfig = {
  experimental: {
    reactCompiler: true,
  },
};
```

> **Lưu ý:** React Compiler đang trong giai đoạn **stable release** với React 19. Nó không thay thế hoàn toàn kiến thức về memoization — bạn vẫn cần hiểu cơ chế để debug khi compiler không tối ưu được.

---

## 7. Khi Nào KHÔNG Nên Dùng

### React.memo Không Giúp Ích Khi

```jsx
// ❌ Component đơn giản, render rất nhanh
const Label = React.memo(({ text }) => <span>{text}</span>);
// Chi phí so sánh props có khi còn cao hơn chi phí re-render thực sự

// ❌ Props luôn thay đổi (không thể tránh khỏi)
const Clock = React.memo(({ time }) => <div>{time}</div>);
// time thay đổi mỗi giây → memo vô dụng, chỉ thêm overhead

// ❌ Component nhận children (children luôn là object mới)
const Wrapper = React.memo(({ children }) => <div>{children}</div>);
// {children} mỗi lần render là JSX object mới → memo không bỏ qua được
```

### useMemo Không Nên Dùng Khi

```jsx
// ❌ Tính toán đơn giản
const sum = useMemo(() => a + b, [a, b]); // Quá nhỏ, không đáng
const sum = a + b; // ✅

// ❌ Tạo object để truyền vào component KHÔNG dùng React.memo
// Vì component con vẫn re-render bất kể
function Parent() {
  const style = useMemo(() => ({ color: "red" }), []); // Không cần
  return <RegularChild style={style} />; // RegularChild không dùng React.memo
}
```

### useCallback Không Nên Dùng Khi

```jsx
// ❌ Hàm không truyền xuống component con
function Counter() {
  const increment = useCallback(() => setCount(c => c + 1), []);
  // Hàm này chỉ dùng trong Counter, không truyền xuống đâu
  // useCallback không giúp ích gì ở đây
  return <button onClick={increment}>Tăng</button>;
}
```

### Quy Tắc Đơn Giản Để Quyết Định

```
Dùng React.memo khi:
  ✅ Component render tốn > ~2ms
  ✅ Component re-render thường xuyên khi parent re-render
  ✅ Props là primitive hoặc được memo hóa ổn định

Dùng useMemo khi:
  ✅ Tính toán tốn > 1ms (filter/sort mảng lớn, transform data phức tạp)
  ✅ Kết quả được truyền xuống React.memo component

Dùng useCallback khi:
  ✅ Hàm được truyền xuống React.memo component con
  ✅ Hàm là dependency của useEffect hoặc useMemo khác
```

---

## 8. Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: React.memo, useMemo và useCallback khác nhau như thế nào?**

A:
- `React.memo` — HOC bọc **component**, bỏ qua re-render khi props không đổi
- `useMemo` — Hook ghi nhớ **giá trị** (result của function), tính lại khi dependencies thay đổi
- `useCallback` — Hook ghi nhớ **reference của function**, tạo hàm mới khi dependencies thay đổi

Về bản chất: `useCallback(fn, deps)` tương đương `useMemo(() => fn, deps)`.

---

**Q: Khi nào React.memo không hoạt động (component vẫn re-render)?**

A: Khi props là object hoặc function được tạo mới mỗi render:
```jsx
// Object mới mỗi render → React.memo thấy props thay đổi → vẫn re-render
<MemoChild config={{ theme: "dark" }} />

// Giải pháp: useMemo cho object, useCallback cho function
const config = useMemo(() => ({ theme: "dark" }), []);
<MemoChild config={config} />
```

---

**Q: useMemo có phải lúc nào cũng giúp tối ưu hiệu năng không?**

A: Không. `useMemo` có chi phí:
- Bộ nhớ để lưu giá trị cũ
- So sánh dependencies mỗi render

Nếu tính toán đơn giản (< 0.1ms), chi phí so sánh của `useMemo` có khi còn cao hơn. Nguyên tắc: **đo lường trước** bằng React Profiler, rồi mới quyết định dùng `useMemo`.

---

**Q: Tại sao useCallback cần thiết khi truyền hàm xuống React.memo component?**

A: Vì mỗi lần component cha render, hàm được định nghĩa lại với **reference mới**. `React.memo` dùng shallow comparison — thấy reference khác → coi như props thay đổi → re-render component con. `useCallback` giữ reference ổn định qua các lần render (trừ khi dependencies thay đổi).

---

**Q: React Compiler có thay thế hoàn toàn useMemo và useCallback không?**

A: Về mặt thực tế, React Compiler tự động hóa hầu hết các trường hợp memoization phổ biến. Tuy nhiên:
- Vẫn cần hiểu cơ chế để debug khi compiler không tối ưu được
- Một số pattern phức tạp (custom comparison, conditional memoization) vẫn cần can thiệp thủ công
- Compiler hiện yêu cầu code tuân thủ **Rules of React** — Quy Tắc React

---

### Câu Hỏi Nâng Cao

**Q: Giải thích vấn đề "stale closure" trong useCallback và cách giải quyết.**

A: **Stale closure** (Closure Cũ) xảy ra khi callback capture giá trị cũ từ lần render trước:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  // ❌ Stale closure: handleLog luôn log "Count: 0" dù count đã tăng
  // vì count bị capture từ lần đầu tiên
  const handleLog = useCallback(() => {
    console.log("Count:", count); // Luôn là 0!
  }, []); // Thiếu count trong deps!

  // ✅ Cách 1: Thêm count vào dependencies
  const handleLog1 = useCallback(() => {
    console.log("Count:", count);
  }, [count]); // Tạo hàm mới mỗi khi count thay đổi

  // ✅ Cách 2: Dùng ref để đọc giá trị mới nhất mà không tạo hàm mới
  const countRef = useRef(count);
  useEffect(() => { countRef.current = count; });

  const handleLog2 = useCallback(() => {
    console.log("Count:", countRef.current); // Luôn đọc giá trị mới nhất
  }, []); // Không cần count trong deps

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Tăng</button>
      <button onClick={handleLog2}>Log Count</button>
    </div>
  );
}
```

---

## 🔗 Điều Hướng

- **Trước đó:** [README.md](./README.md) — Tổng quan Performance
- **Tiếp theo:** [2-code-splitting.md](./2-code-splitting.md) — Code Splitting với React.lazy
- **Liên quan:** [02-hooks/4-usememo-usecallback.md](../02-hooks/4-usememo-usecallback.md) — useMemo & useCallback chi tiết
- **Chỉ mục:** [INDEX.md](../INDEX.md)
