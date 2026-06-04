# TypeScript với React — Typing Components, Hooks, và API

> TypeScript (Kiểu Dữ Liệu Tĩnh cho JavaScript) giúp phát hiện lỗi tại thời điểm biên dịch thay vì runtime, cải thiện developer experience với autocomplete, và làm cho refactoring an toàn hơn trong dự án lớn

---

## 1. Tại Sao Cần TypeScript Trong React?

```typescript
// JavaScript — Lỗi chỉ xuất hiện lúc runtime
function greet(user) {
  return `Xin chào, ${user.name.toUpperCase()}`; // Crash nếu user = null
}

// TypeScript — Lỗi được phát hiện ngay khi viết code
function greet(user: { name: string } | null) {
  return `Xin chào, ${user?.name.toUpperCase() ?? 'Khách'}`; // Buộc xử lý null
}
```

**Lợi ích thực tế:**
- Autocomplete chính xác trong IDE
- Refactoring an toàn (đổi tên prop → tất cả nơi dùng được cập nhật/báo lỗi)
- Tài liệu hóa tự động qua types
- Bắt bugs sớm — trước khi ship lên production

---

## 2. Cài Đặt TypeScript với React

```bash
# Vite + React + TypeScript (khuyến nghị)
npm create vite@latest my-app -- --template react-ts

# Next.js (TypeScript mặc định)
npx create-next-app@latest my-app --typescript

# Thêm TypeScript vào project React có sẵn
npm install -D typescript @types/react @types/react-dom
npx tsc --init  # Tạo tsconfig.json
```

### `tsconfig.json` Phổ Biến Cho React

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    // Module resolution
    "moduleResolution": "bundler",  // Dùng cho Vite
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,                 // Để Vite/Next.js handle emit

    // JSX
    "jsx": "react-jsx",             // Không cần import React nữa

    // Strict mode — Bật tất cả type checking chặt chẽ
    "strict": true,
    "noUnusedLocals": true,         // Lỗi nếu có biến không dùng
    "noUnusedParameters": true,     // Lỗi nếu có param không dùng
    "noFallthroughCasesInSwitch": true,

    // Path aliases
    "paths": {
      "@/*": ["./src/*"]
    },
    "baseUrl": "."
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

---

## 3. Typing Components — Định Kiểu Components

### Functional Component — Component Hàm

```tsx
// Cách 1: Annotate return type (khuyến nghị)
function Greeting({ name, age }: { name: string; age?: number }) {
  return <p>Xin chào {name}{age ? `, ${age} tuổi` : ''}</p>;
}

// Cách 2: Tách interface riêng (dễ đọc hơn cho component phức tạp)
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary' | 'danger'; // Union type — Kiểu hợp nhất
  disabled?: boolean;
  children?: React.ReactNode;  // Bất kỳ React content nào
}

function Button({ label, onClick, variant = 'primary', disabled = false }: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn-${variant}`}
    >
      {label}
    </button>
  );
}
```

### Tránh Dùng `React.FC`

```tsx
// ❌ Không khuyến nghị — React.FC có vài vấn đề
const MyComponent: React.FC<Props> = ({ name }) => <div>{name}</div>;

// ✅ Khuyến nghị — Annotate trực tiếp, đơn giản hơn
function MyComponent({ name }: Props) {
  return <div>{name}</div>;
}
```

---

## 4. Typing Props — Định Kiểu Props

### Children Props

```tsx
interface CardProps {
  title: string;
  children: React.ReactNode;     // Mọi thứ React có thể render
}

interface IconProps {
  children: React.ReactElement;  // Chỉ React element (không phải string/number)
}

interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;  // Render prop
}
```

### Event Handlers — Xử Lý Sự Kiện

```tsx
interface FormProps {
  // Các kiểu event handler phổ biến
  onClick: React.MouseEventHandler<HTMLButtonElement>;
  onChange: React.ChangeEventHandler<HTMLInputElement>;
  onSubmit: React.FormEventHandler<HTMLFormElement>;
  onKeyDown: React.KeyboardEventHandler<HTMLInputElement>;

  // Hoặc inline (ít verbose hơn)
  onClick: (event: React.MouseEvent<HTMLButtonElement>) => void;
}

function SearchInput({ onChange }: { onChange: (value: string) => void }) {
  return (
    <input
      type="text"
      onChange={(e) => onChange(e.target.value)} // e được infer tự động
    />
  );
}
```

### Component Polymorphism — Component Đa Hình

```tsx
// Component có thể render thành nhiều HTML element khác nhau
interface TextProps<T extends React.ElementType = 'span'> {
  as?: T;
  children: React.ReactNode;
  className?: string;
}

type PolymorphicProps<T extends React.ElementType> = TextProps<T> &
  Omit<React.ComponentPropsWithRef<T>, keyof TextProps<T>>;

function Text<T extends React.ElementType = 'span'>({
  as,
  children,
  ...rest
}: PolymorphicProps<T>) {
  const Component = as ?? 'span';
  return <Component {...rest}>{children}</Component>;
}

// Sử dụng:
<Text>Span</Text>
<Text as="h1">Heading</Text>
<Text as="a" href="/about">Link</Text>  // href được type-check!
```

---

## 5. Typing Hooks — Định Kiểu Hooks

### useState

```tsx
// TypeScript infer type từ initial value
const [count, setCount] = useState(0);           // number
const [name, setName] = useState('');            // string
const [user, setUser] = useState<User | null>(null); // Cần type annotation

// Interface phức tạp
interface FormState {
  name: string;
  email: string;
  errors: Partial<Record<'name' | 'email', string>>;
}

const [form, setForm] = useState<FormState>({
  name: '',
  email: '',
  errors: {},
});
```

### useReducer

```tsx
// Định nghĩa action types chặt chẽ với discriminated union
type CartAction =
  | { type: 'ADD_ITEM'; payload: Product }
  | { type: 'REMOVE_ITEM'; payload: { id: string } }
  | { type: 'UPDATE_QUANTITY'; payload: { id: string; quantity: number } }
  | { type: 'CLEAR_CART' };

interface CartState {
  items: CartItem[];
  total: number;
}

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM':
      // action.payload là Product — TypeScript biết điều này
      return { ...state, items: [...state.items, { ...action.payload, quantity: 1 }] };
    case 'CLEAR_CART':
      // action.payload không tồn tại ở case này — TypeScript báo lỗi nếu truy cập
      return { items: [], total: 0 };
    default:
      return state;
  }
}

const [cart, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
dispatch({ type: 'ADD_ITEM', payload: product }); // Type-safe!
```

### useRef

```tsx
// Ref cho DOM element
const inputRef = useRef<HTMLInputElement>(null);
const divRef = useRef<HTMLDivElement>(null);

// Ref cho mutable value (không trigger re-render)
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
const previousValueRef = useRef<string>('');

// Dùng:
inputRef.current?.focus(); // Optional chaining vì có thể null
```

### useContext

```tsx
interface AuthContextType {
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
  isLoading: boolean;
}

// Dùng null làm initial value + type assertion để tránh check null mỗi lần dùng
const AuthContext = createContext<AuthContextType | null>(null);

// Custom hook với validation
function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth phải được dùng trong AuthProvider');
  }
  return context; // Đã loại bỏ null — return type là AuthContextType
}
```

### Custom Hooks

```tsx
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const res = await fetch(url);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const json: T = await res.json();
      setData(json);
    } catch (err) {
      setError(err instanceof Error ? err : new Error('Lỗi không xác định'));
    } finally {
      setLoading(false);
    }
  }, [url]);

  useEffect(() => { fetchData(); }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
}

// Sử dụng với generic type
const { data: users, loading } = useFetch<User[]>('/api/users');
```

---

## 6. Utility Types Hữu Ích Trong React

### `React.ComponentProps` và Variants

```tsx
// Lấy tất cả props của một HTML element
type ButtonProps = React.ComponentProps<'button'>;
type InputProps = React.ComponentProps<'input'>;

// Mở rộng props của HTML element
interface MyButtonProps extends React.ComponentProps<'button'> {
  variant?: 'primary' | 'secondary';
  loading?: boolean;
}

// Lấy props của component khác
type CardProps = React.ComponentProps<typeof Card>;
```

### TypeScript Utility Types Hay Dùng

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
  role: 'admin' | 'user' | 'moderator';
}

// Partial — Tất cả fields optional
type UpdateUserDto = Partial<User>; // { id?: string; name?: string; ... }

// Required — Tất cả fields required
type StrictUser = Required<User>;

// Pick — Chọn một số fields
type UserPreview = Pick<User, 'id' | 'name'>;

// Omit — Bỏ một số fields
type CreateUserDto = Omit<User, 'id'>; // Bỏ id vì server tự tạo

// Readonly — Không thể mutate
type ReadonlyUser = Readonly<User>;

// Record — Map key → value
type UsersByRole = Record<User['role'], User[]>;

// ReturnType — Lấy kiểu trả về của function
type LoaderData = Awaited<ReturnType<typeof loader>>;
```

---

## 7. Generic Components — Component Tổng Quát

```tsx
// Component Table generic — hoạt động với mọi kiểu data
interface TableProps<T> {
  data: T[];
  columns: {
    key: keyof T;        // keyof — Lấy keys của type T
    header: string;
    render?: (value: T[keyof T], row: T) => React.ReactNode;
  }[];
  keyExtractor: (item: T) => string;
}

function Table<T>({ data, columns, keyExtractor }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map(col => (
            <th key={String(col.key)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map(row => (
          <tr key={keyExtractor(row)}>
            {columns.map(col => (
              <td key={String(col.key)}>
                {col.render
                  ? col.render(row[col.key], row)
                  : String(row[col.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Sử dụng — TypeScript infer T = User
<Table<User>
  data={users}
  keyExtractor={u => u.id}
  columns={[
    { key: 'name', header: 'Tên' },
    { key: 'email', header: 'Email' },
    { key: 'role', header: 'Vai Trò', render: (role) => <Badge>{role}</Badge> },
  ]}
/>
```

---

## 8. Typing API Data — Định Kiểu Dữ Liệu API

### Zod — Runtime Validation + Type Inference

```bash
npm install zod
```

```typescript
import { z } from 'zod';

// Schema validation — Kiểm tra dữ liệu từ API
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(150).optional(),
  role: z.enum(['admin', 'user', 'moderator']),
  createdAt: z.string().datetime(),
});

// Infer TypeScript type từ schema — không cần khai báo type riêng!
type User = z.infer<typeof UserSchema>;

// Parse và validate API response
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  const raw = await res.json();

  // Parse kiểm tra và throw nếu data không đúng schema
  return UserSchema.parse(raw);
}
```

### Khi Không Dùng Zod

```typescript
// Type assertion — Chỉ dùng khi tin tưởng API trả đúng kiểu
const user = await res.json() as User; // ⚠️ Không safe!

// Safer: Type guard — Hàm kiểm tra kiểu
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value &&
    'email' in value
  );
}

const data = await res.json();
if (isUser(data)) {
  // data là User trong block này
  console.log(data.name);
}
```

---

## 9. Common TypeScript Errors Trong React

### Error: Property 'X' does not exist on type 'EventTarget'

```tsx
// ❌ Lỗi: EventTarget không có .value
const handleChange = (e: React.ChangeEvent) => {
  console.log(e.target.value); // Error!
};

// ✅ Xác định cụ thể element type
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  console.log(e.target.value); // OK!
};
```

### Error: Type 'X | undefined' không gán được vào 'X'

```tsx
// ❌
function greet(user: User | undefined) {
  return user.name; // Error: user có thể undefined
}

// ✅ Optional chaining
function greet(user: User | undefined) {
  return user?.name ?? 'Khách';
}

// ✅ Hoặc type guard
function greet(user: User | undefined) {
  if (!user) return 'Khách';
  return user.name; // TypeScript biết user là User tại đây
}
```

### Error: Argument of type 'string | null' không gán được

```tsx
// ❌
const value = localStorage.getItem('key'); // string | null
doSomething(value); // Hàm expect string, không phải string | null

// ✅ Non-null assertion (chỉ khi chắc chắn không null)
doSomething(value!);

// ✅ Tốt hơn: Xử lý null
if (value !== null) {
  doSomething(value);
}

// ✅ Hoặc fallback
doSomething(value ?? 'default');
```

---

## 10. Câu Hỏi Phỏng Vấn

### Q: `interface` vs `type` trong TypeScript — dùng cái nào?

**A:**

| | `interface` | `type` |
| - | ----------- | ------ |
| **Declaration merging** | ✅ Có | ❌ Không |
| **Extends** | `extends` keyword | `&` intersection |
| **Union types** | ❌ Không | ✅ `type A = B \| C` |
| **Primitive aliases** | ❌ | ✅ `type ID = string` |
| **Tuples** | ❌ Khó | ✅ `type Pair = [string, number]` |

**Quy tắc thực tế:**
- Dùng `interface` cho object shapes của component props và API data
- Dùng `type` cho union types, function types, utility types

### Q: `unknown` vs `any` — khác gì nhau?

**A:**
- `any` — Tắt type checking hoàn toàn, không nên dùng
- `unknown` — "Tôi không biết kiểu này" nhưng buộc phải kiểm tra trước khi dùng

```typescript
function process(value: unknown) {
  value.toUpperCase(); // Error! unknown phải check trước

  if (typeof value === 'string') {
    value.toUpperCase(); // OK — đã narrowing
  }
}
```

### Q: Tại sao `React.FC` không được khuyến nghị?

**A:** `React.FC` có một số vấn đề:
1. Luôn thêm `children` prop vào type (trước React 18 — gây nhầm lẫn)
2. Không hỗ trợ generic components tốt
3. Không cần thiết — TypeScript tự infer return type

Cách khuyến nghị là annotate props trực tiếp và để TypeScript infer phần còn lại.

---

## ✅ Checklist

- [ ] Setup TypeScript strict mode trong `tsconfig.json`
- [ ] Type component props bằng interface riêng
- [ ] Type hooks phức tạp (useReducer với discriminated union)
- [ ] Viết generic component (ví dụ: `List<T>`, `Select<T>`)
- [ ] Dùng Zod để validate API responses
- [ ] Biết cách dùng utility types: `Partial`, `Pick`, `Omit`, `Record`
- [ ] Xử lý `null` và `undefined` không dùng `!` assertion

---

**Tài Liệu Tham Khảo:**
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
- [Zod Docs](https://zod.dev/)
- [Total TypeScript by Matt Pocock](https://www.totaltypescript.com/)
