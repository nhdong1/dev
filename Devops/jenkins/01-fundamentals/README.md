# 01 — Fundamentals: Nền Tảng Jenkins

> Kiến trúc, cài đặt, cấu hình và các khái niệm cốt lõi cần nắm trước khi tiến sang bất kỳ chủ đề nào khác.

## Mục Tiêu Học

Sau khi hoàn thành chủ đề này, bạn có thể:

- Giải thích mô hình **Master/Agent** (chủ–tác nhân) và vẽ sơ đồ kiến trúc không cần tài liệu
- Phân biệt **Executor** (bộ thực thi), **Build Queue** (hàng đợi build), **Workspace** (không gian làm việc)
- Cài đặt Jenkins theo 3 phương thức: Standalone, Docker, Kubernetes
- Cấu hình Jenkins cơ bản: JVM, URL, Global Tool, System Config
- Mô tả chính xác các khái niệm: Job, Build, Artifact, View

---

## Nội Dung Chủ Đề

| File | Nội Dung | Độ Quan Trọng |
|------|----------|---------------|
| [1-architecture.md](1-architecture.md) | Master/Agent, Executor, Build Queue, Workspace | ⭐⭐⭐ Bắt buộc |
| [2-installation.md](2-installation.md) | Cài đặt Standalone, Docker, Kubernetes | ⭐⭐ Nên biết |
| [3-configuration.md](3-configuration.md) | System config, JVM, Global Tool | ⭐⭐ Nên biết |
| [4-jenkins-concepts.md](4-jenkins-concepts.md) | Job, Build, Artifact, View | ⭐⭐⭐ Bắt buộc |

---

## Tóm Tắt Kiến Trúc

```
┌─────────────────────────────────────────────────────────────┐
│                    Jenkins Controller (Master)               │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Job Config  │  │  Build Queue │  │   Plugin Manager │  │
│  └──────────────┘  └──────┬───────┘  └──────────────────┘  │
│                            │                                  │
│         ┌──────────────────┴──────────────────┐              │
│         ▼                                      ▼              │
│  ┌─────────────┐                       ┌─────────────┐       │
│  │  Agent #1   │                       │  Agent #2   │       │
│  │  ┌───────┐  │                       │  ┌───────┐  │       │
│  │  │ Exec1 │  │                       │  │ Exec1 │  │       │
│  │  ├───────┤  │                       │  ├───────┤  │       │
│  │  │ Exec2 │  │                       │  │ Exec2 │  │       │
│  │  └───────┘  │                       │  └───────┘  │       │
│  │  Workspace  │                       │  Workspace  │       │
│  └─────────────┘                       └─────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

**Controller** điều phối, **Agent** thực thi. Đây là nguyên tắc phân tách trách nhiệm cốt lõi.

---

## Câu Hỏi Ôn Tập

1. Tại sao không nên chạy build trực tiếp trên Jenkins Controller?
2. Executor và Agent khác nhau như thế nào?
3. Workspace được tạo ở đâu — trên Controller hay Agent?
4. Build Queue hoạt động theo nguyên tắc nào khi không có Executor rảnh?
5. Kể 3 phương thức kết nối Agent với Controller.

---

## Thứ Tự Học Đề Xuất

```
1-architecture.md  →  4-jenkins-concepts.md  →  2-installation.md  →  3-configuration.md
```

Học kiến trúc và khái niệm trước khi đụng vào cài đặt — hiểu WHY trước HOW.

---

**Thời Gian Ước Lượng:** 3–4 giờ  
**Độ Khó:** ⭐ Beginner  
**Tiên Quyết:** Không có — đây là điểm khởi đầu
