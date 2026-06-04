# 01 — Fundamentals: Nền Tảng AI/ML

> Module nền tảng: hiểu đúng AI, Machine Learning (Học Máy), Deep Learning (Học Sâu) và Generative AI (Trí Tuệ Nhân Tạo Tạo Sinh) trước khi đi vào các dịch vụ AWS cụ thể.

---

## 🎯 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- [ ] Phân biệt rõ AI, ML, Deep Learning và Generative AI — giải thích được mối quan hệ phân cấp
- [ ] Mô tả 3 loại Học Máy: Supervised (Có Giám Sát), Unsupervised (Không Giám Sát), Reinforcement (Học Tăng Cường)
- [ ] Vẽ được ML Workflow (Quy Trình Học Máy) 6 bước từ dữ liệu đến production
- [ ] Giải thích 3 tầng dịch vụ AI của AWS và chọn đúng tầng cho từng bài toán
- [ ] Định nghĩa LLM (Large Language Model — Mô Hình Ngôn Ngữ Lớn), Foundation Model (Mô Hình Nền Tảng), Prompt, Token và RAG (Retrieval-Augmented Generation — Tạo Sinh Tăng Cường Truy Xuất)

**Thời gian hoàn thành:** 4-6 giờ

---

## 📚 Các File Trong Module

| File | Nội Dung | Thời Gian |
| ---- | -------- | --------- |
| [1-ai-ml-overview.md](./1-ai-ml-overview.md) | AI vs ML vs DL vs Generative AI | 1-1.5 giờ |
| [2-ml-types.md](./2-ml-types.md) | Supervised, Unsupervised, Reinforcement Learning | 1-1.5 giờ |
| [3-ml-workflow.md](./3-ml-workflow.md) | Data prep → Training → Evaluation → Deployment | 1 giờ |
| [4-aws-ai-layers.md](./4-aws-ai-layers.md) | Tầng AI Services, ML Services, Framework & Infra | 1 giờ |
| [5-generative-ai-concepts.md](./5-generative-ai-concepts.md) | LLM, Foundation Model, Prompt, Token, RAG | 1 giờ |

---

## 🗺️ Lộ Trình Đọc

```
1-ai-ml-overview.md
        ↓
2-ml-types.md
        ↓
3-ml-workflow.md
        ↓
4-aws-ai-layers.md        ← kết nối lý thuyết với thực tế AWS
        ↓
5-generative-ai-concepts.md  ← chuẩn bị cho 03-bedrock/
```

---

## 🔑 Khái Niệm Cốt Lõi Cần Nhớ

### Phân Cấp AI

```
Artificial Intelligence (Trí Tuệ Nhân Tạo) — rộng nhất
  └─ Machine Learning (Học Máy) — dùng dữ liệu để học
       └─ Deep Learning (Học Sâu) — mạng thần kinh nhiều lớp
            └─ Generative AI (AI Tạo Sinh) — tạo nội dung mới
```

### Ba Tầng Dịch Vụ AI Của AWS

| Tầng | Dịch Vụ Tiêu Biểu | Dành Cho |
| ---- | ----------------- | -------- |
| **AI Services** (Dịch Vụ AI Sẵn Có) | Rekognition, Comprehend, Polly, Transcribe | Developer không cần ML expertise |
| **ML Services** (Dịch Vụ ML) | SageMaker | Data Scientist, ML Engineer |
| **Framework & Infrastructure** (Khung & Hạ Tầng) | EC2 GPU, TensorFlow, PyTorch trên AWS | Nhà nghiên cứu AI |

---

## ❓ Câu Hỏi Phỏng Vấn Liên Quan

- Giải thích sự khác biệt giữa AI, Machine Learning và Deep Learning?
- Supervised Learning (Học Có Giám Sát) khác Unsupervised Learning (Học Không Giám Sát) như thế nào? Cho ví dụ từng loại.
- Khi nào bạn dùng AI Service có sẵn của AWS thay vì tự train model trên SageMaker?
- Foundation Model là gì? Tại sao nó là bước ngoặt trong AI?
- RAG (Retrieval-Augmented Generation) giải quyết vấn đề gì của LLM?

---

## ➡️ Bước Tiếp Theo

Sau module này → học [02-sagemaker/](../02-sagemaker/README.md) để biết cách train và deploy custom ML model trên AWS.
