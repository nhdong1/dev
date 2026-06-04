# React v19 Hooks Mới — use(), useActionState(), useFormStatus(), useOptimistic()

> React v19 (ra mắt chính thức tháng 12/2024) giới thiệu bốn hooks/API mới tập trung vào **Actions** (Hành Động) và **form handling** (xử lý form). Chúng giải quyết các pattern phổ biến như loading state, error handling, và optimistic updates theo cách khai báo (declarative), giảm thiểu boilerplate đáng kể.

---

## 📌 Mục Lục

1. [Actions — Khái Niệm Cốt Lõi](#1-actions-khái-niệm-cốt-lõi)
2. [use() — Hook Đọc Promise và Context](#2-use-hook)
3. [useActionState() — Quản Lý State Từ Actions](#3-useactionstate)
4. [useFormStatus() — Trạng Thái Submit Form Cha](#4-useformstatus)
5. [useOptimistic() — Cập Nhật UI Lạc Quan](#5-useoptimistic)
6. [Tổng Hợp — Ví Dụ Đầy Đủ](#6-tổng-hợp)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Actions — Khái Niệm Cốt Lõi

### Trước React v19 — Pattern Lặp Đi Lặp Lại

Mọi form submission đều cần viết đi viết lại cùng một boilerplate (mã lặp):

```jsx
// ❌ Pattern cũ — phải viết lại mỗi lần
function OldForm() {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  const [success, setSuccess] = useState(false);

  async function handleSubmit(e) {
    e.preventDefault();
    setIsLoading(true);
    setError(null);

    try {
      await submitData(new FormData(e.target));
      setSuccess(true);
    } catch (err) {
      setError(err.message);
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      {isLoading && <p>Đang xử lý...</p>}
      {error && <p style={{ color: "red" }}>{error}</p>}
      {success && <p>Thành công!</p>}
      <button disabled={isLoading} type="submit">Gửi</button>
    </form>
  );
}
```

### React v19 — Actions Pattern

**Action** là hàm async truyền vào `action` prop của `<form>` hoặc xử lý qua `useActionState`. React tự động quản lý loading/error state:

```jsx
// ✅ React v19 — gọn gàng hơn rất nhiều
async function submitAction(formData) {
  "use server"; // Server Action (trong Next.js) — hoặc bỏ dòng này cho Client Action
  await saveToDatabase(Object.fromEntries(formData));
}

function NewForm() {
  return (
    <form action={submitAction}>
      <input name="email" type="email" />
      <SubmitButton /> {/* useFormStatus() bên trong */}
    </form>
  );
}
```

---

## 2. use() — Hook Đọc Promise và Context

### Khái Niệm

`use()` là API mới (không phải hook thông thường) với hai công dụng:
1. **Đọc Promise** — "unwrap" (mở gói) một Promise trong render, tích hợp với Suspense
2. **Đọc Context** — thay thế `useContext`, nhưng **có thể dùng trong điều kiện**

### use() Với Context

```jsx
import { use, createContext } from "react";

const ThemeContext = createContext("light");

function Button({ children }) {
  // ✅ use() có thể gọi trong điều kiện — khác với useContext
  if (someCondition) {
    const theme = use(ThemeContext); // Hợp lệ trong React v19!
    return <button className={theme}>{children}</button>;
  }

  return <button>{children}</button>;
}
```

> **Lưu ý quan trọng:** `use()` có thể gọi trong điều kiện, nhưng phải **luôn được gọi cùng số lần hoặc ít hơn** trong mỗi render. Đây vẫn là đặc biệt — các hooks thông thường vẫn không được gọi trong điều kiện.

### use() Với Promise — Streaming Data (Dữ Liệu Theo Luồng)

```jsx
import { use, Suspense } from "react";

// Server Component (hoặc nơi tạo Promise)
function DataProvider() {
  const dataPromise = fetch("/api/data").then(res => res.json());

  return (
    <Suspense fallback={<LoadingSkeleton />}>
      <DataDisplay dataPromise={dataPromise} />
    </Suspense>
  );
}

// Client Component nhận Promise qua props
function DataDisplay({ dataPromise }) {
  // use() "suspend" (tạm dừng) component cho đến khi Promise resolve
  const data = use(dataPromise);

  return (
    <ul>
      {data.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

### Xử Lý Lỗi Với use() và Error Boundary

```jsx
import { use, Suspense } from "react";

// Error Boundary — Ranh Giới Lỗi — bắt lỗi từ use()
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <p>Có lỗi xảy ra: {this.state.error.message}</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Thử lại
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

function App() {
  const userPromise = fetchUser(userId);

  return (
    <ErrorBoundary>
      <Suspense fallback={<Spinner />}>
        <UserProfile userPromise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}

function UserProfile({ userPromise }) {
  const user = use(userPromise); // Throw nếu Promise reject → Error Boundary bắt
  return <h1>Xin chào, {user.name}!</h1>;
}
```

---

## 3. useActionState() — Quản Lý State Từ Actions

### Khái Niệm

`useActionState()` — Hook Trạng Thái Action — quản lý state của một action (thường là async function xử lý form). Tự động:
- Theo dõi trạng thái pending (đang chờ)
- Nhận giá trị trả về từ action
- Tích hợp với Server Actions trong Next.js

```jsx
const [state, formAction, isPending] = useActionState(action, initialState);
// state      — state hiện tại (giá trị trả về từ action lần trước)
// formAction — hàm action đã được wrap, truyền vào <form action={...}>
// isPending  — true khi action đang chạy (async)
```

### Ví Dụ — Form Đăng Ký

```jsx
import { useActionState } from "react";

// Action function — có thể là Server Action hoặc Client Action
async function registerAction(prevState, formData) {
  // prevState — state trước đó (lần action chạy trước)
  // formData — FormData từ form

  const email = formData.get("email");
  const password = formData.get("password");

  // Validation
  if (!email.includes("@")) {
    return { success: false, error: "Email không hợp lệ" };
  }

  if (password.length < 8) {
    return { success: false, error: "Mật khẩu phải ít nhất 8 ký tự" };
  }

  try {
    await createUser({ email, password });
    return { success: true, error: null, message: "Đăng ký thành công!" };
  } catch (err) {
    return { success: false, error: err.message };
  }
}

function RegisterForm() {
  const [state, formAction, isPending] = useActionState(
    registerAction,
    { success: false, error: null, message: null } // initialState
  );

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" required />
      </div>

      <div>
        <label htmlFor="password">Mật khẩu</label>
        <input id="password" name="password" type="password" required />
      </div>

      {/* Hiển thị lỗi từ action */}
      {state.error && (
        <p style={{ color: "red" }}>{state.error}</p>
      )}

      {/* Hiển thị thành công */}
      {state.success && (
        <p style={{ color: "green" }}>{state.message}</p>
      )}

      <button type="submit" disabled={isPending}>
        {isPending ? "Đang xử lý..." : "Đăng ký"}
      </button>
    </form>
  );
}
```

### useActionState Với Server Actions (Next.js)

```jsx
// app/actions.ts — Server Actions
"use server";

import { redirect } from "next/navigation";
import { z } from "zod";

const LoginSchema = z.object({
  email: z.string().email("Email không hợp lệ"),
  password: z.string().min(6, "Mật khẩu tối thiểu 6 ký tự"),
});

export async function loginAction(prevState, formData) {
  const parsed = LoginSchema.safeParse({
    email: formData.get("email"),
    password: formData.get("password"),
  });

  if (!parsed.success) {
    return {
      errors: parsed.error.flatten().fieldErrors,
      message: "Thông tin không hợp lệ",
    };
  }

  const { email, password } = parsed.data;
  const result = await authenticateUser(email, password);

  if (!result.success) {
    return { errors: {}, message: "Email hoặc mật khẩu không đúng" };
  }

  redirect("/dashboard");
}
```

```jsx
// app/login/page.tsx — Client Component
"use client";

import { useActionState } from "react";
import { loginAction } from "../actions";

export default function LoginPage() {
  const [state, formAction, isPending] = useActionState(loginAction, {
    errors: {},
    message: null,
  });

  return (
    <form action={formAction}>
      <input name="email" type="email" />
      {state.errors.email && <p>{state.errors.email[0]}</p>}

      <input name="password" type="password" />
      {state.errors.password && <p>{state.errors.password[0]}</p>}

      {state.message && <p>{state.message}</p>}

      <button disabled={isPending}>
        {isPending ? "Đang đăng nhập..." : "Đăng nhập"}
      </button>
    </form>
  );
}
```

---

## 4. useFormStatus() — Trạng Thái Submit Form Cha

### Khái Niệm

`useFormStatus()` — Hook Trạng Thái Form — cho phép component con biết trạng thái submit của `<form>` **cha** gần nhất. Giải quyết vấn đề cũ: cần truyền `isLoading` prop xuống từng button con.

```jsx
const { pending, data, method, action } = useFormStatus();
// pending — boolean — true khi form đang submit
// data    — FormData — dữ liệu đang được submit
// method  — string — "get" hoặc "post"
// action  — string | function — action của form
```

### Ví Dụ — Reusable Submit Button (Nút Submit Tái Sử Dụng)

```jsx
import { useFormStatus } from "react-dom";

// Component nút submit biết trạng thái form cha — không cần props!
function SubmitButton({ children = "Gửi" }) {
  const { pending } = useFormStatus();

  return (
    <button
      type="submit"
      disabled={pending}
      aria-busy={pending} // Accessibility
    >
      {pending ? (
        <>
          <Spinner /> Đang xử lý...
        </>
      ) : (
        children
      )}
    </button>
  );
}

// Dùng trong bất kỳ form nào — SubmitButton tự biết khi nào pending
function ContactForm() {
  return (
    <form action={sendMessageAction}>
      <input name="name" type="text" placeholder="Tên của bạn" />
      <textarea name="message" placeholder="Nội dung" />
      <SubmitButton>Gửi tin nhắn</SubmitButton>
    </form>
  );
}

function NewsletterForm() {
  return (
    <form action={subscribeAction}>
      <input name="email" type="email" placeholder="Email của bạn" />
      <SubmitButton>Đăng ký nhận tin</SubmitButton>
    </form>
  );
}
```

### Hiển Thị Preview Khi Đang Submit

```jsx
function ImageUploadPreview() {
  const { pending, data } = useFormStatus();

  if (!pending) return null;

  // Hiển thị preview ảnh đang được upload
  const file = data?.get("image");
  if (!file || typeof file !== "object") return null;

  return (
    <div className="upload-preview">
      <img
        src={URL.createObjectURL(file)}
        alt="Preview"
        style={{ opacity: 0.6 }}
      />
      <p>Đang tải lên...</p>
    </div>
  );
}

function UploadForm() {
  return (
    <form action={uploadImageAction} encType="multipart/form-data">
      <input name="image" type="file" accept="image/*" />
      <ImageUploadPreview />
      <SubmitButton>Tải lên</SubmitButton>
    </form>
  );
}
```

> **Quan trọng:** `useFormStatus()` chỉ hoạt động trong component **con** (descendant) của `<form>`, không hoạt động trong chính component chứa form.

---

## 5. useOptimistic() — Cập Nhật UI Lạc Quan

### Khái Niệm

**Optimistic UI** (Giao Diện Lạc Quan) là pattern: cập nhật UI ngay lập tức khi người dùng thực hiện hành động, **trước khi** server xác nhận. Nếu server thành công, giữ nguyên. Nếu server lỗi, rollback (khôi phục) về state ban đầu.

`useOptimistic()` — Hook Lạc Quan — quản lý pattern này:

```jsx
const [optimisticState, addOptimistic] = useOptimistic(
  actualState,        // State thật từ server/database
  (currentState, optimisticValue) => {
    // Hàm tính state tạm thời khi đang chờ server
    return mergedState;
  }
);
```

### Ví Dụ — Like Button (Nút Thích)

```jsx
import { useState, useOptimistic } from "react";

function LikeButton({ postId, initialLiked, initialCount }) {
  const [liked, setLiked] = useState(initialLiked);
  const [count, setCount] = useState(initialCount);

  // optimisticLiked: giá trị "lạc quan" hiển thị cho user ngay lập tức
  const [optimisticLiked, toggleOptimisticLiked] = useOptimistic(
    liked,
    (currentLiked) => !currentLiked // Đảo ngược ngay khi click
  );

  const [optimisticCount, updateOptimisticCount] = useOptimistic(
    count,
    (currentCount, delta) => currentCount + delta // Thêm/bớt delta
  );

  async function handleLike() {
    // Bước 1: Cập nhật UI ngay lập tức (optimistic)
    toggleOptimisticLiked();
    updateOptimisticCount(optimisticLiked ? -1 : +1);

    // Bước 2: Gọi API (bất đồng bộ)
    try {
      const result = await toggleLike(postId);
      // Bước 3: Cập nhật state thật từ kết quả server
      setLiked(result.liked);
      setCount(result.count);
    } catch (err) {
      // Bước 3 (lỗi): useOptimistic tự rollback về state thật (liked, count)
      console.error("Lỗi khi like:", err);
    }
  }

  return (
    <button onClick={handleLike}>
      {optimisticLiked ? "❤️" : "🤍"} {optimisticCount}
    </button>
  );
}
```

### Ví Dụ — Todo List Với Optimistic Updates

```jsx
import { useState, useOptimistic, useTransition } from "react";

function TodoList({ initialTodos, userId }) {
  const [todos, setTodos] = useState(initialTodos);
  const [isPending, startTransition] = useTransition();

  const [optimisticTodos, setOptimisticTodo] = useOptimistic(
    todos,
    (currentTodos, { type, payload }) => {
      switch (type) {
        case "ADD":
          return [...currentTodos, { ...payload, id: "temp-" + Date.now(), isOptimistic: true }];
        case "TOGGLE":
          return currentTodos.map(t =>
            t.id === payload.id ? { ...t, done: !t.done, isOptimistic: true } : t
          );
        case "DELETE":
          return currentTodos.filter(t => t.id !== payload.id);
        default:
          return currentTodos;
      }
    }
  );

  async function addTodo(text) {
    const newTodo = { text, done: false, userId };

    startTransition(async () => {
      // Hiển thị ngay lập tức
      setOptimisticTodo({ type: "ADD", payload: newTodo });

      try {
        const saved = await createTodo(newTodo);
        setTodos(prev => [...prev, saved]); // Cập nhật với ID thật từ server
      } catch (err) {
        // useOptimistic tự rollback
        alert("Không thể thêm todo: " + err.message);
      }
    });
  }

  async function deleteTodo(id) {
    startTransition(async () => {
      setOptimisticTodo({ type: "DELETE", payload: { id } });

      try {
        await removeTodo(id);
        setTodos(prev => prev.filter(t => t.id !== id));
      } catch (err) {
        alert("Không thể xóa todo: " + err.message);
      }
    });
  }

  return (
    <div>
      <ul>
        {optimisticTodos.map(todo => (
          <li
            key={todo.id}
            style={{ opacity: todo.isOptimistic ? 0.6 : 1 }}
          >
            <span style={{ textDecoration: todo.done ? "line-through" : "none" }}>
              {todo.text}
            </span>
            {todo.isOptimistic && " (đang lưu...)"}
            <button onClick={() => deleteTodo(todo.id)}>Xóa</button>
          </li>
        ))}
      </ul>
      <form action={formData => addTodo(formData.get("text"))}>
        <input name="text" placeholder="Todo mới..." required />
        <SubmitButton>Thêm</SubmitButton>
      </form>
    </div>
  );
}
```

---

## 6. Tổng Hợp — Ví Dụ Đầy Đủ

Kết hợp cả 4 APIs trong một comment form hoàn chỉnh:

```jsx
"use client";
import { useActionState, useOptimistic, useTransition } from "react";
import { useFormStatus } from "react-dom";
import { postCommentAction } from "./actions";

// SubmitButton dùng useFormStatus
function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? "Đang đăng..." : "Đăng bình luận"}
    </button>
  );
}

function CommentSection({ postId, initialComments }) {
  const [comments, setComments] = useState(initialComments);
  const [isPending, startTransition] = useTransition();

  // Optimistic comments
  const [optimisticComments, addOptimisticComment] = useOptimistic(
    comments,
    (current, newComment) => [...current, { ...newComment, isOptimistic: true }]
  );

  // Action state cho form
  const [formState, formAction] = useActionState(
    async (prevState, formData) => {
      const text = formData.get("comment");
      const tempComment = { id: "temp", text, author: "Bạn", createdAt: new Date() };

      startTransition(() => {
        addOptimisticComment(tempComment); // Hiển thị ngay
      });

      try {
        const saved = await postCommentAction(postId, text);
        setComments(prev => [...prev, saved]); // Cập nhật state thật
        return { success: true, error: null };
      } catch (err) {
        return { success: false, error: err.message };
      }
    },
    { success: false, error: null }
  );

  return (
    <div>
      <h3>Bình luận ({optimisticComments.length})</h3>
      <ul>
        {optimisticComments.map(comment => (
          <li key={comment.id} style={{ opacity: comment.isOptimistic ? 0.5 : 1 }}>
            <strong>{comment.author}</strong>: {comment.text}
            {comment.isOptimistic && " (đang đăng...)"}
          </li>
        ))}
      </ul>

      <form action={formAction}>
        <textarea name="comment" placeholder="Viết bình luận..." required />
        {formState.error && <p style={{ color: "red" }}>{formState.error}</p>}
        <SubmitButton />
      </form>
    </div>
  );
}
```

---

## 7. Câu Hỏi Phỏng Vấn

### Q1: use() khác gì với useEffect cho data fetching?

**Trả lời:**
- `useEffect` + state: Fetch xảy ra **sau** render, cần `isLoading` state, component render 2 lần (lần đầu với loading state, lần sau với data)
- `use(promise)`: Tích hợp với **Suspense**, component "pause" (tạm dừng) cho đến khi Promise resolve, Suspense fallback hiển thị thay cho loading state. Kết quả: code đơn giản hơn, không cần quản lý loading state thủ công.

### Q2: useActionState thay thế gì trong React cũ?

**Trả lời:** Thay thế pattern:
```jsx
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState(null);
const [data, setData] = useState(null);

async function handleSubmit(e) {
  e.preventDefault();
  setIsLoading(true);
  try { ... setData(result); }
  catch (err) { setError(err.message); }
  finally { setIsLoading(false); }
}
```
→ Pattern này xuất hiện ở mọi form → `useActionState` gom tất cả lại.

### Q3: useFormStatus có thể truy cập form data khi đang pending không?

**Trả lời:** Có. `useFormStatus()` trả về `{ pending, data, method, action }`. Khi `pending = true`, `data` là `FormData` đang được submit. Điều này cho phép hiển thị preview của dữ liệu đang upload (ảnh, file...) ngay khi đang chờ server xử lý.

### Q4: useOptimistic rollback như thế nào khi server lỗi?

**Trả lời:** `useOptimistic` chỉ hiển thị optimistic value **trong khi action đang pending**. Khi action hoàn thành (dù thành công hay thất bại), `optimisticState` tự động quay về `actualState` (state thật được truyền vào). Vì vậy, nếu bạn không cập nhật `actualState` sau khi server lỗi, UI sẽ tự rollback về state cũ.

### Q5: Server Actions là gì và khác gì Client Actions?

**Trả lời:**
- **Server Actions** (Hành Động Phía Server): Hàm async được đánh dấu `"use server"`, chạy trên server. Client không thấy code, tự động nhận CSRF protection, có thể trực tiếp query database. Chỉ dùng với Next.js 14+ hoặc frameworks hỗ trợ.
- **Client Actions** (Hành Động Phía Client): Hàm async thông thường trong component, chạy trên browser. Vẫn phải gọi API thủ công để giao tiếp với server.

Cả hai đều hoạt động với `useActionState`, `useFormStatus`, và `form action` prop.

---

## 🔗 Điều Hướng

- **Trở về:** [README.md](./README.md) — Tổng quan Hooks
- **Trước đó:** [5-advanced-hooks.md](./5-advanced-hooks.md) — useId, useTransition, useDeferredValue
- **Tiếp theo:** [7-custom-hooks.md](./7-custom-hooks.md) — Custom Hooks

**Cập Nhật Lần Cuối:** 2026-06-04 | **React Version:** v19 (stable)
