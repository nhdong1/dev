# TanStack Router — Type-Safe Routing Hiện Đại

> **TanStack Router** là thư viện routing 100% **type-safe** (an toàn kiểu dữ liệu) cho React, được xây dựng để giải quyết vấn đề mà React Router không giải quyết được: TypeScript đầy đủ cho routes, search params, và state. TanStack Router hỗ trợ cả **file-based** và **code-based** routing, tích hợp data fetching mạnh mẽ, và là lựa chọn routing ưa thích trong hệ sinh thái TanStack.

---

## 📌 Mục Lục

1. [Tại Sao TanStack Router?](#1-tại-sao-tanstack-router)
2. [Cài Đặt Và Thiết Lập](#2-cài-đặt)
3. [File-Based Routing](#3-file-based-routing)
4. [Code-Based Routing — Khai Báo Bằng Code](#4-code-based-routing)
5. [Type-Safe Search Params](#5-type-safe-search-params)
6. [Type-Safe Navigation — Điều Hướng An Toàn Kiểu](#6-type-safe-navigation)
7. [Route Loaders Và Data Fetching](#7-loaders-và-data-fetching)
8. [TanStack Router DevTools](#8-devtools)
9. [So Sánh Với React Router](#9-so-sánh-với-react-router)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao TanStack Router?

### Vấn Đề Của React Router Với TypeScript

```typescript
// ❌ React Router v6/v7 — type-safety yếu
const { productId } = useParams(); // productId: string | undefined — không type-safe
const [params] = useSearchParams();
const page = params.get("page");   // page: string | null — không biết kiểu thực tế

// Không có autocomplete cho route paths
navigate("/prodcts/42"); // Typo — TypeScript không bắt lỗi này!
navigate(`/products/${id}`); // Không biết /products/:id có tồn tại không
```

```typescript
// ✅ TanStack Router — 100% type-safe
const { productId } = Route.useParams(); // productId: string — đã được type
const { page, sort } = Route.useSearch();
// page: number, sort: "asc" | "desc" — type được định nghĩa trước

// Autocomplete và type-check cho tất cả routes
router.navigate({ to: "/products/$productId", params: { productId: "42" } });
// TypeScript báo lỗi nếu route không tồn tại hoặc thiếu params
```

### So Sánh Điểm Mạnh

| Tính Năng | React Router v7 | TanStack Router |
| --------- | --------------- | --------------- |
| **Route path type-safe** | ✅ Có (v7 mới thêm) | ✅ Đầy đủ hơn |
| **Search params type-safe** | ❌ | ✅ |
| **Navigation type-safe** | ✅ Có (v7) | ✅ |
| **Route state type-safe** | ❌ | ✅ |
| **Built-in data caching** | ❌ | ✅ Có |
| **Devtools** | ❌ | ✅ Có |
| **Bundle size** | ~50 KB | ~35 KB |
| **Ecosystem** | Rất lớn, lâu đời | Đang tăng trưởng nhanh |
| **Framework mode** | ✅ (v7) | ✅ TanStack Start |

---

## 2. Cài Đặt Và Thiết Lập

```bash
npm install @tanstack/react-router

# DevTools — Công Cụ Phát Triển (chỉ development)
npm install @tanstack/router-devtools

# Vite plugin cho file-based routing (tùy chọn)
npm install --save-dev @tanstack/router-plugin
```

### Cấu Hình Vite Plugin

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { TanStackRouterVite } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [
    TanStackRouterVite({
      // Thư mục chứa route files
      routesDirectory: "./src/routes",
      // File output cho generated route tree
      generatedRouteTree: "./src/routeTree.gen.ts",
    }),
    react(),
  ],
});
```

---

## 3. File-Based Routing

### Cấu Trúc Thư Mục

```
src/routes/
├── __root.tsx                ← Root layout (bắt buộc)
├── index.tsx                 ← Route: /
├── about.tsx                 ← Route: /about
├── blog/
│   ├── index.tsx             ← Route: /blog
│   └── $slug.tsx             ← Route: /blog/$slug (dynamic)
├── dashboard/
│   ├── route.tsx             ← Layout cho /dashboard/*
│   ├── index.tsx             ← Route: /dashboard
│   ├── settings.tsx          ← Route: /dashboard/settings
│   └── users/
│       ├── index.tsx         ← Route: /dashboard/users
│       └── $userId.tsx       ← Route: /dashboard/users/$userId
└── _auth/
    ├── route.tsx             ← Pathless layout (không xuất hiện trong URL)
    ├── login.tsx             ← Route: /login
    └── register.tsx          ← Route: /register
```

### Ký Hiệu Tên File

| Ký Hiệu | Ý Nghĩa | URL |
| ------- | ------- | --- |
| `$param` | Dynamic segment — đoạn động | `$userId.tsx` → `/:userId` |
| `_prefix` | Pathless layout prefix | `_auth/login.tsx` → `/login` |
| `(param)` | Optional param — tham số tùy chọn | `(lang)/home.tsx` → `/:lang?/home` |
| `_` (file) | Pathless layout | `_layout.tsx` |
| `index.tsx` | Index route | `/folder/index.tsx` → `/folder` |

### Root Layout — Layout Gốc

```tsx
// src/routes/__root.tsx
import { createRootRoute, Link, Outlet } from "@tanstack/react-router";
import { TanStackRouterDevtools } from "@tanstack/router-devtools";

export const Route = createRootRoute({
  component: RootLayout,
  // notFoundComponent: hiển thị khi không có route khớp
  notFoundComponent: () => <div>404 — Không tìm thấy trang</div>,
});

function RootLayout() {
  return (
    <div>
      <header>
        <nav>
          <Link to="/">Trang Chủ</Link>
          <Link to="/about">Giới Thiệu</Link>
          <Link to="/dashboard">Dashboard</Link>
        </nav>
      </header>

      <main>
        <Outlet />
      </main>

      {/* DevTools chỉ hiển thị trong development */}
      {import.meta.env.DEV && <TanStackRouterDevtools />}
    </div>
  );
}
```

### Dynamic Route File

```tsx
// src/routes/blog/$slug.tsx
import { createFileRoute } from "@tanstack/react-router";

// Route được tự động nhận biết bởi file name
export const Route = createFileRoute("/blog/$slug")({
  // loader — chạy trước khi render, lấy dữ liệu
  loader: async ({ params }) => {
    const post = await fetchBlogPost(params.slug);
    if (!post) throw new Error("Bài viết không tồn tại");
    return { post };
  },

  component: BlogPostPage,
});

function BlogPostPage() {
  // Hoàn toàn type-safe — không cần assertion
  const { slug } = Route.useParams();           // slug: string
  const { post } = Route.useLoaderData();       // post: BlogPost (type từ loader return)

  return (
    <article>
      <h1>{post.title}</h1>
      <p>Slug: {slug}</p>
      <div>{post.content}</div>
    </article>
  );
}
```

---

## 4. Code-Based Routing — Khai Báo Bằng Code

Khi không muốn dùng file-based routing, có thể khai báo routes bằng code:

```typescript
// src/router.ts
import { createRouter, createRoute, createRootRoute } from "@tanstack/react-router";
import RootLayout from "./layouts/RootLayout";
import HomePage from "./pages/HomePage";
import BlogPage from "./pages/BlogPage";
import BlogPostPage from "./pages/BlogPostPage";

// 1. Tạo root route
const rootRoute = createRootRoute({
  component: RootLayout,
});

// 2. Khai báo các routes
const indexRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/",
  component: HomePage,
});

const blogRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/blog",
  component: BlogPage,
});

const blogPostRoute = createRoute({
  getParentRoute: () => blogRoute,
  path: "$slug",
  // Loader type-safe
  loader: async ({ params }) => {
    const post = await fetchBlogPost(params.slug);
    return { post };
  },
  component: BlogPostPage,
});

// 3. Tạo route tree
const routeTree = rootRoute.addChildren([
  indexRoute,
  blogRoute.addChildren([blogPostRoute]),
]);

// 4. Tạo router
export const router = createRouter({ routeTree });

// 5. Khai báo module augmentation (mở rộng module) cho TypeScript
declare module "@tanstack/react-router" {
  interface Register {
    router: typeof router;
  }
}
```

```tsx
// src/main.tsx
import { RouterProvider } from "@tanstack/react-router";
import { router } from "./router";

function App() {
  return <RouterProvider router={router} />;
}
```

---

## 5. Type-Safe Search Params

Đây là tính năng nổi bật nhất của TanStack Router — **search params** (tham số tìm kiếm) được định nghĩa schema và hoàn toàn type-safe:

```typescript
import { createFileRoute } from "@tanstack/react-router";
import { z } from "zod"; // Zod — thư viện validation/schema

// Định nghĩa schema cho search params
const productsSearchSchema = z.object({
  page: z.number().min(1).default(1),
  limit: z.number().min(1).max(100).default(20),
  category: z.string().optional(),
  sort: z.enum(["price-asc", "price-desc", "newest", "popular"]).default("newest"),
  search: z.string().optional(),
  inStock: z.boolean().default(false),
});

export const Route = createFileRoute("/products")({
  validateSearch: productsSearchSchema,

  loader: async ({ context }) => {
    // context.search đã được validate và type-safe
    return fetchProducts(context);
  },

  component: ProductsPage,
});

function ProductsPage() {
  const search = Route.useSearch();
  // search.page: number (không phải string!)
  // search.sort: "price-asc" | "price-desc" | "newest" | "popular"
  // search.inStock: boolean

  const navigate = useNavigate({ from: Route.fullPath });

  const handlePageChange = (newPage: number) => {
    // navigate cũng type-safe — biết search.page phải là number
    navigate({ search: prev => ({ ...prev, page: newPage }) });
  };

  const handleSortChange = (newSort: typeof search.sort) => {
    navigate({ search: prev => ({ ...prev, sort: newSort, page: 1 }) });
  };

  return (
    <div>
      <div className="filters">
        <select value={search.sort} onChange={e => handleSortChange(e.target.value as any)}>
          <option value="newest">Mới nhất</option>
          <option value="price-asc">Giá tăng dần</option>
          <option value="price-desc">Giá giảm dần</option>
          <option value="popular">Phổ biến nhất</option>
        </select>

        <label>
          <input
            type="checkbox"
            checked={search.inStock}
            onChange={e => navigate({ search: prev => ({ ...prev, inStock: e.target.checked }) })}
          />
          Còn hàng
        </label>
      </div>

      <ProductList searchParams={search} />
      <Pagination currentPage={search.page} onPageChange={handlePageChange} />
    </div>
  );
}
```

---

## 6. Type-Safe Navigation — Điều Hướng An Toàn Kiểu

```typescript
import { Link, useNavigate } from "@tanstack/react-router";

function ProductCard({ product }: { product: Product }) {
  const navigate = useNavigate();

  // ✅ Autocomplete cho tất cả routes
  const goToProduct = () => {
    navigate({
      to: "/blog/$slug",            // TypeScript gợi ý tất cả routes có sẵn
      params: { slug: product.slug }, // TypeScript biết route này cần param "slug"
      search: { page: 1 },           // TypeScript biết search params hợp lệ
    });
  };

  // ❌ TypeScript báo lỗi ngay lập tức
  navigate({ to: "/blg/$slug", params: { slug: "test" } }); // Typo trong route
  navigate({ to: "/blog/$slug", params: { id: "42" } });     // Sai tên param

  return (
    <div>
      {/* Link cũng type-safe */}
      <Link
        to="/products/$productId"
        params={{ productId: product.id.toString() }}
        search={{ sort: "newest" }}  // TypeScript validate search params
      >
        {product.name}
      </Link>

      {/* activeProps — props khi link đang active */}
      <Link
        to="/dashboard"
        activeProps={{ className: "nav-link active" }}
        inactiveProps={{ className: "nav-link" }}
      >
        Dashboard
      </Link>
    </div>
  );
}
```

### useNavigate Với From — Điều Hướng Tương Đối Type-Safe

```typescript
import { useNavigate } from "@tanstack/react-router";

function ProductsPage() {
  // from: chỉ định route hiện tại — TypeScript biết search params hợp lệ
  const navigate = useNavigate({ from: "/products" });

  // TypeScript chỉ cho phép search params được định nghĩa trong route /products
  navigate({ search: prev => ({ ...prev, page: prev.page + 1 }) });
}
```

---

## 7. Loaders Và Data Fetching

### Loader Cơ Bản

```typescript
export const Route = createFileRoute("/dashboard/users/$userId")({
  loader: async ({ params, context }) => {
    // Parallel fetching — tải song song
    const [user, permissions] = await Promise.all([
      fetchUser(params.userId),
      fetchUserPermissions(params.userId),
    ]);

    return { user, permissions };
  },

  // Xử lý lỗi — hiển thị khi loader throw error
  errorComponent: ({ error }) => (
    <div>Lỗi: {error.message}</div>
  ),

  // Hiển thị khi đang loading
  pendingComponent: () => <UserDetailSkeleton />,

  component: UserDetailPage,
});
```

### Kết Hợp Với TanStack Query

TanStack Router và TanStack Query — **React Query** (Truy Vấn React) — được thiết kế để dùng cùng nhau:

```typescript
import { queryOptions } from "@tanstack/react-query";

// Định nghĩa query options tái sử dụng
const userQueryOptions = (userId: string) =>
  queryOptions({
    queryKey: ["users", userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000, // 5 phút
  });

// Route context — inject QueryClient
export const Route = createFileRoute("/users/$userId")({
  loader: async ({ params, context }) => {
    // Prefetch (tải trước) vào TanStack Query cache
    // Khi component mount, useQuery dùng data đã có sẵn trong cache
    await context.queryClient.ensureQueryData(
      userQueryOptions(params.userId)
    );
  },

  component: UserPage,
});

function UserPage() {
  const { userId } = Route.useParams();

  // Data đã có sẵn từ loader — không cần loading state
  const { data: user } = useQuery(userQueryOptions(userId));

  return <UserProfile user={user!} />;
}
```

### Cấu Hình Router Context — Ngữ Cảnh Router

```typescript
// types/router.ts
import type { QueryClient } from "@tanstack/react-query";

export interface RouterContext {
  queryClient: QueryClient;
  auth: {
    isLoggedIn: boolean;
    user: User | null;
  };
}
```

```typescript
// src/main.tsx
import { createRouter, RouterProvider } from "@tanstack/react-router";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { routeTree } from "./routeTree.gen";

const queryClient = new QueryClient();

const router = createRouter({
  routeTree,
  context: {
    queryClient,
    auth: undefined!, // Được cung cấp khi render
  },
});

function App() {
  const auth = useAuth(); // Custom auth hook

  return (
    <QueryClientProvider client={queryClient}>
      <RouterProvider
        router={router}
        context={{ queryClient, auth }}
      />
    </QueryClientProvider>
  );
}
```

---

## 8. DevTools — Công Cụ Phát Triển

TanStack Router có DevTools tích hợp để debug routing:

```tsx
import { TanStackRouterDevtools } from "@tanstack/router-devtools";

function RootLayout() {
  return (
    <>
      <Outlet />

      {/* Chỉ hiển thị trong development mode */}
      {import.meta.env.DEV && (
        <TanStackRouterDevtools
          position="bottom-right"
          initialIsOpen={false}
        />
      )}
    </>
  );
}
```

DevTools hiển thị:
- Tất cả routes đã đăng ký và trạng thái
- Match hiện tại — route nào đang active
- Search params và giá trị đang được parse
- Loader state — pending/success/error
- Navigation history — lịch sử điều hướng

---

## 9. So Sánh Với React Router

### Khi Chọn TanStack Router

```
✅ Dự án mới với TypeScript là first-class priority
✅ Search params phức tạp cần validate và type-safe
✅ Team đã dùng TanStack Query — tích hợp rất mượt
✅ Cần DevTools mạnh để debug routing issues
✅ Muốn navigation type-safe hoàn toàn
```

### Khi Chọn React Router

```
✅ Dự án đang dùng React Router v6 — migrate không đáng công
✅ Cần ecosystem lớn, nhiều resources và ví dụ hơn
✅ Team quen với React Router, không muốn học mới
✅ Cần framework mode với SSR (React Router v7)
✅ Project đơn giản, không cần type-safety cao
```

### Bảng So Sánh Đầy Đủ

| Tiêu Chí | React Router v7 | TanStack Router |
| -------- | --------------- | --------------- |
| **TypeScript** | ✅ Tốt (type-safe routes) | ✅ Xuất sắc (type-safe mọi thứ) |
| **Search params** | ❌ Không type-safe | ✅ Schema validation |
| **Bundle size** | ~50 KB | ~35 KB |
| **SSR/Framework** | ✅ Framework mode tốt | ✅ TanStack Start (đang phát triển) |
| **Data loading** | ✅ loader/action | ✅ loader tích hợp TanStack Query |
| **DevTools** | ❌ Không có | ✅ Có |
| **Community** | ⭐⭐⭐⭐⭐ Rất lớn | ⭐⭐⭐ Đang tăng trưởng |
| **Learning curve** | ⭐⭐ Dễ | ⭐⭐⭐ Khó hơn |
| **Docs quality** | ✅ Tốt | ✅ Tốt |
| **Production ready** | ✅ Lâu đời | ✅ Ổn định (v1 stable) |

---

## 10. Câu Hỏi Phỏng Vấn

### Câu 1: TanStack Router giải quyết vấn đề gì mà React Router không làm được?

**Trả lời:**
Vấn đề chính là **type-safety cho search params**. React Router v6/v7 không có schema validation cho query string — `useSearchParams()` luôn trả về `string | null`, developer phải tự parse và validate. TanStack Router cho phép định nghĩa schema với Zod/Valibot, search params được auto-parse thành đúng kiểu (`number`, `boolean`, `enum`), TypeScript bắt lỗi khi truyền sai kiểu.

Ngoài ra: navigation 100% type-safe (React Router v7 mới thêm một phần), DevTools tích hợp sẵn, và tích hợp tự nhiên với TanStack Query.

### Câu 2: File-based routing trong TanStack Router hoạt động như thế nào?

**Trả lời:**
Vite plugin `@tanstack/router-plugin` quét thư mục `routes/` và tự động sinh file `routeTree.gen.ts` chứa toàn bộ route tree với đầy đủ TypeScript types. Tên file được ánh xạ thành routes theo quy ước: `$param.tsx` → dynamic segment, `_prefix` → pathless layout, `index.tsx` → index route. Khi file mới được thêm hoặc sửa đổi, Vite hot-reload (tải lại nóng) và regenerate route tree tự động.

### Câu 3: Tại sao TanStack Router và TanStack Query phối hợp tốt với nhau?

**Trả lời:**
TanStack Router hỗ trợ **router context** — inject dependencies vào tất cả loaders. Khi inject `QueryClient` vào router context:
- Loader có thể gọi `queryClient.ensureQueryData()` để prefetch data vào cache
- Component render ngay với data đã có — không cần loading state
- Khi user quay lại route, TanStack Query dùng cached data — không fetch lại nếu còn fresh
- Invalidation (`queryClient.invalidateQueries()`) sau action tự động trigger refetch đúng queries

Đây là pattern **"pre-loading"** (tải trước) hiệu quả nhất trong hệ sinh thái React.

### Câu 4: Khi nào nên cân nhắc chuyển từ React Router sang TanStack Router?

**Trả lời:**
Nên cân nhắc khi:
1. **TypeScript DX (Developer Experience)** là ưu tiên cao — team than phiền về thiếu type-safety cho search params
2. **Search params phức tạp** — nhiều bộ lọc, pagination, sorting cần validate
3. **Đã dùng TanStack Query** — tích hợp sẵn giúp giảm boilerplate đáng kể
4. **Dự án mới** — chi phí migration thấp

Không nên migrate dự án đang chạy tốt chỉ vì TanStack Router "hot" hơn — chi phí migration không tương xứng lợi ích trong nhiều trường hợp.

---

## 🔗 Điều Hướng

- **Trước đó:** [4-react-router-v7.md](./4-react-router-v7.md) — React Router v7
- **Quay lại:** [README.md](./README.md) — Tổng quan Routing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
- **Tiếp theo topic:** [05-data-fetching/](../05-data-fetching/) — Data Fetching & TanStack Query
