# AWS App2Container (A2C) — Container Hóa Ứng Dụng Java/.NET Tự Động

> **AWS App2Container (A2C)** là công cụ dòng lệnh (CLI — Command-Line Interface) **miễn phí** của AWS, giúp tự động container hóa (containerize) các ứng dụng Java và .NET đang chạy trên máy chủ vật lý hoặc máy ảo — mà không cần thay đổi code. Đây là bước Replatform trong chiến lược 7Rs: giữ nguyên code nhưng chuyển sang nền tảng container hiện đại (ECS Fargate hoặc EKS).

## 📚 Mục Lục (Table of Contents)

1. [App2Container Là Gì?](#app2container-là-gì)
2. [Ứng Dụng Nào Phù Hợp?](#ứng-dụng-nào-phù-hợp)
3. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
4. [Quy Trình 4 Bước Chi Tiết](#quy-trình-4-bước-chi-tiết)
5. [Cài Đặt Và Cấu Hình](#cài-đặt-và-cấu-hình)
6. [Phân Tích analysis.json](#phân-tích-analysisjson)
7. [ECS vs EKS — Chọn Target Nào?](#ecs-vs-eks--chọn-target-nào)
8. [CI/CD Pipeline Tự Động](#cicd-pipeline-tự-động)
9. [Xử Lý Các Trường Hợp Đặc Biệt](#xử-lý-các-trường-hợp-đặc-biệt)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 📦 App2Container Là Gì?

### Khái Niệm Cơ Bản

```
APP2CONTAINER (A2C):

Loại: CLI tool (công cụ dòng lệnh), miễn phí
Cài đặt trên: Server nguồn (source server) có ứng dụng cần containerize
Hỗ trợ OS: Linux (Ubuntu, CentOS, RHEL, Amazon Linux)
             Windows Server (cho .NET apps)

Input (Đầu Vào):
├── Ứng dụng Java đang chạy trên JVM (Java Virtual Machine — Máy Ảo Java)
└── Ứng dụng .NET đang chạy trên IIS (Internet Information Services)

Output (Đầu Ra):
├── Dockerfile
├── Container image được push lên Amazon ECR
│   (Elastic Container Registry — Kho Lưu Trữ Container Đàn Hồi)
├── ECS task definition (cấu hình task ECS) HOẶC EKS deployment manifest
└── CloudFormation template + CodePipeline CI/CD (tùy chọn)

Điều Kiện:
├── Ứng dụng phải ĐANG CHẠY khi thực hiện discover
├── Cần quyền root/administrator trên server nguồn
└── Server nguồn cần kết nối Internet hoặc VPC Endpoint để gọi AWS APIs
```

### Tại Sao A2C Quan Trọng?

```
VẤN ĐỀ CONTAINERIZE THỦ CÔNG:

❌ Không dùng A2C:
├── Developer phải tự viết Dockerfile
├── Phải tìm hiểu tất cả dependencies (thư viện, port, env vars)
├── Dễ bỏ sót config files, JVM arguments, JAVA_OPTS
├── Tốn 1-3 ngày/ứng dụng
└── Error-prone: container không hoạt động đúng như server gốc

✅ Dùng A2C:
├── Tự động phát hiện tất cả dependencies
├── Tự động tạo Dockerfile đúng
├── Bao gồm toàn bộ JVM settings và runtime configuration
├── Tốn vài giờ/ứng dụng
└── Consistent và reproducible
```

---

## 🎯 Ứng Dụng Nào Phù Hợp?

### Hỗ Trợ Tốt

```
ỨNG DỤNG ĐƯỢC HỖ TRỢ TỐT:

Java:
├── Apache Tomcat (phiên bản 7, 8, 9, 10)
├── JBoss / WildFly (Application Server của Red Hat)
├── WebLogic (Oracle Application Server)
├── WebSphere (IBM Application Server)
├── Spring Boot standalone JAR (tự chứa)
└── Jetty, Undertow embedded servers

.NET (trên Windows Server):
├── ASP.NET MVC / Web Forms trên IIS
├── ASP.NET Web API
├── WCF (Windows Communication Foundation — Nền Tảng Giao Tiếp Windows) trên IIS
└── .NET Framework 3.5, 4.x applications
```

### Không Phù Hợp Hoặc Cần Điều Chỉnh Thêm

```
TRƯỜNG HỢP KHÓ HOẶC KHÔNG PHÙ HỢP:

├── Ứng dụng GUI desktop (WinForms, WPF) → không phải web app
├── Ứng dụng dùng local filesystem làm storage chính
│   → Cần refactor để dùng EFS hoặc S3 trước
├── Ứng dụng giữ session state trong memory (in-memory session)
│   → Cần chuyển sang ElastiCache Redis trước containerize
├── Ứng dụng cần Windows-specific drivers hoặc COM components
│   → Có thể dùng Windows Containers nhưng phức tạp hơn
├── Batch jobs không phải web server → App2Container không detect được
│   → Dùng AWS Batch hoặc Step Functions thay thế
└── Ứng dụng cần multi-container architecture ngay từ đầu
    → Cần thiết kế thủ công
```

---

## 🏗️ Kiến Trúc Tổng Quan

```
LUỒNG APP2CONTAINER ĐẦY ĐỦ:

SOURCE SERVER                        AWS CLOUD
┌─────────────────────┐            ┌────────────────────────────────────────┐
│                     │            │                                        │
│  Tomcat App         │            │  Amazon ECR                            │
│  (Java Web App)     │            │  (Container Registry)                  │
│                     │            │  ┌────────────────────────────────┐    │
│  app2container      │──[push]───►│  │ Container Image               │    │
│  CLI tool           │            │  └─────────────────┬──────────────┘    │
│                     │            │                    │ pull              │
└─────────────────────┘            │  ┌─────────────────▼──────────────┐    │
                                   │  │ ECS Fargate Service           │    │
                                   │  │ hoặc EKS Deployment            │    │
                                   │  └─────────────────┬──────────────┘    │
                                   │                    │                   │
                                   │  ┌─────────────────▼──────────────┐    │
                                   │  │ Application Load Balancer      │    │
                                   │  └────────────────────────────────┘    │
                                   │                                        │
                                   │  CloudFormation + CodePipeline         │
                                   │  (Infrastructure as Code +             │
                                   │   CI/CD Pipeline)                      │
                                   └────────────────────────────────────────┘
```

---

## 🔢 Quy Trình 4 Bước Chi Tiết

### Bước 1: DISCOVER (Khám Phá Ứng Dụng)

```bash
# Cài App2Container trên server nguồn
curl -o AWSApp2Container-installer-linux.tar.gz \
  https://app2container-release.s3.amazonaws.com/\
latest/linux/AWSApp2Container-installer-linux.tar.gz

tar xvf AWSApp2Container-installer-linux.tar.gz
sudo ./install.sh

# Khởi tạo (lần đầu tiên)
app2container init
# → Nhập: AWS region, S3 bucket để lưu artifacts, IAM role ARN

# Discover tất cả ứng dụng đang chạy
app2container discover
```

```
OUTPUT CỦA DISCOVER:

Application-Id: java-tomcat-ab1cd2ef
  Process: tomcat (PID 12345)
  Server: /opt/tomcat/bin/catalina.sh
  Port: 8080

Application-Id: java-spring-gh3ij4kl
  Process: java (PID 23456)
  Cmd: java -jar /opt/myapp/app.jar
  Port: 8443

Application-Id: dotnet-iis-mn5op6qr
  Process: w3wp.exe (PID 34567)
  Site: Default Web Site
  Port: 80
```

### Bước 2: ANALYZE (Phân Tích Chi Tiết)

```bash
# Phân tích ứng dụng cụ thể
app2container analyze --application-id java-tomcat-ab1cd2ef
```

```
A2C TỰ ĐỘNG PHÁT HIỆN VÀ GHI VÀO analysis.json:

├── JVM version: OpenJDK 11.0.18
├── JVM arguments: -Xms512m -Xmx2048m -XX:+UseG1GC
├── Classpath và deployed WAR/JAR files
├── Open ports: 8080 (HTTP), 8443 (HTTPS), 8009 (AJP)
├── Environment variables (biến môi trường):
│   ├── DATABASE_URL=jdbc:mysql://db-server:3306/mydb
│   ├── REDIS_HOST=cache-server
│   └── LOG_LEVEL=INFO
├── File system dependencies:
│   ├── /opt/tomcat/conf/server.xml
│   ├── /opt/tomcat/conf/context.xml
│   └── /var/log/tomcat/ (log directory)
├── Network dependencies (từ connection tracking):
│   ├── db-server:3306 (MySQL database)
│   ├── cache-server:6379 (Redis cache)
│   └── api.partner.com:443 (External API)
└── Recommended target: ECS Fargate
```

### Bước 3: CONTAINERIZE (Container Hóa)

```bash
# Containerize ứng dụng
app2container containerize --application-id java-tomcat-ab1cd2ef
```

```
A2C TỰ ĐỘNG:

1. Tạo Dockerfile:
┌────────────────────────────────────────────────────────────┐
│ FROM amazoncorretto:11                                      │
│                                                            │
│ # Copy application files                                   │
│ COPY opt/tomcat /opt/tomcat                                │
│ COPY extracted-artifacts/ /opt/tomcat/webapps/             │
│                                                            │
│ # Set environment                                          │
│ ENV JAVA_OPTS="-Xms512m -Xmx2048m -XX:+UseG1GC"           │
│ ENV CATALINA_HOME=/opt/tomcat                              │
│                                                            │
│ EXPOSE 8080                                                │
│ CMD ["/opt/tomcat/bin/catalina.sh", "run"]                 │
└────────────────────────────────────────────────────────────┘

2. Build Docker image

3. Push lên Amazon ECR:
   123456789.dkr.ecr.ap-southeast-1.amazonaws.com/
   java-tomcat-ab1cd2ef:latest

4. Tạo deployment/task-definition.json (ECS Task Definition)
```

### Bước 4: GENERATE APP DEPLOYMENT (Tạo Pipeline)

```bash
# Tạo CloudFormation + CI/CD pipeline
app2container generate app-deployment \
  --application-id java-tomcat-ab1cd2ef \
  --deploy  # tự động deploy luôn
```

```
A2C TẠO:

CloudFormation Stacks (Ngăn Xếp Hạ Tầng):
├── VPC + Subnets (nếu chưa có)
├── ECS Cluster + Service
├── Application Load Balancer (ALB)
├── Target Group + Health Checks
├── IAM Roles (quyền cho ECS task)
└── CloudWatch Log Group (nhóm log)

CodePipeline CI/CD:
├── Source: CodeCommit hoặc S3 (nơi lưu source code)
├── Build: CodeBuild (build Docker image mới)
├── Deploy: Deploy lên ECS với blue/green deployment
└── Notification: SNS alerts cho pipeline events
```

---

## ⚙️ Cài Đặt Và Cấu Hình

### IAM Permissions Cần Thiết

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "App2ContainerPermissions",
      "Effect": "Allow",
      "Action": [
        "ecr:*",
        "ecs:*",
        "cloudformation:*",
        "codepipeline:*",
        "codebuild:*",
        "s3:*",
        "iam:PassRole",
        "elasticloadbalancing:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### File Cấu Hình init

```yaml
# ~/.app2container/config
version: v1
appVersion: "1.0"
awsAccountId: "123456789012"
awsRegion: ap-southeast-1
s3Bucket: my-a2c-artifacts-bucket
# containerBaseImage: tùy chỉnh base image nếu cần
# remoteContainerization: true  # để chạy containerize từ xa qua SSM
```

### Remote Containerization (Container Hóa Từ Xa)

```
REMOTE MODE — KHÔNG CẦN CÀI A2C TRỰC TIẾP TRÊN SOURCE SERVER:

Thay vì chạy A2C trực tiếp trên source server,
dùng AWS Systems Manager (SSM) để chạy từ xa:

1. Source server cài SSM Agent và có IAM role phù hợp
2. A2C worker machine (EC2 hoặc laptop) gửi lệnh qua SSM
3. A2C tự động SSH vào source server qua SSM Session Manager
4. Thực hiện discover/analyze/containerize từ xa

Lợi ích:
├── Không cần mở SSH port trên source server
├── Audit trail đầy đủ qua AWS CloudTrail
└── Phù hợp môi trường có security restrictions cao
```

---

## 📄 Phân Tích analysis.json

### Các Trường Quan Trọng Cần Review

```json
{
  "containerParameters": {
    "imageRepository": "123456789.dkr.ecr.ap-southeast-1.amazonaws.com/my-app",
    "imageTag": "latest",
    "containerBaseImage": "amazoncorretto:11",
    "appExcludedFiles": [],
    "appSpecificFiles": ["/opt/tomcat/conf/server.xml"],
    "envVariables": [
      "DATABASE_URL=jdbc:mysql://db-server:3306/mydb",
      "REDIS_HOST=cache-server"
    ],
    "jvmOpts": "-Xms512m -Xmx2048m -XX:+UseG1GC",
    "containerPorts": [{"localPort": 8080, "protocol": "tcp"}]
  },
  "ecsParameters": {
    "taskDefinitionName": "java-tomcat-td",
    "taskDefinitionCpu": "2048",
    "taskDefinitionMemory": "4096",
    "taskRoleArn": "",
    "containerMemory": 3072,
    "containerCpu": 1024,
    "enableECSManagedTags": true,
    "enableTaskMetadata": true
  },
  "eksParameters": {
    "deploymentName": "java-tomcat-deployment",
    "serviceType": "LoadBalancer",
    "cpuRequest": "1000m",
    "memoryRequest": "3Gi",
    "cpuLimit": "2000m",
    "memoryLimit": "4Gi",
    "replicaCount": 2
  }
}
```

### Những Gì Cần Chỉnh Sửa Thủ Công

```
SỬA TRƯỚC KHI CONTAINERIZE:

1. envVariables — CỰC KỲ QUAN TRỌNG:
   ❌ Trước: "DATABASE_URL=jdbc:mysql://db-server:3306/mydb"
             (hardcoded hostname on-premises)
   ✅ Sau:   Xóa khỏi envVariables, dùng AWS Secrets Manager:
             Tên secret: /myapp/prod/database-url
             Tham chiếu trong task definition

2. containerPorts — Kiểm tra tất cả port cần expose:
   ├── HTTP: 8080
   ├── HTTPS: 8443 (nếu app tự terminate TLS)
   └── Management: 8009 (AJP — Apache JServ Protocol) — thường KHÔNG expose

3. appExcludedFiles — Loại trừ file không cần thiết:
   └── Thêm: ["/var/log/tomcat/**", "/tmp/**", "*.pid"]

4. containerBaseImage — Chọn image phù hợp:
   ├── amazoncorretto:11 (Amazon's OpenJDK distribution)
   ├── eclipse-temurin:11-jre (tối ưu hóa runtime only)
   └── Tránh dùng full JDK nếu chỉ cần JRE runtime

5. taskDefinitionMemory và containerMemory:
   └── Kiểm tra actual memory usage từ CloudWatch metrics
       trước khi set — không nên set quá cao
```

---

## 🎯 ECS vs EKS — Chọn Target Nào?

### Bảng So Sánh

```
┌──────────────────────────────┬──────────────────────────┬────────────────────────────┐
│ Tiêu Chí                     │ ECS Fargate              │ EKS (Kubernetes)           │
│                              │ (Khuyến Nghị cho A2C)    │                            │
├──────────────────────────────┼──────────────────────────┼────────────────────────────┤
│ Phù hợp cho                 │ Team mới với container,  │ Team đã biết Kubernetes,   │
│                              │ ít container (<20 service)│ nhiều service phức tạp    │
├──────────────────────────────┼──────────────────────────┼────────────────────────────┤
│ Complexity                   │ Thấp — AWS quản lý hoàn  │ Cao hơn — cần hiểu K8s    │
│ (độ phức tạp)               │ toàn (serverless)         │ concepts                  │
├──────────────────────────────┼──────────────────────────┼────────────────────────────┤
│ Operational overhead         │ Rất thấp                  │ Trung bình đến cao        │
│ (gánh nặng vận hành)        │                           │                            │
├──────────────────────────────┼──────────────────────────┼────────────────────────────┤
│ Chi phí                      │ Pay per task (trả theo    │ Control plane: $0.10/giờ  │
│                              │ task đang chạy)           │ + worker node EC2 costs   │
├──────────────────────────────┼──────────────────────────┼────────────────────────────┤
│ Auto Scaling                 │ ECS Service Auto Scaling  │ HPA — Horizontal Pod       │
│                              │ + Application Auto        │ Autoscaler — Tự Động Mở   │
│                              │ Scaling                   │ Rộng Pod Theo Chiều Ngang │
├──────────────────────────────┼──────────────────────────┼────────────────────────────┤
│ Portability                  │ AWS-specific              │ Portable sang bất kỳ K8s  │
│ (khả năng di chuyển)        │                           │ cluster nào               │
├──────────────────────────────┼──────────────────────────┼────────────────────────────┤
│ Khi nào chọn                 │ - Bắt đầu với containers  │ - Multi-cloud requirement  │
│                              │ - Đơn giản hóa operations │ - Team có K8s expertise    │
│                              │ - Ít service (<20)        │ - Nhiều service (>20)      │
└──────────────────────────────┴──────────────────────────┴────────────────────────────┘
```

### Khuyến Nghị Thực Tế

```
QUYẾT ĐỊNH ECS vs EKS:

Bắt đầu với ECS Fargate nếu:
├── Team chưa có Kubernetes experience
├── Số lượng service ít (< 20 microservices)
├── Muốn operational simplicity cao nhất
└── Timeline ngắn — cần go-live nhanh

Chọn EKS nếu:
├── Tổ chức đã có Kubernetes standard
├── Cần multi-cloud strategy (Azure AKS, GKE cũng chạy K8s)
├── Service mesh (Istio, Linkerd) đã được sử dụng
└── Cần advanced scheduling (ví dụ GPU instances cho ML)
```

---

## 🔁 CI/CD Pipeline Tự Động

### Pipeline Được A2C Tạo

```
CODEPIPELINE ARCHITECTURE (KIẾN TRÚC PIPELINE):

Source Stage (Giai Đoạn Nguồn):
  CodeCommit repo hoặc S3 bucket
  └─► Trigger on commit/push

Build Stage (Giai Đoạn Xây Dựng):
  CodeBuild project:
  ├── Pull source code
  ├── Run unit tests (nếu có)
  ├── Build Docker image
  ├── Scan image với Amazon ECR Enhanced Scanning
  │   (kiểm tra lỗ hổng bảo mật trong container image)
  └── Push image mới lên ECR với tag commit SHA

Deploy Stage (Giai Đoạn Triển Khai):
  ECS Blue/Green Deployment (Triển Khai Xanh/Xanh Lá):
  ├── "Blue" = version đang chạy production
  ├── "Green" = version mới được deploy
  ├── Load balancer chuyển traffic sang Green sau health check
  └── Rollback tự động nếu health check thất bại

Approval Stage (tùy chọn):
  ├── Manual approval gate trước khi deploy production
  └── SNS notification cho approver
```

### Cải Thiện Pipeline Sau A2C

```
NHỮNG GÌ CẦN THÊM VÀO PIPELINE SAU KHI A2C TẠO:

1. Secrets Management (Quản Lý Bí Mật):
   ├── Xóa env vars hardcoded trong task definition
   └── Dùng AWS Secrets Manager hoặc SSM Parameter Store:
       Task definition: "secrets": [
         {"name": "DATABASE_URL", "valueFrom": "arn:aws:ssm:..."}
       ]

2. Container Security Scanning:
   ├── Bật ECR image scanning tự động
   └── Thêm Trivy hoặc Snyk scan vào CodeBuild stage

3. CloudWatch Monitoring:
   ├── Custom metrics cho application-level health
   ├── Log metric filters cho error patterns
   └── Dashboard cho ECS service metrics

4. Auto Scaling Policy:
   └── Target tracking: CPU utilization 70% target
       Min: 2 tasks, Max: 10 tasks
```

---

## 🔧 Xử Lý Các Trường Hợp Đặc Biệt

### Ứng Dụng Dùng Local File System

```
VẤN ĐỀ: Container là ephemeral (tạm thời) — file trên local disk bị mất khi container restart

GIẢI PHÁP:

Cho static files và user uploads:
└── Mount Amazon EFS (Elastic File System — Hệ Thống File Đàn Hồi):
    ECS task definition thêm:
    "volumes": [{"name": "efs-volume",
                 "efsVolumeConfiguration": {"fileSystemId": "fs-xxxxx"}}]
    "mountPoints": [{"sourceVolume": "efs-volume",
                     "containerPath": "/opt/tomcat/webapps/uploads"}]

Cho temporary files và session data:
└── Refactor sang ElastiCache Redis:
    - Session: Spring Session với Redis backend
    - Cache: Caffeine cache → Redis cache

Cho configuration files:
└── AWS AppConfig hoặc SSM Parameter Store
    Load config khi container khởi động
```

### Ứng Dụng Dùng In-Memory Session

```
VẤN ĐỀ: Nhiều container instances → session không share

TRƯỚC KHI CONTAINERIZE — REFACTOR SESSION:

Java Spring (Tomcat):
1. Thêm Spring Session dependency:
   <dependency>
     <groupId>org.springframework.session</groupId>
     <artifactId>spring-session-data-redis</artifactId>
   </dependency>

2. Config Redis:
   spring.session.store-type=redis
   spring.redis.host=${REDIS_HOST}
   spring.redis.port=6379

3. AWS: Tạo ElastiCache Redis cluster
   └── ElastiCache Redis → endpoint làm REDIS_HOST

Sau đó mới containerize — container bây giờ stateless
```

### Ứng Dụng WebLogic / WebSphere

```
ORACLE WEBLOGIC / IBM WEBSPHERE:

Thách thức:
├── Licensing phức tạp — Oracle WebLogic tính phí theo processor
├── WebSphere có nhiều proprietary features
└── A2C hỗ trợ nhưng cần thêm manual configuration

Khuyến nghị:
├── WebLogic: Dùng Oracle WebLogic Kubernetes Operator
│   hoặc migrate sang Payara/WildFly (open source) trước A2C
├── WebSphere: Dùng IBM WebSphere Liberty (lighter version)
│   hoặc migrate sang OpenLiberty trước A2C
└── Cân nhắc: Refactor sang Spring Boot thay vì container hóa legacy AS
```

---

## ✅ Best Practices

### Trước Khi Containerize

```
CHUẨN BỊ:

□ Kiểm tra ứng dụng hoạt động đúng trên server nguồn
□ Chạy app2container discover và review toàn bộ danh sách
□ Loại trừ ứng dụng không phù hợp (GUI, batch jobs)
□ Backup server nguồn trước khi thực hiện bất kỳ thay đổi gì
□ Đảm bảo server nguồn có Internet hoặc VPC Endpoint access
□ Chuẩn bị S3 bucket cho A2C artifacts
□ Tạo ECR repository cho container images
□ Review và chỉnh sửa analysis.json trước khi containerize
```

### Sau Khi Containerize

```
VALIDATE:

□ Test container image locally trước khi push lên ECS/EKS:
  docker run -p 8080:8080 \
    -e DATABASE_URL=... \
    <ecr-image-uri>

□ Smoke test (kiểm tra cơ bản): curl http://localhost:8080/health
□ Load test: So sánh response time với server gốc
□ Review CloudWatch logs sau khi deploy — tìm errors, warnings
□ Test rollback procedure: Deploy phiên bản cũ, xác nhận rollback thành công
□ Cấu hình health check đúng cách trong target group
□ Cấu hình Auto Scaling policy
□ Test scale-out: Tăng tải → kiểm tra thêm task tự động
```

### Security Best Practices

```
BẢO MẬT:

□ Không để secrets trong Dockerfile hoặc environment variables rõ ràng
  → Dùng Secrets Manager hoặc SSM Parameter Store
□ Run container với non-root user:
  Dockerfile: RUN useradd -r -u 1001 appuser && USER appuser
□ Bật ECR image vulnerability scanning tự động
□ Dùng immutable image tags (SHA digest thay vì "latest")
□ Task execution role: least privilege (quyền tối thiểu cần thiết)
□ Network: Container chỉ expose port cần thiết
□ Enable VPC endpoint cho ECR, S3, Secrets Manager
  (traffic không qua Internet)
```

---

## 🎤 Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: App2Container là gì và nó giải quyết vấn đề gì?**

> App2Container (A2C) là CLI tool miễn phí của AWS, tự động container hóa ứng dụng Java và .NET đang chạy trên máy chủ, mà không cần sửa code.
>
> Vấn đề nó giải quyết: Container hóa thủ công tốn nhiều thời gian, dễ bỏ sót dependencies, JVM settings và environment variables. A2C tự động phát hiện tất cả các thông tin này bằng cách phân tích process đang chạy, sau đó tạo Dockerfile, push container lên ECR, và tạo ECS/EKS deployment configuration.
>
> Kết quả: Thay vì mất 1-3 ngày/ứng dụng để containerize thủ công, A2C rút ngắn xuống còn vài giờ và ít lỗi hơn.

---

**Q: Tại sao A2C yêu cầu ứng dụng phải đang chạy khi thực hiện discover?**

> Vì A2C phân tích **running process** để thu thập thông tin chính xác:
> - **JVM arguments thực tế**: Không chỉ đọc config file mà đọc trực tiếp từ process `/proc/<pid>/cmdline`
> - **Open ports**: Từ `netstat` hoặc `ss` để biết port nào app thực sự listening
> - **Environment variables thực tế**: Từ `/proc/<pid>/environ` — bao gồm cả env vars được set bởi init scripts
> - **Outbound network connections**: Phát hiện dependency đến database, cache, external APIs
>
> Nếu app không chạy, A2C chỉ có thể đọc static config files → dễ bỏ sót runtime configuration quan trọng.

---

### Câu Hỏi Nâng Cao

**Q: Sau khi A2C tạo container, ứng dụng hoạt động đúng trên server nhưng bị lỗi khi chạy trên ECS. Debug như thế nào?**

> Quy trình debug theo thứ tự:
>
> 1. **Kiểm tra CloudWatch Logs**: ECS task logs — tìm error messages ngay lúc startup
>
> 2. **Test container image local**:
>    ```bash
>    docker run --env-file .env -p 8080:8080 <image-uri>
>    ```
>    Nếu lỗi local → vấn đề trong Dockerfile hoặc environment variables
>
> 3. **Kiểm tra Secrets Manager / env vars**: Database URL, Redis host có trỏ đúng endpoint AWS không (không còn là hostname on-premises)
>
> 4. **Network connectivity**: ECS task có Security Group cho phép kết nối đến RDS (port 3306), ElastiCache (port 6379) không?
>
> 5. **Health check**: ALB Target Group health check path có đúng không? Ứng dụng có ready trả lời health check ngay sau startup không?
>
> 6. **Memory/CPU limits**: Container có bị OOMKilled (bị giết do hết bộ nhớ) không? Xem ECS Task stopped reason trong console.
>
> Nguyên nhân phổ biến nhất: Database connection string vẫn trỏ đến on-premises hostname → không reachable từ VPC AWS.

---

**Q: Khi nào nên dùng App2Container so với viết Dockerfile thủ công?**

> **Dùng App2Container** khi:
> - Có nhiều ứng dụng cần containerize (>5 apps) — tiết kiệm thời gian đáng kể
> - Team không rành Docker/containerization — A2C giảm learning curve
> - Ứng dụng Java Tomcat/JBoss/.NET IIS tiêu chuẩn
> - Cần CI/CD pipeline nhanh chóng đi kèm
>
> **Viết Dockerfile thủ công** khi:
> - Ứng dụng có kiến trúc phức tạp (multi-process, custom init)
> - Cần tối ưu hóa image size nghiêm túc (multi-stage build)
> - Team muốn kiểm soát hoàn toàn Dockerfile
> - A2C không hỗ trợ loại ứng dụng đó
>
> Nguyên tắc: A2C là điểm khởi đầu tốt — có thể dùng A2C để tạo Dockerfile base, sau đó tối ưu thủ công.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
**Thuộc Module:** 08-modernization
