# Protected Routes — Route Được Bảo Vệ Và Auth Guards

> **Protected Routes** (Route Được Bảo Vệ) là cơ chế ngăn người dùng chưa xác thực — **unauthenticated users** — truy cập vào các trang yêu cầu đăng nhập. **Auth Guard** (Lính Canh Xác Thực) là pattern kiểm tra quyền truy cập trước khi cho phép render. Đây là một trong những tính năng thiết yếu trong bất kỳ ứng dụng có authentication nào.

---

## 📌 Mục Lục

1. [Vấn Đề Cần Giải Quyết](#1-vấn-đề)
2. [PrivateRoute Component — Cách Tiếp Cận Cơ Bản](#2-privateroute-component)
3. [Auth Guard Với useNavigate](#3-auth-guard-với-usenavigate)
4. [Protected Routes Với Nested Layout](#4-protected-routes-với-nested-layout)
5. [Redirect Về Sau Khi Đăng Nhập](#5-redirect-về-sau-khi-đăng-nhập)
6. [Role-Based Access Control — Kiểm Soát Truy Cập Theo Vai Trò](#6-rbac)
7. [Auth Với Data APIs — loader](#7-auth-với-loader)
8. [Persistent Auth — Lưu Trạng Thái Đăng Nhập](#8-persistent-auth)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề Cần Giải Quyết

Kịch bản điển hình:

```
User chưa đăng nhập nhập URL: /dashboard/settings
                                    ↓
                    Không có token/session hợp lệ
                                    ↓
                    → Redirect về /login
                    → Sau khi login thành công → /dashboard/settings
```

**Các loại bảo vệ route phổ biến:**

| Loại | Điều Kiện Cho Phép | Ví Dụ |
| ---- | ------------------ | ----- |
| **Authentication Guard** (Lính Canh Xác Thực) | Đã đăng nhập | `/dashboard`, `/profile` |
| **Role Guard** (Lính Canh Vai Trò) | Có đủ vai trò/quyền hạn | `/admin`, `/moderator` |
| **Permission Guard** (Lính Canh Quyền Hạn) | Có quyền cụ thể | `/users/edit`, `/reports/export` |
| **Feature Flag Guard** (Lính Canh Tính Năng) | Tính năng được bật | `/beta-features` |

---

## 2. PrivateRoute Component — Cách Tiếp Cận Cơ Bản

### Cách Đơn Giản Nhất

```jsx
import { Navigate, Outlet } from "react-router-dom";
import { useAuth } from "./hooks/useAuth";

// PrivateRoute — wrapper bảo vệ route
// Nếu chưa đăng nhập → redirect về /login
// Nếu đã đăng nhập → render route con bình thường
function PrivateRoute() {
  const { isLoggedIn, isLoading } = useAuth();

  // Đang kiểm tra trạng thái auth (ví dụ: đang verify token với server)
  if (isLoading) {
    return <div className="auth-loading">Đang kiểm tra phiên đăng nhập...</div>;
  }

  if (!isLoggedIn) {
    // replace: true — không thêm /dashboard vào history
    // → nhấn Back sau khi login không quay về trang bị chặn
    return <Navigate to="/login" replace />;
  }

  // Render route con (sử dụng Outlet cho nested routes)
  return <Outlet />;
}
```

### Sử Dụng PrivateRoute Trong Router Config

```jsx
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    children: [
      { index: true, element: <HomePage /> },
      { path: "about", element: <AboutPage /> },
      { path: "login", element: <LoginPage /> },
      { path: "register", element: <RegisterPage /> },

      // Nhóm các route cần authentication
      {
        element: <PrivateRoute />,          // Không có path — chỉ là guard wrapper
        children: [
          { path: "dashboard", element: <DashboardPage /> },
          { path: "profile", element: <ProfilePage /> },
          { path: "settings", element: <SettingsPage /> },
          {
            path: "orders",
            element: <OrdersLayout />,
            children: [
              { index: true, element: <OrdersListPage /> },
              { path: ":orderId", element: <OrderDetailPage /> },
            ],
          },
        ],
      },

      { path: "*", element: <NotFoundPage /> },
    ],
  },
]);
```

---

## 3. Auth Guard Với useNavigate

Cách tiếp cận khác: Kiểm tra trong chính component hoặc trong custom hook:

```jsx
import { useEffect } from "react";
import { useNavigate, useLocation } from "react-router-dom";
import { useAuth } from "./hooks/useAuth";

// Hook bảo vệ — dùng trong component thay vì wrapper route
function useRequireAuth(redirectTo = "/login") {
  const { isLoggedIn, isLoading } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();

  useEffect(() => {
    if (!isLoading && !isLoggedIn) {
      navigate(redirectTo, {
        replace: true,
        state: { from: location.pathname },
      });
    }
  }, [isLoggedIn, isLoading, navigate, redirectTo, location]);

  return { isLoggedIn, isLoading };
}

// Sử dụng trong component
function DashboardPage() {
  const { isLoading } = useRequireAuth();

  if (isLoading) return <PageSpinner />;

  return (
    <div>
      <h1>Dashboard</h1>
      {/* Nội dung dashboard */}
    </div>
  );
}
```

---

## 4. Protected Routes Với Nested Layout

Kết hợp bảo vệ route với layout chung cho tất cả trang authenticated (đã xác thực):

```jsx
import { Navigate, Outlet } from "react-router-dom";
import { useAuth } from "./hooks/useAuth";

// AuthenticatedLayout — layout bao gồm cả guard + giao diện chung
function AuthenticatedLayout() {
  const { user, isLoggedIn, isLoading } = useAuth();

  if (isLoading) {
    return (
      <div className="full-page-loading">
        <Spinner />
        <p>Đang xác thực...</p>
      </div>
    );
  }

  if (!isLoggedIn) {
    return <Navigate to="/login" replace />;
  }

  return (
    <div className="authenticated-app">
      {/* Header với thông tin user — hiển thị trên tất cả trang authenticated */}
      <header className="app-header">
        <Logo />
        <nav>
          <NavLink to="/dashboard">Dashboard</NavLink>
          <NavLink to="/profile">Hồ Sơ</NavLink>
        </nav>
        <div className="user-menu">
          <span>Xin chào, {user.name}</span>
          <LogoutButton />
        </div>
      </header>

      <div className="app-body">
        {/* Sidebar chung cho tất cả trang authenticated */}
        <aside className="main-sidebar">
          <MainNavigation userRole={user.role} />
        </aside>

        <main className="page-content">
          {/* Nội dung trang cụ thể render ở đây */}
          <Outlet />
        </main>
      </div>
    </div>
  );
}

// Router config sạch sẽ
const router = createBrowserRouter([
  // Public routes — không cần đăng nhập
  { path: "/login", element: <LoginPage /> },
  { path: "/register", element: <RegisterPage /> },
  { path: "/forgot-password", element: <ForgotPasswordPage /> },

  // Private routes — bảo vệ bởi AuthenticatedLayout
  {
    element: <AuthenticatedLayout />,
    children: [
      { path: "/", element: <HomePage /> },
      { path: "/dashboard", element: <DashboardPage /> },
      { path: "/profile", element: <ProfilePage /> },
      { path: "/profile/edit", element: <EditProfilePage /> },
      {
        path: "/orders",
        element: <OrdersLayout />,
        children: [
          { index: true, element: <OrdersListPage /> },
          { path: ":orderId", element: <OrderDetailPage /> },
        ],
      },
    ],
  },

  { path: "*", element: <NotFoundPage /> },
]);
```

---

## 5. Redirect Về Sau Khi Đăng Nhập

Pattern quan trọng: Sau khi đăng nhập thành công, redirect về trang user đã cố gắng truy cập trước đó.

```jsx
// 1. Guard ghi nhớ "from" trong state khi redirect
function PrivateRoute() {
  const { isLoggedIn } = useAuth();
  const location = useLocation();

  if (!isLoggedIn) {
    return (
      <Navigate
        to="/login"
        replace
        state={{ from: location.pathname + location.search }}
        // state.from = "/dashboard/settings?tab=notifications"
      />
    );
  }

  return <Outlet />;
}
```

```jsx
// 2. LoginPage đọc "from" và redirect về sau khi login
import { useNavigate, useLocation } from "react-router-dom";
import { useAuth } from "./hooks/useAuth";

function LoginPage() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();

  // Lấy URL user muốn truy cập trước khi bị chặn
  // Nếu không có "from", mặc định về trang chủ
  const from = location.state?.from ?? "/";

  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);

    try {
      await login(formData.get("email"), formData.get("password"));

      // Redirect về trang ban đầu, không thể nhấn Back về /login
      navigate(from, { replace: true });
    } catch (error) {
      setError("Email hoặc mật khẩu không đúng.");
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <h1>Đăng Nhập</h1>

      {/* Hiển thị thông báo nếu user bị redirect về đây */}
      {location.state?.from && (
        <div className="auth-notice">
          Vui lòng đăng nhập để tiếp tục.
        </div>
      )}

      <input type="email" name="email" placeholder="Email" required />
      <input type="password" name="password" placeholder="Mật khẩu" required />
      {error && <p className="error">{error}</p>}
      <button type="submit">Đăng Nhập</button>
    </form>
  );
}
```

---

## 6. RBAC — Role-Based Access Control (Kiểm Soát Truy Cập Theo Vai Trò)

Kiểm tra không chỉ "đã đăng nhập" mà còn "có quyền truy cập không":

```jsx
// RoleGuard component — kiểm tra role
function RoleGuard({ allowedRoles, children }) {
  const { user, isLoggedIn, isLoading } = useAuth();

  if (isLoading) return <PageSpinner />;

  if (!isLoggedIn) {
    return <Navigate to="/login" replace />;
  }

  // Kiểm tra user có đủ vai trò không
  const hasRequiredRole = allowedRoles.includes(user.role);

  if (!hasRequiredRole) {
    // Đã đăng nhập nhưng không đủ quyền → trang 403 Forbidden (Bị Từ Chối)
    return <Navigate to="/forbidden" replace />;
  }

  return children ?? <Outlet />;
}
```

### Router Config Với RBAC

```jsx
const router = createBrowserRouter([
  // Public routes
  { path: "/login", element: <LoginPage /> },

  // Authenticated routes
  {
    element: <AuthenticatedLayout />,  // Kiểm tra đã đăng nhập
    children: [
      { path: "/dashboard", element: <DashboardPage /> },
      { path: "/profile", element: <ProfilePage /> },

      // Admin-only routes — chỉ role "admin" mới được vào
      {
        element: <RoleGuard allowedRoles={["admin"]} />,
        children: [
          { path: "/admin", element: <AdminDashboardPage /> },
          { path: "/admin/users", element: <AdminUsersPage /> },
          { path: "/admin/settings", element: <AdminSettingsPage /> },
        ],
      },

      // Moderator và Admin được vào
      {
        element: <RoleGuard allowedRoles={["admin", "moderator"]} />,
        children: [
          { path: "/reports", element: <ReportsPage /> },
          { path: "/content-review", element: <ContentReviewPage /> },
        ],
      },
    ],
  },

  { path: "/forbidden", element: <ForbiddenPage /> },
  { path: "*", element: <NotFoundPage /> },
]);
```

### Permission-Based Guard (Kiểm Tra Quyền Chi Tiết Hơn)

```jsx
// Kiểm tra permission (quyền hạn) thay vì chỉ role (vai trò)
function PermissionGuard({ permission }) {
  const { user } = useAuth();

  const hasPermission = user?.permissions?.includes(permission);

  if (!hasPermission) {
    return <Navigate to="/forbidden" replace />;
  }

  return <Outlet />;
}

// Sử dụng
{
  element: <PermissionGuard permission="users:write" />,
  children: [
    { path: "/users/create", element: <CreateUserPage /> },
    { path: "/users/:id/edit", element: <EditUserPage /> },
  ],
}
```

---

## 7. Auth Với Loader — Data API Approach

Khi dùng `createBrowserRouter` với Data APIs, có thể kiểm tra auth trong **loader** thay vì component:

```jsx
import { redirect } from "react-router-dom";

// Utility function kiểm tra auth, dùng trong loader
async function requireAuth(request) {
  const token = localStorage.getItem("token");

  if (!token) {
    // Lấy URL hiện tại để redirect về sau khi login
    const url = new URL(request.url);
    const params = new URLSearchParams({ from: url.pathname + url.search });
    throw redirect(`/login?${params}`);
  }

  // Verify token với server (kiểm tra token còn hợp lệ không)
  try {
    const user = await verifyToken(token);
    return user;
  } catch {
    localStorage.removeItem("token");
    throw redirect("/login");
  }
}

// Loader của trang cần bảo vệ
async function dashboardLoader({ request }) {
  const user = await requireAuth(request); // Ném redirect nếu chưa đăng nhập
  const stats = await fetchDashboardStats(user.id);
  return { user, stats };
}

async function adminLoader({ request }) {
  const user = await requireAuth(request);

  if (user.role !== "admin") {
    throw redirect("/forbidden");
  }

  const adminData = await fetchAdminData();
  return { user, adminData };
}

// Router config
const router = createBrowserRouter([
  {
    path: "/dashboard",
    element: <DashboardPage />,
    loader: dashboardLoader,  // Auth check xảy ra tại đây, trước khi render
  },
  {
    path: "/admin",
    element: <AdminPage />,
    loader: adminLoader,      // Auth + role check
  },
]);
```

### Lợi Ích Của Loader Approach

```
useEffect approach:
  1. Component render → trống hoặc loading
  2. useEffect chạy → kiểm tra auth
  3. Re-render hoặc redirect

loader approach:
  1. loader chạy → kiểm tra auth
  2. Nếu pass → component render với data đã sẵn sàng
  3. Nếu fail → redirect xảy ra TRƯỚC KHI component render
```

---

## 8. Persistent Auth — Lưu Trạng Thái Đăng Nhập

### AuthContext Hoàn Chỉnh Với Persistence (Lưu Trữ Lâu Dài)

```jsx
import { createContext, useContext, useState, useEffect } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(true); // Quan trọng: bắt đầu là true

  // Khôi phục session khi app khởi động
  useEffect(() => {
    const initializeAuth = async () => {
      const token = localStorage.getItem("authToken");

      if (!token) {
        setIsLoading(false);
        return;
      }

      try {
        // Verify token với server — đảm bảo token chưa hết hạn hoặc bị thu hồi
        const response = await fetch("/api/auth/me", {
          headers: { Authorization: `Bearer ${token}` },
        });

        if (response.ok) {
          const userData = await response.json();
          setUser(userData);
        } else {
          // Token không hợp lệ — xóa đi
          localStorage.removeItem("authToken");
        }
      } catch {
        localStorage.removeItem("authToken");
      } finally {
        setIsLoading(false);
      }
    };

    initializeAuth();
  }, []);

  const login = async (email, password) => {
    const response = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message);
    }

    const { user, token } = await response.json();
    localStorage.setItem("authToken", token);
    setUser(user);
  };

  const logout = async () => {
    try {
      // Notify server để invalidate token — vô hiệu hóa token trên server
      await fetch("/api/auth/logout", {
        method: "POST",
        headers: { Authorization: `Bearer ${localStorage.getItem("authToken")}` },
      });
    } finally {
      localStorage.removeItem("authToken");
      setUser(null);
    }
  };

  const updateUser = (updates) => {
    setUser(prev => ({ ...prev, ...updates }));
  };

  const value = {
    user,
    isLoggedIn: !!user,
    isLoading,
    login,
    logout,
    updateUser,
  };

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth phải được dùng trong AuthProvider");
  }
  return context;
}
```

### Xử Lý Token Hết Hạn — Token Expiry

```jsx
// Axios interceptor — tự động xử lý 401 Unauthorized
import axios from "axios";

const api = axios.create({ baseURL: "/api" });

// Request interceptor — đính kèm token vào mỗi request
api.interceptors.request.use(config => {
  const token = localStorage.getItem("authToken");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor — xử lý token hết hạn
api.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Token hết hạn — thử refresh (làm mới) token
      const refreshToken = localStorage.getItem("refreshToken");

      if (refreshToken) {
        try {
          const { data } = await axios.post("/api/auth/refresh", { refreshToken });
          localStorage.setItem("authToken", data.token);
          // Retry request gốc với token mới
          error.config.headers.Authorization = `Bearer ${data.token}`;
          return api.request(error.config);
        } catch {
          // Refresh thất bại → logout
          localStorage.removeItem("authToken");
          localStorage.removeItem("refreshToken");
          window.location.href = "/login";
        }
      }
    }
    return Promise.reject(error);
  }
);
```

---

## 9. Câu Hỏi Phỏng Vấn

### Câu 1: Triển khai Protected Route trong React Router v6 như thế nào?

**Trả lời:**
Pattern phổ biến nhất là tạo một **wrapper component** không có `path`, chỉ chứa logic kiểm tra auth:

```jsx
function PrivateRoute() {
  const { isLoggedIn, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) return <Spinner />;
  if (!isLoggedIn) return <Navigate to="/login" state={{ from: location.pathname }} replace />;
  return <Outlet />;
}
```

Đặt các route cần bảo vệ làm children của `PrivateRoute` trong router config. Điểm mấu chốt: dùng `replace` để không thêm trang bị chặn vào history, và dùng `state.from` để redirect về sau khi login.

### Câu 2: Tại sao isLoading quan trọng trong AuthProvider?

**Trả lời:**
Khi app khởi động, AuthProvider cần thời gian để kiểm tra session (verify token với server, đọc localStorage). Nếu không có `isLoading`:
- `isLoggedIn` ban đầu là `false`
- Protected Route render ngay lập tức → redirect về `/login`
- Token verify xong → user đã bị đẩy đến trang login không cần thiết

Với `isLoading = true` ban đầu, Protected Route hiển thị loading spinner cho đến khi auth state được xác định.

### Câu 3: Sự khác biệt giữa localStorage và sessionStorage để lưu token?

**Trả lời:**

| | localStorage | sessionStorage | httpOnly Cookie |
| - | ------------ | -------------- | --------------- |
| **Tồn tại** | Đến khi xóa | Đến khi đóng tab | Theo max-age |
| **XSS** | Dễ bị đánh cắp | Dễ bị đánh cắp | An toàn hơn |
| **CSRF** | Không bị | Không bị | Cần CSRF token |
| **Khuyến nghị** | ❌ Không nên | ❌ Không nên | ✅ Tốt nhất |

Trong thực tế nhiều SPA dùng `localStorage` vì tiện, nhưng giải pháp an toàn nhất là **httpOnly Cookie** — cookie không thể đọc bởi JavaScript, ngăn chặn XSS (Cross-Site Scripting — Tấn Công Kịch Bản Liên Trang).

### Câu 4: Role-based và Permission-based access control — Khi nào dùng gì?

**Trả lời:**
- **Role-based** (theo vai trò): Đơn giản hơn, phù hợp khi quyền hạn gắn liền với vai trò cố định (admin, user, moderator). Dễ hiểu và quản lý.
- **Permission-based** (theo quyền hạn): Chi tiết hơn, phù hợp khi cần kiểm soát granular — ví dụ user có thể `read:reports` nhưng không có `write:reports`. Phức tạp hơn nhưng linh hoạt hơn cho ứng dụng enterprise.

Thực tế: nhiều hệ thống kết hợp cả hai — roles bao gồm một tập hợp permissions.

---

## 🔗 Điều Hướng

- **Trước đó:** [2-nested-dynamic-routes.md](./2-nested-dynamic-routes.md) — Nested Routes và Dynamic Params
- **Tiếp theo:** [4-react-router-v7.md](./4-react-router-v7.md) — React Router v7
- **Quay lại:** [README.md](./README.md) — Tổng quan Routing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
