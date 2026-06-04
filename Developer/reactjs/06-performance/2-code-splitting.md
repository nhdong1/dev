# Code Splitting — Tách Code Để Tải Nhanh Hơn

> **Code Splitting** (Tách Code) là kỹ thuật chia nhỏ JavaScript bundle — gói JavaScript — thành nhiều **chunk** (khối) nhỏ hơn và chỉ tải chunk cần thiết cho trang hiện tại. Thay vì tải toàn bộ ứng dụng một lần, người dùng chỉ cần tải những gì họ thực sự dùng ngay lúc đó.

---

## 📌 Mục Lục

1. [Vấn Đề: Bundle Quá Lớn](#1-vấn-đề-bundle-quá-lớn)
2. [Dynamic Import — Nhập Động](#2-dynamic-import--nhập-động)
3. [React.lazy — Lazy Loading Component](#3-reactlazy--lazy-loading-component)
4. [Suspense — Hiển Thị Trạng Thái Loading](#4-suspense--hiển-thị-trạng-thái-loading)
5. [Route-based Code Splitting](#5-route-based-code-splitting)
6. [Component-based Code Splitting](#6-component-based-code-splitting)
7. [Prefetching & Preloading](#7-prefetching--preloading)
8. [Named Chunks và Bundle Analysis](#8-named-chunks-và-bundle-analysis)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề: Bundle Quá Lớn

### Tại Sao Bundle Lớn Gây Vấn Đề?

Trình duyệt phải xử lý JavaScript theo ba bước tốn kém:

```
Download (Tải) → Parse (Phân Tích Cú Pháp) → Execute (Thực Thi)

Bundle 1MB:
- Download: ~2s (mạng 4G trung bình)
- Parse + Execute: ~1-2s (CPU điện thoại tầm trung)
- Tổng: ~3-4s trước khi người dùng thấy gì

→ Ảnh hưởng trực tiếp đến TTI (Time to Interactive — Thời Gian Đến Khi Có Thể Tương Tác)
```

### Ứng Dụng React Điển Hình Không Tối Ưu

```
myapp-bundle.js (2.5MB)
├── React core (~150KB)
├── React DOM (~100KB)
├── Redux Toolkit (~50KB)
├── TanStack Query (~50KB)
├── Chart.js (~300KB) ← Chỉ dùng ở trang Analytics
├── Rich text editor (~500KB) ← Chỉ dùng khi tạo bài viết
├── PDF viewer (~400KB) ← Chỉ dùng khi xem hóa đơn
├── Date picker library (~100KB)
└── Tất cả routes và components (~900KB)

→ Người dùng trang chủ phải tải 2.5MB, nhưng chỉ cần ~300KB
```

### Sau Khi Áp Dụng Code Splitting

```
initial-bundle.js (~300KB) ← Tải ngay lập tức
analytics-chunk.js (~350KB) ← Tải khi vào trang Analytics
editor-chunk.js (~550KB) ← Tải khi tạo bài viết
invoice-chunk.js (~420KB) ← Tải khi xem hóa đơn
...

→ Người dùng trang chủ chỉ cần tải 300KB → trang hiển thị 3x nhanh hơn
```

---

## 2. Dynamic Import — Nhập Động

**Dynamic Import** (Nhập Động) là tính năng JavaScript cho phép nhập module theo yêu cầu, thay vì nhập tất cả lúc đầu:

### Static Import vs Dynamic Import

```javascript
// Static import — Nhập Tĩnh (truyền thống)
// Tất cả module được tải và bundle vào initial chunk
import ChartLibrary from "chart.js"; // Luôn tải dù người dùng có vào trang chart không
import PDFViewer from "./PDFViewer";
import RichTextEditor from "./RichTextEditor";

// Dynamic import — Nhập Động (code splitting)
// Module chỉ được tải khi cần → trả về Promise
const ChartLibrary = await import("chart.js");
const module = await import("./PDFViewer");
const { PDFViewer } = module;
```

### Dynamic Import Trong Vanilla JavaScript

```javascript
// Tải module khi người dùng nhấn button
async function handleExportPDF() {
  // Chỉ tải thư viện PDF khi cần
  const { default: jsPDF } = await import("jspdf");
  const doc = new jsPDF();
  doc.text("Hello World", 10, 10);
  doc.save("document.pdf");
}

// Hoặc tải dựa trên điều kiện
async function loadLocale(locale) {
  // Tải file ngôn ngữ tương ứng
  const translations = await import(`./locales/${locale}.json`);
  i18n.setTranslations(translations.default);
}
```

### Webpack Magic Comments — Chú Thích Đặc Biệt Cho Webpack

```javascript
// Đặt tên cho chunk — giúp debug dễ hơn
const module = await import(
  /* webpackChunkName: "pdf-viewer" */
  "./PDFViewer"
);

// Prefetch — Tải trước sau khi trang chính đã tải xong (low priority)
const module = await import(
  /* webpackPrefetch: true */
  /* webpackChunkName: "analytics-dashboard" */
  "./AnalyticsDashboard"
);

// Preload — Tải ngay song song với trang chính (high priority)
const module = await import(
  /* webpackPreload: true */
  /* webpackChunkName: "critical-component" */
  "./CriticalComponent"
);
```

---

## 3. React.lazy — Lazy Loading Component

`React.lazy()` cho phép bạn render một component được import động như thể đó là component bình thường:

### Cú Pháp Cơ Bản

```jsx
import React, { lazy, Suspense } from "react";

// Static import — tải ngay khi app khởi động
import HeavyComponent from "./HeavyComponent"; // ❌ Luôn tải

// Lazy import — chỉ tải khi component được render lần đầu
const HeavyComponent = lazy(() => import("./HeavyComponent")); // ✅

// Default export — export mặc định (bắt buộc cho React.lazy)
// File HeavyComponent.jsx phải có: export default function HeavyComponent() {...}
```

### Yêu Cầu Bắt Buộc

```jsx
// ✅ Đúng: Default export
// HeavyComponent.jsx
export default function HeavyComponent() {
  return <div>Heavy content</div>;
}

// ✅ Đúng: Named export được re-export thành default
const AnalyticsDashboard = lazy(() =>
  import("./AnalyticsDashboard").then(module => ({
    default: module.AnalyticsDashboard // Named export → re-wrap thành default
  }))
);

// ❌ Sai: Named export trực tiếp không được hỗ trợ
const HeavyComponent = lazy(() => import("./file").then(m => m.namedExport));
// Cách này không hoạt động vì React.lazy cần object có key "default"
```

---

## 4. Suspense — Hiển Thị Trạng Thái Loading

`React.lazy` **bắt buộc** phải đặt trong `<Suspense>` — `Suspense` hiển thị **fallback** (nội dung dự phòng) trong khi component đang tải:

### Cú Pháp Cơ Bản

```jsx
import { lazy, Suspense } from "react";

const HeavyChart = lazy(() => import("./HeavyChart"));

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      {/* Suspense bao quanh component lazy */}
      <Suspense fallback={<div>Đang tải biểu đồ...</div>}>
        <HeavyChart />
      </Suspense>
    </div>
  );
}
```

### Fallback Đẹp Hơn Với Loading Skeleton

```jsx
// Skeleton — Bộ Khung Tải — giúp tránh layout shift (dịch chuyển layout)
function ChartSkeleton() {
  return (
    <div className="chart-skeleton">
      <div className="skeleton-header animate-pulse" />
      <div className="skeleton-body animate-pulse" />
      <div className="skeleton-footer animate-pulse" />
    </div>
  );
}

const HeavyChart = lazy(() => import("./HeavyChart"));

function Dashboard() {
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <HeavyChart data={chartData} />
    </Suspense>
  );
}
```

### Nested Suspense — Suspense Lồng Nhau

```jsx
// Suspense lồng nhau cho phép fallback chi tiết hơn
function App() {
  return (
    // Suspense ngoài: toàn bộ app
    <Suspense fallback={<AppLoadingScreen />}>
      <Header />
      <main>
        {/* Suspense trong: chỉ cho phần content */}
        <Suspense fallback={<ContentSkeleton />}>
          <MainContent />
        </Suspense>
        {/* Suspense riêng cho sidebar */}
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />
        </Suspense>
      </main>
    </Suspense>
  );
}
```

### Error Boundary Kết Hợp Suspense

```jsx
import { Component } from "react";

// Error Boundary — Ranh Giới Bắt Lỗi — bắt lỗi khi lazy load thất bại
class LazyErrorBoundary extends Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error("Lỗi khi tải component:", error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <p>Không thể tải component. Vui lòng thử lại.</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Thử lại
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Sử dụng kết hợp
const HeavyChart = lazy(() => import("./HeavyChart"));

function Dashboard() {
  return (
    <LazyErrorBoundary>
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart />
      </Suspense>
    </LazyErrorBoundary>
  );
}
```

---

## 5. Route-based Code Splitting

**Route-based Code Splitting** (Tách Code Theo Route) là chiến lược phổ biến nhất: mỗi route (trang) là một chunk riêng.

### Với React Router v6

```jsx
import { lazy, Suspense } from "react";
import { BrowserRouter, Routes, Route } from "react-router-dom";

// Lazy load từng trang
const HomePage = lazy(() => import("./pages/HomePage"));
const ProductsPage = lazy(() => import("./pages/ProductsPage"));
const ProductDetailPage = lazy(() => import("./pages/ProductDetailPage"));
const CartPage = lazy(() => import("./pages/CartPage"));
const CheckoutPage = lazy(() => import("./pages/CheckoutPage"));
const AdminDashboard = lazy(() => import("./pages/AdminDashboard"));

// Loading component dùng chung cho tất cả routes
function PageLoadingFallback() {
  return (
    <div className="flex items-center justify-center h-screen">
      <div className="animate-spin rounded-full h-16 w-16 border-t-4 border-blue-500" />
    </div>
  );
}

function App() {
  return (
    <BrowserRouter>
      <Header />
      {/* Một Suspense bao tất cả routes */}
      <Suspense fallback={<PageLoadingFallback />}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/products" element={<ProductsPage />} />
          <Route path="/products/:id" element={<ProductDetailPage />} />
          <Route path="/cart" element={<CartPage />} />
          <Route path="/checkout" element={<CheckoutPage />} />
          <Route path="/admin/*" element={<AdminDashboard />} />
        </Routes>
      </Suspense>
      <Footer />
    </BrowserRouter>
  );
}
```

### Với Next.js (Tự Động Code Splitting)

```
Next.js tự động code split theo route — mỗi file trong app/ hoặc pages/
tương ứng với một chunk riêng. Không cần React.lazy!

app/
├── page.tsx           → chunk: home
├── products/
│   ├── page.tsx       → chunk: products-list
│   └── [id]/
│       └── page.tsx   → chunk: product-detail
├── cart/
│   └── page.tsx       → chunk: cart
└── admin/
    └── page.tsx       → chunk: admin
```

```jsx
// next/dynamic — Dynamic Import dành riêng cho Next.js
import dynamic from "next/dynamic";

// Thay thế React.lazy trong Next.js
const HeavyChart = dynamic(() => import("./HeavyChart"), {
  loading: () => <ChartSkeleton />, // Tương đương Suspense fallback
  ssr: false, // Không render trên server (dành cho component chỉ dùng browser APIs)
});

function AnalyticsPage() {
  return (
    <div>
      <h1>Analytics</h1>
      <HeavyChart /> {/* Tải khi component render, không cần Suspense */}
    </div>
  );
}
```

---

## 6. Component-based Code Splitting

### Lazy Load Modal, Drawer, Dialog

```jsx
const UserEditModal = lazy(() => import("./UserEditModal"));
const DeleteConfirmDialog = lazy(() => import("./DeleteConfirmDialog"));

function UserTable({ users }) {
  const [editingUser, setEditingUser] = useState(null);
  const [deletingUserId, setDeletingUserId] = useState(null);

  return (
    <div>
      <table>
        {users.map(user => (
          <tr key={user.id}>
            <td>{user.name}</td>
            <td>
              <button onClick={() => setEditingUser(user)}>Sửa</button>
              <button onClick={() => setDeletingUserId(user.id)}>Xóa</button>
            </td>
          </tr>
        ))}
      </table>

      {/* Modal chỉ tải khi được mở */}
      {editingUser && (
        <Suspense fallback={<ModalSkeleton />}>
          <UserEditModal
            user={editingUser}
            onClose={() => setEditingUser(null)}
          />
        </Suspense>
      )}

      {deletingUserId && (
        <Suspense fallback={null}> {/* null = không hiện fallback */}
          <DeleteConfirmDialog
            userId={deletingUserId}
            onConfirm={handleDelete}
            onCancel={() => setDeletingUserId(null)}
          />
        </Suspense>
      )}
    </div>
  );
}
```

### Lazy Load Thư Viện Nặng

```jsx
// Tải Chart.js chỉ khi component Chart được render
const SalesChart = lazy(() =>
  import("./SalesChart") // SalesChart.jsx import Chart.js bên trong
);

// Tải Rich Text Editor chỉ khi cần
const RichTextEditor = lazy(() =>
  import("./RichTextEditor") // RichTextEditor.jsx import Quill/TipTap bên trong
);

// Tải Map component chỉ khi cần (leaflet là thư viện nặng)
const MapView = lazy(() =>
  import("./MapView") // MapView.jsx import Leaflet bên trong
);

function ContentEditor({ type }) {
  return (
    <div>
      {type === "chart" && (
        <Suspense fallback={<ChartSkeleton />}>
          <SalesChart />
        </Suspense>
      )}
      {type === "article" && (
        <Suspense fallback={<EditorSkeleton />}>
          <RichTextEditor />
        </Suspense>
      )}
      {type === "map" && (
        <Suspense fallback={<MapSkeleton />}>
          <MapView />
        </Suspense>
      )}
    </div>
  );
}
```

### Lazy Load Dựa Trên Điều Kiện (Feature Flag)

```jsx
// Tải component A/B test — chỉ tải variant được assign
function FeatureFlag({ flag, variant }) {
  const VariantComponent = useMemo(() => {
    if (flag === "new-checkout" && variant === "B") {
      return lazy(() => import("./NewCheckoutFlow"));
    }
    return lazy(() => import("./LegacyCheckoutFlow"));
  }, [flag, variant]);

  return (
    <Suspense fallback={<CheckoutSkeleton />}>
      <VariantComponent />
    </Suspense>
  );
}
```

---

## 7. Prefetching & Preloading

**Prefetching** (Tải Trước Ưu Tiên Thấp) và **Preloading** (Tải Trước Ưu Tiên Cao) giúp tải chunk trước khi người dùng thực sự cần:

### Prefetch Khi Hover

```jsx
// Tải chunk khi người dùng hover vào link → sẵn sàng ngay khi click
function NavLink({ to, children }) {
  const prefetchRef = useRef(false);

  const handleMouseEnter = () => {
    if (prefetchRef.current) return;
    prefetchRef.current = true;

    // Trigger dynamic import để tải chunk về browser cache
    if (to === "/analytics") {
      import("./pages/AnalyticsPage");
    } else if (to === "/reports") {
      import("./pages/ReportsPage");
    }
  };

  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  );
}
```

### Prefetch Theo Xác Suất Điều Hướng

```jsx
// Người dùng đang ở trang ProductList → có 70% khả năng vào ProductDetail
// → Prefetch ProductDetail chunk sẵn
function ProductList({ products }) {
  const navigate = useNavigate();

  // Prefetch ProductDetail ngay khi ProductList mount
  useEffect(() => {
    const timer = setTimeout(() => {
      import("./pages/ProductDetailPage");
    }, 2000); // Delay 2s để không cạnh tranh bandwidth với trang hiện tại

    return () => clearTimeout(timer);
  }, []);

  return (
    <ul>
      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onClick={() => navigate(`/products/${product.id}`)}
        />
      ))}
    </ul>
  );
}
```

### Vite-specific: Import Meta Glob

```javascript
// Vite cho phép import nhiều module động với glob pattern — mẫu glob
// Hữu ích cho i18n (internationalization — đa ngôn ngữ), themes, plugins

// Tải tất cả file ngôn ngữ nhưng lazy
const locales = import.meta.glob("./locales/*.json");

async function loadLocale(locale) {
  const loader = locales[`./locales/${locale}.json`];
  if (!loader) throw new Error(`Locale không tồn tại: ${locale}`);
  const module = await loader();
  return module.default;
}

// Eager loading (tải ngay) cho một số locale cơ bản
const eagerLocales = import.meta.glob("./locales/core/*.json", { eager: true });
```

---

## 8. Named Chunks và Bundle Analysis

### Đặt Tên Chunk Để Debug

```javascript
// webpack/Vite cho phép đặt tên chunk qua magic comment
const AnalyticsDashboard = lazy(() =>
  import(
    /* webpackChunkName: "analytics" */
    "./pages/AnalyticsDashboard"
  )
);

// Vite — dùng @vite-ignore hoặc rollupOptions
// vite.config.js
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Tách vendor chunk — chunk thư viện bên thứ ba
          "vendor-react": ["react", "react-dom"],
          "vendor-router": ["react-router-dom"],
          "vendor-query": ["@tanstack/react-query"],
          "vendor-charts": ["chart.js", "recharts"],
        }
      }
    }
  }
});
```

### Bundle Analyzer — Công Cụ Phân Tích Bundle

```bash
# Webpack Bundle Analyzer
npm install --save-dev webpack-bundle-analyzer
npx webpack --profile --json > stats.json
npx webpack-bundle-analyzer stats.json

# Rollup Plugin Visualizer (dùng với Vite)
npm install --save-dev rollup-plugin-visualizer
```

```javascript
// vite.config.js — thêm plugin visualizer
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      open: true, // Tự động mở report sau khi build
      gzipSize: true, // Hiển thị kích thước sau gzip
      brotliSize: true, // Hiển thị kích thước sau brotli
      filename: "bundle-report.html",
    }),
  ],
});
```

```bash
# Next.js Bundle Analyzer
npm install @next/bundle-analyzer

# next.config.js
const withBundleAnalyzer = require("@next/bundle-analyzer")({
  enabled: process.env.ANALYZE === "true",
});
module.exports = withBundleAnalyzer({});

# Chạy phân tích
ANALYZE=true npm run build
```

---

## 9. Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Code Splitting là gì và tại sao cần thiết?**

A: Code Splitting là kỹ thuật chia JavaScript bundle thành nhiều chunk nhỏ hơn và chỉ tải chunk cần thiết. Cần thiết vì:
- Giảm initial bundle size → giảm TTI (Time to Interactive)
- Người dùng không phải tải code của các trang họ không ghé thăm
- Cải thiện trực tiếp Core Web Vitals (LCP, FID, INP)

---

**Q: React.lazy và dynamic import khác nhau như thế nào?**

A:
- `dynamic import` — tính năng JavaScript thuần, trả về Promise, dùng được mọi nơi
- `React.lazy` — wrapper của React, bọc dynamic import, chỉ dùng được với component, bắt buộc dùng với `<Suspense>`

`React.lazy(() => import("./Component"))` = React biết cách render component từ Promise của dynamic import.

---

**Q: Suspense fallback nên là gì?**

A: Phụ thuộc vào context:
- Cho toàn trang: Full-page loading screen
- Cho section riêng: **Skeleton** (bộ khung placeholder) giữ nguyên layout → tránh **CLS** (Cumulative Layout Shift — Độ Dịch Chuyển Layout Tích Lũy)
- Cho modal/overlay: Loading spinner nhỏ
- Cho element nhỏ: `null` hoặc spinner tối giản

Tránh dùng text "Loading..." đơn thuần — gây CLS và trải nghiệm kém.

---

**Q: Tại sao React.lazy yêu cầu default export?**

A: `React.lazy` mong đợi Promise resolve thành một object có key `default` chứa React component. Default export trong ES modules chính xác là cấu trúc này. Nếu component dùng named export, cần re-wrap:

```jsx
const MyComponent = lazy(() =>
  import("./module").then(m => ({ default: m.NamedExport }))
);
```

---

### Câu Hỏi Nâng Cao

**Q: Route-based và component-based code splitting — khi nào dùng gì?**

A:
- **Route-based**: Luôn áp dụng — đây là "quick win" lớn nhất, ít effort nhất
- **Component-based**: Khi component chứa thư viện nặng (Chart.js, Monaco Editor, PDF viewer, Map library) và không phải lúc nào cũng hiển thị (trong modal, tab, hoặc dựa vào feature flag)

Chiến lược: Bắt đầu với route-based → dùng Bundle Analyzer để xác định vendor nặng → áp dụng component-based có chọn lọc.

---

**Q: Làm thế nào để xử lý lỗi khi lazy load thất bại (mạng yếu)?**

A: Dùng **Error Boundary** (Ranh Giới Bắt Lỗi) bao quanh `<Suspense>`:

```jsx
function LazyRoute({ component: Component }) {
  return (
    <ErrorBoundary
      fallback={({ error, resetError }) => (
        <div>
          <p>Tải trang thất bại. Vui lòng kiểm tra kết nối.</p>
          <button onClick={resetError}>Thử lại</button>
        </div>
      )}
    >
      <Suspense fallback={<PageSkeleton />}>
        <Component />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

**Q: Giải thích sự khác biệt giữa prefetch và preload.**

A:
- **Preload** (`<link rel="preload">`): Ưu tiên cao, tải song song với tài nguyên hiện tại. Dùng cho tài nguyên **chắc chắn** cần ngay trong trang hiện tại.
- **Prefetch** (`<link rel="prefetch">`): Ưu tiên thấp, tải sau khi trang hiện tại đã tải xong, dùng bandwidth nhàn rỗi. Dùng cho tài nguyên **có thể** cần ở trang tiếp theo.

Trong context Code Splitting: `webpackPrefetch` cho routes người dùng có thể đến tiếp, `webpackPreload` cho resources cần ngay ở trang hiện tại.

---

## 🔗 Điều Hướng

- **Trước đó:** [1-memoization.md](./1-memoization.md) — React.memo, useMemo, useCallback
- **Tiếp theo:** [3-virtualization.md](./3-virtualization.md) — Virtualize danh sách dài
- **Liên quan:** [4-concurrent-features.md](./4-concurrent-features.md) — Suspense với Concurrent Mode
- **Chỉ mục:** [INDEX.md](../INDEX.md)
