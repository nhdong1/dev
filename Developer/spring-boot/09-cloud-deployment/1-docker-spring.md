# Docker Hóa Spring Boot — Dockerfile, Jib, Layer Caching

> Hướng dẫn đóng gói ứng dụng Spring Boot vào Docker container một cách tối ưu: viết Dockerfile multi-stage, dùng Jib để build không cần Docker daemon, và tận dụng layer caching để tăng tốc CI/CD pipeline.

---

## 1. Tại Sao Docker Hóa Spring Boot?

| Lợi Ích | Mô Tả |
|---------|-------|
| **Tính nhất quán** | "Works on my machine" không còn là vấn đề |
| **Isolation (Cô Lập)** | Ứng dụng không phụ thuộc vào môi trường host |
| **Scalability (Khả Năng Mở Rộng)** | Dễ dàng scale với Kubernetes / Docker Swarm |
| **Immutability (Bất Biến)** | Image đã build không thay đổi giữa các môi trường |
| **Reproducibility (Tái Tạo Được)** | Cùng một image deploy ở dev/staging/production |

---

## 2. Dockerfile Cơ Bản (Không Tối Ưu)

```dockerfile
# ❌ Anti-pattern: không dùng multi-stage, image rất nặng
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "target/myapp.jar"]
```

**Vấn đề:**
- Image chứa cả JDK (nặng ~500MB) và Maven + source code
- Mỗi lần build đều tải lại toàn bộ dependencies
- Source code lộ trong image production

---

## 3. Multi-Stage Dockerfile (Đa Giai Đoạn) — Chuẩn Production

Multi-stage build — Build Đa Giai Đoạn — cho phép dùng nhiều `FROM` để tách biệt giai đoạn build và giai đoạn runtime.

```dockerfile
# ─────────────────────────────────────────
# Stage 1: BUILD — Giai Đoạn Build
# ─────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# Copy dependency manifests trước — tận dụng Docker layer cache
COPY mvnw pom.xml ./
COPY .mvn .mvn

# Download dependencies (layer này được cache nếu pom.xml không đổi)
RUN ./mvnw dependency:go-offline -q

# Copy source code và build
COPY src ./src
RUN ./mvnw package -DskipTests -q

# ─────────────────────────────────────────
# Stage 2: EXTRACT — Tách Layers Của JAR
# ─────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS extractor
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
# Spring Boot Layertools tách JAR thành các layers có thể cache riêng
RUN java -Djarmode=layertools -jar app.jar extract

# ─────────────────────────────────────────
# Stage 3: RUNTIME — Giai Đoạn Chạy
# ─────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Tạo non-root user — người dùng không phải root — để tăng bảo mật
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy từng layer riêng biệt (thứ tự từ ít thay đổi → hay thay đổi)
COPY --from=extractor /app/dependencies/ ./
COPY --from=extractor /app/spring-boot-loader/ ./
COPY --from=extractor /app/snapshot-dependencies/ ./
COPY --from=extractor /app/application/ ./

# Chạy với non-root user
USER appuser

EXPOSE 8080

# Dùng exec form để nhận SIGTERM đúng cách (graceful shutdown)
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

### Tại Sao Copy Từng Layer Riêng?

Spring Boot Layertools — Công Cụ Phân Tầng — tách JAR thành 4 layers:

```
dependencies/          ← Ít thay đổi nhất (3rd party libs)
spring-boot-loader/    ← Hiếm thay đổi
snapshot-dependencies/ ← Thay đổi khi update SNAPSHOT
application/           ← Thay đổi thường xuyên nhất (code của bạn)
```

Docker cache các layer theo thứ tự — khi chỉ code ứng dụng thay đổi, chỉ layer `application/` bị invalidate, các layer khác được dùng lại từ cache.

---

## 4. Cấu Hình Spring Boot Cho Docker

### Bật Layertools Trong `pom.xml`

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <!-- Bật layer index — chỉ mục phân tầng -->
            <enabled>true</enabled>
        </layers>
        <image>
            <!-- Cấu hình cho spring-boot:build-image (Buildpacks) -->
            <name>${project.artifactId}:${project.version}</name>
            <builder>paketobuildpacks/builder-jammy-base</builder>
        </image>
    </configuration>
</plugin>
```

### Cấu Hình `application.properties` Cho Container

```properties
# Graceful shutdown — Tắt Mềm: chờ request hiện tại hoàn thành
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s

# Cổng server
server.port=8080

# Compression — Nén response
server.compression.enabled=true
server.compression.min-response-size=2KB

# Tomcat thread pool — Bể Thread
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=10

# Actuator cho health checks
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
```

---

## 5. Tối Ưu Layer Caching (Cache Theo Tầng)

### Thứ Tự Copy File Trong Dockerfile

```dockerfile
# ✅ Đúng: copy thứ gì ít thay đổi trước
COPY mvnw pom.xml ./          # Ít thay đổi
COPY .mvn .mvn/
RUN ./mvnw dependency:go-offline  # Cached nếu pom.xml không đổi
COPY src/ src/                # Thay đổi thường xuyên
RUN ./mvnw package

# ❌ Sai: copy tất cả trước thì cache luôn bị miss
COPY . .
RUN ./mvnw package
```

### Sử Dụng `.dockerignore`

```dockerignore
# .dockerignore — loại trừ file không cần thiết
target/
.git/
.idea/
*.md
*.log
docker-compose*.yml
Dockerfile*
.mvn/wrapper/maven-wrapper.jar

# Test files
src/test/
```

---

## 6. Jib — Build Image Không Cần Docker Daemon

Jib — do Google phát triển — build image trực tiếp từ Maven/Gradle mà không cần Docker daemon cài đặt, giúp CI pipeline nhanh hơn và đơn giản hơn.

### Cấu Hình Jib Trong `pom.xml`

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>3.4.3</version>
    <configuration>
        <from>
            <!-- Base image — Image gốc -->
            <image>eclipse-temurin:21-jre-alpine</image>
            <!-- Chỉ định platform nếu build cross-platform -->
            <platforms>
                <platform>
                    <architecture>amd64</architecture>
                    <os>linux</os>
                </platform>
                <platform>
                    <architecture>arm64</architecture>
                    <os>linux</os>
                </platform>
            </platforms>
        </from>
        <to>
            <!-- Destination image — Image đích -->
            <image>registry.example.com/my-org/my-app</image>
            <tags>
                <tag>${project.version}</tag>
                <tag>latest</tag>
            </tags>
        </to>
        <container>
            <!-- JVM options -->
            <jvmFlags>
                <jvmFlag>-XX:+UseContainerSupport</jvmFlag>
                <jvmFlag>-XX:MaxRAMPercentage=75.0</jvmFlag>
                <jvmFlag>-Djava.security.egd=file:/dev/./urandom</jvmFlag>
            </jvmFlags>
            <ports>
                <port>8080</port>
            </ports>
            <!-- Labels — nhãn metadata -->
            <labels>
                <version>${project.version}</version>
                <maintainer>team@example.com</maintainer>
            </labels>
            <!-- Đặt timezone -->
            <environment>
                <TZ>Asia/Ho_Chi_Minh</TZ>
            </environment>
            <!-- CreationTime để image reproducible -->
            <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
        </container>
        <!-- Tắt extended client-side auth (nếu dùng credential helper) -->
        <allowInsecureRegistries>false</allowInsecureRegistries>
    </configuration>
</plugin>
```

### Các Lệnh Jib

```bash
# Build và push lên registry
./mvnw jib:build

# Build vào local Docker daemon (cần Docker cài đặt)
./mvnw jib:dockerBuild

# Build ra tar file (dùng trong CI không có Docker daemon)
./mvnw jib:buildTar

# Chỉ định registry credentials trong CLI
./mvnw jib:build \
  -Djib.to.auth.username=$REGISTRY_USER \
  -Djib.to.auth.password=$REGISTRY_PASSWORD

# Build với tag cụ thể
./mvnw jib:build -Dimage=myapp:$(git rev-parse --short HEAD)
```

### Jib vs Dockerfile So Sánh

| Tiêu Chí | Dockerfile | Jib |
|---------|-----------|-----|
| Cần Docker daemon | ✅ Có | ❌ Không |
| Tốc độ build | Trung bình | Nhanh hơn (tối ưu layers tự động) |
| Cấu hình | Linh hoạt | Convention-based (dựa trên quy ước) |
| Non-root user | Phải tự cấu hình | Tự động (UID 1000) |
| Layer optimization | Thủ công | Tự động |
| Tích hợp CI | Cần Docker-in-Docker | Đơn giản hơn |

---

## 7. Spring Boot Buildpacks — Tạo Image Tự Động

Buildpacks (Paketo Buildpacks) tự động phát hiện loại ứng dụng và tạo image tối ưu mà không cần viết Dockerfile.

```bash
# Build image dùng Buildpacks (tích hợp sẵn trong Spring Boot Maven Plugin)
./mvnw spring-boot:build-image \
  -Dspring-boot.build-image.imageName=registry.example.com/my-app:1.0.0

# Hoặc cấu hình trong pom.xml và chạy
./mvnw spring-boot:build-image
```

**Ưu điểm Buildpacks:**
- Auto-detect ngôn ngữ, runtime, dependencies
- Security patches được vá tự động ở tầng buildpack
- Reproducible builds — Build có thể tái tạo
- Hỗ trợ sẵn memory calculator cho JVM

---

## 8. Docker Compose Cho Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime          # Multi-stage: chỉ build đến stage "runtime"
    image: my-app:local
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/myapp
      SPRING_DATASOURCE_USERNAME: myapp
      SPRING_DATASOURCE_PASSWORD: secret
      SPRING_DATA_REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

volumes:
  postgres_data:
```

---

## 9. Các JVM Flags Quan Trọng Cho Container

```bash
# Bắt buộc: giúp JVM nhận biết resource limits của container
-XX:+UseContainerSupport

# Giới hạn heap theo phần trăm RAM container
-XX:MaxRAMPercentage=75.0      # Heap tối đa = 75% RAM container
-XX:InitialRAMPercentage=50.0  # Heap khởi đầu = 50% RAM container

# GC (Garbage Collector — Bộ Thu Gom Rác) cho container
-XX:+UseG1GC                   # G1GC — tốt cho container nhỏ đến vừa
# -XX:+UseZGC                  # ZGC — low-latency, cần Java 21+

# Entropy source cho SecureRandom (tránh blocking)
-Djava.security.egd=file:/dev/./urandom

# Graceful shutdown signal
# Spring Boot tự handle SIGTERM khi server.shutdown=graceful
```

---

## 10. Security Best Practices (Thực Hành Bảo Mật Tốt Nhất)

### Scan Image Lỗ Hổng Bảo Mật

```bash
# Trivy — tool scan image phổ biến
trivy image my-app:1.0.0

# Docker Scout (tích hợp Docker)
docker scout cves my-app:1.0.0

# Snyk
snyk container test my-app:1.0.0
```

### Nguyên Tắc Least Privilege (Đặc Quyền Tối Thiểu)

```dockerfile
# Không chạy root trong container
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Filesystem read-only (chỉ đọc) nếu không cần ghi
# Đặt trong Kubernetes securityContext:
# readOnlyRootFilesystem: true
```

### Không Hardcode Secrets

```dockerfile
# ❌ Không bao giờ làm thế này
ENV DB_PASSWORD=mysecretpassword

# ✅ Dùng runtime environment variables — biến môi trường runtime
# Inject qua Kubernetes Secret hoặc Vault
```

---

## 11. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Multi-stage Dockerfile là gì và tại sao dùng nó?**

A: Multi-stage build — Build Đa Giai Đoạn — dùng nhiều lệnh `FROM` trong một Dockerfile. Giai đoạn build dùng image có JDK + Maven để compile; giai đoạn runtime chỉ cần JRE (nhẹ hơn). Lợi ích: image production nhỏ hơn (~100MB thay vì ~500MB), không lộ source code và build tools trong image cuối.

**Q: Layer caching trong Docker hoạt động thế nào?**

A: Docker cache từng layer theo thứ tự. Nếu một layer thay đổi, tất cả các layer phía sau đều bị invalidate (vô hiệu hóa cache). Chiến lược: copy file ít thay đổi (pom.xml, go.mod) trước → download dependencies → copy source code → build. Như vậy, khi chỉ code thay đổi, dependencies không cần download lại.

**Q: Jib khác Dockerfile như thế nào?**

A: Jib build image trực tiếp từ Maven/Gradle plugin mà không cần Docker daemon. Jib tự động tối ưu layers (dependencies/resources/classes riêng biệt), không cần viết Dockerfile, và chạy được trong môi trường CI không có Docker. Tuy nhiên, Dockerfile linh hoạt hơn cho các trường hợp tùy chỉnh phức tạp.

**Q: `-XX:+UseContainerSupport` có tác dụng gì?**

A: Trước Java 8u191, JVM đọc RAM từ host machine thay vì container, dẫn đến OOM (Out of Memory). `UseContainerSupport` (mặc định bật từ Java 11+) giúp JVM tôn trọng cgroups memory limit của container và tính toán heap size dựa trên RAM của container.

---

## ✅ Checklist

- [ ] Dùng multi-stage Dockerfile để tối giảm image size
- [ ] Bật Spring Boot Layertools để tối ưu layer caching
- [ ] Tạo `.dockerignore` để loại trừ file không cần thiết
- [ ] Chạy container với non-root user
- [ ] Dùng `-XX:+UseContainerSupport` và `-XX:MaxRAMPercentage`
- [ ] Cấu hình `server.shutdown=graceful` cho graceful shutdown
- [ ] Không hardcode secrets trong Dockerfile hoặc image
- [ ] Scan image với Trivy / Snyk trước khi deploy
- [ ] Tag image theo version (không dùng `latest` ở production)
- [ ] Test Dockerfile locally trước khi đưa vào CI
