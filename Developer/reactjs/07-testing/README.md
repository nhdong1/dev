# 07 — Testing (Kiểm Thử React)

> **Testing** (Kiểm Thử) là quá trình xác minh ứng dụng React hoạt động đúng như kỳ vọng từ góc nhìn người dùng và lập trình viên. Nguyên tắc cốt lõi: **"Viết tests theo cách người dùng tương tác — không phải theo cách code hoạt động nội bộ"** (Kent C. Dodds — tác giả React Testing Library).

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Viết **unit tests** (kiểm thử đơn vị) với **Jest** — framework kiểm thử JavaScript phổ biến nhất
- [ ] Kiểm thử React components theo góc nhìn người dùng với **RTL** — React Testing Library (Thư Viện Kiểm Thử React)
- [ ] **Mock** (Giả Lập) API calls, modules, custom hooks để test độc lập
- [ ] Viết **integration tests** (kiểm thử tích hợp) với **MSW** — Mock Service Worker (Trình Giả Lập Dịch Vụ Web)
- [ ] Sử dụng **Vitest** — công cụ kiểm thử nhanh hơn cho Vite projects
- [ ] Triển khai **E2E testing** (kiểm thử đầu cuối) với **Playwright** và **Cypress**
- [ ] Chọn đúng loại test cho từng trường hợp — tránh over-testing và under-testing

---

## 📁 Nội Dung

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-jest-basics.md](./1-jest-basics.md) | Jest — unit testing framework, matchers, async tests | Beginner–Intermediate |
| [2-react-testing-library.md](./2-react-testing-library.md) | RTL — render, query, fireEvent, userEvent | Intermediate |
| [3-mocking.md](./3-mocking.md) | Mock API, modules, custom hooks, timers | Intermediate |
| [4-integration-testing.md](./4-integration-testing.md) | Integration tests với MSW — Mock Service Worker | Intermediate–Advanced |
| [5-vitest.md](./5-vitest.md) | Vitest — thay thế Jest cho Vite, nhanh hơn, config ít hơn | Intermediate |
| [6-e2e-testing.md](./6-e2e-testing.md) | E2E với Playwright và Cypress — test toàn bộ luồng người dùng | Advanced |

---

## 🗺️ Lộ Trình Học

```
Jest Basics → RTL → Mocking → Integration (MSW) → Vitest → E2E
     ↓           ↓       ↓           ↓               ↓        ↓
 "Unit test   "Test    "Giả lập   "Test tích      "Nhanh   "Test
  cơ bản"   component  phụ thuộc"  hợp HTTP"       hơn"    toàn luồng"
```

**Thứ tự khuyến nghị:**
1. Bắt đầu với `1-jest-basics.md` — hiểu nền tảng unit testing
2. Học `2-react-testing-library.md` — cách test component đúng cách
3. Đọc `3-mocking.md` — kiểm soát dependencies khi test
4. Học `4-integration-testing.md` — test luồng đầy đủ với MSW
5. Tham khảo `5-vitest.md` — chuyển sang Vitest nếu dùng Vite
6. Hoàn thành `6-e2e-testing.md` — test end-to-end với Playwright/Cypress

---

## ⏱️ Thời Gian Ước Tính

| Hoạt Động | Thời Gian |
| --------- | --------- |
| Đọc lý thuyết (6 files) | 4–5 giờ |
| Thực hành code theo ví dụ | 3–4 giờ |
| Mini-project: Viết test suite cho Todo App | 3–4 giờ |
| **Tổng cộng** | **10–13 giờ** |

---

## 💡 Tại Sao Testing Quan Trọng?

### Testing Không Phải Là Xa Xỉ

```
Không có tests:
  → Refactor code → Sợ breaking changes (thay đổi làm hỏng thứ khác)
  → Deploy → Pray (cầu nguyện không có bug)
  → Bug in production → Debug mất hàng giờ

Có tests:
  → Refactor code → Chạy tests → Biết ngay có gì bị hỏng
  → Deploy → Tự tin vì CI pipeline đã chạy tests
  → Bug → Tests giúp reproduce (tái hiện) và fix nhanh hơn
```

### Lợi Ích Thực Tế

- **Safety net** (Lưới An Toàn): Tự tin refactor và thêm tính năng mới
- **Documentation** (Tài Liệu Sống): Tests mô tả behavior (hành vi) của code
- **Faster debugging** (Debug Nhanh Hơn): Failing test chỉ đúng chỗ có vấn đề
- **Design feedback** (Phản Hồi Thiết Kế): Code khó test thường là code thiết kế kém
- **Regression prevention** (Ngăn Hồi Quy): Không để bug cũ quay lại

---

## 🏗️ Testing Pyramid — Kim Tự Tháp Kiểm Thử

```
           /\
          /  \
         / E2E\          ← Ít nhất, chậm nhất, tốn kém nhất
        /------\            (Playwright, Cypress)
       /        \
      / Integra- \        ← Vừa phải (MSW + RTL)
     /   tion     \
    /--------------\
   /                \
  /   Unit Tests     \   ← Nhiều nhất, nhanh nhất, rẻ nhất
 /--------------------\     (Jest + RTL)
```

### Tỉ Lệ Khuyến Nghị

| Loại Test | Tỉ Lệ | Công Cụ | Tốc Độ |
| --------- | ----- | ------- | ------ |
| **Unit Tests** (Kiểm Thử Đơn Vị) | 70% | Jest / Vitest | < 1ms/test |
| **Integration Tests** (Kiểm Thử Tích Hợp) | 20% | RTL + MSW | 10–100ms/test |
| **E2E Tests** (Kiểm Thử Đầu Cuối) | 10% | Playwright / Cypress | 1–30s/test |

> **Lưu ý:** Đây là hướng dẫn, không phải quy tắc cứng nhắc. Tùy dự án, tỉ lệ có thể khác nhau.

---

## 🔑 Triết Lý Testing Trong React

### Nguyên Tắc RTL Cốt Lõi

```
"The more your tests resemble the way your software is used,
 the more confidence they can give you."
 — Kent C. Dodds

 "Càng test giống cách người dùng sử dụng phần mềm,
  bạn càng tự tin hơn vào sự đúng đắn của nó."
```

### ❌ Không Nên Test

```jsx
// ❌ Test implementation details (chi tiết triển khai)
// → Test này sẽ bị vỡ khi refactor dù behavior không đổi
test('should set isLoading to true', () => {
  const { result } = renderHook(() => useUser());
  expect(result.current.isLoading).toBe(true); // Test state nội bộ
});

// ❌ Test component internals (nội bộ component)
const wrapper = shallow(<Button />); // Enzyme shallow render — lỗi thời
expect(wrapper.find('span').text()).toBe('Click'); // Quan tâm DOM structure
```

### ✅ Nên Test

```jsx
// ✅ Test behavior từ góc nhìn người dùng
test('should show loading spinner while fetching', async () => {
  render(<UserProfile userId="1" />);
  
  // Người dùng thấy gì?
  expect(screen.getByRole('status', { name: /loading/i })).toBeInTheDocument();
  
  // Sau khi load xong
  await waitFor(() => {
    expect(screen.getByText('Nguyen Van A')).toBeInTheDocument();
  });
});
```

---

## 📊 So Sánh Công Cụ Testing

### Unit & Component Testing

| Tiêu Chí | Jest + RTL | Vitest + RTL | Jest + Enzyme |
| -------- | ---------- | ------------ | ------------ |
| **Tốc độ** (speed) | Trung bình | Nhanh hơn 20-30% | Trung bình |
| **Cấu hình** (config) | Trung bình | Ít hơn (Vite native) | Phức tạp |
| **Ecosystem** | Rất phong phú | Đang phát triển | Legacy |
| **React 18+** | ✅ | ✅ | ⚠️ Vấn đề tương thích |
| **HMR** (Hot Module Replacement) | ❌ | ✅ | ❌ |
| **Khuyến nghị** | CRA, Next.js | Vite projects | Không khuyến nghị |

### E2E Testing

| Tiêu Chí | Playwright | Cypress | Selenium |
| -------- | ---------- | ------- | -------- |
| **Tốc độ** | Nhanh nhất | Nhanh | Chậm |
| **Multi-browser** | Chrome, FF, Safari, Edge | Chrome, FF, Edge | Tất cả |
| **API** | Async/await tự nhiên | Custom chain | WebDriver API |
| **Debug** | Trace viewer, screenshot | Time-travel debug | Khó |
| **CI/CD** | Xuất sắc | Tốt | Được |
| **Khuyến nghị** | Dự án mới | Dự án có CI Cypress | Legacy |

---

## ⚙️ Cài Đặt Nhanh

### Jest + RTL (CRA / Next.js)

```bash
# Next.js — đã có Jest config sẵn
npm install --save-dev @testing-library/react @testing-library/jest-dom @testing-library/user-event

# CRA — Jest đã được tích hợp sẵn, chỉ cần thêm RTL
npm install --save-dev @testing-library/react @testing-library/user-event
```

### Vitest + RTL (Vite)

```bash
npm install --save-dev vitest @vitest/ui @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

### MSW — Mock Service Worker

```bash
npm install --save-dev msw
```

### Playwright

```bash
npm install --save-dev @playwright/test
npx playwright install # Cài trình duyệt
```

### Cypress

```bash
npm install --save-dev cypress
npx cypress open # Mở Cypress UI lần đầu
```

---

## 📁 Cấu Trúc Thư Mục Tests

### Cách 1: Đặt test gần component (khuyến nghị)

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx     ← Unit test
│   │   └── Button.stories.tsx  ← Storybook
│   └── UserProfile/
│       ├── UserProfile.tsx
│       └── UserProfile.test.tsx
├── hooks/
│   ├── useAuth.ts
│   └── useAuth.test.ts
└── __tests__/
    └── integration/            ← Integration tests
        └── checkout.test.tsx
```

### Cách 2: Thư mục tests riêng

```
src/
├── components/
│   ├── Button.tsx
│   └── UserProfile.tsx
└── __tests__/
    ├── unit/
    │   ├── Button.test.tsx
    │   └── UserProfile.test.tsx
    ├── integration/
    │   └── checkout.test.tsx
    └── e2e/
        └── checkout.spec.ts
```

---

## 🔨 Mini-Project: Test Suite Cho Todo App

Sau khi hoàn thành cả 6 file, hãy viết test suite đầy đủ cho **Todo App**:

```
Yêu cầu kiểm thử:
✅ Unit: TodoItem component — render, check/uncheck, delete
✅ Unit: useTodos custom hook — add, remove, toggle
✅ Integration: Thêm todo mới (input → submit → hiển thị)
✅ Integration: Filter todos (All / Active / Completed)
✅ Integration: Đồng bộ với API (MSW giả lập)
✅ E2E: Luồng đầy đủ từ tạo đến hoàn thành todo (Playwright)
```

**Mục tiêu coverage** (độ phủ kiểm thử):
- Statements: > 80%
- Branches: > 70%
- Functions: > 80%

---

## 🔗 Điều Hướng

- **Trước đó:** [06-performance/](../06-performance/) — Tối ưu hiệu năng React
- **Tiếp theo:** [08-styling/](../08-styling/) — Tạo kiểu dáng UI
- **Quay lại:** [README.md tổng quan](../README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
