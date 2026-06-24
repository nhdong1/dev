# Secrets Management — Environment Variables, Vault và AWS Secrets Manager

> **Secrets (bí mật)** — API keys, database passwords, JWT signing keys — không bao giờ commit vào Git. Quản lý đúng cách là yêu cầu bắt buộc cho production Node.js applications.

## Mục Lục

1. [Secrets Là Gì](#secrets-là-gì)
2. [12-Factor App — Config](#12-factor-app--config)
3. [dotenv và Environment Variables](#dotenv-và-environment-variables)
4. [Validate Env at Startup](#validate-env-at-startup)
5. [.env Security Best Practices](#env-security-best-practices)
6. [HashiCorp Vault](#hashicorp-vault)
7. [AWS Secrets Manager](#aws-secrets-manager)
8. [Secret Rotation](#secret-rotation)
9. [CI/CD Secrets](#cicd-secrets)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Secrets Là Gì

| Loại Secret | Ví Dụ | Rủi Ro Nếu Lộ |
| ----------- | ----- | --------------- |
| **Database credentials** | `DATABASE_URL` | Full DB access |
| **JWT signing keys** | `JWT_SECRET` | Forge tokens |
| **API keys** | Stripe, SendGrid, AWS | Financial abuse |
| **OAuth client secrets** | Google, GitHub | Impersonate app |
| **Encryption keys** | AES-256 key | Decrypt sensitive data |
| **Session secrets** | `SESSION_SECRET` | Session hijacking |

---

## 12-Factor App — Config

[12-Factor App](https://12factor.net/config) nguyên tắc #3: **Config trong environment**, không trong code.

```
❌ const DB_PASSWORD = 'supersecret123';
❌ config.json committed to Git
✅ process.env.DATABASE_URL
✅ Secrets injected at runtime (K8s Secret, Vault, AWS SM)
```

---

## dotenv và Environment Variables

```bash
npm install dotenv
```

```typescript
// Chỉ load .env trong development — production dùng injected env
import dotenv from 'dotenv';

if (process.env.NODE_ENV !== 'production') {
  dotenv.config();
}
```

### .env File (Local Development Only)

```env
# .env — KHÔNG COMMIT
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
JWT_SECRET=dev-only-secret-min-32-chars-long!!
JWT_REFRESH_SECRET=dev-refresh-secret-min-32-chars!!
REDIS_URL=redis://localhost:6379
```

### .gitignore

```gitignore
.env
.env.local
.env.*.local
*.pem
*.key
```

### .env.example (Commit This)

```env
# .env.example — template without real values
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
JWT_SECRET=change-me-min-32-characters
JWT_REFRESH_SECRET=change-me-min-32-characters
REDIS_URL=redis://localhost:6379
```

---

## Validate Env at Startup

**Fail fast** nếu thiếu hoặc invalid secrets — không crash giữa request:

```typescript
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_REFRESH_SECRET: z.string().min(32),
  REDIS_URL: z.string().url().optional(),
});

export type Env = z.infer<typeof envSchema>;

function validateEnv(): Env {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    console.error('❌ Invalid environment variables:');
    console.error(result.error.flatten().fieldErrors);
    process.exit(1);
  }
  return result.data;
}

export const env = validateEnv();
```

```typescript
// app.ts — validate trước khi start server
import { env } from './config/env';

const app = express();
app.listen(env.PORT, () => {
  console.log(`Server running on port ${env.PORT}`);
});
```

---

## .env Security Best Practices

| Practice | Mô Tả |
| -------- | ----- |
| **Không commit `.env`** | `.gitignore` + pre-commit hook scan |
| **Different secrets per env** | dev/staging/prod keys khác nhau |
| **Min entropy** | JWT secret ≥ 32 bytes random |
| **Không log secrets** | Mask trong logs: `db://***@host` |
| **Không expose qua API** | `/config`, `/debug` endpoints |
| **Scan Git history** | `git-secrets`, `trufflehog` nếu accidentally commit |

```typescript
// Generate secure random secret
import { randomBytes } from 'crypto';
const secret = randomBytes(32).toString('hex');
console.log(secret); // Dùng một lần, lưu vào secrets manager
```

---

## HashiCorp Vault

**Vault** — centralized secrets management với dynamic secrets, rotation, audit log.

```bash
npm install node-vault
```

```typescript
import vault from 'node-vault';

const vaultClient = vault({
  endpoint: process.env.VAULT_ADDR || 'http://localhost:8200',
  token: process.env.VAULT_TOKEN, // Dev only — production dùng AppRole/K8s auth
});

async function getDatabaseCredentials(): Promise<{ username: string; password: string }> {
  const { data } = await vaultClient.read('secret/data/myapp/database');
  return {
    username: data.data.username,
    password: data.data.password,
  };
}

// Dynamic secrets — Vault tạo short-lived DB credentials
async function getDynamicDbCreds() {
  const { data } = await vaultClient.read('database/creds/myapp-role');
  return { username: data.username, password: data.password, leaseDuration: data.lease_duration };
}
```

### Vault trong Kubernetes

```yaml
# Inject secrets via Vault Agent Sidecar hoặc CSI Driver
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: app-secrets
spec:
  vaultAuthRef: app-auth
  mount: secret
  type: kv-v2
  path: myapp/production
  destination:
    name: app-secrets
    create: true
```

---

## AWS Secrets Manager

```bash
npm install @aws-sdk/client-secrets-manager
```

```typescript
import {
  SecretsManagerClient,
  GetSecretValueCommand,
} from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: 'ap-southeast-1' });

interface AppSecrets {
  DATABASE_URL: string;
  JWT_SECRET: string;
  STRIPE_API_KEY: string;
}

let cachedSecrets: AppSecrets | null = null;

export async function loadSecrets(): Promise<AppSecrets> {
  if (cachedSecrets) return cachedSecrets;

  const command = new GetSecretValueCommand({
    SecretId: 'prod/myapp/secrets',
  });

  const response = await client.send(command);
  cachedSecrets = JSON.parse(response.SecretString!) as AppSecrets;
  return cachedSecrets;
}

// Bootstrap trước khi start app
async function bootstrap() {
  const secrets = await loadSecrets();
  process.env.DATABASE_URL = secrets.DATABASE_URL;
  process.env.JWT_SECRET = secrets.JWT_SECRET;

  const { env } = await import('./config/env');
  // Start server...
}

bootstrap().catch(err => {
  console.error('Failed to load secrets:', err);
  process.exit(1);
});
```

### ECS/EKS Integration

- **ECS:** Inject secrets qua task definition `secrets` field
- **EKS:** External Secrets Operator sync AWS SM → K8s Secret
- **Lambda:** Environment variables từ Secrets Manager reference

---

## Secret Rotation

| Secret | Rotation Frequency | Strategy |
| ------ | ------------------ | -------- |
| JWT signing key | 90 ngày | Dual keys — old + new valid during transition |
| Database password | 30–90 ngày | Vault dynamic credentials |
| API keys | On compromise | Immediate revoke + reissue |
| OAuth client secret | Yearly | Provider dashboard rotation |

### JWT Key Rotation với `kid`

```typescript
const keys = {
  'key-2024-01': process.env.JWT_SECRET_OLD,
  'key-2024-06': process.env.JWT_SECRET,
};

function verifyToken(token: string) {
  const decoded = jwt.decode(token, { complete: true });
  const kid = decoded?.header.kid;
  const secret = keys[kid as keyof typeof keys];
  if (!secret) throw new Error('Unknown key');
  return jwt.verify(token, secret, { algorithms: ['HS256'] });
}

function signToken(payload: object) {
  return jwt.sign(payload, keys['key-2024-06'], {
    algorithm: 'HS256',
    keyid: 'key-2024-06',
  });
}
```

---

## CI/CD Secrets

```yaml
# GitHub Actions — secrets qua repository settings
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
      JWT_SECRET: ${{ secrets.JWT_SECRET }}
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
      # Không echo secrets trong logs
```

**Best practices:**
- GitHub/GitLab encrypted secrets — không plain text trong workflow files
- OIDC federation thay vì long-lived AWS keys
- Separate secrets per environment (staging vs production)
- Audit secret access

---

## Best Practices

1. **Never hardcode secrets** — kể cả "temporary" ones
2. **Validate env at startup** — Zod schema, fail fast
3. **Least privilege** — DB user chỉ có permissions cần thiết
4. **Separate secrets per environment** — dev ≠ prod
5. **Rotate regularly** — automate với Vault/AWS SM
6. **Audit access** — log who accessed which secret
7. **Scan for leaked secrets** — `trufflehog`, `gitleaks` in CI
8. **Encrypt at rest** — K8s Secrets are base64, not encrypted by default — dùng Sealed Secrets hoặc external SM
9. **Không pass secrets qua query strings** — chỉ headers hoặc body

---

## Câu Hỏi Phỏng Vấn

### Câu 1: `.env` file có an toàn trong production không?

**Trả lời:** Không khuyến nghị — file-based secrets khó rotate, audit, và dễ leak qua misconfiguration. Production nên dùng Vault, AWS Secrets Manager, hoặc K8s Secrets với external provider.

### Câu 2: Làm sao handle secrets trong Docker?

**Trả lời:** Không `ENV` secrets trong Dockerfile. Inject runtime qua `docker run -e`, Docker Compose `env_file` (local), hoặc orchestrator secrets (K8s Secret, ECS secrets). Multi-stage build không copy `.env`.

### Câu 3: JWT secret bao nhiêu entropy là đủ?

**Trả lời:** Minimum 256 bits (32 bytes) random. Generate bằng `crypto.randomBytes(32)`. Không dùng dictionary words hoặc predictable strings.

### Câu 4: Secret rotation without downtime?

**Trả lời:** Dual-key period — accept tokens signed bằng old và new key simultaneously. Database: connection pool reconnect với new credentials. Vault dynamic secrets tự động handle.

---

**Quay lại:** [README.md](./README.md) — Tổng quan chủ đề Bảo Mật Node.js API.
