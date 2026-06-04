# Hooks Nâng Cao — useId, useTransition, useDeferredValue, useDebugValue

> Đây là các hooks được giới thiệu trong React 18 nhằm tối ưu **UX** (User Experience — Trải Nghiệm Người Dùng) trong ứng dụng phức tạp. Chúng khai thác **Concurrent Mode** (Chế Độ Đồng Thời) — khả năng của React 18+ trong việc ưu tiên render và giữ UI phản hồi ngay cả khi có tác vụ nặng.

---

## 📌 Mục Lục

1. [useId — Tạo ID Duy Nhất](#1-useid)
2. [useTransition — Đánh Dấu Update Không Khẩn Cấp](#2-usetransition)
3. [useDeferredValue — Trì Hoãn Giá Trị](#3-usedeferredvalue)
4. [useDebugValue — Debug Label Trong DevTools](#4-usedebugvalue)
5. [useTransition vs useDeferredValue](#5-so-sánh-usetransition-vs-usedeferredvalue)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. useId — Tạo ID Duy Nhất

### Vấn Đề Cần Giải Quyết

HTML accessibility (khả năng tiếp cận) yêu cầu `id` duy nhất để liên kết `<label>` với `<input>`. Trong React, việc tự đặt ID cứng (hardcode) gây vấn đề:

```jsx
// ❌ Vấn đề: Nếu dùng FormField nhiều lần, id "email" sẽ bị trùng
function FormField({ label, type }) {
  return (
    <div>
      <label htmlFor="email">{label}</label>
      <input id="email" type={type} />
    </div>
  );
}

function Form() {
  return (
    <>
      <FormField label="Email" type="email" />   {/* id="email" ← trùng! */}
      <FormField label="Email xác nhận" type="email" /> {/* id="email" ← trùng! */}
    </>
  );
}
```

### Giải Pháp Với useId

```jsx
import { useId } from "react";

function FormField({ label, type }) {
  // useId tạo ID duy nhất, ổn định qua renders, an toàn với SSR
  const id = useId();

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type={type} />
    </div>
  );
}

function Form() {
  return (
    <>
      {/* Mỗi FormField có ID riêng: ":r0:", ":r1:", ... */}
      <FormField label="Email" type="email" />
      <FormField label="Email xác nhận" type="email" />
    </>
  );
}
```

### Một ID Cho Nhiều Elements Liên Quan

```jsx
function PasswordField() {
  const id = useId();
  const inputId = `${id}-input`;      // ":r2:-input"
  const descId = `${id}-description`; // ":r2:-description"
  const errorId = `${id}-error`;      // ":r2:-error"

  return (
    <div>
      <label htmlFor={inputId}>Mật khẩu</label>
      <input
        id={inputId}
        type="password"
        aria-describedby={`${descId} ${errorId}`}
      />
      <p id={descId}>Ít nhất 8 ký tự, gồm chữ và số</p>
      <p id={errorId} role="alert">Mật khẩu quá ngắn</p>
    </div>
  );
}
```

### Tại Sao Không Dùng Math.random() hay Counter?

```jsx
// ❌ Math.random(): ID thay đổi mỗi render → accessibility links bị vỡ
const id = `field-${Math.random()}`;

// ❌ Module-level counter: không hoạt động với SSR (Server-Side Rendering)
let counter = 0;
const id = `field-${counter++}`;

// ✅ useId: ổn định qua renders, hoạt động với SSR, hydration-safe
const id = useId();
```

> **useId và SSR:** `useId` tạo ID nhất quán giữa server render và client hydration — tránh **hydration mismatch** (sự không khớp khi client-side React tiếp quản từ server HTML).

---

## 2. useTransition — Đánh Dấu Update Không Khẩn Cấp

### Vấn Đề: UI Bị Chặn Bởi Render Nặng

```jsx
// ❌ Vấn đề: Gõ phím → filter 50.000 items → UI bị đơ (lag)
function SearchPage({ items }) {
  const [query, setQuery] = useState("");

  // Filter ngay lập tức khi query thay đổi → blocking render
  const results = items.filter(item =>
    item.name.toLowerCase().includes(query.toLowerCase())
  );

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)} // Gõ phím bị lag!
        placeholder="Tìm kiếm..."
      />
      <ResultList items={results} />
    </div>
  );
}
```

### Giải Pháp Với useTransition

**useTransition** — Móc Chuyển Trạng Thái — cho phép đánh dấu một số state updates là **không khẩn cấp** (low priority). React ưu tiên xử lý updates khẩn cấp (gõ phím, click) trước, updates không khẩn cấp có thể bị trì hoãn.

```jsx
import { useState, useTransition } from "react";

function SearchPage({ items }) {
  const [query, setQuery] = useState("");
  const [filteredItems, setFilteredItems] = useState(items);
  const [isPending, startTransition] = useTransition();
  // isPending — Đang Chờ — true khi transition đang chờ xử lý
  // startTransition — Bắt Đầu Chuyển — hàm bọc update không khẩn cấp

  function handleSearch(e) {
    const value = e.target.value;
    setQuery(value); // ← Khẩn cấp: update input ngay lập tức

    startTransition(() => {
      // ← Không khẩn cấp: filter có thể bị interrupt (gián đoạn)
      const results = items.filter(item =>
        item.name.toLowerCase().includes(value.toLowerCase())
      );
      setFilteredItems(results);
    });
  }

  return (
    <div>
      <input
        value={query}
        onChange={handleSearch}
        placeholder="Tìm kiếm..."
      />
      {/* Hiển thị trạng thái loading khi filter đang xử lý */}
      {isPending && <p>Đang lọc kết quả...</p>}
      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        <p>{filteredItems.length} kết quả</p>
        <ResultList items={filteredItems} />
      </div>
    </div>
  );
}
```

### useTransition Cho Navigation (Điều Hướng)

```jsx
import { useState, useTransition, Suspense } from "react";

function TabContainer({ tabs }) {
  const [activeTab, setActiveTab] = useState(tabs[0].id);
  const [isPending, startTransition] = useTransition();

  function switchTab(tabId) {
    startTransition(() => {
      setActiveTab(tabId); // Tab content có thể mất thời gian để load
    });
  }

  return (
    <div>
      <nav>
        {tabs.map(tab => (
          <button
            key={tab.id}
            onClick={() => switchTab(tab.id)}
            style={{
              fontWeight: activeTab === tab.id ? "bold" : "normal",
              opacity: isPending ? 0.7 : 1, // Fade nhẹ khi đang chuyển tab
            }}
          >
            {tab.label}
            {isPending && activeTab === tab.id && " (đang tải...)"}
          </button>
        ))}
      </nav>
      <Suspense fallback={<p>Đang tải nội dung tab...</p>}>
        <TabContent tabId={activeTab} />
      </Suspense>
    </div>
  );
}
```

### Quy Tắc Dùng useTransition

```
✅ Nên dùng cho:
  - Filter/search danh sách lớn
  - Chuyển tab, route navigation với Suspense
  - Cập nhật biểu đồ, visualizations phức tạp
  - Bất kỳ update UI nào "có thể chờ" một chút

❌ Không dùng cho:
  - Gõ phím vào input (đây là update khẩn cấp!)
  - Hover effects
  - Controlled inputs
  - Animations cần tức thì
```

---

## 3. useDeferredValue — Trì Hoãn Giá Trị

### Khái Niệm

`useDeferredValue` — Hook Giá Trị Trì Hoãn — nhận một giá trị và trả về phiên bản **trì hoãn** (deferred) của nó. Khi giá trị gốc thay đổi, React giữ giá trị cũ (stale — lỗi thời) để hiển thị, trong khi âm thầm render phiên bản mới ở background (nền).

```jsx
import { useState, useDeferredValue } from "react";

function SearchResults({ items }) {
  const [query, setQuery] = useState("");

  // deferredQuery cập nhật sau query — giữ UI responsive
  const deferredQuery = useDeferredValue(query);

  // render với deferredQuery — có thể "lỗi thời" một chút
  const results = filterItems(items, deferredQuery);

  const isStale = deferredQuery !== query; // Đang hiển thị kết quả cũ

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Tìm kiếm..."
      />
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        {/* Nếu isStale: hiển thị kết quả cũ với opacity thấp hơn */}
        <p>{results.length} kết quả cho "{deferredQuery}"</p>
        {results.map(item => (
          <div key={item.id}>{item.name}</div>
        ))}
      </div>
    </div>
  );
}
```

### useDeferredValue Với Memo (Bộ Nhớ Đệm)

Để `useDeferredValue` hoạt động hiệu quả, component nhận deferred value cần được bọc bằng `memo`:

```jsx
import { memo, useDeferredValue } from "react";

// ✅ Bọc bằng memo: chỉ re-render khi deferredQuery thực sự thay đổi
const HeavyList = memo(function HeavyList({ query, allItems }) {
  const results = allItems.filter(item =>
    item.name.toLowerCase().includes(query.toLowerCase())
  );
  return (
    <ul>
      {results.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
});

function SearchPage({ allItems }) {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
      />
      {/* HeavyList chỉ re-render khi deferredQuery thay đổi */}
      <HeavyList query={deferredQuery} allItems={allItems} />
    </div>
  );
}
```

---

## 4. useDebugValue — Debug Label Trong DevTools

`useDebugValue` — Hook Nhãn Debug — thêm nhãn (label) tùy chỉnh cho Custom Hooks trong **React DevTools** (Công Cụ Phát Triển React). Chỉ dùng trong Custom Hooks.

### Cú Pháp

```jsx
useDebugValue(value);
// Hoặc với formatter (hàm định dạng) — chỉ chạy khi DevTools mở
useDebugValue(value, value => formatValue(value));
```

### Ví Dụ

```jsx
import { useState, useEffect, useDebugValue } from "react";

function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    function handleOnline() { setIsOnline(true); }
    function handleOffline() { setIsOnline(false); }

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  // Label hiển thị trong DevTools: "Online Status: Trực tuyến" hoặc "Ngoại tuyến"
  useDebugValue(isOnline, online => online ? "Trực tuyến" : "Ngoại tuyến");

  return isOnline;
}

function useFormattedDate(timestamp) {
  const date = new Date(timestamp);

  // Formatter chỉ chạy khi DevTools panel mở — tránh chi phí không cần thiết
  useDebugValue(date, d => d.toLocaleDateString("vi-VN", {
    weekday: "long",
    year: "numeric",
    month: "long",
    day: "numeric",
  }));

  return date;
}
```

### Khi Nào Dùng useDebugValue?

```
✅ Nên thêm useDebugValue khi:
  - Custom Hook được chia sẻ trong team hoặc thư viện
  - State phức tạp cần context để hiểu
  - Hook có nhiều trạng thái (loading, error, success...)

❌ Không cần khi:
  - Hook đơn giản, tên tự mô tả đủ
  - Hook chỉ dùng nội bộ, không chia sẻ
```

---

## 5. So Sánh useTransition vs useDeferredValue

| Tiêu Chí | `useTransition` | `useDeferredValue` |
| -------- | --------------- | ------------------ |
| **Kiểm soát** | Bạn kiểm soát **code nào** là không khẩn cấp | Bạn kiểm soát **giá trị nào** được trì hoãn |
| **Khi có quyền** | Khi có thể truy cập vào hàm `setState` | Khi chỉ nhận giá trị qua props |
| **isPending** | Có `isPending` để hiển thị loading indicator | Phải tự so sánh `value !== deferredValue` |
| **Dùng khi** | Bạn sở hữu state update logic | Bạn nhận props từ bên ngoài |
| **Ví dụ** | Navigation, tab switching | Search input từ parent |

### Ví Dụ Minh Họa Sự Khác Biệt

```jsx
// useTransition — Bạn kiểm soát setState
function SearchWithTransition() {
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(query) {
    startTransition(() => {
      setResults(computeResults(query)); // ← Bạn kiểm soát setState này
    });
  }

  return <SearchInput onSearch={handleSearch} pending={isPending} />;
}

// useDeferredValue — Bạn chỉ có giá trị từ props
function SearchResults({ query }) { // ← Nhận query từ bên ngoài
  const deferredQuery = useDeferredValue(query); // ← Không có setState nào để wrap

  const results = useMemo(
    () => computeResults(deferredQuery),
    [deferredQuery]
  );

  return <ResultList items={results} />;
}
```

---

## 6. Câu Hỏi Phỏng Vấn

### Q1: Concurrent Mode là gì và các hooks này liên quan như thế nào?

**Trả lời:** **Concurrent Mode** (Chế Độ Đồng Thời) là khả năng của React 18+ trong việc render component ở background mà không block (chặn) main thread. React có thể bắt đầu render, tạm dừng, và resume sau. `useTransition` và `useDeferredValue` khai thác khả năng này:
- Khi một transition bị interrupt bởi update khẩn cấp hơn, React bỏ phần đã render và bắt đầu lại
- Người dùng không thấy UI trung gian — chỉ thấy kết quả cuối
- Main thread luôn available cho interactions quan trọng

### Q2: useTransition có làm cho app nhanh hơn không?

**Trả lời:** Không theo nghĩa tuyệt đối — tổng lượng work (công việc) không giảm. Nhưng nó làm **UI responsive hơn** bằng cách ưu tiên: updates quan trọng (input, click) được xử lý ngay; updates ít quan trọng hơn (render kết quả filter) được thực hiện khi CPU rảnh. Người dùng cảm thấy app nhanh hơn dù lượng tính toán như cũ.

### Q3: Sự khác nhau giữa useDeferredValue và debouncing?

**Trả lời:**
- **Debouncing** (Giảm Tần Số): Trì hoãn cố định (ví dụ: 300ms) — luôn chờ dù máy nhanh hay chậm
- **useDeferredValue**: Thích nghi — trên máy nhanh, kết quả xuất hiện gần như ngay lập tức; trên máy chậm, React tự điều chỉnh. Không có delay cứng.

### Q4: Tại sao useId an toàn với SSR trong khi Math.random() không?

**Trả lời:** Trong SSR (Server-Side Rendering), React render HTML trên server, sau đó client-side React "tiếp quản" (hydration). Nếu dùng `Math.random()`:
- Server: `id="field-0.123456"` (số ngẫu nhiên)
- Client: `id="field-0.789012"` (số ngẫu nhiên khác)
- Kết quả: **Hydration mismatch** — React không khớp server HTML với client DOM

`useId` sử dụng thuật toán deterministic (xác định) dựa trên vị trí component trong component tree — luôn cho cùng kết quả trên server và client.

### Q5: Khi nào startTransition và khi nào dùng setTimeout để defer update?

**Trả lời:** `startTransition` **tốt hơn** `setTimeout` vì:
1. **Semantic**: React biết đây là low-priority update, không phải "chờ arbitrarily"
2. **Interruptible**: Transition có thể bị React interrupt khi có urgent update; setTimeout không thể
3. **isPending**: startTransition cung cấp `isPending` state; setTimeout không
4. **No delay**: Transition chạy ngay khi CPU rảnh; setTimeout luôn chờ ít nhất 0ms (thực tế 4ms+ do browser clamping)

---

## 🔗 Điều Hướng

- **Trở về:** [README.md](./README.md) — Tổng quan Hooks
- **Trước đó:** [4-usememo-usecallback.md](./4-usememo-usecallback.md) — useMemo và useCallback
- **Tiếp theo:** [6-react19-new-hooks.md](./6-react19-new-hooks.md) — React v19 Hooks Mới

**Cập Nhật Lần Cuối:** 2026-06-04 | **React Version:** v19 (stable)
