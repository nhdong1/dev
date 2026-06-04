# 1 — CSS Modules (Module CSS Theo Phạm Vi Component)

> **CSS Modules** là một hệ thống module hóa CSS, trong đó tất cả tên class và animation được tự động **scoped locally** (giới hạn cục bộ) theo mặc định. Kết quả là bạn có thể dùng tên class đơn giản mà không lo xung đột với style của component khác.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Giải thích cơ chế **local scoping** (giới hạn phạm vi cục bộ) của CSS Modules
- [ ] Tạo và sử dụng file `.module.css` với cú pháp đúng
- [ ] Kết hợp nhiều class với `clsx` hoặc `classnames`
- [ ] Xử lý **global styles** (style toàn cục) và **composition** (kế thừa class)
- [ ] Sử dụng CSS Modules với TypeScript
- [ ] Áp dụng **CSS Custom Properties** (Biến CSS Tùy Chỉnh) cho theming

---

## 1. CSS Modules Là Gì?

### Vấn Đề CSS Toàn Cục

```css
/* styles.css — file CSS thông thường */
.button {
  background: blue;
  color: white;
}
```

```css
/* Ở một file khác cũng có .button */
.button {
  background: red; /* Ghi đè lên .button trên! */
}
```

**Vấn đề:** CSS là toàn cục — hai file cùng dùng tên `.button` sẽ xung đột.

### Giải Pháp CSS Modules

```css
/* Button.module.css */
.button {
  background: blue;
  color: white;
}
```

```jsx
// Button.jsx
import styles from './Button.module.css';

function Button({ children }) {
  return <button className={styles.button}>{children}</button>;
}
```

**Kết quả:** Trình duyệt nhận HTML:
```html
<!-- Tên class được hash tự động → không bao giờ xung đột -->
<button class="Button_button__xK9mF">Click me</button>
```

CSS Modules tự động biến đổi tên class thành dạng `[filename]_[localname]__[hash]`.

---

## 2. Cú Pháp Cơ Bản

### Tạo File CSS Module

```css
/* Card.module.css */

/* Class cục bộ (mặc định) */
.card {
  border: 1px solid #e5e7eb;
  border-radius: 0.5rem;
  padding: 1.5rem;
  background: white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #111827;
  margin-bottom: 0.5rem;
}

.description {
  font-size: 0.875rem;
  color: #6b7280;
  line-height: 1.5;
}

/* Pseudo-class — hoạt động bình thường */
.card:hover {
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

/* Media query — hoạt động bình thường */
@media (max-width: 768px) {
  .card {
    padding: 1rem;
  }
}
```

### Sử Dụng Trong Component

```jsx
// Card.jsx
import styles from './Card.module.css';

function Card({ title, description }) {
  return (
    <div className={styles.card}>
      <h2 className={styles.title}>{title}</h2>
      <p className={styles.description}>{description}</p>
    </div>
  );
}

export default Card;
```

---

## 3. Kết Hợp Nhiều Class

### Cách 1: Template Literals (Chuỗi Nội Suy)

```jsx
import styles from './Button.module.css';

function Button({ variant = 'primary', size = 'md', disabled }) {
  const className = `
    ${styles.button}
    ${styles[variant]}
    ${styles[size]}
    ${disabled ? styles.disabled : ''}
  `.trim();

  return (
    <button className={className} disabled={disabled}>
      Click me
    </button>
  );
}
```

### Cách 2: Dùng `clsx` (Khuyến Nghị)

```bash
npm install clsx
```

```jsx
import styles from './Button.module.css';
import clsx from 'clsx';

function Button({ variant = 'primary', size = 'md', disabled, className }) {
  return (
    <button
      className={clsx(
        styles.button,           // Luôn có
        styles[variant],         // Dynamic — dựa vào prop
        styles[size],
        disabled && styles.disabled,
        className                // Cho phép truyền class từ ngoài vào
      )}
      disabled={disabled}
    >
      Click me
    </button>
  );
}
```

```css
/* Button.module.css */
.button {
  border: none;
  cursor: pointer;
  border-radius: 0.375rem;
  font-weight: 500;
  transition: all 0.2s;
}

/* Variants (Biến thể) */
.primary {
  background: #3b82f6;
  color: white;
}

.primary:hover {
  background: #2563eb;
}

.secondary {
  background: #f3f4f6;
  color: #374151;
}

.secondary:hover {
  background: #e5e7eb;
}

.danger {
  background: #ef4444;
  color: white;
}

/* Sizes (Kích thước) */
.sm {
  padding: 0.25rem 0.75rem;
  font-size: 0.875rem;
}

.md {
  padding: 0.5rem 1rem;
  font-size: 1rem;
}

.lg {
  padding: 0.75rem 1.5rem;
  font-size: 1.125rem;
}

/* States (Trạng thái) */
.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

---

## 4. Global Styles — Style Toàn Cục

Đôi khi cần style không bị scope lại — ví dụ để target class của thư viện bên ngoài:

```css
/* MyComponent.module.css */

.wrapper {
  padding: 1rem;
}

/* :global() — thoát khỏi scoping của CSS Modules */
.wrapper :global(.react-datepicker) {
  font-size: 0.875rem;
}

/* :global không có selector cha — toàn bộ là global */
:global(.my-global-class) {
  color: red;
}
```

```jsx
// Dùng class global trực tiếp bằng string — không qua styles object
<div className={`${styles.wrapper} my-global-class`}>
```

---

## 5. Composition — Kế Thừa Style

CSS Modules hỗ trợ `composes` — cho phép một class kế thừa style của class khác:

```css
/* base.module.css */
.baseButton {
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.2s;
}
```

```css
/* Button.module.css */
.primary {
  composes: baseButton from './base.module.css'; /* Kế thừa từ file khác */
  background: #3b82f6;
  color: white;
}

.secondary {
  composes: baseButton from './base.module.css';
  background: #f3f4f6;
  color: #374151;
}

/* Kế thừa từ class trong cùng file */
.largeButton {
  composes: primary;
  padding: 0.75rem 1.5rem;
  font-size: 1.125rem;
}
```

---

## 6. CSS Modules Với TypeScript

### Vấn Đề

TypeScript không tự động biết file `.module.css` export gì:

```tsx
import styles from './Button.module.css'; // ❌ TS lỗi: "Cannot find module"
```

### Giải Pháp 1: Declaration File (File Khai Báo)

```typescript
// src/types/css-modules.d.ts
declare module '*.module.css' {
  const classes: Record<string, string>;
  export default classes;
}

declare module '*.module.scss' {
  const classes: Record<string, string>;
  export default classes;
}
```

### Giải Pháp 2: Dùng `typescript-plugin-css-modules` (Khuyến Nghị)

```bash
npm install -D typescript-plugin-css-modules
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "plugins": [
      {
        "name": "typescript-plugin-css-modules"
      }
    ]
  }
}
```

Plugin này cung cấp **auto-complete** (tự động gợi ý) cho tên class khi gõ `styles.`.

---

## 7. Theming Với CSS Custom Properties

CSS Modules hoạt động rất tốt với **CSS Custom Properties** (Thuộc Tính Tùy Chỉnh CSS) — còn gọi là **CSS Variables** (Biến CSS):

```css
/* globals.css — style toàn cục */
:root {
  /* Light theme (Chủ đề sáng) */
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;
  --color-bg: #ffffff;
  --color-text: #111827;
  --color-border: #e5e7eb;
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
}

/* Dark theme (Chủ đề tối) */
[data-theme='dark'] {
  --color-primary: #60a5fa;
  --color-primary-hover: #93c5fd;
  --color-bg: #111827;
  --color-text: #f9fafb;
  --color-border: #374151;
}
```

```css
/* Button.module.css — dùng biến thay vì giá trị cứng */
.button {
  background: var(--color-primary);
  color: white;
  border: 1px solid var(--color-border);
}

.button:hover {
  background: var(--color-primary-hover);
}
```

```tsx
// ThemeToggle.tsx
function ThemeToggle() {
  const toggleTheme = () => {
    const root = document.documentElement;
    const current = root.getAttribute('data-theme');
    root.setAttribute('data-theme', current === 'dark' ? 'light' : 'dark');
  };

  return <button onClick={toggleTheme}>Toggle Theme</button>;
}
```

---

## 8. Ví Dụ Thực Tế: Card Component

```css
/* ProductCard.module.css */
.card {
  display: flex;
  flex-direction: column;
  border: 1px solid var(--color-border, #e5e7eb);
  border-radius: 0.75rem;
  overflow: hidden;
  background: var(--color-bg, white);
  transition: transform 0.2s, box-shadow 0.2s;
}

.card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
}

.image {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;
}

.body {
  padding: 1rem;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.badge {
  display: inline-block;
  padding: 0.125rem 0.5rem;
  background: #dbeafe;
  color: #1d4ed8;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 500;
  width: fit-content;
}

.title {
  font-size: 1rem;
  font-weight: 600;
  color: var(--color-text, #111827);
  margin: 0;
}

.price {
  font-size: 1.25rem;
  font-weight: 700;
  color: #059669;
  margin-top: auto;
}

.footer {
  padding: 0.75rem 1rem;
  border-top: 1px solid var(--color-border, #e5e7eb);
  display: flex;
  gap: 0.5rem;
}
```

```tsx
// ProductCard.tsx
import styles from './ProductCard.module.css';
import clsx from 'clsx';

interface ProductCardProps {
  title: string;
  price: number;
  category: string;
  imageUrl: string;
  onAddToCart: () => void;
  className?: string;
}

function ProductCard({
  title,
  price,
  category,
  imageUrl,
  onAddToCart,
  className,
}: ProductCardProps) {
  return (
    <article className={clsx(styles.card, className)}>
      <img src={imageUrl} alt={title} className={styles.image} />

      <div className={styles.body}>
        <span className={styles.badge}>{category}</span>
        <h3 className={styles.title}>{title}</h3>
        <p className={styles.price}>${price.toFixed(2)}</p>
      </div>

      <div className={styles.footer}>
        <button onClick={onAddToCart}>Thêm vào giỏ hàng</button>
      </div>
    </article>
  );
}

export default ProductCard;
```

---

## 9. SCSS Với CSS Modules

Vite và webpack hỗ trợ `.module.scss` với **Sass** (bộ tiền xử lý CSS):

```bash
npm install -D sass
```

```scss
/* Button.module.scss */
$primary: #3b82f6;
$primary-dark: #2563eb;
$transition: all 0.2s ease;

.button {
  background: $primary;
  border: none;
  border-radius: 0.375rem;
  color: white;
  cursor: pointer;
  font-weight: 500;
  transition: $transition;

  &:hover {
    background: $primary-dark;
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  /* Nested variants */
  &.secondary {
    background: #f3f4f6;
    color: #374151;

    &:hover {
      background: #e5e7eb;
    }
  }

  /* Size variants */
  &.sm {
    padding: 0.25rem 0.75rem;
    font-size: 0.875rem;
  }

  &.md {
    padding: 0.5rem 1rem;
    font-size: 1rem;
  }
}
```

---

## 10. Ưu & Nhược Điểm

### ✅ Ưu Điểm

```
1. Zero runtime overhead — không có JavaScript nào chạy để apply style
2. Tên class an toàn — không lo xung đột, dù đặt tên đơn giản
3. Không học cú pháp mới — CSS thuần, chỉ thêm .module.css vào tên file
4. Dead code elimination — bundler có thể tree-shake CSS không dùng
5. Tương thích hoàn toàn với React Server Components (RSC)
6. Browser caching — CSS được tách ra file riêng, cache hiệu quả
7. DevTools thân thiện — tên class vẫn có ý nghĩa (không phải hash thuần túy)
```

### ❌ Nhược Điểm

```
1. Dynamic styles hạn chế — không thể dễ dàng tạo style từ JavaScript runtime
   → Ví dụ: background-color: props.color → phải dùng inline style hoặc CSS Variables

2. Không có Theming API — phải tự setup CSS Variables
   → Không có ThemeProvider hay design tokens system tích hợp

3. Nhiều file — mỗi component cần 2 file: .tsx và .module.css
   → Với dự án lớn, số file tăng gấp đôi

4. Cần cấu hình TypeScript riêng để có auto-complete

5. Class composition phức tạp hơn CSS-in-JS khi logic phức tạp
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q: CSS Modules khác CSS thuần như thế nào?**
> CSS Modules tự động hash tên class để tạo scope cục bộ, tránh xung đột tên. CSS thuần có scope toàn cục — class `.button` ở bất kỳ file nào cũng ảnh hưởng lẫn nhau.

**Q: Làm sao dùng style toàn cục trong CSS Modules?**
> Dùng `:global()` selector: `.wrapper :global(.third-party-class) { }` — phần trong `:global()` sẽ không bị hash.

**Q: CSS Modules có hoạt động với React Server Components không?**
> Có, hoàn toàn tương thích vì CSS Modules không có runtime JavaScript — chỉ là CSS thuần được xử lý ở build time.

**Q: Khi nào nên chọn CSS Modules thay vì Tailwind?**
> CSS Modules phù hợp khi: cần custom design phức tạp, team quen CSS truyền thống, muốn tách biệt rõ CSS và JSX. Tailwind tốt hơn khi: phát triển nhanh, cần consistency cao, làm việc với design system có sẵn.

---

## 12. Checklist Thực Hành

```
□ Tạo component Button với CSS Modules (primary, secondary, danger variants)
□ Kết hợp class động với clsx
□ Setup TypeScript declaration cho .module.css
□ Implement dark mode với CSS Custom Properties
□ Tạo Card component có hover effect
□ Thử SCSS Modules với nesting và variables
□ So sánh output CSS giữa CSS Modules và CSS thuần trong DevTools
```

---

## 🔗 Điều Hướng

- **Tiếp theo:** [2-tailwind-css.md](./2-tailwind-css.md) — Tailwind CSS utility-first
- **README chủ đề:** [README.md](./README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
