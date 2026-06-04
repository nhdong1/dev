# Conditional & List Rendering — Render Có Điều Kiện và Danh Sách

> React không dùng directive (chỉ thị) kiểu `v-if`, `v-for` như Vue hay `*ngIf`, `*ngFor` như Angular. Thay vào đó, toàn bộ logic render là JavaScript thuần — linh hoạt nhưng cần nắm rõ các pattern.

---

## 📌 Mục Lục

1. [Conditional Rendering — Render Có Điều Kiện](#1-conditional-rendering)
2. [List Rendering — Render Danh Sách](#2-list-rendering)
3. [Key — Khóa Định Danh](#3-key)
4. [Kết Hợp Conditional và List Rendering](#4-kết-hợp)
5. [Câu hỏi phỏng vấn](#5-câu-hỏi-phỏng-vấn)

---

## 1. Conditional Rendering — Render Có Điều Kiện

### Cách 1: `if/else` Statement Ngoài JSX

Phù hợp khi logic phức tạp, nhiều trường hợp:

```jsx
function UserStatus({ user }) {
  if (!user) {
    return <p>Chưa đăng nhập</p>;
  }

  if (user.isBanned) {
    return <p>Tài khoản đã bị khóa</p>;
  }

  if (user.role === "admin") {
    return <AdminDashboard user={user} />;
  }

  return <UserDashboard user={user} />;
}
```

**Lợi thế:** Dễ đọc, dễ debug, xử lý được nhiều nhánh phức tạp.

### Cách 2: Ternary Operator (Toán Tử Ba Ngôi) — Phổ Biến Nhất

Dùng trực tiếp trong JSX, phù hợp với 2 trường hợp:

```jsx
function LoginButton({ isLoggedIn }) {
  return (
    <div>
      {/* Inline ternary */}
      {isLoggedIn ? (
        <button>Đăng Xuất</button>
      ) : (
        <button>Đăng Nhập</button>
      )}

      {/* Ternary cho text */}
      <p>Chào, {isLoggedIn ? "người dùng" : "khách"}!</p>

      {/* Ternary cho className */}
      <span className={isLoggedIn ? "status-active" : "status-inactive"}>
        {isLoggedIn ? "Đang hoạt động" : "Ngoại tuyến"}
      </span>
    </div>
  );
}
```

### Cách 3: Logical AND `&&` — Render Hoặc Không Gì Cả

Dùng khi chỉ có một trường hợp hiển thị (không có "else"):

```jsx
function Notifications({ messages }) {
  return (
    <div>
      <h1>Hộp Thư</h1>

      {/* Chỉ render khi có messages */}
      {messages.length > 0 && (
        <div className="notification-badge">
          {messages.length} tin nhắn mới
        </div>
      )}

      {/* Hiện thị empty state khi không có gì */}
      {messages.length === 0 && (
        <p className="empty-state">Không có tin nhắn mới</p>
      )}
    </div>
  );
}
```

**Cạm bẫy `&&` với số 0:**

```jsx
// ❌ Bug phổ biến: 0 được render thành text "0"
{items.length && <List items={items} />}
// Khi items.length = 0 → render số "0" trên màn hình

// ✅ Đúng: chuyển sang boolean tường minh
{items.length > 0 && <List items={items} />}

// ✅ Hoặc dùng Boolean()
{Boolean(items.length) && <List items={items} />}

// ✅ Hoặc dùng ternary
{items.length ? <List items={items} /> : null}
```

### Cách 4: Nullish Coalescing `??` — Ít Gặp Trong JSX

```jsx
function UserName({ name }) {
  return <p>{name ?? "Người dùng ẩn danh"}</p>;
}
// Render "Người dùng ẩn danh" khi name là null hoặc undefined
// Khác && ở chỗ: 0 và "" không bị coi là falsy → không dùng giá trị mặc định
```

### Cách 5: Biến Trung Gian (Variable)

Dùng khi logic phức tạp nhưng muốn giữ JSX sạch:

```jsx
function ProductCard({ product }) {
  let badge = null;

  if (product.isNew) {
    badge = <span className="badge-new">Mới</span>;
  } else if (product.discount > 0) {
    badge = <span className="badge-sale">-{product.discount}%</span>;
  } else if (product.stock === 0) {
    badge = <span className="badge-out">Hết hàng</span>;
  }

  return (
    <div className="product-card">
      {badge}
      <h3>{product.name}</h3>
      <p>{product.price.toLocaleString("vi-VN")}đ</p>
    </div>
  );
}
```

### Loading, Error, và Empty States

Pattern thực tế để xử lý các trạng thái khác nhau khi fetch data:

```jsx
function UserList({ isLoading, error, users }) {
  if (isLoading) {
    return <Spinner />;
  }

  if (error) {
    return (
      <ErrorMessage
        message={`Không thể tải dữ liệu: ${error.message}`}
      />
    );
  }

  if (users.length === 0) {
    return <EmptyState message="Chưa có người dùng nào" />;
  }

  return (
    <ul>
      {users.map((user) => (
        <UserItem key={user.id} user={user} />
      ))}
    </ul>
  );
}
```

---

## 2. List Rendering — Render Danh Sách

### `Array.map()` — Phương Pháp Cơ Bản

```jsx
function FruitList() {
  const fruits = ["Táo", "Chuối", "Cam", "Xoài"];

  return (
    <ul>
      {fruits.map((fruit, index) => (
        <li key={index}>{fruit}</li>
      ))}
    </ul>
  );
}
```

### Render Danh Sách Object (Đối Tượng)

```jsx
function ProductList() {
  const products = [
    { id: 1, name: "Laptop", price: 15_000_000, category: "Electronics" },
    { id: 2, name: "Bàn phím", price: 1_200_000, category: "Electronics" },
    { id: 3, name: "Chuột", price: 500_000, category: "Electronics" },
  ];

  return (
    <div className="product-grid">
      {products.map((product) => (
        <ProductCard
          key={product.id}
          name={product.name}
          price={product.price}
        />
      ))}
    </div>
  );
}

function ProductCard({ name, price }) {
  return (
    <div className="card">
      <h3>{name}</h3>
      <p>{price.toLocaleString("vi-VN")}đ</p>
    </div>
  );
}
```

### Render Nested Lists (Danh Sách Lồng Nhau)

```jsx
function CategoryMenu() {
  const menu = [
    {
      id: 1,
      category: "Điện Tử",
      items: ["Laptop", "Điện Thoại", "Máy Tính Bảng"],
    },
    {
      id: 2,
      category: "Thời Trang",
      items: ["Áo", "Quần", "Giày"],
    },
  ];

  return (
    <nav>
      {menu.map((section) => (
        <div key={section.id}>
          <h3>{section.category}</h3>
          <ul>
            {section.items.map((item) => (
              <li key={item}>{item}</li> {/* item tên duy nhất trong section */}
            ))}
          </ul>
        </div>
      ))}
    </nav>
  );
}
```

### `Array.filter()` Kết Hợp Với `map()`

```jsx
function ActiveUserList({ users }) {
  return (
    <ul>
      {users
        .filter((user) => user.isActive)      // Lọc chỉ lấy active users
        .map((user) => (                       // Render từng user
          <li key={user.id}>{user.name}</li>
        ))}
    </ul>
  );
}
```

### Render Table (Bảng)

```jsx
function DataTable({ columns, rows }) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={col.key}>{col.label}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {rows.map((row) => (
          <tr key={row.id}>
            {columns.map((col) => (
              <td key={col.key}>{row[col.key]}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Sử dụng
function App() {
  const columns = [
    { key: "name", label: "Tên" },
    { key: "email", label: "Email" },
    { key: "role", label: "Vai Trò" },
  ];

  const rows = [
    { id: 1, name: "An Nguyễn", email: "an@example.com", role: "Admin" },
    { id: 2, name: "Bình Trần", email: "binh@example.com", role: "User" },
  ];

  return <DataTable columns={columns} rows={rows} />;
}
```

---

## 3. Key — Khóa Định Danh

### Tại Sao Key Quan Trọng?

`key` giúp React **nhận diện** (identify) từng item trong danh sách, từ đó thực hiện **Reconciliation (Đồng Bộ Hóa)** hiệu quả hơn. Khi danh sách thay đổi (thêm, xóa, sắp xếp lại), React dùng `key` để biết:
- Item nào được thêm → tạo mới
- Item nào bị xóa → unmount
- Item nào di chuyển vị trí → move (không re-create)

### Quy Tắc Dùng Key

```jsx
// ✅ Tốt nhất: ID duy nhất từ database
{users.map((user) => <UserCard key={user.id} user={user} />)}

// ✅ Chấp nhận được: giá trị string duy nhất
{countries.map((country) => <li key={country.code}>{country.name}</li>)}

// ⚠️ Chỉ dùng khi không có lựa chọn tốt hơn: index
{staticItems.map((item, index) => <li key={index}>{item}</li>)}

// ❌ Tuyệt đối không dùng: random/Math.random()
{items.map((item) => <li key={Math.random()}>{item}</li>)}
// → Mỗi render tạo key mới → React unmount và mount lại toàn bộ!
```

### Hậu Quả Khi Dùng Index Làm Key Sai

```jsx
// Tình huống: danh sách có thể sắp xếp lại hoặc xóa từ giữa
const [items, setItems] = useState(["A", "B", "C", "D"]);

// Dùng index làm key:
// Initial:  key=0:A, key=1:B, key=2:C, key=3:D
// Xóa B:   key=0:A, key=1:C, key=2:D
// React nghĩ: key=1 đổi nội dung từ B→C, key=2 đổi C→D, key=3 bị xóa
// → Re-render tất cả thay vì chỉ xóa B → Hiệu năng kém + có thể bug với state nội bộ

// Dùng ID làm key:
// Initial:  key="a":A, key="b":B, key="c":C, key="d":D
// Xóa B:   key="a":A, key="c":C, key="d":D
// React biết chính xác key="b" bị xóa → chỉ unmount B, giữ nguyên A, C, D
```

**Khi nào index làm key là chấp nhận được:**
1. Danh sách **tĩnh** — không thêm, xóa, sắp xếp lại
2. Items **không có state nội bộ** (input, animation, v.v.)
3. Danh sách rất đơn giản (ví dụ: render breadcrumbs)

### Key Phải Duy Nhất Trong Cùng List

Key chỉ cần duy nhất trong phạm vi danh sách đó, không cần global:

```jsx
function App() {
  const posts = [{ id: 1 }, { id: 2 }];
  const comments = [{ id: 1 }, { id: 2 }]; // id trùng với posts nhưng không sao

  return (
    <>
      <ul>
        {posts.map((post) => <li key={post.id}>Bài {post.id}</li>)}
      </ul>
      <ul>
        {comments.map((comment) => <li key={comment.id}>Bình luận {comment.id}</li>)}
      </ul>
    </>
  );
}
```

### Key Không Phải Prop

`key` là thuộc tính đặc biệt của React, **không được truyền vào component** như props thông thường:

```jsx
function UserCard({ id, name }) {
  // key KHÔNG xuất hiện ở đây
  // Nếu muốn id, phải truyền riêng
  return <div>{name}</div>;
}

<UserCard key={user.id} id={user.id} name={user.name} />
//        ^^^^^^^^^^^ React xử lý nội bộ
//                    ^^^^^^^^^^^ mới là prop bạn access được
```

---

## 4. Kết Hợp Conditional Và List Rendering

### Ví Dụ Thực Tế: Product Catalog (Danh Mục Sản Phẩm)

```jsx
function ProductCatalog({ category, searchQuery }) {
  const [products, setProducts] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  // Giả lập fetch data
  useEffect(() => {
    fetchProducts(category)
      .then(setProducts)
      .catch(setError)
      .finally(() => setIsLoading(false));
  }, [category]);

  // Lọc theo search
  const filteredProducts = products.filter((p) =>
    p.name.toLowerCase().includes(searchQuery.toLowerCase())
  );

  // Render các trạng thái khác nhau
  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorBanner message={error.message} />;
  if (filteredProducts.length === 0) {
    return (
      <EmptyState
        message={
          searchQuery
            ? `Không tìm thấy "${searchQuery}"`
            : "Chưa có sản phẩm trong danh mục này"
        }
      />
    );
  }

  return (
    <div className="product-grid">
      {filteredProducts.map((product) => (
        <div key={product.id} className="product-card">
          {/* Conditional: chỉ hiện badge khi có khuyến mãi */}
          {product.discount > 0 && (
            <span className="discount-badge">-{product.discount}%</span>
          )}

          <img src={product.imageUrl} alt={product.name} />
          <h3>{product.name}</h3>

          {/* Conditional: giá hiển thị khác nhau khi có/không có discount */}
          {product.discount > 0 ? (
            <div>
              <span className="price-original">
                {product.originalPrice.toLocaleString("vi-VN")}đ
              </span>
              <span className="price-sale">
                {product.salePrice.toLocaleString("vi-VN")}đ
              </span>
            </div>
          ) : (
            <span className="price">
              {product.price.toLocaleString("vi-VN")}đ
            </span>
          )}

          {/* Conditional: nút mua khác nhau khi hết hàng */}
          <button
            disabled={product.stock === 0}
            className={product.stock === 0 ? "btn-disabled" : "btn-primary"}
          >
            {product.stock === 0 ? "Hết hàng" : "Thêm vào giỏ"}
          </button>
        </div>
      ))}
    </div>
  );
}
```

---

## 5. Câu Hỏi Phỏng Vấn

### Q1: Tại sao cần `key` trong danh sách? Chuyện gì xảy ra nếu không có?

**Trả lời:** `key` giúp React phân biệt các items trong danh sách trong quá trình Reconciliation (Đồng Bộ Hóa). Nếu không có key, React phải dựa vào vị trí (index) — khi danh sách thay đổi thứ tự hoặc có item bị xóa từ giữa, React tính toán sai và có thể: (1) Re-render không cần thiết toàn bộ danh sách — kém hiệu năng. (2) Bug với controlled inputs — state nội bộ như focus, giá trị input bị gán nhầm sang item khác. (3) Animation sai — Framer Motion hay CSS transitions hoạt động dựa trên key.

### Q2: Tại sao không nên dùng Math.random() làm key?

**Trả lời:** Key phải **ổn định** (stable) — giữ nguyên giữa các lần render. `Math.random()` tạo giá trị mới mỗi render, khiến React nghĩ toàn bộ list là mới → unmount tất cả items cũ và mount lại → mất state, mất focus, hiệu năng tệ nhất có thể, animation bị reset. Tương tự với `Date.now()` hay bất kỳ giá trị ngẫu nhiên nào.

### Q3: Khác biệt giữa `&&` và ternary `? :` trong conditional rendering?

**Trả lời:** `&&` (logical AND): render khi true, không render gì (`null`) khi false — chỉ có một trường hợp hiển thị. Cạm bẫy: nếu vế trái là số `0`, React render text "0". Ternary `? :`: có hai trường hợp tường minh (true branch và false branch) — linh hoạt hơn. Chọn `&&` khi chỉ cần hiển thị hoặc không. Chọn ternary khi cần hai giao diện khác nhau.

### Q4: Tại sao index làm key gây bug với Controlled Components?

**Trả lời:** Controlled Component (như input) lưu state bên trong DOM (focus, selection). Khi danh sách sắp xếp lại và dùng index làm key, React giữ nguyên DOM nodes nhưng truyền props mới vào. Ví dụ: có 3 inputs A, B, C. Xóa B → còn A, C. Dùng index: key=0 giờ là A (ổn), key=1 giờ là C nhưng DOM node cũ (của B) được tái sử dụng → C thừa hưởng state nội bộ của B (focus position, value cache). Dùng ID: key=idC → React biết đây là C, giữ nguyên DOM node của C.

### Q5: Có thể render `null`, `undefined`, `false` trong JSX không?

**Trả lời:** Có. React **bỏ qua** các giá trị `null`, `undefined`, `false`, `true` — không render ra bất cứ gì. Đây là cơ sở của conditional rendering với `&&`. Tuy nhiên **số 0** và **string rỗng `""`** sẽ được render (0 thành text "0", string rỗng không thấy được nhưng vẫn tồn tại trong DOM). Đây là lý do pattern `{count && <Component />}` bị bug khi count = 0 — nên dùng `{count > 0 && <Component />}`.

---

## 🔗 Điều Hướng

- **Tiếp theo:** [5-forms-controlled.md](./5-forms-controlled.md) — Forms và Controlled Components
- **Quay lại:** [3-event-handling.md](./3-event-handling.md) — Xử lý sự kiện
