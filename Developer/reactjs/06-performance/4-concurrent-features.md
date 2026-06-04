# Concurrent Features — Tính Năng Đồng Thời React

> **Concurrent Mode** (Chế Độ Đồng Thời) là bộ tính năng React từ v18 cho phép React **ưu tiên hóa** (prioritize) các cập nhật UI. Thay vì cập nhật đồng bộ — blocking — React có thể tạm dừng, tiếp tục, hoặc hủy bỏ một render để ưu tiên những cập nhật quan trọng hơn.

---

## 📌 Mục Lục

1. [Concurrent Mode Là Gì?](#1-concurrent-mode-là-gì)
2. [startTransition — Đánh Dấu Cập Nhật Không Khẩn Cấp](#2-starttransition--đánh-dấu-cập-nhật-không-khẩn-cấp)
3. [useTransition — Hook Cho Transition](#3-usetransition--hook-cho-transition)
4. [useDeferredValue — Trì Hoãn Giá Trị](#4-usedeferredvalue--trì-hoãn-giá-trị)
5. [Suspense Trong Concurrent Mode](#5-suspense-trong-concurrent-mode)
6. [useId — Tạo ID Ổn Định](#6-useid--tạo-id-ổn-định)
7. [So Sánh startTransition vs useDeferredValue](#7-so-sánh-starttransition-vs-usedeferredvalue)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Concurrent Mode Là Gì?

### Render Đồng Bộ vs Concurrent — Trước và Sau React v18

```
Render Đồng Bộ (Synchronous Rendering) — React v17 trở về trước:

setFilter("laptop") → React bắt đầu render → không thể dừng → 
→ Re-render toàn bộ list (200ms) → UI bị "đóng băng" 200ms
→ Người dùng gõ tiếp nhưng input không phản hồi → trải nghiệm tệ

Render Đồng Thời (Concurrent Rendering) — React v18+:

setFilter("laptop") → React bắt đầu render list (non-urgent)
  → Người dùng gõ thêm "laptop pro" → React TẠM DỪNG render list
  → Cập nhật input ngay lập tức (urgent)
  → Tiếp tục render list với filter mới "laptop pro"
  → UI luôn phản hồi, không bao giờ bị đóng băng
```

### Kích Hoạt Concurrent Mode

```jsx
// React v18 — createRoot kích hoạt Concurrent Mode
// ❌ React v17 (legacy mode)
import ReactDOM from "react-dom";
ReactDOM.render(<App />, document.getElementById("root"));

// ✅ React v18 (concurrent mode)
import { createRoot } from "react-dom/client";
const root = createRoot(document.getElementById("root"));
root.render(<App />);
```

### Hai Loại Cập Nhật

React v18 phân loại updates thành hai loại:

```
Urgent Updates (Cập Nhật Khẩn Cấp):
- Phản hồi trực tiếp input của người dùng
- Ví dụ: gõ phím, click, hover
- Yêu cầu: phải cập nhật NGAY LẬP TỨC
- Nếu chậm > 16ms → người dùng cảm thấy lag

Non-urgent Updates (Cập Nhật Không Khẩn Cấp):
- Kết quả tính toán từ urgent update
- Ví dụ: filter list, update chart, load search results
- Có thể trì hoãn vài trăm ms mà người dùng không để ý
```

---

## 2. startTransition — Đánh Dấu Cập Nhật Không Khẩn Cấp

`startTransition` đánh dấu một state update là "transition" — không khẩn cấp, có thể bị interrupt (gián đoạn).

### Cú Pháp Cơ Bản

```jsx
import { startTransition } from "react";

function SearchBox() {
  const [query, setQuery] = useState(""); // Urgent: input value
  const [results, setResults] = useState([]); // Non-urgent: search results

  function handleChange(e) {
    const value = e.target.value;

    // ✅ Cập nhật input ngay lập tức (urgent)
    setQuery(value);

    // ✅ Đánh dấu cập nhật kết quả là non-urgent
    startTransition(() => {
      // Tính toán tốn kém — React có thể trì hoãn hoặc interrupt
      const filtered = hugeDataset.filter(item =>
        item.name.toLowerCase().includes(value.toLowerCase())
      );
      setResults(filtered);
    });
  }

  return (
    <div>
      <input value={query} onChange={handleChange} placeholder="Tìm kiếm..." />
      <ResultList results={results} />
    </div>
  );
}
```

### Ví Dụ Trước và Sau startTransition

```jsx
// ❌ Trước (không có startTransition)
function FilterList({ items }) {
  const [filter, setFilter] = useState("");

  function handleFilterChange(e) {
    setFilter(e.target.value); // Render lại FilterList với 10,000 items → UI lag
  }

  const filtered = items.filter(item =>
    item.name.toLowerCase().includes(filter.toLowerCase())
  );

  return (
    <div>
      <input value={filter} onChange={handleFilterChange} />
      {/* 10,000 items → render tốn 300ms → input lag 300ms */}
      {filtered.map(item => <ItemRow key={item.id} item={item} />)}
    </div>
  );
}

// ✅ Sau (với startTransition)
function FilterList({ items }) {
  const [filter, setFilter] = useState(""); // Input value — luôn responsive
  const [debouncedFilter, setDebouncedFilter] = useState(""); // Filter thực — có thể trì hoãn

  function handleFilterChange(e) {
    const value = e.target.value;
    setFilter(value); // Urgent: cập nhật input ngay

    startTransition(() => {
      setDebouncedFilter(value); // Non-urgent: filter list có thể chậm
    });
  }

  const filtered = useMemo(
    () => items.filter(item =>
      item.name.toLowerCase().includes(debouncedFilter.toLowerCase())
    ),
    [items, debouncedFilter]
  );

  return (
    <div>
      <input value={filter} onChange={handleFilterChange} />
      {/* Render với debouncedFilter — có thể interrupt nếu người dùng gõ tiếp */}
      {filtered.map(item => <ItemRow key={item.id} item={item} />)}
    </div>
  );
}
```

---

## 3. useTransition — Hook Cho Transition

`useTransition` là hook version của `startTransition`, cung cấp thêm **isPending** — trạng thái đang xử lý:

### Cú Pháp

```jsx
import { useTransition } from "react";

const [isPending, startTransition] = useTransition();
// isPending: boolean — true khi đang có transition chưa hoàn thành
// startTransition: function — đánh dấu update là non-urgent
```

### Ví Dụ Với Loading State

```jsx
function SearchPage() {
  const [query, setQuery] = useState("");
  const [searchResults, setSearchResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(e) {
    const value = e.target.value;
    setQuery(value);

    startTransition(() => {
      // Giả lập tìm kiếm phức tạp (đồng bộ)
      const results = performExpensiveSearch(value);
      setSearchResults(results);
    });
  }

  return (
    <div>
      <input
        value={query}
        onChange={handleSearch}
        placeholder="Tìm kiếm sản phẩm..."
      />

      {/* Hiển thị loading indicator khi transition đang chạy */}
      {isPending && (
        <div className="search-loading">
          <Spinner size="sm" />
          <span>Đang tìm kiếm...</span>
        </div>
      )}

      {/* Làm mờ kết quả cũ trong khi chờ kết quả mới */}
      <div style={{ opacity: isPending ? 0.5 : 1, transition: "opacity 0.2s" }}>
        <p>{searchResults.length} kết quả cho "{query}"</p>
        {searchResults.map(result => (
          <SearchResultItem key={result.id} result={result} />
        ))}
      </div>
    </div>
  );
}
```

### Tab Navigation Với Transition

```jsx
function TabContainer() {
  const [activeTab, setActiveTab] = useState("overview");
  const [isPending, startTransition] = useTransition();

  const tabs = [
    { id: "overview", label: "Tổng Quan", Component: OverviewTab },
    { id: "analytics", label: "Phân Tích", Component: AnalyticsTab },
    { id: "settings", label: "Cài Đặt", Component: SettingsTab },
  ];

  const ActiveComponent = tabs.find(t => t.id === activeTab)?.Component;

  return (
    <div>
      <nav>
        {tabs.map(tab => (
          <button
            key={tab.id}
            onClick={() => {
              startTransition(() => setActiveTab(tab.id));
            }}
            style={{
              fontWeight: activeTab === tab.id ? "bold" : "normal",
              opacity: isPending && activeTab !== tab.id ? 0.7 : 1,
            }}
          >
            {tab.label}
            {isPending && activeTab === tab.id && " ⏳"}
          </button>
        ))}
      </nav>

      <div style={{ opacity: isPending ? 0.8 : 1 }}>
        {ActiveComponent && <ActiveComponent />}
      </div>
    </div>
  );
}
```

### Transition Với Async Data Fetching

```jsx
// React 19: startTransition hỗ trợ async functions
function AsyncSearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(value) {
    setQuery(value);

    // React 19: async trong startTransition
    startTransition(async () => {
      const data = await searchAPI(value); // Gọi API async
      setResults(data); // Cập nhật sau khi có dữ liệu
    });
  }

  return (
    <div>
      <input onChange={e => handleSearch(e.target.value)} />
      {isPending ? <Spinner /> : <ResultList results={results} />}
    </div>
  );
}
```

---

## 4. useDeferredValue — Trì Hoãn Giá Trị

`useDeferredValue` trả về phiên bản **trì hoãn** của một giá trị — React sẽ ưu tiên cập nhật giá trị urgent trước, sau đó mới cập nhật deferred value.

### Cú Pháp

```jsx
import { useDeferredValue } from "react";

const deferredValue = useDeferredValue(value);
// deferredValue: phiên bản "cũ" của value trong khi React đang tính toán version mới
// Khi React rảnh → deferredValue bắt kịp với value thực
```

### Ví Dụ Tìm Kiếm

```jsx
function SearchWithDeferred() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  // isStale: true khi query và deferredQuery khác nhau
  // → đang trong quá trình chuyển đổi
  const isStale = query !== deferredQuery;

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Gõ để tìm kiếm..."
      />

      {/* SearchResults chỉ nhận deferredQuery
          → không re-render mỗi keystroke, chỉ re-render khi React "rảnh" */}
      <div style={{
        opacity: isStale ? 0.5 : 1,
        transition: "opacity 0.2s",
      }}>
        <SearchResults query={deferredQuery} />
      </div>
    </div>
  );
}

// SearchResults được memoize để React có thể reuse khi cần
const SearchResults = memo(({ query }) => {
  // Đây là computation tốn kém
  const results = useMemo(
    () => searchInMemory(hugeDataset, query),
    [query]
  );

  return (
    <ul>
      {results.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
});
```

### useDeferredValue Với External Data

```jsx
function SliderDemo() {
  const [sliderValue, setSliderValue] = useState(50);
  const deferredValue = useDeferredValue(sliderValue);

  return (
    <div>
      {/* Input: cập nhật ngay lập tức theo slider */}
      <input
        type="range"
        min={0}
        max={100}
        value={sliderValue}
        onChange={e => setSliderValue(Number(e.target.value))}
      />
      <p>Slider: {sliderValue}</p>

      {/* Chart: cập nhật trì hoãn → slider mượt mà, chart update chậm hơn */}
      <ExpensiveChart
        value={deferredValue}
        // Hiển thị giá trị đang xử lý vs đã hiển thị
        label={`Hiển thị: ${deferredValue} | Thực tế: ${sliderValue}`}
      />
    </div>
  );
}
```

---

## 5. Suspense Trong Concurrent Mode

**Suspense** không chỉ là fallback cho lazy loading — trong Concurrent Mode, Suspense trở thành cơ chế khai báo cho async data:

### Suspense Boundary — Ranh Giới Suspense

```jsx
// React 18+ Suspense hoạt động với bất kỳ "suspendable" resource:
// - React.lazy components
// - use() hook (React 19)
// - Framework data fetching (Next.js App Router)

function ProductPage({ productId }) {
  return (
    <div>
      <h1>Chi Tiết Sản Phẩm</h1>

      {/* Mỗi Suspense boundary xử lý loading state riêng biệt */}
      <Suspense fallback={<ProductDetailSkeleton />}>
        <ProductDetail productId={productId} />
      </Suspense>

      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews productId={productId} />
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations productId={productId} />
      </Suspense>
    </div>
  );
}
// Kết quả: 3 sections tải độc lập → section nào xong trước hiện trước
// Thay vì chờ tất cả xong mới hiện → UX tốt hơn
```

### use() Hook — Đọc Promise Trong Render (React 19)

```jsx
import { use } from "react";

// use() có thể đọc Promise trực tiếp trong render function
// Khi Promise chưa resolve → component suspend → Suspense hiện fallback
function UserProfile({ userPromise }) {
  const user = use(userPromise); // Suspend cho đến khi Promise resolve

  return (
    <div>
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.bio}</p>
    </div>
  );
}

// Tạo Promise bên ngoài component (không tạo mới mỗi render)
const userPromise = fetchUser(userId);

function App() {
  return (
    <Suspense fallback={<ProfileSkeleton />}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

### Suspense Với Server Components (Next.js)

```jsx
// Next.js App Router — Server Component với Suspense
// app/dashboard/page.tsx

import { Suspense } from "react";

// Server component — fetch trực tiếp trên server, không cần API calls từ client
async function SalesSummary() {
  const data = await fetchSalesData(); // Chạy trên server
  return <SalesChart data={data} />;
}

async function RecentOrders() {
  const orders = await fetchRecentOrders(); // Chạy trên server song song
  return <OrderTable orders={orders} />;
}

// Page component — orchestrate các async server components
export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      {/* Mỗi async server component được bọc trong Suspense riêng
          → Stream về client khi sẵn sàng */}
      <Suspense fallback={<ChartSkeleton />}>
        <SalesSummary />
      </Suspense>

      <Suspense fallback={<TableSkeleton />}>
        <RecentOrders />
      </Suspense>
    </div>
  );
}
// Kết quả: HTML được stream về từng phần → người dùng thấy nội dung sớm hơn
```

---

## 6. useId — Tạo ID Ổn Định

`useId` tạo **unique ID** (ID duy nhất) ổn định qua Server và Client — quan trọng để tránh **hydration mismatch** (không khớp khi hydrate).

### Vấn Đề Trước useId

```jsx
// ❌ Cách cũ: dùng Math.random() hoặc counter thủ công
let idCounter = 0;
function OldInput({ label }) {
  // ID khác nhau giữa server và client → hydration error!
  const id = `input-${++idCounter}`;
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </div>
  );
}
```

### Giải Pháp Với useId

```jsx
import { useId } from "react";

function AccessibleInput({ label, type = "text" }) {
  const id = useId(); // Ổn định, unique, giống nhau trên server và client

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type={type} />
    </div>
  );
}

// Một component có nhiều elements cần ID
function Form() {
  const nameId = useId();
  const emailId = useId();
  const descriptionId = useId();

  return (
    <form>
      <div>
        <label htmlFor={nameId}>Tên</label>
        <input id={nameId} aria-describedby={`${nameId}-hint`} />
        <span id={`${nameId}-hint`}>Nhập họ và tên đầy đủ</span>
      </div>
      <div>
        <label htmlFor={emailId}>Email</label>
        <input id={emailId} type="email" />
      </div>
      <div>
        <label htmlFor={descriptionId}>Mô tả</label>
        <textarea id={descriptionId} />
      </div>
    </form>
  );
}
```

---

## 7. So Sánh startTransition vs useDeferredValue

| | `startTransition` | `useDeferredValue` |
| - | ----------------- | ------------------ |
| **Cách dùng** | Wrap state setter | Wrap giá trị đã có |
| **Kiểm soát** | Bạn kiểm soát khi nào transition xảy ra | React tự quyết định |
| **isPending** | ✅ (qua `useTransition`) | ❌ Cần tự detect |
| **Dùng khi** | Bạn có quyền kiểm soát state update | Nhận value từ parent hoặc không thể control update |
| **Ví dụ** | Search input → filter results | Nhận `query` prop từ parent, defer expensive calculation |

### Khi Nào Dùng Gì?

```jsx
// Dùng startTransition khi bạn kiểm soát state setter
function Parent() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  return (
    <input onChange={e => {
      setQuery(e.target.value); // Urgent
      startTransition(() => {
        setResults(search(e.target.value)); // Non-urgent
      });
    }} />
  );
}

// Dùng useDeferredValue khi bạn nhận value từ nơi khác
function ExpensiveList({ query }) { // Không kiểm soát được khi query thay đổi
  const deferredQuery = useDeferredValue(query);

  const results = useMemo(
    () => search(deferredQuery),
    [deferredQuery]
  );

  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>;
}
```

---

## 8. Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Concurrent Mode là gì và được kích hoạt như thế nào?**

A: Concurrent Mode là bộ tính năng React v18 cho phép React ưu tiên hóa cập nhật UI — render có thể bị interrupt, tạm dừng, và tiếp tục. Kích hoạt bằng cách dùng `createRoot` thay vì `ReactDOM.render` (legacy mode).

---

**Q: startTransition làm gì và khi nào cần dùng?**

A: `startTransition` đánh dấu một state update là "transition" — không khẩn cấp. React có thể tạm dừng render transition này để ưu tiên urgent updates (như gõ phím). Dùng khi có tính toán tốn kém (filter, sort, render list lớn) chạy đồng bộ kết hợp với input của người dùng.

---

**Q: Phân biệt startTransition và useDeferredValue.**

A:
- `startTransition`: Bạn wrap state setter — kiểm soát chính xác khi nào transition xảy ra, có `isPending` để hiện loading state
- `useDeferredValue`: Bạn wrap giá trị đầu vào — React tự quyết định khi nào "bắt kịp", phù hợp khi bạn không kiểm soát state setter (nhận từ props)

---

### Câu Hỏi Nâng Cao

**Q: Tại sao Suspense quan trọng trong Concurrent Mode?**

A: Trong Concurrent Mode, Suspense trở thành cơ chế khai báo cho loading states:
- Trước đây: Suspense chỉ cho `React.lazy`
- React 18+: Suspense với concurrent data fetching
- React 19: Suspense với `use()` hook và async Server Components

Multiple Suspense boundaries cho phép stream nội dung từng phần về client — phần nào sẵn sàng trước thì hiển thị trước — thay vì block toàn bộ trang đến khi tất cả xong.

---

**Q: Giải thích "Tearing" trong concurrent rendering và React giải quyết như thế nào?**

A: **Tearing** (Xé Nứt) là tình trạng trong concurrent rendering khi React có thể render cùng component với dữ liệu khác nhau vì store state thay đổi trong khi đang render:

```
Render bắt đầu → store.value = "A" → ComponentA đọc "A"
React tạm dừng → store.value thay đổi thành "B" (external update)
React tiếp tục → ComponentB đọc "B"
→ Cùng store nhưng hai component thấy giá trị khác nhau = Tearing!
```

React giải quyết với **useSyncExternalStore** — hook cho phép external stores (Redux, Zustand...) đăng ký với React để đảm bảo consistency — tính nhất quán trong concurrent renders. Các thư viện state management hiện đại đều đã tích hợp.

---

**Q: startTransition có hoạt động với async operations không?**

A:
- **React 18**: Không — `startTransition` callback phải là synchronous (đồng bộ). Async code bên trong sẽ không được treat như transition sau khi await.
- **React 19**: Có — `startTransition` hỗ trợ async functions. Toàn bộ async operation (kể cả sau await) được treat như transition.

```jsx
// React 19 — async transition
startTransition(async () => {
  const data = await fetchData(query); // Async!
  setResults(data); // Vẫn là transition, không blocking UI
});
```

---

## 🔗 Điều Hướng

- **Trước đó:** [3-virtualization.md](./3-virtualization.md) — Virtualize danh sách dài
- **Tiếp theo:** [5-profiling.md](./5-profiling.md) — React DevTools Profiler
- **Liên quan:** [02-hooks/5-advanced-hooks.md](../02-hooks/5-advanced-hooks.md) — useTransition, useDeferredValue chi tiết
- **Chỉ mục:** [INDEX.md](../INDEX.md)
