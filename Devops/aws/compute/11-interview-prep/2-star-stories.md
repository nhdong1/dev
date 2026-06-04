# 📖 Mẫu Câu Chuyện Incident Theo Phương Pháp STAR

> 5 câu chuyện mẫu về các sự cố AWS Compute thực tế, được trình bày theo phương pháp STAR (Situation-Task-Action-Result — Tình Huống-Nhiệm Vụ-Hành Động-Kết Quả). Điều chỉnh dựa trên kinh nghiệm thực tế của bạn.

## 📚 Hướng Dẫn Sử Dụng

### Phương Pháp STAR

```
S — Situation (Tình Huống):  Bối cảnh cụ thể — hệ thống gì, quy mô thế nào, khi nào
T — Task (Nhiệm Vụ):         Trách nhiệm của bạn trong tình huống đó là gì
A — Action (Hành Động):      Những bước cụ thể bạn đã thực hiện (dùng "tôi", không dùng "chúng tôi")
R — Result (Kết Quả):        Kết quả đo lường được + bài học + impact lâu dài
```

### Mẹo Kể Chuyện Hiệu Quả

- **Cụ thể hóa:** Đừng nói "traffic tăng" — nói "traffic tăng từ 1,000 lên 15,000 RPS trong 10 phút"
- **Định lượng kết quả:** "Giảm error rate từ 15% xuống 0.1%" thay vì "hệ thống ổn định trở lại"
- **Nhấn mạnh vai trò cá nhân:** Kể những gì BẠN làm, không phải team nói chung
- **Bài học rút ra:** Luôn kết thúc bằng what you learned và preventive measures
- **Thời gian:** Mỗi story 2-3 phút khi kể to

---

## Story 1: EC2 Auto Scaling Không Scale Kịp — Black Friday Incident

### Tình Huống (Situation)

Hệ thống e-commerce của công ty tôi phục vụ trung bình 5,000 RPS (Requests Per Second — Yêu Cầu Mỗi Giây) với Auto Scaling Group gồm 10-20 EC2 m5.xlarge instances. Vào ngày Black Friday (thứ Sáu Đen — ngày sale lớn), traffic tăng đột biến lên 80,000 RPS trong vòng 15 phút — gấp 16 lần bình thường. Hệ thống bắt đầu trả về HTTP 503 (Service Unavailable — Dịch Vụ Không Khả Dụng) cho 40% requests. Đây là sự kiện sale quan trọng nhất trong năm, mỗi phút downtime ước tính mất 50,000 USD doanh thu.

### Nhiệm Vụ (Task)

Tôi là Backend Engineer on-call (trực sự cố) ca đêm. Nhiệm vụ của tôi là: xác định root cause, restore service trong thời gian ngắn nhất, và sau đó cải thiện kiến trúc để event tương tự không xảy ra.

### Hành Động (Action)

**Giai đoạn 1 — Triage (Phân Loại Sự Cố) trong 5 phút đầu:**
- Kiểm tra CloudWatch Dashboard: CPU trung bình 95%, ALB (Application Load Balancer) Active Connection Count tăng vọt, Target Response Time tăng từ 200ms lên 8 giây
- Xác nhận ASG đang cố scale-out nhưng launch time mỗi instance mất 8-10 phút (User Data script install dependencies chậm)
- Kết luận: ASG không scale kịp tốc độ traffic tăng

**Giai đoạn 2 — Immediate Mitigation (Giảm Thiểu Tức Thời) trong 10 phút:**
- Manually set ASG Desired Capacity từ 15 lên 50 instances — bỏ qua scaling policy để scale ngay
- Tạm thời reduce connection timeout trên ALB từ 60s xuống 15s để drain các requests đang stall
- Enable Enhanced CloudFront caching cho static assets để giảm origin hits

**Giai đoạn 3 — Monitor Recovery (Theo Dõi Phục Hồi):**
- 15 phút sau: 25 instance mới đã InService, error rate giảm từ 40% xuống 5%
- 30 phút sau: 50 instances online, error rate về 0.2%, response time về 250ms
- Liên tục theo dõi ASG metrics và ALB Target Health để đảm bảo không có instance unhealthy

**Giai đoạn 4 — Root Cause Analysis và Long-term Fix (Sau incident):**
- Phát hiện: Launch Template dùng t2.medium — quá nhỏ cho production load. User Data script mất 8 phút để install 40 packages
- Tạo custom AMI (Amazon Machine Image) với tất cả dependencies pre-installed → launch time giảm từ 8 phút xuống 90 giây
- Implement Predictive Scaling (Co Giãn Dự Báo) dựa trên historical traffic patterns
- Cấu hình Scheduled Scaling: 2 giờ trước Black Friday, tự động scale lên 80 instances
- Thêm Warm Pool (Bể Khởi Động Ấm) — 30 instances luôn ở trạng thái warm, sẵn sàng trong 60 giây
- Set minimum capacity từ 10 lên 30 trong các dịp sale lớn

### Kết Quả (Result)

- Incident được giải quyết trong 35 phút, tổng thiệt hại doanh thu ước tính 1.75M USD (35 phút × 50,000 USD)
- Black Friday năm sau với kiến trúc mới: không có incident, hệ thống xử lý 120,000 RPS peak smoothly
- Launch time giảm từ 8 phút xuống 90 giây nhờ custom AMI
- Chi phí Warm Pool (~3,000 USD/tháng) được justify bởi ROI từ việc không mất doanh thu

**Bài Học:**
- Never rely on default scaling behavior cho events được dự báo trước — pre-scale thủ công nếu cần
- AMI optimization là việc cần làm ngay, không phải khi có vấn đề
- Phải test scaling behavior với load test thực tế, không chỉ test trên giấy

---

## Story 2: Lambda Throttling — Sự Cố Xử Lý Thanh Toán

### Tình Huống (Situation)

Hệ thống thanh toán dùng Lambda function để xử lý payment events từ SQS (Simple Queue Service — Dịch Vụ Hàng Đợi Đơn Giản). Trung bình xử lý 500 payments/phút. Một buổi sáng, đối tác tích hợp (payment gateway) gửi batch 50,000 events trong vài phút sau khi recover từ outage (sự cố gián đoạn) của họ. Lambda bắt đầu throttle (giới hạn tốc độ) — 429 ThrottledException — và nhiều payments bị delay hàng giờ. Support team nhận hàng trăm complaint (phàn nàn) từ customers về giao dịch "pending" bất thường.

### Nhiệm Vụ (Task)

Tôi là engineer phụ trách payment platform. Cần: restore normal payment processing, giải thích với business team tại sao delay xảy ra, và thiết kế giải pháp phòng ngừa.

### Hành Động (Action)

**Ngay lập tức:**
- Kiểm tra CloudWatch Lambda metrics: ConcurrentExecutions đang ở 1,000 (account-level concurrency limit), Throttles metric tăng vọt
- Kiểm tra SQS queue depth: 48,000 messages chưa được xử lý
- Phát hiện root cause: payment Lambda không có Reserved Concurrency (Đồng Thời Được Đặt Trước) riêng, đang cạnh tranh account-level concurrency với 20+ Lambda functions khác

**Giảm thiểu trong 20 phút:**
- Set Reserved Concurrency cho payment Lambda = 200 (đảm bảo concurrency riêng, không bị "ăn mất" bởi function khác)
- Tạm thời reduce SQS batch size từ 10 xuống 1 để tăng throughput bằng cách invoke nhiều Lambda instances hơn
- Enable SQS extended client để xử lý message retention lên 14 ngày (tránh message expired)

**Tiếp theo 2 giờ:**
- Monitor SQS queue drain — messages được xử lý đều đặn
- Verify payment records trong database — không có duplicate, không có missing
- Communicate với business team: tất cả payments đang được xử lý, ETA (Estimated Time of Arrival — Thời Gian Dự Kiến Hoàn Thành) 2 giờ để clear queue

**Long-term improvements (sau incident):**
- Implement circuit breaker (bộ ngắt mạch): nếu payment gateway return error rate > 20%, Lambda stop retrying và send alert
- Set Reserved Concurrency cho tất cả critical functions (payment, order, notification)
- Implement DLQ (Dead Letter Queue — Hàng Đợi Thư Chết) monitoring với CloudWatch Alarm
- Thêm rate limiting phía payment gateway integration: không nhận quá 1,000 events/phút
- Tạo runbook (tài liệu hướng dẫn vận hành) cho scenario throttling

### Kết Quả (Result)

- 100% payments được xử lý thành công sau 2 giờ 15 phút, không có payment bị mất
- Zero data corruption — idempotency key trong DynamoDB ngăn chặn duplicate processing
- Sau incident, payment Lambda luôn có 200 concurrent executions reserved — không còn bị throttle bởi các function khác
- Thiết lập CloudWatch alarm: cảnh báo ngay khi Throttles > 10 trong 5 phút, on-call được notify trước khi impact customer

---

## Story 3: ECS Container OOM — Memory Leak Production

### Tình Huống (Situation)

Một microservice chạy trên ECS Fargate (Serverless Container Engine — Công Cụ Container Không Máy Chủ) phụ trách xử lý ảnh người dùng upload (image upload service). Service được cấu hình với 2 vCPU và 4GB RAM mỗi task, chạy 5 tasks. Sau 2-3 ngày uptime, tasks bắt đầu bị OOM killed (killed vì hết bộ nhớ) — ECS restart task, gây gián đoạn upload trong ~30 giây mỗi lần. Tần suất OOM ngày càng tăng — từ 1 lần/ngày lên 10 lần/ngày trong 1 tuần.

### Nhiệm Vụ (Task)

Tôi là DevOps Engineer phụ trách platform. Cần: xác định memory leak, tìm fix trong code hoặc cấu hình, và đảm bảo service stable trong khi chờ fix.

### Hành Động (Action)

**Điều tra ban đầu:**
- CloudWatch Container Insights: MemoryUtilized tăng đều từ 1GB lên 4GB trong 48 giờ — pattern rõ ràng của memory leak
- Không có sudden spike — memory tăng dần, không liên quan đến traffic
- Kiểm tra application logs: không có exception liên quan đến memory
- Kiểm tra code: service dùng Node.js, xử lý ảnh với sharp library

**Tạm thời giảm impact:**
- Tăng RAM allocation từ 4GB lên 8GB để có thêm thời gian điều tra
- Cấu hình ECS Service health check aggressively: nếu memory > 6GB, task bị replaced sớm hơn
- Enable CloudWatch Alarm: cảnh báo khi memory > 5GB (75% capacity)

**Root cause investigation:**
- Deploy task với heapdump (ảnh chụp heap memory) được bật khi memory > 5GB
- Phân tích heapdump với Chrome DevTools: phát hiện hàng nghìn Buffer objects (vùng đệm bộ nhớ) không được release
- Trace code: hàm `processImage()` mở readable stream từ S3 nhưng không đóng stream khi exception xảy ra — buffer bị giữ indefinitely

**Fix:**
```javascript
// Trước (bị leak)
async function processImage(s3Key) {
    const stream = await s3.getObject({ Key: s3Key }).createReadStream();
    const result = await sharp(stream).resize(800, 600).toBuffer();
    return result;
    // Nếu exception ở sharp → stream không được closed
}

// Sau (không leak)
async function processImage(s3Key) {
    const stream = await s3.getObject({ Key: s3Key }).createReadStream();
    try {
        const result = await sharp(stream).resize(800, 600).toBuffer();
        return result;
    } finally {
        stream.destroy();  // Luôn đóng stream
    }
}
```

**Deploy và verify:**
- Deploy fix lên staging, chạy 72 giờ, memory stable ở 1.5GB
- Deploy production với rolling update — zero downtime
- Monitor 1 tuần: không có OOM event nào

### Kết Quả (Result)

- Memory leak được fix hoàn toàn — tasks chạy stable > 30 ngày mà không cần restart
- Memory footprint giảm từ 4GB (và tăng dần) xuống 1.5GB stable
- RAM allocation giảm từ 8GB tạm thời xuống 2GB — tiết kiệm ~60% chi phí Fargate task
- Thêm memory leak detection vào CI/CD pipeline: mọi PR phải pass memory usage test trước khi merge

---

## Story 4: EKS Node Not Ready — Cluster Outage

### Tình Huống (Situation)

EKS cluster production với 30 managed nodes (EC2 m5.2xlarge) chạy 200+ pods cho payment, order, và notification services của một fintech startup. Vào 2 giờ sáng thứ Hai, 8/30 nodes đột ngột chuyển sang trạng thái NotReady trong 5 phút. 60+ pods bị evict (đẩy ra) và schedule lại, nhưng 15 pods ở trạng thái Pending vì không đủ tài nguyên trên remaining 22 nodes. Order service bị down hoàn toàn trong 12 phút.

### Nhiệm Vụ (Task)

Tôi là Platform Engineer on-call. Cần: recover cluster nhanh nhất, xác định tại sao 8 nodes fail cùng lúc, và ngăn chặn tái diễn.

### Hành Động (Action)

**T+0 (nhận alert):**
- `kubectl get nodes` — xác nhận 8 nodes NotReady, tất cả trong cùng AZ us-east-1b
- `kubectl describe node <node>` — "`kubelet stopped posting node status`", không có NodeCondition cụ thể
- AWS Console: 8 EC2 instances status "impaired" — hardware issue
- **Kết luận ngay:** AZ failure (sự cố vùng khả dụng) — us-east-1b đang có vấn đề

**T+3 phút — Immediate actions:**
- Cordon (cách ly) tất cả 8 nodes: `kubectl cordon <node>` để scheduler không assign pods mới
- Force delete Pending pods để trigger re-schedule: `kubectl delete pod --field-selector=status.phase=Pending -n <namespace>`
- Kiểm tra Pod Anti-affinity (Chống Ái Lực Pod): phát hiện order-service không có PodAntiAffinity — tất cả replicas có thể land trên cùng AZ

**T+8 phút — Expanding cluster capacity:**
- AWS Console: Managed Node Group — manually trigger node replacement cho 8 failed nodes
- Tạm thời tăng node group max từ 30 lên 40 để có headroom (không gian dự phòng)
- Pods re-schedule hoàn tất, order service restore

**T+12 phút — Cluster recovered:**
- Tất cả pods Running, services healthy
- 8 failed nodes được AWS tự động replace với hardware mới trong 15 phút tiếp theo

**Post-incident improvements:**
- Thêm PodAntiAffinity cho tất cả critical services:
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: order-service
      topologyKey: topology.kubernetes.io/zone  # Đảm bảo mỗi AZ ≥1 pod
```
- Tăng minimum nodes từ 8 → 12 (4 nodes mỗi AZ, đảm bảo 1 AZ fail vẫn còn đủ capacity)
- Cấu hình PodDisruptionBudget (PDB — Ngân Sách Gián Đoạn Pod): `minAvailable: 2` cho mọi critical service
- Thêm runbook cho "AZ outage" scenario vào wiki

### Kết Quả (Result)

- Service restored trong 12 phút (so với target RTO 30 phút)
- Sau khi thêm PodAntiAffinity: chạy failure drill (thử nghiệm sự cố) — terminate 1 AZ hoàn toàn, tất cả services vẫn available với degraded capacity
- Documentation và runbook giúp on-call engineer tiếp theo handle tương tự trong < 5 phút
- Đề xuất được merge vào tiêu chuẩn platform: tất cả production workloads phải có PodAntiAffinity và PDB

---

## Story 5: Spot Instance Interruption — Batch Job Recovery

### Tình Huống (Situation)

Hệ thống data pipeline chạy nightly ETL (Extract-Transform-Load — Trích Xuất-Biến Đổi-Tải) job trên Spot EC2 instances để tiết kiệm chi phí (tiết kiệm 70% so với On-Demand). Job mất 4-6 giờ, xử lý 500GB data từ S3, kết quả ghi vào Redshift. Một buổi tối, AWS thu hồi 3 trong 5 Spot instances giữa chừng (sau khi job chạy được 3 giờ). ETL job fail hoàn toàn, phải restart từ đầu — mất 3 giờ xử lý đã hoàn thành, delay báo cáo buổi sáng thêm 3 giờ.

### Nhiệm Vụ (Task)

Tôi là Data Engineer chịu trách nhiệm ETL pipeline. Cần: recover job hiện tại nhanh nhất, và redesign pipeline để chịu được Spot interruption mà không mất toàn bộ progress.

### Hành Động (Action)

**Immediate recovery:**
- Kiểm tra trạng thái job: 3 giờ đầu đã xử lý xong `customer`, `orders`, `inventory` tables (3/7 tables)
- Restart job nhưng skip các tables đã hoàn thành — manual workaround bằng cách comment out completed steps trong job script
- Dùng On-Demand instances cho phần còn lại để tránh interruption thêm lần nữa
- Job hoàn thành sau thêm 2.5 giờ, báo cáo trễ 2.5 giờ thay vì 5.5 giờ

**Root cause analysis:**
- Spot interruption rate ngày hôm đó cao bất thường vì Amazon Prime Day (traffic AWS tăng)
- ETL job không có checkpointing (lưu điểm tiến độ) — fail = mất toàn bộ

**Redesign pipeline với checkpointing:**

```python
# Checkpoint mechanism với DynamoDB
class ETLJob:
    def __init__(self, job_id):
        self.job_id = job_id
        self.checkpoint_table = boto3.resource('dynamodb').Table('etl-checkpoints')
    
    def get_completed_steps(self):
        response = self.checkpoint_table.get_item(Key={'job_id': self.job_id})
        return response.get('Item', {}).get('completed_steps', [])
    
    def mark_step_complete(self, step_name):
        self.checkpoint_table.update_item(
            Key={'job_id': self.job_id},
            UpdateExpression='ADD completed_steps :step',
            ExpressionAttributeValues={':step': {step_name}}
        )
    
    def process_table(self, table_name):
        completed = self.get_completed_steps()
        if table_name in completed:
            print(f"Skipping {table_name} — already completed")
            return
        
        # Xử lý table
        self._do_process(table_name)
        self.mark_step_complete(table_name)

# Spot interruption handler
def setup_interruption_handler():
    def handler(signum, frame):
        # AWS gửi SIGTERM 2 phút trước khi terminate
        logger.warning("Spot interruption notice received — saving state")
        save_current_checkpoint()
        sys.exit(0)  # Graceful exit
    signal.signal(signal.SIGTERM, handler)
```

**Thêm Mixed Instance Policy:**
```json
{
  "SpotAllocationStrategy": "capacity-optimized",
  "InstancesDistribution": {
    "OnDemandBaseCapacity": 1,
    "OnDemandPercentageAboveBaseCapacity": 20,
    "SpotInstancePools": 4
  },
  "LaunchTemplateOverrides": [
    {"InstanceType": "m5.2xlarge"},
    {"InstanceType": "m5a.2xlarge"},
    {"InstanceType": "m4.2xlarge"},
    {"InstanceType": "m5d.2xlarge"}
  ]
}
```

**Monitor Spot interruption:**
- EventBridge rule: `EC2 Spot Instance Interruption Warning` → SNS alert → On-call notification
- Metric: Spot interruption rate per job — nếu > 30% instances interrupted trong 1 tuần, review allocation strategy

### Kết Quả (Result)

- Với checkpointing: nếu interruption xảy ra sau 3 giờ, job chỉ cần xử lý lại phần chưa hoàn thành — mất < 30 phút thay vì 3 giờ
- Mixed instance policy với 4 instance types: chạy 6 tháng tiếp theo, interruption rate giảm từ ~15% xuống ~2%
- Chi phí vẫn tiết kiệm 65% so với On-Demand (thay vì 70% nhưng đổi lại tính ổn định cao hơn)
- Pipeline design pattern trở thành template cho 5 ETL jobs khác trong team

---

## 📋 Chọn Story Nào Cho Từng Câu Hỏi

| Câu Hỏi Phỏng Vấn                               | Story Phù Hợp  |
| ----------------------------------------------- | -------------- |
| "Kể về incident production bạn đã xử lý"        | Story 1 hoặc 5 |
| "Bạn đã tối ưu chi phí AWS như thế nào?"         | Story 5        |
| "Kể về lần debug vấn đề khó trong production"    | Story 3 (memory leak) |
| "Bạn đã cải thiện reliability của hệ thống thế nào?" | Story 4    |
| "Kinh nghiệm với Lambda hoặc serverless?"         | Story 2        |
| "Xử lý sự cố khi hệ thống down giữa đêm?"        | Story 4        |
| "Bạn đã học được gì từ failure?"                  | Bất kỳ story nào |

---

## 💡 Mẹo Tùy Chỉnh Story Theo Kinh Nghiệm Thực Tế

Nếu bạn chưa có kinh nghiệm trực tiếp với các scenario trên, có thể:

1. **Điều chỉnh scale:** Thay 80,000 RPS bằng con số thực tế của hệ thống bạn làm
2. **Thay service:** Thay ECS bằng EC2 hoặc Kubernetes nếu bạn dùng platform đó
3. **Học từ học tập:** Kể về incident bạn phân tích từ post-mortem của công ty
4. **Side project:** Incident từ personal project cũng có giá trị nếu kể đúng cách

**Quan trọng:** Không bịa đặt. Nếu không có kinh nghiệm trực tiếp, hãy nói thật và giải thích cách bạn sẽ xử lý scenario đó dựa trên kiến thức lý thuyết.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành — 5 STAR stories chi tiết
