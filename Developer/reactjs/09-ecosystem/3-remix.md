# Remix — Web Standards Framework

> Remix là meta-framework React tập trung vào **Web Platform Standards** (Tiêu Chuẩn Nền Tảng Web) — tận dụng tối đa Web APIs gốc: `fetch`, `FormData`, HTTP cache headers, thay vì tạo ra abstractions riêng

---

## 1. Triết Lý Remix

### "Embrace the Web" — Ôm Lấy Web

Remix không cố gắng thay thế trình duyệt hay HTTP — nó xây dựng **trên** chúng:

| Web Standard | Remix Approach |
| ------------ | -------------- |
| HTML `<form>` | Hoạt động không cần JavaScript! |
| HTTP `POST`, `GET` | `action`, `loader` map trực tiếp |
| `fetch` API | Dùng nguyên bản trong loaders |
| Browser cache headers | `Cache-Control` header trực tiếp |
| URL params | `params` object gốc |
| `FormData` | Xử lý như thường trong action |

### Kiến Trúc "Full-Stack" Thực Sự

```
Browser Request
    ↓
Remix Loader (server)   ← Fetch data, trả về JSON
    ↓
React Component (render)
    ↓
HTML gửi về Browser
    ↓
(Hydrate với JavaScript)
    ↓
User tương tác → Form Submit
    ↓
Remix Action (server)   ← Xử lý mutation, không cần API riêng
    ↓
Redirect hoặc Response
```

---

## 2. Routing — Điều Hướng

Remix dùng **file-based routing** (Điều Hướng Dựa Trên File) trong thư mục `app/routes/`.

### Cấu Trúc Routes

```
app/
├── root.tsx              ← Root route (giống layout gốc)
├── routes/
│   ├── _index.tsx        ← Trang "/" (index route)
│   ├── about.tsx         ← Trang "/about"
│   │
│   ├── blog._index.tsx   ← "/blog" (index của blog)
│   ├── blog.$slug.tsx    ← "/blog/:slug" (dynamic)
│   │
│   ├── dashboard.tsx     ← Layout route cho "/dashboard/*"
│   ├── dashboard._index.tsx ← "/dashboard"
│   ├── dashboard.settings.tsx ← "/dashboard/settings"
│   │
│   └── ($lang).tsx       ← Optional segment
```

**Quy ước đặt tên file:**
- `.` → phân cấp URL (`blog.post` = `/blog/post`)
- `$param` → dynamic segment (`blog.$slug` = `/blog/:slug`)
- `_` prefix → layout route không thêm vào URL (`_app.tsx`)
- `($param)` → optional segment

---

## 3. Loader — Lấy Dữ Liệu

`loader` chạy trên server khi route được render.

```tsx
// app/routes/blog.$slug.tsx
import { json } from '@remix-run/node';
import { useLoaderData, type MetaFunction } from '@remix-run/react';
import type { LoaderFunctionArgs } from '@remix-run/node';

// Loader chạy trên server — fetch data
export async function loader({ params, request }: LoaderFunctionArgs) {
  const { slug } = params;
  const post = await db.post.findUnique({ where: { slug } });

  if (!post) {
    // throw Response — Remix handle tự động
    throw new Response('Không tìm thấy bài viết', { status: 404 });
  }

  // json() helper tạo Response với Content-Type: application/json
  return json(
    { post },
    {
      headers: {
        // HTTP Cache — trình duyệt cache 5 phút, CDN cache 1 giờ
        'Cache-Control': 'public, max-age=300, s-maxage=3600',
      },
    }
  );
}

// Meta tags cho SEO
export const meta: MetaFunction<typeof loader> = ({ data }) => {
  return [
    { title: data?.post.title ?? 'Bài viết' },
    { name: 'description', content: data?.post.excerpt },
  ];
};

export default function BlogPost() {
  const { post } = useLoaderData<typeof loader>(); // Type-safe!

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
```

### clientLoader — Loader Phía Client

```tsx
// Chạy trên client sau khi hydrate — dùng cho SPA navigation
export async function clientLoader({ serverLoader, params }) {
  // Dùng cache local để tránh request lại
  const cached = cache.get(params.slug);
  if (cached) return cached;

  const data = await serverLoader(); // Gọi server loader nếu cần
  cache.set(params.slug, data);
  return data;
}
```

---

## 4. Action — Xử Lý Mutation

`action` xử lý form submission — POST, PUT, DELETE.

```tsx
// app/routes/contacts.new.tsx
import { redirect, json } from '@remix-run/node';
import { Form, useActionData } from '@remix-run/react';
import type { ActionFunctionArgs } from '@remix-run/node';

export async function action({ request }: ActionFunctionArgs) {
  // Lấy FormData trực tiếp — không cần JSON.parse
  const formData = await request.formData();
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // Validation
  const errors: Record<string, string> = {};
  if (!name) errors.name = 'Tên là bắt buộc';
  if (!email.includes('@')) errors.email = 'Email không hợp lệ';

  if (Object.keys(errors).length > 0) {
    return json({ errors }, { status: 400 });
  }

  await db.contact.create({ data: { name, email } });

  // Redirect sau khi thành công — PRG Pattern (Post-Redirect-Get)
  return redirect('/contacts');
}

export default function NewContactPage() {
  const actionData = useActionData<typeof action>();

  return (
    // Form của Remix tự động submit đến action của route này
    // Hoạt động ngay cả khi JavaScript chưa load! (Progressive Enhancement)
    <Form method="post">
      <div>
        <label>Tên</label>
        <input name="name" />
        {actionData?.errors?.name && (
          <span className="error">{actionData.errors.name}</span>
        )}
      </div>

      <div>
        <label>Email</label>
        <input name="email" type="email" />
        {actionData?.errors?.email && (
          <span className="error">{actionData.errors.email}</span>
        )}
      </div>

      <button type="submit">Tạo Liên Hệ</button>
    </Form>
  );
}
```

---

## 5. Nested Routes & `<Outlet />`

Remix khai thác nested routes (route lồng nhau) để chia sẻ layout và data fetching.

```tsx
// app/routes/dashboard.tsx — Layout route
import { Outlet } from '@remix-run/react';

export async function loader() {
  // Data này có sẵn cho TẤT CẢ child routes
  const user = await getCurrentUser();
  return json({ user });
}

export default function DashboardLayout() {
  const { user } = useLoaderData<typeof loader>();

  return (
    <div className="dashboard">
      <nav>
        <span>Xin chào, {user.name}</span>
        <a href="/dashboard">Tổng quan</a>
        <a href="/dashboard/settings">Cài đặt</a>
      </nav>

      <main>
        <Outlet /> {/* Render child route tại đây */}
      </main>
    </div>
  );
}
```

```tsx
// app/routes/dashboard._index.tsx — Child route
import { useRouteLoaderData } from '@remix-run/react';

export default function DashboardIndex() {
  // Truy cập data từ parent loader
  const { user } = useRouteLoaderData('routes/dashboard');

  return <h1>Dashboard của {user.name}</h1>;
}
```

### Parallel Data Loading — Tải Dữ Liệu Song Song

Điểm mạnh của Remix: **tất cả loaders trong route tree chạy song song** — không có waterfall.

```
/dashboard/analytics → Chạy đồng thời:
  ├── dashboard.tsx loader (user info)
  └── dashboard.analytics.tsx loader (analytics data)
```

---

## 6. Error Handling — Xử Lý Lỗi

```tsx
// app/routes/blog.$slug.tsx
import { useRouteError, isRouteErrorResponse } from '@remix-run/react';

// ErrorBoundary tích hợp cho từng route
export function ErrorBoundary() {
  const error = useRouteError();

  // Lỗi từ throw new Response(...)
  if (isRouteErrorResponse(error)) {
    return (
      <div>
        <h1>{error.status} — {error.statusText}</h1>
        <p>{error.data}</p>
      </div>
    );
  }

  // Lỗi JavaScript bất ngờ
  return (
    <div>
      <h1>Đã xảy ra lỗi</h1>
      <pre>{error instanceof Error ? error.message : 'Lỗi không xác định'}</pre>
    </div>
  );
}
```

---

## 7. Optimistic UI — Giao Diện Lạc Quan

```tsx
'use client';
import { useFetcher } from '@remix-run/react';

export function LikeButton({ post }) {
  const fetcher = useFetcher();

  // Xác định trạng thái hiện tại dựa trên optimistic state
  const isLiked = fetcher.formData
    ? fetcher.formData.get('liked') === 'true' // Optimistic
    : post.isLiked;                              // Actual

  return (
    <fetcher.Form method="post" action="/api/like">
      <input type="hidden" name="postId" value={post.id} />
      <input type="hidden" name="liked" value={String(!isLiked)} />
      <button type="submit">
        {isLiked ? '❤️ Đã thích' : '🤍 Thích'}
        ({post.likeCount + (fetcher.formData ? (isLiked ? 1 : -1) : 0)})
      </button>
    </fetcher.Form>
  );
}
```

---

## 8. Sessions & Authentication

```tsx
// app/sessions.server.ts
import { createCookieSessionStorage } from '@remix-run/node';

export const sessionStorage = createCookieSessionStorage({
  cookie: {
    name: '__session',
    httpOnly: true,
    maxAge: 60 * 60 * 24 * 7, // 7 ngày
    path: '/',
    sameSite: 'lax',
    secrets: [process.env.SESSION_SECRET!],
    secure: process.env.NODE_ENV === 'production',
  },
});

export async function getSession(request: Request) {
  return sessionStorage.getSession(request.headers.get('Cookie'));
}
```

```tsx
// app/routes/login.tsx
export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;

  const user = await verifyCredentials(email, password);
  if (!user) {
    return json({ error: 'Email hoặc mật khẩu không đúng' }, { status: 400 });
  }

  const session = await getSession(request);
  session.set('userId', user.id);

  return redirect('/dashboard', {
    headers: {
      'Set-Cookie': await sessionStorage.commitSession(session),
    },
  });
}
```

---

## 9. Remix vs Next.js — So Sánh Tư Duy

| Khía Cạnh | Remix | Next.js App Router |
| --------- | ----- | ------------------ |
| **Triết lý** | Web standards first | React-first |
| **Data fetching** | `loader` (server, co-located) | `async` Server Components |
| **Mutations** | `action` (form-native) | Server Actions |
| **Caching** | HTTP headers (trình duyệt standard) | Next.js cache system (custom) |
| **Error handling** | `ErrorBoundary` per route | `error.tsx` per route |
| **Progressive Enhancement** | Tích hợp sẵn (form hoạt động không JS) | Cần cẩn thận hơn |
| **Nested layouts** | Tự nhiên, data co-located | `layout.tsx` |
| **Streaming** | Hỗ trợ với `defer()` | Hỗ trợ với Suspense |
| **Bundle size** | Nhỏ hơn | Lớn hơn |
| **Ecosystem** | Nhỏ hơn | Rất lớn (Vercel, ecosystem rộng) |
| **Deployment** | Agnostic (Cloudflare, Fly, Node, ...) | Tối ưu nhất trên Vercel |

**Chọn Remix khi:**
- Cần Progressive Enhancement thực sự (ứng dụng hoạt động không JS)
- Muốn tận dụng HTTP caching tối đa
- Team quen với Web Standards hơn React-specific abstractions
- Deploy lên Edge runtime (Cloudflare Workers)

**Chọn Next.js khi:**
- Cần ecosystem lớn hơn (plugin, examples, community)
- Deploy chủ yếu lên Vercel
- Cần React Server Components với mọi tính năng
- Dự án enterprise cần nhiều tài liệu và hỗ trợ

---

## 10. Câu Hỏi Phỏng Vấn

### Q: Remix "Progressive Enhancement" nghĩa là gì?

**A:** Ứng dụng Remix hoạt động ngay cả **khi JavaScript bị tắt** hoặc chưa load. `<Form>` component của Remix render thành HTML `<form>` thông thường — trình duyệt gửi request, server xử lý, trả về HTML. Khi JavaScript load xong, Remix "nâng cấp" thành SPA experience (không reload trang toàn bộ). Đây là triết lý gốc của web.

### Q: Loaders và Actions có gì khác API Routes trong Next.js?

**A:** Loaders/Actions được **co-located** (đặt cùng file) với component — không cần tạo file API riêng. Chúng cũng được tổ chức theo route tree, nên data loading tự nhiên map với URL structure. Next.js API routes là endpoints riêng biệt, tách khỏi component.

### Q: Tại sao Remix dùng HTTP cache thay vì custom cache?

**A:** HTTP cache (CDN, browser cache, `Cache-Control` headers) đã là hạ tầng chuẩn, battle-tested, và hoạt động với mọi nơi. Remix tin rằng không cần custom layer — chỉ cần đặt `Cache-Control` header đúng là đủ. Ngược lại, Next.js xây dựng hệ thống cache riêng phức tạp hơn nhưng cũng linh hoạt hơn trong một số trường hợp.

---

## ✅ Checklist

- [ ] Hiểu file-based routing convention của Remix
- [ ] Viết `loader` để fetch data server-side
- [ ] Viết `action` để xử lý form submission
- [ ] Implement nested routes với `<Outlet />`
- [ ] Setup session-based authentication
- [ ] Xử lý lỗi với `ErrorBoundary` per route
- [ ] Implement optimistic UI với `useFetcher`

---

**Tài Liệu Tham Khảo:**
- [Remix Docs](https://remix.run/docs)
- [Remix Tutorials](https://remix.run/docs/en/main/start/tutorial)
- [Web Platform APIs](https://developer.mozilla.org/en-US/docs/Web/API)
