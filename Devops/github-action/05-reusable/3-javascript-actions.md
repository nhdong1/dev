# JavaScript Actions — Action Node.js

> Viết custom GitHub Actions bằng JavaScript/TypeScript — phù hợp cho logic phức tạp, gọi GitHub API, xử lý dữ liệu, và tích hợp với external services.

## 📚 Mục Lục

1. [Khái Niệm](#khái-niệm)
2. [Cấu Trúc Dự Án](#cấu-trúc-dự-án)
3. [action.yml cho JavaScript Action](#actionyml-cho-javascript-action)
4. [@actions/core — Toolkit Cốt Lõi](#actionscore--toolkit-cốt-lõi)
5. [@actions/github — GitHub API Client](#actionsgithub--github-api-client)
6. [Bundling với @vercel/ncc](#bundling-với-vercelncc)
7. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
8. [TypeScript Action](#typescript-action)
9. [Testing](#testing)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm

**JavaScript Action** là custom action được viết bằng Node.js. GitHub Actions runner thực thi file JavaScript trực tiếp — không cần Docker, khởi động nhanh hơn Docker action.

### Khi Nào Dùng JavaScript Action?

- **Gọi GitHub API**: Tạo issues, comment PR, label, tạo releases
- **Logic nghiệp vụ phức tạp**: Xử lý JSON, tính toán, điều kiện phức tạp
- **Tích hợp external services**: Jira, Slack API, PagerDuty, Datadog
- **Xử lý file/artifacts**: Parse, transform, validate dữ liệu
- **Khi cần startup nhanh** hơn Docker action

### Toolkit @actions — Thư Viện Chính Thức

| Package | Mô Tả | Dùng Cho |
|---|---|---|
| `@actions/core` | Input/output, logging, fail | Mọi JS action |
| `@actions/github` | Octokit client, context | Tương tác GitHub API |
| `@actions/exec` | Chạy shell commands | Thay thế `child_process.exec` |
| `@actions/io` | File system utilities | Copy, mkdir, rm files |
| `@actions/cache` | Save/restore cache | Cache management |
| `@actions/artifact` | Upload/download artifacts | Artifact management |
| `@actions/glob` | Glob pattern matching | Tìm files theo pattern |

---

## Cấu Trúc Dự Án

```
my-js-action/
├── action.yml          # Metadata — bắt buộc
├── index.js            # Entry point (hoặc src/index.ts cho TypeScript)
├── package.json
├── package-lock.json
├── dist/
│   └── index.js        # Bundled file — file này được commit vào repo
└── node_modules/       # KHÔNG commit — chỉ dùng khi develop
```

> **Quan trọng:** Commit `dist/index.js` (file đã bundle), KHÔNG commit `node_modules/`. Thêm `node_modules/` vào `.gitignore`.

---

## action.yml cho JavaScript Action

```yaml
name: 'Create GitHub Release Comment'
description: 'Tạo comment trên PR với thông tin deployment'
author: 'Platform Team'

inputs:
  github-token:
    description: 'GitHub token để gọi API'
    required: true
  pr-number:
    description: 'Số PR cần comment'
    required: true
  environment:
    description: 'Môi trường đã deploy'
    required: true
  deploy-url:
    description: 'URL của deployment'
    required: false
    default: ''
  message-prefix:
    description: 'Prefix cho comment message'
    required: false
    default: '🚀 Deployment'

outputs:
  comment-id:
    description: 'ID của comment vừa tạo'
  comment-url:
    description: 'URL trực tiếp đến comment'

runs:
  using: 'node20'        # Phiên bản Node.js runner dùng (node20, node16)
  main: 'dist/index.js'  # Entry point — file đã bundle
  post: 'dist/cleanup.js'  # (Tùy chọn) Chạy sau khi job hoàn thành — dùng cho cleanup
  post-if: 'always()'     # Điều kiện chạy post step

branding:
  icon: 'message-circle'
  color: 'blue'
```

---

## @actions/core — Toolkit Cốt Lõi

```javascript
// index.js
const core = require('@actions/core');

async function run() {
  try {
    // === LẤY INPUTS ===
    const token = core.getInput('github-token', { required: true });
    const prNumber = parseInt(core.getInput('pr-number', { required: true }));
    const environment = core.getInput('environment', { required: true });
    const deployUrl = core.getInput('deploy-url');       // Không bắt buộc

    // getMultilineInput — input nhiều dòng, trả về mảng
    const labels = core.getMultilineInput('labels');

    // getBooleanInput — parse 'true'/'false' thành boolean
    const dryRun = core.getBooleanInput('dry-run');

    // === LOGGING ===
    core.info('Bắt đầu xử lý...');
    core.debug(`PR Number: ${prNumber}`);           // Chỉ hiện khi debug mode bật
    core.warning('Chú ý: deploy URL trống');        // Cảnh báo (màu vàng)
    core.error('Lỗi không nghiêm trọng');           // Lỗi (màu đỏ) nhưng không stop

    // === GROUP LOGS ===
    core.startGroup('Chi tiết deployment');
    core.info(`Environment: ${environment}`);
    core.info(`URL: ${deployUrl}`);
    core.endGroup();

    // === MASK SECRET VALUES ===
    core.setSecret(token);                           // Ẩn giá trị trong mọi log

    // === SET OUTPUTS ===
    core.setOutput('comment-id', '12345');
    core.setOutput('comment-url', `https://github.com/...`);

    // === ENVIRONMENT VARIABLES cho subsequent steps ===
    core.exportVariable('DEPLOY_ENV', environment);  // Xuất biến môi trường

    // === PATH — thêm vào PATH ===
    core.addPath('/usr/local/custom-tools');

    // === STATE — lưu state để dùng ở post step ===
    core.saveState('deploy_id', 'abc123');
    const savedId = core.getState('deploy_id');      // Trong post step

  } catch (error) {
    // setFailed — đánh dấu action thất bại VÀ dừng
    core.setFailed(`Action thất bại: ${error.message}`);
  }
}

run();
```

### Summary — Tóm Tắt Kết Quả (Markdown)

```javascript
// Tạo Job Summary — hiển thị trong GitHub Actions UI
await core.summary
  .addHeading('Deployment Report')
  .addTable([
    [
      { data: 'Environment', header: true },
      { data: 'Status', header: true },
      { data: 'URL', header: true }
    ],
    ['staging', '✅ Success', 'https://staging.example.com'],
    ['production', '✅ Success', 'https://example.com']
  ])
  .addRaw('\n\n**Deploy time:** 2m 34s')
  .write();
```

---

## @actions/github — GitHub API Client

```javascript
const core = require('@actions/core');
const github = require('@actions/github');

async function run() {
  const token = core.getInput('github-token', { required: true });

  // Tạo Octokit client (GitHub REST API)
  const octokit = github.getOctokit(token);

  // === CONTEXT — thông tin về workflow run hiện tại ===
  const context = github.context;
  console.log(context.repo);           // { owner: 'myorg', repo: 'myrepo' }
  console.log(context.sha);            // Commit SHA
  console.log(context.ref);            // Branch/tag ref
  console.log(context.actor);          // User kích hoạt workflow
  console.log(context.eventName);      // 'push', 'pull_request', ...
  console.log(context.payload);        // Toàn bộ event payload

  const { owner, repo } = context.repo;
  const prNumber = context.payload.pull_request?.number;

  // === TẠO COMMENT TRÊN PR ===
  const comment = await octokit.rest.issues.createComment({
    owner,
    repo,
    issue_number: prNumber,
    body: `## 🚀 Deployed to Staging\n\n**URL:** https://staging.example.com\n**SHA:** \`${context.sha.slice(0, 7)}\``
  });
  core.setOutput('comment-id', comment.data.id.toString());

  // === TÌM VÀ UPDATE COMMENT CŨ (tránh spam) ===
  const comments = await octokit.rest.issues.listComments({
    owner,
    repo,
    issue_number: prNumber
  });

  const botComment = comments.data.find(c =>
    c.user?.type === 'Bot' && c.body?.includes('## 🚀 Deployed')
  );

  if (botComment) {
    // Update comment hiện có
    await octokit.rest.issues.updateComment({
      owner,
      repo,
      comment_id: botComment.id,
      body: `## 🚀 Deployed to Staging (updated)\n**URL:** https://staging.example.com`
    });
  }

  // === THÊM LABEL VÀO PR ===
  await octokit.rest.issues.addLabels({
    owner,
    repo,
    issue_number: prNumber,
    labels: ['deployed-staging']
  });

  // === TẠO DEPLOYMENT STATUS ===
  const deployment = await octokit.rest.repos.createDeployment({
    owner,
    repo,
    ref: context.sha,
    environment: 'staging',
    auto_merge: false,
    required_contexts: []
  });

  if (deployment.status === 201) {
    await octokit.rest.repos.createDeploymentStatus({
      owner,
      repo,
      deployment_id: deployment.data.id,
      state: 'success',
      environment_url: 'https://staging.example.com',
      description: 'Deployed successfully'
    });
  }

  // === GRAPHQL API (cho queries phức tạp hơn) ===
  const result = await octokit.graphql(`
    query($owner: String!, $repo: String!, $pr: Int!) {
      repository(owner: $owner, name: $repo) {
        pullRequest(number: $pr) {
          title
          reviews(last: 5) {
            nodes {
              state
              author { login }
            }
          }
        }
      }
    }
  `, { owner, repo, pr: prNumber });
}

run().catch(error => core.setFailed(error.message));
```

---

## Bundling với @vercel/ncc

`@vercel/ncc` compile toàn bộ JavaScript action (kể cả `node_modules`) thành một file duy nhất.

### Setup

```json
// package.json
{
  "name": "my-action",
  "version": "1.0.0",
  "main": "dist/index.js",
  "scripts": {
    "build": "ncc build src/index.js -o dist --license licenses.txt",
    "build:minify": "ncc build src/index.js -o dist --minify",
    "package": "npm run build && git add dist -f",
    "test": "jest"
  },
  "dependencies": {
    "@actions/core": "^1.10.1",
    "@actions/github": "^6.0.0"
  },
  "devDependencies": {
    "@vercel/ncc": "^0.38.1",
    "jest": "^29.0.0"
  }
}
```

### Workflow để tự động build

```yaml
# .github/workflows/build-action.yml
name: Build Action

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'package*.json'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Build bundle
        run: npm run build

      - name: Commit dist
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: 'chore: rebuild action bundle'
          file_pattern: 'dist/'
```

### .gitignore

```gitignore
node_modules/
*.js.map
*.d.ts
# KHÔNG gitignore dist/ — cần commit dist/index.js
```

---

## Ví Dụ Thực Tế

### Action: Auto-label PR Theo Kích Thước

```javascript
// src/index.js — Tự động label PR dựa trên số lines changed
const core = require('@actions/core');
const github = require('@actions/github');

const SIZE_LABELS = {
  XS: { max: 10,   color: '3CBF00', label: 'size/XS' },
  S:  { max: 50,   color: '5D9801', label: 'size/S'  },
  M:  { max: 250,  color: 'EEB800', label: 'size/M'  },
  L:  { max: 1000, color: 'EE8500', label: 'size/L'  },
  XL: { max: Infinity, color: 'EE5500', label: 'size/XL' },
};

async function run() {
  try {
    const token = core.getInput('github-token', { required: true });
    const octokit = github.getOctokit(token);
    const { owner, repo } = github.context.repo;
    const prNumber = github.context.payload.pull_request?.number;

    if (!prNumber) {
      core.info('Không phải PR event — bỏ qua');
      return;
    }

    // Lấy thông tin PR
    const { data: pr } = await octokit.rest.pulls.get({ owner, repo, pull_number: prNumber });
    const linesChanged = pr.additions + pr.deletions;

    core.info(`Lines changed: ${linesChanged} (additions: ${pr.additions}, deletions: ${pr.deletions})`);

    // Tính size
    const size = Object.entries(SIZE_LABELS).find(([, config]) => linesChanged <= config.max);
    const [sizeName, sizeConfig] = size || ['XL', SIZE_LABELS.XL];

    // Đảm bảo label tồn tại
    try {
      await octokit.rest.issues.createLabel({
        owner, repo,
        name: sizeConfig.label,
        color: sizeConfig.color
      });
    } catch (e) {
      // Label đã tồn tại — bỏ qua lỗi 422
      if (e.status !== 422) throw e;
    }

    // Xóa size labels cũ
    const existingLabels = pr.labels.map(l => l.name);
    for (const [, config] of Object.entries(SIZE_LABELS)) {
      if (existingLabels.includes(config.label) && config.label !== sizeConfig.label) {
        await octokit.rest.issues.removeLabel({
          owner, repo,
          issue_number: prNumber,
          name: config.label
        }).catch(() => {}); // Ignore nếu label không tồn tại
      }
    }

    // Thêm size label mới
    await octokit.rest.issues.addLabels({
      owner, repo,
      issue_number: prNumber,
      labels: [sizeConfig.label]
    });

    core.setOutput('size', sizeName);
    core.setOutput('lines-changed', linesChanged.toString());
    core.info(`✅ Đã gán label: ${sizeConfig.label}`);

  } catch (error) {
    core.setFailed(error.message);
  }
}

run();
```

---

## TypeScript Action

```typescript
// src/index.ts
import * as core from '@actions/core';
import * as github from '@actions/github';

interface DeploymentInfo {
  environment: string;
  url: string;
  sha: string;
}

async function createDeploymentComment(
  octokit: ReturnType<typeof github.getOctokit>,
  prNumber: number,
  info: DeploymentInfo
): Promise<number> {
  const { owner, repo } = github.context.repo;

  const body = `## 🚀 Deployed to ${info.environment}

| Field | Value |
|---|---|
| **URL** | ${info.url} |
| **SHA** | \`${info.sha.slice(0, 7)}\` |
| **Time** | ${new Date().toISOString()} |`;

  const response = await octokit.rest.issues.createComment({
    owner,
    repo,
    issue_number: prNumber,
    body
  });

  return response.data.id;
}

async function run(): Promise<void> {
  try {
    const token = core.getInput('github-token', { required: true });
    const environment = core.getInput('environment', { required: true });
    const deployUrl = core.getInput('deploy-url');

    const octokit = github.getOctokit(token);
    const prNumber = github.context.payload.pull_request?.number;

    if (!prNumber) {
      core.info('Không chạy trong PR context');
      return;
    }

    const commentId = await createDeploymentComment(octokit, prNumber, {
      environment,
      url: deployUrl,
      sha: github.context.sha
    });

    core.setOutput('comment-id', commentId.toString());

  } catch (error) {
    if (error instanceof Error) {
      core.setFailed(error.message);
    } else {
      core.setFailed('Unknown error');
    }
  }
}

run();
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./lib",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## Testing

```javascript
// __tests__/index.test.js
const core = require('@actions/core');
const github = require('@actions/github');

// Mock các modules
jest.mock('@actions/core');
jest.mock('@actions/github');

describe('PR Size Labeler', () => {
  beforeEach(() => {
    jest.clearAllMocks();

    // Mock github context
    github.context.repo = { owner: 'myorg', repo: 'myrepo' };
    github.context.payload = { pull_request: { number: 42 } };

    // Mock getInput
    core.getInput.mockImplementation((name) => {
      const inputs = { 'github-token': 'fake-token' };
      return inputs[name] || '';
    });
  });

  test('gán label size/S cho PR nhỏ', async () => {
    const mockOctokit = {
      rest: {
        pulls: {
          get: jest.fn().mockResolvedValue({
            data: { additions: 20, deletions: 10, labels: [] }
          })
        },
        issues: {
          createLabel: jest.fn().mockRejectedValue({ status: 422 }),
          addLabels: jest.fn().mockResolvedValue({}),
          removeLabel: jest.fn().mockResolvedValue({})
        }
      }
    };
    github.getOctokit.mockReturnValue(mockOctokit);

    const { run } = require('../src/index');
    await run();

    expect(mockOctokit.rest.issues.addLabels).toHaveBeenCalledWith(
      expect.objectContaining({ labels: ['size/S'] })
    );
    expect(core.setOutput).toHaveBeenCalledWith('size', 'S');
  });
});
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao cần bundle JavaScript action với ncc thay vì commit node_modules?**

A: `node_modules` có thể nặng hàng trăm MB và chứa hàng nghìn files — làm chậm git operations và tăng thời gian checkout. `@vercel/ncc` compile tất cả thành một file duy nhất (~100KB–1MB), giảm thời gian khởi động action và giữ repo sạch sẽ.

**Q: `node16` vs `node20` trong action.yml — chọn cái nào?**

A: `node20` (Node.js 20 LTS). `node16` đã deprecated từ GitHub Actions. Dùng `node20` cho tất cả action mới; migration từ node16 thường chỉ cần update `using:` trong action.yml và rebuild bundle.

**Q: Làm thế nào để action không làm lộ sensitive data trong logs?**

A: Dùng `core.setSecret(value)` để mask giá trị trong mọi log output. GitHub Actions tự động mask các `${{ secrets.* }}` values, nhưng nếu xử lý secrets trong code (ví dụ parse JWT), cần mask tường minh.

**Q: Khi nào nên dùng JavaScript action thay vì composite action?**

A: JavaScript action phù hợp khi cần: (1) Gọi API với error handling phức tạp, (2) Xử lý JSON/dữ liệu với logic điều kiện nhiều nhánh, (3) Tích hợp external services có SDK riêng, (4) Xây dựng action public lên Marketplace. Composite action đủ dùng cho sequencing shell commands và setup tasks đơn giản.

---

**Cập Nhật Lần Cuối:** 2026-05-11
**Trạng Thái:** ✅ Hoàn Thành
