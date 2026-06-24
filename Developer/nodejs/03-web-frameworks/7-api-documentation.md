# API Documentation — Swagger/OpenAPI, Scalar và Versioning

> API documentation (tài liệu API) là contract giữa backend và frontend/consumer. OpenAPI (trước đây Swagger) là chuẩn de facto cho REST API documentation và code generation.

## Mục Lục

1. [OpenAPI Là Gì](#openapi-là-gì)
2. [OpenAPI Specification Structure](#openapi-specification-structure)
3. [Swagger với Express](#swagger-với-express)
4. [NestJS Swagger Integration](#nestjs-swagger-integration)
5. [Scalar — Modern API Reference UI](#scalar--modern-api-reference-ui)
6. [Generate Spec từ Zod](#generate-spec-từ-zod)
7. [API Versioning trong Documentation](#api-versioning-trong-documentation)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## OpenAPI Là Gì

| Thuật Ngữ | Mô Tả |
| --------- | ----- |
| **OpenAPI Specification (OAS)** | Chuẩn mô tả REST API — paths, methods, schemas, auth |
| **Swagger** | Bộ công cụ xung quanh OpenAPI (UI, Editor, Codegen) — tên cũ của spec |
| **Swagger UI** | Interactive documentation — try API trong browser |
| **Scalar** | Modern alternative cho Swagger UI — UX tốt hơn |
| **Redoc** | Static documentation renderer — đẹp, read-only |

**OpenAPI 3.1** (hiện tại) hỗ trợ JSON Schema draft 2020-12.

---

## OpenAPI Specification Structure

```yaml
openapi: 3.1.0
info:
  title: User API
  version: 1.0.0
  description: API quản lý users
  contact:
    name: API Support
    email: support@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: http://localhost:3000/api/v1
    description: Development

paths:
  /users:
    get:
      summary: List users
      tags: [Users]
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
    post:
      summary: Create user
      tags: [Users]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUser'
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'

  /users/{id}:
    get:
      summary: Get user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: Not found

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        email:
          type: string
          format: email
        name:
          type: string
      required: [id, email, name]

    CreateUser:
      type: object
      properties:
        email:
          type: string
          format: email
        name:
          type: string
          minLength: 2
      required: [email, name]

    UserList:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/User'
        meta:
          type: object
          properties:
            page: { type: integer }
            limit: { type: integer }
            total: { type: integer }

security:
  - bearerAuth: []
```

---

## Swagger với Express

### swagger-jsdoc + swagger-ui-express

```bash
npm install swagger-jsdoc swagger-ui-express
```

```javascript
const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const options = {
  definition: {
    openapi: '3.1.0',
    info: {
      title: 'My API',
      version: '1.0.0',
    },
    servers: [{ url: 'http://localhost:3000/api/v1' }],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
        },
      },
    },
  },
  apis: ['./src/routes/*.js'], // Files chứa JSDoc annotations
};

const swaggerSpec = swaggerJsdoc(options);

app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec, {
  explorer: true,
  customSiteTitle: 'My API Docs',
}));

// Serve raw spec
app.get('/docs.json', (req, res) => res.json(swaggerSpec));
```

**JSDoc annotations trong routes:**

```javascript
/**
 * @swagger
 * /users:
 *   get:
 *     summary: List all users
 *     tags: [Users]
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *     responses:
 *       200:
 *         description: List of users
 *         content:
 *           application/json:
 *             schema:
 *               type: array
 *               items:
 *                 $ref: '#/components/schemas/User'
 */
router.get('/users', listUsers);

/**
 * @swagger
 * components:
 *   schemas:
 *     User:
 *       type: object
 *       required: [email, name]
 *       properties:
 *         id:
 *           type: string
 *           format: uuid
 *         email:
 *           type: string
 *           format: email
 *         name:
 *           type: string
 */
```

---

## NestJS Swagger Integration

NestJS có `@nestjs/swagger` tích hợp sâu:

```bash
npm install @nestjs/swagger
```

```typescript
// main.ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

const config = new DocumentBuilder()
  .setTitle('User API')
  .setDescription('API documentation')
  .setVersion('1.0')
  .addBearerAuth()
  .addTag('users')
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('docs', app, document);
```

**Decorators trên DTO và Controller:**

```typescript
import { ApiProperty, ApiTags, ApiOperation, ApiResponse, ApiBearerAuth } from '@nestjs/swagger';

@ApiTags('users')
@Controller('users')
export class UsersController {
  @Get()
  @ApiOperation({ summary: 'List users' })
  @ApiResponse({ status: 200, description: 'Success', type: [UserResponseDto] })
  findAll() { /* ... */ }

  @Post()
  @ApiBearerAuth()
  @ApiResponse({ status: 201, type: UserResponseDto })
  @ApiResponse({ status: 400, description: 'Validation error' })
  create(@Body() dto: CreateUserDto) { /* ... */ }
}

export class CreateUserDto {
  @ApiProperty({ example: 'john@example.com' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'John Doe', minLength: 2 })
  @IsString()
  name: string;
}
```

---

## Scalar — Modern API Reference UI

[Scalar](https://scalar.com/) — API reference UI hiện đại, thay thế Swagger UI:

```bash
npm install @scalar/express-api-reference
```

```javascript
const { apiReference } = require('@scalar/express-api-reference');

app.use('/docs', apiReference({
  spec: {
    content: swaggerSpec, // OpenAPI spec object
  },
  theme: 'purple',
}));
```

**Ưu điểm Scalar vs Swagger UI:**

| Feature | Swagger UI | Scalar |
| ------- | ---------- | ------ |
| UX/UI | Functional | Modern, polished |
| Dark mode | Basic | Native |
| Code examples | Limited | Multi-language |
| Search | Basic | Fast full-text |
| Mobile | Poor | Responsive |

**Redoc alternative:**

```bash
npm install redoc-express
```

```javascript
const redoc = require('redoc-express');
app.get('/docs', redoc({ title: 'API Docs', specUrl: '/docs.json' }));
```

---

## Generate Spec từ Zod

Tránh duplicate schema giữa validation và documentation:

```bash
npm install @asteasolutions/zod-to-openapi
```

```typescript
import { extendZodWithOpenApi } from '@asteasolutions/zod-to-openapi';
import { OpenAPIRegistry, OpenApiGeneratorV3 } from '@asteasolutions/zod-to-openapi';

extendZodWithOpenApi(z);

const UserSchema = z.object({
  id: z.string().uuid().openapi({ example: '550e8400-e29b-41d4-a716-446655440000' }),
  email: z.string().email().openapi({ example: 'john@example.com' }),
  name: z.string().min(2).openapi({ example: 'John Doe' }),
}).openapi('User');

const registry = new OpenAPIRegistry();
registry.register('User', UserSchema);

registry.registerPath({
  method: 'get',
  path: '/users/{id}',
  summary: 'Get user by ID',
  request: {
    params: z.object({ id: z.string().uuid() }),
  },
  responses: {
    200: {
      description: 'User found',
      content: { 'application/json': { schema: UserSchema } },
    },
  },
});

const generator = new OpenApiGeneratorV3(registry.definitions);
const openApiDoc = generator.generateDocument({
  openapi: '3.1.0',
  info: { title: 'API', version: '1.0.0' },
});
```

**Lợi ích:** Single source of truth — Zod schema vừa validate vừa generate OpenAPI spec.

---

## API Versioning trong Documentation

### Multiple Versions

```yaml
# openapi-v1.yaml
info:
  version: 1.0.0
servers:
  - url: https://api.example.com/v1

# openapi-v2.yaml
info:
  version: 2.0.0
servers:
  - url: https://api.example.com/v2
```

```javascript
// Serve multiple doc versions
app.use('/docs/v1', swaggerUi.serve, swaggerUi.setup(specV1));
app.use('/docs/v2', swaggerUi.serve, swaggerUi.setup(specV2));
app.use('/docs', (req, res) => res.redirect('/docs/v2')); // Latest
```

### Deprecation trong Spec

```yaml
paths:
  /users:
    get:
      deprecated: true
      description: |
        **Deprecated** — Use GET /v2/users instead.
        Sunset date: 2027-01-01
```

---

## Best Practices

| Practice | Chi Tiết |
| -------- | -------- |
| **Docs as code** | Spec trong repo, version cùng API code |
| **CI validate spec** | `swagger-cli validate openapi.yaml` trong pipeline |
| **Examples thực tế** | Mỗi endpoint có request/response examples |
| **Document errors** | Liệt kê 400, 401, 404, 500 responses |
| **Auth documented** | Security schemes rõ ràng |
| **Keep in sync** | Auto-generate từ code/DTO — tránh drift |
| **Changelog** | Document breaking changes giữa versions |
| **Try it out** | Enable trong dev/staging, disable production nếu sensitive |

**Workflow khuyến nghị:**

```
1. Design schema (Zod/DTO)
2. Implement endpoint
3. Auto-generate hoặc annotate OpenAPI
4. CI validate spec
5. Deploy docs cùng API
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: OpenAPI vs Postman Collection?

**Trả lời:** OpenAPI là chuẩn mở, machine-readable, hỗ trợ code generation và nhiều tools. Postman Collection là format proprietary của Postman — tốt cho testing/manual exploration. OpenAPI phù hợp làm source of truth; import vào Postman khi cần test.

### Câu 2: Làm sao giữ docs sync với code?

**Trả lời:** Auto-generate từ decorators (NestJS), Zod schemas (`zod-to-openapi`), hoặc route annotations. CI test so sánh generated spec với committed spec. Code review bắt buộc update docs khi thay đổi API.

### Câu 3: Swagger UI trong production — nên hay không?

**Trả lời:** Public API: nên — giúp developers integrate. Internal API: có thể restrict bằng auth hoặc chỉ expose trong staging. Sensitive APIs: disable "Try it out", hoặc không expose docs publicly. Luôn rate limit docs endpoint.

### Câu 4: Contract-first vs Code-first API design?

**Trả lời:** Contract-first: viết OpenAPI spec trước, generate code/stubs, frontend và backend develop parallel. Code-first: implement code, generate spec sau — nhanh hơn cho MVP. Enterprise thường contract-first; startup thường code-first.

---

**Hoàn thành chủ đề:** Quay lại [README.md](./README.md) hoặc tiếp tục [04-data-access/](../04-data-access/).
