# Server Actions — Hành Động Phía Server (React v19 & Next.js)

> **Server Actions** (Hành Động Phía Server) là tính năng cho phép định nghĩa và gọi **hàm bất đồng bộ chạy trực tiếp trên server** từ Client Components hoặc Server Components — không cần tạo API route riêng. Được giới thiệu trong Next.js 13.4 và chính thức ổn định trong React v19, Server Actions là nền tảng của mô hình form và mutation hiện đại.

---

## 📌 Mục Lục

1. [Server Actions Là Gì & Tại Sao Dùng](#1-server-actions-là-gì--tại-sao-dùng)
2. [Khai Báo Server Actions](#2-khai-báo-server-actions)
3. [Dùng Trong Forms — Cách Đơn Giản Nhất](#3-dùng-trong-forms--cách-đơn-giản-nhất)
4. [useActionState — Quản Lý State Từ Action](#4-useactionstate--quản-lý-state-từ-action)
5. [useFormStatus — Trạng Thái Submit Của Form](#5-useformstatus--trạng-thái-submit-của-form)
6. [useOptimistic — Cập Nhật Lạc Quan](#6-useoptimistic--cập-nhật-lạc-quan)
7. [Server Actions Từ Event Handlers](#7-server-actions-từ-event-handlers)
8. [Validation & Error Handling](#8-validation--error-handling)
9. [Revalidation & Redirect Sau Action](#9-revalidation--redirect-sau-action)
10. [Bảo Mật Server Actions](#10-bảo-mật-server-actions)
11. [Câu Hỏi Phỏng Vấn](#11-câu-hỏi-phỏng-vấn)

---

## 1. Server Actions Là Gì & Tại Sao Dùng

### Trước Server Actions — Cách Cũ

```javascript
// 1. Tạo API route riêng (app/api/posts/route.ts)
export async function POST(request: Request) {
  const data = await request.json();
  await db.posts.create({ data });
  return Response.json({ success: true });
}

// 2. Client fetch đến API route đó
async function createPost(title) {
  const res = await fetch('/api/posts', {
    method: 'POST',
    body: JSON.stringify({ title }),
  });
  return res.json();
}

// 3. Dùng trong component
function CreatePostForm() {
  const handleSubmit = async (e) => {
    e.preventDefault();
    await createPost(title);
    router.refresh();
  };
}
```

**Vấn đề:** Nhiều lớp boilerplate (mã soạn sẵn), phải viết cả API route lẫn client fetch function.

### Với Server Actions

```javascript
// app/actions.ts — Server Action
'use server'; // Chỉ thị: hàm này chạy trên server

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  await db.posts.create({ data: { title } });
  revalidatePath('/posts'); // Làm mới cache
}

// app/posts/new/page.tsx — Dùng trực tiếp trong form
export default function NewPostPage() {
  return (
    <form action={createPost}> {/* action nhận Server Action! */}
      <input name="title" placeholder="Tiêu đề" />
      <button type="submit">Đăng bài</button>
    </form>
  );
}
```

**Lợi ích:** Không cần API route, không cần fetch thủ công, hoạt động ngay cả khi JS chưa tải (Progressive Enhancement — Cải Tiến Lũy Tiến).

### Lợi Ích Của Server Actions

- ✅ **Không cần API routes** — Loại bỏ boilerplate
- ✅ **Type-safe** — TypeScript đảm bảo tính đúng đắn khi compile
- ✅ **Progressive Enhancement** — Form vẫn submit được dù JS tắt
- ✅ **Tự động CSRF protection** — Next.js tự xử lý (CSRF — Cross-Site Request Forgery — Giả Mạo Request Liên Site)
- ✅ **Truy cập trực tiếp database, file system, secrets**
- ✅ **Kết hợp tốt với `useActionState`, `useFormStatus`, `useOptimistic`**

---

## 2. Khai Báo Server Actions

### Cách 1: Directive `'use server'` Ở Đầu File

```javascript
// app/actions/posts.ts
'use server'; // Toàn bộ file là Server Actions

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  if (!title || !content) {
    return { error: 'Tiêu đề và nội dung không được rỗng' };
  }

  const post = await db.posts.create({
    data: { title, content, authorId: getCurrentUserId() },
  });

  revalidatePath('/posts');
  redirect(`/posts/${post.id}`);
}

export async function updatePost(id: string, formData: FormData) {
  const title = formData.get('title') as string;
  await db.posts.update({ where: { id }, data: { title } });
  revalidatePath(`/posts/${id}`);
}

export async function deletePost(id: string) {
  await db.posts.delete({ where: { id } });
  revalidatePath('/posts');
  redirect('/posts');
}
```

### Cách 2: Directive `'use server'` Trong Server Component

```javascript
// app/posts/[id]/page.tsx — Server Component
export default async function PostPage({ params }) {
  const post = await db.posts.findUnique({ where: { id: params.id } });

  // Định nghĩa Server Action ngay trong Server Component
  async function handleDelete() {
    'use server'; // Khai báo action inline
    await db.posts.delete({ where: { id: params.id } });
    revalidatePath('/posts');
    redirect('/posts');
  }

  return (
    <article>
      <h1>{post.title}</h1>
      <form action={handleDelete}>
        <button type="submit">Xóa bài viết</button>
      </form>
    </article>
  );
}
```

### Truyền Tham Số Bổ Sung — bind()

```javascript
'use server';

export async function updatePostById(postId: string, formData: FormData) {
  const title = formData.get('title') as string;
  await db.posts.update({ where: { id: postId }, data: { title } });
  revalidatePath(`/posts/${postId}`);
}

// Trong component — dùng bind để "đóng gói" tham số
function EditPostForm({ post }) {
  // Tạo action mới với postId đã bind sẵn
  const updateAction = updatePostById.bind(null, post.id);

  return (
    <form action={updateAction}>
      <input name="title" defaultValue={post.title} />
      <button type="submit">Cập nhật</button>
    </form>
  );
}
```

---

## 3. Dùng Trong Forms — Cách Đơn Giản Nhất

### Form Cơ Bản Với Server Action

```javascript
// app/contact/page.tsx — Progressive Enhancement Form
'use server' là ở file actions.ts — không cần khai báo lại

import { sendContactEmail } from './actions';

export default function ContactPage() {
  return (
    <form action={sendContactEmail}>
      <div>
        <label htmlFor="name">Họ tên</label>
        <input id="name" name="name" type="text" required />
      </div>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" required />
      </div>
      <div>
        <label htmlFor="message">Tin nhắn</label>
        <textarea id="message" name="message" rows={5} required />
      </div>
      <button type="submit">Gửi tin nhắn</button>
    </form>
  );
}
```

### Server Action Xử Lý FormData

```javascript
// app/contact/actions.ts
'use server';

import { z } from 'zod'; // Zod — thư viện validation (xác thực) schema

const contactSchema = z.object({
  name: z.string().min(2, 'Tên phải có ít nhất 2 ký tự'),
  email: z.string().email('Email không hợp lệ'),
  message: z.string().min(10, 'Tin nhắn phải có ít nhất 10 ký tự'),
});

export async function sendContactEmail(formData: FormData) {
  // Trích xuất dữ liệu từ FormData
  const rawData = {
    name: formData.get('name'),
    email: formData.get('email'),
    message: formData.get('message'),
  };

  // Validate với Zod
  const result = contactSchema.safeParse(rawData);
  if (!result.success) {
    return {
      success: false,
      errors: result.error.flatten().fieldErrors,
    };
  }

  // Gửi email (ví dụ: dùng Resend, SendGrid, Nodemailer)
  await emailService.send({
    to: 'admin@example.com',
    subject: `Liên hệ từ ${result.data.name}`,
    text: result.data.message,
    replyTo: result.data.email,
  });

  return { success: true };
}
```

---

## 4. useActionState — Quản Lý State Từ Action

`useActionState` (React v19 — trước đây là `useFormState` trong React DOM) cho phép quản lý state từ kết quả của Server Action.

```javascript
// Cú pháp
const [state, formAction, isPending] = useActionState(action, initialState);
```

### Ví Dụ Đầy Đủ — Form Đăng Ký

```javascript
// app/auth/actions.ts
'use server';

import { z } from 'zod';

const registerSchema = z.object({
  email: z.string().email('Email không hợp lệ'),
  password: z.string().min(8, 'Mật khẩu phải có ít nhất 8 ký tự'),
  name: z.string().min(2, 'Tên phải có ít nhất 2 ký tự'),
});

// ActionState type — kiểu trả về của action
type RegisterState = {
  errors?: {
    email?: string[];
    password?: string[];
    name?: string[];
    general?: string;
  };
  success?: boolean;
};

export async function registerUser(
  prevState: RegisterState, // state trước đó (luôn là tham số đầu tiên)
  formData: FormData
): Promise<RegisterState> {
  const rawData = {
    email: formData.get('email'),
    password: formData.get('password'),
    name: formData.get('name'),
  };

  const result = registerSchema.safeParse(rawData);
  if (!result.success) {
    return {
      errors: result.error.flatten().fieldErrors,
    };
  }

  try {
    const existingUser = await db.users.findUnique({
      where: { email: result.data.email },
    });

    if (existingUser) {
      return { errors: { general: 'Email đã được đăng ký' } };
    }

    await db.users.create({
      data: {
        email: result.data.email,
        password: await hash(result.data.password),
        name: result.data.name,
      },
    });

    return { success: true };
  } catch {
    return { errors: { general: 'Đăng ký thất bại. Vui lòng thử lại.' } };
  }
}
```

```javascript
// app/auth/register/page.tsx
'use client';

import { useActionState } from 'react';
import { registerUser } from '../actions';

export default function RegisterPage() {
  const [state, formAction, isPending] = useActionState(
    registerUser,
    { errors: {} } // initialState — trạng thái ban đầu
  );

  if (state.success) {
    return (
      <div className="success">
        <h2>Đăng ký thành công!</h2>
        <p>Vui lòng kiểm tra email để xác nhận tài khoản.</p>
      </div>
    );
  }

  return (
    <form action={formAction}>
      {/* Hiển thị lỗi chung */}
      {state.errors?.general && (
        <div className="error-banner">{state.errors.general}</div>
      )}

      <div className="form-group">
        <label htmlFor="name">Họ tên</label>
        <input id="name" name="name" type="text" aria-required />
        {state.errors?.name && (
          <span className="field-error">{state.errors.name[0]}</span>
        )}
      </div>

      <div className="form-group">
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" aria-required />
        {state.errors?.email && (
          <span className="field-error">{state.errors.email[0]}</span>
        )}
      </div>

      <div className="form-group">
        <label htmlFor="password">Mật khẩu</label>
        <input id="password" name="password" type="password" aria-required />
        {state.errors?.password && (
          <span className="field-error">{state.errors.password[0]}</span>
        )}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Đang đăng ký...' : 'Đăng ký'}
      </button>
    </form>
  );
}
```

---

## 5. useFormStatus — Trạng Thái Submit Của Form

`useFormStatus` đọc trạng thái của **form cha gần nhất**, cho phép tạo submit button thông minh mà không cần prop drilling (khoan truyền props).

```javascript
// components/SubmitButton.tsx — Reusable submit button
'use client';

import { useFormStatus } from 'react-dom';

interface SubmitButtonProps {
  label?: string;
  loadingLabel?: string;
}

export function SubmitButton({
  label = 'Xác nhận',
  loadingLabel = 'Đang xử lý...',
}: SubmitButtonProps) {
  // useFormStatus phải dùng trong component CON của form
  const { pending, data, method, action } = useFormStatus();

  return (
    <button
      type="submit"
      disabled={pending}
      className={pending ? 'btn-loading' : 'btn-primary'}
      aria-busy={pending}
    >
      {pending ? (
        <>
          <Spinner className="mr-2" />
          {loadingLabel}
        </>
      ) : label}
    </button>
  );
}

// Sử dụng trong form
function CreatePostForm() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Tiêu đề" />
      <textarea name="content" placeholder="Nội dung" />
      {/* SubmitButton tự biết form đang submit nhờ useFormStatus */}
      <SubmitButton label="Đăng bài" loadingLabel="Đang đăng..." />
    </form>
  );
}
```

### Kết Hợp useActionState + useFormStatus

```javascript
// Pattern thực tế đầy đủ
function CommentForm({ postId }) {
  const [state, formAction, isPending] = useActionState(
    addComment,
    { error: null }
  );

  return (
    <form action={formAction}>
      <input type="hidden" name="postId" value={postId} />
      <textarea
        name="content"
        placeholder="Viết bình luận..."
        required
        minLength={1}
      />
      {state.error && <p className="error">{state.error}</p>}
      {/* SubmitButton dùng useFormStatus — không cần truyền isPending */}
      <SubmitButton label="Bình luận" loadingLabel="Đang gửi..." />
    </form>
  );
}
```

---

## 6. useOptimistic — Cập Nhật Lạc Quan

`useOptimistic` cho phép hiển thị kết quả dự kiến của action ngay lập tức, trong khi action thực sự đang chạy trên server.

```javascript
'use client';

import { useOptimistic, useActionState } from 'react';
import { addComment, deleteComment } from './actions';

interface Comment {
  id: string;
  content: string;
  author: string;
  pending?: boolean; // Đánh dấu comment đang chờ xác nhận từ server
}

function CommentSection({ postId, initialComments }: {
  postId: string;
  initialComments: Comment[];
}) {
  const [state, formAction] = useActionState(addComment, null);

  // useOptimistic(actualState, reducerFn)
  // reducerFn: (currentState, optimisticValue) => newState
  const [optimisticComments, addOptimisticComment] = useOptimistic(
    initialComments,
    (currentComments: Comment[], newComment: Comment) => [
      ...currentComments,
      { ...newComment, pending: true }, // pending=true để style khác
    ]
  );

  const handleSubmit = async (formData: FormData) => {
    const content = formData.get('content') as string;

    // Bước 1: Cập nhật UI ngay (optimistic)
    addOptimisticComment({
      id: `temp-${Date.now()}`,
      content,
      author: 'Bạn',
      pending: true,
    });

    // Bước 2: Gọi Server Action thực sự
    await formAction(formData);
    // Sau khi action hoàn thành, optimistic state tự động rollback
    // và actualState (initialComments) được cập nhật
  };

  return (
    <section>
      <h3>Bình luận ({optimisticComments.length})</h3>

      <ul>
        {optimisticComments.map(comment => (
          <li
            key={comment.id}
            className={comment.pending ? 'opacity-50 italic' : ''}
          >
            <strong>{comment.author}</strong>
            <p>{comment.content}</p>
            {comment.pending && <span className="badge">Đang gửi...</span>}
          </li>
        ))}
      </ul>

      <form action={handleSubmit}>
        <input type="hidden" name="postId" value={postId} />
        <textarea name="content" placeholder="Bình luận của bạn..." required />
        <SubmitButton label="Gửi bình luận" />
      </form>
    </section>
  );
}
```

---

## 7. Server Actions Từ Event Handlers

Server Actions không chỉ dùng với form — có thể gọi từ bất kỳ event handler nào.

```javascript
'use client';

import { useState, useTransition } from 'react';
import { toggleLike, followUser } from './actions';

function PostCard({ post, currentUserId }) {
  const [isPending, startTransition] = useTransition();
  const [isLiked, setIsLiked] = useState(post.isLikedByCurrentUser);
  const [likeCount, setLikeCount] = useState(post.likesCount);

  // Gọi Server Action từ onClick với Optimistic UI
  const handleLike = () => {
    // Cập nhật UI ngay lập tức
    setIsLiked(prev => !prev);
    setLikeCount(prev => isLiked ? prev - 1 : prev + 1);

    // startTransition — đánh dấu đây là transition (chuyển đổi)
    // React có thể interrupt (ngắt) transition để xử lý input có độ ưu tiên cao hơn
    startTransition(async () => {
      try {
        await toggleLike(post.id, currentUserId);
        // Server xác nhận — không cần làm gì thêm nếu dùng revalidatePath
      } catch {
        // Rollback nếu thất bại
        setIsLiked(prev => !prev);
        setLikeCount(prev => isLiked ? prev + 1 : prev - 1);
        alert('Không thể like. Vui lòng thử lại.');
      }
    });
  };

  return (
    <div className="post-card">
      <h3>{post.title}</h3>
      <button
        onClick={handleLike}
        disabled={isPending}
        className={isLiked ? 'liked' : ''}
      >
        ❤️ {likeCount}
      </button>
    </div>
  );
}
```

---

## 8. Validation & Error Handling

### Validation Đầy Đủ Với Zod + Trả Về Errors

```javascript
// lib/validations/post.ts
import { z } from 'zod';

export const createPostSchema = z.object({
  title: z
    .string()
    .min(5, 'Tiêu đề phải có ít nhất 5 ký tự')
    .max(200, 'Tiêu đề không được quá 200 ký tự'),
  content: z
    .string()
    .min(100, 'Nội dung phải có ít nhất 100 ký tự'),
  category: z.enum(['tech', 'lifestyle', 'travel', 'food'], {
    errorMap: () => ({ message: 'Danh mục không hợp lệ' }),
  }),
  tags: z.string().optional(),
});

export type CreatePostInput = z.infer<typeof createPostSchema>;
```

```javascript
// app/actions/posts.ts
'use server';

import { createPostSchema } from '@/lib/validations/post';
import { auth } from '@/lib/auth';
import { redirect } from 'next/navigation';
import { revalidatePath } from 'next/cache';

type ActionResult = {
  success?: boolean;
  data?: { id: string };
  errors?: Record<string, string[]>;
  message?: string;
};

export async function createPost(
  prevState: ActionResult,
  formData: FormData
): Promise<ActionResult> {
  // Xác thực người dùng đã đăng nhập
  const session = await auth();
  if (!session?.user) {
    return { message: 'Bạn cần đăng nhập để đăng bài' };
  }

  const rawData = {
    title: formData.get('title'),
    content: formData.get('content'),
    category: formData.get('category'),
    tags: formData.get('tags'),
  };

  // Validate
  const validated = createPostSchema.safeParse(rawData);
  if (!validated.success) {
    return {
      errors: validated.error.flatten().fieldErrors,
      message: 'Vui lòng kiểm tra lại thông tin',
    };
  }

  try {
    const post = await db.posts.create({
      data: {
        ...validated.data,
        authorId: session.user.id,
        tags: validated.data.tags?.split(',').map(t => t.trim()),
      },
    });

    revalidatePath('/posts');
    revalidatePath('/dashboard');

    // Trả về success thay vì redirect để component hiển thị thông báo
    return { success: true, data: { id: post.id } };
  } catch (error) {
    console.error('createPost error:', error);
    return { message: 'Không thể tạo bài viết. Vui lòng thử lại sau.' };
  }
}
```

---

## 9. Revalidation & Redirect Sau Action

```javascript
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';
import { redirect } from 'next/navigation';

export async function updateAndRedirect(id: string, formData: FormData) {
  await db.posts.update({
    where: { id },
    data: { title: formData.get('title') as string },
  });

  // revalidatePath — xóa cache của path cụ thể
  revalidatePath(`/posts/${id}`);        // Trang post detail
  revalidatePath('/posts');               // Trang danh sách
  revalidatePath('/admin/posts', 'page'); // Chỉ revalidate loại 'page'

  // revalidateTag — xóa cache theo tag (dùng với fetch(..., { next: { tags: ['posts'] } }))
  revalidateTag('posts');
  revalidateTag(`post-${id}`);

  // redirect — điều hướng sau khi action hoàn thành
  // redirect phải được gọi BÊN NGOÀI try/catch (nó throw RedirectError)
  redirect(`/posts/${id}`);
}

// Nếu cần điều hướng trong try/catch:
export async function createAndRedirect(formData: FormData) {
  let redirectPath: string | null = null;

  try {
    const post = await db.posts.create({ data: parseFormData(formData) });
    revalidatePath('/posts');
    redirectPath = `/posts/${post.id}`;
  } catch (error) {
    return { error: 'Tạo bài viết thất bại' };
  }

  // redirect ở đây — bên ngoài try/catch
  if (redirectPath) redirect(redirectPath);
}
```

---

## 10. Bảo Mật Server Actions

Server Actions tự động có một số bảo vệ, nhưng vẫn cần xử lý bảo mật đúng cách.

### Authentication & Authorization

```javascript
'use server';

import { auth } from '@/lib/auth';
import { db } from '@/lib/db';

// Luôn xác thực trong mỗi Server Action — không tin tưởng client
export async function deletePost(postId: string) {
  // Bước 1: Xác thực người dùng
  const session = await auth();
  if (!session?.user?.id) {
    throw new Error('Chưa đăng nhập'); // Hoặc return error state
  }

  // Bước 2: Xác minh quyền — chỉ author mới được xóa
  const post = await db.posts.findUnique({
    where: { id: postId },
    select: { authorId: true },
  });

  if (!post) {
    throw new Error('Bài viết không tồn tại');
  }

  // Bước 3: Kiểm tra quyền sở hữu
  if (post.authorId !== session.user.id) {
    throw new Error('Bạn không có quyền xóa bài viết này');
  }

  // Bước 4: Thực hiện thao tác
  await db.posts.delete({ where: { id: postId } });
  revalidatePath('/posts');
}
```

### Input Sanitization — Làm Sạch Đầu Vào

```javascript
'use server';

import DOMPurify from 'isomorphic-dompurify'; // Làm sạch HTML để tránh XSS

export async function createComment(formData: FormData) {
  const rawContent = formData.get('content') as string;

  // Làm sạch HTML — ngăn XSS (Cross-Site Scripting — Tấn Công Chèn Script)
  const sanitizedContent = DOMPurify.sanitize(rawContent, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong'],
  });

  if (!sanitizedContent.trim()) {
    return { error: 'Bình luận không được rỗng' };
  }

  await db.comments.create({ data: { content: sanitizedContent } });
  revalidatePath('/posts');
}
```

### Rate Limiting — Giới Hạn Tần Suất

```javascript
'use server';

import { rateLimit } from '@/lib/rate-limit';

// Rate Limiting — Giới Hạn Tần Suất yêu cầu để ngăn spam/DDoS
export async function submitForm(formData: FormData) {
  const session = await auth();
  const identifier = session?.user?.id || 'anonymous';

  const { success, remaining } = await rateLimit.check(
    `form-submit:${identifier}`,
    { limit: 5, window: '1m' } // Tối đa 5 lần submit mỗi phút
  );

  if (!success) {
    return {
      error: `Bạn đã gửi quá nhiều yêu cầu. Vui lòng thử lại sau ${remaining}s`
    };
  }

  // Xử lý form...
}
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: Server Actions khác gì với API Routes trong Next.js?**

> | | Server Actions | API Routes |
> |---|---|---|
> | **Khai báo** | `'use server'` directive | `app/api/*/route.ts` |
> | **Gọi từ form** | `action={serverAction}` trực tiếp | Cần `fetch('/api/...')` |
> | **Type safety** | ✅ End-to-end TypeScript | Thủ công |
> | **Progressive Enhancement** | ✅ Works without JS | ❌ Cần JS |
> | **CSRF protection** | ✅ Tự động | Phải tự xử lý |
> | **Phù hợp** | Mutations, form submissions | Public APIs, webhooks, third-party |

**Q: useFormStatus khác useActionState như thế nào?**

> - `useFormStatus`: Đọc trạng thái (`pending`) của **form cha gần nhất**. Phải dùng trong component **con** của form. Không liên quan đến action nào cụ thể. Lý tưởng cho reusable submit button.
> - `useActionState`: Hook quản lý **state và action** của một form cụ thể. Nhận action function và trả về `[state, wrappedAction, isPending]`. Dùng trong component chứa form để xử lý response từ action.

**Q: Làm thế nào để handle lỗi trong Server Actions?**

> Có hai cách: (1) **Return error state** — trả về object có `error` field, component nhận và hiển thị. Phù hợp cho validation errors, business logic errors. (2) **Throw error** — Next.js bắt và hiển thị error boundary gần nhất. Phù hợp cho unexpected errors, unauthorized access. Quan trọng: `redirect()` và `notFound()` trong Next.js throw error đặc biệt — không bắt chúng trong try/catch.

**Q: Progressive Enhancement (Cải Tiến Lũy Tiến) với Server Actions là gì?**

> Server Actions kết hợp với HTML form `action` attribute cho phép form hoạt động ngay cả khi JavaScript chưa tải hoặc bị tắt. Browser gửi form submission bình thường → Server Action xử lý → Server render response mới. Khi JS tải xong, React "enhance" form để intercept submission và xử lý mà không cần page reload. Đây là progressive enhancement — bắt đầu từ HTML thuần, cải tiến dần khi JS có sẵn.

---

## 🔗 Điều Hướng

- **Trước đó:** [4-react-server-components.md](./4-react-server-components.md) — RSC — React Server Components
- **Quay lại:** [README.md](./README.md) — Tổng quan Data Fetching
- **Tiếp theo:** [06-performance/](../06-performance/) — Tối ưu hiệu năng
- **Liên quan:** [02-hooks/6-react19-new-hooks.md](../02-hooks/6-react19-new-hooks.md) — useActionState, useFormStatus, useOptimistic
