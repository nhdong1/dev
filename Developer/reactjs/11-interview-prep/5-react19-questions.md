# 🆕 Câu Hỏi Về React v19 — Tính Năng Mới

> React v19 (stable từ tháng 12/2024) mang đến nhiều thay đổi quan trọng. Hiểu được React v19 là điểm cộng lớn trong phỏng vấn Senior — cho thấy bạn luôn cập nhật công nghệ.

---

## 🗂️ Mục Lục

1. [Tổng Quan React v19](#tổng-quan)
2. [Server Components & Server Actions](#server-components)
3. [New Hooks — Hook Mới](#new-hooks)
4. [React Compiler — Trình Biên Dịch](#react-compiler)
5. [API Changes — Thay Đổi API](#api-changes)
6. [Migration Guide — Hướng Dẫn Nâng Cấp](#migration)
7. [Câu Hỏi Phỏng Vấn React v19](#câu-hỏi-pv)

---

## Tổng Quan React v19 {#tổng-quan}

### Câu hỏi: React v19 mang lại những thay đổi gì quan trọng nhất?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐⭐

```
React v19 — Những Thay Đổi Lớn:

1. ACTIONS (Form + Server Integration):
   ├── Server Actions — Hàm Server Gọi Từ Client
   ├── useActionState() — Quản lý state từ Actions
   ├── useFormStatus() — Trạng thái form đang submit
   └── useOptimistic() — Cập nhật UI lạc quan

2. REACT COMPILER:
   ├── Tự động memoization — không cần useMemo/useCallback thủ công
   └── Compile-time optimization thay vì runtime

3. NEW HOOKS:
   ├── use() — Đọc Promise và Context trong render
   └── Các hooks Actions mới

4. API SIMPLIFICATIONS:
   ├── ref as prop — không cần forwardRef nữa
   ├── Context.Provider → dùng Context trực tiếp
   ├── Metadata trong JSX (<title>, <meta> trong component)
   ├── Stylesheet support — quản lý stylesheet trong component
   └── Resource Preloading APIs (preload, preinit, prefetchDNS)

5. IMPROVED ERROR HANDLING:
   ├── Better hydration error messages
   ├── Separate roots cho caught và uncaught errors
   └── Error recovery improvements
```

---

## Server Components & Server Actions {#server-components}

### Câu hỏi: Giải thích React Server Components vs Client Components vs SSR

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐⭐

```
Hiểu đúng các khái niệm:

SSR (Server-Side Rendering — Kết Xuất Phía Server):
- Render HTML trên server cho TỪNG REQUEST
- JavaScript vẫn gửi xuống client để "hydrate" — Bơm Sự Tương Tác
- Tồn tại trước React v18 (Next.js pages router)

RSC (React Server Components — Component Phía Server):
- Components chạy HOÀN TOÀN trên server
- KHÔNG gửi JavaScript xuống client
- Có thể là static hoặc dynamic
- Có thể truy cập DB, file system, secrets trực tiếp
- KHÔNG hỗ trợ: useState, useEffect, event handlers, browser APIs

Client Components ('use client'):
- Chạy trên cả server (SSR) VÀ client
- Có thể dùng tất cả hooks, event handlers
- Nhận serializable props từ Server Components
```

```jsx
// Next.js App Router — Server Component (mặc định)
// app/products/page.tsx
async function ProductsPage({ searchParams }) {
  // ✅ Truy cập DB trực tiếp — zero latency
  const products = await db.product.findMany({
    where: { category: searchParams.category },
    include: { images: true, reviews: { take: 3 } },
  });

  // ✅ Access environment secrets — bí mật môi trường
  const apiKey = process.env.INTERNAL_API_KEY; // An toàn, không lộ client

  return (
    <div>
      <ProductFilters /> {/* Server Component — static UI */}
      <ProductGrid products={products} />
      <AddToCartButton /> {/* Client Component — cần interactive */}
    </div>
  );
}
export default ProductsPage;

// components/AddToCartButton.tsx — Client Component
'use client';
import { useState } from 'react';

function AddToCartButton({ productId }: { productId: string }) {
  const [isAdding, setIsAdding] = useState(false);

  const handleClick = async () => {
    setIsAdding(true);
    await addToCart(productId);
    setIsAdding(false);
  };

  return (
    <button onClick={handleClick} disabled={isAdding}>
      {isAdding ? 'Đang Thêm...' : 'Thêm Vào Giỏ'}
    </button>
  );
}
```

**Quy tắc quan trọng:**
```
Server Component CÓ THỂ import Client Component ✅
Client Component KHÔNG THỂ import Server Component ❌
→ Nhưng có thể truyền Server Component như children/props ✅

// ✅ OK: Server Component truyền Server Component như children
function ServerParent() {
  return (
    <ClientWrapper>  {/* Client Component */}
      <ServerChild /> {/* Server Component */}
    </ClientWrapper>
  );
}
```

---

### Câu hỏi: Server Actions — Hành Động Phía Server hoạt động như thế nào?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

Server Actions là async functions chạy trên server, có thể gọi từ Client Components. Dưới hood, React tạo một endpoint đặc biệt và serialize/deserialize arguments.

```tsx
// actions/user.ts — Phải mark 'use server'
'use server';
import { revalidatePath, revalidateTag } from 'next/cache';
import { redirect } from 'next/navigation';

// Server Action để tạo user
export async function createUser(prevState: any, formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // Validate
  if (!name || name.length < 2) {
    return { error: 'Tên phải có ít nhất 2 ký tự', success: false };
  }
  if (!email.includes('@')) {
    return { error: 'Email không hợp lệ', success: false };
  }

  try {
    // Trực tiếp ghi DB — không cần API endpoint!
    await db.user.create({ data: { name, email } });

    // Invalidate cache — Làm Mới Cache
    revalidatePath('/users');
    revalidateTag('users');

    return { error: null, success: true };
  } catch (error) {
    if (error.code === 'P2002') { // Unique constraint violation
      return { error: 'Email đã tồn tại', success: false };
    }
    return { error: 'Lỗi hệ thống', success: false };
  }
}

// Cách 1: Dùng trực tiếp trong form (Server hoặc Client Component)
function UserForm() {
  return (
    <form action={createUser}>
      <input name="name" placeholder="Họ tên" required />
      <input name="email" type="email" placeholder="Email" required />
      <button type="submit">Tạo Tài Khoản</button>
    </form>
  );
}

// Cách 2: Dùng với useActionState (React v19)
'use client';
import { useActionState } from 'react';

function UserFormWithState() {
  const [state, action, isPending] = useActionState(createUser, {
    error: null,
    success: false,
  });

  return (
    <form action={action}>
      {state.error && <p className="error">{state.error}</p>}
      {state.success && <p className="success">Tạo thành công!</p>}

      <input name="name" placeholder="Họ tên" required />
      <input name="email" type="email" placeholder="Email" required />

      <button type="submit" disabled={isPending}>
        {isPending ? 'Đang Tạo...' : 'Tạo Tài Khoản'}
      </button>
    </form>
  );
}

// Cách 3: Dùng programmatically (không phải form)
'use client';
function DeleteButton({ userId }: { userId: string }) {
  const [isPending, startTransition] = useTransition();

  const handleDelete = () => {
    startTransition(async () => {
      await deleteUser(userId); // Server Action
    });
  };

  return (
    <button onClick={handleDelete} disabled={isPending}>
      {isPending ? 'Đang Xóa...' : 'Xóa'}
    </button>
  );
}
```

---

## New Hooks — Hook Mới {#new-hooks}

### Câu hỏi: use() hook mới trong React v19 là gì?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

`use()` là hook đặc biệt có thể **đọc Promise và Context** trong render function. Khác với các hook khác, `use()` có thể được gọi trong điều kiện (inside if/loop).

```jsx
// Đọc Promise
import { use, Suspense } from 'react';

// Fetch trong Server Component, truyền promise xuống Client Component
async function ServerComponent() {
  const userPromise = fetchUser(userId); // Không await — truyền promise

  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

// Client Component dùng use() để unwrap promise
function UserProfile({ userPromise }) {
  const user = use(userPromise); // Suspends — Treo cho đến khi resolve
  return <div>{user.name}</div>;
}

// Đọc Context (thay thế useContext)
const ThemeContext = createContext('light');

function Button({ children }) {
  // Có thể gọi trong điều kiện — khác useContext!
  const theme = use(ThemeContext);
  return <button className={theme}>{children}</button>;
}

// use() trong điều kiện (điểm khác biệt chính với useContext)
function ConditionalTheme({ showTheme, children }) {
  if (showTheme) {
    const theme = use(ThemeContext); // ✅ OK trong React v19
    return <div className={theme}>{children}</div>;
  }
  return <div>{children}</div>;
}
```

---

### Câu hỏi: useActionState vs useState + useTransition — Khi nào dùng gì?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
// useActionState — dành cho form submissions và Server Actions
// Tích hợp với native <form> action prop
// Cung cấp: [state, action, isPending]

import { useActionState } from 'react';

function ContactForm() {
  const [formState, submitAction, isPending] = useActionState(
    async (prevState, formData) => {
      const result = await sendEmail({
        name: formData.get('name'),
        message: formData.get('message'),
      });

      if (result.success) {
        return { success: true, error: null };
      }
      return { success: false, error: result.error };
    },
    { success: false, error: null } // Initial state
  );

  return (
    <form action={submitAction}>
      <input name="name" required />
      <textarea name="message" required />

      {formState.error && <p>{formState.error}</p>}
      {formState.success && <p>Gửi thành công!</p>}

      <button type="submit" disabled={isPending}>
        {isPending ? 'Đang Gửi...' : 'Gửi'}
      </button>
    </form>
  );
}

// useState + useTransition — cho non-form async actions
// Linh hoạt hơn, không tích hợp với form
function DeleteButton({ itemId }) {
  const [error, setError] = useState(null);
  const [isPending, startTransition] = useTransition();

  const handleDelete = () => {
    startTransition(async () => {
      try {
        await deleteItem(itemId);
      } catch (err) {
        setError(err.message);
      }
    });
  };

  return (
    <>
      {error && <span>{error}</span>}
      <button onClick={handleDelete} disabled={isPending}>
        {isPending ? 'Xóa...' : 'Xóa'}
      </button>
    </>
  );
}
```

---

### Câu hỏi: useFormStatus — Trạng Thái Form hoạt động như thế nào?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
import { useFormStatus } from 'react-dom';

// useFormStatus đọc trạng thái của form CHA — không phải form của chính nó
// Phải dùng trong component CON của <form>

// ❌ Sai — dùng trong cùng component với form
function BadForm() {
  const { pending } = useFormStatus(); // Không hoạt động!
  return (
    <form action={submitAction}>
      <button disabled={pending}>Submit</button>
    </form>
  );
}

// ✅ Đúng — dùng trong component con
function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  // pending: boolean — form đang submit
  // data: FormData — dữ liệu form đang submit
  // method: string — method của form
  // action: string | function — action của form

  return (
    <button type="submit" disabled={pending}>
      {pending ? (
        <>
          <Spinner size="sm" />
          Đang Xử Lý...
        </>
      ) : (
        'Gửi'
      )}
    </button>
  );
}

function Form() {
  return (
    <form action={submitAction}>
      <input name="email" type="email" />
      <SubmitButton /> {/* ✅ Trong component con */}
    </form>
  );
}

// Thực tế: Reusable submit button cho nhiều forms
function LoadingButton({ children, ...props }) {
  const { pending } = useFormStatus();
  return (
    <button {...props} disabled={pending || props.disabled}>
      {pending ? <Spinner /> : children}
    </button>
  );
}
```

---

### Câu hỏi: useOptimistic — Cập Nhật Lạc Quan là gì? Implement như thế nào?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

**Optimistic UI — Giao Diện Lạc Quan:** Cập nhật UI ngay lập tức (trước khi server confirm), sau đó đồng bộ với server response. Nếu server fail — thất bại, rollback về trạng thái cũ.

```jsx
import { useOptimistic, useTransition } from 'react';

// useOptimistic(state, updateFn)
// state: giá trị "thật" (từ server/props)
// updateFn: hàm merge optimistic update với state hiện tại
// Returns: [optimisticState, addOptimistic]

function LikeButton({ post }) {
  const [optimisticPost, addOptimisticLike] = useOptimistic(
    post,
    (currentPost, action) => {
      if (action === 'like') {
        return {
          ...currentPost,
          isLiked: true,
          likeCount: currentPost.likeCount + 1,
        };
      }
      return {
        ...currentPost,
        isLiked: false,
        likeCount: currentPost.likeCount - 1,
      };
    }
  );

  const [isPending, startTransition] = useTransition();

  const handleLike = () => {
    const action = optimisticPost.isLiked ? 'unlike' : 'like';

    startTransition(async () => {
      addOptimisticLike(action); // Cập nhật ngay — lạc quan

      try {
        await toggleLike(post.id, action); // Gọi server
        // Nếu thành công: optimistic state trở thành "thật"
      } catch (error) {
        // Nếu thất bại: React tự động rollback về state cũ!
        console.error('Failed to toggle like');
      }
    });
  };

  return (
    <button onClick={handleLike} disabled={isPending}>
      {optimisticPost.isLiked ? '❤️' : '🤍'} {optimisticPost.likeCount}
    </button>
  );
}

// Optimistic update cho list (thêm item)
function TodoList({ initialTodos }) {
  const [todos, setTodos] = useState(initialTodos);

  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (currentTodos, newTodo) => [
      ...currentTodos,
      { ...newTodo, id: 'temp-' + Date.now(), pending: true } // Mark là đang pending
    ]
  );

  const handleAddTodo = async (text) => {
    const tempTodo = { text, completed: false };

    startTransition(async () => {
      addOptimisticTodo(tempTodo); // Hiện ngay trong list

      const savedTodo = await createTodo(text); // Gọi API
      setTodos(prev => [...prev, savedTodo]); // Cập nhật state thật với ID từ server
    });
  };

  return (
    <ul>
      {optimisticTodos.map(todo => (
        <li key={todo.id} style={{ opacity: todo.pending ? 0.7 : 1 }}>
          {todo.text}
          {todo.pending && ' (đang lưu...)'}
        </li>
      ))}
    </ul>
  );
}
```

---

## React Compiler — Trình Biên Dịch {#react-compiler}

### Câu hỏi: React Compiler hoạt động như thế nào? Nó thay thế được hoàn toàn useMemo không?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```
React Compiler (trước đây: React Forget):
- Babel/SWC plugin phân tích code tại compile time
- Tự động thêm memoization khi cần thiết
- Chỉ hoạt động với code tuân thủ Rules of React

Compiler PHÂN TÍCH:
├── Dependencies của từng computation
├── Khi nào giá trị thực sự thay đổi
└── Khi nào safe để skip re-computation

OUTPUT:
├── Tự động wrap với React.memo khi thích hợp
├── Tự động memo calculations (tương đương useMemo)
└── Tự động memo callbacks (tương đương useCallback)
```

```jsx
// Code bạn viết (không có manual memoization)
function ProductList({ products, onSort, minPrice }) {
  const filteredProducts = products.filter(p => p.price >= minPrice);
  const sortedProducts = filteredProducts.sort((a, b) => a.name.localeCompare(b.name));

  return sortedProducts.map(product => (
    <ProductCard key={product.id} product={product} onSort={onSort} />
  ));
}

// Code React Compiler có thể tạo ra (conceptual — không phải exact output)
function ProductList({ products, onSort, minPrice }) {
  const filteredProducts = useMemo(
    () => products.filter(p => p.price >= minPrice),
    [products, minPrice]
  );

  const sortedProducts = useMemo(
    () => filteredProducts.sort((a, b) => a.name.localeCompare(b.name)),
    [filteredProducts]
  );

  const memoizedOnSort = useCallback(onSort, [onSort]);

  return sortedProducts.map(product => (
    <ProductCard key={product.id} product={product} onSort={memoizedOnSort} />
  ));
}
```

**Compiler KHÔNG thể tối ưu khi:**
```jsx
// 1. Mutation — Thay Đổi Trực Tiếp
function BadComponent() {
  const items = [];
  items.push('new item'); // Mutation! Compiler không thể track

  // 2. Non-local side effects
  externalArray.push('item'); // Thay đổi biến ngoài
}

// 3. Rules of Hooks violation
function BadHook({ flag }) {
  if (flag) {
    const [x] = useState(0); // Hook trong điều kiện
  }
}

// ✅ Code safe để Compiler optimize
function GoodComponent({ items, filter }) {
  const filtered = items.filter(item => item.matches(filter)); // Pure — thuần
  const [selected, setSelected] = useState(null); // Hooks ở top level
  return <List items={filtered} onSelect={setSelected} />;
}
```

---

## API Changes — Thay Đổi API {#api-changes}

### Câu hỏi: ref as prop — Không cần forwardRef nữa trong React v19?

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐

```jsx
// React 18 trở về trước — phải dùng forwardRef
const MyInput = React.forwardRef((props, ref) => {
  return <input ref={ref} {...props} />;
});

// Dùng:
const inputRef = useRef(null);
<MyInput ref={inputRef} />

// React v19 — ref là prop bình thường!
function MyInput({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}

// Vẫn hoạt động như cũ:
const inputRef = useRef(null);
<MyInput ref={inputRef} />

// forwardRef vẫn hoạt động trong v19 (backward compatible)
// Nhưng được deprecate — sẽ bị loại bỏ trong phiên bản tương lai
```

---

### Câu hỏi: Metadata trong JSX — Thay đổi trong React v19

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
// React v19: Hỗ trợ <title>, <meta>, <link> trong component
// React tự động hoist chúng lên <head>

function BlogPost({ post }) {
  return (
    <article>
      {/* Tự động đặt trong <head> */}
      <title>{post.title} | My Blog</title>
      <meta name="description" content={post.excerpt} />
      <meta property="og:title" content={post.title} />
      <meta property="og:image" content={post.coverImage} />

      {/* Nội dung bài viết */}
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}

// Không còn cần react-helmet hoặc next/head cho nhiều use cases!
// (Next.js vẫn có metadata API riêng với nhiều tính năng hơn)
```

---

### Câu hỏi: Context.Provider vs dùng Context trực tiếp trong React v19

**Cấp độ:** Mid | **Tần suất:** ⭐⭐⭐

```jsx
// React 18 trở về trước
const ThemeContext = createContext('light');

// Phải dùng ThemeContext.Provider
<ThemeContext.Provider value="dark">
  <App />
</ThemeContext.Provider>

// React v19 — dùng Context trực tiếp như component
<ThemeContext value="dark"> {/* ✅ Viết ngắn hơn */}
  <App />
</ThemeContext>

// ThemeContext.Provider vẫn hoạt động (deprecated, sẽ loại bỏ sau)
```

---

### Câu hỏi: Resource Preloading APIs mới trong React v19

**Cấp độ:** Senior | **Tần suất:** ⭐⭐

```jsx
import { prefetchDNS, preconnect, preload, preinit } from 'react-dom';

function AppInitializer() {
  // Khai báo trong render — React batch và tự động thêm vào <head>

  prefetchDNS('https://fonts.googleapis.com');
  // → <link rel="dns-prefetch" href="https://fonts.googleapis.com">

  preconnect('https://fonts.gstatic.com');
  // → <link rel="preconnect" href="https://fonts.gstatic.com">

  preload('/fonts/main.woff2', { as: 'font', crossOrigin: 'anonymous' });
  // → <link rel="preload" href="/fonts/main.woff2" as="font" crossOrigin="anonymous">

  preinit('https://example.com/critical-script.js', { as: 'script' });
  // → <link rel="modulepreload" href="...">

  return <App />;
}
```

---

## Migration Guide — Hướng Dẫn Nâng Cấp {#migration}

### Câu hỏi: Cách migrate từ React 18 lên React v19?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```
BREAKING CHANGES cần xử lý:

1. ReactDOM.render → createRoot (đã deprecated từ v18, giờ bị remove):
   // ❌ React 17 cách cũ
   ReactDOM.render(<App />, document.getElementById('root'));

   // ✅ React 18/19 cách mới
   const root = createRoot(document.getElementById('root'));
   root.render(<App />);

2. act() từ react-dom/test-utils → import từ react:
   // ❌ Old
   import { act } from 'react-dom/test-utils';
   // ✅ New
   import { act } from 'react';

3. ReactDOM.hydrate → hydrateRoot (đã deprecated từ v18):
   // ❌ Old
   ReactDOM.hydrate(<App />, document.getElementById('root'));
   // ✅ New
   hydrateRoot(document.getElementById('root'), <App />);

4. Xử lý ref cleanup:
   // React v19: cleanup function từ ref callback
   <div ref={(node) => {
     if (node) { /* setup */ }
     return () => { /* cleanup — mới trong v19 */ };
   }} />

5. useDeferredValue — initial value mới:
   // v19: useDeferredValue có thể nhận initial value
   const deferred = useDeferredValue(value, initialValue);
   // Lần render đầu: dùng initialValue, sau đó dùng value thật
```

**Checklist migration:**
```
□ Cập nhật package.json: react@^19, react-dom@^19
□ Chạy React v19 codemod (nếu có)
□ Fix breaking changes
□ Xóa forwardRef (optional, không phải breaking)
□ Xóa Context.Provider (optional)
□ Thêm react-compiler (optional, nếu muốn)
□ Test toàn bộ test suite
□ Chạy trong Strict Mode để phát hiện vấn đề
```

---

## Câu Hỏi Phỏng Vấn React v19 {#câu-hỏi-pv}

### Q1: Tại sao React v19 giới thiệu Actions?

**Trả lời:**
Actions giải quyết pattern lặp lại khi xử lý async mutations — thay đổi dữ liệu:

```
Pattern trước khi có Actions (phải viết thủ công):
├── isSubmitting state
├── error state
├── success state
├── try/catch
└── Cleanup

Actions tự động xử lý tất cả:
├── isPending (từ useFormStatus, useActionState)
├── Error handling
├── Optimistic updates (useOptimistic)
└── Form data serialization
```

---

### Q2: Khi nào nên dùng Server Actions thay vì REST API endpoint?

**Trả lời:**

```
Dùng Server Actions khi:
✅ Next.js App Router project
✅ Mutation đơn giản (form submit, CRUD)
✅ Muốn type-safe từ client đến server mà không cần API schema
✅ Team nhỏ, muốn ít boilerplate
✅ Truy cập thẳng DB (không cần API layer)

Dùng REST API khi:
✅ Public API (mobile app, third-party consume)
✅ Cần versioning API
✅ Multiple clients (web + mobile + desktop)
✅ Team backend riêng biệt
✅ Cần rate limiting, authentication phức tạp ở API level
✅ Non-Next.js project
```

---

### Q3: React Compiler đã production-ready chưa?

**Trả lời (2026):**

React Compiler đã stable từ React v19. Được dùng trong production tại Meta. Cộng đồng đang dần adopt — chấp nhận.

```bash
# Cài đặt
npm install -D babel-plugin-react-compiler
# hoặc nếu dùng Vite:
npm install -D vite-plugin-react-compiler
```

```javascript
// vite.config.js
import react from '@vitejs/plugin-react';
import reactCompiler from 'vite-plugin-react-compiler';

export default {
  plugins: [
    react(),
    reactCompiler(), // Thêm React Compiler
  ],
};
```

**Lưu ý:** Compiler hoạt động opt-in từng component với `'use memo'` directive, hoặc opt-out với `'use no memo'` directive nếu Compiler tạo ra behavior không mong muốn.

---

### Q4: Giải thích Streaming SSR — Kết Xuất Phía Server Phát Trực Tiếp trong React

**Trả lời:**

Streaming SSR cho phép server gửi HTML đến client theo từng phần (chunk) thay vì chờ render toàn bộ:

```jsx
// Trước Streaming: Chờ tất cả data → render → gửi HTML
// Với Streaming: Gửi HTML từng phần khi data sẵn sàng

// app/products/page.tsx (Next.js App Router)
export default async function ProductsPage() {
  return (
    <div>
      {/* HTML này gửi ngay lập tức (static) */}
      <h1>Sản Phẩm</h1>
      <ProductFilters />

      {/* Suspense boundary: gửi skeleton trước, rồi data sau */}
      <Suspense fallback={<ProductListSkeleton />}>
        <ProductList /> {/* Fetch data bên trong — async */}
      </Suspense>

      <Suspense fallback={<RecommendationSkeleton />}>
        <Recommendations /> {/* Fetch data khác — async */}
      </Suspense>
    </div>
  );
}

// Kết quả: Browser nhận HTML skeleton ngay, data fill in dần dần
// FCP (First Contentful Paint) nhanh hơn nhiều so với SSR thông thường
```

---

## ✅ Checklist React v19 — Senior Level

**Server Components:**
- [ ] Phân biệt Server Component vs Client Component vs SSR
- [ ] Biết khi nào dùng 'use client' directive
- [ ] Không import Server Component vào Client Component

**Server Actions:**
- [ ] Sử dụng với form action prop
- [ ] Kết hợp với useActionState
- [ ] Programmatic invocation trong useTransition

**New Hooks:**
- [ ] use() — đọc Promise và Context
- [ ] useActionState — form state management
- [ ] useFormStatus — phải là component con của form
- [ ] useOptimistic — rollback tự động khi thất bại

**API Changes:**
- [ ] ref as prop — không cần forwardRef
- [ ] Context trực tiếp — không cần .Provider
- [ ] Metadata trong JSX — <title>, <meta>

**Migration:**
- [ ] Breaking changes cần fix
- [ ] React Compiler — cách bật và cách nó hoạt động

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 (stable)
