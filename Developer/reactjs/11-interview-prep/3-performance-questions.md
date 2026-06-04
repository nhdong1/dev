# ⚡ Câu Hỏi Về Hiệu Năng React — Senior Level

> Phần hiệu năng phân biệt Senior với Mid-level. Biết lý thuyết chưa đủ — phải biết đo lường, xác định bottleneck, và áp dụng đúng giải pháp.

---

## 🗂️ Mục Lục

1. [Quy Trình Tối Ưu Hiệu Năng](#quy-trình-tối-ưu)
2. [Memoization — Ghi Nhớ](#memoization)
3. [Re-render Optimization — Tối Ưu Re-render](#re-render)
4. [Code Splitting & Lazy Loading](#code-splitting)
5. [Virtualization — Ảo Hóa Danh Sách](#virtualization)
6. [Concurrent Features — Tính Năng Đồng Thời](#concurrent-features)
7. [Bundle Optimization — Tối Ưu Bundle](#bundle-optimization)
8. [Câu Hỏi Thực Tế Senior](#câu-hỏi-thực-tế)

---

## Quy Trình Tối Ưu Hiệu Năng {#quy-trình-tối-ưu}

### Câu hỏi: Khi nào nên bắt đầu optimize hiệu năng? Quy trình của bạn là gì?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐⭐

**Câu trả lời sai:** "Tôi thêm useMemo và React.memo vào mọi component"

**Câu trả lời đúng:**

```
Quy trình chuẩn: MEASURE → IDENTIFY → FIX → VERIFY

1. MEASURE — ĐO LƯỜNG (không đoán mò)
   └── Dùng React DevTools Profiler, Lighthouse, Chrome Performance
   └── Đặt baseline: FCP, LCP, TTI, INP metrics
   └── Tạo lại scenario gây chậm trong môi trường controlled

2. IDENTIFY — XÁC ĐỊNH BOTTLENECK
   └── Flame graph: component nào mất nhiều thời gian nhất?
   └── "Why did this render?" trong Profiler
   └── Network waterfall: resource nào block rendering?

3. FIX — ÁP DỤNG ĐÚNG GIẢI PHÁP
   └── Rendering bottleneck → React.memo, virtualization
   └── Bundle quá lớn → code splitting
   └── Fetch waterfall → parallel fetching, Suspense
   └── Layout thrashing → useLayoutEffect, CSS instead of JS

4. VERIFY — XÁC NHẬN CẢI THIỆN
   └── Profile lại sau khi sửa
   └── So sánh metrics với baseline
   └── Kiểm tra không có regression
```

**Nguyên tắc vàng:** "Premature optimization is the root of all evil" — Donald Knuth. Chỉ optimize khi đã đo được vấn đề.

---

### Câu hỏi: Sử dụng React DevTools Profiler — Công Cụ Phân Tích Hiệu Năng như thế nào?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

```
Bước 1: Mở React DevTools → Tab "Profiler"
Bước 2: Click "Record" (vòng tròn màu xám/đỏ)
Bước 3: Thực hiện action gây chậm (click, scroll, type...)
Bước 4: Click "Stop"
Bước 5: Phân tích Flame Chart — Biểu Đồ Ngọn Lửa

Flame Chart:
├── Width: Thời gian render (rộng hơn = chậm hơn)
├── Color:
│   ├── Xanh lá: Nhanh (< 1ms)
│   ├── Vàng: Trung bình (1-5ms)
│   └── Đỏ: Chậm (> 5ms)
└── "Commits" tab: Mỗi commit là một lần React cập nhật DOM

Cài đặt quan trọng:
├── "Record why each component rendered" → Bật để thấy nguyên nhân re-render
└── "Highlight updates when components render" → Thấy updates trực quan
```

**Đọc Profiler Output:**
```
Ví dụ Flame Chart:
App (2ms)
├── Header (0.1ms) — nhanh, OK
├── ProductList (85ms) — ❌ CHẬM!
│   ├── ProductCard x 100 (0.8ms each)
│   └── SortPanel (0.2ms)
└── Footer (0.1ms)

→ ProductList là bottleneck. Tại sao?
→ Phân tích: 100 ProductCard render mỗi lần ProductList re-render
→ Fix: React.memo cho ProductCard + kiểm tra tại sao ProductList re-render
```

---

## Memoization — Ghi Nhớ {#memoization}

### Câu hỏi: Giải thích shallow comparison — So Sánh Nông trong React.memo

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐

React.memo dùng **shallow comparison** — so sánh giá trị của props ở level 1 (không đi sâu vào nested objects):

```jsx
// Shallow comparison hoạt động như sau:
function shallowEqual(prevProps, nextProps) {
  const prevKeys = Object.keys(prevProps);
  const nextKeys = Object.keys(nextProps);

  if (prevKeys.length !== nextKeys.length) return false;

  for (const key of prevKeys) {
    if (prevProps[key] !== nextProps[key]) return false; // Chỉ so sánh top-level
  }
  return true;
}
```

**Hệ quả:**
```jsx
const Component = React.memo(({ config, items }) => {
  return <div>{items.length}</div>;
});

function Parent() {
  // ❌ Primitive: OK
  // ❌ Object literal mới mỗi render → memo không hiệu quả
  return <Component config={{ theme: 'dark' }} items={[1, 2, 3]} />;
  //     config = { theme: 'dark' } → reference mới mỗi render
  //     items = [1, 2, 3] → reference mới mỗi render
  //     → Component LUÔN re-render dù data không thay đổi!
}

// ✅ Fix: Memo object/array
const config = useMemo(() => ({ theme: 'dark' }), []);
const items = useMemo(() => [1, 2, 3], []);
return <Component config={config} items={items} />;

// ✅ Hoặc custom comparison function
const Component = React.memo(({ user }) => {
  return <div>{user.name}</div>;
}, (prevProps, nextProps) => {
  // Return true nếu KHÔNG cần re-render (giống shouldComponentUpdate ngược)
  return prevProps.user.id === nextProps.user.id; // Chỉ so sánh id
});
```

---

### Câu hỏi: React.memo, useMemo, useCallback — Khi nào memoization có ROI — Return on Investment dương?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

**Memoization có chi phí (overhead):**
- Lưu giá trị trước đó trong bộ nhớ
- So sánh deps mỗi render
- Complexity tăng → khó debug

**ROI dương khi:**
```
1. Tính toán > ~1ms (đủ nặng để đáng memo)
2. Component render > 2-3 lần không cần thiết (đủ thường)
3. Memoized value được truyền xuống component có React.memo

ROI âm (đừng dùng) khi:
1. Tính toán đơn giản (string concat, simple math)
2. Component render ít
3. Deps thay đổi thường xuyên → memo luôn miss
```

```jsx
// Đánh giá: Liệu useMemo có ích ở đây không?

// ❌ Không cần — tính toán trivial
const title = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);
// Chi phí memo > chi phí string concat

// ✅ Cần — tính toán nặng
const statistics = useMemo(() => {
  return {
    average: data.reduce((sum, item) => sum + item.value, 0) / data.length,
    max: Math.max(...data.map(d => d.value)),
    min: Math.min(...data.map(d => d.value)),
    standardDeviation: calculateStdDev(data),
  };
}, [data]); // data có thể là 10,000 items
```

---

## Re-render Optimization — Tối Ưu Re-render {#re-render}

### Câu hỏi: Tại sao component re-render? Liệt kê tất cả nguyên nhân

**Cấp độ:** Mid/Senior | **Tần suất:** ⭐⭐⭐⭐⭐

```
Component re-render khi:

1. STATE THAY ĐỔI:
   useState setter gọi với giá trị mới (Object.is check)

2. PROPS THAY ĐỔI:
   Parent re-render → truyền props mới → con re-render
   (ngay cả khi props content giống nhau nếu reference khác)

3. CONTEXT THAY ĐỔI:
   Bất kỳ consumer nào re-render khi context value thay đổi

4. PARENT RE-RENDER:
   Khi parent re-render, TẤT CẢ children re-render
   (trừ khi dùng React.memo với stable props)

5. HOOKS:
   useReducer dispatch
   useSyncExternalStore subscription thay đổi
```

**Kiểm tra nguyên nhân:**
```jsx
// Thêm vào component để debug
function MyComponent(props) {
  const renderCount = useRef(0);
  renderCount.current++;

  // In ra props nào thay đổi
  const prevPropsRef = useRef(props);
  useEffect(() => {
    const changedProps = Object.entries(props).reduce((acc, [key, val]) => {
      if (prevPropsRef.current[key] !== val) {
        acc[key] = { from: prevPropsRef.current[key], to: val };
      }
      return acc;
    }, {});

    if (Object.keys(changedProps).length > 0) {
      console.log('Changed props:', changedProps);
    }
    prevPropsRef.current = props;
  });

  console.log(`Render #${renderCount.current}`);
  // ...
}

// Hoặc dùng: why-did-you-render library
import '@welldone-software/why-did-you-render';
MyComponent.whyDidYouRender = true;
```

---

### Câu hỏi: Chiến lược tổng thể để giảm re-render không cần thiết

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
// Chiến lược 1: State colocation — Đặt state gần nơi dùng nhất
// ❌ State ở root → mọi component re-render khi filter thay đổi
function App() {
  const [filter, setFilter] = useState('all'); // State ở quá cao
  return (
    <>
      <Header /> {/* Re-render dù không dùng filter */}
      <ProductList filter={filter} />
      <Footer />
    </>
  );
}

// ✅ Chỉ component cần filter mới re-render
function ProductSection() {
  const [filter, setFilter] = useState('all'); // State đúng chỗ
  return <ProductList filter={filter} />;
}
function App() {
  return (
    <>
      <Header /> {/* Không re-render khi filter thay đổi */}
      <ProductSection />
      <Footer />
    </>
  );
}

// Chiến lược 2: Children as props (slot pattern)
// ❌ Expensive re-renders
function ColorPicker() {
  const [color, setColor] = useState('blue');
  return (
    <div style={{ background: color }}>
      <SlowComponent /> {/* Re-render mỗi khi color thay đổi */}
    </div>
  );
}

// ✅ SlowComponent không re-render vì là stable prop reference từ cha
function ColorPickerWrapper() {
  const [color, setColor] = useState('blue');
  return (
    <ColorPicker color={color} onColorChange={setColor}>
      <SlowComponent /> {/* Tạo ở đây, không tạo lại khi color đổi */}
    </ColorPicker>
  );
}

function ColorPicker({ color, children }) {
  return <div style={{ background: color }}>{children}</div>; // Children ổn định
}
```

---

## Code Splitting & Lazy Loading {#code-splitting}

### Câu hỏi: Chiến lược code splitting cho ứng dụng lớn

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

```jsx
// Chiến lược 1: Route-based splitting (phổ biến nhất)
const HomePage = lazy(() => import('./pages/Home'));
const DashboardPage = lazy(() => import('./pages/Dashboard'));
const AdminPage = lazy(() => import('./pages/Admin'));
const SettingsPage = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<PageLoadingSpinner />}>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/dashboard" element={<DashboardPage />} />
        <Route path="/admin" element={<AdminPage />} />
        <Route path="/settings" element={<SettingsPage />} />
      </Routes>
    </Suspense>
  );
}

// Chiến lược 2: Component-based splitting (heavy components)
const RichTextEditor = lazy(() => import('./components/RichTextEditor'));
const DataVisualization = lazy(() => import('./components/DataVisualization'));
const PDFViewer = lazy(() => import('./components/PDFViewer'));

// Chỉ tải khi modal mở
function DocumentEditor() {
  const [showEditor, setShowEditor] = useState(false);

  return (
    <>
      <button onClick={() => setShowEditor(true)}>Chỉnh Sửa</button>
      {showEditor && (
        <Suspense fallback={<EditorSkeleton />}>
          <RichTextEditor />
        </Suspense>
      )}
    </>
  );
}

// Chiến lược 3: Prefetching — Tải Trước (cải thiện UX)
// Tải trước khi user cần, ví dụ khi hover
function NavLink({ to, children }) {
  const handleMouseEnter = () => {
    // Preload — tải trước khi user click
    import(`./pages/${to}`).catch(() => {});
  };

  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  );
}
```

**Bundle analysis — Phân Tích Bundle:**
```bash
# Vite
npx vite build --mode production
npx vite-bundle-visualizer

# Next.js
npx @next/bundle-analyzer

# Webpack
npx webpack-bundle-analyzer stats.json
```

---

### Câu hỏi: Suspense — Trạng Thái Chờ boundaries và loading states — Trạng Thái Tải tốt nhất

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
// Nested Suspense — Suspense Lồng Nhau
// Mỗi cấp có fallback riêng tùy granularity cần thiết
function App() {
  return (
    <Suspense fallback={<AppShell />}> {/* Level 1: layout skeleton */}
      <Header />
      <Suspense fallback={<ContentSkeleton />}> {/* Level 2: content */}
        <MainContent />
        <Suspense fallback={<SidebarSkeleton />}> {/* Level 3: sidebar */}
          <Sidebar />
        </Suspense>
      </Suspense>
    </Suspense>
  );
}

// Kết hợp Suspense + Error Boundary + Retry
function DataSection({ queryKey, queryFn }) {
  return (
    <ErrorBoundary
      fallback={({ error, reset }) => (
        <ErrorCard error={error} onRetry={reset} />
      )}
    >
      <Suspense fallback={<DataSkeleton />}>
        <DataContent queryKey={queryKey} queryFn={queryFn} />
      </Suspense>
    </ErrorBoundary>
  );
}

// Skeleton screens thay vì spinner (UX tốt hơn)
function ProductCardSkeleton() {
  return (
    <div className="skeleton-card">
      <div className="skeleton-image animate-pulse" />
      <div className="skeleton-title animate-pulse" />
      <div className="skeleton-price animate-pulse" />
    </div>
  );
}
```

---

## Virtualization — Ảo Hóa Danh Sách {#virtualization}

### Câu hỏi: Virtualization giải quyết vấn đề gì? Khi nào nên dùng?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

**Vấn đề:** Render 10,000 DOM nodes → browser lag nặng, memory cao, paint chậm.

**Giải pháp:** Chỉ render items **hiện đang visible — hiển thị** trong viewport — khung nhìn. Items ngoài viewport chỉ là placeholders.

```jsx
// Không virtualize — render 10,000 items
function SlowList({ items }) {
  return (
    <div style={{ height: '400px', overflow: 'auto' }}>
      {items.map(item => <ItemRow key={item.id} item={item} />)}
      // 10,000 DOM nodes → freeze!
    </div>
  );
}

// Với react-window (FixedSizeList — Danh Sách Kích Thước Cố Định)
import { FixedSizeList } from 'react-window';

function FastList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}> {/* style chứa position absolute của row */}
      <ItemRow item={items[index]} />
    </div>
  );

  return (
    <FixedSizeList
      height={400}        // Chiều cao container
      itemCount={items.length}
      itemSize={50}       // Chiều cao mỗi item (cố định)
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}

// Với react-window (VariableSizeList — Kích Thước Thay Đổi)
import { VariableSizeList } from 'react-window';

function DynamicHeightList({ items }) {
  const getItemSize = (index) => {
    return items[index].isExpanded ? 120 : 50; // Kích thước động
  };

  return (
    <VariableSizeList
      height={400}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {({ index, style }) => <ItemRow style={style} item={items[index]} />}
    </VariableSizeList>
  );
}
```

**Khi nào nên virtualize:**
- Danh sách > 100-200 items với re-render thường xuyên
- Danh sách > 1,000 items dù static
- Mobile devices với memory thấp

**Thư viện:**
- `react-window` — nhẹ, đơn giản
- `@tanstack/react-virtual` — linh hoạt, modern API
- `react-virtuoso` — dễ dùng, hỗ trợ infinite scroll

---

## Concurrent Features — Tính Năng Đồng Thời {#concurrent-features}

### Câu hỏi: startTransition — Bắt Đầu Chuyển Tiếp giải quyết vấn đề gì?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

**Vấn đề:** User gõ text vào search box → cùng lúc phải:
1. Cập nhật input (phải nhanh — dưới 50ms)
2. Filter/search kết quả (có thể chậm — vài trăm ms)

**Không có startTransition:** Hai updates có cùng priority — Độ Ưu Tiên → blocking — chặn nhau → UI bị giật khi gõ.

```jsx
// ❌ Không có startTransition — UI bị giật
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState(allItems);

  const handleChange = (e) => {
    setQuery(e.target.value);    // Urgent — Khẩn Cấp
    setResults(filter(e.target.value)); // Nặng → block UI!
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultList results={results} /> {/* 10,000 items */}
    </>
  );
}

// ✅ Với startTransition
import { startTransition, useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState(allItems);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value); // Urgent: update input NGAY

    startTransition(() => {
      // Non-urgent: React có thể interrupt nếu có urgent update mới
      setResults(filter(value));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <LoadingIndicator />}
      <ResultList
        results={results}
        style={{ opacity: isPending ? 0.5 : 1 }} // Visual feedback
      />
    </>
  );
}
```

---

### Câu hỏi: useDeferredValue — Giá Trị Trì Hoãn vs startTransition — khác nhau như thế nào?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
// startTransition: bạn wrap STATE UPDATE
// → Dùng khi bạn kiểm soát setter

const [isPending, startTransition] = useTransition();
const handleChange = (e) => {
  setQuery(e.target.value); // Urgent
  startTransition(() => {
    setFilteredItems(filter(e.target.value)); // Deferred
  });
};

// useDeferredValue: bạn wrap VALUE từ props hoặc state
// → Dùng khi bạn NHẬN value từ ngoài (prop, URL param...)
// → Giá trị "lags behind" giá trị thực

function SearchResults({ query }) { // query từ URL hoặc props
  const deferredQuery = useDeferredValue(query);
  // deferredQuery = giá trị cũ trong khi React tính toán với query mới

  const results = useMemo(() => heavyFilter(deferredQuery), [deferredQuery]);
  const isStale = query !== deferredQuery; // Đang pending

  return (
    <div style={{ opacity: isStale ? 0.6 : 1 }}>
      {results.map(item => <ResultItem key={item.id} item={item} />)}
    </div>
  );
}
```

**Tóm tắt:**
- `startTransition` → wrap setter → bạn kiểm soát update nào là deferred
- `useDeferredValue` → wrap value → React quyết định khi nào update

---

## Bundle Optimization — Tối Ưu Bundle {#bundle-optimization}

### Câu hỏi: Tree shaking — Loại Bỏ Code Chết là gì? Làm thế nào để ensure nó hoạt động?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

Tree shaking — Loại Bỏ Code Chết là quá trình loại bỏ code không được sử dụng khỏi bundle cuối cùng. Hoạt động với ES Modules (import/export tĩnh).

```jsx
// ✅ Named imports — Tree shaking hoạt động
import { debounce } from 'lodash-es'; // Chỉ bundle debounce

// ❌ Default import toàn bộ library
import _ from 'lodash'; // Bundle TOÀN BỘ lodash (~500KB)

// ✅ Dùng lodash-es thay vì lodash
import { groupBy, sortBy } from 'lodash-es'; // ES module → tree shakable

// ✅ Import theo path (cho CommonJS libraries)
import debounce from 'lodash/debounce'; // Chỉ debounce
```

**Kiểm tra tree shaking:**
```bash
# Sau khi build
npx source-map-explorer dist/static/js/*.js
# Hoặc
npx vite-bundle-visualizer
# Kiểm tra: lodash/moment/icon libraries có quá lớn không?
```

**Anti-patterns phổ biến:**
```jsx
// ❌ Import toàn bộ icon library
import { Icons } from '@mui/icons-material'; // Hàng nghìn icons!
// ✅
import SearchIcon from '@mui/icons-material/Search'; // Chỉ 1 icon

// ❌ Import toàn bộ date library
import moment from 'moment'; // ~300KB!
// ✅ Dùng date-fns (tree shakable)
import { format, parseISO } from 'date-fns'; // Chỉ functions cần

// ❌ Import toàn bộ utility library
import * as utils from './utils'; // Dù chỉ dùng 1 function
// ✅
import { formatCurrency } from './utils';
```

---

### Câu hỏi: Tối ưu hình ảnh và assets trong React app

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐

```jsx
// 1. Lazy loading images — Tải Hình Lười
function ProductImage({ src, alt }) {
  return (
    <img
      src={src}
      alt={alt}
      loading="lazy"      // Native browser lazy loading
      decoding="async"    // Non-blocking decode
    />
  );
}

// 2. Next.js Image component (tự động optimize)
import Image from 'next/image';

function OptimizedImage({ src, alt }) {
  return (
    <Image
      src={src}
      alt={alt}
      width={400}
      height={300}
      placeholder="blur"   // Blur hash trong khi tải
      priority={false}     // true cho above-the-fold images
    />
  );
}

// 3. Intersection Observer để lazy load
function LazyImage({ src, alt }) {
  const [isVisible, setIsVisible] = useState(false);
  const imgRef = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) setIsVisible(true); },
      { threshold: 0.1, rootMargin: '200px' } // Load trước 200px
    );

    if (imgRef.current) observer.observe(imgRef.current);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={imgRef}>
      {isVisible ? (
        <img src={src} alt={alt} />
      ) : (
        <div className="image-placeholder" style={{ aspectRatio: '4/3' }} />
      )}
    </div>
  );
}
```

---

## Câu Hỏi Thực Tế Senior {#câu-hỏi-thực-tế}

### Câu hỏi tình huống: App của bạn chậm khi scroll, bạn debug như thế nào?

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

```
Quy trình debug:

1. Xác định loại vấn đề:
   - Scroll jank (giật) → JavaScript blocking main thread
   - FPS thấp → rendering bottleneck
   - Memory leak — Rò Rỉ Bộ Nhớ → component không cleanup

2. Tools sử dụng:
   Chrome DevTools → Performance tab → Record while scrolling
   → Xem:
     - Long tasks (đỏ): JS execution > 50ms
     - Forced reflows: đọc layout sau khi đã write
     - Paint flashing: vùng nào repaint nhiều

3. Nguyên nhân phổ biến và fix:
   
   A. Render quá nhiều DOM nodes trong list:
      → Thêm virtualization (react-window)
   
   B. Scroll event handler nặng:
      → Throttle hoặc Debounce scroll handler
      → Dùng Intersection Observer thay vì scroll event
   
   C. Layout thrashing (đọc/ghi DOM xen kẽ):
      → Đọc tất cả measurements → rồi mới write
      → Dùng CSS thay vì JS animation
   
   D. Images tải không đúng cách:
      → Thêm explicit width/height → tránh layout shifts
      → Lazy loading với Intersection Observer
   
   E. Rerender component trên mỗi scroll:
      → Kiểm tra state trong parent không nên update mỗi scroll event
```

---

### Câu hỏi tình huống: Tối ưu initial page load — Tải Trang Ban Đầu

**Cấp độ:** Senior | **Tần suất:** ⭐⭐⭐⭐

```
Chiến lược toàn diện:

1. CRITICAL PATH OPTIMIZATION:
   - Inline critical CSS (above-the-fold styles)
   - Preload critical fonts
   - Defer non-critical scripts
   
   <link rel="preload" href="/fonts/main.woff2" as="font" crossOrigin />
   <link rel="preconnect" href="https://api.example.com" />

2. JAVASCRIPT OPTIMIZATION:
   - Code splitting: chỉ tải JS cần thiết cho route hiện tại
   - Vendor chunk splitting: tách third-party libraries
   - Tree shaking: loại bỏ dead code
   
3. DATA FETCHING STRATEGY:
   - Parallel data fetching thay vì sequential — Tuần Tự
   - React Server Components: zero JS cho static content
   - Streaming SSR: trả về HTML dần dần
   
4. CACHING STRATEGY:
   - Static assets: long-lived cache + content hash
   - API responses: appropriate stale times
   - Service Worker: offline support
   
5. METRICS TO TRACK:
   - FCP (First Contentful Paint — Vẽ Nội Dung Đầu Tiên): < 1.8s
   - LCP (Largest Contentful Paint — Vẽ Nội Dung Lớn Nhất): < 2.5s
   - TTI (Time to Interactive — Thời Gian Đến Tương Tác): < 3.8s
   - INP (Interaction to Next Paint — Từ Tương Tác Đến Vẽ Tiếp): < 200ms
   - CLS (Cumulative Layout Shift — Độ Dịch Chuyển Layout): < 0.1
```

---

## ✅ Checklist Hiệu Năng — Senior Level

**Measurement:**
- [ ] Sử dụng thành thạo React DevTools Profiler
- [ ] Đọc được Chrome Performance tab flame chart
- [ ] Biết các Core Web Vitals và target values

**Memoization:**
- [ ] Biết khi nào React.memo CÓ và KHÔNG có ích
- [ ] Phân biệt shallow vs deep comparison
- [ ] Tránh premature optimization

**Re-render:**
- [ ] Liệt kê được 5 nguyên nhân re-render
- [ ] State colocation pattern
- [ ] Children as props pattern

**Code Splitting:**
- [ ] Route-based splitting với React.lazy
- [ ] Component-based splitting (heavy modals, charts)
- [ ] Prefetching strategy

**Virtualization:**
- [ ] Biết khi nào cần virtualize
- [ ] Implement được với react-window

**Concurrent:**
- [ ] startTransition — use case cụ thể
- [ ] useDeferredValue — khi nào thích hợp hơn startTransition

**Bundle:**
- [ ] Tree shaking — đảm bảo hoạt động
- [ ] Import patterns tránh bundle bloat

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 (stable)
