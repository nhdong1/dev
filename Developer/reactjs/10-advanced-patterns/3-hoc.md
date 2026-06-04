# 3 — Higher-Order Components — HOC (Component Bậc Cao)

> **Higher-Order Component — HOC** (Component Bậc Cao) là hàm nhận một component và trả về một component mới có thêm chức năng. Đây là pattern chia sẻ **cross-cutting concerns** (mối quan tâm xuyên cắt) như authentication, logging, error handling mà không sửa component gốc.

---

## 🎯 Mục Tiêu Học Tập

Sau khi học xong file này, bạn có thể:

- [ ] Giải thích HOC pattern và phân biệt với Render Props, Custom Hooks
- [ ] Viết HOC đúng chuẩn: **forwarding refs**, **hoisting statics**, **display name**
- [ ] Compose (kết hợp) nhiều HOC với `compose` utility
- [ ] Nhận ra HOC trong các thư viện: `connect()`, `withRouter()`, `memo()`
- [ ] Biết trade-offs và khi nào nên chuyển sang Custom Hooks

---

## 1. Khái Niệm Cơ Bản

### HOC Là Gì?

```tsx
// HOC là hàm: Component → EnhancedComponent
function withLogging<P extends object>(WrappedComponent: React.ComponentType<P>) {
  // Trả về component mới
  function WithLogging(props: P) {
    useEffect(() => {
      console.log(`${WrappedComponent.displayName || WrappedComponent.name} mounted`);
      return () => {
        console.log(`${WrappedComponent.displayName || WrappedComponent.name} unmounted`);
      };
    }, []);

    // Render component gốc, truyền tất cả props
    return <WrappedComponent {...props} />;
  }

  // Đặt displayName để dễ debug trong React DevTools
  WithLogging.displayName = `WithLogging(${WrappedComponent.displayName || WrappedComponent.name})`;

  return WithLogging;
}

// Sử dụng
const UserCardWithLogging = withLogging(UserCard);
const ProductListWithLogging = withLogging(ProductList);
```

---

## 2. HOC Thực Tế — Authentication Guard

```tsx
interface WithAuthProps {
  // Props mà HOC inject vào — người dùng component không cần truyền
  currentUser: User;
}

function withAuth<P extends WithAuthProps>(
  WrappedComponent: React.ComponentType<P>
) {
  function WithAuth(props: Omit<P, keyof WithAuthProps>) {
    const { user, isLoading } = useAuth();
    const navigate = useNavigate();

    if (isLoading) {
      return <FullPageSpinner />;
    }

    if (!user) {
      // Redirect đến trang đăng nhập
      navigate('/login', { replace: true });
      return null;
    }

    // Inject user vào props và render component gốc
    return <WrappedComponent {...(props as P)} currentUser={user} />;
  }

  WithAuth.displayName = `WithAuth(${WrappedComponent.displayName || WrappedComponent.name})`;
  return WithAuth;
}

// Component cần authentication
interface DashboardProps extends WithAuthProps {
  title: string;
}

function Dashboard({ currentUser, title }: DashboardProps) {
  return (
    <div>
      <h1>{title}</h1>
      <p>Xin chào, {currentUser.name}!</p>
    </div>
  );
}

// Áp dụng HOC
const ProtectedDashboard = withAuth(Dashboard);

// Sử dụng — không cần truyền currentUser
<ProtectedDashboard title="Bảng Điều Khiển" />
```

---

## 3. HOC Thực Tế — Loading State

```tsx
interface WithLoadingProps {
  isLoading: boolean;
}

function withLoading<P extends WithLoadingProps>(
  WrappedComponent: React.ComponentType<P>,
  LoadingComponent: React.ComponentType = DefaultSpinner
) {
  function WithLoading({ isLoading, ...rest }: P) {
    if (isLoading) {
      return <LoadingComponent />;
    }
    return <WrappedComponent {...(rest as P)} />;
  }

  WithLoading.displayName = `WithLoading(${WrappedComponent.displayName || WrappedComponent.name})`;
  return WithLoading;
}

// Sử dụng
interface UserListProps extends WithLoadingProps {
  users: User[];
}

function UserList({ users }: UserListProps) {
  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}

const UserListWithLoading = withLoading(UserList);

// Trong parent component
<UserListWithLoading isLoading={isLoading} users={users} />
```

---

## 4. HOC Thực Tế — Error Boundary HOC

```tsx
interface WithErrorBoundaryOptions {
  fallback?: React.ReactNode;
  onError?: (error: Error, info: React.ErrorInfo) => void;
}

function withErrorBoundary<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  options: WithErrorBoundaryOptions = {}
) {
  const { fallback, onError } = options;

  class WithErrorBoundary extends React.Component<P, { hasError: boolean; error: Error | null }> {
    static displayName = `WithErrorBoundary(${WrappedComponent.displayName || WrappedComponent.name})`;

    state = { hasError: false, error: null };

    static getDerivedStateFromError(error: Error) {
      return { hasError: true, error };
    }

    componentDidCatch(error: Error, info: React.ErrorInfo) {
      onError?.(error, info);
      // Gửi lỗi đến monitoring service (Sentry, Datadog...)
      console.error('Component error:', error, info);
    }

    render() {
      if (this.state.hasError) {
        return fallback ?? (
          <div className="error-fallback">
            <h3>Đã xảy ra lỗi</h3>
            <p>{this.state.error?.message}</p>
            <button onClick={() => this.setState({ hasError: false, error: null })}>
              Thử lại
            </button>
          </div>
        );
      }

      return <WrappedComponent {...this.props} />;
    }
  }

  return WithErrorBoundary;
}

// Sử dụng
const SafeProductCard = withErrorBoundary(ProductCard, {
  fallback: <p>Không thể tải thẻ sản phẩm.</p>,
  onError: (err) => Sentry.captureException(err),
});
```

---

## 5. Compose Nhiều HOC

### Vấn Đề: HOC Lồng Nhau Khó Đọc

```tsx
// ❌ Wrapper hell — khó đọc
const EnhancedUserCard = withAuth(
  withLogging(
    withErrorBoundary(
      withPermissions(UserCard)
    )
  )
);
```

### Giải Pháp: Compose Utility

```tsx
// compose: áp dụng từ phải sang trái (right-to-left)
function compose<T>(...fns: Array<(arg: T) => T>) {
  return (x: T) => fns.reduceRight((acc, fn) => fn(acc), x);
}

// ✅ Dễ đọc, dễ thêm/bớt HOC
const enhance = compose(
  withAuth,
  withLogging,
  withErrorBoundary,
  withPermissions
);

const EnhancedUserCard = enhance(UserCard);

// Hoặc dùng pipe (left-to-right) nếu thích
function pipe<T>(...fns: Array<(arg: T) => T>) {
  return (x: T) => fns.reduce((acc, fn) => fn(acc), x);
}

const EnhancedUserCard2 = pipe(
  withPermissions,
  withErrorBoundary,
  withLogging,
  withAuth
)(UserCard);
```

### Redux `connect()` — HOC Kinh Điển

```tsx
// Redux connect() là HOC pattern
import { connect } from 'react-redux';

// mapStateToProps: lấy dữ liệu từ store
const mapStateToProps = (state: RootState) => ({
  user: state.auth.user,
  isLoading: state.auth.loading,
});

// mapDispatchToProps: bind actions
const mapDispatchToProps = {
  logout: authActions.logout,
  updateProfile: authActions.updateProfile,
};

// connect() trả về HOC
const EnhancedProfile = connect(mapStateToProps, mapDispatchToProps)(ProfilePage);
```

---

## 6. Xử Lý Refs — forwardRef Trong HOC

### Vấn Đề: HOC Nuốt Mất Ref

```tsx
// ❌ Ref không đến được WrappedComponent
const InputWithLogging = withLogging(Input);
const ref = useRef<HTMLInputElement>(null);

// ref.current sẽ là null hoặc trỏ vào WithLogging, không phải Input
<InputWithLogging ref={ref} />
```

### Giải Pháp: forwardRef

```tsx
function withLogging<P extends object, T = unknown>(
  WrappedComponent: React.ForwardRefExoticComponent<P & React.RefAttributes<T>>
    | React.ComponentType<P>
) {
  const WithLogging = React.forwardRef<T, P>((props, ref) => {
    useEffect(() => {
      console.log('Component mounted:', WrappedComponent.displayName);
    }, []);

    return <WrappedComponent {...props} ref={ref as any} />;
  });

  WithLogging.displayName = `WithLogging(${
    (WrappedComponent as any).displayName || WrappedComponent.name
  })`;

  return WithLogging;
}

// Sử dụng đúng cách
const LoggedInput = withLogging(React.forwardRef<HTMLInputElement, InputProps>(
  ({ ...props }, ref) => <input ref={ref} {...props} />
));

const ref = useRef<HTMLInputElement>(null);
<LoggedInput ref={ref} placeholder="Type here..." />;
// ref.current bây giờ trỏ đúng vào <input>
```

---

## 7. Hoisting Static Methods — Giữ Lại Static Properties

```tsx
import hoistNonReactStatics from 'hoist-non-react-statics';

// Nếu WrappedComponent có static methods
class UserCard extends React.Component<UserCardProps> {
  // Static method — dùng như utility
  static formatName(user: User) {
    return `${user.firstName} ${user.lastName}`;
  }

  render() { /* ... */ }
}

function withLogging<P extends object>(WrappedComponent: React.ComponentType<P>) {
  function WithLogging(props: P) {
    return <WrappedComponent {...props} />;
  }

  // ❌ Không có hoisting — UserCard.formatName bị mất
  // const LoggedUserCard = withLogging(UserCard);
  // LoggedUserCard.formatName → undefined!

  // ✅ Hoisting — copy tất cả static methods
  hoistNonReactStatics(WithLogging, WrappedComponent);

  WithLogging.displayName = `WithLogging(${WrappedComponent.displayName || WrappedComponent.name})`;
  return WithLogging;
}

const LoggedUserCard = withLogging(UserCard);
LoggedUserCard.formatName(user); // ✅ Hoạt động
```

---

## 8. HOC vs Custom Hooks vs Render Props

| Tiêu Chí | HOC | Custom Hook | Render Props |
| -------- | --- | ----------- | ------------ |
| **Syntax** (Cú pháp) | Wrap component | Hook call | Function prop |
| **Composability** (Kết hợp) | compose() | Tự nhiên | Callback hell |
| **Props namespace** | Có thể clash | Không | Rõ ràng |
| **Debug** | Nhiều wrappers | Sạch | Trung bình |
| **Class components** | ✅ | ❌ | ✅ |
| **Tree shake** (Loại bỏ code không dùng) | Khó | Dễ | Trung bình |

### Decision Tree — Khi Nào Dùng HOC

```
Bạn cần thêm behavior vào component?
│
├─ Behavior chỉ liên quan đến logic/state? → Custom Hook ✅
│
├─ Behavior cần wrap JSX/render? 
│   ├─ Cần dùng với class components? → HOC ✅
│   └─ Chỉ functional components?     → Custom Hook ✅
│
├─ Cross-cutting concerns trong codebase lớn?
│   ├─ Đang dùng Redux?               → connect() HOC là chuẩn
│   └─ Project mới từ đầu?            → Custom Hook + Context
│
└─ Đang maintain legacy code?         → HOC (không phá vỡ existing code)
```

---

## 9. HOC Trong Các Thư Viện Phổ Biến

```tsx
// React.memo() — HOC để memoize rendering
const MemoizedCard = React.memo(ProductCard);
const MemoizedCardCustom = React.memo(ProductCard, (prevProps, nextProps) => {
  return prevProps.productId === nextProps.productId; // true = bỏ qua re-render
});

// React Router v5 — withRouter()
import { withRouter } from 'react-router-dom';
const ComponentWithRouter = withRouter(MyComponent);

// Styled Components — withTheme()
import { withTheme } from 'styled-components';
const ThemedButton = withTheme(Button);

// React Intl — injectIntl()
import { injectIntl } from 'react-intl';
const LocalizedComponent = injectIntl(MyComponent);
```

---

## 10. Vấn Đề Thường Gặp Với HOC

### Props Collision — Xung Đột Tên Props

```tsx
// ❌ HOC inject prop 'user', component cũng có prop 'user'
function withUser<P extends { user?: User }>(Component: React.ComponentType<P>) {
  return function WithUser(props: P) {
    const user = useCurrentUser();
    // Ghi đè user được truyền từ bên ngoài
    return <Component {...props} user={user} />;
  };
}

// ✅ Dùng namespace để tránh collision
function withCurrentUser<P extends object>(Component: React.ComponentType<P>) {
  return function WithCurrentUser(props: P) {
    const currentUser = useCurrentUser();
    return <Component {...props} currentUser={currentUser} />;
  };
}
```

### HOC Order Matters — Thứ Tự HOC Quan Trọng

```tsx
// Thứ tự HOC ảnh hưởng đến kết quả
// compose(f, g, h)(x) = f(g(h(x)))

// ❌ Sai thứ tự — withAuth cần chạy TRƯỚC withLogging
const Wrong = withLogging(withAuth(UserCard));
// Nếu chưa auth → redirect, không có gì để log

// ✅ Đúng thứ tự
const Correct = withAuth(withLogging(UserCard));
// Auth check trước → nếu OK thì logging mới hoạt động
```

---

## 📝 Checklist Tự Đánh Giá

- [ ] Viết được HOC cơ bản với TypeScript generics
- [ ] Xử lý ref forwarding trong HOC
- [ ] Biết `hoist-non-react-statics` và khi nào cần
- [ ] Compose nhiều HOC với `compose` function
- [ ] Phân biệt HOC, Custom Hook, Render Props và chọn đúng

---

## 🔗 Điều Hướng

- **Trước đó:** [2-render-props.md](./2-render-props.md) — Render Props
- **Tiếp theo:** [4-custom-hooks-patterns.md](./4-custom-hooks-patterns.md) — Custom Hooks nâng cao
- **Chỉ mục:** [INDEX.md](../INDEX.md)
