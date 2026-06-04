# 04 — Routing (Điều Hướng Trong React)

> **Routing** (Điều Hướng) là cơ chế cho phép ứng dụng Single Page Application — SPA — Ứng Dụng Một Trang hiển thị các "trang" khác nhau mà không cần tải lại trình duyệt. Phần này bao gồm React Router v6/v7, Protected Routes, và TanStack Router — từ cơ bản đến type-safe routing hiện đại.

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Thiết lập **React Router v6** (Bộ Điều Hướng React phiên bản 6) và cấu hình các route cơ bản
- [ ] Xây dựng **Nested Routes** (Route Lồng Nhau) với `<Outlet>` để tái sử dụng layout
- [ ] Tạo **Dynamic Routes** (Route Động) với URL params để xây dựng trang chi tiết
- [ ] Triển khai **Protected Routes** (Route Được Bảo Vệ) kết hợp authentication — xác thực
- [ ] Hiểu những thay đổi trong **React Router v7** — framework mode vs library mode
- [ ] Sử dụng **TanStack Router** (Bộ Điều Hướng TanStack) với type-safe routing — điều hướng an toàn kiểu dữ liệu
- [ ] Phân biệt và chọn đúng giải pháp routing cho từng loại dự án

---

## 📁 Nội Dung

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-react-router-basics.md](./1-react-router-basics.md) | React Router v6 — cú pháp mới, data APIs | Intermediate |
| [2-nested-dynamic-routes.md](./2-nested-dynamic-routes.md) | Nested Routes, Dynamic Params, Outlet | Intermediate |
| [3-protected-routes.md](./3-protected-routes.md) | Protected Routes, Auth Guards, Redirect | Intermediate |
| [4-react-router-v7.md](./4-react-router-v7.md) | React Router v7 — framework mode, tính năng mới | Intermediate–Advanced |
| [5-tanstack-router.md](./5-tanstack-router.md) | TanStack Router — type-safe, file-based routing | Advanced |

---

## 🗺️ Lộ Trình Học

```
React Router v6 Basics  →  Nested & Dynamic Routes  →  Protected Routes  →  React Router v7  →  TanStack Router
        ↓                          ↓                          ↓                    ↓                  ↓
  "BrowserRouter,            "Outlet, useParams,         "PrivateRoute,       "Framework vs      "Type-safe,
  Routes, Route,              useSearchParams"           Auth Guards"         Library mode"     file-based"
  Link, useNavigate"
```

**Thứ tự khuyến nghị:**
1. Bắt đầu với `1-react-router-basics.md` — nắm vững API cốt lõi của React Router v6
2. Học `2-nested-dynamic-routes.md` — xây dựng layout phức tạp với Outlet
3. Học `3-protected-routes.md` — bảo vệ route với authentication
4. Đọc `4-react-router-v7.md` — hiểu hướng phát triển mới
5. Tham khảo `5-tanstack-router.md` — khi dự án cần type-safety cao

---

## ⏱️ Thời Gian Ước Tính

| Hoạt Động | Thời Gian |
| --------- | --------- |
| Đọc lý thuyết (5 files) | 3–4 giờ |
| Thực hành code theo ví dụ | 2–3 giờ |
| Mini-project: Multi-page App | 2–3 giờ |
| **Tổng cộng** | **7–10 giờ** |

---

## 💡 Routing Là Gì và Tại Sao Cần?

### Trước Khi Có Client-Side Routing (Điều Hướng Phía Máy Khách)

```
User nhấn link
  → Browser gửi request đến server
  → Server trả về HTML mới
  → Browser tải lại toàn bộ trang
  → Trải nghiệm bị gián đoạn (flash, scroll reset)
```

### Với Client-Side Routing (SPA)

```
User nhấn Link component
  → JavaScript cập nhật URL (History API)
  → React render component tương ứng
  → Không có network request — không tải lại trang
  → Trải nghiệm mượt mà như native app
```

### History API — API Lịch Sử Trình Duyệt

React Router v6 sử dụng **History API** của trình duyệt để thay đổi URL mà không reload:

```javascript
// Các phương thức History API
window.history.pushState(state, title, url);    // Thêm entry mới vào history
window.history.replaceState(state, title, url); // Thay thế entry hiện tại
window.history.back();                          // Quay lại trang trước
window.history.forward();                       // Tiến đến trang tiếp theo
```

---

## 📊 So Sánh Các Giải Pháp Routing

| Tiêu Chí | React Router v6 | React Router v7 | TanStack Router | Next.js App Router |
| -------- | --------------- | --------------- | --------------- | ------------------ |
| **Type Safety** (an toàn kiểu) | ❌ Yếu | ❌ Yếu | ✅ Xuất sắc | ✅ Tốt |
| **Bundle Size** (kích thước gói) | ~50 KB | ~50 KB | ~35 KB | N/A (framework) |
| **Data Loading** (tải dữ liệu) | ✅ Loader API | ✅ Loader API | ✅ Tích hợp sẵn | ✅ Server Components |
| **File-based Routing** (điều hướng theo file) | ❌ | ✅ (v7 framework) | ✅ | ✅ |
| **Search Params** (tham số tìm kiếm) | Thủ công | Thủ công | ✅ Type-safe | Thủ công |
| **Nested Layouts** (layout lồng nhau) | ✅ Outlet | ✅ Outlet | ✅ Tích hợp sẵn | ✅ layout.tsx |
| **Phù hợp với** | SPA truyền thống | SPA + SSR | App cần type-safety cao | Full-stack Next.js |
| **Học dễ không?** | ⭐⭐ Trung bình | ⭐⭐ Trung bình | ⭐⭐⭐ Khó hơn | ⭐⭐ Trung bình |

---

## 🔑 Các Khái Niệm Cốt Lõi

### Route — Tuyến Đường

Ánh xạ một **URL pattern** (mẫu URL) sang một **component** cụ thể:

```
URL: /products/123    →   Component: ProductDetailPage
URL: /cart           →   Component: CartPage
URL: /               →   Component: HomePage
```

### Router Types — Các Loại Router

| Router | Cách lưu URL | Dùng khi nào |
| ------ | ------------ | ------------ |
| **BrowserRouter** | `/path/to/page` (sử dụng History API) | Hầu hết ứng dụng web |
| **HashRouter** | `/#/path/to/page` (phần sau `#`) | Deploy trên server không hỗ trợ SPA |
| **MemoryRouter** | Lưu trong bộ nhớ, không thay đổi URL | Testing, React Native |
| **StaticRouter** | Không thay đổi URL | Server-side rendering (SSR) |

### Navigation Hooks — Hook Điều Hướng

```jsx
// Các hook quan trọng trong React Router v6
import {
  useNavigate,      // Điều hướng lập trình (programmatic navigation)
  useParams,        // Đọc dynamic params từ URL (ví dụ: /users/:id)
  useSearchParams,  // Đọc và cập nhật query string (?key=value)
  useLocation,      // Đọc location object (pathname, state, key)
  useMatch,         // Kiểm tra xem route có khớp với URL hiện tại không
} from "react-router-dom";
```

---

## 🔨 Mini-Project: Multi-Page App (Ứng Dụng Nhiều Trang)

Sau khi hoàn thành cả 5 file, hãy xây dựng ứng dụng Blog với đầy đủ routing:

```
✅ Trang chủ (/)
✅ Danh sách bài viết (/posts)
✅ Chi tiết bài viết (/posts/:id)
✅ Trang danh mục (/posts?category=tech)
✅ Dashboard với nested layout (/dashboard/stats, /dashboard/settings)
✅ Trang đăng nhập (/login)
✅ Protected: Trang quản trị (/admin) — chỉ khi đã đăng nhập
✅ Trang 404 (Not Found)
```

**Yêu cầu kỹ thuật:**
1. Dùng `<Outlet>` để tái sử dụng layout header/footer
2. `useSearchParams` để filter bài viết theo danh mục
3. Redirect về `/login` nếu chưa đăng nhập
4. Hiển thị loading state khi chuyển trang

---

## 🔗 Điều Hướng

- **Trước đó:** [03-state-management/](../03-state-management/) — Quản lý trạng thái toàn cục
- **Tiếp theo:** [05-data-fetching/](../05-data-fetching/) — Lấy và đồng bộ dữ liệu
- **Quay lại:** [README.md tổng quan](../README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
