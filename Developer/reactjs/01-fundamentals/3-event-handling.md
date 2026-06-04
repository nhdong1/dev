# Event Handling — Xử Lý Sự Kiện

> React xây dựng hệ thống xử lý sự kiện riêng gọi là **Synthetic Event System (Hệ Thống Sự Kiện Tổng Hợp)** thay vì dùng trực tiếp DOM events. Hệ thống này chuẩn hóa sự kiện giữa các trình duyệt và tích hợp tự nhiên với vòng đời React.

---

## 📌 Mục Lục

1. [Synthetic Events — Sự Kiện Tổng Hợp](#1-synthetic-events)
2. [Cú Pháp Gắn Event Handler](#2-cú-pháp-gắn-event-handler)
3. [Các Sự Kiện Phổ Biến](#3-các-sự-kiện-phổ-biến)
4. [Event Object — Đối Tượng Sự Kiện](#4-event-object)
5. [Event Propagation — Lan Truyền Sự Kiện](#5-event-propagation)
6. [Truyền Tham Số Cho Event Handler](#6-truyền-tham-số)
7. [Câu hỏi phỏng vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Synthetic Events — Sự Kiện Tổng Hợp

### Synthetic Event Là Gì?

**SyntheticEvent (Sự Kiện Tổng Hợp)** là wrapper (lớp bọc) của React xung quanh native browser events (sự kiện trình duyệt gốc). React tạo ra lớp trung gian này để:

1. **Cross-browser compatibility (Tương Thích Trình Duyệt):** Cùng API bất kể Chrome, Firefox, Safari, Edge
2. **Performance (Hiệu Năng):** React dùng **Event Delegation (Ủy Quyền Sự Kiện)** — gắn một listener duy nhất ở root thay vì mỗi element một listener
3. **Integration (Tích Hợp):** Hoạt động tự nhiên với hệ thống re-render của React

### Khác Biệt So Với HTML

```html
<!-- HTML: attribute dạng string, lowercase -->
<button onclick="handleClick()">Bấm</button>
```

```jsx
// JSX: attribute là function reference, camelCase
<button onClick={handleClick}>Bấm</button>

// ❌ Sai — truyền string
<button onClick="handleClick()">Bấm</button>

// ❌ Sai — gọi hàm ngay lập tức (không phải truyền reference)
<button onClick={handleClick()}>Bấm</button>
```

> **Lưu ý quan trọng:** `onClick={handleClick}` — truyền **tham chiếu hàm**, không phải gọi hàm. `onClick={handleClick()}` sẽ gọi ngay khi render, không phải khi click!

---

## 2. Cú Pháp Gắn Event Handler

### Cách 1: Khai Báo Hàm Riêng (Khuyến Nghị)

```jsx
function LoginButton() {
  function handleClick() {
    console.log("Nút đăng nhập được bấm!");
  }

  return <button onClick={handleClick}>Đăng Nhập</button>;
}
```

### Cách 2: Arrow Function Inline (Hàm Mũi Tên Nội Tuyến)

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Đếm: {count}
    </button>
  );
}
```

> **Trade-off:** Arrow function inline dễ viết nhưng tạo hàm mới mỗi lần render. Với component đơn giản không ảnh hưởng hiệu năng, nhưng khi truyền xuống component con có `React.memo`, sẽ gây re-render không cần thiết.

### Cách 3: Arrow Function Trong Class-style Component (ít dùng)

```jsx
// Dùng trong class components — ít gặp trong code hiện đại
class Toggle extends React.Component {
  state = { isOn: false };

  // Arrow function tự động bind `this`
  handleToggle = () => {
    this.setState({ isOn: !this.state.isOn });
  };

  render() {
    return (
      <button onClick={this.handleToggle}>
        {this.state.isOn ? "Tắt" : "Bật"}
      </button>
    );
  }
}
```

---

## 3. Các Sự Kiện Phổ Biến

### Mouse Events (Sự Kiện Chuột)

```jsx
function MouseEvents() {
  return (
    <div
      onClick={() => console.log("Click")}
      onDoubleClick={() => console.log("Double click")}
      onMouseEnter={() => console.log("Chuột vào")}
      onMouseLeave={() => console.log("Chuột ra")}
      onMouseMove={(e) => console.log(`Vị trí: ${e.clientX}, ${e.clientY}`)}
      onContextMenu={(e) => {
        e.preventDefault(); // Ngăn menu chuột phải mặc định
        console.log("Chuột phải");
      }}
    >
      Di chuột vào đây
    </div>
  );
}
```

### Keyboard Events (Sự Kiện Bàn Phím)

```jsx
function SearchInput() {
  const [query, setQuery] = useState("");

  function handleKeyDown(e) {
    if (e.key === "Enter") {
      console.log("Tìm kiếm:", query);
    }
    if (e.key === "Escape") {
      setQuery(""); // Xóa input khi nhấn Escape
    }
  }

  return (
    <input
      type="text"
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      onKeyDown={handleKeyDown}
      placeholder="Nhấn Enter để tìm kiếm..."
    />
  );
}
```

**Các key phổ biến:** `"Enter"`, `"Escape"`, `"ArrowUp"`, `"ArrowDown"`, `"Tab"`, `"Backspace"`, `" "` (Space)

**Modifier keys (Phím Kết Hợp):** `e.ctrlKey`, `e.shiftKey`, `e.altKey`, `e.metaKey` (Cmd trên Mac)

```jsx
function handleKeyDown(e) {
  // Ctrl + S để lưu
  if (e.ctrlKey && e.key === "s") {
    e.preventDefault(); // Ngăn hành vi lưu trang mặc định
    handleSave();
  }
}
```

### Form Events (Sự Kiện Form)

```jsx
function ContactForm() {
  const [formData, setFormData] = useState({ name: "", email: "" });

  function handleChange(e) {
    const { name, value } = e.target;
    setFormData((prev) => ({ ...prev, [name]: value }));
  }

  function handleSubmit(e) {
    e.preventDefault(); // Ngăn form reload trang — BẮT BUỘC
    console.log("Dữ liệu gửi đi:", formData);
  }

  function handleReset() {
    setFormData({ name: "", email: "" });
  }

  return (
    <form onSubmit={handleSubmit} onReset={handleReset}>
      <input
        name="name"
        value={formData.name}
        onChange={handleChange}
        placeholder="Họ tên"
      />
      <input
        name="email"
        value={formData.email}
        onChange={handleChange}
        placeholder="Email"
      />
      <button type="submit">Gửi</button>
      <button type="reset">Đặt lại</button>
    </form>
  );
}
```

### Focus Events (Sự Kiện Focus)

```jsx
function FocusExample() {
  const [isFocused, setIsFocused] = useState(false);

  return (
    <input
      className={isFocused ? "input-focused" : "input-blur"}
      onFocus={() => setIsFocused(true)}
      onBlur={() => setIsFocused(false)}
      placeholder="Click vào để focus"
    />
  );
}
```

### Clipboard Events (Sự Kiện Clipboard)

```jsx
function ClipboardExample() {
  return (
    <input
      onCopy={(e) => console.log("Đã copy")}
      onPaste={(e) => {
        const text = e.clipboardData.getData("text");
        console.log("Đã paste:", text);
      }}
      onCut={(e) => console.log("Đã cut")}
    />
  );
}
```

---

## 4. Event Object — Đối Tượng Sự Kiện

Mọi event handler đều nhận một **SyntheticEvent object** với các thuộc tính quan trọng:

```jsx
function EventObjectDemo() {
  function handleClick(e) {
    // Thông tin về sự kiện
    console.log(e.type);          // "click"
    console.log(e.target);        // DOM element được click
    console.log(e.currentTarget); // DOM element gắn event listener
    console.log(e.timeStamp);     // Thời điểm sự kiện xảy ra

    // Vị trí chuột
    console.log(e.clientX, e.clientY); // Tọa độ trong viewport
    console.log(e.pageX, e.pageY);     // Tọa độ trong trang

    // Các phím đang giữ
    console.log(e.ctrlKey, e.shiftKey, e.altKey);

    // Ngăn hành vi mặc định
    e.preventDefault();

    // Dừng lan truyền sự kiện
    e.stopPropagation();
  }

  return <button onClick={handleClick}>Kiểm tra event</button>;
}
```

### `e.target` vs `e.currentTarget`

```jsx
function TargetDemo() {
  function handleClick(e) {
    // e.target: element thực sự được click (có thể là element con)
    // e.currentTarget: element có gắn event listener
    console.log("target:", e.target.tagName);
    console.log("currentTarget:", e.currentTarget.tagName);
  }

  return (
    <div onClick={handleClick}>  {/* currentTarget luôn là DIV này */}
      <span>Bấm vào span này</span>  {/* target là SPAN */}
    </div>
  );
}
```

---

## 5. Event Propagation — Lan Truyền Sự Kiện

### Bubbling (Nổi Bọt) — Mặc Định

Sự kiện lan truyền từ **element con lên cha** (bottom-up):

```jsx
function BubblingExample() {
  return (
    <div onClick={() => console.log("DIV (cha) được click")}>
      <button onClick={() => console.log("BUTTON được click")}>
        Bấm tôi
      </button>
    </div>
  );
  // Kết quả khi click button:
  // 1. "BUTTON được click"
  // 2. "DIV (cha) được click"  ← sự kiện nổi lên cha
}
```

### `stopPropagation()` — Dừng Lan Truyền

```jsx
function StopPropagationExample() {
  function handleDivClick() {
    console.log("DIV click");
  }

  function handleButtonClick(e) {
    e.stopPropagation(); // Ngăn sự kiện nổi lên div
    console.log("BUTTON click — sự kiện dừng ở đây");
  }

  return (
    <div onClick={handleDivClick}>
      <button onClick={handleButtonClick}>
        Bấm (không nổi lên div)
      </button>
    </div>
  );
}
```

### `preventDefault()` — Ngăn Hành Vi Mặc Định

```jsx
function PreventDefaultExamples() {
  return (
    <div>
      {/* Ngăn link điều hướng */}
      <a
        href="https://example.com"
        onClick={(e) => {
          e.preventDefault();
          console.log("Link bị chặn, không điều hướng");
        }}
      >
        Link bị chặn
      </a>

      {/* Ngăn form reload trang */}
      <form onSubmit={(e) => {
        e.preventDefault();
        console.log("Form submit bị chặn");
      }}>
        <button type="submit">Gửi</button>
      </form>

      {/* Ngăn menu chuột phải */}
      <div onContextMenu={(e) => {
        e.preventDefault();
        console.log("Menu chuột phải bị chặn");
      }}>
        Chuột phải vào đây
      </div>
    </div>
  );
}
```

### Capturing Phase (Giai Đoạn Bắt Sự Kiện) — Ít Dùng

Mặc định React dùng **bubbling phase** (sự kiện nổi từ con lên cha). Để xử lý trong **capturing phase** (cha bắt trước con), dùng hậu tố `Capture`:

```jsx
<div onClickCapture={() => console.log("DIV — capture phase trước")}>
  <button onClick={() => console.log("BUTTON — bubble phase")}>
    Bấm
  </button>
</div>
// Thứ tự: 1. "DIV — capture phase trước"  2. "BUTTON — bubble phase"
```

---

## 6. Truyền Tham Số Cho Event Handler

### Cách Thông Thường

```jsx
function ItemList() {
  const items = ["Táo", "Chuối", "Cam"];

  function handleDelete(itemName, e) {
    // e vẫn nhận được nếu cần
    console.log("Xóa:", itemName);
  }

  return (
    <ul>
      {items.map((item) => (
        <li key={item}>
          {item}
          {/* Arrow function để truyền tham số */}
          <button onClick={(e) => handleDelete(item, e)}>Xóa</button>
        </li>
      ))}
    </ul>
  );
}
```

### Dùng `data-*` Attributes (Thuộc Tính Dữ Liệu)

```jsx
function ItemList() {
  const items = [
    { id: 1, name: "Táo" },
    { id: 2, name: "Chuối" },
  ];

  function handleDelete(e) {
    const itemId = Number(e.currentTarget.dataset.id);
    console.log("Xóa item có id:", itemId);
  }

  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>
          {item.name}
          <button data-id={item.id} onClick={handleDelete}>Xóa</button>
        </li>
      ))}
    </ul>
  );
}
```

---

## 7. Câu Hỏi Phỏng Vấn

### Q1: Synthetic Event là gì? Tại sao React dùng nó?

**Trả lời:** SyntheticEvent là wrapper React xây dựng xung quanh native browser events. Mục đích: (1) **Cross-browser compatibility** — chuẩn hóa API giữa các trình duyệt, tránh các quirks lịch sử. (2) **Performance** qua Event Delegation — React gắn một event listener duy nhất ở root DOM node (`#root`) thay vì gắn vào từng element, giảm memory footprint. (3) **React lifecycle integration** — có thể thực hiện cleanup sau khi event được xử lý.

### Q2: `preventDefault()` và `stopPropagation()` khác nhau như thế nào?

**Trả lời:**
- `e.preventDefault()`: Ngăn **hành vi mặc định của browser** — ví dụ link không navigate, form không reload trang, checkbox không toggle. Sự kiện vẫn tiếp tục lan truyền.
- `e.stopPropagation()`: Ngăn **sự kiện lan truyền** lên các element cha. Hành vi mặc định vẫn xảy ra (trừ khi cũng gọi `preventDefault()`).
- Có thể gọi cả hai khi cần ngăn cả hành vi mặc định lẫn lan truyền.

### Q3: Tại sao không nên gọi hàm trực tiếp trong JSX như `onClick={handleClick()}`?

**Trả lời:** `onClick={handleClick()}` gọi `handleClick` **ngay khi render**, không phải khi click. Kết quả trả về của `handleClick()` (thường là `undefined`) được gán cho `onClick`. Nếu `handleClick` cập nhật state, sẽ trigger render lại → gọi lại → vòng lặp vô tận (infinite loop). Đúng là `onClick={handleClick}` — truyền tham chiếu hàm để React gọi sau khi click xảy ra.

### Q4: Event Delegation trong React hoạt động như thế nào?

**Trả lời:** Thay vì gắn listener vào từng DOM element, React gắn **một listener duy nhất** vào root element (thường là `document` hoặc `#root`). Khi sự kiện xảy ra trên bất kỳ element con nào, nó bubble lên root. React interceptor (chặn) tại root, xác định element nguồn qua `e.target`, tra cứu component tương ứng trong fiber tree và gọi handler. Kết quả: ít event listeners → tiết kiệm bộ nhớ, đặc biệt quan trọng với danh sách dài.

### Q5: Tại sao event handler không nên là async trực tiếp?

**Trả lời:** Không có vấn đề kỹ thuật khi dùng `async` cho event handler (`onClick={async () => { await doSomething(); }}`). Tuy nhiên cần cẩn thận: (1) SyntheticEvent được **pooled** (dùng chung) trong React cũ (trước v17) — nếu access `e` sau `await`, event đã bị reset. React v17+ bỏ event pooling nên issue này không còn. (2) Cần tự handle loading states và errors vì React không auto-handle async errors trong event handlers.

---

## 🔗 Điều Hướng

- **Tiếp theo:** [4-conditional-list-rendering.md](./4-conditional-list-rendering.md) — Render có điều kiện và danh sách
- **Quay lại:** [2-props-state.md](./2-props-state.md) — Props & State
