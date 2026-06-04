# AI vs ML vs Deep Learning vs Generative AI — Tổng Quan Và Phân Biệt

> Hiểu đúng mối quan hệ phân cấp giữa các khái niệm là nền tảng để học mọi thứ tiếp theo một cách có hệ thống.

---

## 1. Bức Tranh Tổng Quát

```
┌─────────────────────────────────────────────────────────────────┐
│   Artificial Intelligence — AI (Trí Tuệ Nhân Tạo)              │
│   Mục tiêu: làm cho máy tính thực hiện được các nhiệm vụ       │
│   đòi hỏi trí tuệ của con người                                 │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  Machine Learning — ML (Học Máy)                         │  │
│   │  Phương pháp: học từ dữ liệu, không lập trình cứng       │  │
│   │                                                           │  │
│   │  ┌────────────────────────────────────────────────────┐  │  │
│   │  │  Deep Learning — DL (Học Sâu)                      │  │  │
│   │  │  Kỹ thuật: mạng thần kinh nhiều lớp (layers)       │  │  │
│   │  │                                                     │  │  │
│   │  │  ┌──────────────────────────────────────────────┐  │  │  │
│   │  │  │  Generative AI — AI Tạo Sinh                 │  │  │  │
│   │  │  │  Ứng dụng: tạo nội dung mới (văn bản,        │  │  │  │
│   │  │  │  hình ảnh, âm thanh, code)                    │  │  │  │
│   │  │  └──────────────────────────────────────────────┘  │  │  │
│   │  └────────────────────────────────────────────────────┘  │  │
│   └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Mối quan hệ:** Generative AI ⊂ Deep Learning ⊂ Machine Learning ⊂ Artificial Intelligence

---

## 2. Artificial Intelligence — AI (Trí Tuệ Nhân Tạo)

### Định Nghĩa

AI là lĩnh vực khoa học máy tính nghiên cứu cách tạo ra hệ thống có khả năng thực hiện các nhiệm vụ thường đòi hỏi trí tuệ của con người: lý luận, học hỏi, lập kế hoạch, nhận diện ngôn ngữ, nhận diện hình ảnh.

### Phân Loại AI Theo Phạm Vi

| Loại | Tên Tiếng Anh | Mô Tả | Ví Dụ |
| ---- | ------------- | ------ | ------ |
| **AI Hẹp** | Narrow AI / Weak AI | Chỉ giỏi một nhiệm vụ cụ thể | Google Search, spam filter, Rekognition |
| **AI Tổng Quát** | Artificial General Intelligence (AGI) | Thông minh ngang con người ở mọi lĩnh vực | Chưa tồn tại |
| **Siêu AI** | Artificial Superintelligence (ASI) | Vượt xa trí tuệ con người | Lý thuyết |

> **Thực tế hiện nay:** 100% AI sản phẩm hàng ngày đều là Narrow AI — kể cả GPT-4 hay Claude cũng là Narrow AI (rất giỏi ngôn ngữ, nhưng không thể tự lái xe hay chơi cờ vua như chuyên gia).

### Các Phương Pháp AI (Không Phải Tất Cả Đều Là ML)

```
AI Approaches (Phương Pháp AI)
├── Rule-based Systems (Hệ Thống Dựa Luật)     ← KHÔNG phải ML
│   Ví dụ: expert system, if-else logic
├── Search & Optimization (Tìm Kiếm & Tối Ưu)  ← KHÔNG phải ML
│   Ví dụ: A* pathfinding, genetic algorithm
├── Machine Learning (Học Máy)                  ← PHẢI HỌC KỸ
│   Học từ dữ liệu, không cần lập trình cứng
└── Knowledge Representation (Biểu Diễn Tri Thức) ← KHÔNG phải ML
    Ví dụ: ontology, knowledge graph
```

---

## 3. Machine Learning — ML (Học Máy)

### Định Nghĩa

ML là tập con của AI, cho phép máy tính **học từ dữ liệu** và **cải thiện hiệu suất theo thời gian** mà không cần lập trình cứng từng quy tắc cụ thể.

**Câu nói nổi tiếng của Arthur Samuel (1959):** *"Machine Learning là lĩnh vực nghiên cứu cho phép máy tính tự học mà không cần lập trình tường minh."*

### So Sánh: Lập Trình Truyền Thống vs Machine Learning

```
Lập Trình Truyền Thống (Traditional Programming):
  Input (Dữ Liệu Đầu Vào) + Rules (Luật) → Output (Kết Quả)

Machine Learning:
  Input (Dữ Liệu Đầu Vào) + Output (Kết Quả) → Rules (Luật — gọi là Model)
```

**Ví dụ thực tế:** Nhận diện spam email
- **Truyền thống:** Dev viết if-else: nếu email chứa "bạn đã trúng thưởng" → spam
- **ML:** Cho model xem 10.000 email spam + 10.000 email bình thường → model tự học pattern

### Khi Nào Dùng ML Thay Vì Lập Trình Truyền Thống?

| Tình Huống | Dùng ML? |
| ---------- | -------- |
| Quy tắc rõ ràng, ít thay đổi | ❌ Không cần |
| Quá nhiều quy tắc, khó viết tay | ✅ Nên dùng |
| Dữ liệu nhiều, pattern phức tạp | ✅ Nên dùng |
| Cần cá nhân hóa theo từng user | ✅ Nên dùng |
| Không có dữ liệu | ❌ Không thể dùng |

---

## 4. Deep Learning — DL (Học Sâu)

### Định Nghĩa

Deep Learning là tập con của ML, sử dụng **Neural Networks (Mạng Thần Kinh Nhân Tạo)** nhiều lớp (nhiều "deep" layers) để học các biểu diễn (representations) ngày càng trừu tượng từ dữ liệu thô.

### Neural Network — Mạng Thần Kinh Nhân Tạo

```
Input Layer       Hidden Layers         Output Layer
(Lớp Đầu Vào)   (Các Lớp Ẩn)         (Lớp Đầu Ra)

  [pixel 1]  →  [neuron] [neuron]  →
  [pixel 2]  →  [neuron] [neuron]  →  [cat/dog]
  [pixel 3]  →  [neuron] [neuron]  →
   ...            ...       ...
```

Mỗi layer học một đặc trưng (feature) khác nhau:
- **Layer 1:** Cạnh, đường thẳng (edges)
- **Layer 2:** Hình dạng cơ bản (shapes)
- **Layer 3:** Bộ phận (parts: tai, mắt)
- **Layer N:** Khái niệm trừu tượng (cat vs dog)

### Các Kiến Trúc Deep Learning Phổ Biến

| Kiến Trúc | Tên Đầy Đủ | Dùng Cho |
| --------- | ---------- | -------- |
| **CNN** | Convolutional Neural Network — Mạng Tích Chập | Hình ảnh, video |
| **RNN** | Recurrent Neural Network — Mạng Hồi Quy | Chuỗi thời gian, văn bản (cũ) |
| **LSTM** | Long Short-Term Memory — Bộ Nhớ Ngắn-Dài Hạn | Chuỗi dài, dịch máy (cũ) |
| **Transformer** | (giữ nguyên) | NLP hiện đại, Generative AI |
| **GAN** | Generative Adversarial Network — Mạng Đối Nghịch Tạo Sinh | Tạo ảnh, deepfake |
| **Diffusion Model** | (giữ nguyên) | Tạo ảnh chất lượng cao (Stable Diffusion) |

### ML vs Deep Learning — Khi Nào Dùng Gì?

| Tiêu Chí | Traditional ML | Deep Learning |
| --------- | -------------- | ------------- |
| **Dữ liệu cần** | Ít (hàng nghìn) | Nhiều (hàng triệu) |
| **Feature Engineering** | Thủ công | Tự động (end-to-end) |
| **Tài nguyên tính toán** | CPU đủ | GPU/TPU cần thiết |
| **Khả năng giải thích** | Dễ giải thích | Khó giải thích (black box) |
| **Tốt cho** | Tabular data, structured | Images, audio, text, video |
| **Ví dụ** | XGBoost, Random Forest | CNN, BERT, GPT |

---

## 5. Generative AI — AI Tạo Sinh

### Định Nghĩa

Generative AI là tập con của Deep Learning, chuyên **tạo ra nội dung mới** — văn bản, hình ảnh, âm thanh, code, video — thay vì chỉ phân loại hoặc dự đoán.

### Generative AI vs Discriminative AI (AI Phân Loại)

```
Discriminative AI (AI Phân Loại) — học ranh giới giữa các class:
  Input: ảnh con mèo → Output: "mèo" hoặc "chó"
  Hỏi: cái này là gì?

Generative AI (AI Tạo Sinh) — học phân phối dữ liệu để tạo mới:
  Input: prompt "một con mèo dễ thương" → Output: ảnh mèo mới
  Hỏi: tạo ra cái này cho tôi
```

### Các Loại Generative AI

| Loại | Đầu Ra | Mô Hình Tiêu Biểu | Dịch Vụ AWS |
| ---- | ------ | ------------------ | ----------- |
| **Text Generation** (Tạo Văn Bản) | Văn bản, code | GPT, Claude, Llama | Bedrock |
| **Image Generation** (Tạo Hình Ảnh) | Hình ảnh | DALL-E, Stable Diffusion | Bedrock (Stability AI) |
| **Audio Generation** (Tạo Âm Thanh) | Giọng nói, nhạc | TTS models | Amazon Polly |
| **Video Generation** (Tạo Video) | Video | Sora (OpenAI) | — |
| **Code Generation** (Tạo Code) | Code, tests | Codex, CodeWhisperer | Amazon Q Developer |

### Tại Sao Generative AI Là Bước Ngoặt?

**Trước 2017 (trước Transformer):** AI giỏi phân loại nhưng kém tạo nội dung mới.

**2017 — "Attention Is All You Need":** Kiến trúc Transformer ra đời — nền tảng của mọi LLM hiện đại.

**2020-nay — Foundation Models (Mô Hình Nền Tảng):** Một model khổng lồ được huấn luyện trước (pre-trained) trên dữ liệu khổng lồ, sau đó fine-tune (tinh chỉnh) cho nhiều tác vụ khác nhau.

---

## 6. So Sánh Tổng Hợp

| | AI (Trí Tuệ Nhân Tạo) | ML (Học Máy) | DL (Học Sâu) | Generative AI (AI Tạo Sinh) |
| -- | --------------------- | ------------ | ------------ | -------------------------- |
| **Phạm vi** | Rộng nhất | Tập con của AI | Tập con của ML | Tập con của DL |
| **Cách hoạt động** | Mọi phương pháp | Học từ dữ liệu | Mạng thần kinh nhiều lớp | Học phân phối dữ liệu |
| **Đầu ra** | Hành động/quyết định | Dự đoán/phân loại | Dự đoán/phân loại phức tạp | Nội dung mới |
| **Dữ liệu cần** | Có thể không cần | Ít đến vừa | Rất nhiều | Rất rất nhiều |
| **Ví dụ AWS** | Rule engine | XGBoost SageMaker | Rekognition | Bedrock (Claude, Llama) |
| **Năm nổi bật** | 1956 | 1980s-2010s | 2012+ (ImageNet) | 2022+ (ChatGPT) |

---

## 7. Ứng Dụng Thực Tế Trên AWS

```
Bài Toán Nhận Diện Khuôn Mặt
  → AI (rộng): có thể dùng rule-based (nhận dạng khuôn mặt đơn giản)
  → ML: train model binary classifier
  → Deep Learning (CNN): Rekognition dùng CNN bên trong
  → (không phải Generative AI — không tạo khuôn mặt mới)

Bài Toán Chatbot Hỗ Trợ Khách Hàng
  → AI (rộng): Amazon Lex (rule-based + ML)
  → ML: NLP model phân loại intent
  → Deep Learning: Transformer-based model
  → Generative AI: Bedrock Claude (tạo câu trả lời mới, không đơn giản chỉ mapping)
```

---

## 8. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: AI, ML, Deep Learning khác nhau thế nào?**

> AI là lĩnh vực rộng nhất, ML là phương pháp học từ dữ liệu bên trong AI, Deep Learning là kỹ thuật dùng mạng thần kinh nhiều lớp bên trong ML. Chúng có quan hệ phân cấp: DL ⊂ ML ⊂ AI.

**Q: Khi nào nên dùng ML thay vì lập trình rule-based?**

> Khi bài toán có quá nhiều quy tắc khó viết tay, có nhiều dữ liệu để học, hoặc cần cá nhân hóa. Ví dụ: nhận diện spam email, gợi ý sản phẩm, phát hiện gian lận.

**Q: Generative AI khác ML truyền thống như thế nào?**

> ML truyền thống học để phân loại hoặc dự đoán từ dữ liệu đã có. Generative AI học phân phối dữ liệu để tạo ra nội dung hoàn toàn mới: văn bản, hình ảnh, code. Generative AI thường cần dữ liệu training lớn hơn nhiều và dùng kiến trúc Transformer.

---

## ➡️ Tiếp Theo

[2-ml-types.md](./2-ml-types.md) — Tìm hiểu 3 loại học máy: Supervised, Unsupervised, Reinforcement Learning với ví dụ thực tế từ AWS.
