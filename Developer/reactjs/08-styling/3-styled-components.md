# 3 — Styled Components (CSS-in-JS Kinh Điển)

> **Styled Components** là thư viện **CSS-in-JS** (CSS Trong JavaScript) phổ biến nhất, cho phép viết CSS thực sự trong JavaScript bằng **tagged template literals** (chuỗi mẫu có nhãn). Style được gắn chặt với component, hỗ trợ dynamic styles, theming mạnh mẽ.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Tạo styled component cơ bản với **tagged template literals**
- [ ] Viết **dynamic styles** (style động) dựa vào props
- [ ] Kế thừa và mở rộng style với `styled(Component)`
- [ ] Implement **theming** (chủ đề) với `ThemeProvider`
- [ ] Xử lý **global styles** với `createGlobalStyle`
- [ ] Dùng `css` helper và `keyframes` cho animations
- [ ] Hiểu cơ chế **class injection** (tiêm class) và SSR

---

## 1. Cài Đặt

```bash
npm install styled-components

# TypeScript types
npm install -D @types/styled-components
```

---

## 2. Cú Pháp Cơ Bản

### Tagged Template Literals — Chuỗi Mẫu Có Nhãn

```tsx
import styled from 'styled-components';

// Tạo component Button được styled
const Button = styled.button`
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 0.375rem;
  padding: 0.5rem 1rem;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s;

  /* Pseudo-classes hoạt động bình thường */
  &:hover {
    background: #2563eb;
  }

  &:focus {
    outline: none;
    box-shadow: 0 0 0 2px #93c5fd;
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;

// Sử dụng như component React bình thường
function App() {
  return (
    <Button onClick={() => alert('Clicked!')}>
      Click me
    </Button>
  );
}
```

### Styled Components Cho Các HTML Elements

```tsx
const Heading = styled.h1`
  font-size: 2rem;
  font-weight: 700;
  color: #111827;
  margin-bottom: 1rem;
`;

const Paragraph = styled.p`
  font-size: 1rem;
  line-height: 1.6;
  color: #4b5563;
`;

const Card = styled.div`
  border: 1px solid #e5e7eb;
  border-radius: 0.75rem;
  padding: 1.5rem;
  background: white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
`;

const Input = styled.input`
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  font-size: 1rem;
  outline: none;

  &:focus {
    border-color: #3b82f6;
    box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.3);
  }
`;
```

---

## 3. Dynamic Styles — Style Động Theo Props

Đây là tính năng mạnh nhất của Styled Components — style thay đổi theo props:

### Cơ Bản

```tsx
interface ButtonProps {
  primary?: boolean;
}

const Button = styled.button<ButtonProps>`
  background: ${props => props.primary ? '#3b82f6' : '#f3f4f6'};
  color: ${props => props.primary ? 'white' : '#374151'};
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  cursor: pointer;
`;

// Sử dụng
<Button>Secondary</Button>
<Button primary>Primary</Button>
```

### Với Nhiều Variants

```tsx
import styled, { css } from 'styled-components';

type Variant = 'primary' | 'secondary' | 'danger' | 'ghost';
type Size = 'sm' | 'md' | 'lg';

interface ButtonProps {
  variant?: Variant;
  size?: Size;
  fullWidth?: boolean;
}

// css helper — tạo CSS snippet có thể tái sử dụng
const variantStyles = {
  primary: css`
    background: #3b82f6;
    color: white;
    &:hover { background: #2563eb; }
  `,
  secondary: css`
    background: #f3f4f6;
    color: #374151;
    &:hover { background: #e5e7eb; }
  `,
  danger: css`
    background: #ef4444;
    color: white;
    &:hover { background: #dc2626; }
  `,
  ghost: css`
    background: transparent;
    color: #374151;
    &:hover { background: #f3f4f6; }
  `,
};

const sizeStyles = {
  sm: css`
    padding: 0.25rem 0.75rem;
    font-size: 0.875rem;
    height: 2rem;
  `,
  md: css`
    padding: 0.5rem 1rem;
    font-size: 0.875rem;
    height: 2.5rem;
  `,
  lg: css`
    padding: 0.75rem 1.5rem;
    font-size: 1rem;
    height: 2.75rem;
  `,
};

const Button = styled.button<ButtonProps>`
  /* Base styles */
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: none;
  border-radius: 0.375rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;

  /* Dynamic width */
  width: ${props => props.fullWidth ? '100%' : 'auto'};

  /* Apply variant styles */
  ${props => variantStyles[props.variant ?? 'primary']}

  /* Apply size styles */
  ${props => sizeStyles[props.size ?? 'md']}

  /* Disabled state */
  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
    pointer-events: none;
  }
`;

// Sử dụng
<Button variant="primary" size="lg">Lớn</Button>
<Button variant="danger" fullWidth>Xóa tất cả</Button>
```

---

## 4. Kế Thừa Styled Component

### Mở Rộng Style Hiện Có

```tsx
// Component gốc
const Button = styled.button`
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
`;

// Kế thừa và thêm style mới
const PrimaryButton = styled(Button)`
  background: #3b82f6;
  color: white;

  &:hover {
    background: #2563eb;
  }
`;

// Kế thừa từ PrimaryButton
const LargePrimaryButton = styled(PrimaryButton)`
  padding: 0.75rem 2rem;
  font-size: 1.125rem;
`;
```

### Thay Đổi HTML Tag — `as` Prop

```tsx
const Button = styled.button`
  background: #3b82f6;
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  display: inline-block;
`;

// Render như <a> thay vì <button> — style giữ nguyên
<Button as="a" href="/login">
  Đăng nhập
</Button>

// Render như React Router Link
import { Link } from 'react-router-dom';
<Button as={Link} to="/dashboard">
  Dashboard
</Button>
```

---

## 5. Theming Với ThemeProvider

**ThemeProvider** (Nhà Cung Cấp Chủ Đề) là cách Styled Components xử lý theming:

### Định Nghĩa Theme

```typescript
// theme.ts
export const lightTheme = {
  colors: {
    primary: '#3b82f6',
    primaryHover: '#2563eb',
    background: '#ffffff',
    surface: '#f9fafb',
    text: '#111827',
    textMuted: '#6b7280',
    border: '#e5e7eb',
    danger: '#ef4444',
    success: '#22c55e',
  },
  spacing: {
    xs: '0.25rem',
    sm: '0.5rem',
    md: '1rem',
    lg: '1.5rem',
    xl: '2rem',
  },
  borderRadius: {
    sm: '0.25rem',
    md: '0.375rem',
    lg: '0.5rem',
    full: '9999px',
  },
  shadows: {
    sm: '0 1px 2px rgba(0, 0, 0, 0.05)',
    md: '0 4px 6px rgba(0, 0, 0, 0.1)',
    lg: '0 10px 15px rgba(0, 0, 0, 0.1)',
  },
};

export const darkTheme: typeof lightTheme = {
  colors: {
    primary: '#60a5fa',
    primaryHover: '#93c5fd',
    background: '#111827',
    surface: '#1f2937',
    text: '#f9fafb',
    textMuted: '#9ca3af',
    border: '#374151',
    danger: '#f87171',
    success: '#4ade80',
  },
  spacing: lightTheme.spacing,
  borderRadius: lightTheme.borderRadius,
  shadows: lightTheme.shadows,
};

// TypeScript — declare theme type cho auto-complete
declare module 'styled-components' {
  export interface DefaultTheme extends typeof lightTheme {}
}
```

### Sử Dụng ThemeProvider

```tsx
// App.tsx
import { ThemeProvider } from 'styled-components';
import { useState } from 'react';
import { lightTheme, darkTheme } from './theme';

function App() {
  const [isDark, setIsDark] = useState(false);

  return (
    <ThemeProvider theme={isDark ? darkTheme : lightTheme}>
      <GlobalStyle />
      <Layout>
        <button onClick={() => setIsDark(prev => !prev)}>
          Toggle Theme
        </button>
        {/* Tất cả children đều có thể access theme */}
      </Layout>
    </ThemeProvider>
  );
}
```

```tsx
// Button sử dụng theme — auto-complete với TypeScript
const Button = styled.button`
  background: ${props => props.theme.colors.primary};
  color: white;
  padding: ${props => `${props.theme.spacing.sm} ${props.theme.spacing.md}`};
  border-radius: ${props => props.theme.borderRadius.md};
  box-shadow: ${props => props.theme.shadows.sm};

  &:hover {
    background: ${props => props.theme.colors.primaryHover};
  }
`;

const Card = styled.div`
  background: ${props => props.theme.colors.surface};
  border: 1px solid ${props => props.theme.colors.border};
  border-radius: ${props => props.theme.borderRadius.lg};
  padding: ${props => props.theme.spacing.lg};
  box-shadow: ${props => props.theme.shadows.md};
`;
```

---

## 6. Global Styles — Style Toàn Cục

```tsx
import { createGlobalStyle } from 'styled-components';

const GlobalStyle = createGlobalStyle`
  /* CSS Reset và base styles */
  *, *::before, *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    background: ${props => props.theme.colors.background};
    color: ${props => props.theme.colors.text};
    line-height: 1.6;
    transition: background 0.2s, color 0.2s;
  }

  a {
    color: ${props => props.theme.colors.primary};
    text-decoration: none;

    &:hover {
      text-decoration: underline;
    }
  }

  img, video {
    max-width: 100%;
    height: auto;
  }
`;

// Render một lần ở root component
function App() {
  return (
    <ThemeProvider theme={lightTheme}>
      <GlobalStyle />  {/* Inject global CSS */}
      <MyApp />
    </ThemeProvider>
  );
}
```

---

## 7. Animations — Hoạt Ảnh

```tsx
import styled, { keyframes } from 'styled-components';

// Định nghĩa keyframe animation
const fadeIn = keyframes`
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
`;

const spin = keyframes`
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
`;

const shimmer = keyframes`
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
`;

// Dùng trong styled component
const FadeInDiv = styled.div`
  animation: ${fadeIn} 0.3s ease-out;
`;

const Spinner = styled.div`
  width: 2rem;
  height: 2rem;
  border: 2px solid #e5e7eb;
  border-top-color: #3b82f6;
  border-radius: 50%;
  animation: ${spin} 0.8s linear infinite;
`;

const SkeletonBox = styled.div`
  height: 1rem;
  border-radius: 0.25rem;
  background: linear-gradient(
    90deg,
    #f3f4f6 25%,
    #e5e7eb 50%,
    #f3f4f6 75%
  );
  background-size: 200% 100%;
  animation: ${shimmer} 1.5s infinite;
`;
```

---

## 8. useTheme Hook — Dùng Theme Trong Logic

```tsx
import { useTheme } from 'styled-components';

function ChartComponent() {
  const theme = useTheme(); // Lấy theme trong function component

  const chartConfig = {
    colors: [theme.colors.primary, theme.colors.success, theme.colors.danger],
    backgroundColor: theme.colors.background,
  };

  return <Chart config={chartConfig} />;
}
```

---

## 9. Ví Dụ Thực Tế: Modal Component

```tsx
import styled, { keyframes, css } from 'styled-components';

const fadeIn = keyframes`
  from { opacity: 0; }
  to { opacity: 1; }
`;

const slideUp = keyframes`
  from {
    opacity: 0;
    transform: translateY(20px) scale(0.95);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
`;

const Overlay = styled.div<{ isOpen: boolean }>`
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 50;
  padding: 1rem;

  ${props => props.isOpen && css`
    animation: ${fadeIn} 0.2s ease-out;
  `}
`;

const ModalContent = styled.div<{ size?: 'sm' | 'md' | 'lg' }>`
  background: ${props => props.theme.colors.background};
  border-radius: ${props => props.theme.borderRadius.lg};
  box-shadow: ${props => props.theme.shadows.lg};
  width: 100%;
  animation: ${slideUp} 0.25s ease-out;

  max-width: ${props => {
    switch (props.size) {
      case 'sm': return '28rem';
      case 'lg': return '56rem';
      default: return '40rem';
    }
  }};
`;

const ModalHeader = styled.div`
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: ${props => props.theme.spacing.lg};
  border-bottom: 1px solid ${props => props.theme.colors.border};
`;

const ModalTitle = styled.h2`
  font-size: 1.25rem;
  font-weight: 600;
  color: ${props => props.theme.colors.text};
  margin: 0;
`;

const ModalBody = styled.div`
  padding: ${props => props.theme.spacing.lg};
`;

const ModalFooter = styled.div`
  display: flex;
  gap: ${props => props.theme.spacing.sm};
  justify-content: flex-end;
  padding: ${props => props.theme.spacing.lg};
  border-top: 1px solid ${props => props.theme.colors.border};
`;

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
  size?: 'sm' | 'md' | 'lg';
  footer?: React.ReactNode;
}

function Modal({ isOpen, onClose, title, children, size = 'md', footer }: ModalProps) {
  if (!isOpen) return null;

  return (
    <Overlay isOpen={isOpen} onClick={onClose}>
      <ModalContent size={size} onClick={e => e.stopPropagation()}>
        <ModalHeader>
          <ModalTitle>{title}</ModalTitle>
          <button onClick={onClose}>✕</button>
        </ModalHeader>
        <ModalBody>{children}</ModalBody>
        {footer && <ModalFooter>{footer}</ModalFooter>}
      </ModalContent>
    </Overlay>
  );
}
```

---

## 10. TypeScript Tips

```tsx
// Khai báo props không được forward xuống DOM
// Dùng transient props — prefix $ để không truyền xuống HTML element
interface ButtonProps {
  $variant?: 'primary' | 'secondary'; // $ prefix = transient prop
  $size?: 'sm' | 'md';
}

const Button = styled.button<ButtonProps>`
  background: ${props => props.$variant === 'primary' ? 'blue' : 'gray'};
  padding: ${props => props.$size === 'sm' ? '0.25rem 0.5rem' : '0.5rem 1rem'};
`;

// ✅ $variant và $size KHÔNG bị truyền xuống <button> HTML element
<Button $variant="primary" $size="sm">Click</Button>
```

---

## 11. Styled Components Với React Server Components

> **Quan trọng:** Styled Components dùng React context và hooks bên trong — **không hoạt động với React Server Components**.

```tsx
// ❌ Không thể dùng trong Server Component
// page.tsx (Server Component mặc định trong Next.js App Router)
import { StyledButton } from './Button'; // Lỗi!

// ✅ Giải pháp: Thêm 'use client' directive
// Button.tsx
'use client';
import styled from 'styled-components';

export const StyledButton = styled.button`...`;
```

```tsx
// ✅ Wrapper pattern — dùng styled-components trong Client Component
// ClientButton.tsx
'use client';
import styled from 'styled-components';

const StyledButton = styled.button`
  background: blue;
  color: white;
`;

export function ClientButton({ children, onClick }) {
  return <StyledButton onClick={onClick}>{children}</StyledButton>;
}

// ServerPage.tsx (Server Component)
import { ClientButton } from './ClientButton';

export default function Page() {
  return <ClientButton>Click</ClientButton>; // ✅ Hoạt động
}
```

---

## 12. SSR — Server-Side Rendering

Với Next.js Pages Router hoặc SSR frameworks, cần setup `ServerStyleSheet` (Bảng Style Server):

```tsx
// pages/_document.tsx (Next.js Pages Router)
import Document, { Html, Head, Main, NextScript, DocumentContext } from 'next/document';
import { ServerStyleSheet } from 'styled-components';

class MyDocument extends Document {
  static async getInitialProps(ctx: DocumentContext) {
    const sheet = new ServerStyleSheet();

    const originalRenderPage = ctx.renderPage;
    ctx.renderPage = () =>
      originalRenderPage({
        enhanceApp: App => props => sheet.collectStyles(<App {...props} />),
      });

    const initialProps = await Document.getInitialProps(ctx);
    return {
      ...initialProps,
      styles: [initialProps.styles, sheet.getStyleElement()],
    };
  }

  render() {
    return (
      <Html>
        <Head />
        <body>
          <Main />
          <NextScript />
        </body>
      </Html>
    );
  }
}
```

---

## 13. Ưu & Nhược Điểm

### ✅ Ưu Điểm

```
1. Co-location (Đặt cùng chỗ) — CSS sống cùng với component logic
2. Dynamic styles cực dễ — props → style, không cần CSS class manipulation
3. Theming mạnh — ThemeProvider truyền theme qua toàn bộ component tree
4. Tự động unique class names — không lo xung đột
5. CSS đầy đủ — pseudo-class, media queries, keyframes... đều dùng được
6. TypeScript support tốt — props type-safe, theme type-safe
7. Component composition tốt — styled(Component) để extend
```

### ❌ Nhược Điểm

```
1. Runtime overhead — JavaScript generate và inject CSS khi render
   → Làm chậm Time to Interactive, đặc biệt trên thiết bị chậm

2. KHÔNG tương thích với React Server Components
   → Cần 'use client' cho tất cả styled components

3. Bundle size lớn hơn — thư viện ~14KB gzip

4. SSR phức tạp — cần setup ServerStyleSheet

5. Styled Components v6 thay đổi nhiều API — breaking changes

6. Trend đang giảm — cộng đồng dịch chuyển sang Tailwind + shadcn/ui
```

---

## 14. Câu Hỏi Phỏng Vấn

**Q: CSS-in-JS là gì? Tại sao Styled Components phổ biến?**
> CSS-in-JS là kỹ thuật viết CSS bên trong JavaScript file thay vì file `.css` riêng. Styled Components phổ biến vì: co-location tốt, dynamic styles tự nhiên với props, theming mạnh với ThemeProvider.

**Q: `css` helper trong Styled Components dùng để làm gì?**
> `css` helper tạo CSS snippet có thể tái sử dụng trong nhiều styled components, và quan trọng là nó hỗ trợ interpolation (nội suy) với props — nếu không dùng `css`, TypeScript sẽ mất type-safety trong nested interpolations.

**Q: Transient props là gì và tại sao cần dùng?**
> Transient props (có prefix `$`) là props chỉ dùng để style, không được forward xuống DOM element. Nếu không dùng `$`, React sẽ warning vì các prop không hợp lệ (như `variant`, `size`) bị truyền xuống HTML element.

**Q: Styled Components có hoạt động với Next.js App Router không?**
> Không hoạt động với Server Components — cần thêm `'use client'` directive. Trong dự án mới với App Router, nên cân nhắc Tailwind CSS hoặc zero-runtime alternatives.

---

## 15. Checklist Thực Hành

```
□ Tạo Button với 4 variants dùng css helper
□ Setup ThemeProvider với light/dark theme
□ Implement theme toggle với useState
□ Tạo Modal component với animation
□ Dùng styled(Component) để extend
□ Thử transient props để fix DOM prop warning
□ Setup SSR với Next.js Pages Router
□ Benchmark: so sánh performance với CSS Modules
```

---

## 🔗 Điều Hướng

- **Trước đó:** [2-tailwind-css.md](./2-tailwind-css.md) — Tailwind CSS
- **Tiếp theo:** [4-emotion.md](./4-emotion.md) — Emotion CSS-in-JS
- **README chủ đề:** [README.md](./README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
