# gRPC — Protocol Buffers, Streaming RPC và @grpc/grpc-js

> gRPC (gRPC Remote Procedure Call — Gọi Thủ Tục Từ Xa) là framework RPC hiệu năng cao dùng Protocol Buffers (protobuf — định dạng serialization nhị phân) — phổ biến cho giao tiếp service-to-service trong microservices.

## Mục Lục

1. [gRPC vs REST](#grpc-vs-rest)
2. [Protocol Buffers](#protocol-buffers)
3. [Service Definition (.proto)](#service-definition-proto)
4. [Node.js gRPC Server](#nodejs-grpc-server)
5. [gRPC Client](#grpc-client)
6. [Streaming RPC Types](#streaming-rpc-types)
7. [Error Handling và Status Codes](#error-handling-và-status-codes)
8. [Interceptors và Metadata](#interceptors-và-metadata)
9. [gRPC với NestJS](#grpc-với-nestjs)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## gRPC vs REST

| Aspect | REST (JSON/HTTP) | gRPC (Protobuf/HTTP2) |
| ------ | ---------------- | --------------------- |
| Protocol | HTTP/1.1 | HTTP/2 |
| Payload | JSON (text) | Protobuf (binary) |
| Contract | OpenAPI (optional) | .proto files (required) |
| Browser support | Native | Cần gRPC-Web proxy |
| Streaming | Limited (SSE, chunked) | Native bidirectional |
| Performance | Good | Excellent (smaller, faster) |
| Human readable | ✅ | ❌ (binary) |
| Code generation | Optional | Built-in |

```
REST:
Client ──HTTP POST /users──► Server
       ◄──JSON response────

gRPC:
Client ──HTTP/2 POST /UserService/GetUser──► Server
       ◄──Protobuf binary response──────────
```

**Dùng gRPC khi:**
- Internal microservices communication
- High throughput, low latency required
- Streaming data (logs, metrics, file transfer)
- Polyglot services (Java, Go, Python, Node.js)

**Giữ REST khi:**
- Public APIs cho browsers/mobile
- Third-party integrations
- Team chưa quen protobuf

---

## Protocol Buffers

Protobuf định nghĩa message schema trong `.proto` files — compiler generate code cho nhiều languages.

### Cài Đặt

```bash
npm install @grpc/grpc-js @grpc/proto-loader

# Protocol buffer compiler (system level)
# Windows: choco install protoc
# macOS: brew install protobuf
# Linux: apt install protobuf-compiler
```

### Basic Message Definition

```protobuf
// user.proto
syntax = "proto3";

package user;

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
}
```

### Scalar Types

| Proto Type | Node.js Type | Notes |
| ---------- | ------------ | ----- |
| `string` | `string` | UTF-8 |
| `int32`, `int64` | `number` | Variable-length encoding |
| `bool` | `boolean` | |
| `bytes` | `Buffer` | Binary data |
| `repeated` | `Array` | List of items |
| `map<K,V>` | `Object` | Key-value pairs |

---

## Service Definition (.proto)

```protobuf
// user_service.proto
syntax = "proto3";

package user;

service UserService {
  // Unary RPC — 1 request, 1 response
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);

  // Server streaming — 1 request, stream responses
  rpc WatchUsers(WatchUsersRequest) returns (stream User);

  // Client streaming — stream requests, 1 response
  rpc ImportUsers(stream User) returns (ImportUsersResponse);

  // Bidirectional streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

---

## Node.js gRPC Server

### Load Proto và Start Server

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const PROTO_PATH = path.join(__dirname, 'user_service.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

// Service implementation
const userService = {
  getUser: async (call, callback) => {
    const { id } = call.request;

    try {
      const user = await db.user.findUnique({ where: { id } });
      if (!user) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: `User ${id} not found`,
        });
      }
      callback(null, { user });
    } catch (err) {
      callback({
        code: grpc.status.INTERNAL,
        message: err.message,
      });
    }
  },

  listUsers: async (call, callback) => {
    const { page = 1, page_size = 10 } = call.request;
    const skip = (page - 1) * page_size;

    const [users, total] = await Promise.all([
      db.user.findMany({ skip, take: page_size }),
      db.user.count(),
    ]);

    callback(null, { users, total });
  },

  createUser: async (call, callback) => {
    const { name, email } = call.request;
    const user = await db.user.create({ data: { name, email } });
    callback(null, { user });
  },
};

function main() {
  const server = new grpc.Server();
  server.addService(userProto.UserService.service, userService);

  const address = '0.0.0.0:50051';
  server.bindAsync(
    address,
    grpc.ServerCredentials.createInsecure(),
    (err, port) => {
      if (err) {
        console.error('Failed to bind:', err);
        return;
      }
      server.start();
      console.log(`gRPC server running on ${address}`);
    }
  );
}

main();
```

### TLS/SSL Server

```javascript
const fs = require('fs');

const credentials = grpc.ServerCredentials.createSsl(
  fs.readFileSync('ca.crt'),
  [{
    cert_chain: fs.readFileSync('server.crt'),
    private_key: fs.readFileSync('server.key'),
  }],
  true // require client cert (mTLS)
);

server.bindAsync('0.0.0.0:50051', credentials, callback);
```

---

## gRPC Client

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('./user_service.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

const client = new userProto.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Promise wrapper
function getUser(id) {
  return new Promise((resolve, reject) => {
    client.getUser({ id }, (err, response) => {
      if (err) reject(err);
      else resolve(response.user);
    });
  });
}

// Sử dụng
async function main() {
  try {
    const user = await getUser('user-123');
    console.log('User:', user);
  } catch (err) {
    console.error('gRPC error:', err.code, err.details);
  }
}
```

### Client với Metadata (Headers)

```javascript
const metadata = new grpc.Metadata();
metadata.add('authorization', 'Bearer jwt-token-here');
metadata.add('x-request-id', 'req-123');

client.getUser({ id: 'user-123' }, metadata, (err, response) => {
  // ...
});
```

---

## Streaming RPC Types

### Server Streaming

Server gửi stream responses cho 1 request — phù hợp cho large datasets, real-time updates.

```javascript
// Server
watchUsers: (call) => {
  const interval = setInterval(async () => {
    const users = await db.user.findMany({ take: 10 });
    for (const user of users) {
      call.write(user);
    }
  }, 5000);

  call.on('cancelled', () => {
    clearInterval(interval);
  });
},

// Client
const stream = client.watchUsers({});

stream.on('data', (user) => {
  console.log('User update:', user);
});

stream.on('end', () => {
  console.log('Stream ended');
});
```

### Client Streaming

Client gửi stream requests, server trả 1 response — bulk upload, batch processing.

```javascript
// Server
importUsers: async (call, callback) => {
  const users = [];

  call.on('data', (user) => {
    users.push(user);
  });

  call.on('end', async () => {
    const result = await db.user.createMany({ data: users });
    callback(null, { imported: result.count });
  });
},

// Client
const stream = client.importUsers((err, response) => {
  console.log(`Imported ${response.imported} users`);
});

for (const user of userList) {
  stream.write(user);
}
stream.end();
```

### Bidirectional Streaming

Cả hai bên stream đồng thời — chat, real-time collaboration.

```javascript
// Server
chat: (call) => {
  call.on('data', (message) => {
    // Broadcast to all connected clients
    call.write({
      ...message,
      timestamp: Date.now(),
    });
  });

  call.on('end', () => {
    call.end();
  });
},
```

---

## Error Handling và Status Codes

### gRPC Status Codes

| Code | Number | Ý Nghĩa | HTTP Equivalent |
| ---- | ------ | ------- | --------------- |
| `OK` | 0 | Success | 200 |
| `INVALID_ARGUMENT` | 3 | Bad request | 400 |
| `NOT_FOUND` | 5 | Resource not found | 404 |
| `ALREADY_EXISTS` | 6 | Duplicate | 409 |
| `PERMISSION_DENIED` | 7 | Forbidden | 403 |
| `UNAUTHENTICATED` | 16 | Unauthorized | 401 |
| `INTERNAL` | 13 | Server error | 500 |
| `UNAVAILABLE` | 14 | Service down | 503 |
| `DEADLINE_EXCEEDED` | 4 | Timeout | 504 |

```javascript
const { status } = require('@grpc/grpc-js');

// Throw gRPC error
function notFound(resource) {
  return {
    code: status.NOT_FOUND,
    message: `${resource} not found`,
  };
}

// Client error handling
client.getUser({ id: 'invalid' }, (err, response) => {
  if (err) {
    switch (err.code) {
      case grpc.status.NOT_FOUND:
        console.log('User not found');
        break;
      case grpc.status.UNAVAILABLE:
        console.log('Service unavailable, retry...');
        break;
      default:
        console.error('Error:', err.details);
    }
  }
});
```

---

## Interceptors và Metadata

### Server Interceptor — Logging & Auth

```javascript
function authInterceptor(methodDescriptor, call) {
  const metadata = call.metadata;
  const token = metadata.get('authorization')[0]?.replace('Bearer ', '');

  if (!token) {
    call.emit('error', {
      code: grpc.status.UNAUTHENTICATED,
      message: 'Missing auth token',
    });
    return;
  }

  try {
    call.user = verifyJwt(token);
  } catch {
    call.emit('error', {
      code: grpc.status.UNAUTHENTICATED,
      message: 'Invalid token',
    });
  }
}

const server = new grpc.Server({
  interceptors: [authInterceptor],
});
```

### Client Interceptor — Retry

```javascript
const { retryInterceptor } = require('@grpc/grpc-js');

const client = new userProto.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure(),
  {
    interceptors: [
      retryInterceptor({
        maxAttempts: 3,
        initialBackoff: '0.1s',
        maxBackoff: '1s',
        backoffMultiplier: 2,
        retryableStatusCodes: [
          grpc.status.UNAVAILABLE,
          grpc.status.DEADLINE_EXCEEDED,
        ],
      }),
    ],
  }
);
```

---

## gRPC với NestJS

NestJS hỗ trợ gRPC transport natively — phù hợp microservices.

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.GRPC,
      options: {
        package: 'user',
        protoPath: join(__dirname, 'user/user.proto'),
        url: '0.0.0.0:50051',
      },
    }
  );

  await app.listen();
}
bootstrap();
```

```typescript
// user.controller.ts
import { Controller } from '@nestjs/common';
import { GrpcMethod } from '@nestjs/microservices';

@Controller()
export class UserController {
  constructor(private userService: UserService) {}

  @GrpcMethod('UserService', 'GetUser')
  async getUser(data: { id: string }) {
    return { user: await this.userService.findById(data.id) };
  }

  @GrpcMethod('UserService', 'ListUsers')
  async listUsers(data: { page: number; page_size: number }) {
    const result = await this.userService.findAll(data);
    return { users: result.items, total: result.total };
  }
}
```

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Version .proto files | Breaking changes cần versioning strategy |
| Set deadlines/timeouts | Prevent hanging requests |
| Use TLS in production | Encrypt traffic, mTLS cho service auth |
| Health checking | gRPC health check protocol |
| Load balancing | Client-side LB hoặc service mesh (Istio) |
| Keep messages small | Large messages → streaming |
| Idempotent operations | Safe retries |

### Deadline Example

```javascript
const deadline = new Date();
deadline.setSeconds(deadline.getSeconds() + 5); // 5 second timeout

client.getUser({ id: '123' }, { deadline }, (err, response) => {
  if (err?.code === grpc.status.DEADLINE_EXCEEDED) {
    console.log('Request timed out');
  }
});
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
| ------- | ------------ |
| gRPC vs REST cho microservices? | gRPC: binary, HTTP/2, streaming, faster; REST: human-readable, browser-friendly |
| Protocol Buffers là gì? | Binary serialization format — nhỏ hơn JSON, parse nhanh hơn |
| 4 loại RPC trong gRPC? | Unary, Server streaming, Client streaming, Bidirectional streaming |
| gRPC hoạt động trên protocol gì? | HTTP/2 — multiplexing, header compression |
| Làm sao auth gRPC requests? | Metadata headers (JWT), mTLS certificates |
| gRPC browser support? | Cần gRPC-Web proxy (Envoy) — không native browser |
| Deadline trong gRPC? | Client-set timeout — server cancel nếu exceed |
