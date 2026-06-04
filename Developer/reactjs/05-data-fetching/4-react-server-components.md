# React Server Components (RSC — Component Phía Server)

> **React Server Components** (RSC — Component Phía Server) là mô hình rendering mới trong React, cho phép viết các component chạy **hoàn toàn trên server**, truy cập trực tiếp database và file system, gửi về client chỉ HTML/dữ liệu mà không kèm JavaScript bundle. Đây là thay đổi lớn nhất trong kiến trúc React kể từ khi Hooks ra đời.

---

## 📌 Mục Lục

1. [RSC Là Gì & Tại Sao Quan Trọng](#1-rsc-là-gì--tại-sao-quan-trọng)
2. [Server Components vs Client Components](#2-server-components-vs-client-components)
3. [Viết Server Components](#3-viết-server-components)
4. [Data Fetching Trong Server Components](#4-data-fetching-trong-server-components)
5. [Kết Hợp Server & Client Components](#5-kết-hợp-server--client-components)
6. [Suspense & Streaming — Phát Trực Tuyến HTML](#6-suspense--streaming--phát-trực-tuyến-html)
7. [Caching Trong Next.js App Router](#7-caching-trong-nextjs-app-router)
8. [Giới Hạn & Lỗi Thường Gặp](#8-giới-hạn--lỗi-thường-gặp)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. RSC Là Gì & Tại Sao Quan Trọng

### Vấn Đề RSC Giải Quyết

Trước RSC, React hoàn toàn chạy trên client. Để lấy dữ liệu, ứng dụng phải:
1. Gửi HTML rỗng về client
2. Client tải JavaScript bundle (có thể rất lớn)
3. React khởi động và hydrate
4. Fetch dữ liệu từ API
5. Render UI với dữ liệu

```
Vấn đề: Waterfall (thác nước) — nhiều bước tuần tự, người dùng phải chờ lâu
Vấn đề: JavaScript bundle lớn — gửi cả thư viện lớn xuống client dù chỉ dùng ở server
Vấn đề: Không thể truy cập database/file system trực tiếp từ component
```

### RSC Giải Quyết Như Thế Nào

```
Server nhận request
    ↓
Server Component chạy — truy cập DB trực tiếp
    ↓
Server render HTML + RSC Payload (định dạng đặc biệt)
    ↓
Gửi về client — HTML có thể hiển thị ngay
    ↓
Client hydrate chỉ những Client Components cần thiết
```

### Lợi Ích Của RSC

- ✅ **Zero bundle size** — Thư viện chỉ dùng trong Server Component không được gửi xuống client
- ✅ **Truy cập trực tiếp** — Database, file system, environment variables bảo mật
- ✅ **Không waterfall** — Fetch song song trên server, gần với database
- ✅ **SEO tốt** — HTML đầy đủ từ server, search engine đọc được ngay
- ✅ **Automatic code splitting** (tách code tự động) — Client chỉ nhận JS của Client Components

---

## 2. Server Components vs Client Components

### Bảng So Sánh Đầy Đủ

| Tính Năng | Server Component | Client Component |
| --------- | ---------------- | ---------------- |
| **Nơi chạy** | Chỉ trên server | Trên client (và server khi SSR) |
| **Chỉ thị** | Mặc định (không cần khai báo) | `'use client'` ở đầu file |
| **Hooks** | ❌ Không hỗ trợ | ✅ Đầy đủ |
| **Event handlers** | ❌ onClick, onChange... | ✅ Đầy đủ |
| **State & Effects** | ❌ useState, useEffect | ✅ Đầy đủ |
| **Browser APIs** | ❌ localStorage, window... | ✅ Đầy đủ |
| **Async/await** | ✅ Có thể dùng trực tiếp | ❌ Chỉ qua hooks |
| **Database/Filesystem** | ✅ Trực tiếp | ❌ Phải qua API |
| **Secrets trong code** | ✅ An toàn | ❌ Lộ ra client |
| **Bundle JS** | ❌ Không đóng góp | ✅ Đóng góp vào bundle |
| **Re-render** | ❌ Không | ✅ Khi state/prop thay đổi |

### Quy Tắc Chọn Loại Component

```
Câu hỏi: Component này có cần gì không?

✅ Fetch data từ DB/API → Server Component
✅ Truy cập file system, env vars bảo mật → Server Component
✅ Render UI tĩnh (header, footer, layout) → Server Component
✅ Không cần interactivity → Server Component

✅ Dùng useState, useReducer → Client Component
✅ Dùng useEffect, useLayoutEffect → Client Component
✅ Xử lý events (onClick, onChange) → Client Component
✅ Dùng browser APIs (localStorage, window) → Client Component
✅ Dùng thư viện chỉ hỗ trợ client (animations, charts) → Client Component
```

---

## 3. Viết Server Components

### Cú Pháp Cơ Bản — Next.js App Router

```javascript
// app/posts/page.tsx — Server Component (mặc định trong App Router)
// Không có 'use client' → đây là Server Component

import { db } from '@/lib/database';

// async/await hoạt động trực tiếp trong Server Component
export default async function PostsPage() {
  // Truy cập database trực tiếp — không cần API route
  const posts = await db.query('SELECT * FROM posts ORDER BY created_at DESC');

  return (
    <main>
      <h1>Bài Viết</h1>
      <ul>
        {posts.map(post => (
          <li key={post.id}>
            <h2>{post.title}</h2>
            <p>Đăng lúc: {new Date(post.created_at).toLocaleDateString('vi-VN')}</p>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

### Server Component Với Prisma — ORM Phổ Biến

```javascript
// app/users/[id]/page.tsx
import { prisma } from '@/lib/prisma';
import { notFound } from 'next/navigation';

interface PageProps {
  params: { id: string };
}

export default async function UserPage({ params }: PageProps) {
  // Truy cập Prisma ORM (Object-Relational Mapper — Ánh Xạ Quan Hệ Đối Tượng) trực tiếp
  const user = await prisma.user.findUnique({
    where: { id: parseInt(params.id) },
    include: {
      posts: { orderBy: { createdAt: 'desc' }, take: 5 },
      _count: { select: { followers: true } },
    },
  });

  if (!user) {
    notFound(); // Render trang 404
  }

  return (
    <div className="user-profile">
      <img src={user.avatar} alt={user.name} />
      <h1>{user.name}</h1>
      <p>{user._count.followers} người theo dõi</p>

      <section>
        <h2>Bài viết gần đây</h2>
        {user.posts.map(post => (
          <article key={post.id}>
            <h3>{post.title}</h3>
            <time>{post.createdAt.toLocaleDateString('vi-VN')}</time>
          </article>
        ))}
      </section>
    </div>
  );
}
```

### Đọc File Hệ Thống Trực Tiếp

```javascript
// app/docs/[slug]/page.tsx
import fs from 'fs/promises';
import path from 'path';
import { marked } from 'marked'; // thư viện parse Markdown — không gửi xuống client!

export default async function DocPage({ params }) {
  const filePath = path.join(process.cwd(), 'content', `${params.slug}.md`);
  
  try {
    const markdown = await fs.readFile(filePath, 'utf-8');
    const html = marked(markdown); // Parse Markdown → HTML trên server
    
    // marked (~50KB) không được gửi xuống client!
    return <article dangerouslySetInnerHTML={{ __html: html }} />;
  } catch {
    notFound();
  }
}
```

---

## 4. Data Fetching Trong Server Components

### Fetch Song Song (Parallel Fetching)

```javascript
// ✅ Fetch song song — hiệu quả nhất
export default async function Dashboard() {
  // Khởi động tất cả fetch cùng lúc — không đợi nhau
  const [user, analytics, notifications] = await Promise.all([
    fetchUser(),
    fetchAnalytics(),
    fetchNotifications(),
  ]);

  return (
    <div>
      <UserSummary user={user} />
      <AnalyticsChart data={analytics} />
      <NotificationList items={notifications} />
    </div>
  );
}
```

### Fetch Tuần Tự Khi Có Phụ Thuộc (Sequential Fetching)

```javascript
// Khi fetch B phụ thuộc vào kết quả fetch A
export default async function UserPosts({ userId }) {
  const user = await fetchUser(userId);
  // Chỉ sau khi có user mới fetch posts
  const posts = await fetchPostsByUser(user.email);

  return (
    <div>
      <UserCard user={user} />
      <PostList posts={posts} />
    </div>
  );
}
```

### Deduplication Tự Động Với Next.js fetch

Next.js mở rộng `fetch` API để tự động deduplication (loại bỏ trùng lặp):

```javascript
// Hàm helper — có thể được gọi từ nhiều nơi
async function getUser(userId: string) {
  // Next.js tự động memoize fetch trong cùng một render pass
  const res = await fetch(`https://api.example.com/users/${userId}`, {
    next: {
      revalidate: 3600, // ISR — Revalidate sau 1 giờ (giây)
      // hoặc: tags: ['user', userId] — để invalidate theo tag
    },
  });
  return res.json();
}

// Nếu cả hai component gọi getUser(userId) cùng userId trong một request:
// Next.js chỉ thực hiện 1 network request duy nhất!
async function Header({ userId }) {
  const user = await getUser(userId); // fetch lần 1
  return <header>{user.name}</header>;
}

async function Sidebar({ userId }) {
  const user = await getUser(userId); // ← request bị deduplicate, dùng cache!
  return <sidebar><Avatar src={user.avatar} /></sidebar>;
}
```

### Fetch Với Cache Control (Kiểm Soát Cache)

```javascript
// Không cache — luôn fetch mới (tương tự SSR)
const data = await fetch('/api/data', { cache: 'no-store' });

// Cache vĩnh viễn (tương tự SSG)
const data = await fetch('/api/data', { cache: 'force-cache' });

// ISR — Incremental Static Regeneration — Tái Tạo Tĩnh Dần
// Revalidate mỗi 60 giây
const data = await fetch('/api/data', { next: { revalidate: 60 } });

// Revalidate theo tag — dùng để invalidate thủ công
const data = await fetch('/api/data', { next: { tags: ['posts'] } });
```

---

## 5. Kết Hợp Server & Client Components

### Pattern Cơ Bản — Server Bọc Client

```
Quy tắc: Server Component CÓ THỂ chứa Client Component
         Client Component KHÔNG THỂ chứa Server Component (trực tiếp)
```

```javascript
// app/dashboard/page.tsx — Server Component (lấy data)
import { AnalyticsChart } from './AnalyticsChart'; // Client Component

export default async function DashboardPage() {
  // Fetch data trên server
  const stats = await fetchDashboardStats();

  return (
    <div>
      <h1>Dashboard</h1>
      {/* Truyền data serializable (có thể serialize) xuống Client Component */}
      <AnalyticsChart data={stats.chartData} />
      {/* Không truyền được: functions, Dates (dùng string), class instances */}
    </div>
  );
}

// app/dashboard/AnalyticsChart.tsx — Client Component
'use client';

import { useState } from 'react';
import { LineChart } from 'recharts'; // thư viện chart chỉ chạy ở client

export function AnalyticsChart({ data }) {
  const [period, setPeriod] = useState('7d');

  return (
    <div>
      <select value={period} onChange={e => setPeriod(e.target.value)}>
        <option value="7d">7 ngày</option>
        <option value="30d">30 ngày</option>
      </select>
      <LineChart data={data} />
    </div>
  );
}
```

### Pattern: Truyền Server Component Làm Children

```javascript
// Cách đúng: truyền Server Component qua children prop
// app/layout.tsx — Server Component
export default async function Layout({ children }) {
  return (
    <html>
      <body>
        <nav>...</nav>
        {/* children là Server Component được render trên server */}
        <main>{children}</main>
      </body>
    </html>
  );
}

// Client Component nhận Server Component làm children
// app/interactive-wrapper.tsx
'use client';

import { useState } from 'react';

export function InteractiveWrapper({ children }) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && (
        // children ở đây có thể là Server Component
        // đã được render trên server trước khi truyền vào
        <div>{children}</div>
      )}
    </div>
  );
}

// app/page.tsx — Cách dùng
export default async function Page() {
  const data = await fetchData(); // Server-side fetch

  return (
    <InteractiveWrapper>
      {/* ServerDataDisplay là Server Component — đã fetch data */}
      <ServerDataDisplay data={data} />
    </InteractiveWrapper>
  );
}
```

### Pattern: Context Trong Môi Trường Có RSC

```javascript
// Context chỉ hoạt động trong Client Component tree
// app/providers.tsx — Client Component bọc providers
'use client';

import { ThemeProvider } from 'next-themes';
import { SessionProvider } from 'next-auth/react';

export function Providers({ children }) {
  return (
    <SessionProvider>
      <ThemeProvider attribute="class" defaultTheme="system">
        {children}
      </ThemeProvider>
    </SessionProvider>
  );
}

// app/layout.tsx — Server Component sử dụng Providers
import { Providers } from './providers';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Providers>
          {children} {/* Có thể là Server Components */}
        </Providers>
      </body>
    </html>
  );
}
```

---

## 6. Suspense & Streaming — Phát Trực Tuyến HTML

**Streaming** (Phát Trực Tuyến) cho phép server gửi HTML từng phần về client ngay khi mỗi phần sẵn sàng, thay vì đợi toàn bộ page render xong.

### Không Streaming vs Có Streaming

```
Không Streaming:
Server: [====fetch 2s====][render][====fetch 1s====] → Gửi HTML sau 3s

Có Streaming:
Server: [====fetch 2s====] → Gửi phần 1 ngay
        [====fetch 1s====] → Gửi phần 2 sau 1s
Người dùng thấy nội dung sau 1s thay vì 3s!
```

### Triển Khai Streaming Với Suspense

```javascript
// app/page.tsx — Server Component với Streaming
import { Suspense } from 'react';
import { UserProfile } from './UserProfile';
import { PostList } from './PostList';
import { Recommendations } from './Recommendations';

export default function HomePage() {
  return (
    <div>
      {/* Phần 1: Hiển thị ngay (không cần fetch) */}
      <header>
        <h1>Chào mừng đến với Blog</h1>
      </header>

      {/* Phần 2: Hiển thị skeleton trong khi UserProfile fetch */}
      <Suspense fallback={<UserProfileSkeleton />}>
        <UserProfile /> {/* async Server Component */}
      </Suspense>

      {/* Phần 3 & 4: Fetch song song, hiển thị khi xong */}
      <div className="grid">
        <Suspense fallback={<PostListSkeleton />}>
          <PostList />
        </Suspense>

        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations />
        </Suspense>
      </div>
    </div>
  );
}

// async Server Component tự động trigger streaming khi dùng trong Suspense
async function UserProfile() {
  const user = await fetchCurrentUser(); // fetch có thể chậm
  return <UserCard user={user} />;
}

async function PostList() {
  const posts = await fetchPosts(); // fetch độc lập, song song với UserProfile
  return <ul>{posts.map(p => <PostItem key={p.id} post={p} />)}</ul>;
}
```

### loading.tsx — Fallback Mặc Định Của Next.js

```javascript
// app/posts/loading.tsx
// Next.js tự động bọc page.tsx trong Suspense với loading.tsx làm fallback
export default function Loading() {
  return (
    <div className="animate-pulse">
      {Array.from({ length: 5 }).map((_, i) => (
        <div key={i} className="post-skeleton">
          <div className="h-6 bg-gray-200 rounded w-3/4 mb-2" />
          <div className="h-4 bg-gray-200 rounded w-1/2" />
        </div>
      ))}
    </div>
  );
}
```

---

## 7. Caching Trong Next.js App Router

Next.js App Router có hệ thống cache nhiều lớp:

```
Request → Router Cache (browser) → Full Route Cache (server) → Data Cache (fetch) → DB
```

### Các Loại Cache

| Cache | Mô Tả | Phạm Vi | Revalidation |
| ----- | ------ | ------- | ------------ |
| **Request Memoization** | Deduplicate fetch trong một request | Per-request | Tự động |
| **Data Cache** | Cache kết quả fetch giữa các request | Persistent | `revalidate`, tags |
| **Full Route Cache** | Cache HTML của static routes | Persistent | Deploy hoặc on-demand |
| **Router Cache** | Cache RSC Payload ở client | Session | Điều hướng |

### Revalidation Theo Tag

```javascript
// app/actions.ts — Server Action để invalidate cache
'use server';
import { revalidateTag } from 'next/cache';

export async function createPost(formData: FormData) {
  await db.posts.create({ data: { title: formData.get('title') } });
  // Invalidate tất cả fetch có tag 'posts'
  revalidateTag('posts');
}

// Trong Server Component — gắn tag vào fetch
async function PostList() {
  const posts = await fetch('/api/posts', {
    next: { tags: ['posts'] } // Tag để invalidate sau này
  }).then(r => r.json());

  return <ul>...</ul>;
}
```

### Revalidation Theo Path

```javascript
'use server';
import { revalidatePath } from 'next/cache';

export async function updatePost(id: string, data: FormData) {
  await db.posts.update({ where: { id }, data: parseFormData(data) });
  // Revalidate trang cụ thể
  revalidatePath(`/posts/${id}`);
  // Hoặc revalidate layout (ảnh hưởng tất cả pages dùng layout này)
  revalidatePath('/posts', 'layout');
}
```

---

## 8. Giới Hạn & Lỗi Thường Gặp

### Những Gì KHÔNG Làm Được Trong Server Components

```javascript
// ❌ KHÔNG ĐƯỢC dùng React hooks
export default async function ServerComp() {
  const [count, setCount] = useState(0); // ❌ Lỗi!
  useEffect(() => {}, []);               // ❌ Lỗi!
  return <div>{count}</div>;
}

// ❌ KHÔNG ĐƯỢC dùng event handlers
export default async function ServerComp() {
  return <button onClick={() => alert('hi')}>Click</button>; // ❌ Lỗi!
}

// ❌ KHÔNG ĐƯỢC dùng browser APIs
export default async function ServerComp() {
  const theme = localStorage.getItem('theme'); // ❌ localStorage không tồn tại ở server
  return <div className={theme}>...</div>;
}

// ❌ KHÔNG ĐƯỢC import 'use client' module từ Server Component
// (nếu module đó dùng browser APIs)
```

### Lỗi Thường Gặp — "You're importing a component that needs useState"

```javascript
// ❌ Lỗi: DatePicker dùng useState nhưng không có 'use client'
// app/form/page.tsx (Server Component)
import DatePicker from 'react-datepicker'; // Thư viện này cần browser

// ✅ Giải pháp: Tạo Client Component wrapper
// app/form/DatePickerClient.tsx
'use client';
import DatePicker from 'react-datepicker';
export { DatePicker as ClientDatePicker };

// app/form/page.tsx — Server Component
import { ClientDatePicker } from './DatePickerClient'; // ✅ Đúng
```

### Props Phải Serializable (Có Thể Tuần Tự Hóa)

```javascript
// Server Component truyền props xuống Client Component
// Props phải có thể serialize thành JSON (tuần tự hóa)

// ❌ KHÔNG được truyền: functions, class instances, non-serializable objects
<ClientComp onSuccess={() => router.push('/')} /> // ❌ function không serialize được

// ✅ Được truyền: string, number, boolean, plain objects, arrays
<ClientComp userId={user.id} name={user.name} /> // ✅ Primitive values
<ClientComp post={{ id: 1, title: 'Test' }} />   // ✅ Plain object
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: React Server Components (RSC) khác gì với SSR (Server-Side Rendering — Hiển Thị Phía Máy Chủ)?**

> - **SSR** (truyền thống): Component render trên server → tạo HTML → gửi về client → client tải JS bundle → hydrate (chạy lại toàn bộ component tree). Component code được gửi xuống cả client.
> - **RSC**: Component chạy **chỉ** trên server → gửi về RSC Payload (định dạng đặc biệt, không phải HTML thuần) → chỉ Client Components được hydrate. Bundle JS nhỏ hơn đáng kể vì Server Component code không gửi xuống client.
> - Điểm mấu chốt: RSC là kiến trúc component mới, SSR là chiến lược rendering — cả hai có thể kết hợp.

**Q: Khi nào nên tách Server Component và Client Component?**

> Nguyên tắc "push client components down the tree" (đẩy Client Component xuống sâu nhất có thể):
> - Server Component: tất cả data fetching, layout, nội dung tĩnh
> - Client Component: chỉ những phần cần interactivity (tương tác), state, browser APIs
> - Ví dụ: Trang blog — toàn bộ là Server Component, chỉ phần comment form là Client Component

**Q: Tại sao không thể import Server Component vào Client Component?**

> Client Component chạy trên cả server (khi SSR) và client. Nếu Client Component import một Server Component, Server Component đó cũng phải chạy ở client — nhưng Server Component có thể truy cập database, file system, environment variables bảo mật — không thể chạy ở client. Do đó, React không cho phép import này. Thay vào đó, dùng `children` prop hoặc composition pattern để kết hợp.

**Q: RSC Payload là gì?**

> RSC Payload là định dạng dữ liệu đặc biệt (không phải JSON, không phải HTML) mà React Server Components tạo ra. Nó mô tả cấu trúc component tree, bao gồm: (1) HTML được render từ Server Components, (2) Placeholder cho Client Components, (3) Dữ liệu cần thiết để client có thể render Client Components đúng cách. Client dùng RSC Payload để "hydrate" chỉ những phần cần thiết, không cần re-render Server Components.

---

## 🔗 Điều Hướng

- **Tiếp theo:** [5-server-actions.md](./5-server-actions.md) — Server Actions (React v19 & Next.js)
- **Trước đó:** [3-swr.md](./3-swr.md) — SWR — Stale-While-Revalidate
- **Liên quan:** [02-hooks/6-react19-new-hooks.md](../02-hooks/6-react19-new-hooks.md) — React v19 Hooks
- **Quay lại:** [README.md](./README.md) — Tổng quan Data Fetching
