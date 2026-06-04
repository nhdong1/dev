# Nested Routes, Dynamic Params và Outlet

> **Nested Routes** (Route Lồng Nhau) cho phép tạo layout phân cấp, nơi component cha hiển thị phần giao diện chung (header, sidebar, navigation) và component con được render vào "lỗ hổng" — `<Outlet>` (Điểm Chèn). Kết hợp với **Dynamic Routes** (Route Động), đây là cơ sở xây dựng ứng dụng nhiều trang phức tạp.

---

## 📌 Mục Lục

1. [Nested Routes — Route Lồng Nhau](#1-nested-routes)
2. [Outlet — Điểm Chèn Component Con](#2-outlet)
3. [Index Routes — Route Mặc Định](#3-index-routes)
4. [Dynamic Routes và useParams](#4-dynamic-routes)
5. [Relative Paths — Đường Dẫn Tương Đối](#5-relative-paths)
6. [useMatch — Kiểm Tra Route Khớp](#6-usematch)
7. [Layout Patterns Thực Tế](#7-layout-patterns)
8. [Lazy Loading Routes — Tải Route Lười Biếng](#8-lazy-loading-routes)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Nested Routes — Route Lồng Nhau

### Khái Niệm Cơ Bản

Thay vì khai báo từng route độc lập, **Nested Routes** cho phép tổ chức route theo cấu bậc, phản ánh cấu trúc UI:

```
URL:        /dashboard/analytics
Layout:     RootLayout
              └── DashboardLayout (sidebar + header)
                    └── AnalyticsPage (nội dung chính)
```

### Khai Báo Nested Routes

```jsx
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,       // Layout toàn ứng dụng (nav bar, footer)
    children: [
      {
        index: true,               // Route mặc định khi URL là "/"
        element: <HomePage />,
      },
      {
        path: "dashboard",
        element: <DashboardLayout />,  // Layout riêng của dashboard (sidebar)
        children: [
          {
            index: true,               // Mặc định khi URL là "/dashboard"
            element: <DashboardOverview />,
          },
          {
            path: "analytics",         // /dashboard/analytics
            element: <AnalyticsPage />,
          },
          {
            path: "settings",          // /dashboard/settings
            element: <SettingsPage />,
          },
          {
            path: "users",             // /dashboard/users
            element: <UsersPage />,
            children: [
              {
                path: ":userId",       // /dashboard/users/42
                element: <UserDetailPage />,
              },
            ],
          },
        ],
      },
      {
        path: "about",
        element: <AboutPage />,
      },
      {
        path: "*",
        element: <NotFoundPage />,
      },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

---

## 2. Outlet — Điểm Chèn Component Con

`<Outlet>` là placeholder — vị trí giữ chỗ — nơi React Router sẽ render component con tương ứng với URL hiện tại.

### RootLayout — Layout Toàn Ứng Dụng

```jsx
import { Outlet, NavLink } from "react-router-dom";

function RootLayout() {
  return (
    <div className="app">
      <header>
        <nav>
          <NavLink to="/" end>Trang Chủ</NavLink>
          <NavLink to="/dashboard">Dashboard</NavLink>
          <NavLink to="/about">Giới Thiệu</NavLink>
        </nav>
      </header>

      <main>
        {/* Component con của route hiện tại được render tại đây */}
        <Outlet />
      </main>

      <footer>
        <p>© 2026 Công ty ABC</p>
      </footer>
    </div>
  );
}
```

### DashboardLayout — Layout Cho Dashboard

```jsx
import { Outlet, NavLink } from "react-router-dom";

function DashboardLayout() {
  return (
    <div className="dashboard">
      {/* Sidebar chỉ xuất hiện trên các trang dashboard */}
      <aside className="sidebar">
        <h2>Dashboard</h2>
        <nav>
          <NavLink to="/dashboard" end>Tổng Quan</NavLink>
          <NavLink to="/dashboard/analytics">Phân Tích</NavLink>
          <NavLink to="/dashboard/users">Người Dùng</NavLink>
          <NavLink to="/dashboard/settings">Cài Đặt</NavLink>
        </nav>
      </aside>

      {/* Nội dung chính thay đổi theo route con */}
      <div className="content">
        <Outlet />
      </div>
    </div>
  );
}
```

### Outlet Context — Truyền Dữ Liệu Từ Layout Xuống

```jsx
import { Outlet, useOutletContext } from "react-router-dom";

function DashboardLayout() {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchCurrentUser().then(setUser);
  }, []);

  return (
    <div className="dashboard">
      <aside>
        <UserAvatar user={user} />
        {/* Sidebar navigation */}
      </aside>
      <main>
        {/* Truyền context xuống tất cả route con */}
        <Outlet context={{ user, setUser }} />
      </main>
    </div>
  );
}

// Component con đọc context từ Outlet cha
function SettingsPage() {
  const { user, setUser } = useOutletContext();

  const handleUpdateProfile = async (data) => {
    const updated = await updateUserProfile(data);
    setUser(updated); // Cập nhật user trong DashboardLayout
  };

  return (
    <div>
      <h1>Cài Đặt Tài Khoản</h1>
      <p>Xin chào, {user?.name}</p>
      {/* Form chỉnh sửa */}
    </div>
  );
}
```

---

## 3. Index Routes — Route Mặc Định

**Index Route** (Route Chỉ Mục) là route hiển thị khi URL khớp với route cha nhưng không có route con nào được chọn thêm.

```jsx
const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    children: [
      // index: true — render khi URL là "/" (không có path)
      { index: true, element: <HomePage /> },
      {
        path: "products",
        element: <ProductsLayout />,
        children: [
          // index: true — render khi URL là "/products" (không phải "/products/123")
          { index: true, element: <ProductsListPage /> },
          { path: ":id", element: <ProductDetailPage /> },
          { path: "new", element: <CreateProductPage /> },
        ],
      },
    ],
  },
]);
```

```
URL: /products       → ProductsLayout + ProductsListPage (index)
URL: /products/42    → ProductsLayout + ProductDetailPage
URL: /products/new   → ProductsLayout + CreateProductPage
```

---

## 4. Dynamic Routes — Route Động

### Cấu Trúc Dynamic Segment

```
Route pattern:  /blog/:year/:month/:slug
URL thực tế:    /blog/2026/06/react-router-v6

useParams() trả về:
{
  year: "2026",
  month: "06",
  slug: "react-router-v6"
}
```

### Ví Dụ Thực Tế: Blog

```jsx
// Router config
const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: "blog",
        element: <BlogLayout />,
        children: [
          { index: true, element: <BlogListPage /> },
          {
            path: ":slug",               // /blog/ten-bai-viet
            element: <BlogPostPage />,
            loader: blogPostLoader,      // Tải bài viết trước khi render
          },
          {
            path: "tag/:tagName",        // /blog/tag/react
            element: <BlogTagPage />,
          },
          {
            path: "author/:authorId",    // /blog/author/nguyen-an
            element: <AuthorPage />,
          },
        ],
      },
    ],
  },
]);
```

```jsx
// BlogPostPage sử dụng useParams
import { useParams, useLoaderData } from "react-router-dom";

async function blogPostLoader({ params }) {
  const response = await fetch(`/api/blog/${params.slug}`);
  if (!response.ok) throw new Response("Bài viết không tồn tại", { status: 404 });
  return response.json();
}

function BlogPostPage() {
  const { slug } = useParams();
  const post = useLoaderData();

  return (
    <article>
      <h1>{post.title}</h1>
      <time>{new Date(post.publishedAt).toLocaleDateString("vi-VN")}</time>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
      <p>
        <Link to={`/blog/author/${post.author.id}`}>{post.author.name}</Link>
      </p>
    </article>
  );
}
```

### Splat Route — Bắt Đường Dẫn Nhiều Cấp

```jsx
// Route bắt tất cả đường dẫn con, kể cả nhiều cấp
// Route pattern: /files/*
// URL: /files/documents/2026/report.pdf

function FileBrowser() {
  const params = useParams();
  const currentPath = params["*"]; // "documents/2026/report.pdf"

  const pathParts = currentPath.split("/").filter(Boolean);
  // ["documents", "2026", "report.pdf"]

  return (
    <div>
      {/* Breadcrumb navigation */}
      <nav className="breadcrumb">
        <Link to="/files">Tất cả files</Link>
        {pathParts.map((part, index) => {
          const partPath = pathParts.slice(0, index + 1).join("/");
          return (
            <span key={partPath}>
              {" / "}
              <Link to={`/files/${partPath}`}>{part}</Link>
            </span>
          );
        })}
      </nav>

      <FileList path={currentPath} />
    </div>
  );
}
```

---

## 5. Relative Paths — Đường Dẫn Tương Đối

React Router v6 hỗ trợ **Relative Paths** (Đường Dẫn Tương Đối) trong `to` prop của Link và navigate():

```jsx
// Giả sử đang ở route: /dashboard/users/42

function UserDetailPage() {
  return (
    <div>
      {/* Absolute path — luôn từ root */}
      <Link to="/dashboard/users">Danh sách người dùng</Link>

      {/* Relative path — relative đến route hiện tại */}
      <Link to="..">Quay lại (../= /dashboard/users)</Link>
      <Link to="../43">Người dùng 43 (../43 = /dashboard/users/43)</Link>

      {/* relative="path" — relative đến URL path, không phải route hierarchy */}
      <Link to=".." relative="path">Lên một cấp URL</Link>
    </div>
  );
}
```

### useNavigate với Relative Paths

```jsx
function UserDetailPage() {
  const navigate = useNavigate();

  const handleDelete = async (userId) => {
    await deleteUser(userId);
    // Sau khi xóa, quay về danh sách users (lên 1 cấp trong route hierarchy)
    navigate("..");
  };

  return (
    <button onClick={() => handleDelete(params.userId)}>
      Xóa người dùng
    </button>
  );
}
```

---

## 6. useMatch — Kiểm Tra Route Khớp

```jsx
import { useMatch } from "react-router-dom";

function Breadcrumbs() {
  // Kiểm tra xem URL có khớp với pattern này không
  const isDashboard = useMatch("/dashboard/*");
  const isUserDetail = useMatch("/dashboard/users/:userId");

  if (isUserDetail) {
    const { userId } = isUserDetail.params;
    return (
      <nav>
        <Link to="/dashboard">Dashboard</Link>
        {" > "}
        <Link to="/dashboard/users">Người dùng</Link>
        {" > "}
        <span>User #{userId}</span>
      </nav>
    );
  }

  if (isDashboard) {
    return (
      <nav>
        <Link to="/dashboard">Dashboard</Link>
      </nav>
    );
  }

  return null;
}
```

---

## 7. Layout Patterns Thực Tế

### Pattern 1: Pathless Layout Route (Route Layout Không Có Path)

```jsx
// Layout route không có path — chỉ cung cấp layout wrapper, không thay đổi URL
const router = createBrowserRouter([
  {
    element: <AuthLayout />,     // Không có "path" — đây là layout wrapper
    children: [
      { path: "/login", element: <LoginPage /> },
      { path: "/register", element: <RegisterPage /> },
      { path: "/forgot-password", element: <ForgotPasswordPage /> },
    ],
  },
  {
    element: <MainLayout />,     // Layout chính
    children: [
      { path: "/", element: <HomePage /> },
      { path: "/about", element: <AboutPage /> },
      {
        element: <DashboardLayout />,  // Layout dashboard lồng thêm
        children: [
          { path: "/dashboard", element: <DashboardOverview /> },
          { path: "/dashboard/settings", element: <SettingsPage /> },
        ],
      },
    ],
  },
]);
```

### Pattern 2: Multi-Column Layout

```jsx
function ProductsLayout() {
  return (
    <div className="products-page">
      <aside className="filters-panel">
        <FiltersPanel />
      </aside>

      <div className="products-content">
        {/* Route con (list hoặc detail) render ở đây */}
        <Outlet />
      </div>

      <aside className="cart-panel">
        <CartSummary />
      </aside>
    </div>
  );
}
```

### Pattern 3: Tab Navigation Với Nested Routes

```jsx
// URL: /profile/posts, /profile/followers, /profile/following
function ProfileLayout() {
  const { userId } = useParams();

  return (
    <div>
      <ProfileHeader userId={userId} />

      <nav className="profile-tabs">
        <NavLink to="posts" end>Bài Viết</NavLink>
        <NavLink to="followers">Người Theo Dõi</NavLink>
        <NavLink to="following">Đang Theo Dõi</NavLink>
      </nav>

      <div className="tab-content">
        <Outlet />
      </div>
    </div>
  );
}

// Router config
{
  path: "profile/:userId",
  element: <ProfileLayout />,
  children: [
    { index: true, element: <Navigate to="posts" replace /> },
    { path: "posts", element: <UserPostsPage /> },
    { path: "followers", element: <FollowersPage /> },
    { path: "following", element: <FollowingPage /> },
  ],
}
```

---

## 8. Lazy Loading Routes — Tải Route Lười Biếng

**Lazy Loading** (Tải Lười Biếng) kết hợp với **Code Splitting** (Tách Code) giảm kích thước bundle ban đầu, chỉ tải code khi cần:

```jsx
import { lazy, Suspense } from "react";
import { createBrowserRouter } from "react-router-dom";

// lazy() tạo component được tải async — chỉ fetch khi user điều hướng đến route đó
const DashboardPage = lazy(() => import("./pages/DashboardPage"));
const AnalyticsPage = lazy(() => import("./pages/AnalyticsPage"));
const SettingsPage = lazy(() => import("./pages/SettingsPage"));

const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    children: [
      { index: true, element: <HomePage /> },      // Không lazy — cần load ngay
      {
        path: "dashboard",
        element: (
          <Suspense fallback={<PageLoadingSpinner />}>
            <DashboardPage />
          </Suspense>
        ),
        children: [
          {
            path: "analytics",
            element: (
              <Suspense fallback={<PageLoadingSpinner />}>
                <AnalyticsPage />
              </Suspense>
            ),
          },
        ],
      },
    ],
  },
]);
```

### Lazy Loading Với React Router v6.9+ lazy prop

```jsx
// Cú pháp mới hơn — lazy prop trên route object
const router = createBrowserRouter([
  {
    path: "/dashboard",
    // lazy: route component + loader được tải song song
    lazy: async () => {
      const { DashboardPage, loader } = await import("./pages/DashboardPage");
      return { element: <DashboardPage />, loader };
    },
  },
]);
```

---

## 9. Câu Hỏi Phỏng Vấn

### Câu 1: Outlet là gì và tại sao nó quan trọng?

**Trả lời:**
`<Outlet>` là một placeholder trong component cha, nơi React Router render component con tương ứng với URL hiện tại. Nhờ `<Outlet>`, ta có thể:
- **Tái sử dụng layout** — header, sidebar, footer chỉ viết một lần
- **Tạo nested layouts phức tạp** — nhiều cấp layout lồng nhau
- **Truyền dữ liệu xuống** qua `<Outlet context={...}>` và `useOutletContext()`

Nếu không có `<Outlet>`, mỗi trang phải viết lại toàn bộ layout — vi phạm nguyên tắc DRY (Don't Repeat Yourself — Không Lặp Lại Chính Mình).

### Câu 2: Index Route là gì? Khi nào dùng?

**Trả lời:**
Index Route là route con với `index: true`, được render khi URL khớp đúng với path của route cha — không có segment nào thêm. Dùng khi:
- Cần hiển thị "trang mặc định" khi mới vào một section (ví dụ: Dashboard Overview là mặc định của `/dashboard`)
- Thay thế nội dung trống khi chưa chọn item nào trong layout có Outlet

### Câu 3: Nested Routes giúp gì cho performance (hiệu năng)?

**Trả lời:**
Với Nested Routes và Data APIs (loader):
- **Parallel data loading** (tải dữ liệu song song): React Router tải data cho tất cả route trong hierarchy cùng lúc, tránh **waterfall** (thác nước) — tình trạng request này phải chờ request kia xong mới bắt đầu.
- **Selective re-rendering** (render lại có chọn lọc): Khi điều hướng giữa các route con, layout cha không unmount/remount — chỉ phần Outlet được cập nhật.
- **Lazy loading**: Kết hợp `lazy()` để chỉ tải code của route khi user thực sự điều hướng đến.

### Câu 4: Phân biệt `to=".."` và `to="../"` trong Link?

**Trả lời:**
Trong React Router v6, `to=".."` điều hướng lên một cấp trong **route hierarchy** (cấu trúc phân cấp route), không phải URL path. Ví dụ:
- Route: `/dashboard/users/:userId` → `to=".."` → `/dashboard/users` (lên 1 cấp route)

Để điều hướng theo URL path, dùng `relative="path"`:
- URL: `/dashboard/users/42` → `to=".." relative="path"` → `/dashboard/users`

Thêm prop `relative="path"` để hành vi giống `cd ..` trong terminal.

---

## 🔗 Điều Hướng

- **Trước đó:** [1-react-router-basics.md](./1-react-router-basics.md) — React Router v6 cơ bản
- **Tiếp theo:** [3-protected-routes.md](./3-protected-routes.md) — Protected Routes và Auth Guards
- **Quay lại:** [README.md](./README.md) — Tổng quan Routing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
