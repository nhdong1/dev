# JWT Implementation — Xác Thực Không Trạng Thái

> Hướng dẫn implement **JWT** (JSON Web Token — Token Web JSON) hoàn chỉnh trong Spring Boot:
> Access Token, Refresh Token, JwtFilter, Token Rotation, và xử lý logout.

---

## 1. JWT Là Gì?

**JWT** (JSON Web Token — Token Web JSON) là tiêu chuẩn mở [RFC 7519](https://tools.ietf.org/html/rfc7519) để truyền thông tin an toàn giữa các bên dưới dạng JSON object được ký số.

### Cấu Trúc JWT

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header (Base64URL)
.
eyJzdWIiOiJ1c2VyMTIzIiwicm9sZXMiOlsiUk9MRV9VU0VSIl0sImlhdCI6MTcxNzAwMDAwMCwiZXhwIjoxNzE3MDAzNjAwfQ
                                           ← Payload (Base64URL)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
                                           ← Signature (HMAC-SHA256)
```

**Header** — Thuật toán và loại token:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload** — Claims (Các Tuyên Bố) — dữ liệu được nhúng trong token:
```json
{
  "sub": "user123",               // subject — định danh user
  "roles": ["ROLE_USER"],         // custom claim
  "iat": 1717000000,              // issued at — thời điểm tạo
  "exp": 1717003600               // expiration — thời điểm hết hạn
}
```

**Signature** — Chữ ký xác thực:
```
HMAC-SHA256(base64url(header) + "." + base64url(payload), secret_key)
```

---

## 2. Access Token vs Refresh Token

| Thuộc Tính | Access Token | Refresh Token |
|------------|-------------|---------------|
| **Mục đích** | Truy cập API | Lấy Access Token mới |
| **Thời hạn** | Ngắn (15 phút – 1 giờ) | Dài (7–30 ngày) |
| **Lưu trữ** | Memory (JavaScript) | HttpOnly Cookie |
| **Gửi theo** | Authorization header | Cookie tự động |
| **Bị đánh cắp** | Thiệt hại giới hạn | Nguy hiểm hơn |

---

## 3. Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

---

## 4. Cấu Hình Application Properties

```yaml
# application.yml
app:
  jwt:
    secret: "404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970"   # 256-bit hex key
    access-token-expiration: 900000       # 15 phút (ms)
    refresh-token-expiration: 604800000   # 7 ngày (ms)
```

---

## 5. JwtUtil — Lớp Tiện Ích JWT

```java
@Component
public class JwtUtil {

    @Value("${app.jwt.secret}")
    private String secretKey;

    @Value("${app.jwt.access-token-expiration}")
    private long accessTokenExpiration;

    @Value("${app.jwt.refresh-token-expiration}")
    private long refreshTokenExpiration;

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    // Tạo Access Token
    public String generateAccessToken(UserDetails userDetails) {
        List<String> roles = userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList();

        return Jwts.builder()
            .subject(userDetails.getUsername())
            .claim("roles", roles)
            .claim("type", "access")
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + accessTokenExpiration))
            .signWith(getSigningKey())
            .compact();
    }

    // Tạo Refresh Token
    public String generateRefreshToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .claim("type", "refresh")
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + refreshTokenExpiration))
            .signWith(getSigningKey())
            .compact();
    }

    // Trích xuất username từ token
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    // Kiểm tra token còn hạn không
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    // Kiểm tra loại token (access hay refresh)
    public boolean isAccessToken(String token) {
        return "access".equals(extractClaim(token, claims -> claims.get("type", String.class)));
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    private <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        Claims claims = Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
        return claimsResolver.apply(claims);
    }
}
```

---

## 6. JwtAuthenticationFilter — Bộ Lọc Xác Thực JWT

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // 1. Trích xuất JWT từ Authorization header
        final String authHeader = request.getHeader("Authorization");

        // Bỏ qua nếu không có Bearer token
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        final String jwt = authHeader.substring(7); // Bỏ "Bearer " (7 ký tự)

        try {
            // 2. Trích xuất username từ token
            final String username = jwtUtil.extractUsername(jwt);

            // 3. Chỉ xử lý nếu username hợp lệ và chưa có authentication
            if (username != null &&
                    SecurityContextHolder.getContext().getAuthentication() == null) {

                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // 4. Kiểm tra token hợp lệ và là access token
                if (jwtUtil.isTokenValid(jwt, userDetails) && jwtUtil.isAccessToken(jwt)) {

                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,                          // credentials — null sau khi xác thực
                            userDetails.getAuthorities()   // roles/permissions
                        );

                    // Thêm request details vào authentication
                    authToken.setDetails(
                        new WebAuthenticationDetailsSource().buildDetails(request)
                    );

                    // 5. Set authentication vào SecurityContext
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (JwtException e) {
            // Token không hợp lệ — tiếp tục chain, filter sau sẽ xử lý 401
            // Không throw exception ở đây
        }

        filterChain.doFilter(request, response);
    }
}
```

### Đăng Ký Filter vào SecurityFilterChain

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final CustomUserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(AbstractHttpConfigurer::disable)
            // Thêm JwtFilter TRƯỚC UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## 7. AuthController — Endpoint Xác Thực

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final UserDetailsService userDetailsService;
    private final JwtUtil jwtUtil;
    private final RefreshTokenService refreshTokenService;

    // Đăng nhập — trả về access + refresh token
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        // Xác thực username/password — ném exception nếu sai
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(),
                request.getPassword()
            )
        );

        UserDetails userDetails = userDetailsService.loadUserByUsername(request.getUsername());

        String accessToken = jwtUtil.generateAccessToken(userDetails);
        String refreshToken = jwtUtil.generateRefreshToken(userDetails);

        // Lưu refresh token vào DB để có thể revoke
        refreshTokenService.save(request.getUsername(), refreshToken);

        return ResponseEntity.ok(new AuthResponse(accessToken, refreshToken));
    }

    // Làm mới access token bằng refresh token
    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshRequest request) {
        String username = jwtUtil.extractUsername(request.getRefreshToken());

        // Kiểm tra refresh token trong DB
        refreshTokenService.validate(username, request.getRefreshToken());

        UserDetails userDetails = userDetailsService.loadUserByUsername(username);
        String newAccessToken = jwtUtil.generateAccessToken(userDetails);

        return ResponseEntity.ok(new AuthResponse(newAccessToken, request.getRefreshToken()));
    }

    // Đăng xuất — xóa refresh token
    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@AuthenticationPrincipal UserDetails userDetails) {
        refreshTokenService.deleteByUsername(userDetails.getUsername());
        SecurityContextHolder.clearContext();
        return ResponseEntity.noContent().build();
    }
}
```

---

## 8. RefreshTokenService — Quản Lý Refresh Token

```java
@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;

    public void save(String username, String token) {
        // Xóa token cũ trước khi lưu mới (Token Rotation — Luân Chuyển Token)
        refreshTokenRepository.deleteByUsername(username);

        RefreshToken refreshToken = RefreshToken.builder()
            .username(username)
            .token(token)
            .expiresAt(Instant.now().plus(7, ChronoUnit.DAYS))
            .build();

        refreshTokenRepository.save(refreshToken);
    }

    public void validate(String username, String token) {
        RefreshToken storedToken = refreshTokenRepository
            .findByUsernameAndToken(username, token)
            .orElseThrow(() -> new InvalidTokenException("Refresh token không hợp lệ"));

        if (storedToken.getExpiresAt().isBefore(Instant.now())) {
            refreshTokenRepository.delete(storedToken);
            throw new InvalidTokenException("Refresh token đã hết hạn");
        }
    }

    public void deleteByUsername(String username) {
        refreshTokenRepository.deleteByUsername(username);
    }
}
```

### Entity RefreshToken

```java
@Entity
@Table(name = "refresh_tokens")
@Builder
@Data
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false, length = 512)
    private String token;

    @Column(nullable = false)
    private Instant expiresAt;
}
```

---

## 9. Token Rotation (Luân Chuyển Token)

Token Rotation là kỹ thuật bảo mật: mỗi khi dùng Refresh Token để lấy Access Token mới, Refresh Token cũ bị xóa và một Refresh Token mới được tạo ra.

```
Lần 1: Login  → AccessToken_1 + RefreshToken_1
Lần 2: Refresh với RefreshToken_1 → AccessToken_2 + RefreshToken_2 (RefreshToken_1 bị xóa)
Lần 3: Refresh với RefreshToken_2 → AccessToken_3 + RefreshToken_3 (RefreshToken_2 bị xóa)

Nếu kẻ tấn công dùng RefreshToken_1 sau lần 2 → 401 Unauthorized (đã bị xóa)
```

```java
@PostMapping("/refresh")
public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshRequest request) {
    String username = jwtUtil.extractUsername(request.getRefreshToken());
    refreshTokenService.validate(username, request.getRefreshToken());

    UserDetails userDetails = userDetailsService.loadUserByUsername(username);
    String newAccessToken = jwtUtil.generateAccessToken(userDetails);
    String newRefreshToken = jwtUtil.generateRefreshToken(userDetails);    // Token mới

    // Lưu token mới, xóa token cũ (token rotation)
    refreshTokenService.rotate(username, request.getRefreshToken(), newRefreshToken);

    return ResponseEntity.ok(new AuthResponse(newAccessToken, newRefreshToken));
}
```

---

## 10. DTO Classes

```java
// Request đăng nhập
public record LoginRequest(
    @NotBlank String username,
    @NotBlank String password
) {}

// Request làm mới token
public record RefreshRequest(
    @NotBlank String refreshToken
) {}

// Response sau đăng nhập / làm mới
public record AuthResponse(
    String accessToken,
    String refreshToken,
    String tokenType,       // "Bearer"
    long expiresIn          // seconds
) {
    public AuthResponse(String accessToken, String refreshToken) {
        this(accessToken, refreshToken, "Bearer", 900); // 15 phút
    }
}
```

---

## 11. Bảo Mật Nâng Cao

### 11.1 Xác Thực với RSA (Bất Đối Xứng) thay vì HMAC

Dùng RSA khi nhiều service cần xác thực token (public key có thể chia sẻ an toàn):

```java
// Tạo RSA key pair (lưu vào file hoặc Vault)
@Bean
public JwtDecoder jwtDecoder(RSAPublicKey publicKey) {
    return NimbusJwtDecoder.withPublicKey(publicKey).build();
}

@Bean
public JwtEncoder jwtEncoder(RSAPrivateKey privateKey, RSAPublicKey publicKey) {
    RSAKey rsaKey = new RSAKey.Builder(publicKey).privateKey(privateKey).build();
    JWKSource<SecurityContext> jwkSource = new ImmutableJWKSet<>(new JWKSet(rsaKey));
    return new NimbusJwtEncoder(jwkSource);
}
```

### 11.2 Token Blacklist (Danh Sách Đen Token)

Khi cần revoke access token ngay lập tức (trước khi hết hạn):

```java
@Service
@RequiredArgsConstructor
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;

    // Thêm vào blacklist với TTL bằng thời gian còn lại của token
    public void blacklist(String jti, long remainingMs) {
        redisTemplate.opsForValue()
            .set("blacklist:" + jti, "revoked",
                 Duration.ofMillis(remainingMs));
    }

    public boolean isBlacklisted(String jti) {
        return Boolean.TRUE.equals(
            redisTemplate.hasKey("blacklist:" + jti));
    }
}
```

---

## 12. Lỗi Thường Gặp

| Lỗi | Nguyên Nhân | Cách Sửa |
|-----|-------------|----------|
| `SignatureException` | Secret key thay đổi hoặc token bị tamper | Đảm bảo secret key nhất quán |
| `ExpiredJwtException` | Token đã hết hạn | Client cần gọi `/auth/refresh` |
| `MalformedJwtException` | Token bị cắt bớt hoặc sai format | Kiểm tra client gửi đúng token |
| `401 nhưng token còn hạn` | Filter không được đăng ký hoặc sai thứ tự | Kiểm tra `addFilterBefore` |
| `403 dù đúng role` | Role không có `ROLE_` prefix | Dùng `.roles("USER")` thay vì `.authorities("USER")` |

---

## 13. Checklist JWT

- [ ] JwtUtil tạo và validate token đúng cách
- [ ] JwtFilter chạy trước `UsernamePasswordAuthenticationFilter`
- [ ] Phân biệt Access Token (ngắn hạn) và Refresh Token (dài hạn)
- [ ] Refresh Token được lưu trong DB để có thể revoke
- [ ] Implement Token Rotation khi refresh
- [ ] Logout xóa Refresh Token khỏi DB
- [ ] Không lưu thông tin nhạy cảm trong JWT payload
- [ ] Secret key đủ độ phức tạp (≥256 bit)

---

## 🔗 Bài Tiếp Theo

→ [3-oauth2-oidc.md](3-oauth2-oidc.md) — OAuth2 & OIDC — Đăng Nhập Qua Bên Thứ Ba
