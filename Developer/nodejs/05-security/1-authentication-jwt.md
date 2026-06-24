# Authentication với JWT — Access Token, Refresh Token và Token Rotation

> JWT (JSON Web Token — Mã Thông Báo Web JSON) là chuẩn phổ biến nhất cho stateless authentication trong Node.js API. Hiểu rõ cấu trúc, security considerations, và refresh token rotation là bắt buộc cho phỏng vấn Backend.

## Mục Lục

1. [JWT Là Gì](#jwt-là-gì)
2. [Cấu Trúc JWT](#cấu-trúc-jwt)
3. [Signing Algorithms](#signing-algorithms)
4. [Access Token vs Refresh Token](#access-token-vs-refresh-token)
5. [Implement JWT với Node.js](#implement-jwt-với-nodejs)
6. [Token Storage — Cookie vs Header](#token-storage--cookie-vs-header)
7. [Token Rotation và Revocation](#token-rotation-và-revocation)
8. [Common JWT Attacks](#common-jwt-attacks)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## JWT Là Gì

JWT là **compact, URL-safe token** chứa claims (thông tin xác thực) được ký cryptographically. Server verify signature để tin tưởng payload mà không cần query database mỗi request.

```
xxxxx.yyyyy.zzzzz
Header.Payload.Signature
```

**Ưu điểm:** Stateless, scale horizontal, cross-service (microservices).  
**Nhược điểm:** Khó revoke trước expiry, payload không encrypted (chỉ encoded Base64).

---

## Cấu Trúc JWT

### Header

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-1"
}
```

### Payload (Claims)

```json
{
  "iss": "auth.example.com",
  "sub": "user-123",
  "aud": "api.example.com",
  "exp": 1706000000,
  "iat": 1705999000,
  "jti": "unique-token-id",
  "roles": ["user"]
}
```

| Claim | Ý Nghĩa |
| ----- | ------- |
| `iss` (Issuer) | Ai phát hành token |
| `sub` (Subject) | User ID |
| `aud` (Audience) | API nào được phép accept token |
| `exp` (Expiration) | Thời điểm hết hạn (Unix timestamp) |
| `iat` (Issued At) | Thời điểm phát hành |
| `jti` (JWT ID) | Unique ID — dùng cho blacklist/revocation |

**Quan trọng:** JWT payload chỉ Base64URL encoded — **không phải encrypted**. Không lưu password, SSN, hoặc data nhạy cảm trong payload.

---

## Signing Algorithms

| Algorithm | Loại | Use Case |
| --------- | ---- | -------- |
| **HS256** | Symmetric (HMAC) | Monolith — một secret shared |
| **RS256** | Asymmetric (RSA) | Microservices — Auth Server ký private key, API verify public key |
| **ES256** | Asymmetric (ECDSA) | Tương tự RS256, key nhỏ hơn |

```typescript
// HS256 — đơn giản, monolith
import jwt from 'jsonwebtoken';
const token = jwt.sign({ sub: 'user-123' }, process.env.JWT_SECRET!, {
  expiresIn: '15m',
  algorithm: 'HS256',
});

// RS256 — microservices
const token = jwt.sign({ sub: 'user-123' }, privateKey, {
  algorithm: 'RS256',
  keyid: 'key-2024-01',
});
const payload = jwt.verify(token, publicKey, { algorithms: ['RS256'] });
```

**Luôn whitelist algorithm** khi verify — chống Algorithm Confusion attack.

---

## Access Token vs Refresh Token

| | Access Token | Refresh Token |
| - | ------------ | ------------- |
| **Thời hạn** | Ngắn (5–15 phút) | Dài (7–30 ngày) |
| **Dùng cho** | Mọi API request | Chỉ endpoint `/auth/refresh` |
| **Lưu trữ** | Memory hoặc short-lived cookie | HttpOnly Secure cookie |
| **Chứa gì** | `sub`, `roles`, permissions | Chỉ `sub` + `jti` (opaque hoặc JWT) |

```
Login ──► access_token (15m) + refresh_token (7d)
              │
              ▼
API calls với access_token trong Authorization header
              │
              ▼ (access_token expired)
POST /auth/refresh với refresh_token
              │
              ▼
New access_token + new refresh_token (rotation)
```

---

## Implement JWT với Node.js

```bash
npm install jsonwebtoken bcryptjs zod
npm install -D @types/jsonwebtoken @types/bcryptjs
```

### Password Hashing với bcrypt

```typescript
import bcrypt from 'bcryptjs';

const SALT_ROUNDS = 12;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

### Issue Tokens

```typescript
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

const ACCESS_EXPIRY = '15m';
const REFRESH_EXPIRY = '7d';

interface TokenPayload {
  sub: string;
  roles: string[];
}

export function issueAccessToken(payload: TokenPayload): string {
  return jwt.sign(payload, process.env.JWT_SECRET!, {
    expiresIn: ACCESS_EXPIRY,
    issuer: 'auth.example.com',
    audience: 'api.example.com',
  });
}

export function issueRefreshToken(userId: string): string {
  return jwt.sign(
    { sub: userId, jti: crypto.randomUUID() },
    process.env.JWT_REFRESH_SECRET!,
    { expiresIn: REFRESH_EXPIRY }
  );
}
```

### Auth Middleware

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

export interface AuthRequest extends Request {
  user?: { sub: string; roles: string[] };
}

export function authenticate(req: AuthRequest, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing or invalid token' });
  }

  const token = authHeader.slice(7);
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!, {
      algorithms: ['HS256'],
      issuer: 'auth.example.com',
      audience: 'api.example.com',
    }) as { sub: string; roles: string[] };

    req.user = { sub: payload.sub, roles: payload.roles };
    next();
  } catch (err) {
    if (err instanceof jwt.TokenExpiredError) {
      return res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### Login Endpoint

```typescript
app.post('/auth/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await findUserByEmail(email);
  if (!user || !(await verifyPassword(password, user.passwordHash))) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const accessToken = issueAccessToken({ sub: user.id, roles: user.roles });
  const refreshToken = issueRefreshToken(user.id);

  // Lưu refresh token hash vào DB/Redis để validate rotation
  await storeRefreshToken(user.id, hashToken(refreshToken));

  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000,
  });

  res.json({ accessToken });
});
```

---

## Token Storage — Cookie vs Header

| Phương Pháp | Ưu Điểm | Nhược Điểm |
| ----------- | ------- | ---------- |
| **Authorization: Bearer** | Đơn giản, SPA/mobile friendly | Dễ bị XSS steal nếu lưu localStorage |
| **HttpOnly Cookie** | JavaScript không đọc được — chống XSS | Cần CSRF protection |
| **Memory only (SPA)** | Không persist qua refresh | Mất khi đóng tab |

**Khuyến nghị production:**
- Access token: short-lived, có thể trong memory hoặc Authorization header
- Refresh token: **HttpOnly + Secure + SameSite=Strict** cookie

---

## Token Rotation và Revocation

### Refresh Token Rotation

Mỗi lần refresh, phát hành **cả access token và refresh token mới**, invalidate token cũ:

```typescript
app.post('/auth/refresh', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) return res.status(401).json({ error: 'No refresh token' });

  try {
    const payload = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET!) as {
      sub: string;
      jti: string;
    };

    const stored = await getRefreshToken(payload.sub, payload.jti);
    if (!stored || stored.revoked) {
      // Possible token reuse attack — revoke all tokens for user
      await revokeAllRefreshTokens(payload.sub);
      return res.status(401).json({ error: 'Invalid refresh token' });
    }

    await revokeRefreshToken(payload.jti);

    const newAccess = issueAccessToken({ sub: payload.sub, roles: stored.roles });
    const newRefresh = issueRefreshToken(payload.sub);
    await storeRefreshToken(payload.sub, hashToken(newRefresh));

    res.cookie('refreshToken', newRefresh, { httpOnly: true, secure: true, sameSite: 'strict' });
    res.json({ accessToken: newAccess });
  } catch {
    return res.status(401).json({ error: 'Invalid refresh token' });
  }
});
```

### Revocation Strategies

| Strategy | Mô Tả |
| -------- | ----- |
| **Short TTL** | Access token 15 phút — chấp nhận window nhỏ |
| **Token Blacklist (Redis)** | Lưu `jti` đã revoke, check mỗi request |
| **Refresh Token Store** | Chỉ refresh token trong DB — revoke bằng xóa record |
| **Version Field** | User có `tokenVersion` — increment khi logout all devices |

---

## Common JWT Attacks

### Algorithm None Attack

Attacker set `alg: "none"` — server skip verify. **Fix:** Luôn specify `algorithms: ['HS256']` khi verify.

### Algorithm Confusion (RS256 → HS256)

Attacker dùng public key làm HMAC secret. **Fix:** Whitelist algorithms, dùng thư viện verify đúng cách.

### Token Sidejacking (XSS)

Steal token từ localStorage qua XSS. **Fix:** HttpOnly cookies, CSP headers, sanitize output.

### Không Verify Claims

Chỉ decode Base64 mà không verify signature. **Fix:** Luôn dùng `jwt.verify()`, validate `exp`, `iss`, `aud`.

```typescript
// ❌ KHÔNG BAO GIỜ
const payload = JSON.parse(Buffer.from(token.split('.')[1], 'base64url').toString());

// ✅ ĐÚNG
const payload = jwt.verify(token, secret, { algorithms: ['HS256'] });
```

---

## Best Practices

1. **Access token ngắn** — 5–15 phút
2. **Refresh token rotation** — detect reuse attack
3. **Không lưu sensitive data** trong JWT payload
4. **RS256 cho distributed systems** — không share secret giữa services
5. **Validate tất cả claims** — `exp`, `iss`, `aud`, `sub`
6. **Separate secrets** — `JWT_SECRET` ≠ `JWT_REFRESH_SECRET`
7. **Logout** — revoke refresh token trong DB/Redis
8. **Rate limit** `/login` và `/auth/refresh`

---

## Câu Hỏi Phỏng Vấn

### Câu 1: JWT có stateless không? Làm sao logout?

**Trả lời:** JWT access token stateless về mặt verify; nhưng logout cần state — revoke refresh token trong DB/Redis, hoặc blacklist `jti`, hoặc increment `tokenVersion` trên user record.

### Câu 2: HS256 vs RS256 — khi nào dùng gì?

**Trả lời:** HS256 khi một service ký và verify (monolith). RS256 khi Auth Server ký, nhiều API services verify bằng public key — không cần share secret.

### Câu 3: Refresh token rotation là gì? Tại sao cần?

**Trả lời:** Mỗi refresh phát token mới và invalidate cũ. Nếu attacker steal refresh token và victim cũng refresh — token reuse phát hiện attack, revoke all sessions.

### Câu 4: Lưu JWT trong localStorage có an toàn không?

**Trả lời:** Không nếu có XSS risk — JavaScript đọc được localStorage. Prefer HttpOnly cookie cho refresh token; access token trong memory hoặc short-lived header.

---

**Xem tiếp:** [2-passport-strategies.md](./2-passport-strategies.md) — Passport.js strategies cho OAuth2 và social login.
