# 5 — Portals và Error Boundaries

> **Portals** (Cổng Render) cho phép render component ra ngoài DOM hierarchy của component cha, giải quyết các vấn đề với `z-index`, `overflow: hidden`, và positioning. **Error Boundaries** (Ranh Giới Bắt Lỗi) là cơ chế bắt lỗi JavaScript trong component tree để hiển thị fallback UI thay vì crash toàn bộ ứng dụng.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Giải thích khi nào cần dùng **Portals** và triển khai chính xác
- [ ] Xử lý **focus trap** (bẫy focus) và **keyboard navigation** (điều hướng bàn phím) trong modal
- [ ] Implement **Error Boundary** class component đúng chuẩn
- [ ] Thiết kế **error handling strategy** (chiến lược xử lý lỗi) ở cấp độ ứng dụng
- [ ] Kết hợp Error Boundary với **Suspense** boundary

---

## PHẦN 1: PORTALS

## 1. Vấn Đề Portals Giải Quyết

### Vấn Đề: Overflow và Z-index

```
Cấu trúc DOM điển hình:
<div class="app">
  <div class="sidebar" style="overflow: hidden;">
    <div class="user-card">
      <button>Xem thêm</button>
      <!-- Tooltip render ở đây bị CẮT bởi overflow:hidden của sidebar! -->
      <div class="tooltip">Thông tin bổ sung</div>
    </div>
  </div>
</div>

Giải pháp với Portal:
<div class="app">
  <div class="sidebar" style="overflow: hidden;">
    <div class="user-card">
      <button>Xem thêm</button>
    </div>
  </div>
</div>
<!-- Tooltip render ở đây, ngoài sidebar, không bị cắt -->
<div class="tooltip">Thông tin bổ sung</div>
```

### API Cơ Bản

```tsx
import { createPortal } from 'react-dom';

function Tooltip({ children, targetRef }: { children: ReactNode; targetRef: RefObject<Element> }) {
  // Render children vào document.body, không phải vào DOM parent
  return createPortal(
    <div className="tooltip">{children}</div>,
    document.body  // Container đích — có thể là bất kỳ DOM element nào
  );
}
```

---

## 2. Implement Modal Với Portal

```tsx
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
  title?: string;
}

function Modal({ isOpen, onClose, children, title }: ModalProps) {
  const modalRoot = document.getElementById('modal-root') ?? document.body;

  // Focus trap — bẫy focus trong modal khi đang mở
  const firstFocusableRef = useRef<HTMLButtonElement>(null);

  useEffect(() => {
    if (!isOpen) return;

    // Focus vào element đầu tiên khi mở
    firstFocusableRef.current?.focus();

    // Ngăn scroll của body khi modal mở
    document.body.style.overflow = 'hidden';
    return () => {
      document.body.style.overflow = '';
    };
  }, [isOpen]);

  // Đóng khi nhấn Escape
  useEffect(() => {
    if (!isOpen) return;
    const handler = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <>
      {/* Overlay — lớp mờ phía sau */}
      <div
        className="modal-overlay"
        onClick={onClose}
        aria-hidden="true"
        style={{
          position: 'fixed', inset: 0,
          background: 'rgba(0,0,0,0.5)',
          zIndex: 1000,
        }}
      />

      {/* Dialog — hộp thoại */}
      <div
        role="dialog"
        aria-modal="true"
        aria-labelledby={title ? 'modal-title' : undefined}
        style={{
          position: 'fixed',
          top: '50%', left: '50%',
          transform: 'translate(-50%, -50%)',
          background: 'white',
          borderRadius: '8px',
          padding: '24px',
          zIndex: 1001,
          maxWidth: '500px',
          width: '90vw',
        }}
      >
        {title && <h2 id="modal-title">{title}</h2>}

        {children}

        <button
          ref={firstFocusableRef}
          onClick={onClose}
          aria-label="Đóng modal"
          style={{ position: 'absolute', top: 8, right: 8 }}
        >
          ✕
        </button>
      </div>
    </>,
    modalRoot
  );
}
```

### Setup HTML — Cần Có Container

```html
<!-- index.html -->
<body>
  <div id="root"></div>
  <!-- Container dành riêng cho modals -->
  <div id="modal-root"></div>
</body>
```

---

## 3. Focus Trap — Bẫy Focus Đầy Đủ

```tsx
function useFocusTrap(containerRef: RefObject<HTMLElement>, isActive: boolean) {
  useEffect(() => {
    if (!isActive || !containerRef.current) return;

    const container = containerRef.current;

    // Tìm tất cả elements có thể focus
    const focusableSelectors = [
      'a[href]', 'button:not([disabled])', 'input:not([disabled])',
      'select:not([disabled])', 'textarea:not([disabled])',
      '[tabindex]:not([tabindex="-1"])',
    ].join(', ');

    const focusableElements = container.querySelectorAll<HTMLElement>(focusableSelectors);
    const firstElement = focusableElements[0];
    const lastElement = focusableElements[focusableElements.length - 1];

    // Focus element đầu tiên khi trap active
    firstElement?.focus();

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        // Shift+Tab → focus backward
        if (document.activeElement === firstElement) {
          e.preventDefault();
          lastElement?.focus();
        }
      } else {
        // Tab → focus forward
        if (document.activeElement === lastElement) {
          e.preventDefault();
          firstElement?.focus();
        }
      }
    };

    container.addEventListener('keydown', handleKeyDown);
    return () => container.removeEventListener('keydown', handleKeyDown);
  }, [containerRef, isActive]);
}

// Sử dụng trong Modal
function AccessibleModal({ isOpen, onClose, children }: ModalProps) {
  const dialogRef = useRef<HTMLDivElement>(null);
  useFocusTrap(dialogRef, isOpen);

  // Lưu element đang focus trước khi mở modal
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement as HTMLElement;
    } else {
      // Khôi phục focus khi đóng modal
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return createPortal(
    <div ref={dialogRef} role="dialog" aria-modal="true">
      {children}
      <button onClick={onClose}>Đóng</button>
    </div>,
    document.body
  );
}
```

---

## 4. Tooltip Với Portal

```tsx
interface TooltipProps {
  content: string;
  children: React.ReactElement;
}

function Tooltip({ content, children }: TooltipProps) {
  const [isVisible, setIsVisible] = useState(false);
  const [position, setPosition] = useState({ top: 0, left: 0 });
  const triggerRef = useRef<HTMLElement>(null);

  const updatePosition = useCallback(() => {
    if (!triggerRef.current) return;
    const rect = triggerRef.current.getBoundingClientRect();
    setPosition({
      top: rect.bottom + window.scrollY + 8,
      left: rect.left + window.scrollX + rect.width / 2,
    });
  }, []);

  const showTooltip = () => {
    updatePosition();
    setIsVisible(true);
  };

  const hideTooltip = () => setIsVisible(false);

  return (
    <>
      {React.cloneElement(children, {
        ref: triggerRef,
        onMouseEnter: showTooltip,
        onMouseLeave: hideTooltip,
        onFocus: showTooltip,
        onBlur: hideTooltip,
        'aria-describedby': isVisible ? 'tooltip' : undefined,
      })}

      {isVisible && createPortal(
        <div
          id="tooltip"
          role="tooltip"
          style={{
            position: 'absolute',
            top: position.top,
            left: position.left,
            transform: 'translateX(-50%)',
            background: '#333',
            color: 'white',
            padding: '4px 8px',
            borderRadius: '4px',
            fontSize: '14px',
            zIndex: 9999,
            pointerEvents: 'none',
          }}
        >
          {content}
        </div>,
        document.body
      )}
    </>
  );
}

// Sử dụng
<Tooltip content="Xem thêm thông tin về tài khoản">
  <button>Tài khoản</button>
</Tooltip>
```

---

## PHẦN 2: ERROR BOUNDARIES

## 5. Error Boundary — Cơ Bản

```tsx
interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ReactNode | ((error: Error, reset: () => void) => React.ReactNode);
  onError?: (error: Error, info: React.ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = { hasError: false, error: null };

  // Được gọi khi có lỗi trong render/lifecycle của component con
  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  // Được gọi sau khi lỗi được bắt — nơi để log
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    this.props.onError?.(error, info);

    // Log to monitoring services
    console.error('Error Boundary caught:', {
      error: error.message,
      stack: error.stack,
      componentStack: info.componentStack,
    });
  }

  reset = () => this.setState({ hasError: false, error: null });

  render() {
    if (this.state.hasError && this.state.error) {
      const { fallback } = this.props;

      if (typeof fallback === 'function') {
        return fallback(this.state.error, this.reset);
      }

      return fallback ?? (
        <div className="error-boundary-fallback">
          <h2>Đã xảy ra lỗi không mong muốn</h2>
          <p>{this.state.error.message}</p>
          <button onClick={this.reset}>Thử lại</button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

---

## 6. Sử Dụng Error Boundary

```tsx
// Wrap toàn bộ app — lỗi nghiêm trọng
function App() {
  return (
    <ErrorBoundary
      fallback={<AppCrashPage />}
      onError={(err) => monitoringService.captureException(err)}
    >
      <Router>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Router>
    </ErrorBoundary>
  );
}

// Wrap từng section — lỗi cục bộ, không crash toàn app
function Dashboard() {
  return (
    <div className="dashboard">
      <ErrorBoundary fallback={<p>Widget thống kê không khả dụng.</p>}>
        <StatsWidget />
      </ErrorBoundary>

      <ErrorBoundary fallback={<p>Biểu đồ không thể tải.</p>}>
        <RevenueChart />
      </ErrorBoundary>

      <ErrorBoundary
        fallback={(error, reset) => (
          <div>
            <p>Lỗi: {error.message}</p>
            <button onClick={reset}>Tải lại widget</button>
          </div>
        )}
      >
        <ActivityFeed />
      </ErrorBoundary>
    </div>
  );
}
```

---

## 7. Error Boundary Với React Query — Reset Tự Động

```tsx
// Kết hợp với React Query để reset khi query thành công
function QueryErrorBoundary({ children }: { children: ReactNode }) {
  const queryClient = useQueryClient();

  return (
    <ErrorBoundary
      fallback={(error, reset) => (
        <div>
          <p>Lỗi tải dữ liệu: {error.message}</p>
          <button
            onClick={() => {
              // Reset React Query cache trước khi reset boundary
              queryClient.clear();
              reset();
            }}
          >
            Tải lại
          </button>
        </div>
      )}
    >
      {children}
    </ErrorBoundary>
  );
}
```

---

## 8. useErrorBoundary Hook (React 18+)

React 18 cung cấp API mới để throw error từ async code vào Error Boundary:

```tsx
import { useErrorBoundary } from 'react-error-boundary'; // Thư viện phổ biến

function UserProfile({ userId }: { userId: string }) {
  const { showBoundary } = useErrorBoundary();

  const handleAsyncError = async () => {
    try {
      await deleteUser(userId);
    } catch (error) {
      // Throw lỗi từ event handler vào Error Boundary gần nhất
      showBoundary(error);
    }
  };

  return (
    <div>
      <h2>Profile</h2>
      <button onClick={handleAsyncError}>Xóa tài khoản</button>
    </div>
  );
}
```

---

## 9. Kết Hợp Error Boundary + Suspense

```tsx
// Suspense + Error Boundary là cặp đôi hoàn hảo
function AsyncUserCard({ userId }: { userId: string }) {
  return (
    <ErrorBoundary fallback={<p>Không thể tải thông tin người dùng.</p>}>
      <Suspense fallback={<Skeleton />}>
        {/* UserData dùng use() hook hoặc TanStack Query với suspense: true */}
        <UserData userId={userId} />
      </Suspense>
    </ErrorBoundary>
  );
}

// Component dùng Suspense-enabled data fetching
function UserData({ userId }: { userId: string }) {
  // Với TanStack Query v5 — suspense mode
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // Không cần kiểm tra loading/error ở đây
  // Suspense xử lý loading, Error Boundary xử lý error
  return <UserCard user={data} />;
}
```

---

## 10. Chiến Lược Error Boundary Ở Cấp Ứng Dụng

```
Kiến trúc Error Boundaries nhiều tầng:

Level 1 — App Level (Ứng Dụng):
  <ErrorBoundary fallback={<FullPageError />}>
    → Bắt lỗi nghiêm trọng, toàn bộ app crash

Level 2 — Route Level (Tuyến Đường):
  <ErrorBoundary fallback={<PageError />}>
    → Lỗi trong một route/page cụ thể

Level 3 — Feature Level (Tính Năng):
  <ErrorBoundary fallback={<WidgetError />}>
    → Lỗi trong widget/section nhỏ

Level 4 — Component Level (Component):
  <ErrorBoundary fallback={<ComponentError />}>
    → Lỗi nhỏ, không ảnh hưởng page

Nguyên tắc: Granularity = Resilience
(Chi tiết hóa = Khả Năng Chịu Lỗi)
Boundary càng nhỏ → App càng ít bị ảnh hưởng khi có lỗi
```

---

## 11. Những Gì Error Boundary KHÔNG Bắt Được

```
❌ Error Boundary KHÔNG bắt:
1. Lỗi trong event handlers (onClick, onChange, ...)
   → Dùng try/catch bình thường
   
2. Lỗi trong async code (setTimeout, fetch, ...)
   → Dùng try/catch + setState để hiển thị lỗi
   
3. Lỗi trong server-side rendering (SSR)
   → Xử lý ở tầng server

4. Lỗi trong chính Error Boundary component
   → Error Boundary cha sẽ bắt

✅ Error Boundary BẮT được:
1. Lỗi trong render method
2. Lỗi trong lifecycle methods
3. Lỗi trong constructor của component con
4. Lỗi trong các components con khi render
```

---

## 📝 Checklist Tự Đánh Giá

- [ ] Giải thích được khi nào cần Portal và lý do kỹ thuật
- [ ] Implement Modal với Portal, focus trap, Escape key
- [ ] Viết Error Boundary class component với `getDerivedStateFromError` và `componentDidCatch`
- [ ] Phân biệt những loại lỗi Error Boundary bắt được và không bắt được
- [ ] Thiết kế chiến lược Error Boundary nhiều tầng cho ứng dụng thực tế

---

## 🔗 Điều Hướng

- **Trước đó:** [4-custom-hooks-patterns.md](./4-custom-hooks-patterns.md) — Custom Hooks nâng cao
- **Tiếp theo:** [6-accessibility.md](./6-accessibility.md) — Accessibility (A11y)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
