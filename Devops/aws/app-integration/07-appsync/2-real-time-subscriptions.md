# Real-time Subscriptions — Đăng Ký Thời Gian Thực Qua WebSocket

> **Subscription** (Đăng Ký) trong AppSync cho phép client lắng nghe sự kiện real-time qua kết nối **WebSocket** được quản lý hoàn toàn bởi AWS — không cần tự xây dựng WebSocket server.

---

## 📚 Mục Lục

1. [Cơ Chế Hoạt Động](#1-cơ-chế-hoạt-động)
2. [Subscription Connection Lifecycle](#2-subscription-connection-lifecycle)
3. [Định Nghĩa Subscription Trong Schema](#3-định-nghĩa-subscription-trong-schema)
4. [Subscription Filtering — Lọc Subscription](#4-subscription-filtering--lọc-subscription)
5. [Enhanced Subscription Filtering](#5-enhanced-subscription-filtering)
6. [Conflict Resolution & Ordering](#6-conflict-resolution--ordering)
7. [Triển Khai Thực Tế — Code Examples](#7-triển-khai-thực-tế--code-examples)
8. [Kiến Trúc Fan-out Với Subscription](#8-kiến-trúc-fan-out-với-subscription)
9. [Giới Hạn & Best Practices](#9-giới-hạn--best-practices)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Cơ Chế Hoạt Động

### Mô Hình Pub/Sub Trong AppSync

AppSync Subscriptions hoạt động theo mô hình **Pub/Sub** (Publisher/Subscriber — Nhà Phát/Người Đăng Ký):

```
CLIENT (Subscriber)          AWS APPSYNC             BACKEND
      │                           │                      │
      │── WebSocket Connect ──►   │                      │
      │◄── Connection ACK ──────  │                      │
      │                           │                      │
      │── Subscribe to            │                      │
      │   onOrderStatusChanged ─► │                      │
      │◄── Subscription ACK ───   │                      │
      │                           │                      │
      │                           │  ◄── Mutation ───────│
      │                           │     updateOrder(...)  │
      │                           │                      │
      │   (AppSync evaluates      │                      │
      │    subscription filter)   │                      │
      │                           │                      │
      │◄── Subscription Event ─── │                      │
      │   { orderId, newStatus }  │                      │
      │                           │                      │
      │── Unsubscribe ──────────► │                      │
      │── WebSocket Disconnect ─► │                      │
```

### Luồng Kỹ Thuật Chi Tiết

```
1. CLIENT gửi HTTP POST đến AppSync để lấy WebSocket URL
   → POST https://<appsync-id>.appsync-api.<region>.amazonaws.com/graphql
   → Header: Authorization token
   → Response: { wssUrl, realtimeUrl }

2. CLIENT mở WebSocket tới wssUrl
   → Giao thức: MQTT over WebSocket hoặc graphql-ws protocol
   → AppSync trả về connection_ack

3. CLIENT gửi subscription registration message
   → Chứa GraphQL subscription query
   → Chứa variables (biến số) như orderId filter

4. MUTATION được thực thi (bởi bất kỳ client nào)
   → AppSync resolver thực thi mutation
   → AppSync kiểm tra: có subscription nào liên quan không?

5. AppSync EVALUATE subscription filters
   → So sánh mutation result với từng subscription
   → Chỉ gửi đến subscribers phù hợp

6. AppSync PUSH event đến subscribers qua WebSocket

7. CLIENT nhận event, cập nhật UI
```

### MQTT over WebSocket vs graphql-ws

| | MQTT over WebSocket | graphql-ws Protocol |
|---|---|---|
| **Giao thức** | MQTT (Message Queue Telemetry Transport) | graphql-ws (chuẩn mới hơn) |
| **Amplify** | Dùng mặc định | Được hỗ trợ từ Amplify v6 |
| **IoT devices** | Phổ biến | Ít phổ biến hơn |
| **Web apps** | Hoạt động tốt | Hoạt động tốt |

---

## 2. Subscription Connection Lifecycle

### Vòng Đời Kết Nối Đầy Đủ

```
┌─────────────────────────────────────────────────────────┐
│              SUBSCRIPTION CONNECTION LIFECYCLE           │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │DISCONNECT│  │CONNECTING│  │CONNECTED │  │SUBSCRI-│ │
│  │          │  │          │  │          │  │BED     │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
│       ▲              │              │             │     │
│       │         Auth fail      Connection       Sub    │
│       │         Network err    established      success│
│       │              │              │             │     │
│       └──────────────┴──────────────┴─────────────┘    │
│                                                         │
│  Sự kiện:                                               │
│  onConnect(provider, event) — Khi kết nối thành công   │
│  onDisconnect(provider, event) — Khi mất kết nối       │
│  onSubscriptionError(error) — Khi subscription lỗi     │
└─────────────────────────────────────────────────────────┘
```

### Timeout & Keep-alive (Giữ Kết Nối)

```
- Max connection time: 2 giờ (7,200 giây)
  → Sau 2 giờ, AppSync đóng connection
  → Client cần reconnect (kết nối lại)

- Keep-alive interval: AppSync gửi ping mỗi ~5 phút
  → Client phải respond để duy trì connection

- Idle timeout: Connection đóng sau ~5 phút không hoạt động
  → Amplify tự động handle reconnect

- Subscription limit per connection: 100 subscriptions
  → Không nên subscribe quá nhiều topic từ một connection
```

### Reconnection Strategy (Chiến Lược Kết Nối Lại)

```javascript
// Amplify tự động xử lý reconnect với exponential backoff
// (tăng dần thời gian chờ theo lũy thừa)

// Thủ công nếu cần custom logic:
let retryCount = 0;
const MAX_RETRIES = 5;
const BASE_DELAY_MS = 1000;  // 1 giây

async function connectWithRetry() {
  while (retryCount < MAX_RETRIES) {
    try {
      await setupSubscription();
      retryCount = 0;  // Reset khi thành công
      break;
    } catch (error) {
      retryCount++;
      const delay = BASE_DELAY_MS * Math.pow(2, retryCount);  // 2s, 4s, 8s...
      console.error(`Kết nối thất bại, thử lại sau ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

---

## 3. Định Nghĩa Subscription Trong Schema

### Cú Pháp Cơ Bản

```graphql
type Subscription {
  # Lắng nghe khi bất kỳ order nào được tạo
  onCreateOrder: Order
    @aws_subscribe(mutations: ["createOrder"])

  # Lắng nghe khi một order cụ thể được cập nhật
  # Argument "orderId" dùng để filter (lọc)
  onOrderUpdated(orderId: ID): Order
    @aws_subscribe(mutations: ["updateOrder", "updateOrderStatus"])

  # Lắng nghe nhiều mutations cùng lúc
  onOrderChanged(userId: ID): Order
    @aws_subscribe(mutations: ["createOrder", "updateOrder", "deleteOrder"])
}
```

### Directive @aws_subscribe

```graphql
# @aws_subscribe(mutations: [...]) — Khai báo mutation nào trigger subscription này

# Có thể liệt kê nhiều mutations:
onAnyChange: SomeType
  @aws_subscribe(mutations: ["createFoo", "updateFoo", "deleteFoo"])

# Argument trong subscription = filter key
# Client subscribe với orderId="abc123" chỉ nhận event của order đó
onOrderUpdated(orderId: ID): Order
  @aws_subscribe(mutations: ["updateOrder"])
```

### Schema Đầy Đủ — Ví Dụ Order System

```graphql
type Order {
  id: ID!
  userId: ID!
  status: OrderStatus!
  items: [OrderItem!]!
  total: Float!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

type Mutation {
  createOrder(input: CreateOrderInput!): Order!
  updateOrderStatus(orderId: ID!, status: OrderStatus!): Order!
  cancelOrder(orderId: ID!): Order!
}

type Subscription {
  # 1. Seller lắng nghe orders mới trong shop của họ
  onNewOrderForSeller(sellerId: ID!): Order
    @aws_subscribe(mutations: ["createOrder"])

  # 2. Customer theo dõi đơn hàng của mình
  onMyOrderStatusChanged(userId: ID!): Order
    @aws_subscribe(mutations: ["updateOrderStatus", "cancelOrder"])

  # 3. Admin xem tất cả thay đổi (không có filter)
  onAnyOrderChanged: Order
    @aws_subscribe(mutations: ["createOrder", "updateOrderStatus", "cancelOrder"])
}
```

---

## 4. Subscription Filtering — Lọc Subscription

### Cơ Chế Filtering Mặc Định (Argument-based)

AppSync mặc định filter subscription dựa trên **arguments** (đối số) mà client truyền vào khi subscribe:

```graphql
# Schema
type Subscription {
  onOrderUpdated(orderId: ID, userId: ID): Order
    @aws_subscribe(mutations: ["updateOrder"])
}
```

```javascript
// Client A subscribe với orderId="order-123"
// → Chỉ nhận events liên quan đến order-123

// Client B subscribe với userId="user-456"
// → Chỉ nhận events liên quan đến user-456

// Client C subscribe không có filter
// → Nhận TẤT CẢ updateOrder events

// AppSync so sánh:
// subscription.orderId == mutation_result.orderId ?
// subscription.userId == mutation_result.userId ?
// Nếu đúng → gửi event đến client đó
```

### Vấn Đề Với Argument-based Filtering

```
Chỉ hỗ trợ equality check (so sánh bằng) đơn giản:
  orderId == "abc123"  ✅ Hỗ trợ
  
Không hỗ trợ complex expressions:
  status IN ["PENDING", "CONFIRMED"]  ❌ Không hỗ trợ
  total > 100  ❌ Không hỗ trợ
  tags CONTAINS "urgent"  ❌ Không hỗ trợ

→ Giải pháp: Enhanced Subscription Filtering (phần tiếp theo)
```

---

## 5. Enhanced Subscription Filtering

**Enhanced Subscription Filtering** (Lọc Đăng Ký Nâng Cao) cho phép viết **filter expressions phức tạp** trong subscription resolver.

### Kích Hoạt Enhanced Filtering

Enhanced filtering được cấu hình trong **Subscription Resolver** — thường dùng resolver kiểu "None" data source.

### Ví Dụ: Filter Đơn Giản

```javascript
// Subscription Resolver — Request Template
// (Dùng None data source)

export function request(ctx) {
  // ctx.args chứa subscription arguments từ client
  const { userId } = ctx.args;

  // Trả về filter expression
  extensions.setSubscriptionFilter(
    util.transform.toSubscriptionFilter({
      // Chỉ gửi event nếu order.userId == subscription.userId
      userId: { eq: userId },
      // Và status không phải CANCELLED
      status: { ne: 'CANCELLED' },
    })
  );

  return null;
}

export function response(ctx) {
  return null;
}
```

### Ví Dụ: Filter Phức Tạp

```javascript
export function request(ctx) {
  const { userId, minAmount, statusList } = ctx.args;

  extensions.setSubscriptionFilter(
    util.transform.toSubscriptionFilter({
      // Điều kiện AND (tất cả phải đúng)
      and: [
        { userId: { eq: userId } },
        {
          // Điều kiện OR (ít nhất một đúng)
          or: [
            { status: { eq: 'CONFIRMED' } },
            { status: { eq: 'SHIPPED' } },
            { status: { eq: 'DELIVERED' } },
          ],
        },
        // Nếu truyền minAmount → lọc theo số tiền
        ...(minAmount ? [{ total: { ge: minAmount } }] : []),
      ],
    })
  );

  return null;
}
```

### Các Toán Tử Filtering Hỗ Trợ

| Toán Tử | Ý Nghĩa | Ví Dụ |
|---|---|---|
| `eq` | Bằng (equal) | `{ status: { eq: "ACTIVE" } }` |
| `ne` | Không bằng (not equal) | `{ status: { ne: "DELETED" } }` |
| `le` | Nhỏ hơn hoặc bằng (less or equal) | `{ price: { le: 100 } }` |
| `lt` | Nhỏ hơn (less than) | `{ priority: { lt: 5 } }` |
| `ge` | Lớn hơn hoặc bằng (greater or equal) | `{ total: { ge: 50 } }` |
| `gt` | Lớn hơn (greater than) | `{ rating: { gt: 3 } }` |
| `contains` | Chứa chuỗi con | `{ name: { contains: "Prime" } }` |
| `notContains` | Không chứa | `{ tags: { notContains: "spam" } }` |
| `beginsWith` | Bắt đầu bằng | `{ id: { beginsWith: "ORD-" } }` |
| `in` | Trong danh sách | `{ status: { in: ["A","B"] } }` |
| `notIn` | Không trong danh sách | `{ region: { notIn: ["us-east-1"] } }` |
| `between` | Trong khoảng | `{ age: { between: [18, 65] } }` |

### Lambda Authorizer Kết Hợp Với Subscription Filtering

```javascript
// Lambda Authorizer có thể truyền thêm context cho resolver
// resolver có thể dùng context đó để filter an toàn hơn

export function request(ctx) {
  // ctx.identity.resolverContext chứa thông tin từ Lambda Authorizer
  const { userId, allowedSellerIds } = ctx.identity.resolverContext;

  extensions.setSubscriptionFilter(
    util.transform.toSubscriptionFilter({
      or: [
        // User chỉ thấy orders của chính mình
        { customerId: { eq: userId } },
        // Seller chỉ thấy orders trong shop của họ
        { sellerId: { in: allowedSellerIds } },
      ],
    })
  );

  return null;
}
```

---

## 6. Conflict Resolution & Ordering

### Thứ Tự Giao Vận (Delivery Ordering)

```
AppSync KHÔNG đảm bảo thứ tự giao vận subscription events:
- Event 1 (mutation lúc 10:00:00) có thể đến AFTER Event 2 (mutation lúc 10:00:01)
- Network latency, connection state khác nhau giữa các clients

Giải pháp:
1. Thêm "version" hoặc "updatedAt" vào event payload
2. Client tự sort events theo timestamp/version
3. Dùng "optimistic UI" (cập nhật UI ngay, điều chỉnh khi nhận event)
```

### Idempotency (Tính Bất Biến) Trong Subscription Handler

```javascript
// Client-side: tránh xử lý event trùng lặp
const processedEventIds = new Set();

function handleSubscriptionEvent(event) {
  const eventId = event.id || `${event.orderId}-${event.updatedAt}`;

  if (processedEventIds.has(eventId)) {
    console.log('Event đã xử lý, bỏ qua:', eventId);
    return;
  }

  processedEventIds.add(eventId);

  // Xử lý event...
  updateOrderInUI(event);
}
```

---

## 7. Triển Khai Thực Tế — Code Examples

### React + Amplify v6 — Subscription Cơ Bản

```typescript
import { generateClient } from 'aws-amplify/api';
import { useEffect, useState } from 'react';

const client = generateClient();

// GraphQL subscription document
const ON_ORDER_STATUS_CHANGED = `
  subscription OnOrderStatusChanged($orderId: ID!) {
    onOrderStatusChanged(orderId: $orderId) {
      id
      status
      updatedAt
    }
  }
`;

function OrderTracker({ orderId }: { orderId: string }) {
  const [orderStatus, setOrderStatus] = useState<string>('PENDING');
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // Khởi tạo subscription
    const subscription = client
      .graphql({
        query: ON_ORDER_STATUS_CHANGED,
        variables: { orderId },
      })
      .subscribe({
        next: ({ data }) => {
          const newStatus = data.onOrderStatusChanged?.status;
          if (newStatus) {
            setOrderStatus(newStatus);
          }
        },
        error: (err) => {
          console.error('Subscription error:', err);
          setError('Mất kết nối real-time. Đang kết nối lại...');
        },
      });

    // Cleanup — hủy subscription khi component unmount
    return () => {
      subscription.unsubscribe();
    };
  }, [orderId]);  // Re-subscribe khi orderId thay đổi

  return (
    <div>
      {error && <p style={{ color: 'red' }}>{error}</p>}
      <p>Trạng thái đơn hàng: <strong>{orderStatus}</strong></p>
    </div>
  );
}
```

### React — Quản Lý Nhiều Subscriptions

```typescript
import { generateClient } from 'aws-amplify/api';
import { useEffect, useRef } from 'react';

const client = generateClient();

function OrderDashboard({ userId }: { userId: string }) {
  // Dùng ref để lưu trữ subscriptions (không trigger re-render)
  const subscriptionsRef = useRef<any[]>([]);

  useEffect(() => {
    // Subscribe đến nhiều events cùng lúc
    const subs = [
      // 1. Lắng nghe orders mới
      client.graphql({
        query: ON_NEW_ORDER,
        variables: { userId },
      }).subscribe({
        next: ({ data }) => handleNewOrder(data.onNewOrder),
        error: console.error,
      }),

      // 2. Lắng nghe thay đổi status
      client.graphql({
        query: ON_ORDER_STATUS_CHANGED,
        variables: { userId },
      }).subscribe({
        next: ({ data }) => handleStatusChange(data.onOrderStatusChanged),
        error: console.error,
      }),
    ];

    subscriptionsRef.current = subs;

    // Cleanup tất cả subscriptions
    return () => {
      subscriptionsRef.current.forEach(sub => sub.unsubscribe());
      subscriptionsRef.current = [];
    };
  }, [userId]);

  return <div>Dashboard</div>;
}
```

### iOS — Swift + Amplify

```swift
import Amplify

class OrderTracker {
    private var subscription: AmplifyAsyncThrowingSequence<GraphQLSubscriptionEvent<Order>>?

    func startTracking(orderId: String) async {
        subscription = Amplify.API.subscribe(
            request: .init(document: onOrderStatusChanged, variables: ["orderId": orderId])
        )

        do {
            for try await subscriptionEvent in subscription! {
                switch subscriptionEvent {
                case .connection(let connectionState):
                    print("Trạng thái kết nối: \(connectionState)")

                case .data(let result):
                    switch result {
                    case .success(let order):
                        await MainActor.run {
                            updateUI(with: order.status)
                        }
                    case .failure(let error):
                        print("Lỗi dữ liệu: \(error)")
                    }
                }
            }
        } catch {
            print("Lỗi subscription: \(error)")
        }
    }

    func stopTracking() {
        subscription?.cancel()
    }
}
```

### Trigger Subscription Từ Lambda (Không Qua AppSync Mutation)

```python
import boto3
import json

# Trường hợp: Backend process cập nhật order, cần notify clients
# qua AppSync subscription — nhưng không thông qua GraphQL mutation

appsync_client = boto3.client('appsync')

def notify_order_updated(order_id: str, new_status: str, user_id: str):
    """
    Gọi AppSync mutation từ Lambda để trigger subscriptions.
    Lambda cần có IAM permission: appsync:GraphQL
    """
    mutation = """
        mutation UpdateOrderStatus($input: UpdateOrderStatusInput!) {
            updateOrderStatus(input: $input) {
                id
                status
                updatedAt
            }
        }
    """

    response = appsync_client.graphql(
        apiId='YOUR_APPSYNC_API_ID',
        query=mutation,
        variables=json.dumps({
            'input': {
                'orderId': order_id,
                'status': new_status,
            }
        }),
        operationName='UpdateOrderStatus',
    )

    return response
```

---

## 8. Kiến Trúc Fan-out Với Subscription

### Pattern: SNS/SQS → Lambda → AppSync → Clients

```
┌──────────────┐     ┌───────┐     ┌────────────┐
│  Order       │────►│  SNS  │────►│  SQS Queue │
│  Service     │     │ Topic │     │            │
└──────────────┘     └───────┘     └──────┬─────┘
                                          │
                                          ▼
                                   ┌────────────┐
                                   │   Lambda   │
                                   │ (processor)│
                                   └──────┬─────┘
                                          │
                                          │ GraphQL Mutation
                                          │ (via IAM auth)
                                          ▼
                                   ┌────────────┐
                                   │  AppSync   │
                                   │  (GraphQL) │
                                   └──────┬─────┘
                                          │
                          ┌───────────────┼───────────────┐
                          │ WebSocket     │ WebSocket     │ WebSocket
                          ▼               ▼               ▼
                     ┌────────┐     ┌────────┐     ┌────────┐
                     │ Web    │     │ Mobile │     │ Admin  │
                     │ Client │     │  App   │     │ Panel  │
                     └────────┘     └────────┘     └────────┘
```

### Khi Nào Dùng Kiến Trúc Này?

```
✅ Backend services không phải AppSync cần push events đến clients
✅ Nhiều services khác nhau cần trigger cùng một subscription
✅ Xử lý async (bất đồng bộ) — không muốn block HTTP request
✅ Cần queue/buffer events khi AppSync tạm thời không có client
```

---

## 9. Giới Hạn & Best Practices

### Service Limits (Giới Hạn Dịch Vụ)

| Giới Hạn | Giá Trị Mặc Định | Ghi Chú |
|---|---|---|
| **Max connections per API** (Kết nối tối đa) | 1,000,000 | Tăng được qua AWS Support |
| **Subscriptions per connection** (Đăng ký mỗi kết nối) | 100 | Không thay đổi được |
| **Max connection duration** (Thời gian kết nối tối đa) | 7,200 giây (2h) | Cố định |
| **Message payload size** (Kích thước payload) | 128 KB | Cố định |
| **Filter expressions per subscription** (Biểu thức lọc) | Unlimited | Phức tạp → tốn CPU |

### Best Practices (Thực Tiễn Tốt Nhất)

```
1. LUÔN unsubscribe khi component/screen bị hủy
   → Tránh memory leak và chi phí thừa

2. Implement reconnection logic với exponential backoff
   → Network có thể bị gián đoạn

3. Tránh subscribe quá rộng (không có filter)
   → Chi phí tăng vì AppSync gửi event đến nhiều clients hơn

4. Dùng Enhanced Subscription Filtering thay vì filter ở client-side
   → Giảm bandwidth, giảm load cho client

5. Thiết kế payload subscription đủ nhỏ
   → Chỉ include những fields cần thiết để update UI
   → Nếu cần thêm data, client tự query sau khi nhận event

6. Tránh dùng subscription thay cho polling khi data không thực sự real-time
   → Subscription connection có chi phí (connection minutes)
   → Nếu update vài phút một lần → query thường xuyên đủ dùng

7. Giới hạn số subscriptions đồng thời trên một page
   → Nhiều subscriptions = nhiều WebSocket frames = tốn băng thông
```

### Lỗi Phổ Biến & Cách Xử Lý

```
Lỗi: "Connection timeout" (Hết thời gian kết nối)
→ Nguyên nhân: Client bị inactive quá lâu
→ Xử lý: Implement reconnect khi nhận disconnect event

Lỗi: "Unauthorized" khi subscribe
→ Nguyên nhân: Token hết hạn trong khi đang subscribed
→ Xử lý: Refresh token trước khi hết hạn, reconnect sau khi refresh

Lỗi: Không nhận được events dù mutation thành công
→ Nguyên nhân 1: Subscription filter không match mutation result
→ Nguyên nhân 2: Subscription type không liên kết đúng mutation
→ Xử lý: Kiểm tra @aws_subscribe directive và filter expressions

Lỗi: "Too many connections" (Quá nhiều kết nối)
→ Nguyên nhân: Đạt giới hạn connections
→ Xử lý: Liên hệ AWS Support tăng limit, hoặc optimize để giảm connections
```

---

## 10. Câu Hỏi Phỏng Vấn

### Q1: AppSync Subscriptions hoạt động như thế nào bên dưới?

**Trả lời mẫu:**

AppSync Subscriptions dùng **WebSocket** (giao thức kết nối hai chiều liên tục). Khi client subscribe:

1. Client thiết lập WebSocket connection đến AppSync endpoint
2. Client gửi subscription registration với GraphQL query và variables
3. Khi **mutation** được thực thi (bởi bất kỳ client nào), AppSync kiểm tra xem mutation đó có trigger subscription nào không (thông qua `@aws_subscribe` directive)
4. AppSync **evaluate filter** — so sánh subscription arguments với mutation result
5. Chỉ các subscribers **phù hợp filter** mới nhận được event qua WebSocket

Điểm quan trọng: AppSync quản lý hoàn toàn WebSocket infrastructure — không cần tự xây dựng WebSocket server.

---

### Q2: Làm sao đảm bảo một user chỉ nhận events của chính họ?

**Trả lời mẫu:**

Có hai cách:

**Cách 1 — Argument-based filtering (Đơn giản):**
Thêm `userId` vào subscription argument. AppSync tự động so sánh `userId` trong subscription với `userId` trong mutation result. Chỉ match khi bằng nhau.

**Cách 2 — Enhanced Subscription Filtering + Lambda Authorizer (An toàn hơn):**
Lambda Authorizer xác minh token và trả về `userId` trong resolver context. Subscription resolver dùng context đó để set filter expression — client không thể tự override userId vì nó đến từ token đã xác thực.

Cách 2 an toàn hơn vì ngăn client "giả mạo" userId trong subscription arguments.

---

### Q3: Khi nào nên dùng Subscription thay vì polling?

**Trả lời mẫu:**

Tôi chọn **Subscription** khi:
- Cần **cập nhật tức thì** (< 1 giây) — ví dụ: chat, live notification
- Dữ liệu thay đổi **thường xuyên và không đoán trước được** thời điểm
- Có **nhiều clients** cùng theo dõi (subscription fan-out hiệu quả hơn nhiều polling)

Tôi chọn **polling** khi:
- Dữ liệu chỉ cần cập nhật sau **vài phút** một lần
- Client không thể duy trì WebSocket lâu dài (môi trường network hạn chế)
- Chỉ cần **background refresh** — user không cần thấy ngay lập tức

---

### Q4: Nếu client bị mất kết nối và reconnect, sẽ bị miss events không?

**Trả lời mẫu:**

**Có** — AppSync không lưu buffer events khi client disconnect. Đây là trade-off của WebSocket push model.

Để giải quyết "missed events" (bỏ lỡ sự kiện):

1. **Re-query sau khi reconnect** — khi kết nối lại, luôn fetch latest state từ server
2. **Dùng timestamp/version** — sau khi reconnect, query "tất cả events sau timestamp X"
3. **Optimistic UI** — cập nhật UI ngay khi mutation thực hiện, subscription chỉ để sync

Đây là lý do tại sao subscription thường đi kèm với initial query — fetch dữ liệu hiện tại khi load, sau đó dùng subscription để nhận updates.

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
**Module:** 07-appsync | App Integration Series
