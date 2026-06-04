# useEffect & useLayoutEffect — Quản Lý Side Effects

> `useEffect` và `useLayoutEffect` là các hooks dùng để thực hiện **side effects** (tác dụng phụ) — những thao tác ảnh hưởng đến thế giới bên ngoài React như: fetch data (lấy dữ liệu), thao tác DOM, đăng ký event listeners, timers, và subscriptions (đăng ký theo dõi).

---

## 📌 Mục Lục

1. [Side Effects Là Gì?](#1-side-effects-là-gì)
2. [useEffect — Cú Pháp và Cách Dùng](#2-useeffect)
3. [Dependency Array — Mảng Phụ Thuộc](#3-dependency-array)
4. [Cleanup Function — Hàm Dọn Dẹp](#4-cleanup-function)
5. [useLayoutEffect — Khi Nào Cần?](#5-uselayouteffect)
6. [Các Pattern Phổ Biến](#6-các-pattern-phổ-biến)
7. [Lỗi Thường Gặp](#7-lỗi-thường-gặp)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Side Effects Là Gì?

Trong React, **side effect** (tác dụng phụ) là bất kỳ thao tác nào ảnh hưởng đến thứ **nằm ngoài phạm vi của component** đang render:

```
✅ Có side effects:                    ❌ Không phải side effects:
  - Fetch API                            - Tính toán từ props/state
  - Đọc/ghi localStorage/sessionStorage  - Tạo JSX để render
  - Thao tác DOM trực tiếp              - Xử lý event (onClick...)
  - Đăng ký event listeners             - Conditional rendering
  - setInterval / setTimeout
  - WebSocket subscriptions
  - Logging
  - Cập nhật document.title
```

Side effects không được phép trong render phase (giai đoạn render) vì render phải là **pure** (thuần túy — không có tác dụng phụ, cùng input luôn cho cùng output).

---

## 2. useEffect — Cú Pháp và Cách Dùng

### Cú Pháp

```jsx
useEffect(() => {
  // Thân effect: code chạy sau mỗi lần render
  // ...

  return () => {
    // Cleanup function (không bắt buộc): chạy trước khi effect chạy lại
    // hoặc khi component unmount (bị gỡ khỏi DOM)
  };
}, [dependency1, dependency2]); // Dependency array (không bắt buộc)
```

### Ví Dụ Cơ Bản — Cập Nhật Document Title

```jsx
import { useState, useEffect } from "react";

function PageTitle({ title }) {
  useEffect(() => {
    // Chạy sau render, khi title thay đổi
    document.title = title;

    // Cleanup: khôi phục title khi component unmount
    return () => {
      document.title = "React App";
    };
  }, [title]); // Chỉ chạy lại khi title thay đổi

  return <h1>{title}</h1>;
}
```

---

## 3. Dependency Array — Mảng Phụ Thuộc

Dependency Array (Mảng Phụ Thuộc) kiểm soát **khi nào** effect chạy:

### Ba Trường Hợp

```jsx
// Trường hợp 1: Không có dependency array — chạy SAU MỖI lần render
useEffect(() => {
  console.log("Chạy sau mỗi lần render");
});

// Trường hợp 2: Dependency array rỗng [] — chỉ chạy MỘT LẦN sau render đầu tiên
useEffect(() => {
  console.log("Chỉ chạy một lần (componentDidMount)");
}, []);

// Trường hợp 3: Có dependencies — chạy khi BẤT KỲ dependency nào thay đổi
useEffect(() => {
  console.log(`userId hoặc page thay đổi: ${userId}, ${page}`);
}, [userId, page]);
```

### Ví Dụ Thực Tế — Fetch Data

```jsx
import { useState, useEffect } from "react";

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // Reset state khi userId thay đổi
    setLoading(true);
    setError(null);

    const controller = new AbortController(); // AbortController — Bộ Điều Khiển Hủy

    async function fetchUser() {
      try {
        const response = await fetch(`/api/users/${userId}`, {
          signal: controller.signal, // Cho phép hủy request
        });

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();
        setUser(data);
      } catch (err) {
        if (err.name !== "AbortError") {
          // Bỏ qua lỗi do hủy request, chỉ xử lý lỗi thật sự
          setError(err.message);
        }
      } finally {
        setLoading(false);
      }
    }

    fetchUser();

    // Cleanup: hủy request khi userId thay đổi hoặc component unmount
    return () => {
      controller.abort();
    };
  }, [userId]); // Chạy lại mỗi khi userId thay đổi

  if (loading) return <p>Đang tải...</p>;
  if (error) return <p>Lỗi: {error}</p>;
  if (!user) return null;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

### Exhaustive Dependencies — Khai Báo Đầy Đủ Dependencies

ESLint plugin `exhaustive-deps` yêu cầu khai báo **tất cả** giá trị được dùng trong effect vào dependency array:

```jsx
function SearchResults({ query, userId }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    // query và userId đều được dùng → phải có trong deps
    fetchResults(query, userId).then(setResults);
  }, [query, userId]); // ✅ Đầy đủ dependencies

  // ❌ Thiếu userId → ESLint cảnh báo, có thể gây bug
  // }, [query]);
}
```

---

## 4. Cleanup Function — Hàm Dọn Dẹp

Cleanup function (Hàm Dọn Dẹp) là hàm được trả về từ effect, chạy khi:
1. **Component unmount** (bị gỡ khỏi cây DOM)
2. **Effect sắp chạy lại** (trước khi dependency thay đổi)

### Ví Dụ — Event Listener (Lắng Nghe Sự Kiện)

```jsx
function WindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }

    // Đăng ký event listener
    window.addEventListener("resize", handleResize);

    // Cleanup: hủy đăng ký khi component unmount
    return () => {
      window.removeEventListener("resize", handleResize);
    };
  }, []); // Chỉ chạy một lần

  return (
    <p>
      Kích thước cửa sổ: {size.width} × {size.height}
    </p>
  );
}
```

### Ví Dụ — Timer (Đồng Hồ Đếm)

```jsx
function Countdown({ seconds }) {
  const [timeLeft, setTimeLeft] = useState(seconds);

  useEffect(() => {
    if (timeLeft === 0) return;

    const timerId = setInterval(() => {
      setTimeLeft(prev => prev - 1);
    }, 1000);

    // Cleanup: xóa interval khi effect chạy lại hoặc unmount
    return () => {
      clearInterval(timerId);
    };
  }, [timeLeft]);

  return <p>Còn lại: {timeLeft} giây</p>;
}
```

### Ví Dụ — WebSocket Subscription (Đăng Ký WebSocket)

```jsx
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    // Kết nối WebSocket khi roomId thay đổi
    const socket = new WebSocket(`wss://chat.example.com/${roomId}`);

    socket.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      setMessages(prev => [...prev, msg]);
    };

    // Cleanup: đóng kết nối khi đổi phòng hoặc unmount
    return () => {
      socket.close();
    };
  }, [roomId]); // Reconnect khi đổi phòng

  return (
    <ul>
      {messages.map((msg, i) => (
        <li key={i}>{msg.text}</li>
      ))}
    </ul>
  );
}
```

---

## 5. useLayoutEffect — Khi Nào Cần?

### So Sánh Timeline (Dòng Thời Gian)

```
useEffect:
  [React renders component]
       ↓
  [DOM cập nhật]
       ↓
  [Trình duyệt paint (vẽ) lên màn hình] ← người dùng thấy UI
       ↓
  [useEffect chạy] ← bất đồng bộ, không block paint

useLayoutEffect:
  [React renders component]
       ↓
  [DOM cập nhật]
       ↓
  [useLayoutEffect chạy] ← đồng bộ, BLOCK paint
       ↓
  [Trình duyệt paint lên màn hình] ← người dùng thấy UI
```

### Khi Nào Dùng useLayoutEffect?

Dùng `useLayoutEffect` khi cần đọc layout DOM và cập nhật trước khi trình duyệt vẽ (paint) — tránh **visual flicker** (nhấp nháy giao diện):

```jsx
import { useRef, useLayoutEffect, useState } from "react";

function Tooltip({ children, text }) {
  const tooltipRef = useRef(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    if (!tooltipRef.current) return;

    const rect = tooltipRef.current.getBoundingClientRect();
    // Tính toán vị trí tooltip dựa trên kích thước thật của DOM
    const newLeft = rect.width > window.innerWidth / 2
      ? rect.right - rect.width  // Hiện bên trái
      : rect.left;               // Hiện bên phải

    setPosition({ top: rect.bottom + 8, left: newLeft });
    // useLayoutEffect: vị trí được tính TRƯỚC khi paint
    // → người dùng không thấy tooltip nhảy vị trí
  });

  return (
    <div>
      <div ref={tooltipRef}>{children}</div>
      <div
        style={{ position: "fixed", top: position.top, left: position.left }}
        className="tooltip"
      >
        {text}
      </div>
    </div>
  );
}
```

### Quy Tắc Sử Dụng

```
✅ Dùng useEffect (95% trường hợp):
  - Fetch data
  - Event listeners
  - Timers
  - Logging, analytics
  - Subscriptions

✅ Dùng useLayoutEffect (5% trường hợp):
  - Đọc kích thước/vị trí DOM để render lại
  - Animations cần biết layout trước
  - Tránh visual flicker khi cập nhật DOM
  - Tích hợp thư viện bên thứ ba thao tác DOM trực tiếp

⚠️ Lưu ý: useLayoutEffect không hoạt động trên Server (SSR — Server-Side Rendering).
   Dùng useEffect hoặc điều kiện `typeof window !== "undefined"`.
```

---

## 6. Các Pattern Phổ Biến

### Pattern 1: Debounce Search (Tìm Kiếm Có Độ Trễ)

```jsx
function SearchBox() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (!query.trim()) {
      setResults([]);
      return;
    }

    // Debounce — trì hoãn 400ms trước khi fetch
    const timeoutId = setTimeout(() => {
      fetch(`/api/search?q=${query}`)
        .then(res => res.json())
        .then(setResults);
    }, 400);

    // Cleanup: hủy timeout nếu query thay đổi trước 400ms
    return () => clearTimeout(timeoutId);
  }, [query]);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Tìm kiếm..."
      />
      <ul>
        {results.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Pattern 2: Sync State Với localStorage

```jsx
function usePersistentState(key, defaultValue) {
  const [value, setValue] = useState(() => {
    const saved = localStorage.getItem(key);
    return saved !== null ? JSON.parse(saved) : defaultValue;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// Sử dụng Custom Hook
function Settings() {
  const [theme, setTheme] = usePersistentState("theme", "light");

  return (
    <button onClick={() => setTheme(t => t === "light" ? "dark" : "light")}>
      Chuyển sang {theme === "light" ? "dark" : "light"} mode
    </button>
  );
}
```

### Pattern 3: Intersection Observer (Theo Dõi Phần Tử Trong Viewport)

```jsx
function LazyImage({ src, alt }) {
  const [isVisible, setIsVisible] = useState(false);
  const imgRef = useRef(null);

  useEffect(() => {
    const element = imgRef.current;
    if (!element) return;

    // IntersectionObserver — Bộ Theo Dõi Giao Cắt với Viewport
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect(); // Dừng theo dõi sau khi hiển thị
        }
      },
      { threshold: 0.1 } // Hiển thị khi 10% phần tử vào viewport
    );

    observer.observe(element);

    return () => observer.disconnect();
  }, []);

  return (
    <div ref={imgRef}>
      {isVisible ? (
        <img src={src} alt={alt} />
      ) : (
        <div className="placeholder">Đang tải ảnh...</div>
      )}
    </div>
  );
}
```

---

## 7. Lỗi Thường Gặp

### Lỗi 1: Infinite Loop (Vòng Lặp Vô Tận)

```jsx
// ❌ Bug: Mỗi render → effect chạy → setState → re-render → effect chạy → ...
function BadComponent() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch("/api/data").then(res => res.json()).then(setData);
    // Không có dependency array → chạy SAU MỖI render
    // setData gây re-render → effect lại chạy → vòng lặp vô tận!
  }); // Thiếu dependency array

  return <div>{JSON.stringify(data)}</div>;
}

// ✅ Đúng
function GoodComponent() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch("/api/data").then(res => res.json()).then(setData);
  }, []); // [] → chỉ chạy một lần

  return <div>{JSON.stringify(data)}</div>;
}
```

### Lỗi 2: Object/Function Trong Dependency Array

```jsx
// ❌ Bug: object options được tạo mới mỗi lần render → effect chạy vô tận
function ProblematicComponent({ userId }) {
  const [data, setData] = useState(null);
  const options = { headers: { Authorization: "Bearer token" } }; // Tạo mới mỗi render!

  useEffect(() => {
    fetchWithOptions(userId, options).then(setData);
  }, [userId, options]); // options luôn "thay đổi" vì là object mới!
}

// ✅ Giải pháp 1: Move object ra ngoài component (nếu không phụ thuộc props/state)
const OPTIONS = { headers: { Authorization: "Bearer token" } };

function Fixed1({ userId }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchWithOptions(userId, OPTIONS).then(setData);
  }, [userId]); // OPTIONS không đổi → không cần trong deps
}

// ✅ Giải pháp 2: useMemo hoặc useCallback để ổn định reference
function Fixed2({ userId, token }) {
  const [data, setData] = useState(null);
  const options = useMemo(
    () => ({ headers: { Authorization: `Bearer ${token}` } }),
    [token] // Chỉ tạo lại khi token thay đổi
  );

  useEffect(() => {
    fetchWithOptions(userId, options).then(setData);
  }, [userId, options]);
}
```

### Lỗi 3: Race Condition (Điều Kiện Tranh Chấp)

```jsx
// ❌ Bug: Race Condition — request trước có thể về sau request sau
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data)); // Request cũ có thể ghi đè data mới!
  }, [userId]);
}

// ✅ Giải pháp: Dùng AbortController hoặc ignore flag
function UserProfileFixed({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let ignore = false; // Cờ để bỏ qua kết quả cũ

    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        if (!ignore) setUser(data); // Chỉ cập nhật nếu request này vẫn còn valid
      });

    return () => {
      ignore = true; // Đánh dấu request này đã obsolete (lỗi thời)
    };
  }, [userId]);
}
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: Giải thích vòng đời của useEffect?

**Trả lời:**
1. **Mount** (Gắn vào DOM): Effect chạy lần đầu sau khi component render xong
2. **Update** (Cập nhật): Nếu dependencies thay đổi, cleanup của effect cũ chạy trước, rồi effect mới chạy
3. **Unmount** (Gỡ khỏi DOM): Cleanup của effect cuối chạy

```
Mount:   render → DOM update → paint → effect chạy
Update:  render → DOM update → paint → cleanup cũ → effect mới chạy
Unmount: cleanup cuối chạy
```

### Q2: Tại sao useEffect với [] không bằng componentDidMount?

**Trả lời:** Về mặt **timing** (thời điểm), chúng tương đương — đều chạy sau render đầu tiên. Nhưng về **semantics** (ý nghĩa), khác nhau: `useEffect(fn, [])` nghĩa là "effect này không phụ thuộc vào bất kỳ giá trị nào từ component". Với Strict Mode (Chế Độ Nghiêm Ngặt) trong React 18+, effect chạy **hai lần** trong development để phát hiện cleanup bị thiếu — `componentDidMount` chỉ chạy một lần.

### Q3: Khi nào cần cleanup function?

**Trả lời:** Cần cleanup khi effect tạo ra subscriptions (đăng ký), connections (kết nối), timers, hoặc bất kỳ thứ gì cần "hủy" khi component unmount hoặc effect chạy lại. Thiếu cleanup gây **memory leak** (rò rỉ bộ nhớ) — component đã unmount nhưng setTimeout vẫn chạy, WebSocket vẫn mở...

### Q4: useEffect vs useLayoutEffect — khi nào dùng gì?

**Trả lời:** Dùng `useLayoutEffect` khi cần đọc layout DOM (getBoundingClientRect, scrollTop...) và cập nhật DOM hoặc state dựa trên đó, **trước khi** trình duyệt vẽ ra màn hình. Điều này tránh "flash" (nhấp nháy) — người dùng thấy UI bị sai vị trí rồi nhảy. Nếu không cần đọc layout, luôn dùng `useEffect` vì nó không block paint → UI responsive hơn.

### Q5: Giải thích Race Condition trong useEffect và cách fix?

**Trả lời:** Race Condition (Điều Kiện Tranh Chấp) xảy ra khi: userId thay đổi từ A → B, request A được gửi, request B được gửi, nhưng request A về **sau** request B → kết quả của A ghi đè kết quả của B → hiển thị dữ liệu sai. Fix bằng cách: sử dụng `AbortController` để cancel request cũ, hoặc dùng `ignore` flag để bỏ qua kết quả của request đã outdated.

---

## 🔗 Điều Hướng

- **Trở về:** [README.md](./README.md) — Tổng quan Hooks
- **Trước đó:** [1-usestate-usereducer.md](./1-usestate-usereducer.md) — useState và useReducer
- **Tiếp theo:** [3-usecontext-useref.md](./3-usecontext-useref.md) — useContext và useRef

**Cập Nhật Lần Cuối:** 2026-06-04 | **React Version:** v19 (stable)
