# Các Loại Học Máy — Supervised, Unsupervised, Reinforcement Learning

> Ba paradigm (mô hình tư duy) cốt lõi của Machine Learning (Học Máy) — nắm vững để chọn đúng thuật toán và dịch vụ AWS cho từng bài toán.

---

## 1. Tổng Quan Ba Loại

```
Machine Learning (Học Máy)
├── Supervised Learning (Học Có Giám Sát)
│   Dữ liệu: có nhãn (labeled)
│   Học từ: cặp (Input → Output) đã biết
│   Mục tiêu: dự đoán Output cho Input mới
│
├── Unsupervised Learning (Học Không Giám Sát)
│   Dữ liệu: không có nhãn (unlabeled)
│   Học từ: cấu trúc ẩn trong dữ liệu
│   Mục tiêu: khám phá pattern, nhóm, quan hệ
│
└── Reinforcement Learning (Học Tăng Cường)
    Dữ liệu: không có sẵn — agent tự sinh ra qua tương tác
    Học từ: phần thưởng (reward) và hình phạt (penalty)
    Mục tiêu: tìm chính sách (policy) tối ưu hóa reward lâu dài
```

---

## 2. Supervised Learning — Học Có Giám Sát

### Nguyên Lý

Model học từ **training data (dữ liệu huấn luyện) có nhãn** — tức là mỗi mẫu đầu vào (input) đã biết trước đầu ra (label/output) đúng là gì.

```
Training:
  [Email text] → [spam]     ← labeled pair
  [Email text] → [not spam] ← labeled pair
  [Email text] → [spam]     ← labeled pair
        ↓ (train)
     Model (học pattern)

Inference (Suy Luận):
  [Email text mới] → Model → [spam / not spam]
```

### Hai Nhóm Bài Toán Chính

#### Classification — Phân Loại

Dự đoán **nhãn rời rạc** (discrete label) cho một mẫu.

| Loại | Mô Tả | Ví Dụ |
| ---- | ------ | ------ |
| **Binary Classification** (Phân Loại Nhị Phân) | 2 lớp | spam/không spam, bệnh/không bệnh |
| **Multi-class Classification** (Phân Loại Đa Lớp) | N lớp, chọn 1 | loại cây, giống mèo, ngôn ngữ |
| **Multi-label Classification** (Phân Loại Đa Nhãn) | N lớp, chọn nhiều | tag ảnh: {mèo, ngoài trời, ban ngày} |

**Thuật toán phổ biến:** Logistic Regression (Hồi Quy Logistic), Decision Tree (Cây Quyết Định), Random Forest (Rừng Ngẫu Nhiên), SVM (Support Vector Machine — Máy Vectơ Hỗ Trợ), XGBoost, Neural Network.

#### Regression — Hồi Quy

Dự đoán **giá trị liên tục** (continuous value).

| Ví Dụ | Input | Output |
| ------ | ----- | ------ |
| Dự đoán giá nhà | Diện tích, vị trí, số phòng | Giá (VNĐ) |
| Dự đoán doanh thu | Lịch sử bán hàng, mùa vụ | Doanh thu tuần tới |
| Dự đoán nhiệt độ | Dữ liệu khí tượng | Nhiệt độ ngày mai |

**Thuật toán phổ biến:** Linear Regression (Hồi Quy Tuyến Tính), Ridge, Lasso, XGBoost, Neural Network.

### Supervised Learning Trên AWS

| Dịch Vụ AWS | Supervised Learning Ứng Dụng |
| ----------- | ---------------------------- |
| **SageMaker XGBoost** | Tabular data classification & regression |
| **SageMaker Linear Learner** | Binary/multi-class classification, regression |
| **SageMaker Image Classification** | Multi-class image classification |
| **Amazon Rekognition Custom Labels** | Binary/multi-class image classification tùy chỉnh |
| **Amazon Comprehend Custom Classification** | Multi-class text classification |
| **Amazon Forecast** | Time series forecasting (regression theo thời gian) |

### Khi Nào Dùng Supervised Learning?

✅ Có dữ liệu đã được gán nhãn (labeled data)
✅ Biết rõ bài toán là phân loại hay dự đoán giá trị
✅ Cần độ chính xác cao và có thể đo lường được

❌ Gán nhãn dữ liệu tốn kém hoặc không khả thi
❌ Chưa biết "đúng" là gì (exploratory analysis)

---

## 3. Unsupervised Learning — Học Không Giám Sát

### Nguyên Lý

Model học từ **dữ liệu không có nhãn** — tự tìm cấu trúc, pattern và mối quan hệ ẩn bên trong dữ liệu.

```
Dữ liệu không nhãn:
  [Customer A: tuổi=25, mua=thường xuyên, giá trị=cao]
  [Customer B: tuổi=55, mua=hiếm, giá trị=thấp]
  [Customer C: tuổi=30, mua=thường xuyên, giá trị=cao]
  ...
        ↓ (unsupervised learning)
  Cluster 1: khách hàng trẻ, trung thành, chi tiêu cao ← tự phát hiện
  Cluster 2: khách hàng lớn tuổi, ít mua
```

### Các Nhóm Bài Toán

#### Clustering — Phân Cụm

Nhóm các điểm dữ liệu tương đồng lại với nhau mà **không biết trước** có bao nhiêu nhóm hay nhóm đó là gì.

| Thuật Toán | Mô Tả | Dùng Khi |
| ---------- | ------ | --------- |
| **K-Means** | Chia thành K cụm bằng centroid | K biết trước, cluster hình cầu |
| **DBSCAN** | Cluster dựa trên mật độ điểm | Cluster hình dạng bất kỳ, có noise |
| **Hierarchical Clustering** (Phân Cụm Phân Cấp) | Cây phân cấp các cluster | Muốn xem cấu trúc phân cấp |

**Ứng dụng thực tế:** Phân khúc khách hàng (Customer Segmentation), nhóm tài liệu, phát hiện topic ẩn trong corpus văn bản.

#### Dimensionality Reduction — Giảm Chiều Dữ Liệu

Nén dữ liệu nhiều chiều xuống ít chiều hơn mà vẫn giữ được thông tin quan trọng.

| Thuật Toán | Tên Đầy Đủ | Dùng Khi |
| ---------- | ---------- | --------- |
| **PCA** | Principal Component Analysis — Phân Tích Thành Phần Chính | Dữ liệu tuyến tính |
| **t-SNE** | t-Distributed Stochastic Neighbor Embedding | Visualization 2D/3D |
| **UMAP** | Uniform Manifold Approximation and Projection | Visualization + downstream tasks |
| **Autoencoder** | (giữ nguyên) | Compression, denoising |

**Ứng dụng thực tế:** Tiền xử lý (preprocessing) trước khi train, visualization, xử lý curse of dimensionality (Lời Nguyền Chiều Cao).

#### Anomaly Detection — Phát Hiện Bất Thường

Tìm các điểm dữ liệu khác biệt đáng kể so với phần còn lại.

**Ứng dụng thực tế:** Phát hiện gian lận giao dịch (fraud detection), lỗi máy móc (predictive maintenance — bảo trì dự đoán), bất thường mạng.

**Dịch vụ AWS:** Amazon Lookout for Metrics, Amazon Lookout for Equipment, SageMaker Random Cut Forest.

#### Association Learning — Học Liên Kết

Tìm các quy tắc "thường đi cùng nhau" trong tập dữ liệu.

**Ví dụ kinh điển (Market Basket Analysis — Phân Tích Giỏ Hàng):** "Khách mua tã lót thường cũng mua bia vào cuối tuần."

**Thuật toán:** Apriori, FP-Growth.

### Unsupervised Learning Trên AWS

| Dịch Vụ AWS | Unsupervised Learning Ứng Dụng |
| ----------- | ------------------------------ |
| **SageMaker K-Means** | Customer segmentation (phân khúc khách hàng) |
| **SageMaker PCA** | Feature reduction trước training |
| **SageMaker Random Cut Forest** | Anomaly detection (phát hiện bất thường) |
| **SageMaker IP Insights** | Phát hiện hành vi IP bất thường |
| **SageMaker LDA** | Latent Dirichlet Allocation — topic modeling văn bản |
| **Amazon Lookout for Metrics** | Anomaly detection không cần ML expertise |
| **Amazon Personalize** | Collaborative filtering (Lọc Cộng Tác) — học hành vi user |

---

## 4. Reinforcement Learning — Học Tăng Cường

### Nguyên Lý

**Agent (Tác Nhân)** học cách hành động trong **Environment (Môi Trường)** để tối đa hóa tổng **Reward (Phần Thưởng)** tích lũy theo thời gian.

```
                    Action (Hành Động)
Agent ─────────────────────────────────→ Environment
  ↑                                           │
  │    State (Trạng Thái)                     │
  │    Reward (Phần Thưởng)                   │
  └───────────────────────────────────────────┘

Vòng lặp:
  1. Agent quan sát State hiện tại
  2. Agent thực hiện Action
  3. Environment trả về State mới + Reward
  4. Agent cập nhật Policy (Chính Sách) để maximize tổng Reward
```

### Các Khái Niệm Cốt Lõi

| Khái Niệm | Giải Thích | Ví Dụ (Game Cờ Vua) |
| ---------- | ---------- | -------------------- |
| **Agent** | Thực thể học và ra quyết định | AI cờ vua |
| **Environment** | Thế giới mà agent tương tác | Bàn cờ |
| **State** (Trạng Thái) | Snapshot hiện tại của environment | Vị trí các quân cờ |
| **Action** (Hành Động) | Hành động agent thực hiện | Đi quân |
| **Reward** (Phần Thưởng) | Tín hiệu phản hồi từ environment | +1 thắng, -1 thua, 0 tiếp tục |
| **Policy** (Chính Sách) | Chiến lược: State → Action | Hàm quyết định đi nước nào |
| **Value Function** (Hàm Giá Trị) | Ước tính tổng reward từ state | Đánh giá thế cờ |

### Exploration vs Exploitation — Khám Phá vs Khai Thác

Thách thức cốt lõi trong RL:
- **Exploration (Khám Phá):** Thử hành động mới chưa biết để có thể tìm ra điều tốt hơn
- **Exploitation (Khai Thác):** Dùng hành động đã biết là tốt để maximize reward ngay lập tức

**Ví dụ:** Robot học đi — cần thử nhiều dáng đi (exploration) trước khi tìm ra dáng đi hiệu quả nhất (exploitation).

### Ứng Dụng Thực Tế

| Lĩnh Vực | Ứng Dụng | Ví Dụ |
| --------- | --------- | ------ |
| **Game Playing** | Chơi game | AlphaGo, Dota 2 OpenAI Five |
| **Robotics** (Robot) | Điều khiển robot | Robot cánh tay công nghiệp |
| **Recommendation** (Gợi Ý) | Tối ưu gợi ý | YouTube, TikTok feed optimization |
| **Finance** (Tài Chính) | Trading tự động | Algorithmic trading |
| **Resource Management** | Quản lý tài nguyên | Tối ưu data center cooling (Google) |
| **Healthcare** (Y Tế) | Cá nhân hóa điều trị | Điều chỉnh liều thuốc |

### Reinforcement Learning Trên AWS

| Dịch Vụ AWS | RL Ứng Dụng |
| ----------- | ------------ |
| **Amazon SageMaker RL** | Train RL model với Ray RLlib, Stable Baselines |
| **AWS DeepRacer** | Xe tự lái miniature — học RL qua cuộc thi |
| **AWS DeepComposer** | Tạo nhạc với Generative AI + RL |

> **Lưu ý thực tế:** RL ít phổ biến hơn Supervised và Unsupervised Learning trong ứng dụng doanh nghiệp thông thường. Hầu hết bài toán business dùng Supervised Learning.

---

## 5. Bảng So Sánh Tổng Hợp

| | Supervised (Có Giám Sát) | Unsupervised (Không Giám Sát) | Reinforcement (Tăng Cường) |
| -- | ------------------------ | ----------------------------- | -------------------------- |
| **Dữ liệu** | Có nhãn | Không nhãn | Tự sinh qua tương tác |
| **Mục tiêu** | Dự đoán/phân loại | Khám phá cấu trúc | Tối đa hóa reward |
| **Phản hồi** | Ground truth labels | Không có | Reward/penalty |
| **Ứng dụng phổ biến** | Nhận diện ảnh, spam, dự báo | Clustering, anomaly detection | Game, robotics, tối ưu |
| **Độ phổ biến trong business** | ⭐⭐⭐ Rất cao | ⭐⭐ Khá cao | ⭐ Chuyên biệt |
| **AWS SageMaker built-in** | XGBoost, Linear Learner, Image Classification | K-Means, PCA, RCF, LDA | SageMaker RL (Ray) |

---

## 6. Ví Dụ Thực Tế: E-commerce (Thương Mại Điện Tử)

```
Bài Toán E-commerce — Áp Dụng Cả Ba Loại ML:

SUPERVISED LEARNING:
  → Dự đoán khách hàng có mua hay không (binary classification)
  → Dự đoán giá trị đơn hàng tiếp theo (regression)
  → Phân loại review: tích cực/tiêu cực (sentiment classification)

UNSUPERVISED LEARNING:
  → Phân khúc khách hàng thành nhóm tương đồng (clustering)
  → Phát hiện giao dịch gian lận (anomaly detection)
  → Tìm sản phẩm thường mua cùng nhau (association learning)

REINFORCEMENT LEARNING:
  → Tối ưu thứ tự hiển thị sản phẩm gợi ý để maximize click-through
  → Tối ưu chiến lược giảm giá động (dynamic pricing)
```

---

## 7. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Supervised Learning khác Unsupervised Learning thế nào? Cho ví dụ mỗi loại.**

> Supervised Learning cần dữ liệu có nhãn — học từ các cặp (input, output) đã biết. Ví dụ: phân loại email spam (label: spam/không spam), dự đoán giá nhà (label: giá thực tế). Unsupervised Learning không cần nhãn — tự tìm cấu trúc ẩn. Ví dụ: phân khúc khách hàng theo hành vi mua hàng mà không biết trước có bao nhiêu nhóm.

**Q: Reinforcement Learning phù hợp cho bài toán nào? Khi nào KHÔNG nên dùng?**

> RL phù hợp khi: có thể định nghĩa reward rõ ràng, agent có thể tương tác lặp đi lặp lại với environment, quyết định có tính tuần tự. Không nên dùng khi: có dữ liệu labeled sẵn (supervised học tốt hơn), cost per interaction cao, reward khó định nghĩa.

**Q: Nếu có dữ liệu không nhãn, bạn có những lựa chọn ML nào?**

> (1) Unsupervised Learning trực tiếp: clustering để khám phá cấu trúc. (2) Semi-supervised Learning (Học Bán Giám Sát): gán nhãn một phần nhỏ rồi dùng phần không nhãn để cải thiện. (3) Self-supervised Learning (Học Tự Giám Sát): tạo label tự động từ dữ liệu (ví dụ: BERT mask token). (4) Active Learning (Học Chủ Động): model tự chọn những sample cần gán nhãn nhất.

---

## ➡️ Tiếp Theo

[3-ml-workflow.md](./3-ml-workflow.md) — Quy trình hoàn chỉnh từ dữ liệu thô đến mô hình production: Data Collection → Feature Engineering → Training → Evaluation → Deployment → Monitoring.
