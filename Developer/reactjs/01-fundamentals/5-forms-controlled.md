# Forms, Controlled & Uncontrolled Components — Quản Lý Form Trong React

> React cung cấp hai chiến lược quản lý form: **Controlled Components (Component Được Kiểm Soát)** — React là nguồn sự thật duy nhất; và **Uncontrolled Components (Component Không Được Kiểm Soát)** — DOM tự quản lý state. Hiểu rõ khi nào dùng cái gì là kỹ năng thiết yếu.

---

## 📌 Mục Lục

1. [Controlled Components — Component Được Kiểm Soát](#1-controlled-components)
2. [Uncontrolled Components — Component Không Được Kiểm Soát](#2-uncontrolled-components)
3. [Controlled vs Uncontrolled — So Sánh](#3-controlled-vs-uncontrolled)
4. [Form Phức Tạp](#4-form-phức-tạp)
5. [Validation — Kiểm Tra Dữ Liệu](#5-validation)
6. [Câu hỏi phỏng vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Controlled Components — Component Được Kiểm Soát

### Khái Niệm

**Controlled Component** là input element mà React kiểm soát hoàn toàn giá trị thông qua `state`. Mỗi lần người dùng gõ, React cập nhật state → re-render → input hiển thị giá trị từ state. Vòng lặp: **Input → onChange → setState → re-render → Input**.

React là **Single Source of Truth (Nguồn Sự Thật Duy Nhất)** cho giá trị input.

### Input Text Cơ Bản

```jsx
import { useState } from "react";

function NameInput() {
  const [name, setName] = useState("");

  return (
    <div>
      <input
        type="text"
        value={name}           // ← React kiểm soát value
        onChange={(e) => setName(e.target.value)}  // ← Cập nhật state khi thay đổi
        placeholder="Nhập tên của bạn"
      />
      <p>Xin chào, {name || "người lạ"}!</p>
    </div>
  );
}
```

> **Lưu ý:** Nếu gán `value` mà không có `onChange`, input sẽ bị read-only và React sẽ cảnh báo. Muốn read-only tường minh thì dùng prop `readOnly`.

### Các Loại Input Controlled

```jsx
function AllInputTypes() {
  const [formData, setFormData] = useState({
    text: "",
    email: "",
    password: "",
    number: 0,
    textarea: "",
    select: "hanoi",
    checkbox: false,
    radio: "male",
  });

  function handleChange(e) {
    const { name, value, type, checked } = e.target;
    setFormData((prev) => ({
      ...prev,
      // Checkbox dùng `checked` thay vì `value`
      [name]: type === "checkbox" ? checked : value,
    }));
  }

  return (
    <form>
      {/* Text Input */}
      <input
        type="text"
        name="text"
        value={formData.text}
        onChange={handleChange}
      />

      {/* Email Input */}
      <input
        type="email"
        name="email"
        value={formData.email}
        onChange={handleChange}
      />

      {/* Password Input */}
      <input
        type="password"
        name="password"
        value={formData.password}
        onChange={handleChange}
      />

      {/* Number Input */}
      <input
        type="number"
        name="number"
        value={formData.number}
        onChange={(e) =>
          setFormData((prev) => ({ ...prev, number: Number(e.target.value) }))
        }
      />

      {/* Textarea */}
      <textarea
        name="textarea"
        value={formData.textarea}
        onChange={handleChange}
        rows={4}
      />

      {/* Select */}
      <select name="select" value={formData.select} onChange={handleChange}>
        <option value="hanoi">Hà Nội</option>
        <option value="hcm">TP. Hồ Chí Minh</option>
        <option value="danang">Đà Nẵng</option>
      </select>

      {/* Checkbox */}
      <input
        type="checkbox"
        name="checkbox"
        checked={formData.checkbox}
        onChange={handleChange}
      />

      {/* Radio */}
      <label>
        <input
          type="radio"
          name="radio"
          value="male"
          checked={formData.radio === "male"}
          onChange={handleChange}
        />
        Nam
      </label>
      <label>
        <input
          type="radio"
          name="radio"
          value="female"
          checked={formData.radio === "female"}
          onChange={handleChange}
        />
        Nữ
      </label>
    </form>
  );
}
```

### Multiple Checkboxes (Nhiều Checkbox)

```jsx
function HobbySelector() {
  const [selectedHobbies, setSelectedHobbies] = useState([]);
  const hobbies = ["Đọc sách", "Thể thao", "Nấu ăn", "Du lịch", "Âm nhạc"];

  function handleHobbyChange(hobby) {
    setSelectedHobbies((prev) =>
      prev.includes(hobby)
        ? prev.filter((h) => h !== hobby)   // Bỏ chọn
        : [...prev, hobby]                   // Thêm vào
    );
  }

  return (
    <div>
      <p>Chọn sở thích:</p>
      {hobbies.map((hobby) => (
        <label key={hobby} style={{ display: "block" }}>
          <input
            type="checkbox"
            checked={selectedHobbies.includes(hobby)}
            onChange={() => handleHobbyChange(hobby)}
          />
          {hobby}
        </label>
      ))}
      <p>Đã chọn: {selectedHobbies.join(", ") || "Chưa chọn gì"}</p>
    </div>
  );
}
```

---

## 2. Uncontrolled Components — Component Không Được Kiểm Soát

### Khái Niệm

**Uncontrolled Component** để DOM tự quản lý state của input. React chỉ đọc giá trị khi cần (thường lúc submit) thông qua **ref (tham chiếu DOM)**.

```jsx
import { useRef } from "react";

function UncontrolledForm() {
  const nameRef = useRef(null);
  const emailRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    // Đọc giá trị từ DOM khi submit
    const name = nameRef.current.value;
    const email = emailRef.current.value;
    console.log({ name, email });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        ref={nameRef}
        defaultValue="Nguyễn Văn A"  // defaultValue: giá trị ban đầu (không controlled)
        placeholder="Họ tên"
      />
      <input
        type="email"
        ref={emailRef}
        placeholder="Email"
      />
      <button type="submit">Gửi</button>
    </form>
  );
}
```

### `defaultValue` vs `value`

```jsx
// Controlled: value được React kiểm soát, cần onChange
<input value={name} onChange={(e) => setName(e.target.value)} />

// Uncontrolled: defaultValue chỉ đặt giá trị ban đầu, DOM tự quản lý sau đó
<input defaultValue="Giá trị mặc định" ref={inputRef} />

// Tương tự với checkbox:
<input type="checkbox" defaultChecked={true} ref={checkRef} />

// Tương tự với select:
<select defaultValue="hanoi" ref={selectRef}>
  <option value="hanoi">Hà Nội</option>
</select>
```

### Khi Nào Dùng Uncontrolled?

- **File inputs:** `<input type="file">` **luôn phải là uncontrolled** — không thể set value bằng JavaScript vì lý do bảo mật
- **Tích hợp với thư viện non-React:** Khi form được quản lý bởi thư viện khác
- **Form cực kỳ đơn giản:** Chỉ cần đọc giá trị khi submit, không cần real-time validation

```jsx
function FileUpload() {
  const fileRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    const file = fileRef.current.files[0];
    if (file) {
      console.log("File được chọn:", file.name, file.size);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* File input: LUÔN phải là uncontrolled */}
      <input type="file" ref={fileRef} accept="image/*" />
      <button type="submit">Upload</button>
    </form>
  );
}
```

---

## 3. Controlled vs Uncontrolled — So Sánh

| Đặc Điểm | Controlled | Uncontrolled |
| --------- | ---------- | ------------ |
| **Nguồn dữ liệu** | React state | DOM |
| **Đọc giá trị** | Từ state bất kỳ lúc nào | Qua ref, thường lúc submit |
| **Real-time validation** | ✅ Dễ dàng | ❌ Phức tạp |
| **Conditional disable** | ✅ Trực tiếp | ❌ Cần thêm code |
| **Enforce format** | ✅ Kiểm soát hoàn toàn | ❌ Khó |
| **Boilerplate code** | Nhiều hơn | Ít hơn |
| **Re-render** | Mỗi keystroke | Chỉ khi cần |
| **File input** | ❌ Không dùng được | ✅ Bắt buộc |
| **React Forms best practice** | ✅ Khuyến nghị | Hạn chế dùng |

**Kết luận:** Dùng **Controlled** cho hầu hết form. Dùng **Uncontrolled** chỉ khi tích hợp với non-React code hoặc với `<input type="file">`.

---

## 4. Form Phức Tạp

### Pattern: Generic Handler Với `name` Attribute

```jsx
function RegistrationForm() {
  const [formData, setFormData] = useState({
    firstName: "",
    lastName: "",
    email: "",
    phone: "",
    city: "",
    agreeToTerms: false,
  });

  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  // Generic handler — xử lý tất cả input
  function handleChange(e) {
    const { name, value, type, checked } = e.target;
    setFormData((prev) => ({
      ...prev,
      [name]: type === "checkbox" ? checked : value,
    }));

    // Xóa error khi người dùng bắt đầu sửa
    if (errors[name]) {
      setErrors((prev) => ({ ...prev, [name]: "" }));
    }
  }

  function validate() {
    const newErrors = {};
    if (!formData.firstName.trim()) {
      newErrors.firstName = "Vui lòng nhập tên";
    }
    if (!formData.email.includes("@")) {
      newErrors.email = "Email không hợp lệ";
    }
    if (formData.phone && !/^[0-9]{10}$/.test(formData.phone)) {
      newErrors.phone = "Số điện thoại phải có 10 chữ số";
    }
    if (!formData.agreeToTerms) {
      newErrors.agreeToTerms = "Bạn phải đồng ý với điều khoản sử dụng";
    }
    return newErrors;
  }

  async function handleSubmit(e) {
    e.preventDefault();

    const validationErrors = validate();
    if (Object.keys(validationErrors).length > 0) {
      setErrors(validationErrors);
      return;
    }

    setIsSubmitting(true);
    try {
      await submitRegistration(formData);
      alert("Đăng ký thành công!");
    } catch (error) {
      setErrors({ submit: "Đăng ký thất bại. Vui lòng thử lại." });
    } finally {
      setIsSubmitting(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <div>
        <label htmlFor="firstName">Tên *</label>
        <input
          id="firstName"
          name="firstName"
          type="text"
          value={formData.firstName}
          onChange={handleChange}
        />
        {errors.firstName && (
          <span className="error">{errors.firstName}</span>
        )}
      </div>

      <div>
        <label htmlFor="email">Email *</label>
        <input
          id="email"
          name="email"
          type="email"
          value={formData.email}
          onChange={handleChange}
        />
        {errors.email && (
          <span className="error">{errors.email}</span>
        )}
      </div>

      <div>
        <label>
          <input
            type="checkbox"
            name="agreeToTerms"
            checked={formData.agreeToTerms}
            onChange={handleChange}
          />
          Tôi đồng ý với điều khoản sử dụng
        </label>
        {errors.agreeToTerms && (
          <span className="error">{errors.agreeToTerms}</span>
        )}
      </div>

      {errors.submit && (
        <div className="error-banner">{errors.submit}</div>
      )}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Đang gửi..." : "Đăng ký"}
      </button>
    </form>
  );
}
```

### Reset Form

```jsx
function ContactForm() {
  const initialState = { name: "", message: "" };
  const [formData, setFormData] = useState(initialState);

  function handleSubmit(e) {
    e.preventDefault();
    console.log(formData);
    setFormData(initialState); // Reset về trạng thái ban đầu
  }

  function handleReset() {
    setFormData(initialState);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="name"
        value={formData.name}
        onChange={(e) =>
          setFormData((prev) => ({ ...prev, name: e.target.value }))
        }
      />
      <textarea
        name="message"
        value={formData.message}
        onChange={(e) =>
          setFormData((prev) => ({ ...prev, message: e.target.value }))
        }
      />
      <button type="submit">Gửi</button>
      <button type="button" onClick={handleReset}>Đặt lại</button>
    </form>
  );
}
```

---

## 5. Validation — Kiểm Tra Dữ Liệu

### Validation Khi Submit

```jsx
function LoginForm() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [errors, setErrors] = useState({});

  function handleSubmit(e) {
    e.preventDefault();
    const newErrors = {};

    if (!email) {
      newErrors.email = "Email là bắt buộc";
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      newErrors.email = "Định dạng email không hợp lệ";
    }

    if (!password) {
      newErrors.password = "Mật khẩu là bắt buộc";
    } else if (password.length < 8) {
      newErrors.password = "Mật khẩu phải có ít nhất 8 ký tự";
    }

    setErrors(newErrors);

    if (Object.keys(newErrors).length === 0) {
      // Không có lỗi → tiến hành đăng nhập
      console.log("Đăng nhập với:", { email, password });
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className={errors.email ? "input-error" : ""}
          placeholder="Email"
        />
        {errors.email && <p className="error-text">{errors.email}</p>}
      </div>

      <div>
        <input
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          className={errors.password ? "input-error" : ""}
          placeholder="Mật khẩu"
        />
        {errors.password && <p className="error-text">{errors.password}</p>}
      </div>

      <button type="submit">Đăng nhập</button>
    </form>
  );
}
```

### Real-time Validation (Kiểm Tra Theo Thời Gian Thực)

```jsx
function PasswordStrengthInput() {
  const [password, setPassword] = useState("");

  const hasMinLength = password.length >= 8;
  const hasUpperCase = /[A-Z]/.test(password);
  const hasNumber = /\d/.test(password);
  const hasSpecialChar = /[!@#$%^&*]/.test(password);

  const strength =
    [hasMinLength, hasUpperCase, hasNumber, hasSpecialChar].filter(Boolean)
      .length;

  const strengthLabel = ["Rất yếu", "Yếu", "Trung bình", "Mạnh", "Rất mạnh"][strength];

  return (
    <div>
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Nhập mật khẩu"
      />

      {password && (
        <div className="password-strength">
          <div
            className="strength-bar"
            style={{
              width: `${(strength / 4) * 100}%`,
              backgroundColor: ["red", "orange", "yellow", "lightgreen", "green"][strength],
            }}
          />
          <p>Độ mạnh: {strengthLabel}</p>

          <ul>
            <li className={hasMinLength ? "valid" : "invalid"}>
              {hasMinLength ? "✅" : "❌"} Ít nhất 8 ký tự
            </li>
            <li className={hasUpperCase ? "valid" : "invalid"}>
              {hasUpperCase ? "✅" : "❌"} Có chữ hoa
            </li>
            <li className={hasNumber ? "valid" : "invalid"}>
              {hasNumber ? "✅" : "❌"} Có chữ số
            </li>
            <li className={hasSpecialChar ? "valid" : "invalid"}>
              {hasSpecialChar ? "✅" : "❌"} Có ký tự đặc biệt (!@#$%^&*)
            </li>
          </ul>
        </div>
      )}
    </div>
  );
}
```

---

## 6. Câu Hỏi Phỏng Vấn

### Q1: Controlled Component là gì? Giải thích luồng dữ liệu?

**Trả lời:** Controlled Component là form element mà **giá trị được kiểm soát hoàn toàn bởi React state**. Luồng: người dùng gõ → `onChange` event → `setState()` → React re-render → input hiển thị giá trị từ state. React là single source of truth. Lợi ích: luôn biết giá trị input bất kỳ lúc nào, dễ validate real-time, dễ transform input (uppercase, format phone số), dễ reset, dễ disable/enable có điều kiện.

### Q2: Khi nào nên dùng Uncontrolled Component?

**Trả lời:** Hầu hết trường hợp nên dùng Controlled. Dùng Uncontrolled khi: (1) **`<input type="file">`** — bắt buộc vì lý do bảo mật trình duyệt không cho phép set `value` bằng JavaScript. (2) Tích hợp với **thư viện form non-React** như jQuery Form. (3) **Performance critical** với rất nhiều inputs — tránh re-render mỗi keystroke. Ngoài các trường hợp trên, luôn ưu tiên Controlled.

### Q3: Tại sao cần `e.preventDefault()` khi submit form?

**Trả lời:** Mặc định, submit form HTML gây **full page reload (tải lại trang)** — trình duyệt gửi form data đến URL trong `action` attribute và tải lại trang. Trong SPA (Single Page Application — Ứng Dụng Một Trang), ta xử lý submit bằng JavaScript (fetch API, axios), không cần reload. `e.preventDefault()` ngăn hành vi mặc định đó, giữ React app chạy liên tục.

### Q4: `value` và `defaultValue` khác nhau như thế nào?

**Trả lời:** `value` kèm `onChange` → Controlled Component. React kiểm soát giá trị hoàn toàn. `defaultValue` → Uncontrolled Component. React chỉ đặt giá trị ban đầu khi component mount (lần đầu render), sau đó DOM tự quản lý. Nếu dùng `value` mà không có `onChange` → React cảnh báo và input bị readonly. Tương tự: `checked`/`defaultChecked` cho checkbox, `selected`/`defaultValue` cho select.

### Q5: Làm thế nào để reset một controlled form?

**Trả lời:** Đặt tất cả state về giá trị ban đầu. Pattern tốt: lưu `initialState` bên ngoài component và gọi `setFormData(initialState)` khi reset. Với Uncontrolled, dùng `ref.current.value = ""` hoặc HTML native `form.reset()` thông qua `formRef.current.reset()`.

---

## ✅ Checklist Hoàn Thành Topic 01-Fundamentals

Sau khi đọc cả 5 file:

- [ ] Có thể viết JSX thành thạo, hiểu các quy tắc cú pháp
- [ ] Tạo được Functional Components độc lập, có thể tái sử dụng
- [ ] Truyền và nhận Props với destructuring, default values
- [ ] Quản lý State với `useState` đúng cách (không mutate trực tiếp)
- [ ] Xử lý sự kiện với Synthetic Events, biết `preventDefault` và `stopPropagation`
- [ ] Áp dụng conditional rendering với ternary, `&&`, biến trung gian
- [ ] Render danh sách với `key` hợp lệ (ID từ data, không dùng Math.random)
- [ ] Xây dựng Controlled Form với validation cơ bản
- [ ] Biết khi nào dùng Uncontrolled và `ref`

**Bước tiếp theo:** Xây dựng **Todo App** theo hướng dẫn trong [README.md](./README.md)

---

## 🔗 Điều Hướng

- **Tiếp theo:** [02-hooks/README.md](../02-hooks/README.md) — React Hooks chuyên sâu
- **Quay lại:** [4-conditional-list-rendering.md](./4-conditional-list-rendering.md) — Render có điều kiện
- **Chỉ mục:** [INDEX.md](../INDEX.md)
