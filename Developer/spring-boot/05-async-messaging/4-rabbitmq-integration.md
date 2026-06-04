# RabbitMQ Integration với Spring Boot

> Tích hợp RabbitMQ (Hệ Thống Hàng Đợi Nhắn Tin) vào Spring Boot —
> từ cấu hình Exchange (Bộ Trao Đổi), Queue (Hàng Đợi), Binding (Liên Kết),
> gửi message với `RabbitTemplate`, nhận với `@RabbitListener`,
> đến Dead Letter Queue (Hàng Đợi Thư Chết), retry và các exchange patterns.

---

## 1. Kiến Trúc RabbitMQ

### AMQP Model (Mô Hình AMQP — Advanced Message Queuing Protocol)

```
Producer (Nhà Sản Xuất)      Exchange (Bộ Trao Đổi)        Queue (Hàng Đợi)     Consumer
                            ┌────────────────────────┐
                            │                        │──── Binding ────► [Queue A] ──► Consumer 1
Publisher ──► routingKey ──►│   Exchange             │
                            │   (Direct/Fanout/      │──── Binding ────► [Queue B] ──► Consumer 2
                            │    Topic/Headers)      │
                            └────────────────────────┘                   [Queue C] ──► Consumer 3
                                                                         (unrouted → discard/DLQ)
```

### Các Loại Exchange

| Exchange Type | Cách Route | Use Case |
|--------------|-----------|----------|
| **Direct** (Trực Tiếp) | So khớp chính xác `routingKey` với `bindingKey` | Task queue, point-to-point |
| **Fanout** (Phát Rộng) | Gửi đến **tất cả** queues bound | Broadcast, pub/sub |
| **Topic** (Chủ Đề) | Pattern matching với `*` (một word) và `#` (nhiều words) | Flexible routing |
| **Headers** (Tiêu Đề) | Match dựa trên message headers (không dùng routingKey) | Phức tạp, ít dùng |

### So Sánh Với Kafka

| | RabbitMQ | Apache Kafka |
|---|---------|-------------|
| **Mô hình** | Push-based — broker đẩy message đến consumer | Pull-based — consumer tự kéo |
| **Message sau consume** | Xóa khỏi queue | Giữ lại (configurable retention) |
| **Replay** | Không (phải lưu vào DLQ) | Có — consumer có thể seek offset |
| **Routing** | Linh hoạt — Direct/Fanout/Topic/Headers | Đơn giản — topic + partition |
| **Throughput** | Vừa phải (hàng chục nghìn/giây) | Cực cao (triệu/giây) |
| **Setup** | Đơn giản | Phức tạp (Zookeeper/KRaft) |

---

## 2. Cấu Hình (Configuration)

### 2.1 Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 2.2 application.properties

```properties
# ─── RabbitMQ Connection ──────────────────────────────────────────────────
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/

# ─── Listener Config ─────────────────────────────────────────────────────
# simple → 1 container/queue; direct → dùng consumer thread trực tiếp
spring.rabbitmq.listener.type=simple

# Số concurrent consumers (Số Consumer Song Song)
spring.rabbitmq.listener.simple.concurrency=3
spring.rabbitmq.listener.simple.max-concurrency=10

# Prefetch Count (Số Message Tải Trước): số message consumer nhận mà chưa ack
# = 1: fair dispatch — chỉ nhận message mới sau khi ack message cũ
# Tăng lên để tăng throughput
spring.rabbitmq.listener.simple.prefetch=1

# Acknowledgment Mode (Chế Độ Xác Nhận)
# auto → Spring tự ack/nack dựa trên exception
# manual → code phải ack/nack thủ công
spring.rabbitmq.listener.simple.acknowledge-mode=auto

# Retry (Thử Lại) trong container
spring.rabbitmq.listener.simple.retry.enabled=true
spring.rabbitmq.listener.simple.retry.max-attempts=3
spring.rabbitmq.listener.simple.retry.initial-interval=1000ms
spring.rabbitmq.listener.simple.retry.multiplier=2
spring.rabbitmq.listener.simple.retry.max-interval=10000ms
```

### 2.3 Java Config — Khai Báo Exchange, Queue, Binding

```java
@Configuration
public class RabbitMQConfig {

    // ─── Constants ────────────────────────────────────────────────────────
    public static final String ORDER_EXCHANGE     = "order.exchange";
    public static final String ORDER_QUEUE        = "order.queue";
    public static final String ORDER_ROUTING_KEY  = "order.created";

    public static final String DLX_EXCHANGE       = "order.dlx.exchange";   // Dead Letter Exchange
    public static final String DLQ_QUEUE          = "order.dlq";             // Dead Letter Queue

    // ─── Dead Letter Infrastructure ──────────────────────────────────────
    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange(DLX_EXCHANGE);
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable(DLQ_QUEUE).build();
    }

    @Bean
    public Binding deadLetterBinding() {
        return BindingBuilder.bind(deadLetterQueue())
                             .to(deadLetterExchange())
                             .with(ORDER_ROUTING_KEY);
    }

    // ─── Main Exchange & Queue ────────────────────────────────────────────
    @Bean
    public DirectExchange orderExchange() {
        return ExchangeBuilder.directExchange(ORDER_EXCHANGE)
                              .durable(true)   // Tồn tại sau khi RabbitMQ restart
                              .build();
    }

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
            // Dead Letter Exchange — messages bị reject/expire → gửi đến DLX
            .withArgument("x-dead-letter-exchange", DLX_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", ORDER_ROUTING_KEY)
            // TTL (Time-To-Live — Thời Gian Sống): message tự expire sau 24h
            .withArgument("x-message-ttl", 86400000)
            build();
    }

    @Bean
    public Binding orderBinding() {
        return BindingBuilder.bind(orderQueue())
                             .to(orderExchange())
                             .with(ORDER_ROUTING_KEY);
    }

    // ─── Message Converter (Bộ Chuyển Đổi Message) ───────────────────────
    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        return template;
    }
}
```

---

## 3. Các Loại Exchange — Ví Dụ Chi Tiết

### 3.1 Direct Exchange (Trao Đổi Trực Tiếp)

```java
// Config
@Bean
public DirectExchange notificationExchange() {
    return new DirectExchange("notification.exchange");
}

@Bean
public Queue emailQueue() { return QueueBuilder.durable("notification.email").build(); }

@Bean
public Queue smsQueue() { return QueueBuilder.durable("notification.sms").build(); }

@Bean
public Binding emailBinding() {
    return BindingBuilder.bind(emailQueue())
                         .to(notificationExchange())
                         .with("email");    // routingKey = "email"
}

@Bean
public Binding smsBinding() {
    return BindingBuilder.bind(smsQueue())
                         .to(notificationExchange())
                         .with("sms");      // routingKey = "sms"
}

// Producer
rabbitTemplate.convertAndSend("notification.exchange", "email", emailPayload);
rabbitTemplate.convertAndSend("notification.exchange", "sms", smsPayload);
```

### 3.2 Fanout Exchange (Trao Đổi Phát Rộng) — Broadcast

```java
@Bean
public FanoutExchange userEventExchange() {
    return new FanoutExchange("user.event.fanout");
}

// Tất cả queues bound đều nhận message — bỏ qua routingKey
@Bean
public Binding analyticsBinding() {
    return BindingBuilder.bind(analyticsQueue()).to(userEventExchange());
}

@Bean
public Binding auditBinding() {
    return BindingBuilder.bind(auditQueue()).to(userEventExchange());
}

// Producer — routingKey bị bỏ qua với Fanout
rabbitTemplate.convertAndSend("user.event.fanout", "", userEvent);
```

### 3.3 Topic Exchange (Trao Đổi Theo Chủ Đề) — Flexible Routing

```java
@Bean
public TopicExchange appExchange() {
    return new TopicExchange("app.exchange");
}

// Pattern: * = 1 word, # = 0 hoặc nhiều words
@Bean
public Binding orderBinding() {
    // Nhận "order.*" → order.created, order.paid, order.cancelled
    return BindingBuilder.bind(orderQueue()).to(appExchange()).with("order.*");
}

@Bean
public Binding allEventsBinding() {
    // Nhận "#" → mọi message
    return BindingBuilder.bind(auditQueue()).to(appExchange()).with("#");
}

@Bean
public Binding usEventsBinding() {
    // Nhận "us.#" → us.order.created, us.user.registered...
    return BindingBuilder.bind(usQueue()).to(appExchange()).with("us.#");
}

// Producer
rabbitTemplate.convertAndSend("app.exchange", "order.created", event);     // → orderQueue + auditQueue
rabbitTemplate.convertAndSend("app.exchange", "us.order.paid", event);     // → usQueue + auditQueue
rabbitTemplate.convertAndSend("app.exchange", "user.registered", event);   // → auditQueue
```

---

## 4. RabbitTemplate — Gửi Message

### 4.1 Gửi Object (Auto Convert)

```java
@Service
public class OrderMessageProducer {

    private final RabbitTemplate rabbitTemplate;

    public OrderMessageProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void publishOrderCreated(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_ROUTING_KEY,
            event
        );
    }

    // Gửi với message properties (thuộc tính message)
    public void publishWithProperties(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_ROUTING_KEY,
            event,
            message -> {
                message.getMessageProperties().setExpiration("30000"); // TTL 30s
                message.getMessageProperties().setPriority(5);         // Priority
                message.getMessageProperties().setCorrelationId(UUID.randomUUID().toString());
                return message;
            }
        );
    }
}
```

### 4.2 RPC Pattern (Remote Procedure Call — Gọi Thủ Tục Từ Xa)

```java
// Gửi và chờ response — synchronous RPC qua RabbitMQ
public PriceQuote requestPriceQuote(PriceRequest request) {
    return (PriceQuote) rabbitTemplate.convertSendAndReceive(
        "pricing.exchange",
        "pricing.request",
        request
    );
}

// Consumer phía server
@RabbitListener(queues = "pricing.request.queue")
public PriceQuote handlePricingRequest(PriceRequest request) {
    return pricingService.calculate(request); // Return value → gửi về requester
}
```

---

## 5. @RabbitListener — Nhận Message

### 5.1 Cơ Bản

```java
@Component
public class OrderConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderConsumer.class);

    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE)
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Received order: {}", event.orderId());
        inventoryService.deductStock(event);
    }
}
```

### 5.2 Với Message Headers và Channel

```java
@Component
public class ReliableOrderConsumer {

    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE,
                    ackMode = "MANUAL")
    public void handleOrder(
            @Payload OrderCreatedEvent event,
            @Headers Map<String, Object> headers,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        try {
            processOrder(event);
            channel.basicAck(deliveryTag, false);    // ← Xác nhận thành công
        } catch (RecoverableException ex) {
            // Nack với requeue=true → message quay lại queue để retry
            channel.basicNack(deliveryTag, false, true);
        } catch (UnrecoverableException ex) {
            // Nack với requeue=false → message đến DLQ
            channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

### 5.3 Declare Queue Trực Tiếp Trong @RabbitListener

```java
@Component
public class DynamicQueueConsumer {

    @RabbitListener(bindings = @QueueBinding(
        value = @Queue(
            value = "notification.queue",
            durable = "true",
            arguments = {
                @Argument(name = "x-dead-letter-exchange", value = "notification.dlx"),
                @Argument(name = "x-message-ttl", value = "3600000", type = "java.lang.Long")
            }
        ),
        exchange = @Exchange(value = "notification.exchange", type = ExchangeTypes.TOPIC),
        key = "notification.#"
    ))
    public void handleNotification(NotificationEvent event) {
        notificationService.send(event);
    }
}
```

### 5.4 Concurrent Consumers (Consumer Song Song)

```java
@RabbitListener(
    queues = "order.queue",
    concurrency = "3-10"  // min 3, max 10 concurrent consumers
)
public void handleOrder(OrderCreatedEvent event) { ... }
```

---

## 6. Dead Letter Queue (Hàng Đợi Thư Chết)

### Khi Nào Message Vào DLQ?

```
Message vào DLQ (Dead Letter Queue) khi:
1. Consumer reject với requeue=false:  channel.basicNack(tag, false, false)
2. Message hết TTL (x-message-ttl)
3. Queue đầy (x-max-length đạt giới hạn)
4. Spring retry exhausted (spring.rabbitmq.listener.simple.retry.max-attempts)
```

### Xử Lý DLQ

```java
@Component
public class DeadLetterQueueConsumer {

    @RabbitListener(queues = RabbitMQConfig.DLQ_QUEUE)
    public void handleDeadLetter(
            Message message,
            @Header(value = "x-death", required = false) List<Map<String, Object>> deaths) {

        log.error("Dead letter message: body={}", new String(message.getBody()));

        if (deaths != null && !deaths.isEmpty()) {
            Map<String, Object> lastDeath = deaths.get(0);
            log.error("  Reason: {}, Queue: {}, Count: {}",
                lastDeath.get("reason"),
                lastDeath.get("queue"),
                lastDeath.get("count"));
        }

        // Lưu vào DB để điều tra
        deadLetterRepository.save(DeadLetterRecord.from(message));

        // Gửi alert
        alertService.sendAlert("Dead letter detected", message);
    }
}
```

---

## 7. Message Conversion (Chuyển Đổi Message)

### JSON Converter (Mặc Định Được Dùng)

```java
@Bean
public MessageConverter jsonMessageConverter() {
    Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();

    // Custom ObjectMapper nếu cần
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    objectMapper.registerModule(new JavaTimeModule());

    return new Jackson2JsonMessageConverter(objectMapper);
}
```

### Xử Lý Nhiều Event Types Từ 1 Queue

```java
@RabbitListener(queues = "event.queue")
public void handleEvents(@Payload String jsonPayload,
                         @Header("__TypeId__") String typeId) {
    if (typeId.contains("OrderCreatedEvent")) {
        OrderCreatedEvent event = objectMapper.readValue(jsonPayload, OrderCreatedEvent.class);
        handleOrderCreated(event);
    } else if (typeId.contains("PaymentEvent")) {
        PaymentEvent event = objectMapper.readValue(jsonPayload, PaymentEvent.class);
        handlePayment(event);
    }
}
```

---

## 8. Publisher Confirms & Returns (Xác Nhận & Trả Về Từ Publisher)

```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    template.setMessageConverter(jsonMessageConverter());

    // Publisher Confirms (Xác Nhận Publisher): broker xác nhận đã nhận message
    template.setConfirmCallback((correlationData, ack, cause) -> {
        if (!ack) {
            log.error("Message not confirmed: correlationId={}, cause={}",
                    correlationData != null ? correlationData.getId() : null, cause);
            // Retry hoặc lưu vào outbox
        }
    });

    // Mandatory + Returns: nhận lại message nếu không route được
    template.setMandatory(true);
    template.setReturnsCallback(returned -> {
        log.error("Message returned: exchange={}, routingKey={}, replyCode={}",
                returned.getExchange(), returned.getRoutingKey(), returned.getReplyCode());
    });

    return template;
}

// Bật confirms trong connection factory
@Bean
public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory("localhost");
    factory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);
    factory.setPublisherReturns(true);
    return factory;
}
```

---

## 9. Outbox Pattern (Mẫu Hộp Thư) — Đảm Bảo Reliability

Vấn đề: Lưu DB thành công nhưng gửi RabbitMQ thất bại → mất event.

```java
// Bước 1: Lưu vào outbox table cùng transaction với business data
@Transactional
public Order createOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));

    // Lưu event vào outbox — cùng transaction!
    outboxRepository.save(new OutboxEvent(
        "ORDER_CREATED",
        objectMapper.writeValueAsString(new OrderCreatedEvent(order.getId(), ...))
    ));

    return order;
}

// Bước 2: Scheduler poll outbox và publish
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishOutboxEvents() {
    List<OutboxEvent> pendingEvents = outboxRepository.findByStatus(PENDING);
    for (OutboxEvent event : pendingEvents) {
        try {
            rabbitTemplate.convertAndSend(
                event.getExchange(),
                event.getRoutingKey(),
                event.getPayload()
            );
            event.setStatus(PUBLISHED);
            outboxRepository.save(event);
        } catch (Exception ex) {
            log.error("Failed to publish outbox event: {}", event.getId(), ex);
        }
    }
}
```

---

## 10. Monitoring với Management UI

RabbitMQ có Management UI tại `http://localhost:15672`:
- Xem queue depth (chiều sâu hàng đợi), consumer count, message rates
- Manual publish, purge queue, check DLQ

```yaml
# docker-compose.yml
rabbitmq:
  image: rabbitmq:3-management
  ports:
    - "5672:5672"    # AMQP port
    - "15672:15672"  # Management UI port
  environment:
    RABBITMQ_DEFAULT_USER: guest
    RABBITMQ_DEFAULT_PASS: guest
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Direct vs Topic Exchange — khi nào dùng cái nào?**
> **Direct**: routing đơn giản, 1 routingKey khớp chính xác 1 queue — dùng cho task queues. **Topic**: routing linh hoạt với wildcard `*` và `#` — dùng khi cần filter events theo pattern (vd: `order.*` nhận tất cả order events, `us.#` nhận tất cả events từ US).

**Q: Dead Letter Queue là gì và khi nào message vào DLQ?**
> DLQ nhận messages bị "chết" — không xử lý được. Xảy ra khi: (1) consumer nack với requeue=false, (2) message hết TTL, (3) queue đầy. DLQ cho phép: điều tra lỗi, retry thủ công, alert monitoring.

**Q: Prefetch count ảnh hưởng thế nào đến performance?**
> `prefetch=1`: fair dispatch — consumer bận thì không nhận message mới → throughput thấp nhưng phân phối đều. `prefetch=N`: consumer nhận N messages trước rồi mới ack → throughput cao hơn nhưng không fair khi consumers có tốc độ khác nhau. Không set prefetch → broker gửi không giới hạn → consumer chậm bị overwhelmed (quá tải).

**Q: Cách đảm bảo message không mất khi app crash?**
> (1) Queue và Exchange `durable=true` — tồn tại sau restart. (2) Message persistent — `deliveryMode=2`. (3) Publisher confirms — biết broker đã nhận. (4) Manual acknowledgment — commit sau khi xử lý xong. (5) Outbox pattern — đảm bảo at-least-once với DB transaction.
