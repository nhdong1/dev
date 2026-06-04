# Storybook, Turborepo, Nx — Dev Tools & Monorepo

> Storybook giúp phát triển UI components độc lập. Turborepo và Nx là các công cụ quản lý Monorepo (Kho Lưu Trữ Đơn — một repo chứa nhiều packages/apps). Bộ ba này tạo nên workflow phát triển chuyên nghiệp cho team lớn

---

## Phần 1: Storybook — Phát Triển UI Độc Lập

### Storybook Là Gì?

Storybook là môi trường phát triển component cô lập — bạn xây dựng và kiểm tra từng component một mà **không cần chạy cả ứng dụng**.

```
Không có Storybook:
  Muốn test UI component Button với state "loading"
  → Phải: Start app → Navigate đến trang cụ thể → Trigger loading state
  → Mất 2-5 phút, phụ thuộc backend

Với Storybook:
  Mở Storybook → Chọn Button → Chọn story "Loading"
  → Thấy ngay kết quả
  → Mất 5 giây, không cần backend
```

### Cài Đặt Storybook

```bash
# Cài tự động vào project React có sẵn
npx storybook@latest init

# Storybook tự detect framework (React, Vue, Angular, ...)
# và cấu hình phù hợp

# Chạy Storybook dev server
npm run storybook   # Mở tại http://localhost:6006

# Build Storybook thành static site (để deploy)
npm run build-storybook
```

### Viết Stories — Kịch Bản Component

Story là một trạng thái cụ thể của component. Mỗi component có thể có nhiều stories.

```tsx
// src/components/Button/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

// Meta — Metadata của component
const meta: Meta<typeof Button> = {
  title: 'UI/Button',      // Đường dẫn trong sidebar Storybook
  component: Button,
  parameters: {
    layout: 'centered',    // Căn giữa component trong canvas
  },
  tags: ['autodocs'],      // Tự động tạo documentation page
  argTypes: {
    variant: {
      control: 'select',   // UI control để thay đổi prop
      options: ['primary', 'secondary', 'danger'],
      description: 'Kiểu hiển thị của button',
    },
    onClick: { action: 'clicked' }, // Log action trong Storybook
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

// Mỗi named export là một story
export const Primary: Story = {
  args: {
    label: 'Button Chính',
    variant: 'primary',
  },
};

export const Secondary: Story = {
  args: {
    label: 'Button Phụ',
    variant: 'secondary',
  },
};

export const Loading: Story = {
  args: {
    label: 'Đang xử lý...',
    loading: true,
    disabled: true,
  },
};

export const WithIcon: Story = {
  args: {
    label: 'Lưu',
  },
  render: (args) => (
    <Button {...args}>
      <SaveIcon /> {args.label}
    </Button>
  ),
};
```

### Args, Controls & Actions

```tsx
// Component với nhiều variants — Storybook tạo controls tự động
interface AlertProps {
  message: string;
  type: 'info' | 'success' | 'warning' | 'error';
  dismissible?: boolean;
  onDismiss?: () => void;
}

// src/components/Alert/Alert.stories.tsx
const meta: Meta<typeof Alert> = {
  component: Alert,
  argTypes: {
    type: {
      control: { type: 'radio' },
      options: ['info', 'success', 'warning', 'error'],
    },
    onDismiss: { action: 'dismissed' }, // Log khi gọi
  },
};

export const AllVariants: Story = {
  render: () => (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
      <Alert message="Thông tin" type="info" />
      <Alert message="Thành công!" type="success" />
      <Alert message="Cảnh báo" type="warning" />
      <Alert message="Lỗi xảy ra" type="error" />
    </div>
  ),
};
```

### Decorators — Bọc Stories

```tsx
// .storybook/preview.tsx — Cấu hình global
import { Provider } from 'react-redux';
import { MemoryRouter } from 'react-router-dom';
import { store } from '../src/store';
import '../src/styles/globals.css';

export const decorators = [
  // Bọc tất cả stories với Redux Provider và Router
  (Story) => (
    <Provider store={store}>
      <MemoryRouter>
        <Story />
      </MemoryRouter>
    </Provider>
  ),
];

export const parameters = {
  actions: { argTypesRegex: '^on[A-Z].*' }, // Auto-detect event handlers
  backgrounds: {
    default: 'light',
    values: [
      { name: 'light', value: '#ffffff' },
      { name: 'dark', value: '#1a1a1a' },
    ],
  },
};
```

### Storybook Addons Quan Trọng

| Addon | Công Dụng |
| ----- | --------- |
| `@storybook/addon-controls` | Controls panel để thay đổi props interactively |
| `@storybook/addon-actions` | Log event handler calls |
| `@storybook/addon-docs` | Auto-generate documentation từ JSDoc + stories |
| `@storybook/addon-a11y` | Kiểm tra accessibility — khả năng tiếp cận |
| `@storybook/addon-viewport` | Test responsive design ở nhiều màn hình |
| `storybook-dark-mode` | Toggle dark/light mode |
| `@chromatic-com/storybook` | Visual regression testing — Kiểm thử trực quan |

### Storybook Testing — Kiểm Thử Với Storybook

```tsx
// Dùng stories trong unit tests — tái sử dụng setup
import { render, screen } from '@testing-library/react';
import { composeStories } from '@storybook/react';
import * as stories from './Button.stories';

const { Primary, Loading } = composeStories(stories);

test('Button hiển thị label', () => {
  render(<Primary />);
  expect(screen.getByText('Button Chính')).toBeInTheDocument();
});

test('Button disabled khi loading', () => {
  render(<Loading />);
  expect(screen.getByRole('button')).toBeDisabled();
});
```

---

## Phần 2: Monorepo — Kho Lưu Trữ Đơn

### Monorepo Là Gì?

```
Polyrepo (Kho Đa) — Truyền thống:
  github.com/company/frontend-app     (repo riêng)
  github.com/company/mobile-app       (repo riêng)
  github.com/company/design-system    (repo riêng)
  github.com/company/shared-utils     (repo riêng)

  Vấn đề:
  - Chia sẻ code phải publish npm package
  - Version mismatch giữa các repo
  - Duplicate CI/CD config
  - Khó refactor xuyên suốt

Monorepo (Kho Đơn) — Hiện đại:
  github.com/company/platform         (một repo)
  ├── apps/
  │   ├── web/          (Next.js app)
  │   └── mobile/       (React Native)
  └── packages/
      ├── design-system/
      ├── shared-utils/
      └── typescript-config/

  Lợi ích:
  - Chia sẻ code trực tiếp, không cần publish
  - Atomic commits (một commit thay đổi nhiều packages)
  - Consistent tooling (ESLint, TypeScript, testing)
  - Dễ refactor xuyên suốt
```

### npm/pnpm Workspaces — Nền Tảng Monorepo

```json
// package.json (root)
{
  "name": "my-platform",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ]
}
```

```
Cấu trúc monorepo cơ bản:
my-platform/
├── package.json          ← Root workspace config
├── turbo.json            ← Turborepo config
├── apps/
│   ├── web/
│   │   └── package.json  ← { "name": "@platform/web" }
│   └── docs/
│       └── package.json  ← { "name": "@platform/docs" }
└── packages/
    ├── ui/
    │   └── package.json  ← { "name": "@platform/ui" }
    ├── utils/
    │   └── package.json  ← { "name": "@platform/utils" }
    └── tsconfig/
        └── package.json  ← { "name": "@platform/tsconfig" }
```

---

## Phần 3: Turborepo — Task Runner Thông Minh

### Turborepo Là Gì?

Turborepo là **task runner** (Trình Chạy Tác Vụ) cho monorepo — quản lý việc build, test, lint với **caching thông minh** để không làm lại việc đã làm.

```bash
# Cài đặt
npm install -D turbo

# Hoặc tạo monorepo mới với Turborepo
npx create-turbo@latest
```

### `turbo.json` — Cấu Hình Pipeline

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],  // ^ = Phải build dependencies trước
      "outputs": [".next/**", "dist/**"]  // Cache các thư mục này
    },
    "test": {
      "dependsOn": ["build"],  // Test sau khi build
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []  // Không cache output (chỉ quan tâm exit code)
    },
    "dev": {
      "cache": false,  // Không cache dev servers
      "persistent": true  // Task chạy mãi (long-running)
    }
  }
}
```

### Chạy Tasks Với Turborepo

```bash
# Build tất cả apps và packages (song song, thứ tự đúng)
turbo build

# Chỉ build app web
turbo build --filter=@platform/web

# Build web và tất cả dependencies của nó
turbo build --filter=@platform/web...

# Dev tất cả apps
turbo dev

# Xem graph phụ thuộc
turbo build --graph
```

### Remote Caching — Cache Từ Xa

Turborepo có thể cache build artifacts lên cloud — các developers trong team và CI không cần rebuild những gì đã được build.

```bash
# Đăng nhập vào Vercel (provider cache mặc định)
npx turbo login
npx turbo link   # Link repo với Vercel Remote Cache

# CI pipeline tự động dùng remote cache
# → Lần đầu build: 5 phút
# → Các lần sau (nếu không có thay đổi): < 10 giây!
```

### Ví Dụ Thực Tế — Monorepo Với Turborepo

```
platform/
├── turbo.json
├── package.json
│
├── apps/
│   ├── web/                     (@platform/web — Next.js)
│   │   ├── package.json
│   │   └── src/
│   │
│   └── storybook/               (@platform/storybook)
│       └── package.json
│
└── packages/
    ├── ui/                      (@platform/ui — Design System)
    │   ├── package.json
    │   ├── src/
    │   │   ├── Button/
    │   │   │   ├── Button.tsx
    │   │   │   └── Button.stories.tsx
    │   │   └── index.ts         ← Export tất cả
    │   └── tsconfig.json
    │
    ├── utils/                   (@platform/utils)
    │   └── src/
    │       └── format.ts
    │
    └── tsconfig/                (@platform/tsconfig)
        ├── base.json
        ├── react.json
        └── next.json
```

```json
// packages/ui/package.json
{
  "name": "@platform/ui",
  "version": "0.0.0",
  "main": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  },
  "peerDependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  }
}
```

```json
// apps/web/package.json
{
  "name": "@platform/web",
  "dependencies": {
    "@platform/ui": "*",      // * = Dùng version local trong workspace
    "@platform/utils": "*"
  }
}
```

---

## Phần 4: Nx — Monorepo Framework Toàn Diện

### Nx vs Turborepo

| Tiêu Chí | Nx | Turborepo |
| -------- | -- | --------- |
| **Complexity** | Cao hơn, nhiều tính năng | Đơn giản hơn, ít opinionated |
| **Code generation** | ✅ Generators mạnh mẽ | ❌ Không có |
| **Dependency graph** | ✅ Visualization tốt | ✅ Basic |
| **Plugin ecosystem** | ✅ Rất lớn (React, Angular, ...) | ⚠️ Ít hơn |
| **CI optimization** | ✅ Affected commands | ✅ Filtering |
| **Remote cache** | ✅ Nx Cloud | ✅ Vercel / Self-hosted |
| **Setup** | Phức tạp hơn | Đơn giản hơn |
| **Phù hợp** | Enterprise, large team | Startup, small-medium team |

### Nx Commands

```bash
# Tạo Nx workspace
npx create-nx-workspace@latest my-org --preset=react-monorepo

# Generate components, libraries, apps
nx generate @nx/react:component Button --project=ui
nx generate @nx/react:app new-app
nx generate @nx/react:library shared-utils

# Chạy chỉ tasks bị ảnh hưởng bởi thay đổi (CI optimization)
nx affected --target=build
nx affected --target=test

# Xem dependency graph trong browser
nx graph
```

---

## Phần 5: Chromatic — Visual Testing Tích Hợp Storybook

Chromatic là dịch vụ visual regression testing (Kiểm Thử Hồi Quy Trực Quan) tích hợp với Storybook.

```bash
npm install -D chromatic

# Publish Storybook lên Chromatic và so sánh visual changes
npx chromatic --project-token=<your-token>
```

**Workflow:**

```
Developer push code
  → CI chạy Chromatic
  → Chromatic chụp snapshot tất cả stories
  → So sánh với baseline (snapshot cũ)
  → Nếu có thay đổi visual → Thông báo reviewer
  → Reviewer approve/reject thay đổi trong Chromatic UI
```

---

## 6. Câu Hỏi Phỏng Vấn

### Q: Khi nào nên dùng Monorepo?

**A:** Monorepo phù hợp khi:
- Có nhiều apps/packages chia sẻ code chung (design system, utils)
- Team cùng làm việc trên nhiều sản phẩm liên quan
- Muốn đảm bảo consistency về tooling (TypeScript config, ESLint rules)
- Cần atomic refactoring xuyên suốt nhiều packages

Không nên dùng khi team nhỏ với một ứng dụng duy nhất — overhead không đáng.

### Q: Turborepo cache hoạt động như thế nào?

**A:** Turborepo tạo hash từ **inputs** của task: source files, environment variables, config files. Nếu hash giống lần trước → task output được lấy từ cache (local hoặc remote), không cần chạy lại. Điều này làm cho CI pipeline từ 10 phút có thể xuống còn 30 giây nếu chỉ thay đổi một phần nhỏ.

### Q: Storybook có thể thay thế unit tests không?

**A:** Không hoàn toàn. Storybook và tests bổ sung cho nhau:
- **Storybook** → Phát triển UI, visual review, accessibility check, documentation
- **Unit tests (RTL/Jest)** → Kiểm tra behavior, logic, interactions
- **Chromatic** → Visual regression (phát hiện thay đổi UI ngoài ý muốn)

Kết hợp cả ba cho coverage tốt nhất. Có thể **tái sử dụng stories trong unit tests** bằng `composeStories` để tránh duplicate setup.

---

## ✅ Checklist

### Storybook
- [ ] Cài Storybook vào project React hiện có
- [ ] Viết stories cho Button, Input, Modal components
- [ ] Thêm global decorators (Provider, Router)
- [ ] Dùng Controls addon để demo các states
- [ ] Setup a11y addon kiểm tra accessibility

### Turborepo
- [ ] Tạo monorepo với Turborepo và pnpm workspaces
- [ ] Config `turbo.json` với pipeline đúng dependencies
- [ ] Tạo shared `ui` package và dùng trong `web` app
- [ ] Setup shared TypeScript config trong `tsconfig` package
- [ ] Thử remote caching với Vercel

---

**Tài Liệu Tham Khảo:**
- [Storybook Docs](https://storybook.js.org/docs)
- [Turborepo Docs](https://turbo.build/repo/docs)
- [Nx Docs](https://nx.dev/)
- [Chromatic Docs](https://www.chromatic.com/docs/)
- [pnpm Workspaces](https://pnpm.io/workspaces)
