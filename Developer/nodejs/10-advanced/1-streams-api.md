# Streams API — Readable, Writable, Transform và Pipeline

> Streams (Luồng Dữ Liệu) là một trong những tính năng mạnh nhất của Node.js — cho phép xử lý dữ liệu lớn với memory footprint (dấu chân bộ nhớ) cố định thay vì load toàn bộ vào RAM.

## Mục Lục

1. [Tại Sao Cần Streams](#tại-sao-cần-streams)
2. [Các Loại Stream](#các-loại-stream)
3. [Readable Stream](#readable-stream)
4. [Writable Stream](#writable-stream)
5. [Transform Stream](#transform-stream)
6. [Duplex Stream](#duplex-stream)
7. [Backpressure — Áp Lực Ngược](#backpressure--áp-lực-ngược)
8. [pipeline() và stream/promises](#pipeline-và-streampromises)
9. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Streams

### Vấn Đề: Đọc File Lớn Sync

```javascript
const fs = require('fs');

// ❌ Load toàn bộ 2GB file vào memory
const data = fs.readFileSync('./large-file.csv');
console.log(data.length); // ~2GB RAM consumed
```

### Giải Pháp: Streams

```javascript
const fs = require('fs');

// ✅ Xử lý từng chunk ~64KB
const stream = fs.createReadStream('./large-file.csv');

stream.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes`);
  // Memory usage: ~64KB constant
});
```

| Approach | File 2GB | Memory Usage |
| -------- | -------- | ------------ |
| `readFileSync()` | Load all | ~2GB |
| `createReadStream()` | Chunk by chunk | ~64KB |

```
Traditional I/O:
┌──────────┐     ┌──────────┐     ┌──────────┐
│   Disk   │────►│   RAM    │────►│ Process  │
│  (2GB)   │     │  (2GB)   │     │          │
└──────────┘     └──────────┘     └──────────┘

Stream I/O:
┌──────────┐     ┌──────────┐     ┌──────────┐
│   Disk   │─64KB►│  Buffer  │─64KB►│ Process  │
│  (2GB)   │     │  (64KB)  │     │ (chunk)  │
└──────────┘     └──────────┘     └──────────┘
      ▲                                  │
      └──────── next chunk ──────────────┘
```

---

## Các Loại Stream

Node.js có 4 loại stream cơ bản:

| Loại | Mô Tả | Ví Dụ |
| ---- | ----- | ----- |
| **Readable** | Nguồn dữ liệu — đọc được | `fs.createReadStream`, `http.IncomingMessage` |
| **Writable** | Đích dữ liệu — ghi được | `fs.createWriteStream`, `http.ServerResponse` |
| **Duplex** | Vừa đọc vừa ghi độc lập | TCP socket, `net.Socket` |
| **Transform** | Duplex + transform data khi truyền | `zlib.createGzip`, custom encryption |

```
Readable ──► Transform ──► Writable
   │              │              │
 Read file    Gzip compress   Write to disk
```

---

## Readable Stream

### Tạo Readable Stream

```javascript
const { Readable } = require('stream');

// Object mode — stream objects thay vì buffers
const objectStream = new Readable({
  objectMode: true,
  read() {
    this.push({ id: 1, name: 'Alice' });
    this.push({ id: 2, name: 'Bob' });
    this.push(null); // Signal end of stream
  },
});
```

### Đọc với Async Iterator (Node.js 10+)

```javascript
const fs = require('fs');

async function processFile(filePath) {
  const stream = fs.createReadStream(filePath, {
  encoding: 'utf8',
  highWaterMark: 16 * 1024, // 16KB chunks (default: 64KB)
});

  for await (const chunk of stream) {
    console.log(`Processing chunk: ${chunk.length} chars`);
  }
}
```

### Readable Events

| Event | Khi Nào Fire |
| ----- | ------------ |
| `data` | Có chunk mới (switch sang flowing mode) |
| `end` | Không còn data |
| `error` | Lỗi xảy ra |
| `close` | Stream đóng (có thể trước `end` nếu abort) |
| `readable` | Có data trong buffer, đọc bằng `read()` (paused mode) |

### Flowing vs Paused Mode

```javascript
// Flowing mode — data tự động emit
stream.on('data', (chunk) => { /* ... */ });

// Paused mode — đọc thủ công
stream.on('readable', () => {
  let chunk;
  while ((chunk = stream.read()) !== null) {
    console.log(chunk);
  }
});
```

---

## Writable Stream

### Tạo Writable Stream

```javascript
const { Writable } = require('stream');

const logStream = new Writable({
  write(chunk, encoding, callback) {
    console.log(`LOG: ${chunk.toString()}`);
    callback(); // Signal write complete
  },
});

logStream.write('Hello\n');
logStream.write('World\n');
logStream.end(); // Signal no more writes
```

### Ghi File với Backpressure Handling

```javascript
const fs = require('fs');

function writeLargeData(filePath, dataGenerator) {
  const writeStream = fs.createWriteStream(filePath);

  function writeNext() {
    let chunk;
    let canContinue = true;

    while (canContinue && (chunk = dataGenerator.next().value) !== undefined) {
      canContinue = writeStream.write(chunk);
    }

    if (chunk === undefined) {
      writeStream.end();
      return;
    }

    if (!canContinue) {
      // Buffer đầy — đợi drain event
      writeStream.once('drain', writeNext);
    }
  }

  writeNext();
}
```

### Writable Events

| Event | Khi Nào Fire |
| ----- | ------------ |
| `drain` | Buffer trống, có thể write tiếp |
| `finish` | `end()` được gọi, tất cả data đã flush |
| `error` | Lỗi khi write |
| `close` | Stream đóng |

---

## Transform Stream

Transform stream nhận input, biến đổi, rồi output — lý tưởng cho encryption, compression, parsing.

```javascript
const { Transform } = require('stream');

const upperCaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  },
});

process.stdin.pipe(upperCaseTransform).pipe(process.stdout);
```

### CSV Line Parser Transform

```javascript
const { Transform } = require('stream');

class CsvLineParser extends Transform {
  constructor(options) {
    super({ ...options, objectMode: true });
    this.buffer = '';
  }

  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop(); // Giữ incomplete line

    for (const line of lines) {
      if (line.trim()) {
        const [id, name, email] = line.split(',');
        this.push({ id, name, email });
      }
    }
    callback();
  }

  _flush(callback) {
    if (this.buffer.trim()) {
      const [id, name, email] = this.buffer.split(',');
      this.push({ id, name, email });
    }
    callback();
  }
}
```

---

## Duplex Stream

Duplex stream có 2 channel độc lập — readable side và writable side không liên quan trực tiếp.

```javascript
const { Duplex } = require('stream');

const echoDuplex = new Duplex({
  write(chunk, encoding, callback) {
    // Writable side: nhận input
    console.log('Received:', chunk.toString());
    callback();
  },
  read() {
    // Readable side: push output
    this.push(`Echo: ${this.lastChunk}`);
  },
});
```

**Ví dụ thực tế:** TCP socket (`net.Socket`) — gửi và nhận data độc lập trên cùng connection.

---

## Backpressure — Áp Lực Ngược

Backpressure xảy ra khi **producer (nguồn)** nhanh hơn **consumer (người tiêu thụ)** — buffer đầy, cần pause producer.

```
Fast Producer ──► [Buffer 16KB] ──► Slow Consumer
                        │
                        ▼ FULL
              Producer PAUSED (backpressure)
                        │
                        ▼ drain event
              Producer RESUMED
```

### Cơ Chế Backpressure

```javascript
const readable = fs.createReadStream('input.txt');
const writable = fs.createWriteStream('output.txt');

readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);

  if (!canContinue) {
    // Buffer đầy — pause readable
    readable.pause();
    writable.once('drain', () => {
      readable.resume();
    });
  }
});

readable.on('end', () => writable.end());
```

**`pipe()` tự động xử lý backpressure** — lý do nên dùng `pipe()` hoặc `pipeline()` thay vì manual event handling.

---

## pipeline() và stream/promises

### pipeline() — Error Handling Tốt Hơn pipe()

```javascript
const { pipeline } = require('stream');
const fs = require('fs');
const zlib = require('zlib');
const { promisify } = require('util');

const pipelineAsync = promisify(pipeline);

async function compressFile(input, output) {
  try {
    await pipelineAsync(
      fs.createReadStream(input),
      zlib.createGzip(),
      fs.createWriteStream(output)
    );
    console.log('Compression complete');
  } catch (err) {
    console.error('Pipeline failed:', err);
    // pipeline tự động destroy streams khi lỗi
  }
}
```

### stream/promises (Node.js 15+)

```javascript
const { pipeline } = require('stream/promises');
const fs = require('fs');
const zlib = require('zlib');

async function compressFile(input, output) {
  await pipeline(
    fs.createReadStream(input),
    zlib.createGzip(),
    fs.createWriteStream(output)
  );
}
```

| Method | Error Handling | Promise Support |
| ------ | -------------- | --------------- |
| `pipe()` | Manual, streams có thể leak | Không |
| `pipeline()` (callback) | Tự destroy streams | Qua promisify |
| `stream/promises.pipeline()` | Tự destroy streams | Native async/await |

---

## Ví Dụ Thực Tế

### 1. HTTP Response Streaming

```javascript
const express = require('express');
const fs = require('fs');

const app = express();

app.get('/download/:file', (req, res) => {
  const filePath = `./files/${req.params.file}`;

  res.setHeader('Content-Type', 'application/octet-stream');
  res.setHeader('Content-Disposition', `attachment; filename="${req.params.file}"`);

  const stream = fs.createReadStream(filePath);
  stream.pipe(res);

  stream.on('error', (err) => {
    res.status(404).json({ error: 'File not found' });
  });
});
```

### 2. Gzip Compression Pipeline

```javascript
const { pipeline } = require('stream/promises');
const fs = require('fs');
const zlib = require('zlib');
const { createCipheriv, randomBytes, scryptSync } = require('crypto');

async function secureArchive(inputPath, outputPath, password) {
  const key = scryptSync(password, 'salt', 32);
  const iv = randomBytes(16);
  const cipher = createCipheriv('aes-256-gcm', key, iv);

  await pipeline(
    fs.createReadStream(inputPath),
    cipher,
    zlib.createGzip(),
    fs.createWriteStream(outputPath)
  );
}
```

### 3. Parse Large JSON Lines File

```javascript
const { createReadStream } = require('fs');
const { createInterface } = require('readline');

async function processJsonLines(filePath) {
  const rl = createInterface({
    input: createReadStream(filePath),
    crlfDelay: Infinity,
  });

  for await (const line of rl) {
    const record = JSON.parse(line);
    await processRecord(record); // Insert to DB, etc.
  }
}
```

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Dùng `pipeline()` thay `pipe()` | Error handling + cleanup tự động |
| Set `highWaterMark` phù hợp | Balance memory vs throughput |
| Luôn handle `error` event | Unhandled stream errors crash process |
| Dùng `objectMode` cho structured data | Stream objects thay vì parse buffer manually |
| Prefer async iterator (`for await`) | Cleaner code, tự handle backpressure |
| Destroy streams khi abort | `stream.destroy()` giải phóng resources |

### Anti-patterns

```javascript
// ❌ Không handle error
readable.pipe(writable);

// ✅ Dùng pipeline
await pipeline(readable, writable);

// ❌ Accumulate chunks trong memory
const chunks = [];
stream.on('data', (chunk) => chunks.push(chunk));
stream.on('end', () => process(Buffer.concat(chunks))); // defeats purpose!

// ✅ Process từng chunk
for await (const chunk of stream) {
  await processChunk(chunk);
}
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
| ------- | ------------ |
| Streams giải quyết vấn đề gì? | Xử lý data lớn với memory constant — không load all vào RAM |
| 4 loại stream trong Node.js? | Readable, Writable, Duplex, Transform |
| Backpressure là gì? | Cơ chế pause producer khi consumer buffer đầy |
| `pipe()` vs `pipeline()`? | `pipeline()` có error handling, tự destroy streams khi lỗi |
| `highWaterMark` là gì? | Buffer size threshold — khi đầy trigger backpressure |
| Khi nào dùng `objectMode`? | Stream JavaScript objects thay vì Buffer/string chunks |
| HTTP request/response là stream gì? | `IncomingMessage` = Readable, `ServerResponse` = Writable |
