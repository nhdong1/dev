# Child Processes — spawn, exec, fork và IPC

> Module `child_process` cho phép Node.js chạy các lệnh hệ điều hành (OS commands), shell scripts, và tiến trình con — mở rộng khả năng ngoài JavaScript runtime đơn luồng.

## Mục Lục

1. [Tại Sao Cần Child Processes](#tại-sao-cần-child-processes)
2. [spawn vs exec vs execFile vs fork](#spawn-vs-exec-vs-excfile-vs-fork)
3. [spawn — Streaming Output](#spawn--streaming-output)
4. [exec — Shell Commands](#exec--shell-commands)
5. [fork — Node.js Child Process](#fork--nodejs-child-process)
6. [IPC — Inter-Process Communication](#ipc--inter-process-communication)
7. [Error Handling và Exit Codes](#error-handling-và-exit-codes)
8. [Child Process vs Worker Threads](#child-process-vs-worker-threads)
9. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Child Processes

Node.js single-threaded — không phù hợp cho:

| Use Case | Giải Pháp |
| -------- | --------- |
| Chạy shell commands (`git`, `ffmpeg`, `imagemagick`) | `spawn` / `exec` |
| CPU-intensive isolation | `fork` hoặc Worker Threads |
| Chạy legacy scripts (Python, Ruby) | `spawn` với interpreter |
| Parallel processing với process isolation | Multiple `fork` |

```
┌─────────────────────────────────────────┐
│           Node.js Main Process           │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ spawn   │  │  fork   │  │  exec   │ │
│  │ (CLI)   │  │ (Node)  │  │ (shell) │ │
│  └────┬────┘  └────┬────┘  └────┬────┘ │
│       │            │            │       │
└───────┼────────────┼────────────┼───────┘
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ ffmpeg  │  │ worker  │  │  bash   │
   │ process │  │ .js     │  │  -c     │
   └─────────┘  └─────────┘  └─────────┘
```

---

## spawn vs exec vs execFile vs fork

| Method | Shell | Buffer Limit | Streaming | Use Case |
| ------ | ----- | ------------ | --------- | -------- |
| `spawn` | Tùy chọn (`shell: true`) | Không | ✅ stdout/stderr streams | Long-running, large output |
| `exec` | Luôn dùng shell | 1MB default (maxBuffer) | ❌ Buffer all output | Short commands, shell features |
| `execFile` | Không shell | 1MB default | ❌ Buffer all output | Direct executable, no shell injection risk |
| `fork` | Không | N/A | IPC channel | Node.js child với message passing |

---

## spawn — Streaming Output

Phù hợp cho commands có output lớn hoặc chạy lâu (video encoding, log tailing).

```javascript
const { spawn } = require('child_process');

function runCommand(command, args, options = {}) {
  return new Promise((resolve, reject) => {
    const child = spawn(command, args, {
      stdio: ['pipe', 'pipe', 'pipe'],
      ...options,
    });

    let stdout = '';
    let stderr = '';

    child.stdout.on('data', (data) => {
      stdout += data.toString();
      console.log(`[stdout] ${data}`);
    });

    child.stderr.on('data', (data) => {
      stderr += data.toString();
      console.error(`[stderr] ${data}`);
    });

    child.on('close', (code) => {
      if (code === 0) {
        resolve({ stdout, stderr, code });
      } else {
        reject(new Error(`Process exited with code ${code}: ${stderr}`));
      }
    });

    child.on('error', (err) => {
      reject(new Error(`Failed to start process: ${err.message}`));
    });
  });
}

// Sử dụng
await runCommand('ffmpeg', ['-i', 'input.mp4', '-c:v', 'libx264', 'output.mp4']);
```

### Spawn với stdin

```javascript
const { spawn } = require('child_process');

const child = spawn('grep', ['pattern']);

child.stdin.write('line 1\nline 2 with pattern\nline 3\n');
child.stdin.end();

child.stdout.on('data', (data) => {
  console.log(data.toString()); // "line 2 with pattern\n"
});
```

---

## exec — Shell Commands

Chạy command qua shell — hỗ trợ pipes, redirects, environment variables.

```javascript
const { exec } = require('child_process');
const { promisify } = require('util');

const execAsync = promisify(exec);

async function getDiskUsage() {
  const { stdout } = await execAsync('df -h / | tail -1 | awk \'{print $5}\'');
  return stdout.trim();
}

// Với shell features
const { stdout } = await execAsync('ls -la | grep ".js" | wc -l');
console.log(`JS files: ${stdout.trim()}`);
```

### exec với maxBuffer

```javascript
const { exec } = require('child_process');

exec('cat large-file.log', { maxBuffer: 10 * 1024 * 1024 }, (err, stdout) => {
  if (err?.code === 'ERR_CHILD_PROCESS_STDIO_MAXBUFFER') {
    console.error('Output too large — use spawn instead');
  }
});
```

**⚠️ Security:** `exec` với user input → shell injection risk. Dùng `spawn` với array args hoặc sanitize input.

```javascript
// ❌ Shell injection vulnerability
exec(`grep ${userInput} file.txt`);

// ✅ Safe — args array, no shell
spawn('grep', [userInput, 'file.txt']);
```

---

## fork — Node.js Child Process

`fork()` là wrapper của `spawn()` chuyên cho Node.js — tự động tạo IPC channel.

### Parent Process

```javascript
const { fork } = require('child_process');
const path = require('path');

const worker = fork(path.join(__dirname, 'worker.js'));

worker.send({ type: 'COMPUTE', data: [1, 2, 3, 4, 5] });

worker.on('message', (result) => {
  console.log('Result from worker:', result);
});

worker.on('exit', (code) => {
  console.log(`Worker exited with code ${code}`);
});
```

### Child Process (worker.js)

```javascript
process.on('message', (msg) => {
  if (msg.type === 'COMPUTE') {
    const sum = msg.data.reduce((a, b) => a + b, 0);
    process.send({ type: 'RESULT', sum });
  }
});
```

### fork Options

```javascript
const worker = fork('./worker.js', [], {
  env: { ...process.env, WORKER_ID: '1' },
  execArgv: ['--max-old-space-size=4096'], // Child memory limit
  silent: true, // Pipe stdout/stderr to parent
  detached: false,
});
```

---

## IPC — Inter-Process Communication

IPC (Inter-Process Communication — Giao Tiếp Giữa Các Tiến Trình) qua `process.send()` và `message` event.

### Structured Message Protocol

```javascript
// Parent
const workers = new Map();

function createWorker(id) {
  const worker = fork('./task-worker.js');
  workers.set(id, worker);

  worker.on('message', (msg) => {
    switch (msg.type) {
      case 'READY':
        worker.send({ type: 'TASK', payload: getNextTask() });
        break;
      case 'RESULT':
        handleResult(msg.payload);
        worker.send({ type: 'TASK', payload: getNextTask() });
        break;
      case 'ERROR':
        handleError(msg.error);
        break;
    }
  });

  return worker;
}
```

### Shared Memory (Advanced)

Worker Threads hỗ trợ `SharedArrayBuffer` — child processes không share memory (isolated). Muốn share data giữa processes → dùng Redis, file, hoặc message passing.

---

## Error Handling và Exit Codes

```javascript
const { spawn } = require('child_process');

function runWithTimeout(command, args, timeoutMs = 30000) {
  return new Promise((resolve, reject) => {
    const child = spawn(command, args);
    let killed = false;

    const timer = setTimeout(() => {
      killed = true;
      child.kill('SIGTERM');
      setTimeout(() => child.kill('SIGKILL'), 5000); // Force kill after 5s
      reject(new Error(`Process timed out after ${timeoutMs}ms`));
    }, timeoutMs);

    child.on('close', (code, signal) => {
      clearTimeout(timer);
      if (killed) return;

      if (code === 0) {
        resolve({ code });
      } else {
        reject(new Error(`Exit code ${code}, signal ${signal}`));
      }
    });

    child.on('error', reject);
  });
}
```

### Exit Codes Phổ Biến

| Code | Ý Nghĩa |
| ---- | ------- |
| `0` | Success |
| `1` | General error |
| `2` | Misuse of shell command |
| `137` | Killed by SIGKILL (128 + 9) |
| `143` | Killed by SIGTERM (128 + 15) |

### Graceful Shutdown

```javascript
process.on('SIGTERM', () => {
  console.log('Received SIGTERM, cleaning up...');
  // Cleanup resources
  process.exit(0);
});
```

---

## Child Process vs Worker Threads

| Aspect | Child Process | Worker Threads |
| ------ | ------------- | -------------- |
| Isolation | Process riêng biệt | Shared process, separate V8 isolate |
| Memory | Không share | `SharedArrayBuffer` possible |
| Startup | Chậm hơn (~30-50ms) | Nhanh hơn (~5-10ms) |
| Crash impact | Child crash ≠ parent crash | Worker crash có thể affect process |
| Use case | CLI tools, isolation | CPU-intensive JS computation |
| IPC | `process.send()` | `postMessage()` |

```
CPU-bound JS task:
  → Worker Threads (nhẹ hơn, share memory)

Run external CLI (ffmpeg, git):
  → Child Process spawn

Full isolation needed:
  → Child Process fork
```

---

## Ví Dụ Thực Tế

### 1. Image Processing với ImageMagick

```javascript
const { spawn } = require('child_process');
const path = require('path');

async function resizeImage(inputPath, outputPath, width, height) {
  return new Promise((resolve, reject) => {
    const args = [
      inputPath,
      '-resize', `${width}x${height}`,
      '-quality', '85',
      outputPath,
    ];

    const convert = spawn('convert', args);

    convert.on('close', (code) => {
      code === 0 ? resolve(outputPath) : reject(new Error(`convert failed: ${code}`));
    });
  });
}
```

### 2. Worker Pool với fork

```javascript
const { fork } = require('child_process');
const os = require('os');

class WorkerPool {
  constructor(workerPath, size = os.cpus().length) {
    this.workers = [];
    this.queue = [];

    for (let i = 0; i < size; i++) {
      this.createWorker(workerPath);
    }
  }

  createWorker(workerPath) {
    const worker = fork(workerPath);

    worker.on('message', (result) => {
      const { resolve } = worker.currentTask;
      worker.busy = false;
      worker.currentTask = null;
      resolve(result);
      this.processQueue();
    });

    worker.busy = false;
    this.workers.push(worker);
  }

  exec(data) {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this.processQueue();
    });
  }

  processQueue() {
    const available = this.workers.find((w) => !w.busy);
    const task = this.queue.shift();

    if (available && task) {
      available.busy = true;
      available.currentTask = task;
      available.send(task.data);
    }
  }
}
```

### 3. Git Operations

```javascript
const { execFile } = require('child_process');
const { promisify } = require('util');

const execFileAsync = promisify(execFile);

async function getGitLog(repoPath, limit = 10) {
  const { stdout } = await execFileAsync('git', [
    '-C', repoPath,
    'log', `--max-count=${limit}`,
    '--pretty=format:%H|%an|%s|%ai',
  ], { encoding: 'utf8' });

  return stdout.trim().split('\n').map((line) => {
    const [hash, author, subject, date] = line.split('|');
    return { hash, author, subject, date };
  });
}
```

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| Dùng `spawn` thay `exec` cho output lớn | Tránh maxBuffer limit, streaming |
| Không pass user input vào `exec` shell | Prevent shell injection |
| Set timeout cho long-running processes | Tránh zombie processes |
| Handle `SIGTERM`/`SIGKILL` gracefully | Cleanup temp files, connections |
| Dùng `fork` cho Node.js workers | IPC built-in, dễ message passing |
| Limit concurrent child processes | Tránh fork bomb, resource exhaustion |
| Log stdout/stderr riêng biệt | Debug dễ hơn |

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
| ------- | ------------ |
| `spawn` vs `exec` khác gì? | `spawn` stream output, no buffer limit; `exec` buffer all, có maxBuffer 1MB |
| `fork` khác `spawn` thế nào? | `fork` chỉ cho Node.js, có IPC channel sẵn |
| Shell injection risk ở đâu? | `exec` với user input — dùng `spawn` với args array |
| Child process vs Worker Thread? | Process: isolated, chạy CLI; Thread: shared memory, CPU-bound JS |
| IPC trong Node.js hoạt động thế nào? | `process.send()` / `message` event qua internal channel |
| Exit code 137 nghĩa là gì? | Process bị kill bằng SIGKILL (OOM killer hoặc manual) |
| Làm sao kill child process? | `child.kill('SIGTERM')` graceful, `SIGKILL` force |
