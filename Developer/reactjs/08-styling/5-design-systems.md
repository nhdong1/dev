# 5 — Design Systems (Hệ Thống Thiết Kế)

> **Design System** (Hệ Thống Thiết Kế) là tập hợp các quy tắc thiết kế, component UI tái sử dụng, và hướng dẫn nhất quán cho toàn bộ sản phẩm. Trong React ecosystem, các thư viện như **shadcn/ui**, **Radix UI**, và **Material UI (MUI)** cung cấp các component có sẵn, đã xử lý **accessibility** (khả năng tiếp cận), giúp xây dựng UI chuyên nghiệp nhanh hơn nhiều.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Hiểu sự khác biệt giữa **Headless UI** (UI Không Có Style) và **Styled Component Libraries** (Thư Viện Component Có Style)
- [ ] Cài đặt và sử dụng **shadcn/ui** — copy-paste component library với Tailwind
- [ ] Dùng **Radix UI** primitives để xây dựng component accessible từ đầu
- [ ] Integrate và customize **Material UI (MUI)** với theme system
- [ ] Biết khi nào chọn thư viện nào cho loại dự án khác nhau
- [ ] Customize component library theo **brand design** (thiết kế thương hiệu)

---

## 1. Phân Loại Component Libraries

### Headless UI — UI Không Có Style

> Cung cấp **behavior** (hành vi) và **accessibility** (khả năng tiếp cận) — không có style visual. Bạn tự tạo style theo ý muốn.

```
Ưu điểm:
✅ Hoàn toàn tự do về thiết kế visual
✅ Không bị ràng buộc bởi design system nào
✅ Bundle size nhỏ hơn (không có CSS)
✅ Dễ tích hợp với bất kỳ styling solution nào

Nhược điểm:
❌ Phải tự làm toàn bộ CSS
❌ Mất nhiều thời gian hơn để có UI đẹp

Thư viện: Radix UI, Headless UI, Ariakit, React Aria
```

### Styled Component Libraries — Thư Viện Có Style Sẵn

> Cung cấp cả behavior lẫn visual design. Bạn dùng và customize.

```
Ưu điểm:
✅ Nhanh — có UI đẹp ngay từ đầu
✅ Nhất quán — toàn bộ component theo cùng design language
✅ Ít phải tự làm hơn

Nhược điểm:
❌ Khó customize sâu (vượt ra ngoài theme)
❌ Bundle size lớn hơn
❌ Phụ thuộc vào design system của thư viện

Thư viện: MUI, Ant Design, Chakra UI, Mantine
```

### Hybrid — shadcn/ui Model

> Copy source code vào project — không phải npm dependency. Có style sẵn nhưng hoàn toàn kiểm soát.

```
Ưu điểm:
✅ Own your components — không phụ thuộc vào thư viện
✅ Dễ customize vì source code ngay trong project
✅ Tích hợp tốt với Tailwind CSS
✅ Tree-shakeable — chỉ có component bạn cần

Nhược điểm:
❌ Cần copy lại khi có update
❌ Phải quản lý code nhiều hơn

Thư viện: shadcn/ui, ui.shadcn.com
```

---

## 2. shadcn/ui — Copy-Paste Component Library

**shadcn/ui** (đọc: shadow-cn-ui) là tập hợp component được xây dựng trên **Radix UI** (cho behavior + accessibility) và **Tailwind CSS** (cho styling). Thay vì install như npm package, bạn **copy source code** vào project.

### Cài Đặt

```bash
# Với Next.js
npx shadcn@latest init

# Với Vite
npx shadcn@latest init

# Cài component theo nhu cầu — không lấy tất cả
npx shadcn@latest add button
npx shadcn@latest add dialog
npx shadcn@latest add form
npx shadcn@latest add table
npx shadcn@latest add toast
```

Lệnh `add` copy file component vào `src/components/ui/`.

### Cấu Trúc Sau Khi Init

```
src/
├── components/
│   └── ui/              ← Các component shadcn/ui
│       ├── button.tsx   ← Source code của bạn, tùy chỉnh tự do
│       ├── dialog.tsx
│       ├── form.tsx
│       └── ...
├── lib/
│   └── utils.ts         ← Hàm cn() (clsx + tailwind-merge)
└── app/ (hoặc pages/)
```

### Sử Dụng Button Component

```tsx
// Sau khi: npx shadcn@latest add button
import { Button } from "@/components/ui/button"

function Page() {
  return (
    <div className="flex gap-4">
      <Button>Default</Button>
      <Button variant="destructive">Xóa</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="ghost">Ghost</Button>
      <Button variant="link">Link</Button>
      <Button size="sm">Nhỏ</Button>
      <Button size="lg">Lớn</Button>
      <Button disabled>Disabled</Button>
    </div>
  )
}
```

### Customize Component

Vì source code trong project, bạn chỉnh sửa trực tiếp:

```tsx
// src/components/ui/button.tsx — Customize theo brand
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  // Base classes
  "inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
        // ✅ Thêm variant tùy chỉnh theo brand
        brand: "bg-violet-600 text-white hover:bg-violet-700 shadow-md",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
        // ✅ Thêm size tùy chỉnh
        xl: "h-14 px-10 text-base rounded-xl",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)
```

### shadcn/ui Theming

Màu sắc được quản lý qua CSS Variables trong `globals.css`:

```css
/* src/app/globals.css */
@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --border: 214.3 31.8% 91.4%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    /* ... */
  }
}
```

> **Chỉ cần đổi giá trị CSS Variables** để thay đổi toàn bộ màu sắc của hệ thống — không cần chỉnh từng component.

---

## 3. Radix UI — Headless Primitives

**Radix UI** (UI Nguyên Thủy Radix) là nền tảng của shadcn/ui — cung cấp các primitive component với đầy đủ behavior và accessibility, không có style.

### Cài Đặt

```bash
# Cài từng package riêng theo nhu cầu
npm install @radix-ui/react-dialog
npm install @radix-ui/react-dropdown-menu
npm install @radix-ui/react-select
npm install @radix-ui/react-tooltip
npm install @radix-ui/react-accordion
npm install @radix-ui/react-tabs
```

### Ví Dụ: Dialog (Hộp Thoại)

```tsx
import * as Dialog from '@radix-ui/react-dialog';
import styles from './Dialog.module.css';

function CustomDialog({ trigger, title, description, children }) {
  return (
    <Dialog.Root>
      {/* Trigger — phần tử kích hoạt dialog */}
      <Dialog.Trigger asChild>
        {trigger}
      </Dialog.Trigger>

      {/* Portal — render ra ngoài DOM hierarchy */}
      <Dialog.Portal>
        {/* Overlay — lớp phủ mờ */}
        <Dialog.Overlay className={styles.overlay} />

        {/* Content — nội dung dialog */}
        <Dialog.Content className={styles.content}>
          <Dialog.Title className={styles.title}>{title}</Dialog.Title>
          <Dialog.Description className={styles.description}>
            {description}
          </Dialog.Description>

          {children}

          {/* Close button */}
          <Dialog.Close asChild>
            <button className={styles.closeButton} aria-label="Đóng">
              ✕
            </button>
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

```css
/* Dialog.module.css — Bạn tự định nghĩa style */
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  animation: overlayShow 0.15s ease;
}

.content {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: min(90vw, 32rem);
  background: white;
  border-radius: 0.75rem;
  padding: 1.5rem;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: contentShow 0.2s ease;
}

@keyframes overlayShow {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes contentShow {
  from {
    opacity: 0;
    transform: translate(-50%, -48%) scale(0.96);
  }
  to {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
}
```

### Ví Dụ: Dropdown Menu

```tsx
import * as DropdownMenu from '@radix-ui/react-dropdown-menu';

function UserMenu({ user }) {
  return (
    <DropdownMenu.Root>
      <DropdownMenu.Trigger asChild>
        <button className="flex items-center gap-2 rounded-lg px-3 py-2 hover:bg-gray-100">
          <img src={user.avatar} className="w-8 h-8 rounded-full" alt={user.name} />
          <span>{user.name}</span>
        </button>
      </DropdownMenu.Trigger>

      <DropdownMenu.Portal>
        <DropdownMenu.Content
          className="min-w-48 bg-white rounded-lg shadow-lg border border-gray-200 p-1"
          sideOffset={5}
          align="end"
        >
          <DropdownMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm rounded cursor-pointer hover:bg-gray-100 outline-none"
            onSelect={() => console.log('Profile')}
          >
            Trang cá nhân
          </DropdownMenu.Item>

          <DropdownMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm rounded cursor-pointer hover:bg-gray-100 outline-none"
          >
            Cài đặt
          </DropdownMenu.Item>

          <DropdownMenu.Separator className="my-1 h-px bg-gray-200" />

          <DropdownMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm rounded cursor-pointer hover:bg-red-50 text-red-600 outline-none"
            onSelect={() => console.log('Logout')}
          >
            Đăng xuất
          </DropdownMenu.Item>
        </DropdownMenu.Content>
      </DropdownMenu.Portal>
    </DropdownMenu.Root>
  );
}
```

### Radix UI — Tính Năng Accessibility

```
Radix UI xử lý tự động:
→ ARIA attributes — role, aria-expanded, aria-haspopup, aria-modal...
→ Keyboard navigation — Arrow keys, Enter, Escape, Tab
→ Focus management — Focus trap trong modal, restore focus khi đóng
→ Screen reader announcements — Thông báo cho trình đọc màn hình
→ Click outside to close — Đóng khi click ra ngoài
→ Scroll lock — Khóa scroll khi modal mở

Bạn CHỈ cần lo về visual styling — Radix lo toàn bộ behavior.
```

---

## 4. Material UI (MUI) — Bộ Component Đầy Đủ

**Material UI** (MUI) là thư viện component React triển khai **Material Design** (Thiết Kế Vật Liệu) của Google — bộ component đầy đủ nhất, thích hợp cho dashboard và internal tools.

### Cài Đặt

```bash
npm install @mui/material @emotion/react @emotion/styled
npm install @mui/icons-material  # Icon pack tùy chọn
```

### Sử Dụng Cơ Bản

```tsx
import Button from '@mui/material/Button';
import TextField from '@mui/material/TextField';
import Card from '@mui/material/Card';
import CardContent from '@mui/material/CardContent';
import Typography from '@mui/material/Typography';
import Stack from '@mui/material/Stack';
import Chip from '@mui/material/Chip';
import Avatar from '@mui/material/Avatar';

function ProfileCard({ user }) {
  return (
    <Card sx={{ maxWidth: 400, borderRadius: 2 }}>
      <CardContent>
        <Stack direction="row" alignItems="center" spacing={2} mb={2}>
          <Avatar src={user.avatar} sx={{ width: 56, height: 56 }}>
            {user.name[0]}
          </Avatar>
          <div>
            <Typography variant="h6" component="h2">
              {user.name}
            </Typography>
            <Typography variant="body2" color="text.secondary">
              {user.role}
            </Typography>
          </div>
        </Stack>

        <Stack direction="row" spacing={1} flexWrap="wrap" gap={1}>
          {user.skills.map(skill => (
            <Chip key={skill} label={skill} size="small" variant="outlined" />
          ))}
        </Stack>

        <Stack direction="row" spacing={1} mt={2}>
          <Button variant="contained" fullWidth>
            Theo dõi
          </Button>
          <Button variant="outlined" fullWidth>
            Nhắn tin
          </Button>
        </Stack>
      </CardContent>
    </Card>
  );
}
```

### MUI Theme — Custom Theme

```tsx
import { createTheme, ThemeProvider, CssBaseline } from '@mui/material';

// Tạo theme tùy chỉnh
const theme = createTheme({
  palette: {
    // Mode: 'light' | 'dark'
    mode: 'light',

    primary: {
      main: '#6d28d9',      // Tím — màu chính của brand
      light: '#8b5cf6',
      dark: '#5b21b6',
      contrastText: '#ffffff',
    },
    secondary: {
      main: '#06b6d4',      // Cyan
      light: '#22d3ee',
      dark: '#0891b2',
    },
    error: {
      main: '#ef4444',
    },
    background: {
      default: '#f8fafc',
      paper: '#ffffff',
    },
  },

  typography: {
    fontFamily: '"Inter", "Roboto", "Helvetica", "Arial", sans-serif',
    h1: { fontSize: '2.25rem', fontWeight: 700 },
    h2: { fontSize: '1.875rem', fontWeight: 700 },
    button: { textTransform: 'none', fontWeight: 600 }, // Bỏ uppercase mặc định
  },

  shape: {
    borderRadius: 8,  // Border radius toàn cục
  },

  components: {
    // Customize từng component
    MuiButton: {
      styleOverrides: {
        root: {
          borderRadius: '0.5rem',
          padding: '0.5rem 1.25rem',
          boxShadow: 'none',
          '&:hover': {
            boxShadow: 'none',
          },
        },
        contained: {
          '&:hover': {
            boxShadow: '0 4px 12px rgba(109, 40, 217, 0.3)',
          },
        },
      },
      defaultProps: {
        disableElevation: true, // Bỏ elevation shadow mặc định
      },
    },
    MuiCard: {
      styleOverrides: {
        root: {
          boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
          '&:hover': {
            boxShadow: '0 4px 12px rgba(0,0,0,0.1)',
          },
        },
      },
    },
    MuiTextField: {
      defaultProps: {
        variant: 'outlined',
        size: 'small',
      },
    },
  },
});

function App() {
  return (
    <ThemeProvider theme={theme}>
      <CssBaseline />  {/* Reset CSS theo Material Design */}
      <MyApp />
    </ThemeProvider>
  );
}
```

### `sx` Prop — Inline Styling Với Theme Access

```tsx
// sx prop — shorthand cho Emotion styling với theme access
<Box
  sx={{
    display: 'flex',
    gap: 2,           // theme.spacing(2) = 16px
    p: 3,             // padding: theme.spacing(3) = 24px
    bgcolor: 'primary.main',    // theme.palette.primary.main
    color: 'primary.contrastText',
    borderRadius: 2,  // theme.shape.borderRadius * 2 = 16px
    boxShadow: 3,     // theme.shadows[3]

    // Responsive
    flexDirection: { xs: 'column', md: 'row' },

    // Hover
    '&:hover': {
      bgcolor: 'primary.dark',
    },
  }}
>
  Content
</Box>
```

### Data Grid — Lưới Dữ Liệu Nâng Cao

```tsx
import { DataGrid, type GridColDef } from '@mui/x-data-grid';

const columns: GridColDef[] = [
  { field: 'id', headerName: 'ID', width: 70 },
  { field: 'name', headerName: 'Tên', flex: 1 },
  { field: 'email', headerName: 'Email', flex: 1.5 },
  {
    field: 'status',
    headerName: 'Trạng thái',
    width: 120,
    renderCell: (params) => (
      <Chip
        label={params.value}
        color={params.value === 'active' ? 'success' : 'default'}
        size="small"
      />
    ),
  },
  {
    field: 'actions',
    headerName: 'Hành động',
    width: 120,
    renderCell: () => (
      <Button size="small" variant="outlined">Chi tiết</Button>
    ),
  },
];

function UserTable({ users }) {
  return (
    <DataGrid
      rows={users}
      columns={columns}
      pageSizeOptions={[10, 25, 50]}
      initialState={{
        pagination: { paginationModel: { pageSize: 10 } },
      }}
      checkboxSelection
      disableRowSelectionOnClick
      sx={{ border: 'none', borderRadius: 2 }}
    />
  );
}
```

---

## 5. So Sánh Các Thư Viện

| Tiêu Chí | shadcn/ui | Radix UI | MUI |
| -------- | --------- | -------- | --- |
| **Mô hình** | Copy-paste source | Headless primitives | Styled components |
| **Styling** | Tailwind CSS | Bạn tự làm | Emotion/CSS-in-JS |
| **Accessibility** | ✅ (via Radix) | ✅ Xuất sắc | ✅ Tốt |
| **Bundle size** | Chỉ component bạn dùng | Rất nhỏ | ~100KB+ gzip |
| **Customization** | Rất dễ — sửa source | Rất dễ — tự style | Có thể, qua theme |
| **Component số lượng** | ~50+ component | Primitives | 100+ component |
| **Design system** | Neutral, dễ brand | Không có | Material Design |
| **Learning curve** | Thấp | Trung bình | Trung bình |
| **Use case** | Web apps, SaaS | Custom components | Dashboard, admin |
| **RSC support** | ✅ (server-side parts) | ✅ | ⚠️ Cần 'use client' |

---

## 6. Khi Nào Chọn Thư Viện Nào?

```
🎯 shadcn/ui — Chọn khi:
→ Dự án mới với Next.js App Router + Tailwind
→ Cần UI đẹp nhanh nhưng muốn kiểm soát hoàn toàn style
→ Team đã biết Tailwind
→ Cần custom sâu để phù hợp brand

🎯 Radix UI — Chọn khi:
→ Xây dựng design system riêng từ đầu
→ Cần accessibility đảm bảo, nhưng style hoàn toàn theo ý
→ Làm component library cho nhiều dự án
→ Dùng CSS Modules hoặc CSS-in-JS thay vì Tailwind

🎯 MUI — Chọn khi:
→ Dashboard, admin panel, internal tools
→ Cần nhiều component phức tạp (DataGrid, DatePicker, Charts)
→ Team quen Material Design
→ Tốc độ ra mắt sản phẩm là ưu tiên, không cần highly custom

🎯 Không dùng component library — Khi nào:
→ Landing page, marketing site — cần pixel-perfect design
→ Dự án cực nhỏ — overhead của thư viện không đáng
→ Đã có design system riêng chặt chẽ
```

---

## 7. Cva — Class Variance Authority

**CVA** — **Class Variance Authority** (Cơ Quan Quản Lý Class Biến Thể) — là thư viện giúp tạo component với variants, được shadcn/ui sử dụng:

```bash
npm install class-variance-authority
```

```tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

// Định nghĩa variants với type-safety
const buttonVariants = cva(
  // Base classes — luôn có
  'inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-blue-600 text-white hover:bg-blue-700',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
        danger: 'bg-red-600 text-white hover:bg-red-700',
        outline: 'border border-gray-300 hover:bg-gray-100',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4 text-sm',
        lg: 'h-12 px-6 text-base',
      },
      // Compound variants — kết hợp điều kiện
    },
    compoundVariants: [
      // Khi variant=danger VÀ size=lg → thêm class đặc biệt
      {
        variant: 'danger',
        size: 'lg',
        class: 'uppercase tracking-wide',
      },
    ],
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

// TypeScript tự động infer props từ variants
interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

function Button({ variant, size, className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size }), className)}
      {...props}
    />
  );
}

// Sử dụng — có auto-complete cho variant và size
<Button variant="danger" size="lg">Xóa</Button>
```

---

## 8. Ant Design — Thư Viện Phổ Biến Cho Enterprise

**Ant Design** (AntD) là thư viện component lớn từ Alibaba, phổ biến trong môi trường enterprise và thị trường châu Á:

```bash
npm install antd
```

```tsx
import { Button, Table, Form, Input, Select, DatePicker, Modal } from 'antd';
import { ConfigProvider, theme } from 'antd';

// Customize theme
function App() {
  return (
    <ConfigProvider
      theme={{
        algorithm: theme.defaultAlgorithm, // hoặc theme.darkAlgorithm
        token: {
          colorPrimary: '#6d28d9',
          borderRadius: 8,
          fontFamily: 'Inter, sans-serif',
        },
      }}
    >
      <MyApp />
    </ConfigProvider>
  );
}

// Sử dụng component
function UserForm() {
  const [form] = Form.useForm();

  return (
    <Form form={form} layout="vertical" onFinish={values => console.log(values)}>
      <Form.Item name="name" label="Họ tên" rules={[{ required: true }]}>
        <Input placeholder="Nhập họ tên" />
      </Form.Item>

      <Form.Item name="role" label="Vai trò">
        <Select>
          <Select.Option value="admin">Quản trị viên</Select.Option>
          <Select.Option value="user">Người dùng</Select.Option>
        </Select>
      </Form.Item>

      <Button type="primary" htmlType="submit">
        Lưu
      </Button>
    </Form>
  );
}
```

---

## 9. Accessibility (A11y) — Khả Năng Tiếp Cận

Một trong những lý do chính để dùng component libraries là **accessibility** tích hợp sẵn:

```
Các tiêu chuẩn:
→ WCAG 2.1 — Web Content Accessibility Guidelines — Hướng Dẫn Tiếp Cận Nội Dung Web
→ ARIA — Accessible Rich Internet Applications — Ứng Dụng Internet Phong Phú Tiếp Cận Được
→ Screen reader support — Hỗ trợ đọc màn hình (NVDA, JAWS, VoiceOver)
→ Keyboard navigation — Điều hướng bằng bàn phím

Radix UI cung cấp:
→ Focus management — quản lý focus đúng chuẩn
→ ARIA roles và attributes — tự động thêm đúng ARIA
→ Keyboard shortcuts — phím tắt theo chuẩn WAI-ARIA
→ Reduced motion — tôn trọng prefers-reduced-motion của OS
```

```tsx
// Radix Tooltip — tự động xử lý accessibility
import * as Tooltip from '@radix-ui/react-tooltip';

function InfoButton({ content }) {
  return (
    <Tooltip.Provider delayDuration={200}>
      <Tooltip.Root>
        <Tooltip.Trigger asChild>
          <button aria-label="Thông tin thêm">
            ℹ️
          </button>
        </Tooltip.Trigger>
        <Tooltip.Portal>
          <Tooltip.Content
            className="bg-gray-900 text-white text-sm px-3 py-1.5 rounded"
            sideOffset={5}
          >
            {content}
            <Tooltip.Arrow className="fill-gray-900" />
          </Tooltip.Content>
        </Tooltip.Portal>
      </Tooltip.Root>
    </Tooltip.Provider>
  );
}
```

---

## 10. Câu Hỏi Phỏng Vấn

**Q: shadcn/ui khác gì so với một npm UI package thông thường như MUI?**
> shadcn/ui không phải npm package — bạn copy source code vào project. Điều này cho phép tùy chỉnh hoàn toàn mà không bị ràng buộc bởi API của thư viện, không bị breaking changes, và không thêm bundle dependency.

**Q: Radix UI là gì và tại sao nó quan trọng?**
> Radix UI cung cấp headless (không có style) UI primitives với behavior và accessibility hoàn chỉnh. Nó quan trọng vì xây dựng component accessible từ đầu rất khó — Radix đã giải quyết phần khó đó, bạn chỉ cần lo về visual design.

**Q: Khi nào nên dùng MUI thay vì shadcn/ui?**
> MUI phù hợp khi: cần nhiều component phức tạp ngay lập tức (DataGrid, DatePicker...), theo Material Design, xây dựng dashboard/admin panel. shadcn/ui tốt hơn khi cần custom design, dùng Tailwind, muốn kiểm soát hoàn toàn component.

**Q: CVA — Class Variance Authority là gì?**
> CVA là thư viện giúp tạo component với typed variants. Thay vì if-else để xử lý variants, CVA cho phép khai báo declarative và TypeScript tự infer types của props từ variants đã định nghĩa.

---

## 11. Checklist Thực Hành

```
□ Setup shadcn/ui với Next.js App Router
□ Thêm Button, Dialog, Form, Table từ shadcn/ui
□ Customize shadcn/ui theme qua CSS Variables
□ Build Dialog component với Radix UI từ đầu (không dùng shadcn)
□ Setup MUI với custom theme (đổi primary color, border radius, font)
□ Tạo form với MUI Form components (TextField, Select, DatePicker)
□ Implement accessible Dropdown Menu với Radix DropdownMenu
□ So sánh bundle size: shadcn/ui vs MUI cho cùng một UI
□ Tạo Button component với CVA (variants + sizes + compound variants)
```

---

## 🔗 Điều Hướng

- **Trước đó:** [4-emotion.md](./4-emotion.md) — Emotion CSS-in-JS
- **Tiếp theo:** [09-ecosystem/](../09-ecosystem/) — Hệ sinh thái React
- **README chủ đề:** [README.md](./README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
