# React Router v7 — Tính Năng Mới Và Framework Mode

> **React Router v7** (phát hành tháng 11/2024) là bước chuyển đổi lớn nhất trong lịch sử thư viện — từ một **routing library** (thư viện điều hướng) trở thành một **full-stack framework** (framework toàn ngăn xếp) hoàn chỉnh. Phiên bản này hợp nhất React Router v6 và Remix v2, mang đến **framework mode** với SSR, **file-based routing**, và nhiều tính năng mới.

---

## 📌 Mục Lục

1. [Tổng Quan React Router v7](#1-tổng-quan)
2. [Hai Chế Độ Sử Dụng — Library vs Framework](#2-library-vs-framework-mode)
3. [Upgrade Từ v6 Lên v7](#3-upgrade-từ-v6)
4. [Framework Mode — Chế Độ Framework](#4-framework-mode)
5. [File-based Routing — Điều Hướng Dựa Trên File](#5-file-based-routing)
6. [Server-Side Rendering — SSR — Kết Xuất Phía Server](#6-ssr)
7. [Type-Safe Routes Với TypeScript](#7-type-safe-routes)
8. [Các Cải Tiến Khác](#8-cải-tiến-khác)
9. [Khi Nào Nâng Cấp Lên v7?](#9-khi-nào-nâng-cấp)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan React Router v7

### Lịch Sử Hội Tụ

```
React Router v6 (2021)          Remix v2 (2023)
      ↓                               ↓
      ↓ ←─────── Hội tụ ────────────→ ↓
      ↓
React Router v7 (2024)
  ├── Library mode    (tương đương React Router v6 — không thay đổi nhiều)
  └── Framework mode  (tương đương Remix v2 — full-stack, SSR, file-based routing)
```

### Tóm Tắt Những Gì Mới

| Tính Năng | v6 | v7 |
| --------- | -- | -- |
| **Library mode** | ✅ | ✅ (không đổi nhiều) |
| **Framework mode** | ❌ | ✅ Mới hoàn toàn |
| **File-based routing** | ❌ | ✅ (framework mode) |
| **SSR tích hợp sẵn** | ❌ | ✅ (framework mode) |
| **TypeScript type-safe routes** | ❌ Thủ công | ✅ Tự động sinh type |
| **Vite plugin** | ❌ | ✅ `@react-router/dev` |
| **React 19 integration** | Hạn chế | ✅ Tích hợp sâu |
| **Streaming SSR** | ❌ | ✅ `defer` + Suspense |

---

## 2. Library Mode vs Framework Mode

### Library Mode — Chế Độ Thư Viện (Backwards Compatible — Tương Thích Ngược)

```jsx
// Library mode — giống hệt React Router v6
// Dùng khi: migrate dần từ v6, không muốn thay đổi cấu trúc dự án

import { BrowserRouter, Routes, Route } from "react-router-dom";
// HOẶC
import { createBrowserRouter, RouterProvider } from "react-router-dom";

// Không thay đổi gì so với v6 — đây là backward-compatible mode
```

### Framework Mode — Chế Độ Framework (Mới Hoàn Toàn)

```jsx
// Framework mode — dùng Vite plugin, file-based routing, SSR
// Dùng khi: dự án mới, muốn full-stack capability, tương tự Next.js

// app/root.tsx — entry point của ứng dụng
import { Links, Meta, Outlet, Scripts, ScrollRestoration } from "react-router";

export function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="vi">
      <head>
        <Meta />
        <Links />
      </head>
      <body>
        {children}
        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  );
}

export default function Root() {
  return <Outlet />;
}
```

---

## 3. Upgrade Từ v6 Lên v7

### Bước 1: Cập Nhật Package

```bash
npm install react-router-dom@7
# hoặc
npm install react-router@7
```

### Bước 2: Cập Nhật Imports (Thay Đổi Nhỏ)

```jsx
// React Router v6
import { /* ... */ } from "react-router-dom";

// React Router v7 — package được đổi tên
import { /* ... */ } from "react-router"; // Thay "react-router-dom" bằng "react-router"
// Chú ý: "react-router-dom" vẫn hoạt động nhưng "react-router" là tên mới
```

### Bước 3: Xử Lý Breaking Changes (Thay Đổi Phá Vỡ Tương Thích)

```jsx
// ❌ v6: <Route> component (JSX route config)
import { Routes, Route } from "react-router-dom";

function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
    </Routes>
  );
}

// ✅ v7: Khuyến nghị dùng createBrowserRouter (đã có từ v6.4)
import { createBrowserRouter, RouterProvider } from "react-router";

const router = createBrowserRouter([
  { path: "/", element: <Home /> },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

```jsx
// ❌ v6: future flags chưa bắt buộc
const router = createBrowserRouter(routes, {
  future: {
    v7_startTransition: true,
    v7_relativeSplatPath: true,
    v7_fetcherPersist: true,
    v7_normalizeFormMethod: true,
    v7_partialHydration: true,
  },
});

// ✅ v7: Tất cả future flags từ v6 là mặc định trong v7 — không cần cấu hình thêm
const router = createBrowserRouter(routes);
```

---

## 4. Framework Mode — Chế Độ Framework

### Thiết Lập Dự Án Mới

```bash
# Tạo dự án React Router v7 framework mode
npx create-react-router@latest my-app
cd my-app
npm run dev
```

### Cấu Trúc Dự Án

```
my-app/
├── app/
│   ├── root.tsx              ← Layout toàn ứng dụng
│   ├── routes.ts             ← Khai báo routes (hoặc dùng file-based)
│   └── routes/
│       ├── home.tsx          ← Route: /
│       ├── about.tsx         ← Route: /about
│       └── blog/
│           ├── index.tsx     ← Route: /blog
│           └── $slug.tsx     ← Route: /blog/:slug (dynamic)
├── public/
├── react-router.config.ts    ← Cấu hình React Router
├── vite.config.ts
└── package.json
```

### react-router.config.ts — Cấu Hình

```typescript
import type { Config } from "@react-router/dev/config";

export default {
  // SSR (Server-Side Rendering — Kết Xuất Phía Server): true/false
  ssr: true,

  // Cấu hình app directory
  appDirectory: "app",

  // Tự động tạo type cho routes
  future: {
    unstable_optimizeDeps: true,
  },
} satisfies Config;
```

---

## 5. File-based Routing — Điều Hướng Dựa Trên File

Framework mode hỗ trợ **File-based Routing** — cấu trúc URL được suy ra từ cấu trúc thư mục file.

### Quy Tắc Đặt Tên File

```
app/routes/
├── home.tsx                 → /
├── about.tsx                → /about
├── contact.tsx              → /contact
│
├── blog.tsx                 → /blog (layout, có Outlet)
├── blog._index.tsx          → /blog (index page)
├── blog.$slug.tsx           → /blog/:slug
├── blog.tag.$tag.tsx        → /blog/tag/:tag
│
├── dashboard.tsx            → /dashboard (layout)
├── dashboard._index.tsx     → /dashboard (index)
├── dashboard.settings.tsx   → /dashboard/settings
│
├── ($lang).home.tsx         → /:lang?/ (optional param — tham số tùy chọn)
│
└── $.tsx                    → /* (splat/catch-all route)
```

### Ký Hiệu Đặc Biệt

| Ký Hiệu | Ý Nghĩa | Ví Dụ |
| ------- | ------- | ----- |
| `$param` | Dynamic segment — đoạn động | `blog.$slug.tsx` → `/blog/:slug` |
| `_index` | Index route — route mặc định | `blog._index.tsx` → `/blog` |
| `_layout` | Prefix không xuất hiện trong URL | `_auth.login.tsx` → `/login` |
| `($param)` | Optional param — tham số tùy chọn | `($lang).home.tsx` → `/:lang?/` |
| `.` | Path separator thay cho `/` | `blog.settings.tsx` → `/blog/settings` |
| `$` (file) | Splat/catch-all | `$.tsx` → `/*` |

### Ví Dụ Route File

```tsx
// app/routes/blog.$slug.tsx
import type { Route } from "./+types/blog.$slug"; // Type tự động sinh

// loader chạy trên server (hoặc client với SSR=false)
export async function loader({ params }: Route.LoaderArgs) {
  const post = await fetchBlogPost(params.slug);

  if (!post) {
    throw new Response("Bài viết không tồn tại", { status: 404 });
  }

  return { post };
}

// meta function — tạo <title> và <meta> cho SEO
export function meta({ data }: Route.MetaArgs) {
  return [
    { title: data?.post.title ?? "Bài Viết" },
    { name: "description", content: data?.post.excerpt },
    { property: "og:title", content: data?.post.title },
  ];
}

// Component chính
export default function BlogPost({ loaderData }: Route.ComponentProps) {
  const { post } = loaderData;

  return (
    <article>
      <h1>{post.title}</h1>
      <time dateTime={post.publishedAt}>
        {new Date(post.publishedAt).toLocaleDateString("vi-VN")}
      </time>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

---

## 6. SSR — Server-Side Rendering (Kết Xuất Phía Server)

React Router v7 framework mode hỗ trợ SSR tích hợp sẵn.

### Loader Chạy Trên Server

```tsx
// app/routes/products.tsx

// Loader chạy trên SERVER trước khi trả HTML về client
export async function loader({ request }: Route.LoaderArgs) {
  const url = new URL(request.url);
  const category = url.searchParams.get("category") ?? "all";

  // Có thể gọi database trực tiếp (không qua API)
  const products = await db.products.findMany({
    where: category !== "all" ? { category } : undefined,
    orderBy: { createdAt: "desc" },
    take: 20,
  });

  return { products, category };
}

export default function ProductsPage({ loaderData }: Route.ComponentProps) {
  const { products, category } = loaderData;

  return (
    <div>
      <h1>Sản Phẩm ({category})</h1>
      <ProductGrid products={products} />
    </div>
  );
}
```

### Streaming SSR Với defer

```tsx
import { defer, Await } from "react-router";
import { Suspense } from "react";

export async function loader() {
  // criticalData: tải ngay, chặn đến khi có
  const criticalData = await fetchCriticalData();

  // deferredData: stream sau — không chặn initial render
  const deferredData = fetchDeferredData(); // Không await — trả về Promise

  return defer({
    criticalData,
    deferredData, // Promise — được stream sau
  });
}

export default function Page({ loaderData }: Route.ComponentProps) {
  return (
    <div>
      {/* Dữ liệu quan trọng hiển thị ngay */}
      <CriticalContent data={loaderData.criticalData} />

      {/* Dữ liệu thứ yếu hiển thị khi sẵn sàng */}
      <Suspense fallback={<LoadingSpinner />}>
        <Await resolve={loaderData.deferredData}>
          {(data) => <DeferredContent data={data} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

### Action Trên Server

```tsx
import { redirect } from "react-router";

export async function action({ request, params }: Route.ActionArgs) {
  const formData = await request.formData();
  const intent = formData.get("intent");

  if (intent === "delete") {
    await db.product.delete({ where: { id: params.id } });
    return redirect("/products");
  }

  if (intent === "update") {
    const name = formData.get("name") as string;
    const price = Number(formData.get("price"));

    // Validation (kiểm tra hợp lệ) phía server
    if (!name || name.length < 3) {
      return { errors: { name: "Tên phải có ít nhất 3 ký tự" } };
    }

    await db.product.update({
      where: { id: params.id },
      data: { name, price },
    });

    return redirect(`/products/${params.id}`);
  }

  return { error: "Hành động không hợp lệ" };
}
```

---

## 7. Type-Safe Routes Với TypeScript

React Router v7 tự động sinh types từ route config — **generated types** (type được sinh tự động):

```typescript
// app/routes/blog.$slug.tsx

// Type được tự động sinh bởi React Router v7 (không cần viết thủ công)
import type { Route } from "./+types/blog.$slug";

export async function loader({ params }: Route.LoaderArgs) {
  // params.slug — TypeScript biết kiểu là string (không cần assertion)
  const post = await fetchPost(params.slug);
  return { post };
}

export function meta({ data }: Route.MetaArgs) {
  // data.post — TypeScript biết kiểu dữ liệu từ loader return type
  return [{ title: data?.post.title }];
}

export default function BlogPost({ loaderData }: Route.ComponentProps) {
  // loaderData.post — có đầy đủ type hints
  return <h1>{loaderData.post.title}</h1>;
}
```

```typescript
// Dùng Link với type-safe href
import { href } from "react-router";

// ✅ TypeScript báo lỗi nếu route không tồn tại hoặc thiếu params
<Link to={href("/blog/:slug", { slug: "ten-bai-viet" })}>
  Đọc bài viết
</Link>

// ❌ TypeScript báo lỗi — route /blog/:slug cần param slug
<Link to={href("/blog/:slug", {})}>Lỗi TypeScript</Link>
```

---

## 8. Cải Tiến Khác

### Single Fetch — Một Request Duy Nhất

React Router v7 sử dụng **Single Fetch** (Lấy Dữ Liệu Một Lần) — tất cả loaders cho một navigation được gộp thành một HTTP request duy nhất thay vì nhiều request song song:

```
v6 (nhiều request):
  GET /api/dashboard          ← loader DashboardLayout
  GET /api/dashboard/stats    ← loader StatsWidget

v7 (một request):
  GET /_data?routes=dashboard,dashboard.stats   ← Single Fetch
```

### clientLoader và clientAction

```tsx
// clientLoader: Loader chạy hoàn toàn trên client (browser)
// Phù hợp cho dữ liệu không cần SSR, lấy từ localStorage, IndexedDB
export async function clientLoader({ serverLoader }: Route.ClientLoaderArgs) {
  // Kiểm tra cache trước
  const cached = cache.get("dashboard-data");
  if (cached) return cached;

  // Gọi server loader nếu không có cache
  const data = await serverLoader();
  cache.set("dashboard-data", data);
  return data;
}
// Báo cho React Router biết clientLoader có thể chạy trước hydration
clientLoader.hydrate = true as const;

// clientAction: Action chạy trên client
export async function clientAction({ serverAction }: Route.ClientActionArgs) {
  // Optimistic update trước khi gọi server
  optimisticUpdate();
  return serverAction();
}
```

### Middleware (Phần Mềm Trung Gian) — Unstable

```typescript
// react-router.config.ts — unstable middleware
export default {
  middleware: [
    // Logging middleware
    async ({ request, next }) => {
      console.log(`[${request.method}] ${request.url}`);
      const start = Date.now();
      const response = await next();
      console.log(`Completed in ${Date.now() - start}ms`);
      return response;
    },
    // Auth middleware
    async ({ request, next }) => {
      const token = request.headers.get("Authorization");
      if (!token && request.url.includes("/api/")) {
        return new Response("Unauthorized", { status: 401 });
      }
      return next();
    },
  ],
} satisfies Config;
```

---

## 9. Khi Nào Nâng Cấp Lên v7?

### Library Mode — Nâng Cấp Ngay Nếu

- ✅ Đang dùng React Router v6.4+ với `createBrowserRouter`
- ✅ Đã enable tất cả future flags của v6
- ✅ Không dùng deprecated APIs (`<Switch>`, `useHistory`, v.v.)

**Effort:** Thấp — chủ yếu đổi tên import từ `react-router-dom` → `react-router`

### Framework Mode — Xem Xét Khi

- ✅ Dự án mới cần SSR hoặc static generation
- ✅ Muốn full-stack TypeScript với type-safe routes
- ✅ Đang dùng Remix v2 (migration path rất rõ ràng)
- ⚠️ Dự án hiện tại muốn migrate — cần effort đáng kể

### Không Nên Nâng Cấp Khi

- ❌ Dự án ổn định, đang production, không có bug routing
- ❌ Team chưa quen với file-based routing
- ❌ Đang dùng `<Routes>/<Route>` JSX API nhiều (cần refactor lớn)

---

## 10. Câu Hỏi Phỏng Vấn

### Câu 1: React Router v7 khác v6 như thế nào?

**Trả lời:**
React Router v7 có hai chế độ:
- **Library mode**: Backward-compatible với v6, chủ yếu đổi tên package và bật các future flags. Ít thay đổi code.
- **Framework mode**: Hoàn toàn mới — file-based routing, SSR tích hợp, TypeScript type-safe, Vite plugin, loaders/actions chạy trên server. Tương đương Remix v2 (đây là kết quả hợp nhất Remix vào React Router).

### Câu 2: File-based routing (điều hướng theo file) có ưu nhược điểm gì?

**Trả lời:**

**Ưu điểm:**
- URL structure rõ ràng từ cấu trúc thư mục — không cần đọc code
- Convention over configuration — ít quyết định phải đưa ra
- Dễ tìm file route tương ứng với URL

**Nhược điểm:**
- Khó tùy chỉnh routing logic phức tạp
- Tên file có thể dài và khó đọc (`blog._index.$category.tsx`)
- Không quen với developer từ background React Router truyền thống

### Câu 3: Single Fetch trong v7 giải quyết vấn đề gì?

**Trả lời:**
Trong v6, mỗi loader gửi một HTTP request riêng biệt. Khi một trang có nhiều route lồng nhau (mỗi route có loader), có thể có 4-5 request song song đến server. Single Fetch gộp tất cả thành một request duy nhất, giảm overhead của HTTP connections và cải thiện performance khi có nhiều nested loaders. Server trả về streaming response với tất cả data được encode cùng nhau.

---

## 🔗 Điều Hướng

- **Trước đó:** [3-protected-routes.md](./3-protected-routes.md) — Protected Routes và Auth Guards
- **Tiếp theo:** [5-tanstack-router.md](./5-tanstack-router.md) — TanStack Router
- **Quay lại:** [README.md](./README.md) — Tổng quan Routing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
