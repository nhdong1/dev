# Authorization và RBAC — Role-Based Access Control

> Authentication xác định *ai* đang gọi API; **Authorization (Phân Quyền)** xác định *họ được phép làm gì*. RBAC (Role-Based Access Control — Phân Quyền Theo Vai Trò) là pattern phổ biến nhất trong Node.js Backend.

## Mục Lục

1. [Authentication vs Authorization](#authentication-vs-authorization)
2. [RBAC Model](#rbac-model)
3. [Permissions Matrix](#permissions-matrix)
4. [Implement RBAC Middleware](#implement-rbac-middleware)
5. [Resource-Level Authorization](#resource-level-authorization)
6. [ABAC — Attribute-Based Access Control](#abac--attribute-based-access-control)
7. [NestJS Guards và Decorators](#nestjs-guards-và-decorators)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Authentication vs Authorization

```
POST /auth/login     → Authentication: verify password, issue JWT
GET  /api/users      → Authorization: user có role 'admin' không?
GET  /api/users/123  → Authorization: user 123 có phải chính mình hoặc admin?
DELETE /api/users/456 → Authorization: chỉ admin được xóa user khác
```

**OWASP A01 — Broken Access Control:** Lỗi phổ biến nhất — chỉ check login mà không check quyền trên resource cụ thể.

---

## RBAC Model

```
User ──has──► Role(s) ──grants──► Permission(s) ──on──► Resource/Action
```

| Thành Phần | Ví Dụ |
| ---------- | ----- |
| **User** | `alice@example.com` |
| **Role** | `admin`, `editor`, `viewer` |
| **Permission** | `users:read`, `users:write`, `posts:delete` |
| **Resource** | `users`, `posts`, `orders` |
| **Action** | `create`, `read`, `update`, `delete` (CRUD) |

### Database Schema Điển Hình

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE roles (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL  -- 'admin', 'editor', 'viewer'
);

CREATE TABLE user_roles (
  user_id UUID REFERENCES users(id),
  role_id INT REFERENCES roles(id),
  PRIMARY KEY (user_id, role_id)
);

CREATE TABLE permissions (
  id SERIAL PRIMARY KEY,
  resource VARCHAR(50) NOT NULL,   -- 'users'
  action VARCHAR(50) NOT NULL,     -- 'delete'
  UNIQUE (resource, action)
);

CREATE TABLE role_permissions (
  role_id INT REFERENCES roles(id),
  permission_id INT REFERENCES permissions(id),
  PRIMARY KEY (role_id, permission_id)
);
```

---

## Permissions Matrix

| Role | users:read | users:write | users:delete | posts:read | posts:write |
| ---- | ---------- | ----------- | ------------ | ---------- | ----------- |
| **admin** | ✅ all | ✅ all | ✅ | ✅ | ✅ |
| **editor** | ✅ own | ✅ own | ❌ | ✅ all | ✅ all |
| **viewer** | ✅ own | ❌ | ❌ | ✅ all | ❌ |

**Own vs All:** `editor` đọc mọi post nhưng chỉ sửa profile của mình — cần resource-level check.

---

## Implement RBAC Middleware

### Role Check Middleware

```typescript
import { Response, NextFunction } from 'express';
import { AuthRequest } from './auth.middleware';

export function requireRole(...allowedRoles: string[]) {
  return (req: AuthRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const hasRole = req.user.roles.some(role => allowedRoles.includes(role));
    if (!hasRole) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    next();
  };
}

// Usage
app.delete('/api/users/:id',
  authenticate,
  requireRole('admin'),
  deleteUserHandler
);
```

### Permission Check Middleware

```typescript
export function requirePermission(resource: string, action: string) {
  return async (req: AuthRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const permissions = await getPermissionsForUser(req.user.sub);
    const required = `${resource}:${action}`;

    if (!permissions.includes(required)) {
      return res.status(403).json({
        error: 'Forbidden',
        required: required,
      });
    }

    next();
  };
}

app.post('/api/posts',
  authenticate,
  requirePermission('posts', 'create'),
  createPostHandler
);
```

### Embed Permissions trong JWT (Trade-off)

```typescript
// JWT payload
{ "sub": "user-123", "permissions": ["users:read", "posts:write"] }
```

**Ưu điểm:** Không query DB mỗi request.  
**Nhược điểm:** Permission change không có hiệu lực đến khi token expire — cần short TTL hoặc permission version.

---

## Resource-Level Authorization

Kiểm tra user có quyền trên **resource cụ thể**, không chỉ role chung:

```typescript
export async function canAccessUser(
  requesterId: string,
  requesterRoles: string[],
  targetUserId: string
): Promise<boolean> {
  if (requesterRoles.includes('admin')) return true;
  return requesterId === targetUserId;
}

app.get('/api/users/:id', authenticate, async (req: AuthRequest, res) => {
  const targetId = req.params.id;

  if (!(await canAccessUser(req.user!.sub, req.user!.roles, targetId))) {
    return res.status(403).json({ error: 'Cannot access this user' });
  }

  const user = await findUserById(targetId);
  res.json(user);
});
```

### IDOR Prevention (Insecure Direct Object Reference)

```typescript
// ❌ Broken — user đổi :id trong URL truy cập data người khác
app.get('/api/orders/:id', authenticate, async (req, res) => {
  const order = await findOrderById(req.params.id);
  res.json(order);
});

// ✅ Fixed — luôn filter theo owner
app.get('/api/orders/:id', authenticate, async (req: AuthRequest, res) => {
  const order = await findOrderByIdAndUserId(req.params.id, req.user!.sub);
  if (!order) return res.status(404).json({ error: 'Not found' });
  res.json(order);
});
```

**Nguyên tắc:** Query luôn include `WHERE user_id = :currentUserId` trừ khi là admin.

---

## ABAC — Attribute-Based Access Control

RBAC cứng nhắc khi cần rules phức tạp. **ABAC (Attribute-Based Access Control — Phân Quyền Theo Thuộc Tính)** dựa trên attributes:

```typescript
interface AccessContext {
  subject: { id: string; department: string; role: string };
  resource: { type: string; ownerId: string; department: string };
  action: string;
  environment: { time: Date; ip: string };
}

function canAccess(ctx: AccessContext): boolean {
  // Admin bypass
  if (ctx.subject.role === 'admin') return true;

  // Owner access
  if (ctx.resource.ownerId === ctx.subject.id) return true;

  // Same department read
  if (ctx.action === 'read' && ctx.resource.department === ctx.subject.department) {
    return true;
  }

  // Business hours only for sensitive actions
  if (ctx.action === 'export' && !isBusinessHours(ctx.environment.time)) {
    return false;
  }

  return false;
}
```

| Pattern | Phù Hợp |
| ------- | ------- |
| **RBAC** | Startup, CRUD apps, roles rõ ràng |
| **ABAC** | Enterprise, multi-tenant, complex policies |
| **Hybrid** | RBAC cho coarse-grained + ABAC cho fine-grained |

---

## NestJS Guards và Decorators

```typescript
// roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user.roles?.includes(role));
  }
}

// controller
@Roles('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
@Delete('users/:id')
deleteUser(@Param('id') id: string) {
  return this.usersService.delete(id);
}
```

---

## Best Practices

1. **Deny by default** — mọi route protected trừ khi explicitly public
2. **Server-side checks** — không tin client-side role UI
3. **Resource-level authorization** — không chỉ route-level
4. **Log access denied** — audit trail cho security incidents
5. **Principle of least privilege** — role tối thiểu cần thiết
6. **Separate admin routes** — `/admin/*` với stricter checks
7. **Test authorization** — unit test "user A cannot access user B data"

---

## Câu Hỏi Phỏng Vấn

### Câu 1: RBAC vs ABAC — khi nào dùng gì?

**Trả lời:** RBAC đơn giản, dễ maintain — phù hợp hầu hết apps. ABAC khi cần policies phức tạp (department, time, resource attributes). Nhiều hệ thống dùng hybrid.

### Câu 2: IDOR là gì? Fix trong Node.js?

**Trả lời:** Insecure Direct Object Reference — user thay ID trong URL truy cập resource người khác. Fix: luôn authorize ownership trong query (`WHERE user_id = ?`), không chỉ authenticate.

### Câu 3: Permission trong JWT có nên không?

**Trả lời:** Có thể cho read-heavy, ít thay đổi permission — kèm short TTL. Sensitive permission changes cần revoke hoặc query DB. Enterprise thường check permissions server-side mỗi request.

### Câu 4: 403 vs 401 — khi nào dùng gì?

**Trả lời:** 401 Unauthorized — chưa authenticate hoặc token invalid. 403 Forbidden — đã authenticate nhưng không có quyền. Không dùng 404 che giấu existence (trừ khi policy yêu cầu).

---

**Xem tiếp:** [4-owasp-top10.md](./4-owasp-top10.md) — OWASP Top 10 vulnerabilities và prevention.
