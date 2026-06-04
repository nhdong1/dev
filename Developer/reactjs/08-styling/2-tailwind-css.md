# 2 — Tailwind CSS (Utility-First CSS Framework)

> **Tailwind CSS** là một **utility-first CSS framework** (framework CSS theo tiện ích) — thay vì viết CSS trong file riêng, bạn áp dụng các class nhỏ có sẵn trực tiếp vào HTML/JSX. Kết quả: phát triển UI nhanh hơn, nhất quán hơn, và không phải đặt tên class nữa.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Cài đặt và cấu hình Tailwind CSS với Vite / Next.js
- [ ] Dùng các **utility classes** (class tiện ích) cơ bản: spacing, typography, color, flexbox, grid
- [ ] Xây dựng **responsive design** (thiết kế đáp ứng) với breakpoint prefix
- [ ] Implement **dark mode** (chế độ tối) với `dark:` modifier
- [ ] Tạo **custom design tokens** (token thiết kế tùy chỉnh) trong `tailwind.config.js`
- [ ] Tổ chức Tailwind classes tốt với `clsx` + `tailwind-merge`
- [ ] Hiểu cơ chế **PurgeCSS** (Xóa CSS Không Dùng) / Content scanning

---

## 1. Triết Lý Utility-First

### CSS Truyền Thống

```css
/* Viết CSS riêng cho mỗi component */
.alert {
  display: flex;
  align-items: center;
  padding: 1rem 1.25rem;
  background-color: #fef3c7;
  border: 1px solid #fde68a;
  border-radius: 0.5rem;
  color: #92400e;
}
```

```html
<div class="alert">Cảnh báo!</div>
```

### Tailwind Utility-First

```html
<!-- Không cần file CSS riêng — style trực tiếp trong HTML/JSX -->
<div class="flex items-center px-5 py-4 bg-amber-100 border border-amber-200 rounded-lg text-amber-800">
  Cảnh báo!
</div>
```

**Tại sao điều này tốt?**
- Không phải nghĩ tên class
- Không phải chuyển qua lại giữa file .css và .jsx
- Dễ nhìn thấy style ngay trong JSX
- Khi xóa component, CSS đi theo — không để lại CSS thừa (dead CSS)

---

## 2. Cài Đặt

### Với Vite

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install -D tailwindcss @tailwindcss/vite
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
  ],
})
```

```css
/* src/index.css */
@import "tailwindcss";
```

### Với Next.js

```bash
npx create-next-app@latest my-app --typescript --tailwind --app
# Tailwind được cấu hình sẵn khi chọn --tailwind
```

---

## 3. Các Utility Classes Cơ Bản

### Spacing — Khoảng Cách

Tailwind dùng hệ thống scale số: `1 = 4px`, `2 = 8px`, `4 = 16px`, `8 = 32px`

```jsx
// Padding (bên trong)
<div className="p-4">    {/* padding: 16px tất cả chiều */}
<div className="px-4">   {/* padding-left + padding-right: 16px */}
<div className="py-2">   {/* padding-top + padding-bottom: 8px */}
<div className="pt-6">   {/* padding-top: 24px */}

// Margin (bên ngoài)
<div className="m-4">
<div className="mx-auto"> {/* margin-left: auto; margin-right: auto — căn giữa */}
<div className="mt-8 mb-4">

// Gap (trong flexbox/grid)
<div className="flex gap-4">  {/* gap: 16px giữa các phần tử */}
```

### Typography — Kiểu Chữ

```jsx
<h1 className="text-4xl font-bold tracking-tight text-gray-900">
  Tiêu đề lớn
</h1>

<p className="text-base text-gray-600 leading-relaxed">
  Đoạn văn thông thường
</p>

<span className="text-sm font-medium text-blue-600 uppercase">
  Badge nhỏ
</span>
```

| Class | CSS tương đương |
| ----- | --------------- |
| `text-xs` | `font-size: 0.75rem` |
| `text-sm` | `font-size: 0.875rem` |
| `text-base` | `font-size: 1rem` |
| `text-lg` | `font-size: 1.125rem` |
| `text-xl` | `font-size: 1.25rem` |
| `text-2xl` | `font-size: 1.5rem` |
| `font-normal` | `font-weight: 400` |
| `font-medium` | `font-weight: 500` |
| `font-semibold` | `font-weight: 600` |
| `font-bold` | `font-weight: 700` |
| `leading-tight` | `line-height: 1.25` |
| `leading-relaxed` | `line-height: 1.625` |
| `tracking-tight` | `letter-spacing: -0.025em` |

### Colors — Màu Sắc

Tailwind có bảng màu mặc định với các shade từ 50–950:

```jsx
// Màu chữ
<p className="text-gray-900">Màu tối</p>
<p className="text-blue-600">Màu xanh</p>
<p className="text-red-500">Màu đỏ</p>
<p className="text-green-700">Màu xanh lá đậm</p>

// Màu nền
<div className="bg-white">
<div className="bg-gray-50">  {/* Xám rất nhạt */}
<div className="bg-blue-500">
<div className="bg-gradient-to-r from-blue-500 to-purple-600">

// Màu border
<div className="border border-gray-200">
<div className="border-2 border-blue-500">
```

### Flexbox & Grid

```jsx
// Flexbox
<div className="flex items-center justify-between gap-4">
  <span>Trái</span>
  <span>Phải</span>
</div>

// Flex direction
<div className="flex flex-col gap-2">

// Flex wrap
<div className="flex flex-wrap gap-2">

// CSS Grid
<div className="grid grid-cols-3 gap-6">
  <div>1</div>
  <div>2</div>
  <div>3</div>
</div>

// Responsive grid
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
```

### Sizing — Kích Thước

```jsx
// Width
<div className="w-full">     {/* width: 100% */}
<div className="w-1/2">      {/* width: 50% */}
<div className="w-64">       {/* width: 256px */}
<div className="w-screen">   {/* width: 100vw */}
<div className="max-w-xl">   {/* max-width: 36rem */}

// Height
<div className="h-full">
<div className="h-screen">
<div className="min-h-screen">

// Square
<div className="size-12">    {/* width: 48px; height: 48px */}
```

---

## 4. Responsive Design — Thiết Kế Đáp Ứng

Tailwind dùng **mobile-first approach** (ưu tiên thiết bị di động): class không có prefix áp dụng cho tất cả màn hình, prefix chỉ áp dụng từ breakpoint đó trở lên.

### Breakpoints Mặc Định

| Prefix | Kích Thước Màn Hình |
| ------ | ------------------- |
| *(không có)* | `< 640px` — Mobile |
| `sm:` | `≥ 640px` — Small tablet |
| `md:` | `≥ 768px` — Tablet |
| `lg:` | `≥ 1024px` — Desktop |
| `xl:` | `≥ 1280px` — Large desktop |
| `2xl:` | `≥ 1536px` — Extra large |

```jsx
<div className="
  grid
  grid-cols-1
  sm:grid-cols-2
  lg:grid-cols-3
  xl:grid-cols-4
  gap-4
  p-4
  md:p-8
">
  {/* 1 cột trên mobile, 2 cột trên tablet, 3 trên desktop, 4 trên XL */}
</div>
```

```jsx
// Navigation responsive
<nav className="flex flex-col md:flex-row items-start md:items-center gap-4">
  <a className="text-sm md:text-base font-medium">Trang chủ</a>
  <a className="text-sm md:text-base font-medium">Giới thiệu</a>
</nav>
```

---

## 5. Dark Mode — Chế Độ Tối

### Cấu Hình Dark Mode

```css
/* index.css — Tailwind v4 */
@import "tailwindcss";

@custom-variant dark (&:where(.dark, .dark *));
```

### Sử Dụng `dark:` Modifier

```jsx
<div className="bg-white dark:bg-gray-900 min-h-screen">
  <h1 className="text-gray-900 dark:text-white text-2xl font-bold">
    Tiêu đề
  </h1>

  <p className="text-gray-600 dark:text-gray-400">
    Đoạn văn thích nghi với dark mode
  </p>

  <button className="
    bg-blue-600 hover:bg-blue-700
    dark:bg-blue-500 dark:hover:bg-blue-400
    text-white px-4 py-2 rounded-lg
  ">
    Nút bấm
  </button>
</div>
```

### Toggle Dark Mode

```tsx
// useDarkMode.ts — Custom hook quản lý dark mode
import { useState, useEffect } from 'react';

function useDarkMode() {
  const [isDark, setIsDark] = useState(() => {
    return localStorage.getItem('theme') === 'dark';
  });

  useEffect(() => {
    const root = document.documentElement;
    if (isDark) {
      root.classList.add('dark');
      localStorage.setItem('theme', 'dark');
    } else {
      root.classList.remove('dark');
      localStorage.setItem('theme', 'light');
    }
  }, [isDark]);

  return { isDark, toggle: () => setIsDark(prev => !prev) };
}

// DarkModeToggle.tsx
function DarkModeToggle() {
  const { isDark, toggle } = useDarkMode();

  return (
    <button
      onClick={toggle}
      className="p-2 rounded-lg bg-gray-100 dark:bg-gray-800"
    >
      {isDark ? '☀️' : '🌙'}
    </button>
  );
}
```

---

## 6. State Modifiers — Biến Đổi Theo Trạng Thái

```jsx
<button className="
  bg-blue-600
  hover:bg-blue-700      /* Khi di chuột vào */
  active:bg-blue-800     /* Khi đang click */
  focus:outline-none
  focus:ring-2
  focus:ring-blue-500
  focus:ring-offset-2    /* Hiệu ứng focus cho accessibility */
  disabled:opacity-50
  disabled:cursor-not-allowed
  transition-colors      /* Hiệu ứng chuyển màu mượt */
  duration-200
">
  Nút bấm
</button>

{/* Group hover — hover vào cha, style con thay đổi */}
<div className="group border rounded-lg p-4 hover:border-blue-500">
  <h3 className="text-gray-900 group-hover:text-blue-600">Tiêu đề</h3>
  <p className="text-gray-500 group-hover:text-gray-700">Mô tả</p>
</div>

{/* Peer — anh chị em tương tác nhau */}
<input className="peer border rounded px-3 py-2" placeholder="Email" />
<p className="text-red-500 hidden peer-invalid:block">Email không hợp lệ</p>
```

---

## 7. Tùy Chỉnh Trong `tailwind.config.js`

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    './index.html',
    './src/**/*.{js,ts,jsx,tsx}',
  ],
  darkMode: 'class',
  theme: {
    extend: {
      // Thêm màu tùy chỉnh — mở rộng, không ghi đè màu mặc định
      colors: {
        brand: {
          50:  '#eff6ff',
          100: '#dbeafe',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          900: '#1e3a8a',
        },
      },

      // Font chữ tùy chỉnh
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
      },

      // Khoảng cách tùy chỉnh
      spacing: {
        '18': '4.5rem',
        '22': '5.5rem',
        '88': '22rem',
      },

      // Border radius tùy chỉnh
      borderRadius: {
        '4xl': '2rem',
      },

      // Animation tùy chỉnh
      keyframes: {
        shimmer: {
          '0%': { backgroundPosition: '-200% 0' },
          '100%': { backgroundPosition: '200% 0' },
        },
      },
      animation: {
        shimmer: 'shimmer 2s linear infinite',
      },

      // Box shadow tùy chỉnh
      boxShadow: {
        'card': '0 2px 8px rgba(0, 0, 0, 0.08)',
        'card-hover': '0 8px 24px rgba(0, 0, 0, 0.12)',
      },
    },
  },
  plugins: [
    // Plugin chính thức của Tailwind
    // require('@tailwindcss/typography'),    // Prose styles cho markdown
    // require('@tailwindcss/forms'),         // Reset form styles đẹp hơn
    // require('@tailwindcss/aspect-ratio'), // aspect-ratio utilities
  ],
};
```

---

## 8. Tổ Chức Class Với `clsx` và `tailwind-merge`

### Vấn Đề Với Class Dài

```jsx
// ❌ Khó đọc, khó maintain
<button className="inline-flex items-center justify-center rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50 bg-primary text-primary-foreground hover:bg-primary/90 h-10 px-4 py-2">
```

### Giải Pháp: `cn()` Utility

```bash
npm install clsx tailwind-merge
```

```typescript
// src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

/**
 * cn — Class Names — Hàm kết hợp class Tailwind thông minh
 * clsx: xử lý điều kiện (nếu false thì bỏ qua)
 * twMerge: giải quyết xung đột class Tailwind (bg-red-500 vs bg-blue-500 → dùng cái cuối)
 */
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
// Button.tsx — Component có variant và size
import { cn } from '@/lib/utils';

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
}

const buttonVariants = {
  base: 'inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-blue-500 disabled:opacity-50 disabled:cursor-not-allowed',
  variant: {
    primary: 'bg-blue-600 text-white hover:bg-blue-700',
    secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
    danger: 'bg-red-600 text-white hover:bg-red-700',
    ghost: 'hover:bg-gray-100 text-gray-700',
  },
  size: {
    sm: 'h-8 px-3 text-sm',
    md: 'h-10 px-4 text-sm',
    lg: 'h-11 px-6 text-base',
  },
};

function Button({
  variant = 'primary',
  size = 'md',
  className,
  children,
  ...props
}: ButtonProps) {
  return (
    <button
      className={cn(
        buttonVariants.base,
        buttonVariants.variant[variant],
        buttonVariants.size[size],
        className  // Cho phép override từ bên ngoài
      )}
      {...props}
    >
      {children}
    </button>
  );
}
```

### Tại Sao Cần `tailwind-merge`?

```tsx
// Vấn đề: clsx không giải quyết xung đột Tailwind
cn('bg-red-500', 'bg-blue-500')
// clsx → "bg-red-500 bg-blue-500" → cả 2 class đều có, browser dùng class sau
// tailwind-merge → "bg-blue-500" → class sau thắng, class trước bị bỏ
```

---

## 9. Ví Dụ Thực Tế: Product Card

```tsx
// ProductCard.tsx — Tailwind full
import { cn } from '@/lib/utils';

interface ProductCardProps {
  title: string;
  price: number;
  category: string;
  imageUrl: string;
  inStock: boolean;
  onAddToCart: () => void;
  className?: string;
}

function ProductCard({
  title,
  price,
  category,
  imageUrl,
  inStock,
  onAddToCart,
  className,
}: ProductCardProps) {
  return (
    <article
      className={cn(
        'group flex flex-col rounded-xl overflow-hidden border border-gray-200',
        'bg-white dark:bg-gray-800 dark:border-gray-700',
        'transition-shadow hover:shadow-lg',
        className
      )}
    >
      {/* Hình ảnh */}
      <div className="relative overflow-hidden aspect-video">
        <img
          src={imageUrl}
          alt={title}
          className="w-full h-full object-cover transition-transform duration-300 group-hover:scale-105"
        />
        {!inStock && (
          <div className="absolute inset-0 bg-black/50 flex items-center justify-center">
            <span className="text-white font-semibold text-sm">Hết hàng</span>
          </div>
        )}
      </div>

      {/* Nội dung */}
      <div className="flex flex-col gap-2 p-4 flex-1">
        {/* Badge danh mục */}
        <span className="text-xs font-medium text-blue-700 dark:text-blue-400 bg-blue-50 dark:bg-blue-900/30 px-2 py-0.5 rounded-full w-fit">
          {category}
        </span>

        <h3 className="font-semibold text-gray-900 dark:text-white line-clamp-2">
          {title}
        </h3>

        <p className="text-xl font-bold text-green-600 dark:text-green-400 mt-auto">
          ${price.toFixed(2)}
        </p>
      </div>

      {/* Footer */}
      <div className="p-4 pt-0">
        <button
          onClick={onAddToCart}
          disabled={!inStock}
          className={cn(
            'w-full py-2 px-4 rounded-lg text-sm font-medium transition-colors',
            inStock
              ? 'bg-blue-600 hover:bg-blue-700 text-white'
              : 'bg-gray-100 text-gray-400 cursor-not-allowed'
          )}
        >
          {inStock ? 'Thêm vào giỏ hàng' : 'Hết hàng'}
        </button>
      </div>
    </article>
  );
}
```

---

## 10. Arbitrary Values — Giá Trị Tùy Chỉnh

Khi cần giá trị cụ thể không có trong scale mặc định:

```jsx
// Dùng ngoặc vuông cho giá trị tùy chỉnh
<div className="w-[350px] h-[200px]">
<div className="bg-[#1a1a2e]">
<div className="text-[13px] leading-[1.4]">
<div className="top-[72px]">           {/* Cho positioning */}
<div className="grid-cols-[1fr_2fr_1fr]"> {/* Grid template columns */}
<div className="shadow-[0_4px_20px_rgba(0,0,0,0.15)]">

/* Arbitrary CSS property */
<div className="[mask-image:linear-gradient(to_bottom,white,transparent)]">
```

---

## 11. Performance — PurgeCSS và JIT

### JIT — Just-In-Time Compiler (Biên Dịch Ngay Lúc Dùng)

Tailwind v3+ mặc định dùng JIT engine — chỉ tạo CSS cho các class **thực sự được dùng** trong code.

```javascript
// tailwind.config.js — Content scanning
export default {
  content: [
    './src/**/*.{js,ts,jsx,tsx}', // Quét tất cả file trong src
    './index.html',
  ],
  // ...
};
```

```
Build output:
- Development: CSS đầy đủ, rebuild nhanh
- Production: chỉ ~5-20KB CSS (thay vì 3MB+ nếu không purge)
```

### Anti-pattern — Class Dynamic Bị Purge

```tsx
// ❌ Tailwind không quét string concatenation — class sẽ bị purge
const color = 'red';
<div className={`text-${color}-500`}>  {/* "text-red-500" bị xóa khỏi build! */}

// ✅ Safelist hoặc dùng class đầy đủ
const colorMap = {
  red: 'text-red-500',
  blue: 'text-blue-500',
  green: 'text-green-500',
};
<div className={colorMap[color]}>  {/* "text-red-500" được giữ lại */}
```

---

## 12. Tailwind Plugins Hữu Ích

```bash
# Typography — prose classes cho markdown/rich text content
npm install -D @tailwindcss/typography

# Forms — reset và style đẹp cho form elements
npm install -D @tailwindcss/forms

# Animate — animation utilities
npm install tailwindcss-animate
```

```jsx
// Typography plugin — prose class
<article className="prose lg:prose-xl dark:prose-invert max-w-none">
  {/* Markdown content được render đẹp tự động */}
  <h1>Tiêu đề</h1>
  <p>Nội dung với link, bold, code...</p>
</article>
```

---

## 13. Ưu & Nhược Điểm

### ✅ Ưu Điểm

```
1. Tốc độ phát triển cao — không cần đặt tên, không cần chuyển file
2. Nhất quán — design system tích hợp sẵn (spacing, colors, typography)
3. Responsive cực dễ — md:, lg:, xl: prefix ngay trong JSX
4. Dark mode đơn giản — dark: prefix
5. Bundle nhỏ trong production — JIT chỉ include class đang dùng
6. Không có dead CSS — class nào dùng thì có, không dùng thì không có
7. Tương thích RSC — không có JavaScript runtime
8. Tùy biến cao — design tokens, extend theme
```

### ❌ Nhược Điểm

```
1. JSX trông "bẩn" — className rất dài với nhiều class
   → Giải pháp: tách component nhỏ hơn, dùng cn() utility

2. Learning curve — phải nhớ tên class Tailwind (thường mất 1-2 tuần)
   → Giải pháp: IntelliSense extension cho VSCode

3. Không phù hợp highly custom design — khi design cần pixel-perfect, không follow grid
   → Giải pháp: Arbitrary values [450px], hoặc CSS Modules

4. Dynamic class bị purge — không thể tạo class name bằng string interpolation

5. Khó đọc cho người mới — cần biết Tailwind mới hiểu style đang làm gì
```

---

## 14. Extensions Hữu Ích

```
VSCode Extensions:
→ "Tailwind CSS IntelliSense" — autocomplete, hover preview, linting
→ "Headwind" — tự động sắp xếp class theo thứ tự nhất quán
→ "PostCSS Language Support" — syntax highlighting

ESLint:
→ "eslint-plugin-tailwindcss" — lint class Tailwind, phát hiện lỗi chính tả
```

---

## 15. Câu Hỏi Phỏng Vấn

**Q: Tailwind CSS là gì và tại sao nó phổ biến?**
> Tailwind là utility-first CSS framework — áp dụng class nhỏ trực tiếp vào HTML thay vì viết CSS riêng. Phổ biến vì tăng tốc độ phát triển, đảm bảo nhất quán design, tránh đặt tên class, và bundle size nhỏ nhờ JIT purging.

**Q: Tailwind khác Bootstrap như thế nào?**
> Bootstrap cung cấp component có sẵn (button, card, navbar...) với style cứng — bạn dùng và override. Tailwind cung cấp utility primitives — bạn tự tổ hợp để tạo component theo ý muốn. Tailwind linh hoạt hơn, dễ customize hơn.

**Q: Làm sao xử lý dynamic class trong Tailwind?**
> Không được dùng string interpolation (`text-${color}-500`) vì JIT không thể quét. Thay vào đó: tạo mapping object với full class names, hoặc thêm class vào `safelist` trong config.

**Q: `twMerge` dùng để làm gì?**
> Giải quyết xung đột class Tailwind. Khi merge `bg-red-500 bg-blue-500`, `twMerge` giữ `bg-blue-500` (class cuối), thay vì để cả hai class tồn tại gây hành vi không xác định.

---

## 16. Checklist Thực Hành

```
□ Setup Tailwind với Vite + TypeScript
□ Tạo Button component với variant (primary, secondary, danger) bằng cn()
□ Build responsive card grid (1 cột mobile → 3 cột desktop)
□ Implement dark mode toggle
□ Customize theme: thêm brand colors và font
□ Tạo Navigation bar responsive với hamburger menu
□ Build form với validation states (error, success)
□ Animate element với Tailwind animation classes
```

---

## 🔗 Điều Hướng

- **Trước đó:** [1-css-modules.md](./1-css-modules.md) — CSS Modules
- **Tiếp theo:** [3-styled-components.md](./3-styled-components.md) — Styled Components
- **README chủ đề:** [README.md](./README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
