# Amazon Personalize — Hệ Thống Gợi Ý Cá Nhân Hóa

> **Amazon Personalize** — Hệ Thống Gợi Ý Cá Nhân Hóa là dịch vụ ML được quản lý hoàn toàn (fully-managed) của AWS, cho phép xây dựng **recommendation system** (hệ thống gợi ý) chất lượng cao mà không cần ML expertise. Sử dụng cùng công nghệ với hệ thống gợi ý của **Amazon.com** — deploy trong vài giờ thay vì vài tháng.

---

## 🎯 Amazon Personalize Giải Quyết Bài Toán Gì?

### Recommendation System (Hệ Thống Gợi Ý) Là Gì?

**Recommendation system** dự đoán items mà một user **có khả năng thích / tương tác** dựa trên:
- **Lịch sử hành vi** (behavioral history): những gì user đã xem, mua, click, đánh giá
- **Đặc điểm tương đồng** (collaborative filtering — lọc cộng tác): users tương tự nhau thích những gì
- **Thuộc tính item** (content-based filtering — lọc theo nội dung): items tương tự nhau

### Tại Sao Dùng Personalize Thay Vì Tự Build?

| Tự Build | Amazon Personalize |
|---|---|
| Cần data science team xây model | API-based, không cần ML expertise |
| 3-6 tháng để đến production | Deploy trong 1-3 ngày |
| Khó xử lý real-time event updates | `PutEvents` API cập nhật real-time |
| Phải tự quản lý infra scaling | Fully managed, auto-scaling |
| A/B testing phức tạp | Built-in với Amazon Personalize |

---

## 📐 Kiến Trúc Tổng Quan

```
┌────────────────────────────────────────────────────────────────────────┐
│                         Amazon Personalize                             │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                       Dataset Group                              │  │
│  │  ┌────────────────────┐  ┌──────────────┐  ┌────────────────┐  │  │
│  │  │ Interactions        │  │    Users     │  │     Items      │  │  │
│  │  │ Dataset (bắt buộc) │  │   Dataset    │  │    Dataset     │  │  │
│  │  │                    │  │  (tùy chọn)  │  │  (tùy chọn)   │  │  │
│  │  │ USER_ID            │  │  USER_ID     │  │  ITEM_ID       │  │  │
│  │  │ ITEM_ID            │  │  age         │  │  GENRE         │  │  │
│  │  │ TIMESTAMP          │  │  gender      │  │  PRICE         │  │  │
│  │  │ EVENT_TYPE         │  │  location    │  │  CATEGORY      │  │  │
│  │  └────────────────────┘  └──────────────┘  └────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │           Solution + Solution Version (Training)                 │  │
│  │           Recipe: USER_PERSONALIZATION / RELATED_ITEMS / ...     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│           ┌──────────────────┼────────────────────┐                   │
│           ▼                                        ▼                   │
│  ┌─────────────────┐                    ┌──────────────────────────┐  │
│  │    Campaign     │                    │  Batch Inference Job     │  │
│  │ (Real-time API) │                    │  (Offline bulk export)   │  │
│  └─────────────────┘                    └──────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
         │
         │ Real-time Events (PutEvents)
         ▲
    User Actions (click, buy, watch...)
```

---

## 📦 Datasets — Tập Dữ Liệu

### 1. Interactions Dataset (Tập Dữ Liệu Tương Tác) — Bắt Buộc

Lịch sử **hành vi tương tác** giữa user và item — đây là dataset cốt lõi nhất.

**Schema bắt buộc:**

| Cột | Kiểu | Mô Tả |
|---|---|---|
| `USER_ID` | string | Định danh duy nhất của người dùng |
| `ITEM_ID` | string | Định danh duy nhất của item (sản phẩm, phim, bài nhạc...) |
| `TIMESTAMP` | long | Unix timestamp (giây) của sự kiện |

**Schema mở rộng (tùy chọn nhưng rất quan trọng):**

| Cột | Kiểu | Mô Tả |
|---|---|---|
| `EVENT_TYPE` | string | Loại tương tác: `click`, `purchase`, `watch`, `rate`, `add_to_cart` |
| `EVENT_VALUE` | float | Giá trị tương tác: rating (1-5), watch_percentage (0-1), purchase_amount |

**Ví dụ dữ liệu:**

```csv
USER_ID,ITEM_ID,TIMESTAMP,EVENT_TYPE,EVENT_VALUE
user_001,movie_101,1704067200,watch,0.95
user_001,movie_205,1704153600,watch,0.30
user_001,movie_312,1704240000,purchase,1.0
user_002,movie_101,1704067200,click,null
user_002,movie_405,1704326400,watch,0.85
```

**Yêu cầu tối thiểu:**
- Ít nhất **1,000 unique users** (người dùng duy nhất)
- Ít nhất **25 unique items** (items duy nhất)
- Ít nhất **25 interactions per user** trung bình (một số recipe yêu cầu ít hơn)
- Lịch sử **ít nhất 2 năm** hoặc **tối thiểu 2 interactions per user**

### 2. Users Dataset (Tập Dữ Liệu Người Dùng) — Tùy Chọn

Metadata về người dùng — cải thiện recommendations cho **cold users** (user mới ít interactions).

```csv
USER_ID,AGE,GENDER,MEMBERSHIP_LEVEL,LOCATION
user_001,28,M,premium,hanoi
user_002,35,F,standard,hcm
```

**Reserved keywords:** `USER_ID` phải khớp chính xác với Interactions Dataset.

### 3. Items Dataset (Tập Dữ Liệu Mặt Hàng) — Tùy Chọn

Metadata về items — cải thiện recommendations cho **cold items** (item mới ít interactions) và content-based filtering.

```csv
ITEM_ID,GENRE,PRICE,BRAND,CATEGORY,CREATION_TIMESTAMP
movie_101,action|sci-fi,9.99,StudioA,movie,1609459200
movie_205,romance,7.99,StudioB,movie,1612137600
product_A,electronics,299.99,BrandX,laptop,1614556800
```

**`CREATION_TIMESTAMP`:** Quan trọng! Personalize dùng để xử lý temporal dynamics (động học thời gian) — items mới hơn được boost trong recommendations.

---

## 🍳 Recipes — Công Thức

**Recipe** (Công Thức) là thuật toán ML được định nghĩa sẵn cho từng use case cụ thể.

### Nhóm User Personalization (Cá Nhân Hóa Cho Người Dùng)

| Recipe | Mô Tả | Use Case |
|---|---|---|
| `aws-user-personalization` | Gợi ý items phù hợp nhất cho từng user | "Đề xuất cho bạn" — main recommendation |
| `aws-user-personalization-v2` | Phiên bản mới hơn với better exploration | Như trên, khuyến nghị cho dự án mới |
| `aws-trending-now` | Items đang trending trong toàn hệ thống | "Đang phổ biến" — không cần USER_ID |
| `aws-popularity-count` | Items phổ biến nhất theo số interactions | Fallback cho cold users |

### Nhóm Personalized Ranking (Xếp Hạng Cá Nhân Hóa)

| Recipe | Mô Tả | Use Case |
|---|---|---|
| `aws-personalized-ranking` | Re-rank một danh sách items theo preference của user | Search result personalization, curated list |

### Nhóm Related Items (Items Liên Quan)

| Recipe | Mô Tả | Use Case |
|---|---|---|
| `aws-similar-items` | Items tương tự với một item cụ thể (content + collab) | "Sản phẩm tương tự", "Phim cùng thể loại" |
| `aws-sims` (legacy) | Items liên quan dựa thuần CF (Collaborative Filtering) | Đã cũ — dùng `aws-similar-items` thay thế |

### Nhóm User Segmentation (Phân Khúc Người Dùng)

| Recipe | Mô Tả | Use Case |
|---|---|---|
| `aws-item-affinity` | Users có affinity (thiên ái) cao với một item | Marketing: "Ai muốn nhận thông báo về sản phẩm này?" |
| `aws-item-attribute-affinity` | Users có affinity với một attribute của items | "Ai thích thể loại Action?" |

---

## 🔧 Solution & Solution Version (Giải Pháp & Phiên Bản)

### Solution

**Solution** là cấu hình training: Recipe + Dataset Group + hyperparameters.

```python
import boto3

personalize = boto3.client("personalize", region_name="us-east-1")

# Tạo Solution
response = personalize.create_solution(
    name="movie-recommendation-solution",
    datasetGroupArn="arn:aws:personalize:us-east-1:123456789:dataset-group/movies-dg",
    recipeArn="arn:aws:personalize:::recipe/aws-user-personalization",
    solutionConfig={
        "algorithmHyperParameters": {
            "hidden_dimension": "149",      # Kích thước embedding layer
            "bptt": "32",                   # Backpropagation Through Time steps
            "recency_mask": "true"          # Ưu tiên interactions gần đây hơn
        },
        "eventValueThreshold": "0.5",      # Chỉ dùng events có value >= 0.5 (đã xem >50%)
        "trainingDataConfig": {
            "excludedDatasetColumns": {
                "INTERACTIONS": ["EVENT_VALUE"]  # Loại trừ cột này khỏi training
            }
        }
    }
)
solution_arn = response["solutionArn"]
```

### Solution Version

**Solution Version** là một lần training cụ thể của Solution — có thể có nhiều versions.

```python
# Train Solution Version
response = personalize.create_solution_version(
    solutionArn=solution_arn,
    trainingMode="FULL"  # "FULL" (train lại hoàn toàn) hoặc "UPDATE" (incremental)
)
solution_version_arn = response["solutionVersionArn"]

# Kiểm tra trạng thái training
import time
while True:
    response = personalize.describe_solution_version(
        solutionVersionArn=solution_version_arn
    )
    status = response["solutionVersion"]["status"]
    print(f"Status: {status}")
    if status in ["ACTIVE", "CREATE FAILED"]:
        break
    time.sleep(60)  # Kiểm tra mỗi phút
```

### Đánh Giá Solution Version

```python
# Lấy metrics đánh giá model
response = personalize.get_solution_metrics(
    solutionVersionArn=solution_version_arn
)
metrics = response["metrics"]

print(f"Precision@10: {metrics['coverage']:.4f}")
print(f"NDCG@10:      {metrics['mean_reciprocal_rank_at_25']:.4f}")
print(f"Coverage:     {metrics['coverage']:.4f}")
```

**Metrics quan trọng:**

| Metric | Ý Nghĩa | Giá Trị Tốt |
|---|---|---|
| **Precision@K** | Trong top-K gợi ý, bao nhiêu % là relevant | > 0.05 (5%) |
| **NDCG@K** (Normalized Discounted Cumulative Gain — Lợi Nhuận Tích Lũy Chiết Khấu Chuẩn Hóa) | Đánh giá thứ tự ranking của items relevant | > 0.05 |
| **Mean Reciprocal Rank** (MRR — Thứ Hạng Đối Nghịch Trung Bình) | Vị trí trung bình của item relevant đầu tiên | > 0.05 |
| **Coverage** (Độ Phủ) | % items trong catalog được recommend ít nhất một lần | > 0.1 |

---

## 🚀 Campaign (Chiến Dịch) — Real-time Serving

**Campaign** là endpoint triển khai Solution Version để nhận real-time recommendation requests.

```python
# Tạo Campaign
response = personalize.create_campaign(
    name="movie-recommendation-campaign",
    solutionVersionArn=solution_version_arn,
    minProvisionedTPS=1,   # TPS — Transactions Per Second (Giao Dịch Mỗi Giây) tối thiểu
    campaignConfig={
        "enableMetadataWithRecommendations": True,  # Trả về metadata của item trong response
        "itemExplorationConfig": {
            "explorationWeight": "0.3",    # 30% recommendations từ exploration (khám phá)
            "explorationItemAgeCutOff": "30"  # Chỉ explore items ≤ 30 ngày tuổi
        }
    }
)
campaign_arn = response["campaignArn"]
```

### Get Recommendations (Lấy Gợi Ý)

```python
import boto3

personalize_runtime = boto3.client("personalize-runtime", region_name="us-east-1")

def get_recommendations(user_id: str, num_results: int = 10):
    """Lấy top-N gợi ý cho một user cụ thể."""
    response = personalize_runtime.get_recommendations(
        campaignArn=campaign_arn,
        userId=user_id,
        numResults=num_results,
        context={
            # Context features (đặc trưng ngữ cảnh): giá trị hiện tại của user
            "DEVICE": "mobile",
            "TIME_OF_DAY": "evening"
        },
        filterArn="arn:aws:personalize:...:filter/exclude-purchased"  # Tùy chọn
    )
    return response["itemList"]

# Kết quả trả về:
# [{"itemId": "movie_101", "score": 0.0523}, {"itemId": "movie_205", "score": 0.0487}, ...]
```

### Personalized Ranking (Xếp Hạng Cá Nhân Hóa)

```python
def get_personalized_ranking(user_id: str, item_list: list):
    """
    Re-rank một danh sách items có sẵn theo preferences của user.
    Dùng cho search results, curated lists, category pages.
    """
    response = personalize_runtime.get_personalized_ranking(
        campaignArn=ranking_campaign_arn,
        userId=user_id,
        inputList=item_list  # Danh sách item_ids cần re-rank
    )
    return response["personalizedRanking"]

# Ví dụ:
items = ["movie_A", "movie_B", "movie_C", "movie_D"]
ranked = get_personalized_ranking("user_001", items)
# Output: [{"itemId": "movie_C", "score": 0.09}, {"itemId": "movie_A", "score": 0.07}, ...]
```

---

## ⚡ Real-time Event Tracking (Theo Dõi Sự Kiện Thời Gian Thực)

### PutEvents — Cập Nhật Hành Vi User Ngay Lập Tức

Khi user thực hiện action mới (click, purchase...), dùng `PutEvents` để cập nhật recommendations **ngay lập tức** mà không cần retrain model.

```python
import time
import uuid

personalize_events = boto3.client("personalize-events", region_name="us-east-1")

def track_user_event(user_id: str, item_id: str, event_type: str, event_value: float = None):
    """
    Track real-time user event để cải thiện recommendations tức thì.
    Gọi hàm này ngay khi user thực hiện action trong ứng dụng.
    """
    event = {
        "eventId": str(uuid.uuid4()),
        "eventType": event_type,        # "click", "purchase", "watch", "rate"
        "sentAt": int(time.time()),     # Unix timestamp hiện tại
        "itemId": item_id,
        "properties": json.dumps({
            "eventValue": event_value   # Rating, watch percentage, purchase amount
        })
    }

    personalize_events.put_events(
        trackingId=os.environ["PERSONALIZE_TRACKING_ID"],
        userId=user_id,
        sessionId=request.session_id,  # Nhóm các events trong cùng phiên
        eventList=[event]
    )

# Ví dụ sử dụng trong ứng dụng:
# User vừa mua sản phẩm → track ngay
track_user_event("user_001", "product_A", "purchase", 1.0)
# → Lần tiếp theo get_recommendations cho user_001 sẽ reflect purchase này
```

### Event Tracker (Bộ Theo Dõi Sự Kiện)

```python
# Tạo Event Tracker trước khi dùng PutEvents
response = personalize.create_event_tracker(
    name="website-event-tracker",
    datasetGroupArn="arn:aws:personalize:...:dataset-group/movies-dg"
)
tracking_id = response["trackingId"]
# Lưu tracking_id vào environment variable / config
```

---

## 📤 Batch Inference Job (Công Việc Suy Luận Hàng Loạt)

Dùng Batch Inference khi cần **recommendations cho toàn bộ users** cùng lúc (nightly job, email campaigns):

```python
# File input JSON (mỗi dòng một user):
# {"userId": "user_001"}
# {"userId": "user_002"}
# {"userId": "user_003"}

personalize.create_batch_inference_job(
    jobName="nightly-recommendation-batch",
    solutionVersionArn=solution_version_arn,
    numResults=20,  # Top 20 gợi ý cho mỗi user
    jobInput={
        "s3DataSource": {
            "path": "s3://my-bucket/batch-input/users.json",
            "kmsKeyArn": "arn:aws:kms:..."  # Tùy chọn encryption
        }
    },
    jobOutput={
        "s3DataDestination": {
            "path": "s3://my-bucket/batch-output/",
        }
    },
    roleArn="arn:aws:iam::123456789:role/PersonalizeRole"
)

# Output JSON (mỗi dòng một user):
# {"input": {"userId": "user_001"}, "output": {"recommendedItems": ["movie_101", "movie_205", ...]}}
```

---

## 🔍 Filters (Bộ Lọc)

**Filter** loại trừ hoặc include items dựa trên điều kiện — áp dụng tại thời điểm inference (không cần retrain).

```python
# Tạo Filter: Loại bỏ items user đã mua
personalize.create_filter(
    name="exclude-purchased-items",
    datasetGroupArn="arn:aws:personalize:...:dataset-group/movies-dg",
    filterExpression='EXCLUDE itemId WHERE INTERACTIONS.event_type = "purchase"'
)

# Tạo Filter: Chỉ gợi ý items thuộc một category cụ thể
personalize.create_filter(
    name="only-action-movies",
    datasetGroupArn="...",
    filterExpression='INCLUDE itemId WHERE ITEMS.GENRE = "action"'
)

# Tạo Filter với dynamic parameter (tham số động khi gọi API)
personalize.create_filter(
    name="genre-filter",
    datasetGroupArn="...",
    filterExpression='INCLUDE itemId WHERE ITEMS.GENRE = $GENRE'
)

# Dùng filter với dynamic value khi get recommendations
personalize_runtime.get_recommendations(
    campaignArn=campaign_arn,
    userId="user_001",
    filterArn="arn:aws:personalize:...:filter/genre-filter",
    filterValues={"GENRE": '"sci-fi"'}  # Chú ý cặp ngoặc kép trong chuỗi
)
```

---

## 🔄 Incremental Training & AutoML

### Training Mode

| Mode | Mô Tả | Khi Dùng |
|---|---|---|
| `FULL` | Train lại model từ đầu với toàn bộ data | Mỗi tuần/tháng để học long-term patterns mới |
| `UPDATE` | Chỉ cập nhật model với interactions mới (nhanh hơn nhiều) | Hàng ngày hoặc mỗi vài giờ |

```python
# Training đầy đủ (hàng tuần)
personalize.create_solution_version(solutionVersionArn=solution_arn, trainingMode="FULL")

# Incremental update (hàng ngày — nhanh và rẻ hơn)
personalize.create_solution_version(solutionVersionArn=solution_arn, trainingMode="UPDATE")
```

### Automatic Training (Huấn Luyện Tự Động)

```python
# Bật automatic training khi tạo Solution
personalize.create_solution(
    name="auto-train-solution",
    datasetGroupArn="...",
    recipeArn="arn:aws:personalize:::recipe/aws-user-personalization",
    solutionConfig={
        "autoTrainingConfig": {
            "schedulingExpression": "rate(7 days)"  # Train mỗi 7 ngày
        }
    }
)
```

---

## 💰 Pricing — Mô Hình Tính Phí

| Thành Phần | Đơn Vị Tính Phí | Ghi Chú |
|---|---|---|
| **Training** | $ per TPS-hour (Training) | Training thường tốn vài chục $ cho medium dataset |
| **Campaign (TPS provisioned)** | $ per TPS-hour | Phí cao nhất — tính theo TPS tối thiểu cam kết, **24/7** |
| **Real-time event ingestion** | $ per event (PutEvents) | Thường rất thấp |
| **Batch Inference** | $ per user processed | Rẻ hơn Campaign nhiều |

**Lưu ý quan trọng:** Campaign tính phí theo `minProvisionedTPS` **kể cả khi không có traffic**. Nếu cần tiết kiệm chi phí cho traffic thấp, dùng Batch Inference thay vì Campaign.

```python
# Xóa Campaign khi không cần nữa để tránh phát sinh chi phí
personalize.delete_campaign(campaignArn=campaign_arn)
```

---

## 🔗 Tích Hợp Với Ứng Dụng

### Backend API Pattern (Mẫu API Backend)

```python
from fastapi import FastAPI
import boto3

app = FastAPI()
personalize_runtime = boto3.client("personalize-runtime", region_name="us-east-1")
CAMPAIGN_ARN = os.environ["PERSONALIZE_CAMPAIGN_ARN"]

@app.get("/recommendations/{user_id}")
async def get_recommendations(user_id: str, limit: int = 10):
    """API endpoint trả về gợi ý cho user."""
    try:
        response = personalize_runtime.get_recommendations(
            campaignArn=CAMPAIGN_ARN,
            userId=user_id,
            numResults=limit
        )
        item_ids = [item["itemId"] for item in response["itemList"]]

        # Lấy metadata từ DynamoDB/RDS theo item_ids
        items_detail = fetch_items_from_db(item_ids)
        return {"userId": user_id, "recommendations": items_detail}

    except personalize_runtime.exceptions.ResourceNotFoundException:
        # Fallback cho new user (cold start): trả về popular items
        return get_popular_items(limit)
```

### Cache Layer (Tầng Cache) Với ElastiCache

```
User Request → API Gateway → Lambda
                               │
                               ├── Cache Hit (ElastiCache Redis)
                               │   └─► Trả kết quả ngay (<1ms)
                               │
                               └── Cache Miss
                                   ├── Gọi Personalize Campaign (~50-100ms)
                                   ├── Lưu vào Redis (TTL = 1 giờ)
                                   └─► Trả kết quả cho user
```

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Personalize khác gì Bedrock cho recommendation use case?**

A: Personalize là **purpose-built** (được xây dựng chuyên biệt) cho recommendation systems với collaborative filtering, hành vi người dùng theo thời gian, và real-time event tracking. Bedrock (LLMs) có thể tạo recommendations dạng text nhưng không scale tốt cho hàng triệu users/items và không học patterns từ behavioral data theo thời gian. Personalize phù hợp cho high-volume recommendation (triệu user), Bedrock phù hợp cho conversational recommendations ("gợi ý quà tặng cho bạn gái thích đọc sách").

**Q: Làm thế nào xử lý cold start cho user mới trong Personalize?**

A: Bốn chiến lược: (1) **Users Dataset metadata** — nếu biết age/gender/location của user mới, Personalize dùng để bootstrap. (2) **Popular items fallback** — dùng `aws-popularity-count` recipe hoặc filter. (3) **Exploration** trong `aws-user-personalization` — `explorationWeight` cao hơn = nhiều exploration hơn cho users ít interactions. (4) **Onboarding flow** — hỏi user về preferences ngay từ đầu, track qua PutEvents trước khi gọi GetRecommendations.

**Q: Campaign vs Batch Inference — khi nào dùng cái nào?**

A: **Campaign** (real-time) khi: cần recommendations ngay lập tức khi user đang online, cần reflect real-time events (vừa click/mua), latency < 100ms quan trọng. **Batch Inference** khi: pre-compute recommendations cho email marketing campaigns, cần recommendations cho toàn bộ user base cùng lúc, traffic thấp và không cần real-time.

**Q: Filter expression syntax như thế nào?**

A: `EXCLUDE itemId WHERE INTERACTIONS.event_type = "purchase"` (loại trừ đã mua); `INCLUDE itemId WHERE ITEMS.CATEGORY IN ("electronics", "gadgets")` (chỉ category cụ thể); Dùng `$PARAM` cho dynamic values: `INCLUDE itemId WHERE ITEMS.PRICE < $MAX_PRICE`.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
