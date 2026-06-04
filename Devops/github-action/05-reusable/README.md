# 05 — Reusable Workflows & Custom Actions

> Xây dựng thư viện CI/CD nội bộ — tái sử dụng logic workflow và đóng gói automation thành actions có thể chia sẻ.

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Sơ Đồ Quyết Định — Dùng Loại Nào?](#sơ-đồ-quyết-định)
3. [Các File Trong Chủ Đề Này](#các-file-trong-chủ-đề-này)
4. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
5. [So Sánh Các Loại](#so-sánh-các-loại)
6. [Checklist Thực Hành](#checklist-thực-hành)

---

## Tổng Quan

GitHub Actions cung cấp nhiều cơ chế để tái sử dụng (reuse) và đóng gói (encapsulate) automation logic:

| Cơ Chế | Mô Tả | Dùng Khi |
|---|---|---|
| **Reusable Workflow** | Toàn bộ workflow được gọi từ workflow khác | Cần tái sử dụng cả job/environment logic |
| **Composite Action** | Nhóm nhiều steps thành một action | Gói gọn steps lặp đi lặp lại |
| **JavaScript Action** | Action viết bằng Node.js | Logic phức tạp, gọi API, xử lý dữ liệu |
| **Docker Action** | Action chạy trong Docker container | Cần môi trường tùy chỉnh, tools đặc biệt |

---

## Sơ Đồ Quyết Định

```
Cần tái sử dụng logic CI/CD?
│
├─ Toàn bộ workflow (nhiều jobs, environments, secrets)?
│   └─ → Reusable Workflow (workflow_call)
│
└─ Một nhóm steps cụ thể?
    │
    ├─ Chỉ là shell commands / actions đơn giản?
    │   └─ → Composite Action
    │
    ├─ Cần tương tác GitHub API, xử lý dữ liệu phức tạp?
    │   └─ → JavaScript Action (Node.js)
    │
    └─ Cần môi trường đặc biệt, tools không có sẵn?
        └─ → Docker Action
```

---

## Các File Trong Chủ Đề Này

| File | Nội Dung |
|---|---|
| [1-reusable-workflows.md](1-reusable-workflows.md) | `workflow_call` — inputs, outputs, secrets, caller pattern |
| [2-composite-actions.md](2-composite-actions.md) | Composite Actions — `action.yml`, đóng gói steps |
| [3-javascript-actions.md](3-javascript-actions.md) | Node.js Actions — `@actions/core`, `@actions/github` |
| [4-docker-actions.md](4-docker-actions.md) | Docker Container Actions — Dockerfile, entrypoint |
| [5-marketplace-guide.md](5-marketplace-guide.md) | Marketplace — đánh giá, pin SHA, fork khi cần |

---

## Khái Niệm Cốt Lõi

### Reusable Workflow (Workflow Tái Sử Dụng)

```yaml
# Workflow được tái sử dụng — định nghĩa trong .github/workflows/
on:
  workflow_call:          # Event đặc biệt — workflow này được gọi từ workflow khác
    inputs:
      environment:
        type: string
        required: true
    secrets:
      deploy_key:
        required: true
    outputs:
      deploy_url:
        value: ${{ jobs.deploy.outputs.url }}
```

- Được lưu trong cùng repo hoặc repo khác trong organization
- Hỗ trợ `inputs` (giá trị đầu vào), `outputs` (giá trị đầu ra), `secrets` (bí mật)
- Giới hạn: tối đa **4 cấp lồng nhau** (nesting levels)

### Composite Action (Action Kết Hợp)

```yaml
# action.yml — định nghĩa composite action
name: 'Setup Node and Cache'
description: 'Setup Node.js với npm cache tối ưu'
inputs:
  node-version:
    description: 'Phiên bản Node.js'
    default: '20'
runs:
  using: 'composite'      # Loại action: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    - run: npm ci
      shell: bash
```

- Lưu tại gốc repo hoặc thư mục bất kỳ
- Cần chỉ định `shell:` cho mỗi `run` step
- Không hỗ trợ `services` hay `container`

### JavaScript Action

```
my-action/
├── action.yml        # Metadata
├── index.js          # Entry point
├── package.json
└── node_modules/     # Phải commit (hoặc dùng @vercel/ncc để bundle)
```

### Docker Action

```
my-docker-action/
├── action.yml        # Metadata
├── Dockerfile        # Container image
└── entrypoint.sh     # Script chạy trong container
```

---

## So Sánh Các Loại

| Tiêu Chí | Reusable Workflow | Composite Action | JS Action | Docker Action |
|---|---|---|---|---|
| **Đơn vị tái sử dụng** | Toàn bộ workflow | Nhóm steps | Một step | Một step |
| **Hỗ trợ jobs** | ✅ Nhiều jobs | ❌ | ❌ | ❌ |
| **Hỗ trợ environments** | ✅ | ❌ | ❌ | ❌ |
| **Secrets riêng** | ✅ `secrets:` block | ❌ (dùng input) | ❌ (dùng input) | ❌ (dùng input) |
| **Ngôn ngữ** | YAML | YAML / Shell | JavaScript / TypeScript | Bất kỳ ngôn ngữ |
| **Phụ thuộc runtime** | GitHub runner | GitHub runner | Node.js | Docker |
| **Startup time** | Chậm hơn (job mới) | Nhanh (cùng job) | Nhanh | Chậm hơn (pull image) |
| **Phù hợp cho** | Deploy pipeline, multi-env | Setup steps, lint, test | API calls, data processing | Tools đặc biệt, scripts phức tạp |

---

## Checklist Thực Hành

### Reusable Workflow
- [ ] Định nghĩa inputs có `description` và `default` rõ ràng
- [ ] Khai báo secrets cần thiết trong `on.workflow_call.secrets`
- [ ] Dùng `outputs` để truyền giá trị về caller
- [ ] Thêm `permissions` block với quyền tối thiểu
- [ ] Test với `workflow_dispatch` trước khi share

### Composite Action
- [ ] Đặt `action.yml` đúng vị trí
- [ ] Chỉ định `shell:` cho mỗi `run` step
- [ ] Dùng `${{ inputs.xxx }}` thay vì hardcode giá trị
- [ ] Thêm `description` cho action và từng input

### JavaScript Action
- [ ] Bundle code với `@vercel/ncc` — tránh commit `node_modules` thô
- [ ] Dùng `@actions/core` cho output, logging, error handling
- [ ] Set `fail-fast` với `core.setFailed()` khi có lỗi
- [ ] Thêm unit tests cho logic chính

### Docker Action
- [ ] Dùng image nhỏ nhất có thể (alpine, slim)
- [ ] Đặt `WORKDIR` và copy files cần thiết
- [ ] Test Dockerfile locally trước khi deploy
- [ ] Set `ENV` variables qua GitHub Actions input

### Marketplace Actions
- [ ] Pin theo SHA thay vì tag: `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`
- [ ] Kiểm tra số stars, last commit, license
- [ ] Đọc CHANGELOG trước khi nâng version
- [ ] Fork nếu action critical và không được maintained

---

## Câu Hỏi Phỏng Vấn Thường Gặp

1. **Khi nào dùng reusable workflow vs composite action?**
   - Reusable workflow: cần nhiều jobs, environments, approval gates
   - Composite action: chỉ cần nhóm steps trong cùng một job

2. **Giới hạn của reusable workflows là gì?**
   - Tối đa 4 cấp lồng nhau
   - Caller workflow không thể override `env` của callee
   - Cần `secrets: inherit` hoặc khai báo tường minh từng secret

3. **Tại sao phải bundle JavaScript action?**
   - `node_modules` có thể rất lớn
   - `@vercel/ncc` compile thành single file — commit nhanh, sạch hơn

4. **Sự khác nhau giữa `secrets: inherit` và khai báo tường minh?**
   - `secrets: inherit` truyền tất cả secrets từ caller — tiện nhưng kém an toàn
   - Khai báo tường minh: chỉ truyền secrets cần thiết — least-privilege

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Trạng Thái:** ✅ Hoàn Thành
