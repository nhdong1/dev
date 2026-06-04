# Redux Toolkit (RTK) — Chuẩn Công Nghiệp

> **Redux Toolkit — RTK** (Bộ Công Cụ Redux Chính Thức) là cách hiện đại, khuyến nghị để viết Redux. RTK giải quyết các vấn đề của Redux cũ: boilerplate quá nhiều, cấu hình phức tạp. Đây là giải pháp được ưa chuộng nhất trong môi trường doanh nghiệp (enterprise).

---

## 📌 Mục Lục

1. [Redux là gì? Tại sao cần Redux Toolkit?](#1-redux-và-redux-toolkit)
2. [Cài đặt và Cấu hình Store](#2-cài-đặt-và-cấu-hình-store)
3. [createSlice — Tạo Slice State](#3-createslice)
4. [createAsyncThunk — Xử Lý Bất Đồng Bộ](#4-createasyncthunk)
5. [Selectors — Lấy Dữ Liệu Từ Store](#5-selectors)
6. [Ví Dụ Thực Tế — Todo App](#6-ví-dụ-thực-tế)
7. [Redux DevTools](#7-redux-devtools)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Redux Và Redux Toolkit

### Flux Architecture (Kiến Trúc Luồng Dữ Liệu)

Redux được xây dựng trên **Flux pattern** — luồng dữ liệu một chiều:

```
Action (Hành Động)  →  Reducer (Bộ Xử Lý)  →  Store (Kho Lưu Trữ)  →  UI
      ↑                                                                    |
      └──────────────────── dispatch (gửi hành động) ──────────────────────┘
```

- **Store:** Nơi chứa toàn bộ state của ứng dụng — "nguồn sự thật duy nhất" (single source of truth)
- **Action:** Object mô tả điều gì xảy ra `{ type: "counter/increment", payload: 1 }`
- **Reducer:** Hàm thuần túy (pure function) nhận state cũ + action → trả về state mới
- **Dispatch:** Hàm để gửi action đến store

### Tại Sao Redux Toolkit Thay Vì Redux Cũ?

```
Redux cũ (nhiều boilerplate):                Redux Toolkit (gọn gàng):

// action types                              // Tất cả trong createSlice
const INCREMENT = "counter/increment";
const DECREMENT = "counter/decrement";

// action creators                           // Tự động tạo
const increment = () => ({ type: INCREMENT });
const decrement = () => ({ type: DECREMENT });

// reducer                                   // Reducer tích hợp trong slice
function counterReducer(state = 0, action) {
  switch (action.type) {
    case INCREMENT: return state + 1;
    case DECREMENT: return state - 1;
    default: return state;
  }
}
```

RTK tích hợp sẵn: **Immer** (cho phép viết "mutating" code), **Redux DevTools**, **createAsyncThunk**.

---

## 2. Cài Đặt Và Cấu Hình Store

```bash
npm install @reduxjs/toolkit react-redux
```

### Tạo Store (Kho Lưu Trữ)

```javascript
// src/store/index.js
import { configureStore } from "@reduxjs/toolkit";
import counterReducer from "./counterSlice";
import todosReducer from "./todosSlice";
import userReducer from "./userSlice";

export const store = configureStore({
  reducer: {
    counter: counterReducer,
    todos: todosReducer,
    user: userReducer,
  },
  // middleware (phần mềm trung gian), devTools được tự động thêm trong development mode
});

// Xuất kiểu TypeScript (nếu dùng TypeScript)
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Kết Nối Store với React

```jsx
// src/main.jsx
import { Provider } from "react-redux";
import { store } from "./store";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")).render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

---

## 3. createSlice — Tạo Slice State

**Slice** (lát cắt) là một phần state kèm theo reducer và actions liên quan.

### Ví Dụ Counter Slice

```javascript
// src/store/counterSlice.js
import { createSlice } from "@reduxjs/toolkit";

const counterSlice = createSlice({
  name: "counter",  // Tiền tố cho tên action: "counter/increment"
  initialState: {
    value: 0,
    step: 1,
  },
  reducers: {
    // RTK dùng Immer — có thể viết code "mutating" nhưng thực chất immutable
    increment(state) {
      state.value += state.step;  // Trông như mutation nhưng Immer xử lý đúng
    },
    decrement(state) {
      state.value -= state.step;
    },
    incrementByAmount(state, action) {
      state.value += action.payload;
    },
    setStep(state, action) {
      state.step = action.payload;
    },
    reset(state) {
      state.value = 0;
    },
  },
});

// Xuất action creators (hàm tạo hành động) — tự động tạo bởi createSlice
export const { increment, decrement, incrementByAmount, setStep, reset } = counterSlice.actions;

// Xuất reducer
export default counterSlice.reducer;
```

### Sử Dụng Trong Component

```jsx
import { useSelector, useDispatch } from "react-redux";
import { increment, decrement, incrementByAmount, reset } from "../store/counterSlice";

function Counter() {
  // useSelector — đọc dữ liệu từ store (subscribe — đăng ký nhận thay đổi)
  const { value, step } = useSelector((state) => state.counter);

  // useDispatch — lấy hàm dispatch để gửi actions
  const dispatch = useDispatch();

  return (
    <div>
      <h2>Bộ Đếm: {value}</h2>
      <p>Bước nhảy: {step}</p>
      <button onClick={() => dispatch(decrement())}>−</button>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(incrementByAmount(10))}>+10</button>
      <button onClick={() => dispatch(reset())}>Đặt lại</button>
    </div>
  );
}
```

### Slice Với State Phức Tạp Hơn

```javascript
// src/store/todosSlice.js
import { createSlice, nanoid } from "@reduxjs/toolkit";

const todosSlice = createSlice({
  name: "todos",
  initialState: {
    items: [],
    filter: "all",  // "all" | "active" | "completed"
  },
  reducers: {
    addTodo: {
      // prepare callback — chuẩn bị payload trước khi vào reducer
      prepare(text) {
        return {
          payload: {
            id: nanoid(),  // nanoid() — tạo ID ngẫu nhiên
            text,
            completed: false,
            createdAt: new Date().toISOString(),
          },
        };
      },
      reducer(state, action) {
        state.items.push(action.payload);
      },
    },
    toggleTodo(state, action) {
      const todo = state.items.find(t => t.id === action.payload);
      if (todo) todo.completed = !todo.completed;
    },
    deleteTodo(state, action) {
      state.items = state.items.filter(t => t.id !== action.payload);
    },
    editTodo(state, action) {
      const { id, text } = action.payload;
      const todo = state.items.find(t => t.id === id);
      if (todo) todo.text = text;
    },
    setFilter(state, action) {
      state.filter = action.payload;
    },
  },
});

export const { addTodo, toggleTodo, deleteTodo, editTodo, setFilter } = todosSlice.actions;
export default todosSlice.reducer;
```

---

## 4. createAsyncThunk — Xử Lý Bất Đồng Bộ

**Thunk** là một hàm trả về hàm khác — dùng để xử lý async logic (logic bất đồng bộ) trước khi dispatch action.

```javascript
// src/store/postsSlice.js
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

// Tạo async thunk (thunk bất đồng bộ)
export const fetchPosts = createAsyncThunk(
  "posts/fetchAll",  // Tên action type
  async (_, { rejectWithValue }) => {
    try {
      const response = await fetch("https://jsonplaceholder.typicode.com/posts");
      if (!response.ok) throw new Error("Lỗi mạng");
      return await response.json();
    } catch (error) {
      // rejectWithValue — trả về giá trị lỗi có thể serialize được
      return rejectWithValue(error.message);
    }
  }
);

export const createPost = createAsyncThunk(
  "posts/create",
  async (postData, { rejectWithValue }) => {
    try {
      const response = await fetch("https://jsonplaceholder.typicode.com/posts", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(postData),
      });
      return await response.json();
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

const postsSlice = createSlice({
  name: "posts",
  initialState: {
    items: [],
    status: "idle",   // "idle" | "loading" | "succeeded" | "failed"
    error: null,
  },
  reducers: {},
  // extraReducers xử lý actions từ bên ngoài slice (thunk actions)
  extraReducers: (builder) => {
    builder
      // fetchPosts
      .addCase(fetchPosts.pending, (state) => {
        state.status = "loading";
        state.error = null;
      })
      .addCase(fetchPosts.fulfilled, (state, action) => {
        state.status = "succeeded";
        state.items = action.payload;
      })
      .addCase(fetchPosts.rejected, (state, action) => {
        state.status = "failed";
        state.error = action.payload;
      })
      // createPost
      .addCase(createPost.fulfilled, (state, action) => {
        state.items.unshift(action.payload);  // Thêm bài mới lên đầu
      });
  },
});

export default postsSlice.reducer;
```

### Sử Dụng Async Thunk Trong Component

```jsx
import { useEffect } from "react";
import { useSelector, useDispatch } from "react-redux";
import { fetchPosts, createPost } from "../store/postsSlice";

function PostsList() {
  const dispatch = useDispatch();
  const { items, status, error } = useSelector((state) => state.posts);

  useEffect(() => {
    // Chỉ fetch một lần
    if (status === "idle") {
      dispatch(fetchPosts());
    }
  }, [status, dispatch]);

  const handleCreate = async () => {
    // unwrap() — throw error nếu thunk bị rejected, cho phép try/catch
    try {
      await dispatch(createPost({ title: "Bài mới", body: "Nội dung..." })).unwrap();
      alert("Tạo bài thành công!");
    } catch (error) {
      alert(`Lỗi: ${error}`);
    }
  };

  if (status === "loading") return <div>Đang tải...</div>;
  if (status === "failed") return <div>Lỗi: {error}</div>;

  return (
    <div>
      <button onClick={handleCreate}>Tạo bài viết</button>
      {items.map(post => (
        <article key={post.id}>
          <h3>{post.title}</h3>
          <p>{post.body}</p>
        </article>
      ))}
    </div>
  );
}
```

---

## 5. Selectors — Lấy Dữ Liệu Từ Store

**Selector** (bộ chọn) là hàm nhận state → trả về dữ liệu cần thiết.

### Selector Cơ Bản

```javascript
// Inline selector trong component
const todos = useSelector((state) => state.todos.items);

// Có thể tính toán trong selector
const activeTodos = useSelector(
  (state) => state.todos.items.filter(t => !t.completed)
);
```

### Memoized Selectors Với createSelector

`createSelector` từ `@reduxjs/toolkit` (re-exported từ Reselect) — chỉ tính toán lại khi input thay đổi:

```javascript
// src/store/todosSelectors.js
import { createSelector } from "@reduxjs/toolkit";

// Input selectors — hàm chọn dữ liệu thô
const selectTodosState = (state) => state.todos;
const selectAllItems = (state) => state.todos.items;
const selectFilter = (state) => state.todos.filter;

// Memoized selector — chỉ tính lại khi items hoặc filter thay đổi
export const selectFilteredTodos = createSelector(
  [selectAllItems, selectFilter],
  (items, filter) => {
    switch (filter) {
      case "active":    return items.filter(t => !t.completed);
      case "completed": return items.filter(t => t.completed);
      default:          return items;
    }
  }
);

export const selectTodoStats = createSelector(
  [selectAllItems],
  (items) => ({
    total: items.length,
    completed: items.filter(t => t.completed).length,
    active: items.filter(t => !t.completed).length,
  })
);
```

---

## 6. Ví Dụ Thực Tế — Todo App Với Redux Toolkit

```
src/
├── store/
│   ├── index.js          ← configureStore
│   ├── todosSlice.js     ← slice + actions + reducer
│   └── todosSelectors.js ← memoized selectors
└── components/
    ├── TodoApp.jsx
    ├── TodoForm.jsx
    ├── TodoList.jsx
    └── TodoStats.jsx
```

```jsx
// src/components/TodoApp.jsx
import TodoForm from "./TodoForm";
import TodoList from "./TodoList";
import TodoStats from "./TodoStats";
import TodoFilter from "./TodoFilter";

function TodoApp() {
  return (
    <div className="todo-app">
      <h1>Danh Sách Công Việc</h1>
      <TodoStats />
      <TodoForm />
      <TodoFilter />
      <TodoList />
    </div>
  );
}

// src/components/TodoForm.jsx
import { useState } from "react";
import { useDispatch } from "react-redux";
import { addTodo } from "../store/todosSlice";

function TodoForm() {
  const [text, setText] = useState("");
  const dispatch = useDispatch();

  const handleSubmit = (e) => {
    e.preventDefault();
    if (!text.trim()) return;
    dispatch(addTodo(text.trim()));
    setText("");
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={text}
        onChange={(e) => setText(e.target.value)}
        placeholder="Thêm công việc mới..."
      />
      <button type="submit">Thêm</button>
    </form>
  );
}

// src/components/TodoList.jsx
import { useSelector, useDispatch } from "react-redux";
import { toggleTodo, deleteTodo } from "../store/todosSlice";
import { selectFilteredTodos } from "../store/todosSelectors";

function TodoList() {
  const todos = useSelector(selectFilteredTodos);
  const dispatch = useDispatch();

  if (todos.length === 0) return <p>Không có công việc nào.</p>;

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id} style={{ textDecoration: todo.completed ? "line-through" : "none" }}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => dispatch(toggleTodo(todo.id))}
          />
          <span>{todo.text}</span>
          <button onClick={() => dispatch(deleteTodo(todo.id))}>Xóa</button>
        </li>
      ))}
    </ul>
  );
}

// src/components/TodoStats.jsx
import { useSelector } from "react-redux";
import { selectTodoStats } from "../store/todosSelectors";

function TodoStats() {
  const { total, completed, active } = useSelector(selectTodoStats);
  return (
    <div>
      <span>Tổng: {total}</span> |
      <span> Đang làm: {active}</span> |
      <span> Đã xong: {completed}</span>
    </div>
  );
}
```

---

## 7. Redux DevTools

**Redux DevTools** (Công Cụ Phát Triển Redux) là một trong những lợi thế lớn nhất của Redux:

- **Time-travel debugging** (debug theo thời gian): Quay lui/tiến đến bất kỳ state nào trong lịch sử
- **Action log** (nhật ký hành động): Xem toàn bộ các action đã dispatch theo thứ tự
- **State diff** (so sánh state): Xem state thay đổi như thế nào sau mỗi action
- **Import/Export state**: Chia sẻ state snapshot để tái tạo bug

```bash
# Cài extension cho Chrome/Firefox: "Redux DevTools"
# RTK tự động kết nối DevTools trong development mode — không cần config thêm
```

---

## 8. Câu Hỏi Phỏng Vấn

### Câu 1: Redux Toolkit khác Redux cũ như thế nào?

**Trả lời:**

| Khía Cạnh | Redux Cũ | Redux Toolkit |
| --------- | -------- | ------------- |
| Boilerplate | Nhiều (action types, action creators, switch-case) | Ít (createSlice tự động tạo) |
| Immutability | Phải viết thủ công (`...spread`) | Immer xử lý tự động |
| Async | Cần middleware riêng (redux-thunk, redux-saga) | createAsyncThunk tích hợp sẵn |
| DevTools | Cần config thủ công | Tự động trong dev mode |
| Best practices | Phải tự áp dụng | Tích hợp sẵn |

### Câu 2: Immer trong RTK hoạt động như thế nào?

**Trả lời:**

Immer tạo ra một **draft state** (bản nháp state) — bạn có thể "mutate" (thay đổi trực tiếp) draft này và Immer sẽ tự động tạo state mới immutable:

```javascript
// Trông như mutation nhưng thực chất immutable
reducers: {
  addItem(state, action) {
    state.items.push(action.payload);  // Immer chặn mutation thật, tạo state mới
  }
}

// Tương đương với viết immutable thủ công:
// return { ...state, items: [...state.items, action.payload] }
```

### Câu 3: Khi nào dùng createAsyncThunk vs RTK Query?

**Trả lời:**
- **createAsyncThunk:** Phù hợp cho async logic phức tạp, mutations một lần (submit form), hoặc khi cần control hoàn toàn
- **RTK Query:** Phù hợp cho data fetching và caching — tự động handle loading state, caching, refetching, invalidation
- Quy tắc thực tế: Nếu cần GET data với caching → RTK Query. Nếu cần POST/PUT/DELETE hoặc logic phức tạp → createAsyncThunk

### Câu 4: Tại sao không nên đặt mọi state vào Redux?

**Trả lời:**

State nên ở Redux chỉ khi:
- Nhiều component không liên quan nhau cần cùng dữ liệu
- Cần persist (lưu trữ) state qua navigation
- Cần debug phức tạp với DevTools

State KHÔNG nên vào Redux:
- UI state cục bộ (modal mở/đóng, input value) → `useState`
- Server data với caching → RTK Query / TanStack Query
- Form state → React Hook Form

---

## 🔗 Điều Hướng

- **Tiếp theo:** [3-rtk-query.md](./3-rtk-query.md) — RTK Query: data fetching & caching
- **Trước đó:** [1-context-api.md](./1-context-api.md) — Context API
- **Quay lại:** [README.md](./README.md) — Tổng quan State Management
