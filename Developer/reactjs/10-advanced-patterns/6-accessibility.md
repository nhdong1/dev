# 6 — Accessibility — A11y (Khả Năng Tiếp Cận)

> **Accessibility — A11y** (viết tắt: A + 11 chữ cái + y = "accessibility") là việc xây dựng ứng dụng có thể sử dụng bởi **tất cả mọi người**, bao gồm người khuyết tật — người dùng màn hình đọc (screen reader), người điều hướng bằng bàn phím, người có thị lực kém. **WCAG 2.1** (Web Content Accessibility Guidelines — Hướng Dẫn Tiếp Cận Nội Dung Web) là tiêu chuẩn quốc tế.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Hiểu và áp dụng **WCAG 2.1** (Web Content Accessibility Guidelines) ở mức **AA**
- [ ] Sử dụng **ARIA** (Accessible Rich Internet Applications — Ứng Dụng Internet Phong Phú Có Thể Tiếp Cận) đúng cách
- [ ] Implement **keyboard navigation** (điều hướng bàn phím) đầy đủ
- [ ] Xây dựng **accessible forms** (form có khả năng tiếp cận) với error messages rõ ràng
- [ ] Test accessibility với công cụ tự động và thủ công

---

## 1. Tại Sao Accessibility Quan Trọng?

```
Số liệu thực tế:
- 15% dân số thế giới sống với một dạng khuyết tật
- 253 triệu người bị suy giảm thị lực
- 466 triệu người bị suy giảm thính lực
- Ở Mỹ: WCAG tuân thủ có thể là yêu cầu pháp lý (ADA — Americans with Disabilities Act)

Lợi ích kỹ thuật:
✅ SEO — Screen readers đọc giống như search engine crawlers
✅ UX tốt hơn cho TẤT CẢ người dùng (không chỉ người khuyết tật)
✅ Keyboard navigation hữu ích cho power users
✅ Code sạch hơn — semantic HTML buộc viết HTML đúng cách
```

---

## 2. WCAG 2.1 — 4 Nguyên Tắc Cốt Lõi (POUR)

```
P — Perceivable (Có Thể Nhận Thấy)
  → Thông tin phải được trình bày theo nhiều cách
  → Ví dụ: Hình ảnh cần alt text, video cần captions

O — Operable (Có Thể Vận Hành)
  → Giao diện phải dùng được với bàn phím
  → Ví dụ: Không có bẫy keyboard, đủ thời gian để đọc

U — Understandable (Có Thể Hiểu Được)
  → Nội dung và cách vận hành phải rõ ràng
  → Ví dụ: Ngôn ngữ đơn giản, thông báo lỗi rõ ràng

R — Robust (Bền Vững)
  → Hoạt động với nhiều user agents (trình duyệt, screen readers)
  → Ví dụ: HTML hợp lệ, ARIA đúng cách
```

### Các Mức Độ Tuân Thủ

```
Level A   — Tối thiểu bắt buộc
Level AA  — Chuẩn thực tế, hầu hết ứng dụng hướng tới
Level AAA — Mức cao nhất, không thực tế cho mọi content
```

---

## 3. Semantic HTML — Nền Tảng Của Accessibility

### Dùng HTML Elements Đúng Ngữ Nghĩa

```tsx
// ❌ Semantic HTML sai — div không có ý nghĩa với screen readers
function BadNavigation() {
  return (
    <div onClick={handleClick} style={{ cursor: 'pointer' }}>
      Về trang chủ
    </div>
  );
}

// ✅ Semantic HTML đúng
function GoodNavigation() {
  return (
    <nav aria-label="Điều hướng chính">
      <ul>
        <li><a href="/">Trang chủ</a></li>
        <li><a href="/products">Sản phẩm</a></li>
        <li><a href="/contact">Liên hệ</a></li>
      </ul>
    </nav>
  );
}

// Heading hierarchy — Thứ bậc tiêu đề quan trọng
// ❌ Sai — bỏ qua h2, h3
<h1>Trang chủ</h1>
<h4>Giới thiệu</h4>

// ✅ Đúng — tuần tự từ h1 đến h6
<h1>Trang chủ</h1>
<h2>Giới thiệu</h2>
<h3>Sứ mệnh của chúng tôi</h3>
```

### Landmark Roles — Vùng Điều Hướng

```tsx
function PageLayout() {
  return (
    <>
      {/* Landmarks — Screen reader dùng để nhảy nhanh giữa các vùng */}
      <header>
        <nav aria-label="Điều hướng chính">
          {/* ... */}
        </nav>
      </header>

      <main id="main-content">
        {/* Skip link target — cho keyboard users */}
        <h1>Nội dung chính</h1>
        {/* ... */}
      </main>

      <aside aria-label="Nội dung liên quan">
        {/* Sidebar */}
      </aside>

      <footer>
        <nav aria-label="Điều hướng footer">
          {/* ... */}
        </nav>
      </footer>
    </>
  );
}
```

---

## 4. ARIA Attributes — Khi HTML Không Đủ

> **Nguyên tắc vàng:** Không dùng ARIA khi HTML native đã đủ. ARIA bổ sung, không thay thế semantic HTML.

### ARIA Roles — Vai Trò

```tsx
// Chỉ dùng role khi HTML element không có sẵn semantic phù hợp
<div role="tablist">
  <button role="tab" aria-selected={activeTab === 'home'} id="tab-home">
    Trang chủ
  </button>
</div>

<div role="tabpanel" aria-labelledby="tab-home" hidden={activeTab !== 'home'}>
  Nội dung tab
</div>

// Các roles phổ biến:
// role="alert"       — thông báo quan trọng, đọc ngay lập tức
// role="dialog"      — modal dialog
// role="tooltip"     — tooltip
// role="tablist"     — container của tabs
// role="tab"         — tab button
// role="tabpanel"    — nội dung của tab
// role="menu"        — dropdown menu
// role="menuitem"    — item trong menu
// role="listbox"     — select-like list
// role="option"      — option trong listbox
```

### ARIA States — Trạng Thái

```tsx
// aria-expanded — mở rộng hay thu gọn
<button
  aria-expanded={isOpen}
  aria-controls="menu-content"
  onClick={() => setIsOpen(!isOpen)}
>
  Menu ▼
</button>

<div id="menu-content" hidden={!isOpen}>
  {/* Menu items */}
</div>

// aria-selected — được chọn hay không (tabs, options)
<button role="tab" aria-selected={isActive}>Tab 1</button>

// aria-checked — checkbox/radio state
<div
  role="checkbox"
  aria-checked={isChecked}
  tabIndex={0}
  onClick={() => setIsChecked(!isChecked)}
  onKeyDown={(e) => e.key === ' ' && setIsChecked(!isChecked)}
>
  Tùy chọn
</div>

// aria-disabled — vô hiệu hóa
<button aria-disabled={isDisabled} onClick={isDisabled ? undefined : handleClick}>
  {isDisabled ? 'Không khả dụng' : 'Thực hiện'}
</button>

// aria-busy — đang tải
<div aria-busy={isLoading} aria-live="polite">
  {isLoading ? <Spinner /> : <Content />}
</div>
```

### ARIA Labels — Nhãn và Mô Tả

```tsx
// aria-label — label text trực tiếp (khi không có visible label)
<button aria-label="Đóng hộp thoại" onClick={onClose}>
  ✕  {/* Biểu tượng không đọc được bởi screen readers */}
</button>

// aria-labelledby — tham chiếu đến element khác làm label
<h2 id="dialog-title">Xác nhận xóa</h2>
<div role="dialog" aria-labelledby="dialog-title">
  {/* Screen reader đọc "Xác nhận xóa dialog" */}
</div>

// aria-describedby — mô tả bổ sung
<input
  id="password"
  type="password"
  aria-describedby="password-hint password-error"
/>
<p id="password-hint">Mật khẩu cần ít nhất 8 ký tự.</p>
<p id="password-error" role="alert">
  {errors.password}
</p>

// aria-live — thông báo thay đổi động
<div aria-live="polite" aria-atomic="true">
  {/* Nội dung thay đổi sẽ được screen reader đọc */}
  {statusMessage}
</div>

{/* aria-live="assertive" — đọc ngay, ngắt content đang đọc (chỉ cho lỗi quan trọng) */}
{/* aria-live="polite" — đọc khi screen reader rảnh (thông báo thông thường) */}
```

---

## 5. Accessible Forms — Form Có Khả Năng Tiếp Cận

```tsx
function AccessibleLoginForm() {
  const [errors, setErrors] = useState<Record<string, string>>({});

  return (
    <form onSubmit={handleSubmit} noValidate>
      {/* Thông báo lỗi tổng hợp ở đầu form */}
      {Object.keys(errors).length > 0 && (
        <div role="alert" aria-live="assertive">
          <h3>Vui lòng sửa các lỗi sau:</h3>
          <ul>
            {Object.entries(errors).map(([field, msg]) => (
              <li key={field}>
                <a href={`#${field}`}>{msg}</a>
              </li>
            ))}
          </ul>
        </div>
      )}

      <div className="form-group">
        {/* Label liên kết với input qua htmlFor/id */}
        <label htmlFor="email">
          Địa chỉ email
          <span aria-hidden="true"> *</span> {/* Ký hiệu * - ẩn khỏi screen reader */}
          <span className="sr-only">(bắt buộc)</span> {/* Screen reader only */}
        </label>
        <input
          id="email"
          name="email"
          type="email"
          autoComplete="email"
          required
          aria-required="true"
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : 'email-hint'}
        />
        <p id="email-hint" className="hint">
          Ví dụ: ten@domain.com
        </p>
        {errors.email && (
          <p id="email-error" role="alert" className="error">
            {errors.email}
          </p>
        )}
      </div>

      <div className="form-group">
        <label htmlFor="password">
          Mật khẩu
          <span className="sr-only">(bắt buộc)</span>
        </label>
        <input
          id="password"
          name="password"
          type="password"
          autoComplete="current-password"
          required
          aria-required="true"
          aria-invalid={!!errors.password}
          aria-describedby="password-requirements"
        />
        <p id="password-requirements">
          Ít nhất 8 ký tự, gồm chữ và số.
        </p>
      </div>

      <button type="submit">Đăng nhập</button>
    </form>
  );
}
```

---

## 6. Keyboard Navigation — Điều Hướng Bàn Phím

### Nguyên Tắc Keyboard Navigation

```
Tab      → Di chuyển focus về phía trước
Shift+Tab → Di chuyển focus về phía sau
Enter    → Kích hoạt button, link, submit form
Space    → Kích hoạt button, checkbox, toggle
Escape   → Đóng modal, dropdown, tooltip
Arrow    → Di chuyển trong menu, listbox, tabs
Home/End → Đến đầu/cuối danh sách
```

### Implement Keyboard Navigation Cho Custom Component

```tsx
// Ví dụ: Custom Listbox (Select dropdown)
function Listbox({ options, value, onChange }: ListboxProps) {
  const [activeIndex, setActiveIndex] = useState(
    options.findIndex((o) => o.value === value)
  );
  const listRef = useRef<HTMLUListElement>(null);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setActiveIndex((i) => Math.min(i + 1, options.length - 1));
        break;

      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex((i) => Math.max(i - 1, 0));
        break;

      case 'Home':
        e.preventDefault();
        setActiveIndex(0);
        break;

      case 'End':
        e.preventDefault();
        setActiveIndex(options.length - 1);
        break;

      case 'Enter':
      case ' ':
        e.preventDefault();
        if (activeIndex >= 0) {
          onChange(options[activeIndex].value);
        }
        break;

      case 'Escape':
        e.preventDefault();
        // Đóng dropdown
        break;
    }
  };

  // Scroll active item vào viewport
  useEffect(() => {
    const list = listRef.current;
    const activeItem = list?.children[activeIndex] as HTMLElement;
    activeItem?.scrollIntoView({ block: 'nearest' });
  }, [activeIndex]);

  return (
    <ul
      ref={listRef}
      role="listbox"
      aria-activedescendant={`option-${activeIndex}`}
      tabIndex={0}
      onKeyDown={handleKeyDown}
    >
      {options.map((option, index) => (
        <li
          key={option.value}
          id={`option-${index}`}
          role="option"
          aria-selected={option.value === value}
          className={index === activeIndex ? 'focused' : ''}
          onClick={() => onChange(option.value)}
        >
          {option.label}
        </li>
      ))}
    </ul>
  );
}
```

### Skip Link — Bỏ Qua Điều Hướng

```tsx
// Skip link giúp keyboard users bỏ qua navigation, nhảy thẳng vào nội dung
function SkipLink() {
  return (
    <a
      href="#main-content"
      className="skip-link"
      // Chỉ hiển thị khi focus (ẩn bình thường, hiện khi Tab)
    >
      Bỏ qua điều hướng, đến nội dung chính
    </a>
  );
}

// CSS tương ứng
const skipLinkStyles = `
.skip-link {
  position: absolute;
  transform: translateY(-100%);
  transition: transform 0.1s;
  background: #000;
  color: #fff;
  padding: 8px 16px;
  z-index: 9999;
}

.skip-link:focus {
  transform: translateY(0);
}
`;
```

---

## 7. Color Contrast — Tương Phản Màu Sắc

```
Yêu cầu WCAG 2.1 Level AA:
- Text thường (normal text): Tỷ lệ tương phản ≥ 4.5:1
- Text lớn (large text ≥ 18px hoặc 14px bold): Tỷ lệ ≥ 3:1
- UI components (borders, icons): Tỷ lệ ≥ 3:1

Ví dụ thực tế:
❌ #888888 trên #FFFFFF  → Tỷ lệ 3.54:1 (FAIL cho text thường)
✅ #595959 trên #FFFFFF  → Tỷ lệ 7.0:1 (PASS AA và AAA)
✅ #0000FF trên #FFFFFF  → Tỷ lệ 8.59:1 (PASS)

Công cụ kiểm tra:
- WebAIM Contrast Checker: https://webaim.org/resources/contrastchecker/
- Chrome DevTools → Elements → Accessibility pane
```

---

## 8. Focus Management — Quản Lý Focus

```tsx
// Focus indicator — Chỉ báo focus (outline)
// KHÔNG được xóa outline mà không thay thế bằng cái khác!

/* ❌ Xóa outline hoàn toàn — vi phạm WCAG */
button:focus { outline: none; }

/* ✅ Thay thế bằng custom focus indicator */
button:focus-visible {
  outline: 2px solid #005FCC;
  outline-offset: 2px;
}
/* :focus-visible chỉ áp dụng khi dùng keyboard, không khi click chuột */
```

```tsx
// Quản lý focus khi content thay đổi động
function NotificationList() {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const announcerRef = useRef<HTMLDivElement>(null);

  const addNotification = (msg: string) => {
    setNotifications((prev) => [...prev, { id: Date.now(), message: msg }]);
    // Thông báo với screen reader
    if (announcerRef.current) {
      announcerRef.current.textContent = msg;
    }
  };

  return (
    <>
      {/* Live region để thông báo với screen reader */}
      <div
        ref={announcerRef}
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="sr-only"  /* Ẩn về mặt thị giác, nhưng screen reader vẫn đọc */
      />

      {notifications.map((n) => (
        <div key={n.id} className="notification">
          {n.message}
        </div>
      ))}
    </>
  );
}
```

---

## 9. Accessible Images và Media

```tsx
// Alt text cho hình ảnh
// ✅ Ảnh thông tin — mô tả nội dung
<img src="/chart.png" alt="Biểu đồ doanh thu Q1 2026: tăng 23% so với Q4 2025" />

// ✅ Ảnh trang trí — alt rỗng để screen reader bỏ qua
<img src="/decorative-line.svg" alt="" role="presentation" />

// ✅ Icon button — label thay thế
<button aria-label="Xóa mục này">
  <TrashIcon aria-hidden="true" />  {/* Ẩn icon khỏi screen reader */}
</button>

// ✅ Icon kèm text — ẩn icon
<button>
  <TrashIcon aria-hidden="true" />
  <span>Xóa</span>
</button>

// SVG inline
<svg
  aria-label="Logo công ty"
  role="img"
  viewBox="0 0 100 100"
>
  <title>Logo ABC Company</title>
  {/* SVG paths */}
</svg>
```

---

## 10. Testing Accessibility

### Công Cụ Tự Động

```bash
# axe-core — thư viện kiểm tra a11y phổ biến nhất
npm install --save-dev axe-core @axe-core/react

# jest-axe — kiểm tra trong unit tests
npm install --save-dev jest-axe @types/jest-axe
```

```tsx
// Unit test với jest-axe
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('LoginForm không có vi phạm accessibility', async () => {
  const { container } = render(<LoginForm />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

// Hoặc với Vitest
import { describe, it, expect } from 'vitest';
import { render } from '@testing-library/react';
import { axe } from 'jest-axe';

describe('Button', () => {
  it('accessible với icon only', async () => {
    const { container } = render(
      <button aria-label="Đóng">✕</button>
    );
    expect(await axe(container)).toHaveNoViolations();
  });
});
```

### Kiểm Tra Thủ Công

```
Checklist kiểm tra thủ công:

Keyboard Navigation:
☐ Tab qua tất cả interactive elements theo thứ tự hợp lý
☐ Không có "keyboard trap" (bị kẹt ở một chỗ)
☐ Focus indicator luôn hiển thị rõ ràng
☐ Modal có focus trap đúng cách
☐ Escape đóng modal/dropdown

Screen Reader (VoiceOver trên macOS/iOS, NVDA/JAWS trên Windows):
☐ Tất cả hình ảnh có alt text có nghĩa
☐ Form labels liên kết đúng với inputs
☐ Thông báo lỗi được đọc khi xảy ra
☐ Heading hierarchy logic (h1 → h2 → h3)
☐ Landmarks (main, nav, aside, footer) được thông báo

Visual:
☐ Color contrast đạt WCAG AA
☐ Không dùng màu sắc đơn thuần để truyền thông tin
☐ Text có thể phóng to 200% mà không mất nội dung
☐ Không có animation tự động gây khó chịu (prefers-reduced-motion)
```

### Responsive Motion — Giảm Chuyển Động

```css
/* Tôn trọng cài đặt của người dùng */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

```tsx
// Trong React — hook kiểm tra prefers-reduced-motion
function usePrefersReducedMotion() {
  const [prefersReducedMotion, setPrefersReducedMotion] = useState(
    () => window.matchMedia('(prefers-reduced-motion: reduce)').matches
  );

  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
    const handler = () => setPrefersReducedMotion(mediaQuery.matches);
    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, []);

  return prefersReducedMotion;
}

// Sử dụng
function AnimatedCard() {
  const reducedMotion = usePrefersReducedMotion();

  return (
    <motion.div
      animate={{ opacity: 1, y: 0 }}
      transition={{
        duration: reducedMotion ? 0 : 0.3,
      }}
    >
      {/* ... */}
    </motion.div>
  );
}
```

---

## 11. Screen-Reader-Only Class

```css
/* Class ẩn nội dung về mặt thị giác nhưng screen reader vẫn đọc được */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}

/* Tailwind CSS: class="sr-only" */
```

```tsx
// Sử dụng sr-only để thêm context cho screen readers
function LoadingButton({ isLoading, children }: LoadingButtonProps) {
  return (
    <button disabled={isLoading}>
      {isLoading && (
        <>
          <Spinner aria-hidden="true" />
          <span className="sr-only">Đang xử lý, vui lòng chờ...</span>
        </>
      )}
      <span aria-hidden={isLoading}>{children}</span>
    </button>
  );
}

// Thêm context cho links
<a href="/products/123">
  Xem chi tiết
  <span className="sr-only"> sản phẩm Áo thun cotton</span>
</a>
```

---

## 12. Checklist WCAG 2.1 AA — Tóm Tắt

```
✅ Nhận Thấy (Perceivable):
  □ Hình ảnh có alt text mô tả đúng nội dung
  □ Video có captions và audio descriptions
  □ Color contrast text ≥ 4.5:1 (thường), ≥ 3:1 (lớn)
  □ Không dùng màu đơn thuần để truyền thông tin
  □ Text có thể phóng to 200% mà không bị cắt

✅ Vận Hành (Operable):
  □ Toàn bộ chức năng dùng được bằng keyboard
  □ Skip links có mặt
  □ Focus indicator rõ ràng
  □ Không có keyboard traps
  □ Đủ thời gian đọc nội dung (không tự-timeout ngắn)

✅ Hiểu Được (Understandable):
  □ lang attribute trên <html>
  □ Form labels liên kết đúng với inputs
  □ Error messages rõ ràng, chỉ ra cách sửa
  □ aria-describedby cho hướng dẫn bổ sung

✅ Bền Vững (Robust):
  □ HTML hợp lệ (validate HTML)
  □ ARIA roles đúng
  □ Name, role, value cho custom components
  □ Status messages qua aria-live
```

---

## 📝 Checklist Tự Đánh Giá

- [ ] Giải thích được 4 nguyên tắc POUR của WCAG
- [ ] Viết form accessible với labels, errors, aria attributes đầy đủ
- [ ] Implement keyboard navigation cho custom dropdown/listbox
- [ ] Chạy axe-core tests và sửa violations
- [ ] Test với VoiceOver hoặc NVDA (thực tế)

---

## 🔗 Tài Nguyên Học Thêm

- [WCAG 2.1 Official](https://www.w3.org/TR/WCAG21/) — Tiêu chuẩn chính thức
- [WebAIM](https://webaim.org/) — Hướng dẫn thực tế
- [A11y Project](https://www.a11yproject.com/) — Community checklist
- [Inclusive Components](https://inclusive-components.design/) — Patterns cụ thể
- [Radix UI](https://www.radix-ui.com/) — Accessible headless components

---

## 🔗 Điều Hướng

- **Trước đó:** [5-portals-boundaries.md](./5-portals-boundaries.md) — Portals và Error Boundaries
- **Tiếp theo:** [11-interview-prep/](../11-interview-prep/) — Chuẩn bị phỏng vấn
- **Quay lại:** [README.md](./README.md) — Tổng quan Advanced Patterns
- **Chỉ mục:** [INDEX.md](../INDEX.md)
