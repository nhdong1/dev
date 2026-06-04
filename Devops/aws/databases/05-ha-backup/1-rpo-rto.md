# RPO & RTO — Mục Tiêu Khôi Phục Trong Thiết Kế Database

> RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục) và RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục) là hai chỉ số quan trọng nhất trong thiết kế Disaster Recovery (DR — Khôi Phục Thảm Họa). Chúng xuất phát từ yêu cầu kinh doanh và quyết định toàn bộ kiến trúc HA & Backup.

---

## 🎯 RPO và RTO Là Gì?

### Định Nghĩa

```
Timeline sự cố:

  Backup/Sync   Disaster Occurs   Recovery Complete
  cuối cùng     Sự cố xảy ra      Hệ thống hoạt động
       │               │                  │
       ▼               ▼                  ▼
───────●───────────────●──────────────────●──────────►
       │◄─────────────►│◄─────────────────►│
              RPO                RTO
      "Dữ liệu tối đa      "Thời gian tối đa
       bị mất bao nhiêu?"   để phục hồi?"
```

**RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục):**
- Khoảng thời gian dữ liệu tối đa có thể bị mất khi xảy ra sự cố
- Xác định tần suất backup cần thiết
- Ví dụ: RPO = 1 giờ → tối đa mất 1 giờ dữ liệu gần nhất

**RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục):**
- Thời gian tối đa để hệ thống phục hồi và hoạt động trở lại sau sự cố
- Xác định loại cơ chế HA/DR cần dùng
- Ví dụ: RTO = 5 phút → hệ thống phải online lại trong 5 phút

---

## 📊 RPO/RTO Theo Ngành Nghề

### Yêu Cầu Điển Hình

| Ngành | RPO | RTO | Chiến Lược Phù Hợp |
|-------|-----|-----|-------------------|
| **Ngân hàng / Tài chính** | < 1 phút | < 5 phút | Multi-AZ + Aurora Global |
| **E-commerce (Thương mại điện tử)** | < 5 phút | < 15 phút | Multi-AZ + Cross-Region Replica |
| **Healthcare (Y tế)** | < 1 giờ | < 4 giờ | Multi-AZ + Automated Backup |
| **SaaS B2B** | < 1 giờ | < 8 giờ | Multi-AZ + Daily Snapshot |
| **Internal tools (Công cụ nội bộ)** | < 24 giờ | < 48 giờ | Automated Backup chỉ |
| **Dev/Test (Phát triển/Kiểm thử)** | Không yêu cầu | Không yêu cầu | Snapshot không thường xuyên |

### Tam Giác Đánh Đổi (Trade-off Triangle)

```
                    Chi Phí Thấp
                        ▲
                       /|\
                      / | \
                     /  |  \
                    /   |   \
                   /    |    \
      RPO dài     /     |     \  RTO dài
      (Nhiều dữ  /      |      \ (Downtime
       liệu mất)/       |       \ dài hơn)
               /________|________\
                        ▼
               RPO ngắn + RTO ngắn
               (Cần chi phí cao nhất)

Quy tắc: RPO và RTO càng ngắn → Chi phí càng cao
```

---

## 🔧 Ánh Xạ RPO/RTO Sang Kiến Trúc AWS

### RPO → Chiến Lược Backup/Replication

```
RPO = 0 (Không mất dữ liệu)
├── Aurora Multi-AZ (Synchronous replication — Đồng bộ tức thì)
└── RDS Multi-AZ (Synchronous standby)

RPO < 1 phút
└── Aurora Global Database (Replication lag < 1 giây)

RPO < 5 phút
└── RDS Read Replica cross-region (Asynchronous — Không đồng bộ, lag vài giây)

RPO < 1 giờ
├── Automated Backup + Transaction logs (PITR với độ chính xác 5 phút)
└── DynamoDB Point-in-Time Recovery

RPO < 24 giờ
└── Daily Automated Backup

RPO > 24 giờ
└── Manual Snapshot theo lịch
```

### RTO → Loại Failover Mechanism

```
RTO < 30 giây
└── Aurora Failover (Chuyển đổi dự phòng tự động trong cluster)

RTO < 2 phút
└── RDS Multi-AZ Auto Failover

RTO < 15 phút
└── Aurora Global Database Managed Failover

RTO < 1 giờ
├── Restore từ Automated Backup (khôi phục từ backup tự động)
└── Promote Read Replica thủ công

RTO < 4 giờ
└── Restore từ Manual Snapshot

RTO > 4 giờ
└── Rebuild từ scratch với backup lâu nhất
```

---

## 💡 Framework Thiết Kế RPO/RTO

### Bước 1: Xác Định Business Requirements (Yêu Cầu Kinh Doanh)

```
Câu hỏi cần hỏi stakeholders (Các Bên Liên Quan):

1. "Nếu database ngừng hoạt động 1 giờ, thiệt hại tài chính là bao nhiêu?"
   → Xác định RTO tolerance (ngưỡng chịu đựng RTO)

2. "Nếu mất 1 giờ dữ liệu giao dịch, có phục hồi được không?"
   → Xác định RPO tolerance (ngưỡng chịu đựng RPO)

3. "Yêu cầu pháp lý/compliance nào áp dụng?"
   → PCI-DSS: RPO ≤ 24h, RTO ≤ 24h (tối thiểu)
   → HIPAA: Yêu cầu DR plan rõ ràng nhưng không quy định cụ thể số giờ

4. "SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) với khách hàng là gì?"
   → 99.9% uptime = tối đa 8.76 giờ downtime/năm → RTO < 8.76h
   → 99.99% uptime = tối đa 52.56 phút/năm → RTO < 52 phút
```

### Bước 2: Tính Chi Phí Downtime

```python
# Ước tính chi phí downtime mỗi giờ
def calculate_downtime_cost(
    hourly_revenue,          # Doanh thu mỗi giờ
    recovery_cost_per_hour,  # Chi phí nhân lực phục hồi
    reputation_factor=0.1    # 10% doanh thu bị ảnh hưởng dài hạn
):
    direct_loss = hourly_revenue
    operational_cost = recovery_cost_per_hour
    reputation_loss = hourly_revenue * reputation_factor
    return direct_loss + operational_cost + reputation_loss

# Ví dụ: E-commerce với $10,000/giờ doanh thu
# direct_loss = $10,000
# operational = $2,000 (nhân lực)
# reputation  = $1,000 (10%)
# Tổng: $13,000/giờ downtime
```

### Bước 3: Chọn Kiến Trúc Phù Hợp Chi Phí

```
┌───────────────────────────────────────────────────────────────┐
│           Ma Trận Lựa Chọn Kiến Trúc                         │
│                                                               │
│  RPO\RTO │ < 1 phút      │ 1-15 phút   │ 15-60 phút │ > 1h  │
│ ─────────┼───────────────┼─────────────┼────────────┼─────── │
│ RPO ≈ 0  │ Aurora Global │ Aurora      │ Multi-AZ   │ Multi  │
│          │ + Multi-AZ    │ Multi-AZ    │ only       │ -AZ    │
│ ─────────┼───────────────┼─────────────┼────────────┼─────── │
│ RPO <5m  │ Aurora Global │ Multi-AZ +  │ Cross-rgn  │ Cross  │
│          │               │ Cross-Rgn   │ Replica    │ -Rgn   │
│ ─────────┼───────────────┼─────────────┼────────────┼─────── │
│ RPO <1h  │ Không thực    │ Multi-AZ +  │ PITR +     │ PITR   │
│          │ tế với chi phí│ Backup      │ Backup     │        │
│ ─────────┼───────────────┼─────────────┼────────────┼─────── │
│ RPO <24h │ Không phù hợp │ Không phù   │ Snapshot   │ Auto   │
│          │               │ hợp         │ + Restore  │ Backup │
└───────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Ví Dụ Thiết Kế Thực Tế

### Ví Dụ 1 — E-commerce với RPO=1 phút, RTO=5 phút

```
Yêu cầu:
  - RPO = 1 phút (tối đa mất 1 phút giao dịch)
  - RTO = 5 phút (phải online trong 5 phút)
  - Budget (Ngân Sách): $2,000/tháng cho database

Kiến Trúc Đề Xuất:
┌────────────────────────────────────────────────────┐
│                  us-east-1 (Region Chính)          │
│                                                    │
│  ┌────────────────────────────────────────────┐   │
│  │         Aurora MySQL Cluster               │   │
│  │                                            │   │
│  │  ┌──────────┐     ┌──────────┐            │   │
│  │  │ Writer   │     │ Reader 1 │            │   │
│  │  │ AZ-a     │     │ AZ-b     │            │   │
│  │  └──────────┘     └──────────┘            │   │
│  │                                            │   │
│  │  Shared Storage: 6 copies (2 AZ × 3)      │   │
│  │  Failover: 15-30 giây tự động             │   │
│  └────────────────────────────────────────────┘   │
│                                                    │
│  Automated Backup: 7 ngày retention               │
│  PITR: Enabled (Đã bật)                           │
└────────────────────────────────────────────────────┘

RPO đạt được: ~0 (synchronous replication trong cluster)
RTO đạt được: ~30 giây (Aurora auto-failover)
Chi phí: ~$800/tháng → Đáp ứng budget, vượt yêu cầu
```

### Ví Dụ 2 — SaaS B2B với RPO=1 giờ, RTO=4 giờ

```
Yêu cầu:
  - RPO = 1 giờ (mất tối đa 1 giờ log hành động người dùng)
  - RTO = 4 giờ (ops team có 4 tiếng để phục hồi)
  - Budget: $300/tháng

Kiến Trúc Đề Xuất:
┌────────────────────────────────────────────────────┐
│                  us-west-2 (Region Chính)          │
│                                                    │
│  ┌────────────────────────────────────────────┐   │
│  │         RDS MySQL Multi-AZ                 │   │
│  │                                            │   │
│  │  ┌──────────┐     ┌──────────┐            │   │
│  │  │ Primary  │────►│ Standby  │            │   │
│  │  │ AZ-a     │sync │ AZ-b     │            │   │
│  │  └──────────┘     └──────────┘            │   │
│  └────────────────────────────────────────────┘   │
│                                                    │
│  Automated Backup: 14 ngày retention              │
│  PITR: Enabled → RPO thực tế ~5 phút              │
└────────────────────────────────────────────────────┘

RPO đạt được: ~5 phút (PITR với transaction logs)
RTO đạt được: ~2 phút (Multi-AZ auto-failover)
Chi phí: ~$200/tháng → Đáp ứng budget, vượt yêu cầu
```

### Ví Dụ 3 — Hệ Thống Ngân Hàng với RPO=0, RTO=1 phút

```
Yêu cầu:
  - RPO = 0 (không mất bất kỳ giao dịch nào)
  - RTO = 1 phút (yêu cầu pháp lý nghiêm ngặt)
  - Multi-region (nhiều vùng địa lý)

Kiến Trúc Đề Xuất:
┌──────────────────────┐      ┌──────────────────────┐
│   us-east-1 (Chính)  │      │  us-west-2 (DR)      │
│                      │      │                      │
│  Aurora Global       │◄────►│  Aurora Global       │
│  Primary Cluster     │ <1s  │  Secondary Cluster   │
│  Writer + 2 Readers  │ lag  │  (Read-only)         │
│                      │      │                      │
│  Automated Backup    │      │  Cross-region copy   │
│  35 ngày retention   │      │  Automated           │
└──────────────────────┘      └──────────────────────┘

RPO đạt được: < 1 giây (Aurora Global replication lag)
RTO đạt được: < 1 phút (Aurora Global managed failover)
Chi phí: Premium — nhưng justified (hợp lý) với yêu cầu ngân hàng
```

---

## 📏 Đo Lường RPO/RTO Thực Tế

### Kiểm Tra RPO

```bash
# Tạo record test trước khi simulate sự cố
INSERT INTO dr_test_log (test_id, created_at) VALUES ('test_001', NOW());

# Simulate sự cố (failover hoặc restore từ backup)
# ...

# Sau khi phục hồi, kiểm tra dữ liệu cuối cùng có mặt
SELECT MAX(created_at) as last_data_point FROM dr_test_log;

# RPO thực tế = NOW() tại thời điểm sự cố - last_data_point
```

### Kiểm Tra RTO

```bash
#!/bin/bash
# Đo thời gian phục hồi thực tế

START_TIME=$(date +%s)

# Trigger failover hoặc restore
aws rds reboot-db-instance \
  --db-instance-identifier my-prod-db \
  --force-failover

# Chờ database available
while true; do
  STATUS=$(aws rds describe-db-instances \
    --db-instance-identifier my-prod-db \
    --query 'DBInstances[0].DBInstanceStatus' \
    --output text)
  
  if [ "$STATUS" = "available" ]; then
    break
  fi
  sleep 10
done

END_TIME=$(date +%s)
RTO_ACTUAL=$((END_TIME - START_TIME))
echo "RTO thực tế: ${RTO_ACTUAL} giây"
```

---

## 📋 DR Testing (Kiểm Tra Khôi Phục Thảm Họa) Best Practices

### Lịch Kiểm Tra Định Kỳ

```
Hàng tháng:
  - Restore từ snapshot vào môi trường test
  - Verify (xác minh) dữ liệu integrity (tính toàn vẹn)
  - Đo RTO thực tế vs mục tiêu

Hàng quý:
  - Full failover test (Kiểm tra chuyển đổi dự phòng đầy đủ)
  - Cross-region restore test
  - Cập nhật runbook nếu cần

Hàng năm:
  - Full DR drill (Diễn tập khôi phục thảm họa toàn diện)
  - Kiểm tra tất cả kịch bản sự cố
  - Review và cập nhật RPO/RTO với business stakeholders
```

### Checklist Sau Mỗi DR Test

```markdown
## DR Test Report — [Date]

### Thông Tin Test
- Loại test: [ ] Failover  [ ] Restore  [ ] Full DR drill
- Environment (Môi trường): [ ] Test  [ ] Staging  [ ] Production

### Kết Quả Đo Lường
- RPO mục tiêu: _____ phút
- RPO thực tế: _____ phút  → [ ] Đạt  [ ] Không đạt
- RTO mục tiêu: _____ phút
- RTO thực tế: _____ phút  → [ ] Đạt  [ ] Không đạt

### Dữ Liệu
- [ ] Kiểm tra data integrity sau restore
- [ ] Row count match (Số hàng khớp)
- [ ] Business-critical data verified (Dữ liệu quan trọng đã xác minh)

### Issues Found (Vấn Đề Phát Hiện)
- [Liệt kê issues]

### Action Items (Hành Động Tiếp Theo)
- [Liệt kê cải tiến cần thực hiện]
```

---

## 🎯 Câu Hỏi Phỏng Vấn Về RPO/RTO

### Câu Hỏi Lý Thuyết

**Q: RPO và RTO khác nhau như thế nào?**
> RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục) đo lường tối đa bao nhiêu dữ liệu có thể bị mất (khoảng thời gian từ backup cuối đến lúc sự cố). RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục) đo lường tối đa bao lâu hệ thống có thể ngừng hoạt động trước khi phải online lại. RPO liên quan đến dữ liệu, RTO liên quan đến thời gian.

**Q: Nếu RPO = 0, điều đó có nghĩa gì về kiến trúc?**
> RPO = 0 nghĩa là không được phép mất bất kỳ giao dịch nào. Điều này đòi hỏi synchronous replication (đồng bộ tức thì) — mỗi write phải được ghi vào ít nhất 2 nơi trước khi xác nhận thành công với client. Trên AWS, điều này đạt được với RDS Multi-AZ (synchronous standby) hoặc Aurora (ít nhất 4/6 bản sao xác nhận trước khi acknowledge).

### Câu Hỏi Thiết Kế

**Q: Khách hàng yêu cầu RPO = 15 phút, RTO = 1 giờ, budget $500/tháng. Bạn thiết kế gì?**
> Đề xuất: RDS MySQL Multi-AZ (db.t3.medium ~$100/tháng) + Automated Backup 14 ngày + PITR enabled. Multi-AZ đảm bảo RTO < 2 phút (vượt yêu cầu). PITR đảm bảo RPO ~5 phút (vượt yêu cầu). Chi phí khoảng $150/tháng, nằm trong budget.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
