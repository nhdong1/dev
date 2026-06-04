# Next.js Pages Router — SSR, SSG, ISR

> Pages Router là kiến trúc routing truyền thống của Next.js (từ v1 đến nay), dựa trên thư mục `pages/`. Vẫn được hỗ trợ đầy đủ và phổ biến trong các dự án kế thừa (legacy projects)

---

## 1. Cấu Trúc Pages Router

```
pages/
├── _app.tsx          ← App wrapper — bọc toàn bộ ứng dụng
├── _document.tsx     ← HTML document — tùy chỉnh <html>, <head>, <body>
├── index.tsx         ← Trang "/"
├── about.tsx         ← Trang "/about"
├── 404.tsx           ← Trang lỗi 404 tùy chỉnh
├── 500.tsx           ← Trang lỗi 500 tùy chỉnh
│
├── blog/
│   ├── index.tsx     ← Trang "/blog"
│   └── [slug].tsx    ← Trang "/blog/:slug" (dynamic route)
│
├── dashboard/
│   └── [...params].tsx  ← Catch-all route "/dashboard/*"
│
└── api/
    └── users.ts      ← API endpoint "/api/users"
```

---

## 2. Ba Chiến Lược Rendering Cốt Lõi

### SSG — Static Site Generation (Tạo Trang Tĩnh)

Render trang tại **thời điểm build** — nhanh nhất, phù hợp nội dung ít thay đổi.

```tsx
// pages/about.tsx — SSG thuần (không cần data fetching)
export default function AboutPage() {
  return <h1>Giới Thiệu</h1>;
}
// Tự động là SSG vì không có getServerSideProps
```

**SSG với dữ liệu — `getStaticProps`:**

```tsx
// pages/blog/index.tsx
import { GetStaticProps, InferGetStaticPropsType } from 'next';

interface Post {
  id: number;
  title: string;
  slug: string;
}

// Chạy tại BUILD TIME — không chạy trên client hay mỗi request
export const getStaticProps: GetStaticProps<{ posts: Post[] }> = async () => {
  const res = await fetch('https://api.example.com/posts');
  const posts: Post[] = await res.json();

  return {
    props: { posts },
    // revalidate: 3600,  // Kích hoạt ISR nếu thêm dòng này
  };
};

export default function BlogPage({ posts }: InferGetStaticPropsType<typeof getStaticProps>) {
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>
          <a href={`/blog/${post.slug}`}>{post.title}</a>
        </li>
      ))}
    </ul>
  );
}
```

**SSG cho dynamic routes — `getStaticPaths`:**

```tsx
// pages/blog/[slug].tsx
import { GetStaticPaths, GetStaticProps } from 'next';

export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await fetchAllPosts();

  return {
    // Pre-render các trang này lúc build
    paths: posts.map(post => ({ params: { slug: post.slug } })),

    // fallback: false     → 404 nếu slug chưa được pre-render
    // fallback: true      → Hiển thị trạng thái loading, render sau
    // fallback: 'blocking' → SSR cho slug mới, không có loading state
    fallback: 'blocking',
  };
};

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const post = await fetchPostBySlug(params!.slug as string);

  if (!post) {
    return { notFound: true }; // Trả 404
  }

  return { props: { post } };
};

export default function BlogPost({ post }: { post: Post }) {
  return <article><h1>{post.title}</h1></article>;
}
```

---

### SSR — Server-Side Rendering (Render Phía Server)

Render trang tại **mỗi request** — luôn fresh data, phù hợp nội dung cá nhân hóa.

```tsx
// pages/dashboard.tsx
import { GetServerSideProps } from 'next';

// Chạy trên SERVER cho MỖI request — có access vào headers, cookies
export const getServerSideProps: GetServerSideProps = async (context) => {
  const { req, res, params, query } = context;

  // Lấy session từ cookie
  const session = await getSession(req);

  if (!session) {
    return {
      redirect: {
        destination: '/login',
        permanent: false,
      },
    };
  }

  const dashboardData = await fetchDashboardData(session.userId);

  return {
    props: { dashboardData, user: session.user },
  };
};

export default function DashboardPage({ dashboardData, user }) {
  return (
    <div>
      <h1>Chào mừng, {user.name}!</h1>
      <DashboardStats data={dashboardData} />
    </div>
  );
}
```

**Khi nào dùng SSR:**
- Dữ liệu cá nhân hóa theo user
- Cần headers, cookies từ request
- SEO quan trọng nhưng data thay đổi liên tục
- Giỏ hàng, notifications, feed cá nhân

---

### ISR — Incremental Static Regeneration (Tái Tạo Tĩnh Tăng Dần)

Kết hợp SSG (nhanh) và SSR (luôn mới) — **tái tạo tự động sau khoảng thời gian xác định**.

```tsx
// pages/products/[id].tsx
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const product = await fetchProduct(params!.id as string);

  return {
    props: { product },
    revalidate: 60, // Tái tạo trang sau 60 giây nếu có request mới
  };
};
```

**Cơ chế ISR — Stale-While-Revalidate:**

```
Request 1 (0s)     → Trả trang từ cache (build)
Request 2 (10s)    → Trả trang từ cache + KHÔNG trigger rebuild (chưa hết 60s)
Request 3 (65s)    → Trả trang từ cache + TRIGGER rebuild ngầm
Request 4 (66s)    → Trả trang MỚI từ rebuild
```

**On-demand ISR — Tái tạo theo yêu cầu:**

```tsx
// pages/api/revalidate.ts
import { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // Xác thực token webhook
  if (req.query.secret !== process.env.REVALIDATION_TOKEN) {
    return res.status(401).json({ message: 'Token không hợp lệ' });
  }

  try {
    // Tái tạo trang cụ thể ngay lập tức
    await res.revalidate('/products/' + req.query.id);
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).send('Lỗi khi tái tạo');
  }
}
```

---

## 3. `_app.tsx` và `_document.tsx`

### `_app.tsx` — App Wrapper

```tsx
// pages/_app.tsx
import type { AppProps } from 'next/app';
import { Provider } from 'react-redux';
import { store } from '../store';
import '../styles/globals.css';

// Mọi trang đều được bọc trong component này
export default function App({ Component, pageProps }: AppProps) {
  return (
    <Provider store={store}>
      <Layout>
        <Component {...pageProps} /> {/* Trang hiện tại */}
      </Layout>
    </Provider>
  );
}
```

### `_document.tsx` — HTML Document Tùy Chỉnh

```tsx
// pages/_document.tsx
import { Html, Head, Main, NextScript } from 'next/document';

export default function Document() {
  return (
    <Html lang="vi">
      <Head>
        {/* Font, meta tags toàn cục */}
        <link rel="preconnect" href="https://fonts.googleapis.com" />
      </Head>
      <body className="antialiased">
        <Main />      {/* Nội dung ứng dụng */}
        <NextScript /> {/* Script Next.js */}
      </body>
    </Html>
  );
}
```

---

## 4. API Routes

```tsx
// pages/api/users/index.ts
import { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  switch (req.method) {
    case 'GET': {
      const users = await db.users.findMany();
      return res.status(200).json(users);
    }

    case 'POST': {
      const { name, email } = req.body;
      const user = await db.users.create({ data: { name, email } });
      return res.status(201).json(user);
    }

    default:
      res.setHeader('Allow', ['GET', 'POST']);
      return res.status(405).json({ error: `Method ${req.method} không được phép` });
  }
}
```

---

## 5. Image và Font Optimization

### `next/image` — Tối Ưu Ảnh Tự Động

```tsx
import Image from 'next/image';

export function ProductCard({ product }) {
  return (
    <div>
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={400}
        height={300}
        priority={true}      // Preload ảnh quan trọng (above-the-fold)
        placeholder="blur"   // Blur placeholder trong khi loading
        blurDataURL="data:image/..." // Base64 preview nhỏ
      />
    </div>
  );
}
```

**Tính năng `next/image`:**
- Tự động chuyển sang WebP/AVIF
- Lazy loading mặc định
- Responsive images với `srcset`
- Tránh CLS — Cumulative Layout Shift (Dịch Chuyển Bố Cục Tích Lũy)

### `next/font` — Tối Ưu Font Tự Động

```tsx
// app/layout.tsx hoặc pages/_app.tsx
import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin', 'vietnamese'],
  variable: '--font-inter',  // CSS variable
});

export default function App({ Component, pageProps }: AppProps) {
  return (
    <main className={inter.className}>
      <Component {...pageProps} />
    </main>
  );
}
```

---

## 6. So Sánh Chi Tiết: Pages Router vs App Router

| Khía Cạnh | Pages Router | App Router |
| --------- | ------------ | ---------- |
| **Thư mục** | `pages/` | `app/` |
| **Data fetching** | `getStaticProps`, `getServerSideProps` | `async` Server Components |
| **Layouts** | `_app.tsx` (một tầng) | `layout.tsx` lồng nhau |
| **Loading states** | Tự xử lý | `loading.tsx` tự động |
| **Error handling** | ErrorBoundary thủ công | `error.tsx` tự động |
| **Streaming** | Không hỗ trợ | Hỗ trợ với Suspense |
| **Server Components** | Không | Có (mặc định) |
| **Learning curve** | Thấp | Cao hơn |
| **Ổn định** | Rất ổn định | Ổn định từ Next.js 14 |
| **Phù hợp** | Dự án cũ, migrate dần | Dự án mới |

---

## 7. Di Chuyển Pages Router → App Router

Next.js hỗ trợ **cùng tồn tại** hai router — bạn có thể di chuyển dần từng trang.

```
my-app/
├── app/           ← App Router (trang mới)
│   └── settings/
│       └── page.tsx
│
└── pages/         ← Pages Router (trang cũ, vẫn hoạt động)
    ├── index.tsx
    └── blog/
        └── [slug].tsx
```

**Chiến lược di chuyển từng bước:**

```
Bước 1: Tạo thư mục app/ với root layout.tsx
Bước 2: Di chuyển _app.tsx logic vào app/layout.tsx
Bước 3: Di chuyển trang ít phức tạp trước (static pages)
Bước 4: Di chuyển trang có data fetching
Bước 5: Di chuyển trang có getServerSideProps sang SSR Server Components
Bước 6: Xóa pages/ khi hoàn thành
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q: Sự khác nhau giữa getStaticProps và getServerSideProps?

**A:**

| | `getStaticProps` | `getServerSideProps` |
| - | - | - |
| **Thời điểm chạy** | Build time | Mỗi request |
| **Output** | Static HTML | Dynamic HTML |
| **Hiệu năng** | Nhanh nhất (CDN cache) | Chậm hơn (server xử lý) |
| **Access request** | Không | Có (headers, cookies) |
| **Phù hợp** | Blog, marketing, catalog | Dashboard, profile, giỏ hàng |

### Q: Giải thích fallback trong getStaticPaths?

**A:**
- `fallback: false` → 404 cho paths chưa build → dùng khi có ít paths cố định
- `fallback: true` → Hiển thị loading state, server build async → user thấy loading rồi nội dung
- `fallback: 'blocking'` → Server build đồng bộ, không có loading → user chờ nhưng thấy trang đầy đủ ngay

### Q: ISR có vấn đề gì cần lưu ý?

**A:**
- **Race condition**: Nhiều request đến gần cùng lúc sau khi hết `revalidate` → chỉ một rebuild được trigger, các request khác vẫn nhận trang cũ
- **Không guarantee thời gian**: `revalidate: 60` nghĩa là "tối thiểu 60 giây mới rebuild", không phải chính xác 60 giây
- **On-demand ISR**: Giải quyết vấn đề trên bằng cách gọi `res.revalidate()` từ webhook khi data thay đổi

---

## ✅ Checklist

- [ ] Hiểu khi nào dùng SSG vs SSR vs ISR
- [ ] Viết `getStaticProps` với TypeScript đúng kiểu
- [ ] Implement dynamic routes với `getStaticPaths` và fallback
- [ ] Xử lý redirect và notFound trong data fetching functions
- [ ] Setup `_app.tsx` với global providers
- [ ] Tối ưu ảnh với `next/image`
- [ ] Implement on-demand ISR với webhook

---

**Tài Liệu Tham Khảo:**
- [Pages Router Docs](https://nextjs.org/docs/pages)
- [Data Fetching Overview](https://nextjs.org/docs/pages/building-your-application/data-fetching)
- [Migration Guide to App Router](https://nextjs.org/docs/app/building-your-application/upgrading/app-router-migration)
