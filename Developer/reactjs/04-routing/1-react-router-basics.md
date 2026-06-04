# React Router v6 — Cơ Bản

> **React Router v6** là phiên bản lớn nhất của thư viện điều hướng phổ biến nhất cho React, được phát hành năm 2021 với nhiều thay đổi breaking — phá vỡ khả năng tương thích ngược. Phiên bản này đơn giản hóa API, tích hợp **Data APIs** (API Dữ Liệu) mạnh mẽ, và là nền tảng cho React Router v7.

---

## 📌 Mục Lục

1. [Cài Đặt và Thiết Lập](#1-cài-đặt-và-thiết-lập)
2. [BrowserRouter và Routes](#2-browserrouter-và-routes)
3. [Link và NavLink](#3-link-và-navlink)
4. [useNavigate — Điều Hướng Lập Trình](#4-usenavigate)
5. [useParams — Đọc URL Parameters](#5-useparams)
6. [useSearchParams — Query String](#6-usesearchparams)
7. [useLocation — Thông Tin Vị Trí](#7-uselocation)
8. [Data APIs — loader và action](#8-data-apis)
9. [404 Not Found và Fallback Route](#9-404-not-found)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Cài Đặt Và Thiết Lập

```bash
npm install react-router-dom
```

### Phiên Bản Và Compatibility (Tương Thích)

| Phiên Bản | React | Node.js | Ghi Chú |
| --------- | ----- | ------- | ------- |
| v6.x | ≥ 16.8 | ≥ 14 | Phiên bản hiện tại (stable) |
| v7.x | ≥ 18 | ≥ 20 | Framework mode, SSR |
| v5.x | ≥ 15 | ≥ 8 | Legacy — không khuyến nghị cho dự án mới |

---

## 2. BrowserRouter Và Routes

### Thiết Lập Cơ Bản

```jsx
// main.jsx hoặc index.jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import App from "./App";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

### Cấu Hình Routes (Khai Báo Route)

```jsx
// App.jsx
import { Routes, Route } from "react-router-dom";
import HomePage from "./pages/HomePage";
import AboutPage from "./pages/AboutPage";
import ProductsPage from "./pages/ProductsPage";
import ProductDetailPage from "./pages/ProductDetailPage";
import NotFoundPage from "./pages/NotFoundPage";

function App() {
  return (
    <Routes>
      {/* Route chính xác: / */}
      <Route path="/" element={<HomePage />} />

      {/* Route tĩnh: /about */}
      <Route path="/about" element={<AboutPage />} />

      {/* Route tĩnh: /products */}
      <Route path="/products" element={<ProductsPage />} />

      {/* Route động: /products/123 */}
      <Route path="/products/:id" element={<ProductDetailPage />} />

      {/* Wildcard route — bắt tất cả URL không khớp → 404 */}
      <Route path="*" element={<NotFoundPage />} />
    </Routes>
  );
}
```

### So Sánh v5 vs v6

```jsx
// ❌ React Router v5 — cú pháp cũ
import { Switch, Route } from "react-router-dom";

function App() {
  return (
    <Switch>
      <Route exact path="/" component={HomePage} />
      <Route path="/about" component={AboutPage} />
      <Route path="/products/:id" component={ProductDetailPage} />
    </Switch>
  );
}

// ✅ React Router v6 — cú pháp mới
import { Routes, Route } from "react-router-dom";

function App() {
  return (
    <Routes>
      {/* Mặc định đã là "exact" — không cần thêm prop */}
      <Route path="/" element={<HomePage />} />
      <Route path="/about" element={<AboutPage />} />
      <Route path="/products/:id" element={<ProductDetailPage />} />
    </Routes>
  );
}
```

**Những thay đổi chính từ v5 lên v6:**
- `Switch` → `Routes`
- `component={Comp}` → `element={<Comp />}` (JSX thay vì reference)
- Mặc định tất cả route là "exact" — không cần prop `exact`
- `Redirect` → `Navigate` component
- `useHistory` → `useNavigate`

---

## 3. Link và NavLink

### Link — Liên Kết Không Tải Lại Trang

```jsx
import { Link } from "react-router-dom";

function Navbar() {
  return (
    <nav>
      {/* Link thay thế thẻ <a> thông thường */}
      <Link to="/">Trang Chủ</Link>
      <Link to="/about">Giới Thiệu</Link>
      <Link to="/products">Sản Phẩm</Link>

      {/* Link với state — truyền dữ liệu ngầm khi điều hướng */}
      <Link
        to="/checkout"
        state={{ from: "cart", items: cartItems }}
      >
        Thanh Toán
      </Link>

      {/* Link tương đối — relative path */}
      <Link to="../settings">Cài Đặt</Link>

      {/* Link thay thế history entry thay vì thêm mới */}
      <Link to="/login" replace>
        Đăng Nhập
      </Link>
    </nav>
  );
}
```

### NavLink — Liên Kết Với Trạng Thái Active (Đang Hoạt Động)

```jsx
import { NavLink } from "react-router-dom";

function Navbar() {
  return (
    <nav>
      {/* NavLink tự động thêm class "active" khi route khớp */}
      <NavLink to="/" end>
        Trang Chủ
      </NavLink>
      {/* "end" prop: chỉ active khi URL là "/" chính xác, không active khi "/about" */}

      {/* Tùy chỉnh className dựa trên trạng thái active */}
      <NavLink
        to="/products"
        className={({ isActive, isPending }) =>
          isActive ? "nav-link active" : isPending ? "nav-link pending" : "nav-link"
        }
      >
        Sản Phẩm
      </NavLink>

      {/* Tùy chỉnh style */}
      <NavLink
        to="/about"
        style={({ isActive }) => ({
          color: isActive ? "#ff6b6b" : "#333",
          fontWeight: isActive ? "bold" : "normal",
          textDecoration: "none",
        })}
      >
        Giới Thiệu
      </NavLink>
    </nav>
  );
}
```

---

## 4. useNavigate — Điều Hướng Lập Trình

**Programmatic Navigation** (Điều Hướng Lập Trình) — điều hướng bằng code thay vì click link, thường dùng sau khi submit form hoặc xử lý logic nghiệp vụ.

```jsx
import { useNavigate } from "react-router-dom";

function LoginForm() {
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);

    try {
      await loginApi(formData.get("email"), formData.get("password"));

      // Điều hướng đến trang dashboard sau khi đăng nhập thành công
      navigate("/dashboard");

      // Thay thế history entry (không thể nhấn Back để quay lại trang login)
      navigate("/dashboard", { replace: true });

      // Điều hướng kèm state (truyền dữ liệu ngầm)
      navigate("/dashboard", {
        replace: true,
        state: { welcomeMessage: "Đăng nhập thành công!" },
      });
    } catch (error) {
      console.error("Đăng nhập thất bại:", error.message);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" name="email" placeholder="Email" required />
      <input type="password" name="password" placeholder="Mật khẩu" required />
      <button type="submit">Đăng Nhập</button>
    </form>
  );
}
```

### navigate() với Số Âm — Quay Lại

```jsx
function ProductDetailPage() {
  const navigate = useNavigate();

  return (
    <div>
      <h1>Chi Tiết Sản Phẩm</h1>

      {/* Quay lại trang trước trong history stack */}
      <button onClick={() => navigate(-1)}>← Quay Lại</button>

      {/* Tiến đến trang tiếp theo */}
      <button onClick={() => navigate(1)}>Tiếp →</button>

      {/* Quay lại 2 bước */}
      <button onClick={() => navigate(-2)}>Quay Lại 2 Bước</button>
    </div>
  );
}
```

---

## 5. useParams — Đọc URL Parameters

**URL Parameters** (Tham Số URL) — phần động của URL được khai báo bằng `:paramName` trong route.

```jsx
import { useParams } from "react-router-dom";

// Route: <Route path="/products/:productId" element={<ProductDetailPage />} />

function ProductDetailPage() {
  const { productId } = useParams();
  // URL: /products/abc-123 → productId = "abc-123"
  // URL: /products/42     → productId = "42" (luôn là string!)

  const productIdNumber = Number(productId); // Chuyển sang number nếu cần

  return <div>Chi tiết sản phẩm ID: {productId}</div>;
}
```

### Nhiều Params

```jsx
// Route: <Route path="/categories/:categorySlug/products/:productId" element={<ProductPage />} />

function ProductPage() {
  const { categorySlug, productId } = useParams();
  // URL: /categories/electronics/products/42
  // → categorySlug = "electronics", productId = "42"

  return (
    <div>
      <p>Danh mục: {categorySlug}</p>
      <p>Sản phẩm: {productId}</p>
    </div>
  );
}
```

### Optional Params với Splat Routes

```jsx
// Route: <Route path="/files/*" element={<FileBrowser />} />

function FileBrowser() {
  const params = useParams();
  const filePath = params["*"];
  // URL: /files/documents/report.pdf → filePath = "documents/report.pdf"

  return <div>Đường dẫn: {filePath}</div>;
}
```

---

## 6. useSearchParams — Query String

**Query String** (Chuỗi Truy Vấn) — phần URL sau dấu `?`, dùng để truyền dữ liệu không ảnh hưởng đến route hierarchy — ví dụ: `/products?category=phone&sort=price`.

```jsx
import { useSearchParams } from "react-router-dom";

function ProductsPage() {
  const [searchParams, setSearchParams] = useSearchParams();

  // Đọc query params
  const category = searchParams.get("category") ?? "all";  // "phone" hoặc null
  const sort = searchParams.get("sort") ?? "newest";
  const page = Number(searchParams.get("page") ?? "1");

  // Cập nhật search params — URL thay đổi, component re-render
  const handleCategoryChange = (newCategory) => {
    setSearchParams(prev => {
      prev.set("category", newCategory);
      prev.set("page", "1"); // Reset về trang 1 khi đổi category
      return prev;
    });
  };

  const handleSortChange = (newSort) => {
    setSearchParams(prev => {
      prev.set("sort", newSort);
      return prev;
    });
  };

  return (
    <div>
      <div className="filters">
        <select
          value={category}
          onChange={e => handleCategoryChange(e.target.value)}
        >
          <option value="all">Tất cả</option>
          <option value="phone">Điện thoại</option>
          <option value="laptop">Laptop</option>
        </select>

        <select
          value={sort}
          onChange={e => handleSortChange(e.target.value)}
        >
          <option value="newest">Mới nhất</option>
          <option value="price-asc">Giá tăng dần</option>
          <option value="price-desc">Giá giảm dần</option>
        </select>
      </div>

      {/* Hiển thị danh sách theo category và sort */}
      <ProductList category={category} sort={sort} page={page} />
    </div>
  );
}
```

### Utility Hook: useQueryParam — Tùy Chỉnh

```jsx
// Custom hook tiện lợi cho một search param đơn lẻ
function useQueryParam(key, defaultValue = "") {
  const [searchParams, setSearchParams] = useSearchParams();

  const value = searchParams.get(key) ?? defaultValue;

  const setValue = (newValue) => {
    setSearchParams(prev => {
      if (newValue === defaultValue) {
        prev.delete(key); // Xóa param nếu bằng giá trị mặc định — URL gọn hơn
      } else {
        prev.set(key, String(newValue));
      }
      return prev;
    });
  };

  return [value, setValue];
}

// Sử dụng
function SearchPage() {
  const [query, setQuery] = useQueryParam("q", "");
  const [page, setPage] = useQueryParam("page", "1");

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Tìm kiếm..."
      />
      <p>Trang: {page}</p>
    </div>
  );
}
```

---

## 7. useLocation — Thông Tin Vị Trí

**Location Object** (Đối Tượng Vị Trí) chứa thông tin về URL hiện tại: `pathname`, `search`, `hash`, `state`, `key`.

```jsx
import { useLocation } from "react-router-dom";

function CurrentLocationDebug() {
  const location = useLocation();

  // Ví dụ URL: /products?category=phone#section-2
  console.log(location.pathname); // "/products"
  console.log(location.search);   // "?category=phone"
  console.log(location.hash);     // "#section-2"
  console.log(location.state);    // Dữ liệu truyền qua navigate() hoặc Link state
  console.log(location.key);      // Unique key cho mỗi navigation entry

  return <pre>{JSON.stringify(location, null, 2)}</pre>;
}
```

### Dùng location.state Để Truyền Dữ Liệu Ngầm

```jsx
// Trang A — Truyền state khi điều hướng
function CartPage() {
  const navigate = useNavigate();
  const { items } = useCart();

  const handleCheckout = () => {
    navigate("/checkout", {
      state: {
        cartItems: items,
        totalPrice: calculateTotal(items),
        discountCode: "SAVE10",
      },
    });
  };

  return <button onClick={handleCheckout}>Thanh Toán</button>;
}

// Trang B — Đọc state được truyền
function CheckoutPage() {
  const location = useLocation();
  const { cartItems, totalPrice, discountCode } = location.state ?? {};

  // Nếu không có state (user truy cập trực tiếp qua URL), redirect về cart
  if (!cartItems) {
    return <Navigate to="/cart" replace />;
  }

  return (
    <div>
      <h1>Thanh Toán</h1>
      <p>Tổng tiền: {totalPrice?.toLocaleString("vi-VN")} VNĐ</p>
      <p>Mã giảm giá: {discountCode}</p>
    </div>
  );
}
```

### Theo Dõi Lịch Sử Navigation

```jsx
// Hook theo dõi "trang trước" để hiển thị nút Back thông minh
function usePreviousLocation() {
  const location = useLocation();
  const prevLocationRef = useRef(null);

  useEffect(() => {
    prevLocationRef.current = location;
  }, [location]);

  return prevLocationRef.current;
}
```

---

## 8. Data APIs — loader và action

React Router v6.4+ giới thiệu **Data APIs** (API Dữ Liệu) — tích hợp data fetching trực tiếp vào route config thay vì dùng useEffect trong component.

### createBrowserRouter — Khai Báo Route Với Data

```jsx
import { createBrowserRouter, RouterProvider } from "react-router-dom";

// Loader — hàm chạy trước khi render component, load dữ liệu cần thiết
async function productLoader({ params }) {
  const response = await fetch(`/api/products/${params.id}`);
  if (!response.ok) {
    throw new Response("Không tìm thấy sản phẩm", { status: 404 });
  }
  return response.json();
}

// Action — hàm xử lý form submission (POST/PUT/DELETE)
async function productEditAction({ request, params }) {
  const formData = await request.formData();
  const updates = Object.fromEntries(formData);

  await updateProduct(params.id, updates);
  return redirect(`/products/${params.id}`);
}

// Khai báo router với data APIs
const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: "products",
        element: <ProductsPage />,
        loader: productsLoader, // Chạy trước khi render ProductsPage
      },
      {
        path: "products/:id",
        element: <ProductDetailPage />,
        loader: productLoader,
        errorElement: <ProductNotFound />,
      },
      {
        path: "products/:id/edit",
        element: <ProductEditPage />,
        loader: productLoader,
        action: productEditAction, // Xử lý form submit
      },
    ],
  },
]);

// main.jsx
function App() {
  return <RouterProvider router={router} />;
}
```

### useLoaderData — Đọc Dữ Liệu Từ Loader

```jsx
import { useLoaderData, useNavigation } from "react-router-dom";

function ProductDetailPage() {
  const product = useLoaderData(); // Lấy data từ loader, đã được resolve
  const navigation = useNavigation();

  // navigation.state: "idle" | "loading" | "submitting"
  if (navigation.state === "loading") {
    return <div>Đang tải...</div>;
  }

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p className="price">{product.price.toLocaleString("vi-VN")} VNĐ</p>
    </div>
  );
}
```

### Form Component — Xử Lý Action

```jsx
import { Form, useActionData, useNavigation } from "react-router-dom";

function ProductEditPage() {
  const product = useLoaderData();
  const actionData = useActionData(); // Kết quả trả về từ action sau submit
  const navigation = useNavigation();

  const isSubmitting = navigation.state === "submitting";

  return (
    // Form của React Router — tự động gọi action khi submit
    <Form method="put">
      <input name="name" defaultValue={product.name} required />
      <textarea name="description" defaultValue={product.description} />
      <input name="price" type="number" defaultValue={product.price} />

      {/* actionData chứa lỗi validation (kiểm tra hợp lệ) nếu có */}
      {actionData?.errors && (
        <ul className="errors">
          {Object.values(actionData.errors).map(err => (
            <li key={err}>{err}</li>
          ))}
        </ul>
      )}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Đang lưu..." : "Lưu Thay Đổi"}
      </button>
    </Form>
  );
}
```

---

## 9. 404 Not Found Và Fallback Route

```jsx
// Trang 404 đơn giản
function NotFoundPage() {
  const location = useLocation();
  const navigate = useNavigate();

  return (
    <div className="not-found">
      <h1>404 — Không Tìm Thấy Trang</h1>
      <p>
        Trang <code>{location.pathname}</code> không tồn tại.
      </p>
      <div>
        <button onClick={() => navigate(-1)}>← Quay Lại</button>
        <button onClick={() => navigate("/")}>Về Trang Chủ</button>
      </div>
    </div>
  );
}

// Trong Routes config — luôn đặt wildcard route cuối cùng
function App() {
  return (
    <Routes>
      <Route path="/" element={<HomePage />} />
      <Route path="/products" element={<ProductsPage />} />
      <Route path="/products/:id" element={<ProductDetailPage />} />
      {/* Wildcard — bắt tất cả URL không khớp với các route trên */}
      <Route path="*" element={<NotFoundPage />} />
    </Routes>
  );
}
```

### Navigate Component — Redirect (Chuyển Hướng)

```jsx
import { Navigate } from "react-router-dom";

// Chuyển hướng đơn giản
function OldProductPage() {
  return <Navigate to="/products" replace />;
}

// Chuyển hướng có điều kiện
function DashboardPage() {
  const { isLoggedIn } = useAuth();

  if (!isLoggedIn) {
    // replace: true — không thêm entry vào history, không thể nhấn Back về đây
    return <Navigate to="/login" replace state={{ from: "/dashboard" }} />;
  }

  return <DashboardContent />;
}
```

---

## 10. Câu Hỏi Phỏng Vấn

### Câu 1: Sự khác biệt chính giữa React Router v5 và v6?

**Trả lời:**

| Tính Năng | v5 | v6 |
| --------- | -- | -- |
| Bao bọc | `<Switch>` | `<Routes>` |
| Khai báo route | `component={Comp}` | `element={<Comp />}` |
| Chính xác | Cần `exact` prop | Mặc định exact |
| Điều hướng | `useHistory()` | `useNavigate()` |
| Nested routes | Phức tạp, phải đặt trong component con | Đơn giản với `<Outlet>` |
| Relative paths | Không hỗ trợ | Hỗ trợ |
| Data loading | Không có | `loader`, `action` APIs |

### Câu 2: Link vs NavLink vs useNavigate — Khi Nào Dùng Gì?

**Trả lời:**
- **`<Link>`**: Điều hướng thông thường, không cần biết route có đang active không. Dùng trong content, breadcrumbs.
- **`<NavLink>`**: Khi cần highlight (làm nổi bật) link đang active — menu điều hướng, tabs.
- **`useNavigate()`**: Khi điều hướng sau một hành động (sau login, submit form, xử lý logic nghiệp vụ).

### Câu 3: useParams luôn trả về string — Tại sao phải biết điều này?

**Trả lời:**
`useParams()` luôn trả về kiểu `string` cho tất cả params, kể cả URL `/products/42` thì `id` vẫn là `"42"` (string), không phải `42` (number). Điều này gây lỗi khi so sánh `=== 42` hoặc dùng `parseInt` để lấy ID. Cần chuyển đổi kiểu thủ công: `const id = Number(params.id)` hoặc `parseInt(params.id, 10)`.

### Câu 4: useSearchParams khác useParams như thế nào?

**Trả lời:**
- **`useParams`**: Đọc **path params** — phần URL được khai báo trong route pattern (`/products/:id`). Thay đổi path params nghĩa là chuyển sang route khác.
- **`useSearchParams`**: Đọc và **ghi** vào **query string** (phần `?key=value`). Thay đổi search params không thay đổi route, component không unmount — chỉ re-render. Phù hợp cho bộ lọc, phân trang, tìm kiếm.

### Câu 5: Lợi ích của Data APIs (loader/action) so với useEffect?

**Trả lời:**
| | useEffect | loader/action |
| - | --------- | ------------- |
| **Khi fetch** | Sau khi render (waterfall) | Trước khi render (parallel) |
| **Loading state** | Phải tự quản lý | `navigation.state` có sẵn |
| **Error handling** | Phải try/catch thủ công | `errorElement` tự động |
| **Code splitting** | Không liên quan | Loader load song song với code |
| **Progressive enhancement** | Không | Form hoạt động không cần JS |

---

## 🔗 Điều Hướng

- **Tiếp theo:** [2-nested-dynamic-routes.md](./2-nested-dynamic-routes.md) — Nested Routes và Dynamic Params
- **Quay lại:** [README.md](./README.md) — Tổng quan Routing
- **Chỉ mục:** [INDEX.md](../INDEX.md)
