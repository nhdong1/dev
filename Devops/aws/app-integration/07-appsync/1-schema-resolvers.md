# Schema, Resolver & Data Sources — Nền Tảng AppSync

> **Schema** (Lược Đồ) định nghĩa "hợp đồng API", **Resolver** (Bộ Giải Quyết) là cầu nối đến dữ liệu, **Data Source** (Nguồn Dữ Liệu) là nơi dữ liệu thực sự được lưu trữ hoặc xử lý.

---

## 📚 Mục Lục

1. [GraphQL Schema Trong AppSync](#1-graphql-schema-trong-appsync)
2. [Resolver — Bộ Giải Quyết](#2-resolver--bộ-giải-quyết)
3. [VTL — Velocity Template Language](#3-vtl--velocity-template-language)
4. [JavaScript Resolvers — Runtime Mới](#4-javascript-resolvers--runtime-mới)
5. [Pipeline Resolvers — Bộ Giải Quyết Đường Ống](#5-pipeline-resolvers--bộ-giải-quyết-đường-ống)
6. [Data Sources — Nguồn Dữ Liệu](#6-data-sources--nguồn-dữ-liệu)
7. [Direct Lambda Resolver](#7-direct-lambda-resolver)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. GraphQL Schema Trong AppSync

### Cấu Trúc Schema Đầy Đủ

```graphql
# ─────────────────────────────────────────────
# SCALAR TYPES (Kiểu Vô Hướng) — Tùy Chỉnh
# ─────────────────────────────────────────────
scalar AWSDateTime    # ISO 8601 date-time string
scalar AWSJSON        # JSON string
scalar AWSEmail       # Email address
scalar AWSURL         # URL string
scalar AWSPhone       # Phone number
scalar AWSIPAddress   # IPv4 hoặc IPv6

# ─────────────────────────────────────────────
# OBJECT TYPES (Kiểu Đối Tượng)
# ─────────────────────────────────────────────
type User {
  id: ID!                      # ! = Non-nullable (bắt buộc có giá trị)
  email: AWSEmail!
  name: String!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime
  orders: [Order!]             # Danh sách Order liên quan
  profile: UserProfile         # Có thể null (nullable)
}

type Order {
  id: ID!
  userId: ID!
  status: OrderStatus!
  items: [OrderItem!]!
  total: Float!
  createdAt: AWSDateTime!
}

type OrderItem {
  productId: ID!
  quantity: Int!
  price: Float!
}

type UserProfile {
  bio: String
  avatarUrl: AWSURL
}

# ─────────────────────────────────────────────
# ENUM TYPE (Kiểu Liệt Kê)
# ─────────────────────────────────────────────
enum OrderStatus {
  PENDING      # Đang chờ xử lý
  CONFIRMED    # Đã xác nhận
  PROCESSING   # Đang xử lý
  SHIPPED      # Đã gửi hàng
  DELIVERED    # Đã giao hàng
  CANCELLED    # Đã hủy
}

# ─────────────────────────────────────────────
# INPUT TYPES (Kiểu Đầu Vào) — Dùng cho Mutation
# ─────────────────────────────────────────────
input CreateOrderInput {
  userId: ID!
  items: [OrderItemInput!]!
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
  price: Float!
}

input UpdateOrderStatusInput {
  orderId: ID!
  status: OrderStatus!
}

# ─────────────────────────────────────────────
# QUERY TYPE — Định Nghĩa Các Truy Vấn Đọc Dữ Liệu
# ─────────────────────────────────────────────
type Query {
  getUser(id: ID!): User
  listUsers(limit: Int, nextToken: String): UserConnection
  getOrder(id: ID!): Order
  listOrdersByUser(userId: ID!, status: OrderStatus): [Order!]
}

# ─────────────────────────────────────────────
# MUTATION TYPE — Định Nghĩa Các Thao Tác Ghi Dữ Liệu
# ─────────────────────────────────────────────
type Mutation {
  createOrder(input: CreateOrderInput!): Order!
  updateOrderStatus(input: UpdateOrderStatusInput!): Order!
  deleteOrder(id: ID!): Boolean!
}

# ─────────────────────────────────────────────
# SUBSCRIPTION TYPE — Định Nghĩa Lắng Nghe Real-time
# ─────────────────────────────────────────────
type Subscription {
  # Lắng nghe khi một order mới được tạo
  onCreateOrder(userId: ID): Order
    @aws_subscribe(mutations: ["createOrder"])

  # Lắng nghe khi status thay đổi
  onOrderStatusChanged(orderId: ID): Order
    @aws_subscribe(mutations: ["updateOrderStatus"])
}

# ─────────────────────────────────────────────
# CONNECTION TYPE — Pagination Pattern (Mẫu Phân Trang)
# ─────────────────────────────────────────────
type UserConnection {
  items: [User!]
  nextToken: String    # Cursor cho trang tiếp theo (null = hết trang)
}
```

### Các Directive (Chỉ Thị) Quan Trọng Trong AppSync

```graphql
# @aws_subscribe — Kết nối subscription với mutation triggers
type Subscription {
  onCreateUser: User
    @aws_subscribe(mutations: ["createUser"])
}

# @deprecated — Đánh dấu field không còn được dùng
type User {
  oldField: String @deprecated(reason: "Dùng newField thay thế")
  newField: String
}

# @aws_auth — Yêu cầu xác thực Cognito
type Query {
  getPrivateData: String @aws_auth(cognito_groups: ["Admins"])
}

# @aws_cognito_user_pools — Chỉ cho Cognito users
# @aws_api_key — Cho phép API key access
# @aws_iam — Cho phép IAM access
# @aws_oidc — Cho phép OIDC access
# @aws_lambda — Dùng Lambda authorizer
type Post @aws_cognito_user_pools @aws_api_key {
  id: ID!
  title: String!
  secretContent: String @aws_cognito_user_pools  # Chỉ authenticated users
}
```

---

## 2. Resolver — Bộ Giải Quyết

### Khái Niệm Cốt Lõi

Mỗi **field** trong Query, Mutation, Subscription cần một Resolver để biết:
1. Lấy dữ liệu từ đâu (data source nào)
2. Dữ liệu cần được transform (biến đổi) như thế nào
3. Trả kết quả về dưới dạng gì

```
GraphQL Request
      │
      ▼
┌─────────────────────────────────────────┐
│              RESOLVER                    │
│                                          │
│  ┌─────────────────┐                    │
│  │ Request Mapping │  Transform request  │
│  │    Template     │  → data source format│
│  └────────┬────────┘                    │
│           ▼                             │
│  ┌─────────────────┐                    │
│  │   Data Source   │  Execute operation  │
│  │  (DynamoDB /    │  against actual data │
│  │   Lambda / ...) │                    │
│  └────────┬────────┘                    │
│           ▼                             │
│  ┌─────────────────┐                    │
│  │ Response Mapping│  Transform response │
│  │    Template     │  → GraphQL format   │
│  └─────────────────┘                    │
└─────────────────────────────────────────┘
      │
      ▼
GraphQL Response
```

### Hai Loại Resolver

```
1. Unit Resolver (Bộ Giải Quyết Đơn)
   → Kết nối trực tiếp 1 field với 1 data source
   → Đơn giản, đủ dùng cho CRUD cơ bản

2. Pipeline Resolver (Bộ Giải Quyết Đường Ống)
   → Chuỗi nhiều "function" chạy tuần tự
   → Dùng khi cần nhiều bước: auth check → query DB → transform
```

---

## 3. VTL — Velocity Template Language

VTL (Velocity Template Language — Ngôn Ngữ Template Velocity) là cú pháp truyền thống để viết request/response mapping templates trong AppSync.

### Request Mapping Template — DynamoDB GetItem

```vtl
## Lấy một item từ DynamoDB theo primary key
{
  "version": "2018-05-29",
  "operation": "GetItem",
  "key": {
    "id": $util.dynamodb.toDynamoDBJson($ctx.args.id)
  }
}
```

### Response Mapping Template — Cơ Bản

```vtl
## $ctx.result chứa response từ data source
## $util.toJson() chuyển đổi sang JSON

## Trả về trực tiếp nếu không có lỗi
#if($ctx.error)
  $util.error($ctx.error.message, $ctx.error.type)
#end
$util.toJson($ctx.result)
```

### Request Mapping Template — DynamoDB Query

```vtl
## Query DynamoDB với index
{
  "version": "2018-05-29",
  "operation": "Query",
  "index": "userId-createdAt-index",
  "query": {
    "expression": "userId = :userId",
    "expressionValues": {
      ":userId": $util.dynamodb.toDynamoDBJson($ctx.args.userId)
    }
  },
  #if($ctx.args.status)
  "filter": {
    "expression": "#status = :status",
    "expressionNames": {
      "#status": "status"
    },
    "expressionValues": {
      ":status": $util.dynamodb.toDynamoDBJson($ctx.args.status)
    }
  },
  #end
  "limit": $util.defaultIfNull($ctx.args.limit, 20),
  "nextToken": $util.toJson($util.defaultIfNullOrEmpty($ctx.args.nextToken, null))
}
```

### Request Mapping Template — DynamoDB PutItem (Mutation)

```vtl
## Tạo item mới trong DynamoDB
{
  "version": "2018-05-29",
  "operation": "PutItem",
  "key": {
    "id": $util.dynamodb.toDynamoDBJson($util.autoId())
  },
  "attributeValues": $util.dynamodb.toMapValuesJson({
    "userId": $ctx.args.input.userId,
    "status": "PENDING",
    "total": $ctx.args.input.total,
    "createdAt": $util.time.nowISO8601(),
    "updatedAt": $util.time.nowISO8601()
  }),
  "condition": {
    "expression": "attribute_not_exists(id)"
  }
}
```

### Các $util Helpers Quan Trọng Trong VTL

```vtl
## Tạo UUID ngẫu nhiên
$util.autoId()

## Chuyển đổi sang DynamoDB format
$util.dynamodb.toDynamoDBJson($value)
$util.dynamodb.toMapValuesJson($map)

## Lấy timestamp hiện tại
$util.time.nowISO8601()      # ISO 8601 string
$util.time.nowEpochMilliSeconds()   # Milliseconds

## Xử lý null / default
$util.defaultIfNull($value, $default)
$util.defaultIfNullOrEmpty($str, $default)

## Throw error (báo lỗi)
$util.error("Không tìm thấy user", "NotFound")
$util.unauthorized()     # Lỗi 401

## Conditional
$util.isNullOrEmpty($str)
$util.isNull($value)

## Encode/Decode
$util.base64Encode($str)
$util.base64Decode($str)
```

---

## 4. JavaScript Resolvers — Runtime Mới

AWS giới thiệu **APPSYNC_JS runtime** — cho phép viết resolver bằng **JavaScript (ES6+)** thay cho VTL. Đây là hướng được AWS khuyến nghị cho các dự án mới.

### So Sánh VTL vs JavaScript Resolver

| Tiêu Chí | VTL (Cũ) | JavaScript (Mới) |
|---|---|---|
| **Cú pháp** | Template language — khó debug | JavaScript — thân quen hơn |
| **Type safety** (Kiểm tra kiểu) | Không có | Có (với TypeScript) |
| **Testing** (Kiểm thử) | Khó | Dễ hơn với local testing |
| **Error messages** (Thông báo lỗi) | Khó hiểu | Rõ ràng hơn |
| **AWS hỗ trợ** | Legacy | Mới — được khuyến nghị |
| **Libraries** (Thư viện) | Không có | Subset của JS (không có Node.js) |

### JavaScript Resolver — Ví Dụ DynamoDB GetItem

```javascript
// request() — Tạo request gửi đến data source
export function request(ctx) {
  return {
    operation: 'GetItem',
    key: util.dynamodb.toMapValues({ id: ctx.args.id }),
  };
}

// response() — Xử lý response từ data source
export function response(ctx) {
  const { error, result } = ctx;

  if (error) {
    util.error(error.message, error.type);
  }

  if (!result) {
    util.error('Không tìm thấy user', 'NotFound');
  }

  return result;
}
```

### JavaScript Resolver — DynamoDB Query Với Filtering

```javascript
import { util } from '@aws-appsync/utils';
import * as ddb from '@aws-appsync/utils/dynamodb';

export function request(ctx) {
  const { userId, status, limit = 20, nextToken } = ctx.args;

  // Xây dựng query expression
  const query = ddb.operations.query({
    index: 'userId-createdAt-index',
    query: ddb.filter.eq('userId', userId),
    filter: status ? ddb.filter.eq('status', status) : undefined,
    limit,
    nextToken,
    scanIndexForward: false,  // Sort giảm dần (mới nhất trước)
  });

  return query;
}

export function response(ctx) {
  if (ctx.error) {
    util.error(ctx.error.message, ctx.error.type);
  }

  return {
    items: ctx.result.items,
    nextToken: ctx.result.nextToken,
  };
}
```

### JavaScript Resolver — PutItem (Mutation)

```javascript
import { util } from '@aws-appsync/utils';
import * as ddb from '@aws-appsync/utils/dynamodb';

export function request(ctx) {
  const { input } = ctx.args;
  const id = util.autoId();
  const now = util.time.nowISO8601();

  return ddb.operations.put({
    key: { id },
    item: {
      ...input,
      id,
      status: 'PENDING',
      createdAt: now,
      updatedAt: now,
    },
    condition: { attributeExists: false },  // Chặn duplicate
  });
}

export function response(ctx) {
  if (ctx.error) {
    util.error(ctx.error.message, ctx.error.type);
  }
  return ctx.result;
}
```

---

## 5. Pipeline Resolvers — Bộ Giải Quyết Đường Ống

Pipeline Resolver cho phép thực thi **nhiều functions tuần tự** trong một request. Mỗi function nhận output của function trước.

### Khi Nào Dùng Pipeline Resolver?

```
✅ Cần kiểm tra quyền trước khi truy vấn DB
✅ Cần ghi log sau mỗi mutation
✅ Cần kết hợp dữ liệu từ nhiều data sources
✅ Cần validation phức tạp trước khi ghi
✅ Cần post-processing sau khi đọc dữ liệu
```

### Cấu Trúc Pipeline Resolver

```
GraphQL Request
      │
      ▼
┌──────────────────────────────────────────────────┐
│                PIPELINE RESOLVER                  │
│                                                   │
│  Before Mapping Template (Chuẩn bị đầu vào)     │
│       │                                           │
│       ▼                                           │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐   │
│  │Function 1│ →  │Function 2│ →  │Function 3│   │
│  │(Auth     │    │(Query DB)│    │(Log/     │   │
│  │ Check)   │    │          │    │ Audit)   │   │
│  └──────────┘    └──────────┘    └──────────┘   │
│       │                                           │
│       ▼                                           │
│  After Mapping Template (Chuẩn bị kết quả)      │
└──────────────────────────────────────────────────┘
      │
      ▼
GraphQL Response
```

### Ví Dụ: Pipeline Resolver Với JavaScript

```javascript
// ═══════════════════════════════════════════
// FUNCTION 1: Auth Check (Kiểm Tra Quyền)
// ═══════════════════════════════════════════

// authCheck/request.js
export function request(ctx) {
  // Kiểm tra user có quyền truy cập không
  const requestedUserId = ctx.args.userId;
  const currentUserId = ctx.identity.sub;  // Cognito user ID

  if (requestedUserId !== currentUserId) {
    // Kiểm tra có phải admin không (qua custom Lambda authorizer context)
    const isAdmin = ctx.identity.claims['cognito:groups']?.includes('Admins');
    if (!isAdmin) {
      util.unauthorized();
    }
  }

  // Truyền sang function tiếp theo qua stash
  ctx.stash.isAdmin = isAdmin ?? false;

  // Return {} để không gọi data source
  return {};
}

// authCheck/response.js
export function response(ctx) {
  return ctx.prev.result;  // Truyền tiếp kết quả
}


// ═══════════════════════════════════════════
// FUNCTION 2: Query DynamoDB
// ═══════════════════════════════════════════

// queryOrders/request.js
export function request(ctx) {
  const { userId, limit = 20, nextToken } = ctx.args;

  return {
    operation: 'Query',
    index: 'userId-createdAt-index',
    query: {
      expression: 'userId = :userId',
      expressionValues: util.dynamodb.toMapValues({ ':userId': userId }),
    },
    limit,
    nextToken: nextToken ?? null,
  };
}

// queryOrders/response.js
export function response(ctx) {
  if (ctx.error) {
    util.error(ctx.error.message, ctx.error.type);
  }
  return ctx.result;
}


// ═══════════════════════════════════════════
// PIPELINE RESOLVER: Before & After Templates
// ═══════════════════════════════════════════

// before.js — Chạy trước tất cả functions
export function request(ctx) {
  // Ghi nhận thông tin request để log
  ctx.stash.requestTime = util.time.nowISO8601();
  ctx.stash.requesterId = ctx.identity?.sub ?? 'anonymous';
  return {};
}

// after.js — Chạy sau tất cả functions
export function response(ctx) {
  if (ctx.error) {
    util.error(ctx.error.message, ctx.error.type);
  }
  // ctx.result chứa kết quả của function cuối cùng
  return ctx.result;
}
```

### ctx.stash — Truyền Dữ Liệu Giữa Functions

```javascript
// ctx.stash là object shared giữa các functions trong pipeline
// Function 1 — ghi vào stash
ctx.stash.userId = 'abc123';
ctx.stash.permissions = ['read', 'write'];

// Function 2 — đọc từ stash
const userId = ctx.stash.userId;
const canWrite = ctx.stash.permissions.includes('write');
```

---

## 6. Data Sources — Nguồn Dữ Liệu

### DynamoDB Data Source — CRUD Đầy Đủ

```javascript
// GetItem — Lấy một item theo key
export function request(ctx) {
  return {
    operation: 'GetItem',
    key: util.dynamodb.toMapValues({ id: ctx.args.id }),
  };
}

// PutItem — Tạo hoặc ghi đè item
export function request(ctx) {
  const { input } = ctx.args;
  return {
    operation: 'PutItem',
    key: util.dynamodb.toMapValues({ id: util.autoId() }),
    attributeValues: util.dynamodb.toMapValues({
      ...input,
      createdAt: util.time.nowISO8601(),
    }),
  };
}

// UpdateItem — Cập nhật một số field
export function request(ctx) {
  const { id, status } = ctx.args;
  return {
    operation: 'UpdateItem',
    key: util.dynamodb.toMapValues({ id }),
    update: {
      expression: 'SET #status = :status, updatedAt = :updatedAt',
      expressionNames: { '#status': 'status' },
      expressionValues: util.dynamodb.toMapValues({
        ':status': status,
        ':updatedAt': util.time.nowISO8601(),
      }),
    },
  };
}

// DeleteItem — Xóa item
export function request(ctx) {
  return {
    operation: 'DeleteItem',
    key: util.dynamodb.toMapValues({ id: ctx.args.id }),
  };
}

// BatchGetItem — Lấy nhiều items một lúc (tối đa 100)
export function request(ctx) {
  return {
    operation: 'BatchGetItem',
    tables: {
      'orders-table': {
        keys: ctx.args.ids.map((id) => util.dynamodb.toMapValues({ id })),
      },
    },
  };
}
```

### HTTP Data Source — Gọi Third-party REST API

```javascript
// Gọi một REST API bên ngoài (ví dụ: payment service)
export function request(ctx) {
  const { orderId, amount, currency } = ctx.args.input;

  return {
    method: 'POST',
    resourcePath: '/v1/charges',
    params: {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${ctx.stash.stripeApiKey}`,
      },
      body: JSON.stringify({
        amount,
        currency,
        metadata: { orderId },
      }),
    },
  };
}

export function response(ctx) {
  if (ctx.error) {
    util.error(ctx.error.message, 'PaymentError');
  }

  const body = JSON.parse(ctx.result.body);

  if (ctx.result.statusCode !== 200) {
    util.error(body.error?.message ?? 'Payment failed', 'PaymentFailed');
  }

  return {
    chargeId: body.id,
    status: body.status,
  };
}
```

### None Data Source — Local Resolver (Dùng Cho Subscription)

```javascript
// None data source — không gọi bất kỳ backend nào
// Thường dùng để trigger subscriptions sau mutation

export function request(ctx) {
  // Truyền thẳng mutation arguments sang response
  return {
    payload: ctx.args.input,
  };
}

export function response(ctx) {
  return ctx.result;
}
```

---

## 7. Direct Lambda Resolver

**Direct Lambda Resolver** (Bộ Giải Quyết Lambda Trực Tiếp) bỏ qua mapping templates — AppSync gọi Lambda với payload chuẩn, Lambda trả về kết quả trực tiếp.

### Payload Lambda Nhận

```json
{
  "arguments": {
    "id": "user-123"
  },
  "identity": {
    "sub": "cognito-user-id",
    "issuer": "https://cognito-idp...",
    "username": "john_doe",
    "claims": {
      "cognito:groups": ["Users"]
    }
  },
  "source": null,
  "request": {
    "headers": {}
  },
  "info": {
    "fieldName": "getUser",
    "parentTypeName": "Query",
    "variables": {}
  },
  "prev": null,
  "stash": {}
}
```

### Lambda Function Ví Dụ

```python
import boto3
import json
from typing import Any

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users-table')

def handler(event: dict, context: Any) -> dict:
    field_name = event['info']['fieldName']
    arguments = event['arguments']
    identity = event.get('identity', {})

    # Route theo field name
    if field_name == 'getUser':
        return get_user(arguments['id'])
    elif field_name == 'listUsers':
        return list_users(arguments)
    elif field_name == 'createUser':
        return create_user(arguments['input'], identity)
    else:
        raise Exception(f'Unknown field: {field_name}')


def get_user(user_id: str) -> dict | None:
    response = table.get_item(Key={'id': user_id})
    return response.get('Item')


def list_users(args: dict) -> dict:
    params = {
        'Limit': args.get('limit', 20),
    }
    if args.get('nextToken'):
        params['ExclusiveStartKey'] = json.loads(args['nextToken'])

    response = table.scan(**params)
    return {
        'items': response.get('Items', []),
        'nextToken': json.dumps(response.get('LastEvaluatedKey')) if response.get('LastEvaluatedKey') else None,
    }
```

### Khi Nào Dùng Direct Lambda Resolver?

```
✅ Business logic phức tạp, khó viết trong VTL/JS
✅ Cần gọi nhiều AWS services trong một resolver
✅ Cần xử lý file, binary data
✅ Cần SDK đầy đủ (boto3, AWS SDK for JS)
✅ Team quen viết Lambda hơn VTL

❌ Tránh dùng cho CRUD đơn giản với DynamoDB
   → VTL/JS resolver nhanh hơn (không qua Lambda cold start)
❌ Chi phí cao hơn (mỗi request = 1 Lambda invocation)
❌ Latency cao hơn (cold start Lambda)
```

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: Schema trong AppSync khác Schema trong REST API như thế nào?

**Trả lời mẫu:**

GraphQL Schema trong AppSync định nghĩa **"hợp đồng"** đầy đủ của API — bao gồm tất cả types, queries, mutations, và subscriptions. Client có thể xem schema và biết chính xác dữ liệu nào có thể lấy được và ở dạng nào thông qua **introspection** (tự mô tả).

Khác với REST API nơi mỗi endpoint trả về cấu trúc cố định, GraphQL cho phép **client chỉ định chính xác fields nào cần lấy**, tránh over-fetching. Schema AppSync cũng hỗ trợ **scalar types đặc biệt của AWS** như `AWSDateTime`, `AWSEmail`, `AWSJSON`.

---

### Q2: Resolver là gì và tại sao cần thiết?

**Trả lời mẫu:**

Resolver là **cầu nối** giữa GraphQL field và data source. Nếu Schema là "hợp đồng API", thì Resolver là "người thực thi hợp đồng". 

Mỗi field trong Query/Mutation có thể có một resolver riêng. Resolver bao gồm:
- **Request mapping template**: transform GraphQL arguments sang format data source cần (ví dụ: DynamoDB key format)
- **Response mapping template**: transform kết quả từ data source về GraphQL type

Không có resolver, AppSync không biết lấy dữ liệu từ đâu.

---

### Q3: Khi nào dùng Pipeline Resolver thay vì Unit Resolver?

**Trả lời mẫu:**

Tôi dùng Pipeline Resolver khi một operation cần **nhiều bước logic độc lập** theo thứ tự:

1. **Auth/Permission check** (kiểm tra quyền) trước khi query — ví dụ: người dùng chỉ được xem orders của chính mình
2. **Multi-source aggregation** — lấy dữ liệu từ DynamoDB rồi enrich (bổ sung) từ một REST API
3. **Audit logging** — ghi log sau mỗi mutation quan trọng

Pipeline Resolver giúp tái sử dụng từng "function" độc lập. Ví dụ: function "auth check" có thể dùng chung cho nhiều resolvers khác nhau.

---

### Q4: VTL vs JavaScript Resolver — chọn cái nào?

**Trả lời mẫu:**

Với **dự án mới**, tôi chọn **JavaScript Resolver** vì:
- Cú pháp JavaScript quen thuộc, dễ debug hơn
- Hỗ trợ local testing
- AWS đang đầu tư phát triển hướng này

VTL vẫn được dùng khi:
- Maintain codebase cũ đã dùng VTL
- Team đã quen VTL
- Một số edge cases chưa có equivalent trong JS runtime

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Phiên Bản:** 1.0
**Module:** 07-appsync | App Integration Series
