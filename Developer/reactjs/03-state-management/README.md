# 03 — State Management (Quản Lý Trạng Thái Toàn Cục)

> State management (Quản Lý Trạng Thái) là một trong những chủ đề được hỏi nhiều nhất trong phỏng vấn React. Phần này bao gồm Context API, Redux Toolkit, RTK Query, Zustand, Jotai — từ giải pháp tích hợp sẵn đến thư viện bên thứ ba.

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Phân biệt **local state** (trạng thái cục bộ) vs **global state** (trạng thái toàn cục) và biết khi nào dùng loại nào
- [ ] Triển khai **Context API** (API Ngữ Cảnh) đúng cách, tránh re-render (hiển thị lại) không cần thiết
- [ ] Sử dụng **Redux Toolkit — RTK** (Bộ Công Cụ Redux Chính Thức) với `createSlice`, `createAsyncThunk`
- [ ] Fetch và cache dữ liệu server với **RTK Query** (Query Của Redux Toolkit)
- [ ] Quản lý global state đơn giản với **Zustand** (thư viện state nhẹ, hiện đại)
- [ ] Hiểu **atomic state management** (quản lý trạng thái nguyên tử) với **Jotai**
- [ ] Chọn đúng giải pháp state management cho từng dự án thực tế

---

## 📁 Nội Dung

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-context-api.md](./1-context-api.md) | Context API — giải pháp tích hợp sẵn trong React | Intermediate |
| [2-redux-toolkit.md](./2-redux-toolkit.md) | Redux Toolkit (RTK) — chuẩn công nghiệp | Intermediate |
| [3-rtk-query.md](./3-rtk-query.md) | RTK Query — data fetching & caching tích hợp Redux | Intermediate–Advanced |
| [4-zustand.md](./4-zustand.md) | Zustand — nhẹ, đơn giản, hiện đại | Intermediate |
| [5-jotai.md](./5-jotai.md) | Jotai — atomic state management | Intermediate–Advanced |
| [6-comparison-guide.md](./6-comparison-guide.md) | Khi nào dùng gì — hướng dẫn lựa chọn | Mọi cấp độ |

---

## 🗺️ Lộ Trình Học

```
Context API  →  Redux Toolkit  →  RTK Query  →  Zustand  →  Jotai  →  So sánh & Lựa chọn
     ↓               ↓               ↓             ↓           ↓              ↓
"Tích hợp sẵn"  "Chuẩn công nghiệp"  "Server state"  "Đơn giản"  "Atomic"  "Khi nào dùng gì?"
```

**Thứ tự khuyến nghị:**
1. Bắt đầu với `1-context-api.md` để hiểu vấn đề Context API giải quyết
2. Học `2-redux-toolkit.md` — phổ biến nhất trong môi trường doanh nghiệp
3. Học `3-rtk-query.md` — mở rộng RTK cho server state
4. Học `4-zustand.md` — lựa chọn hiện đại, ngày càng phổ biến
5. Tham khảo `5-jotai.md` — nếu dự án cần fine-grained reactivity
6. Đọc `6-comparison-guide.md` để biết cách chọn phù hợp

---

## ⏱️ Thời Gian Ước Tính

| Hoạt Động | Thời Gian |
| --------- | --------- |
| Đọc lý thuyết (6 files) | 4–5 giờ |
| Thực hành code theo ví dụ | 3–4 giờ |
| Mini-project: Shopping Cart | 3–4 giờ |
| **Tổng cộng** | **10–13 giờ** |

---

## 💡 Vấn Đề State Management Giải Quyết

### Prop Drilling (Khoan Truyền Props)

Khi ứng dụng lớn dần, việc truyền dữ liệu từ component cha xuống component con nhiều cấp trở nên cồng kềnh — gọi là **prop drilling**.

```
App (có user data)
  ↓ truyền props
  Layout
    ↓ truyền props
    Sidebar
      ↓ truyền props
      UserAvatar  ← component thực sự cần user data
```

Giải pháp: **Global State** (Trạng Thái Toàn Cục) — lưu trữ ở một nơi trung tâm, bất kỳ component nào cũng có thể truy cập trực tiếp.

### Phân Loại State

| Loại State | Ví Dụ | Giải Pháp |
| ---------- | ----- | --------- |
| **Local state** (trạng thái cục bộ) | Form input, modal open/close | `useState`, `useReducer` |
| **Global UI state** (trạng thái UI toàn cục) | Theme, language, sidebar | Context API, Zustand |
| **Server state** (trạng thái máy chủ) | API data, user profile | TanStack Query, RTK Query, SWR |
| **URL state** (trạng thái URL) | Query params, route params | React Router |
| **Form state** (trạng thái biểu mẫu) | Form validation, submission | React Hook Form, Formik |

### Quy Tắc Vàng

> **"Đừng dùng global state cho mọi thứ."** Chỉ nâng state lên global khi thực sự cần nhiều component không liên quan nhau cùng truy cập.

---

## 📊 So Sánh Nhanh Các Giải Pháp

| Tiêu Chí | Context API | Redux Toolkit | Zustand | Jotai |
| -------- | ----------- | ------------- | ------- | ----- |
| **Bundle size** (kích thước gói) | 0 KB (tích hợp sẵn) | ~11 KB | ~1 KB | ~3 KB |
| **Boilerplate** (mã soạn sẵn) | Thấp | Cao (nhưng RTK giảm đáng kể) | Rất thấp | Rất thấp |
| **DevTools** (công cụ phát triển) | ❌ | ✅ Xuất sắc | ✅ Tốt | ✅ Có |
| **Server state** | ❌ | ✅ (RTK Query) | ❌ (cần thêm) | ❌ (cần thêm) |
| **TypeScript** | Tốt | Xuất sắc | Xuất sắc | Xuất sắc |
| **Learning curve** (độ khó học) | Thấp | Cao | Thấp | Trung bình |
| **Phù hợp với** | App nhỏ–vừa | App lớn/enterprise | App mọi quy mô | Fine-grained reactivity |

---

## 🔨 Mini-Project: Shopping Cart (Giỏ Hàng)

Sau khi hoàn thành cả 6 file, hãy xây dựng **Shopping Cart** với yêu cầu:

```
✅ Hiển thị danh sách sản phẩm
✅ Thêm / Xóa sản phẩm vào giỏ hàng
✅ Cập nhật số lượng sản phẩm
✅ Tính tổng giá tiền
✅ Lưu giỏ hàng vào localStorage (browser storage — lưu trữ trình duyệt)
```

**Thực hiện 3 lần với 3 giải pháp khác nhau:**
1. Chỉ dùng Context API + useReducer
2. Dùng Redux Toolkit
3. Dùng Zustand

So sánh trải nghiệm viết code và nhận xét ưu/nhược điểm.

---

## 🔗 Điều Hướng

- **Trước đó:** [02-hooks/](../02-hooks/) — React Hooks chuyên sâu
- **Tiếp theo:** [04-routing/](../04-routing/) — Điều hướng trong React
- **Quay lại:** [README.md tổng quan](../README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
