# Profiling — Đo Lường Và Tìm Bottleneck Hiệu Năng

> **Profiling** (Phân Tích Hiệu Năng) là quá trình đo lường và xác định các điểm nghẽn cổ chai — bottleneck — trong ứng dụng React. Nguyên tắc cốt lõi: **"Đo lường trước, tối ưu sau"** — không đoán mò, không tối ưu mù quáng. Chỉ tối ưu những gì bạn đã đo được.

---

## 📌 Mục Lục

1. [Tại Sao Phải Đo Lường Trước?](#1-tại-sao-phải-đo-lường-trước)
2. [React DevTools Profiler](#2-react-devtools-profiler)
3. [Chrome Performance Panel](#3-chrome-performance-panel)
4. [Web Vitals — Chỉ Số Trải Nghiệm Web](#4-web-vitals--chỉ-số-trải-nghiệm-web)
5. [Profiler API — Đo Lường Trong Code](#5-profiler-api--đo-lường-trong-code)
6. [Performance API — Web API Đo Lường](#6-performance-api--web-api-đo-lường)
7. [Lighthouse — Kiểm Tra Chất Lượng Tổng Thể](#7-lighthouse--kiểm-tra-chất-lượng-tổng-thể)
8. [Quy Trình Debug Hiệu Năng](#8-quy-trình-debug-hiệu-năng)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Phải Đo Lường Trước?

### Anti-pattern — Mẫu Chống Hiệu Quả: Tối Ưu Đoán Mò

```jsx
// ❌ Developer đoán "đây có vẻ chậm" và thêm memo vào mọi thứ
const Button = React.memo(({ onClick, children }) => (
  <button onClick={onClick}>{children}</button>
));
// Button rất đơn giản, re-render chỉ tốn < 0.01ms
// useMemo và React.memo tạo overhead cao hơn lợi ích!

// ❌ Developer thêm useMemo "cho chắc"
const label = useMemo(() => `Hello ${name}`, [name]);
// Template literal đơn giản, không cần memo
```

### Quy Trình Đúng

```
1. Xác định symptom (triệu chứng):
   → "Khi tìm kiếm, UI bị lag 300ms"
   → "Scroll trong danh sách sản phẩm bị giật"
   → "Chuyển tab mất 500ms"

2. Đo lường baseline (cơ sở) với Profiler

3. Xác định root cause (nguyên nhân gốc rễ):
   → Component nào render lâu?
   → Component nào render nhiều lần không cần thiết?
   → Tính toán nào chiếm nhiều CPU?

4. Áp dụng giải pháp phù hợp

5. Đo lường lại → so sánh với baseline
```

---

## 2. React DevTools Profiler

### Cài Đặt

```
Chrome/Firefox Extension: "React Developer Tools"
Tải tại: https://react.dev/learn/react-developer-tools

Sau khi cài → mở DevTools (F12) → Tab "Components" và "Profiler"
```

### Workflow Cơ Bản

```
Bước 1: Mở DevTools → Tab "Profiler"
Bước 2: Nhấn nút Record (hình tròn đỏ)
Bước 3: Thực hiện hành động cần profile (gõ tìm kiếm, click tab, scroll...)
Bước 4: Nhấn Stop
Bước 5: Phân tích kết quả
```

### Đọc Flame Graph — Biểu Đồ Lửa

```
Flame Graph (Biểu Đồ Lửa) hiển thị:
- Trục ngang: thứ tự trong component tree (không phải thời gian)
- Trục dọc: độ sâu của component tree (call stack)
- Màu sắc:
  → Màu vàng/cam đậm: tốn nhiều thời gian render → cần chú ý
  → Màu vàng nhạt: render bình thường
  → Màu xám: component đã được memo hóa, không re-render

Ví dụ Flame Graph:
┌─────────────────────────────────────────────────┐
│  App (3ms)                                      │  ← Root
├──────────────┬──────────────────────────────────┤
│ Header (0.1ms)│ MainContent (2.8ms)             │  ← Level 1
├──────────────┴───────┬──────────────────────────┤
│  Sidebar (0.2ms)     │  ProductList (2.5ms) 🔴  │  ← Level 2 — đây là vấn đề!
├──────────────────────┴──────────────────────────┤
│  ProductCard × 100 (2.4ms total)               │  ← Level 3
└─────────────────────────────────────────────────┘

→ ProductList là bottleneck: chiếm 2.5ms / 3ms tổng thời gian
```

### Ranked Chart — Biểu Đồ Xếp Hạng

```
Ranked Chart sắp xếp component theo thời gian render giảm dần:

1. ProductList     ████████████████ 2.5ms  ← Ưu tiên tối ưu nhất
2. FilterBar       ████ 0.8ms
3. SearchInput     ██ 0.3ms
4. Header          █ 0.1ms
5. Sidebar         █ 0.1ms
```

### "Why Did This Render?" — Tại Sao Re-render?

```
Khi click vào một component trong Profiler, React DevTools hiển thị:

Component: ProductList
Render duration: 2.5ms
Render #: 3 (lần render thứ 3 trong session)

Why did this render?
→ Props changed: "products" (so sánh object reference)

→ Điều này cho biết: products prop có reference mới dù có thể nội dung không đổi
→ Giải pháp: useMemo cho products hoặc React.memo với custom comparison
```

### Bật "Record why each component rendered"

```
Profiler Settings → Tick "Record why each component rendered"
→ Bây giờ Profiler sẽ ghi lại chính xác lý do mỗi component re-render
→ Thông tin: "Props changed", "State changed", "Context changed", "Hooks changed"
```

### Cấu Hình Profiler Nâng Cao

```
Profiler Settings:
☑ Highlight updates when components render
  → Màu xanh lá xuất hiện xung quanh component đang re-render
  → Rất hữu ích để xem re-render trực quan mà không cần record

☑ Hide commits below (ms): 2
  → Ẩn các commit quá nhanh (< 2ms) → tập trung vào commit chậm

☑ Record why each component rendered
  → Ghi lại nguyên nhân re-render chi tiết
```

---

## 3. Chrome Performance Panel

### Khi Nào Dùng Performance Panel?

React DevTools Profiler đo React layer — còn **Chrome Performance Panel** (Bảng Hiệu Năng Chrome) đo ở mức browser thấp hơn:
- Scripting (thực thi JavaScript)
- Rendering (layout, paint, composite)
- Memory usage
- Frame rate (FPS — Khung Hình Mỗi Giây)

### Workflow

```
F12 → Tab "Performance" → Record → Thực hiện hành động → Stop

Đọc kết quả:
┌─────────────────────────────────────────────────────────────────┐
│ FPS: ▇▇▇▇▇▇▃▃▁▁▁▃▃▇▇▇  ← Chỗ thấp = lag (dưới 30fps)         │
│ CPU: ░░▓▓████████▓▓░░  ← CPU spike = JavaScript nặng           │
│                                                                 │
│ Main Thread (Luồng Chính):                                      │
│ [Task] [Task] [Long Task 😱 300ms] [Task] [Task]               │
│                                                                 │
│ Long Task: khối > 50ms → block main thread → UI không responsive│
└─────────────────────────────────────────────────────────────────┘
```

### Long Tasks — Khối Tác Vụ Dài

```
Bất kỳ tác vụ > 50ms đều là "Long Task"
→ Trong khi Long Task đang chạy, browser KHÔNG THỂ:
  - Xử lý input của người dùng
  - Paint frame mới
  - Chạy animation

→ Tác vụ 300ms = UI đóng băng 300ms = trải nghiệm tệ

Nguyên nhân phổ biến:
- Render list lớn không virtualize
- Tính toán nặng trong render function
- Heavy JSON parsing
- Synchronous operations trong event handlers
```

### Đọc Flame Chart Trong Performance Panel

```
Main Thread Flame Chart:
┌──────────────────────────────────────────────────────────────┐
│ Event: click                                                │
│   └─ React setState                                         │
│         └─ React render (reconciliation)                    │
│               └─ ProductList render (200ms!) 🔴             │
│                     └─ × 500 ProductCard renders            │
│                           └─ DOM update                     │
└──────────────────────────────────────────────────────────────┘

→ ProductList render 200ms → Long Task → cần virtualization
```

---

## 4. Web Vitals — Chỉ Số Trải Nghiệm Web

### Core Web Vitals — Chỉ Số Cốt Lõi Của Google

```
LCP — Largest Contentful Paint — Thời Gian Hiển Thị Nội Dung Lớn Nhất:
→ Đo: Khi phần tử lớn nhất (ảnh hero, heading chính) xuất hiện
→ Tốt: < 2.5s | Cần cải thiện: 2.5-4s | Kém: > 4s
→ Cải thiện: Lazy load ảnh, Code splitting, Server-side rendering

FID — First Input Delay — Độ Trễ Tương Tác Đầu Tiên (đã deprecated, thay bằng INP):
→ Đo: Thời gian từ khi user click lần đầu đến khi browser xử lý

INP — Interaction to Next Paint — Thời Gian Từ Tương Tác Đến Vẽ Khung Tiếp Theo:
→ Đo: 99th percentile của thời gian phản hồi tất cả interactions
→ Tốt: < 200ms | Cần cải thiện: 200-500ms | Kém: > 500ms
→ Cải thiện: startTransition, chia nhỏ Long Tasks, giảm re-renders

CLS — Cumulative Layout Shift — Độ Dịch Chuyển Layout Tích Lũy:
→ Đo: Tổng mức độ layout bị xáo trộn không mong đợi
→ Tốt: < 0.1 | Cần cải thiện: 0.1-0.25 | Kém: > 0.25
→ Cải thiện: Skeleton với fixed dimensions, không insert content đột ngột
```

### Đo Web Vitals Trong Code

```javascript
import { onCLS, onFID, onLCP, onINP, onFCP, onTTFB } from "web-vitals";

function sendToAnalytics(metric) {
  // Gửi lên analytics service (Google Analytics, Datadog, Sentry...)
  console.log({
    name: metric.name,
    value: metric.value,
    rating: metric.rating, // "good" | "needs-improvement" | "poor"
    id: metric.id,
  });
}

// Đăng ký tất cả Web Vitals
onCLS(sendToAnalytics);
onFID(sendToAnalytics);
onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

```javascript
// Tích hợp với Google Analytics 4
import { onLCP, onINP, onCLS } from "web-vitals";

function sendToGA4({ name, value, rating, id }) {
  window.gtag("event", name, {
    value: Math.round(name === "CLS" ? value * 1000 : value),
    metric_id: id,
    metric_value: value,
    metric_rating: rating,
  });
}

onLCP(sendToGA4);
onINP(sendToGA4);
onCLS(sendToGA4);
```

---

## 5. Profiler API — Đo Lường Trong Code

React cung cấp `<Profiler>` component để đo lường theo chương trình:

### Cú Pháp

```jsx
import { Profiler } from "react";

function onRenderCallback(
  id,          // "id" prop của Profiler component
  phase,       // "mount" (lần đầu) hoặc "update" (các lần sau)
  actualDuration, // Thời gian render thực tế (ms)
  baseDuration,   // Thời gian render ước tính không có memo (ms)
  startTime,      // Thời điểm bắt đầu render
  commitTime      // Thời điểm commit lên DOM
) {
  if (actualDuration > 16) { // > 16ms → dưới 60fps
    console.warn(`${id} render chậm: ${actualDuration.toFixed(2)}ms`);
  }
}

// Bọc component cần đo lường
function App() {
  return (
    <Profiler id="ProductList" onRender={onRenderCallback}>
      <ProductList products={products} />
    </Profiler>
  );
}
```

### Gửi Metrics Lên Server

```jsx
// Gửi performance data lên monitoring service — dịch vụ giám sát
function createPerformanceReporter(endpoint) {
  const buffer = []; // Buffer để batch gửi
  let flushTimer = null;

  return function onRender(id, phase, actualDuration) {
    buffer.push({
      component: id,
      phase,
      duration: actualDuration,
      timestamp: Date.now(),
      url: window.location.href,
    });

    // Batch flush sau 5s — tránh gửi quá nhiều request
    clearTimeout(flushTimer);
    flushTimer = setTimeout(() => {
      if (buffer.length === 0) return;
      const payload = [...buffer];
      buffer.length = 0;

      fetch(endpoint, {
        method: "POST",
        body: JSON.stringify(payload),
        headers: { "Content-Type": "application/json" },
      });
    }, 5000);
  };
}

const reporter = createPerformanceReporter("/api/metrics");

function App() {
  return (
    <Profiler id="App" onRender={reporter}>
      <Router />
    </Profiler>
  );
}
```

### Profiler Chỉ Trong Development

```jsx
// Profiler có chi phí nhỏ → chỉ nên dùng trong development/staging
// KHÔNG dùng trong production trừ khi cần monitor có chủ đích

const ProfilerWrapper = process.env.NODE_ENV === "development"
  ? ({ id, children }) => (
      <Profiler id={id} onRender={(id, phase, duration) => {
        if (duration > 16) {
          console.warn(`⚠️ ${id}: ${duration.toFixed(1)}ms (${phase})`);
        }
      }}>
        {children}
      </Profiler>
    )
  : ({ children }) => children; // Production: không wrap gì

function SlowComponent() {
  return (
    <ProfilerWrapper id="SlowComponent">
      <ActualComponent />
    </ProfilerWrapper>
  );
}
```

---

## 6. Performance API — Web API Đo Lường

Browser cung cấp **Performance API** để đo lường chính xác:

### performance.mark và performance.measure

```javascript
// Đo lường thời gian thực thi của bất kỳ đoạn code nào
function measureOperation(label, fn) {
  performance.mark(`${label}-start`);
  const result = fn();
  performance.mark(`${label}-end`);

  performance.measure(label, `${label}-start`, `${label}-end`);
  const entry = performance.getEntriesByName(label)[0];
  console.log(`${label}: ${entry.duration.toFixed(2)}ms`);

  return result;
}

// Sử dụng
const filteredProducts = measureOperation("filter-products", () =>
  products.filter(p => p.price < maxPrice)
);
// Log: "filter-products: 45.23ms"
```

### React useEffect + Performance API

```jsx
function useRenderTimer(componentName) {
  const renderStart = useRef(performance.now());

  // Sau mỗi render
  useEffect(() => {
    const renderDuration = performance.now() - renderStart.current;
    if (renderDuration > 16) {
      console.warn(`${componentName} render: ${renderDuration.toFixed(1)}ms`);
    }
    renderStart.current = performance.now(); // Reset cho lần render tiếp
  });
}

function SlowComponent({ data }) {
  useRenderTimer("SlowComponent");
  // ... render logic
}
```

### Theo Dõi Memory — Bộ Nhớ

```javascript
// Chỉ hoạt động trong Chrome với flag enabled
function logMemoryUsage(label) {
  if (!performance.memory) {
    console.log("Memory API không khả dụng (chỉ có trên Chrome)");
    return;
  }

  const { usedJSHeapSize, totalJSHeapSize, jsHeapSizeLimit } = performance.memory;
  console.log(`[${label}] Memory:`, {
    used: `${(usedJSHeapSize / 1024 / 1024).toFixed(1)}MB`,
    total: `${(totalJSHeapSize / 1024 / 1024).toFixed(1)}MB`,
    limit: `${(jsHeapSizeLimit / 1024 / 1024).toFixed(1)}MB`,
  });
}

// Gọi trước và sau một thao tác để kiểm tra memory leak
logMemoryUsage("Before render");
renderHeavyComponent();
logMemoryUsage("After render");
```

---

## 7. Lighthouse — Kiểm Tra Chất Lượng Tổng Thể

**Lighthouse** là công cụ tự động kiểm tra Performance, Accessibility — Khả Năng Tiếp Cận, SEO, và Best Practices:

### Chạy Lighthouse

```bash
# Qua CLI (Command Line Interface — Giao Diện Dòng Lệnh)
npm install -g lighthouse
lighthouse https://your-app.com --output html --output-path report.html --view

# Hoặc: Chrome DevTools → Lighthouse tab → Analyze page load
# Hoặc: PageSpeed Insights tại https://pagespeed.web.dev/
```

### Đọc Kết Quả Lighthouse

```
Performance Score: 67 🟡 (0-49: đỏ, 50-89: vàng, 90-100: xanh)

Diagnostics (Chẩn đoán):
❌ Largest Contentful Paint: 4.5s (cần < 2.5s)
   → Fix: Tối ưu ảnh, sử dụng CDN, preload LCP resource

❌ Total Blocking Time: 850ms (cần < 200ms)
   → Fix: Code splitting, giảm Long Tasks, defer non-critical JS

⚠️ Cumulative Layout Shift: 0.12 (cần < 0.1)
   → Fix: Thêm explicit width/height cho ảnh, dùng Skeleton

✅ First Contentful Paint: 1.2s

Opportunities (Cơ Hội Cải Thiện):
• Eliminate render-blocking resources: 650ms
  → Defer loading của fonts, non-critical CSS
• Properly size images: savings 240KB
  → Sử dụng next-gen formats (WebP, AVIF)
• Unused JavaScript: 280KB
  → Code splitting, tree-shaking
```

### Lighthouse CI — Tích Hợp Vào CI/CD

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on: [push]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3

      - name: Build
        run: npm ci && npm run build

      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli@latest
          lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      startServerCommand: "npm run preview",
      url: ["http://localhost:4173/", "http://localhost:4173/products"],
    },
    assert: {
      // Fail CI nếu score thấp hơn ngưỡng
      assertions: {
        "categories:performance": ["error", { minScore: 0.8 }],
        "categories:accessibility": ["error", { minScore: 0.9 }],
        "first-contentful-paint": ["error", { maxNumericValue: 2000 }],
        "interactive": ["error", { maxNumericValue: 5000 }],
      },
    },
    upload: {
      target: "temporary-public-storage",
    },
  },
};
```

---

## 8. Quy Trình Debug Hiệu Năng

### Symptom → Tool → Fix

```
Symptom: "Khi gõ vào search box, UI bị lag"
Tool: React DevTools Profiler (record khi gõ)
Tìm thấy: ProductList re-render 2.5ms mỗi keystroke, 500 ProductCards re-render
Fix: useMemo cho filter logic + React.memo cho ProductCard + startTransition

───

Symptom: "Trang ban đầu tải chậm (> 4s)"
Tool: Lighthouse + Network tab
Tìm thấy: Bundle.js 3.5MB, Chart.js (400KB) load trong initial bundle
Fix: React.lazy cho ChartDashboard + route-based code splitting

───

Symptom: "Scroll trong danh sách 5000 sản phẩm bị giật"
Tool: Chrome Performance Panel (record khi scroll)
Tìm thấy: Long Task 250ms mỗi lần scroll, 5000 DOM nodes
Fix: Virtualization với @tanstack/virtual

───

Symptom: "Chuyển tab mất 800ms"
Tool: React DevTools Profiler (record khi click tab)
Tìm thấy: DataGrid component mount tốn 600ms (API call + render)
Fix: useTransition khi set active tab + Suspense fallback cho DataGrid
```

### Checklist Trước Khi Deploy

```
□ Lighthouse Performance score > 80
□ LCP < 2.5s
□ INP < 200ms
□ CLS < 0.1
□ No Long Tasks > 200ms trong user flows chính
□ Bundle size initial < 200KB (gzip)
□ Không có memory leak trong long sessions
□ Re-renders: kiểm tra với "Highlight updates" trong DevTools
```

---

## 9. Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Làm thế nào để tìm component nào đang re-render không cần thiết?**

A: Dùng React DevTools:
1. Bật "Highlight updates when components render" trong Settings → Profiler
2. Thao tác trên UI → quan sát màu xanh xuất hiện ở đâu
3. Nếu nghi ngờ → Record trong Profiler → xem "Why did this render?"
4. Tìm component có "Props changed" với reference mới dù giá trị không đổi

---

**Q: Web Vitals là gì và tại sao quan trọng?**

A: Web Vitals là bộ chỉ số của Google đo lường trải nghiệm người dùng thực tế:
- **LCP**: Tốc độ hiển thị nội dung chính
- **INP**: Khả năng phản hồi interactions
- **CLS**: Độ ổn định layout

Quan trọng vì: (1) Ảnh hưởng trực tiếp đến SEO ranking — Google dùng Core Web Vitals trong algorithm; (2) Tương quan trực tiếp với conversion rate và revenue.

---

**Q: Phân biệt actualDuration và baseDuration trong Profiler API.**

A:
- `actualDuration`: Thời gian thực tế render component lần này (có tính memo hóa)
- `baseDuration`: Thời gian render ước tính nếu KHÔNG có memo hóa

So sánh hai giá trị cho biết memoization hiệu quả như thế nào:
- `actualDuration << baseDuration` → memo hoạt động tốt (nhiều component bỏ qua re-render)
- `actualDuration ≈ baseDuration` → memo không giúp ích (tất cả re-render đầy đủ)

---

### Câu Hỏi Nâng Cao

**Q: Giải thích cách tích hợp performance monitoring vào production.**

A: Chiến lược:
1. **Real User Monitoring (RUM)** — Giám Sát Người Dùng Thực: Dùng `web-vitals` library gửi CWV metrics lên analytics (GA4, Datadog, Sentry)
2. **`<Profiler>` có chọn lọc**: Bật cho components nghi ngờ chậm, gửi metrics khi `actualDuration > threshold`
3. **Synthetic Monitoring**: Lighthouse CI trong pipeline — kiểm tra mỗi PR
4. **Error boundary + Sentry**: Bắt và log performance exceptions

```javascript
// Monitoring production-safe
const SLOW_THRESHOLD_MS = 16; // Dưới 60fps

function trackSlowRender(id, phase, duration) {
  if (duration > SLOW_THRESHOLD_MS) {
    Sentry.addBreadcrumb({
      category: "performance",
      message: `Slow render: ${id}`,
      data: { phase, duration },
      level: "warning",
    });
  }
}
```

---

**Q: Long Task là gì và cách phân tích trong Performance Panel?**

A: **Long Task** là bất kỳ tác vụ trên main thread > 50ms — block browser khỏi xử lý input và paint frames. Phân tích:
1. Performance Panel → Record → Nhìn Timeline phía trên
2. Các block màu đỏ với "!" = Long Tasks
3. Click vào → xem Call Stack phía dưới → tìm function nào chiếm nhiều nhất
4. Common causes: Heavy render, large JSON.parse, complex calculations trong render, synchronous storage operations

Fix: Code splitting, Web Workers cho heavy computation, startTransition, virtualization.

---

## 🔗 Điều Hướng

- **Trước đó:** [4-concurrent-features.md](./4-concurrent-features.md) — Concurrent Mode, startTransition
- **Tiếp theo:** [6-bundle-optimization.md](./6-bundle-optimization.md) — Tối ưu kích thước bundle
- **Liên quan:** [1-memoization.md](./1-memoization.md) — Sau khi xác định vấn đề bằng Profiler
- **Chỉ mục:** [INDEX.md](../INDEX.md)
