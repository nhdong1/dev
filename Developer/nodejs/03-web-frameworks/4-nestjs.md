# NestJS — Modules, DI, Guards, Interceptors và Pipes

> NestJS là progressive Node.js framework xây dựng trên Express/Fastify, lấy cảm hứng từ Angular — cung cấp kiến trúc có cấu trúc với Dependency Injection (DI — Tiêm Phụ Thuộc), decorators, và enterprise patterns.

## Mục Lục

1. [NestJS Là Gì](#nestjs-là-gì)
2. [Project Structure](#project-structure)
3. [Modules, Controllers, Providers](#modules-controllers-providers)
4. [Dependency Injection](#dependency-injection)
5. [Pipes — Validation và Transformation](#pipes--validation-và-transformation)
6. [Guards — Authentication và Authorization](#guards--authentication-và-authorization)
7. [Interceptors](#interceptors)
8. [Exception Filters](#exception-filters)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## NestJS Là Gì

| Đặc Điểm | Chi Tiết |
| -------- | -------- |
| **Architecture** | Modular, layered (Controller → Service → Repository) |
| **DI Container** | IoC (Inversion of Control — Đảo Ngược Điều Khiển) built-in |
| **HTTP Adapter** | Express (mặc định) hoặc Fastify |
| **TypeScript** | First-class, decorators bắt buộc |
| **Ecosystem** | TypeORM, Prisma, Passport, Swagger tích hợp tốt |

```bash
npm i -g @nestjs/cli
nest new my-api
cd my-api
npm run start:dev
```

---

## Project Structure

```
src/
├── main.ts                 # Bootstrap application
├── app.module.ts           # Root module
├── app.controller.ts
├── app.service.ts
├── users/
│   ├── users.module.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   └── entities/
│       └── user.entity.ts
└── common/
    ├── guards/
    ├── interceptors/
    ├── filters/
    └── pipes/
```

---

## Modules, Controllers, Providers

### Module — Đơn Vị Tổ Chức

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // Cho module khác inject
})
export class UsersModule {}
```

### Controller — HTTP Layer

```typescript
// users/users.controller.ts
import { Controller, Get, Post, Body, Param, ParseUUIDPipe } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.usersService.findOne(id);
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

### Provider (Service) — Business Logic

```typescript
// users/users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';

@Injectable()
export class UsersService {
  private users = [];

  findAll() {
    return this.users;
  }

  findOne(id: string) {
    const user = this.users.find(u => u.id === id);
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  create(dto: CreateUserDto) {
    const user = { id: crypto.randomUUID(), ...dto };
    this.users.push(user);
    return user;
  }
}
```

---

## Dependency Injection

NestJS tự động resolve dependencies qua constructor injection:

```typescript
@Injectable()
export class OrdersService {
  constructor(
    private readonly usersService: UsersService,
    private readonly paymentService: PaymentService,
  ) {}
}

// users.module.ts phải export UsersService
// orders.module.ts import UsersModule
@Module({
  imports: [UsersModule, PaymentModule],
  providers: [OrdersService],
})
export class OrdersModule {}
```

**Custom providers:**

```typescript
@Module({
  providers: [
    {
      provide: 'CONFIG',
      useValue: { apiKey: process.env.API_KEY },
    },
    {
      provide: DatabaseConnection,
      useFactory: async (config: ConfigService) => {
        return createConnection(config.get('DATABASE_URL'));
      },
      inject: [ConfigService],
    },
  ],
})
export class AppModule {}
```

**Scopes:**

| Scope | Mô Tả |
| ----- | ----- |
| `DEFAULT` (Singleton) | Một instance cho toàn app |
| `REQUEST` | Instance mới mỗi request |
| `TRANSIENT` | Instance mới mỗi lần inject |

---

## Pipes — Validation và Transformation

Pipes transform hoặc validate data trước khi đến handler:

```typescript
// dto/create-user.dto.ts
import { IsEmail, IsString, MinLength, IsOptional, IsInt } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(2)
  name: string;

  @IsOptional()
  @IsInt()
  age?: number;
}
```

**Global ValidationPipe:**

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,           // Strip unknown properties
    forbidNonWhitelisted: true, // Throw nếu có unknown properties
    transform: true,           // Auto transform types (string → number)
    transformOptions: {
      enableImplicitConversion: true,
    },
  }));

  await app.listen(3000);
}
```

**Built-in pipes:**

| Pipe | Mục Đích |
| ---- | -------- |
| `ParseIntPipe` | String → integer |
| `ParseUUIDPipe` | Validate UUID format |
| `ParseBoolPipe` | String → boolean |
| `ValidationPipe` | class-validator integration |

---

## Guards — Authentication và Authorization

Guards quyết định request có được xử lý hay không (`canActivate`):

```typescript
// guards/jwt-auth.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

// guards/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user.roles?.includes(role));
  }
}

// Decorator
import { SetMetadata } from '@nestjs/common';
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// Sử dụng
@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get('users')
  @Roles('admin')
  getUsers() { /* ... */ }
}
```

---

## Interceptors

Interceptors wrap request/response — logging, transform response, caching:

```typescript
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, ApiResponse<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<ApiResponse<T>> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const req = context.switchToHttp().getRequest();
    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        console.log(`${req.method} ${req.url} ${duration}ms`);
      }),
    );
  }
}

// Global
app.useGlobalInterceptors(new TransformInterceptor());
```

**Request lifecycle trong NestJS:**

```
Middleware → Guards → Interceptors (before) → Pipes → Controller → Interceptors (after) → Exception Filters
```

---

## Exception Filters

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    let status = 500;
    let message = 'Internal Server Error';
    let code = 'INTERNAL_ERROR';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exResponse = exception.getResponse();
      message = typeof exResponse === 'string' ? exResponse : (exResponse as any).message;
      code = (exResponse as any).error || 'HTTP_ERROR';
    }

    response.status(status).json({
      success: false,
      error: { code, message },
    });
  }
}

// Built-in exceptions
throw new NotFoundException('User not found');
throw new BadRequestException('Invalid input');
throw new UnauthorizedException();
throw new ForbiddenException();
```

---

## Bootstrap Configuration

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log'],
  });

  app.setGlobalPrefix('api/v1');
  app.enableCors({ origin: process.env.ALLOWED_ORIGINS?.split(',') });
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
  app.useGlobalFilters(new GlobalExceptionFilter());

  // Swagger
  const config = new DocumentBuilder()
    .setTitle('My API')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document);

  await app.listen(process.env.PORT || 3000);
}
bootstrap();
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: NestJS khác Express thế nào?

**Trả lời:** NestJS là framework opinionated với kiến trúc modular, DI, decorators. Express là minimal library — bạn tự tổ chức code. NestJS chạy trên Express/Fastify adapter. NestJS phù hợp enterprise; Express phù hợp prototype và flexibility.

### Câu 2: Guards vs Middleware vs Interceptors?

**Trả lời:** Middleware: generic, chạy trước route, không biết handler nào sẽ chạy. Guards: quyết định authorize/authenticate, có access ExecutionContext. Interceptors: wrap handler execution, transform request/response. Pipes: validate/transform input data.

### Câu 3: Tại sao NestJS dùng decorators?

**Trả lời:** Declarative metadata cho framework biết cách wire components. `@Controller()`, `@Injectable()`, `@Get()` đăng ký routes và DI bindings. TypeScript decorators + reflect-metadata enable runtime introspection.

### Câu 4: NestJS performance so với Express thuần?

**Trả lời:** Chậm hơn do DI container overhead và reflection. Mitigate bằng Fastify adapter thay Express. Trade-off: structure và maintainability vs raw performance. Đủ tốt cho hầu hết enterprise APIs.

---

**Xem tiếp:** [5-rest-api-design.md](./5-rest-api-design.md) — REST API design principles.
