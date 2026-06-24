# PM2 Process Manager — Cluster Mode, Zero-downtime Reload

> PM2 (Process Manager 2 — Trình Quản Lý Tiến Trình) là giải pháp phổ biến để chạy Node.js app trên single server hoặc VM — cluster mode (chế độ cụm), auto-restart, log management, và zero-downtime reload (tải lại không downtime).

## Mục Lục

1. [Khi Nào Dùng PM2](#khi-nào-dùng-pm2)
2. [Cài Đặt và Khởi Chạy Cơ Bản](#cài-đặt-và-khởi-chạy-cơ-bản)
3. [Ecosystem File — Cấu Hình Production](#ecosystem-file--cấu-hình-production)
4. [Cluster Mode](#cluster-mode)
5. [Zero-downtime Reload](#zero-downtime-reload)
6. [Log Management](#log-management)
7. [Monitoring và Metrics](#monitoring-và-metrics)
8. [Graceful Shutdown với PM2](#graceful-shutdown-với-pm2)
9. [PM2 vs Docker/K8s](#pm2-vs-dockerk8s)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khi Nào Dùng PM2

| Scenario | PM2 Phù Hợp? |
| -------- | ------------ |
| Single VPS / bare metal server | ✅ Rất phù hợp |
| Docker container trên K8s | ❌ Không cần — K8s quản lý processes |
| Startup MVP, budget thấp | ✅ Đơn giản, hiệu quả |
| Enterprise multi-region | ❌ Dùng K8s/ECS |
| Staging server | ✅ Nhanh setup |

```
Single Server Architecture với PM2:

                    ┌─────────────────┐
                    │   Nginx/Caddy   │
                    │  (Reverse Proxy)│
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         ┌─────────┐   ┌─────────┐   ┌─────────┐
         │ PM2 #1  │   │ PM2 #2  │   │ PM2 #3  │  ← cluster mode
         │ Worker  │   │ Worker  │   │ Worker  │
         └─────────┘   └─────────┘   └─────────┘
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                    ┌─────────────────┐
                    │   PostgreSQL    │
                    └─────────────────┘
```

---

## Cài Đặt và Khởi Chạy Cơ Bản

```bash
# Cài global
npm install -g pm2

# Khởi chạy app
pm2 start dist/main.js --name api

# Các lệnh thường dùng
pm2 list                    # Danh sách processes
pm2 logs api                # Xem logs real-time
pm2 monit                   # Dashboard monitoring
pm2 restart api             # Restart (có downtime ngắn)
pm2 reload api              # Zero-downtime reload
pm2 stop api
pm2 delete api

# Lưu process list — auto-start sau reboot
pm2 save
pm2 startup                 # Tạo systemd/init script
```

---

## Ecosystem File — Cấu Hình Production

`ecosystem.config.js` — declarative configuration cho toàn bộ apps.

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'api',
      script: './dist/main.js',
      instances: 'max',           // hoặc số cụ thể: 4
      exec_mode: 'cluster',       // cluster | fork
      env: {
        NODE_ENV: 'development',
        PORT: 3000,
      },
      env_production: {
        NODE_ENV: 'production',
        PORT: 3000,
      },
      // Auto-restart
      max_memory_restart: '512M',
      exp_backoff_restart_delay: 100,

      // Graceful shutdown
      kill_timeout: 5000,         // ms chờ trước khi force kill
      listen_timeout: 10000,      // ms chờ app listen sau start
      wait_ready: true,           // chờ process.send('ready')

      // Logging
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      error_file: '/var/log/pm2/api-error.log',
      out_file: '/var/log/pm2/api-out.log',
      merge_logs: true,

      // Watch (chỉ development)
      watch: false,
      ignore_watch: ['node_modules', 'logs'],
    },
  ],
};
```

```bash
# Deploy với ecosystem file
pm2 start ecosystem.config.js --env production
```

### Signal `ready` trong App

```typescript
const server = app.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
  // Báo PM2 app đã sẵn sàng nhận traffic
  if (process.send) {
    process.send('ready');
  }
});
```

---

## Cluster Mode

PM2 cluster mode fork nhiều worker processes — mỗi worker chạy trên một CPU core.

```
Master Process (PM2)
    │
    ├── Worker 1 ──► Event Loop ──► Handle requests
    ├── Worker 2 ──► Event Loop ──► Handle requests
    ├── Worker 3 ──► Event Loop ──► Handle requests
    └── Worker 4 ──► Event Loop ──► Handle requests

OS load balances incoming connections (round-robin by default)
```

| Setting | Giá Trị | Ý Nghĩa |
| ------- | ------- | ------- |
| `instances: 'max'` | = số CPU cores | Tận dụng tối đa hardware |
| `instances: 4` | 4 workers | Kiểm soát memory usage |
| `exec_mode: 'cluster'` | cluster | Shared port, load balanced |
| `exec_mode: 'fork'` | fork | Single process, phù hợp dev |

### Sticky Sessions

WebSocket và session-based auth cần sticky sessions — cùng client luôn đến cùng worker.

```javascript
// ecosystem.config.js
{
  instances: 4,
  exec_mode: 'cluster',
  instance_var: 'INSTANCE_ID',  // process.env.INSTANCE_ID
}
```

Với HTTP stateless + JWT — **không cần sticky sessions**.

---

## Zero-downtime Reload

```
pm2 restart  → Stop ALL workers → Start ALL workers  ❌ Downtime
pm2 reload   → Reload từng worker một                 ✅ Zero downtime

Timeline pm2 reload:
Worker 1: [running] → [reloading] → [running]
Worker 2: [running] ──────────────► [reloading] → [running]
Worker 3: [running] ────────────────────────────► [reloading]
Worker 4: [running] ──────────────────────────────────────► ...

Traffic luôn được handle bởi ít nhất N-1 workers
```

```bash
# Zero-downtime reload
pm2 reload api

# Hoặc graceful reload tất cả apps
pm2 reload all
```

**Yêu cầu:** App phải handle `SIGINT`/`SIGTERM` gracefully và có `wait_ready: true`.

---

## Log Management

```bash
# Xem logs
pm2 logs api --lines 100
pm2 logs api --err          # Chỉ error logs

# Flush logs
pm2 flush

# Log rotation (module)
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
pm2 set pm2-logrotate:compress true
```

### Structured Logging với PM2

PM2 ghi logs dạng text. Kết hợp Pino để output JSON:

```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // PM2 capture stdout — Pino ghi JSON ra stdout
});
```

---

## Monitoring và Metrics

```bash
# Built-in monitoring
pm2 monit

# PM2 Plus (cloud monitoring — optional)
pm2 link <secret> <public>
```

### Tích Hợp Prometheus

PM2 không expose metrics native. Dùng `prom-client` trong app:

```typescript
import { Registry, collectDefaultMetrics } from 'prom-client';

const register = new Registry();
collectDefaultMetrics({ register });

app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

---

## Graceful Shutdown với PM2

```typescript
function gracefulShutdown() {
  console.log('Shutting down gracefully...');

  server.close(() => {
    console.log('HTTP server closed');
    Promise.all([
      db.disconnect(),
      redis.quit(),
    ]).then(() => {
      console.log('All connections closed');
      process.exit(0);
    });
  });
}

process.on('SIGINT', gracefulShutdown);
process.on('SIGTERM', gracefulShutdown);
```

PM2 config liên quan:

| Option | Giá Trị Khuyến Nghị | Mô Tả |
| ------ | ------------------- | ----- |
| `kill_timeout` | 5000–30000 | Thời gian chờ graceful shutdown |
| `wait_ready` | true | Chờ `process.send('ready')` |
| `listen_timeout` | 10000 | Timeout nếu app không start |

---

## PM2 vs Docker/K8s

| Tiêu Chí | PM2 | Docker + K8s |
| -------- | --- | ------------ |
| Setup complexity | Thấp | Cao |
| Multi-server scaling | Thủ công | Tự động (HPA) |
| Zero-downtime deploy | `pm2 reload` | Rolling update |
| Resource isolation | Không | Container cgroups |
| Cost | Thấp (1 VPS) | Cao hơn (cluster) |
| Phù hợp | MVP, small team | Production scale |

**Quy tắc:** PM2 **trong** Docker container = anti-pattern. Chọn một: PM2 trên bare metal **hoặc** Docker/K8s (một process per container).

---

## Best Practices

1. **Dùng ecosystem file** — không hardcode config trong CLI commands
2. **`instances: 'max'`** cho CPU-bound I/O workloads trên dedicated server
3. **`max_memory_restart`** — auto-restart khi memory leak
4. **`pm2 save` + `pm2 startup`** — survive server reboot
5. **Không dùng `watch: true` trong production**
6. **Log rotation** — tránh disk full
7. **Health check qua reverse proxy** — Nginx `proxy_pass` + upstream health

### Nginx Reverse Proxy

```nginx
upstream api {
    server 127.0.0.1:3000;
    keepalive 64;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
        proxy_pass http://api/health;
        access_log off;
    }
}
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Đáp Án Ngắn |
| ------- | ----------- |
| PM2 cluster mode khác fork mode? | Cluster: nhiều workers share port, load balanced; Fork: single process |
| `pm2 restart` vs `pm2 reload`? | Restart có downtime; reload zero-downtime, reload từng worker |
| Tại sao không dùng PM2 trong Docker? | Violates one-process-per-container; K8s handles scaling/restart |
| `wait_ready` dùng để làm gì? | PM2 chờ app signal ready trước khi route traffic |
| `max_memory_restart` hoạt động thế nào? | Auto-restart process khi vượt memory threshold — safety net cho memory leaks |
