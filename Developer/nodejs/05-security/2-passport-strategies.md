# Passport.js Strategies — Local, JWT, OAuth2 và Social Login

> Passport.js là middleware authentication phổ biến nhất cho Express — cung cấp **strategies (chiến lược)** plug-and-play cho Local, JWT, OAuth2, Google, GitHub, và nhiều provider khác.

## Mục Lục

1. [Passport.js Là Gì](#passportjs-là-gì)
2. [Cài Đặt và Cấu Hình Cơ Bản](#cài-đặt-và-cấu-hình-cơ-bản)
3. [Local Strategy — Email/Password](#local-strategy--emailpassword)
4. [JWT Strategy — Stateless API](#jwt-strategy--stateless-api)
5. [OAuth2 Strategy — Google, GitHub](#oauth2-strategy--google-github)
6. [Serialize/Deserialize User](#serializedeserialize-user)
7. [NestJS Passport Integration](#nestjs-passport-integration)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Passport.js Là Gì

Passport là **authentication middleware** cho Node.js — không enforce cách lưu user hay format token, chỉ cung cấp framework strategies:

```
Request ──► passport.authenticate('strategy') ──► Verify credentials ──► req.user
```

| Strategy | Use Case |
| -------- | -------- |
| `passport-local` | Username/password form login |
| `passport-jwt` | Bearer token API authentication |
| `passport-oauth2` | Generic OAuth2 provider |
| `passport-google-oauth20` | Google Sign-In |
| `passport-github2` | GitHub login |

---

## Cài Đặt và Cấu Hình Cơ Bản

```bash
npm install passport passport-local passport-jwt
npm install -D @types/passport @types/passport-local @types/passport-jwt
```

```typescript
import express from 'express';
import passport from 'passport';
import session from 'express-session';

const app = express();

// Session chỉ cần cho OAuth redirect flow (không cần cho pure JWT API)
app.use(session({
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, httpOnly: true, maxAge: 24 * 60 * 60 * 1000 },
}));

app.use(passport.initialize());
app.use(passport.session()); // Chỉ khi dùng session-based OAuth
```

---

## Local Strategy — Email/Password

```typescript
import { Strategy as LocalStrategy } from 'passport-local';
import bcrypt from 'bcryptjs';

passport.use(new LocalStrategy(
  { usernameField: 'email', passwordField: 'password' },
  async (email, password, done) => {
    try {
      const user = await findUserByEmail(email);
      if (!user) {
        return done(null, false, { message: 'Invalid credentials' });
      }

      const valid = await bcrypt.compare(password, user.passwordHash);
      if (!valid) {
        return done(null, false, { message: 'Invalid credentials' });
      }

      return done(null, user);
    } catch (err) {
      return done(err);
    }
  }
));
```

### Login Route

```typescript
app.post('/auth/login',
  passport.authenticate('local', { session: false }),
  (req, res) => {
    const accessToken = issueAccessToken({
      sub: req.user!.id,
      roles: req.user!.roles,
    });
    res.json({ accessToken, user: { id: req.user!.id, email: req.user!.email } });
  }
);
```

`session: false` — không tạo server session, chỉ dùng Passport để verify và trả JWT.

---

## JWT Strategy — Stateless API

```typescript
import { Strategy as JwtStrategy, ExtractJwt } from 'passport-jwt';

passport.use(new JwtStrategy(
  {
    jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
    secretOrKey: process.env.JWT_SECRET!,
    issuer: 'auth.example.com',
    audience: 'api.example.com',
    algorithms: ['HS256'],
  },
  async (payload, done) => {
    try {
      const user = await findUserById(payload.sub);
      if (!user) return done(null, false);
      return done(null, user);
    } catch (err) {
      return done(err, false);
    }
  }
));
```

### Protected Route

```typescript
app.get('/api/profile',
  passport.authenticate('jwt', { session: false }),
  (req, res) => {
    res.json({ user: req.user });
  }
);
```

### Optional Auth Middleware

```typescript
export function optionalAuth(req: Request, res: Response, next: NextFunction) {
  passport.authenticate('jwt', { session: false }, (err, user) => {
    if (user) req.user = user;
    next();
  })(req, res, next);
}
```

---

## OAuth2 Strategy — Google, GitHub

### Google OAuth2

```bash
npm install passport-google-oauth20
```

```typescript
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';

passport.use(new GoogleStrategy(
  {
    clientID: process.env.GOOGLE_CLIENT_ID!,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    callbackURL: '/auth/google/callback',
    scope: ['profile', 'email'],
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      let user = await findUserByProvider('google', profile.id);
      if (!user) {
        user = await createUser({
          email: profile.emails?.[0]?.value,
          name: profile.displayName,
          provider: 'google',
          providerId: profile.id,
        });
      }
      return done(null, user);
    } catch (err) {
      return done(err);
    }
  }
));
```

### OAuth2 Routes

```typescript
app.get('/auth/google',
  passport.authenticate('google', { scope: ['profile', 'email'] })
);

app.get('/auth/google/callback',
  passport.authenticate('google', { session: false, failureRedirect: '/login' }),
  (req, res) => {
    const token = issueAccessToken({ sub: req.user!.id, roles: req.user!.roles });
    res.redirect(`/auth/success?token=${token}`); // Hoặc set cookie
  }
);
```

### OAuth2 Flows

| Flow | Use Case | Node.js |
| ---- | -------- | ------- |
| **Authorization Code** | Server-side web app | Passport redirect flow |
| **Authorization Code + PKCE** | SPA, mobile app | `passport-oauth2` với PKCE params |
| **Client Credentials** | Machine-to-machine | Direct token request, không Passport |

```
User ──► GET /auth/google ──► Google consent screen
     ◄── Redirect /callback?code=xxx
Server ──► Exchange code for tokens (server-to-server)
     ──► Create/find user ──► Issue own JWT
```

---

## Serialize/Deserialize User

Khi dùng **session-based auth** (OAuth redirect), Passport cần serialize user vào session:

```typescript
passport.serializeUser((user: Express.User, done) => {
  done(null, user.id); // Chỉ lưu ID trong session
});

passport.deserializeUser(async (id: string, done) => {
  try {
    const user = await findUserById(id);
    done(null, user);
  } catch (err) {
    done(err);
  }
});
```

Với **JWT stateless API**, thường không cần serialize/deserialize — `session: false` trên mọi route.

---

## NestJS Passport Integration

NestJS wrap Passport trong Guards (Bảo Vệ):

```typescript
// jwt.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: process.env.JWT_SECRET,
    });
  }

  async validate(payload: { sub: string; roles: string[] }) {
    return { userId: payload.sub, roles: payload.roles };
  }
}

// jwt-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

// controller
@UseGuards(JwtAuthGuard)
@Get('profile')
getProfile(@Request() req) {
  return req.user;
}
```

---

## Best Practices

1. **`session: false`** cho JWT API — tránh session fixation
2. **Không leak provider tokens** — chỉ dùng để lấy profile, issue JWT riêng
3. **Link accounts** — cùng email từ Google/GitHub có thể merge vào một user
4. **Validate OAuth state parameter** — chống CSRF trong OAuth flow
5. **Rate limit** OAuth callback endpoints
6. **Store minimal profile data** — không cần lưu toàn bộ OAuth response

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Passport.js khác gì tự implement JWT middleware?

**Trả lời:** Passport cung cấp strategy pattern chuẩn hóa — dễ swap Local/JWT/OAuth2, tích hợp NestJS Guards. Tự implement nhẹ hơn nếu chỉ cần JWT đơn giản.

### Câu 2: OAuth2 Authorization Code flow hoạt động thế nào?

**Trả lời:** User redirect đến provider → authenticate → redirect về callback với `code` → server exchange `code` lấy access token (server-to-server, bảo mật secret) → tạo session/JWT riêng.

### Câu 3: PKCE dùng khi nào?

**Trả lời:** Public clients (SPA, mobile) không giữ được `client_secret`. PKCE (Proof Key for Code Exchange) thêm `code_verifier`/`code_challenge` chống authorization code interception.

### Câu 4: `passport.authenticate` failure xử lý thế nào?

**Trả lời:** Custom callback hoặc `failureRedirect`. Production nên trả JSON 401 thống nhất, không redirect cho API endpoints.

---

**Xem tiếp:** [3-authorization-rbac.md](./3-authorization-rbac.md) — RBAC và resource-level permissions.
