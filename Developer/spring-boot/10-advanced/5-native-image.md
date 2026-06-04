# GraalVM Native Image — Biên Dịch Native

> **GraalVM Native Image** (Ảnh Nhị Phân Gốc) biên dịch ứng dụng Java/Spring Boot thành file thực thi native — không cần JVM runtime, khởi động trong vài mili-giây, tiêu thụ ít RAM hơn đáng kể. Điều này được thực hiện qua **AOT** (Ahead-of-Time Compilation — Biên Dịch Trước Thời Gian) thay vì JIT (Just-In-Time Compilation — Biên Dịch Đúng Lúc) truyền thống.

---

## 📋 Mục Lục

1. [JVM vs Native Image — So Sánh](#jvm-vs-native-image--so-sánh)
2. [GraalVM Hoạt Động Như Thế Nào?](#graalvm-hoạt-động-như-thế-nào)
3. [Giới Hạn Của Native Image](#giới-hạn-của-native-image)
4. [Spring AOT Processing](#spring-aot-processing)
5. [Thiết Lập Build Native Image](#thiết-lập-build-native-image)
6. [Reflection Hints — Cung Cấp Gợi Ý Reflection](#reflection-hints--cung-cấp-gợi-ý-reflection)
7. [Native Profiles & Testing](#native-profiles--testing)
8. [Docker — Tối Ưu Container Size](#docker--tối-ưu-container-size)
9. [Troubleshooting — Xử Lý Sự Cố](#troubleshooting--xử-lý-sự-cố)
10. [Benchmark Thực Tế](#benchmark-thực-tế)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## JVM vs Native Image — So Sánh

```
                    JVM (HotSpot)               NATIVE IMAGE (GraalVM)
                    ─────────────               ──────────────────────
Runtime:            JVM required                Standalone binary
Startup Time:       3–10 giây (Spring Boot)     50–200 mili-giây
Memory (idle):      ~256MB heap                 ~30–80MB RSS
Memory (loaded):    ~512MB+                     ~100–200MB
Peak Throughput:    Cao (JIT optimizes)         Thấp hơn 10–20%
Build Time:         10–30 giây                  3–15 phút
Reflection:         Full dynamic                Cần khai báo trước
JVM flags (-Xmx):   Có thể tune runtime         Cố định lúc build
Profiling tools:    JFR, async-profiler         Hạn chế

PHÙ HỢP NATIVE IMAGE:
✅ AWS Lambda / Google Cloud Functions (serverless)
✅ Kubernetes pods với khởi động nhanh (scale-out nhanh)
✅ CLI tools viết bằng Java (micronaut, quarkus, spring shell)
✅ Microservices nhỏ, ít phụ thuộc phức tạp
✅ Môi trường RAM hạn chế (edge computing)

KHÔNG PHÙ HỢP:
❌ Ứng dụng chạy lâu dài cần JIT optimization
❌ Sử dụng nhiều reflection động (CGLIB proxies cũ)
❌ Thư viện chưa tương thích Native
❌ Team cần debug nhanh (stack traces khác)
```

---

## GraalVM Hoạt Động Như Thế Nào?

```
BIÊN DỊCH TRUYỀN THỐNG (JIT):
Source (.java) → Bytecode (.class) → [Chạy trên JVM] → JIT compile khi cần → Machine code

NATIVE IMAGE (AOT):
Source (.java) → Bytecode (.class) → [GraalVM phân tích] → Machine code (native binary)

GIAI ĐOẠN AOT COMPILATION:
┌─────────────────────────────────────────────────────────────────┐
│  1. CLOSED WORLD ASSUMPTION (Giả Định Thế Giới Đóng)           │
│     - Phân tích tất cả code có thể được gọi                     │
│     - Mọi thứ phải biết tại build time (không dynamic)          │
│                                                                   │
│  2. POINTS-TO ANALYSIS (Phân Tích Con Trỏ)                     │
│     - Xác định kiểu đối tượng tại runtime                       │
│     - Loại bỏ dead code (code không bao giờ được gọi)          │
│                                                                   │
│  3. REFLECTION REGISTRATION (Đăng Ký Reflection)               │
│     - Reflection phải được khai báo trước                       │
│     - Tạo reachability-metadata.json                            │
│                                                                   │
│  4. IMAGE GENERATION (Tạo Ảnh)                                  │
│     - Tạo heap snapshot (bao gồm initialized state)             │
│     - Compile thành native binary                               │
└─────────────────────────────────────────────────────────────────┘

OUTPUT: Một file binary duy nhất, chạy không cần JVM
```

---

## Giới Hạn Của Native Image

```
NHỮNG GÌ KHÔNG HOẠT ĐỘNG MẶC ĐỊNH:

1. REFLECTION ĐỘNG
   - Class.forName("com.example.MyClass")
   - Method.invoke()
   - Jackson JSON deserialization (reflection-based)
   → Cần đăng ký trong hints

2. DYNAMIC PROXY (Proxy Động)
   - java.lang.reflect.Proxy
   - CGLIB subclass proxies (Spring AOP truyền thống dùng CGLIB)
   → Spring AOT tự generate proxy code lúc build time

3. CLASSLOADING ĐỘNG
   - ClassLoader.loadClass() lúc runtime
   → Không hỗ trợ plugin system động

4. JNI (Java Native Interface)
   - Gọi native libraries
   → Cần khai báo trong native-image.properties

5. SERIALIZATION JAVA TRUYỀN THỐNG
   - java.io.Serializable với ObjectOutputStream
   → Cần đăng ký serialization hints

SPRING BOOT 3 GIẢI QUYẾT:
✅ Spring AOT tự động generate hints cho Spring beans
✅ @Autowired, @Component, @Configuration hoạt động tự động
✅ Spring MVC, Spring Data JPA, Spring Security hỗ trợ native
⚠️ Third-party libraries có thể cần hints thêm
```

---

## Spring AOT Processing

**Spring AOT** (Spring Ahead-of-Time) là layer trung gian — phân tích Spring ApplicationContext lúc build time và generate code thay cho dynamic reflection.

```
BUILD TIME:
Spring ApplicationContext → AOT Processor → Generated Code
                                  ↓
                         BeanDefinition proxies
                         Reflection hints
                         Resource hints
                         Serialization hints

RUNTIME (Native):
Generated Code → Native Image (không cần Spring introspection)
```

```java
// AOT tự động xử lý những annotation này:
@Component, @Service, @Repository, @Controller
@Configuration, @Bean
@Autowired, @Value, @Qualifier
@Transactional
@RequestMapping, @GetMapping, ...
@Entity, @Table, @Column (JPA)
@PreAuthorize (Spring Security)

// Generated code nằm trong target/spring-aot/main/
// Bạn có thể inspect nhưng không nên edit
```

---

## Thiết Lập Build Native Image

### Yêu Cầu

```bash
# Cài GraalVM JDK 21+
# Khuyên dùng SDKMAN:
sdk install java 21.0.3-graal
sdk use java 21.0.3-graal

# Hoặc download từ https://www.graalvm.org/downloads/
# native-image tool đã được bundled sẵn từ GraalVM 22.3+

# Kiểm tra
java -version        # graalvm ce 21...
native-image --version
```

### Maven

```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
            <!-- version được quản lý bởi spring-boot-starter-parent -->
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <image>
                    <!-- Dùng Buildpacks thay vì native-image trực tiếp -->
                    <builder>paketobuildpacks/builder-jammy-tiny</builder>
                    <env>
                        <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
                    </env>
                </image>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Cách Build

```bash
# Cách 1: Build executable JAR thông thường (JVM)
./mvnw package

# Cách 2: Build Native Image trực tiếp (cần GraalVM installed)
./mvnw -Pnative native:compile
# Output: target/myapp (binary file ~50–100MB)
./target/myapp  # Chạy ngay, không cần JVM

# Cách 3: Build Docker image có native binary (khuyên dùng cho CI/CD)
# Không cần GraalVM cài local — dùng Docker builder
./mvnw spring-boot:build-image -Pnative
docker run myapp:0.0.1-SNAPSHOT

# Cách 4: Chạy test với Native Image (slow)
./mvnw -Pnative test
```

---

## Reflection Hints — Cung Cấp Gợi Ý Reflection

### Cách 1: RuntimeHintsRegistrar (Khuyên Dùng)

```java
@Configuration
@ImportRuntimeHints(MyRuntimeHints.class)
public class AppConfig {}

public class MyRuntimeHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Đăng ký class cần reflection (Jackson deserialization, v.v.)
        hints.reflection()
            .registerType(UserDto.class,
                MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                MemberCategory.DECLARED_FIELDS)
            .registerType(OrderDto.class,
                MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                MemberCategory.DECLARED_FIELDS);

        // Đăng ký resources (file properties, templates)
        hints.resources()
            .registerPattern("*.sql")
            .registerPattern("messages/*.properties");

        // Đăng ký JDK proxy interfaces
        hints.proxies()
            .registerJdkProxy(UserRepository.class);

        // Đăng ký serialization
        hints.serialization()
            .registerType(UserEvent.class);
    }
}
```

### Cách 2: @RegisterReflectionForBinding

```java
// Dành cho DTOs cần Jackson binding
@RegisterReflectionForBinding({UserDto.class, OrderDto.class, ErrorDto.class})
@RestController
public class UserController {
    // ...
}
```

### Cách 3: File JSON (Legacy / Third-party Libraries)

```json
// src/main/resources/META-INF/native-image/reflect-config.json
[
  {
    "name": "com.example.dto.UserDto",
    "allDeclaredConstructors": true,
    "allDeclaredFields": true,
    "allDeclaredMethods": true
  }
]
```

### Cách 4: Tracing Agent — Tự Động Tạo Hints

```bash
# Chạy app với Java agent để trace reflection usage
java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image \
     -jar target/myapp.jar

# Sau đó chạy integration tests để trace đầy đủ
# Agent tự tạo các file JSON hints
```

---

## Native Profiles & Testing

```xml
<!-- Profile native trong pom.xml -->
<profiles>
    <profile>
        <id>native</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.graalvm.buildtools</groupId>
                    <artifactId>native-maven-plugin</artifactId>
                    <executions>
                        <execution>
                            <id>build-native</id>
                            <goals>
                                <goal>compile-no-fork</goal>
                            </goals>
                            <phase>package</phase>
                        </execution>
                        <execution>
                            <id>test-native</id>
                            <goals>
                                <goal>test</goal>
                            </goals>
                            <phase>test</phase>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

```java
// Test với @NativeTest — chạy test trong native context
@NativeTest  // Hoặc @SpringBootTest với native build
class UserServiceNativeTest {

    @Autowired
    private UserService userService;

    @Test
    void testCreateUser() {
        User user = userService.createUser(
            new CreateUserInput("Test", "test@example.com")
        );
        assertThat(user.getId()).isNotNull();
        assertThat(user.getName()).isEqualTo("Test");
    }
}
```

---

## Docker — Tối Ưu Container Size

```dockerfile
# Multi-stage build cho Native Image
# Stage 1: Build với GraalVM
FROM ghcr.io/graalvm/native-image-community:21 AS builder

WORKDIR /app
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline

COPY src/ src/
RUN ./mvnw -Pnative native:compile -DskipTests

# Stage 2: Minimal runtime image
FROM debian:bookworm-slim
# Hoặc dùng distroless cho security tốt hơn:
# FROM gcr.io/distroless/base-debian12

RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring

COPY --from=builder /app/target/myapp /app/myapp

EXPOSE 8080
ENTRYPOINT ["/app/myapp"]
```

```
Image size comparison (So Sánh Kích Thước):
JVM (eclipse-temurin:21):  ~200MB (JDK) + ~50MB (app JAR) = ~250MB
JVM slim:                  ~100MB (JRE) + ~50MB (app JAR) = ~150MB
Native (debian-slim):      ~60MB (OS libs) + ~70MB (binary) = ~130MB
Native (distroless):       ~10MB (minimal) + ~70MB (binary) = ~80MB
```

---

## Troubleshooting — Xử Lý Sự Cố

### Lỗi Phổ Biến Khi Build Native Image

```bash
# 1. ClassNotFoundException lúc runtime
Error: Class not found: com.example.SomeClass
→ Thêm reflection hints cho class đó

# 2. NoSuchMethodException — constructor/method không tìm thấy
→ Đăng ký INVOKE_DECLARED_CONSTRUCTORS hoặc INVOKE_PUBLIC_METHODS

# 3. Proxy class không generate được
→ Thêm hints.proxies().registerJdkProxy(MyInterface.class)

# 4. Resource not found (file .properties, templates)
→ hints.resources().registerPattern("templates/**")

# 5. Serialization issue
→ hints.serialization().registerType(MyClass.class)
```

```java
// Debug: In ra class không được xử lý
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        // Trong development, in warnings về missing hints
        if (Boolean.getBoolean("spring.aot.enabled")) {
            System.setProperty("spring.native.trace", "true");
        }
        SpringApplication.run(Application.class, args);
    }
}
```

### Compatibility Check (Kiểm Tra Tương Thích)

```xml
<!-- Kiểm tra thư viện nào tương thích Native -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <configuration>
        <!-- Tạo report về compatibility -->
        <metadataRepository>
            <enabled>true</enabled>
        </metadataRepository>
    </configuration>
</plugin>
```

```
Thư viện đã hỗ trợ Native (Spring Boot 3+):
✅ spring-web, spring-webmvc, spring-webflux
✅ spring-data-jpa, spring-data-redis
✅ spring-security
✅ Hibernate 6+
✅ Jackson 2.14+
✅ HikariCP
✅ Micrometer

Cần thêm hints:
⚠️ Một số Hibernate enhancements
⚠️ QueryDSL (cần apt plugin)
⚠️ MapStruct (thường tự động)
⚠️ Custom serializers/deserializers

Chưa/kém hỗ trợ:
❌ Aspectj weaving (dùng Spring AOP proxy-based thay thế)
❌ JAXB (XML binding cũ)
❌ Một số JPA features động
```

---

## Benchmark Thực Tế

```
Ứng dụng: Spring Boot REST API đơn giản
          - Spring MVC + Spring Data JPA + Spring Security
          - PostgreSQL database

STARTUP TIME (Thời Gian Khởi Động):
─────────────────────────────────────
JVM (HotSpot 21):           2.8 giây
JVM + Class Data Sharing:   1.5 giây
Native Image:               0.08 giây (80 milli-giây)

MEMORY AT IDLE (RAM Khi Rảnh):
──────────────────────────────
JVM (HotSpot 21):           220MB RSS
Native Image:               48MB RSS

MEMORY UNDER LOAD (RAM Khi Có Tải):
─────────────────────────────────────
JVM:                        512MB RSS (JIT compiled, heap full)
Native Image:               150MB RSS (không có JIT heap overhead)

THROUGHPUT (Thông Lượng) @ 100 concurrent users:
──────────────────────────────────────────────────
JVM (sau warm-up):          ~12,000 req/s
Native Image:               ~10,500 req/s  (thua ~12% sau warm-up)
Native Image (cold):        ~10,500 req/s  (KHÔNG có warm-up penalty!)

BUILD TIME (Thời Gian Build):
──────────────────────────────
./mvnw package (JVM):       15 giây
./mvnw native:compile:      8 phút

KẾT LUẬN:
- Native Image tốt hơn cho: startup, idle memory, serverless/k8s scale-out
- JVM tốt hơn cho: peak throughput, debug, build speed, thư viện phức tạp
```

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa JIT và AOT compilation?**

A: **JIT** (Just-In-Time) biên dịch bytecode → machine code lúc runtime, dựa vào profiling để tối ưu hot paths — nên throughput tăng dần theo thời gian (warm-up). **AOT** (Ahead-of-Time) biên dịch trước khi chạy, tạo native binary — startup nhanh, không cần JVM, nhưng không thể tối ưu dựa trên runtime behavior. GraalVM Native Image dùng AOT.

---

**Q: Closed World Assumption là gì và tại sao nó là giới hạn của Native Image?**

A: Closed World Assumption (Giả Định Thế Giới Đóng) là yêu cầu GraalVM phải biết **tất cả** code có thể chạy tại build time — không có dynamic class loading, không có reflection không được khai báo, không có bytecode generation lúc runtime. Đây là giới hạn vì Java được thiết kế để dynamic — nhiều framework (Spring cũ, Hibernate lazy proxies, CGLIB) dùng reflection và dynamic proxies. Spring Boot 3 + Spring AOT giải quyết bằng cách generate static code thay thế.

---

**Q: Spring AOT giúp gì cho Native Image?**

A: Spring AOT phân tích ApplicationContext lúc build time và generate code thay thế cho các cơ chế reflection/proxy động: generate BeanDefinition factory methods (thay vì reflection), generate proxy classes source code (thay vì CGLIB runtime), và tạo reflection/resource/serialization hints cho GraalVM. Kết quả là Spring Native Image hoạt động mà không cần developer tự viết tất cả hints cho Spring internals.

---

**Q: Khi nào NÊN và KHÔNG NÊN dùng GraalVM Native Image?**

A: **Nên dùng** khi: serverless functions (AWS Lambda — pay-per-invocation, startup critical), Kubernetes pods cần scale nhanh, CLI tools viết Java, microservices nhỏ với ít phụ thuộc phức tạp, môi trường RAM hạn chế. **Không nên dùng** khi: ứng dụng chạy 24/7 cần JIT throughput cao, dùng nhiều thư viện chưa tương thích, team cần debug nhanh, CI/CD không chấp nhận build 8–15 phút.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
