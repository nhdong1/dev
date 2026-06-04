# Jenkins Interview Prep — Hướng Dẫn Chuẩn Bị Phỏng Vấn

> Bộ tài liệu chuẩn bị phỏng vấn Jenkins dành cho kỹ sư DevOps và Backend: từ câu hỏi lý thuyết, câu chuyện STAR thực tế, đến bài toán System Design CI/CD.

## Mục Lục

1. [Tổng Quan Phỏng Vấn Jenkins](#tổng-quan-phỏng-vấn-jenkins)
2. [Cấu Trúc Thư Mục](#cấu-trúc-thư-mục)
3. [Kế Hoạch Ôn Tập](#kế-hoạch-ôn-tập)
4. [Chủ Đề Hay Bị Hỏi](#chủ-đề-hay-bị-hỏi)
5. [Chiến Lược Trả Lời](#chiến-lược-trả-lời)
6. [Lộ Trình Ôn Thi 2 Tuần](#lộ-trình-ôn-thi-2-tuần)

---

## Tổng Quan Phỏng Vấn Jenkins

Phỏng vấn Jenkins thường xảy ra trong các vị trí:

| Vị Trí                        | Trọng Tâm Jenkins                                         | Cấp Độ  |
| ----------------------------- | --------------------------------------------------------- | ------- |
| Junior DevOps Engineer        | Pipeline cơ bản, Jenkinsfile, plugin thiết yếu            | Beginner |
| DevOps Engineer               | Distributed Builds, Security, Troubleshooting             | Intermediate |
| Senior DevOps / Platform Eng  | Shared Libraries, System Design, HA, JCasC                | Advanced |
| SRE (Site Reliability Engineer) | Monitoring, Incident Response, Performance Tuning       | Advanced |

### Tỷ Lệ Xuất Hiện Theo Chủ Đề

```
Pipeline (Declarative/Scripted)      ████████████████████ 95%
Distributed Builds / Agent           ████████████████░░░░ 80%
Security & Credentials               ███████████████░░░░░ 75%
Troubleshooting                      ██████████████░░░░░░ 70%
Shared Libraries                     ████████████░░░░░░░░ 60%
Integration (Git, Docker, K8s)       ████████████░░░░░░░░ 60%
System Design                        ██████████░░░░░░░░░░ 50%
Monitoring & Maintenance             █████████░░░░░░░░░░░ 45%
```

---

## Cấu Trúc Thư Mục

```
11-interview-prep/
├── README.md                   Hướng dẫn này — đọc trước tiên
├── INTERVIEW_GUIDE.md          Top 20 câu hỏi phỏng vấn + gợi ý trả lời chi tiết
├── 1-star-stories.md           Câu chuyện STAR về CI/CD incidents thực tế
└── 2-system-design-scenarios.md  Bài toán thiết kế hệ thống CI/CD pipeline
```

### Đọc Theo Thứ Tự

```
1. README.md (file này)              ← Nắm bức tranh tổng thể
2. INTERVIEW_GUIDE.md                ← Ôn lý thuyết + câu hỏi thường gặp
3. 1-star-stories.md                 ← Chuẩn bị câu chuyện thực tế
4. 2-system-design-scenarios.md      ← Luyện thiết kế hệ thống
```

---

## Kế Hoạch Ôn Tập

### Nếu Có 1 Tuần

```
Ngày 1-2: INTERVIEW_GUIDE.md — đọc toàn bộ, ghi chú key points
Ngày 3:   Ôn lại 02-pipeline/ (Declarative, Scripted, Jenkinsfile)
Ngày 4:   Ôn 05-distributed-builds/ và 06-security/
Ngày 5:   1-star-stories.md — chọn 2-3 câu phù hợp kinh nghiệm
Ngày 6:   2-system-design-scenarios.md — luyện thiết kế pipeline
Ngày 7:   Mock interview — tự trả lời to không nhìn tài liệu
```

### Nếu Có 3 Ngày

```
Ngày 1:   INTERVIEW_GUIDE.md — phần lý thuyết (Q1–Q10)
Ngày 2:   INTERVIEW_GUIDE.md — phần thực tế (Q11–Q20) + STAR stories
Ngày 3:   System Design + mock interview toàn bộ
```

### Nếu Chỉ Có 1 Ngày

```
Sáng:   INTERVIEW_GUIDE.md — đọc nhanh 20 câu, tập trung TOP 10
Chiều:  1-star-stories.md — ghi nhớ 2 câu chuyện STAR
Tối:    2-system-design-scenarios.md — 1 bài toán end-to-end
```

---

## Chủ Đề Hay Bị Hỏi

### Nhóm 1: Pipeline (Hay Bị Hỏi Nhất)

- Sự khác biệt giữa Declarative Pipeline (pipeline khai báo) và Scripted Pipeline (pipeline kịch bản)
- Cấu trúc Jenkinsfile — `agent`, `stages`, `steps`, `post`
- Parallel Stages (giai đoạn song song) — khi nào dùng, cách viết
- Shared Libraries (thư viện dùng chung) — mục đích, cách tổ chức
- Multibranch Pipeline (pipeline đa nhánh) — cách hoạt động với PR

### Nhóm 2: Distributed Builds

- Master/Agent model (mô hình chủ-tác nhân) — phân biệt vai trò
- JNLP Agent vs SSH Agent — khác nhau thế nào
- Docker Agent — lợi ích, giới hạn
- Kubernetes Agent — Pod Template, Dynamic Provisioning (cấp phát động)

### Nhóm 3: Security (Bảo Mật)

- Credentials (thông tin xác thực) — các loại, cách dùng trong pipeline
- Matrix-based Security vs Role Strategy Plugin
- Script Security Plugin — Groovy sandbox, ScriptApproval
- Secrets management — không để secret trong log, environment variable leak

### Nhóm 4: Troubleshooting (Xử Lý Sự Cố)

- Cách debug build failure có hệ thống
- Agent mất kết nối giữa chừng build — nguyên nhân, xử lý
- Jenkins chạy chậm — JVM heap, GC, executor saturation

### Nhóm 5: System Design

- Thiết kế CI/CD pipeline end-to-end (từ đầu đến cuối)
- Jenkins High Availability (tính sẵn sàng cao)
- Migration từ Jenkins sang Kubernetes-native setup

---

## Chiến Lược Trả Lời

### Câu Hỏi Lý Thuyết (Conceptual)

**Cấu trúc trả lời DEFINE → WHY → HOW → EXAMPLE:**

```
1. DEFINE:   Định nghĩa khái niệm ngắn gọn (1-2 câu)
2. WHY:      Tại sao cần nó? Vấn đề gì nó giải quyết?
3. HOW:      Cách hoạt động / cách cấu hình
4. EXAMPLE:  Ví dụ thực tế hoặc code snippet ngắn
```

**Ví dụ — "Shared Libraries là gì?"**
```
DEFINE:  Shared Library là Groovy code tái sử dụng, lưu trong Git repo riêng,
         dùng chung cho nhiều Jenkins Pipeline.
WHY:     Tránh copy-paste logic CI/CD giữa các team; chuẩn hóa quy trình.
HOW:     Cấu hình trong Jenkins > Manage Jenkins > System > Global Pipeline Libraries.
         Pipeline dùng @Library('my-lib') annotation để import.
EXAMPLE: vars/dockerBuild.groovy định nghĩa step buildAndPush(),
         Jenkinsfile gọi: dockerBuild.buildAndPush('myapp', '1.0.0')
```

### Câu Hỏi Thực Tế (Behavioral / STAR)

**Cấu trúc STAR:**

```
S — Situation:  Bối cảnh, môi trường, quy mô hệ thống
T — Task:       Nhiệm vụ, trách nhiệm của bạn
A — Action:     Hành động cụ thể bạn đã làm (dùng công nghệ gì, ra quyết định gì)
R — Result:     Kết quả đo được (thời gian, %, tỷ lệ lỗi)
```

**Lưu ý khi kể STAR:**
- Luôn dùng "Tôi" không phải "Chúng tôi" (interviewer muốn biết bạn làm gì)
- Kết quả phải có số liệu cụ thể: "giảm 40% thời gian build", "zero downtime 6 tháng"
- Độ dài lý tưởng: 2-3 phút nói chuyện

### Câu Hỏi System Design

**Cấu trúc trả lời Design:**

```
1. Clarify requirements:  Hỏi lại để hiểu rõ ràng scale, constraints
2. High-level design:     Vẽ/mô tả architecture tổng thể
3. Component deep-dive:   Đi sâu vào từng phần quan trọng
4. Trade-offs:            Nêu ra ưu/nhược điểm của lựa chọn
5. Production concerns:   Security, monitoring, scalability, failure scenarios
```

---

## Lộ Trình Ôn Thi 2 Tuần

### Tuần 1: Nền Tảng và Lý Thuyết

| Ngày | Chủ Đề                              | Tài Liệu                           |
| ---- | ----------------------------------- | ---------------------------------- |
| 1    | Kiến trúc Master/Agent, Job, Build  | `01-fundamentals/`                 |
| 2    | Declarative Pipeline cơ bản         | `02-pipeline/1-declarative-pipeline.md` |
| 3    | Scripted Pipeline và Jenkinsfile    | `02-pipeline/2-scripted-pipeline.md` |
| 4    | Multibranch Pipeline và PR builds   | `02-pipeline/5-multibranch-pipeline.md` |
| 5    | Distributed Builds: Docker Agent    | `05-distributed-builds/2-docker-agents.md` |
| 6    | Distributed Builds: K8s Agent       | `05-distributed-builds/3-kubernetes-agents.md` |
| 7    | Security: Credentials và Auth       | `06-security/`                     |

### Tuần 2: Thực Hành và Phỏng Vấn

| Ngày | Chủ Đề                              | Tài Liệu                           |
| ---- | ----------------------------------- | ---------------------------------- |
| 8    | Shared Libraries                    | `08-shared-libraries/`             |
| 9    | Integration: Git, Docker, K8s       | `07-integration/`                  |
| 10   | Troubleshooting thực tế             | `10-troubleshooting/`              |
| 11   | INTERVIEW_GUIDE — Q1-Q10            | `INTERVIEW_GUIDE.md`               |
| 12   | INTERVIEW_GUIDE — Q11-Q20           | `INTERVIEW_GUIDE.md`               |
| 13   | STAR Stories + System Design        | `1-star-stories.md`, `2-system-design-scenarios.md` |
| 14   | Mock Interview — không nhìn tài liệu | Tự luyện / nhờ người hỏi         |

---

## Dấu Hiệu Sẵn Sàng Phỏng Vấn

Bạn sẵn sàng khi:

- [ ] Giải thích được Master/Agent model mà không cần tài liệu (< 2 phút)
- [ ] Viết được Declarative Pipeline hoàn chỉnh từ đầu, không copy-paste
- [ ] Kể được ít nhất 2 câu chuyện STAR về CI/CD, mỗi câu có số liệu kết quả
- [ ] Thiết kế được pipeline end-to-end cho một ứng dụng microservice trên bảng trắng
- [ ] Trả lời được "Jenkins vs GitHub Actions — khi nào dùng cái nào?"
- [ ] Debug được scenario: build fail vì agent mất kết nối — nêu 3 nguyên nhân và cách kiểm tra

---

**Xem tiếp:** [INTERVIEW_GUIDE.md](INTERVIEW_GUIDE.md) — Top 20 câu hỏi phỏng vấn với gợi ý trả lời đầy đủ.
