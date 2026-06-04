# 05 — Data Fetching (Lấy và Đồng Bộ Dữ Liệu)

> Data fetching (Lấy Dữ Liệu) là thách thức trung tâm của mọi ứng dụng web hiện đại. Phần này bao gồm TanStack Query, SWR, React Server Components (RSC — Component Phía Server), và Server Actions (Hành Động Phía Server) — từ chiến lược client-side đến server-side rendering.

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Phân biệt **client state** (trạng thái cục bộ) vs **server state** (trạng thái máy chủ) và tại sao cần thư viện riêng
- [ ] Sử dụng **TanStack Query** (React Query) với `useQuery`, `useMutation`, caching và invalidation
- [ ] Áp dụng chiến lược **Optimistic Updates** (Cập Nhật Lạc Quan) để cải thiện UX
- [ ] Dùng **SWR** — Stale-While-Revalidate — thư viện data fetching nhẹ từ Vercel
- [ ] Hiểu và viết **React Server Components (RSC)** — component chạy hoàn toàn trên server
- [ ] Triển khai **Server Actions** (React v19 & Next.js) để gọi hàm server trực tiếp từ client
- [ ] Chọn đúng chiến lược fetching cho từng use case (SSR, SSG, ISR, CSR, streaming)

---

## 📁 Nội Dung

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-tanstack-query-basics.md](./1-tanstack-query-basics.md) | TanStack Query — useQuery, useMutation, cơ bản | Intermediate |
| [2-tanstack-query-advanced.md](./2-tanstack-query-advanced.md) | Caching, Invalidation, Optimistic Updates nâng cao | Intermediate–Advanced |
| [3-swr.md](./3-swr.md) | SWR — Stale-While-Revalidate từ Vercel | Intermediate |
| [4-react-server-components.md](./4-react-server-components.md) | RSC — React Server Components | Advanced |
| [5-server-actions.md](./5-server-actions.md) | Server Actions (React v19 & Next.js) | Advanced |

---

## 🗺️ Lộ Trình Học

```
TanStack Query Basics → TanStack Query Advanced → SWR → RSC → Server Actions
        ↓                       ↓                  ↓     ↓          ↓
  "useQuery cơ bản"     "Caching & Optimistic"  "Nhẹ hơn"  "Trên server"  "Gọi server từ form"
```

**Thứ tự khuyến nghị:**
1. Bắt đầu với `1-tanstack-query-basics.md` — hiểu server state management (quản lý trạng thái máy chủ)
2. Học `2-tanstack-query-advanced.md` — caching strategies, optimistic updates nâng cao
3. Đọc `3-swr.md` — so sánh với TanStack Query, biết khi nào dùng
4. Học `4-react-server-components.md` — mô hình mới, quan trọng với Next.js App Router
5. Hoàn thành `5-server-actions.md` — tính năng React v19, thay thế API routes trong nhiều trường hợp

---

## ⏱️ Thời Gian Ước Tính

| Hoạt Động | Thời Gian |
| --------- | --------- |
| Đọc lý thuyết (5 files) | 4–5 giờ |
| Thực hành code theo ví dụ | 3–4 giờ |
| Mini-project: GitHub User Explorer | 3–4 giờ |
| **Tổng cộng** | **10–13 giờ** |

---

## 💡 Vấn Đề Data Fetching Giải Quyết

### Server State vs Client State

Đây là sự phân biệt quan trọng nhất trong data fetching hiện đại:

| Đặc Điểm | Client State (Trạng Thái Cục Bộ) | Server State (Trạng Thái Máy Chủ) |
| -------- | -------------------------------- | ---------------------------------- |
| **Lưu trữ** | Trong bộ nhớ ứng dụng | Trên server/database |
| **Quyền sở hữu** | Ứng dụng kiểm soát hoàn toàn | Server là nguồn sự thật |
| **Đồng bộ** | Không cần | Cần đồng bộ định kỳ |
| **Ví dụ** | Modal open, theme, form input | User profile, product list, orders |
| **Giải pháp** | useState, Zustand | TanStack Query, SWR, RTK Query |

### Thách Thức Của Server State

Khi lấy dữ liệu từ server theo cách thủ công, bạn phải tự xử lý:

```javascript
// ❌ Cách thủ công — nhiều boilerplate (mã soạn sẵn), dễ có lỗi
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err);
        setLoading(false);
      });
  }, [userId]);

  // Không có: caching, background refetching, stale data handling,
  //           request deduplication, optimistic updates, ...
}
```

Các vấn đề phát sinh:
- **No caching** — mỗi lần render là một lần fetch mới
- **No deduplication** — nhiều component cùng fetch dữ liệu giống nhau
- **No background sync** — dữ liệu stale (cũ) không được tự động làm mới
- **No loading/error state management** — phải tự quản lý thủ công
- **No retry logic** — request thất bại không được thử lại
- **Race conditions** — request cũ có thể ghi đè request mới

---

## 📊 So Sánh Nhanh Các Giải Pháp Data Fetching

| Tiêu Chí | TanStack Query | SWR | RTK Query | Apollo Client |
| -------- | -------------- | --- | --------- | ------------- |
| **Bundle size** (kích thước gói) | ~13 KB | ~4 KB | ~11 KB (trong RTK) | ~33 KB |
| **Cache strategy** (chiến lược cache) | Rất linh hoạt | Đơn giản, hiệu quả | Linh hoạt | Chuẩn cho GraphQL |
| **Mutation** (thay đổi dữ liệu) | `useMutation` | `mutate()` | Endpoint mutation | `useMutation` |
| **Optimistic updates** | ✅ Đầy đủ | ✅ Đơn giản | ✅ Đầy đủ | ✅ Có |
| **DevTools** (công cụ phát triển) | ✅ Xuất sắc | ❌ Không | ✅ Redux DevTools | ✅ Apollo Studio |
| **TypeScript** | Xuất sắc | Tốt | Xuất sắc | Tốt |
| **GraphQL** | Có thể dùng | Có thể dùng | ❌ Không native | ✅ Chuyên biệt |
| **Phù hợp với** | Mọi dự án | Dự án Vercel/Next.js | Dự án đã có Redux | GraphQL APIs |

---

## 🏗️ Các Chiến Lược Rendering & Fetching

### Client-Side Rendering (CSR — Hiển Thị Phía Trình Duyệt)

```
Browser → Fetch HTML (rỗng) → Tải JS → React khởi động → Fetch data → Render UI
```
- **Ưu điểm:** Tương tác nhanh sau khi tải xong, dynamic
- **Nhược điểm:** SEO kém, initial load chậm
- **Dùng khi:** Dashboard, app cần authentication, dữ liệu thay đổi thường xuyên

### Server-Side Rendering (SSR — Hiển Thị Phía Máy Chủ)

```
Browser → Request → Server fetch data → Render HTML đầy đủ → Gửi về Browser → Hydration
```
- **Ưu điểm:** SEO tốt, First Contentful Paint (FCP — Thời Gian Hiển Thị Nội Dung Đầu Tiên) nhanh
- **Nhược điểm:** Server phải xử lý nhiều hơn, TTFB (Time to First Byte — Thời Gian Đến Byte Đầu Tiên) có thể chậm
- **Dùng khi:** Trang sản phẩm, blog, trang cần SEO

### Static Site Generation (SSG — Tạo Trang Tĩnh)

```
Build time → Fetch data → Render HTML → Deploy CDN → Browser nhận HTML tĩnh
```
- **Ưu điểm:** Nhanh nhất, CDN-friendly, rẻ nhất
- **Nhược điểm:** Dữ liệu có thể stale (cũ) giữa các lần build
- **Dùng khi:** Blog, docs, landing pages, nội dung ít thay đổi

### Incremental Static Regeneration (ISR — Tái Tạo Tĩnh Dần)

```
Deploy → Serve static → Sau interval → Rebuild page ở background → Serve mới
```
- **Ưu điểm:** Kết hợp tốc độ SSG và tính mới của SSR
- **Nhược điểm:** Có thể phức tạp, cache invalidation khó
- **Dùng khi:** E-commerce, news sites — cần mới nhưng không real-time

### Streaming (Phát Trực Tuyến)

```
Browser → Request → Server gửi HTML theo từng phần → Browser render dần
```
- **Ưu điểm:** Perceived performance (hiệu năng cảm nhận) tốt hơn, TTFB nhanh
- **Nhược điểm:** Phức tạp hơn SSR
- **Dùng khi:** Next.js App Router với React Suspense, dữ liệu nhiều components

---

## 🔨 Mini-Project: GitHub User Explorer

Sau khi hoàn thành cả 5 file, hãy xây dựng **GitHub User Explorer** với yêu cầu:

```
✅ Tìm kiếm user GitHub theo username
✅ Hiển thị profile với avatar, bio, followers
✅ Liệt kê repositories của user (có pagination — phân trang)
✅ Cache kết quả tìm kiếm trong 5 phút
✅ Optimistic update khi star/unstar repository
✅ Loading skeleton (khung xương tải) và error states
✅ Sử dụng TanStack Query cho data fetching
```

**Mở rộng nếu dùng Next.js:**
- Render user profile phía server (RSC)
- Dùng Server Action để submit form tìm kiếm

---

## 🔗 Điều Hướng

- **Trước đó:** [04-routing/](../04-routing/) — Điều hướng trong React
- **Tiếp theo:** [06-performance/](../06-performance/) — Tối ưu hiệu năng
- **Quay lại:** [README.md tổng quan](../README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
