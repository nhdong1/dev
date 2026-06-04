# TanStack Query Nâng Cao — Caching, Invalidation & Optimistic Updates

> Phần nâng cao của TanStack Query bao gồm các chiến lược **caching** (lưu cache), **cache invalidation** (vô hiệu hóa cache), **optimistic updates** (cập nhật lạc quan), **prefetching** (tải trước), và các kỹ thuật tối ưu hiệu năng trong ứng dụng thực tế.

---

## 📌 Mục Lục

1. [Caching Strategy — Chiến Lược Lưu Cache](#1-caching-strategy--chiến-lược-lưu-cache)
2. [Cache Invalidation — Vô Hiệu Hóa Cache](#2-cache-invalidation--vô-hiệu-hóa-cache)
3. [setQueryData — Cập Nhật Cache Thủ Công](#3-setquerydata--cập-nhật-cache-thủ-công)
4. [Optimistic Updates — Cập Nhật Lạc Quan](#4-optimistic-updates--cập-nhật-lạc-quan)
5. [Prefetching — Tải Trước Dữ Liệu](#5-prefetching--tải-trước-dữ-liệu)
6. [Polling & Refetch Strategies](#6-polling--refetch-strategies)
7. [Query Filters & Bulk Operations](#7-query-filters--bulk-operations)
8. [Hydration — SSR với Next.js](#8-hydration--ssr-với-nextjs)
9. [Custom Hooks — Đóng Gói Logic](#9-custom-hooks--đóng-gói-logic)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Caching Strategy — Chiến Lược Lưu Cache

### Cơ Chế Cache Của TanStack Query

TanStack Query lưu trữ cache theo **query key** (khóa truy vấn) trong bộ nhớ của `QueryClient`. Mỗi entry trong cache có trạng thái: **fresh** (tươi), **stale** (cũ), hoặc **inactive** (không hoạt động).

```
[Query thực hiện]
       ↓
[Data trả về → Lưu vào cache]
       ↓
[fresh: staleTime còn hạn → Serve từ cache, KHÔNG fetch]
       ↓
[stale: quá staleTime → Serve từ cache + fetch mới ở background]
       ↓
[Không có observer nào → inactive → đếm ngược gcTime]
       ↓
[Hết gcTime → Xóa khỏi cache]
```

### Cấu Hình Cache Theo Loại Dữ Liệu

```javascript
// Chiến lược cache khác nhau cho từng loại dữ liệu
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60, // 1 phút (mặc định)
      gcTime: 1000 * 60 * 5, // 5 phút (mặc định)
    },
  },
});

// Dữ liệu thay đổi thường xuyên — staleTime ngắn
const { data: notifications } = useQuery({
  queryKey: ['notifications'],
  queryFn: fetchNotifications,
  staleTime: 0, // luôn stale → luôn refetch khi có trigger
  refetchInterval: 30000, // polling mỗi 30 giây
});

// Dữ liệu ít thay đổi — staleTime dài
const { data: countries } = useQuery({
  queryKey: ['countries'],
  queryFn: fetchCountries,
  staleTime: Infinity, // không bao giờ stale → fetch một lần duy nhất
  gcTime: Infinity,    // không bao giờ bị garbage collect
});

// Dữ liệu người dùng — staleTime trung bình
const { data: userProfile } = useQuery({
  queryKey: ['profile', userId],
  queryFn: () => fetchUserProfile(userId),
  staleTime: 1000 * 60 * 5, // 5 phút
});
```

### Hiểu Rõ staleTime vs gcTime

```
Timeline:
t=0s    → Query fetch thành công, data = fresh
t=0→5m  → staleTime còn hạn → Data vẫn fresh → Component mới mount KHÔNG trigger fetch
t=5m    → staleTime hết → Data trở thành stale
t=5m+   → Mỗi khi có trigger (focus, mount, manual) → Serve data cũ + fetch mới ở background
t=?     → Component unmount → Query trở thành inactive
t=?+10m → gcTime hết → Cache entry bị xóa khỏi bộ nhớ
```

---

## 2. Cache Invalidation — Vô Hiệu Hóa Cache

**Cache Invalidation** (Vô Hiệu Hóa Cache) là kỹ thuật đánh dấu cache là stale và kích hoạt refetch ngay lập tức, thường được thực hiện sau khi mutation thành công.

### invalidateQueries — Vô Hiệu Hóa Theo Key

```javascript
import { useQueryClient } from '@tanstack/react-query';

function PostActions() {
  const queryClient = useQueryClient();

  // Invalidate MỘT query cụ thể
  const invalidatePost = (postId) => {
    queryClient.invalidateQueries({ queryKey: ['posts', postId] });
  };

  // Invalidate TOÀN BỘ queries có prefix 'posts'
  // ['posts'], ['posts', 1], ['posts', 'infinite'], ... đều bị invalidate
  const invalidateAllPosts = () => {
    queryClient.invalidateQueries({ queryKey: ['posts'] });
  };

  // Invalidate CHÍNH XÁC một key (không invalidate subtrees)
  const invalidateExact = () => {
    queryClient.invalidateQueries({
      queryKey: ['posts'],
      exact: true, // chỉ invalidate ['posts'], không invalidate ['posts', 1]
    });
  };

  return (
    <button onClick={invalidateAllPosts}>Làm mới danh sách bài viết</button>
  );
}
```

### Invalidation Sau Mutation — Pattern Phổ Biến

```javascript
const queryClient = useQueryClient();

// Pattern 1: Invalidate sau khi tạo/xóa
const createPostMutation = useMutation({
  mutationFn: createPost,
  onSuccess: () => {
    // Invalidate list → refetch để có bài viết mới
    queryClient.invalidateQueries({ queryKey: ['posts'] });
  },
});

// Pattern 2: Invalidate có liên quan chéo
const updateUserMutation = useMutation({
  mutationFn: updateUser,
  onSuccess: (updatedUser) => {
    // Invalidate cả user profile lẫn danh sách posts của user đó
    queryClient.invalidateQueries({ queryKey: ['users', updatedUser.id] });
    queryClient.invalidateQueries({ queryKey: ['posts', 'byUser', updatedUser.id] });
  },
});

// Pattern 3: Invalidate có điều kiện
const deleteMutation = useMutation({
  mutationFn: deletePost,
  onSuccess: (_, postId) => {
    // Xóa cache của post đã xóa
    queryClient.removeQueries({ queryKey: ['posts', postId] });
    // Invalidate danh sách
    queryClient.invalidateQueries({ queryKey: ['posts'] });
  },
});
```

---

## 3. setQueryData — Cập Nhật Cache Thủ Công

`setQueryData` cho phép cập nhật cache trực tiếp mà không cần network request, giúp UI cập nhật ngay lập tức.

```javascript
const queryClient = useQueryClient();

const updatePostMutation = useMutation({
  mutationFn: updatePost,
  onSuccess: (updatedPost) => {
    // Thay vì invalidate (gây refetch), cập nhật trực tiếp vào cache
    // Hiệu quả hơn khi server trả về object đã cập nhật
    queryClient.setQueryData(
      ['posts', updatedPost.id],
      updatedPost // data mới — thay thế hoàn toàn
    );

    // Cập nhật danh sách: tìm và thay thế item trong array
    queryClient.setQueryData(['posts'], (oldPosts) => {
      if (!oldPosts) return oldPosts;
      return oldPosts.map(post =>
        post.id === updatedPost.id ? updatedPost : post
      );
    });
  },
});

// Cập nhật từng phần — chỉ thay đổi một trường
const toggleLikeMutation = useMutation({
  mutationFn: toggleLike,
  onSuccess: (_, postId) => {
    queryClient.setQueryData(['posts', postId], (oldPost) => ({
      ...oldPost,
      liked: !oldPost.liked,
      likesCount: oldPost.liked ? oldPost.likesCount - 1 : oldPost.likesCount + 1,
    }));
  },
});
```

---

## 4. Optimistic Updates — Cập Nhật Lạc Quan

**Optimistic Updates** (Cập Nhật Lạc Quan) là kỹ thuật cập nhật UI **trước khi** server xác nhận, tạo cảm giác ứng dụng phản hồi tức thì. Nếu request thất bại, rollback (hoàn nguyên) về trạng thái cũ.

### Luồng Hoạt Động

```
User click "Like"
    ↓
onMutate: Cập nhật cache ngay (UI thay đổi tức thì)
    ↓
[Request gửi đến server ở background]
    ↓
    ├── Thành công → onSuccess: Invalidate để sync với server data
    └── Thất bại  → onError: Rollback về data cũ (dùng context từ onMutate)
```

### Triển Khai Optimistic Update Đầy Đủ

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function PostCard({ post }) {
  const queryClient = useQueryClient();

  const likeMutation = useMutation({
    mutationFn: (postId) => toggleLikePost(postId),

    // onMutate chạy TRƯỚC khi request gửi đi
    onMutate: async (postId) => {
      // Bước 1: Hủy các refetch đang chạy để tránh ghi đè optimistic update
      await queryClient.cancelQueries({ queryKey: ['posts', postId] });

      // Bước 2: Lưu snapshot (ảnh chụp) data hiện tại để rollback nếu lỗi
      const previousPost = queryClient.getQueryData(['posts', postId]);

      // Bước 3: Cập nhật cache ngay lập tức (optimistic)
      queryClient.setQueryData(['posts', postId], (old) => ({
        ...old,
        liked: !old.liked,
        likesCount: old.liked ? old.likesCount - 1 : old.likesCount + 1,
      }));

      // Bước 4: Return context chứa snapshot để dùng trong onError
      return { previousPost };
    },

    // onError: Rollback về data cũ nếu request thất bại
    onError: (error, postId, context) => {
      // Khôi phục data từ snapshot lưu trong onMutate
      queryClient.setQueryData(['posts', postId], context.previousPost);
      alert('Không thể like bài viết. Đã hoàn nguyên.');
    },

    // onSettled: Luôn chạy sau mutation (dù thành công hay thất bại)
    onSettled: (data, error, postId) => {
      // Đồng bộ với server data để đảm bảo tính nhất quán
      queryClient.invalidateQueries({ queryKey: ['posts', postId] });
    },
  });

  return (
    <div className="post-card">
      <h3>{post.title}</h3>
      <button
        onClick={() => likeMutation.mutate(post.id)}
        disabled={likeMutation.isPending}
        className={post.liked ? 'liked' : ''}
      >
        ❤️ {post.likesCount}
      </button>
    </div>
  );
}
```

### Optimistic Update Cho Danh Sách — Add/Remove Item

```javascript
// Optimistic add item vào danh sách
const addTodoMutation = useMutation({
  mutationFn: createTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });
    const previousTodos = queryClient.getQueryData(['todos']);

    queryClient.setQueryData(['todos'], (old) => [
      ...old,
      {
        // Tạo temporary ID — ID tạm thời cho optimistic item
        id: `temp-${Date.now()}`,
        ...newTodo,
        status: 'pending', // cho biết đây là optimistic item
      },
    ]);

    return { previousTodos };
  },

  onSuccess: (serverTodo) => {
    // Replace temporary item với real data từ server
    queryClient.setQueryData(['todos'], (old) =>
      old.map(todo =>
        todo.id.startsWith('temp-') ? serverTodo : todo
      )
    );
  },

  onError: (error, _, context) => {
    queryClient.setQueryData(['todos'], context.previousTodos);
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

---

## 5. Prefetching — Tải Trước Dữ Liệu

**Prefetching** (Tải Trước) là kỹ thuật fetch dữ liệu trước khi người dùng thực sự cần, giúp navigation cảm giác tức thì.

### prefetchQuery — Tải Trước Trên Client

```javascript
function ProductCard({ product }) {
  const queryClient = useQueryClient();

  // Prefetch khi hover vào product card
  const handleMouseEnter = () => {
    queryClient.prefetchQuery({
      queryKey: ['product', product.id],
      queryFn: () => fetchProductDetail(product.id),
      // Không prefetch nếu data đã fresh trong 60 giây
      staleTime: 60000,
    });
  };

  return (
    <div onMouseEnter={handleMouseEnter}>
      <Link to={`/products/${product.id}`}>
        <img src={product.thumbnail} alt={product.name} />
        <h3>{product.name}</h3>
      </Link>
    </div>
  );
}
```

### Prefetch Trang Kế Tiếp Trong Pagination

```javascript
function PaginatedPosts({ currentPage }) {
  const queryClient = useQueryClient();

  const { data } = useQuery({
    queryKey: ['posts', currentPage],
    queryFn: () => fetchPostsPage(currentPage),
  });

  // Prefetch trang tiếp theo ngay khi đang xem trang hiện tại
  useEffect(() => {
    if (data?.hasNextPage) {
      queryClient.prefetchQuery({
        queryKey: ['posts', currentPage + 1],
        queryFn: () => fetchPostsPage(currentPage + 1),
      });
    }
  }, [currentPage, data?.hasNextPage, queryClient]);

  return (
    <div>
      {data?.posts.map(post => <PostCard key={post.id} post={post} />)}
      <Pagination
        currentPage={currentPage}
        totalPages={data?.totalPages}
      />
    </div>
  );
}
```

---

## 6. Polling & Refetch Strategies

### Polling — Kiểm Tra Định Kỳ

```javascript
// Polling mỗi 10 giây để theo dõi trạng thái job
function JobStatus({ jobId }) {
  const { data: job } = useQuery({
    queryKey: ['job', jobId],
    queryFn: () => fetchJobStatus(jobId),
    // Tự động refetch mỗi 10 giây
    refetchInterval: 10000,
    // Chỉ poll khi tab đang active (người dùng đang xem)
    refetchIntervalInBackground: false,
  });

  // Dừng polling khi job hoàn thành
  const { data: jobDynamic } = useQuery({
    queryKey: ['job', jobId],
    queryFn: () => fetchJobStatus(jobId),
    // refetchInterval có thể là hàm — nhận data và trả về interval (ms) hoặc false
    refetchInterval: (query) => {
      const data = query.state.data;
      if (!data || data.status === 'completed' || data.status === 'failed') {
        return false; // dừng polling
      }
      return 5000; // tiếp tục poll mỗi 5 giây
    },
  });

  return (
    <div>
      <p>Trạng thái: {job?.status}</p>
      <progress value={job?.progress} max={100} />
    </div>
  );
}
```

### Refetch Strategies — Chiến Lược Refetch

```javascript
const { data, refetch } = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  // Các trigger cho background refetch:
  refetchOnMount: true,            // refetch khi component mount (mặc định: true)
  refetchOnWindowFocus: true,      // refetch khi focus vào tab (mặc định: true)
  refetchOnReconnect: true,        // refetch khi internet reconnect (mặc định: true)
  refetchInterval: false,          // không polling (mặc định)
});

// Refetch thủ công khi user click
<button onClick={() => refetch()}>Làm mới</button>

// Refetch với force (bỏ qua staleTime)
<button onClick={() => refetch({ cancelRefetch: false })}>
  Buộc làm mới
</button>
```

---

## 7. Query Filters & Bulk Operations

### Bulk Invalidation Có Điều Kiện

```javascript
const queryClient = useQueryClient();

// Invalidate tất cả queries đang active (đang có observer)
queryClient.invalidateQueries({
  predicate: (query) => query.isActive(),
});

// Invalidate tất cả queries liên quan đến một user
queryClient.invalidateQueries({
  predicate: (query) =>
    query.queryKey[0] === 'user' || query.queryKey[0] === 'posts',
});

// Reset tất cả queries về initial state (xóa data và error)
queryClient.resetQueries({ queryKey: ['posts'] });

// Cancel tất cả outgoing requests
queryClient.cancelQueries({ queryKey: ['posts'] });

// Xóa hoàn toàn khỏi cache
queryClient.removeQueries({ queryKey: ['posts', 'old'] });
```

### Global Cache Management

```javascript
// Xóa toàn bộ cache khi user logout
function useLogout() {
  const queryClient = useQueryClient();

  return async () => {
    await logoutAPI();
    // Xóa toàn bộ cache — quan trọng để tránh data leak
    queryClient.clear();
    navigate('/login');
  };
}
```

---

## 8. Hydration — SSR với Next.js

**Hydration** (Ngậm Nước) trong TanStack Query là quá trình chuyển cache data từ server xuống client, giúp tránh fetch lại dữ liệu đã có trên server.

### Setup Với Next.js App Router

```javascript
// src/lib/query-client.ts
import { QueryClient } from '@tanstack/react-query';
import { cache } from 'react';

// cache() của React đảm bảo mỗi request có QueryClient riêng
// (tránh share state giữa các request khác nhau)
export const getQueryClient = cache(() =>
  new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 phút
      },
    },
  })
);
```

```javascript
// app/posts/page.tsx — Server Component
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';
import { getQueryClient } from '@/lib/query-client';
import { PostList } from './PostList';

export default async function PostsPage() {
  const queryClient = getQueryClient();

  // Prefetch trên server — chạy song song với các prefetch khác
  await Promise.all([
    queryClient.prefetchQuery({
      queryKey: ['posts'],
      queryFn: fetchPosts,
    }),
    queryClient.prefetchQuery({
      queryKey: ['featured-post'],
      queryFn: fetchFeaturedPost,
    }),
  ]);

  // dehydrate (giải nước) — serialize cache thành JSON để gửi về client
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      {/* Client Components bên trong có thể dùng useQuery với data từ server cache */}
      <PostList />
    </HydrationBoundary>
  );
}
```

```javascript
// app/posts/PostList.tsx — Client Component
'use client';

import { useQuery } from '@tanstack/react-query';

export function PostList() {
  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    // Data đã có từ server — không fetch lại nếu còn trong staleTime
  });

  // Render ngay lập tức từ hydrated cache — không có loading flash
  return (
    <ul>
      {posts?.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  );
}
```

---

## 9. Custom Hooks — Đóng Gói Logic

Đây là **best practice** (thực hành tốt nhất) — đóng gói mọi query/mutation vào custom hooks để tái sử dụng và tập trung logic.

```javascript
// src/hooks/usePosts.js
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { postsApi } from '../api/posts';

// Query key factory — tập trung quản lý keys
const postKeys = {
  all: ['posts'],
  lists: () => [...postKeys.all, 'list'],
  list: (filters) => [...postKeys.lists(), filters],
  details: () => [...postKeys.all, 'detail'],
  detail: (id) => [...postKeys.details(), id],
};

// Hook lấy danh sách posts với filter
export function usePosts(filters = {}) {
  return useQuery({
    queryKey: postKeys.list(filters),
    queryFn: () => postsApi.getAll(filters),
    staleTime: 2 * 60 * 1000,
  });
}

// Hook lấy một post theo ID
export function usePost(id) {
  return useQuery({
    queryKey: postKeys.detail(id),
    queryFn: () => postsApi.getById(id),
    enabled: !!id,
  });
}

// Hook tạo post mới
export function useCreatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: postsApi.create,
    onSuccess: (newPost) => {
      // Invalidate list queries
      queryClient.invalidateQueries({ queryKey: postKeys.lists() });

      // Thêm vào cache detail ngay lập tức
      queryClient.setQueryData(postKeys.detail(newPost.id), newPost);
    },
  });
}

// Hook cập nhật post với optimistic update
export function useUpdatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: postsApi.update,
    onMutate: async ({ id, ...updates }) => {
      await queryClient.cancelQueries({ queryKey: postKeys.detail(id) });
      const previous = queryClient.getQueryData(postKeys.detail(id));

      queryClient.setQueryData(postKeys.detail(id), (old) => ({
        ...old,
        ...updates,
      }));

      return { previous, id };
    },
    onError: (err, { id }, context) => {
      queryClient.setQueryData(postKeys.detail(id), context.previous);
    },
    onSettled: (data, err, { id }) => {
      queryClient.invalidateQueries({ queryKey: postKeys.detail(id) });
    },
  });
}

// Hook xóa post
export function useDeletePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: postsApi.delete,
    onSuccess: (_, id) => {
      queryClient.removeQueries({ queryKey: postKeys.detail(id) });
      queryClient.invalidateQueries({ queryKey: postKeys.lists() });
    },
  });
}

// Sử dụng trong component — cực kỳ gọn
function PostManager() {
  const { data: posts, isLoading } = usePosts({ category: 'tech' });
  const createMutation = useCreatePost();
  const updateMutation = useUpdatePost();
  const deleteMutation = useDeletePost();

  if (isLoading) return <Skeleton />;

  return (
    <div>
      {posts?.map(post => (
        <PostItem
          key={post.id}
          post={post}
          onUpdate={(data) => updateMutation.mutate({ id: post.id, ...data })}
          onDelete={() => deleteMutation.mutate(post.id)}
        />
      ))}
    </div>
  );
}
```

---

## 10. Câu Hỏi Phỏng Vấn

### Câu Hỏi Trung Bình

**Q: Giải thích Optimistic Update (Cập Nhật Lạc Quan) và khi nào nên dùng?**

> Optimistic Update là kỹ thuật cập nhật UI ngay lập tức trước khi server xác nhận, dựa trên giả định request sẽ thành công. Nếu thất bại, rollback về trạng thái cũ.
>
> **Nên dùng khi:** (1) Hành động có tỷ lệ thành công cao (like, toggle, reorder). (2) Latency (độ trễ) cao — người dùng thấy ngay kết quả thay vì chờ. (3) UX là ưu tiên hàng đầu.
>
> **Không nên dùng khi:** (1) Hành động có tỷ lệ thất bại cao (payment). (2) Rollback gây confusing cho người dùng. (3) Dữ liệu phức tạp, khó rollback chính xác.

**Q: Tại sao cần cancelQueries trong onMutate của Optimistic Update?**

> Nếu đang có một background refetch đang chạy khi onMutate thực thi, refetch đó có thể hoàn thành sau khi ta đã set optimistic data — ghi đè data optimistic bằng data cũ từ server. `cancelQueries` hủy request đó để tránh race condition (điều kiện đua) này.

**Q: Khác biệt giữa invalidateQueries và setQueryData?**

> - `invalidateQueries`: Đánh dấu cache là stale → kích hoạt refetch → network request → data mới từ server. Dùng khi muốn đồng bộ với server.
> - `setQueryData`: Cập nhật cache trực tiếp trong bộ nhớ → không có network request → cập nhật tức thì. Dùng khi server đã trả về data mới trong mutation response, hoặc cho optimistic updates.

### Câu Hỏi Nâng Cao

**Q: Làm thế nào để handle authentication token trong TanStack Query?**

```javascript
// Tạo custom fetcher với auth header
const authenticatedFetch = async (url, options = {}) => {
  const token = getAuthToken(); // lấy từ localStorage hoặc cookie
  const response = await fetch(url, {
    ...options,
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  if (response.status === 401) {
    // Token hết hạn → logout hoặc refresh token
    await refreshToken();
    // Hoặc redirect to login
    throw new Error('Unauthorized');
  }

  return response.json();
};

// Sử dụng trong queries
useQuery({
  queryKey: ['protected-data'],
  queryFn: () => authenticatedFetch('/api/protected'),
  // Chỉ fetch khi đã authenticated
  enabled: !!getAuthToken(),
});
```

**Q: Làm thế nào để implement global error handling trong TanStack Query?**

```javascript
// Cấu hình global error handler trong QueryClient
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Log lỗi lên monitoring service (ví dụ: Sentry)
      Sentry.captureException(error, { extra: { queryKey: query.queryKey } });

      // Hiển thị toast notification (thông báo nổi)
      if (error.status === 500) {
        toast.error('Lỗi server. Vui lòng thử lại sau.');
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      toast.error(`Thao tác thất bại: ${error.message}`);
    },
  }),
});
```

---

## 🔗 Điều Hướng

- **Tiếp theo:** [3-swr.md](./3-swr.md) — SWR — Stale-While-Revalidate
- **Trước đó:** [1-tanstack-query-basics.md](./1-tanstack-query-basics.md) — TanStack Query cơ bản
- **Quay lại:** [README.md](./README.md) — Tổng quan Data Fetching
