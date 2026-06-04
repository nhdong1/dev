# 1 — Compound Components (Component Hợp Thành)

> **Compound Components** (Component Hợp Thành) là pattern cho phép xây dựng một nhóm component liên kết chặt chẽ với nhau, chia sẻ trạng thái nội bộ qua **Context API**, mang lại API linh hoạt và biểu cảm hơn so với component đơn với hàng chục props.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Giải thích Compound Components pattern và vấn đề nó giải quyết
- [ ] Implement Compound Components với **Context API** nội bộ
- [ ] Thiết kế **sub-components** (component con) được gắn vào parent component
- [ ] Xây dựng cả **uncontrolled** (không kiểm soát) và **controlled** (kiểm soát) variants
- [ ] Áp dụng pattern này cho các UI phức tạp như Tabs, Accordion, Select, Modal

---

## 1. Vấn Đề Cần Giải Quyết

### ❌ Cách Tiếp Cận Naive — Props Drilling Không Linh Hoạt

```tsx
// Vấn đề: quá nhiều props, khó mở rộng
<Tabs
  tabs={['Thông tin', 'Lịch sử', 'Cài đặt']}
  activeTab={0}
  onTabChange={setActiveTab}
  tabStyle="underline"
  tabSize="large"
  contentPadding={16}
  showIcons={true}
  icons={[UserIcon, HistoryIcon, SettingsIcon]}
/>
```

Vấn đề với cách này:
- **Prop explosion** (bùng nổ props): Mỗi tính năng mới thêm một prop
- **Inflexible** (thiếu linh hoạt): Không thể thay đổi thứ tự, thêm nội dung tùy chỉnh
- **Opaque** (tối nghĩa): Không rõ cấu trúc HTML/DOM bên trong

### ✅ Compound Components — API Linh Hoạt

```tsx
// Biểu cảm, linh hoạt, dễ đọc
<Tabs defaultValue="info">
  <Tabs.List>
    <Tabs.Trigger value="info">
      <UserIcon /> Thông tin
    </Tabs.Trigger>
    <Tabs.Trigger value="history">
      <HistoryIcon /> Lịch sử
    </Tabs.Trigger>
    <Tabs.Trigger value="settings">
      <SettingsIcon /> Cài đặt
    </Tabs.Trigger>
  </Tabs.List>

  <Tabs.Content value="info">
    <ProfileInfo />
  </Tabs.Content>
  <Tabs.Content value="history">
    <ActivityHistory />
  </Tabs.Content>
  <Tabs.Content value="settings">
    <UserSettings />
  </Tabs.Content>
</Tabs>
```

---

## 2. Implement Cơ Bản — Accordion

### Bước 1: Tạo Context Nội Bộ

```tsx
import React, { createContext, useContext, useState } from 'react';

// --- Định nghĩa kiểu dữ liệu ---
interface AccordionContextValue {
  openItems: string[];
  toggleItem: (value: string) => void;
  multiple: boolean;
}

// Context chỉ dùng nội bộ — không export ra ngoài
const AccordionContext = createContext<AccordionContextValue | null>(null);

// Hook helper để dùng context — báo lỗi nếu dùng sai
function useAccordion() {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error(
      'Accordion sub-components phải được dùng bên trong <Accordion>'
    );
  }
  return context;
}
```

### Bước 2: Parent Component

```tsx
interface AccordionProps {
  children: React.ReactNode;
  multiple?: boolean;          // Cho phép mở nhiều item cùng lúc
  defaultValue?: string[];     // Uncontrolled: giá trị mặc định
  value?: string[];            // Controlled: giá trị từ bên ngoài
  onChange?: (values: string[]) => void;
}

function Accordion({
  children,
  multiple = false,
  defaultValue = [],
  value,
  onChange,
}: AccordionProps) {
  // Uncontrolled state — dùng khi không truyền value từ ngoài
  const [internalOpen, setInternalOpen] = useState<string[]>(defaultValue);

  // Nếu có value prop → controlled mode
  const openItems = value !== undefined ? value : internalOpen;

  const toggleItem = (itemValue: string) => {
    let nextOpen: string[];

    if (openItems.includes(itemValue)) {
      // Đang mở → đóng lại
      nextOpen = openItems.filter((v) => v !== itemValue);
    } else if (multiple) {
      // Cho phép nhiều → thêm vào danh sách
      nextOpen = [...openItems, itemValue];
    } else {
      // Chỉ một → thay thế
      nextOpen = [itemValue];
    }

    // Uncontrolled: tự cập nhật state nội bộ
    if (value === undefined) {
      setInternalOpen(nextOpen);
    }
    // Controlled: thông báo ra ngoài
    onChange?.(nextOpen);
  };

  return (
    <AccordionContext.Provider value={{ openItems, toggleItem, multiple }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}
```

### Bước 3: Sub-components

```tsx
interface AccordionItemProps {
  value: string;
  children: React.ReactNode;
}

function AccordionItem({ value, children }: AccordionItemProps) {
  const { openItems, toggleItem } = useAccordion();
  const isOpen = openItems.includes(value);

  return (
    <div className={`accordion-item ${isOpen ? 'open' : ''}`}>
      {/* Clone children để truyền thêm props nếu cần */}
      {React.Children.map(children, (child) => {
        if (React.isValidElement(child)) {
          return React.cloneElement(child, { value, isOpen } as any);
        }
        return child;
      })}
    </div>
  );
}

interface AccordionTriggerProps {
  children: React.ReactNode;
  value?: string;   // Được truyền từ AccordionItem qua cloneElement
  isOpen?: boolean; // Được truyền từ AccordionItem qua cloneElement
}

function AccordionTrigger({ children, value, isOpen }: AccordionTriggerProps) {
  const { toggleItem } = useAccordion();

  return (
    <button
      className="accordion-trigger"
      onClick={() => value && toggleItem(value)}
      aria-expanded={isOpen}         // ARIA — cho screen readers
      aria-controls={`panel-${value}`}
    >
      {children}
      <span className={`icon ${isOpen ? 'rotate' : ''}`}>▼</span>
    </button>
  );
}

interface AccordionContentProps {
  children: React.ReactNode;
  value?: string;
  isOpen?: boolean;
}

function AccordionContent({ children, value, isOpen }: AccordionContentProps) {
  return (
    <div
      id={`panel-${value}`}
      role="region"
      aria-labelledby={`trigger-${value}`}
      hidden={!isOpen}
      className="accordion-content"
    >
      {children}
    </div>
  );
}
```

### Bước 4: Gắn Sub-components Vào Parent

```tsx
// Phương pháp 1: Gắn trực tiếp như static properties
Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;

export { Accordion };
```

### Cách Sử Dụng

```tsx
// Uncontrolled — tự quản lý state nội bộ
<Accordion defaultValue={['item-1']} multiple>
  <Accordion.Item value="item-1">
    <Accordion.Trigger>Câu hỏi 1: React là gì?</Accordion.Trigger>
    <Accordion.Content>
      React là thư viện JavaScript để xây dựng UI.
    </Accordion.Content>
  </Accordion.Item>

  <Accordion.Item value="item-2">
    <Accordion.Trigger>Câu hỏi 2: Hooks là gì?</Accordion.Trigger>
    <Accordion.Content>
      Hooks cho phép dùng state và lifecycle trong functional components.
    </Accordion.Content>
  </Accordion.Item>
</Accordion>

// Controlled — parent component kiểm soát
function ControlledAccordion() {
  const [open, setOpen] = useState<string[]>([]);

  return (
    <Accordion value={open} onChange={setOpen}>
      {/* ... items */}
    </Accordion>
  );
}
```

---

## 3. Pattern Nâng Cao — Tabs Component

### Thiết Kế API Trước (API-first Design)

```tsx
// Mục tiêu thiết kế API trước khi implement
<Tabs defaultValue="overview">
  <Tabs.List aria-label="Thông tin sản phẩm">
    <Tabs.Trigger value="overview">Tổng quan</Tabs.Trigger>
    <Tabs.Trigger value="specs">Thông số</Tabs.Trigger>
    <Tabs.Trigger value="reviews" disabled>
      Đánh giá (sắp có)
    </Tabs.Trigger>
  </Tabs.List>

  <Tabs.Content value="overview">
    <OverviewPanel />
  </Tabs.Content>
  <Tabs.Content value="specs">
    <SpecsPanel />
  </Tabs.Content>
</Tabs>
```

### Implement Tabs

```tsx
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (value: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabs() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Phải dùng bên trong <Tabs>');
  return ctx;
}

// --- Parent ---
interface TabsProps {
  children: React.ReactNode;
  defaultValue?: string;
  value?: string;
  onChange?: (value: string) => void;
}

function Tabs({ children, defaultValue = '', value, onChange }: TabsProps) {
  const [internal, setInternal] = useState(defaultValue);
  const activeTab = value !== undefined ? value : internal;

  const setActiveTab = (v: string) => {
    if (value === undefined) setInternal(v);
    onChange?.(v);
  };

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

// --- Sub-components ---
function TabsList({ children, ...props }: React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div role="tablist" className="tabs-list" {...props}>
      {children}
    </div>
  );
}

interface TabsTriggerProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  value: string;
  children: React.ReactNode;
}

function TabsTrigger({ value, children, disabled, ...props }: TabsTriggerProps) {
  const { activeTab, setActiveTab } = useTabs();
  const isActive = activeTab === value;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      aria-disabled={disabled}
      tabIndex={isActive ? 0 : -1}  // Keyboard navigation
      onClick={() => !disabled && setActiveTab(value)}
      className={`tabs-trigger ${isActive ? 'active' : ''} ${disabled ? 'disabled' : ''}`}
      {...props}
    >
      {children}
    </button>
  );
}

interface TabsContentProps {
  value: string;
  children: React.ReactNode;
}

function TabsContent({ value, children }: TabsContentProps) {
  const { activeTab } = useTabs();

  if (activeTab !== value) return null;

  return (
    <div role="tabpanel" className="tabs-content">
      {children}
    </div>
  );
}

// Gắn sub-components
Tabs.List = TabsList;
Tabs.Trigger = TabsTrigger;
Tabs.Content = TabsContent;
```

---

## 4. Flexible Compound Components — Không Yêu Cầu Cấu Trúc Lồng Nhau Cố Định

Đôi khi bạn muốn sub-components có thể đặt ở bất kỳ đâu trong cây, không nhất thiết phải là con trực tiếp.

```tsx
// Context-based: sub-components có thể ở bất kỳ độ sâu nào
<Select defaultValue="vn">
  <div className="custom-wrapper">
    <Select.Trigger placeholder="Chọn quốc gia" />
  </div>
  <Select.Content>
    <Select.Item value="vn">🇻🇳 Việt Nam</Select.Item>
    <Select.Item value="us">🇺🇸 Hoa Kỳ</Select.Item>
    <Select.Item value="jp">🇯🇵 Nhật Bản</Select.Item>
  </Select.Content>
</Select>
```

Context truyền qua React tree cho phép nesting tùy ý — đây là ưu điểm lớn so với `React.cloneElement`.

---

## 5. So Sánh: cloneElement vs Context

| Tiêu Chí | `React.cloneElement` | Context API |
| -------- | -------------------- | ----------- |
| **Nesting** (Độ lồng nhau) | Chỉ direct children | Bất kỳ độ sâu |
| **TypeScript** | Khó type-safe | Dễ hơn |
| **Performance** (Hiệu năng) | Clone = re-render | Chỉ consumer re-render |
| **Debug** | Khó trace props | Dễ với DevTools |
| **Khuyến nghị** | Legacy, simple cases | Modern, preferred |

```tsx
// ❌ cloneElement — chỉ hoạt động với direct children
function RadioGroup({ children, name }: { children: ReactNode; name: string }) {
  return (
    <div>
      {React.Children.map(children, (child) =>
        React.cloneElement(child as ReactElement, { name })
      )}
    </div>
  );
}

// ✅ Context — hoạt động với bất kỳ độ sâu nào
const RadioGroupContext = createContext<{ name: string } | null>(null);

function RadioGroup({ children, name }: { children: ReactNode; name: string }) {
  return (
    <RadioGroupContext.Provider value={{ name }}>
      <div>{children}</div>
    </RadioGroupContext.Provider>
  );
}
```

---

## 6. Ví Dụ Thực Tế — Modal Component

```tsx
interface ModalContextValue {
  isOpen: boolean;
  close: () => void;
}

const ModalContext = createContext<ModalContextValue | null>(null);

function useModal() {
  const ctx = useContext(ModalContext);
  if (!ctx) throw new Error('Phải dùng bên trong <Modal>');
  return ctx;
}

// Parent
interface ModalProps {
  children: React.ReactNode;
  open: boolean;
  onClose: () => void;
}

function Modal({ children, open, onClose }: ModalProps) {
  // Đóng khi nhấn Escape
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    if (open) document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [open, onClose]);

  if (!open) return null;

  return (
    <ModalContext.Provider value={{ isOpen: open, close: onClose }}>
      {/* Overlay */}
      <div
        className="modal-overlay"
        onClick={onClose}
        aria-hidden="true"
      />
      {/* Dialog */}
      <div
        role="dialog"
        aria-modal="true"
        className="modal-dialog"
      >
        {children}
      </div>
    </ModalContext.Provider>
  );
}

// Sub-components
function ModalHeader({ children }: { children: ReactNode }) {
  return <div className="modal-header">{children}</div>;
}

function ModalBody({ children }: { children: ReactNode }) {
  return <div className="modal-body">{children}</div>;
}

function ModalFooter({ children }: { children: ReactNode }) {
  return <div className="modal-footer">{children}</div>;
}

function ModalCloseButton() {
  const { close } = useModal();
  return (
    <button onClick={close} aria-label="Đóng modal" className="modal-close">
      ✕
    </button>
  );
}

// Gắn vào parent
Modal.Header = ModalHeader;
Modal.Body = ModalBody;
Modal.Footer = ModalFooter;
Modal.CloseButton = ModalCloseButton;

// Sử dụng
function App() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button onClick={() => setOpen(true)}>Mở Modal</button>

      <Modal open={open} onClose={() => setOpen(false)}>
        <Modal.Header>
          <h2>Xác nhận xóa</h2>
          <Modal.CloseButton />
        </Modal.Header>
        <Modal.Body>
          <p>Bạn có chắc muốn xóa mục này không?</p>
        </Modal.Body>
        <Modal.Footer>
          <button onClick={() => setOpen(false)}>Hủy</button>
          <button className="danger" onClick={handleDelete}>Xóa</button>
        </Modal.Footer>
      </Modal>
    </>
  );
}
```

---

## 7. Pattern Trong Thực Tế — Radix UI và shadcn/ui

Radix UI là thư viện component sử dụng Compound Components rộng rãi:

```tsx
// Radix UI Dialog — ví dụ compound components trong production
import * as Dialog from '@radix-ui/react-dialog';

<Dialog.Root open={open} onOpenChange={setOpen}>
  <Dialog.Trigger asChild>
    <button>Mở dialog</button>
  </Dialog.Trigger>

  <Dialog.Portal>
    <Dialog.Overlay className="overlay" />
    <Dialog.Content className="content">
      <Dialog.Title>Tiêu đề</Dialog.Title>
      <Dialog.Description>Mô tả ngắn</Dialog.Description>
      {/* Nội dung */}
      <Dialog.Close asChild>
        <button>Đóng</button>
      </Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

---

## 8. TypeScript Best Practices — Typing Compound Components

```tsx
// Định nghĩa type đầy đủ cho compound component
interface AccordionComponent extends React.FC<AccordionProps> {
  Item: typeof AccordionItem;
  Trigger: typeof AccordionTrigger;
  Content: typeof AccordionContent;
}

// Cast để TypeScript nhận ra các static properties
const Accordion = AccordionRoot as AccordionComponent;
Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;

// Bây giờ TypeScript biết Accordion.Item, .Trigger, .Content tồn tại
```

---

## 9. Bài Tập Thực Hành

### Bài 1 — Cơ Bản (30 phút)
Xây dựng `Accordion` component với:
- Single và multiple mode
- Controlled và uncontrolled
- Animation khi mở/đóng

### Bài 2 — Trung Bình (45 phút)
Xây dựng `Tabs` component với:
- Keyboard navigation (Arrow keys, Home, End)
- ARIA attributes đầy đủ
- Lazy loading content (chỉ render tab đang active)

### Bài 3 — Nâng Cao (60 phút)
Xây dựng `Select` / `Combobox` component:
- Search/filter options
- Multi-select
- Keyboard navigation trong dropdown
- Accessible với ARIA Listbox pattern

---

## 📝 Checklist Tự Đánh Giá

- [ ] Hiểu tại sao Compound Components giải quyết "prop explosion"
- [ ] Implement được Compound Components với Context API nội bộ
- [ ] Phân biệt controlled vs uncontrolled trong Compound Components
- [ ] Biết khi nào dùng `cloneElement` vs Context
- [ ] Thêm ARIA attributes cơ bản vào component

---

## 🔗 Điều Hướng

- **Trước đó:** [README.md](./README.md) — Tổng quan Advanced Patterns
- **Tiếp theo:** [2-render-props.md](./2-render-props.md) — Render Props pattern
- **Chỉ mục:** [INDEX.md](../INDEX.md)
