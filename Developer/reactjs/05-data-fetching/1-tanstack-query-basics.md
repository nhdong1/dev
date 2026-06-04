# TanStack Query — Cơ Bản (useQuery & useMutation)

> **TanStack Query** (trước đây là React Query) là thư viện quản lý **server state** (trạng thái máy chủ) phổ biến nhất trong hệ sinh thái React. Nó giải quyết việc fetch, cache, đồng bộ và cập nhật dữ liệu từ server một cách khai báo và hiệu quả.

---

## 📌 Mục Lục

1. [TanStack Query Là Gì?](#1-tanstack-query-là-gì)
2. [Cài Đặt & Cấu Hình](#2-cài-đặt--cấu-hình)
3. [useQuery — Lấy Dữ Liệu](#3-usequery--lấy-dữ-liệu)
4. [Query Keys — Khóa Truy Vấn](#4-query-keys--khóa-truy-vấn)
5. [Loading, Error, Success States](#5-loading-error-success-states)
6. [useMutation — Thay Đổi Dữ Liệu](#6-usemutation--thay-đổi-dữ-liệu)
7. [useInfiniteQuery — Cuộn Vô Hạn](#7-useinfinitequery--cuộn-vô-hạn)
8. [Dependent Queries — Truy Vấn Phụ Thuộc](#8-dependent-queries--truy-vấn-phụ-thuộc)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. TanStack Query Là Gì?

TanStack Query giải quyết bài toán **server state management** mà các giải pháp như Redux hay Zustand không được tối ưu để xử lý.

### Những Gì TanStack Query Tự Động Xử Lý

- ✅ **Caching** (Lưu Cache) — lưu kết quả fetch, tái sử dụng khi cùng query key
- ✅ **Background refetching** (Lấy Lại Nền) — tự động làm mới dữ liệu cũ
- ✅ **Request deduplication** (Loại Bỏ Request Trùng) — nhiều component cùng query → chỉ 1 request
- ✅ **Stale-while-revalidate** — hiển thị data cũ ngay, fetch mới ở nền
- ✅ **Pagination & infinite queries** (Phân Trang & Cuộn Vô Hạn)
- ✅ **Optimistic updates** (Cập Nhật Lạc Quan)
- ✅ **Retry logic** (Thử Lại) — tự động retry khi request thất bại
- ✅ **Window focus refetch** — tự động refetch khi người dùng quay lại tab

### So Sánh Với Cách Fetch Thủ Công

```javascript
// ❌ Fetch thủ công — 30 dòng cho một tác vụ đơn giản
function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    
    fetch('/api/users')
      .then(res => {
        if (!res.ok) throw new Error('Fetch thất bại');
        return res.json();
      })
      .then(data => {
        if (!cancelled) {
          setUsers(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      });
    
    return () => { cancelled = true; }; // cleanup để tránh race condition
  }, []);
  // Không có caching, không có background refetch, ...
}

// ✅ Với TanStack Query — 5 dòng, đầy đủ tính năng
function UserList() {
  const { data: users, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(res => res.json()),
  });
  // Tự động có: caching, background refetch, retry, deduplication...
}
```

---

## 2. Cài Đặt & Cấu Hình

### Cài Đặt

```bash
npm install @tanstack/react-query
# Khuyến nghị: cài thêm DevTools để debug
npm install @tanstack/react-query-devtools
```

### Cấu Hình QueryClient và QueryClientProvider

`QueryClient` — Đối Tượng Client Truy Vấn là trung tâm lưu trữ toàn bộ cache và cấu hình của TanStack Query.

```javascript
// src/main.jsx hoặc src/App.jsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// Tạo QueryClient với cấu hình mặc định
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // staleTime — Thời Gian Dữ Liệu Còn Tươi (ms)
      // Trong staleTime, dữ liệu được dùng từ cache, không fetch lại
      staleTime: 1000 * 60 * 5, // 5 phút

      // gcTime — Garbage Collection Time — Thời Gian Thu Dọn Cache (ms)
      // Cache bị xóa sau gcTime khi không có observer nào
      gcTime: 1000 * 60 * 10, // 10 phút

      // retry — số lần thử lại khi request thất bại
      retry: 3,

      // refetchOnWindowFocus — tự động refetch khi focus vào tab
      refetchOnWindowFocus: true,
    },
    mutations: {
      // retry mutations không được khuyến nghị (có thể gây double submit)
      retry: 0,
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router>
        <AppRoutes />
      </Router>
      {/* Chỉ hiển thị DevTools trong môi trường development */}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

---

## 3. useQuery — Lấy Dữ Liệu

`useQuery` là hook chính để fetch dữ liệu từ server.

### Cú Pháp Cơ Bản

```javascript
const result = useQuery({
  queryKey: [...],   // khóa duy nhất xác định query
  queryFn: () => {}, // hàm fetch dữ liệu, phải return Promise
  // ...options tùy chọn
});

// Destructure các giá trị cần dùng
const {
  data,         // dữ liệu trả về từ queryFn
  isLoading,    // true khi fetch lần đầu, chưa có data
  isFetching,   // true khi đang fetch (kể cả background fetch)
  isError,      // true khi có lỗi
  error,        // object lỗi
  isSuccess,    // true khi fetch thành công
  refetch,      // hàm gọi để fetch lại thủ công
  status,       // 'pending' | 'error' | 'success'
  fetchStatus,  // 'fetching' | 'paused' | 'idle'
} = result;
```

### Ví Dụ Thực Tế — Lấy Danh Sách Bài Viết

```javascript
// src/api/posts.js — tách hàm fetch riêng (best practice — thực hành tốt nhất)
export const fetchPosts = async () => {
  const response = await fetch('https://jsonplaceholder.typicode.com/posts');
  if (!response.ok) {
    // Throw error để TanStack Query bắt được
    throw new Error(`HTTP error! status: ${response.status}`);
  }
  return response.json();
};

// src/components/PostList.jsx
import { useQuery } from '@tanstack/react-query';
import { fetchPosts } from '../api/posts';

function PostList() {
  const {
    data: posts,
    isLoading,
    isError,
    error,
    isFetching,
  } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    staleTime: 1000 * 60 * 2, // 2 phút — override default
  });

  // Trạng thái loading lần đầu (chưa có data trong cache)
  if (isLoading) {
    return <div className="skeleton-list">Đang tải bài viết...</div>;
  }

  if (isError) {
    return <div className="error">Lỗi: {error.message}</div>;
  }

  return (
    <div>
      {/* isFetching true khi background refetch — hiển thị indicator nhỏ */}
      {isFetching && <span className="sync-indicator">Đang đồng bộ...</span>}
      
      <ul>
        {posts.map(post => (
          <li key={post.id}>
            <h3>{post.title}</h3>
            <p>{post.body}</p>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Lấy Dữ Liệu Theo ID — Parameterized Query

```javascript
// src/api/posts.js
export const fetchPostById = async (id) => {
  const response = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);
  if (!response.ok) throw new Error('Post không tồn tại');
  return response.json();
};

// src/components/PostDetail.jsx
function PostDetail({ postId }) {
  const { data: post, isLoading, isError } = useQuery({
    queryKey: ['posts', postId], // queryKey bao gồm tham số
    queryFn: () => fetchPostById(postId),
    // enabled — chỉ fetch khi postId tồn tại
    enabled: !!postId,
  });

  if (isLoading) return <div>Đang tải bài viết #{postId}...</div>;
  if (isError) return <div>Không tìm thấy bài viết</div>;

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}
```

### Với Axios — Thư Viện HTTP Phổ Biến

```javascript
import axios from 'axios';

// Tạo axios instance (thực thể axios) với cấu hình chung
const apiClient = axios.create({
  baseURL: 'https://api.example.com',
  headers: { 'Content-Type': 'application/json' },
});

// Hook tùy chỉnh bọc useQuery — best practice
function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const { data } = await apiClient.get('/users');
      return data;
    },
  });
}

// Sử dụng trong component
function UserList() {
  const { data: users, isLoading } = useUsers();
  // ...
}
```

---

## 4. Query Keys — Khóa Truy Vấn

**Query Key** (Khóa Truy Vấn) là yếu tố quan trọng nhất trong TanStack Query — nó xác định danh tính của mỗi query, dùng để caching và invalidation.

### Quy Tắc Query Key

```javascript
// ✅ String đơn giản — cho global queries
queryKey: ['users']
queryKey: ['settings']

// ✅ Array với tham số — cho parameterized queries
queryKey: ['users', userId]          // fetch user theo ID
queryKey: ['posts', { page: 1 }]     // fetch posts trang 1
queryKey: ['posts', postId, 'comments'] // fetch comments của post

// ✅ Object — cho queries phức tạp hơn
queryKey: ['products', { category: 'electronics', sort: 'price', page: 1 }]
```

### Tổ Chức Query Keys — Best Practice

```javascript
// queryKeys.js — tập trung tất cả query keys
export const queryKeys = {
  // Factories cho phép tạo key nhất quán
  users: {
    all: ['users'],
    lists: () => [...queryKeys.users.all, 'list'],
    list: (filters) => [...queryKeys.users.lists(), filters],
    details: () => [...queryKeys.users.all, 'detail'],
    detail: (id) => [...queryKeys.users.details(), id],
  },
  posts: {
    all: ['posts'],
    byUser: (userId) => [...queryKeys.posts.all, 'user', userId],
    detail: (id) => [...queryKeys.posts.all, id],
  },
};

// Sử dụng
useQuery({ queryKey: queryKeys.users.detail(userId), queryFn: ... });
useQuery({ queryKey: queryKeys.posts.byUser(userId), queryFn: ... });

// Invalidate toàn bộ users cache
queryClient.invalidateQueries({ queryKey: queryKeys.users.all });
// Invalidate chỉ user detail cache
queryClient.invalidateQueries({ queryKey: queryKeys.users.details() });
```

---

## 5. Loading, Error, Success States

### Phân Biệt isLoading và isFetching

```javascript
function DataComponent() {
  const { data, isLoading, isFetching, isStale } = useQuery({
    queryKey: ['data'],
    queryFn: fetchData,
    staleTime: 30000,
  });

  return (
    <div>
      {/* isLoading: true chỉ khi KHÔNG có data trong cache và đang fetch */}
      {isLoading && <FullPageSpinner />}

      {/* isFetching: true bất cứ khi nào đang fetch (kể cả có data cũ) */}
      {isFetching && !isLoading && <SmallRefreshIndicator />}

      {/* isStale: data đã quá staleTime */}
      {isStale && <span>Dữ liệu có thể đã cũ</span>}

      {data && <DataDisplay data={data} />}
    </div>
  );
}
```

### Xử Lý Lỗi Hiệu Quả

```javascript
import { useQuery } from '@tanstack/react-query';

function ProductList() {
  const { data, isLoading, isError, error, refetch } = useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts,
    retry: 2,           // thử lại 2 lần trước khi báo lỗi
    retryDelay: 1000,   // chờ 1 giây giữa các lần retry
  });

  if (isLoading) return <LoadingSkeleton count={5} />;

  if (isError) {
    return (
      <div className="error-container">
        <p>Không thể tải sản phẩm: {error.message}</p>
        <button onClick={() => refetch()}>Thử lại</button>
      </div>
    );
  }

  return <ProductGrid products={data} />;
}
```

### Kết Hợp Với React Suspense

```javascript
// Với React Suspense — Error Boundary xử lý lỗi
function ProductListSuspense() {
  const { data: products } = useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts,
    // Bật suspend để dùng với React Suspense
    // (chỉ dùng khi có ErrorBoundary bao ngoài)
  });

  // Không cần kiểm tra isLoading — Suspense xử lý
  return <ProductGrid products={products} />;
}

// Component cha
function App() {
  return (
    <ErrorBoundary fallback={<ErrorPage />}>
      <Suspense fallback={<LoadingSkeleton />}>
        <ProductListSuspense />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## 6. useMutation — Thay Đổi Dữ Liệu

`useMutation` dùng để thực hiện các thao tác **CUD** (Create — Tạo, Update — Cập Nhật, Delete — Xóa).

### Cú Pháp Cơ Bản

```javascript
const mutation = useMutation({
  mutationFn: (variables) => {}, // hàm thực hiện mutation, nhận variables
  onSuccess: (data, variables, context) => {}, // callback khi thành công
  onError: (error, variables, context) => {},  // callback khi thất bại
  onSettled: (data, error, variables, context) => {}, // luôn chạy sau mutation
  onMutate: async (variables) => {},           // chạy TRƯỚC mutation (dùng cho optimistic update)
});

// Kích hoạt mutation
mutation.mutate(variables);       // không await được trong component
mutation.mutateAsync(variables);  // await được, cần try/catch
```

### Ví Dụ — Tạo, Cập Nhật, Xóa Bài Viết

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';

// API functions
const createPost = async (newPost) => {
  const res = await fetch('/api/posts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(newPost),
  });
  if (!res.ok) throw new Error('Tạo bài viết thất bại');
  return res.json();
};

const updatePost = async ({ id, ...data }) => {
  const res = await fetch(`/api/posts/${id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!res.ok) throw new Error('Cập nhật thất bại');
  return res.json();
};

const deletePost = async (id) => {
  const res = await fetch(`/api/posts/${id}`, { method: 'DELETE' });
  if (!res.ok) throw new Error('Xóa thất bại');
};

// Component sử dụng các mutations
function PostManager() {
  const queryClient = useQueryClient();

  // --- Mutation tạo bài viết ---
  const createMutation = useMutation({
    mutationFn: createPost,
    onSuccess: (newPost) => {
      // Sau khi tạo thành công, invalidate cache để refetch danh sách
      queryClient.invalidateQueries({ queryKey: ['posts'] });
      console.log('Đã tạo bài viết:', newPost.id);
    },
    onError: (error) => {
      alert(`Lỗi: ${error.message}`);
    },
  });

  // --- Mutation cập nhật bài viết ---
  const updateMutation = useMutation({
    mutationFn: updatePost,
    onSuccess: (updatedPost) => {
      // Cập nhật trực tiếp cache thay vì refetch (hiệu quả hơn)
      queryClient.setQueryData(['posts', updatedPost.id], updatedPost);
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  // --- Mutation xóa bài viết ---
  const deleteMutation = useMutation({
    mutationFn: deletePost,
    onSuccess: (_, deletedId) => {
      // Xóa khỏi cache ngay lập tức
      queryClient.removeQueries({ queryKey: ['posts', deletedId] });
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  const handleCreate = () => {
    createMutation.mutate({
      title: 'Bài viết mới',
      body: 'Nội dung...',
      userId: 1,
    });
  };

  return (
    <div>
      <button
        onClick={handleCreate}
        disabled={createMutation.isPending}
      >
        {createMutation.isPending ? 'Đang tạo...' : 'Tạo bài viết'}
      </button>

      <button
        onClick={() => updateMutation.mutate({ id: 1, title: 'Tiêu đề mới' })}
        disabled={updateMutation.isPending}
      >
        Cập nhật
      </button>

      <button
        onClick={() => deleteMutation.mutate(1)}
        disabled={deleteMutation.isPending}
      >
        {deleteMutation.isPending ? 'Đang xóa...' : 'Xóa'}
      </button>

      {createMutation.isError && (
        <p className="error">{createMutation.error.message}</p>
      )}
    </div>
  );
}
```

### mutateAsync — Dùng Với async/await

```javascript
function CreatePostForm() {
  const queryClient = useQueryClient();
  const [title, setTitle] = useState('');

  const mutation = useMutation({
    mutationFn: createPost,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const newPost = await mutation.mutateAsync({ title, userId: 1 });
      // mutateAsync cho phép await và xử lý kết quả trực tiếp
      alert(`Tạo thành công! ID: ${newPost.id}`);
      setTitle('');
    } catch (error) {
      // Lỗi được throw ra ngoài
      console.error('Tạo thất bại:', error.message);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={title}
        onChange={e => setTitle(e.target.value)}
        placeholder="Tiêu đề bài viết"
      />
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Đang gửi...' : 'Đăng bài'}
      </button>
    </form>
  );
}
```

---

## 7. useInfiniteQuery — Cuộn Vô Hạn

`useInfiniteQuery` (Truy Vấn Vô Hạn) dùng để implement **infinite scroll** (cuộn vô hạn) hoặc **load more** (tải thêm).

```javascript
import { useInfiniteQuery } from '@tanstack/react-query';
import { useInView } from 'react-intersection-observer';

// API trả về { data: [], nextPage: number | null, total: number }
const fetchPostsPaginated = async ({ pageParam = 1 }) => {
  const res = await fetch(`/api/posts?page=${pageParam}&limit=10`);
  return res.json();
};

function InfinitePostList() {
  const { ref, inView } = useInView(); // hook theo dõi khi element visible

  const {
    data,
    fetchNextPage,    // gọi để tải trang tiếp theo
    hasNextPage,      // true nếu còn trang tiếp theo
    isFetchingNextPage, // true khi đang tải trang tiếp
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: fetchPostsPaginated,
    // getNextPageParam — hàm xác định tham số của trang tiếp theo
    getNextPageParam: (lastPage) => lastPage.nextPage ?? undefined,
    initialPageParam: 1,
  });

  // Khi sentinel element (phần tử theo dõi) vào tầm nhìn → tải tiếp
  useEffect(() => {
    if (inView && hasNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, fetchNextPage]);

  if (isLoading) return <LoadingSkeleton />;

  return (
    <div>
      {/* data.pages là mảng các trang, mỗi trang có data riêng */}
      {data.pages.map((page, pageIndex) => (
        <Fragment key={pageIndex}>
          {page.data.map(post => (
            <PostCard key={post.id} post={post} />
          ))}
        </Fragment>
      ))}

      {/* Sentinel element — khi element này visible → trigger fetchNextPage */}
      <div ref={ref}>
        {isFetchingNextPage
          ? <LoadingSpinner />
          : hasNextPage
          ? <span>Cuộn xuống để tải thêm</span>
          : <span>Đã hiển thị tất cả bài viết</span>
        }
      </div>
    </div>
  );
}
```

---

## 8. Dependent Queries — Truy Vấn Phụ Thuộc

Đôi khi cần query B phụ thuộc vào kết quả của query A.

```javascript
function UserPosts({ userId }) {
  // Query 1: lấy thông tin user
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // Query 2: chỉ chạy khi đã có user.email
  const { data: posts, isLoading } = useQuery({
    queryKey: ['posts', user?.email],
    queryFn: () => fetchPostsByEmail(user.email),
    // enabled — query chỉ chạy khi điều kiện này true
    enabled: !!user?.email,
  });

  if (isLoading) return <div>Đang tải bài viết...</div>;

  return (
    <div>
      <h2>Bài viết của {user?.name}</h2>
      {posts?.map(post => <PostCard key={post.id} post={post} />)}
    </div>
  );
}
```

### Parallel Queries — Truy Vấn Song Song

```javascript
function Dashboard() {
  // Chạy song song — không phụ thuộc nhau
  const usersQuery = useQuery({ queryKey: ['users'], queryFn: fetchUsers });
  const productsQuery = useQuery({ queryKey: ['products'], queryFn: fetchProducts });
  const ordersQuery = useQuery({ queryKey: ['orders'], queryFn: fetchOrders });

  const isLoading = usersQuery.isLoading || productsQuery.isLoading || ordersQuery.isLoading;

  if (isLoading) return <DashboardSkeleton />;

  return (
    <Dashboard
      users={usersQuery.data}
      products={productsQuery.data}
      orders={ordersQuery.data}
    />
  );
}

// Hoặc dùng useQueries cho dynamic parallel queries
import { useQueries } from '@tanstack/react-query';

function MultipleUserProfiles({ userIds }) {
  const queries = useQueries({
    queries: userIds.map(id => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
    })),
  });

  const profiles = queries
    .filter(q => q.isSuccess)
    .map(q => q.data);

  return <ProfileList profiles={profiles} />;
}
```

---

## 9. Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: TanStack Query khác gì so với fetch thủ công trong useEffect?**

> TanStack Query cung cấp tự động: caching (lưu cache), background refetching (lấy lại nền), request deduplication (loại bỏ request trùng), retry logic (thử lại), stale data handling, và DevTools. Fetch thủ công đòi hỏi tự xây dựng tất cả những thứ này.

**Q: staleTime và gcTime (cacheTime cũ) khác nhau như thế nào?**

> - **staleTime** — Thời gian data được coi là "tươi" (fresh). Trong khoảng này, không fetch lại dù có trigger. Mặc định: 0 (luôn stale).
> - **gcTime** — Thời gian cache tồn tại sau khi không còn observer nào. Sau thời gian này, cache bị xóa khỏi bộ nhớ. Mặc định: 5 phút.
>
> Ví dụ: staleTime=2min, gcTime=10min → Trong 2 phút đầu: dùng cache, không fetch. Sau 2 phút: stale, fetch mới ở background. Sau 10 phút không dùng: cache bị xóa.

**Q: Khi nào dùng isLoading vs isFetching?**

> - `isLoading` (hoặc `isPending` trong v5): Dùng để show loading skeleton lần đầu. True chỉ khi KHÔNG có data trong cache VÀ đang fetch.
> - `isFetching`: Dùng để show indicator nhỏ khi refetch. True bất cứ khi nào đang fetch, kể cả khi đã có data cũ trong cache.

**Q: Tại sao query key quan trọng?**

> Query key là định danh duy nhất của mỗi query trong cache. TanStack Query dùng key để: (1) tìm kiếm cache, (2) invalidate và refetch đúng queries, (3) tạo subscription — nếu hai component dùng cùng key, chúng chia sẻ cache và chỉ tạo một request. Key nên bao gồm mọi tham số ảnh hưởng đến kết quả fetch.

### Câu Hỏi Nâng Cao

**Q: Làm thế nào để prefetch data trước khi người dùng điều hướng?**

```javascript
const queryClient = useQueryClient();

// Prefetch khi hover vào link
function PostLink({ postId }) {
  const handleMouseEnter = () => {
    queryClient.prefetchQuery({
      queryKey: ['post', postId],
      queryFn: () => fetchPost(postId),
      staleTime: 60000, // không prefetch lại nếu đã có data tươi
    });
  };

  return (
    <Link to={`/posts/${postId}`} onMouseEnter={handleMouseEnter}>
      Xem bài viết
    </Link>
  );
}
```

**Q: Làm thế nào để chia sẻ QueryClient giữa Server và Client trong Next.js?**

> Dùng **HydrationBoundary** (Ranh Giới Hydration) — prefetch data trên server và dehydrate (giải nước) thành JSON, sau đó hydrate (ngậm nước) trên client để tái sử dụng cache mà không fetch lại.

```javascript
// Server Component (Next.js App Router)
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';

export default async function PostsPage() {
  const queryClient = new QueryClient();
  
  // Prefetch trên server
  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  return (
    // Truyền cache từ server xuống client
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostList /> {/* Client Component — dùng cache từ server */}
    </HydrationBoundary>
  );
}
```

---

## 🔗 Điều Hướng

- **Tiếp theo:** [2-tanstack-query-advanced.md](./2-tanstack-query-advanced.md) — Caching nâng cao & Optimistic Updates
- **Trước đó:** [05-data-fetching/README.md](./README.md) — Tổng quan Data Fetching
- **Liên quan:** [03-state-management/3-rtk-query.md](../03-state-management/3-rtk-query.md) — RTK Query (so sánh)
