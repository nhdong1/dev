# SRR — Same-Region Replication — Sao Chép Cùng Vùng

> SRR (Same-Region Replication — Sao Chép Cùng Vùng) tự động sao chép object giữa hai S3 bucket trong cùng một AWS Region. Khác với CRR, SRR không mất phí data transfer và thường dùng cho log aggregation (tổng hợp nhật ký), compliance (tuân thủ), và test/prod isolation (phân tách môi trường).

## 📚 Mục Lục

1. [SRR Là Gì Và Khác CRR Ở Đâu?](#1-srr-là-gì-và-khác-crr-ở-đâu)
2. [Điều Kiện Và Cấu Hình](#2-điều-kiện-và-cấu-hình)
3. [Use Cases Thực Tế](#3-use-cases-thực-tế)
4. [Multi-Source Replication — Nhiều Nguồn Một Đích](#4-multi-source-replication--nhiều-nguồn-một-đích)
5. [Chi Phí SRR](#5-chi-phí-srr)
6. [Hạn Chế Quan Trọng](#6-hạn-chế-quan-trọng)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. SRR Là Gì Và Khác CRR Ở Đâu?

### So Sánh Tổng Quan

| Đặc Điểm | SRR | CRR |
| --------- | --- | --- |
| Vị trí đích | Cùng Region | Khác Region |
| Data transfer fee | **Không có** | $0.02/GB |
| Latency replication | Thấp hơn (trong cùng Region) | Cao hơn (xuyên vùng) |
| Mục đích chính | Log aggregation, compliance, test/prod | DR, low-latency toàn cầu |
| Chủ quyền dữ liệu | Dữ liệu không rời khỏi Region | Dữ liệu chuyển Region |
| RTC hỗ trợ | Có | Có |

### Luồng Dữ Liệu SRR

```
                    AWS Region: us-east-1
   ┌──────────────────────────────────────────────────┐
   │                                                  │
   │   Source Bucket A          Source Bucket B       │
   │   (Bucket Nguồn A)         (Bucket Nguồn B)      │
   │   app-logs-team-alpha      app-logs-team-beta     │
   │         │                        │               │
   │         └──────────┬─────────────┘               │
   │                    │ SRR                         │
   │                    ▼                             │
   │          Central Log Bucket                      │
   │          (Bucket Nhật Ký Trung Tâm)              │
   │          company-central-logs                    │
   │                                                  │
   └──────────────────────────────────────────────────┘
```

---

## 2. Điều Kiện Và Cấu Hình

### Điều Kiện Bắt Buộc

```
✅ Versioning bật ở CẢ HAI bucket (nguồn và đích)
✅ Hai bucket phải ở CÙNG AWS Region
✅ IAM Role đủ quyền (giống CRR)
✅ Bucket đích phải là bucket khác (không thể replicate vào chính nó)
```

### Cấu Hình Qua AWS CLI

```bash
# Bật versioning ở cả hai bucket
aws s3api put-bucket-versioning \
  --bucket source-logs-bucket \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-versioning \
  --bucket central-logs-bucket \
  --versioning-configuration Status=Enabled

# Áp dụng replication rule (giống CRR, chỉ khác destination region)
aws s3api put-bucket-replication \
  --bucket source-logs-bucket \
  --replication-configuration file://srr-config.json
```

### File srr-config.json

```json
{
  "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
  "Rules": [
    {
      "ID": "AggregateLogsToCenter",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "2026/"
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::central-logs-bucket",
        "StorageClass": "STANDARD_IA",
        "AccessControlTranslation": {
          "Owner": "Destination"
        },
        "Account": "123456789012"
      },
      "DeleteMarkerReplication": {
        "Status": "Disabled"
      }
    }
  ]
}
```

---

## 3. Use Cases Thực Tế

### Use Case 1: Log Aggregation — Tổng Hợp Nhật Ký

```
Bài toán: 5 team khác nhau dùng 5 bucket khác nhau để lưu log ứng dụng.
Cần một nơi trung tâm để chạy Athena queries và phân tích tổng hợp.

Thiết lập:
- 5 source buckets: team-alpha-logs, team-beta-logs, ... (cùng us-east-1)
- 1 destination bucket: company-analytics-logs
- SRR rule từ mỗi bucket nguồn sang bucket đích
- Prefix giúp phân biệt nguồn: /team=alpha/, /team=beta/, ...

Lợi ích:
- Không mất phí data transfer (cùng Region)
- Athena chỉ cần một data source duy nhất
- Team ownership không thay đổi ở bucket gốc
```

### Use Case 2: Test/Production Data Isolation

```
Bài toán: Developer cần môi trường test với dữ liệu gần giống production
nhưng không được truy cập trực tiếp vào production bucket.

Thiết lập:
- production-data-bucket (us-east-1): Bucket production thật
- test-replica-bucket (us-east-1): Bucket replica cho team dev
- SRR rule: production → test-replica
- IAM: Dev team chỉ có quyền đọc/ghi test-replica-bucket
- Object Lock hoặc bucket policy: Không cho phép write từ developer vào production

Lợi ích:
- Dev có dữ liệu thật để test (ẩn danh hóa data nhạy cảm trước)
- Production không bị ảnh hưởng dù dev thao tác sai
```

### Use Case 3: Compliance Backup — Backup Tuân Thủ Cùng Vùng

```
Bài toán: Quy định nội bộ yêu cầu mọi dữ liệu phải có bản backup độc lập,
nhưng không cần backup sang region khác (dữ liệu nhạy cảm không được rời Region).

Thiết lập:
- primary-data-bucket: Bucket chính
- compliance-backup-bucket: Bucket backup với:
  - Bucket policy ngăn xóa bởi application user
  - Object Lock Compliance Mode cho dữ liệu tài chính
  - SRR rule từ primary → compliance-backup

Lợi ích:
- Backup tự động, không cần cron job
- Dữ liệu ở trong cùng Region (tuân thủ quy định)
- Tách biệt IAM: Application chỉ ghi primary, không ghi được backup
```

### Use Case 4: Anonymization Pipeline — Quy Trình Ẩn Danh Hóa

```
Bài toán: GDPR yêu cầu ẩn danh hóa PII (Personally Identifiable Information —
Thông Tin Nhận Dạng Cá Nhân) trong log trước khi chia sẻ với analytics team.

Thiết lập:
- raw-data-bucket: Nhận dữ liệu thô với PII
- SRR rule trigger S3 Event Notification → Lambda
- Lambda: Ẩn danh hóa PII, ghi vào anonymized-bucket
- Analytics team chỉ truy cập anonymized-bucket

Luồng xử lý:
Upload PII data → raw-bucket → SRR → Event → Lambda anonymize → anonymized-bucket
                                                                       ↑
                                                               Analytics team đọc
```

---

## 4. Multi-Source Replication — Nhiều Nguồn Một Đích

S3 hỗ trợ replicate từ nhiều bucket nguồn vào một bucket đích. Tuy nhiên, phải cấu hình rule riêng ở từng bucket nguồn.

```
bucket-region-a  ──[SRR rule]──┐
bucket-region-b  ──[SRR rule]──┤──▶ central-bucket
bucket-region-c  ──[SRR rule]──┘
```

### Tránh Xung Đột Prefix

Khi nhiều bucket replicate vào một bucket đích, dùng prefix để tránh ghi đè nhau:

```
bucket-A ghi vào prefix: source=team-alpha/
bucket-B ghi vào prefix: source=team-beta/
bucket-C ghi vào prefix: source=team-gamma/
```

Cách thực hiện: Application khi write vào bucket nguồn luôn dùng prefix phân biệt, SRR giữ nguyên key structure.

### Vòng Lặp Vô Hạn (Infinite Loop) — Cảnh Báo

```
⚠️ NGUY HIỂM: Nếu cấu hình cả A→B và B→A replication, sẽ không xảy ra infinite loop vì:
S3 tự động đánh dấu object đã replicate (ReplicationStatus = REPLICA) và
KHÔNG replicate các object có status này sang đích khác.

→ An toàn: Có thể cấu hình bidirectional (hai chiều) mà không lo vòng lặp.
```

---

## 5. Chi Phí SRR

### Tiết Kiệm So Với CRR

| Thành Phần Chi Phí | SRR | CRR |
| ------------------- | --- | --- |
| Storage đích | Có (theo storage class) | Có |
| Data transfer | **$0** | $0.02/GB |
| Request fee (PUT đích) | $0.005/1,000 | $0.005/1,000 |
| RTC (nếu bật) | $0.015/GB | $0.015/GB |

### Ví Dụ Tính Chi Phí SRR

```
Scenario: 1TB/tháng log từ 3 source buckets vào central-bucket (us-east-1)

Storage nguồn (Standard):     1,024GB × 3 × $0.023 = $70.66
Storage đích (Standard-IA):   1,024GB × $0.0125    = $12.80
Request fee (ước 1M PUT):     1,000 × $0.005        = $5.00
Data transfer:                                        $0.00
                                                    ────────
Tổng chi phí SRR thêm:                              $17.80

So với CRR cùng scenario:
Data transfer: 1,024GB × $0.02 = $20.48 thêm
→ SRR tiết kiệm $20.48/tháng mỗi 1TB
```

---

## 6. Hạn Chế Quan Trọng

### Không Replicate Object Đã Tồn Tại

Giống CRR, SRR không retroactively (hồi tố) replicate object đã có trong bucket trước khi bật rule. Cần dùng S3 Batch Replication để sync dữ liệu cũ.

### Không Replicate Glacier-Class Objects Từ Nguồn

Nếu object nguồn đang ở Glacier (đã được lifecycle chuyển sang Glacier), SRR KHÔNG replicate object đó. SRR chỉ hoạt động với object đang ở Standard, Standard-IA, và Intelligent-Tiering.

### Không Thể Replicate Vào Chính Bucket Đó

Bucket nguồn và đích phải khác nhau, không thể là cùng một bucket.

### Giới Hạn Rule

Tối đa 1,000 replication rules mỗi bucket.

---

## 7. Câu Hỏi Phỏng Vấn

**Q1: Khi nào chọn SRR thay vì CRR?**

> Chọn SRR khi: (1) Dữ liệu không được rời khỏi Region vì lý do compliance hoặc data sovereignty, (2) Mục đích là log aggregation hoặc test/prod isolation trong cùng Region, (3) Muốn tiết kiệm phí data transfer cross-region. Chọn CRR khi: DR ở region khác, giảm latency cho user ở các vùng địa lý khác nhau, hoặc data sovereignty yêu cầu backup ở region khác.

**Q2: SRR có bảo vệ khỏi accidental deletion (xóa nhầm) không?**

> Có một phần. Nếu tắt `DeleteMarkerReplication`, xóa object ở nguồn KHÔNG tạo delete marker ở đích — object vẫn còn ở backup bucket. Tuy nhiên, nếu ai đó có quyền xóa trực tiếp trong bucket đích, object vẫn có thể bị xóa. Để bảo vệ tuyệt đối, kết hợp với Object Lock trên bucket đích.

**Q3: Bidirectional SRR (A→B và B→A) có gây vòng lặp vô hạn không?**

> Không. S3 tự động đánh dấu `ReplicationStatus = REPLICA` cho object được replicate. Các object có status này không được replicate tiếp, ngăn vòng lặp vô hạn. Vì vậy, bidirectional replication an toàn và thường dùng cho active-active sync.

**Q4: Tại sao object từ Glacier không được replicate?**

> Object ở Glacier về mặt logic vẫn là một S3 object nhưng dữ liệu thực tế được lưu trong hệ thống lưu trữ lạnh riêng biệt. S3 Replication chỉ hoạt động với object ở trạng thái "hot" (Standard, Standard-IA, Intelligent-Tiering). Để replicate dữ liệu đang ở Glacier, phải restore trước rồi mới replicate, hoặc dùng S3 Batch Replication sau khi restore.

---

**Tiếp Theo:** [4-s3-object-lock.md](./4-s3-object-lock.md) — WORM và compliance
