# AWS Database Services — Bảng Chỉ Mục Đầy Đủ

> Hướng dẫn toàn diện về AWS Database Services — dịch vụ cơ sở dữ liệu trên Amazon Web Services

## 📁 Cấu Trúc Thư Mục

```
Devops/aws/databases/
├── README.md                                   [BẮT ĐẦU TỪ ĐÂY] Lộ trình học & tổng quan
├── INDEX.md                                    Bảng chỉ mục đầy đủ (file này)
│
├── 01-rds-fundamentals/
│   ├── README.md                               Tổng quan RDS, engine types, storage
│   ├── 1-engine-types.md                       (Tạo sau) MySQL, PostgreSQL, MariaDB, Oracle, SQL Server
│   ├── 2-instance-storage.md                   (Tạo sau) Instance classes, storage types, IOPS
│   ├── 3-multi-az.md                           (Tạo sau) Multi-AZ deployment, failover tự động
│   ├── 4-read-replicas.md                      (Tạo sau) Read replicas, cross-region replication
│   ├── 5-parameter-groups.md                   (Tạo sau) Parameter groups, option groups, tuning
│   └── 6-rds-proxy.md                          (Tạo sau) RDS Proxy, connection pooling
│
├── 02-aurora/
│   ├── README.md                               (Tạo sau) Tổng quan Aurora, kiến trúc, so sánh
│   ├── 1-aurora-architecture.md                (Tạo sau) Shared storage, cluster endpoints
│   ├── 2-aurora-serverless.md                  (Tạo sau) Aurora Serverless v2, auto-scaling
│   ├── 3-aurora-global.md                      (Tạo sau) Global Database, cross-region replication
│   ├── 4-aurora-vs-rds.md                      (Tạo sau) Trade-offs, cost comparison
│   └── 5-aurora-ha-failover.md                 (Tạo sau) High availability, failover mechanisms
│
├── 03-dynamodb/
│   ├── README.md                               (Tạo sau) Tổng quan DynamoDB, data model
│   ├── 1-data-model.md                         (Tạo sau) Tables, partition key, sort key, attributes
│   ├── 2-capacity-modes.md                     (Tạo sau) On-demand vs provisioned, auto-scaling
│   ├── 3-indexes.md                            (Tạo sau) GSI, LSI — thiết kế và use cases
│   ├── 4-streams-lambda.md                     (Tạo sau) DynamoDB Streams, Lambda integration
│   ├── 5-transactions-acid.md                  (Tạo sau) Transactions, ACID trên DynamoDB
│   ├── 6-dax.md                                (Tạo sau) DynamoDB Accelerator, in-memory cache
│   └── 7-access-patterns.md                    (Tạo sau) Single-table design, access patterns
│
├── 04-elasticache/
│   ├── README.md                               Tổng quan ElastiCache, Redis vs Memcached, kiến trúc
│   ├── 1-redis-vs-memcached.md                 So sánh chi tiết Redis & Memcached, khi nào dùng gì
│   ├── 2-redis-cluster.md                      Cluster mode, Replication Groups, Sharding, Failover
│   ├── 3-caching-strategies.md                 Lazy Loading, Write-Through, Write-Around, TTL, Invalidation
│   ├── 4-persistence.md                        RDB Snapshots, AOF Logging, Backup & Restore trên AWS
│   └── 5-security.md                           VPC, Security Groups, Encryption (TLS/KMS), Auth Tokens, IAM
│
├── 05-ha-backup/
│   ├── README.md                               Tổng quan HA & Backup, kiến trúc, lộ trình học
│   ├── 1-rpo-rto.md                            RPO, RTO — yêu cầu kinh doanh, thiết kế chiến lược
│   ├── 2-automated-backups.md                  RDS Automated Backups, retention policy, cấu hình
│   ├── 3-snapshots.md                          Manual Snapshots, cross-region copy, lifecycle
│   ├── 4-pitr.md                               Point-in-Time Recovery, quy trình khôi phục
│   └── 5-disaster-recovery.md                  DR strategies (4 loại), multi-region failover, runbook
│
├── 06-security/
│   ├── README.md                               Bảo mật cơ sở dữ liệu toàn diện, Defense in Depth
│   ├── 1-vpc-security-groups.md                VPC design, private subnets, security groups, NACLs
│   ├── 2-iam-authentication.md                 IAM roles, database authentication, Least Privilege
│   ├── 3-encryption.md                         KMS, encryption at rest & in transit, TLS/SSL
│   ├── 4-secrets-management.md                 Secrets Manager, Parameter Store, automatic rotation
│   └── 5-audit-compliance.md                   CloudTrail, Activity Streams, PCI/HIPAA/GDPR
│
├── 07-performance-tuning/
│   ├── README.md                               Tối ưu hiệu năng — methodology, công cụ, vòng lặp tối ưu
│   ├── 1-performance-insights.md               RDS Performance Insights, AAS, wait events, top SQL
│   ├── 2-cloudwatch-metrics.md                 Key metrics RDS/Aurora/DynamoDB/ElastiCache, Enhanced Monitoring
│   ├── 3-slow-query-analysis.md                Slow query log, EXPLAIN plans, mysqldumpslow, pt-query-digest
│   ├── 4-connection-pooling.md                 RDS Proxy, pgBouncer, connection limits, IAM Auth
│   ├── 5-dynamodb-performance.md               Hot partitions, throttling, write sharding, DAX, Adaptive Capacity
│   └── 6-index-optimization.md                 Index types, composite index, covering index, GSI optimization
│
├── 08-migration/
│   ├── README.md                               Tổng quan migration — chiến lược, công cụ, quy trình
│   ├── 1-dms-overview.md                       AWS DMS architecture, replication instance, task types
│   ├── 2-sct.md                                Schema Conversion Tool, heterogeneous migration, data type mapping
│   ├── 3-cdc-online-migration.md               CDC, online migration, zero-downtime, validation
│   ├── 4-migration-strategies.md               Lift-and-shift, re-platform, re-architect, decision framework
│   └── 5-cutover-runbook.md                    Cutover planning, rollback, post-migration validation
│
├── 09-monitoring/
│   ├── README.md                               Tổng quan Monitoring & Observability, kiến trúc, lộ trình học
│   ├── 1-cloudwatch-dashboards.md              CloudWatch Metrics, key metrics theo dịch vụ, tạo dashboard
│   ├── 2-alerting-strategy.md                  SLO, SLA, thresholds, alert routing, chống alert fatigue
│   ├── 3-rds-enhanced-monitoring.md            OS-level metrics, process monitoring, phân tích sự cố
│   ├── 4-dynamodb-monitoring.md                Capacity alarms, throttling detection, Contributor Insights
│   └── 5-activity-streams.md                   Database Activity Streams, Kinesis, SIEM, compliance PCI/HIPAA
│
├── 10-advanced/
│   ├── README.md                               Tổng quan chủ đề nâng cao — multi-region, analytics, graph, document, time-series
│   ├── 1-redshift.md                           Redshift architecture, distribution keys, sort keys, Spectrum, Serverless
│   ├── 2-aurora-global-advanced.md             Aurora Global Database, multi-region active-active, write forwarding, conflict resolution
│   ├── 3-dynamodb-global-tables.md             Global Tables v2, eventual consistency, last-write-wins, conflict patterns
│   ├── 4-neptune.md                            Graph database, Gremlin, SPARQL, openCypher, fraud detection, social graph
│   ├── 5-documentdb.md                         DocumentDB, MongoDB compatibility, aggregation pipeline, migration
│   └── 6-timestream.md                         Time-series database, IoT use cases, memory/magnetic store, scheduled queries
│
├── 11-cost-optimization/
│   ├── README.md                               Tổng quan, framework tối ưu chi phí, quick wins
│   ├── 1-rds-pricing.md                        RDS pricing models, Reserved Instances, storage tiers
│   ├── 2-aurora-cost.md                        Aurora I/O-Optimized vs Standard, Serverless v2 cost model
│   ├── 3-dynamodb-cost.md                      On-demand vs provisioned, auto-scaling, TTL, bẫy chi phí
│   ├── 4-elasticache-cost.md                   ElastiCache pricing, Reserved Nodes, Serverless, right-sizing
│   └── 5-rightsizing.md                        Instance sizing methodology, storage optimization, automation
│
└── 12-interview-prep/
    ├── README.md                               Tổng quan chuẩn bị phỏng vấn, lộ trình 2 tuần, framework trả lời
    ├── INTERVIEW_GUIDE.md                      Top 20 câu hỏi thường gặp, câu trả lời mẫu, tips phỏng vấn
    ├── system-design-scenarios.md              5 kịch bản thiết kế hệ thống hoàn chỉnh với database considerations
    ├── trade-off-discussions.md                SQL vs NoSQL, RDS vs Aurora vs DynamoDB, caching, consistency
    └── star-stories.md                         10 mẫu câu chuyện STAR về database incidents & achievements
```

---

## ✅ Những Gì Đã Được Tạo

| Chủ Đề                                        | File                                        | Trạng Thái | Chất Lượng    |
| --------------------------------------------- | ------------------------------------------- | ---------- | ------------- |
| **Tổng Quan & Lộ Trình**                      | README.md                                   | ✅          | Toàn Diện    |
| **Bảng Chỉ Mục Đầy Đủ**                      | INDEX.md                                    | ✅          | Toàn Diện    |
| **RDS Fundamentals — Tổng Quan**              | 01-rds-fundamentals/README.md               | ✅          | Toàn Diện    |
| **RDS — Engine Types**                        | 01-rds-fundamentals/1-engine-types.md       | ✅          | Toàn Diện    |
| **RDS — Instance & Storage**                  | 01-rds-fundamentals/2-instance-storage.md   | ✅          | Toàn Diện    |
| **RDS — Multi-AZ Deployment**                 | 01-rds-fundamentals/3-multi-az.md           | ✅          | Toàn Diện    |
| **RDS — Read Replicas**                       | 01-rds-fundamentals/4-read-replicas.md      | ✅          | Toàn Diện    |
| **RDS — Parameter Groups & Option Groups**    | 01-rds-fundamentals/5-parameter-groups.md   | ✅          | Toàn Diện    |
| **RDS — RDS Proxy**                           | 01-rds-fundamentals/6-rds-proxy.md          | ✅          | Toàn Diện    |
| **Aurora — Tổng Quan & Kiến Trúc**           | 02-aurora/README.md                         | ✅          | Toàn Diện    |
| **Aurora — Shared Storage & Cluster Endpoints** | 02-aurora/1-aurora-architecture.md       | ✅          | Toàn Diện    |
| **Aurora — Serverless v2 & Auto-Scaling**     | 02-aurora/2-aurora-serverless.md            | ✅          | Toàn Diện    |
| **Aurora — Global Database**                  | 02-aurora/3-aurora-global.md                | ✅          | Toàn Diện    |
| **Aurora — Aurora vs RDS Trade-offs**         | 02-aurora/4-aurora-vs-rds.md                | ✅          | Toàn Diện    |
| **Aurora — High Availability & Failover**     | 02-aurora/5-aurora-ha-failover.md           | ✅          | Toàn Diện    |
| **DynamoDB — Tổng Quan & Data Model**         | 03-dynamodb/README.md                       | ✅          | Toàn Diện    |
| **DynamoDB — Tables, Partition Key, Sort Key**| 03-dynamodb/1-data-model.md                 | ✅          | Toàn Diện    |
| **DynamoDB — On-Demand vs Provisioned, Auto Scaling** | 03-dynamodb/2-capacity-modes.md     | ✅          | Toàn Diện    |
| **DynamoDB — GSI, LSI & Index Design**        | 03-dynamodb/3-indexes.md                    | ✅          | Toàn Diện    |
| **DynamoDB — Streams & Lambda Integration**   | 03-dynamodb/4-streams-lambda.md             | ✅          | Toàn Diện    |
| **DynamoDB — Transactions & ACID**            | 03-dynamodb/5-transactions-acid.md          | ✅          | Toàn Diện    |
| **DynamoDB — DAX (DynamoDB Accelerator)**     | 03-dynamodb/6-dax.md                        | ✅          | Toàn Diện    |
| **DynamoDB — Access Patterns & Single-Table** | 03-dynamodb/7-access-patterns.md            | ✅          | Toàn Diện    |
| **ElastiCache — Tổng Quan & Kiến Trúc**      | 04-elasticache/README.md                    | ✅          | Toàn Diện    |
| **ElastiCache — Redis vs Memcached**          | 04-elasticache/1-redis-vs-memcached.md      | ✅          | Toàn Diện    |
| **ElastiCache — Redis Cluster & Replication** | 04-elasticache/2-redis-cluster.md           | ✅          | Toàn Diện    |
| **ElastiCache — Caching Strategies**          | 04-elasticache/3-caching-strategies.md      | ✅          | Toàn Diện    |
| **ElastiCache — Persistence (RDB & AOF)**     | 04-elasticache/4-persistence.md             | ✅          | Toàn Diện    |
| **ElastiCache — Security**                    | 04-elasticache/5-security.md                | ✅          | Toàn Diện    |
| **HA & Backup — Tổng Quan & Kiến Trúc**      | 05-ha-backup/README.md                      | ✅          | Toàn Diện    |
| **HA & Backup — RPO & RTO**                   | 05-ha-backup/1-rpo-rto.md                   | ✅          | Toàn Diện    |
| **HA & Backup — Automated Backups**           | 05-ha-backup/2-automated-backups.md         | ✅          | Toàn Diện    |
| **HA & Backup — Manual Snapshots**            | 05-ha-backup/3-snapshots.md                 | ✅          | Toàn Diện    |
| **HA & Backup — Point-in-Time Recovery**      | 05-ha-backup/4-pitr.md                      | ✅          | Toàn Diện    |
| **HA & Backup — Disaster Recovery**           | 05-ha-backup/5-disaster-recovery.md         | ✅          | Toàn Diện    |
| **Security — Tổng Quan & Defense in Depth**   | 06-security/README.md                       | ✅          | Toàn Diện    |
| **Security — VPC & Security Groups**          | 06-security/1-vpc-security-groups.md        | ✅          | Toàn Diện    |
| **Security — IAM Authentication**             | 06-security/2-iam-authentication.md         | ✅          | Toàn Diện    |
| **Security — KMS Encryption**                 | 06-security/3-encryption.md                 | ✅          | Toàn Diện    |
| **Security — Secrets Manager & Parameter Store** | 06-security/4-secrets-management.md      | ✅          | Toàn Diện    |
| **Security — Audit & Compliance**             | 06-security/5-audit-compliance.md           | ✅          | Toàn Diện    |
| **Performance Tuning — Tổng Quan & Methodology** | 07-performance-tuning/README.md          | ✅          | Toàn Diện    |
| **Performance Tuning — RDS Performance Insights** | 07-performance-tuning/1-performance-insights.md | ✅   | Toàn Diện    |
| **Performance Tuning — CloudWatch & Enhanced Monitoring** | 07-performance-tuning/2-cloudwatch-metrics.md | ✅ | Toàn Diện    |
| **Performance Tuning — Slow Query & EXPLAIN** | 07-performance-tuning/3-slow-query-analysis.md  | ✅       | Toàn Diện    |
| **Performance Tuning — Connection Pooling & RDS Proxy** | 07-performance-tuning/4-connection-pooling.md | ✅ | Toàn Diện    |
| **Performance Tuning — DynamoDB Hot Partitions & DAX** | 07-performance-tuning/5-dynamodb-performance.md | ✅ | Toàn Diện   |
| **Performance Tuning — Index & GSI Optimization** | 07-performance-tuning/6-index-optimization.md  | ✅       | Toàn Diện    |
| **Migration — Tổng Quan & Quy Trình**             | 08-migration/README.md                          | ✅       | Toàn Diện    |
| **Migration — AWS DMS Architecture & Task Types** | 08-migration/1-dms-overview.md                  | ✅       | Toàn Diện    |
| **Migration — SCT & Heterogeneous Migration**     | 08-migration/2-sct.md                           | ✅       | Toàn Diện    |
| **Migration — CDC & Zero-Downtime Online Migration** | 08-migration/3-cdc-online-migration.md       | ✅       | Toàn Diện    |
| **Migration — Chiến Lược Di Chuyển**              | 08-migration/4-migration-strategies.md          | ✅       | Toàn Diện    |
| **Migration — Cutover Runbook & Rollback**        | 08-migration/5-cutover-runbook.md               | ✅       | Toàn Diện    |
| **Monitoring — Tổng Quan & Kiến Trúc**            | 09-monitoring/README.md                         | ✅       | Toàn Diện    |
| **Monitoring — CloudWatch Dashboards & Key Metrics** | 09-monitoring/1-cloudwatch-dashboards.md      | ✅       | Toàn Diện    |
| **Monitoring — Alerting Strategy & SLO**          | 09-monitoring/2-alerting-strategy.md            | ✅       | Toàn Diện    |
| **Monitoring — RDS Enhanced Monitoring**          | 09-monitoring/3-rds-enhanced-monitoring.md      | ✅       | Toàn Diện    |
| **Monitoring — DynamoDB CloudWatch & Contributor Insights** | 09-monitoring/4-dynamodb-monitoring.md | ✅       | Toàn Diện    |
| **Monitoring — Database Activity Streams**        | 09-monitoring/5-activity-streams.md             | ✅       | Toàn Diện    |
| **Advanced — Tổng Quan Chủ Đề Nâng Cao**         | 10-advanced/README.md                           | ✅       | Toàn Diện    |
| **Advanced — Redshift Data Warehouse**            | 10-advanced/1-redshift.md                       | ✅       | Toàn Diện    |
| **Advanced — Aurora Global Database Nâng Cao**    | 10-advanced/2-aurora-global-advanced.md         | ✅       | Toàn Diện    |
| **Advanced — DynamoDB Global Tables**             | 10-advanced/3-dynamodb-global-tables.md         | ✅       | Toàn Diện    |
| **Advanced — Neptune Graph Database**             | 10-advanced/4-neptune.md                        | ✅       | Toàn Diện    |
| **Advanced — DocumentDB MongoDB Compatible**      | 10-advanced/5-documentdb.md                     | ✅       | Toàn Diện    |
| **Advanced — Timestream Time-Series Database**    | 10-advanced/6-timestream.md                     | ✅       | Toàn Diện    |
| **Cost Optimization — Tổng Quan & Framework**     | 11-cost-optimization/README.md                  | ✅       | Toàn Diện    |
| **Cost Optimization — RDS Pricing & Reserved Instances** | 11-cost-optimization/1-rds-pricing.md      | ✅       | Toàn Diện    |
| **Cost Optimization — Aurora I/O & Serverless**   | 11-cost-optimization/2-aurora-cost.md           | ✅       | Toàn Diện    |
| **Cost Optimization — DynamoDB On-Demand vs Provisioned** | 11-cost-optimization/3-dynamodb-cost.md  | ✅       | Toàn Diện    |
| **Cost Optimization — ElastiCache Reserved Nodes**| 11-cost-optimization/4-elasticache-cost.md      | ✅       | Toàn Diện    |
| **Cost Optimization — Right-sizing & Storage**    | 11-cost-optimization/5-rightsizing.md           | ✅       | Toàn Diện    |
| **Interview Prep — Tổng Quan & Framework**        | 12-interview-prep/README.md                     | ✅       | Toàn Diện    |
| **Interview Prep — Top 20 Câu Hỏi & Câu Trả Lời**| 12-interview-prep/INTERVIEW_GUIDE.md            | ✅       | Toàn Diện    |
| **Interview Prep — System Design Scenarios**      | 12-interview-prep/system-design-scenarios.md    | ✅       | Toàn Diện    |
| **Interview Prep — Trade-off Discussions**        | 12-interview-prep/trade-off-discussions.md      | ✅       | Toàn Diện    |
| **Interview Prep — STAR Stories**                 | 12-interview-prep/star-stories.md               | ✅       | Toàn Diện    |

---

## 🎯 Cần Tạo Tiếp (Theo Thứ Tự Ưu Tiên)

### Ưu Tiên Cao — Kỹ Năng Core

- [x] `01-rds-fundamentals/README.md` — RDS overview, engine types, Multi-AZ, Read Replicas ✅ **Hoàn thành**
- [x] `02-aurora/README.md` — Aurora architecture, cluster, serverless, global database ✅ **Hoàn thành**
- [x] `03-dynamodb/README.md` — DynamoDB data model, capacity modes, indexes ✅ **Hoàn thành**
- [x] `03-dynamodb/1-data-model.md` — Tables, Partition Key, Sort Key, Attributes ✅ **Hoàn thành**
- [x] `03-dynamodb/2-capacity-modes.md` — On-Demand vs Provisioned, Auto Scaling, Throttling ✅ **Hoàn thành**
- [x] `03-dynamodb/3-indexes.md` — GSI, LSI, Sparse Index, GSI Overloading ✅ **Hoàn thành**
- [x] `03-dynamodb/4-streams-lambda.md` — DynamoDB Streams, Lambda Integration ✅ **Hoàn thành**
- [x] `03-dynamodb/5-transactions-acid.md` — Transactions, ACID, TransactWrite/Get ✅ **Hoàn thành**
- [x] `03-dynamodb/6-dax.md` — DAX, Item Cache, Query Cache ✅ **Hoàn thành**
- [x] `03-dynamodb/7-access-patterns.md` — Single-Table Design, Access Patterns ✅ **Hoàn thành**
- [x] `04-elasticache/README.md` — ElastiCache overview, Redis vs Memcached ✅ **Hoàn thành**
- [x] `04-elasticache/1-redis-vs-memcached.md` — So sánh Redis & Memcached ✅ **Hoàn thành**
- [x] `04-elasticache/2-redis-cluster.md` — Cluster Mode, Replication, Sharding ✅ **Hoàn thành**
- [x] `04-elasticache/3-caching-strategies.md` — Lazy Loading, Write-Through, TTL ✅ **Hoàn thành**
- [x] `04-elasticache/4-persistence.md` — RDB, AOF, Backup/Restore ✅ **Hoàn thành**
- [x] `04-elasticache/5-security.md` — VPC, TLS, KMS, Auth, IAM ✅ **Hoàn thành**
- [x] `05-ha-backup/README.md` — HA strategies, backup, PITR, disaster recovery ✅ **Hoàn thành**
- [x] `05-ha-backup/1-rpo-rto.md` — RPO, RTO, thiết kế chiến lược ✅ **Hoàn thành**
- [x] `05-ha-backup/2-automated-backups.md` — Automated Backups, retention policy ✅ **Hoàn thành**
- [x] `05-ha-backup/3-snapshots.md` — Manual Snapshots, cross-region copy ✅ **Hoàn thành**
- [x] `05-ha-backup/4-pitr.md` — Point-in-Time Recovery, runbook khẩn cấp ✅ **Hoàn thành**
- [x] `05-ha-backup/5-disaster-recovery.md` — DR strategies, multi-region failover ✅ **Hoàn thành**
- [x] `12-interview-prep/README.md` — Tổng quan, framework, lộ trình 2 tuần ✅ **Hoàn thành**
- [x] `12-interview-prep/INTERVIEW_GUIDE.md` — Top 20 câu hỏi phỏng vấn ✅ **Hoàn thành**

### Ưu Tiên Trung Bình — Kỹ Năng Nâng Cao
- [x] `06-security/README.md` — Tổng quan, Defense in Depth ✅ **Hoàn thành**
- [x] `06-security/1-vpc-security-groups.md` — VPC, private subnets, Security Groups, NACLs ✅ **Hoàn thành**
- [x] `06-security/2-iam-authentication.md` — IAM Roles, IAM Auth cho RDS, Least Privilege ✅ **Hoàn thành**
- [x] `06-security/3-encryption.md` — KMS, Encryption at Rest & in Transit, TLS ✅ **Hoàn thành**
- [x] `06-security/4-secrets-management.md` — Secrets Manager, Parameter Store, auto rotation ✅ **Hoàn thành**
- [x] `06-security/5-audit-compliance.md` — CloudTrail, Activity Streams, PCI/HIPAA/GDPR ✅ **Hoàn thành**
- [x] `07-performance-tuning/README.md` — Performance Insights, slow query, RDS Proxy ✅ **Hoàn thành**
- [x] `07-performance-tuning/1-performance-insights.md` — RDS Performance Insights, AAS, wait events ✅ **Hoàn thành**
- [x] `07-performance-tuning/2-cloudwatch-metrics.md` — Key metrics, Enhanced Monitoring ✅ **Hoàn thành**
- [x] `07-performance-tuning/3-slow-query-analysis.md` — Slow query log, EXPLAIN plans ✅ **Hoàn thành**
- [x] `07-performance-tuning/4-connection-pooling.md` — RDS Proxy, pgBouncer, connection limits ✅ **Hoàn thành**
- [x] `07-performance-tuning/5-dynamodb-performance.md` — Hot partitions, throttling, DAX ✅ **Hoàn thành**
- [x] `07-performance-tuning/6-index-optimization.md` — Index design, GSI optimization ✅ **Hoàn thành**
- [x] `08-migration/README.md` — DMS, SCT, CDC, cutover planning ✅ **Hoàn thành**
- [x] `08-migration/1-dms-overview.md` — AWS DMS architecture, task types ✅ **Hoàn thành**
- [x] `08-migration/2-sct.md` — Schema Conversion Tool, heterogeneous migration ✅ **Hoàn thành**
- [x] `08-migration/3-cdc-online-migration.md` — CDC, zero-downtime techniques ✅ **Hoàn thành**
- [x] `08-migration/4-migration-strategies.md` — Lift-and-shift, re-platform, re-architect ✅ **Hoàn thành**
- [x] `08-migration/5-cutover-runbook.md` — Cutover planning, rollback, validation ✅ **Hoàn thành**

### Ưu Tiên Thấp Hơn — Tham Khảo

- [x] `09-monitoring/README.md` — Monitoring & Observability overview ✅ **Hoàn thành**
- [x] `09-monitoring/1-cloudwatch-dashboards.md` — CloudWatch Metrics, key metrics, dashboards ✅ **Hoàn thành**
- [x] `09-monitoring/2-alerting-strategy.md` — SLO, alerting thresholds, alert routing ✅ **Hoàn thành**
- [x] `09-monitoring/3-rds-enhanced-monitoring.md` — OS-level metrics, process monitoring ✅ **Hoàn thành**
- [x] `09-monitoring/4-dynamodb-monitoring.md` — DynamoDB alarms, Contributor Insights ✅ **Hoàn thành**
- [x] `09-monitoring/5-activity-streams.md` — Activity Streams, SIEM, compliance ✅ **Hoàn thành**
- [x] `10-advanced/README.md` — Tổng quan advanced topics ✅ **Hoàn thành**
- [x] `10-advanced/1-redshift.md` — Redshift architecture, distribution keys, Spectrum ✅ **Hoàn thành**
- [x] `10-advanced/2-aurora-global-advanced.md` — Aurora Global, multi-region active-active ✅ **Hoàn thành**
- [x] `10-advanced/3-dynamodb-global-tables.md` — Global Tables, conflict resolution ✅ **Hoàn thành**
- [x] `10-advanced/4-neptune.md` — Neptune Graph Database, Gremlin, SPARQL ✅ **Hoàn thành**
- [x] `10-advanced/5-documentdb.md` — DocumentDB, MongoDB migration ✅ **Hoàn thành**
- [x] `10-advanced/6-timestream.md` — Timestream, IoT, time-series ✅ **Hoàn thành**
- [x] `11-cost-optimization/README.md` — Framework tối ưu chi phí, quick wins ✅ **Hoàn thành**
- [x] `11-cost-optimization/1-rds-pricing.md` — RDS pricing, Reserved Instances, storage tiers ✅ **Hoàn thành**
- [x] `11-cost-optimization/2-aurora-cost.md` — Aurora I/O-Optimized, Serverless v2, Aurora vs RDS cost ✅ **Hoàn thành**
- [x] `11-cost-optimization/3-dynamodb-cost.md` — On-Demand vs Provisioned, TTL, DAX ROI, GSI cost ✅ **Hoàn thành**
- [x] `11-cost-optimization/4-elasticache-cost.md` — Reserved Nodes, right-sizing, Serverless ✅ **Hoàn thành**
- [x] `11-cost-optimization/5-rightsizing.md` — Instance sizing, storage optimization, automation ✅ **Hoàn thành**
- [x] `12-interview-prep/system-design-scenarios.md` — 5 kịch bản thiết kế hệ thống ✅ **Hoàn thành**
- [x] `12-interview-prep/trade-off-discussions.md` — SQL vs NoSQL, caching, consistency ✅ **Hoàn thành**
- [x] `12-interview-prep/star-stories.md` — 10 mẫu câu chuyện STAR ✅ **Hoàn thành**

---

## 🚀 Cách Sử Dụng Knowledge Base Này

### Cho Tự Học

```
1. Bắt đầu với README.md
2. Chọn Lộ Trình Học (Beginner/Intermediate/Advanced)
3. Học từng section theo thứ tự
4. Thực hành trên AWS Console (dùng Free Tier)
5. Xây dựng portfolio project kết hợp nhiều dịch vụ
```

### Cho Chuẩn Bị Phỏng Vấn

```
1. Đọc 12-interview-prep/INTERVIEW_GUIDE.md
2. Tập trung vào RDS + Aurora + DynamoDB (luôn được hỏi)
3. Học 05-ha-backup/ (backup, HA luôn xuất hiện trong phỏng vấn)
4. Chuẩn bị trade-off discussions (SQL vs NoSQL, RDS vs DynamoDB)
5. Luyện tập system design với database considerations
6. Chuẩn bị câu chuyện STAR về database incidents
```

### Cho Vai Trò Thực Tế

```
Dùng làm tài liệu tham khảo:
- Vấn đề hiệu năng:   Xem 07-performance-tuning/
- Sự cố & giám sát:   Xem 09-monitoring/ và 05-ha-backup/
- Di chuyển database: Xem 08-migration/
- Bảo mật:            Xem 06-security/
- Tối ưu chi phí:     Xem 11-cost-optimization/
```

### Cho Thiết Kế Hệ Thống

```
1. Đọc README.md — phần Tổng Quan Các Dịch Vụ để chọn DB phù hợp
2. Tham khảo 02-aurora/ cho high-traffic OLTP
3. Tham khảo 03-dynamodb/ cho key-value/document workloads
4. Dùng 05-ha-backup/ để thiết kế DR strategy
5. Dùng 04-elasticache/ để thêm caching layer
```

---

## 📊 Ước Tính Thời Gian Học

| Section                                       | Thời Gian   | Độ Khó  | Ưu Tiên |
| --------------------------------------------- | ----------- | ------- | ------- |
| RDS Fundamentals — Nền Tảng RDS               | 4-6 giờ     | ⭐       | Bắt buộc |
| Aurora — Cơ Sở Dữ Liệu Đám Mây               | 4-6 giờ     | ⭐⭐     | Bắt buộc |
| DynamoDB — NoSQL Serverless                   | 8-10 giờ    | ⭐⭐⭐   | Bắt buộc |
| ElastiCache — Bộ Nhớ Đệm                      | 3-4 giờ     | ⭐⭐     | Bắt buộc |
| HA & Backup — Sẵn Sàng Cao & Sao Lưu         | 4-6 giờ     | ⭐⭐     | Bắt buộc |
| Security — Bảo Mật                            | 4-6 giờ     | ⭐⭐     | Bắt buộc |
| Performance Tuning — Tối Ưu Hiệu Năng        | 6-8 giờ     | ⭐⭐⭐   | Nên Có  |
| Migration — Di Chuyển Database               | 4-6 giờ     | ⭐⭐     | Nên Có  |
| Monitoring — Giám Sát                         | 3-4 giờ     | ⭐⭐     | Nên Có  |
| Advanced — Nâng Cao (Redshift, Neptune...)    | 10-15 giờ   | ⭐⭐⭐   | Tốt Nếu Có |
| Cost Optimization — Tối Ưu Chi Phí           | 2-3 giờ     | ⭐       | Tốt Nếu Có |

**Tổng thời gian: 52-74 giờ để nắm vững AWS Database Services**

---

## 🎓 Mức Độ Kỹ Năng

### Beginner — Mới Bắt Đầu (0-1 năm kinh nghiệm)

- [ ] Phân biệt RDS, Aurora, DynamoDB
- [ ] Hiểu Multi-AZ (Đa Vùng Sẵn Sàng) và Read Replicas (Bản Sao Đọc)
- [ ] Cơ bản DynamoDB — table, partition key, sort key
- [ ] Cấu hình automated backup (sao lưu tự động)
- [ ] Bảo mật cơ bản với VPC và Security Groups

**Thời gian đạt mức này:** 1-2 tháng

### Intermediate — Trung Cấp (1-3 năm kinh nghiệm)

- [ ] Thiết kế Aurora Cluster với HA
- [ ] DynamoDB access patterns & GSI design
- [ ] ElastiCache Redis cho caching & sessions
- [ ] RDS Performance Insights & slow query analysis
- [ ] Database migration với DMS
- [ ] Security hardening — KMS, Secrets Manager, IAM

**Thời gian để nâng cấp:** 2-3 tháng

### Advanced — Nâng Cao (3-5+ năm kinh nghiệm)

- [ ] Multi-region active-active architecture (Kiến Trúc Đa Vùng Chủ-Chủ)
- [ ] DynamoDB Global Tables & conflict resolution
- [ ] Redshift architecture & query optimization
- [ ] Capacity planning (Lập Kế Hoạch Năng Lực) & cost modeling
- [ ] Compliance frameworks (PCI-DSS, HIPAA, GDPR)
- [ ] Zero-downtime migration strategies

**Thời gian:** Học liên tục

---

## 🔗 Điều Hướng Nhanh

| Cần Gì                                  | Vị Trí                                                                             |
| --------------------------------------- | ---------------------------------------------------------------------------------- |
| Tổng quan nhanh                         | [README.md](README.md)                                                             |
| RDS basics                              | [01-rds-fundamentals/README.md](01-rds-fundamentals/README.md)                     |
| Aurora deep dive                        | [02-aurora/README.md](02-aurora/README.md)                                         |
| DynamoDB patterns                       | [03-dynamodb/README.md](03-dynamodb/README.md)                                     |
| ElastiCache Redis                       | [04-elasticache/README.md](04-elasticache/README.md)                               |
| Backup & disaster recovery              | [05-ha-backup/README.md](05-ha-backup/README.md)                                   |
| Bảo mật database                        | [06-security/README.md](06-security/README.md)                                     |
| Tối ưu hiệu năng                        | [07-performance-tuning/README.md](07-performance-tuning/README.md)                 |
| Di chuyển database                      | [08-migration/README.md](08-migration/README.md)                                   |
| Câu hỏi phỏng vấn                       | [12-interview-prep/INTERVIEW_GUIDE.md](12-interview-prep/INTERVIEW_GUIDE.md)       |

---

## 📈 Theo Dõi Tiến Độ Học Tập

Sao chép và theo dõi tiến độ của bạn:

```markdown
## AWS Database Services — Tiến Độ Hoàn Thành

### Giai Đoạn 1: Nền Tảng (Tuần 1-2)

- [ ] RDS engine types & storage
- [ ] Multi-AZ vs Read Replicas
- [ ] DynamoDB table design cơ bản
- [ ] Automated backups & snapshots
- [ ] VPC & security groups

### Giai Đoạn 2: Kỹ Năng Cốt Lõi (Tuần 3-6)

- [ ] Aurora cluster & serverless
- [ ] DynamoDB GSI/LSI & access patterns
- [ ] ElastiCache Redis caching strategies
- [ ] RDS Performance Insights
- [ ] KMS encryption & Secrets Manager
- [ ] DMS database migration

### Giai Đoạn 3: Vận Hành Nâng Cao (Tuần 7-10)

- [ ] Multi-region HA design
- [ ] DynamoDB Global Tables
- [ ] Redshift basics
- [ ] Cost optimization strategies
- [ ] Monitoring & alerting setup

### Giai Đoạn 4: Chuyên Sâu (Tuần 11+)

- [ ] Neptune & DocumentDB
- [ ] Aurora Global Database advanced
- [ ] Compliance frameworks
- [ ] System design với database
- [ ] Mock interviews
```

---

## 🎯 Tiêu Chí Thành Công

Sau khi hoàn thành knowledge base này, bạn có thể:

### ✅ Năng Lực Nền Tảng

- [ ] Giải thích sự khác biệt RDS, Aurora, DynamoDB, ElastiCache không cần ghi chú
- [ ] Thiết kế backup strategy cho RPO/RTO đã cho
- [ ] Lựa chọn database phù hợp cho từng use case
- [ ] Đọc và phân tích CloudWatch metrics cơ bản

### ✅ Năng Lực Vận Hành

- [ ] Troubleshoot slow queries với Performance Insights
- [ ] Thiết lập Multi-AZ và Read Replicas đúng cách
- [ ] Thực hiện database migration với DMS không có downtime
- [ ] Implement security best practices (KMS, VPC, IAM)

### ✅ Sẵn Sàng Phỏng Vấn

- [ ] Trả lời 20 câu hỏi AWS Database tự tin
- [ ] Kể 2-3 câu chuyện database incident (STAR format)
- [ ] Thiết kế hệ thống có database considerations
- [ ] Thảo luận trade-offs giữa các dịch vụ
- [ ] Hiểu sâu ít nhất 1 dịch vụ (RDS/Aurora hoặc DynamoDB)

---

## 🚀 Bước Tiếp Theo

### Ngay Bây Giờ (Tuần Này)

1. Đọc README.md đầy đủ
2. Chọn lộ trình học phù hợp với kinh nghiệm của bạn
3. Xem qua `01-rds-fundamentals/README.md`
4. Tạo tài khoản AWS Free Tier nếu chưa có

### Ngắn Hạn (2 Tuần Tiếp)

1. Hoàn thành `01-rds-fundamentals/` và `02-aurora/`
2. Bắt đầu học DynamoDB cơ bản — tạo table thử nghiệm
3. Thực hành trên AWS Console — tạo RDS instance (Free Tier)

### Trung Hạn (4 Tuần Tiếp)

1. Hoàn thành tất cả core sections (01-06)
2. Deep dive một dịch vụ (Aurora hoặc DynamoDB)
3. Chuẩn bị 2-3 câu chuyện database incident
4. Thực hiện mock interview với đồng nghiệp

### Dài Hạn (3 Tháng Tiếp)

1. Nắm vững một dịch vụ hoàn toàn
2. Hiểu trade-offs giữa tất cả dịch vụ
3. Xây dựng portfolio project dùng nhiều AWS database services
4. Bắt đầu ứng tuyển vào vai trò liên quan đến AWS

---

## 💡 Mẹo Học Hiệu Quả

1. **Học bằng thực hành:** Đừng chỉ đọc — tạo thật sự trên AWS Console, làm hỏng, rồi fix
2. **So sánh liên tục:** Sau mỗi dịch vụ mới, so sánh với dịch vụ đã biết
3. **Hiểu WHY, không chỉ WHAT:** Tại sao Aurora nhanh hơn RDS? Tại sao DynamoDB có millisecond latency?
4. **Chia sẻ kiến thức:** Giảng lại cho người khác giúp consolidate learning
5. **Theo dõi AWS updates:** AWS release tính năng mới thường xuyên — đọc AWS Blog hàng tuần
6. **Biết cost trước khi dùng:** Luôn ước tính chi phí trước khi tạo tài nguyên
7. **Test backups:** Restore test là điều bắt buộc — backup không được test là backup chưa tồn tại
8. **Ghi lại incidents:** Mọi sự cố đều là cơ hội học tập

---

## 📞 Đóng Góp

Tìm thấy lỗi? Muốn thêm nội dung?

Đây là tài liệu sống. Chào đón đóng góp:

- [ ] Sửa lỗi trong nội dung hiện có
- [ ] Thêm section cho chủ đề chưa được đề cập
- [ ] Ví dụ thực tế từ kinh nghiệm của bạn
- [ ] Giải thích rõ hơn các khái niệm phức tạp
- [ ] Hướng dẫn dịch vụ AWS mới

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 2.2
**Trạng Thái:** ✅ README & INDEX Hoàn Thành | ✅ 01-rds-fundamentals Hoàn Thành | ✅ 02-aurora Hoàn Thành | ✅ 03-dynamodb Hoàn Thành | ✅ 04-elasticache Hoàn Thành | ✅ 05-ha-backup Hoàn Thành | ✅ 06-security Hoàn Thành | ✅ 07-performance-tuning Hoàn Thành | ✅ 08-migration Hoàn Thành | ✅ 09-monitoring Hoàn Thành | ✅ 10-advanced Hoàn Thành | ✅ 11-cost-optimization Hoàn Thành | ✅ 12-interview-prep Hoàn Thành
