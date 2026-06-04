# AWS Snowball Edge — Storage Optimized vs Compute Optimized

> **AWS Snowball Edge** là thiết bị trung tâm của Snow Family — cân bằng giữa khả năng lưu trữ lớn và tính toán mạnh mẽ. Snowball Edge phục vụ hai mục đích: **di chuyển dữ liệu lớn** (10-80 TB mỗi thiết bị, có thể clustering lên đến hàng PB) và **edge computing** (tính toán tại điểm biên) trong môi trường công nghiệp, quân sự, hoặc vùng có kết nối hạn chế.

## 📚 Mục Lục (Table of Contents)

1. [Snowball Edge Là Gì? Hai Phiên Bản](#snowball-edge-là-gì-hai-phiên-bản)
2. [Storage Optimized — Tối Ưu Lưu Trữ](#storage-optimized--tối-ưu-lưu-trữ)
3. [Compute Optimized — Tối Ưu Tính Toán](#compute-optimized--tối-ưu-tính-toán)
4. [So Sánh Hai Phiên Bản](#so-sánh-hai-phiên-bản)
5. [Edge Computing Trên Snowball Edge](#edge-computing-trên-snowball-edge)
6. [Clustering — Kết Nối Nhiều Thiết Bị](#clustering--kết-nối-nhiều-thiết-bị)
7. [Quy Trình Di Chuyển Dữ Liệu](#quy-trình-di-chuyển-dữ-liệu)
8. [Network Interfaces — Giao Diện Mạng](#network-interfaces--giao-diện-mạng)
9. [Bảo Mật](#bảo-mật)
10. [Giá Và Chi Phí](#giá-và-chi-phí)
11. [Các Trường Hợp Sử Dụng Điển Hình](#các-trường-hợp-sử-dụng-điển-hình)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Snowball Edge Là Gì? Hai Phiên Bản

### Tổng Quan

```
AWS Snowball Edge = Thiết bị Snow Family cỡ trung, hai phiên bản chuyên biệt

Kích thước vật lý: Tương đương vali kéo nhỏ
Trọng lượng:      ~22 kg (Storage) / ~22.3 kg (Compute)
Nguồn điện:       Cần ổ cắm điện tiêu chuẩn (không dùng pin)
Chứng nhận:       IP67 (chống nước, bụi), chịu điều kiện khắc nghiệt
```

### Lý Do Tồn Tại Hai Phiên Bản

```
AWS tách Snowball Edge thành hai phiên bản vì:

1. Storage Optimized (Tối Ưu Lưu Trữ):
   Mục tiêu chính: Di chuyển NHIỀU dữ liệu nhất có thể (80 TB)
   Tính toán: Đủ để chạy basic applications khi vận chuyển
   Use case: Data center migration, large backup, archive transfer

2. Compute Optimized (Tối Ưu Tính Toán):
   Mục tiêu chính: Xử lý dữ liệu NẶNG tại chỗ (ML, video, HPC)
   Storage: Giảm xuống để nhường tài nguyên cho CPU/RAM/GPU
   Use case: ML inference, video processing, industrial analytics

→ Không có phiên bản nào "tốt hơn" — chọn theo use case
```

---

## 🗄️ Storage Optimized — Tối Ưu Lưu Trữ

### Thông Số Kỹ Thuật

```
┌─────────────────────────────────────────────────────────────────┐
│          SNOWBALL EDGE STORAGE OPTIMIZED — SPECS                │
├─────────────────────────────┬───────────────────────────────────┤
│ Storage (lưu trữ)           │ 80 TB HDD (usable — sử dụng được)│
│ Block storage               │ 1 TB SSD NVMe (cho EC2 instances) │
│ vCPU (bộ xử lý ảo)          │ 40 vCPUs                          │
│ RAM (bộ nhớ)                │ 80 GB                             │
│ GPU (bộ xử lý đồ họa)       │ Không có                          │
├─────────────────────────────┼───────────────────────────────────┤
│ Network interfaces          │ 1 × 25 GbE (SFP28)               │
│                             │ 2 × 10 GbE (RJ45)                │
│                             │ 1 × 100 GbE (QSFP28) (tùy chọn) │
├─────────────────────────────┼───────────────────────────────────┤
│ EC2 Instance types          │ sbe1.small → sbe1.4xlarge         │
│ Lambda Functions            │ ✅                                │
│ Clustering (ghép cụm)       │ ✅ Tối đa 15 nodes               │
├─────────────────────────────┼───────────────────────────────────┤
│ Mã hóa                      │ AES-256                           │
│ Thời gian thuê tối đa        │ 360 ngày/lần                      │
└─────────────────────────────┴───────────────────────────────────┘
```

### EC2 Instance Types Trên Storage Optimized

```
┌──────────────┬────────┬──────────┬────────────────────────────────┐
│ Instance Type│ vCPU   │ RAM      │ Phù Hợp Cho                    │
├──────────────┼────────┼──────────┼────────────────────────────────┤
│ sbe1.small   │ 1      │ 1 GB     │ Lightweight agents, monitors   │
│ sbe1.medium  │ 1      │ 2 GB     │ Small web apps, proxies        │
│ sbe1.large   │ 2      │ 4 GB     │ Data preprocessing scripts     │
│ sbe1.xlarge  │ 4      │ 8 GB     │ Database nhỏ, app servers      │
│ sbe1.2xlarge │ 8      │ 16 GB    │ Medium workloads               │
│ sbe1.4xlarge │ 16     │ 32 GB    │ Larger batch processing        │
└──────────────┴────────┴──────────┴────────────────────────────────┘

Lưu ý: Tổng tài nguyên EC2 bị giới hạn bởi 40 vCPU và 80 GB RAM của thiết bị
```

### Khi Nào Chọn Storage Optimized

```
✅ Chọn Storage Optimized khi:
├── Cần di chuyển dữ liệu tối đa: 80 TB mỗi thiết bị
├── Data migration là mục tiêu chính, edge compute là phụ
├── Large file transfers: video archives, backup datasets, NAS migration
├── Clustering để scale: 15 nodes × 80 TB = 1.2 PB tổng cộng
└── Cần S3-compatible storage (S3-API) tại biên
```

---

## 💻 Compute Optimized — Tối Ưu Tính Toán

### Thông Số Kỹ Thuật

```
┌─────────────────────────────────────────────────────────────────┐
│          SNOWBALL EDGE COMPUTE OPTIMIZED — SPECS                │
├─────────────────────────────┬───────────────────────────────────┤
│ Storage (không có GPU)      │ 28 TB SSD NVMe (usable)           │
│ Storage (có GPU)            │ 28 TB SSD NVMe (usable)           │
│ Block storage               │ 7.68 TB SSD NVMe (cho EC2)        │
│ vCPU (bộ xử lý ảo)          │ 52 vCPUs (AMD EPYC)               │
│ RAM (bộ nhớ)                │ 208 GB (không GPU) hoặc 416 GB    │
│ GPU (tùy chọn)              │ NVIDIA Tesla V100 (32 GB HBM2)    │
├─────────────────────────────┼───────────────────────────────────┤
│ Network interfaces          │ 2 × 25 GbE (SFP28)               │
│                             │ 2 × 10 GbE (RJ45)                │
│                             │ 1 × 100 GbE (QSFP28) (tùy chọn) │
├─────────────────────────────┼───────────────────────────────────┤
│ EC2 Instance types          │ sbe-c.xlarge → sbe-c.56xlarge     │
│ Lambda Functions            │ ✅                                │
│ Clustering                  │ ✅ Tối đa 15 nodes               │
├─────────────────────────────┼───────────────────────────────────┤
│ Mã hóa                      │ AES-256                           │
│ Thời gian thuê tối đa        │ 360 ngày/lần                      │
└─────────────────────────────┴───────────────────────────────────┘
```

### GPU: NVIDIA Tesla V100 — Ứng Dụng Thực Tế

```
NVIDIA Tesla V100 (trên Snowball Edge Compute Optimized):
├── 32 GB HBM2 memory (High Bandwidth Memory)
├── 14 TFLOPS FP32 / 112 TOPS INT8
├── Phù hợp cho: Deep learning inference (dự đoán học sâu)
│
Ứng dụng thực tế tại biên (edge):
│
├── 1. Computer Vision (Thị Giác Máy Tính):
│   ├── Nhận diện phương tiện vi phạm trên đường cao tốc
│   ├── Quality control (kiểm soát chất lượng) trong nhà máy
│   └── Phát hiện xâm nhập từ camera an ninh
│
├── 2. NLP Inference (Suy Diễn Ngôn Ngữ Tự Nhiên):
│   └── Chatbot, document classification không cần internet
│
├── 3. Seismic Data Processing (Xử Lý Dữ Liệu Địa Chấn):
│   └── Phân tích dữ liệu thăm dò dầu khí tại chỗ
│
├── 4. Military Applications (Ứng Dụng Quân Sự):
│   ├── Phân tích ảnh vệ tinh hoặc ảnh drone thực địa
│   └── Target recognition (nhận diện mục tiêu) không cần cloud
│
└── 5. Scientific Computing (Tính Toán Khoa Học):
    └── Molecular simulation, climate modeling tại thực địa
```

### EC2 Instance Types Trên Compute Optimized

```
┌──────────────────┬────────┬──────────┬──────────────────────────────┐
│ Instance Type    │ vCPU   │ RAM      │ Phù Hợp Cho                  │
├──────────────────┼────────┼──────────┼──────────────────────────────┤
│ sbe-c.xlarge     │ 4      │ 8 GB     │ Microservices nhẹ            │
│ sbe-c.2xlarge    │ 8      │ 16 GB    │ Data processing              │
│ sbe-c.4xlarge    │ 16     │ 32 GB    │ Medium ML workloads          │
│ sbe-c.8xlarge    │ 32     │ 64 GB    │ Heavy ML inference           │
│ sbe-c.16xlarge   │ 52     │ 208 GB   │ Full device resources        │
│ sbe-c.56xlarge   │ 52     │ 416 GB   │ GPU model (full resources)   │
├──────────────────┴────────┴──────────┴──────────────────────────────┤
│ p3dn.48xlarge    │ 52     │ 416 GB   │ + NVIDIA V100 GPU (ML/AI)   │
└──────────────────┴────────┴──────────┴──────────────────────────────┘
```

---

## 📊 So Sánh Hai Phiên Bản

```
┌─────────────────────────────────────────────────────────────────────┐
│          STORAGE OPTIMIZED vs COMPUTE OPTIMIZED                     │
├────────────────────────┬────────────────────┬───────────────────────┤
│ Tiêu Chí               │ Storage Optimized  │ Compute Optimized     │
├────────────────────────┼────────────────────┼───────────────────────┤
│ Dung lượng (usable)    │ 80 TB HDD          │ 28 TB SSD NVMe        │
│ Block storage cho EC2  │ 1 TB SSD           │ 7.68 TB SSD NVMe      │
│ vCPU                   │ 40                 │ 52 (AMD EPYC)         │
│ RAM                    │ 80 GB              │ 208 GB (416 GB w/GPU) │
│ GPU                    │ Không              │ NVIDIA Tesla V100     │
├────────────────────────┼────────────────────┼───────────────────────┤
│ Network tối đa         │ 100 GbE (QSFP28)   │ 100 GbE (QSFP28)     │
│ Clustering             │ ✅ 15 nodes        │ ✅ 15 nodes           │
├────────────────────────┼────────────────────┼───────────────────────┤
│ Điểm mạnh              │ Nhiều storage nhất │ CPU/RAM/GPU mạnh nhất │
│ Chi phí                │ Thấp hơn           │ Cao hơn               │
├────────────────────────┼────────────────────┼───────────────────────┤
│ USE CASE CHÍNH         │ Data migration,    │ ML/AI inference,      │
│                        │ large archive      │ video processing,     │
│                        │ backup offload     │ scientific computing  │
└────────────────────────┴────────────────────┴───────────────────────┘

Chọn Storage Optimized khi:          Chọn Compute Optimized khi:
├── Ưu tiên di chuyển nhiều dữ liệu  ├── Cần GPU (ML, AI, video)
├── Budget eo hẹp hơn                ├── Ứng dụng CPU/RAM nặng
└── Storage là bottleneck chính      └── Cần SSD nhanh cho I/O
```

---

## 💻 Edge Computing Trên Snowball Edge

### Kiến Trúc Edge Computing

```
[Physical Environment — Môi Trường Thực Địa]
        │
        │  (cameras, sensors, machines)
        ▼
[Snowball Edge Device — Thiết Bị Snowball Edge]
┌────────────────────────────────────────────────┐
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  EC2 Instances (AMI từ AWS Region)        │   │
│  │  ├── Data preprocessing app              │   │
│  │  ├── Local database (RDS-like)           │   │
│  │  ├── ML inference service (Compute)      │   │
│  │  └── Local web UI cho field workers      │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Lambda Functions (Event-driven)          │   │
│  │  ├── Trigger: file upload → process       │   │
│  │  └── Trigger: timer → batch job          │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Local S3-compatible Storage              │   │
│  │  └── Bucket trên thiết bị (S3 API)       │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
└────────────────────────────────────────────────┘
        │
        │ (khi có kết nối internet)
        ▼
[AWS Cloud — Đám Mây AWS]
├── S3 (dữ liệu đã xử lý)
├── EC2 (mở rộng khi cần)
└── SageMaker (cập nhật ML models)
```

### IoT Greengrass V2 Tích Hợp

```
AWS IoT Greengrass V2 (Cỏ Xanh IoT Phiên Bản 2):
├── Chạy trên Snowball Edge
├── Quản lý deployments: push updates từ AWS Console
├── Component-based: mỗi ứng dụng là một Greengrass component
├── Offline resilience: tiếp tục hoạt động khi mất internet
└── Shadow sync: đồng bộ trạng thái thiết bị với AWS IoT Core

Use case công nghiệp:
[Nhà máy sản xuất] → Camera → Snowball Edge + Greengrass
                                      │
                          Greengrass component:
                          └── Quality inspection ML model
                              ├── Phát hiện sản phẩm lỗi
                              ├── Ghi log cục bộ
                              └── Alert (cảnh báo) khi tỷ lệ lỗi cao
                                          │
                          Khi có internet: sync log và alert lên AWS
```

---

## 🔗 Clustering — Kết Nối Nhiều Thiết Bị

### Tại Sao Cần Clustering?

```
Vấn đề: Snowball Edge tối đa 80 TB/thiết bị
Nếu bạn cần 500 TB → cần 7 thiết bị
Quản lý 7 thiết bị riêng lẻ: phức tạp, không hiệu quả

Giải pháp: Snowball Edge Clustering (Ghép Cụm)
└── Tối đa 15 nodes trong một cluster = 15 × 80 TB = 1.2 PB
    Cluster xuất hiện như MỘT hệ thống thống nhất
```

### Cách Clustering Hoạt Động

```
CLUSTER TOPOLOGY (Cấu Trúc Ghép Cụm):

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Snowball    │─────│ Snowball    │─────│ Snowball    │
│ Edge Node 1 │     │ Edge Node 2 │     │ Edge Node 3 │
│ (Leader)    │     │ (Member)    │     │ (Member)    │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
               [10 GbE Internal Network]
               (Mạng nội bộ ghép cluster)

Leader Node (Node Trưởng):
├── Điều phối tất cả I/O requests (yêu cầu đọc/ghi)
├── Phân phối data evenly (đều) giữa các nodes
└── Health monitoring (theo dõi sức khỏe) của cluster

Fault Tolerance (Khả Năng Chịu Lỗi):
└── Cluster có thể chịu được 1 node failure (lỗi 1 node)
    mà không mất dữ liệu (data redundancy — dự phòng dữ liệu)
```

### Scale-out Storage với Clustering

```
Cluster Size → Total Usable Storage:

1 node  →  80 TB      (không phải cluster)
3 nodes →  210 TB     (cluster tối thiểu)
5 nodes →  350 TB
10 nodes →  700 TB
15 nodes → 1,050 TB ≈ 1 PB   (cluster tối đa)

Sử dụng nhiều cluster song song:
3 clusters × 15 nodes × 80 TB = 3.6 PB → tiếp cận Snowmobile territory
```

---

## 🔄 Quy Trình Di Chuyển Dữ Liệu

### Import Data vào AWS (Offline Transfer)

```
BƯỚC 1: ĐẶT JOB TRONG AWS CONSOLE
├── Job type: "Import into Amazon S3"
├── Phiên bản: Storage Optimized hoặc Compute Optimized
├── Số lượng: 1 thiết bị hoặc đặt cluster
└── Địa chỉ giao hàng + S3 bucket đích

BƯỚC 2: NHẬN VÀ UNLOCK THIẾT BỊ
├── Cài AWS OpsHub hoặc Snowball client
├── Nhập credentials từ Console để unlock
└── Verify (xác minh) thiết bị bằng manifest file

BƯỚC 3: COPY DỮ LIỆU (các phương thức)

  Phương thức 1: Snowball client CLI
  ┌────────────────────────────────────────────────────────┐
  │  snowballEdge start-service --service-id s3            │
  │  snowballEdge list-buckets                             │
  │  # Copy data to S3 bucket trên thiết bị               │
  │  aws s3 sync /source/data s3://my-bucket              │
  │    --endpoint-url https://<device-ip>:8443            │
  └────────────────────────────────────────────────────────┘

  Phương thức 2: NFS Mount (gắn kết NFS)
  ┌────────────────────────────────────────────────────────┐
  │  mount -t nfs <device-ip>:/buckets/s3/my-bucket /mnt  │
  │  rsync -avz /source/data/ /mnt/                       │
  └────────────────────────────────────────────────────────┘

  Phương thức 3: AWS OpsHub (drag-and-drop GUI)

  Phương thức 4: Trực tiếp qua S3 API
  └── Mọi S3 client đều hoạt động với endpoint thiết bị

BƯỚC 4: NGẮT KẾT NỐI VÀ GỬI VỀ AWS
├── Snowball client: stop-service
├── E-ink label tự cập nhật địa chỉ AWS facility
└── Ship về AWS (UPS, DHL)

BƯỚC 5: AWS IMPORT VÀO S3
├── AWS giải mã và import dữ liệu vào S3 bucket
└── Nhận notification (thông báo) qua email và SNS
```

### Export Data từ AWS (Tải Dữ Liệu Từ S3 Ra)

```
Snowball Edge cũng hỗ trợ EXPORT (xuất dữ liệu từ S3 ra thiết bị):

Use case:
├── Phân phối content (nội dung) đến vùng xa không có internet
├── Cập nhật phần mềm cho hệ thống offline (air-gapped systems)
├── Cài đặt môi trường mới: mang data từ cloud về datacenter mới
└── Disaster recovery testing: mang dữ liệu backup ra để test

Quy trình export:
[S3 bucket] → [AWS copy vào Snowball] → [Ship đến bạn]
            → [Bạn copy từ Snowball ra local storage]
```

---

## 🌐 Network Interfaces — Giao Diện Mạng

```
Snowball Edge có nhiều interface mạng cho nhiều tình huống:

┌─────────────────┬──────────┬────────────────────────────────────┐
│ Interface       │ Tốc Độ   │ Dùng Cho                           │
├─────────────────┼──────────┼────────────────────────────────────┤
│ RJ45 (x2)       │ 10 GbE   │ Kết nối thông thường, server rack  │
│ SFP28 (x1-2)    │ 25 GbE   │ Kết nối switch cao cấp             │
│ QSFP28 (x1)     │ 100 GbE  │ Kết nối datacenter tốc độ cao      │
└─────────────────┴──────────┴────────────────────────────────────┘

Tốc độ copy thực tế:
├── 10 GbE → ~1 GB/s → 80 TB mất ~22 giờ
├── 25 GbE → ~2.5 GB/s → 80 TB mất ~9 giờ
└── 100 GbE → ~10 GB/s → 80 TB mất ~2 giờ

VLAN Tagging: Hỗ trợ — kết nối vào nhiều mạng VLAN khác nhau
Bonding: Hỗ trợ bonding 2 interface để tăng throughput
```

---

## 🔒 Bảo Mật

```
MÃ HÓA ĐẦU-CUỐI (End-to-End Encryption):
├── Tất cả dữ liệu được mã hóa AES-256 ngay khi ghi
├── Khóa mã hóa lưu trong AWS KMS — không bao giờ lưu trên thiết bị
├── CMK — Customer Master Key (Khóa Chính Khách Hàng): bạn tự kiểm soát
└── AWS không thể truy cập dữ liệu kể cả khi có thiết bị trong tay

BẢO VỆ VẬT LÝ:
├── TPM — Trusted Platform Module (Module Bảo Mật Tin Cậy):
│   ├── Chip phần cứng xác minh tính toàn vẹn
│   └── Nếu thiết bị bị tháo mở: TPM kích hoạt, khóa toàn bộ dữ liệu
├── Secure boot: Chỉ chạy firmware (phần mềm nhúng) do AWS ký số
├── Tamper-evident seals (niêm phong phát hiện can thiệp)
└── No user-accessible internal components (không có bộ phận nào người dùng tháo được)

KIỂM SOÁT TRUY CẬP:
├── Manifest file + unlock code: xác thực thiết bị khi nhận
├── IAM policies: kiểm soát ai được tạo/quản lý Snow jobs
├── S3 bucket policies: kiểm soát quyền ghi data vào S3
└── CloudTrail: ghi log mọi API call liên quan đến Snow

SAU KHI HOÀN THÀNH:
└── Secure erase NIST 800-88 (National Institute of Standards
    and Technology — Viện Tiêu Chuẩn Quốc Gia Mỹ)
    AWS cung cấp Certificate of Data Destruction (Chứng chỉ Hủy Dữ Liệu) khi cần

TUÂN THỦ (Compliance):
├── HIPAA (Health Insurance Portability and Accountability Act)
├── PCI-DSS (Payment Card Industry Data Security Standard)
├── SOC 1, SOC 2, SOC 3
└── FedRAMP, DoD CC SRG IL2-IL5 (cho ứng dụng chính phủ Mỹ)
```

---

## 💰 Giá Và Chi Phí

```
Mô hình tính phí (on-demand, không commit trước):

PHẦN 1: PHÍ THIẾT BỊ (Device Fee)
├── Snowball Edge Storage Optimized: ~$300/lần dùng (0-10 ngày đầu)
│   Sau đó: ~$30/ngày thêm
└── Snowball Edge Compute Optimized: ~$300 + phí GPU nếu có

PHẦN 2: PHÍ VẬN CHUYỂN (Shipping Fee)
├── Phụ thuộc địa điểm và tốc độ giao hàng
└── Thường $100-300 một chiều cho thiết bị lớn

PHẦN 3: DATA TRANSFER VÀO S3
└── MIỄN PHÍ — Import vào S3 không mất phí

PHẦN 4: STORAGE S3
└── Standard S3 pricing (~$0.023/GB/tháng)

VÍ DỤ TÍNH TOÁN (80 TB migration):
├── Thiết bị: ~$300
├── Ship 2 chiều: ~$200
├── Tổng phần cứng + vận chuyển: ~$500
├── Data transfer: $0
└── So sánh: 80 TB × $0.09/GB egress = $7,200 nếu dùng internet
    → Snowball tiết kiệm ~$6,700 so với internet!

Long-term rental (thuê dài hạn cho edge computing):
└── ~$360/tháng cho Storage Optimized (không tính ship)
```

---

## 🏭 Các Trường Hợp Sử Dụng Điển Hình

### Case 1: Large Data Center Migration

```
Tình huống:
Công ty có 300 TB dữ liệu cần lên S3 trong 6 tuần.
Kết nối: 500 Mbps (thực tế ~200 Mbps do traffic khác)

Tính toán network:
300 TB qua 200 Mbps: 300,000 GB ÷ 0.025 GB/s ≈ 12M giây ≈ 139 ngày
→ Không khả thi

Giải pháp với Snowball Edge:
├── Đặt 4 × Snowball Edge Storage Optimized (4 × 80 TB = 320 TB)
├── Copy song song: 4 nhóm copy cùng lúc
├── Mỗi nhóm mất ~22 giờ copy (qua 10 GbE)
├── Ship về AWS: 3-5 ngày
├── AWS import: 2-3 ngày
└── Tổng: ~2 tuần → Nhanh hơn 10 lần so với network

Chi phí: 4 × $300 = $1,200 thiết bị + vận chuyển
So với internet: Không khả thi + tốn băng thông sản xuất
```

### Case 2: Industrial IoT — Nhà Máy Thông Minh

```
Tình huống:
Nhà máy lắp ráp xe hơi cần kiểm soát chất lượng bằng AI
tại 10 camera trên dây chuyền lắp ráp.
Kết nối internet: Không ổn định (WAN backup, latency cao)

Yêu cầu:
├── Real-time defect detection (phát hiện lỗi) < 100ms
├── Không được gửi dữ liệu nhạy cảm ra ngoài
└── Hoạt động ngay cả khi mất internet

Giải pháp với Snowball Edge Compute Optimized + GPU:
├── Triển khai 1 Snowball Edge Compute (NVIDIA V100)
├── Chạy EC2 instance với TensorFlow/PyTorch model
├── 10 cameras → frame processing trực tiếp trên thiết bị
├── Kết quả defect report lưu local → sync lên S3 khi có internet
└── AWS cập nhật ML model mới định kỳ qua Greengrass
```

### Case 3: Quân Sự / Chính Phủ — Môi Trường Air-Gapped

```
Air-gapped system (hệ thống hoàn toàn cô lập, không có internet):
├── Snowball Edge hoạt động hoàn toàn offline
├── Tất cả dữ liệu mã hóa → đủ điều kiện bảo mật cao
├── FedRAMP/DoD CC SRG compliance
└── Mang AI capabilities vào vùng chiến trường

Use case:
└── Phân tích ảnh do thám từ drone ngay tại thực địa
    mà không cần kết nối về trung tâm chỉ huy
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q1: Phân biệt Snowball Edge Storage Optimized và Compute Optimized?**

> - **Storage Optimized**: 80 TB HDD, 40 vCPU, 80 GB RAM, không GPU. Dùng khi mục tiêu chính là di chuyển dữ liệu lớn lên S3. Cost-effective hơn cho pure data migration.
> - **Compute Optimized**: 28 TB SSD, 52 vCPU, 208-416 GB RAM, tùy chọn NVIDIA Tesla V100. Dùng khi cần chạy ML inference, video processing, hoặc workload CPU/GPU nặng tại biên.

**Q2: Snowball Edge clustering hoạt động như thế nào?**

> Có thể ghép cụm tối đa 15 Snowball Edge nodes lại, xuất hiện như một hệ thống thống nhất với tổng 1.2 PB (15 × 80 TB). Cluster có fault tolerance (1 node fail không mất data). Leader node phân phối data đều giữa các nodes. Dùng khi cần nhiều TB hơn dung lượng 1 thiết bị cho phép.

**Q3: Khi nào nên dùng nhiều Snowball Edge thay vì Snowmobile?**

> - Dưới 10 PB: Dùng nhiều Snowball Edge (linh hoạt hơn, không cần cơ sở hạ tầng đặc biệt)
> - Trên 10 PB: Snowmobile hiệu quả hơn (tốc độ 1 Tbps, ít overhead hơn)
> - Cần edge compute: Chỉ Snowball Edge mới có — Snowmobile không có compute

**Q4: Làm thế nào để tăng tốc copy data vào Snowball Edge?**

> 1. Dùng 100 GbE QSFP28 interface thay vì 10 GbE RJ45 (10x nhanh hơn)
> 2. Copy song song từ nhiều source servers cùng lúc
> 3. Dùng nhiều thiết bị Snowball Edge song song
> 4. Tránh copy nhiều small files — gộp thành tar archives trước (I/O overhead giảm)
> 5. Dùng S3 multipart upload để tận dụng pipeline

**Q5: Snowball Edge có thể chạy Docker không?**

> Có — thông qua EC2 instances chạy trên Snowball Edge. Cài Docker trên AMI, launch instance trên thiết bị, chạy containers. Tuy nhiên tài nguyên bị giới hạn (40 vCPU/80 GB RAM với Storage Optimized, 52 vCPU/208-416 GB với Compute Optimized).

**Q6: Bạn cần di chuyển 600 TB trong 4 tuần, không có internet tốt. Bao nhiêu Snowball Edge?**

> 600 TB ÷ 80 TB = 8 thiết bị. Chiến lược:
> - Đặt 8 Snowball Edge Storage Optimized đồng thời
> - Chia dữ liệu và copy song song (mỗi thiết bị ~22 giờ copy qua 10 GbE)
> - Ship về AWS: 3-5 ngày
> - AWS import: 3-5 ngày
> - Tổng: ~2 tuần — nằm trong deadline 4 tuần

---

## 🗺️ Điều Hướng

- [README.md](./README.md) — Tổng quan & so sánh Snow Family
- [1-snowcone.md](./1-snowcone.md) — Snowcone: 8-14 TB, edge computing nhỏ gọn
- [3-snowmobile.md](./3-snowmobile.md) — Snowmobile: 100 PB, exabyte migration

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Thuộc Module:** 06-snow-family (AWS Migration & Transfer Services)
