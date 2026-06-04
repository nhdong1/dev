# 4 — Emotion (CSS-in-JS Hiệu Năng Cao)

> **Emotion** là thư viện **CSS-in-JS** (CSS Trong JavaScript) tập trung vào hiệu năng và tính linh hoạt. Emotion cung cấp hai cách viết style: **`css` prop** (đơn giản, trực tiếp) và **`styled` API** (giống Styled Components). Đây là thư viện nền tảng của **Material UI v5+**.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Cài đặt và cấu hình Emotion với React
- [ ] Dùng **`css` prop** để style trực tiếp trên element
- [ ] Sử dụng **`styled` API** (tương tự Styled Components)
- [ ] Tạo reusable styles với **`css` helper** — object và string syntax
- [ ] Implement **theming** với `ThemeProvider` từ `@emotion/react`
- [ ] Tận dụng **`cx`** (className concatenation — nối tên class) cho conditional styles
- [ ] Hiểu điểm khác biệt giữa Emotion và Styled Components

---

## 1. Emotion vs Styled Components

| Tiêu Chí | Emotion | Styled Components |
| -------- | ------- | ----------------- |
| **Bundle size** | ~7KB gzip | ~14KB gzip |
| **Performance** | Nhanh hơn (ít overhead hơn) | Chậm hơn một chút |
| **API style** | `css` prop + `styled` | Chủ yếu `styled` |
| **SSR** | Tốt, dễ setup hơn | Cần `ServerStyleSheet` |
| **Được dùng bởi** | Material UI v5+, Chakra UI | Nhiều dự án cũ |
| **String syntax** | ✅ Hỗ trợ | ✅ Hỗ trợ |
| **Object syntax** | ✅ Hỗ trợ tốt hơn | ⚠️ Hạn chế |

---

## 2. Cài Đặt

### `@emotion/react` — Core package

```bash
npm install @emotion/react @emotion/styled
```

### Với Vite — Cần Babel Plugin

```bash
npm install -D @emotion/babel-plugin
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      jsxImportSource: '@emotion/react', // Cho phép dùng css prop
      babel: {
        plugins: ['@emotion/babel-plugin'],
      },
    }),
  ],
});
```

### Hoặc Dùng JSX Pragma (Pragma — Chỉ Thị JSX)

```tsx
/** @jsxImportSource @emotion/react */
// Dòng trên phải ở đầu file để enable css prop
import { css } from '@emotion/react';
```

---

## 3. `css` Prop — Cách Dùng Trực Tiếp

`css` prop là tính năng nổi bật nhất của Emotion — không cần tạo styled component riêng:

### String Syntax — Cú Pháp Chuỗi

```tsx
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react';

function Button({ children }) {
  return (
    <button
      css={css`
        background: #3b82f6;
        color: white;
        border: none;
        padding: 0.5rem 1rem;
        border-radius: 0.375rem;
        cursor: pointer;

        &:hover {
          background: #2563eb;
        }
      `}
    >
      {children}
    </button>
  );
}
```

### Object Syntax — Cú Pháp Object (Thường Được Ưa Thích Với TypeScript)

```tsx
function Button({ children, disabled }) {
  return (
    <button
      css={{
        background: disabled ? '#9ca3af' : '#3b82f6',
        color: 'white',
        border: 'none',
        padding: '0.5rem 1rem',
        borderRadius: '0.375rem',
        cursor: disabled ? 'not-allowed' : 'pointer',
        opacity: disabled ? 0.5 : 1,
        transition: 'background 0.2s',

        '&:hover': !disabled && {
          background: '#2563eb',
        },
      }}
      disabled={disabled}
    >
      {children}
    </button>
  );
}
```

---

## 4. `css` Helper — Tạo Style Có Thể Tái Sử Dụng

```tsx
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react';

// Định nghĩa style snippet có thể tái sử dụng
const baseButtonStyles = css`
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: none;
  border-radius: 0.375rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  padding: 0.5rem 1rem;
`;

const primaryStyles = css`
  background: #3b82f6;
  color: white;

  &:hover {
    background: #2563eb;
  }
`;

const dangerStyles = css`
  background: #ef4444;
  color: white;

  &:hover {
    background: #dc2626;
  }
`;

// Kết hợp nhiều css snippets — Emotion tự merge thông minh
function Button({ variant = 'primary', children }) {
  return (
    <button
      css={[
        baseButtonStyles,
        variant === 'primary' && primaryStyles,
        variant === 'danger' && dangerStyles,
      ]}
    >
      {children}
    </button>
  );
}
```

---

## 5. `styled` API — Giống Styled Components

```tsx
import styled from '@emotion/styled';

// Cú pháp hoàn toàn giống Styled Components
const Button = styled.button`
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  cursor: pointer;

  &:hover {
    background: #2563eb;
  }
`;

// Object syntax với styled
const Card = styled.div({
  border: '1px solid #e5e7eb',
  borderRadius: '0.75rem',
  padding: '1.5rem',
  background: 'white',
  boxShadow: '0 1px 3px rgba(0, 0, 0, 0.1)',

  '&:hover': {
    boxShadow: '0 4px 6px rgba(0, 0, 0, 0.1)',
  },
});
```

### Dynamic Styles Với Props

```tsx
import styled from '@emotion/styled';

interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
  fullWidth?: boolean;
}

// String syntax với props
const Button = styled.button<ButtonProps>`
  background: ${props => {
    switch (props.variant) {
      case 'secondary': return '#f3f4f6';
      case 'danger': return '#ef4444';
      default: return '#3b82f6';
    }
  }};
  color: ${props => props.variant === 'secondary' ? '#374151' : 'white'};
  width: ${props => props.fullWidth ? '100%' : 'auto'};
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  cursor: pointer;
`;

// Object syntax với props
const ButtonObj = styled.button<ButtonProps>(props => ({
  background: props.variant === 'danger' ? '#ef4444' : '#3b82f6',
  color: 'white',
  width: props.fullWidth ? '100%' : 'auto',
  border: 'none',
  padding: '0.5rem 1rem',
  borderRadius: '0.375rem',
  cursor: 'pointer',

  '&:hover': {
    background: props.variant === 'danger' ? '#dc2626' : '#2563eb',
  },
}));
```

---

## 6. Theming Với `@emotion/react`

```typescript
// theme.ts
export interface Theme {
  colors: {
    primary: string;
    primaryHover: string;
    background: string;
    surface: string;
    text: string;
    textMuted: string;
    border: string;
  };
  spacing: {
    xs: string;
    sm: string;
    md: string;
    lg: string;
  };
  borderRadius: {
    sm: string;
    md: string;
    lg: string;
  };
}

export const lightTheme: Theme = {
  colors: {
    primary: '#3b82f6',
    primaryHover: '#2563eb',
    background: '#ffffff',
    surface: '#f9fafb',
    text: '#111827',
    textMuted: '#6b7280',
    border: '#e5e7eb',
  },
  spacing: {
    xs: '0.25rem',
    sm: '0.5rem',
    md: '1rem',
    lg: '1.5rem',
  },
  borderRadius: {
    sm: '0.25rem',
    md: '0.375rem',
    lg: '0.5rem',
  },
};

export const darkTheme: Theme = {
  colors: {
    primary: '#60a5fa',
    primaryHover: '#93c5fd',
    background: '#111827',
    surface: '#1f2937',
    text: '#f9fafb',
    textMuted: '#9ca3af',
    border: '#374151',
  },
  spacing: lightTheme.spacing,
  borderRadius: lightTheme.borderRadius,
};

// Khai báo augmentation để TypeScript biết theme type
declare module '@emotion/react' {
  export interface Theme extends import('./theme').Theme {}
}
```

```tsx
// App.tsx
import { ThemeProvider } from '@emotion/react';
import { lightTheme, darkTheme } from './theme';
import { useState } from 'react';

function App() {
  const [isDark, setIsDark] = useState(false);

  return (
    <ThemeProvider theme={isDark ? darkTheme : lightTheme}>
      <button onClick={() => setIsDark(p => !p)}>Toggle</button>
      <MyComponents />
    </ThemeProvider>
  );
}
```

### Dùng Theme Trong Components

```tsx
/** @jsxImportSource @emotion/react */
import { css, useTheme } from '@emotion/react';
import styled from '@emotion/styled';

// Cách 1: styled API — theme tự động từ ThemeProvider
const ThemedButton = styled.button`
  background: ${props => props.theme.colors.primary};
  color: white;
  padding: ${props => props.theme.spacing.sm} ${props => props.theme.spacing.md};
  border-radius: ${props => props.theme.borderRadius.md};
`;

// Cách 2: css prop với useTheme hook
function ThemedCard({ children }) {
  const theme = useTheme();

  return (
    <div
      css={{
        background: theme.colors.surface,
        border: `1px solid ${theme.colors.border}`,
        borderRadius: theme.borderRadius.lg,
        padding: theme.spacing.lg,
      }}
    >
      {children}
    </div>
  );
}

// Cách 3: css helper với theme function
const cardStyles = (theme) => css`
  background: ${theme.colors.surface};
  border: 1px solid ${theme.colors.border};
  border-radius: ${theme.borderRadius.lg};
  padding: ${theme.spacing.lg};
`;
```

---

## 7. Global Styles Với Emotion

```tsx
import { Global, css } from '@emotion/react';

const globalStyles = css`
  *, *::before, *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    line-height: 1.6;
  }
`;

function App() {
  return (
    <>
      <Global styles={globalStyles} />
      {/* Hoặc dùng theme */}
      <Global
        styles={theme => css`
          body {
            background: ${theme.colors.background};
            color: ${theme.colors.text};
          }
        `}
      />
      <MyComponents />
    </>
  );
}
```

---

## 8. `cx` — Kết Hợp Class Names

```tsx
import { cx, css } from '@emotion/css'; // @emotion/css — standalone package

const base = css`
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
`;

const active = css`
  background: #3b82f6;
  color: white;
`;

const disabled = css`
  opacity: 0.5;
  cursor: not-allowed;
`;

// cx kết hợp class names (kể cả falsy values)
function Button({ isActive, isDisabled, children }) {
  return (
    <button
      className={cx(
        base,
        isActive && active,
        isDisabled && disabled
      )}
    >
      {children}
    </button>
  );
}
```

---

## 9. Keyframe Animations

```tsx
/** @jsxImportSource @emotion/react */
import { keyframes, css } from '@emotion/react';
import styled from '@emotion/styled';

const pulse = keyframes`
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
`;

const float = keyframes`
  0%, 100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
`;

// Dùng trong styled component
const LoadingDot = styled.div`
  width: 0.75rem;
  height: 0.75rem;
  border-radius: 50%;
  background: #3b82f6;
  animation: ${pulse} 1.5s ease-in-out infinite;
`;

// Dùng với css prop và delay cho nhiều dots
function LoadingDots() {
  return (
    <div css={{ display: 'flex', gap: '0.5rem' }}>
      {[0, 0.2, 0.4].map((delay, i) => (
        <div
          key={i}
          css={css`
            width: 0.75rem;
            height: 0.75rem;
            border-radius: 50%;
            background: #3b82f6;
            animation: ${pulse} 1.5s ease-in-out ${delay}s infinite;
          `}
        />
      ))}
    </div>
  );
}
```

---

## 10. Ví Dụ Thực Tế: Alert Component

```tsx
/** @jsxImportSource @emotion/react */
import { css, useTheme } from '@emotion/react';
import styled from '@emotion/styled';

type AlertVariant = 'info' | 'success' | 'warning' | 'error';

interface AlertConfig {
  bg: string;
  border: string;
  text: string;
  icon: string;
}

const alertConfig: Record<AlertVariant, AlertConfig> = {
  info: {
    bg: '#eff6ff',
    border: '#bfdbfe',
    text: '#1e40af',
    icon: 'ℹ️',
  },
  success: {
    bg: '#f0fdf4',
    border: '#bbf7d0',
    text: '#166534',
    icon: '✅',
  },
  warning: {
    bg: '#fffbeb',
    border: '#fde68a',
    text: '#92400e',
    icon: '⚠️',
  },
  error: {
    bg: '#fef2f2',
    border: '#fecaca',
    text: '#991b1b',
    icon: '❌',
  },
};

interface AlertProps {
  variant?: AlertVariant;
  title?: string;
  children: React.ReactNode;
  onClose?: () => void;
}

function Alert({ variant = 'info', title, children, onClose }: AlertProps) {
  const config = alertConfig[variant];

  return (
    <div
      css={{
        display: 'flex',
        gap: '0.75rem',
        padding: '1rem 1.25rem',
        backgroundColor: config.bg,
        border: `1px solid ${config.border}`,
        borderRadius: '0.5rem',
        color: config.text,
        position: 'relative',
      }}
      role="alert"
    >
      <span css={{ fontSize: '1.25rem', flexShrink: 0 }}>{config.icon}</span>

      <div css={{ flex: 1 }}>
        {title && (
          <p css={{ fontWeight: 600, marginBottom: '0.25rem' }}>{title}</p>
        )}
        <div css={{ fontSize: '0.875rem' }}>{children}</div>
      </div>

      {onClose && (
        <button
          onClick={onClose}
          css={{
            position: 'absolute',
            top: '0.75rem',
            right: '0.75rem',
            background: 'transparent',
            border: 'none',
            cursor: 'pointer',
            color: config.text,
            opacity: 0.7,
            fontSize: '1rem',
            lineHeight: 1,

            '&:hover': {
              opacity: 1,
            },
          }}
        >
          ✕
        </button>
      )}
    </div>
  );
}

// Sử dụng
function Page() {
  return (
    <div css={{ display: 'flex', flexDirection: 'column', gap: '1rem' }}>
      <Alert variant="success" title="Thành công!">
        Dữ liệu đã được lưu thành công.
      </Alert>
      <Alert variant="error" title="Lỗi" onClose={() => {}}>
        Không thể kết nối đến server.
      </Alert>
      <Alert variant="warning">
        Phiên đăng nhập sắp hết hạn.
      </Alert>
    </div>
  );
}
```

---

## 11. SSR Với Next.js App Router

Emotion hỗ trợ SSR tốt hơn Styled Components. Với Next.js App Router:

```tsx
// app/layout.tsx
import { CacheProvider } from '@emotion/react';
import createCache from '@emotion/cache';

// Với Next.js App Router, cần dùng 'use client' cho CacheProvider
// Hoặc dùng @emotion/server cho extraction tốt hơn
```

> **Lưu ý thực tế:** Với Next.js App Router (React Server Components), cả Emotion lẫn Styled Components đều cần `'use client'`. Nếu dùng RSC nhiều, hãy cân nhắc Tailwind CSS hoặc CSS Modules.

---

## 12. Emotion Vs Styled Components — So Sánh Chi Tiết

```
Khi nào chọn Emotion:
✅ Cần cả string và object syntax
✅ Dùng với Material UI (MUI v5+ dùng Emotion)
✅ Cần bundle nhỏ hơn (~7KB vs ~14KB)
✅ Performance là ưu tiên
✅ Dùng css prop để style nhanh mà không tạo component mới

Khi nào chọn Styled Components:
✅ Team đã quen với Styled Components
✅ Cần API đơn giản hơn (chỉ styled)
✅ Dự án legacy đang dùng Styled Components
✅ ThemeProvider API quen thuộc hơn

Khi nào không dùng cả hai:
❌ Dự án Next.js App Router tập trung vào RSC
❌ Cần zero runtime overhead
→ Dùng Tailwind CSS, CSS Modules, hoặc Panda CSS
```

---

## 13. Zero-Runtime Alternatives — Lựa Chọn Không Có Runtime

Nếu cần CSS-in-JS nhưng tương thích RSC, xem xét:

```
Vanilla Extract — CSS typed, zero-runtime, build-time extraction
  import { style } from '@vanilla-extract/css';

Panda CSS — Utility CSS + CSS-in-JS syntax, zero-runtime
  import { css } from '../styled-system/css';

StyleX (Meta) — Facebook's CSS-in-JS, compile-time
  import stylex from '@stylexjs/stylex';
```

---

## 14. Câu Hỏi Phỏng Vấn

**Q: Emotion khác Styled Components ở điểm nào?**
> Emotion nhỏ hơn (~7KB vs ~14KB), nhanh hơn, hỗ trợ `css` prop (style trực tiếp mà không cần tạo styled component), và hỗ trợ object syntax tốt hơn. Styled Components chỉ có `styled` API. MUI v5+ dùng Emotion làm styling engine.

**Q: `css` prop của Emotion là gì?**
> `css` prop cho phép viết style trực tiếp trên JSX element mà không cần tạo styled component riêng. Cần cấu hình `jsxImportSource` hoặc JSX pragma để hoạt động.

**Q: Emotion có hoạt động với RSC không?**
> Không, giống Styled Components, Emotion cần `'use client'` vì phụ thuộc vào React context. Với RSC, dùng CSS Modules, Tailwind, hoặc zero-runtime alternatives như Panda CSS, Vanilla Extract.

**Q: Khi nào chọn object syntax thay vì string syntax trong Emotion?**
> Object syntax tốt hơn khi: làm việc nhiều với TypeScript (type-safe hơn), cần dynamic styles từ JavaScript values (không cần interpolation phức tạp), và performance quan trọng (object không cần parse template literal).

---

## 15. Checklist Thực Hành

```
□ Setup Emotion với Vite và jsxImportSource
□ Tạo component dùng css prop (object syntax)
□ Tạo cùng component đó dùng styled API
□ Setup ThemeProvider với light/dark theme
□ Implement Global styles với Emotion
□ Tạo animation với keyframes
□ So sánh bundle size: Emotion vs Styled Components vs CSS Modules
□ Tích hợp Emotion với Material UI
```

---

## 🔗 Điều Hướng

- **Trước đó:** [3-styled-components.md](./3-styled-components.md) — Styled Components
- **Tiếp theo:** [5-design-systems.md](./5-design-systems.md) — Design Systems
- **README chủ đề:** [README.md](./README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
