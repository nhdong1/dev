# Rerun with SSH — Gỡ Lỗi Pipeline Qua SSH

> **Rerun with SSH** (Chạy Lại Kèm SSH) cho phép SSH vào container/VM đang chạy job CircleCI — môi trường gần giống lần build fail — để chạy lệnh thủ công, kiểm tra file, mạng và biến môi trường.

---

## 📚 Mục Lục

1. [Khi nào dùng SSH debug](#khi-nào-dùng-ssh-debug)
2. [Cách bật Rerun with SSH](#cách-bật-rerun-with-ssh)
3. [Kết nối và phiên làm việc](#kết-nối-và-phiên-làm-việc)
4. [Giới hạn và bảo mật](#giới-hạn-và-bảo-mật)
5. [Mẹo debug hiệu quả](#mẹo-debug-hiệu-quả)
6. [So sánh với local execute](#so-sánh-với-local-execute)
7. [Câu hỏi phỏng vấn](#câu-hỏi-phỏng-vấn)

---

## Khi nào dùng SSH debug

| Nên dùng | Không cần SSH |
|----------|----------------|
| Pass local, fail trên CI (path, env, quyền) | Lỗi rõ trong log (syntax, test assertion) |
| Cần `curl`, `dig`, kiểm tra service nội bộ | Chỉ cần sửa config YAML |
| Debug Docker layer / file system trong job | Đã reproduce bằng `circleci local execute` |
| Kiểm tra biến Context có inject đúng không | Secret leak — **không** in secret ra log |

SSH **không thay** việc viết test hoặc fix code; nó rút ngắn vòng “đoán → thử”.

---

## Cách bật Rerun with SSH

### Trên CircleCI UI

```
1. Mở pipeline đã fail (hoặc success nhưng cần điều tra)
2. Chọn job cần debug
3. Nút "Rerun" → "Rerun job with SSH"
4. UI hiển thị lệnh ssh và fingerprint (khi job sẵn sàng)
```

### Điều kiện thường gặp

- Plan hỗ trợ SSH (thường có trên các gói trả phí — kiểm tra tài liệu plan hiện tại)
- Job dùng executor hỗ trợ SSH (Docker, Machine — phổ biến; một số setup đặc biệt có thể hạn chế)
- Job không bị cancel trước khi bước SSH mở

---

## Kết nối và phiên làm việc

### Ví dụ lệnh (UI cung cấp, không copy cứng)

```bash
# CircleCI hiển thị dạng tương tự:
ssh -p PORT XX.XX.XX.XX

# Trong phiên SSH, bạn thường ở thư mục project đã checkout
pwd
ls -la
env | sort
```

### Việc nên làm trong phiên

```bash
# Chạy lại step fail (ví dụ test)
npm test -- --testPathPattern=AuthService

# Kiểm tra tool version khác local
node -v
docker version

# Kiểm tra file/cache
ls -la ~/.npm
df -h    # disk đầy?

# Mạng (nếu policy cho phép)
curl -I https://registry.npmjs.org/
```

### Kết thúc

- Thoát SSH (`exit`) — job có thể tiếp tục hoặc kết thúc tùy UI
- **Không** để phiên SSH mở lâu không cần thiết (tốn credit — Tín Dụng CircleCI)

---

## Giới hạn và bảo mật

| Chủ đề | Chi tiết |
|--------|----------|
| **Thời gian** | Phiên SSH có timeout; job vẫn tính thời gian chạy |
| **Secrets** | Context vars có trong env — tránh `echo $AWS_SECRET`, screen share |
| **Compliance** | Một số tổ chức hạn chế SSH vào production deploy job |
| **Thay đổi state** | Sửa file trong container **không** persist sang rerun khác |
| **Audit** | Hành vi SSH có thể được ghi nhận — tuân policy nội bộ |

**Best practice:** chỉ developer có quyền project/org được SSH; dùng [Restricted Context](../06-security/1-contexts.md) cho prod secrets.

---

## Mẹo debug hiệu quả

### 1. Thu hẹp step

Tạm tách step lớn thành nhiều `run` nhỏ trong config để biết step nào fail (sau đó revert).

```yaml
- run: npm ci
- run: npm run build
- run: npm test
```

### 2. In thông tin không nhạy cảm

```yaml
- run:
    name: Debug context (không secret)
    command: |
      echo "Branch: $CIRCLE_BRANCH"
      echo "Node: $(node -v)"
      uname -a
```

### 3. `store_artifacts` thay SSH

Log file, screenshot test, dump:

```yaml
- store_artifacts:
    path: test-results/
    destination: junit
```

### 4. `no_output_timeout`

Step treo im lặng → tăng timeout hoặc fix hang:

```yaml
- run:
    name: Long integration test
    command: ./run-integration.sh
    no_output_timeout: 20m
```

---

## So sánh với local execute

| | **Rerun with SSH** | **circleci local execute** |
|--|-------------------|---------------------------|
| Môi trường | Runner CircleCI thật | Docker trên máy dev |
| Secrets/Context | Đầy đủ như CI | Thường thiếu hoặc mock |
| Network/VPC | Giống production CI | Khác |
| Chi phí | Credit CircleCI | Local |

Workflow đề xuất: `config validate` → local (nếu được) → push → nếu fail chỉ trên cloud → SSH.

---

## Câu hỏi phỏng vấn

**H: SSH debug an toàn không?**

**Đ:** Có rủi ro lộ secret và bypass review nếu lạm dụng; nên giới hạn quyền, không log secrets, ưu tiên artifact và test tái hiện. Production deploy nên dùng approval + audit.

**H: Pipeline pass local nhưng fail CI — bạn làm gì?**

**Đ:** So sánh version runtime, env vars, working directory, quyền file; dùng SSH chạy đúng lệnh fail; kiểm tra dependency on `main` vs lockfile; xem [3-common-errors.md](./3-common-errors.md).

---

**Tiếp theo:** [3-common-errors.md](./3-common-errors.md) — tra cứu lỗi theo triệu chứng.
