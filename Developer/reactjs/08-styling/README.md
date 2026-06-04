# 08 — Styling (Tạo Kiểu Dáng UI trong React)

> **Styling** (Tạo Kiểu Dáng) trong React không có một "chuẩn vàng" duy nhất — mỗi dự án, mỗi team có lựa chọn khác nhau. Hiểu rõ **trade-offs** (sự đánh đổi) của từng phương pháp giúp bạn chọn đúng công cụ cho đúng bối cảnh.

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Dùng **CSS Modules** (Module CSS) để cô lập style theo phạm vi component
- [ ] Áp dụng **Tailwind CSS** — framework utility-first (tiện ích-đầu tiên) — nhanh và nhất quán
- [ ] Xây dựng component với **Styled Components** — CSS-in-JS với tagged template literals
- [ ] Tận dụng **Emotion** — CSS-in-JS hiệu năng cao, linh hoạt hơn Styled Components
- [ ] Tích hợp **Design Systems** (Hệ Thống Thiết Kế) như **shadcn/ui**, **Radix UI**, **Material UI**
- [ ] Chọn đúng giải pháp styling phù hợp với yêu cầu dự án

---

## 📁 Nội Dung

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-css-modules.md](./1-css-modules.md) | CSS Modules — scope CSS theo component, tránh xung đột tên class | Beginner |
| [2-tailwind-css.md](./2-tailwind-css.md) | Tailwind CSS — utility-first, responsive, dark mode, custom design tokens | Beginner–Intermediate |
| [3-styled-components.md](./3-styled-components.md) | Styled Components — CSS-in-JS, dynamic styles, theming với ThemeProvider | Intermediate |
| [4-emotion.md](./4-emotion.md) | Emotion — CSS-in-JS hiệu năng cao, css prop, styled API, SSR | Intermediate |
| [5-design-systems.md](./5-design-systems.md) | shadcn/ui, Radix UI, Material UI — thư viện component và hệ thống thiết kế | Intermediate–Advanced |

---

## 🗺️ Lộ Trình Học

```
CSS Modules  →  Tailwind CSS  →  Styled Components  →  Emotion  →  Design Systems
     ↓               ↓                  ↓                  ↓              ↓
"Scoped CSS,    "Utility classes,   "CSS-in-JS,        "CSS prop,     "shadcn/ui,
 tách file,      không cần đặt       dynamic styles,    performant,    Radix UI,
 zero runtime"   tên class"          theming"           SSR-friendly"  MUI, Ant"
```

**Thứ tự khuyến nghị:**
1. Bắt đầu với `1-css-modules.md` — nền tảng, dễ hiểu, không cần cài thêm gì
2. Học `2-tailwind-css.md` — phổ biến nhất hiện tại, tốc độ phát triển rất cao
3. Đọc `3-styled-components.md` — CSS-in-JS kinh điển, nhiều dự án vẫn dùng
4. Tham khảo `4-emotion.md` — khi cần hiệu năng tốt hơn hoặc dùng với MUI
5. Kết thúc với `5-design-systems.md` — học cách dùng và customize component libraries

---

## ⏱️ Thời Gian Ước Tính

| Hoạt Động | Thời Gian |
| --------- | --------- |
| Đọc lý thuyết (5 files) | 3–4 giờ |
| Thực hành code theo ví dụ | 2–3 giờ |
| Mini-project: Build UI component library nhỏ | 2–3 giờ |
| **Tổng cộng** | **7–10 giờ** |

---

## 💡 Tại Sao Styling Quan Trọng?

### Thách Thức Trong Styling React

```
Vấn đề cốt lõi của CSS toàn cục:
1. Name Collision (Xung Đột Tên) → .button ở module A ghi đè .button ở module B
2. Specificity Wars (Chiến Tranh Độ Ưu Tiên) → !important tràn lan
3. Dead Code (Code Chết) → CSS không biết class nào còn dùng, class nào không
4. Theming khó → thay đổi màu sắc toàn ứng dụng phức tạp
5. Responsive design → viết breakpoints lặp đi lặp lại
```

### Các Giải Pháp Phổ Biến

```
Lịch sử phát triển:
CSS thuần → BEM / OOCSS → CSS Preprocessors (Sass/Less) → CSS Modules
                                                          → CSS-in-JS (Styled Components, Emotion)
                                                          → Utility-First (Tailwind)
                                                          → Zero-Runtime (Vanilla Extract, Panda CSS)
```

---

## 🆚 So Sánh Nhanh Các Giải Pháp

| Tiêu Chí | CSS Modules | Tailwind CSS | Styled Components | Emotion | shadcn/ui |
| -------- | ----------- | ------------ | ----------------- | ------- | --------- |
| **Runtime overhead** (Chi phí runtime) | Không có | Không có | Có (nhẹ) | Có (rất nhẹ) | Tùy thư viện base |
| **Bundle size** (Kích thước bundle) | Nhỏ | Nhỏ (PurgeCSS) | Trung bình | Nhỏ hơn SC | Nhỏ (copy vào src) |
| **TypeScript support** | Tốt (với types) | Tốt | Rất tốt | Rất tốt | Xuất sắc |
| **Dynamic styles** (Style động) | Hạn chế | Hạn chế | Rất dễ | Rất dễ | Dùng props |
| **Theming** | CSS Variables | Design tokens | ThemeProvider | ThemeProvider | CSS Variables |
| **SSR** (Server-Side Rendering) | ✅ Tốt | ✅ Tốt | ⚠️ Cần setup | ✅ Tốt | ✅ Tốt |
| **Learning curve** (Độ khó học) | Thấp | Trung bình | Trung bình | Trung bình | Thấp |
| **Phổ biến (2025)** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## 🎯 Khi Nào Dùng Gì?

### CSS Modules — Chọn Khi:
```
✅ Muốn giữ CSS thuần, không học thêm cú pháp mới
✅ Team quen với CSS/SCSS truyền thống
✅ Dự án không có nhu cầu dynamic styling phức tạp
✅ Cần bundle size nhỏ nhất, zero runtime overhead
✅ Vite / Create React App — setup out of the box
```

### Tailwind CSS — Chọn Khi:
```
✅ Muốn phát triển UI nhanh, không đặt tên class
✅ Dự án cần design system nhất quán
✅ Team thích làm việc trực tiếp trong JSX
✅ Responsive và dark mode thường xuyên
✅ Nhiều component dùng chung pattern
⚠️ Không phù hợp nếu cần highly custom, pixel-perfect design
```

### Styled Components / Emotion — Chọn Khi:
```
✅ Cần dynamic styles phụ thuộc vào props
✅ Cần Theming mạnh (nhiều theme, white-label)
✅ Component library cần encapsulation cao
✅ Team quen với CSS-in-JS workflow
⚠️ Tránh khi SSR performance là ưu tiên hàng đầu
```

### shadcn/ui / Radix UI — Chọn Khi:
```
✅ Cần component có sẵn, accessible, customizable
✅ Muốn tùy chỉnh hoàn toàn (shadcn copy source vào project)
✅ Kết hợp với Tailwind CSS
✅ Không muốn "bị kẹt" trong design system cứng nhắc
```

### Material UI (MUI) — Chọn Khi:
```
✅ Cần bộ component phong phú, đầy đủ ngay lập tức
✅ Thiết kế theo Material Design guidelines
✅ Dashboard, internal tools, admin panels
⚠️ Bundle size lớn hơn các lựa chọn khác
```

---

## 📊 Xu Hướng Styling (2025–2026)

```
Tăng mạnh:
→ Tailwind CSS — tiếp tục thống trị
→ shadcn/ui — "copy-paste component library" mới nhất
→ Panda CSS, Vanilla Extract — zero-runtime CSS-in-JS
→ CSS Layers (@layer) — native CSS cascade management

Ổn định:
→ CSS Modules — vẫn là lựa chọn an toàn, bền vững
→ Emotion — được MUI v5+ sử dụng

Giảm dần:
→ Styled Components — ít được chọn cho project mới
→ CSS-in-JS có runtime overhead — do React Server Components không hỗ trợ
```

---

## ⚠️ CSS-in-JS Và React Server Components

> Đây là điểm quan trọng khi chọn giải pháp styling cho ứng dụng Next.js App Router hoặc RSC:

```
Vấn đề: Nhiều thư viện CSS-in-JS (Styled Components, Emotion theo cách truyền thống)
sử dụng React context và hooks → KHÔNG hoạt động trong Server Components.

Giải pháp:
1. Dùng CSS Modules hoặc Tailwind — không có runtime, hoạt động với RSC ✅
2. Dùng Emotion với "use client" directive — chỉ ở Client Components ⚠️
3. Dùng zero-runtime CSS-in-JS: Panda CSS, Vanilla Extract, StyleX ✅
4. Dùng shadcn/ui (Tailwind-based) — fully compatible với RSC ✅
```

---

## 🔑 Khái Niệm Cần Nắm

### CSS Specificity — Độ Ưu Tiên CSS

```
Thứ tự ưu tiên (từ cao đến thấp):
1. !important — Quan trọng (tránh dùng)
2. Inline styles — Style nội tuyến (style={{ color: 'red' }})
3. ID selectors — Chọn theo ID (#myId)
4. Class, attribute, pseudo-class — (.class, [attr], :hover)
5. Element, pseudo-element — (div, ::before)
```

### CSS Variables — Biến CSS (Custom Properties)

```css
/* Định nghĩa biến ở :root để dùng toàn cục */
:root {
  --color-primary: #3b82f6;
  --spacing-4: 1rem;
  --radius-md: 0.375rem;
}

/* Sử dụng trong component */
.button {
  background-color: var(--color-primary);
  padding: var(--spacing-4);
  border-radius: var(--radius-md);
}
```

### Design Tokens — Token Thiết Kế

```
Design tokens là các giá trị thiết kế được đặt tên và có thể tái sử dụng:
- Colors: --color-primary, --color-danger
- Spacing: --space-1 (4px), --space-2 (8px), --space-4 (16px)
- Typography: --font-size-sm, --font-weight-bold
- Border radius: --radius-sm, --radius-md, --radius-full
- Shadows: --shadow-sm, --shadow-md, --shadow-xl

→ Được Tailwind CSS, shadcn/ui, MUI... đều xây dựng trên nền tảng này
```

---

## 🔨 Mini-Project: Xây Dựng Design System Nhỏ

Sau khi hoàn thành cả 5 file, hãy thực hành:

```
Bước 1: Tạo Button component theo 3 cách
  → CSS Modules: Button.module.css
  → Tailwind: className="bg-blue-500 hover:bg-blue-600 ..."
  → Styled Components: const StyledButton = styled.button`...`

Bước 2: Thêm variants (primary, secondary, danger) và sizes (sm, md, lg)

Bước 3: Implement dark mode với CSS Variables

Bước 4: So sánh DX (Developer Experience) và bundle output của từng cách

Bước 5: Dùng shadcn/ui Button — customize lại theo brand color
```

---

## 🔗 Điều Hướng

- **Trước đó:** [07-testing/](../07-testing/) — Kiểm thử React
- **Tiếp theo:** [09-ecosystem/](../09-ecosystem/) — Hệ sinh thái React
- **Quay lại:** [README.md tổng quan](../README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
