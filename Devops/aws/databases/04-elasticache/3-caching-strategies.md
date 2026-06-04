# Caching Strategies — Chiến Lược Bộ Nhớ Đệm

> Chiến lược caching quyết định **khi nào** dữ liệu được đưa vào cache và **khi nào** cache được cập nhật khi dữ liệu thay đổi. Chọn sai chiến lược dẫn đến stale data (dữ liệu cũ), cache stampede (cơn lũ cache), hoặc tốn RAM không cần thiết.

---

## 🗺️ Tổng Quan Các Chiến Lược

| Chiến Lược | Tên Khác | Cache Được Điền Bởi | Khi Nào Cập Nhật |
|-----------|---------|-------------------|-----------------|
| **Lazy Loading** | Cache-Aside, Read-Through | Application (ứng dụng) | Chỉ khi cache miss |
| **Write-Through** | Write-Through | Application (khi ghi) | Mỗi lần ghi vào DB |
| **Write-Around** | Write-Bypass | Application (khi ghi) | Không cache khi ghi |
| **Write-Behind** | Write-Back | Application | Ghi DB bất đồng bộ |
| **Read-Through** | - | Cache layer tự động | Khi cache miss |
| **Refresh-Ahead** | Pre-Fetch | Background process | Trước khi TTL hết hạn |

---

## 1️⃣ Lazy Loading — Tải Chậm (Cache-Aside — Cache Bên Cạnh)

### Cơ Chế Hoạt Động

```
READ FLOW (Luồng Đọc):

Application → Cache → Hit? → Trả về data
                  ↓ Miss
             → Database → Trả về data → Lưu vào Cache → Trả về data
```

### Sơ Đồ Chi Tiết

```
┌─────────────────────────────────────────────────────┐
│                    Lazy Loading                      │
│                                                      │
│  Request: GET product:123                            │
│                │                                     │
│                ▼                                     │
│       ┌──────────────────┐                          │
│       │  Cache (Redis)   │                          │
│       │  product:123 ?   │                          │
│       └────────┬─────────┘                          │
│                │                                     │
│         ┌──────┴──────┐                             │
│         │             │                             │
│    Cache HIT     Cache MISS                         │
│         │             │                             │
│         ▼             ▼                             │
│    Return data   Query Database                     │
│    (~0.1ms)           │                             │
│                       ▼                             │
│                 Get data (~10ms)                    │
│                       │                             │
│                       ▼                             │
│                 SET cache key                       │
│                 (with TTL)                          │
│                       │                             │
│                       ▼                             │
│                 Return data                         │
└─────────────────────────────────────────────────────┘
```

### Triển Khai Code

```python
import redis
import json
import time

redis_client = redis.Redis(host='my-redis.cache.amazonaws.com', port=6379)

def get_product(product_id: int) -> dict:
    cache_key = f"product:{product_id}"
    
    # Bước 1: Đọc từ cache trước
    cached_data = redis_client.get(cache_key)
    
    if cached_data:
        # Cache HIT — trả về ngay, cực nhanh
        return json.loads(cached_data)
    
    # Cache MISS — query database
    product = database.query(
        "SELECT * FROM products WHERE id = %s", 
        (product_id,)
    )
    
    if product:
        # Lưu vào cache với TTL (Time-To-Live) 1 giờ
        redis_client.setex(
            name=cache_key,
            time=3600,         # 3600 giây = 1 giờ
            value=json.dumps(product)
        )
    
    return product

def update_product(product_id: int, data: dict) -> None:
    # Cập nhật database
    database.execute(
        "UPDATE products SET ... WHERE id = %s", 
        (product_id,)
    )
    
    # Xóa cache cũ (Cache Invalidation — Vô Hiệu Hóa Cache)
    # Lần đọc tiếp theo sẽ tạo cache mới từ DB
    redis_client.delete(f"product:{product_id}")
```

### Ưu & Nhược Điểm

| ✅ Ưu Điểm | ❌ Nhược Điểm |
|-----------|--------------|
| Chỉ cache data thực sự được đọc (tiết kiệm RAM) | **Cache miss penalty** — request đầu tiên chậm hơn |
| Cache failure không block reads | **Stale data** — DB thay đổi, cache có thể cũ |
| Simple to implement (đơn giản triển khai) | **Cache stampede** — nhiều request cùng miss |
| Linh hoạt per-object TTL | Cần viết cache invalidation logic |

### Vấn Đề Cache Stampede (Cơn Lũ Cache) & Giải Pháp

```
Vấn đề:
  TTL hết → 1000 requests cùng lúc → 1000 DB queries → DB quá tải

Giải pháp 1: Mutex Lock (Khóa Loại Trừ)
  - Chỉ 1 request được query DB, rest (còn lại) chờ
  - Dùng Redis SETNX (Set if Not eXists — Đặt Nếu Không Tồn Tại) làm lock

Giải pháp 2: Probabilistic Early Expiration (Hết Hạn Xác Suất Sớm)
  - Trước khi TTL hết, một số requests ngẫu nhiên refresh cache sớm
  - Phân tán load refresh

Giải pháp 3: Background Refresh (Làm Mới Nền)
  - Một worker riêng refresh cache định kỳ
  - Requests luôn được phục vụ từ cache (có thể hơi cũ)
```

```python
import threading

def get_product_with_mutex(product_id: int) -> dict:
    cache_key = f"product:{product_id}"
    lock_key  = f"lock:product:{product_id}"
    
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Thử lấy lock với TTL 5 giây
    lock_acquired = redis_client.set(
        lock_key, "1", 
        nx=True,   # Only if Not eXists
        ex=5       # TTL 5 giây — tự giải phóng nếu process chết
    )
    
    if lock_acquired:
        try:
            # Tôi có lock → query DB
            product = database.query("SELECT * FROM products WHERE id = %s", (product_id,))
            redis_client.setex(cache_key, 3600, json.dumps(product))
            return product
        finally:
            redis_client.delete(lock_key)  # Giải phóng lock
    else:
        # Người khác đang query → chờ và retry
        time.sleep(0.1)
        cached = redis_client.get(cache_key)
        return json.loads(cached) if cached else database.query(...)
```

---

## 2️⃣ Write-Through — Ghi Xuyên

### Cơ Chế Hoạt Động

```
WRITE FLOW (Luồng Ghi):

Application → Cache (Ghi) → Database (Ghi)
                            (Ghi song song hoặc tuần tự)

READ FLOW:
Application → Cache → HIT (luôn có data mới nhất) → Trả về
```

### Sơ Đồ Chi Tiết

```
┌──────────────────────────────────────────────────────┐
│                   Write-Through                       │
│                                                       │
│  Update product:123 price=200                         │
│                │                                      │
│                ▼                                      │
│       ┌──────────────────┐                           │
│       │  Application     │                           │
│       └──────┬───────────┘                           │
│              │                                        │
│       ┌──────▼──────────────────────────────┐        │
│       │  Bước 1: Ghi vào Cache              │        │
│       │  SET product:123 {price: 200}        │        │
│       └──────────────────────────────────────┘        │
│              │                                        │
│       ┌──────▼──────────────────────────────┐        │
│       │  Bước 2: Ghi vào Database           │        │
│       │  UPDATE products SET price=200       │        │
│       │  WHERE id=123                        │        │
│       └──────────────────────────────────────┘        │
│                                                       │
│  Kết quả: Cache và DB luôn đồng bộ                   │
└──────────────────────────────────────────────────────┘
```

### Triển Khai Code

```python
def update_product(product_id: int, new_data: dict) -> None:
    # Write-Through: Cập nhật cache VÀ database
    
    # Bước 1: Cập nhật cache ngay
    cache_key = f"product:{product_id}"
    redis_client.setex(cache_key, 3600, json.dumps(new_data))
    
    # Bước 2: Cập nhật database
    database.execute(
        "UPDATE products SET name=%s, price=%s WHERE id=%s",
        (new_data["name"], new_data["price"], product_id)
    )
    # Lưu ý: Nếu DB write fail, cache và DB sẽ không đồng bộ
    # Cần transaction hoặc compensation logic (logic bù đắp)

def get_product(product_id: int) -> dict:
    # READ luôn hit cache (data luôn được ghi vào cache khi update)
    cached = redis_client.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)
    
    # Chỉ miss khi key bị expire hoặc chưa từng ghi
    return lazy_load_from_db(product_id)
```

### Ưu & Nhược Điểm

| ✅ Ưu Điểm | ❌ Nhược Điểm |
|-----------|--------------|
| Cache luôn fresh (mới nhất) | **Write penalty** — ghi chậm hơn (phải ghi cả cache) |
| Không có stale reads sau update | **Cache bloat** — cache data chưa từng đọc |
| Read cache hit rate cao | Phức tạp hơn Lazy Loading |
| | Cần xử lý khi ghi cache thành công nhưng DB fail |

---

## 3️⃣ Write-Around — Ghi Vòng Qua Cache

### Cơ Chế Hoạt Động

```
WRITE FLOW:
Application → Database (Bỏ qua Cache hoàn toàn khi ghi)

READ FLOW (lần đầu):
Application → Cache MISS → Database → Cache (lưu vào) → Trả về

READ FLOW (lần tiếp):
Application → Cache HIT → Trả về
```

### Triển Khai Code

```python
def update_product(product_id: int, new_data: dict) -> None:
    # Write-Around: Chỉ ghi vào DB, không đụng cache
    database.execute(
        "UPDATE products SET name=%s, price=%s WHERE id=%s",
        (new_data["name"], new_data["price"], product_id)
    )
    # Cache cũ vẫn tồn tại cho đến khi TTL hết
    # Hoặc chủ động xóa cache để force re-fetch
    redis_client.delete(f"product:{product_id}")

def get_product(product_id: int) -> dict:
    # READ vẫn dùng Lazy Loading
    return lazy_load_from_db(product_id)  # Cache sẽ được điền khi miss
```

### Khi Nào Dùng Write-Around?

```
Phù hợp:
  ✅ Data được ghi nhiều nhưng ít khi đọc lại ngay (ví dụ: log entries)
  ✅ One-time write, read sau vài ngày (batch jobs, reports)
  ✅ Muốn tránh cache đầy với data ít dùng

Không phù hợp:
  ❌ Data thay đổi và cần đọc lại ngay (stale data problem)
  ❌ Real-time dashboards
```

---

## 4️⃣ Write-Behind (Write-Back) — Ghi Trễ

### Cơ Chế Hoạt Động

```
WRITE FLOW (Luồng Ghi):
Application → Cache (Ghi ngay, nhanh) → [Async] → Database (Ghi sau)
                                          Worker
```

### Triển Khai Ý Tưởng

```python
def update_product_write_behind(product_id: int, new_data: dict) -> None:
    # Bước 1: Ghi vào cache ngay → trả về ngay cho user
    cache_key = f"product:{product_id}"
    redis_client.setex(cache_key, 3600, json.dumps(new_data))
    
    # Bước 2: Đẩy vào queue để ghi DB sau
    write_queue = {
        "table": "products",
        "id": product_id,
        "data": new_data,
        "timestamp": time.time()
    }
    redis_client.rpush("db_write_queue", json.dumps(write_queue))

# Worker riêng chạy async để flush vào DB
def db_write_worker():
    while True:
        item = redis_client.lpop("db_write_queue")
        if item:
            data = json.loads(item)
            database.execute(f"UPDATE {data['table']} SET ... WHERE id = {data['id']}")
        else:
            time.sleep(0.01)

# Ưu điểm: Writes cực nhanh (chỉ ghi cache)
# Nhược điểm: Rủi ro mất data nếu cache crash trước khi ghi DB
#              Không dùng với financial transactions (giao dịch tài chính)
```

### ⚠️ Cảnh Báo Quan Trọng

```
Write-Behind thích hợp cho:
  ✅ Metrics / counters (bộ đếm, số liệu)
  ✅ User activity logs
  ✅ Leaderboard scores
  ✅ Non-critical updates

KHÔNG dùng cho:
  ❌ Giao dịch tài chính — tiền có thể mất
  ❌ Order management — đơn hàng có thể mất
  ❌ Bất kỳ data nào cần ACID guarantees
```

---

## 5️⃣ TTL (Time-To-Live — Thời Gian Tồn Tại) Strategy

### Tầm Quan Trọng Của TTL

TTL quyết định bao lâu data cũ tồn tại trong cache trước khi bị làm mới:

```
TTL quá ngắn → Cache miss nhiều → DB chịu tải nhiều → Tốc độ chậm
TTL quá dài  → Stale data      → User thấy thông tin cũ
```

### Hướng Dẫn Đặt TTL Theo Loại Data

```python
TTL_CONFIG = {
    # Ổn định, thay đổi ít
    "config":           86400 * 7,   # 7 ngày — cấu hình hệ thống
    "product_catalog":  3600 * 24,   # 1 ngày  — danh mục sản phẩm
    
    # Thay đổi vừa
    "product_detail":   3600,        # 1 giờ   — chi tiết sản phẩm
    "user_profile":     3600 * 4,    # 4 giờ   — hồ sơ người dùng
    "category_list":    3600 * 6,    # 6 giờ   — danh sách danh mục
    
    # Thay đổi thường xuyên
    "search_results":   300,         # 5 phút  — kết quả tìm kiếm
    "product_stock":    60,          # 1 phút  — tồn kho (inventory)
    "news_feed":        120,         # 2 phút  — bảng tin
    
    # Dựa trên business logic
    "session":          86400,       # 1 ngày  — phiên đăng nhập
    "auth_token":       3600,        # 1 giờ   — token xác thực
    "rate_limit":       60,          # = window size
    
    # Realtime — không nên cache
    "stock_price":      0,           # Không cache — thay đổi liên tục
    "balance":          0,           # Không cache — tài khoản ngân hàng
}
```

### TTL Jitter — Tránh Cache Stampede Khi Nhiều Keys Hết Hạn Cùng Lúc

```python
import random

def set_with_jitter(key: str, value: str, base_ttl: int, jitter_ratio: float = 0.1) -> None:
    """
    Thêm ngẫu nhiên ±10% vào TTL để tránh hàng nghìn keys 
    hết hạn cùng lúc (thundering herd problem)
    """
    jitter = int(base_ttl * jitter_ratio)
    actual_ttl = base_ttl + random.randint(-jitter, jitter)
    redis_client.setex(key, actual_ttl, value)

# Ví dụ: base TTL = 3600s, jitter ±10% = ±360s
# Key A hết hạn: 3240s, Key B: 3780s, Key C: 3560s
# → Không đồng loạt hết hạn → không stampede
```

---

## 6️⃣ Cache Invalidation — Vô Hiệu Hóa Cache

Phil Karlton nói: *"There are only two hard things in Computer Science: cache invalidation and naming things."* (Chỉ có hai thứ khó trong Khoa Học Máy Tính: vô hiệu hóa cache và đặt tên.)

### Các Chiến Lược Invalidation

#### Strategy 1: TTL-Based (Dựa Trên TTL)

```python
# Đơn giản nhất — chờ TTL hết
redis_client.setex("product:1", ttl=3600, value=data)
# Sau 3600 giây, cache tự xóa → request tiếp theo sẽ re-fetch
# Nhược điểm: Có thể serve stale data trong tối đa 3600 giây
```

#### Strategy 2: Event-Driven Invalidation (Vô Hiệu Hóa Dựa Trên Sự Kiện)

```python
# Khi data thay đổi → xóa cache liên quan ngay lập tức
def update_product(product_id: int, data: dict) -> None:
    # 1. Cập nhật DB
    database.execute("UPDATE products SET ... WHERE id = %s", (product_id,))
    
    # 2. Xóa cache liên quan (Pattern-based deletion — Xóa Theo Mẫu)
    keys_to_delete = [
        f"product:{product_id}",           # Chi tiết sản phẩm
        f"product:category:{data['cat']}",  # Cache danh mục
        "homepage:featured_products",       # Cache trang chủ
        f"search:*",                        # Xóa tất cả cache search (cẩn thận!)
    ]
    redis_client.delete(*keys_to_delete)
```

#### Strategy 3: Cache Versioning (Versioning Cache — Đánh Phiên Bản Cache)

```python
# Thay vì xóa cache, thay đổi phiên bản → cache cũ tự trở nên không hợp lệ
def get_product_v2(product_id: int) -> dict:
    # Lấy version hiện tại từ DB hoặc meta-store
    current_version = get_product_version(product_id)  # ví dụ: "v7"
    
    cache_key = f"product:{product_id}:{current_version}"
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)
    
    product = database.query("SELECT * FROM products WHERE id = %s", (product_id,))
    redis_client.setex(cache_key, 3600, json.dumps(product))
    return product

def update_product_v2(product_id: int, data: dict) -> None:
    # Chỉ cần tăng version → cache cũ tự động không dùng nữa
    database.execute("UPDATE products SET ..., version = version + 1 WHERE id = %s", (product_id,))
    # Không cần xóa cache! Keys cũ sẽ expire theo TTL
```

---

## 🔄 Read-Through — Đọc Xuyên

Khác với Lazy Loading: trong Read-Through, **cache layer** (không phải application) tự động fetch từ DB khi miss:

```
READ FLOW:
App → Cache → HIT → Trả về
           → MISS → [Cache tự query DB] → Lưu → Trả về

Khác biệt với Lazy Loading:
  Lazy Loading: App tự handle cache miss logic
  Read-Through: Cache layer tự xử lý (cần cache library hỗ trợ)
```

---

## 🔄 Refresh-Ahead — Làm Mới Trước

```
Cơ chế: Trước khi TTL hết hạn, background process refresh cache
         → Requests không bao giờ thấy cache miss (gần như)

Ví dụ:
  TTL = 3600 giây
  Refresh threshold = 90% TTL = 3240 giây
  
  Tại giây 3240: Background worker phát hiện key sắp hết hạn
                → Query DB → Cập nhật cache với TTL mới
                → Requests tiếp theo vẫn hit cache liên tục

Phù hợp: Hot data thường xuyên được đọc, không chấp nhận được cache miss
```

---

## 🎯 Chọn Chiến Lược Phù Hợp

```
                   Dữ liệu thay đổi bao lâu một lần?
                              │
         ┌────────────────────┼────────────────────┐
    Thường xuyên         Thỉnh thoảng         Rất ít
    (nhiều lần/phút)    (vài lần/ngày)        (hàng tuần)
         │                    │                    │
         ▼                    ▼                    ▼
  Write-Through       Lazy Loading +         Lazy Loading
  (Luôn fresh)       TTL phù hợp          TTL dài (ngày)

                              │
               Cần tốc độ ghi tối đa?
                              │
              ┌───────────────┴──────────────┐
              │ Có, data non-critical        │ Không
              ▼                              ▼
        Write-Behind                 Write-Through
     (nhanh nhưng rủi ro)           hoặc Lazy Loading
```

---

## 📋 Bảng Tổng Hợp

| | Lazy Loading | Write-Through | Write-Around | Write-Behind |
|--|-------------|--------------|--------------|-------------|
| **Tốc độ Đọc** | Chậm lần đầu | Nhanh | Chậm lần đầu | Nhanh |
| **Tốc độ Ghi** | N/A | Chậm hơn | Nhanh | Rất nhanh |
| **Data Freshness** | TTL-dependent | Luôn fresh | TTL-dependent | TTL-dependent |
| **RAM Usage** | Tiết kiệm | Nhiều hơn | Tiết kiệm | Nhiều hơn |
| **Complexity** | Thấp | Trung bình | Thấp | Cao |
| **Risk** | Stale data | Cache bloat | Stale data | Data loss |
| **Use Case** | Read-heavy | Read+Write balance | Write-heavy | Non-critical writes |

---

## 🎓 Câu Hỏi Phỏng Vấn

### Q: "Lazy Loading vs Write-Through — khi nào chọn cái nào?"

**Trả lời mẫu:**
> "Tôi chọn Lazy Loading khi ứng dụng read-heavy và dữ liệu không cần luôn fresh — như product catalog, user profiles. Cache chỉ được điền khi có demand thực sự, tiết kiệm RAM. Nhược điểm là cache miss đầu tiên chậm và stale data window bằng TTL.
>
> Write-Through thích hợp khi stale data không chấp nhận được — ví dụ: inventory (tồn kho) hiển thị cho user. Mỗi write cập nhật cả cache và DB, đảm bảo luôn nhất quán. Trade-off là write chậm hơn và cache có thể đầy với data chưa từng đọc.
>
> Trong thực tế, tôi thường kết hợp: Lazy Loading cho reads với Write-Through cho data quan trọng, và TTL jitter để tránh stampede."

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
