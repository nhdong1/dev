# Spring Boot Basics — Auto-configuration, Starters, @SpringBootApplication

> Spring Boot "đoán" cấu hình dựa trên những gì có trong classpath và tự động cấu hình chúng.
> Đây là lý do tại sao Spring Boot giúp bạn chạy ứng dụng chỉ trong vài phút.

---

## 📋 Mục Tiêu

- [ ] Hiểu `@SpringBootApplication` thực sự làm gì
- [ ] Giải thích **Auto-configuration** (Tự Động Cấu Hình) hoạt động thế nào
- [ ] Biết **Starters** là gì và cách chọn đúng Starter
- [ ] Đọc hiểu **Auto-configuration Report** để debug
- [ ] Tắt Auto-configuration khi cần
- [ ] Tạo **Custom Auto-configuration** (Tự Động Cấu Hình Tùy Chỉnh) cơ bản

---

## 1. @SpringBootApplication — Annotation Khởi Đầu

`@SpringBootApplication` là một **meta-annotation** (annotation tổng hợp) gồm 3 annotation:

```java
@SpringBootApplication
// Tương đương:
@SpringBootConfiguration   // = @Configuration — đây là class cấu hình
@EnableAutoConfiguration   // Bật Auto-configuration
@ComponentScan             // Quét @Component, @Service, @Repository... từ package này

public class MyApplication {
    public static void main(String[] args) {
        // ApplicationContext context =
        SpringApplication.run(MyApplication.class, args);
        // Spring Boot:
        // 1. Tạo ApplicationContext
        // 2. Component Scanning
        // 3. Auto-configuration
        // 4. Khởi động Embedded Tomcat
        // 5. Ứng dụng sẵn sàng nhận request
    }
}
```

### Cấu Trúc Package Quan Trọng

```
com.example.myapp/
├── MyApplication.java       ← @SpringBootApplication phải ở ROOT package
├── controller/
│   └── UserController.java
├── service/
│   └── UserService.java
└── repository/
    └── UserRepository.java
```

> **Quy tắc:** Đặt class `@SpringBootApplication` ở **root package** để Component Scan bao phủ toàn bộ dự án. Nếu đặt sai vị trí, Spring sẽ không tìm thấy Bean.

---

## 2. Auto-configuration — Tự Động Cấu Hình

### Cơ Chế Hoạt Động

Spring Boot đọc file `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3.x) trong mỗi JAR để biết có những Auto-configuration class nào.

```
Bước 1: Đọc danh sách Auto-configuration classes
        (từ spring-boot-autoconfigure.jar)

Bước 2: Với mỗi Auto-configuration class,
        kiểm tra điều kiện @Conditional

Bước 3: Nếu điều kiện thỏa mãn → Áp dụng cấu hình
        Nếu không → Bỏ qua

Bước 4: User config luôn thắng Auto-configuration
```

### @ConditionalOn* — Điều Kiện Kích Hoạt

Đây là trái tim của Auto-configuration:

```java
// Ví dụ đơn giản về cách DataSource được auto-configure
@AutoConfiguration
@ConditionalOnClass(DataSource.class)           // 1. Chỉ áp dụng nếu có DataSource trong classpath
@ConditionalOnMissingBean(DataSource.class)     // 2. Chỉ áp dụng nếu USER chưa định nghĩa DataSource
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnProperty(                     // 3. Chỉ áp dụng nếu có config trong application.properties
        prefix = "spring.datasource",
        name = "url"
    )
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
            .url(properties.getUrl())
            .username(properties.getUsername())
            .password(properties.getPassword())
            .build();
    }
}
```

### Bảng @ConditionalOn* Quan Trọng

| Annotation | Điều Kiện Kích Hoạt |
|------------|---------------------|
| `@ConditionalOnClass(X.class)` | X có trong classpath |
| `@ConditionalOnMissingClass("x.Y")` | X không có trong classpath |
| `@ConditionalOnBean(X.class)` | Bean X đã được đăng ký |
| `@ConditionalOnMissingBean(X.class)` | Bean X chưa được đăng ký |
| `@ConditionalOnProperty("app.feature")` | Property tồn tại và có giá trị |
| `@ConditionalOnWebApplication` | Đang chạy trong môi trường web |
| `@ConditionalOnExpression("${app.enabled:true}")` | SpEL expression trả về true |

### Ví Dụ Thực Tế — Redis Auto-configuration

```yaml
# application.yml
spring:
  redis:
    host: localhost
    port: 6379
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Khi thêm dependency trên, Spring Boot tự động:
1. Phát hiện `RedisAutoConfiguration` vì `Lettuce` (Redis client) có trong classpath
2. Tạo `RedisConnectionFactory` với host/port từ `application.yml`
3. Tạo `RedisTemplate<String, Object>` sẵn sàng inject
4. Không cần viết bất kỳ `@Configuration` class nào!

---

## 3. Starters — Bộ Dependency Được Đóng Gói

**Starters** là các dependency group được Spring Boot đóng gói sẵn, đảm bảo các thư viện tương thích với nhau.

### Starters Quan Trọng Nhất

```xml
<!-- Web API — bao gồm: Spring MVC, Tomcat, Jackson, Validation -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring Data JPA — bao gồm: Hibernate, Spring Data, Transactions -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Spring Security — bao gồm: Security Filter Chain, Core, Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Testing — bao gồm: JUnit 5, Mockito, AssertJ, Spring Test -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Actuator — monitoring & health checks -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- Validation (Bean Validation — Xác Thực) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- WebFlux — Reactive Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### Lợi Ích Của Starters

```
Không dùng Starter:
  spring-core:5.3.27
  spring-web:5.3.27
  spring-webmvc:5.3.27
  jackson-databind:2.14.2
  jackson-datatype-jsr310:2.14.2    ← phải tự đảm bảo tương thích
  hibernate-validator:8.0.0
  tomcat-embed-core:10.1.8
  tomcat-embed-el:10.1.8
  ... (10+ dependencies)

Dùng Starter:
  spring-boot-starter-web           ← 1 dòng duy nhất, Spring Boot lo phần còn lại
```

---

## 4. Auto-configuration Report — Báo Cáo Tự Động Cấu Hình

Khi muốn biết Auto-configuration nào được áp dụng hay bị từ chối, hãy bật debug mode:

```yaml
# application.yml
debug: true
```

Hoặc:
```bash
java -jar myapp.jar --debug
```

Output khi khởi động sẽ có section:
```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches (Auto-config đã áp dụng):
--------------------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource' (DataSourceAutoConfiguration)
      - @ConditionalOnMissingBean (types: javax.sql.DataSource) did not find any beans

Negative matches (Auto-config bị từ chối):
--------------------------
   MongoAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'com.mongodb.MongoClient'
```

### Actuator Endpoint — Xem Conditions Khi Runtime

```yaml
management:
  endpoints:
    web:
      exposure:
        include: conditions
```

Truy cập `GET /actuator/conditions` để xem toàn bộ Conditions Report khi app đang chạy.

---

## 5. Tắt Auto-configuration

Đôi khi Auto-configuration mặc định không phù hợp — bạn có thể tắt:

```java
// Tắt toàn bộ Security Auto-configuration (khi build pure REST API với JWT riêng)
@SpringBootApplication(exclude = {
    SecurityAutoConfiguration.class,
    UserDetailsServiceAutoConfiguration.class
})
public class MyApplication { ... }
```

```yaml
# Hoặc trong application.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

---

## 6. SpringApplication — Tùy Chỉnh Khởi Động

```java
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(MyApplication.class);

        // Tắt banner khi khởi động (con chữ Spring ASCII art)
        app.setBannerMode(Banner.Mode.OFF);

        // Thêm default properties
        app.setDefaultProperties(Map.of(
            "server.port", "8080",
            "spring.profiles.active", "local"
        ));

        app.run(args);
    }
}
```

### ApplicationRunner và CommandLineRunner — Chạy Code Khi Khởi Động

```java
// ApplicationRunner — chạy sau khi Spring Context đã sẵn sàng
@Component
public class DataInitializer implements ApplicationRunner {

    private final UserRepository userRepository;

    public DataInitializer(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Chạy một lần khi app khởi động
        if (userRepository.count() == 0) {
            userRepository.save(new User("admin", "admin@example.com"));
            log.info("Đã tạo dữ liệu mẫu");
        }
    }
}

// CommandLineRunner — tương tự nhưng nhận String[] args
@Component
@Order(1) // Thứ tự chạy khi có nhiều Runner
public class SchemaValidator implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        // validate database schema, v.v.
    }
}
```

---

## 7. Embedded Server — Server Nhúng

Spring Boot tích hợp sẵn Tomcat (mặc định), Jetty, hoặc Undertow — không cần deploy WAR.

```
Ứng dụng truyền thống:
  Code → WAR → Deploy lên Tomcat ngoài

Spring Boot:
  Code → JAR (có Tomcat nhúng bên trong) → java -jar app.jar
```

### Đổi Sang Jetty

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### Cấu Hình Server

```yaml
server:
  port: 8080
  servlet:
    context-path: /api      # prefix cho tất cả URL: /api/users, /api/products
  tomcat:
    max-threads: 200        # Số thread tối đa xử lý request
    connection-timeout: 5s  # Timeout kết nối
  compression:
    enabled: true           # Nén response (gzip)
    min-response-size: 1024 # Nén nếu response > 1KB
```

---

## 8. Build & Run

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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

```bash
# Build fat JAR (JAR chứa tất cả dependencies)
mvn clean package -DskipTests

# Chạy
java -jar target/myapp-0.0.1-SNAPSHOT.jar

# Chạy với profile cụ thể
java -jar target/myapp.jar --spring.profiles.active=production

# Chạy trực tiếp với Maven (cho development)
mvn spring-boot:run
```

### Gradle

```groovy
// build.gradle
plugins {
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'java'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

```bash
./gradlew bootJar   # Build JAR
./gradlew bootRun   # Chạy trực tiếp
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Q: Auto-configuration hoạt động thế nào?

**Trả lời:** Spring Boot sử dụng `@EnableAutoConfiguration` để scan tất cả JAR trong classpath, tìm file `AutoConfiguration.imports`. Mỗi Auto-configuration class được đánh dấu với các điều kiện `@ConditionalOn*`. Spring Boot chỉ áp dụng Auto-configuration khi tất cả điều kiện thỏa mãn. Quan trọng nhất là `@ConditionalOnMissingBean` — nếu user đã định nghĩa Bean cùng loại, Auto-configuration bị bỏ qua (user config luôn ưu tiên hơn).

### Q: Spring Boot Starter là gì? Tại sao cần Starters?

**Trả lời:** Starter là bộ dependency được đóng gói sẵn, đảm bảo tất cả thư viện trong bộ tương thích với nhau. Thay vì tự thêm 10 dependencies riêng lẻ và lo về version conflict, chỉ cần 1 Starter. Ví dụ `spring-boot-starter-web` tự động kéo Spring MVC, Tomcat, Jackson, Validation với các version đã được kiểm tra tương thích.

### Q: @SpringBootApplication gồm những gì?

**Trả lời:** Gồm 3 annotation:
1. `@SpringBootConfiguration` — đây là class cấu hình Spring (tương đương `@Configuration`)
2. `@EnableAutoConfiguration` — bật Auto-configuration
3. `@ComponentScan` — quét Bean từ package hiện tại và tất cả sub-packages

---

## ✅ Checklist Tự Kiểm Tra

- [ ] Tạo Spring Boot project từ [start.spring.io](https://start.spring.io) và chạy được
- [ ] Bật `debug: true` và đọc hiểu Auto-configuration Report
- [ ] Giải thích tại sao chỉ cần thêm `spring-boot-starter-data-jpa` và `DataSource` được tự động cấu hình
- [ ] Tắt một Auto-configuration cụ thể và kiểm tra hoạt động
- [ ] Viết một `ApplicationRunner` để tạo dữ liệu mẫu khi khởi động

---

**Tiếp Theo:** [4-bean-lifecycle.md](4-bean-lifecycle.md) — Bean Lifecycle và Scopes
