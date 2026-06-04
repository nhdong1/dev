# gRPC Integration — Spring Boot & Protocol Buffers

> **gRPC** (gRPC Remote Procedure Call — Gọi Thủ Tục Từ Xa) là framework RPC hiệu năng cao, mã nguồn mở của Google, sử dụng **Protocol Buffers** (Protobuf — Bộ Đệm Giao Thức) làm định dạng serialization (tuần tự hóa) và HTTP/2 làm transport. gRPC phù hợp cho internal microservice communication (giao tiếp nội bộ giữa các microservices) với độ trễ thấp và hỗ trợ streaming.

---

## 📋 Mục Lục

1. [gRPC vs REST — Khi Nào Dùng Cái Nào?](#grpc-vs-rest--khi-nào-dùng-cái-nào)
2. [Protocol Buffers — Định Nghĩa Schema](#protocol-buffers--định-nghĩa-schema)
3. [Các Kiểu Giao Tiếp gRPC](#các-kiểu-giao-tiếp-grpc)
4. [Thiết Lập Dự Án Spring Boot gRPC](#thiết-lập-dự-án-spring-boot-grpc)
5. [gRPC Server — Phía Máy Chủ](#grpc-server--phía-máy-chủ)
6. [gRPC Client — Phía Máy Khách](#grpc-client--phía-máy-khách)
7. [Streaming — Luồng Dữ Liệu](#streaming--luồng-dữ-liệu)
8. [Error Handling — Xử Lý Lỗi](#error-handling--xử-lý-lỗi)
9. [Interceptors — Bộ Chặn](#interceptors--bộ-chặn)
10. [Security — Bảo Mật](#security--bảo-mật)
11. [Testing — Kiểm Thử](#testing--kiểm-thử)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## gRPC vs REST — Khi Nào Dùng Cái Nào?

```
                    REST / HTTP + JSON           gRPC / HTTP2 + Protobuf
                    ─────────────────           ───────────────────────
Protocol:           HTTP/1.1 (text)             HTTP/2 (binary)
Data Format:        JSON (human-readable)        Protobuf (binary, compact)
Schema:             OpenAPI (optional)           .proto file (bắt buộc)
Code Generation:    Manual / tools              Automatic từ .proto
Streaming:          Hạn chế (SSE, WebSocket)    Built-in bidirectional streaming
Browser Support:    Hoàn toàn ✅               Cần grpc-web proxy ⚠️
Performance:        ~                           5-10x nhanh hơn (ít payload hơn)
Debugging:          curl, Postman dễ dùng       Cần grpcurl / Postman gRPC
Human Readable:     ✅                          ❌ (binary)

DÙNG gRPC KHI:
✅ Internal microservice communication (không expose ra ngoài)
✅ Cần throughput cao và latency thấp
✅ Streaming data (real-time telemetry, log streaming)
✅ Polyglot services (Java gọi Go gọi Python — cùng .proto)
✅ Mobile backend khi bandwidth quan trọng

DÙNG REST KHI:
✅ Public API cho browser, mobile
✅ Team chưa quen với Protobuf
✅ Cần human-readable logs và debug dễ
✅ Third-party integrations
```

---

## Protocol Buffers — Định Nghĩa Schema

```protobuf
// src/main/proto/user_service.proto
syntax = "proto3";

package com.example.user;

option java_package = "com.example.grpc.user";
option java_outer_classname = "UserServiceProto";
option java_multiple_files = true;  // Tạo file Java riêng cho mỗi message

// Message — tương đương DTO
message UserRequest {
    int64 id = 1;  // Field number — không thay đổi sau khi deploy!
}

message CreateUserRequest {
    string name = 1;     // required trong proto3 = có giá trị mặc định
    string email = 2;
    int32 age = 3;
    repeated string roles = 4;  // Mảng
    optional string phone = 5;  // Có thể null
}

message UserResponse {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    repeated string roles = 5;
    google.protobuf.Timestamp created_at = 6;  // Dùng well-known types
    UserStatus status = 7;
}

// Enum
enum UserStatus {
    USER_STATUS_UNSPECIFIED = 0;  // Giá trị 0 là default trong proto3
    USER_STATUS_ACTIVE = 1;
    USER_STATUS_INACTIVE = 2;
    USER_STATUS_BANNED = 3;
}

message UserListResponse {
    repeated UserResponse users = 1;
    int32 total = 2;
    int32 page = 3;
}

// Service — định nghĩa các RPC methods
service UserService {
    // Unary — 1 request → 1 response
    rpc GetUser (UserRequest) returns (UserResponse);
    rpc CreateUser (CreateUserRequest) returns (UserResponse);

    // Server Streaming — 1 request → nhiều responses
    rpc ListUsers (ListUsersRequest) returns (stream UserResponse);

    // Client Streaming — nhiều requests → 1 response
    rpc BulkCreateUsers (stream CreateUserRequest) returns (BulkCreateResponse);

    // Bidirectional Streaming — nhiều requests → nhiều responses
    rpc SyncUsers (stream UserSyncRequest) returns (stream UserSyncResponse);
}
```

### Quy Tắc Quan Trọng Của Protobuf

```
BACKWARD COMPATIBILITY (Tương Thích Ngược):
- KHÔNG BAO GIỜ thay đổi field numbers đã deploy
- KHÔNG BAO GIỜ thay đổi tên service/method (ảnh hưởng URL)
- CÓ THỂ thêm field mới (backward compatible)
- CÓ THỂ xóa field (dùng `reserved` để đánh dấu)

reserved 6, 7;           // Giữ số để tránh tái sử dụng
reserved "old_field";    // Giữ tên để tránh tái sử dụng

WIRE TYPES (Kiểu Dữ Liệu Serialization):
int32, int64, bool, enum → Varint (nhỏ gọn cho số nhỏ)
string, bytes, nested messages → Length-delimited
float, double → 32-bit / 64-bit fixed
```

---

## Các Kiểu Giao Tiếp gRPC

```
1. UNARY RPC (Đơn Phương):
   Client ──[Request]──► Server
   Client ◄─[Response]── Server

2. SERVER STREAMING (Máy Chủ Phát Luồng):
   Client ──[Request]──────────► Server
   Client ◄─[Response 1]──────── Server
   Client ◄─[Response 2]──────── Server
   Client ◄─[Response N]──────── Server

3. CLIENT STREAMING (Máy Khách Phát Luồng):
   Client ──[Request 1]──────── ► Server
   Client ──[Request 2]──────── ► Server
   Client ──[Request N]──────── ► Server
   Client ◄─[Response]──────────  Server

4. BIDIRECTIONAL STREAMING (Hai Chiều):
   Client ──[Request 1]─►   Server
   Client ◄─[Response 1]──  Server
   Client ──[Request 2]─►   Server
   Client ◄─[Response 2]──  Server
   (Không cần theo thứ tự)
```

---

## Thiết Lập Dự Án Spring Boot gRPC

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Boot gRPC (grpc-spring-boot-starter từ net.devh) -->
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-spring-boot-starter</artifactId>
        <version>3.1.0.RELEASE</version>
    </dependency>

    <!-- Protobuf Java runtime -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <!-- Compile .proto files → Java classes -->
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>
                    com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}
                </protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>
                    io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}
                </pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

```yaml
# application.yml — Server config
grpc:
  server:
    port: 9090              # gRPC server port (khác với HTTP port 8080)
    max-inbound-message-size: 10MB

# application.yml — Client config
grpc:
  client:
    user-service:           # Tên channel
      address: static://localhost:9090
      negotiation-type: plaintext  # TLS trong production
```

---

## gRPC Server — Phía Máy Chủ

```java
// Service implementation — extend generated base class
@GrpcService  // Annotation của net.devh — đăng ký gRPC service
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {

    private final UserRepository userRepository;
    private final UserMapper mapper;

    // Unary RPC
    @Override
    public void getUser(UserRequest request,
                        StreamObserver<UserResponse> responseObserver) {
        try {
            User user = userRepository.findById(request.getId())
                .orElseThrow(() -> new StatusRuntimeException(
                    Status.NOT_FOUND.withDescription("User " + request.getId() + " không tồn tại")
                ));

            responseObserver.onNext(mapper.toProto(user));
            responseObserver.onCompleted();  // Kết thúc response

        } catch (StatusRuntimeException e) {
            responseObserver.onError(e);
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.getMessage())
                    .asRuntimeException()
            );
        }
    }

    // Unary RPC — tạo user
    @Override
    public void createUser(CreateUserRequest request,
                           StreamObserver<UserResponse> responseObserver) {
        // Validate
        if (request.getEmail().isBlank()) {
            responseObserver.onError(
                Status.INVALID_ARGUMENT
                    .withDescription("Email không được để trống")
                    .asRuntimeException()
            );
            return;
        }

        User user = userRepository.save(mapper.fromProto(request));
        responseObserver.onNext(mapper.toProto(user));
        responseObserver.onCompleted();
    }

    // Server Streaming RPC
    @Override
    public void listUsers(ListUsersRequest request,
                          StreamObserver<UserResponse> responseObserver) {
        // Phát từng user một qua stream
        userRepository.findAll().forEach(user -> {
            responseObserver.onNext(mapper.toProto(user));
        });
        responseObserver.onCompleted();  // Kết thúc sau khi gửi hết
    }
}
```

---

## gRPC Client — Phía Máy Khách

```java
@Service
public class UserServiceClient {

    // Inject channel và tạo stub tự động
    @GrpcClient("user-service")  // Tên khớp với application.yml grpc.client.user-service
    private UserServiceGrpc.UserServiceBlockingStub blockingStub;  // Blocking (đồng bộ)

    @GrpcClient("user-service")
    private UserServiceGrpc.UserServiceFutureStub futureStub;      // Async với ListenableFuture

    @GrpcClient("user-service")
    private UserServiceGrpc.UserServiceStub asyncStub;             // Async với StreamObserver

    // Blocking call (đồng bộ — giống REST)
    public UserDto getUser(Long id) {
        try {
            UserResponse response = blockingStub
                .withDeadlineAfter(5, TimeUnit.SECONDS)  // Timeout 5 giây
                .getUser(UserRequest.newBuilder().setId(id).build());

            return mapper.fromProto(response);

        } catch (StatusRuntimeException e) {
            if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
                throw new UserNotFoundException(id);
            }
            throw new ServiceException("Lỗi gRPC: " + e.getMessage(), e);
        }
    }

    // Async call — dùng CompletableFuture
    public CompletableFuture<UserDto> getUserAsync(Long id) {
        ListenableFuture<UserResponse> future = futureStub
            .withDeadlineAfter(5, TimeUnit.SECONDS)
            .getUser(UserRequest.newBuilder().setId(id).build());

        return CompletableFuture.supplyAsync(() -> {
            try {
                return mapper.fromProto(future.get(5, TimeUnit.SECONDS));
            } catch (Exception e) {
                throw new ServiceException("Lỗi gRPC async", e);
            }
        });
    }

    // Đọc server streaming
    public List<UserDto> listAllUsers() {
        List<UserDto> users = new ArrayList<>();
        CountDownLatch latch = new CountDownLatch(1);

        asyncStub.listUsers(
            ListUsersRequest.getDefaultInstance(),
            new StreamObserver<UserResponse>() {
                @Override
                public void onNext(UserResponse value) {
                    users.add(mapper.fromProto(value));
                }

                @Override
                public void onError(Throwable t) {
                    log.error("Stream lỗi: {}", t.getMessage());
                    latch.countDown();
                }

                @Override
                public void onCompleted() {
                    latch.countDown();
                }
            }
        );

        latch.await(30, TimeUnit.SECONDS);
        return users;
    }
}
```

---

## Streaming — Luồng Dữ Liệu

### Client Streaming — Upload Hàng Loạt

```java
// Server
@Override
public StreamObserver<CreateUserRequest> bulkCreateUsers(
        StreamObserver<BulkCreateResponse> responseObserver) {

    List<User> createdUsers = new ArrayList<>();

    return new StreamObserver<CreateUserRequest>() {
        @Override
        public void onNext(CreateUserRequest request) {
            // Nhận từng request từ client
            User user = userRepository.save(mapper.fromProto(request));
            createdUsers.add(user);
        }

        @Override
        public void onError(Throwable t) {
            log.error("Client stream lỗi: {}", t.getMessage());
        }

        @Override
        public void onCompleted() {
            // Client đã gửi xong — gửi response tổng kết
            responseObserver.onNext(
                BulkCreateResponse.newBuilder()
                    .setCreatedCount(createdUsers.size())
                    .build()
            );
            responseObserver.onCompleted();
        }
    };
}

// Client
public BulkCreateResponseDto bulkCreateUsers(List<CreateUserDto> users) {
    CountDownLatch latch = new CountDownLatch(1);
    AtomicReference<BulkCreateResponse> result = new AtomicReference<>();

    StreamObserver<CreateUserRequest> requestObserver =
        asyncStub.bulkCreateUsers(new StreamObserver<BulkCreateResponse>() {
            @Override
            public void onNext(BulkCreateResponse response) {
                result.set(response);
            }

            @Override
            public void onCompleted() {
                latch.countDown();
            }
            // ... onError
        });

    // Gửi từng request
    users.forEach(dto ->
        requestObserver.onNext(mapper.toProto(dto))
    );
    requestObserver.onCompleted(); // Báo client đã xong

    latch.await(60, TimeUnit.SECONDS);
    return mapper.fromProto(result.get());
}
```

---

## Error Handling — Xử Lý Lỗi

```java
// gRPC Status Codes — tương đương HTTP status
Status.OK               // 200 OK
Status.NOT_FOUND        // 404 Not Found
Status.INVALID_ARGUMENT // 400 Bad Request
Status.ALREADY_EXISTS   // 409 Conflict
Status.PERMISSION_DENIED// 403 Forbidden
Status.UNAUTHENTICATED  // 401 Unauthorized
Status.INTERNAL         // 500 Internal Server Error
Status.UNAVAILABLE      // 503 Service Unavailable
Status.DEADLINE_EXCEEDED// 504 Timeout

// Gửi lỗi có chi tiết (Status Details)
@Override
public void createUser(CreateUserRequest request,
                       StreamObserver<UserResponse> responseObserver) {
    List<FieldViolation> violations = validateRequest(request);
    if (!violations.isEmpty()) {
        // Thêm metadata vào error
        Metadata metadata = new Metadata();
        BadRequest badRequest = BadRequest.newBuilder()
            .addAllFieldViolations(violations)
            .build();

        responseObserver.onError(
            Status.INVALID_ARGUMENT
                .withDescription("Dữ liệu không hợp lệ")
                .asException(toTrailers(badRequest)) // Gắn metadata
        );
        return;
    }
    // ...
}

// Global exception handler với Interceptor
@GrpcGlobalServerInterceptor
public class ExceptionHandlingInterceptor implements ServerInterceptor {

    @Override
    public <Req, Resp> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Resp> call,
            Metadata headers,
            ServerCallHandler<Req, Resp> next) {

        ServerCall.Listener<Req> delegate = next.startCall(call, headers);

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<Req>(delegate) {
            @Override
            public void onHalfClose() {
                try {
                    super.onHalfClose();
                } catch (Exception e) {
                    call.close(
                        Status.INTERNAL.withDescription(e.getMessage()),
                        new Metadata()
                    );
                }
            }
        };
    }
}
```

---

## Interceptors — Bộ Chặn

```java
// Server Interceptor — logging, auth, metrics
@GrpcGlobalServerInterceptor
@Slf4j
public class LoggingServerInterceptor implements ServerInterceptor {

    @Override
    public <Req, Resp> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Resp> call,
            Metadata headers,
            ServerCallHandler<Req, Resp> next) {

        String method = call.getMethodDescriptor().getFullMethodName();
        long startTime = System.currentTimeMillis();

        log.info("gRPC call: {}", method);

        ServerCall<Req, Resp> loggingCall = new ForwardingServerCall
                .SimpleForwardingServerCall<Req, Resp>(call) {
            @Override
            public void close(Status status, Metadata trailers) {
                long duration = System.currentTimeMillis() - startTime;
                log.info("gRPC {} hoàn thành: {} ({}ms)", method, status.getCode(), duration);
                super.close(status, trailers);
            }
        };

        return next.startCall(loggingCall, headers);
    }
}

// Client Interceptor — thêm authentication header
@GrpcGlobalClientInterceptor
public class AuthClientInterceptor implements ClientInterceptor {

    private final JwtTokenProvider tokenProvider;

    @Override
    public <Req, Resp> ClientCall<Req, Resp> interceptCall(
            MethodDescriptor<Req, Resp> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<Req, Resp>(
                next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<Resp> responseListener, Metadata headers) {
                // Thêm JWT token vào metadata
                headers.put(
                    Metadata.Key.of("Authorization", Metadata.ASCII_STRING_MARSHALLER),
                    "Bearer " + tokenProvider.generateServiceToken()
                );
                super.start(responseListener, headers);
            }
        };
    }
}
```

---

## Security — Bảo Mật

```yaml
# TLS cho production
grpc:
  server:
    port: 443
    security:
      certificate-chain: classpath:server.crt
      private-key: classpath:server.key
  client:
    user-service:
      address: static://user-service:443
      security:
        certificate-chain: classpath:ca.crt
        negotiation-type: tls
```

```java
// Xác thực JWT trong server interceptor
@GrpcGlobalServerInterceptor
public class JwtServerInterceptor implements ServerInterceptor {

    private static final Metadata.Key<String> AUTH_KEY =
        Metadata.Key.of("Authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <Req, Resp> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Resp> call,
            Metadata headers,
            ServerCallHandler<Req, Resp> next) {

        String authHeader = headers.get(AUTH_KEY);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED.withDescription("Token thiếu"), new Metadata());
            return new ServerCall.Listener<Req>() {};
        }

        String token = authHeader.substring(7);
        try {
            Authentication auth = jwtValidator.validate(token);
            Context ctx = Context.current()
                .withValue(AuthContext.AUTH_KEY, auth);
            return Contexts.interceptCall(ctx, call, headers, next);
        } catch (JwtException e) {
            call.close(Status.UNAUTHENTICATED.withDescription("Token không hợp lệ"), new Metadata());
            return new ServerCall.Listener<Req>() {};
        }
    }
}
```

---

## Testing — Kiểm Thử

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest(properties = {
    "grpc.server.in-process-name=test",
    "grpc.server.port=-1"  // Tắt network server trong test
})
@DirtiesContext
class UserGrpcServiceTest {

    @GrpcClient("test")
    private UserServiceGrpc.UserServiceBlockingStub stub;

    @MockBean
    private UserRepository userRepository;

    @Test
    void testGetUser() {
        // Arrange
        when(userRepository.findById(1L))
            .thenReturn(Optional.of(new User(1L, "Nguyen Van A", "a@example.com")));

        // Act
        UserResponse response = stub.getUser(
            UserRequest.newBuilder().setId(1L).build()
        );

        // Assert
        assertThat(response.getId()).isEqualTo(1L);
        assertThat(response.getName()).isEqualTo("Nguyen Van A");
    }

    @Test
    void testGetUserNotFound() {
        when(userRepository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() ->
            stub.getUser(UserRequest.newBuilder().setId(99L).build())
        )
        .isInstanceOf(StatusRuntimeException.class)
        .matches(e -> ((StatusRuntimeException) e).getStatus().getCode()
                      == Status.Code.NOT_FOUND);
    }
}
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao gRPC nhanh hơn REST/JSON?**

A: Protobuf (binary) nhỏ hơn JSON 3-10 lần, serialization/deserialization nhanh hơn. HTTP/2 cho phép multiplexing (nhiều request trên 1 connection), header compression (nén header), và server push. Kết hợp cả hai, gRPC thường nhanh hơn 5-10x so với REST/JSON trong internal communication.

---

**Q: Sự khác biệt giữa 4 loại gRPC calls?**

A:
- **Unary**: 1 request → 1 response (giống REST GET/POST)
- **Server streaming**: 1 request → nhiều responses (ví dụ: export data, real-time feed)
- **Client streaming**: nhiều requests → 1 response (ví dụ: bulk upload, file upload)
- **Bidirectional streaming**: nhiều requests ↔ nhiều responses (ví dụ: chat, live collaboration)

---

**Q: Tại sao field numbers trong Protobuf quan trọng?**

A: Field numbers dùng để identify fields trong binary format, không phải field names. Nếu thay đổi field number, các clients dùng schema cũ sẽ đọc sai field. Quy tắc vàng: **không bao giờ thay đổi field numbers đã được deployed**. Có thể thêm field mới, xóa field (đánh `reserved`), nhưng không thay đổi hoặc tái sử dụng số cũ.

---

**Q: gRPC xử lý timeout như thế nào?**

A: Mỗi gRPC call có thể có `Deadline` (thời hạn tuyệt đối) hoặc timeout. Deadline được propagate tự động qua các service — nếu client A gọi B gọi C với deadline 5 giây, C sẽ biết còn bao nhiêu thời gian. Dùng `stub.withDeadlineAfter(5, TimeUnit.SECONDS)` hoặc `stub.withDeadline(Deadline.after(5, SECONDS))`.

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
