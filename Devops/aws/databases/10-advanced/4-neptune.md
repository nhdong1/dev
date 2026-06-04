# 4. Amazon Neptune — Cơ Sở Dữ Liệu Đồ Thị

> Amazon Neptune là dịch vụ **graph database** (cơ sở dữ liệu đồ thị) được quản lý hoàn toàn, được thiết kế để lưu trữ và truy vấn dữ liệu có mối quan hệ phức tạp — ví dụ: mạng xã hội (social graph), phát hiện gian lận (fraud detection), đồ thị kiến thức (knowledge graph).

## 📚 Mục Lục

1. [Graph Database là Gì?](#graph-database-là-gì)
2. [Kiến Trúc Neptune](#kiến-trúc-neptune)
3. [Gremlin — Graph Traversal Language](#gremlin--graph-traversal-language)
4. [SPARQL — RDF Query Language](#sparql--rdf-query-language)
5. [openCypher — Cypher Dialect](#opencypher--cypher-dialect)
6. [Use Cases Thực Tế](#use-cases-thực-tế)
7. [So Sánh Neptune vs Relational Database](#so-sánh-neptune-vs-relational-database)
8. [High Availability & Performance](#high-availability--performance)
9. [Neptune Serverless](#neptune-serverless)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Graph Database là Gì?

### Mô Hình Dữ Liệu

Graph database lưu dữ liệu dưới dạng **nodes** (nút) và **edges** (cạnh):

```
                   WORKS_AT
   [Alice] ──────────────────► [Company ABC]
      │                              │
      │ KNOWS                        │ HAS_OFFICE
      ▼                              ▼
   [Bob]  ──────────────────► [New York Office]
                   LOCATED_IN
```

- **Node (Nút):** Thực thể — Alice, Bob, Company ABC
- **Edge (Cạnh):** Mối quan hệ — WORKS_AT, KNOWS, LOCATED_IN
- **Property (Thuộc Tính):** Metadata của node hoặc edge

### Tại Sao Dùng Graph Database?

Với **relational database** (cơ sở dữ liệu quan hệ), truy vấn mối quan hệ nhiều chiều (multi-hop) yêu cầu nhiều JOIN tốn kém:

```sql
-- Tìm bạn bè của bạn bè của Alice (2-hop)
SELECT DISTINCT u3.name
FROM users u1
JOIN friendships f1 ON u1.id = f1.user_id
JOIN users u2 ON f1.friend_id = u2.id
JOIN friendships f2 ON u2.id = f2.user_id
JOIN users u3 ON f2.friend_id = u3.id
WHERE u1.name = 'Alice' AND u3.id != u1.id;
-- Càng nhiều hop → càng nhiều JOIN → càng chậm
```

```gremlin
// Neptune Gremlin: 2-hop traverse
g.V().has('name', 'Alice')
 .out('KNOWS').out('KNOWS')
 .dedup()
 .values('name')
// Tự nhiên hơn, tối ưu hơn cho graph traversal
```

---

## Kiến Trúc Neptune

### Cấu Trúc Tổng Thể

```
┌───────────────────────────────────────────────────────────────────┐
│                       AMAZON NEPTUNE CLUSTER                       │
│                                                                   │
│  ┌──────────────────┐    ┌──────────────────────────────────────┐│
│  │  Primary Instance│    │    Read Replicas (Tối đa 15)         ││
│  │  (Nút Ghi Chính) │    │    (Các Nút Đọc)                     ││
│  │                  │    │  ┌──────────┐  ┌──────────┐          ││
│  │  - Gremlin API   │    │  │ Replica 1│  │ Replica 2│  ...     ││
│  │  - SPARQL API    │    │  └──────────┘  └──────────┘          ││
│  │  - openCypher    │    └──────────────────────────────────────┘│
│  └──────────────────┘                                            │
│           │                         ↑ Replication                │
│           └─────────────────────────┘                            │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │          Neptune Cluster Storage (64TB max)                │   │
│  │          Shared, replicated, distributed (như Aurora)     │   │
│  └───────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

### Kiến Trúc Giống Aurora

Neptune dùng **shared cluster storage** (lưu trữ cluster dùng chung) tương tự Aurora:
- Storage tự động grow, không cần provision
- 6 bản sao trên 3 Availability Zones
- Failover tự động (< 30 giây)

### Query Languages Hỗ Trợ

| Language | Standard | Mô Tả |
|----------|----------|-------|
| **Gremlin** | Apache TinkerPop | Property graph traversal |
| **SPARQL** | W3C Standard | RDF (Resource Description Framework) triplestores |
| **openCypher** | openCypher project | Cypher-compatible (như Neo4j) |

---

## Gremlin — Graph Traversal Language

Gremlin là ngôn ngữ traversal (duyệt đồ thị) cho **Property Graph** (Đồ Thị Thuộc Tính).

### Cú Pháp Cơ Bản

```gremlin
// Thêm vertices (đỉnh/nút)
g.addV('person').property('name', 'Alice').property('age', 30)
g.addV('person').property('name', 'Bob').property('age', 25)
g.addV('company').property('name', 'TechCorp')

// Thêm edges (cạnh/mối quan hệ)
g.V().has('name', 'Alice').addE('KNOWS').to(g.V().has('name', 'Bob'))
g.V().has('name', 'Alice').addE('WORKS_AT').to(g.V().has('name', 'TechCorp'))

// Truy vấn cơ bản
g.V().has('name', 'Alice')               // Tìm node Alice
g.V().has('name', 'Alice').out('KNOWS')  // Bạn bè của Alice
g.V().has('name', 'Alice').out('KNOWS').values('name')  // Tên bạn bè
```

### Traversal Phức Tạp

```gremlin
// Fraud detection: Tìm tài khoản dùng cùng thiết bị trong 24h
g.V().has('account', 'id', 'suspect_account')
 .out('USED_DEVICE')
 .in('USED_DEVICE')
 .where(neq('suspect_account'))
 .dedup()
 .limit(100)

// Social graph: Gợi ý kết bạn (bạn chung > 3 người)
g.V().has('person', 'id', 'alice')
 .out('KNOWS').aggregate('alice_friends')
 .out('KNOWS')
 .where(without('alice_friends'))
 .groupCount()
 .unfold()
 .where(select(values).is(gte(3)))  // Có ít nhất 3 bạn chung
 .select(keys)
 .values('name')

// Shortest path (Đường đi ngắn nhất) giữa hai người
g.V().has('name', 'Alice')
 .repeat(out('KNOWS').simplePath())
 .until(has('name', 'Carol'))
 .path()
 .limit(1)
```

### Gremlin với Python

```python
from gremlin_python.driver import client, serializer

# Kết nối đến Neptune
neptune_endpoint = 'wss://your-neptune-endpoint:8182/gremlin'
gremlin_client = client.Client(
    neptune_endpoint,
    'g',
    message_serializer=serializer.GraphSONMessageSerializer()
)

# Chạy query
query = "g.V().has('person', 'name', 'Alice').out('KNOWS').values('name')"
result = gremlin_client.submit(query)
friends = result.all().result()
print(friends)  # ['Bob', 'Carol']
```

---

## SPARQL — RDF Query Language

SPARQL dùng cho **RDF** (Resource Description Framework — Khung Mô Tả Tài Nguyên) data model — lưu dữ liệu dưới dạng **triples** (bộ ba): Subject - Predicate - Object.

### RDF Triple (Bộ Ba RDF)

```
<Alice>  <knows>   <Bob>        # Subject - Predicate - Object
<Alice>  <worksAt> <TechCorp>
<TechCorp> <hasOffice> <NewYork>
```

### SPARQL Query

```sparql
# Tìm tất cả người Alice biết và nơi họ làm việc
PREFIX ex: <http://example.org/>

SELECT ?friend ?company
WHERE {
    ex:Alice ex:knows ?friend .
    ?friend ex:worksAt ?company .
}
```

### Khi Nào Dùng SPARQL vs Gremlin?

| | SPARQL / RDF | Gremlin / Property Graph |
|-|-------------|--------------------------|
| **Tiêu chuẩn** | W3C standard, linked data | Apache TinkerPop |
| **Use case** | Knowledge graphs, semantic web | Social graphs, fraud, recommendations |
| **Data model** | Triples, schema flexible | Nodes + edges với properties |
| **Phổ biến hơn** | Research, government, biomedical | Commercial applications |

---

## openCypher — Cypher Dialect

openCypher là implementation của Cypher query language (tương thích Neo4j) trên Neptune.

```cypher
// Tạo nodes và relationships
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 25})
CREATE (alice)-[:KNOWS {since: 2020}]->(bob)

// Truy vấn
MATCH (p:Person)-[:KNOWS]->(friend:Person)
WHERE p.name = 'Alice'
RETURN friend.name, friend.age

// Pattern matching (So Khớp Mẫu) phức tạp
MATCH path = shortestPath(
  (alice:Person {name: 'Alice'})-[:KNOWS*]-(carol:Person {name: 'Carol'})
)
RETURN path
```

---

## Use Cases Thực Tế

### 1. Social Graph (Đồ Thị Mạng Xã Hội)

```
Nodes:   User, Post, Comment, Group
Edges:   FOLLOWS, LIKES, POSTED, MEMBER_OF, COMMENTED_ON

Queries:
- Gợi ý kết bạn (mutual friends)
- Feed: Bài đăng của người tôi follow
- Phát hiện community (nhóm người kết nối dày đặc)
```

### 2. Fraud Detection (Phát Hiện Gian Lận)

```
Nodes:   Account, Device, IP_Address, Phone, Email
Edges:   USED_DEVICE, LOGGED_FROM, REGISTERED_WITH

Query: Tìm cluster tài khoản chia sẻ thiết bị/IP → dấu hiệu fraud ring
g.V().has('account', 'id', suspect)
 .out('USED_DEVICE').in('USED_DEVICE')  # Tài khoản khác dùng cùng device
 .out('REGISTERED_WITH')                # Email đăng ký
 .groupCount()
```

### 3. Knowledge Graph (Đồ Thị Kiến Thức)

```
Nodes:   Entity (Drug, Disease, Gene, Protein)
Edges:   TREATS, CAUSES, INTERACTS_WITH, EXPRESSED_IN

Use case: Biomedical research, drug discovery
SPARQL: Tìm tất cả gene liên quan đến một bệnh thông qua proteins
```

### 4. Recommendation Engine (Hệ Thống Gợi Ý)

```
Nodes:   User, Product, Category
Edges:   PURCHASED, VIEWED, BELONGS_TO, SIMILAR_TO

Query: Gợi ý sản phẩm — user mua A và B thường cũng mua C
g.V().has('user', 'id', current_user)
 .out('PURCHASED')
 .in('PURCHASED').where(neq(current_user))  # Users khác mua cùng sản phẩm
 .out('PURCHASED')
 .where(not(__.in('PURCHASED').has('user', 'id', current_user)))  # Chưa mua
 .groupCount()
 .order(local).by(values, desc)
 .limit(local, 10)
```

---

## So Sánh Neptune vs Relational Database

### Khi Nào Dùng Neptune?

```
Dùng Neptune khi:
✅ Mối quan hệ LÀ dữ liệu quan trọng, không chỉ là foreign keys
✅ Multi-hop traversal (duyệt nhiều bước) là query chính
✅ Schema linh hoạt — nodes có thể có properties khác nhau
✅ Graph algorithms: shortest path, PageRank, community detection

Dùng RDS/Aurora khi:
✅ Dữ liệu có cấu trúc bảng rõ ràng
✅ Cần SQL, joins, aggregations phức tạp
✅ ACID transactions đầy đủ ở production scale
✅ Reporting và analytics SQL-based
```

### Benchmark: Multi-hop Query

```
Tìm tất cả friends-of-friends trong mạng 1M users:

PostgreSQL (3-hop JOIN):
  Execution time: 45 giây
  Query: 3 self-joins trên bảng friends (1M rows)

Neptune (Gremlin):
  Execution time: 0.3 giây
  Query: g.V(user).repeat(out('KNOWS')).times(3).dedup()
  
→ Neptune nhanh hơn 150x cho graph traversal
```

---

## High Availability & Performance

### Multi-AZ Setup

Neptune tự động replicates storage qua 3 AZs (Availability Zones — Vùng Sẵn Sàng). Failover < 30 giây.

### Read Replicas

- Tối đa 15 read replicas trong cùng region
- Dùng cho read-heavy workloads (analytics, recommendations)
- Replica lag rất thấp (cùng shared storage)

### Neptune Streams

```
Neptune Streams — tương tự DynamoDB Streams
→ Capture mọi thay đổi trong graph
→ Đẩy vào Kinesis, Lambda
→ Use case: Audit log, real-time sync, change data capture
```

---

## Neptune Serverless

**Neptune Serverless** (Neptune Không Máy Chủ) tự động scale compute, phù hợp cho variable workload:

```bash
aws neptune create-db-cluster \
  --db-cluster-identifier my-neptune-cluster \
  --engine neptune \
  --serverless-v2-scaling-configuration MinCapacity=1,MaxCapacity=32
```

| | Neptune Provisioned | Neptune Serverless |
|-|--------------------|--------------------|
| **Quản lý** | Chọn instance type | Không cần chọn |
| **Pricing** | Per instance-hour | Per NCU (Neptune Capacity Unit) giây |
| **Scale** | Manual resize | Tự động |
| **Use case** | Predictable workload | Dev/test, variable workload |

---

## Câu Hỏi Phỏng Vấn

### Q1: Khi nào dùng Neptune thay vì DynamoDB?

> **Trả lời:** Dùng Neptune khi **mối quan hệ giữa entities là core** của data model, cần **multi-hop traversal** (duyệt qua nhiều mối quan hệ) hiệu quả. Ví dụ: fraud detection cần tìm cluster tài khoản qua 3-4 hops; social graph cần "bạn của bạn của bạn".
>
> DynamoDB tốt cho key-value lookups, session data, event store — khi relationship queries không phải core workload.

### Q2: Giải thích Property Graph vs RDF

> **Trả lời:**
> - **Property Graph:** Nodes và edges có properties (thuộc tính) tùy ý. Dùng Gremlin hoặc openCypher. Phổ biến cho commercial apps (social, fraud, recommendations).
> - **RDF (Resource Description Framework):** Lưu dữ liệu dưới dạng triples (Subject-Predicate-Object). Dùng SPARQL. Phổ biến cho linked data, semantic web, biomedical knowledge graphs.

### Q3: Neptune xử lý failover thế nào?

> **Trả lời:** Neptune dùng shared cluster storage (giống Aurora). Storage tự động replicated qua 3 AZs. Khi primary instance fail, Neptune tự động promote read replica thành primary — thường < 30 giây. Ứng dụng cần retry logic khi kết nối bị gián đoạn trong thời gian failover.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
