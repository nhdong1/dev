# 10 — Advanced Patterns (Mẫu Thiết Kế Nâng Cao trong React)

> **Advanced Patterns** (Mẫu Thiết Kế Nâng Cao) là các kỹ thuật tổ chức code và thiết kế component API được đúc kết từ kinh nghiệm thực tế của cộng đồng React. Nắm vững các pattern này giúp bạn xây dựng component linh hoạt, tái sử dụng cao và dễ bảo trì — dấu hiệu của **Senior Frontend Engineer**.

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Thiết kế **Compound Components** (Component Hợp Thành) với API linh hoạt, dễ sử dụng
- [ ] Áp dụng **Render Props** (Props Hàm Render) để chia sẻ logic giữa các component
- [ ] Xây dựng **Higher-Order Components — HOC** (Component Bậc Cao) để tái sử dụng behavior
- [ ] Thiết kế **Custom Hooks nâng cao** với composition và state machines
- [ ] Sử dụng **Portals** (Cổng Render) để render ngoài DOM hierarchy
- [ ] Implement **Error Boundaries** (Ranh Giới Bắt Lỗi) để xử lý lỗi gracefully
- [ ] Xây dựng ứng dụng **Accessible** (Có Khả Năng Tiếp Cận) theo chuẩn **WCAG 2.1** (Web Content Accessibility Guidelines — Hướng Dẫn Tiếp Cận Nội Dung Web)

---

## 📁 Nội Dung

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-compound-components.md](./1-compound-components.md) | Compound Components — API linh hoạt, Context nội bộ, controlled/uncontrolled | Advanced |
| [2-render-props.md](./2-render-props.md) | Render Props — chia sẻ logic qua prop hàm, so sánh với Custom Hooks | Intermediate–Advanced |
| [3-hoc.md](./3-hoc.md) | Higher-Order Components (HOC) — bọc component, thêm behavior, compose HOCs | Advanced |
| [4-custom-hooks-patterns.md](./4-custom-hooks-patterns.md) | Custom Hooks nâng cao — state machines, composition, factory hooks | Advanced |
| [5-portals-boundaries.md](./5-portals-boundaries.md) | Portals và Error Boundaries — render ra ngoài DOM, bắt lỗi component tree | Intermediate–Advanced |
| [6-accessibility.md](./6-accessibility.md) | A11y — ARIA, keyboard navigation, screen readers, WCAG 2.1 | Intermediate–Advanced |

---

## 🗺️ Lộ Trình Học

```
Compound Components → Render Props → HOC → Custom Hooks Patterns → Portals & Boundaries → A11y
        ↓                  ↓           ↓            ↓                       ↓                ↓
  "API Component      "Logic         "Behavior   "Reusable               "DOM            "Inclusive
   linh hoạt,          chia sẻ        tái sử      logic phức              escape          design,
   sub-components"     qua prop"      dụng"       tạp"                    hatch"          WCAG"
```

**Thứ tự khuyến nghị:**
1. `1-compound-components.md` — pattern phổ biến nhất, hay gặp trong phỏng vấn senior
2. `2-render-props.md` — hiểu lịch sử, vì sao Custom Hooks thay thế
3. `3-hoc.md` — vẫn xuất hiện nhiều trong codebase legacy và libraries
4. `4-custom-hooks-patterns.md` — pattern hiện đại nhất, áp dụng hằng ngày
5. `5-portals-boundaries.md` — kỹ thuật cần thiết cho modals, tooltips, error UX
6. `6-accessibility.md` — thường bị bỏ qua nhưng quan trọng ở cấp senior/lead

---

## ⏱️ Thời Gian Ước Tính

| Hoạt Động | Thời Gian |
| --------- | --------- |
| Đọc lý thuyết (6 files) | 4–5 giờ |
| Thực hành code theo ví dụ | 3–4 giờ |
| Mini-project: Build Component Library Pattern | 3–4 giờ |
| **Tổng cộng** | **10–13 giờ** |

---

## 💡 Tại Sao Advanced Patterns Quan Trọng?

### Vấn Đề Mà Các Pattern Này Giải Quyết

```
Bài toán thực tế khi xây dựng component libraries:

1. Rigid API (API Cứng Nhắc):
   Xấu: <Select options={[...]} onChange={fn} value={v} />
   Tốt: <Select><Select.Option value="a">A</Select.Option></Select>
   → Compound Components giải quyết điều này

2. Logic Duplication (Trùng Lặp Logic):
   Xấu: Copy-paste xử lý hover, focus, click vào nhiều component
   Tốt: Chia sẻ qua Custom Hook hoặc Render Props
   → Render Props & Custom Hooks giải quyết

3. Cross-cutting Concerns (Mối Quan Tâm Xuyên Cắt):
   Xấu: Thêm logging, auth check, error handling vào từng component
   Tốt: Bọc component bằng HOC
   → HOC giải quyết

4. DOM Constraints (Ràng Buộc DOM):
   Xấu: Modal render trong parent div, bị ảnh hưởng bởi overflow: hidden
   Tốt: Render thẳng vào document.body với Portal
   → Portals giải quyết

5. Unhandled Errors (Lỗi Không Được Xử Lý):
   Xấu: Một component crash làm toàn bộ app trắng màn hình
   Tốt: Error Boundary bắt lỗi, hiển thị fallback UI
   → Error Boundaries giải quyết
```

---

## 🆚 So Sánh Nhanh Các Pattern

| Pattern | Mục Đích | Khi Nào Dùng | Nhược Điểm |
| ------- | -------- | ------------ | ---------- |
| **Compound Components** | API component linh hoạt | Component có nhiều sub-parts (Select, Tabs, Accordion) | Cần Context nội bộ |
| **Render Props** | Chia sẻ stateful logic | Trước React Hooks; hiện tại ít dùng | Callback hell, verbose |
| **HOC** | Thêm behavior vào component | Cross-cutting concerns, legacy code | Wrapper hell, props collision |
| **Custom Hooks** | Tái sử dụng stateful logic | Hầu hết mọi trường hợp hiện đại | Chỉ dùng được trong functional components |
| **Portals** | Render ngoài DOM parent | Modals, tooltips, dropdowns | Cần quản lý z-index, focus trap |
| **Error Boundaries** | Bắt lỗi render | Mọi app production | Chỉ dùng class component |

---

## 📊 Pattern Nào Phổ Biến Nhất (2025–2026)?

```
Xu hướng hiện tại:

⭐⭐⭐⭐⭐ Custom Hooks — Pattern số 1, áp dụng mọi nơi
⭐⭐⭐⭐⭐ Compound Components — Phổ biến trong design systems
⭐⭐⭐⭐   Error Boundaries — Bắt buộc trong production
⭐⭐⭐⭐   Portals — Cần cho modals, tooltips đúng chuẩn
⭐⭐⭐    HOC — Vẫn dùng trong libraries, legacy code
⭐⭐     Render Props — Đang được thay bởi Custom Hooks
```

---

## 🎯 Liên Hệ Với Phỏng Vấn

### Câu Hỏi Thường Gặp Về Advanced Patterns

```
Beginner → Intermediate:
- "Compound Components là gì? Cho ví dụ thực tế?"
- "Khi nào dùng HOC, khi nào dùng Custom Hooks?"
- "Error Boundary hoạt động như thế nào?"

Intermediate → Senior:
- "Thiết kế API cho Select component hỗ trợ single/multi-select?"
- "HOC vs Render Props vs Custom Hooks — trade-offs?"
- "Implement một accessible Modal từ đầu?"

Senior → Lead:
- "Xây dựng Component Library sử dụng Compound Components pattern?"
- "Chiến lược Error Boundary ở cấp độ ứng dụng lớn?"
- "WCAG 2.1 compliance — kiểm tra và đảm bảo như thế nào?"
```

---

## 🔑 Khái Niệm Cốt Lõi Cần Nắm

### Inversion of Control (IoC) — Đảo Ngược Quyền Kiểm Soát

```
Ý tưởng cốt lõi của Advanced Patterns:
"Thay vì component quyết định mọi thứ, hãy trao quyền cho người dùng component"

Ví dụ:
// ❌ Component kiểm soát tất cả — rigid
<DataTable
  columns={columns}
  sortable={true}
  filterable={true}
  pagination={{ pageSize: 10 }}
/>

// ✅ Người dùng kiểm soát — flexible (IoC)
<DataTable data={data}>
  <DataTable.Header>
    <DataTable.Column sortable>Name</DataTable.Column>
  </DataTable.Header>
  <DataTable.Body />
  <DataTable.Pagination pageSize={10} />
</DataTable>
```

### Separation of Concerns (SoC) — Tách Biệt Mối Quan Tâm

```
Mỗi pattern phục vụ một trách nhiệm:
- Compound Components → Cấu trúc và API
- Custom Hooks        → Logic và state
- HOC                → Cross-cutting behavior
- Portals            → DOM positioning
- Error Boundaries   → Error handling
- A11y               → Inclusive UX
```

---

## 🔗 Điều Hướng

- **Trước đó:** [09-ecosystem/](../09-ecosystem/) — Hệ sinh thái React
- **Tiếp theo:** [11-interview-prep/](../11-interview-prep/) — Chuẩn bị phỏng vấn
- **Quay lại:** [README.md tổng quan](../README.md)
- **Chỉ mục:** [INDEX.md](../INDEX.md)
