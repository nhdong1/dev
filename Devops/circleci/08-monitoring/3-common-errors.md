# Lỗi Pipeline Thường Gặp — Common Errors

> Tra cứu nhanh **OOM** (Out Of Memory — Hết Bộ Nhớ), **timeout**, **exit code**, cache, permission và lỗi cấu hình YAML khi vận hành CircleCI hàng ngày.

---

## 📚 Mục Lục

1. [Đọc log và exit code](#đọc-log-và-exit-code)
2. [OOM và tín hiệu 137](#oom-và-tín-hiệu-137)
3. [Timeout](#timeout)
4. [Exit code phổ biến](#exit-code-phổ-biến)
5. [Cache miss và workspace](#cache-miss-và-workspace)
6. [Docker / setup_remote_docker](#docker--setup_remote_docker)
7. [Secrets và permission](#secrets-và-permission)
8. [Config YAML](#config-yaml)
9. [Bảng tra cứu nhanh](#bảng-tra-cứu-nhanh)

---

## Đọc log và exit code

CircleCI đánh dấu **step** fail khi lệnh trả exit code khác `0`.

```
Step "Run tests" exited with code 1
```

**Quy trình:**

1. Cuộn **cuối log** step đỏ (thông báo lỗi thật thường ở đây)
2. Tìm từ khóa: `Killed`, `ENOMEM`, `timeout`, `Permission denied`, `command not found`
3. Xem step trước có warning (cache miss, deprecated) không

Biến hữu ích: `CIRCLE_JOB`, `CIRCLE_WORKFLOW_ID`, `CIRCLE_BUILD_NUM` (built-in env).

---

## OOM và tín hiệu 137

### Triệu chứng

```
Killed
# hoặc
Exited with code 137
```

**137** = 128 + 9 (SIGKILL — tín hiệu kill, thường do OOM killer).

### Nguyên nhân

- **Resource class** quá nhỏ cho `npm test`, compiler, Docker build
- **Parallelism** quá cao trên executor nhỏ (nhiều process cùng lúc)
- Memory leak trong test hoặc build
- Docker build không giới hạn layer cache trên disk/RAM

### Hướng xử lý

```yaml
# Tăng resource class
jobs:
  test:
    docker:
      - image: cimg/node:20.0
    resource_class: large   # hoặc xlarge — xem pricing

# Giảm parallelism tạm thời để xác nhận OOM
    parallelism: 2   # thay vì 8
```

- Chia nhỏ test suite ([05-optimization/3-test-splitting.md](../05-optimization/3-test-splitting.md))
- JVM/Node: tăng heap có kiểm soát (`NODE_OPTIONS=--max-old-space-size=4096`)
- Machine executor khi cần RAM lớn hơn Docker mặc định

---

## Timeout

### Các loại timeout

| Loại | Mô tả |
|------|--------|
| **Step `no_output_timeout`** | Không có output trong khoảng thời gian → fail |
| **Job / workflow** | Giới hạn tổng thời gian (plan/org) |
| **Network** | `curl`, registry, API chậm/treo |
| **Queue** | Chờ runner lâu — hiến thị pending, không phải step timeout |

### Ví dụ cấu hình

```yaml
- run:
    name: Integration tests
    command: ./scripts/integration.sh
    no_output_timeout: 30m

- run:
    name: Verbose progress
    command: |
      while true; do echo "still running..."; sleep 60; done
    no_output_timeout: 10m
```

**Fix:** thêm log định kỳ, tối ưu test chậm, mock service ngoài, tăng timeout có lý do (không che hang vô hạn).

---

## Exit code phổ biến

| Code | Ý nghĩa thường gặp | Gợi ý |
|------|-------------------|--------|
| **0** | Thành công | — |
| **1** | Lỗi generic (test fail, script lỗi) | Đọc stack trace |
| **2** | Misuse shell command | Kiểm tra shebang, `set -e` |
| **125** | Docker run lỗi | Image không pull được, quyền |
| **126** | Command không execute được | Không executable (`chmod +x`) |
| **127** | Command not found | Thiếu package trên image |
| **137** | OOM / SIGKILL | Xem mục OOM |
| **143** | SIGTERM — thường cancel/timeout | Job bị hủy hoặc timeout |

### `set -e` trong bash

```yaml
- run:
    command: |
      set -euo pipefail
      ./deploy.sh
```

Một lệnh fail giữa pipe → cả step fail (đúng mong muốn CI).

---

## Cache miss và workspace

### Cache miss

**Triệu chứng:** bước `restore_cache` không hit, `npm ci` / `bundle install` chạy lâu bất thường.

**Nguyên nhân:**

- Đổi `package-lock.json` → key đổi (đúng hành vi)
- Sai pattern key hoặc thiếu fallback key
- Cache branch khác (`npm-v1-` vs `npm-v2-`)

Xem [05-optimization/1-caching-strategies.md](../05-optimization/1-caching-strategies.md).

### Workspace

**Triệu chứng:** job sau không thấy file artifact.

```
Unable to attach workspace / file not found
```

**Nguyên nhân:** quên `persist_to_workspace`, sai `root`/`paths`, job parallel không share workspace.

Xem [05-optimization/2-workspace.md](../05-optimization/2-workspace.md).

---

## Docker / setup_remote_docker

### Lỗi thường gặp

| Log | Nguyên nhân |
|-----|-------------|
| `Cannot connect to the Docker daemon` | Thiếu `setup_remote_docker` |
| `no space left on device` | Disk layer Docker đầy |
| `denied: requested access to the resource is denied` | Sai credential registry |

```yaml
- setup_remote_docker:
    version: 20.10.24
    docker_layer_caching: true   # plan hỗ trợ DLC — Docker Layer Caching
```

Chi tiết build/push: [07-integration/1-docker-build-push.md](../07-integration/1-docker-build-push.md).

---

## Secrets và permission

| Triệu chứng | Khả năng |
|-------------|----------|
| `Access Denied` AWS | Sai key, hết hạn, thiếu OIDC role |
| Biến rỗng | Quên gán Context cho job |
| `401` registry | Token ECR/GCR hết hạn |

```yaml
workflows:
  deploy:
    jobs:
      - deploy:
          context:
            - aws-production   # bắt buộc nếu secret trong context
```

OIDC: [06-security/2-oidc-integration.md](../06-security/2-oidc-integration.md).

---

## Config YAML

| Lỗi validate | Cách xử lý |
|--------------|------------|
| `Error in config.yml` | `circleci config validate` local |
| `Cannot find orb` | Sai tên/version orb |
| `requires unknown job` | Tên job typo trong workflow |
| `Too much config` | Tách file với [Dynamic Config](../09-advanced/1-dynamic-config.md) (khi có module) |

---

## Bảng tra cứu nhanh

| Triệu chứng | File / Hành động |
|-------------|------------------|
| `137`, `Killed` | Tăng `resource_class`, giảm `parallelism` |
| Treo không log | `no_output_timeout`, thêm echo progress |
| Chỉ fail CI | [2-ssh-debugging.md](./2-ssh-debugging.md) |
| Test đỏ ngẫu nhiên | [4-flaky-tests.md](./4-flaky-tests.md) |
| Chậm dần | Insights + [05-optimization/](../05-optimization/) |
| Fail deploy | Context, OIDC, approval workflow |

---

**Tiếp theo:** [4-flaky-tests.md](./4-flaky-tests.md) — giảm nhiễu từ kiểm thử không ổn định.
