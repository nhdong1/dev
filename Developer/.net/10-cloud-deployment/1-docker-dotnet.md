# Docker cho .NET — Dockerfile, Multi-Stage Build, Image Optimization

> Docker cho phép đóng gói ứng dụng .NET cùng toàn bộ dependencies — phụ thuộc vào một container image, đảm bảo chạy nhất quán từ máy developer đến production. Bài này trình bày cách viết Dockerfile hiệu quả, tối ưu kích thước image, và vận hành .NET app trong container.

---

## 1. Dockerfile Cơ Bản cho .NET

### Cấu Trúc Dockerfile Đơn Giản (không tối ưu)

```dockerfile
# Dùng image .NET SDK để build — KHÔNG nên dùng trực tiếp cho production
FROM mcr.microsoft.com/dotnet/sdk:8.0

WORKDIR /app

# Copy toàn bộ source code
COPY . .

# Restore dependencies và build
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish

# Expose port
EXPOSE 8080

# Chạy ứng dụng
ENTRYPOINT ["dotnet", "/app/publish/MyApp.dll"]
```

**Vấn đề:** Image này chứa toàn bộ .NET SDK (~800MB), rất lớn và có nhiều tool không cần thiết trong production.

---

## 2. Multi-Stage Build — Build Nhiều Giai Đoạn

Multi-stage build — build nhiều giai đoạn — cho phép dùng SDK để build nhưng chỉ copy artifact — sản phẩm build — vào runtime image nhỏ hơn nhiều.

### Dockerfile Multi-Stage Chuẩn

```dockerfile
# ============================================================
# Stage 1: Build — Giai đoạn build
# ============================================================
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

# Copy project files trước để tận dụng Docker layer cache
# (chỉ rebuild khi .csproj thay đổi)
COPY ["src/MyApp.Api/MyApp.Api.csproj", "src/MyApp.Api/"]
COPY ["src/MyApp.Application/MyApp.Application.csproj", "src/MyApp.Application/"]
COPY ["src/MyApp.Domain/MyApp.Domain.csproj", "src/MyApp.Domain/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj", "src/MyApp.Infrastructure/"]

# Restore NuGet packages
RUN dotnet restore "src/MyApp.Api/MyApp.Api.csproj"

# Copy source code
COPY . .

# Build và publish
WORKDIR "/src/src/MyApp.Api"
RUN dotnet build "MyApp.Api.csproj" -c Release -o /app/build

RUN dotnet publish "MyApp.Api.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# ============================================================
# Stage 2: Runtime — Giai đoạn chạy
# ============================================================
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

# Tạo user không phải root để tăng bảo mật
RUN adduser --disabled-password --gecos "" appuser

WORKDIR /app

# Chỉ copy artifact từ stage build
COPY --from=build /app/publish .

# Chuyển sang user không phải root
USER appuser

# Port mặc định ASP.NET Core 8+ là 8080 (không cần ASPNETCORE_URLS)
EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### So Sánh Kích Thước Image

| Image Base | Kích Thước | Dùng Cho |
|------------|-----------|----------|
| `dotnet/sdk:8.0` | ~800 MB | Build stage — không dùng production |
| `dotnet/aspnet:8.0` | ~220 MB | Runtime với ASP.NET Core |
| `dotnet/runtime:8.0` | ~190 MB | Runtime không có ASP.NET |
| `dotnet/runtime-deps:8.0` | ~120 MB | Self-contained app |
| `dotnet/aspnet:8.0-alpine` | ~100 MB | Alpine-based, nhỏ nhất |

---

## 3. Tối Ưu Docker Layer Cache — Bộ Cache Layer

Docker cache mỗi layer — tầng. Nếu một layer không thay đổi, Docker tái sử dụng cache thay vì rebuild. Sắp xếp thứ tự COPY đúng cách giúp build nhanh hơn nhiều.

### Nguyên Tắc: Từ Ít Thay Đổi → Nhiều Thay Đổi

```dockerfile
# ❌ Sai thứ tự — copy hết rồi mới restore
COPY . .
RUN dotnet restore  # Mỗi lần code thay đổi đều phải restore lại

# ✅ Đúng thứ tự — copy .csproj trước
COPY ["MyApp.Api.csproj", "."]
RUN dotnet restore  # Chỉ chạy lại khi .csproj thay đổi

COPY . .            # Code thay đổi thường xuyên — đặt sau
RUN dotnet build
```

### Minh Họa Cache Hit/Miss

```
Lần 1 (chưa có cache):
  Layer 1: FROM aspnet:8.0       → MISS → download base image
  Layer 2: COPY *.csproj         → MISS → copy files
  Layer 3: RUN dotnet restore    → MISS → download packages (chậm ~30s)
  Layer 4: COPY . .              → MISS → copy source
  Layer 5: RUN dotnet build      → MISS → compile

Lần 2 (chỉ code thay đổi, .csproj không đổi):
  Layer 1: FROM aspnet:8.0       → HIT ✓ (dùng cache)
  Layer 2: COPY *.csproj         → HIT ✓ (dùng cache)
  Layer 3: RUN dotnet restore    → HIT ✓ (dùng cache — tiết kiệm 30s!)
  Layer 4: COPY . .              → MISS → copy source mới
  Layer 5: RUN dotnet build      → MISS → compile lại
```

---

## 4. Dockerfile cho Kiến Trúc Multi-Project

Khi solution — giải pháp có nhiều projects (Domain, Application, Infrastructure, API):

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Dùng glob để copy tất cả .csproj trong một lệnh
COPY **/*.csproj ./
# Cấu trúc thư mục bị flat, cần restore từng project
RUN find . -name "*.csproj" | xargs -I {} dotnet restore {}

# Hoặc dùng cách khác — copy từng .csproj với đúng đường dẫn
COPY ["src/MyApp.Domain/MyApp.Domain.csproj",              "src/MyApp.Domain/"]
COPY ["src/MyApp.Application/MyApp.Application.csproj",    "src/MyApp.Application/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj","src/MyApp.Infrastructure/"]
COPY ["src/MyApp.Api/MyApp.Api.csproj",                    "src/MyApp.Api/"]

# Restore từ solution file — đơn giản hơn
COPY ["MyApp.sln", "."]
RUN dotnet restore

COPY . .

RUN dotnet publish "src/MyApp.Api/MyApp.Api.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

---

## 5. .dockerignore — Loại Trừ File Không Cần Thiết

File `.dockerignore` giống `.gitignore` nhưng cho Docker build context — ngữ cảnh build:

```dockerignore
# .dockerignore
**/.git
**/.gitignore
**/.vs
**/.vscode
**/bin
**/obj
**/*.user
**/*.suo
**/node_modules
**/.env
**/.env.*
**/docker-compose*.yml
**/Dockerfile*
**/README.md
**/wwwroot/node_modules
**/.dockerignore
```

**Lợi ích:**
- Giảm kích thước build context gửi lên Docker daemon
- Tránh copy file nhạy cảm (.env) vào image
- Build nhanh hơn

---

## 6. Environment Variables — Biến Môi Trường trong Docker

### Truyền Config qua Environment Variables

```dockerfile
# Trong Dockerfile — giá trị mặc định
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:8080
```

```bash
# Override khi run container
docker run -e ASPNETCORE_ENVIRONMENT=Staging \
           -e ConnectionStrings__Default="Server=..." \
           -p 8080:8080 myapp:latest

# Dùng --env-file
docker run --env-file .env.production -p 8080:8080 myapp:latest
```

### Dùng Docker Secrets (production)

```bash
# Tạo secret
echo "Server=prod-server;..." | docker secret create db_connection -

# Trong docker-compose.yml hoặc service
services:
  api:
    image: myapp:latest
    secrets:
      - db_connection
    environment:
      - ConnectionStrings__Default=/run/secrets/db_connection

secrets:
  db_connection:
    external: true
```

---

## 7. Docker Compose cho Development

```yaml
# docker-compose.yml — cho môi trường development
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: build          # Dừng ở stage build để có hot-reload
    image: myapp-api:dev
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Default=Host=db;Database=myapp;Username=postgres;Password=devpassword
      - Redis__ConnectionString=redis:6379
    volumes:
      - ./src:/src           # Mount code để hot-reload
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: dotnet watch run --project src/MyApp.Api/MyApp.Api.csproj

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: devpassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## 8. Health Check — Kiểm Tra Sức Khỏe trong Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

# Thêm health check
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=60s \
            --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

```csharp
// Program.cs — expose health endpoint
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString)  // kiểm tra DB
    .AddRedis(redisConnectionString); // kiểm tra Redis

app.MapHealthChecks("/health");
```

---

## 9. Image Security — Bảo Mật Image

### Chạy Container với Non-Root User

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

# Tạo group và user riêng
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app
COPY --from=build /app/publish .

# Đặt ownership cho thư mục app
RUN chown -R appuser:appgroup /app

# Chuyển sang non-root user
USER appuser

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### Dùng Distroless Image — Image Tối Giản

```dockerfile
# Distroless — image chỉ chứa runtime, không có shell, package manager
FROM gcr.io/distroless/dotnet:8 AS runtime

WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["MyApp.Api"]  # Phải dùng self-contained executable
```

### Scan Vulnerabilities — Quét Lỗ Hổng

```bash
# Dùng Docker Scout
docker scout cves myapp:latest

# Dùng Trivy
trivy image myapp:latest

# Dùng Snyk
snyk container test myapp:latest
```

---

## 10. Build Arguments — Tham Số Build

```dockerfile
# Khai báo ARG — argument
ARG DOTNET_VERSION=8.0
ARG BUILD_CONFIGURATION=Release
ARG VERSION=1.0.0

FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build

WORKDIR /src
COPY . .

RUN dotnet publish \
    -c ${BUILD_CONFIGURATION} \
    -p:Version=${VERSION} \
    -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION} AS runtime
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

```bash
# Truyền ARG khi build
docker build \
    --build-arg VERSION=2.1.0 \
    --build-arg BUILD_CONFIGURATION=Release \
    -t myapp:2.1.0 .
```

---

## 11. Labeling — Metadata cho Image

```dockerfile
LABEL maintainer="team@company.com"
LABEL version="1.0.0"
LABEL description="MyApp API Service"
LABEL org.opencontainers.image.source="https://github.com/company/myapp"
LABEL org.opencontainers.image.revision="${GIT_COMMIT}"
LABEL org.opencontainers.image.created="${BUILD_DATE}"
```

---

## 12. Tổng Hợp: Dockerfile Production-Ready

```dockerfile
# ============================================================
# Stage 1: restore — tận dụng cache
# ============================================================
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS restore

WORKDIR /src

COPY ["src/MyApp.Domain/MyApp.Domain.csproj",               "src/MyApp.Domain/"]
COPY ["src/MyApp.Application/MyApp.Application.csproj",     "src/MyApp.Application/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj","src/MyApp.Infrastructure/"]
COPY ["src/MyApp.Api/MyApp.Api.csproj",                     "src/MyApp.Api/"]

RUN dotnet restore "src/MyApp.Api/MyApp.Api.csproj" \
    --runtime linux-x64

# ============================================================
# Stage 2: build và publish
# ============================================================
FROM restore AS build

ARG VERSION=1.0.0
ARG BUILD_DATE

COPY . .

RUN dotnet publish "src/MyApp.Api/MyApp.Api.csproj" \
    -c Release \
    -r linux-x64 \
    --no-restore \
    --self-contained false \
    -p:Version=${VERSION} \
    -o /app/publish

# ============================================================
# Stage 3: runtime image
# ============================================================
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

LABEL org.opencontainers.image.created="${BUILD_DATE}"
LABEL org.opencontainers.image.version="${VERSION}"

# Cài curl cho health check
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

# Non-root user
RUN adduser --disabled-password --gecos "" --uid 10001 appuser

WORKDIR /app

COPY --from=build /app/publish .

RUN chown -R appuser:appuser /app
USER appuser

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080

ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

---

## Checklist Docker .NET

- [ ] Dùng multi-stage build để giảm kích thước image
- [ ] `.dockerignore` đầy đủ, không copy `bin/`, `obj/`, `.env`
- [ ] Copy `.csproj` trước khi copy source code (tận dụng cache)
- [ ] Chạy với non-root user trong production
- [ ] HEALTHCHECK được định nghĩa
- [ ] Không hardcode secrets trong Dockerfile
- [ ] Scan image với Trivy hoặc Docker Scout
- [ ] Tag image với version cụ thể, không dùng `latest` trong production
- [ ] Dùng `aspnet` runtime image, không phải `sdk` image

---

## Câu Hỏi Phỏng Vấn

**Q: Multi-stage build là gì và tại sao nên dùng?**
> Multi-stage build cho phép dùng nhiều `FROM` trong một Dockerfile. Stage đầu dùng SDK để compile code, stage cuối chỉ copy artifact vào runtime image nhỏ hơn. Kết quả: image production giảm từ ~800MB xuống ~220MB, ít attack surface — bề mặt tấn công hơn.

**Q: Docker layer cache hoạt động như thế nào?**
> Docker cache từng instruction thành một layer. Nếu instruction và context không thay đổi, Docker dùng cache thay vì chạy lại. Khi một layer thay đổi, tất cả layer sau đó đều bị invalidate. Vì vậy cần đặt các instruction ít thay đổi (COPY .csproj, RUN restore) trước các instruction thay đổi thường xuyên (COPY source code).

**Q: Tại sao nên chạy container với non-root user?**
> Nếu attacker — kẻ tấn công khai thác được lỗ hổng trong app và có được shell trong container, việc chạy dưới non-root user giới hạn những gì họ có thể làm. Theo nguyên tắc least privilege — đặc quyền tối thiểu.

**Cập Nhật Lần Cuối:** 2026-06-02
