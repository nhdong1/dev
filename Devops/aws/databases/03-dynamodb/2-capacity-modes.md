# DynamoDB — Capacity Modes (Chế Độ Năng Lực)

> Hiểu On-Demand vs Provisioned capacity, RCU/WCU tính toán, Auto Scaling (Tự Động Co Giãn), Throttling (Giới Hạn Tốc Độ), và chiến lược tối ưu chi phí cho DynamoDB.

## 📚 Mục Lục

1. [Tổng Quan Capacity Modes](#tổng-quan-capacity-modes)
2. [RCU & WCU — Đơn Vị Năng Lực Đọc/Ghi](#rcu--wcu--đơn-vị-năng-lực-đọcghi)
3. [On-Demand Mode — Chế Độ Theo Yêu Cầu](#on-demand-mode--chế-độ-theo-yêu-cầu)
4. [Provisioned Mode — Chế Độ Được Cung Cấp Sẵn](#provisioned-mode--chế-độ-được-cung-cấp-sẵn)
5. [Auto Scaling — Tự Động Co Giãn](#auto-scaling--tự-động-co-giãn)
6. [Throttling — Giới Hạn Tốc Độ](#throttling--giới-hạn-tốc-độ)
7. [Burst Capacity — Năng Lực Đột Biến](#burst-capacity--năng-lực-đột-biến)
8. [Tính Toán RCU/WCU](#tính-toán-rcuwcu)
9. [So Sánh Chi Phí](#so-sánh-chi-phí)
10. [Chiến Lược Chọn Lựa](#chiến-lược-chọn-lựa)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Capacity Modes

DynamoDB cung cấp 2 chế độ năng lực, có thể chuyển đổi qua lại (mỗi bảng chỉ chuyển được 2 lần/ngày):

```
┌─────────────────────────────┬─────────────────────────────┐
│       ON-DEMAND             │       PROVISIONED           │
│   (Theo Yêu Cầu)            │   (Được Cung Cấp Sẵn)      │
├─────────────────────────────┼─────────────────────────────┤
│ Không cần cấu hình trước    │ Đặt trước RCU + WCU         │
│ Scale tự động ngay lập tức  │ Auto Scaling điều chỉnh dần │
│ Trả theo request            │ Trả theo giờ (capacity đặt) │
│ Giá cao hơn ~2.5x           │ Giá thấp hơn nếu dự đoán đc│
│ Phù hợp: unpredictable load │ Phù hợp: predictable load   │
└─────────────────────────────┴─────────────────────────────┘
```

---

## RCU & WCU — Đơn Vị Năng Lực Đọc/Ghi

### RCU — Read Capacity Unit (Đơn Vị Năng Lực Đọc)

```
1 RCU = 1 strongly consistent read/giây cho item ≤ 4 KB
      = 2 eventually consistent reads/giây cho item ≤ 4 KB
      = 0.5 transactional read/giây cho item ≤ 4 KB
```

**Giải thích:**
- **Strongly consistent read** (Đọc Nhất Quán Mạnh): Đảm bảo đọc data mới nhất. Tốn 1 RCU/read.
- **Eventually consistent read** (Đọc Nhất Quán Cuối Cùng): Có thể đọc data cũ vài giây. Tốn 0.5 RCU/read (hiệu quả hơn).
- **Transactional read** (Đọc Giao Dịch): Trong transaction, tốn 2 RCU/read.

### WCU — Write Capacity Unit (Đơn Vị Năng Lực Ghi)

```
1 WCU = 1 standard write/giây cho item ≤ 1 KB
      = 0.5 transactional write/giây cho item ≤ 1 KB (tức là 2 WCU/write)
```

**Lưu ý:** Item lớn hơn tự động làm tròn lên:
- Item 1.5 KB → 2 WCU
- Item 3.2 KB → 4 WCU
- Item 3.9 KB → 4 WCU

### Bảng Tóm Tắt RCU/WCU

| Thao Tác                           | Công Thức Tính                              |
| ---------------------------------- | ------------------------------------------- |
| Strongly consistent read           | ceil(item_size_KB / 4) × 1 RCU             |
| Eventually consistent read         | ceil(item_size_KB / 4) × 0.5 RCU           |
| Transactional read                 | ceil(item_size_KB / 4) × 2 RCU             |
| Standard write                     | ceil(item_size_KB / 1) × 1 WCU             |
| Transactional write                | ceil(item_size_KB / 1) × 2 WCU             |

---

## On-Demand Mode — Chế Độ Theo Yêu Cầu

### Cơ Chế Hoạt Động

On-Demand Mode tự động scale capacity theo traffic thực tế — không cần cấu hình hay quản lý.

```
Traffic Thực Tế:
   10:00 —  1,000 requests/s  → DynamoDB xử lý tất cả
   10:30 — 50,000 requests/s  → DynamoDB tự scale lên
   11:00 —  2,000 requests/s  → DynamoDB scale về
   (Không cần làm gì — hoàn toàn tự động)
```

### Giá On-Demand (us-east-1)

```
Read Request Units  (RRU): $0.25  per million
Write Request Units (WRU): $1.25  per million
Storage:                   $0.25  per GB/month

(Giá tham khảo, có thể thay đổi — kiểm tra AWS Pricing page)
```

### Giới Hạn On-Demand

- **Instantaneous doubling**: DynamoDB có thể tăng gấp đôi throughput ngay lập tức so với peak trước đó
- **Cold table**: Bảng mới (chưa có traffic) được cấp throughput mặc định ban đầu
- Không có throttling trong điều kiện bình thường (trừ khi exceed service limits)

### Khi Nào Dùng On-Demand

```
✅ Ứng dụng mới — chưa biết traffic pattern
✅ Traffic không đoán được hoặc có spike lớn (ví dụ: flash sale)
✅ Development/testing environments
✅ Traffic rất thấp (< vài nghìn requests/ngày) — rẻ hơn provisioned
✅ Ứng dụng serverless cần zero management
```

---

## Provisioned Mode — Chế Độ Được Cung Cấp Sẵn

### Cơ Chế Hoạt Động

Bạn đặt trước số lượng RCU và WCU. DynamoDB đảm bảo capacity đó luôn sẵn sàng.

```
Cấu Hình:  ReadCapacityUnits  = 1,000 RCU/s
           WriteCapacityUnits = 500  WCU/s

Kết Quả:
- Requests ≤ 1,000 reads/s  → Xử lý bình thường
- Requests ≤ 500  writes/s  → Xử lý bình thường
- Requests > giới hạn       → ProvisionedThroughputExceededException
```

### Giá Provisioned (us-east-1)

```
Read Capacity Unit  (RCU): $0.00013 per RCU-hour
Write Capacity Unit (WCU): $0.00065 per WCU-hour

Ví dụ: 1,000 RCU + 100 WCU trong 1 tháng (720 giờ):
- Read:  1,000 × $0.00013 × 720 = $93.6/tháng
- Write: 100   × $0.00065 × 720 = $46.8/tháng
- Tổng:  ~$140.4/tháng (chưa tính storage)
```

### Reserved Capacity (Capacity Dự Trữ)

Giống Reserved Instances của EC2 — cam kết 1 hoặc 3 năm để giảm giá lên đến 77%:

```
On-Demand: $0.00013/RCU-hour
Reserved (1-year): ~$0.00006/RCU-hour → tiết kiệm ~54%
Reserved (3-year): ~$0.00003/RCU-hour → tiết kiệm ~77%
```

---

## Auto Scaling — Tự Động Co Giãn

### Cách Hoạt Động

Auto Scaling điều chỉnh provisioned RCU/WCU tự động dựa trên CloudWatch metrics, nhưng **không phải tức thì** — có độ trễ vài phút.

```
     Actual Usage (%)
          │
    100%  │          ┌──────┐
     75%  │─ Target ─┤      ├──────────
     50%  │          │      │     Auto Scaling
     25%  │          │      │     tăng capacity
      0%  └──────────┴──────┴────────────────> Thời gian

CloudWatch thấy usage > target (75%) trong ~2-3 phút
→ Trigger Scale Up
→ Mất ~1-3 phút để có hiệu lực
→ Trong thời gian đó: THROTTLING có thể xảy ra!
```

### Cấu Hình Auto Scaling

```json
{
  "TableName": "Orders",
  "ScalingPolicy": {
    "ReadCapacity": {
      "MinCapacity": 5,
      "MaxCapacity": 10000,
      "TargetUtilization": 70
    },
    "WriteCapacity": {
      "MinCapacity": 5,
      "MaxCapacity": 5000,
      "TargetUtilization": 70
    }
  }
}
```

**TargetUtilization = 70%** là mặc định an toàn:
- Khi usage đạt 70% × capacity → bắt đầu scale up
- Buffer 30% xử lý traffic tăng trong thời gian scale

### Auto Scaling Không Đủ Nhanh

Nếu traffic tăng đột ngột (spike — đột biến lớn), Auto Scaling không kịp phản ứng:

```
Giải Pháp:
1. Tăng MaxCapacity cao hơn dự đoán
2. Dùng On-Demand cho workloads có spike bất thường
3. Kết hợp DAX (DynamoDB Accelerator) để giảm tải đọc
4. Implement application-level retry với exponential backoff
```

---

## Throttling — Giới Hạn Tốc Độ

### Khi Nào Throttling Xảy Ra

```
Provisioned Mode:
- Requests vượt quá RCU/WCU đã đặt
- Một partition bị hot (vượt 3,000 RCU hoặc 1,000 WCU/giây)

On-Demand Mode:
- Rất hiếm khi throttling
- Chỉ xảy ra nếu traffic tăng > 2x peak trong thời gian rất ngắn
```

### Lỗi Throttling

```
ProvisionedThroughputExceededException
→ HTTP 400, không phải 500
→ Cần retry từ phía client
```

### Xử Lý Throttling

```python
import boto3
from botocore.exceptions import ClientError
import time

def get_item_with_retry(table, key, max_retries=5):
    for attempt in range(max_retries):
        try:
            return table.get_item(Key=key)
        except ClientError as e:
            if e.response['Error']['Code'] == 'ProvisionedThroughputExceededException':
                # Exponential backoff (Lui Dần Theo Hàm Mũ)
                wait_time = (2 ** attempt) + (random.random() * 0.1)
                print(f"Throttled. Retry {attempt+1}/{max_retries} sau {wait_time:.2f}s")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

AWS SDK tự động retry với exponential backoff cho ProvisionedThroughputExceededException — nên dùng SDK thay vì tự implement nếu có thể.

### Adaptive Capacity (Năng Lực Thích Ứng)

DynamoDB có cơ chế Adaptive Capacity tự động redistribute (phân phối lại) capacity từ partitions ít dùng sang partitions bị throttle. Tuy nhiên:

- **Không giải quyết** hot partition do partition key design xấu
- **Chỉ giúp** khi traffic không đều giữa các partitions trong cùng bảng

---

## Burst Capacity — Năng Lực Đột Biến

### Cơ Chế Burst Capacity

DynamoDB dự trữ capacity chưa dùng trong 5 phút gần nhất làm "burst capacity" (năng lực đột biến):

```
Ví Dụ:
Provisioned: 1,000 WCU/s
Thực tế dùng 5 phút qua: 500 WCU/s (chỉ dùng 50%)
→ Dư 500 WCU/s × 300s = 150,000 WCU dự trữ

Khi spike xảy ra:
- Có thể dùng đến 150,000 WCU "ngay lập tức"
- Hữu ích cho spikes ngắn hạn (<5 phút)
```

**Lưu ý quan trọng:** Burst capacity không đảm bảo — AWS không cam kết luôn luôn có. Đừng thiết kế hệ thống phụ thuộc vào burst capacity.

---

## Tính Toán RCU/WCU

### Ví Dụ Thực Tế 1: Blog Platform

```
Yêu Cầu:
- 1,000,000 bài đọc mỗi ngày
- 10,000 bài viết mỗi ngày
- Kích thước trung bình mỗi bài: 8 KB
- 80% eventually consistent reads, 20% strongly consistent

Tính Reads:
- Eventually: 800,000 reads/day × ceil(8KB/4KB) × 0.5 RCU
  = 800,000 × 2 × 0.5 = 800,000 RCU/day
- Strongly:   200,000 reads/day × ceil(8KB/4KB) × 1 RCU
  = 200,000 × 2 × 1 = 400,000 RCU/day
- Tổng: 1,200,000 RCU/day ÷ 86,400s ≈ 14 RCU/s

Tính Writes:
- 10,000 writes/day × ceil(8KB/1KB) = 10,000 × 8 = 80,000 WCU/day
- 80,000 ÷ 86,400s ≈ 1 WCU/s

Provisioned: 20 RCU, 5 WCU (với buffer)
```

### Ví Dụ Thực Tế 2: E-Commerce Flash Sale

```
Kịch Bản: Flash sale kéo dài 1 giờ
- Peak: 50,000 orders/phút = ~833 orders/giây
- Mỗi order = 2 writes (insert order + update inventory) × 2 KB
- Reads đồng thời: 200,000 product views/phút, item 1 KB

Writes:
- 833 orders/s × 2 writes × ceil(2KB/1KB) = 833 × 2 × 2 = 3,332 WCU/s

Reads (eventually consistent):
- 200,000/60 ≈ 3,333 reads/s × ceil(1KB/4KB) × 0.5 = ~834 RCU/s

→ Kết Luận: On-Demand phù hợp hơn cho flash sale
   (hoặc Provisioned với Auto Scaling + cao MaxCapacity)
```

---

## So Sánh Chi Phí

### Điểm Hòa Vốn On-Demand vs Provisioned

```
Với 1 million requests:
On-Demand  Read:  $0.25/million  RRU
Provisioned Read: $0.00013/RCU-hour × 24h × 30 days = ~$0.094/RCU/month

Nếu cần 1 RCU thường xuyên suốt tháng:
- On-Demand:   1M reads × $0.25 = $250 (nếu 1M reads/tháng)
- Provisioned: 1 RCU × $0.094 = $0.094/tháng (nếu dùng đều)

→ Provisioned rẻ hơn khi utilization > ~40%
```

### Công Cụ Ước Tính Chi Phí

```
AWS Pricing Calculator:
https://calculator.aws.amazon.com/

Nhập vào:
- Số reads/giây (RCU)
- Số writes/giây (WCU)
- Storage (GB)
- Consistency type
→ AWS tự tính chi phí On-Demand và Provisioned
```

---

## Chiến Lược Chọn Lựa

### Decision Tree (Cây Quyết Định)

```
Câu hỏi 1: Traffic có dự đoán được không?
│
├── Không (spike bất thường, ứng dụng mới)
│   └── → On-Demand Mode
│
└── Có (stable hoặc predictable pattern)
    │
    Câu hỏi 2: Có spike ngắn hạn không?
    │
    ├── Có spike nhưng biết trước (sale events, marketing)
    │   └── → Provisioned + Auto Scaling + manual scale trước sự kiện
    │
    └── Không có spike đáng kể
        └── → Provisioned + Auto Scaling (tiết kiệm nhất)
```

### Hybrid Strategy (Chiến Lược Kết Hợp)

Dùng **Provisioned với Auto Scaling** cho baseline traffic + **On-Demand** trong periods có spike lớn:

```python
# Trước flash sale: chuyển sang On-Demand
dynamodb.update_table(
    TableName='Products',
    BillingMode='PAY_PER_REQUEST'
)

# Sau flash sale: chuyển về Provisioned tiết kiệm
dynamodb.update_table(
    TableName='Products',
    BillingMode='PROVISIONED',
    ProvisionedThroughput={
        'ReadCapacityUnits': 500,
        'WriteCapacityUnits': 100
    }
)

# Lưu ý: chỉ chuyển được 2 lần/24 giờ
```

---

## Câu Hỏi Phỏng Vấn

**Q: On-Demand vs Provisioned — khi nào chọn cái nào?**

A: Chọn **On-Demand** khi: traffic không đoán được, ứng dụng mới, có flash sale bất thường, hoặc workload rất thấp. Chọn **Provisioned** (với Auto Scaling) khi: traffic ổn định và có thể dự đoán, muốn tối ưu chi phí (rẻ hơn ~40-70%), hoặc cần Reserved Capacity để tiết kiệm thêm. Trade-off chính: Provisioned rẻ hơn nhưng cần quản lý; On-Demand đắt hơn nhưng zero-hassle.

**Q: Throttling là gì và cách xử lý?**

A: Throttling trong DynamoDB xảy ra khi requests vượt quá provisioned capacity hoặc partition limit (3,000 RCU / 1,000 WCU mỗi partition). Kết quả là `ProvisionedThroughputExceededException` (HTTP 400). Xử lý: (1) AWS SDK tự động retry với exponential backoff; (2) Implement retry logic nếu dùng raw API; (3) Dùng DAX để giảm read load; (4) Thiết kế partition key tốt hơn nếu hot partition; (5) Chuyển sang On-Demand cho burst traffic.

**Q: Tính toán RCU cần cho use case: 10,000 reads/giây, mỗi item 6 KB, eventually consistent?**

A: `RCU = ceil(6KB / 4KB) × 0.5 × 10,000 = 2 × 0.5 × 10,000 = 10,000 RCU/giây`. Nếu strongly consistent: `2 × 1 × 10,000 = 20,000 RCU/giây`. Đây là lý do tại sao nên dùng eventually consistent khi business logic cho phép — tiết kiệm 50% RCU.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn Thành
