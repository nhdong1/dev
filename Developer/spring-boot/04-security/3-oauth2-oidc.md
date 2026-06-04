# OAuth2 & OIDC — Xác Thực Ủy Quyền

> **OAuth2** (Open Authorization — Ủy Quyền Mở) và **OIDC** (OpenID Connect — Kết Nối Định Danh Mở)
> là hai tiêu chuẩn phổ biến nhất cho phép đăng nhập qua bên thứ ba (Google, GitHub, Keycloak)
> và bảo vệ API trong kiến trúc microservices.

---

## 1. OAuth2 Là Gì?

OAuth2 là framework **ủy quyền** (authorization), không phải xác thực. Nó cho phép ứng dụng thứ ba truy cập tài nguyên thay mặt người dùng mà không cần biết mật khẩu.

### Các Vai Trò Trong OAuth2

```
Resource Owner   → Người dùng (bạn)
Client           → Ứng dụng muốn truy cập (app của bạn)
Authorization Server (AS) → Keycloak, Auth0, Google, GitHub
Resource Server  → API của bạn — được bảo vệ bởi token
```

---

## 2. OAuth2 Flows (Các Luồng OAuth2)

### 2.1 Authorization Code Flow (Luồng Mã Ủy Quyền) — Phổ Biến Nhất

Dành cho web app có backend. Bảo mật nhất vì client secret không lộ ra browser.

```
User           Browser/App         Your Backend      Authorization Server
  │                │                    │                    │
  │── Click Login ►│                    │                    │
  │                │── Redirect ────────────────────────────►│
  │                │   ?response_type=code                   │
  │                │   &client_id=xxx                        │
  │                │   &redirect_uri=xxx                     │
  │                │   &scope=openid profile email           │
  │                │   &state=random_string                  │
  │◄──────────────│ Login Page ─────────────────────────────│
  │── Login ──────────────────────────────────────────────►│
  │                │◄── Redirect với code ───────────────────│
  │                │    /callback?code=AUTH_CODE             │
  │                │── POST /token ─────────────────────────►│
  │                │   code + client_secret                  │
  │                │◄── access_token + refresh_token ────────│
  │                │    + id_token (OIDC)                    │
  │                │── GET /api/resource ──►│                │
  │                │   Bearer access_token  │                │
```

### 2.2 Authorization Code + PKCE (Proof Key for Code Exchange — Bằng Chứng Trao Đổi Mã)

Dành cho SPA (Single Page Application — Ứng Dụng Trang Đơn) và mobile app — không có client secret.

```
// Client tạo code_verifier và code_challenge
code_verifier  = random_string(43-128 chars)
code_challenge = BASE64URL(SHA256(code_verifier))

// Gửi code_challenge trong authorization request
// Gửi code_verifier khi exchange code → server tự verify
```

### 2.3 Client Credentials Flow (Luồng Thông Tin Client)

Dành cho machine-to-machine communication — không có người dùng:

```
Service A ──► POST /token ──► Authorization Server
             grant_type=client_credentials
             client_id + client_secret
             ◄── access_token
Service A ──► GET /api/data ──► Service B (Resource Server)
             Bearer access_token
```

---

## 3. OIDC (OpenID Connect) — Tầng Xác Thực

OIDC là lớp **xác thực** (authentication) xây dựng trên OAuth2. Nó bổ sung **ID Token** (JWT chứa thông tin người dùng).

### ID Token Claims Chuẩn

```json
{
  "iss": "https://accounts.google.com",      // issuer — ai cấp token
  "sub": "10965150351106250955",             // subject — user ID
  "aud": "1234987819200.apps.googleusercontent.com",  // audience — app của bạn
  "exp": 1311281970,                          // expiration
  "iat": 1311280970,                          // issued at
  "email": "user@example.com",               // email
  "email_verified": true,
  "name": "Nguyễn Văn A",
  "picture": "https://..."
}
```

### OIDC Scopes (Phạm Vi)

| Scope | Thông Tin Trả Về |
|-------|-----------------|
| `openid` | Bắt buộc — kích hoạt OIDC, trả về `sub` |
| `profile` | `name`, `given_name`, `picture` |
| `email` | `email`, `email_verified` |
| `address` | `address` |
| `phone` | `phone_number` |

---

## 4. Spring Boot OAuth2 Client (Đăng Nhập Qua Bên Thứ Ba)

### 4.1 Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

### 4.2 Cấu Hình application.yml

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"

          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email

          keycloak:
            client-id: my-app
            client-secret: ${KEYCLOAK_SECRET}
            authorization-grant-type: authorization_code
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"

        provider:
          keycloak:
            issuer-uri: http://localhost:8080/realms/my-realm
```

### 4.3 Security Config cho OAuth2 Login

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login**", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)     // Xử lý user từ OAuth2
                    .oidcUserService(customOidcUserService)   // Xử lý user từ OIDC
                )
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .deleteCookies("JSESSIONID")
            );

        return http.build();
    }
}
```

### 4.4 Custom OAuth2UserService — Lưu User Vào DB

```java
@Service
@RequiredArgsConstructor
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oauth2User = super.loadUser(userRequest);

        String provider = userRequest.getClientRegistration().getRegistrationId(); // "google", "github"
        String providerId = oauth2User.getName(); // subject
        String email = oauth2User.getAttribute("email");
        String name = oauth2User.getAttribute("name");

        // Tìm hoặc tạo user trong DB
        User user = userRepository.findByProviderAndProviderId(provider, providerId)
            .orElseGet(() -> createNewUser(provider, providerId, email, name));

        return oauth2User;
    }

    private User createNewUser(String provider, String providerId, String email, String name) {
        User user = User.builder()
            .provider(provider)
            .providerId(providerId)
            .email(email)
            .name(name)
            .roles(Set.of("ROLE_USER"))
            .build();
        return userRepository.save(user);
    }
}
```

---

## 5. Spring Boot OAuth2 Resource Server (Bảo Vệ API)

Resource Server dùng để **bảo vệ API** — xác thực Bearer token trong request.

### 5.1 Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### 5.2 Cấu Hình Resource Server với JWT

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Cách 1: Dùng JWKS URI từ Authorization Server (khuyến nghị)
          jwk-set-uri: http://localhost:8080/realms/my-realm/protocol/openid-connect/certs

          # Cách 2: Dùng issuer-uri (tự động discover JWKS)
          issuer-uri: http://localhost:8080/realms/my-realm
```

### 5.3 Security Config cho Resource Server

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS));

        return http.build();
    }

    // Chuyển đổi JWT claims thành Spring Security authorities
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
            new JwtGrantedAuthoritiesConverter();

        // Keycloak lưu roles trong claim "realm_access.roles"
        grantedAuthoritiesConverter.setAuthoritiesClaimName("realm_access.roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
        jwtConverter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return jwtConverter;
    }
}
```

### 5.4 Lấy Thông Tin User Từ JWT Token

```java
@RestController
@RequestMapping("/api")
public class ApiController {

    // Cách 1: @AuthenticationPrincipal Jwt
    @GetMapping("/me")
    public Map<String, Object> getCurrentUser(@AuthenticationPrincipal Jwt jwt) {
        return Map.of(
            "subject", jwt.getSubject(),
            "username", jwt.getClaimAsString("preferred_username"),
            "email", jwt.getClaimAsString("email"),
            "roles", jwt.getClaimAsStringList("roles")
        );
    }

    // Cách 2: @AuthenticationPrincipal OidcUser (khi dùng OAuth2 Login)
    @GetMapping("/profile")
    public Map<String, Object> getProfile(@AuthenticationPrincipal OidcUser oidcUser) {
        return Map.of(
            "name", oidcUser.getFullName(),
            "email", oidcUser.getEmail(),
            "picture", oidcUser.getPicture()
        );
    }
}
```

---

## 6. Tích Hợp Keycloak (Identity Provider Phổ Biến)

Keycloak là **Identity Provider** (Nhà Cung Cấp Định Danh) mã nguồn mở phổ biến nhất cho Spring Boot.

### 6.1 Cấu Hình Keycloak

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: spring-app
            client-secret: ${KEYCLOAK_SECRET}
            scope: openid, profile, email, roles
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/keycloak"
        provider:
          keycloak:
            issuer-uri: http://keycloak:8080/realms/my-realm
            user-name-attribute: preferred_username

      resourceserver:
        jwt:
          issuer-uri: http://keycloak:8080/realms/my-realm
```

### 6.2 Xử Lý Roles Từ Keycloak JWT

Keycloak lưu roles ở các vị trí khác nhau trong token:

```json
{
  "realm_access": {
    "roles": ["ROLE_USER", "ROLE_ADMIN"]   // ← Realm roles
  },
  "resource_access": {
    "spring-app": {
      "roles": ["ROLE_APP_ADMIN"]           // ← Client roles
    }
  }
}
```

```java
@Bean
public JwtAuthenticationConverter keycloakAuthenticationConverter() {
    Converter<Jwt, Collection<GrantedAuthority>> rolesConverter = jwt -> {
        // Lấy realm roles
        Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
        List<String> realmRoles = realmAccess != null
            ? (List<String>) realmAccess.get("roles")
            : List.of();

        // Lấy client roles
        Map<String, Object> resourceAccess = jwt.getClaimAsMap("resource_access");
        List<String> clientRoles = resourceAccess != null
            ? extractClientRoles(resourceAccess, "spring-app")
            : List.of();

        return Stream.concat(realmRoles.stream(), clientRoles.stream())
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
            .collect(Collectors.toList());
    };

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(rolesConverter);
    return converter;
}
```

---

## 7. OAuth2 vs JWT Tự Implement

| Tiêu Chí | JWT Tự Implement | OAuth2 / Keycloak |
|----------|-----------------|-------------------|
| **Độ phức tạp** | Thấp hơn | Cao hơn |
| **Single Sign-On** | Không | Có |
| **Quản lý user** | Tự build | Keycloak có UI sẵn |
| **MFA** | Tự build | Keycloak hỗ trợ sẵn |
| **Social Login** | Tự tích hợp | Keycloak hỗ trợ sẵn |
| **Token revocation** | Cần Redis blacklist | Keycloak xử lý |
| **Phù hợp với** | App nhỏ, ít service | Microservices, enterprise |

---

## 8. Checklist OAuth2 & OIDC

- [ ] Hiểu sự khác biệt giữa OAuth2 (authorization) và OIDC (authentication)
- [ ] Biết khi nào dùng Authorization Code Flow vs Client Credentials Flow
- [ ] Cấu hình Spring Boot như OAuth2 Resource Server
- [ ] Biết convert JWT claims thành Spring Security roles
- [ ] Lấy thông tin user từ `@AuthenticationPrincipal Jwt`
- [ ] Hiểu PKCE và khi nào cần dùng
- [ ] Biết tích hợp với Keycloak trong môi trường microservices

---

## 🔗 Bài Tiếp Theo

→ [4-method-security.md](4-method-security.md) — Method-Level Security với @PreAuthorize
