# 09 — Ecosystem — Hệ Sinh Thái React

> Tổng quan về các framework, công cụ build, và thư viện bổ trợ tạo nên hệ sinh thái React hiện đại

---

## 🗂️ Nội Dung Chương Này

| File | Chủ Đề | Mức Độ |
| ---- | ------- | ------ |
| [1-nextjs-app-router.md](./1-nextjs-app-router.md) | Next.js App Router — Server/Client Components, RSC | ⭐⭐⭐ |
| [2-nextjs-pages-router.md](./2-nextjs-pages-router.md) | Next.js Pages Router — SSR, SSG, ISR | ⭐⭐ |
| [3-remix.md](./3-remix.md) | Remix — Web Standards, Loader, Action | ⭐⭐ |
| [4-vite-tooling.md](./4-vite-tooling.md) | Vite, ESBuild, Turbopack — Build tooling | ⭐⭐ |
| [5-typescript-react.md](./5-typescript-react.md) | TypeScript với React — Typing Components, Hooks | ⭐⭐⭐ |
| [6-storybook-monorepo.md](./6-storybook-monorepo.md) | Storybook, Turborepo, Nx — Dev Tools & Monorepo | ⭐⭐ |

---

## 🎯 Tại Sao Cần Học Hệ Sinh Thái?

React là một **thư viện UI** (User Interface Library — Thư Viện Giao Diện Người Dùng), không phải framework đầy đủ. Để xây dựng ứng dụng thực tế cần:

- **Meta-framework** (Next.js, Remix) — xử lý routing, SSR, data fetching
- **Build tool** (Vite, Turbopack) — đóng gói và tối ưu code
- **Type system** (TypeScript) — an toàn kiểu tĩnh
- **Dev tooling** (Storybook, Monorepo tools) — workflow phát triển chuyên nghiệp

---

## 🏗️ Bức Tranh Tổng Thể — Big Picture

```
┌─────────────────────────────────────────────────────────┐
│                    Ứng Dụng React                       │
├────────────────┬────────────────┬───────────────────────┤
│  Meta-Framework│   Build Tool   │    Dev Tooling        │
│                │                │                       │
│  Next.js       │  Vite          │  Storybook            │
│  Remix         │  Turbopack     │  Turborepo            │
│  Gatsby        │  ESBuild       │  Nx                   │
├────────────────┴────────────────┴───────────────────────┤
│                  TypeScript (Type Safety)                │
├─────────────────────────────────────────────────────────┤
│               React Core (UI Library)                   │
└─────────────────────────────────────────────────────────┘
```

---

## 🆚 So Sánh Meta-Framework

| Tiêu Chí | Next.js App Router | Next.js Pages Router | Remix |
| -------- | ------------------ | -------------------- | ----- |
| **Rendering** | RSC + SSR + SSG | SSR + SSG + ISR | SSR (server-first) |
| **Routing** | File-based (app/) | File-based (pages/) | File-based (routes/) |
| **Data Fetching** | async Server Components | getServerSideProps / getStaticProps | loader / action |
| **Mutations** | Server Actions | API Routes | action |
| **Bundle Size** | Trung bình | Trung bình | Nhỏ hơn |
| **Learning Curve** | Cao (khái niệm mới) | Thấp hơn | Trung bình |
| **Streaming** | Hỗ trợ tốt | Hạn chế | Hỗ trợ tốt |
| **Phù hợp** | Ứng dụng lớn, enterprise | Dự án kế thừa | Web standards thuần |

---

## ⚡ So Sánh Build Tool

| Tiêu Chí | Vite | Turbopack | Create React App (CRA) |
| -------- | ---- | --------- | ---------------------- |
| **Ngôn ngữ core** | Go (ESBuild) | Rust | JavaScript |
| **Dev server** | Cực nhanh (ESM native) | Rất nhanh | Chậm |
| **HMR** | < 50ms | < 10ms | Chậm |
| **Tích hợp** | Vite projects | Next.js 13+ | Legacy |
| **Cộng đồng** | Rất lớn | Đang phát triển | Không còn được duy trì |
| **Khuyến nghị** | ✅ SPA, Vite apps | ✅ Next.js | ❌ Không dùng mới |

---

## 🗺️ Lộ Trình Học Trong Chương Này

### Bước 1 — Nắm Next.js (Quan trọng nhất)

```
1-nextjs-app-router.md  ← Bắt đầu tại đây (App Router là tương lai)
2-nextjs-pages-router.md ← Hiểu để maintain dự án cũ
```

### Bước 2 — TypeScript (Bắt buộc với dự án thực tế)

```
5-typescript-react.md ← Typing chặt = ít bug hơn
```

### Bước 3 — Build Tools & Dev Workflow

```
4-vite-tooling.md    ← Setup project React không dùng Next.js
6-storybook-monorepo.md ← Quy mô lớn, nhiều package
```

### Bước 4 — Remix (Tùy chọn nếu quan tâm)

```
3-remix.md ← Web standards approach, khác tư duy Next.js
```

---

## 🎯 Câu Hỏi Phỏng Vấn Hay Gặp

- Next.js App Router khác Pages Router như thế nào?
- Server Components (SC — Component Phía Server) có thể làm gì mà Client Components không làm được?
- ISR — Incremental Static Regeneration — Tái Tạo Tĩnh Tăng Dần là gì?
- Vite nhanh hơn Webpack vì lý do gì?
- TypeScript giúp gì cho một dự án React lớn?
- Monorepo (Kho Lưu Trữ Đơn) vs Polyrepo — khi nào nên dùng?

---

## 📖 Tài Liệu Tham Khảo

- [Next.js Docs](https://nextjs.org/docs) — Tài liệu chính thức Next.js
- [Remix Docs](https://remix.run/docs) — Tài liệu chính thức Remix
- [Vite Docs](https://vitejs.dev) — Tài liệu chính thức Vite
- [TypeScript Handbook](https://www.typescriptlang.org/docs/) — Sách hướng dẫn TypeScript
- [Storybook Docs](https://storybook.js.org/docs) — Tài liệu Storybook

---

**Cập Nhật Lần Cuối:** 2026-06-04
**React Version:** v19 | **Next.js Version:** 15 | **TypeScript:** 5.x
