# Apache Kafka Integration với Spring Boot

> Tích hợp Apache Kafka (Nền Tảng Streaming Sự Kiện Phân Tán) vào Spring Boot —
> từ cấu hình cơ bản, producer với `KafkaTemplate` (Template Kafka), consumer với `@KafkaListener`,
> quản lý Consumer Groups (Nhóm Consumer), partitions (phân vùng), đến xử lý lỗi và đảm bảo delivery.

---

## 1. Kafka Là Gì và Khi Nào Dùng?

### Kiến Trúc Kafka

```
Producer (Nhà Sản Xuất)          Kafka Cluster (Cụm Kafka)         Consumer (Người Tiêu Thụ)
                                ┌─────────────────────────┐
                                │  Topic: order-events    │
┌──────────────┐  publish       │  ┌─────────────────┐   │   subscribe  ┌──────────────────┐
│ OrderService │──────────────► │  │  Partition 0    │   │ ────────────► │InventoryService  │
└──────────────┘                │  │  [msg0][msg1].. │   │              │ (Consumer Group A)│
                                │  ├─────────────────┤   │              └──────────────────┘
┌──────────────┐                │  │  Partition 1    │   │
│ PaymentSvc   │──────────────► │  │  [msg0][msg1].. │   │   subscribe  ┌──────────────────┐
└──────────────┘                │  ├─────────────────┤   │ ────────────► │ NotificationSvc  │
                                │  │  Partition 2    │   │              │ (Consumer Group B)│
                                │  │  [msg0][msg1].. │   │              └──────────────────┘
                                └─────────────────────────┘
```

### Khái Niệm Cốt Lõi

| Khái Niệm | Giải Thích |
|-----------|------------|
| **Topic** (Chủ Đề) | Kênh message, giống "category" |
| **Partition** (Phân Vùng) | Chia nhỏ topic để scale — đơn vị song song |
| **Offset** (Vị Trí) | Số thứ tự message trong partition — consumer theo dõi đã đọc đến đâu |
| **Consumer Group** (Nhóm Consumer) | Nhiều consumer chia sẻ tải — mỗi partition chỉ do 1 consumer trong group đọc |
| **Broker** (Máy Chủ Kafka) | Server Kafka — lưu và phục vụ messages |
| **Replication Factor** (Hệ Số Nhân Bản) | Số bản sao partition — đảm bảo fault tolerance (chịu lỗi) |
| **Retention** (Lưu Giữ) | Thời gian/dung lượng Kafka giữ message trước khi xóa |

### Kafka vs RabbitMQ — Chọn Cái Nào?

```
Dùng Kafka khi:
├── Cần throughput (thông lượng) cực cao (triệu msg/giây)
├── Cần replay (phát lại) message
├── Event sourcing / audit log
├── Nhiều consumer group độc lập đọc cùng topic
└── Streaming analytics, data pipeline

Dùng RabbitMQ khi:
├── Task queue — phân phối công việc cho workers
├── Cần routing phức tạp (dead letter, priority queue)
├── RPC (Remote Procedure Call — Gọi Thủ Tục Từ Xa) pattern
├── Cần per-message TTL (Time-To-Live — Thời Gian Sống)
└── Setup đơn giản, throughput vừa phải
```

---

## 2. Cấu Hình (Configuration)

### 2.1 Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <!-- Phiên bản quản lý bởi Spring Boot BOM -->
</dependency>
```

### 2.2 application.properties — Cấu Hình Cơ Bản

```properties
# ─── Kafka Broker Connection ───────────────────────────────────────────────
spring.kafka.bootstrap-servers=localhost:9092

# ─── Producer (Nhà Sản Xuất) ───────────────────────────────────────────────
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

# Acknowledgment (Xác Nhận):
# acks=0 → không chờ xác nhận (nhanh nhất, mất mát cao nhất)
# acks=1 → chờ leader broker xác nhận
# acks=all → chờ tất cả replicas xác nhận (chậm nhất, an toàn nhất)
spring.kafka.producer.acks=all

# Retry (Thử Lại) — số lần thử lại khi gửi thất bại
spring.kafka.producer.retries=3

# Batch (Lô) — gom nhiều messages vào 1 batch để gửi — tăng throughput
spring.kafka.producer.batch-size=16384
spring.kafka.producer.linger-ms=5

# ─── Consumer (Người Tiêu Thụ) ─────────────────────────────────────────────
spring.kafka.consumer.group-id=order-service-group
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer

# Từ đâu bắt đầu đọc khi không có offset đã lưu
# earliest → từ đầu topic (replay tất cả messages)
# latest   → chỉ đọc messages mới sau khi consumer start
spring.kafka.consumer.auto-offset-reset=earliest

# Tắt auto-commit — quản lý offset thủ công để đảm bảo at-least-once
spring.kafka.consumer.enable-auto-commit=false

# Trusted packages cho JsonDeserializer — tránh lỗi security
spring.kafka.consumer.properties.spring.json.trusted.packages=com.example.events
```

### 2.3 Java Config — Cấu Hình Chi Tiết

```java
@Configuration
@EnableKafka  // ← Kích hoạt @KafkaListener
public class KafkaConfig {

    // ─── Producer Config ──────────────────────────────────────────────────
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // Idempotent producer
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    // ─── Consumer Config ──────────────────────────────────────────────────
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service-group");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.events");
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());

        // Manual acknowledgment — commit offset sau khi xử lý xong
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        // Số concurrent consumers — tối đa bằng số partitions
        factory.setConcurrency(3);

        return factory;
    }
}
```

---

## 3. KafkaTemplate — Gửi Message

### 3.1 Gửi Cơ Bản

```java
@Service
public class OrderEventProducer {

    private static final String TOPIC = "order-events";
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // Gửi đơn giản — Kafka tự chọn partition
    public void publishOrderCreated(OrderCreatedEvent event) {
        kafkaTemplate.send(TOPIC, event);
    }

    // Gửi với key — cùng key → cùng partition → đảm bảo ordering
    public void publishWithKey(String orderId, OrderCreatedEvent event) {
        kafkaTemplate.send(TOPIC, orderId, event);
    }

    // Gửi với partition cụ thể
    public void publishToPartition(OrderCreatedEvent event, int partition) {
        kafkaTemplate.send(TOPIC, partition, null, event);
    }
}
```

### 3.2 Xử Lý Kết Quả Gửi (Async Callback)

```java
@Service
public class OrderEventProducer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventProducer.class);

    public void publishWithCallback(String orderId, OrderCreatedEvent event) {
        CompletableFuture<SendResult<String, Object>> future =
                kafkaTemplate.send("order-events", orderId, event);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to send event: orderId={}, error={}",
                        orderId, ex.getMessage(), ex);
                // Fallback: lưu vào outbox table, retry later
            } else {
                RecordMetadata metadata = result.getRecordMetadata();
                log.info("Event sent: topic={}, partition={}, offset={}",
                        metadata.topic(), metadata.partition(), metadata.offset());
            }
        });
    }
}
```

### 3.3 Transactional Producer (Producer Có Giao Dịch)

```java
// application.properties
// spring.kafka.producer.transaction-id-prefix=tx-order-

@Service
public class TransactionalOrderService {

    @Transactional("kafkaTransactionManager") // Kafka transaction
    public void processAndPublish(Order order) {
        // Gửi nhiều messages trong 1 transaction Kafka
        kafkaTemplate.send("order-events", new OrderCreatedEvent(order.getId()));
        kafkaTemplate.send("inventory-events", new StockDeductedEvent(order.getProductId()));
        // Cả hai messages commit hoặc rollback cùng nhau
    }
}
```

---

## 4. @KafkaListener — Nhận Message

### 4.1 Cơ Bản

```java
@Component
public class OrderEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-group"
    )
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Received order event: {}", event);
        inventoryService.deductStock(event.productId(), event.quantity());
    }
}
```

### 4.2 Với Manual Acknowledgment (Xác Nhận Thủ Công)

```java
@Component
public class ReliableOrderConsumer {

    @KafkaListener(topics = "order-events", groupId = "notification-group")
    public void handleOrder(
            @Payload OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        log.info("Processing message: topic={}, partition={}, offset={}", topic, partition, offset);

        try {
            emailService.sendConfirmation(event.customerEmail());
            ack.acknowledge(); // ← Commit offset chỉ khi xử lý thành công
        } catch (Exception ex) {
            log.error("Failed to process message at offset {}", offset, ex);
            // Không ack → Kafka sẽ redelivery khi consumer restart
            // Hoặc xử lý với error handler
        }
    }
}
```

### 4.3 Batch Consumer (Consumer Theo Lô)

```java
@Component
public class BatchOrderConsumer {

    @KafkaListener(topics = "order-events", groupId = "analytics-group",
                   containerFactory = "batchKafkaListenerContainerFactory")
    public void handleBatch(List<OrderCreatedEvent> events,
                            Acknowledgment ack) {
        log.info("Processing batch of {} events", events.size());

        // Xử lý tất cả cùng lúc — hiệu quả hơn từng message
        analyticsService.processBatch(events);

        ack.acknowledge();
    }
}

// Config cho batch consumer
@Bean("batchKafkaListenerContainerFactory")
public ConcurrentKafkaListenerContainerFactory<String, Object> batchFactory() {
    ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setBatchListener(true); // ← Bật batch mode
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
    return factory;
}
```

### 4.4 Multiple Topics & Topic Patterns

```java
@Component
public class MultiTopicConsumer {

    // Nhiều topics
    @KafkaListener(topics = {"order-events", "payment-events"}, groupId = "audit-group")
    public void handleMultipleTopics(ConsumerRecord<String, Object> record) {
        log.info("Topic: {}, Key: {}, Value: {}", record.topic(), record.key(), record.value());
    }

    // Topic pattern (Regex) — nhận tất cả topics khớp pattern
    @KafkaListener(topicPattern = ".*-events", groupId = "logger-group")
    public void handleAllEventTopics(ConsumerRecord<String, Object> record) {
        auditService.log(record.topic(), record.value());
    }
}
```

---

## 5. Consumer Groups & Scaling (Mở Rộng)

### Cách Consumer Group Hoạt Động

```
Topic: order-events (6 partitions)
  Partition 0 ──► Consumer A (Group: inventory-group)
  Partition 1 ──► Consumer A
  Partition 2 ──► Consumer B (Group: inventory-group)
  Partition 3 ──► Consumer B
  Partition 4 ──► Consumer C (Group: inventory-group)
  Partition 5 ──► Consumer C

Rule: 1 partition → 1 consumer trong 1 group tại 1 thời điểm
→ Thêm consumer 4 → consumer 4 nhàn rỗi (idle) vì hết partition
→ Muốn scale thêm → tăng partition trước
```

### Cấu Hình Concurrency (Song Song)

```java
@KafkaListener(
    topics = "order-events",
    groupId = "inventory-group",
    concurrency = "3"   // ← Tạo 3 consumer threads — mỗi thread xử lý 1 partition
)
public void handleOrder(OrderCreatedEvent event) { ... }

// Hoặc trong container factory
factory.setConcurrency(3);
```

---

## 6. Error Handling (Xử Lý Lỗi)

### 6.1 DefaultErrorHandler (Spring Kafka 2.8+)

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());

    // Retry với backoff: thử lại 3 lần với 1s, 2s, 4s delay
    DefaultErrorHandler errorHandler = new DefaultErrorHandler(
        new DeadLetterPublishingRecoverer(kafkaTemplate), // Gửi sang Dead Letter Topic
        new FixedBackOff(1000L, 3)  // interval=1s, maxAttempts=3
    );

    // Không retry với các lỗi không phục hồi được
    errorHandler.addNotRetryableExceptions(
        DeserializationException.class,
        MessageConversionException.class
    );

    factory.setCommonErrorHandler(errorHandler);
    return factory;
}
```

### 6.2 Dead Letter Topic (Chủ Đề Thư Chết)

```java
@Bean
public DeadLetterPublishingRecoverer deadLetterPublishingRecoverer() {
    return new DeadLetterPublishingRecoverer(
        kafkaTemplate,
        // Gửi sang topic "{original-topic}.DLT"
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
    );
}

// Consumer để xử lý DLT (Dead Letter Topic — Chủ Đề Thư Chết)
@Component
public class DeadLetterConsumer {

    @KafkaListener(topics = "order-events.DLT", groupId = "dlt-handler-group")
    public void handleDeadLetter(ConsumerRecord<String, Object> record,
                                 @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exMessage) {
        log.error("Dead letter message: topic={}, key={}, error={}",
                record.topic(), record.key(), exMessage);

        // Lưu vào DB để manual investigation
        deadLetterRepository.save(new DeadLetterRecord(record, exMessage));

        // Alert Slack/PagerDuty
        alertService.sendAlert("Dead letter in " + record.topic(), exMessage);
    }
}
```

### 6.3 Retry Topic Pattern (Kafka Retry Topics — Spring Kafka 2.7+)

```java
@RetryableTopic(
    attempts = "4",                                    // Thử 1 lần + 3 lần retry
    backoff = @Backoff(delay = 1000, multiplier = 2),  // Exponential backoff
    autoCreateTopics = "true",
    dltTopicSuffix = ".DLT"
)
@KafkaListener(topics = "order-events", groupId = "inventory-group")
public void handleOrder(OrderCreatedEvent event) {
    inventoryService.process(event); // Throw exception → tự động retry
}

@DltHandler  // Xử lý message sau khi hết lần retry
public void handleDlt(OrderCreatedEvent event,
                      @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    log.error("Order event exhausted retries: topic={}, event={}", topic, event);
    alertService.sendAlert("Order processing failed", event.toString());
}
```

---

## 7. Serialization (Tuần Tự Hóa)

### JSON Serialization với Type Header

```properties
# Producer
spring.kafka.producer.properties.spring.json.add.type.headers=true

# Consumer — trust specific packages
spring.kafka.consumer.properties.spring.json.trusted.packages=com.example.events
```

### Custom Deserializer — Multiple Event Types

```java
@Bean
public DefaultKafkaConsumerFactory<String, Object> consumerFactory() {
    JsonDeserializer<Object> deserializer = new JsonDeserializer<>();
    deserializer.addTrustedPackages("com.example.events");
    deserializer.setUseTypeHeaders(true); // Dùng type header để chọn class

    return new DefaultKafkaConsumerFactory<>(
        consumerProperties(),
        new StringDeserializer(),
        deserializer
    );
}
```

---

## 8. Monitoring & Observability (Giám Sát & Quan Sát)

### Metrics với Micrometer (Đồng Hồ Đo)

```properties
# Bật Kafka metrics
management.metrics.tags.application=${spring.application.name}
```

Key metrics cần theo dõi:
- `kafka.consumer.records-lag-max` — Lag (Độ Trễ) của consumer — bao nhiêu messages chưa đọc
- `kafka.producer.record-send-rate` — Tốc độ gửi messages
- `kafka.consumer.fetch-rate` — Tốc độ đọc messages

### Kafka Lag Alert (Cảnh Báo Lag)

```yaml
# Prometheus alert rule
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_records_lag_max > 1000
  for: 5m
  annotations:
    summary: "Consumer lag too high — processing falling behind"
```

---

## 9. Thực Hành Tốt Nhất (Best Practices)

### Producer

```java
// ✅ Luôn dùng key để đảm bảo ordering trong partition
kafkaTemplate.send(topic, orderId, event);

// ✅ Bật idempotent producer — ngăn duplicate messages
config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

// ✅ Handle send callback — phát hiện failure
kafkaTemplate.send(topic, event).whenComplete((result, ex) -> {
    if (ex != null) log.error("Send failed", ex);
});
```

### Consumer

```java
// ✅ Dùng manual acknowledgment
// ✅ Số concurrency ≤ số partitions
// ✅ Idempotent consumer — xử lý message trùng không gây side effect

// ✅ Ghi log với context đầy đủ
log.info("Processing: topic={}, partition={}, offset={}, key={}",
    record.topic(), record.partition(), record.offset(), record.key());
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Consumer Group hoạt động thế nào? Khi nào xảy ra rebalance?**
> Mỗi partition chỉ được đọc bởi 1 consumer trong group. Rebalance (Cân Bằng Lại) xảy ra khi: consumer join/leave group, partition thay đổi, consumer timeout (session.timeout.ms). Rebalance tạm ngừng consumption — cần minimize qua heartbeat tuning.

**Q: At-least-once vs exactly-once delivery trong Kafka?**
> **At-least-once** (Ít Nhất Một Lần): commit offset sau khi xử lý → crash trước commit → replay từ offset cũ → duplicate. **Exactly-once** (Đúng Một Lần): dùng Kafka transactions + idempotent consumer — phức tạp, cần `enable.idempotence=true` và `transactional.id`. Thực tế thường implement at-least-once + idempotent consumer.

**Q: Cách đảm bảo ordering (thứ tự) trong Kafka?**
> Ordering chỉ đảm bảo trong 1 partition. Để đảm bảo: (1) Gửi cùng key → cùng partition, (2) Giới hạn `max.in.flight.requests.per.connection=1` (không batch), (3) Bật idempotent producer để không bị reorder khi retry.

**Q: Kafka retention (lưu giữ) — messages bị xóa sau bao lâu?**
> Mặc định 7 ngày (`log.retention.hours=168`). Có thể cấu hình theo thời gian (`log.retention.ms`) hoặc dung lượng (`log.retention.bytes`). Consumer có thể replay messages miễn là chưa quá retention period — đây là điểm khác biệt lớn so với RabbitMQ.
