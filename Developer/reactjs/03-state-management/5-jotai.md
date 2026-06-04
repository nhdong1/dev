# Jotai — Atomic State Management (Quản Lý Trạng Thái Nguyên Tử)

> **Jotai** (tiếng Nhật nghĩa là "trạng thái") là thư viện state management theo mô hình **atomic** (nguyên tử) — mỗi đơn vị state nhỏ nhất gọi là **atom**. Lấy cảm hứng từ Recoil của Meta nhưng API đơn giản hơn, bundle nhỏ hơn (~3KB).

---

## 📌 Mục Lục

1. [Atomic State Model Là Gì?](#1-atomic-state-model)
2. [Atom Cơ Bản](#2-atom-cơ-bản)
3. [Derived Atoms — Atoms Dẫn Xuất](#3-derived-atoms)
4. [Async Atoms — Atoms Bất Đồng Bộ](#4-async-atoms)
5. [atomWithStorage, atomWithReset](#5-atom-utilities)
6. [Jotai vs Zustand vs Context](#6-so-sánh)
7. [Ví Dụ Thực Tế](#7-ví-dụ-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Atomic State Model Là Gì?

### Mô Hình Truyền Thống vs Atomic

```
Mô hình truyền thống (Redux/Zustand):
┌─────────────────────────┐
│         Store           │
│  { user, cart, posts,   │
│    settings, ui, ... }  │   ← Tất cả state trong 1 nơi
└─────────────────────────┘

Mô hình Atomic (Jotai/Recoil):
  atom(user)     atom(cartItems)     atom(theme)
      │                │                  │
  Component A    Component B,C        Component D
```

### Ưu Điểm Của Atomic Model

- **Fine-grained reactivity** (phản ứng chi tiết): Component chỉ re-render khi atom nó subscribe thay đổi
- **Code splitting tự nhiên**: Atoms được định nghĩa ở file riêng, lazy load cùng feature
- **Không cần selector phức tạp**: Derived atoms thay thế selector
- **Phù hợp với concurrent React**: Hoạt động tốt với Suspense và concurrent features

---

## 2. Atom Cơ Bản

```bash
npm install jotai
```

### Tạo Và Dùng Atom

```javascript
import { atom, useAtom, useAtomValue, useSetAtom } from "jotai";

// Tạo atom — đơn vị state nhỏ nhất
const countAtom = atom(0);
const nameAtom = atom("Nguyễn An");
const isOpenAtom = atom(false);
const userAtom = atom(null);
```

```jsx
function Counter() {
  // useAtom — tương tự useState nhưng shared (chia sẻ) giữa mọi component
  const [count, setCount] = useAtom(countAtom);

  return (
    <div>
      <p>Đếm: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <button onClick={() => setCount(c => c - 1)}>-</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}

function CountDisplay() {
  // useAtomValue — chỉ đọc, không gây re-render khi setter thay đổi
  const count = useAtomValue(countAtom);
  return <span>Bộ đếm: {count}</span>;
}

function ToggleButton() {
  // useSetAtom — chỉ lấy setter, KHÔNG bao giờ re-render
  const setIsOpen = useSetAtom(isOpenAtom);
  return <button onClick={() => setIsOpen(v => !v)}>Toggle</button>;
}
```

### Jotai Không Cần Provider

```jsx
// Không cần Provider bọc! Atoms hoạt động global mặc định

function App() {
  return (
    // Không cần <Provider store={store}>
    <div>
      <Counter />
      <CountDisplay />  {/* Cùng atom → cùng state */}
    </div>
  );
}
```

### Provider Scope — Cô Lập Atoms

```jsx
import { Provider } from "jotai";

// Dùng Provider khi cần isolate (cô lập) state — ví dụ: component có nhiều instance
function App() {
  return (
    <div>
      <Provider>  {/* Instance 1 có count riêng */}
        <Counter />
      </Provider>
      <Provider>  {/* Instance 2 có count riêng */}
        <Counter />
      </Provider>
    </div>
  );
}
```

---

## 3. Derived Atoms — Atoms Dẫn Xuất

**Derived atom** (atom dẫn xuất) — atom được tính toán từ các atoms khác.

### Read-only Derived Atom

```javascript
import { atom } from "jotai";

const itemsAtom = atom([
  { id: 1, text: "Mua sữa", done: false },
  { id: 2, text: "Đi bộ 30 phút", done: true },
  { id: 3, text: "Đọc sách", done: false },
]);

const filterAtom = atom("all");  // "all" | "active" | "completed"

// Derived atom — tính toán từ itemsAtom và filterAtom
const filteredItemsAtom = atom((get) => {
  const items = get(itemsAtom);
  const filter = get(filterAtom);

  switch (filter) {
    case "active":    return items.filter(i => !i.done);
    case "completed": return items.filter(i => i.done);
    default:          return items;
  }
});

const statsAtom = atom((get) => {
  const items = get(itemsAtom);
  return {
    total: items.length,
    done: items.filter(i => i.done).length,
    active: items.filter(i => !i.done).length,
  };
});
```

### Read-Write Derived Atom

```javascript
// Derived atom có thể đọc và ghi
const uppercaseNameAtom = atom(
  // Getter — đọc
  (get) => get(nameAtom).toUpperCase(),
  // Setter — ghi (tùy chọn)
  (get, set, newUppercase) => {
    set(nameAtom, newUppercase.toLowerCase());
  }
);

function NameComponent() {
  const [uppercaseName, setUppercase] = useAtom(uppercaseNameAtom);
  return (
    <input
      value={uppercaseName}
      onChange={(e) => setUppercase(e.target.value)}
    />
  );
}
```

### Write-only Atom (Action Atoms)

```javascript
// Atom chỉ để ghi — như action trong Redux
const addItemAtom = atom(
  null,  // Không có giá trị đọc
  (get, set, text) => {
    const items = get(itemsAtom);
    set(itemsAtom, [...items, { id: Date.now(), text, done: false }]);
  }
);

const toggleItemAtom = atom(
  null,
  (get, set, id) => {
    set(itemsAtom, get(itemsAtom).map(i =>
      i.id === id ? { ...i, done: !i.done } : i
    ));
  }
);

function AddItemForm() {
  const [text, setText] = useState("");
  const addItem = useSetAtom(addItemAtom);

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      if (text.trim()) { addItem(text); setText(""); }
    }}>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button>Thêm</button>
    </form>
  );
}
```

---

## 4. Async Atoms — Atoms Bất Đồng Bộ

Jotai tích hợp tự nhiên với **Suspense** (tạm dừng) và **async/await**:

```javascript
import { atom } from "jotai";

// Async atom — tự động tích hợp với Suspense
const usersAtom = atom(async () => {
  const response = await fetch("https://jsonplaceholder.typicode.com/users");
  return response.json();
});

// Async derived atom
const userCountAtom = atom(async (get) => {
  const users = await get(usersAtom);
  return users.length;
});
```

```jsx
import { Suspense } from "react";
import { useAtomValue } from "jotai";

function UserList() {
  // Jotai tự động suspend (tạm dừng) component khi data đang load
  const users = useAtomValue(usersAtom);
  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}

function App() {
  return (
    // Suspense xử lý loading state
    <Suspense fallback={<div>Đang tải...</div>}>
      <UserList />
    </Suspense>
  );
}
```

### atomWithQuery — Tích Hợp TanStack Query

```javascript
import { atomWithQuery } from "jotai-tanstack-query";

const postsAtom = atomWithQuery(() => ({
  queryKey: ["posts"],
  queryFn: async () => {
    const res = await fetch("/api/posts");
    return res.json();
  },
}));
```

---

## 5. Atom Utilities

### atomWithStorage — Tự Động Persist

```javascript
import { atomWithStorage } from "jotai/utils";

// Tự động sync với localStorage
const themeAtom = atomWithStorage("theme", "light");
const languageAtom = atomWithStorage("language", "vi");

// Tự động sync với sessionStorage (lưu trữ phiên làm việc)
import { createJSONStorage } from "jotai/utils";
const sessionAtom = atomWithStorage("key", null, createJSONStorage(() => sessionStorage));
```

### atomWithReset — Atom Có Giá Trị Reset

```javascript
import { atomWithReset, useResetAtom } from "jotai/utils";

const filterAtom = atomWithReset("all");

function FilterButtons() {
  const [filter, setFilter] = useAtom(filterAtom);
  const resetFilter = useResetAtom(filterAtom);  // Đặt lại về giá trị ban đầu

  return (
    <div>
      <button onClick={() => setFilter("all")}>Tất cả</button>
      <button onClick={() => setFilter("active")}>Đang làm</button>
      <button onClick={() => setFilter("completed")}>Đã xong</button>
      <button onClick={resetFilter}>Đặt lại bộ lọc</button>
    </div>
  );
}
```

### atomFamily — Atoms Theo Tham Số

```javascript
import { atomFamily } from "jotai/utils";

// Tạo atom cho từng post theo id
const postAtomFamily = atomFamily((postId) =>
  atom(async () => {
    const res = await fetch(`/api/posts/${postId}`);
    return res.json();
  })
);

function PostDetail({ postId }) {
  const post = useAtomValue(postAtomFamily(postId));
  return <div>{post.title}</div>;
}
```

---

## 6. So Sánh

| Khía Cạnh | Jotai | Zustand | Context API |
| --------- | ----- | ------- | ----------- |
| **Mô hình** | Atomic (nguyên tử) | Centralized (tập trung) | Provider/Consumer |
| **Bundle size** | ~3KB | ~1KB | 0KB |
| **Reactivity** | Fine-grained (chi tiết) | Selector-based | Coarse (thô) |
| **Suspense** | Hỗ trợ tự nhiên | Cần xử lý thủ công | Cần xử lý thủ công |
| **Code splitting** | Tự nhiên (atoms ở file riêng) | Cần chia slice thủ công | N/A |
| **Learning curve** | Trung bình | Thấp | Thấp |
| **Phù hợp** | Fine-grained UI, Suspense | App mọi quy mô | State ít thay đổi |

### Khi Nào Chọn Jotai

- ✅ Cần fine-grained reactivity (nhiều component nhỏ, mỗi cái chỉ cần 1 giá trị)
- ✅ Muốn tích hợp tự nhiên với Suspense
- ✅ State có nhiều phụ thuộc chồng chéo (derived state phức tạp)
- ✅ Dự án dùng React Server Components

---

## 7. Ví Dụ Thực Tế

### Todo App Với Jotai

```jsx
// src/atoms/todoAtoms.js
import { atom } from "jotai";
import { atomWithStorage } from "jotai/utils";

export const todosAtom = atomWithStorage("todos", []);
export const filterAtom = atom("all");

// Derived atoms
export const filteredTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  const filter = get(filterAtom);
  if (filter === "active")    return todos.filter(t => !t.done);
  if (filter === "completed") return todos.filter(t => t.done);
  return todos;
});

export const statsAtom = atom((get) => {
  const todos = get(todosAtom);
  return {
    total: todos.length,
    active: todos.filter(t => !t.done).length,
    completed: todos.filter(t => t.done).length,
  };
});

// Action atoms
export const addTodoAtom = atom(null, (get, set, text) => {
  set(todosAtom, [...get(todosAtom), { id: Date.now(), text, done: false }]);
});

export const toggleTodoAtom = atom(null, (get, set, id) => {
  set(todosAtom, get(todosAtom).map(t => t.id === id ? { ...t, done: !t.done } : t));
});

export const deleteTodoAtom = atom(null, (get, set, id) => {
  set(todosAtom, get(todosAtom).filter(t => t.id !== id));
});
```

```jsx
// src/components/TodoApp.jsx
import { useAtomValue, useSetAtom, useAtom } from "jotai";
import { useState } from "react";
import {
  filteredTodosAtom,
  statsAtom,
  filterAtom,
  addTodoAtom,
  toggleTodoAtom,
  deleteTodoAtom,
} from "../atoms/todoAtoms";

function TodoStats() {
  const stats = useAtomValue(statsAtom);
  return (
    <p>Tổng: {stats.total} | Đang làm: {stats.active} | Đã xong: {stats.completed}</p>
  );
}

function TodoForm() {
  const [text, setText] = useState("");
  const addTodo = useSetAtom(addTodoAtom);

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      if (text.trim()) { addTodo(text.trim()); setText(""); }
    }}>
      <input value={text} onChange={e => setText(e.target.value)} placeholder="Thêm việc cần làm..." />
      <button type="submit">Thêm</button>
    </form>
  );
}

function TodoFilter() {
  const [filter, setFilter] = useAtom(filterAtom);
  return (
    <div>
      {["all", "active", "completed"].map(f => (
        <button
          key={f}
          onClick={() => setFilter(f)}
          style={{ fontWeight: filter === f ? "bold" : "normal" }}
        >
          {f === "all" ? "Tất cả" : f === "active" ? "Đang làm" : "Đã xong"}
        </button>
      ))}
    </div>
  );
}

function TodoList() {
  const todos = useAtomValue(filteredTodosAtom);
  const toggleTodo = useSetAtom(toggleTodoAtom);
  const deleteTodo = useSetAtom(deleteTodoAtom);

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input type="checkbox" checked={todo.done} onChange={() => toggleTodo(todo.id)} />
          <span style={{ textDecoration: todo.done ? "line-through" : "none" }}>
            {todo.text}
          </span>
          <button onClick={() => deleteTodo(todo.id)}>Xóa</button>
        </li>
      ))}
    </ul>
  );
}

export function TodoApp() {
  return (
    <div>
      <h1>Công Việc Hôm Nay</h1>
      <TodoStats />
      <TodoForm />
      <TodoFilter />
      <TodoList />
    </div>
  );
}
```

---

## 8. Câu Hỏi Phỏng Vấn

### Câu 1: Atomic state management khác gì centralized state (state tập trung)?

**Trả lời:**

- **Centralized:** Một store duy nhất chứa toàn bộ state. Component truy cập bằng selector. Re-render khi phần state đó thay đổi. (Redux, Zustand)
- **Atomic:** State được chia thành các đơn vị nhỏ nhất gọi là atoms. Component subscribe trực tiếp vào atom cụ thể. Re-render cực kỳ chính xác — chỉ khi atom đó thay đổi.

Atomic model phù hợp hơn khi cần granular (chi tiết) control over re-renders.

### Câu 2: Tại sao Jotai tích hợp tốt với Suspense?

**Trả lời:**

Jotai cho phép định nghĩa async atoms — atoms trả về Promise. Khi component đọc async atom đang pending (đang chờ), Jotai tự động trigger Suspense — component bị "suspend" (tạm dừng) và Suspense boundary hiển thị fallback. Điều này loại bỏ boilerplate `isLoading`/`isError` thủ công.

```jsx
// Không cần if (isLoading) return ... — Suspense lo phần này
const users = useAtomValue(usersAtom);  // Tự động suspend khi đang fetch
```

### Câu 3: Khi nào chọn Jotai vs Zustand?

**Trả lời:**
- **Jotai:** Khi cần fine-grained reactivity, Suspense integration, hoặc state có nhiều derived dependencies
- **Zustand:** Khi cần solution đơn giản với async actions, store ngoài React, hoặc team quen với centralized model
- Trong thực tế: cả hai đều hoạt động tốt cho hầu hết use cases — chọn theo preference của team

---

## 🔗 Điều Hướng

- **Tiếp theo:** [6-comparison-guide.md](./6-comparison-guide.md) — Khi nào dùng gì
- **Trước đó:** [4-zustand.md](./4-zustand.md) — Zustand
- **Quay lại:** [README.md](./README.md) — Tổng quan State Management
