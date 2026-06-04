# Spring Security — Kiến Thức Nền Tảng

> Hiểu sâu về `SecurityFilterChain` (Chuỗi Bộ Lọc Bảo Mật), `AuthenticationManager` (Trình Quản Lý Xác Thực),
> `UserDetailsService` (Dịch Vụ Chi Tiết Người Dùng), và cách cấu hình Spring Security từ đầu.

---

## 1. Tại Sao Cần Spring Security?

Spring Security giải quyết các vấn đề bảo mật phổ biến:

- **Authentication** (Xác Thực) — xác minh danh tính người dùng
- **Authorization** (Phân Quyền) — kiểm soát quyền truy cập tài nguyên
- **Session Management** (Quản Lý Phiên) — duy trì trạng thái đăng nhập
- **CSRF Protection** (Bảo Vệ Giả Mạo Yêu Cầu) — ngăn tấn công cross-site
- **Security Headers** (Header Bảo Mật) — `X-Frame-Options`, `Content-Security-Policy`

---

## 2. Kiến Trúc Cốt Lõi

### 2.1 Security Filter Chain (Chuỗi Bộ Lọc Bảo Mật)

Spring Security hoạt động như một chuỗi `Servlet Filter`. Mỗi request HTTP đi qua các filter theo thứ tự:

```
HTTP Request
     │
     ▼
DelegatingFilterProxy          ← Servlet filter bridge sang Spring bean
     │
     ▼
FilterChainProxy               ← Quản lý nhiều SecurityFilterChain
     │
     ▼
SecurityFilterChain            ← Chuỗi filter được cấu hình của bạn
  ├── DisableEncodeUrlFilter
  ├── SecurityContextHolderFilter
  ├── HeaderWriterFilter         ← Thêm security headers
  ├── CorsFilter
  ├── CsrfFilter
  ├── LogoutFilter
  ├── UsernamePasswordAuthenticationFilter
  ├── BasicAuthenticationFilter
  ├── RequestCacheAwareFilter
  ├── SecurityContextHolderAwareRequestFilter
  ├── AnonymousAuthenticationFilter  ← Gán Anonymous nếu chưa auth
  ├── SessionManagementFilter
  ├── ExceptionTranslationFilter     ← Chuyển exception thành HTTP response
  └── AuthorizationFilter            ← Kiểm tra quyền truy cập
```

### 2.2 Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Chỉ thêm dependency này, Spring Boot **tự động**:
- Yêu cầu xác thực cho tất cả endpoints
- Tạo default user với password ngẫu nhiên (in ra console)
- Bật CSRF protection
- Bật HTTP Basic và Form Login

---

## 3. Cấu Hình SecurityFilterChain

### 3.1 Cấu Hình Cơ Bản

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Cấu hình authorization rules (quy tắc phân quyền)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/auth/**").permitAll()   // Cho phép tất cả
                .requestMatchers("/admin/**").hasRole("ADMIN")           // Chỉ ADMIN
                .requestMatchers(HttpMethod.GET, "/api/**").authenticated() // Cần đăng nhập
                .anyRequest().authenticated()
            )
            // Tắt CSRF cho REST API (stateless)
            .csrf(AbstractHttpConfigurer::disable)
            // Stateless session — không dùng HttpSession
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            // Xử lý khi unauthorized hoặc forbidden
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(customAuthEntryPoint)
                .accessDeniedHandler(customAccessDeniedHandler)
            );

        return http.build();
    }
}
```

### 3.2 Multiple Security Filter Chains (Nhiều Chuỗi Bộ Lọc)

Dùng khi có nhiều loại API (public API & admin API) với cấu hình khác nhau:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // Chain cho API công khai — ưu tiên cao hơn
    @Bean
    @Order(1)
    public SecurityFilterChain publicApiChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/public/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .csrf(AbstractHttpConfigurer::disable);
        return http.build();
    }

    // Chain mặc định cho toàn bộ ứng dụng
    @Bean
    @Order(2)
    public SecurityFilterChain defaultChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }
}
```

---

## 4. AuthenticationManager & AuthenticationProvider

### 4.1 Luồng Xác Thực

```
UsernamePasswordAuthenticationToken (chưa xác thực)
         │
         ▼
   AuthenticationManager
   (ProviderManager — Trình Quản Lý Provider)
         │
         ├──► DaoAuthenticationProvider
         │         │
         │         ▼
         │    UserDetailsService.loadUserByUsername(username)
         │         │
         │         ▼
         │    PasswordEncoder.matches(rawPassword, encodedPassword)
         │         │
         │         ▼
         └──► UsernamePasswordAuthenticationToken (đã xác thực)
                   │
                   ▼
            SecurityContextHolder.getContext().setAuthentication(...)
```

### 4.2 Khai Báo AuthenticationManager

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private UserDetailsService userDetailsService;

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // cost factor = 12
    }
}
```

---

## 5. UserDetailsService (Dịch Vụ Chi Tiết Người Dùng)

`UserDetailsService` là interface duy nhất cần implement để tích hợp với nguồn dữ liệu người dùng của bạn.

### 5.1 Implement UserDetailsService

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() ->
                new UsernameNotFoundException("Không tìm thấy user: " + username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPassword())    // Đã được BCrypt hash
            .roles(user.getRoles().toArray(new String[0]))  // Tự động thêm ROLE_ prefix
            .accountLocked(user.isLocked())
            .credentialsExpired(user.isPasswordExpired())
            .build();
    }
}
```

### 5.2 Entity User

```java
@Entity
@Table(name = "users")
@Data
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;   // BCrypt hash, không bao giờ plaintext

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "user_roles")
    private Set<String> roles = new HashSet<>();

    private boolean locked = false;
    private boolean passwordExpired = false;
    private boolean enabled = true;
}
```

---

## 6. PasswordEncoder (Bộ Mã Hóa Mật Khẩu)

### 6.1 BCryptPasswordEncoder

BCrypt là lựa chọn mặc định được khuyến nghị:

```java
PasswordEncoder encoder = new BCryptPasswordEncoder(12);

// Mã hóa mật khẩu khi đăng ký
String encoded = encoder.encode("myPassword123");
// Kết quả: $2a$12$eKx8oU.../rjL9HVHqbz.eS...

// Kiểm tra khi đăng nhập — luôn dùng matches(), không encode rồi so sánh
boolean valid = encoder.matches("myPassword123", encoded); // true
```

**Lưu ý về cost factor (hệ số chi phí):**

| Cost Factor | Thời gian (approx.) | Khuyến nghị |
|-------------|---------------------|-------------|
| 10 | ~100ms | Minimum cho production |
| 12 | ~300ms | **Khuyến nghị** |
| 14 | ~1000ms | Cho hệ thống bảo mật cao |

### 6.2 DelegatingPasswordEncoder (Hỗ Trợ Nâng Cấp)

Spring khuyến nghị dùng `DelegatingPasswordEncoder` để hỗ trợ migration giữa các thuật toán:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    // Mặc định dùng BCrypt, hỗ trợ đọc các loại hash cũ
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
// Password được lưu dạng: {bcrypt}$2a$10$...
```

---

## 7. SecurityContextHolder (Bộ Giữ Ngữ Cảnh Bảo Mật)

`SecurityContextHolder` lưu trữ thông tin người dùng đang đăng nhập trong thread hiện tại.

```java
// Lấy thông tin người dùng hiện tại từ bất kỳ đâu trong code
Authentication auth = SecurityContextHolder.getContext().getAuthentication();

if (auth != null && auth.isAuthenticated()) {
    String username = auth.getName();
    Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();

    // Cast sang UserDetails nếu cần thêm thông tin
    if (auth.getPrincipal() instanceof UserDetails userDetails) {
        System.out.println(userDetails.getUsername());
    }
}
```

**Lấy user hiện tại trong Controller:**

```java
@RestController
public class UserController {

    // Cách 1: @AuthenticationPrincipal (được khuyến nghị)
    @GetMapping("/me")
    public UserProfileResponse getProfile(
            @AuthenticationPrincipal UserDetails userDetails) {
        return userProfileService.getProfile(userDetails.getUsername());
    }

    // Cách 2: SecurityContextHolder
    @GetMapping("/me")
    public UserProfileResponse getProfile() {
        String username = SecurityContextHolder.getContext()
            .getAuthentication().getName();
        return userProfileService.getProfile(username);
    }
}
```

---

## 8. AuthenticationEntryPoint & AccessDeniedHandler

### 8.1 AuthenticationEntryPoint — Xử Lý 401 Unauthorized

```java
@Component
public class CustomAuthEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    @Override
    public void commence(HttpServletRequest request,
                         HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        Map<String, Object> body = Map.of(
            "status", 401,
            "error", "Unauthorized",
            "message", "Cần đăng nhập để truy cập tài nguyên này",
            "path", request.getServletPath()
        );

        objectMapper.writeValue(response.getOutputStream(), body);
    }
}
```

### 8.2 AccessDeniedHandler — Xử Lý 403 Forbidden

```java
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request,
                       HttpServletResponse response,
                       AccessDeniedException accessDeniedException) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);

        Map<String, Object> body = Map.of(
            "status", 403,
            "error", "Forbidden",
            "message", "Bạn không có quyền truy cập tài nguyên này",
            "path", request.getServletPath()
        );

        objectMapper.writeValue(response.getOutputStream(), body);
    }
}
```

---

## 9. Authorization Rules (Quy Tắc Phân Quyền)

### 9.1 requestMatchers (Khớp URL) — Spring Security 6+

```java
.authorizeHttpRequests(auth -> auth
    // Cho phép không cần đăng nhập
    .requestMatchers("/", "/error", "/actuator/health").permitAll()
    .requestMatchers(HttpMethod.POST, "/auth/login", "/auth/register").permitAll()

    // Yêu cầu role cụ thể
    .requestMatchers("/admin/**").hasRole("ADMIN")          // Phải có ROLE_ADMIN
    .requestMatchers("/api/reports/**").hasAnyRole("ADMIN", "MANAGER")

    // Yêu cầu authority cụ thể (không có ROLE_ prefix)
    .requestMatchers("/api/users/**").hasAuthority("USER_READ")

    // Kết hợp nhiều điều kiện với SpEL
    .requestMatchers("/api/sensitive/**").access(
        new WebExpressionAuthorizationManager("hasRole('ADMIN') and hasIpAddress('192.168.0.0/16')")
    )

    // Tất cả request còn lại cần đăng nhập
    .anyRequest().authenticated()
)
```

### 9.2 Sự Khác Biệt Role vs Authority

```
hasRole("ADMIN")      → kiểm tra GrantedAuthority: "ROLE_ADMIN"
hasAuthority("ADMIN") → kiểm tra GrantedAuthority: "ADMIN" (không thêm prefix)

// Khi dùng .roles("ADMIN") → Spring tự động thêm "ROLE_" prefix
// Khi dùng .authorities("ADMIN") → dùng nguyên như vậy
```

---

## 10. Ví Dụ Hoàn Chỉnh — Basic Auth Setup

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final CustomUserDetailsService userDetailsService;
    private final CustomAuthEntryPoint authEntryPoint;
    private final CustomAccessDeniedHandler accessDeniedHandler;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**", "/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider())
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s ->
                s.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            );

        return http.build();
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

---

## 11. Anti-Patterns Cần Tránh

| Anti-Pattern | Vấn Đề | Giải Pháp |
|---|---|---|
| Hardcode credentials trong code | Security risk, không thay đổi được | Dùng environment variables hoặc Vault |
| `permitAll()` cho tất cả endpoints | Vô hiệu hóa bảo mật | Chỉ permit những gì thực sự cần |
| Lưu password dạng plaintext | Rủi ro cực cao khi data breach | Luôn dùng BCryptPasswordEncoder |
| Dùng MD5/SHA1 cho password | Có thể brute-force | BCrypt/Argon2/scrypt với adaptive cost |
| Ignore exception từ authentication | User không biết lỗi gì | Implement AuthenticationEntryPoint |
| Dùng `antMatchers()` (deprecated) | Bị xóa trong Spring Security 6 | Dùng `requestMatchers()` |

---

## 12. Checklist Kiến Thức

- [ ] Hiểu SecurityFilterChain và thứ tự filter
- [ ] Biết implement `UserDetailsService` tùy chỉnh
- [ ] Cấu hình được `SecurityFilterChain` với authorization rules
- [ ] Hiểu sự khác biệt giữa role và authority
- [ ] Implement custom `AuthenticationEntryPoint` và `AccessDeniedHandler`
- [ ] Biết dùng `@AuthenticationPrincipal` trong controller
- [ ] Hiểu BCrypt cost factor và tác động hiệu năng

---

## 🔗 Bài Tiếp Theo

→ [2-jwt-implementation.md](2-jwt-implementation.md) — Implement JWT từ đầu với Access/Refresh Token
