# WebSockets — Socket.io, ws và Real-time Broadcasting

> WebSockets cung cấp kênh giao tiếp **full-duplex (hai chiều)** persistent giữa client và server — lý tưởng cho chat, notifications, live dashboards, và collaborative applications.

## Mục Lục

1. [WebSockets vs HTTP Polling](#websockets-vs-http-polling)
2. [ws Library — Low-level](#ws-library--low-level)
3. [Socket.io — High-level Framework](#socketio--high-level-framework)
4. [Rooms và Namespaces](#rooms-và-namespaces)
5. [Authentication](#authentication)
6. [Scaling WebSockets](#scaling-websockets)
7. [Server-Sent Events (SSE)](#server-sent-events-sse)
8. [Error Handling và Reconnection](#error-handling-và-reconnection)
9. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## WebSockets vs HTTP Polling

| Approach | Latency | Bandwidth | Complexity | Direction |
| -------- | ------- | --------- | ---------- | --------- |
| **Short Polling** | Cao (interval) | Lãng phí | Thấp | Client → Server |
| **Long Polling** | Trung bình | Trung bình | Trung bình | Client → Server |
| **SSE** | Thấp | Hiệu quả | Thấp | Server → Client only |
| **WebSockets** | Rất thấp | Hiệu quả nhất | Cao hơn | Bidirectional |

```
HTTP Polling:
Client ──request──► Server
Client ◄──empty──── Server
Client ──request──► Server  (repeat every N seconds)
Client ◄──data───── Server

WebSocket:
Client ══handshake══► Server  (HTTP Upgrade)
Client ◄════════════► Server  (persistent connection)
       real-time both ways
```

**Chọn WebSockets khi:**
- Bidirectional communication (chat, gaming)
- Low latency critical (<100ms)
- High message frequency

**Chọn SSE khi:**
- Chỉ cần server → client push (notifications, live feeds)
- Muốn đơn giản hơn, HTTP-compatible

---

## ws Library — Low-level

`ws` là WebSocket implementation nhẹ, RFC 6455 compliant — không có fallback hay rooms built-in.

### Basic Server

```javascript
const { WebSocketServer } = require('ws');
const http = require('http');

const server = http.createServer();
const wss = new WebSocketServer({ server });

const clients = new Set();

wss.on('connection', (ws, req) => {
  const clientId = req.headers['sec-websocket-key'];
  clients.add(ws);

  console.log(`Client connected: ${clientId}, total: ${clients.size}`);

  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());
    console.log('Received:', message);

    // Broadcast to all clients
    broadcast({ type: 'message', payload: message });
  });

  ws.on('close', () => {
    clients.delete(ws);
    console.log(`Client disconnected, total: ${clients.size}`);
  });

  ws.on('error', (err) => {
    console.error('WebSocket error:', err);
  });

  // Send welcome message
  ws.send(JSON.stringify({ type: 'welcome', clientId }));
});

function broadcast(data) {
  const message = JSON.stringify(data);
  for (const client of clients) {
    if (client.readyState === client.OPEN) {
      client.send(message);
    }
  }
}

server.listen(3001, () => {
  console.log('WebSocket server on ws://localhost:3001');
});
```

### Client

```javascript
const WebSocket = require('ws');

const ws = new WebSocket('ws://localhost:3001');

ws.on('open', () => {
  ws.send(JSON.stringify({ type: 'chat', text: 'Hello!' }));
});

ws.on('message', (data) => {
  console.log('Received:', JSON.parse(data.toString()));
});
```

---

## Socket.io — High-level Framework

Socket.io thêm: auto-reconnection, rooms, namespaces, fallback transports (long-polling), binary support.

### Server Setup với Express

```javascript
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: { origin: 'http://localhost:3000' },
  pingTimeout: 60000,
  pingInterval: 25000,
});

io.on('connection', (socket) => {
  console.log(`Connected: ${socket.id}`);

  socket.on('join-room', (roomId) => {
    socket.join(roomId);
    socket.to(roomId).emit('user-joined', { userId: socket.id });
  });

  socket.on('chat-message', ({ roomId, text, userId }) => {
    io.to(roomId).emit('chat-message', {
      text,
      userId,
      timestamp: Date.now(),
    });
  });

  socket.on('disconnect', (reason) => {
    console.log(`Disconnected: ${socket.id}, reason: ${reason}`);
  });
});

httpServer.listen(3001);
```

### Client (Browser)

```javascript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3001', {
  auth: { token: 'jwt-token-here' },
  reconnection: true,
  reconnectionAttempts: 5,
  reconnectionDelay: 1000,
});

socket.on('connect', () => {
  socket.emit('join-room', 'room-123');
});

socket.on('chat-message', (msg) => {
  console.log(`${msg.userId}: ${msg.text}`);
});

socket.emit('chat-message', {
  roomId: 'room-123',
  text: 'Hello everyone!',
  userId: 'user-1',
});
```

---

## Rooms và Namespaces

### Rooms — Group Connections

```javascript
// Join multiple rooms
socket.join('room-1');
socket.join('room-2');

// Emit to specific room (excluding sender)
socket.to('room-1').emit('event', data);

// Emit to room including sender
io.in('room-1').emit('event', data);

// Leave room
socket.leave('room-1');

// Get room members
const room = io.sockets.adapter.rooms.get('room-1');
console.log(`Room size: ${room?.size}`);
```

### Namespaces — Logical Separation

```javascript
// /chat namespace
const chatNamespace = io.of('/chat');
chatNamespace.on('connection', (socket) => {
  socket.on('message', (msg) => {
    chatNamespace.emit('message', msg);
  });
});

// /notifications namespace
const notifNamespace = io.of('/notifications');
notifNamespace.on('connection', (socket) => {
  socket.join(`user-${socket.userId}`);
});
```

```
Default namespace (/)
├── Room: general
├── Room: support

/chat namespace
├── Room: room-1
├── Room: room-2

/notifications namespace
├── Room: user-123
├── Room: user-456
```

---

## Authentication

### JWT Authentication Middleware

```javascript
const jwt = require('jsonwebtoken');

io.use((socket, next) => {
  const token = socket.handshake.auth.token;

  if (!token) {
    return next(new Error('Authentication required'));
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = decoded.sub;
    socket.userRole = decoded.role;
    next();
  } catch (err) {
    next(new Error('Invalid token'));
  }
});

io.on('connection', (socket) => {
  // Auto-join user's personal room
  socket.join(`user:${socket.userId}`);

  // Send notification only to specific user
  // io.to(`user:${userId}`).emit('notification', data);
});
```

### Authorization trong Event Handlers

```javascript
socket.on('delete-message', async ({ messageId, roomId }) => {
  const message = await db.message.findUnique({ where: { id: messageId } });

  if (message.authorId !== socket.userId && socket.userRole !== 'admin') {
    socket.emit('error', { message: 'Unauthorized' });
    return;
  }

  await db.message.delete({ where: { id: messageId } });
  io.to(roomId).emit('message-deleted', { messageId });
});
```

---

## Scaling WebSockets

WebSocket connections **stateful** — cần strategy khi scale multiple instances.

### Problem

```
Load Balancer
     │
  ┌──┴──┐
  ▼     ▼
App-1  App-2
  │     │
User-A  User-B  ← User-A gửi message, User-B không nhận được (khác instance)
```

### Solution: Redis Adapter

```javascript
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(createAdapter(pubClient, subClient));

// Giờ io.to('room-1').emit() broadcast across ALL instances
```

```
Load Balancer (sticky sessions recommended)
     │
  ┌──┴──┐
  ▼     ▼
App-1  App-2
  │     │
  └──┬──┘
     ▼
  Redis Pub/Sub  ← Sync events between instances
```

### Sticky Sessions

```nginx
# Nginx config
upstream websocket {
    ip_hash;  # Sticky sessions
    server app1:3001;
    server app2:3001;
}
```

---

## Server-Sent Events (SSE)

SSE (Server-Sent Events — Sự Kiện Gửi Từ Server) — HTTP-based, server push only, đơn giản hơn WebSockets.

```javascript
const express = require('express');
const app = express();

const clients = new Set();

app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders();

  const clientId = Date.now();
  clients.add(res);

  res.write(`data: ${JSON.stringify({ type: 'connected', clientId })}\n\n`);

  req.on('close', () => {
    clients.delete(res);
  });
});

function broadcastEvent(event, data) {
  const message = `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`;
  for (const client of clients) {
    client.write(message);
  }
}

// Trigger từ business logic
broadcastEvent('order-updated', { orderId: '123', status: 'shipped' });
```

### Client

```javascript
const eventSource = new EventSource('/events');

eventSource.addEventListener('order-updated', (e) => {
  const data = JSON.parse(e.data);
  console.log('Order updated:', data);
});

eventSource.onerror = () => {
  console.log('SSE connection error, will auto-reconnect');
};
```

---

## Error Handling và Reconnection

### Server-side Connection Management

```javascript
io.on('connection', (socket) => {
  socket.conn.on('packet', ({ type }) => {
  if (type === 'pong') {
    socket.lastPong = Date.now();
  }
});

  const heartbeat = setInterval(() => {
    if (Date.now() - socket.lastPong > 90000) {
      socket.disconnect(true);
    }
  }, 30000);

  socket.on('disconnect', () => {
    clearInterval(heartbeat);
  });
});
```

### Client Reconnection Strategy

```javascript
const socket = io('http://localhost:3001', {
  reconnection: true,
  reconnectionAttempts: Infinity,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
});

socket.on('connect', () => {
  // Re-join rooms after reconnect
  socket.emit('join-room', currentRoomId);
});

socket.on('disconnect', (reason) => {
  if (reason === 'io server disconnect') {
    socket.connect(); // Manual reconnect if server kicked
  }
});
```

---

## Ví Dụ Thực Tế

### Live Dashboard — Real-time Metrics

```javascript
const io = require('socket.io')(httpServer);

// Emit metrics mỗi 5 giây
setInterval(async () => {
  const metrics = await collectMetrics();
  io.to('dashboard').emit('metrics-update', metrics);
}, 5000);

io.on('connection', (socket) => {
  socket.on('subscribe-dashboard', () => {
    socket.join('dashboard');
  });
});
```

### Collaborative Document Editing

```javascript
const documentState = new Map();

socket.on('join-document', (docId) => {
  socket.join(`doc:${docId}`);

  if (!documentState.has(docId)) {
    documentState.set(docId, { content: '', version: 0 });
  }

  socket.emit('document-state', documentState.get(docId));
});

socket.on('edit', ({ docId, operation, version }) => {
  const doc = documentState.get(docId);

  if (version !== doc.version) {
    socket.emit('conflict', { currentVersion: doc.version });
    return;
  }

  doc.content = applyOperation(doc.content, operation);
  doc.version++;

  socket.to(`doc:${docId}`).emit('edit', { operation, version: doc.version });
});
```

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Authenticate at connection time | Không cho unauthenticated sockets vào |
| Validate message payload | Prevent malformed data, injection |
| Rate limit events per socket | Prevent abuse, DoS |
| Redis adapter cho multi-instance | Cross-instance broadcasting |
| Heartbeat/ping-pong | Detect dead connections |
| Graceful shutdown | Notify clients trước khi server restart |
| Message size limits | Prevent memory exhaustion |

### Rate Limiting Events

```javascript
const rateLimit = require('socket.io-rate-limit');

io.use(rateLimit({
  tokensPerInterval: 10,
  interval: 1000, // 10 messages per second per socket
}));
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
| ------- | ------------ |
| WebSockets vs HTTP polling? | WebSocket: persistent, low latency, bidirectional; Polling: simple, higher latency |
| Socket.io vs ws library? | Socket.io: rooms, fallback, reconnection; ws: lightweight, RFC compliant |
| Làm sao scale WebSockets? | Redis adapter + sticky sessions hoặc pub/sub |
| WebSockets vs SSE? | WebSocket: bidirectional; SSE: server→client only, HTTP-based, auto-reconnect |
| WebSocket authentication? | JWT trong handshake auth, middleware `io.use()` |
| Sticky sessions tại sao cần? | Stateful connection — client phải reconnect cùng instance (nếu không dùng Redis) |
| Heartbeat/ping-pong mục đích? | Detect dead/zombie connections, keep NAT timeouts alive |
