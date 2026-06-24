# Built-in Modules — Module Lõi Node.js

> Node.js cung cấp nhiều built-in modules (module tích hợp sẵn) không cần cài qua npm. Nắm vững `fs`, `path`, `http`, `crypto`, `events`, và `buffer` là nền tảng trước khi dùng framework.

## Mục Lục

1. [Tổng Quan Built-in Modules](#tổng-quan-built-in-modules)
2. [path — Xử Lý Đường Dẫn](#path--xử-lý-đường-dẫn)
3. [fs — File System](#fs--file-system)
4. [os — Operating System](#os--operating-system)
5. [http / https — HTTP Server](#http--https--http-server)
6. [events — EventEmitter](#events--eventemitter)
7. [buffer — Binary Data](#buffer--binary-data)
8. [crypto — Mã Hóa](#crypto--mã-hóa)
9. [util — Utilities](#util--utilities)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Built-in Modules

```javascript
// CommonJS
const fs = require('fs');
const path = require('path');
const http = require('http');
const crypto = require('crypto');
const { EventEmitter } = require('events');

// ESM
import fs from 'fs/promises';  // Promise-based API (khuyến nghị)
import path from 'path';
import http from 'http';
import crypto from 'crypto';
import { EventEmitter } from 'events';
```

### Phân Loại

| Module | Mục Đích | Sync API? |
| ------ | -------- | --------- |
| `path` | Đường dẫn file (cross-platform) | Có (pure functions) |
| `fs` | Đọc/ghi file, directories | Có (tránh trong production) |
| `os` | Thông tin hệ điều hành | Có |
| `http`/`https` | HTTP server và client | Async |
| `events` | Event-driven programming | Async |
| `buffer` | Binary data | Sync |
| `crypto` | Hash, encrypt, random bytes | Cả hai |
| `url` | Parse và format URL | Sync |
| `stream` | Streaming data | Async |

---

## path — Xử Lý Đường Dẫn

`path` module xử lý đường dẫn file **cross-platform** (Windows dùng `\`, Linux/macOS dùng `/`).

```javascript
import path from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// Join paths — tự động dùng separator đúng OS
path.join('/users', 'docs', 'file.txt');
// Linux: '/users/docs/file.txt'
// Windows: '\users\docs\file.txt'

// Resolve — absolute path
path.resolve('src', 'config', 'app.json');
// '/current/working/dir/src/config/app.json'

// Parse và format
path.parse('/home/user/file.txt');
// { root: '/', dir: '/home/user', base: 'file.txt', ext: '.txt', name: 'file' }

path.format({ dir: '/home/user', base: 'file.txt' });
// '/home/user/file.txt'

// Basename, dirname, extname
path.basename('/path/to/file.txt');     // 'file.txt'
path.dirname('/path/to/file.txt');      // '/path/to'
path.extname('/path/to/file.txt');      // '.txt'

// Relative path giữa hai paths
path.relative('/data/oracle', '/data/mysql/backup'); // '../mysql/backup'

// Normalize — loại bỏ redundant separators và `.` segments
path.normalize('/users/../admin/./config'); // '/admin/config'
```

**Best practice:** Luôn dùng `path.join()` thay vì string concatenation với `/`.

---

## fs — File System

Node.js cung cấp 3 style API: callback, sync, và promise.

### Promise-based API (Khuyến Nghị)

```javascript
import fs from 'fs/promises';
import path from 'path';

// Đọc file
const content = await fs.readFile('config.json', 'utf-8');
const config = JSON.parse(content);

// Ghi file
await fs.writeFile('output.txt', 'Hello World', 'utf-8');

// Append
await fs.appendFile('logs/app.log', `${new Date().toISOString()} - Started\n`);

// Kiểm tra tồn tại
try {
  await fs.access('config.json');
  console.log('File exists');
} catch {
  console.log('File not found');
}

// File stats
const stats = await fs.stat('app.js');
stats.isFile();      // true
stats.isDirectory(); // false
stats.size;          // bytes
stats.mtime;         // last modified Date

// Directory operations
await fs.mkdir('uploads', { recursive: true });
const files = await fs.readdir('./src');
await fs.rename('old.txt', 'new.txt');
await fs.copyFile('source.txt', 'dest.txt');
await fs.unlink('temp.txt');           // delete file
await fs.rm('temp-dir', { recursive: true }); // delete directory
```

### Callback API (Legacy)

```javascript
const fs = require('fs');

fs.readFile('data.txt', 'utf-8', (err, data) => {
  if (err) {
    console.error('Error:', err.message);
    return;
  }
  console.log(data);
});
```

### Sync API — Tránh Trong Production

```javascript
// BLOCKS event loop — chỉ dùng startup scripts, CLI tools
const data = fs.readFileSync('config.json', 'utf-8');
```

### Watch File Changes

```javascript
import fs from 'fs';

const watcher = fs.watch('./config.json', (eventType, filename) => {
  console.log(`File ${filename} changed: ${eventType}`);
});

// Cleanup
watcher.close();
```

---

## os — Operating System

```javascript
import os from 'os';

os.platform();        // 'linux', 'darwin', 'win32'
os.arch();            // 'x64', 'arm64'
os.cpus();            // CPU info array
os.totalmem();        // Total RAM bytes
os.freemem();         // Free RAM bytes
os.hostname();        // Machine hostname
os.homedir();         // User home directory
os.tmpdir();          // Temp directory path
os.uptime();          // System uptime seconds
os.networkInterfaces(); // Network interfaces

// Load average (1, 5, 15 minutes)
os.loadavg();         // [0.5, 0.3, 0.2]

// EOL (End of Line) character
os.EOL;               // '\n' on Linux, '\r\n' on Windows
```

**Use case:** Health check endpoint báo cáo system metrics, dynamic worker count dựa trên CPU cores.

```javascript
import os from 'os';
import cluster from 'cluster';

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  for (let i = 0; i < numCPUs; i++) cluster.fork();
}
```

---

## http / https — HTTP Server

### Tạo HTTP Server Cơ Bản

```javascript
import http from 'http';
import fs from 'fs/promises';
import path from 'path';

const server = http.createServer(async (req, res) => {
  const { method, url } = req;

  // Routing đơn giản
  if (method === 'GET' && url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', uptime: process.uptime() }));
    return;
  }

  if (method === 'GET' && url === '/users') {
    const data = await fs.readFile('./users.json', 'utf-8');
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(data);
    return;
  }

  res.writeHead(404, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ error: 'Not Found' }));
});

server.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

### Đọc Request Body

```javascript
function readBody(req) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    req.on('data', chunk => chunks.push(chunk));
    req.on('end', () => resolve(Buffer.concat(chunks).toString()));
    req.on('error', reject);
  });
}

server.on('request', async (req, res) => {
  if (req.method === 'POST') {
    const body = await readBody(req);
    const data = JSON.parse(body);
    // process data...
  }
});
```

### HTTP Client

```javascript
import http from 'http';

// Callback style
http.get('http://api.example.com/users', (res) => {
  let data = '';
  res.on('data', chunk => data += chunk);
  res.on('end', () => console.log(JSON.parse(data)));
});

// Modern: dùng fetch (Node.js 18+ built-in)
const response = await fetch('https://api.example.com/users');
const users = await response.json();
```

**Production:** Dùng Express/Fastify thay vì raw `http` module — nhưng hiểu `http` giúp hiểu framework hoạt động thế nào.

---

## events — EventEmitter

EventEmitter là pattern cốt lõi của Node.js — nhiều built-in objects kế thừa từ nó.

```javascript
import { EventEmitter } from 'events';

const emitter = new EventEmitter();

// Subscribe
emitter.on('user:created', (user) => {
  console.log('New user:', user.name);
});

emitter.once('server:ready', () => {
  console.log('Server started — chỉ fire một lần');
});

// Emit
emitter.emit('user:created', { id: 1, name: 'Alice' });

// Error handling — PHẢI có listener cho 'error'
emitter.on('error', (err) => {
  console.error('Emitter error:', err.message);
});

// Remove listener
const handler = (data) => console.log(data);
emitter.on('data', handler);
emitter.off('data', handler);  // hoặc removeListener
```

### Custom EventEmitter Class

```javascript
import { EventEmitter } from 'events';

class OrderService extends EventEmitter {
  async createOrder(items) {
    const order = { id: Date.now(), items, status: 'pending' };
    await this.saveToDb(order);
    this.emit('order:created', order);
    return order;
  }

  async saveToDb(order) { /* ... */ }
}

const orderService = new OrderService();
orderService.on('order:created', (order) => {
  // Send notification, update inventory...
});
```

### EventEmitter trong Node.js Internals

```
http.Server extends EventEmitter  → 'request', 'connection', 'close'
fs.ReadStream extends EventEmitter → 'data', 'end', 'error'
process extends EventEmitter      → 'exit', 'SIGTERM', 'uncaughtException'
```

### Max Listeners Warning

```javascript
import { EventEmitter } from 'events';
EventEmitter.defaultMaxListeners = 20; // Default: 10
```

---

## buffer — Binary Data

Buffer là fixed-size chunk of memory để xử lý binary data — streams, file I/O, network protocols.

```javascript
// Tạo Buffer
const buf1 = Buffer.alloc(10);           // 10 bytes, zero-filled
const buf2 = Buffer.from('Hello');       // từ string
const buf3 = Buffer.from([0x48, 0x65]);  // từ array

// Đọc/ghi
buf2.toString('utf-8');    // 'Hello'
buf2.toString('hex');      // '48656c6c6f'
buf2.toString('base64');   // 'SGVsbG8='

buf2.length;               // 5 (bytes)
buf2[0];                   // 72 (ASCII 'H')

// Concat buffers
const combined = Buffer.concat([buf2, Buffer.from(' World')]);

// Compare
Buffer.from('abc').equals(Buffer.from('abc')); // true

// Slice — shared memory (cẩn thận mutation)
const slice = buf2.subarray(0, 2); // <Buffer 48 65>
```

### Buffer vs String

| Khía Cạnh | String | Buffer |
| --------- | ------ | ------ |
| Encoding | UTF-16 internally | Raw bytes |
| Use case | Text data | Binary, streams, crypto |
| Immutable | Yes (JS strings) | Mutable |

**Lưu ý:** `Buffer.alloc()` an toàn hơn `Buffer.allocUnsafe()` — unsafe có thể chứa data cũ từ memory.

---

## crypto — Mã Hóa

```javascript
import crypto from 'crypto';

// Hash (one-way — một chiều)
const hash = crypto
  .createHash('sha256')
  .update('password123')
  .digest('hex');
// 'ef92b778bafe771e89245b89ecbc08a44a4e166c06659911881f383d4473e94f'

// HMAC (Hash-based Message Authentication Code)
const hmac = crypto
  .createHmac('sha256', 'secret-key')
  .update('message to sign')
  .digest('hex');

// Random bytes — tokens, IDs
const token = crypto.randomBytes(32).toString('hex');
const uuid = crypto.randomUUID(); // Node.js 16+

// Password hashing — dùng bcrypt/argon2 trong production
// crypto.pbkdf2 là built-in alternative
const salt = crypto.randomBytes(16).toString('hex');
crypto.pbkdf2('password', salt, 100000, 64, 'sha512', (err, derivedKey) => {
  const hash = derivedKey.toString('hex');
});

// AES encryption (symmetric — đối xứng)
const algorithm = 'aes-256-gcm';
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);

const cipher = crypto.createCipheriv(algorithm, key, iv);
let encrypted = cipher.update('secret data', 'utf8', 'hex');
encrypted += cipher.final('hex');
const authTag = cipher.getAuthTag(); // GCM authentication tag
```

**Production password hashing:** Dùng `bcrypt` hoặc `argon2` package — không dùng plain SHA-256.

```javascript
import bcrypt from 'bcrypt';

const hash = await bcrypt.hash('password', 12);
const isValid = await bcrypt.compare('password', hash);
```

---

## util — Utilities

```javascript
import util from 'util';
import fs from 'fs';

// Promisify — chuyển callback API sang Promise
const readFile = util.promisify(fs.readFile);
const content = await readFile('data.txt', 'utf-8');

// util.format — printf-style
util.format('%s:%s', 'foo', 'bar'); // 'foo:bar'

// util.inspect — debug objects
console.log(util.inspect(complexObject, { depth: 3, colors: true }));

// util.parseArgs — CLI argument parsing (Node.js 18+)
import { parseArgs } from 'util';
const { values } = parseArgs({
  options: {
    port: { type: 'string', default: '3000' },
    env: { type: 'string' },
  },
});
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: `fs.readFile` vs `fs.readFileSync` — khi nào dùng gì?

**Gợi ý trả lời:** `readFile` (async) không block Event Loop — dùng trong production request handlers. `readFileSync` block main thread — chỉ dùng startup (đọc config lúc khởi động) hoặc CLI scripts. Prefer `fs/promises` với async/await.

### Câu 2: EventEmitter memory leak — nguyên nhân và cách tránh?

**Gợi ý trả lời:** Thêm listeners mà không remove khi object bị destroy → memory leak. Triệu chứng: `MaxListenersExceededWarning`. Giải pháp: `emitter.off()` khi cleanup, dùng `once()` cho one-time events, hoặc `emitter.removeAllListeners()` khi destroy.

### Câu 3: Tại sao không hash password bằng SHA-256?

**Gợi ý trả lời:** SHA-256 quá nhanh — attacker có thể brute-force hàng tỷ hashes/giây với GPU. Password hashing cần slow algorithm có salt: bcrypt, scrypt, hoặc argon2. Chúng có work factor (cost) điều chỉnh được.

### Câu 4: Buffer và Stream khác nhau thế nào?

**Gợi ý trả lời:** Buffer là fixed-size chunk memory — load toàn bộ data vào RAM. Stream xử lý data theo chunks qua thời gian — phù hợp file lớn, video, HTTP response lớn. Stream tiết kiệm memory nhờ **backpressure (áp lực ngược)** — consumer điều tiết tốc độ producer.

### Câu 5: `path.join` vs `path.resolve`?

**Gợi ý trả lời:** `join` nối path segments với separator đúng OS — kết quả có thể relative. `resolve` tạo absolute path từ right-to-left, kết hợp với cwd nếu cần. Dùng `join` cho relative paths trong project, `resolve` khi cần absolute path chắc chắn.

---

**Hoàn thành chủ đề nền tảng.** Tiếp theo: [02-async-programming/](../02-async-programming/) — Event Loop và lập trình bất đồng bộ.
