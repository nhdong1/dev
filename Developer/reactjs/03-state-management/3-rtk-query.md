# RTK Query — Data Fetching & Caching Tích Hợp Redux

> **RTK Query** (Query Của Redux Toolkit) là giải pháp data fetching (lấy dữ liệu) và caching (lưu cache) được tích hợp sẵn trong Redux Toolkit. Nó tự động hóa hầu hết boilerplate liên quan đến việc fetch, cache, và đồng bộ server state — tương đương TanStack Query nhưng sống trong hệ sinh thái Redux.

---

## 📌 Mục Lục

1. [RTK Query Là Gì?](#1-rtk-query-là-gì)
2. [createApi — Tạo API Service](#2-createapi)
3. [useQuery — Lấy Dữ Liệu](#3-usequery)
4. [useMutation — Thay Đổi Dữ Liệu](#4-usemutation)
5. [Cache Invalidation — Vô Hiệu Hóa Cache](#5-cache-invalidation)
6. [Optimistic Updates — Cập Nhật Lạc Quan](#6-optimistic-updates)
7. [Ví Dụ Thực Tế — Blog API](#7-ví-dụ-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. RTK Query Là Gì?

RTK Query giải quyết **server state management** (quản lý trạng thái máy chủ) — dữ liệu sống trên server và cần được đồng bộ với client.

### Vấn Đề Khi Dùng createAsyncThunk Thủ Công

```javascript
// ❌ Với createAsyncThunk — phải viết tất cả thủ công
const fetchUsers = createAsyncThunk("users/fetchAll", async () => {
  const res = await fetch("/api/users");
  return res.json();
});

const usersSlice = createSlice({
  name: "users",
  initialState: { data: [], status: "idle", error: null },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, state => { state.status = "loading"; })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.status = "succeeded";
        state.data = action.payload;
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.status = "failed";
        state.error = action.error.message;
      });
  },
});
// Phải lặp lại pattern này cho mỗi endpoint (điểm cuối API)!
```

```javascript
// ✅ Với RTK Query — khai báo endpoint, RTK Query lo phần còn lại
const usersApi = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
  endpoints: (builder) => ({
    getUsers: builder.query({ query: () => "/users" }),
  }),
});
// Tự động có: loading state, error state, caching, refetching, ...
```

### RTK Query Tự Động Cung Cấp

- ✅ **Loading/error state** tự động
- ✅ **Caching** — cache data và tái sử dụng trong `keepUnusedDataFor` giây
- ✅ **Automatic refetching** (tự động lấy lại) — khi window focus, khi reconnect
- ✅ **Request deduplication** (loại bỏ request trùng lặp)
- ✅ **Polling** (kiểm tra định kỳ)
- ✅ **Optimistic updates** (cập nhật lạc quan)
- ✅ **TypeScript code generation** từ OpenAPI specs

---

## 2. createApi — Tạo API Service

```javascript
// src/store/api/postsApi.js
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export const postsApi = createApi({
  // reducerPath — tên slice trong Redux store
  reducerPath: "postsApi",

  // baseQuery — cấu hình request gốc
  baseQuery: fetchBaseQuery({
    baseUrl: "https://jsonplaceholder.typicode.com",
    // prepareHeaders — thêm headers vào mọi request (ví dụ: Authorization token)
    prepareHeaders: (headers, { getState }) => {
      const token = getState().auth.token;
      if (token) {
        headers.set("Authorization", `Bearer ${token}`);
      }
      return headers;
    },
  }),

  // tagTypes — nhãn dùng cho cache invalidation
  tagTypes: ["Post", "User"],

  // keepUnusedDataFor — giữ cache bao nhiêu giây sau khi không còn subscriber
  keepUnusedDataFor: 60,

  endpoints: (builder) => ({
    // Query endpoint (GET) — lấy dữ liệu
    getPosts: builder.query({
      query: () => "/posts",
      providesTags: ["Post"],  // Gắn nhãn cache này là "Post"
    }),

    getPostById: builder.query({
      query: (id) => `/posts/${id}`,
      providesTags: (result, error, id) => [{ type: "Post", id }],
    }),

    getPostsByUser: builder.query({
      query: (userId) => `/posts?userId=${userId}`,
      providesTags: (result, error, userId) =>
        result
          ? [...result.map(({ id }) => ({ type: "Post", id })), "Post"]
          : ["Post"],
    }),

    // Mutation endpoint (POST/PUT/DELETE) — thay đổi dữ liệu
    createPost: builder.mutation({
      query: (newPost) => ({
        url: "/posts",
        method: "POST",
        body: newPost,
      }),
      // Sau khi mutation thành công, invalidate (vô hiệu) cache "Post"
      invalidatesTags: ["Post"],
    }),

    updatePost: builder.mutation({
      query: ({ id, ...patch }) => ({
        url: `/posts/${id}`,
        method: "PATCH",
        body: patch,
      }),
      // Chỉ invalidate cache của post cụ thể
      invalidatesTags: (result, error, { id }) => [{ type: "Post", id }],
    }),

    deletePost: builder.mutation({
      query: (id) => ({
        url: `/posts/${id}`,
        method: "DELETE",
      }),
      invalidatesTags: (result, error, id) => [{ type: "Post", id }],
    }),
  }),
});

// RTK Query tự động tạo hooks từ các endpoints
export const {
  useGetPostsQuery,
  useGetPostByIdQuery,
  useGetPostsByUserQuery,
  useCreatePostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} = postsApi;
```

### Đăng Ký API Với Store

```javascript
// src/store/index.js
import { configureStore } from "@reduxjs/toolkit";
import { postsApi } from "./api/postsApi";

export const store = configureStore({
  reducer: {
    [postsApi.reducerPath]: postsApi.reducer,
  },
  // Thêm middleware của RTK Query — bắt buộc để caching hoạt động
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(postsApi.middleware),
});
```

---

## 3. useQuery — Lấy Dữ Liệu

### Cơ Bản

```jsx
import { useGetPostsQuery } from "../store/api/postsApi";

function PostsList() {
  const {
    data: posts,          // Dữ liệu trả về
    isLoading,            // true khi đang tải lần đầu
    isFetching,           // true khi đang tải (bao gồm cả refetch)
    isSuccess,            // true khi fetch thành công
    isError,              // true khi có lỗi
    error,                // Đối tượng lỗi
    refetch,              // Hàm để gọi lại request thủ công
  } = useGetPostsQuery();

  if (isLoading) return <div>Đang tải lần đầu...</div>;
  if (isError) return <div>Lỗi: {error.message}</div>;

  return (
    <div>
      {isFetching && <div>Đang cập nhật...</div>}
      <button onClick={refetch}>Làm mới</button>
      {posts?.map(post => (
        <article key={post.id}>
          <h3>{post.title}</h3>
        </article>
      ))}
    </div>
  );
}
```

### Truyền Tham Số

```jsx
function PostDetail({ postId }) {
  const { data: post, isLoading } = useGetPostByIdQuery(postId);

  // Skip query — bỏ qua request khi điều kiện không thỏa
  const { data: userPosts } = useGetPostsByUserQuery(post?.userId, {
    skip: !post?.userId,  // Không fetch cho đến khi có post.userId
  });

  if (isLoading) return <div>Đang tải...</div>;
  return <div>{post?.title}</div>;
}
```

### Các Options Hữu Ích

```jsx
const { data } = useGetPostsQuery(undefined, {
  // pollingInterval — tự động refetch mỗi N milliseconds
  pollingInterval: 30000,  // Refetch mỗi 30 giây

  // refetchOnFocus — refetch khi user quay lại tab
  refetchOnFocus: true,

  // refetchOnReconnect — refetch khi mạng kết nối lại
  refetchOnReconnect: true,

  // selectFromResult — chỉ lấy một phần data, giảm re-render
  selectFromResult: ({ data }) => ({
    posts: data?.filter(p => p.published) ?? [],
  }),
});
```

---

## 4. useMutation — Thay Đổi Dữ Liệu

```jsx
import {
  useCreatePostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} from "../store/api/postsApi";

function PostForm() {
  const [createPost, { isLoading, isSuccess, isError, error }] = useCreatePostMutation();

  const handleSubmit = async (formData) => {
    try {
      // unwrap() — throw error nếu mutation thất bại
      const newPost = await createPost(formData).unwrap();
      console.log("Tạo thành công:", newPost);
    } catch (err) {
      console.error("Lỗi:", err);
    }
  };

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      handleSubmit({ title: "Bài mới", body: "Nội dung..." });
    }}>
      <button type="submit" disabled={isLoading}>
        {isLoading ? "Đang tạo..." : "Tạo bài viết"}
      </button>
      {isSuccess && <p>Tạo thành công!</p>}
      {isError && <p>Lỗi: {error.message}</p>}
    </form>
  );
}

function PostActions({ post }) {
  const [updatePost] = useUpdatePostMutation();
  const [deletePost] = useDeletePostMutation();

  return (
    <div>
      <button onClick={() => updatePost({ id: post.id, title: "Tiêu đề mới" })}>
        Sửa
      </button>
      <button onClick={() => deletePost(post.id)}>
        Xóa
      </button>
    </div>
  );
}
```

---

## 5. Cache Invalidation — Vô Hiệu Hóa Cache

Cache invalidation (vô hiệu hóa cache) là cơ chế tự động refetch data sau khi mutation thay đổi server.

### Cơ Chế Tags

```javascript
// Cơ chế hoạt động:
// 1. Query "getPosts" cung cấp (providesTags) nhãn "Post"
// 2. Mutation "createPost" vô hiệu (invalidatesTags) nhãn "Post"
// 3. RTK Query tự động refetch "getPosts" sau khi "createPost" thành công

endpoints: (builder) => ({
  // List endpoint — cung cấp nhãn cho toàn bộ list
  getUsers: builder.query({
    query: () => "/users",
    providesTags: (result) =>
      result
        ? [
            // Tag cho từng item
            ...result.map(({ id }) => ({ type: "User", id })),
            // Tag cho toàn bộ list
            { type: "User", id: "LIST" },
          ]
        : [{ type: "User", id: "LIST" }],
  }),

  // Detail endpoint — cung cấp nhãn cho item cụ thể
  getUserById: builder.query({
    query: (id) => `/users/${id}`,
    providesTags: (result, error, id) => [{ type: "User", id }],
  }),

  // Create — invalidate toàn bộ list
  createUser: builder.mutation({
    query: (user) => ({ url: "/users", method: "POST", body: user }),
    invalidatesTags: [{ type: "User", id: "LIST" }],
  }),

  // Update — chỉ invalidate item cụ thể
  updateUser: builder.mutation({
    query: ({ id, ...patch }) => ({ url: `/users/${id}`, method: "PATCH", body: patch }),
    invalidatesTags: (result, error, { id }) => [{ type: "User", id }],
  }),

  // Delete — invalidate item và list
  deleteUser: builder.mutation({
    query: (id) => ({ url: `/users/${id}`, method: "DELETE" }),
    invalidatesTags: (result, error, id) => [
      { type: "User", id },
      { type: "User", id: "LIST" },
    ],
  }),
}),
```

---

## 6. Optimistic Updates — Cập Nhật Lạc Quan

**Optimistic Updates** (Cập Nhật Lạc Quan) — cập nhật UI ngay lập tức trước khi server xác nhận, rollback (hoàn tác) nếu server báo lỗi.

```javascript
updatePost: builder.mutation({
  query: ({ id, ...patch }) => ({
    url: `/posts/${id}`,
    method: "PATCH",
    body: patch,
  }),

  // onQueryStarted chạy khi mutation bắt đầu
  async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
    // Cập nhật cache ngay lập tức (optimistic)
    const patchResult = dispatch(
      postsApi.util.updateQueryData("getPosts", undefined, (draft) => {
        const post = draft.find(p => p.id === id);
        if (post) Object.assign(post, patch);
      })
    );

    try {
      // Chờ server xác nhận
      await queryFulfilled;
    } catch {
      // Server báo lỗi — rollback (hoàn tác) về state cũ
      patchResult.undo();
    }
  },

  invalidatesTags: (result, error, { id }) => [{ type: "Post", id }],
}),
```

---

## 7. Ví Dụ Thực Tế — Blog API Hoàn Chỉnh

```jsx
// src/features/blog/BlogPage.jsx
import { useState } from "react";
import {
  useGetPostsQuery,
  useCreatePostMutation,
  useDeletePostMutation,
} from "../../store/api/postsApi";

function BlogPage() {
  const [page, setPage] = useState(1);

  const {
    data: posts,
    isLoading,
    isFetching,
  } = useGetPostsQuery({ page, limit: 10 });

  const [createPost, { isLoading: isCreating }] = useCreatePostMutation();
  const [deletePost] = useDeletePostMutation();

  const handleCreate = async () => {
    await createPost({
      title: `Bài viết mới ${Date.now()}`,
      body: "Nội dung bài viết...",
      userId: 1,
    }).unwrap();
  };

  if (isLoading) return <div>Đang tải...</div>;

  return (
    <div>
      <header>
        <h1>Blog</h1>
        <button onClick={handleCreate} disabled={isCreating}>
          {isCreating ? "Đang tạo..." : "Viết bài mới"}
        </button>
      </header>

      {isFetching && <div className="loading-bar">Đang cập nhật...</div>}

      <div className="posts-grid">
        {posts?.map(post => (
          <article key={post.id}>
            <h2>{post.title}</h2>
            <p>{post.body.slice(0, 100)}...</p>
            <button onClick={() => deletePost(post.id)}>Xóa</button>
          </article>
        ))}
      </div>

      <div className="pagination">
        <button onClick={() => setPage(p => p - 1)} disabled={page === 1}>
          Trang trước
        </button>
        <span>Trang {page}</span>
        <button onClick={() => setPage(p => p + 1)}>
          Trang sau
        </button>
      </div>
    </div>
  );
}
```

---

## 8. Câu Hỏi Phỏng Vấn

### Câu 1: RTK Query khác createAsyncThunk như thế nào?

**Trả lời:**

| Khía Cạnh | createAsyncThunk | RTK Query |
| --------- | ---------------- | --------- |
| Mục đích | Async logic tổng quát | Data fetching chuyên biệt |
| Caching | Không tích hợp | Tự động caching |
| Boilerplate | Nhiều (3 cases mỗi thunk) | Rất ít (khai báo endpoint) |
| Invalidation | Thủ công | Tự động qua tags |
| Polling | Phải tự implement | Có sẵn `pollingInterval` |
| Optimistic updates | Phức tạp | Hỗ trợ qua `onQueryStarted` |

### Câu 2: Giải thích cơ chế cache invalidation trong RTK Query.

**Trả lời:**

RTK Query dùng hệ thống **tags** (nhãn):
1. **`providesTags`:** Query khai báo nó cung cấp dữ liệu được gắn nhãn gì
2. **`invalidatesTags`:** Mutation khai báo khi nó hoàn thành, các nhãn nào bị vô hiệu
3. Khi một nhãn bị vô hiệu, RTK Query tự động refetch tất cả queries đang active (hoạt động) có nhãn đó

### Câu 3: Khi nào dùng RTK Query và khi nào dùng TanStack Query?

**Trả lời:**
- **RTK Query:** Khi dự án đã dùng Redux — tích hợp tự nhiên, không cần thêm dependency
- **TanStack Query:** Khi không dùng Redux, hoặc muốn giải pháp nhẹ hơn và linh hoạt hơn cho server state
- Về tính năng: hai thư viện khá tương đương. TanStack Query có hệ sinh thái và cộng đồng lớn hơn, nhưng RTK Query có ưu thế nếu đã có Redux store

---

## 🔗 Điều Hướng

- **Tiếp theo:** [4-zustand.md](./4-zustand.md) — Zustand
- **Trước đó:** [2-redux-toolkit.md](./2-redux-toolkit.md) — Redux Toolkit
- **Quay lại:** [README.md](./README.md) — Tổng quan State Management
