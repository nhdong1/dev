# Di Chuyển Từ On-Premises Sang Amazon MQ

> Hướng dẫn từng bước để lift-and-shift (nâng và di chuyển) message broker từ môi trường on-premises (tại chỗ) lên Amazon MQ với rủi ro tối thiểu và thời gian downtime ngắn nhất.

## 📚 Mục Lục

1. [Tại Sao Cần Migration?](#tại-sao-cần-migration)
2. [Chiến Lược Migration Tổng Quan](#chiến-lược-migration-tổng-quan)
3. [Giai Đoạn 1 — Đánh Giá Hiện Trạng](#giai-đoạn-1--đánh-giá-hiện-trạng)
4. [Giai Đoạn 2 — Lên Kế Hoạch](#giai-đoạn-2--lên-kế-hoạch)
5. [Giai Đoạn 3 — Chuẩn Bị Amazon MQ](#giai-đoạn-3--chuẩn-bị-amazon-mq)
6. [Giai Đoạn 4 — Migration Thực Tế](#giai-đoạn-4--migration-thực-tế)
7. [Giai Đoạn 5 — Cutover và Verification](#giai-đoạn-5--cutover-và-verification)
8. [Xử Lý Tình Huống Phức Tạp](#xử-lý-tình-huống-phức-tạp)
9. [Rollback Plan](#rollback-plan)
10. [Post-Migration Optimization](#post-migration-optimization)
11. [Checklist Phỏng Vấn](#checklist-phỏng-vấn)

---

## Tại Sao Cần Migration?

### Vấn Đề Của On-Premises Broker

```
On-Premises Message Broker Problems (Vấn Đề Của Broker Tại Chỗ):

┌─────────────────────────────────────────────────────────────┐
│  ❌ Operational Overhead (Gánh Nặng Vận Hành)               │
│     - Patch OS, JVM, broker software thường xuyên          │
│     - Cấu hình HA thủ công — phức tạp và dễ sai            │
│     - On-call 24/7 khi broker gặp sự cố                     │
│                                                             │
│  ❌ Scaling Thủ Công (Manual Scaling)                        │
│     - Mua hardware mới mất hàng tuần                        │
│     - Over-provisioning (quá mức cần thiết) để an toàn      │
│     - Khó scale đột ngột khi traffic tăng                   │
│                                                             │
│  ❌ Disaster Recovery Phức Tạp (Phục Hồi Thảm Họa)          │
│     - DR site tốn chi phí vận hành thứ hai                  │
│     - Test failover phức tạp, rủi ro                        │
│     - Backup và restore thủ công                            │
│                                                             │
│  ❌ Chi Phí Ẩn (Hidden Costs)                                │
│     - Phần cứng: depreciation (khấu hao)                    │
│     - Điện, làm mát, data center                            │
│     - Nhân lực vận hành chuyên biệt                         │
└─────────────────────────────────────────────────────────────┘
```

### Lợi Ích Khi Chuyển Lên Amazon MQ

```
Amazon MQ Benefits (Lợi Ích):

✅ Managed service (dịch vụ được quản lý) — AWS xử lý patching, HA
✅ Không thay đổi code — tương thích giao thức 100%
✅ Tích hợp native với AWS services (IAM, CloudWatch, VPC)
✅ Backup tự động hàng ngày
✅ Multi-AZ HA built-in
✅ Giảm operational burden cho engineering team
```

---

## Chiến Lược Migration Tổng Quan

### Ba Chiến Lược Chính

```
┌─────────────────────────────────────────────────────────────┐
│              Migration Strategies (Chiến Lược)              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Big Bang Migration (Di Chuyển Toàn Bộ Một Lúc)         │
│     - Dừng hệ thống, di chuyển tất cả, khởi động lại       │
│     - Rủi ro cao, downtime dài                              │
│     - Chỉ phù hợp: hệ thống nhỏ, ít traffic                │
│                                                             │
│  2. Blue/Green Migration (Di Chuyển Xanh/Xanh Lá)          │
│     - Chạy song song on-premises (blue) + Amazon MQ (green) │
│     - Chuyển traffic dần dần qua green                      │
│     - An toàn hơn, rollback dễ dàng                        │
│     - Được khuyến nghị cho hầu hết trường hợp              │
│                                                             │
│  3. Strangler Fig Pattern (Mẫu Bóp Nghẹt)                  │
│     - Di chuyển từng queue/topic một                        │
│     - Service by service migration                          │
│     - Ít rủi ro nhất, nhưng mất nhiều thời gian nhất       │
└─────────────────────────────────────────────────────────────┘
```

### Khuyến Nghị Theo Quy Mô

| Quy Mô Hệ Thống | Chiến Lược Khuyến Nghị | Thời Gian Dự Kiến |
|---|---|---|
| < 10 queues, < 1K msg/giây | Big Bang + maintenance window | 1–2 ngày |
| 10–50 queues, 1K–100K msg/giây | Blue/Green Migration | 1–2 tuần |
| > 50 queues, > 100K msg/giây | Strangler Fig Pattern | 1–3 tháng |

---

## Giai Đoạn 1 — Đánh Giá Hiện Trạng

### Inventory (Kiểm Kê) Broker Hiện Tại

Trước khi migration, cần kiểm kê đầy đủ:

```bash
# ActiveMQ — Liệt kê tất cả queues và topics
activemq-admin --jmxurl service:jmx:rmi://localhost:1099/jndi/rmi://localhost:1099/jmxrmi \
  query "org.apache.activemq:type=Broker,brokerName=localhost,destinationType=Queue,*"

# ActiveMQ — Xem queue statistics (thống kê hàng đợi)
# Qua Web Console: http://localhost:8161/admin/queues.jsp

# RabbitMQ — Liệt kê tất cả exchanges, queues, bindings
rabbitmqctl list_queues name messages consumers durable
rabbitmqctl list_exchanges name type durable
rabbitmqctl list_bindings
```

### Checklist Đánh Giá

```
□ Danh sách tất cả Queues (tên, kích thước, consumer count)
□ Danh sách tất cả Topics / Exchanges
□ Message throughput hiện tại (msg/giây theo giờ, ngày)
□ Message size trung bình và lớn nhất
□ Retention period (thời gian lưu giữ) yêu cầu
□ Consumer count theo queue
□ Giao thức đang dùng (OpenWire, AMQP, MQTT, STOMP)
□ Authentication mechanism (cơ chế xác thực) — username/password hay certificate
□ TLS đang dùng không?
□ Message Selector đang dùng không?
□ XA Transaction đang dùng không?
□ Virtual Destinations (ActiveMQ) đang dùng không?
□ Dead Letter Queue configuration (cấu hình)
□ Dependencies: danh sách tất cả services kết nối
```

### Đánh Giá Rủi Ro

| Tính Năng | Tương Thích Amazon MQ | Ghi Chú |
|---|---|---|
| Standard AMQP queues | ✅ 100% | Không cần thay đổi |
| JMS API (Java) | ✅ 100% | Chỉ đổi connection URL |
| Message Selector | ✅ Hỗ trợ | Kiểm tra syntax |
| XA Transaction | ✅ ActiveMQ | Không có cho RabbitMQ |
| Virtual Topics | ✅ ActiveMQ | Cấu hình giống nhau |
| Custom plugins | ⚠️ Hạn chế | Một số plugin không được hỗ trợ |
| Custom network bridges | ⚠️ Cần kiểm tra | Cấu hình phức tạp |
| Proprietary features | ❌ Có thể không | Phải đánh giá từng trường hợp |

---

## Giai Đoạn 2 — Lên Kế Hoạch

### Chọn Instance Type (Loại Instance)

Amazon MQ dùng các instance type sau:

| Instance Type | vCPU | RAM | Throughput | Dùng Cho |
|---|---|---|---|---|
| `mq.t3.micro` | 2 | 1 GB | Thấp | Dev/Test |
| `mq.m5.large` | 2 | 8 GB | Trung bình | Small production |
| `mq.m5.xlarge` | 4 | 16 GB | Cao | Medium production |
| `mq.m5.2xlarge` | 8 | 32 GB | Rất cao | Large production |
| `mq.m5.4xlarge` | 16 | 64 GB | Cực cao | High-throughput |

### Công Thức Ước Tính Instance Size

```
Throughput cần thiết = (msg/giây) × (message size trung bình KB)

Ví dụ:
  1,000 msg/giây × 2 KB = 2 MB/giây
  → mq.m5.large là đủ (8GB RAM)
  → Chọn mq.m5.xlarge để có headroom (dư phòng) 2x

Quy tắc ngón tay cái:
  < 500 msg/giây    → mq.m5.large
  500-2K msg/giây   → mq.m5.xlarge
  2K-5K msg/giây    → mq.m5.2xlarge
  > 5K msg/giây     → mq.m5.4xlarge hoặc xem xét Kinesis/MSK
```

### Kiến Trúc Mạng (Network Architecture)

```
┌──────────────────────────────────────────────────────────────┐
│                    AWS VPC Architecture                      │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Private Subnet AZ-a        Private Subnet AZ-b     │    │
│  │  ┌────────────────────┐     ┌────────────────────┐  │    │
│  │  │  Amazon MQ         │     │  Amazon MQ         │  │    │
│  │  │  Active Broker     │────▶│  Standby Broker    │  │    │
│  │  └────────────────────┘     └────────────────────┘  │    │
│  │                                                     │    │
│  │  ┌────────────────────┐                             │    │
│  │  │  Application       │ ← Kết nối qua private IP    │    │
│  │  │  Servers           │                             │    │
│  │  └────────────────────┘                             │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  On-Premises ──[AWS Direct Connect / VPN]──▶ VPC            │
│  (Nếu migration dần dần cần hybrid connectivity)            │
└──────────────────────────────────────────────────────────────┘
```

---

## Giai Đoạn 3 — Chuẩn Bị Amazon MQ

### Tạo Broker Với AWS CLI

```bash
# Tạo ActiveMQ broker Active/Standby HA
aws mq create-broker \
  --broker-name "production-activemq" \
  --engine-type ACTIVEMQ \
  --engine-version "5.17.6" \
  --deployment-mode ACTIVE_STANDBY_MULTI_AZ \
  --host-instance-type "mq.m5.xlarge" \
  --publicly-accessible false \
  --subnet-ids subnet-aaaaaaaa subnet-bbbbbbbb \
  --security-groups sg-xxxxxxxx \
  --users '[
    {
      "Username": "admin",
      "Password": "SecurePassword123!",
      "Groups": ["admin"]
    }
  ]' \
  --logs '{"General": true, "Audit": true}' \
  --encryption-options '{"UseAwsOwnedKey": false, "KmsKeyId": "arn:aws:kms:..."}'
```

```bash
# Tạo RabbitMQ broker 3-node cluster
aws mq create-broker \
  --broker-name "production-rabbitmq" \
  --engine-type RABBITMQ \
  --engine-version "3.13" \
  --deployment-mode CLUSTER_MULTI_AZ \
  --host-instance-type "mq.m5.large" \
  --publicly-accessible false \
  --subnet-ids subnet-aaaaaaaa subnet-bbbbbbbb subnet-cccccccc \
  --security-groups sg-xxxxxxxx \
  --users '[{"Username": "admin", "Password": "SecurePassword123!"}]'
```

### Cấu Hình Broker (ActiveMQ)

Tạo file cấu hình `activemq.xml` trước khi tạo broker:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<broker xmlns="http://activemq.apache.org/schema/core"
        schedulePeriodForDestinationPurge="10000">

  <destinationPolicy>
    <policyMap>
      <policyEntries>
        <!-- Cấu hình mặc định cho tất cả queues -->
        <policyEntry queue=">" producerFlowControl="true"
                     memoryLimit="1mb">
          <!-- Dead Letter Queue configuration -->
          <deadLetterStrategy>
            <individualDeadLetterStrategy
              queuePrefix="DLQ." useQueueForQueueMessages="true"/>
          </deadLetterStrategy>
        </policyEntry>

        <!-- Virtual Topics — fan-out kết hợp load balancing -->
        <policyEntry topic="VirtualTopic.>" producerFlowControl="true"/>
      </policyEntries>
    </policyMap>
  </destinationPolicy>

  <!-- Memory limits (giới hạn bộ nhớ) -->
  <systemUsage>
    <systemUsage>
      <memoryUsage>
        <memoryUsage limit="6 gb"/>   <!-- 75% của 8GB RAM -->
      </memoryUsage>
      <storeUsage>
        <storeUsage limit="50 gb"/>
      </storeUsage>
      <tempUsage>
        <tempUsage limit="10 gb"/>
      </tempUsage>
    </systemUsage>
  </systemUsage>
</broker>
```

### Security Group Rules (Quy Tắc Nhóm Bảo Mật)

```
Inbound Rules (Quy Tắc Vào):
┌────────────────────────────────────────────────────────────┐
│  Protocol  │  Port   │  Source           │  Dịch Vụ       │
│────────────────────────────────────────────────────────────│
│  TCP       │  61617  │  App SG           │  OpenWire TLS  │
│  TCP       │  5671   │  App SG           │  AMQP TLS      │
│  TCP       │  8883   │  IoT devices      │  MQTT TLS      │
│  TCP       │  61614  │  Web clients      │  STOMP/WS TLS  │
│  TCP       │  443    │  Admin SG         │  Web Console   │
└────────────────────────────────────────────────────────────┘
```

---

## Giai Đoạn 4 — Migration Thực Tế

### Blue/Green Migration — Quy Trình Chi Tiết

```
                    Migration Timeline (Dòng Thời Gian Di Chuyển)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Week 1    Week 2    Week 3    Week 4    Week 5                             │
│  ───────   ───────   ───────   ───────   ──────                            │
│                                                                             │
│  [Prepare] [Test]    [Dual]    [Migrate] [Done]                            │
│     │         │        │          │        │                               │
│     ▼         ▼        ▼          ▼        ▼                               │
│  Setup MQ  Load     Run both   Switch   Decommission                       │
│  broker    testing  in parallel traffic  on-prem                           │
│                     verify               broker                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bước 1: Dual-Write Setup (Ghi Song Song)

Trong giai đoạn transition (chuyển tiếp), producers ghi vào cả hai brokers:

```java
public class DualWriteProducer {
    private MessageProducer onPremProducer;    // Broker on-premises
    private MessageProducer amazonMqProducer;  // Amazon MQ

    public void sendOrder(Order order) {
        String json = objectMapper.writeValueAsString(order);

        // Ghi vào cả hai broker song song
        CompletableFuture<Void> onPremFuture = CompletableFuture.runAsync(() -> {
            try {
                onPremProducer.send(createMessage(json));
            } catch (JMSException e) {
                log.error("On-prem send failed: {}", e.getMessage());
                // Không throw — vẫn tiếp tục ghi vào Amazon MQ
            }
        });

        CompletableFuture<Void> amazonFuture = CompletableFuture.runAsync(() -> {
            try {
                amazonMqProducer.send(createMessage(json));
            } catch (JMSException e) {
                log.error("Amazon MQ send failed: {}", e.getMessage());
                // Throw nếu Amazon MQ là primary
                throw new RuntimeException(e);
            }
        });

        // Chờ cả hai hoàn thành (hoặc chỉ primary)
        CompletableFuture.allOf(onPremFuture, amazonFuture).join();
    }
}
```

### Bước 2: Message Drain (Tháo Cạn Tin Nhắn)

Trước cutover, cần đảm bảo on-premises broker không còn message tồn đọng:

```bash
#!/bin/bash
# drain-check.sh — Kiểm tra on-premises broker đã rỗng chưa

BROKER_URL="tcp://onprem-broker:61616"
MAX_WAIT=3600  # 1 giờ tối đa
CHECK_INTERVAL=30

echo "Bắt đầu kiểm tra message drain..."

elapsed=0
while [ $elapsed -lt $MAX_WAIT ]; do
    # Lấy tổng số message còn lại
    total_msgs=$(activemq-admin --url "$BROKER_URL" \
        query "org.apache.activemq:type=Broker,brokerName=*,destinationType=Queue,*" \
        --view QueueSize | awk '{sum+=$1} END {print sum}')

    echo "[$(date)] Messages còn lại: $total_msgs"

    if [ "$total_msgs" -eq "0" ]; then
        echo "✅ Broker đã rỗng. Sẵn sàng cutover!"
        exit 0
    fi

    sleep $CHECK_INTERVAL
    elapsed=$((elapsed + CHECK_INTERVAL))
done

echo "❌ Timeout — vẫn còn message sau $MAX_WAIT giây"
exit 1
```

### Bước 3: Connection String Update (Cập Nhật Chuỗi Kết Nối)

Thay đổi connection string trong application config — đây thường là **thay đổi duy nhất** cần thiết:

```yaml
# application.yml — Before (Trước)
messaging:
  broker-url: "ssl://onprem-activemq.internal:61617"
  username: "${BROKER_USER}"
  password: "${BROKER_PASS}"

# application.yml — After (Sau)
messaging:
  broker-url: "ssl://b-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx-1.mq.ap-southeast-1.amazonaws.com:61617"
  username: "${AWS_MQ_USER}"
  password: "${AWS_MQ_PASS}"
```

```python
# Python — pika (RabbitMQ)
# Before (Trước)
params = pika.ConnectionParameters(host='onprem-rabbitmq.internal', port=5671)

# After (Sau)  
params = pika.ConnectionParameters(
    host='b-xxxxxxxx.mq.ap-southeast-1.amazonaws.com',
    port=5671,
    credentials=pika.PlainCredentials(
        os.environ['AWS_MQ_USER'],
        os.environ['AWS_MQ_PASS']
    ),
    ssl_options=pika.SSLOptions(ssl.create_default_context())
)
```

---

## Giai Đoạn 5 — Cutover và Verification

### Cutover Checklist (Danh Sách Kiểm Tra Chuyển Đổi)

```
Pre-Cutover (Trước Chuyển Đổi):
□ On-premises broker đã drain (rỗng message)
□ Amazon MQ broker đã test kỹ ở môi trường staging
□ Connection strings đã cập nhật trong config management (Ansible/Terraform)
□ Monitoring và alerting đã cấu hình trên CloudWatch
□ Rollback plan đã được review và test
□ Maintenance window đã thông báo đến stakeholders

During Cutover (Trong Lúc Chuyển Đổi):
□ Enable maintenance mode trên ứng dụng nếu cần
□ Chờ on-premises broker rỗng hoàn toàn
□ Deploy config mới với Amazon MQ connection string
□ Monitor first messages flowing through Amazon MQ
□ Verify consumer count khớp với expected

Post-Cutover (Sau Chuyển Đổi):
□ Monitor message throughput (thông lượng tin nhắn) trong 1 giờ đầu
□ Verify DLQ count = 0 (không có message bị lỗi)
□ Kiểm tra latency (độ trễ) trong CloudWatch
□ Confirm tất cả services đã kết nối Amazon MQ
□ Disable dual-write nếu đang dùng
□ Lên lịch decommission on-premises broker sau 2 tuần
```

### Verification Script (Script Kiểm Tra)

```python
import boto3
import time

def verify_amazon_mq_health(broker_id, expected_queue_count):
    """
    Kiểm tra Amazon MQ broker sau cutover (sau chuyển đổi)
    """
    mq_client = boto3.client('mq')
    cw_client = boto3.client('cloudwatch')

    print(f"Bắt đầu verification cho broker {broker_id}...")

    # 1. Kiểm tra broker state (trạng thái broker)
    broker = mq_client.describe_broker(BrokerId=broker_id)
    state = broker['BrokerState']
    print(f"Broker state: {state}")
    assert state == 'RUNNING', f"❌ Broker không ở trạng thái RUNNING: {state}"
    print("✅ Broker đang chạy")

    # 2. Kiểm tra metrics trên CloudWatch
    now = time.time()
    metrics_to_check = [
        ('TotalMessageCount', 'Sum', 'Số message hiện có'),
        ('ConsumerCount', 'Average', 'Số consumer đang kết nối'),
        ('EnqueueCount', 'Sum', 'Số message đã nhận'),
    ]

    for metric_name, stat, description in metrics_to_check:
        response = cw_client.get_metric_statistics(
            Namespace='AWS/AmazonMQ',
            MetricName=metric_name,
            Dimensions=[{'Name': 'Broker', 'Value': broker_id}],
            StartTime=now - 300,  # 5 phút gần nhất
            EndTime=now,
            Period=300,
            Statistics=[stat]
        )
        if response['Datapoints']:
            value = response['Datapoints'][0][stat]
            print(f"✅ {description}: {value}")
        else:
            print(f"⚠️  {description}: Chưa có data (có thể do chưa đủ 5 phút)")

    print("\n✅ Verification hoàn thành. Amazon MQ đang hoạt động bình thường!")
```

---

## Xử Lý Tình Huống Phức Tạp

### Tình Huống 1: Messages Đang In-Flight Khi Cutover

```
Vấn đề: Message đang được xử lý trên on-premises khi chuyển đổi

Giải pháp:
┌─────────────────────────────────────────────────────────────┐
│  1. Sử dụng "Graceful Consumer Shutdown":                   │
│     - Stop producers trước                                  │
│     - Chờ consumers drain hàng đợi                         │
│     - Chuyển consumers sang Amazon MQ                       │
│     - Start producers trên Amazon MQ                        │
│                                                             │
│  2. Sử dụng Message ID Tracking (Theo Dõi ID Tin Nhắn):     │
│     - Lưu đã-xử lý message IDs vào Redis/DynamoDB           │
│     - Consumer kiểm tra trước khi xử lý (idempotency)      │
│     - Chạy cả hai consumers trong thời gian overlap         │
└─────────────────────────────────────────────────────────────┘
```

### Tình Huống 2: Custom Plugin Không Được Hỗ Trợ

```
Ví dụ: On-premises ActiveMQ dùng custom authentication plugin

Giải pháp cho Amazon MQ:
┌─────────────────────────────────────────────────────────────┐
│  Amazon MQ hỗ trợ:                                          │
│  ✅ Username/Password authentication (xác thực)             │
│  ✅ LDAP integration (tích hợp LDAP)                        │
│  ✅ TLS client certificates                                  │
│                                                             │
│  Nếu cần custom auth plugin:                                │
│  → Đặt auth proxy (proxy xác thực) trước Amazon MQ         │
│  → Hoặc chuyển sang IAM-based access với VPC endpoints      │
└─────────────────────────────────────────────────────────────┘
```

### Tình Huống 3: Hybrid Architecture Trong Giai Đoạn Chuyển Tiếp

Khi một số services vẫn on-premises trong khi others đã lên AWS:

```
┌─────────────────────────────────────────────────────────────┐
│              Hybrid Messaging Bridge                        │
│              (Cầu Nối Nhắn Tin Lai)                         │
│                                                             │
│  On-Premises          AWS                                   │
│  ┌──────────────┐     ┌──────────────────────────────────┐  │
│  │  Old Broker  │────▶│  Network Bridge / Camel Route    │  │
│  │  (ActiveMQ)  │◀────│  (Cầu nối mạng / Tuyến Camel)    │  │
│  └──────────────┘     │  ┌──────────────────────────────┐│  │
│                        │  │  Amazon MQ                   ││  │
│  ┌──────────────┐     │  │  (Target Broker)             ││  │
│  │  Legacy Apps │     │  └──────────────────────────────┘│  │
│  └──────────────┘     └──────────────────────────────────┘  │
│                                                             │
│  Kết nối qua: AWS Direct Connect hoặc Site-to-Site VPN     │
└─────────────────────────────────────────────────────────────┘
```

```java
// Apache Camel Bridge — Forward messages từ on-prem sang Amazon MQ
@Component
public class BrokerBridgeRoute extends RouteBuilder {
    @Override
    public void configure() {
        // Forward tất cả messages từ on-prem queue sang Amazon MQ
        from("activemq:queue:ORDERS?brokerURL=tcp://onprem-broker:61616")
            .log("Forwarding message: ${body}")
            .to("activemq:queue:ORDERS?brokerURL=" +
                "ssl://amazon-mq-broker.amazonaws.com:61617" +
                "&username={{aws.mq.user}}" +
                "&password={{aws.mq.pass}}");
    }
}
```

---

## Rollback Plan

### Khi Nào Cần Rollback (Quay Lại)

```
Trigger Rollback Ngay Khi:
❌ Latency tăng > 3x so với on-premises
❌ Error rate > 1% trong 5 phút đầu sau cutover
❌ Consumer count < expected (consumers không kết nối được)
❌ Message loss phát hiện (số lượng không khớp)
❌ Amazon MQ broker state = ERROR
```

### Rollback Procedure (Quy Trình Quay Lại)

```bash
#!/bin/bash
# rollback.sh — Quay lại on-premises broker

echo "⚠️  Bắt đầu rollback về on-premises broker..."

# Bước 1: Cập nhật config về on-premises connection string
kubectl set env deployment/order-service \
  BROKER_URL="ssl://onprem-broker.internal:61617" \
  BROKER_USER="$OLD_BROKER_USER" \
  BROKER_PASS="$OLD_BROKER_PASS"

# Bước 2: Restart services để pick up config mới
kubectl rollout restart deployment/order-service
kubectl rollout restart deployment/notification-service

# Bước 3: Chờ rollout hoàn thành
kubectl rollout status deployment/order-service

# Bước 4: Verify connections
kubectl exec -it $(kubectl get pod -l app=order-service -o name | head -1) -- \
  curl -s http://localhost:8080/actuator/health | jq .components.messaging

echo "✅ Rollback hoàn thành. Đang dùng on-premises broker."
```

---

## Post-Migration Optimization

### Tối Ưu Sau Migration

**1. Cấu Hình Pre-Fetch Size (Kích Thước Lấy Trước)**

```java
// Tăng prefetch để tăng throughput
ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory(brokerUrl);
factory.getPrefetchPolicy().setQueuePrefetch(500);  // Mặc định là 1000
factory.getPrefetchPolicy().setTopicPrefetch(32766);
```

**2. Connection Pooling (Tổng Hợp Kết Nối)**

```java
// Dùng PooledConnectionFactory để tái sử dụng connections
@Bean
public ConnectionFactory connectionFactory() {
    PooledConnectionFactory pooled = new PooledConnectionFactory();
    pooled.setConnectionFactory(new ActiveMQSslConnectionFactory(brokerUrl));
    pooled.setMaxConnections(10);
    pooled.setMaximumActiveSessionPerConnection(500);
    pooled.setIdleTimeout(30000);
    return pooled;
}
```

**3. CloudWatch Alarms Sau Migration**

```bash
# Tạo alarm khi queue tích tụ quá nhiều message
aws cloudwatch put-metric-alarm \
  --alarm-name "AmazonMQ-HighMessageCount" \
  --metric-name "TotalMessageCount" \
  --namespace "AWS/AmazonMQ" \
  --dimensions Name=Broker,Value=production-activemq \
  --statistic Average \
  --period 300 \
  --threshold 10000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:ap-southeast-1:account-id:alerts-topic"

# Alarm khi không có consumer
aws cloudwatch put-metric-alarm \
  --alarm-name "AmazonMQ-NoConsumers" \
  --metric-name "ConsumerCount" \
  --namespace "AWS/AmazonMQ" \
  --dimensions Name=Broker,Value=production-activemq \
  --statistic Minimum \
  --period 60 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions "arn:aws:sns:ap-southeast-1:account-id:alerts-topic"
```

### Sau 2 Tuần Ổn Định — Xem Xét Modernization (Hiện Đại Hóa)

```
Migration Path (Lộ Trình Chuyển Đổi) sau Amazon MQ:

Amazon MQ (Phase 1)
       │
       │ Sau khi stable 1-3 tháng
       ▼
Evaluate SQS/SNS Migration (Đánh giá di chuyển sang SQS/SNS)
  - Ứng dụng có thể thay đổi code không?
  - Protocol dependency có bắt buộc không?
       │
       ├── Có thể thay code → Migrate sang SQS/SNS (cheaper, cloud-native)
       └── Không thể đổi code → Tiếp tục dùng Amazon MQ
```

---

## Checklist Phỏng Vấn

### Câu Hỏi Thường Gặp

**Q: Tại sao dùng Amazon MQ thay vì tự dựng ActiveMQ/RabbitMQ trên EC2?**

> Amazon MQ tự động xử lý patching (vá lỗi), backup, HA failover và monitoring. Tự dựng trên EC2 cần quản lý tất cả điều này thủ công — tốn nhân lực vận hành và rủi ro cao hơn. Trade-off: Amazon MQ ít kiểm soát hơn nhưng operational burden (gánh nặng vận hành) thấp hơn nhiều.

**Q: Khi migration từ on-premises, tại sao không cần thay đổi code?**

> Vì Amazon MQ dùng cùng engine (ActiveMQ/RabbitMQ) và hỗ trợ cùng giao thức (AMQP, MQTT, STOMP, OpenWire). Ứng dụng chỉ cần thay connection URL và credentials — logic xử lý message không thay đổi.

**Q: Chiến lược Blue/Green Migration cho Amazon MQ hoạt động thế nào?**

> Chạy on-premises broker (blue) và Amazon MQ (green) song song, dual-write (ghi song song) vào cả hai. Dần dần chuyển consumers sang Amazon MQ, verify không có lỗi, rồi cutover producers sang Amazon MQ hoàn toàn. Giữ on-premises broker standby (dự phòng) thêm 2 tuần trước khi decommission.

**Q: Làm thế nào để zero-downtime migration cho critical system?**

> Dùng dual-write pattern kết hợp message ID tracking và idempotency ở consumer. Producers ghi vào cả hai, consumers có thể đọc từ cả hai và bỏ qua duplicates dựa trên message ID. Sau khi verify Amazon MQ ổn định, tắt on-premises dần dần.

### Kiến Thức Cần Nắm

- [ ] 3 chiến lược migration và khi nào dùng từng loại
- [ ] Blue/Green Migration — dual-write pattern
- [ ] Message drain procedure trước cutover
- [ ] Rollback triggers và rollback procedure
- [ ] Post-migration monitoring: metrics quan trọng
- [ ] Hybrid architecture khi migration từng phần
- [ ] Connection pooling và prefetch optimization

---

## 🔗 Điều Hướng

| Bước Trước | Bước Tiếp Theo |
|---|---|
| [1-activemq-vs-rabbitmq.md](./1-activemq-vs-rabbitmq.md) — So Sánh Engine | [07-appsync/README.md](../07-appsync/README.md) — AWS AppSync |

Module tiếp theo: [07-appsync/](../07-appsync/README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-18
