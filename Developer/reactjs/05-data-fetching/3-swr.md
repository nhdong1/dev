# SWR — Stale-While-Revalidate

> **SWR** là thư viện data fetching (lấy dữ liệu) nhẹ từ **Vercel**, lấy tên từ chiến lược cache HTTP **Stale-While-Revalidate** (Phục Vụ Dữ Liệu Cũ Trong Khi Làm Mới). Triết lý: hiển thị data có trong cache ngay lập tức (stale), đồng thời fetch dữ liệu mới ở background (revalidate), rồi cập nhật UI khi có kết quả mới.

---

## 📌 Mục Lục

1. [SWR Là Gì & Khi Nào Dùng](#1-swr-là-gì--khi-nào-dùng)
2. [Cài Đặt & Cấu Hình](#2-cài-đặt--cấu-hình)
3. [useSWR — Hook Cơ Bản](#3-useswr--hook-cơ-bản)
4. [Các Tùy Chọn Quan Trọng](#4-các-tùy-chọn-quan-trọng)
5. [Mutation & Revalidation](#5-mutation--revalidation)
6. [useSWRInfinite — Cuộn Vô Hạn](#6-useswrinfinite--cuộn-vô-hạn)
7. [SWRConfig — Cấu Hình Toàn Cục](#7-swrconfig--cấu-hình-toàn-cục)
8. [SWR vs TanStack Query — So Sánh Chi Tiết](#8-swr-vs-tanstack-query--so-sánh-chi-tiết)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. SWR Là Gì & Khi Nào Dùng

### Nguyên Tắc Stale-While-Revalidate

SWR hoạt động theo 3 bước:

```
1. STALE   → Trả về data từ cache ngay lập tức (UI hiển thị tức thì, dù data có thể cũ)
2. WHILE   → Trong khi đó, gửi request fetch dữ liệu mới ở background
3. REVALIDATE → Khi nhận được response mới, cập nhật UI với data mới nhất
```

### Ví Dụ Trực Quan

```
Lần 1: Cache rỗng → Hiển thị skeleton → Fetch → Hiển thị data → Lưu vào cache
Lần 2: Cache có data → Hiển thị data cũ NGAY → Fetch mới → Cập nhật nếu data thay đổi
```

Người dùng không thấy skeleton ở lần 2 — trải nghiệm mượt mà hơn nhiều.

### Khi Nào Chọn SWR

| Tình Huống | Khuyến Nghị |
| ---------- | ----------- |
| Dự án Next.js (đặc biệt với Vercel) | ✅ SWR — tích hợp tốt, nhẹ |
| Cần bundle size nhỏ (~4KB vs ~13KB) | ✅ SWR |
| API đơn giản, không cần DevTools phức tạp | ✅ SWR |
| Cần Optimistic Updates phức tạp | ✅ TanStack Query |
| Cần Infinite Query nâng cao | ✅ TanStack Query |
| Cần cache invalidation linh hoạt | ✅ TanStack Query |
| Team đã quen với Redux ecosystem | ✅ RTK Query |

---

## 2. Cài Đặt & Cấu Hình

```bash
npm install swr
```

SWR không cần provider (nhà cung cấp) bao bên ngoài như TanStack Query — có thể dùng ngay.

```javascript
// Cách dùng cơ bản nhất — không cần setup
import useSWR from 'swr';

// Fetcher function — nhận key và trả về Promise
const fetcher = (url) => fetch(url).then(res => res.json());

function UserProfile() {
  const { data, error, isLoading } = useSWR('/api/user', fetcher);
  // ...
}
```

---

## 3. useSWR — Hook Cơ Bản

### Cú Pháp

```javascript
const { data, error, isLoading, isValidating, mutate } = useSWR(key, fetcher, options);
```

| Tham Số | Mô Tả |
| ------- | ----- |
| `key` | Định danh duy nhất của request (thường là URL). `null` để tạm dừng fetch |
| `fetcher` | Hàm nhận key và trả về Promise với data |
| `options` | Object cấu hình tùy chọn |

| Giá Trị Trả Về | Mô Tả |
| -------------- | ----- |
| `data` | Dữ liệu trả về từ fetcher (undefined nếu chưa load) |
| `error` | Lỗi từ fetcher (undefined nếu không có lỗi) |
| `isLoading` | true khi không có data và đang fetch lần đầu |
| `isValidating` | true khi đang fetch (kể cả revalidation ở background) |
| `mutate` | Hàm cập nhật cache thủ công |

### Ví Dụ Thực Tế

```javascript
import useSWR from 'swr';

// Fetcher có thể xử lý lỗi HTTP
const fetcher = async (url) => {
  const res = await fetch(url);
  if (!res.ok) {
    const error = new Error('Fetch thất bại');
    error.status = res.status;
    throw error;
  }
  return res.json();
};

// Hook đơn giản — lấy user profile
function UserCard({ userId }) {
  const { data: user, error, isLoading } = useSWR(
    `/api/users/${userId}`,
    fetcher
  );

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage message={error.message} />;

  return (
    <div className="user-card">
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.bio}</p>
    </div>
  );
}
```

### Key Là Null — Tạm Dừng Fetch (Conditional Fetching)

```javascript
function ConditionalData({ isLoggedIn, userId }) {
  // Chỉ fetch khi đã đăng nhập
  const { data } = useSWR(
    isLoggedIn ? `/api/users/${userId}` : null,
    fetcher
  );
  // Khi key là null, SWR không gửi request
  return <div>{data?.name}</div>;
}
```

### Key Là Mảng — Fetch Với Tham Số

```javascript
// Key là mảng — tương tự queryKey trong TanStack Query
function SearchResults({ query, page }) {
  const { data } = useSWR(
    // Key thay đổi khi query hoặc page thay đổi → tự động refetch
    query ? ['/api/search', query, page] : null,
    // Fetcher nhận key (spread nếu là mảng)
    ([url, q, p]) => fetch(`${url}?q=${q}&page=${p}`).then(r => r.json())
  );

  return <SearchResultList results={data?.items} />;
}
```

### Key Là Hàm — Fetch Động

```javascript
// Key là hàm — nếu throw thì SWR không fetch
function DynamicData({ token }) {
  const { data } = useSWR(
    () => {
      if (!token) throw new Error('No token'); // abort fetch
      return `/api/data?token=${token}`;
    },
    fetcher
  );

  return <DataDisplay data={data} />;
}
```

---

## 4. Các Tùy Chọn Quan Trọng

```javascript
const { data } = useSWR('/api/data', fetcher, {
  // --- Revalidation (làm mới) ---
  revalidateOnFocus: true,       // Refetch khi focus vào tab (mặc định: true)
  revalidateOnReconnect: true,   // Refetch khi internet reconnect (mặc định: true)
  revalidateIfStale: true,       // Refetch khi có stale data (mặc định: true)
  revalidateOnMount: true,       // Refetch khi component mount (mặc định: true)

  // --- Polling (kiểm tra định kỳ) ---
  refreshInterval: 0,            // Polling interval (ms). 0 = tắt polling

  // --- Retry (thử lại) ---
  errorRetryCount: 3,            // Số lần retry khi lỗi
  errorRetryInterval: 5000,      // Khoảng cách giữa các lần retry (ms)
  shouldRetryOnError: true,      // Có retry khi lỗi không (mặc định: true)

  // --- Data ---
  fallbackData: [],              // Data mặc định khi chưa có data
  keepPreviousData: true,        // Giữ data cũ trong khi đang fetch mới (tránh flicker)
  dedupingInterval: 2000,        // Khoảng thời gian loại bỏ request trùng (ms)
  
  // --- Callbacks ---
  onSuccess: (data) => {},       // Callback khi fetch thành công
  onError: (error) => {},        // Callback khi fetch thất bại
  onLoadingSlow: (key) => {},    // Callback khi request quá chậm
  
  // --- Loading chậm ---
  loadingTimeout: 3000,          // Ngưỡng báo loading chậm (ms)
});
```

### keepPreviousData — Tránh Flicker Khi Phân Trang

```javascript
function PaginatedList({ page }) {
  const { data, isLoading, isValidating } = useSWR(
    `/api/posts?page=${page}`,
    fetcher,
    { keepPreviousData: true } // Giữ data trang cũ khi đang load trang mới
  );

  return (
    <div>
      {/* isValidating thay vì isLoading để tránh ẩn UI khi load trang mới */}
      {isValidating && <RefreshIndicator />}
      <PostList posts={data?.posts} />
    </div>
  );
}
```

### fallbackData — Dữ Liệu Mặc Định

```javascript
function NotificationBadge() {
  const { data: count } = useSWR('/api/notifications/count', fetcher, {
    fallbackData: 0,          // Hiển thị 0 trong khi chờ fetch
    refreshInterval: 60000,   // Cập nhật mỗi phút
  });

  return <span className="badge">{count}</span>;
}
```

---

## 5. Mutation & Revalidation

SWR dùng hàm `mutate` để cập nhật cache — khác với TanStack Query có `useMutation` riêng.

### mutate Cục Bộ (Bound Mutate)

```javascript
function UserProfile() {
  const { data: user, mutate } = useSWR('/api/user', fetcher);

  const updateBio = async (newBio) => {
    // mutate(data, options)
    // Cập nhật cache ngay (optimistic), đồng thời gọi revalidation
    await mutate(
      // Hàm nhận data cũ, trả về data mới (optimistic)
      { ...user, bio: newBio },
      {
        // Gọi lại fetcher sau khi mutate để đồng bộ với server
        revalidate: true,
        // Callback async thực hiện mutation thực sự
        populateCache: true,
        // Nếu optimistc update thất bại — rollback tự động
        rollbackOnError: true,
        // Hoặc: truyền async function trả về data mới từ server
      }
    );

    // Cách khác — dùng optimistic update đầy đủ
    try {
      await mutate(
        // async function — SWR đợi promise này resolve rồi cập nhật cache
        async () => {
          const res = await fetch('/api/user', {
            method: 'PATCH',
            body: JSON.stringify({ bio: newBio }),
          });
          return res.json();
        },
        {
          optimisticData: { ...user, bio: newBio }, // Hiển thị ngay
          rollbackOnError: true,                     // Rollback nếu lỗi
          revalidate: false,                         // Không refetch sau khi thành công
        }
      );
    } catch (error) {
      alert('Cập nhật thất bại');
    }
  };

  return (
    <div>
      <p>{user?.bio}</p>
      <button onClick={() => updateBio('Bio mới của tôi')}>Cập nhật Bio</button>
    </div>
  );
}
```

### mutate Toàn Cục (Global Mutate)

```javascript
import { mutate } from 'swr';

// Có thể gọi từ bất kỳ đâu, kể cả ngoài component
async function handleLogout() {
  await logoutAPI();
  // Invalidate tất cả SWR cache
  mutate(() => true, undefined, { revalidate: false });
}

// Invalidate một key cụ thể
async function afterCreatePost(newPost) {
  await createPostAPI(newPost);
  // Trigger revalidation cho '/api/posts'
  mutate('/api/posts');
}
```

### useSWRMutation — Cho Thao Tác Thay Đổi Dữ Liệu

`useSWRMutation` là hook chuyên biệt cho mutations (thay đổi dữ liệu), tương tự `useMutation` trong TanStack Query.

```javascript
import useSWRMutation from 'swr/mutation';

// Fetcher cho mutation — nhận key và { arg }
const sendPost = async (url, { arg }) => {
  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(arg),
  });
  if (!res.ok) throw new Error('Đăng bài thất bại');
  return res.json();
};

function CreatePostForm() {
  const [title, setTitle] = useState('');

  const { trigger, isMutating, error } = useSWRMutation(
    '/api/posts',
    sendPost,
    {
      onSuccess: () => {
        // Revalidate danh sách posts sau khi tạo thành công
        mutate('/api/posts');
        setTitle('');
      },
    }
  );

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      // trigger gọi sendPost với arg là { title }
      const newPost = await trigger({ title, userId: 1 });
      console.log('Đã tạo post:', newPost.id);
    } catch (err) {
      console.error('Lỗi:', err.message);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={title}
        onChange={e => setTitle(e.target.value)}
        placeholder="Tiêu đề bài viết"
      />
      <button type="submit" disabled={isMutating}>
        {isMutating ? 'Đang đăng...' : 'Đăng bài'}
      </button>
      {error && <p className="error">{error.message}</p>}
    </form>
  );
}
```

---

## 6. useSWRInfinite — Cuộn Vô Hạn

```javascript
import useSWRInfinite from 'swr/infinite';
import { useEffect, useRef } from 'react';

// Hàm tạo key cho từng trang
const getKey = (pageIndex, previousPageData) => {
  // Đã đến trang cuối → dừng
  if (previousPageData && !previousPageData.hasMore) return null;
  // Trang đầu tiên
  if (pageIndex === 0) return '/api/posts?page=1';
  // Các trang tiếp theo
  return `/api/posts?page=${pageIndex + 1}`;
};

function InfinitePostList() {
  const bottomRef = useRef(null);

  const { data, size, setSize, isLoading, isValidating } = useSWRInfinite(
    getKey,
    fetcher,
    { revalidateFirstPage: false } // Không revalidate trang đầu khi load trang mới
  );

  // Gộp tất cả trang thành một mảng phẳng
  const posts = data ? data.flatMap(page => page.items) : [];
  const hasMore = data ? data[data.length - 1]?.hasMore : true;
  const isLoadingMore = isValidating && data && data.length === size;

  // Intersection Observer (Quan Sát Giao Điểm) để phát hiện khi cuộn đến cuối
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasMore && !isValidating) {
          setSize(prev => prev + 1); // Tải trang tiếp theo
        }
      },
      { threshold: 1.0 }
    );

    if (bottomRef.current) observer.observe(bottomRef.current);
    return () => observer.disconnect();
  }, [hasMore, isValidating, setSize]);

  if (isLoading) return <PostSkeleton count={5} />;

  return (
    <div>
      {posts.map(post => <PostCard key={post.id} post={post} />)}
      
      {/* Sentinel element — phần tử theo dõi ở cuối danh sách */}
      <div ref={bottomRef} style={{ height: 1 }} />
      
      {isLoadingMore && <LoadingSpinner />}
      {!hasMore && <p>Đã xem hết tất cả bài viết</p>}
    </div>
  );
}
```

---

## 7. SWRConfig — Cấu Hình Toàn Cục

```javascript
import { SWRConfig } from 'swr';

// Fetcher toàn cục — không cần truyền vào từng useSWR
const globalFetcher = async (url) => {
  const token = localStorage.getItem('auth_token');
  const res = await fetch(url, {
    headers: token ? { Authorization: `Bearer ${token}` } : {},
  });
  if (!res.ok) {
    const error = new Error('API Error');
    error.status = res.status;
    throw error;
  }
  return res.json();
};

function App() {
  return (
    <SWRConfig
      value={{
        fetcher: globalFetcher,          // Fetcher mặc định
        revalidateOnFocus: false,        // Tắt refetch khi focus (tùy dự án)
        errorRetryCount: 3,              // Retry 3 lần
        onError: (error) => {
          if (error.status === 401) {
            // Token hết hạn → redirect to login
            window.location.href = '/login';
          }
        },
        // Cấu hình provider — dùng Map làm cache storage (tùy chỉnh)
        // provider: () => new Map(),
      }}
    >
      <Router>
        <Routes />
      </Router>
    </SWRConfig>
  );
}

// Với cấu hình global fetcher, dùng useSWR đơn giản hơn
function UserCard({ userId }) {
  // Không cần truyền fetcher
  const { data } = useSWR(`/api/users/${userId}`);
  return <div>{data?.name}</div>;
}
```

### Nested SWRConfig — Ghi Đè Cấu Hình Theo Khu Vực

```javascript
function AdminPanel() {
  return (
    // Ghi đè revalidateOnFocus chỉ trong Admin Panel
    <SWRConfig value={{ revalidateOnFocus: true, refreshInterval: 5000 }}>
      <AdminDashboard />
    </SWRConfig>
  );
}
```

---

## 8. SWR vs TanStack Query — So Sánh Chi Tiết

| Tiêu Chí | SWR | TanStack Query |
| -------- | --- | -------------- |
| **Bundle size** (kích thước gói) | ~4 KB (gzip) | ~13 KB (gzip) |
| **API đơn giản** | ✅ Rất đơn giản | Trung bình |
| **Provider bắt buộc** | ❌ Không cần | ✅ Cần QueryClientProvider |
| **DevTools** | ❌ Không có sẵn | ✅ Rất tốt |
| **Mutation hook riêng** | `useSWRMutation` (v2+) | `useMutation` (đầy đủ hơn) |
| **Optimistic updates** | Có nhưng phức tạp hơn | ✅ Pattern rõ ràng, đầy đủ |
| **Infinite scroll** | `useSWRInfinite` | `useInfiniteQuery` |
| **Pagination** | Cần tự handle | `keepPreviousData` + `placeholderData` |
| **Dependent queries** | `null` key | `enabled` option |
| **Cache invalidation** | `mutate(key)` | `invalidateQueries` linh hoạt hơn |
| **Background sync** | ✅ Tốt | ✅ Tốt |
| **TypeScript** | Tốt | Xuất sắc |
| **Cộng đồng** | Lớn | Rất lớn |
| **Phù hợp với** | Dự án nhỏ-vừa, Next.js | Mọi quy mô, nhiều tính năng |

### Ví Dụ So Sánh API

```javascript
// --- SWR ---
import useSWR from 'swr';

function Posts() {
  const { data, error, isLoading } = useSWR('/api/posts', fetcher);
  if (isLoading) return <Skeleton />;
  if (error) return <Error />;
  return <PostList posts={data} />;
}

// --- TanStack Query ---
import { useQuery } from '@tanstack/react-query';

function Posts() {
  const { data, isLoading, isError } = useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then(r => r.json()),
  });
  if (isLoading) return <Skeleton />;
  if (isError) return <Error />;
  return <PostList posts={data} />;
}
```

API khá tương đồng nhưng TanStack Query có nhiều option và control hơn.

---

## 9. Câu Hỏi Phỏng Vấn

**Q: SWR là viết tắt của gì và nguyên tắc hoạt động?**

> SWR — Stale-While-Revalidate — Phục Vụ Dữ Liệu Cũ Trong Khi Làm Mới. Khi component yêu cầu data, SWR: (1) Trả về data từ cache ngay lập tức (stale — có thể cũ), (2) Đồng thời gửi request fetch data mới ở background, (3) Cập nhật UI khi nhận được response mới. Điều này giúp UI luôn responsive và data luôn được làm mới.

**Q: Khi nào chọn SWR thay vì TanStack Query?**

> Chọn SWR khi: (1) Dự án Next.js đơn giản — SWR từ Vercel nên tích hợp tốt. (2) Bundle size là ưu tiên — SWR nhỏ hơn ~3x. (3) API đơn giản, không cần nhiều tính năng nâng cao. (4) Team chưa quen với TanStack Query. Chọn TanStack Query khi: cần DevTools, cần optimistic updates phức tạp, cần cache invalidation linh hoạt, hoặc ứng dụng lớn nhiều queries.

**Q: Làm thế nào để implement optimistic update trong SWR?**

```javascript
function LikeButton({ post }) {
  const { data, mutate } = useSWR(`/api/posts/${post.id}`, fetcher);

  const handleLike = () => {
    mutate(
      // Gọi API và trả về data mới từ server
      async () => {
        const res = await fetch(`/api/posts/${post.id}/like`, { method: 'POST' });
        return res.json();
      },
      {
        // Hiển thị data optimistic ngay lập tức (trước khi API trả về)
        optimisticData: { ...data, liked: !data.liked, likes: data.likes + (data.liked ? -1 : 1) },
        // Rollback về data cũ nếu API thất bại
        rollbackOnError: true,
        // Không refetch sau khi API trả về (dùng response từ API)
        revalidate: false,
        // Cập nhật cache với response từ API
        populateCache: true,
      }
    );
  };

  return (
    <button onClick={handleLike}>
      {data?.liked ? '❤️' : '🤍'} {data?.likes}
    </button>
  );
}
```

**Q: Tại sao SWR hiển thị data cũ trước thay vì skeleton?**

> Đây là ưu điểm cốt lõi của SWR. Khi component unmount rồi mount lại (ví dụ: điều hướng trang), cache vẫn còn data từ lần fetch trước. SWR trả về data đó ngay lập tức (`isLoading = false`, `data = cachedData`), đồng thời `isValidating = true` để báo đang fetch mới ở background. UI không bị trắng/skeleton — trải nghiệm người dùng tốt hơn nhiều. Dùng `keepPreviousData: true` trong TanStack Query để đạt hiệu ứng tương tự.

---

## 🔗 Điều Hướng

- **Tiếp theo:** [4-react-server-components.md](./4-react-server-components.md) — RSC — React Server Components
- **Trước đó:** [2-tanstack-query-advanced.md](./2-tanstack-query-advanced.md) — TanStack Query nâng cao
- **Quay lại:** [README.md](./README.md) — Tổng quan Data Fetching
