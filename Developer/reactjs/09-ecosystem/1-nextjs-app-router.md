# Next.js App Router — Server & Client Components

> Next.js App Router (giới thiệu từ Next.js 13, ổn định từ Next.js 14) là mô hình routing dựa trên thư mục `app/`, tích hợp sâu React Server Components (RSC — Component Phía Server)

---

## 1. Tổng Quan App Router

### App Router vs Pages Router

| Khía Cạnh | App Router (`app/`) | Pages Router (`pages/`) |
| --------- | ------------------- | ----------------------- |
| **Mặc định** | Server Components | Client Components |
| **Data Fetching** | `async/await` trong component | `getServerSideProps`, `getStaticProps` |
| **Layout** | `layout.tsx` lồng nhau | `_app.tsx` (một cấp) |
| **Loading** | `loading.tsx` tích hợp | Tự xử lý |
| **Error** | `error.tsx` tích hợp | ErrorBoundary thủ công |
| **Streaming** | Hỗ trợ gốc | Không hỗ trợ |
| **Caching** | Hệ thống cache đa tầng | Đơn giản hơn |

### Cấu Trúc Thư Mục App Router

```
app/
├── layout.tsx          ← Root Layout — Bố cục gốc (bắt buộc)
├── page.tsx            ← Trang chủ "/"
├── loading.tsx         ← Loading UI cho route này
├── error.tsx           ← Error UI cho route này
├── not-found.tsx       ← 404 page
├── globals.css
│
├── dashboard/
│   ├── layout.tsx      ← Layout riêng cho /dashboard/*
│   ├── page.tsx        ← Trang /dashboard
│   └── analytics/
│       └── page.tsx    ← Trang /dashboard/analytics
│
├── blog/
│   ├── page.tsx        ← Trang /blog
│   └── [slug]/         ← Dynamic segment — Đoạn động
│       └── page.tsx    ← Trang /blog/:slug
│
└── api/
    └── users/
        └── route.ts    ← API Route Handler
```

---

## 2. Server Components vs Client Components

### Server Components — Component Phía Server (Mặc Định)

Server Components chạy **chỉ trên server**, không gửi JavaScript xuống browser.

**Đặc điểm:**
- Có thể `async/await` trực tiếp
- Truy cập filesystem, database, biến môi trường server
- Không có state, hooks, hoặc event listeners
- Không re-render trên client
- Giảm bundle size đáng kể

```tsx
// app/users/page.tsx — Đây là Server Component (không cần khai báo)
// Không có 'use client' ở đầu file

async function getUsersFromDB() {
  // Gọi thẳng DB hoặc internal API — không bị lộ ra client
  const users = await db.query('SELECT * FROM users');
  return users;
}

export default async function UsersPage() {
  const users = await getUsersFromDB(); // await trực tiếp trong component!

  return (
    <main>
      <h1>Danh Sách Người Dùng</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>{user.name} — {user.email}</li>
        ))}
      </ul>
    </main>
  );
}
```

### Client Components — Component Phía Client

Thêm `'use client'` directive (chỉ thị) ở đầu file.

**Khi nào dùng Client Components:**
- Cần `useState`, `useEffect`, hooks khác
- Xử lý sự kiện (onClick, onChange, ...)
- Dùng browser API (localStorage, window, ...)
- Cần animation hoặc real-time updates

```tsx
'use client'; // Chỉ thị này đánh dấu đây là Client Component

import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Số đếm: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Tăng</button>
    </div>
  );
}
```

### Quy Tắc Kết Hợp — Composition Rules

```
Server Component (cha)
├── Server Component (con) ✅ Bình thường
├── Client Component (con) ✅ Import và dùng như thường
│   └── Server Component (cháu) ❌ KHÔNG được — Client không import Server
│
└── Truyền Server Component vào Client qua props/children ✅
```

**Pattern đúng — Server Component truyền qua children:**

```tsx
// ServerWrapper.tsx — Server Component
import { ClientShell } from './ClientShell';

export default function ServerWrapper() {
  const data = await fetchData(); // Chỉ Server mới làm được

  return (
    <ClientShell>
      {/* ServerCard là Server Component, truyền vào qua children */}
      <ServerCard data={data} />
    </ClientShell>
  );
}
```

```tsx
'use client';
// ClientShell.tsx — Client Component
import { useState } from 'react';

export function ClientShell({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setOpen(!open)}>Toggle</button>
      {open && children} {/* children là Server Component — OK! */}
    </div>
  );
}
```

---

## 3. Routing Nâng Cao

### Dynamic Routes — Route Động

```
app/
└── blog/
    └── [slug]/
        └── page.tsx   ← /blog/bai-viet-1, /blog/react-hooks
```

```tsx
// app/blog/[slug]/page.tsx
interface PageProps {
  params: { slug: string };
  searchParams: { [key: string]: string | string[] | undefined };
}

export default async function BlogPost({ params }: PageProps) {
  const post = await getPostBySlug(params.slug);

  if (!post) {
    notFound(); // Hiển thị not-found.tsx
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}

// generateStaticParams — Tạo tham số tĩnh trước (SSG — Static Site Generation)
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map(post => ({ slug: post.slug }));
}
```

### Catch-all Routes — Route Bắt Tất Cả

```
app/docs/[...slug]/page.tsx   ← /docs/a, /docs/a/b, /docs/a/b/c
app/docs/[[...slug]]/page.tsx ← Cả /docs/ cũng khớp (optional catch-all)
```

### Route Groups — Nhóm Route

```
app/
├── (marketing)/          ← Nhóm, KHÔNG ảnh hưởng URL
│   ├── about/page.tsx    ← /about
│   └── contact/page.tsx  ← /contact
│
└── (app)/               ← Nhóm khác, layout khác
    ├── layout.tsx        ← Layout chỉ cho nhóm (app)
    ├── dashboard/page.tsx ← /dashboard
    └── settings/page.tsx  ← /settings
```

### Parallel Routes — Route Song Song

```
app/
└── @modal/            ← Slot tên "modal"
    └── (.)photo/      ← Intercepting route
        └── [id]/
            └── page.tsx
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  modal,         // Nhận slot @modal
}: {
  children: React.ReactNode;
  modal: React.ReactNode;
}) {
  return (
    <>
      {children}
      {modal}  {/* Render modal song song với nội dung chính */}
    </>
  );
}
```

---

## 4. Layouts & Templates

### Nested Layouts — Layout Lồng Nhau

```tsx
// app/layout.tsx — Root Layout (bắt buộc)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="vi">
      <body>
        <header>Navigation toàn cục</header>
        <main>{children}</main>
        <footer>Footer toàn cục</footer>
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx — Dashboard Layout lồng trong Root
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="dashboard">
      <aside>Sidebar chỉ cho dashboard</aside>
      <section>{children}</section>
    </div>
  );
}
```

### Template vs Layout

- **Layout**: Giữ state, không re-mount khi navigate giữa các route con
- **Template** (`template.tsx`): Re-mount mỗi lần navigate — dùng khi cần animation, effect trên mỗi route change

---

## 5. Data Fetching Trong App Router

### Fetch Trực Tiếp Trong Server Component

```tsx
// app/products/page.tsx
export default async function ProductsPage() {
  // Next.js extend fetch() với caching tự động
  const res = await fetch('https://api.example.com/products', {
    next: {
      revalidate: 3600, // ISR — Tái tạo sau 3600 giây (1 giờ)
      // revalidate: 0     // SSR — Luôn fetch mới
      // cache: 'force-cache' // SSG — Cache vĩnh viễn (mặc định)
    },
  });
  const products = await res.json();

  return <ProductList products={products} />;
}
```

### Caching Strategies — Chiến Lược Cache

| Strategy | Config | Hành Vi |
| -------- | ------ | ------- |
| **Static** (SSG) | `cache: 'force-cache'` hoặc mặc định | Render lúc build, cache mãi |
| **Dynamic** (SSR) | `cache: 'no-store'` hoặc `revalidate: 0` | Render mỗi request |
| **ISR** | `revalidate: N` (giây) | Static + tự động tái tạo sau N giây |
| **On-demand ISR** | `revalidateTag()` / `revalidatePath()` | Tái tạo theo yêu cầu (webhook, event) |

### Parallel Data Fetching — Lấy Dữ Liệu Song Song

```tsx
export default async function Dashboard() {
  // Lấy song song — Promise.all tránh waterfall
  const [user, orders, analytics] = await Promise.all([
    fetchUser(),
    fetchOrders(),
    fetchAnalytics(),
  ]);

  return (
    <div>
      <UserCard user={user} />
      <OrderList orders={orders} />
      <AnalyticsDashboard data={analytics} />
    </div>
  );
}
```

### Streaming với Suspense

```tsx
import { Suspense } from 'react';

export default function Page() {
  return (
    <main>
      <h1>Dashboard</h1>

      {/* SlowComponent stream về sau — không block toàn trang */}
      <Suspense fallback={<Skeleton />}>
        <SlowDataComponent />
      </Suspense>

      <Suspense fallback={<ChartSkeleton />}>
        <AnalyticsChart />
      </Suspense>
    </main>
  );
}
```

---

## 6. Server Actions — Hành Động Phía Server

Server Actions cho phép gọi hàm server trực tiếp từ component — không cần tạo API endpoint riêng.

```tsx
// app/actions.ts
'use server'; // Chỉ thị — toàn bộ file này là Server Actions

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // Validation
  if (!email.includes('@')) {
    return { error: 'Email không hợp lệ' };
  }

  // Lưu vào DB — chạy trên server, an toàn
  await db.users.create({ name, email });

  // Revalidate cache sau khi thay đổi dữ liệu
  revalidatePath('/users');
  return { success: true };
}
```

```tsx
// app/users/new/page.tsx — Server Component dùng Server Action
import { createUser } from '../actions';

export default function NewUserPage() {
  return (
    <form action={createUser}>  {/* action nhận Server Action trực tiếp */}
      <input name="name" placeholder="Tên" required />
      <input name="email" type="email" placeholder="Email" required />
      <button type="submit">Tạo Người Dùng</button>
    </form>
  );
}
```

**Dùng với useActionState (React v19):**

```tsx
'use client';
import { useActionState } from 'react';
import { createUser } from '../actions';

export function CreateUserForm() {
  const [state, formAction, isPending] = useActionState(createUser, null);

  return (
    <form action={formAction}>
      <input name="name" />
      <input name="email" type="email" />
      <button disabled={isPending}>
        {isPending ? 'Đang tạo...' : 'Tạo Người Dùng'}
      </button>
      {state?.error && <p className="error">{state.error}</p>}
    </form>
  );
}
```

---

## 7. Metadata — Thẻ Meta Động

```tsx
// app/blog/[slug]/page.tsx
import { Metadata } from 'next';

// Static metadata — Metadata tĩnh
export const metadata: Metadata = {
  title: 'Blog | My App',
  description: 'Đọc các bài viết mới nhất',
};

// Dynamic metadata — Metadata động
export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const post = await getPostBySlug(params.slug);

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      images: [post.coverImage],
    },
  };
}
```

---

## 8. Middleware — Phần Mềm Trung Gian

```tsx
// middleware.ts (đặt ở root, ngang hàng với app/)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value;

  // Bảo vệ route /dashboard/*
  if (request.nextUrl.pathname.startsWith('/dashboard')) {
    if (!token) {
      // Redirect về trang đăng nhập
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  return NextResponse.next(); // Cho phép tiếp tục
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/protected/:path*'],
};
```

---

## 9. API Route Handlers — Xử Lý API

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET /api/users
export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const page = searchParams.get('page') ?? '1';

  const users = await db.users.findMany({ skip: (parseInt(page) - 1) * 10 });
  return NextResponse.json(users);
}

// POST /api/users
export async function POST(request: NextRequest) {
  const body = await request.json();
  const user = await db.users.create(body);
  return NextResponse.json(user, { status: 201 });
}
```

---

## 10. Câu Hỏi Phỏng Vấn Quan Trọng

### Q: Server Component có thể dùng useState không?

**A:** Không. Server Components chạy trên server và không có lifecycle. `useState`, `useEffect`, event handlers — tất cả đều không khả dụng. Khi cần tương tác, tách phần đó sang Client Component với `'use client'`.

### Q: Khi nào dùng Server Component, khi nào dùng Client Component?

**A:**

| Nhu Cầu | Dùng |
| ------- | ---- |
| Fetch data từ DB/API | Server Component |
| Dùng `useState`, `useReducer` | Client Component |
| Xử lý onClick, onChange | Client Component |
| Truy cập localStorage, cookies | Client Component |
| Render nặng, không tương tác | Server Component |
| Real-time updates, WebSocket | Client Component |

**Nguyên tắc:** Đẩy `'use client'` boundary (ranh giới) xuống sâu nhất có thể trong cây component.

### Q: App Router cache hoạt động như thế nào?

**A:** Next.js App Router có 4 tầng cache:
1. **Request Memoization** — Cache trong một request, tránh fetch trùng lặp
2. **Data Cache** — Cache kết quả fetch() giữa các request, persist qua deployments
3. **Full Route Cache** — Cache HTML + RSC payload của static routes
4. **Router Cache** — Cache phía client, tránh request khi navigate back/forward

### Q: ISR — Incremental Static Regeneration khác SSR như thế nào?

**A:**
- **SSR** (Server-Side Rendering — Render Phía Server): Mỗi request đều render lại → luôn mới nhất nhưng chậm hơn
- **ISR** (Incremental Static Regeneration — Tái Tạo Tĩnh Tăng Dần): Render lúc build (nhanh), tự động tái tạo sau `revalidate` giây — cân bằng giữa hiệu năng và độ tươi của data

---

## ✅ Checklist Thực Hành

- [ ] Tạo Next.js project với App Router: `npx create-next-app@latest --app`
- [ ] Tạo nested layout với `layout.tsx`
- [ ] Viết Server Component async fetch data
- [ ] Thêm Client Component với `'use client'`
- [ ] Implement dynamic route `[slug]`
- [ ] Tạo Server Action để xử lý form
- [ ] Setup middleware bảo vệ route
- [ ] Dùng Suspense để streaming

---

**Tài Liệu Tham Khảo:**
- [Next.js App Router Docs](https://nextjs.org/docs/app)
- [React Server Components RFC](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)
- [Next.js Caching Docs](https://nextjs.org/docs/app/building-your-application/caching)
