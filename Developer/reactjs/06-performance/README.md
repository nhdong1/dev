# 06 — Performance (Tối Ưu Hiệu Năng React)

> **Performance Optimization** (Tối Ưu Hiệu Năng) là quá trình xác định và loại bỏ các điểm nghẽn cổ chai — bottleneck — trong ứng dụng React để đạt trải nghiệm người dùng mượt mà, phản hồi nhanh. Nguyên tắc vàng: **đo lường trước, tối ưu sau** — "profile first, optimize second".

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Áp dụng đúng **React.memo**, **useMemo**, **useCallback** — biết khi nào nên và không nên dùng
- [ ] Triển khai **Code Splitting** (Tách Code) với `React.lazy` và **dynamic import** (nhập động)
- [ ] **Virtualize** (Ảo Hóa) danh sách dài bằng `react-window` hoặc `@tanstack/virtual`
- [ ] Sử dụng **Concurrent Features** (Tính Năng Đồng Thời) — `startTransition`, `useDeferredValue`, `Suspense`
- [ ] Dùng **React DevTools Profiler** (Trình Phân Tích Hiệu Năng) để tìm component bị render lại thừa
- [ ] Tối ưu **Bundle Size** (Kích Thước Gói) — tree-shaking, lazy loading, phân tích bundle

---

## 📁 Nội Dung

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-memoization.md](./1-memoization.md) | React.memo, useMemo, useCallback — ghi nhớ để tránh render thừa | Intermediate |
| [2-code-splitting.md](./2-code-splitting.md) | Code Splitting, React.lazy, dynamic import, Suspense | Intermediate |
| [3-virtualization.md](./3-virtualization.md) | Virtualize danh sách dài với react-window, @tanstack/virtual | Intermediate–Advanced |
| [4-concurrent-features.md](./4-concurrent-features.md) | Concurrent Mode, startTransition, useDeferredValue, Suspense | Advanced |
| [5-profiling.md](./5-profiling.md) | React DevTools Profiler, tìm bottleneck, đo lường hiệu năng | Intermediate |
| [6-bundle-optimization.md](./6-bundle-optimization.md) | Bundle size, tree-shaking, lazy loading, phân tích bundle | Intermediate–Advanced |

---

## 🗺️ Lộ Trình Học

```
Profiler (đo lường)  →  Memoization  →  Code Splitting  →  Virtualization  →  Concurrent Features  →  Bundle Optimization
        ↓                   ↓                ↓                   ↓                    ↓                       ↓
  "Tìm vấn đề        "React.memo,      "React.lazy,        "react-window,       "startTransition,      "Tree-shaking,
   trước khi         useMemo,          dynamic import,      tanstack/virtual"    useDeferredValue,      code splitting,
   tối ưu"           useCallback"      Suspense"                                 Suspense"              bundle analysis"
```

**Thứ tự khuyến nghị:**
1. Bắt đầu với `5-profiling.md` — học cách đo lường trước khi tối ưu mù quáng
2. Học `1-memoization.md` — kỹ thuật phổ biến nhất, hay bị dùng sai
3. Học `2-code-splitting.md` — giảm thời gian tải trang ban đầu
4. Học `3-virtualization.md` — xử lý danh sách hàng nghìn phần tử
5. Đọc `4-concurrent-features.md` — kỹ thuật nâng cao từ React v18+
6. Tham khảo `6-bundle-optimization.md` — tối ưu kích thước bundle tổng thể

---

## ⏱️ Thời Gian Ước Tính

| Hoạt Động | Thời Gian |
| --------- | --------- |
| Đọc lý thuyết (6 files) | 4–5 giờ |
| Thực hành code theo ví dụ | 3–4 giờ |
| Mini-project: Tối ưu app có vấn đề | 3–4 giờ |
| **Tổng cộng** | **10–13 giờ** |

---

## 💡 Hiệu Năng Là Gì Và Tại Sao Quan Trọng?

### Hiệu Năng Ảnh Hưởng Trực Tiếp Đến Kinh Doanh

```
Nghiên cứu từ Google và Amazon:
- Mỗi 100ms delay → giảm 1% conversion rate (tỉ lệ chuyển đổi)
- Trang load chậm hơn 1 giây → giảm 7% revenue (doanh thu)
- 53% người dùng mobile rời bỏ nếu tải > 3 giây
```

### Các Loại Vấn Đề Hiệu Năng Phổ Biến

```
1. Unnecessary Re-renders (Render Lại Không Cần Thiết)
   → Component render lại khi props/state không thực sự thay đổi
   → Giải pháp: React.memo, useMemo, useCallback

2. Large Bundle Size (Kích Thước Bundle Lớn)
   → JavaScript quá nhiều → parse & execute lâu → Time to Interactive cao
   → Giải pháp: Code splitting, tree-shaking, lazy loading

3. Long Lists (Danh Sách Quá Dài)
   → Render 10,000 DOM nodes → trình duyệt bị chậm
   → Giải pháp: Virtualization (chỉ render phần tử đang thấy)

4. Expensive Calculations (Tính Toán Tốn Kém)
   → Filter/sort/transform dữ liệu lớn trong render
   → Giải pháp: useMemo để cache kết quả

5. Blocking UI Updates (Cập Nhật Giao Diện Bị Chặn)
   → Tìm kiếm, filter real-time làm UI giật lag
   → Giải pháp: startTransition, useDeferredValue
```

---

## 📏 Nguyên Tắc Vàng: Đo Lường Trước

### ❌ Anti-pattern — Mẫu Chống Hiệu Quả: Tối Ưu Mù Quáng

```jsx
// Bọc tất cả mọi thứ bằng memo "cho chắc"
const MyComponent = React.memo(({ name }) => {
  const result = useMemo(() => name.toUpperCase(), [name]); // Không cần thiết!
  const handleClick = useCallback(() => {}, []); // Không cần thiết!
  return <div onClick={handleClick}>{result}</div>;
});
```

Vấn đề: `useMemo` và `useCallback` **cũng có chi phí** — bộ nhớ để lưu giá trị cũ, so sánh dependencies mỗi render. Nếu không có vấn đề thực sự, tối ưu này làm code phức tạp hơn mà không đem lại lợi ích.

### ✅ Quy Trình Đúng

```
Bước 1: Xác định vấn đề (đo lường)
  → Dùng React DevTools Profiler
  → Xác định component nào render lâu, render thừa

Bước 2: Tìm nguyên nhân gốc rễ
  → Re-render do props reference thay đổi? → React.memo
  → Tính toán nặng trong render? → useMemo
  → Callback mới mỗi render? → useCallback
  → Danh sách dài? → Virtualization
  → Bundle lớn? → Code splitting

Bước 3: Áp dụng giải pháp

Bước 4: Đo lường lại → xác nhận cải thiện
```

---

## 📊 Công Cụ Đo Lường Hiệu Năng

### React DevTools Profiler

Công cụ tích hợp trong **React Developer Tools** — Extension Công Cụ Phát Triển React:

```
Cài đặt: Chrome/Firefox Extension "React Developer Tools"
Sử dụng: Tab "Profiler" → Record → Thao tác trên UI → Stop → Phân tích
```

### Web Vitals — Chỉ Số Trải Nghiệm Web Cốt Lõi

| Chỉ Số | Tên Đầy Đủ | Ý Nghĩa | Ngưỡng Tốt |
| ------ | ---------- | ------- | ---------- |
| **LCP** | Largest Contentful Paint — Thời Gian Hiển Thị Nội Dung Lớn Nhất | Tốc độ tải nội dung chính | < 2.5s |
| **FID** | First Input Delay — Độ Trễ Tương Tác Đầu Tiên | Khả năng phản hồi input | < 100ms |
| **CLS** | Cumulative Layout Shift — Độ Dịch Chuyển Layout Tích Lũy | Độ ổn định layout | < 0.1 |
| **INP** | Interaction to Next Paint — Thời Gian Từ Tương Tác Đến Vẽ | Độ mượt mà tổng thể | < 200ms |
| **FCP** | First Contentful Paint — Thời Gian Hiển Thị Nội Dung Đầu Tiên | Tốc độ phản hồi ban đầu | < 1.8s |
| **TTFB** | Time to First Byte — Thời Gian Nhận Byte Đầu Tiên | Tốc độ server | < 800ms |

### Lighthouse — Công Cụ Kiểm Tra Chất Lượng Web

```bash
# Chạy Lighthouse trong terminal
npx lighthouse https://your-app.com --output html --output-path report.html

# Hoặc dùng Chrome DevTools → Lighthouse tab
```

---

## 🔑 Tổng Quan Các Kỹ Thuật

### Memoization — Ghi Nhớ Kết Quả

| Kỹ Thuật | Mục Đích | Dùng Khi |
| -------- | -------- | -------- |
| `React.memo` | Bỏ qua re-render khi props không đổi | Component render tốn kém, nhận props nguyên thủy |
| `useMemo` | Cache kết quả tính toán | Tính toán nặng (filter, sort, transform dữ liệu lớn) |
| `useCallback` | Cache reference của hàm | Hàm được truyền xuống component con đã wrap bằng `React.memo` |

### Code Splitting — Tách Code

```
Vấn đề: Bundle 2MB → người dùng phải download và parse 2MB JavaScript trước khi thấy bất cứ gì

Giải pháp: Chia nhỏ bundle → chỉ tải code cần thiết cho trang hiện tại
  - Route-based splitting: mỗi route = 1 chunk riêng
  - Component-based splitting: Modal, Chart, Editor → tải khi cần
  - Vendor splitting: thư viện bên thứ ba → chunk riêng có thể cache lâu dài
```

### Virtualization — Ảo Hóa

```
Vấn đề: 10,000 phần tử → 10,000 DOM nodes → layout/paint tốn kém

Giải pháp: Chỉ render DOM nodes đang hiển thị trong viewport (vùng nhìn thấy)
  - 10,000 phần tử → thực tế chỉ render ~20-30 phần tử
  - Khi scroll → thay thế nội dung phần tử, không tạo mới DOM
```

### Concurrent Features — Tính Năng Đồng Thời

```
React v18+ cho phép ưu tiên hóa các cập nhật UI:
  - Urgent updates (Cập Nhật Khẩn Cấp): typing, clicking → phải phản hồi ngay
  - Non-urgent updates (Cập Nhật Không Khẩn Cấp): filter results, re-sort → có thể trì hoãn

startTransition(() => { setFilter(value); }); // Đánh dấu là non-urgent
```

---

## 🔨 Mini-Project: Tối Ưu Ứng Dụng Có Vấn Đề

Sau khi hoàn thành cả 6 file, hãy thực hành với ứng dụng Dashboard có vấn đề hiệu năng:

```
Bước 1: Xây dựng ứng dụng với các vấn đề cố ý
  ✅ Danh sách 5,000 sản phẩm (không virtualize)
  ✅ Filter theo tên sản phẩm (không debounce)
  ✅ Chart phức tạp tải ở màn hình chính (không lazy load)
  ✅ Component con re-render mỗi khi parent render

Bước 2: Đo lường bằng Profiler → ghi lại baseline

Bước 3: Áp dụng từng kỹ thuật → đo lại
  → Virtualize danh sách sản phẩm
  → useMemo cho filter logic
  → React.lazy cho Chart component
  → React.memo cho ProductCard

Bước 4: So sánh trước/sau → báo cáo cải thiện
```

---

## 📊 So Sánh Kỹ Thuật Theo Use Case

| Vấn Đề | Kỹ Thuật | Độ Phức Tạp | Hiệu Quả |
| ------- | -------- | ----------- | -------- |
| Component re-render thừa | `React.memo` | Thấp | Trung bình |
| Tính toán nặng trong render | `useMemo` | Thấp | Cao |
| Callback reference thay đổi | `useCallback` | Thấp | Thấp–Trung bình |
| Bundle quá lớn | `React.lazy` + dynamic import | Trung bình | Rất cao |
| Danh sách dài (> 500 items) | Virtualization | Trung bình | Rất cao |
| UI giật khi tìm kiếm | `startTransition` | Thấp | Cao |
| Hiển thị dữ liệu cũ khi fetch | `useDeferredValue` | Thấp | Cao |
| Bundle phình to theo thời gian | Tree-shaking + Bundle analysis | Cao | Rất cao |

---

## 🔗 Điều Hướng

- **Trước đó:** [05-data-fetching/](../05-data-fetching/) — Lấy và đồng bộ dữ liệu
- **Tiếp theo:** [07-testing/](../07-testing/) — Kiểm thử React
- **Quay lại:** [README.md tổng quan](../README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
