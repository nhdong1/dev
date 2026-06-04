# 🛠️ Hands-on Exercises — Bài Tập Thực Hành AWS Networking

> Bộ bài tập thực hành theo thứ tự từ cơ bản đến nâng cao. Mỗi bài tập có mục tiêu học rõ ràng, hướng dẫn step-by-step, và câu hỏi kiểm tra hiểu biết.

---

## 📋 Yêu Cầu Trước Khi Bắt Đầu

```
✅ AWS Account (Free Tier đủ cho hầu hết bài tập)
✅ AWS CLI configured với credentials
✅ Terraform >= 1.5 (tùy chọn — cho bài tập nâng cao)
✅ Cài đặt: aws cli, jq, curl, nc (netcat)

Lưu ý chi phí:
- Hầu hết bài tập dùng Free Tier hoặc < $1/ngày
- Bài tập Direct Connect KHÔNG thể thực hành free (chỉ học lý thuyết)
- Nhớ terminate (xóa) resources sau mỗi bài tập để tránh phí phát sinh
```

---

## 🔵 Lab 1: Tạo VPC 3-tier Từ Đầu (Không Dùng Wizard)

**Mục tiêu học:**
- Hiểu sâu VPC components qua thực hành thủ công
- Nắm vững sự khác biệt Public/Private subnets qua config thực tế
- Hiểu cách route tables điều khiển traffic

**Thời gian:** 60-90 phút

---

### Bước 1: Tạo VPC

```bash
# Tạo VPC với CIDR 10.0.0.0/16
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-prod-vpc}]' \
  --query 'Vpc.VpcId' \
  --output text)

echo "VPC ID: $VPC_ID"

# Bật DNS hostnames (cần cho EC2 có hostname)
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames

# Bật DNS support
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-support
```

### Bước 2: Tạo Subnets

```bash
AZ1="ap-southeast-1a"
AZ2="ap-southeast-1b"

# Public Subnets (cho Load Balancer, NAT Gateway)
PUBLIC_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone $AZ1 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PUBLIC_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone $AZ2 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1b}]' \
  --query 'Subnet.SubnetId' --output text)

# Private Subnets (cho Application Servers)
PRIVATE_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.10.0/24 \
  --availability-zone $AZ1 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.11.0/24 \
  --availability-zone $AZ2 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-1b}]' \
  --query 'Subnet.SubnetId' --output text)

# Database Subnets (isolated — không cần Internet)
DB_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.20.0/24 \
  --availability-zone $AZ1 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=db-subnet-1a}]' \
  --query 'Subnet.SubnetId' --output text)

DB_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.21.0/24 \
  --availability-zone $AZ2 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=db-subnet-1b}]' \
  --query 'Subnet.SubnetId' --output text)

echo "Public: $PUBLIC_SUBNET_1, $PUBLIC_SUBNET_2"
echo "Private: $PRIVATE_SUBNET_1, $PRIVATE_SUBNET_2"
echo "Database: $DB_SUBNET_1, $DB_SUBNET_2"
```

### Bước 3: Internet Gateway và Route Tables

```bash
# Tạo và attach Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway \
  --vpc-id $VPC_ID \
  --internet-gateway-id $IGW_ID

echo "IGW: $IGW_ID"

# Tạo Public Route Table
PUBLIC_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

# Thêm route 0.0.0.0/0 → IGW
aws ec2 create-route \
  --route-table-id $PUBLIC_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associate public subnets với public route table
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET_1
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET_2

# Bật auto-assign public IP cho public subnets
aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_SUBNET_1 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_SUBNET_2 --map-public-ip-on-launch
```

### Bước 4: NAT Gateway

```bash
# Tạo Elastic IP cho NAT Gateway
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=nat-eip}]' \
  --query 'AllocationId' --output text)

# Tạo NAT Gateway trong PUBLIC subnet
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_1 \
  --allocation-id $EIP_ALLOC \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=my-nat-gw}]' \
  --query 'NatGateway.NatGatewayId' --output text)

echo "NAT GW: $NAT_GW_ID — Chờ available..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID
echo "NAT GW ready!"

# Tạo Private Route Table với route qua NAT Gateway
PRIVATE_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id $PRIVATE_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_ID

aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $PRIVATE_SUBNET_1
aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $PRIVATE_SUBNET_2
```

### Kiểm Tra Kết Quả

```bash
# Verify VPC structure
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query 'Vpcs[0].{ID:VpcId,CIDR:CidrBlock,State:State}'

# Verify subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].{Name:Tags[?Key==`Name`]|[0].Value,Subnet:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone}'

# Verify route tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].{Name:Tags[?Key==`Name`]|[0].Value,Routes:Routes[*].{Dest:DestinationCidrBlock,Via:GatewayId}}'
```

### Câu Hỏi Kiểm Tra Hiểu Biết

1. Tại sao Database subnets không có route đến NAT Gateway?
2. Nếu bạn có 3 AZs, bạn có cần 3 NAT Gateways không? Tại sao?
3. Sự khác biệt giữa "main route table" và custom route table?
4. Khi bạn associate một subnet với route table mới, chuyện gì xảy ra với association cũ?

### Dọn Dẹp

```bash
# QUAN TRỌNG: Xóa theo thứ tự ngược lại để tránh dependency errors

# 1. Xóa NAT Gateway (tốn tiền nếu giữ lại!)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_ID
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_GW_ID

# 2. Release Elastic IP
aws ec2 release-address --allocation-id $EIP_ALLOC

# 3. Detach và xóa Internet Gateway
aws ec2 detach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 4. Xóa route tables (custom ones, không phải main)
aws ec2 delete-route-table --route-table-id $PUBLIC_RT
aws ec2 delete-route-table --route-table-id $PRIVATE_RT

# 5. Xóa subnets
for subnet in $PUBLIC_SUBNET_1 $PUBLIC_SUBNET_2 $PRIVATE_SUBNET_1 $PRIVATE_SUBNET_2 $DB_SUBNET_1 $DB_SUBNET_2; do
  aws ec2 delete-subnet --subnet-id $subnet
done

# 6. Xóa VPC
aws ec2 delete-vpc --vpc-id $VPC_ID
echo "Cleanup hoàn thành!"
```

---

## 🟡 Lab 2: Security Groups và Network ACLs — Debug Traffic

**Mục tiêu học:**
- Trải nghiệm trực tiếp stateful vs stateless behavior
- Debug connectivity issues theo hệ thống
- Hiểu sâu NACL rule numbering và evaluation order

**Thời gian:** 60-90 phút

---

### Setup Lab Environment

```bash
# Giả sử bạn đã có VPC từ Lab 1
# Tạo 2 EC2 instances để test

# Security Group cho Web Server
WEB_SG=$(aws ec2 create-security-group \
  --group-name "web-sg" \
  --description "Security Group for Web Server" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Allow SSH từ your IP (thay YOUR_IP)
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp --port 22 \
  --cidr "$(curl -s ifconfig.me)/32"

# Allow HTTP/HTTPS từ anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $WEB_SG \
  --protocol tcp --port 80 \
  --cidr 0.0.0.0/0

# Security Group cho App Server (chỉ allow từ Web Server SG)
APP_SG=$(aws ec2 create-security-group \
  --group-name "app-sg" \
  --description "Security Group for App Server" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $APP_SG \
  --protocol tcp --port 8080 \
  --source-group $WEB_SG  # Chỉ allow từ Web Server SG — không phải IP
```

### Thí Nghiệm 1: Stateful Security Group

```bash
# Tạo EC2 Web Server trong Public Subnet
WEB_INSTANCE=$(aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --subnet-id $PUBLIC_SUBNET_1 \
  --security-group-ids $WEB_SG \
  --key-name your-key-pair \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server}]' \
  --query 'Instances[0].InstanceId' --output text)

# SSH vào và test outbound traffic
# Thử kết nối ra Internet (NAT Gateway)
# Xem Security Group KHÔNG cần outbound rule — stateful!
```

### Thí Nghiệm 2: NACL Stateless Behavior

```bash
# Tạo NACL chặt chẽ
CUSTOM_NACL=$(aws ec2 create-network-acl \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=custom-nacl}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Thêm inbound rule — Allow HTTP (port 80)
aws ec2 create-network-acl-entry \
  --network-acl-id $CUSTOM_NACL \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --ingress

# BẬP BẪY: Thêm inbound rule nhưng QUÊN outbound ephemeral ports
# Outbound: chỉ allow HTTP response (port 80) — SẼ BREAK responses!
aws ec2 create-network-acl-entry \
  --network-acl-id $CUSTOM_NACL \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --egress

# Associate NACL với subnet
aws ec2 replace-network-acl-association \
  --association-id $(aws ec2 describe-network-acls \
    --filters "Name=association.subnet-id,Values=$PUBLIC_SUBNET_1" \
    --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' \
    --output text) \
  --network-acl-id $CUSTOM_NACL

# TEST: Kết nối đến web server sẽ FAIL vì thiếu outbound ephemeral rule
# Sau đó thêm outbound rule cho ephemeral ports và test lại:
aws ec2 create-network-acl-entry \
  --network-acl-id $CUSTOM_NACL \
  --rule-number 200 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --egress
# Bây giờ connections work → hiểu rõ stateless!
```

---

## 🟢 Lab 3: ALB với Path-based và Host-based Routing

**Mục tiêu học:**
- Cấu hình ALB listener rules phức tạp
- Understand Target Groups và Health Checks
- Implement SSL/TLS với ACM

**Thời gian:** 90 phút

---

### Tạo Target Groups

```bash
# Target Group cho API service
API_TG=$(aws elbv2 create-target-group \
  --name api-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id $VPC_ID \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Target Group cho Web service
WEB_TG=$(aws elbv2 create-target-group \
  --name web-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --health-check-path /index.html \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
```

### Tạo ALB và Listener Rules

```bash
# Security Group cho ALB
ALB_SG=$(aws ec2 create-security-group \
  --group-name alb-sg \
  --description "ALB Security Group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0

# Tạo ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name my-app-alb \
  --subnets $PUBLIC_SUBNET_1 $PUBLIC_SUBNET_2 \
  --security-groups $ALB_SG \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

echo "ALB ARN: $ALB_ARN"
aws elbv2 wait load-balancer-available --load-balancer-arns $ALB_ARN

# Tạo Default Listener (HTTP → Redirect to HTTPS)
LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}' \
  --query 'Listeners[0].ListenerArn' --output text)

# Path-based routing rules:
# Rule 1: /api/* → API Target Group
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 100 \
  --conditions Field=path-pattern,Values='/api/*' \
  --actions Type=forward,TargetGroupArn=$API_TG

# Rule 2: /health → Return 200 directly (không cần backend)
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 50 \
  --conditions Field=path-pattern,Values='/health' \
  --actions 'Type=fixed-response,FixedResponseConfig={StatusCode=200,ContentType=text/plain,MessageBody=OK}'
```

### Kiểm Tra Health Checks

```bash
# Xem trạng thái health check của targets
aws elbv2 describe-target-health \
  --target-group-arn $API_TG \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}'

# Giải thích các trạng thái:
# healthy: Target đang nhận traffic
# unhealthy: Health check fail — không nhận traffic
# initial: Mới register, đang chờ đủ healthy checks
# draining: Đang deregister — waiting for in-flight requests
# unused: Target group chưa được dùng bởi listener rule
```

---

## 🔴 Lab 4: CloudFront Distribution với S3 Origin và OAC

**Mục tiêu học:**
- Bảo vệ S3 bucket chỉ cho phép access qua CloudFront
- Cấu hình cache behaviors khác nhau cho từng path
- Test cache hit/miss behavior

**Thời gian:** 60-90 phút

---

### Bước 1: Setup S3 Bucket

```bash
BUCKET_NAME="my-cf-demo-$(date +%s)"
REGION="ap-southeast-1"

# Tạo S3 bucket
aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION

# QUAN TRỌNG: Block ALL public access
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Upload sample files
echo "<html><body><h1>Hello from CloudFront!</h1></body></html>" > /tmp/index.html
echo "{"status":"ok"}" > /tmp/health.json
aws s3 cp /tmp/index.html s3://$BUCKET_NAME/
aws s3 cp /tmp/health.json s3://$BUCKET_NAME/api/health.json
```

### Bước 2: Tạo CloudFront OAC và Distribution

```bash
# Tạo OAC (Origin Access Control — Kiểm Soát Truy Cập Nguồn Gốc)
OAC_CONFIG='{
  "Name": "my-s3-oac",
  "Description": "OAC for S3 origin",
  "SigningProtocol": "sigv4",
  "SigningBehavior": "always",
  "OriginAccessControlOriginType": "s3"
}'

OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config "$OAC_CONFIG" \
  --query 'OriginAccessControl.Id' --output text)

echo "OAC ID: $OAC_ID"

# Tạo CloudFront Distribution
DIST_CONFIG=$(cat <<EOF
{
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3Origin",
      "DomainName": "${BUCKET_NAME}.s3.${REGION}.amazonaws.com",
      "S3OriginConfig": {"OriginAccessIdentity": ""},
      "OriginAccessControlId": "${OAC_ID}"
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3Origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "CacheBehaviors": {
    "Quantity": 1,
    "Items": [{
      "PathPattern": "/api/*",
      "TargetOriginId": "S3Origin",
      "ViewerProtocolPolicy": "https-only",
      "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
      "Compress": false
    }]
  },
  "Comment": "Demo distribution",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "PriceClass": "PriceClass_100"
}
EOF
)

CF_DIST=$(aws cloudfront create-distribution \
  --distribution-config "$DIST_CONFIG" \
  --query 'Distribution.{Id:Id,Domain:DomainName}')

echo "Distribution: $CF_DIST"
```

### Bước 3: Update S3 Bucket Policy

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DIST_ID=$(echo $CF_DIST | jq -r '.Id')

# Bucket policy chỉ cho phép CloudFront đọc
BUCKET_POLICY=$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::${BUCKET_NAME}/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::${ACCOUNT_ID}:distribution/${DIST_ID}"
      }
    }
  }]
}
EOF
)

aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy "$BUCKET_POLICY"
```

### Bước 4: Test Cache Behavior

```bash
DIST_DOMAIN=$(echo $CF_DIST | jq -r '.Domain')

# Chờ distribution deploy (5-10 phút)
echo "Chờ distribution available..."
aws cloudfront wait distribution-deployed --id $DIST_ID

# Test 1: Access file qua CloudFront (lần đầu — cache MISS)
curl -I "https://$DIST_DOMAIN/index.html"
# Tìm header: X-Cache: Miss from cloudfront

# Test 2: Access lần 2 (cache HIT)
curl -I "https://$DIST_DOMAIN/index.html"
# Tìm header: X-Cache: Hit from cloudfront

# Test 3: Access trực tiếp S3 → Phải bị DENIED
aws s3 presign s3://$BUCKET_NAME/index.html  # Không nên access được vì bucket policy
curl "https://${BUCKET_NAME}.s3.${REGION}.amazonaws.com/index.html"
# Expected: 403 Access Denied

echo "Test hoàn thành! CloudFront domain: $DIST_DOMAIN"
```

---

## 🟣 Lab 5: VPC Flow Logs và Debug Kết Nối Với Athena

**Mục tiêu học:**
- Enable và phân tích VPC Flow Logs thực tế
- Dùng Athena để query flow logs như SQL
- Debug connectivity issues bằng data thực

**Thời gian:** 60-90 phút

---

### Bước 1: Enable Flow Logs

```bash
# Tạo S3 bucket để lưu flow logs
FLOW_LOG_BUCKET="vpc-flow-logs-$(date +%s)"
aws s3api create-bucket \
  --bucket $FLOW_LOG_BUCKET \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION

# Enable Flow Logs cho VPC — gửi đến S3
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination "arn:aws:s3:::$FLOW_LOG_BUCKET" \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status}'
```

### Bước 2: Generate Traffic và Query

```bash
# Sau khi có traffic (10-15 phút để logs appear trong S3)
# Tạo Athena table để query

ATHENA_DB="vpc_flow_logs_db"
ATHENA_TABLE="flow_logs"

# Tạo database trong Athena
aws athena start-query-execution \
  --query-string "CREATE DATABASE IF NOT EXISTS $ATHENA_DB" \
  --result-configuration "OutputLocation=s3://$FLOW_LOG_BUCKET/athena-results/"

# Tạo table (thay BUCKET và ACCOUNT_ID)
CREATE_TABLE_SQL="
CREATE EXTERNAL TABLE IF NOT EXISTS ${ATHENA_DB}.${ATHENA_TABLE} (
  version INT,
  account_id STRING,
  interface_id STRING,
  srcaddr STRING,
  dstaddr STRING,
  srcport INT,
  dstport INT,
  protocol BIGINT,
  packets BIGINT,
  bytes BIGINT,
  start BIGINT,
  end BIGINT,
  action STRING,
  log_status STRING
)
PARTITIONED BY (dt STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ' '
LOCATION 's3://${FLOW_LOG_BUCKET}/AWSLogs/${ACCOUNT_ID}/vpcflowlogs/${REGION}/'
TBLPROPERTIES ('skip.header.line.count'='1')
"

aws athena start-query-execution \
  --query-string "$CREATE_TABLE_SQL" \
  --result-configuration "OutputLocation=s3://$FLOW_LOG_BUCKET/athena-results/"
```

### Bước 3: Query Flow Logs

```sql
-- Query 1: Top 10 source IPs với nhiều rejected connections nhất
SELECT srcaddr, COUNT(*) as reject_count
FROM vpc_flow_logs_db.flow_logs
WHERE action = 'REJECT'
GROUP BY srcaddr
ORDER BY reject_count DESC
LIMIT 10;

-- Query 2: Kiểm tra traffic đến database port (debug connectivity)
SELECT srcaddr, dstaddr, dstport, action, COUNT(*) as count
FROM vpc_flow_logs_db.flow_logs
WHERE dstport IN (3306, 5432, 1433)  -- MySQL, PostgreSQL, MSSQL
GROUP BY srcaddr, dstaddr, dstport, action
ORDER BY count DESC;

-- Query 3: Bytes transferred theo destination
SELECT dstaddr, SUM(bytes)/1e9 as gb_transferred
FROM vpc_flow_logs_db.flow_logs
WHERE action = 'ACCEPT'
GROUP BY dstaddr
ORDER BY gb_transferred DESC
LIMIT 20;

-- Query 4: Tìm port scanning (nhiều dstport khác nhau từ cùng srcaddr)
SELECT srcaddr, COUNT(DISTINCT dstport) as port_count
FROM vpc_flow_logs_db.flow_logs
WHERE action = 'REJECT'
GROUP BY srcaddr
HAVING COUNT(DISTINCT dstport) > 20
ORDER BY port_count DESC;
```

---

## 📊 Checklist Hoàn Thành Labs

```
Lab 1 — VPC Fundamentals:
[ ] Tạo VPC với 6 subnets (Public/Private/DB × 2 AZs) thành công
[ ] Verify route tables đúng (public → IGW, private → NAT GW)
[ ] EC2 trong private subnet có thể reach internet qua NAT GW
[ ] EC2 trong DB subnet KHÔNG có internet access
[ ] Trả lời được 4 câu hỏi kiểm tra

Lab 2 — Security Groups & NACLs:
[ ] Demonstrate stateful behavior của Security Group
[ ] Reproduce NACL stateless issue (quên ephemeral ports)
[ ] Fix NACL issue bằng cách thêm outbound rule
[ ] Explain sự khác biệt bằng lời nói (không nhìn notes)

Lab 3 — ALB Routing:
[ ] ALB với path-based routing /api/* → API targets
[ ] Health check configured đúng
[ ] HTTP redirect đến HTTPS working
[ ] Explain listener rule priority

Lab 4 — CloudFront + S3:
[ ] S3 bucket private, block all public access
[ ] CloudFront OAC configured đúng
[ ] Access qua CloudFront OK, direct S3 access → 403
[ ] Cache hit/miss trong response headers

Lab 5 — VPC Flow Logs:
[ ] Flow logs enabled và data xuất hiện trong S3
[ ] Athena table created và queryable
[ ] Chạy ít nhất 3 queries và interpret kết quả
[ ] Identify 1 insight thú vị từ actual data
```

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
