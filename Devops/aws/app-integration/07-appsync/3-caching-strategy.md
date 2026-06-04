# Caching Strategy — Chiến Lược Bộ Nhớ Đệm Trong AppSync

> **Server-Side Caching** (Bộ Nhớ Đệm Phía Máy Chủ) trong AppSync giảm số lần gọi đến data sources, tăng tốc độ response và giảm chi phí — đặc biệt quan trọng với DynamoDB và Lambda có chi phí theo request.

---

## 📚 Mục Lục

1. [Tổng Quan Caching Trong AppSync](#1-tổng-quan-caching-trong-appsync)
2. [Cấp Độ Caching](#2-cấp-độ-caching)
3. [Server-Side Caching Cấu Hình](#3-server-side-caching-cấu-hình)
4. [Per-Resolver Caching](#4-per-resolver-caching)
5. [Cache Keys — Khóa Bộ Nhớ Đệm](#5-cache-keys--khóa-bộ-nhớ-đệm)
6. [Cache Invalidation — Vô Hiệu Hóa Cache](#6-cache-invalidation--vô-hiệu-hóa-cache)
7. [Full Request Caching vs Per-Resolver Caching](#7-full-request-caching-vs-per-resolver-caching)
8. [Client-Side Caching Với Apollo](#8-client-side-caching-với-apollo)
9. [Chiến Lược Tổng Hợp](#9-chiến-lược-tổng-hợp)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Caching Trong AppSync

### Tại Sao Cần Cache?

```
Không có Cache:
Client → AppSync → DynamoDB → AppSync → Client
  10ms      5ms    15–30ms     5ms    = ~35–50ms tổng

Với Cache (Cache Hit — Trúng Cache):
Client → AppSync Cache → Client
  10ms       1–2ms      = ~12ms tổng  (3–4x nhanh hơn)

Lợi ích:
  ✅ Latency thấp hơn — response nhanh hơn
  ✅ Chi phí thấp hơn — ít DynamoDB reads, Lambda invocations
  ✅ Giảm tải cho data sources — quan trọng khi traffic cao
  ✅ Tăng throughput tổng thể của API
```

### AppSync Caching Dựa Trên ElastiCache

Bên dưới, AppSync Server-Side Caching dùng **Amazon ElastiCache for Redis** (bộ nhớ đệm Redis được quản lý). AWS quản lý toàn bộ ElastiCache cluster — không cần tự cài đặt.

```
┌──────────────────────────────────────────────────────────┐
│                    AWS APPSYNC                            │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              ElastiCache for Redis                │   │
│  │                                                  │   │
│  │  Key: hash(query + variables + identity)         │   │
│  │  Value: GraphQL response                         │   │
│  │  TTL: 1 giây đến 3600 giây (1 giờ)              │   │
│  └──────────────────────────────────────────────────┘   │
│         ▲ Cache Hit        │ Cache Miss                  │
│         │ (Trúng cache)    ▼ (Trượt cache)               │
│  ┌──────────────┐   ┌──────────────────────────────┐    │
│  │   Response   │   │  Resolver → Data Source      │    │
│  │  Từ Cache   │   │  (DynamoDB / Lambda / ...)    │    │
│  └──────────────┘   └──────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Cấp Độ Caching

AppSync hỗ trợ **2 cấp độ** caching:

### Cấp Độ 1: Full Request Caching (Bộ Nhớ Đệm Toàn Request)

```
Khi FULL REQUEST CACHING được bật:
→ Cache key = hash(toàn bộ GraphQL request)
  Bao gồm: query string, variables, authorization header
→ Nếu cache hit: KHÔNG chạy resolver nào cả
→ Trả về response cache ngay lập tức

Ưu điểm:
  ✅ Nhanh nhất — hoàn toàn bỏ qua resolver layer
  ✅ Tiết kiệm nhất — không gọi DynamoDB / Lambda

Nhược điểm:
  ❌ Toàn bộ request giống hệt mới hit cache
  ❌ Không phân biệt được "sub-queries" khác nhau
  ❌ Khó cache khi request thường xuyên thay đổi variables
```

### Cấp Độ 2: Per-Resolver Caching (Bộ Nhớ Đệm Mỗi Resolver)

```
Khi PER-RESOLVER CACHING được bật trên từng resolver:
→ Cache key = hash(resolver-specific fields được cấu hình)
  Ví dụ: resolver "getUser" cache theo { id }
→ Cho phép caching độc lập theo từng resolver
→ Resolver khác vẫn chạy bình thường

Ưu điểm:
  ✅ Kiểm soát chi tiết hơn — cache từng resolver riêng
  ✅ Phù hợp với data có lifecycle (vòng đời) khác nhau
  ✅ Một query phức tạp có thể mix: resolver A từ cache, resolver B fresh

Nhược điểm:
  ❌ Cấu hình phức tạp hơn
  ❌ Không cache được toàn bộ response (chỉ từng data source call)
```

---

## 3. Server-Side Caching Cấu Hình

### Bước 1: Chọn Cache Instance Type (Loại Instance Bộ Nhớ Đệm)

```
Các loại instance AppSync Caching hỗ trợ:
(Giá tính theo giờ, vùng us-east-1)

SMALL instances:
  cache.t3.small   — 1.5 GB RAM  — ~$0.034/giờ  — Dev/Testing
  cache.t3.medium  — 3.2 GB RAM  — ~$0.068/giờ  — Small production

MEDIUM instances:
  cache.r6g.large  — 13.07 GB RAM — ~$0.152/giờ  — Medium traffic
  cache.r6g.xlarge — 26.32 GB RAM — ~$0.304/giờ  — High traffic

LARGE instances:
  cache.r6g.2xlarge — 52.82 GB RAM — ~$0.608/giờ — Very high traffic
```

### Bước 2: Chọn Caching Behavior (Hành Vi Bộ Nhớ Đệm)

```
Có 3 lựa chọn:

1. NONE — Tắt caching hoàn toàn (mặc định)

2. FULL_REQUEST_CACHING — Bật Full Request Caching
   → Mọi query đều được cache nếu cùng request

3. PER_RESOLVER_CACHING — Mỗi resolver tự cấu hình cache
   → Instance được cấp phát nhưng chỉ cache resolver được đánh dấu
```

### Bước 3: Cấu Hình Qua AWS Console / Terraform

```hcl
# Terraform — Bật AppSync Caching
resource "aws_appsync_api_cache" "example" {
  api_id = aws_appsync_graphql_api.example.id

  # Loại instance cache
  type = "SMALL"  # SMALL | MEDIUM | LARGE | XLARGE | LARGE_2X | LARGE_4X | LARGE_8X | LARGE_12X

  # Hành vi cache
  api_caching_behavior = "PER_RESOLVER_CACHING"
  # Hoặc: "FULL_REQUEST_CACHING"

  # TTL (Time-To-Live — Thời Gian Sống) mặc định tính bằng giây
  ttl = 300  # 5 phút

  # Bật mã hóa dữ liệu trong cache
  at_rest_encryption_enabled = true
  # Bật mã hóa dữ liệu trong transit (truyền)
  transit_encryption_enabled = true
}
```

```yaml
# AWS CDK (TypeScript)
import * as appsync from 'aws-cdk-lib/aws-appsync';

const api = new appsync.GraphqlApi(this, 'Api', {
  name: 'my-api',
  schema: appsync.SchemaFile.fromAsset('schema.graphql'),
});

// Bật caching
const caching = new appsync.CfnApiCache(this, 'ApiCache', {
  apiId: api.apiId,
  type: 'SMALL',
  apiCachingBehavior: 'PER_RESOLVER_CACHING',
  ttl: 300,
  atRestEncryptionEnabled: true,
  transitEncryptionEnabled: true,
});
```

---

## 4. Per-Resolver Caching

### Bật Cache Trên Resolver Cụ Thể

```hcl
# Terraform — Resolver với caching được bật
resource "aws_appsync_resolver" "get_user" {
  api_id      = aws_appsync_graphql_api.example.id
  type        = "Query"
  field       = "getUser"
  data_source = aws_appsync_datasource.dynamodb.name

  # Bật caching cho resolver này
  caching_config {
    # TTL cho resolver này (override TTL mặc định)
    ttl = 60  # 60 giây

    # Cache keys — xác định khi nào 2 requests share cùng cache entry
    caching_keys = [
      "$context.arguments.id",           # Argument "id"
      "$context.identity.sub",            # User ID từ Cognito
    ]
  }
}
```

### Caching Keys Cho Các Loại Query

```hcl
# Query lấy user theo ID — cache theo ID
resource "aws_appsync_resolver" "get_user" {
  caching_config {
    ttl          = 300
    caching_keys = ["$context.arguments.id"]
  }
}

# Query list với filter — cache theo từng combination
resource "aws_appsync_resolver" "list_orders" {
  caching_config {
    ttl = 60
    caching_keys = [
      "$context.arguments.userId",
      "$context.arguments.status",     # null nếu không truyền
      "$context.arguments.limit",
    ]
  }
}

# Query cần phân biệt theo user — mỗi user có cache riêng
resource "aws_appsync_resolver" "get_my_profile" {
  caching_config {
    ttl          = 600
    caching_keys = ["$context.identity.sub"]  # Cache riêng theo Cognito User ID
  }
}

# Query public — tất cả users share cùng cache entry
resource "aws_appsync_resolver" "list_products" {
  caching_config {
    ttl          = 3600  # 1 giờ — data thay đổi ít
    caching_keys = [
      "$context.arguments.category",
      "$context.arguments.limit",
    ]
  }
}
```

---

## 5. Cache Keys — Khóa Bộ Nhớ Đệm

### Cách AppSync Tạo Cache Key

```
Cache Key = HASH(
  resolver_field_name    +
  caching_keys_values    +
  optional: auth_context
)

Ví dụ với caching_keys = ["$context.arguments.id", "$context.identity.sub"]:

Request A: getUser(id: "user-123"), identity.sub = "cognito-abc"
→ Cache Key Hash = hash("getUser" + "user-123" + "cognito-abc")
→ = "a1b2c3d4..."

Request B: getUser(id: "user-123"), identity.sub = "cognito-xyz"
→ Cache Key Hash = hash("getUser" + "user-123" + "cognito-xyz")
→ = "e5f6g7h8..."   ← Khác! → Cache Miss riêng biệt

Request C: getUser(id: "user-456"), identity.sub = "cognito-abc"
→ Cache Key Hash = hash("getUser" + "user-456" + "cognito-abc")
→ = "i9j0k1l2..."   ← Khác! → Cache Miss riêng biệt
```

### Các Context Variables Hữu Ích Cho Cache Keys

```
$context.arguments.<argName>        — Argument của query/mutation
$context.identity.sub               — Cognito User ID (UUID)
$context.identity.username          — Cognito username
$context.identity.claims.<key>      — Custom claims từ Cognito
$context.request.headers.<header>   — HTTP request headers
$context.source.<fieldName>         — Parent object field (cho nested resolvers)
```

### Ví Dụ: Cache Keys Cho Nested Resolver

```hcl
# Schema:
# type User {
#   id: ID!
#   orders: [Order!]  ← Nested resolver
# }

# Resolver cho User.orders — cache theo userId (= parent.id)
resource "aws_appsync_resolver" "user_orders" {
  type  = "User"
  field = "orders"

  caching_config {
    ttl = 120
    caching_keys = [
      "$context.source.id",   # ID của User object (parent)
    ]
  }
}
```

---

## 6. Cache Invalidation — Vô Hiệu Hóa Cache

### Vấn Đề: Cache Staleness (Cache Cũ)

```
Scenario (Tình Huống):
  T=0s: Client A query getUser(id:"123") → DB trả về {name: "Nguyễn A"}
         → Cached: key="getUser+123", value="{name: Nguyễn A}", TTL=300s

  T=30s: User đổi tên → Mutation updateUser(id:"123", name: "Nguyễn B")
         → DB được cập nhật thành công

  T=60s: Client B query getUser(id:"123")
         → Cache HIT! Trả về {name: "Nguyễn A"} ← DỮ LIỆU CŨ!

  T=300s: Cache expires → Client C query → Cache MISS → DB trả về {name: "Nguyễn B"}
```

### Chiến Lược 1: TTL-based Invalidation (Đơn Giản)

```
→ Đặt TTL phù hợp với "freshness requirement" (yêu cầu độ tươi mới)

Data thay đổi thường:
  user profile, order status → TTL: 30–60 giây

Data thay đổi ít:
  product catalog, config → TTL: 1–24 giờ

Data gần như tĩnh:
  category list, country list → TTL: 24 giờ+

Trade-off:
  TTL thấp → Data fresh hơn, nhưng cache hit rate thấp hơn
  TTL cao → Cache hit rate cao hơn, nhưng data có thể stale (cũ)
```

### Chiến Lược 2: Mutation-triggered Invalidation (Chủ Động)

AppSync không có built-in cache invalidation khi mutation xảy ra. Cần implement thủ công:

```javascript
// Resolver mutation "updateUser" — sau khi update, flush (xóa) cache
// Dùng Pipeline Resolver:
//   Function 1: Update DynamoDB
//   Function 2: Gọi Lambda để invalidate cache entry

// Lambda function — xóa cache key cụ thể qua ElastiCache
import { createClient } from 'redis';

const redis = createClient({
  url: process.env.ELASTICACHE_URL,
});

export async function handler(event) {
  const { userId } = event.arguments;

  // Pattern key để tìm và xóa
  // AppSync tạo key theo pattern: appsync:{apiId}:{resolverHash}
  // Cần biết pattern key để xóa đúng

  // Cách 1: Xóa theo pattern (cẩn thận với SCAN trên production)
  const pattern = `appsync:*getUser*${userId}*`;
  const keys = await redis.keys(pattern);

  if (keys.length > 0) {
    await redis.del(keys);
  }

  return { invalidated: keys.length };
}
```

### Chiến Lược 3: Write-Through Caching (Ghi Xuyên Cache)

```
Thay vì cache trực tiếp từ AppSync, implement cache layer trong Lambda:

Client → AppSync → Lambda → Cache (Redis) → DynamoDB

Khi Read (Đọc):
  1. Lambda kiểm tra Redis cache trước
  2. Cache HIT → trả về từ Redis
  3. Cache MISS → đọc DynamoDB, lưu vào Redis, trả về

Khi Write (Ghi):
  1. Lambda ghi vào DynamoDB
  2. Xóa cache entry liên quan (cache invalidation)
  3. Hoặc: ghi luôn vào Redis (write-through)

Ưu điểm:
  ✅ Kiểm soát hoàn toàn invalidation logic
  ✅ Không bị giới hạn bởi AppSync cache API

Nhược điểm:
  ❌ Phức tạp hơn
  ❌ Thêm Lambda invocation overhead
  ❌ Chi phí quản lý ElastiCache riêng
```

### Chiến Lược 4: Stale-While-Revalidate

```
Khái niệm:
  - Trả về cached data ngay (dù có thể stale)
  - Đồng thời trigger background refresh
  - Lần tiếp theo → data mới nhất

Trong context AppSync:
  - TTL ngắn để "stale window" nhỏ
  - Dùng AppSync caching cho reads
  - Mutations invalidate cache ngay lập tức (qua Lambda)
  - Acceptable cho data không cần real-time perfect accuracy
```

---

## 7. Full Request Caching vs Per-Resolver Caching

### So Sánh Chi Tiết

| Tiêu Chí | Full Request Caching | Per-Resolver Caching |
|---|---|---|
| **Phạm vi** | Toàn bộ GraphQL response | Từng resolver riêng lẻ |
| **Cache key** | Hash(query + variables + auth) | Hash(resolver-specific keys) |
| **Tốc độ khi hit** | Nhanh nhất — bỏ qua toàn bộ resolver layer | Nhanh — bỏ qua data source call |
| **Độ linh hoạt** | Thấp — tất cả hoặc không gì cả | Cao — cấu hình từng resolver |
| **Phù hợp với** | Queries ít thay đổi, public data | Queries phức tạp với nhiều resolvers |
| **Mutation caching** | Không — mutations không được cache | Không — mutations không được cache |
| **Setup** | Đơn giản | Phức tạp hơn |

### Khi Nào Dùng Full Request Caching?

```
✅ API gần như read-only (ít mutations)
✅ Queries ít variables, public (không phụ thuộc user context)
✅ Data update theo batch (cập nhật hàng loạt) — không real-time
✅ Ví dụ: Product catalog API, public blog API, configuration API
```

### Khi Nào Dùng Per-Resolver Caching?

```
✅ API mix read/write (có cả query và mutation thường xuyên)
✅ Một số fields cần fresh data, một số có thể cache
✅ Cache granularity (độ chi tiết) quan trọng theo user context
✅ Ví dụ: E-commerce API (cache products, không cache cart)
```

---

## 8. Client-Side Caching Với Apollo

AppSync tương thích với **Apollo Client** — framework GraphQL phổ biến nhất — có built-in InMemoryCache (Bộ Nhớ Đệm Trong Bộ Nhớ).

### Thiết Lập Apollo Client Với AppSync

```typescript
import {
  ApolloClient,
  InMemoryCache,
  createHttpLink,
  ApolloProvider,
} from '@apollo/client';
import { createAuthLink } from 'aws-appsync-auth-link';
import { createSubscriptionHandshakeLink } from 'aws-appsync-subscription-link';
import { ApolloLink } from '@apollo/client';

const url = 'https://<appsync-id>.appsync-api.<region>.amazonaws.com/graphql';
const region = 'ap-southeast-1';
const auth = {
  type: 'AMAZON_COGNITO_USER_POOLS',
  jwtToken: async () => {
    const session = await Auth.currentSession();
    return session.getAccessToken().getJwtToken();
  },
};

// Tạo HTTP link với AppSync auth
const httpLink = createAuthLink({ url, region, auth });

// Tạo subscription link cho WebSocket
const wsLink = createSubscriptionHandshakeLink({ url, region, auth }, httpLink);

const client = new ApolloClient({
  link: ApolloLink.from([wsLink]),
  cache: new InMemoryCache({
    // Cấu hình cache policies (chính sách cache) theo type
    typePolicies: {
      User: {
        keyFields: ['id'],  // Cache key theo User.id
      },
      Order: {
        keyFields: ['id'],
      },
      Query: {
        fields: {
          // listOrders — pagination với merge policy (chính sách gộp)
          listOrders: {
            keyArgs: ['userId', 'status'],  // Phân biệt cache theo args này
            merge(existing = { items: [] }, incoming) {
              return {
                ...incoming,
                items: [...existing.items, ...incoming.items],
              };
            },
          },
        },
      },
    },
  }),
});
```

### Fetch Policies (Chính Sách Lấy Dữ Liệu) Trong Apollo

```typescript
// cache-first (Ưu tiên cache — mặc định)
// → Dùng cache nếu có, không fetch từ network
// Tốt cho: Data hiếm khi thay đổi
const { data } = useQuery(GET_USER, {
  variables: { id: userId },
  fetchPolicy: 'cache-first',
});

// network-only (Chỉ dùng network)
// → Luôn fetch từ server, không dùng cache
// Tốt cho: Data cần luôn fresh (tươi)
const { data } = useQuery(GET_ORDER, {
  variables: { id: orderId },
  fetchPolicy: 'network-only',
});

// cache-and-network (Cache + Network)
// → Trả về cache ngay (nếu có), đồng thời fetch từ network để update
// Tốt cho: UX tốt nhất — hiển thị ngay, rồi update khi có data mới
const { data, loading } = useQuery(LIST_ORDERS, {
  variables: { userId },
  fetchPolicy: 'cache-and-network',
});

// no-cache (Không dùng cache)
// → Không đọc cache, không ghi vào cache
// Tốt cho: Sensitive data (dữ liệu nhạy cảm)
const { data } = useQuery(GET_PAYMENT_INFO, {
  fetchPolicy: 'no-cache',
});

// cache-only (Chỉ cache)
// → Chỉ đọc từ cache, lỗi nếu không có
// Tốt cho: Offline mode (chế độ ngoại tuyến)
const { data } = useQuery(GET_CACHED_USER, {
  fetchPolicy: 'cache-only',
});
```

### Cập Nhật Cache Sau Mutation

```typescript
// Sau mutation createOrder, cập nhật Apollo cache tự động
const [createOrder] = useMutation(CREATE_ORDER, {
  // update() — Hàm cập nhật Apollo cache thủ công
  update(cache, { data: { createOrder } }) {
    // Đọc query hiện tại từ cache
    const existing = cache.readQuery({
      query: LIST_ORDERS,
      variables: { userId: currentUserId },
    });

    if (existing) {
      // Ghi lại query với order mới được thêm vào đầu danh sách
      cache.writeQuery({
        query: LIST_ORDERS,
        variables: { userId: currentUserId },
        data: {
          listOrders: {
            ...existing.listOrders,
            items: [createOrder, ...existing.listOrders.items],
          },
        },
      });
    }
  },

  // optimisticResponse — Cập nhật UI ngay lập tức trước khi server trả lời
  optimisticResponse: {
    createOrder: {
      __typename: 'Order',
      id: `temp-${Date.now()}`,  // Temporary ID
      status: 'PENDING',
      total: orderTotal,
      createdAt: new Date().toISOString(),
      ...otherFields,
    },
  },
});
```

---

## 9. Chiến Lược Tổng Hợp

### Quyết Định Cache Cho Từng Loại Data

```
┌─────────────────────────────────────────────────────────────┐
│                   CACHE DECISION MATRIX                      │
│                  (Ma Trận Quyết Định Cache)                  │
│                                                              │
│  Tần suất thay đổi DATA:                                     │
│                                                              │
│  Cao (mỗi phút)        Trung bình (mỗi giờ)  Thấp (mỗi ngày)│
│         │                     │                    │         │
│         ▼                     ▼                    ▼         │
│    Không cache          TTL = 60–300s         TTL = 1–24h   │
│    hoặc TTL < 30s                                            │
│                                                              │
│  Ví dụ:                                                      │
│  - Order status         - Product inventory  - Product list  │
│  - Chat messages        - User profile       - Category      │
│  - Live prices          - Search results     - Config        │
└─────────────────────────────────────────────────────────────┘
```

### Kiến Trúc Caching Đa Tầng

```
Tầng 1: CDN Cache (CloudFront) — cho public queries
  → TTL: Minutes to hours
  → Cache key: URL + headers
  → Không phù hợp với authenticated queries

Tầng 2: AppSync Server-Side Cache (ElastiCache)
  → TTL: Seconds to minutes
  → Cache key: Query + args + user context
  → Phù hợp với authenticated queries

Tầng 3: Apollo InMemoryCache (Client-side)
  → TTL: Managed by Apollo (no expiry by default)
  → Cache key: Type + id fields
  → Tốt cho UX — offline, instant updates

Tầng 4: React state / Local storage
  → Cho persisted data khi offline
  → Amplify DataStore handles this automatically
```

### Checklist Trước Khi Bật Cache

```
Trước khi bật AppSync Server-Side Caching:

□ Xác định TTL phù hợp cho từng resolver
□ Quyết định cache keys đủ specific để tránh false cache hits
□ Đảm bảo mutations có strategy invalidate cache
□ Kiểm tra sensitive data KHÔNG được cache
  (financial, medical, personal info)
□ Tính toán chi phí ElastiCache instance vs savings từ ít reads
□ Test với load để verify cache hit rate đủ cao (> 50%)
□ Bật encryption at-rest và in-transit cho compliance
□ Setup CloudWatch metrics theo dõi cache hit/miss rate
```

### CloudWatch Metrics Quan Trọng Cho Cache

```
1. CacheHits — Số lần cache hit
   → Mục tiêu: Càng cao càng tốt (> 60% là tốt)

2. CacheMisses — Số lần cache miss
   → Mục tiêu: Càng thấp càng tốt

3. Hit Rate = CacheHits / (CacheHits + CacheMisses)
   → < 30%: TTL quá ngắn hoặc cache keys quá specific
   → > 90%: Tốt, nhưng kiểm tra data freshness

4. ElastiCache FreeableMemory — Bộ nhớ còn trống
   → < 10% → Nên nâng cấp instance size

5. ElastiCache Evictions — Số entries bị xóa sớm do hết RAM
   → > 0 liên tục → Instance size cần tăng
```

---

## 10. Câu Hỏi Phỏng Vấn

### Q1: AppSync Server-Side Caching hoạt động như thế nào?

**Trả lời mẫu:**

AppSync Server-Side Caching dùng **Amazon ElastiCache for Redis** được quản lý bởi AWS. Khi resolver được cấu hình với caching:

1. AppSync tạo **cache key** bằng cách hash các fields được cấu hình (arguments, identity, v.v.)
2. Trước khi gọi data source, AppSync **kiểm tra cache** — nếu có (cache hit) → trả về ngay
3. Nếu không có (cache miss) → gọi data source bình thường → **lưu kết quả vào cache** với TTL
4. Sau TTL → entry bị xóa, request tiếp theo sẽ là cache miss

Có hai chế độ: **Full Request Caching** (cache toàn bộ response) và **Per-Resolver Caching** (cache từng resolver độc lập).

---

### Q2: Làm sao invalidate cache khi data thay đổi?

**Trả lời mẫu:**

AppSync không có built-in cache invalidation khi mutation xảy ra. Các chiến lược tôi đã dùng:

**1. TTL-based:** Đặt TTL phù hợp với mức độ "staleness acceptable" — đơn giản nhất, phù hợp khi data không cần instant consistency.

**2. Mutation-triggered invalidation:** Dùng Pipeline Resolver — sau mutation chính, chạy thêm function gọi Lambda để xóa Redis keys liên quan. Phức tạp hơn nhưng consistent hơn.

**3. Write-through trong Lambda:** Thay vì dùng AppSync cache, implement cache layer trong Lambda resolver — kiểm soát hoàn toàn khi nào invalidate.

Trade-off: TTL đơn giản nhưng có "stale window"; mutation-triggered phức tạp nhưng consistent hơn.

---

### Q3: Khi nào không nên cache trong AppSync?

**Trả lời mẫu:**

Không nên cache khi:

1. **Dữ liệu nhạy cảm** (tài chính, y tế, thông tin cá nhân) — dù có encryption, tránh cache để giảm attack surface
2. **Dữ liệu thay đổi rất thường xuyên** (mỗi giây) — cache hit rate sẽ thấp, chi phí ElastiCache không đáng
3. **Mutations** — AppSync không cache mutations (đúng đắn — ghi dữ liệu phải luôn gọi data source)
4. **Dữ liệu theo real-time** như chat messages — cần fresh data ngay lập tức
5. **Khi không đủ traffic** để justify chi phí ElastiCache instance — với API ít request, cost savings không bù được instance cost

---

### Q4: Apollo InMemoryCache khác AppSync Server-Side Cache như thế nào?

**Trả lời mẫu:**

| | Apollo InMemoryCache | AppSync Server-Side Cache |
|---|---|---|
| **Vị trí** | Client browser/mobile | AWS server |
| **Scope** | Chỉ client đó | Tất cả clients |
| **Persistence** | Mất khi refresh trang | Tồn tại theo TTL |
| **Mục đích** | Tránh re-fetch data đã biết | Giảm tải data sources |
| **Security** | Không chia sẻ giữa users | Cần cẩn thận với cache keys |

Thực tế, cả hai thường được dùng cùng nhau: Server-side cache giảm tải cho DynamoDB/Lambda, client-side cache cải thiện UX và hỗ trợ offline.

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
**Module:** 07-appsync | App Integration Series
