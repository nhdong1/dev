# FSx — Amazon File System Nâng Cao: Tổng Quan

> FSx — Amazon FSx — là tập hợp các dịch vụ hệ thống tệp được quản lý hoàn toàn (fully managed file systems) trên AWS, được thiết kế cho các workload doanh nghiệp đòi hỏi hiệu suất cao, tính năng đặc thù của từng hệ thống tệp, và khả năng tích hợp sâu với on-premises.

---

## Mục Lục

1. [FSx là gì?](#fsx-là-gì)
2. [Bốn Biến Thể FSx](#bốn-biến-thể-fsx)
3. [Khi Nào Dùng FSx Thay Vì EFS?](#khi-nào-dùng-fsx-thay-vì-efs)
4. [Kiến Trúc Triển Khai](#kiến-trúc-triển-khai)
5. [Các File Chi Tiết](#các-file-chi-tiết)

---

## FSx là gì?

**Amazon FSx** cung cấp các hệ thống tệp được quản lý hoàn toàn với hiệu suất và tính năng tương đương môi trường on-premises, nhưng không cần quản lý hạ tầng. AWS lo phần cứng, cập nhật phần mềm, backup, và high availability — người dùng chỉ cần mount và dùng.

### Tại Sao Cần FSx?

| Tình Huống | Vấn Đề Với EFS | Giải Pháp FSx |
|-----------|---------------|---------------|
| Windows workload cần SMB và Active Directory | EFS chỉ hỗ trợ NFS, không tích hợp AD | FSx for Windows |
| HPC — High Performance Computing cần hàng trăm GB/s | EFS không đủ throughput | FSx for Lustre |
| Di chuyển từ NetApp on-premises | Không tương thích protocol | FSx for NetApp ONTAP |
| Cần ZFS snapshots tức thì và clones | EFS không có ZFS | FSx for OpenZFS |

---

## Bốn Biến Thể FSx

### 1. FSx for Windows File Server

- **Protocol** (Giao thức): SMB — Server Message Block — Giao Thức Chia Sẻ Tệp Windows
- **Tích hợp**: Active Directory — AD — Dịch Vụ Thư Mục Quản Lý Danh Tính
- **Use case**: Lift-and-shift Windows applications, home directories, SharePoint
- **Throughput**: Lên đến 2 GB/s
- **Chi tiết**: [1-fsx-for-windows.md](./1-fsx-for-windows.md)

### 2. FSx for Lustre

- **Lustre** (từ "Linux" + "cluster"): Hệ thống tệp song song hiệu suất cao
- **Protocol**: Lustre client (Linux only)
- **Use case**: ML training, genomics, financial modeling, rendering
- **Throughput**: Lên đến hàng trăm GB/s
- **Chi tiết**: [2-fsx-for-lustre.md](./2-fsx-for-lustre.md)

### 3. FSx for NetApp ONTAP

- **ONTAP** (Open Network Technology for Adaptable Partitioning): Hệ điều hành lưu trữ của NetApp
- **Protocol**: NFS, SMB, iSCSI — hỗ trợ đa giao thức
- **Use case**: Di chuyển NetApp on-premises lên cloud, enterprise workloads
- **Chi tiết**: [3-fsx-for-netapp-ontap.md](./3-fsx-for-netapp-ontap.md)

### 4. FSx for OpenZFS

- **OpenZFS** (Zettabyte File System — Hệ Thống Tệp Zettabyte, mã nguồn mở): Hệ thống tệp có snapshot tức thì
- **Protocol**: NFS
- **Use case**: Dev/test environments, database cloning, ZFS migration
- **Chi tiết**: [4-fsx-for-openzfs.md](./4-fsx-for-openzfs.md)

---

## So Sánh Nhanh Bốn Biến Thể

| Tiêu Chí              | Windows FS  | Lustre        | NetApp ONTAP  | OpenZFS      |
|----------------------|-------------|---------------|---------------|--------------|
| **Protocol**          | SMB         | Lustre        | NFS/SMB/iSCSI | NFS          |
| **OS hỗ trợ**         | Windows/Linux| Linux         | Windows/Linux | Linux/macOS  |
| **Throughput tối đa** | 2 GB/s      | Hundreds GB/s | Hàng chục GB/s| 21 GB/s      |
| **Storage tối đa**    | 64 TB       | Petabytes     | Petabytes     | 512 TB       |
| **Snapshot**          | VSS         | Không         | SnapMirror    | Tức thì      |
| **Deduplication**     | Có          | Không         | Có            | Có (inline)  |
| **Active Directory**  | Native      | Không         | Có            | Không        |
| **S3 Integration**    | Không       | Có (native)   | Không         | Không        |
| **Phù hợp nhất**      | Windows WL  | HPC / ML      | Enterprise    | Dev/Test     |

---

## Khi Nào Dùng FSx Thay Vì EFS?

```
EFS phù hợp khi:
├── Workload Linux thuần túy
├── Shared NFS đơn giản
├── Container (ECS/EKS) cần shared storage
├── Không có yêu cầu đặc thù về file system
└── Muốn chi phí đơn giản (trả theo dùng)

FSx phù hợp khi:
├── Windows workloads (→ FSx Windows)
├── HPC / ML training (→ FSx Lustre)
├── Di chuyển từ on-premises NetApp (→ FSx ONTAP)
├── Cần ZFS features (snapshots, clones) (→ FSx OpenZFS)
└── Cần multi-protocol (NFS + SMB + iSCSI) (→ FSx ONTAP)
```

---

## Kiến Trúc Triển Khai

### Single-AZ vs Multi-AZ

```
Single-AZ Deployment (Triển Khai Một Vùng Khả Dụng):
┌─────────────────────────────────────┐
│            AZ-A (us-east-1a)        │
│  ┌──────────────────────────────┐   │
│  │     FSx File System          │   │
│  │  Primary + Standby (same AZ) │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
→ Giá thấp hơn, vẫn có HA trong AZ

Multi-AZ Deployment (Triển Khai Đa Vùng Khả Dụng):
┌────────────────┐    ┌────────────────┐
│     AZ-A       │    │     AZ-B       │
│  ┌──────────┐  │    │  ┌──────────┐  │
│  │ Primary  │◄─┼────┼─►│ Standby  │  │
│  └──────────┘  │    │  └──────────┘  │
└────────────────┘    └────────────────┘
→ Chịu được mất một AZ, RTO thấp hơn
```

### Kết Nối FSx

```
On-Premises ─── Direct Connect / VPN ──► FSx
                                          │
EC2 Instances ───────────────────────────►│
                                          │
ECS/EKS Containers ──────────────────────►│
                                          │
Lambda (qua VPC) ────────────────────────►│
```

---

## Mô Hình Định Giá

| Biến Thể | Phí Lưu Trữ | Phí Throughput | Multi-AZ |
|---------|-------------|----------------|---------|
| Windows | ~$0.13/GB-month (SSD) | Tùy cấu hình | +~50% |
| Lustre | $0.14/GB-month (Persistent 1) | Bao gồm | N/A |
| NetApp ONTAP | ~$0.125/GB-month | Tùy cấu hình | +~50% |
| OpenZFS | ~$0.09/GB-month | Tùy cấu hình | N/A |

---

## Điểm Kiểm Tra Cho Phỏng Vấn

### Câu Hỏi Thường Gặp

**Q: Khi nào dùng FSx for Lustre thay vì S3?**
> A: FSx Lustre cho latency cực thấp (microsecond) và throughput hàng trăm GB/s trong quá trình tính toán. Sau khi tính toán xong, kết quả thường được ghi lại về S3 để lưu trữ dài hạn. Hai dịch vụ bổ trợ nhau, không thay thế.

**Q: FSx for Windows có thể dùng trên Linux không?**
> A: Có, qua Samba client. Nhưng nếu workload chủ yếu là Linux, hãy cân nhắc EFS hoặc FSx ONTAP.

**Q: FSx ONTAP khác NetApp on-premises thế nào?**
> A: API và protocol (NFS/SMB/iSCSI) giống hệt, nhưng AWS quản lý phần cứng, HA, backup. Migration thường chỉ cần SnapMirror để sao chép dữ liệu.

---

## Các File Chi Tiết Trong Module Này

| File | Nội Dung | Độ Khó |
|------|----------|--------|
| [1-fsx-for-windows.md](./1-fsx-for-windows.md) | Active Directory, SMB, DFS, shadow copies | ⭐⭐ |
| [2-fsx-for-lustre.md](./2-fsx-for-lustre.md) | HPC, ML training, S3 integration, deployment options | ⭐⭐⭐ |
| [3-fsx-for-netapp-ontap.md](./3-fsx-for-netapp-ontap.md) | Multi-protocol, dedup, SnapMirror, migration | ⭐⭐⭐ |
| [4-fsx-for-openzfs.md](./4-fsx-for-openzfs.md) | Snapshots tức thì, clones, ZFS features | ⭐⭐ |
| [5-fsx-comparison.md](./5-fsx-comparison.md) | So sánh toàn diện, decision tree, scenarios | ⭐⭐ |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Trạng Thái:** ✅ Hoàn thành
