# 🆚 So Sánh Công Cụ CI/CD — CircleCI vs Jenkins vs GitHub Actions

> Tài liệu phỏng vấn: so sánh có cấu trúc, không thiên vị. Dùng kèm [01-fundamentals/5-circleci-vs-alternatives.md](../01-fundamentals/5-circleci-vs-alternatives.md) để đi sâu ví dụ config.

---

## 🎯 Câu Hỏi Phỏng Vấn Điển Hình

> "Tại sao team bạn chọn CircleCI?" / "Khi nào bạn khuyên chuyển sang GitHub Actions?"

**Khung trả lời 60 giây:**

1. **Bối cảnh:** Quy mô team, VCS, compliance, ngân sách.  
2. **Yêu cầu kỹ thuật:** Tốc độ build, monorepo, mobile, on-prem.  
3. **Khuyến nghị:** Một công cụ chính + tiêu chí rõ; tránh "cài thêm CI thứ hai" không kiểm soát.

---

## 📊 Bảng So Sánh Tổng Hợp

| Tiêu chí | **CircleCI** | **Jenkins** | **GitHub Actions (GHA)** |
| -------- | ------------ | ----------- | ------------------------ |
| Mô hình | SaaS + optional self-hosted runner | Self-managed server/agents | SaaS tích hợp GitHub |
| Cấu hình | `.circleci/config.yml` | `Jenkinsfile` (Groovy) / YAML plugins | `.github/workflows/*.yml` |
| Learning curve | Trung bình | Cao (ops + Groovy) | Thấp–trung bình |
| Tốc độ build | Cao (resource class, infra riêng) | Phụ thuộc hardware | Khả biến; lớn có thể chậm |
| Tái sử dụng | **Orbs** | **Plugins** (1800+) | **Actions** Marketplace |
| Debug | **SSH vào job** | SSH agent / replay | Khó hơn; phụ thuộc tmate/self-hosted |
| Test splitting | Native `circleci tests split` | Plugin / tự script | Matrix + script tùy chỉnh |
| Chi phí | Credits (private); free tier có hạn | Server + nhân sự vận hành | Free public; private có minute cap |
| Lock-in | Trung bình (config CircleCI) | Thấp (self-host) | Cao nếu all-in GitHub |
| Phù hợp | Team muốn CI mạnh, ít ops | Enterprise on-prem, tùy biến sâu | GitHub-centric, OSS |

---

## ⚙️ CircleCI — Khi Nào Là Lựa Chọn Tốt

**Chọn khi:**

- Muốn **time-to-value** nhanh, không có đội vận hành Jenkins.  
- Pipeline dài cần **parallelism + timing-based test split**.  
- Cần **Pipeline Insights** và flaky test detection.  
- **Dynamic Config / path filtering** cho monorepo.  
- Cần **Rerun with SSH** debug thường xuyên.

**Hạn chế:**

- Chi phí credit khi scale team lớn + parallelism cao.  
- Không "native" bằng GHA nếu toàn bộ workflow nằm trên GitHub UI/PR.

---

## 🏭 Jenkins — Khi Nào Là Lựa Chọn Tốt

**Chọn khi:**

- **Data residency / air-gapped** — build trong DC nội bộ.  
- Cần plugin cho hệ thống legacy (mainframe, đặc thù).  
- Đã có **Jenkins ops mature** và sunk cost lớn.

**Hạn chế:**

- **MTTR** cho Jenkins master/agent thường cao hơn SaaS.  
- Bảo mật plugin cần quy trình riêng (CVE, pin version).

**Câu phỏng vấn:** "Migrate Jenkins → CircleCI?" → Làm từng repo, map stages → jobs, thay credentials bằng Contexts/OIDC, giữ Jenkins song song đến khi parity test.

---

## 🐙 GitHub Actions — Khi Nào Là Lựa Chọn Tốt

**Chọn khi:**

- **100% GitHub**; muốn PR checks, environments, deployments trong một nơi.  
- **Open source** — free generous.  
- Workflow đơn giản (lint, test, publish package).

**Hạn chế:**

- Org lớn đôi khi gặp **queue** hoặc minute limits.  
- Advanced CI (timing split, cost analytics) kém tiện hơn CircleCI cho một số team.

---

## 🧭 Decision Framework — Khung Quyết Định

```
                    ┌─────────────────┐
                    │ Compliance cần  │
                    │ on-prem build?  │
                    └────────┬────────┘
                             │
              Có ────────────┼──────────── Không
              ▼                             ▼
        Jenkins /                    VCS là GitHub?
        Self-hosted                         │
        CircleCI runner              Có ────┴──── Không
              │                      ▼              ▼
              │               GHA đủ đơn?    CircleCI
              │                      │         (mạnh CI)
              │                 Có → GHA
              │                 Không → CircleCI
              └──────────────────────────────────┘
```

---

## 💬 Ví Dụ Trả Lời Ngắn (Phỏng Vấn)

**"CircleCI vs GHA cho monorepo 20 service?"**

> "Tôi ưu tiên CircleCI vì path-filtering + dynamic config + test split timings đã chứng minh giảm 60% thời gian PR ở dự án trước. GHA matrix vẫn làm được nhưng logic conditional và cache phân tán hơn; với team không chuyên viết workflow phức tạp, CircleCI ít boilerplate hơn. Nếu công ty chuẩn hóa GitHub Enterprise và budget GHA larger runners đủ, có thể cân nhắc GHA để giảm vendor."

**"Khi nào vẫn giữ Jenkins?"**

> "Khi regulatory yêu cầu build artifact không rời khỏi mạng nội bộ và đã có Jenkins HA với plugin SonarQube, Artifactory tích hợp sâu — migration cost cao hơn lợi ích ngắn hạn."

---

## 📎 Liên Kết Trong Repo

| Chủ đề | File |
| ------ | ---- |
| So sánh chi tiết + ví dụ Jenkinsfile | [5-circleci-vs-alternatives.md](../01-fundamentals/5-circleci-vs-alternatives.md) |
| Top 20 câu hỏi | [1-INTERVIEW_GUIDE.md](./1-INTERVIEW_GUIDE.md) |
| Thiết kế hệ thống | [3-system-design-scenarios.md](./3-system-design-scenarios.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-20
