# AWS DataSync Nâng Cao — Bandwidth Throttling, Filtering, Verification, Best Practices

> Sau khi nắm vững kiến trúc cơ bản (agent, location, task), phần nâng cao này tập trung vào ba kỹ thuật quan trọng trong môi trường production: **Bandwidth Throttling** — Giới Hạn Băng Thông (để không ảnh hưởng production traffic), **Filtering** — Lọc File (để chỉ transfer đúng dữ liệu cần), và **Data Verification** — Xác Minh Toàn Vẹn Dữ Liệu (để đảm bảo không mất dữ liệu).

## 📚 Mục Lục (Table of Contents)

1. [Bandwidth Throttling — Giới Hạn Băng Thông](#bandwidth-throttling--giới-hạn-băng-thông)
2. [Filtering — Lọc File và Thư Mục](#filtering--lọc-file-và-thư-mục)
3. [Data Verification — Xác Minh Toàn Vẹn Dữ Liệu](#data-verification--xác-minh-toàn-vẹn-dữ-liệu)
4. [Performance Optimization — Tối Ưu Hiệu Suất](#performance-optimization--tối-ưu-hiệu-suất)
5. [Error Handling — Xử Lý Lỗi](#error-handling--xử-lý-lỗi)
6. [Multi-Agent Architecture — Kiến Trúc Nhiều Agent](#multi-agent-architecture--kiến-trúc-nhiều-agent)
7. [Cost Optimization — Tối Ưu Chi Phí](#cost-optimization--tối-ưu-chi-phí)
8. [Best Practices Tổng Hợp](#best-practices-tổng-hợp)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🚦 Bandwidth Throttling — Giới Hạn Băng Thông

### Tại sao cần Bandwidth Throttling?

```
Vấn đề không có throttling:
├── DataSync mặc định dùng tối đa băng thông có thể
├── Transfer 5 TB qua đường 1 Gbps → chiếm ~900 Mbps
├── Production traffic (traffic sản xuất) bị chậm, latency tăng
└── Nguy hiểm khi chạy DataSync trong giờ cao điểm (peak hours)

Với throttling:
├── Giới hạn DataSync chỉ dùng X Mbps
├── Còn lại dành cho production
└── Transfer lâu hơn nhưng không ảnh hưởng business
```

### Cấu hình Bandwidth Throttling trong DataSync

```
Có hai cấp độ throttling:

Level 1: Agent-level throttling (Giới hạn ở cấp Agent)
├── Áp dụng cho tất cả tasks chạy qua agent đó
├── Cấu hình trong: DataSync Console → Agents → Chọn agent → Edit
└── Setting: "Bandwidth limit" → nhập giá trị Mbps (ví dụ: 100 Mbps)

Level 2: Task-level throttling (Giới hạn ở cấp Task)
├── Áp dụng riêng cho từng task
├── Cấu hình trong: Task → Settings → "Bandwidth limit"
└── Override (ghi đè) agent-level setting cho task cụ thể

Giá trị:
├── -1 hoặc 0: không giới hạn (dùng hết băng thông)
└── > 0: giới hạn theo Mbps (megabits per second)
```

### Chiến lược Throttling điển hình

```
Chiến lược 1: Off-hours transfer (Chạy ngoài giờ cao điểm)
├── Không cần throttling nghiêm ngặt vì chạy 2-4 AM
├── Set bandwidth = -1 (không giới hạn) cho nhanh
└── Xong trước 7 AM trước khi business hours bắt đầu

Chiến lược 2: Daytime throttled (Chạy ban ngày với giới hạn)
├── Đường 1 Gbps, production cần ~600 Mbps
├── Set throttle = 300 Mbps cho DataSync
├── Còn lại 700 Mbps cho production (có buffer)
└── Transfer chậm hơn nhưng an toàn

Chiến lược 3: Weekend burst (Tăng tốc cuối tuần)
├── Thứ Bảy/Chủ Nhật: traffic thấp
├── Schedule task chạy cuối tuần với bandwidth = -1
└── Hoàn thành initial migration trước Thứ Hai

Ví dụ tính toán:
├── Dữ liệu: 10 TB
├── Bandwidth throttle: 500 Mbps = 62.5 MB/s
├── Thời gian lý thuyết: 10 × 1024 × 1024 MB ÷ 62.5 MB/s = 167,772 giây ≈ 46.6 giờ
└── Thực tế (+30% overhead): ~60 giờ = 2.5 ngày
```

---

## 🔍 Filtering — Lọc File và Thư Mục

### Include Filters và Exclude Filters

DataSync hỗ trợ hai loại filter (bộ lọc) trên Task level:

```
Include filters (Bộ lọc bao gồm):
└── Chỉ transfer các file/folder KHỚP với pattern
    Ví dụ: chỉ transfer file *.csv

Exclude filters (Bộ lọc loại trừ):
└── Transfer tất cả TRỪ file/folder khớp với pattern
    Ví dụ: bỏ qua thư mục /tmp và file *.log

Ưu tiên: Exclude filters được áp dụng trước Include filters
```

### Cú pháp Filter Patterns

```
Wildcard characters (ký tự đại diện):
├── * — khớp với bất kỳ chuỗi ký tự nào (không gồm /)
├── ? — khớp với đúng một ký tự
└── [abc] — khớp với một ký tự trong tập hợp

Ví dụ patterns phổ biến:

Exclude patterns:
├── /tmp/*           → Loại trừ toàn bộ thư mục /tmp
├── *.log            → Loại trừ tất cả file .log
├── *.tmp            → Loại trừ file tạm thời
├── .DS_Store        → Loại trừ macOS metadata files
├── /archive/*       → Loại trừ thư mục archive cũ
└── */node_modules/* → Loại trừ node_modules ở mọi nơi

Include patterns:
├── *.csv            → Chỉ transfer file CSV
├── /data/2025/*     → Chỉ transfer data năm 2025
└── /reports/*.pdf   → Chỉ transfer PDF trong thư mục reports
```

### Ví dụ thực tế về Filtering

```
Bài toán: Transfer NAS on-premises lên S3, nhưng:
├── Không cần file backup (*.bak, *.old)
├── Không cần logs (*.log, /var/log/*)
├── Không cần temp files (/tmp/*, *.tmp, *.temp)
└── Chỉ cần data từ năm 2023 trở về sau (/data/202[3-9]/*)

Cấu hình:
Exclude filters:
  ├── *.bak
  ├── *.old
  ├── *.log
  ├── /var/log/*
  ├── /tmp/*
  ├── *.tmp
  └── *.temp

Include filters:
  └── /data/202[3-9]/*

Kết quả:
├── Giảm lượng dữ liệu cần transfer đáng kể
├── Tiết kiệm thời gian và chi phí
└── Chỉ transfer dữ liệu thực sự cần thiết
```

---

## ✅ Data Verification — Xác Minh Toàn Vẹn Dữ Liệu

### Tại sao cần Verification?

```
Rủi ro không có verification:
├── Bit rot (lỗi bit ngẫu nhiên) trong quá trình truyền
├── Network packet corruption (gói tin bị hỏng)
├── Disk error tại source hoặc destination
└── Software bug trong copy process

Với DataSync verification:
├── DataSync tính checksum (mã băm kiểm tra toàn vẹn) cho từng file
│   └── Dùng thuật toán: phụ thuộc loại storage, thường là MD5 hoặc ETag
├── Sau khi copy: tính lại checksum tại destination
└── So sánh: nếu khác nhau → báo lỗi, không âm thầm bỏ qua
```

### Ba chế độ Verification

```
1. "Verify only the data transferred" (mặc định — Chỉ verify dữ liệu vừa transfer):
   ├── Chỉ verify các file được copy trong execution này
   ├── Nhanh nhất trong ba chế độ
   └── Phù hợp: incremental daily backup

2. "Verify all data in the destination" (Verify toàn bộ destination):
   ├── Verify TẤT CẢ file tại destination, kể cả file đã copy trước đó
   ├── Chậm hơn nhiều — phải đọc toàn bộ destination
   └── Phù hợp: sau initial migration, cần xác nhận 100% dữ liệu đúng

3. "No verification" (Không verify):
   ├── Bỏ qua bước verification hoàn toàn
   ├── Nhanh nhất nhưng rủi ro cao
   └── Chỉ dùng khi: dữ liệu không quan trọng, hoặc test transfer speed
```

### Verification Workflow (Luồng Xác Minh)

```
Quá trình verify cho từng file:

Source file: report.csv (1 MB)
├── DataSync tính checksum nguồn: SHA-256(report.csv) = abc123...
├── Transfer file qua mạng với TLS
├── DataSync nhận file tại destination: report.csv (1 MB)
├── Tính lại checksum: SHA-256(report.csv) = abc123... ✅ MATCH
└── File được đánh dấu "verified"

Nếu checksum không khớp:
├── File bị đánh dấu "ERROR"
├── Ghi vào error report (báo cáo lỗi) trong CloudWatch Logs
└── DataSync tự động retry (thử lại) transfer file đó
```

### Xử lý kết quả Verification

```
Sau execution hoàn thành, xem kết quả tại:
├── DataSync Console → Task Executions → chọn execution → View details
│   ├── Files transferred: 10,000
│   ├── Files verified: 10,000
│   └── Files with errors: 0 ✅
│
└── CloudWatch Logs (nếu có lỗi):
    ├── /aws/datasync/task/task-xxx/execution/exec-xxx
    └── Ví dụ log lỗi:
        {
          "Type": "VERIFY",
          "ErrorCode": "E_CHECKSUM_MISMATCH",
          "ErrorDetail": "Checksum mismatch for file /data/report.csv"
        }
```

---

## ⚡ Performance Optimization — Tối Ưu Hiệu Suất

### Yếu tố ảnh hưởng hiệu suất

```
1. Network bandwidth (Băng thông mạng):
   ├── Bottleneck (điểm nghẽn cổ chai) phổ biến nhất
   ├── DataSync tận dụng 100% băng thông khả dụng nếu không throttle
   └── Direct Connect > Site-to-Site VPN về độ ổn định và throughput

2. Số lượng file nhỏ (Small files problem):
   ├── Transfer 1 triệu file × 1 KB CHẬM hơn transfer 1 file × 1 TB
   ├── Vì mỗi file có overhead: enumerate (liệt kê), open, close, verify
   └── Giải pháp: tar/zip nhóm file nhỏ trước khi transfer (nếu cần)

3. Agent instance size:
   ├── Agent VM càng nhiều vCPU và RAM → throughput càng cao
   └── Khuyến nghị: 4 vCPU, 32 GB RAM cho production

4. Source storage performance:
   └── NFS/SMB server phải đủ mạnh để cấp dữ liệu kịp cho agent đọc
```

### Parallel Transfer (Truyền Song Song)

```
DataSync tự động dùng parallel transfer:
├── Chia file thành nhiều chunks (phần nhỏ) và transfer song song
├── Nhiều file được transfer đồng thời (concurrent transfers)
└── Không cần cấu hình thêm — DataSync tự tối ưu

Multi-agent parallelism (Nhiều agent song song):
├── Có thể chạy nhiều agent trỏ vào cùng source
├── Chia thư mục: agent1 → /data/A-M, agent2 → /data/N-Z
└── Tăng tổng throughput tuyến tính
```

### Network Configuration (Cấu Hình Mạng)

```
Best practice cho performance:
├── Dùng AWS Direct Connect (kết nối chuyên dụng vật lý đến AWS):
│   ├── Bandwidth ổn định, latency thấp, không chia sẻ với internet
│   └── 1 Gbps hoặc 10 Gbps dedicated connection
│
├── Dùng VPC Endpoints (điểm cuối VPC — giữ traffic trong mạng AWS):
│   ├── DataSync traffic không ra internet public
│   └── Tăng security và đôi khi tăng performance
│
└── Tránh dùng Site-to-Site VPN cho large-scale transfer:
    ├── VPN có overhead từ encryption/decryption
    └── Throughput thực tế thường < 70% nominal bandwidth
```

---

## ⚠️ Error Handling — Xử Lý Lỗi

### Các loại lỗi phổ biến

```
1. Permission denied (Không có quyền truy cập):
   ├── Nguyên nhân: agent không có quyền đọc file tại source
   ├── Phân biệt: file bị lock bởi process khác
   └── Xử lý: kiểm tra NFS export permissions, file ACLs

2. Path too long (Đường dẫn quá dài):
   ├── Windows: giới hạn MAX_PATH = 260 ký tự
   ├── Linux: giới hạn 4096 ký tự (hiếm khi đạt)
   └── Xử lý: đổi tên file/folder ngắn lại trước khi transfer

3. Checksum mismatch (Không khớp checksum):
   ├── Nguyên nhân: bit flip trong quá trình truyền (rất hiếm)
   ├── DataSync tự động retry file đó
   └── Nếu retry vẫn lỗi → kiểm tra source file có bị hỏng không

4. Agent offline (Agent mất kết nối):
   ├── Nguyên nhân: network issue, VM crash, firewall block port 443
   └── Xử lý: kiểm tra kết nối từ agent VM ra internet/AWS

5. Insufficient permissions on destination:
   └── IAM role của DataSync task không có quyền ghi vào S3 bucket
```

### Retry và Error Recovery

```
DataSync retry behavior:
├── File-level retry: retry từng file lỗi tự động (trong cùng execution)
├── Execution-level: nếu execution thất bại, chạy lại execution mới
└── DataSync chỉ transfer file chưa transfer thành công (incremental)

Khi nào nên chạy lại task:
├── Phần lớn file thành công, chỉ vài file lỗi
├── Lần chạy lại: DataSync chỉ retry file lỗi (nếu dùng "Changed files only")
└── Không cần transfer lại toàn bộ từ đầu
```

---

## 🔄 Multi-Agent Architecture — Kiến Trúc Nhiều Agent

### Khi nào cần nhiều Agent?

```
Kịch bản cần Multi-agent:
├── Dữ liệu lớn (> 100 TB) và cần hoàn thành trong thời gian ngắn
├── Source storage ở nhiều vị trí địa lý khác nhau
├── Một agent không đủ throughput dù bandwidth đủ
└── Fault tolerance: nếu một agent lỗi, agent khác tiếp tục

Ví dụ:
├── Agent 1: copy /data/finance/ → S3 bucket
├── Agent 2: copy /data/hr/ → S3 bucket  
├── Agent 3: copy /data/engineering/ → S3 bucket
└── Tổng throughput: 3× so với một agent
```

### Task với Multiple Agents

```
Cấu hình task với nhiều agent:
├── Một task chỉ có thể gán VÀO một agent tại một thời điểm
├── Để dùng nhiều agent song song → tạo nhiều TASK khác nhau
│   ├── Task 1: agent1 + source location /data/A-M
│   ├── Task 2: agent2 + source location /data/N-Z
│   └── Cùng destination S3 bucket
└── Chạy cả hai task đồng thời → parallel ingest (nhập song song)
```

---

## 💰 Cost Optimization — Tối Ưu Chi Phí

### Mô hình tính phí DataSync

```
DataSync tính phí theo GB đã transfer:
└── Giá: $0.0125 per GB (ở region US) — kiểm tra AWS pricing page để có giá mới nhất

Ví dụ tính chi phí:
├── Transfer 10 TB = 10,240 GB
├── Chi phí DataSync: 10,240 × $0.0125 = $128
├── Chi phí S3 storage (sau khi transfer): theo S3 pricing
└── Chi phí network egress (nếu dùng Direct Connect): theo Direct Connect pricing

Không có phí:
├── Transfer giữa hai AWS services trong cùng region (ví dụ S3 → EFS)
└── Chỉ tính phí data ingest vào AWS, không tính egress trong nhiều trường hợp
```

### Tối ưu chi phí

```
Mẹo 1: Dùng Exclude filters để giảm dữ liệu transfer
└── Loại trừ logs, temp files, duplicates → giảm 20-40% data volume

Mẹo 2: Chọn đúng S3 storage class
├── Active data → S3 Standard
├── Infrequent access → S3 Standard-IA (tiết kiệm 40%)
├── Archive → S3 Glacier Instant Retrieval (tiết kiệm 70%)
└── DataSync cho phép chọn storage class tại task level

Mẹo 3: Compress data trước khi transfer (nếu phù hợp)
├── DataSync có built-in compression cho một số loại data
└── File text, CSV, JSON nén tốt → giảm data transferred đáng kể

Mẹo 4: Tắt verification cho data không quan trọng
└── Verification tốn thêm bandwidth để đọc lại file tại destination
```

---

## 📋 Best Practices Tổng Hợp

### Checklist trước khi chạy DataSync Production

```
Chuẩn bị:
✅ Agent đã được deploy và ở trạng thái "Online"
✅ Source location được test — agent có thể đọc dữ liệu
✅ Destination location được test — DataSync có thể ghi
✅ IAM role có đủ permissions
✅ Bandwidth throttle đã được cấu hình phù hợp
✅ CloudWatch Logs được bật để capture errors
✅ Exclude filters đã loại trừ file không cần thiết

Khi chạy:
✅ Chạy test execution với một thư mục nhỏ trước
✅ Verify số lượng file và kích thước khớp với source
✅ Monitor CloudWatch metrics trong giờ đầu
✅ Xác nhận không có impact đến production traffic

Sau khi hoàn thành:
✅ Kiểm tra verification report — 0 errors
✅ So sánh file count và total size giữa source và destination
✅ Sample test: mở random 10-20 file để kiểm tra nội dung
✅ Document kết quả migration (thời gian, data volume, errors)
```

### Anti-patterns (Điều nên tránh)

```
❌ Chạy DataSync giờ cao điểm không có bandwidth throttle
   → Ảnh hưởng production traffic

❌ Tắt verification cho critical data (dữ liệu quan trọng)
   → Rủi ro mất/hỏng dữ liệu mà không biết

❌ Dùng "Verify all files in destination" cho task chạy thường xuyên
   → Tốn thời gian và chi phí không cần thiết (dùng cho initial verification)

❌ Không monitor task execution trong initial migration
   → Có thể phát hiện vấn đề muộn, mất nhiều thời gian debug

❌ Quên set exclude filters cho temp/log files
   → Tốn chi phí transfer dữ liệu vô ích
```

---

## 🎓 Câu Hỏi Phỏng Vấn

1. **Bandwidth throttling trong DataSync hoạt động ở cấp độ nào? Cấu hình ở đâu?**
   - Có hai cấp: agent-level (áp dụng cho tất cả tasks) và task-level (ghi đè cho task cụ thể). Cấu hình trong DataSync Console.

2. **DataSync có bao nhiêu chế độ verification? Khi nào dùng từng chế độ?**
   - Ba chế độ: chỉ verify file vừa transfer (daily incremental), verify toàn bộ destination (sau initial migration), không verify (test/non-critical). Mặc định là chế độ đầu tiên.

3. **Nếu có 1 triệu file nhỏ 1 KB cần transfer, DataSync có vấn đề gì?**
   - Small files problem: mỗi file có overhead enumerate/open/close/verify. Throughput thực tế thấp hơn nhiều so với transfer ít file lớn. Giải pháp: tar/zip nhóm file nhỏ lại nếu có thể.

4. **Làm thế nào tăng throughput khi một agent không đủ nhanh?**
   - Deploy thêm agent, tạo nhiều task với mỗi task gán một agent khác nhau, chia source data theo thư mục giữa các task, chạy song song.

5. **Explain cách DataSync detect file nào cần transfer trong incremental sync.**
   - So sánh modification time (mtime) và file size giữa source và destination. Nếu mtime mới hơn hoặc size khác → cần transfer. Không dùng checksum ở bước này để tránh phải đọc toàn bộ file (chỉ checksum sau khi transfer).

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
