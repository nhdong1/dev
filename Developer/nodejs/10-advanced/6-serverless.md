# Serverless — AWS Lambda, Cold Start và Event-Driven

> Serverless (Không Máy Chủ) là mô hình cloud computing — bạn viết code, cloud provider quản lý infrastructure (hạ tầng), scaling (mở rộng), và billing (thanh toán) theo execution time (thời gian thực thi).

## Mục Lục

1. [Serverless là gì](#serverless-là-gì)
2. [AWS Lambda Fundamentals](#aws-lambda-fundamentals)
3. [Lambda Handler Patterns](#lambda-handler-patterns)
4. [Cold Start — Vấn Đề và Giải Pháp](#cold-start--vấn-đề-và-giải-pháp)
5. [Event Sources](#event-sources)
6. [API Gateway Integration](#api-gateway-integration)
7. [Serverless Framework](#serverless-framework)
8. [Local Development](#local-development)
9. [Production Best Practices](#production-best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Serverless là gì

### Traditional vs Serverless

```
Traditional Server:
┌─────────────────────────────────────┐
│  EC2/VM — Always Running            │
│  Pay 24/7 regardless of traffic     │
│  You manage: OS, scaling, patching  │
└─────────────────────────────────────┘

Serverless (Lambda):
┌─────────────────────────────────────┐
│  Function — Runs on demand          │
│  Pay per invocation + duration      │
│  AWS manages: scaling, patching     │
└─────────────────────────────────────┘
```

| Aspect | Traditional | Serverless |
| ------ | ----------- | ---------- |
| Scaling | Manual/auto-scaling groups | Automatic, instant |
| Cost model | Pay for provisioned capacity | Pay per execution |
| Cold start | Không có | Có (100ms–3s) |
| Max execution | Unlimited | 15 minutes (Lambda) |
| State | Stateful possible | Stateless (use external store) |
| Vendor lock-in | Thấp hơn | Cao hơn |

**Phù hợp Serverless:**
- Spiky/unpredictable traffic
- Event-driven workflows (S3 upload → process)
- Scheduled tasks (cron jobs)
- Micro-APIs, webhooks
- Prototyping, MVP

**Không phù hợp:**
- Long-running processes (>15 min)
- WebSocket persistent connections
- Low-latency requirements (<50ms consistently)
- Heavy CPU workloads liên tục

---

## AWS Lambda Fundamentals

### Basic Lambda Function

```javascript
// handler.js
exports.handler = async (event, context) => {
  console.log('Event:', JSON.stringify(event));
  console.log('Request ID:', context.awsRequestId);

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      message: 'Hello from Lambda!',
      requestId: context.awsRequestId,
    }),
  };
};
```

### Context Object

```javascript
exports.handler = async (event, context) => {
  // Useful context properties
  console.log({
    functionName: context.functionName,
    functionVersion: context.functionVersion,
    memoryLimitInMB: context.memoryLimitInMB,
    awsRequestId: context.awsRequestId,
    remainingTimeInMillis: context.getRemainingTimeInMillis(),
  });

  // Prevent Lambda from waiting for event loop
  context.callbackWaitsForEmptyEventLoop = false;
};
```

### Lambda Configuration

| Setting | Range | Impact |
| ------- | ----- | ------ |
| Memory | 128MB – 10,240MB | More memory = more CPU |
| Timeout | 1s – 900s (15 min) | Max execution time |
| Ephemeral storage | 512MB – 10,240MB | `/tmp` directory |
| Concurrency | Reserved + account limit | Parallel executions |

**Quy tắc:** Memory cao hơn → CPU nhanh hơn → có thể rẻ hơn vì execution time ngắn hơn.

---

## Lambda Handler Patterns

### API Handler với Express (serverless-http)

```javascript
const serverless = require('serverless-http');
const express = require('express');

const app = express();
app.use(express.json());

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.get('/users/:id', async (req, res) => {
  const user = await getUser(req.params.id);
  if (!user) return res.status(404).json({ error: 'Not found' });
  res.json(user);
});

module.exports.handler = serverless(app);
```

### S3 Event Handler

```javascript
const { S3Client, GetObjectCommand } = require('@aws-sdk/client-s3');

const s3 = new S3Client({});

exports.handler = async (event) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));

    console.log(`Processing: s3://${bucket}/${key}`);

    const response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
    const content = await response.Body.transformToString();

    await processFile(content, key);
  }
};
```

### SQS Queue Handler

```javascript
exports.handler = async (event) => {
  const batchItemFailures = [];

  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.body);
      await processMessage(message);
    } catch (err) {
      console.error('Failed to process:', record.messageId, err);
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  // Partial batch failure — chỉ retry failed messages
  return { batchItemFailures };
};
```

---

## Cold Start — Vấn Đề và Giải Pháp

### Cold Start là gì

```
Request 1 (Cold Start):
Client ──► API Gateway ──► [Init Runtime + Load Code + Execute] ──► Response
                              ↑ 500ms–3000ms overhead

Request 2+ (Warm):
Client ──► API Gateway ──► [Execute only] ──► Response
                              ↑ 10–100ms
```

### Cold Start Factors

| Factor | Impact |
| ------ | ------ |
| Runtime (Node.js 20) | ~200–400ms init |
| Package size | Larger = slower load |
| Memory allocation | More memory = faster init |
| VPC configuration | +1–10s nếu trong VPC |
| Provisioned concurrency | Eliminates cold start |

### Optimization Strategies

```javascript
// ❌ Heavy init at module level — runs every cold start
const heavyLib = require('massive-library');
const db = await connectDatabase(); // Top-level await in init

// ✅ Lazy initialization
let dbConnection = null;

async function getDb() {
  if (!dbConnection) {
    dbConnection = await connectDatabase();
  }
  return dbConnection;
}

exports.handler = async (event) => {
  const db = await getDb();
  // ...
};
```

### Bundle Size Optimization

```bash
# Dùng esbuild/webpack tree-shaking
npm install esbuild

# esbuild.config.js
require('esbuild').build({
  entryPoints: ['handler.js'],
  bundle: true,
  platform: 'node',
  target: 'node20',
  outfile: 'dist/handler.js',
  external: ['aws-sdk'], // AWS SDK included in Lambda runtime
  minify: true,
});
```

### Provisioned Concurrency

```yaml
# serverless.yml
functions:
  api:
    handler: handler.main
    provisionedConcurrency: 5  # Always warm instances
    reservedConcurrency: 100   # Max concurrent executions
```

**Trade-off:** Provisioned concurrency có cost cố định — chỉ dùng cho latency-critical endpoints.

---

## Event Sources

| Event Source | Trigger | Use Case |
| ------------ | ------- | -------- |
| **API Gateway** | HTTP request | REST APIs |
| **S3** | Object created/deleted | File processing |
| **SQS** | Message in queue | Async processing |
| **SNS** | Notification published | Fan-out patterns |
| **DynamoDB Streams** | Table change | CDC (Change Data Capture) |
| **EventBridge** | Scheduled/event rules | Cron jobs, event routing |
| **Kinesis** | Stream records | Real-time analytics |

### EventBridge Scheduled (Cron)

```yaml
functions:
  dailyReport:
    handler: reports.generate
    events:
      - schedule:
          rate: cron(0 8 * * ? *)  # 8 AM UTC daily
          enabled: true
```

```javascript
// reports.js
exports.generate = async (event) => {
  console.log('Generating daily report...');
  const report = await generateReport();
  await sendEmail(report);
  return { status: 'completed' };
};
```

---

## API Gateway Integration

### HTTP API (v2) — Recommended

```yaml
# serverless.yml
service: my-api

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-southeast-1
  environment:
    DATABASE_URL: ${env:DATABASE_URL}

functions:
  api:
    handler: dist/handler.main
    events:
      - httpApi:
          path: /{proxy+}
          method: ANY
```

### Lambda Response Format

```javascript
// API Gateway v2 (HTTP API) response
return {
  statusCode: 200,
  headers: {
    'Content-Type': 'application/json',
    'Access-Control-Allow-Origin': '*',
  },
  body: JSON.stringify({ data: result }),
  isBase64Encoded: false,
};
```

### CORS với API Gateway

```yaml
provider:
  httpApi:
    cors:
      allowedOrigins:
        - https://myapp.com
      allowedHeaders:
        - Content-Type
        - Authorization
      allowedMethods:
        - GET
        - POST
        - PUT
        - DELETE
```

---

## Serverless Framework

### Project Structure

```
my-serverless-app/
├── serverless.yml
├── src/
│   ├── handlers/
│   │   ├── api.js
│   │   └── processor.js
│   └── lib/
│       └── db.js
├── package.json
└── esbuild.config.js
```

### serverless.yml

```yaml
service: order-service

frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs20.x
  stage: ${opt:stage, 'dev'}
  region: ap-southeast-1
  memorySize: 512
  timeout: 30
  environment:
    STAGE: ${self:provider.stage}
    ORDERS_TABLE: ${self:custom.ordersTable}

  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:Query
          Resource: !GetAtt OrdersTable.Arn

custom:
  ordersTable: orders-${self:provider.stage}
  esbuild:
    bundle: true
    minify: true

plugins:
  - serverless-esbuild

functions:
  createOrder:
    handler: src/handlers/orders.create
    events:
      - httpApi:
          path: /orders
          method: POST

  processOrder:
    handler: src/handlers/orders.process
    events:
      - sqs:
          arn: !GetAtt OrderQueue.Arn
          batchSize: 10

resources:
  Resources:
    OrdersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:custom.ordersTable}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
```

### Deploy Commands

```bash
# Install
npm install -g serverless

# Deploy to dev
serverless deploy --stage dev

# Deploy single function (faster)
serverless deploy function -f createOrder

# View logs
serverless logs -f createOrder --tail

# Remove all resources
serverless remove --stage dev
```

---

## Local Development

### serverless-offline

```bash
npm install --save-dev serverless-offline
```

```yaml
plugins:
  - serverless-esbuild
  - serverless-offline

custom:
  serverless-offline:
    httpPort: 3000
```

```bash
serverless offline start
# API available at http://localhost:3000
```

### AWS SAM Local

```bash
sam local start-api
sam local invoke CreateOrderFunction -e events/create-order.json
```

---

## Production Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Keep functions small & focused | Single responsibility, faster cold start |
| Minimize bundle size | Tree-shake, exclude devDependencies |
| Use environment variables | Config per stage, secrets via SSM/Secrets Manager |
| Set appropriate timeout | Không quá cao — fail fast |
| Implement idempotency | Duplicate invocations có thể xảy ra |
| Structured logging | CloudWatch Logs Insights queries |
| X-Ray tracing | Debug latency issues |
| Dead letter queue (DLQ) | Capture failed async invocations |

### Idempotency Pattern

```javascript
const { DynamoDBClient, PutItemCommand } = require('@aws-sdk/client-dynamodb');

const dynamo = new DynamoDBClient({});

async function processPayment(orderId, amount) {
  // Idempotency key check
  try {
    await dynamo.send(new PutItemCommand({
      TableName: 'IdempotencyKeys',
      Item: {
        key: { S: `payment-${orderId}` },
        ttl: { N: String(Math.floor(Date.now() / 1000) + 86400) },
      },
      ConditionExpression: 'attribute_not_exists(#k)',
      ExpressionAttributeNames: { '#k': 'key' },
    }));
  } catch (err) {
    if (err.name === 'ConditionalCheckFailedException') {
      console.log('Already processed, skipping');
      return { status: 'already_processed' };
    }
    throw err;
  }

  // Process payment...
  return { status: 'completed' };
}
```

### Structured Logging

```javascript
const logger = {
  info: (message, data = {}) => {
    console.log(JSON.stringify({
      level: 'info',
      message,
      requestId: process.env.AWS_REQUEST_ID,
      timestamp: new Date().toISOString(),
      ...data,
    }));
  },
  error: (message, error, data = {}) => {
    console.error(JSON.stringify({
      level: 'error',
      message,
      error: error.message,
      stack: error.stack,
      requestId: process.env.AWS_REQUEST_ID,
      timestamp: new Date().toISOString(),
      ...data,
    }));
  },
};
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
| ------- | ------------ |
| Cold start là gì? | Latency khi Lambda init runtime lần đầu — 200ms–3s |
| Cách giảm cold start? | Smaller bundle, more memory, provisioned concurrency, avoid VPC |
| Lambda limitations? | 15 min timeout, 10GB memory, ephemeral /tmp storage |
| Stateless nghĩa là gì? | Không lưu state giữa invocations — dùng DB/cache external |
| Lambda trong VPC? | Cần cho RDS access, nhưng cold start +1–10s |
| Idempotency tại sao quan trọng? | SQS/Lambda có thể deliver duplicate — cần handle |
| Serverless vs containers? | Serverless: pay-per-use, auto-scale; Containers: more control, no cold start |
| API Gateway types? | REST API (feature-rich), HTTP API (cheaper, faster), WebSocket API |
