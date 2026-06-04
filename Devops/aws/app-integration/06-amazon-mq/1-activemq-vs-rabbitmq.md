# ActiveMQ vs RabbitMQ — So Sánh Chi Tiết

> Hai engine được Amazon MQ hỗ trợ có triết lý thiết kế khác nhau rõ rệt. Hiểu được sự khác biệt giúp bạn chọn đúng công cụ cho từng bài toán.

## 📚 Mục Lục

1. [Tổng Quan Kiến Trúc](#tổng-quan-kiến-trúc)
2. [So Sánh Toàn Diện](#so-sánh-toàn-diện)
3. [ActiveMQ — Phân Tích Sâu](#activemq--phân-tích-sâu)
4. [RabbitMQ — Phân Tích Sâu](#rabbitmq--phân-tích-sâu)
5. [Giao Thức và Protocol](#giao-thức-và-protocol)
6. [Routing Models (Mô Hình Định Tuyến)](#routing-models-mô-hình-định-tuyến)
7. [High Availability](#high-availability)
8. [Khi Nào Chọn Cái Nào](#khi-nào-chọn-cái-nào)
9. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
10. [Checklist Phỏng Vấn](#checklist-phỏng-vấn)

---

## Tổng Quan Kiến Trúc

### Apache ActiveMQ

```
┌──────────────────────────────────────────────────────────────┐
│                     Apache ActiveMQ                          │
│                                                              │
│  ┌──────────────┐        ┌──────────────────────────────┐   │
│  │  Producers   │───────▶│       Broker Core            │   │
│  └──────────────┘        │  ┌──────────┐ ┌──────────┐   │   │
│                           │  │ Queue    │ │  Topic   │   │   │
│  ┌──────────────┐        │  │(Hàng đợi)│ │(Chủ đề)  │   │   │
│  │  Consumers   │◀───────│  └──────────┘ └──────────┘   │   │
│  └──────────────┘        │                              │   │
│                           │  Virtual Destinations        │   │
│                           │  (Đích ảo: Composite,        │   │
│                           │   Virtual Topics, etc.)      │   │
│                           └──────────────────────────────┘   │
│                                                              │
│  Model: JMS (Java Message Service — Dịch Vụ Tin Nhắn Java)  │
└──────────────────────────────────────────────────────────────┘
```

### RabbitMQ

```
┌──────────────────────────────────────────────────────────────┐
│                        RabbitMQ                              │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │  Producers   │───▶│   Exchange   │───▶│    Queue     │   │
│  └──────────────┘    │  (Bộ Trao    │    │  (Hàng Đợi)  │   │
│                       │   Đổi)       │    └──────┬───────┘   │
│                       │              │           │           │
│                       │ Binding Keys │           ▼           │
│                       │ (Khóa Ràng   │    ┌──────────────┐   │
│                       │  Buộc)       │    │  Consumers   │   │
│                       └──────────────┘    └──────────────┘   │
│                                                              │
│  Model: AMQP — Producer → Exchange → Binding → Queue        │
└──────────────────────────────────────────────────────────────┘
```

**Sự khác biệt cốt lõi**: ActiveMQ broker trực tiếp quản lý destinations (đích — queue hoặc topic). RabbitMQ thêm tầng **Exchange** (Bộ Trao Đổi) ở giữa, cho phép định tuyến linh hoạt hơn nhiều.

---

## So Sánh Toàn Diện

| Tiêu Chí | Apache ActiveMQ | RabbitMQ |
|---|---|---|
| **Ngôn ngữ triển khai** | Java | Erlang |
| **Triết lý** | JMS-first, nhiều giao thức | AMQP-first, Exchange-based routing |
| **Message Model** (Mô Hình Tin Nhắn) | Destination-based (queue/topic) | Exchange → Binding → Queue |
| **Giao thức chính** | OpenWire, AMQP, MQTT, STOMP | AMQP 0-9-1, MQTT, STOMP |
| **Routing** (Định Tuyến) | Đơn giản | Rất linh hoạt (4 loại exchange) |
| **Plugin Ecosystem** | Hạn chế | Phong phú (Management UI, Shovel, Federation...) |
| **Message Priority** (Ưu Tiên Tin Nhắn) | Hỗ trợ (0–9) | Hỗ trợ (0–255) |
| **Message TTL** (Thời Gian Sống) | Hỗ trợ | Hỗ trợ |
| **Dead Letter Queue — DLQ** | Dead Letter Queue | Dead Letter Exchange — DLX |
| **Transaction** (Giao Dịch) | XA Transaction, JMS Transaction | Publisher Confirms, Consumer Acks |
| **Persistence** (Lưu Trữ Bền Vững) | KahaDB, JDBC | Durable Queues (hàng đợi bền vững) |
| **Clustering** (Phân Cụm) | Phức tạp, ít phổ biến | Classic Mirror / Quorum Queues |
| **Performance tổng thể** | Khá tốt, nhiều tính năng | Cao hơn với Quorum Queues |
| **Management UI** | Web console cơ bản | RabbitMQ Management Plugin — rất tốt |
| **Cloud-native** | Kém hơn | Tốt hơn |
| **Cộng đồng** | Đang chậm lại | Đang phát triển mạnh |
| **Hỗ trợ Amazon MQ** | Đầy đủ | Đầy đủ |

---

## ActiveMQ — Phân Tích Sâu

### Điểm Mạnh

**1. Hỗ Trợ Giao Thức Rộng Nhất**

ActiveMQ hỗ trợ tất cả giao thức quan trọng trên cùng một broker:

```
Port 61616  → OpenWire (giao thức nhị phân riêng, tốt nhất cho Java clients)
Port 5672   → AMQP 1.0
Port 1883   → MQTT (dành cho IoT devices)
Port 61613  → STOMP (giao thức text-based, đơn giản)
Port 61614  → WebSocket/STOMP
```

**2. JMS (Java Message Service — Dịch Vụ Tin Nhắn Java) Compliance**

JMS là API chuẩn Java cho messaging. ActiveMQ là JMS provider đầy đủ nhất:

```java
// Code JMS chuẩn — chạy được với ActiveMQ không cần thay đổi
ConnectionFactory factory = new ActiveMQConnectionFactory(brokerUrl);
Connection connection = factory.createConnection();
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

// Queue — Point-to-Point (Điểm-Tới-Điểm)
Queue queue = session.createQueue("ORDER_QUEUE");
MessageProducer producer = session.createProducer(queue);

TextMessage message = session.createTextMessage("Order #12345");
producer.send(message);
```

**3. Virtual Destinations (Đích Ảo)**

ActiveMQ cho phép fan-out qua **Virtual Topics** (Chủ Đề Ảo) — tính năng độc đáo:

```
Virtual Topic: VirtualTopic.ORDERS
  → Tự động fan-out tới:
    Consumer.SERVICE_A.VirtualTopic.ORDERS (queue cho Service A)
    Consumer.SERVICE_B.VirtualTopic.ORDERS (queue cho Service B)
```

Khác với SNS fan-out, đây là fan-out **có load balancing** — nhiều instance của Service A chia sẻ nhau.

**4. XA Transaction (Giao Dịch XA)**

XA là tiêu chuẩn cho **distributed transactions** (giao dịch phân tán) — atomically (nguyên tử) commit/rollback database + messaging cùng lúc:

```java
// XA Transaction — đảm bảo tính nhất quán DB + Message
XAConnectionFactory xaFactory = new ActiveMQXAConnectionFactory(brokerUrl);
XAConnection xaConn = xaFactory.createXAConnection();
XASession xaSession = xaConn.createXASession();
XAResource xaResource = xaSession.getXAResource();

// Tham gia vào distributed transaction coordinator (điều phối viên giao dịch phân tán)
// Nếu DB commit thất bại → message cũng bị rollback
```

**5. Message Selector (Bộ Chọn Lọc Tin Nhắn)**

Consumer có thể lọc message bằng SQL-like syntax (cú pháp giống SQL):

```java
// Chỉ nhận order có priority > 5 và region = 'VN'
String selector = "priority > 5 AND region = 'VN'";
MessageConsumer consumer = session.createConsumer(queue, selector);
```

### Hạn Chế

- **Nặng về Java** — không phải cloud-native
- **Clustering phức tạp** — không scale tốt theo chiều ngang
- **Plugin ecosystem hạn chế**
- **Cộng đồng đang chậm lại** so với RabbitMQ và Kafka

---

## RabbitMQ — Phân Tích Sâu

### Exchange Types (Loại Bộ Trao Đổi) — Trái Tim Của RabbitMQ

RabbitMQ có **4 loại Exchange**, mỗi loại có cách định tuyến khác nhau:

#### 1. Direct Exchange (Trao Đổi Trực Tiếp)

```
Producer ──[routing_key: "payment"]──▶ Exchange ──▶ Queue "payment-queue"
                                                ├──▶ Queue "audit-queue"
                                                     (nếu có binding key "payment")
```

- Định tuyến theo **routing key** (khóa định tuyến) khớp chính xác
- Dùng cho: task queues, point-to-point messaging

#### 2. Fanout Exchange (Trao Đổi Khuếch Tán)

```
Producer ──[bất kỳ key]──▶ Exchange ──▶ Queue A (tất cả đều nhận)
                                    ├──▶ Queue B
                                    └──▶ Queue C
```

- Bỏ qua routing key, gửi đến **tất cả** queue đã bind
- Dùng cho: broadcast notifications (thông báo phát sóng), logging

#### 3. Topic Exchange (Trao Đổi Chủ Đề)

```
Routing key pattern:
  "order.vn.payment"  ──▶ "order.*.payment" (khớp)
  "order.vn.payment"  ──▶ "order.#"         (khớp — # = nhiều từ)
  "log.error.db"      ──▶ "log.error.*"     (khớp — * = một từ)

Producer ──[order.vn.payment]──▶ Exchange
                               ├──▶ Queue A (binding: "order.*.payment") ✅
                               ├──▶ Queue B (binding: "order.#")         ✅
                               └──▶ Queue C (binding: "log.#")           ❌
```

- Routing theo **pattern với wildcard** (ký tự đại diện):
  - `*` = đúng một từ
  - `#` = không hoặc nhiều từ
- Dùng cho: phân loại sự kiện phức tạp, multi-tenant routing

#### 4. Headers Exchange (Trao Đổi Tiêu Đề)

```
Producer gửi message với headers:
  { "format": "pdf", "type": "report" }

Exchange ──▶ Queue A (binding: format=pdf AND type=report) ✅
         ├──▶ Queue B (binding: format=pdf OR type=log)    ✅ (chỉ nếu x-match=any)
         └──▶ Queue C (binding: format=csv)                ❌
```

- Định tuyến theo **message headers** (tiêu đề tin nhắn), không dùng routing key
- Dùng cho: routing theo metadata phức tạp

### Quorum Queues (Hàng Đợi Quorum) — RabbitMQ 3.8+

Quorum Queues là tính năng HA hiện đại của RabbitMQ, dùng **Raft consensus algorithm** (thuật toán đồng thuận Raft):

```
┌─────────────────────────────────────────────────────────────┐
│                   Quorum Queue Cluster                      │
│                                                             │
│  Node 1 (Leader)   Node 2 (Follower)   Node 3 (Follower)   │
│  ┌─────────────┐   ┌─────────────┐    ┌─────────────┐      │
│  │  Queue      │──▶│  Replica    │    │  Replica    │      │
│  │  (Leader)   │   │  (Follower) │    │  (Follower) │      │
│  └─────────────┘   └─────────────┘    └─────────────┘      │
│                                                             │
│  Ghi được xác nhận khi majority (đa số) nodes đồng ý       │
│  Nếu 1 node chết → 2 nodes còn lại vẫn hoạt động           │
└─────────────────────────────────────────────────────────────┘
```

**Ưu điểm so với Classic Mirror Queues** (Hàng Đợi Gương Cổ Điển):
- Không mất dữ liệu khi leader crash
- Tự động bầu leader mới
- Throughput cao hơn

### Dead Letter Exchange — DLX (Bộ Trao Đổi Thư Chết)

```
Normal Queue ──[message rejected/expired]──▶ Dead Letter Exchange
                                                     │
                                                     ▼
                                            Dead Letter Queue
                                          (Hàng Đợi Thư Chết)
```

Cấu hình DLX khi tạo queue:

```python
import pika

channel.queue_declare(
    queue='order-queue',
    arguments={
        'x-dead-letter-exchange': 'dlx-exchange',    # Exchange nhận dead letters
        'x-dead-letter-routing-key': 'dead-orders',  # Routing key cho DLX
        'x-message-ttl': 30000,                      # TTL 30 giây (ms)
        'x-max-length': 10000                        # Tối đa 10,000 messages
    }
)
```

---

## Giao Thức và Protocol

### ActiveMQ — Đa Giao Thức

```
┌──────────────────────────────────────────────────────────────┐
│                     ActiveMQ Broker                          │
│                                                              │
│   Java Client ──[OpenWire:61616]──▶ ┌──────────────────┐    │
│   IoT Device  ──[MQTT:1883]──────▶  │   Broker Core    │    │
│   Python App  ──[STOMP:61613]─────▶ │                  │    │
│   .NET Client ──[AMQP:5672]───────▶ └──────────────────┘    │
│   Browser     ──[WS:61614]────────▶                          │
└──────────────────────────────────────────────────────────────┘
```

### RabbitMQ — AMQP với Extension

```
┌──────────────────────────────────────────────────────────────┐
│                     RabbitMQ Broker                          │
│                                                              │
│   Any Lang    ──[AMQP 0-9-1:5672]──▶ ┌──────────────────┐   │
│   IoT Device  ──[MQTT:1883]──────▶   │   Broker Core    │   │
│   Python App  ──[STOMP:61613]─────▶  │                  │   │
│   HTTP Client ──[Management API]───▶ └──────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## High Availability

### Amazon MQ — ActiveMQ HA

```
┌──────────────────────────────────────────────────────────────┐
│              Active/Standby Pair                             │
│                                                              │
│  AZ-a                          AZ-b                         │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │  ACTIVE Broker   │          │  STANDBY Broker  │         │
│  │  (đang phục vụ)  │◀────────▶│  (chờ)           │         │
│  └────────┬─────────┘          └────────┬─────────┘         │
│           │                             │                   │
│           └────────────┬────────────────┘                   │
│                        │                                    │
│               ┌────────▼───────┐                            │
│               │   Amazon EFS   │                            │
│               │ (Shared data)  │                            │
│               └────────────────┘                            │
│                                                              │
│  Failover time: ~30 giây                                     │
│  RPO (Recovery Point Objective): 0 (không mất data)         │
│  RTO (Recovery Time Objective): ~30 giây                    │
└──────────────────────────────────────────────────────────────┘
```

### Amazon MQ — RabbitMQ Cluster

```
┌──────────────────────────────────────────────────────────────┐
│              RabbitMQ 3-Node Cluster                         │
│                                                              │
│  AZ-a          AZ-b          AZ-c                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │  Node 1  │──│  Node 2  │──│  Node 3  │                  │
│  │(Quorum   │  │(Quorum   │  │(Quorum   │                  │
│  │ Leader)  │  │ Follower)│  │ Follower)│                  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │
│       └─────────────┼─────────────┘                         │
│                     │                                       │
│              ┌──────▼──────┐                                │
│              │     NLB     │ ← Clients kết nối qua đây      │
│              │(Network Load│                                │
│              │  Balancer)  │                                │
│              └─────────────┘                                │
│                                                              │
│  Failover: Tức thì (Raft election)                          │
│  Min nodes để hoạt động: 2/3                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Khi Nào Chọn Cái Nào

### Chọn ActiveMQ Khi

```
✅ Ứng dụng Java dùng JMS API chuẩn
✅ Cần hỗ trợ nhiều giao thức trên cùng một broker (OpenWire + MQTT + STOMP)
✅ Cần XA Transaction (distributed transaction với database)
✅ Đang migration từ ActiveMQ on-premises
✅ Cần Virtual Topics (fan-out + load balancing kết hợp)
✅ Hệ thống IoT cần MQTT + Queue trong cùng broker
✅ Cần Message Selector với SQL-like syntax
```

### Chọn RabbitMQ Khi

```
✅ Cần routing linh hoạt (topic pattern, header-based)
✅ Đang migration từ RabbitMQ on-premises
✅ Cần Management UI tốt và plugin phong phú
✅ Muốn Quorum Queues cho HA mạnh mẽ hơn
✅ Hệ thống đa ngôn ngữ không gắn chặt với Java
✅ Cần xây dựng complex event routing pipeline
✅ Yêu cầu cộng đồng active và tài liệu phong phú
```

### Không Chọn Amazon MQ Khi

```
❌ Xây dựng ứng dụng mới trên AWS → Dùng SQS/SNS/EventBridge
❌ Cần scale tự động không giới hạn → Dùng SQS
❌ Serverless architecture → Dùng SQS + Lambda
❌ Cần event streaming thời gian thực → Dùng Kinesis
❌ Throughput cực cao (hàng triệu msg/giây) → Xem xét MSK (Managed Kafka)
```

---

## Ví Dụ Thực Tế

### Ví Dụ 1: Hệ Thống Thông Báo Đa Kênh Với RabbitMQ

```
┌─────────────────────────────────────────────────────────────┐
│           Notification System (Hệ Thống Thông Báo)         │
│                                                             │
│  Order Service  ──[notification.order.created]──▶ Topic    │
│                                                   Exchange  │
│                                                      │      │
│                          ┌───────────────────────────┤      │
│                          │           │               │      │
│                   [*.order.*]   [*.*.created]  [#.urgent.#]│
│                          │           │               │      │
│                          ▼           ▼               ▼      │
│                   Email Queue   Push Queue    SMS Queue     │
│                          │           │               │      │
│                   Email Worker  Push Worker   SMS Worker    │
└─────────────────────────────────────────────────────────────┘
```

```python
import pika

# Kết nối Amazon MQ RabbitMQ
params = pika.ConnectionParameters(
    host='your-broker.mq.region.amazonaws.com',
    port=5671,
    credentials=pika.PlainCredentials('user', 'pass'),
    ssl_options=pika.SSLOptions(context)
)

connection = pika.BlockingConnection(params)
channel = connection.channel()

# Khai báo Topic Exchange
channel.exchange_declare(exchange='notifications', exchange_type='topic', durable=True)

# Khai báo Queue và Binding
channel.queue_declare(queue='email-queue', durable=True)
channel.queue_bind(
    exchange='notifications',
    queue='email-queue',
    routing_key='*.order.*'    # Nhận tất cả event liên quan đến order
)

channel.queue_declare(queue='sms-urgent', durable=True)
channel.queue_bind(
    exchange='notifications',
    queue='sms-urgent',
    routing_key='#.urgent.#'   # Nhận mọi event có "urgent" trong routing key
)

# Gửi thông báo
channel.basic_publish(
    exchange='notifications',
    routing_key='customer.order.created',
    body='{"order_id": "12345", "amount": 500000}',
    properties=pika.BasicProperties(
        delivery_mode=2,  # persistent (bền vững)
        content_type='application/json'
    )
)
```

### Ví Dụ 2: Order Processing Với ActiveMQ

```java
import org.apache.activemq.ActiveMQConnectionFactory;
import javax.jms.*;

public class OrderProcessor {

    private static final String BROKER_URL =
        "ssl://your-broker.mq.region.amazonaws.com:61617";

    public void sendOrder(String orderId, int priority) throws JMSException {
        ConnectionFactory factory = new ActiveMQConnectionFactory(
            "user", "pass", BROKER_URL
        );

        try (Connection conn = factory.createConnection();) {
            conn.start();
            Session session = conn.createSession(false, Session.CLIENT_ACKNOWLEDGE);

            // Queue với priority support
            Queue queue = session.createQueue("ORDER_PROCESSING");
            MessageProducer producer = session.createProducer(queue);

            TextMessage msg = session.createTextMessage("{\"orderId\":\"" + orderId + "\"}");
            msg.setIntProperty("priority", priority);
            msg.setStringProperty("region", "VN");  // Dùng cho Message Selector

            // Gửi với message priority (độ ưu tiên)
            producer.send(msg, DeliveryMode.PERSISTENT, priority, 0);
        }
    }

    public void consumeHighPriorityOrders() throws JMSException {
        ConnectionFactory factory = new ActiveMQConnectionFactory(
            "user", "pass", BROKER_URL
        );

        try (Connection conn = factory.createConnection();) {
            conn.start();
            Session session = conn.createSession(false, Session.CLIENT_ACKNOWLEDGE);
            Queue queue = session.createQueue("ORDER_PROCESSING");

            // Message Selector: Chỉ nhận order priority > 7 từ VN
            String selector = "priority > 7 AND region = 'VN'";
            MessageConsumer consumer = session.createConsumer(queue, selector);

            consumer.setMessageListener(msg -> {
                try {
                    System.out.println("Xử lý đơn hàng ưu tiên cao: " + ((TextMessage)msg).getText());
                    msg.acknowledge();  // Xác nhận đã xử lý
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            });
        }
    }
}
```

---

## Checklist Phỏng Vấn

### Câu Hỏi Thường Gặp

**Q: Sự khác biệt chính giữa ActiveMQ và RabbitMQ?**

> ActiveMQ tập trung vào JMS compliance (tuân thủ JMS) và hỗ trợ nhiều giao thức, lý tưởng cho Java enterprise applications và IoT. RabbitMQ có Exchange-based routing linh hoạt hơn, Management UI tốt hơn, và Quorum Queues hiện đại hơn cho HA.

**Q: Khi nào dùng Topic Exchange (Trao Đổi Chủ Đề) trong RabbitMQ?**

> Khi bạn cần định tuyến message dựa trên pattern (ví dụ: "nhận tất cả event liên quan đến orders") mà routing key thay đổi theo runtime. Direct Exchange không đủ linh hoạt, Fanout Exchange thì không lọc được.

**Q: XA Transaction trong ActiveMQ giải quyết vấn đề gì?**

> XA Transaction (giao dịch XA) cho phép atomically commit (cam kết nguyên tử) database operation và message operation. Nếu DB commit thành công nhưng message gửi thất bại — hoặc ngược lại — cả hai sẽ bị rollback. Quan trọng để đảm bảo tính nhất quán trong distributed systems.

**Q: Quorum Queue khác Classic Mirror Queue như thế nào?**

> Quorum Queue dùng Raft consensus algorithm — đảm bảo không mất dữ liệu khi leader fail và tự động bầu leader mới nhanh hơn. Classic Mirror Queue cũ hơn, có nguy cơ mất data trong một số edge case, không còn được khuyến nghị cho production mới.

### Kiến Thức Cần Nắm

- [ ] 4 loại Exchange trong RabbitMQ và use case của mỗi loại
- [ ] JMS API cơ bản: ConnectionFactory, Session, MessageProducer, MessageConsumer
- [ ] Message Selector trong ActiveMQ — SQL-like syntax
- [ ] Virtual Topics trong ActiveMQ — fan-out + load balancing
- [ ] Dead Letter Exchange (RabbitMQ) vs Dead Letter Queue (ActiveMQ)
- [ ] Quorum Queue vs Classic Mirror Queue
- [ ] XA Transaction — khi nào cần, trade-off là gì
- [ ] Active/Standby HA (ActiveMQ) vs Cluster HA (RabbitMQ)

---

## 🔗 Điều Hướng

| Bước Trước | Bước Tiếp Theo |
|---|---|
| [README.md](./README.md) — Tổng quan Amazon MQ | [2-migration-guide.md](./2-migration-guide.md) — Di Chuyển Lên Amazon MQ |

---

**Cập Nhật Lần Cuối:** 2026-05-18
