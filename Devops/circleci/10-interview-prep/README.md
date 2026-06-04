# 🎯 10 — Chuẩn Bị Phỏng Vấn — Interview Prep (CircleCI & CI/CD)

> Module tổng hợp câu hỏi phỏng vấn, câu trả lời mẫu, câu chuyện STAR (Situation — Tình Huống / Task — Nhiệm Vụ / Action — Hành Động / Result — Kết Quả), bài toán thiết kế CI/CD (Continuous Integration / Continuous Deployment — Tích Hợp Liên Tục / Triển Khai Liên Tục), và so sánh công cụ — giúp bạn chuyển kiến thức từ các module 01–09 thành điểm mạnh trong phỏng vấn.

---

## 📁 Cấu Trúc Thư Mục

```
10-interview-prep/
├── README.md                      Tổng quan & lộ trình (file này)
├── 1-INTERVIEW_GUIDE.md           Top 20 câu hỏi CircleCI / CI/CD kèm đáp án
├── 2-star-stories.md              Mẫu & ví dụ câu chuyện incident pipeline
├── 3-system-design-scenarios.md   Kịch bản thiết kế hệ thống CI/CD
└── 4-tool-comparison.md           CircleCI vs Jenkins vs GitHub Actions
```

---

## 🎯 Mục Tiêu Module

Sau khi hoàn thành `10-interview-prep/`, bạn có thể:

- [ ] Trả lời tự tin **20 câu hỏi** hay gặp về CircleCI và CI/CD
- [ ] Kể **2–3 câu chuyện STAR** về tối ưu pipeline, sự cố build, hoặc migration CI
- [ ] Phác thảo **kiến trúc CI/CD** cho monorepo, microservices, hoặc mobile trong ~30 phút
- [ ] **So sánh và chọn** CircleCI, Jenkins, GitHub Actions theo bối cảnh dự án
- [ ] Giải thích **trade-offs** về chi phí (credits), bảo mật (Contexts, OIDC), và tốc độ build

---

## 📋 Checklist Trước Phỏng Vấn

### Kiến Thức — Ôn Từ Các Module

- [ ] Pipeline → Workflow → Job → Step ([01-fundamentals](../01-fundamentals/))
- [ ] `.circleci/config.yml`: executors, commands, parameters ([02-configuration](../02-configuration/))
- [ ] Workflow: `requires`, filters, approval ([03-workflows](../03-workflows/))
- [ ] Orbs — Gói Tích Hợp tái sử dụng ([04-orbs](../04-orbs/))
- [ ] Cache vs Workspace, test splitting, parallelism ([05-optimization](../05-optimization/))
- [ ] Contexts, OIDC — OpenID Connect ([06-security](../06-security/))
- [ ] Deploy AWS/GCP/K8s ([07-integration](../07-integration/))
- [ ] SSH debug, flaky tests ([08-monitoring](../08-monitoring/))
- [ ] Dynamic Config, monorepo ([09-advanced](../09-advanced/))

### Thực Hành — Nên Có Trong CV / Portfolio

- [ ] Ít nhất một pipeline production: test → build → deploy với approval
- [ ] Đã dùng `circleci config validate` hoặc pre-commit validate
- [ ] Một ví dụ giảm thời gian build (số liệu % hoặc phút)
- [ ] Một ví dụ quản lý secret an toàn (Context hoặc OIDC)

### Câu Chuyện — Chuẩn Bị STAR

- [ ] Pipeline chậm / OOM (Out Of Memory — Hết Bộ Nhớ) / timeout
- [ ] Secret lộ hoặc hardening sau audit
- [ ] Migration từ Jenkins/GitHub Actions sang CircleCI (hoặc ngược lại)
- [ ] Monorepo: chỉ chạy CI cho service bị ảnh hưởng

---

## 🗺️ Lộ Trình 2 Tuần

### Tuần 1 — Lý Thuyết & Q&A

| Ngày | Nội dung | File |
| ---- | -------- | ---- |
| 1–2 | CI/CD, kiến trúc CircleCI, executors | `1-INTERVIEW_GUIDE.md` Câu 1–5 |
| 3–4 | Config YAML, workflow, optimization | Câu 6–12 |
| 5–6 | Security, cloud deploy, advanced | Câu 13–20 |
| 7 | Ôn lại + ghi chú điểm yếu cá nhân | `4-tool-comparison.md` |

### Tuần 2 — Kể Chuyện & Thiết Kế

| Ngày | Nội dung | File |
| ---- | -------- | ---- |
| 1–2 | Viết & nói to 3 STAR stories | `2-star-stories.md` |
| 3–4 | Vẽ sơ đồ 3 kịch bản system design | `3-system-design-scenarios.md` |
| 5 | Mock interview (đồng nghiệp hoặc tự ghi âm) | Toàn module |
| 6–7 | Ôn top 10 câu + nghỉ ngơi | `1-INTERVIEW_GUIDE.md` |

---

## 💡 Câu Hỏi Nên Hỏi Ngược Interviewer

1. **"Team đang dùng CircleCI cloud hay Self-Hosted Runner — Máy Chạy Tự Quản Lý?"** — Cho thấy bạn nghĩ đến vận hành thực tế.
2. **"Chiến lược deploy production: approval manual, GitOps, hay CD tự động?"** — Liên quan workflow design.
3. **"Monorepo hay multi-repo? Có Dynamic Config / Path Filtering chưa?"** — Đúng pain point scale-up.
4. **"Metric nào team coi là thành công CI: lead time, MTTR (Mean Time To Recovery — Thời Gian Phục Hồi Trung Bình), hay % flaky tests?"** — Thể hiện DevOps mindset.

---

## 📊 Ma Trận Câu Hỏi Theo Cấp Độ

| Cấp độ | Ví dụ |
| ------ | ----- |
| Junior | "Job khác Workflow thế nào?" |
| Mid | "Thiết kế pipeline Node.js deploy lên AWS với approval" |
| Senior | "CI/CD cho monorepo 20 service, giảm credit và thời gian chờ" |
| Staff/Architect | "Chuẩn hóa CI/CD đa team, compliance, cost governance" |

---

## 🔗 Điều Hướng

| File | Mô tả | Thời gian ôn (ước lượng) |
| ---- | ----- | ------------------------- |
| [1-INTERVIEW_GUIDE.md](./1-INTERVIEW_GUIDE.md) | Top 20 Q&A | 4–6 giờ |
| [2-star-stories.md](./2-star-stories.md) | STAR templates | 2–3 giờ |
| [3-system-design-scenarios.md](./3-system-design-scenarios.md) | 4 kịch bản design | 3–5 giờ |
| [4-tool-comparison.md](./4-tool-comparison.md) | So sánh công cụ | 1–2 giờ |

← [09-advanced/](../09-advanced/README.md) | [INDEX.md](../INDEX.md)

---

**Cập Nhật Lần Cuối:** 2026-05-20  
**Độ Khó Module:** ⭐⭐⭐ (Tổng hợp kiến thức toàn lộ trình)
