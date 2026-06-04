# 🔧 5 Bài Tập Thực Hành AWS Compute — Hướng Dẫn Step-by-step

> 5 bài lab thực hành từ cơ bản đến nâng cao, bao gồm hướng dẫn chi tiết từng bước, lệnh AWS CLI (Command Line Interface — Giao Diện Dòng Lệnh), và checklist kiểm tra kết quả. Yêu cầu AWS Free Tier account (tài khoản miễn phí).

## 📋 Yêu Cầu Trước Khi Bắt Đầu

```bash
# Kiểm tra AWS CLI đã cài đặt
aws --version

# Cấu hình credentials (thông tin xác thực)
aws configure
# AWS Access Key ID: <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name: ap-southeast-1
# Default output format: json

# Kiểm tra kết nối
aws sts get-caller-identity
```

---

## Lab 1: EC2 Web Server với Auto Scaling Group

**Mục tiêu:** Deploy web server trên EC2, cấu hình Auto Scaling Group, và kiểm tra tự động scale.
**Thời gian:** 60-90 phút
**Chi phí ước tính:** < 1 USD (trong Free Tier)

### Bước 1: Tạo Security Group (Nhóm Bảo Mật)

```bash
# Tạo VPC Security Group cho web server
SG_ID=$(aws ec2 create-security-group \
  --group-name "webserver-sg" \
  --description "Security group for web server" \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"

# Cho phép HTTP (port 80) từ internet
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Cho phép SSH (port 22) từ IP của bạn
MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr "$MY_IP/32"

echo "Security Group configured: $SG_ID"
```

### Bước 2: Tạo Key Pair (Cặp Khóa SSH)

```bash
aws ec2 create-key-pair \
  --key-name "lab-keypair" \
  --query 'KeyMaterial' \
  --output text > lab-keypair.pem

chmod 400 lab-keypair.pem
echo "Key pair created: lab-keypair.pem"
```

### Bước 3: Tạo Launch Template (Mẫu Khởi Chạy)

```bash
# Lấy AMI ID mới nhất cho Amazon Linux 2023
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

echo "AMI ID: $AMI_ID"

# Tạo Launch Template
aws ec2 create-launch-template \
  --launch-template-name "webserver-template" \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"t3.micro\",
    \"KeyName\": \"lab-keypair\",
    \"SecurityGroupIds\": [\"$SG_ID\"],
    \"UserData\": \"$(echo '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2 - $(hostname)</h1>" > /var/www/html/index.html' | base64)\",
    \"TagSpecifications\": [{
      \"ResourceType\": \"instance\",
      \"Tags\": [{\"Key\": \"Name\", \"Value\": \"webserver-lab\"}]
    }]
  }"
```

### Bước 4: Tạo Application Load Balancer (Cân Bằng Tải Ứng Dụng)

```bash
# Lấy danh sách public subnets
SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=defaultForAz,Values=true" \
  --query 'Subnets[*].SubnetId' \
  --output json | jq -r 'join(" ")')

echo "Subnets: $SUBNETS"

# Tạo ALB Security Group
ALB_SG=$(aws ec2 create-security-group \
  --group-name "alb-sg" \
  --description "ALB security group" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0

# Tạo Application Load Balancer
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name "webserver-alb" \
  --subnets $SUBNETS \
  --security-groups $ALB_SG \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# Tạo Target Group (Nhóm Đích)
TG_ARN=$(aws elbv2 create-target-group \
  --name "webserver-tg" \
  --protocol HTTP --port 80 \
  --vpc-id $(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text) \
  --health-check-path "/" \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# Tạo Listener (Bộ Lắng Nghe)
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### Bước 5: Tạo Auto Scaling Group

```bash
TEMPLATE_ID=$(aws ec2 describe-launch-templates \
  --filters "Name=launch-template-name,Values=webserver-template" \
  --query 'LaunchTemplates[0].LaunchTemplateId' \
  --output text)

aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "webserver-asg" \
  --launch-template "LaunchTemplateId=$TEMPLATE_ID,Version=\$Latest" \
  --min-size 2 \
  --max-size 6 \
  --desired-capacity 2 \
  --availability-zones $(aws ec2 describe-availability-zones \
    --query 'AvailabilityZones[*].ZoneName' --output text | tr '\t' ' ') \
  --target-group-arns $TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 60

# Thêm Target Tracking Scaling Policy (Chính Sách Co Giãn Theo Dõi Mục Tiêu)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "webserver-asg" \
  --policy-name "cpu-target-tracking" \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration "{
    \"PredefinedMetricSpecification\": {
      \"PredefinedMetricType\": \"ASGAverageCPUUtilization\"
    },
    \"TargetValue\": 50.0
  }"
```

### Bước 6: Kiểm Tra Kết Quả

```bash
# Lấy DNS của ALB
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --names "webserver-alb" \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

echo "ALB DNS: $ALB_DNS"
echo "Test URL: http://$ALB_DNS"

# Chờ instances healthy
aws elbv2 describe-target-health --target-group-arn $TG_ARN

# Kiểm tra web server
curl http://$ALB_DNS
```

### ✅ Checklist Kiểm Tra

- [ ] ALB trả về HTTP 200 với nội dung từ EC2 instance
- [ ] Refresh nhiều lần thấy hostname khác nhau (load balancing hoạt động)
- [ ] ASG có 2 instances InService
- [ ] CloudWatch metrics hiển thị CPU và request count

### 🧹 Dọn Dẹp (Tránh Phát Sinh Chi Phí)

```bash
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name "webserver-asg" --force-delete
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 delete-target-group --target-group-arn $TG_ARN
aws ec2 delete-launch-template --launch-template-name "webserver-template"
aws ec2 delete-security-group --group-id $SG_ID
aws ec2 delete-security-group --group-id $ALB_SG
```

---

## Lab 2: Lambda Function với SQS Trigger

**Mục tiêu:** Tạo Lambda function xử lý messages từ SQS queue, hiểu Event Source Mapping (Ánh Xạ Nguồn Sự Kiện).
**Thời gian:** 45-60 phút
**Chi phí:** Miễn phí (trong Free Tier)

### Bước 1: Tạo SQS Queue

```bash
# Tạo Standard SQS Queue
QUEUE_URL=$(aws sqs create-queue \
  --queue-name "order-processing-queue" \
  --attributes VisibilityTimeout=30,MessageRetentionPeriod=86400 \
  --query 'QueueUrl' --output text)

# Lấy Queue ARN
QUEUE_ARN=$(aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

echo "Queue URL: $QUEUE_URL"
echo "Queue ARN: $QUEUE_ARN"

# Tạo Dead Letter Queue (Hàng Đợi Thư Chết)
DLQ_URL=$(aws sqs create-queue \
  --queue-name "order-processing-dlq" \
  --query 'QueueUrl' --output text)

DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url $DLQ_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

# Cấu hình DLQ cho main queue (sau 3 lần fail → vào DLQ)
aws sqs set-queue-attributes \
  --queue-url $QUEUE_URL \
  --attributes "{
    \"RedrivePolicy\": \"{\\\"deadLetterTargetArn\\\":\\\"$DLQ_ARN\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"
  }"
```

### Bước 2: Tạo IAM Role cho Lambda

```bash
# Trust policy cho Lambda
cat > lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Tạo IAM Role (Vai Trò IAM)
ROLE_ARN=$(aws iam create-role \
  --role-name "lambda-sqs-role" \
  --assume-role-policy-document file://lambda-trust-policy.json \
  --query 'Role.Arn' --output text)

# Gán policy cho phép Lambda đọc SQS và ghi CloudWatch Logs
aws iam attach-role-policy \
  --role-name "lambda-sqs-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole

echo "Role ARN: $ROLE_ARN"
sleep 10  # Chờ IAM propagate
```

### Bước 3: Viết Lambda Function Code

```bash
# Tạo file lambda_function.py
cat > lambda_function.py << 'EOF'
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def handler(event, context):
    """
    Xử lý order messages từ SQS.
    Mỗi SQS batch có thể chứa nhiều records.
    """
    processed = 0
    failed = []
    
    for record in event['Records']:
        try:
            # Parse SQS message body
            body = json.loads(record['body'])
            order_id = body.get('order_id')
            amount = body.get('amount')
            
            logger.info(f"Processing order: {order_id}, amount: {amount}")
            
            # Giả lập xử lý order
            process_order(order_id, amount)
            processed += 1
            
        except Exception as e:
            logger.error(f"Failed to process record: {str(e)}")
            # Thêm vào danh sách fail để báo cáo partial batch failure
            failed.append({'itemIdentifier': record['messageId']})
    
    logger.info(f"Processed: {processed}, Failed: {len(failed)}")
    
    # Trả về partial batch failure response
    # Chỉ các messageId trong danh sách này sẽ bị retry
    if failed:
        return {'batchItemFailures': failed}
    
    return {'statusCode': 200, 'body': f'Processed {processed} orders'}

def process_order(order_id, amount):
    """Giả lập xử lý order — trong thực tế sẽ gọi database, payment API, v.v."""
    if not order_id or not amount:
        raise ValueError(f"Invalid order data: id={order_id}, amount={amount}")
    
    logger.info(f"Order {order_id} processed successfully, amount: ${amount}")
EOF

# Zip để upload lên Lambda
zip lambda_function.zip lambda_function.py
```

### Bước 4: Tạo Lambda Function

```bash
LAMBDA_ARN=$(aws lambda create-function \
  --function-name "order-processor" \
  --runtime python3.12 \
  --handler lambda_function.handler \
  --role $ROLE_ARN \
  --zip-file fileb://lambda_function.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment "Variables={ENVIRONMENT=development}" \
  --query 'FunctionArn' --output text)

echo "Lambda ARN: $LAMBDA_ARN"

# Tạo Event Source Mapping (Ánh Xạ Nguồn Sự Kiện) — Lambda tự động đọc từ SQS
aws lambda create-event-source-mapping \
  --function-name "order-processor" \
  --event-source-arn $QUEUE_ARN \
  --batch-size 10 \
  --maximum-batching-window-in-seconds 5 \
  --function-response-types ReportBatchItemFailures  # Bật partial batch failure
```

### Bước 5: Test và Giám Sát

```bash
# Gửi test messages vào SQS
for i in {1..5}; do
  aws sqs send-message \
    --queue-url $QUEUE_URL \
    --message-body "{\"order_id\": \"ORD-00$i\", \"amount\": $((i * 10))}"
  echo "Sent order ORD-00$i"
done

# Gửi 1 invalid message để test error handling
aws sqs send-message \
  --queue-url $QUEUE_URL \
  --message-body "{\"invalid\": \"data\"}"

# Chờ Lambda xử lý (20-30 giây)
sleep 30

# Kiểm tra CloudWatch Logs
LOG_GROUP="/aws/lambda/order-processor"
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name $LOG_GROUP \
  --order-by LastEventTime \
  --descending \
  --query 'logStreams[0].logStreamName' \
  --output text)

aws logs get-log-events \
  --log-group-name $LOG_GROUP \
  --log-stream-name "$LOG_STREAM" \
  --query 'events[*].message' \
  --output text

# Kiểm tra DLQ có nhận invalid message không
aws sqs get-queue-attributes \
  --queue-url $DLQ_URL \
  --attribute-names ApproximateNumberOfMessages
```

### ✅ Checklist Kiểm Tra

- [ ] 5 valid orders được xử lý thành công (thấy trong CloudWatch Logs)
- [ ] Invalid message bị fail và sau 3 lần retry vào DLQ
- [ ] DLQ có 1 message (ApproximateNumberOfMessages = 1)
- [ ] Lambda metrics: Invocations, Duration, Errors có trong CloudWatch

---

## Lab 3: ECS Fargate Service với Load Balancer

**Mục tiêu:** Deploy containerized API lên ECS Fargate, hiểu Task Definition, Service, và health checks.
**Thời gian:** 60-90 phút
**Chi phí:** ~0.5 USD/giờ (Fargate không có Free Tier)

### Bước 1: Tạo ECR Repository và Push Image

```bash
# Tạo ECR (Elastic Container Registry — Kho Lưu Container) repository
REPO_URI=$(aws ecr create-repository \
  --repository-name "sample-api" \
  --query 'repository.repositoryUri' \
  --output text)

echo "ECR Repository: $REPO_URI"

# Tạo Dockerfile đơn giản
cat > Dockerfile << 'EOF'
FROM python:3.12-slim
WORKDIR /app
RUN pip install flask
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
EOF

# Tạo Flask API đơn giản
cat > app.py << 'EOF'
from flask import Flask, jsonify
import socket
import os

app = Flask(__name__)

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'hostname': socket.gethostname()})

@app.route('/')
def index():
    return jsonify({
        'message': 'Hello from ECS Fargate!',
        'hostname': socket.gethostname(),
        'environment': os.getenv('ENVIRONMENT', 'unknown')
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF

# Login và push image lên ECR
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)

aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin $REPO_URI

docker build -t sample-api .
docker tag sample-api:latest $REPO_URI:latest
docker push $REPO_URI:latest

echo "Image pushed: $REPO_URI:latest"
```

### Bước 2: Tạo ECS Cluster và IAM Roles

```bash
# Tạo ECS Cluster
aws ecs create-cluster --cluster-name "lab-cluster" --capacity-providers FARGATE

# Tạo Task Execution Role (Vai Trò Thực Thi Task)
cat > ecs-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

EXEC_ROLE_ARN=$(aws iam create-role \
  --role-name "ecs-task-execution-role" \
  --assume-role-policy-document file://ecs-trust-policy.json \
  --query 'Role.Arn' --output text)

aws iam attach-role-policy \
  --role-name "ecs-task-execution-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

echo "Execution Role: $EXEC_ROLE_ARN"
```

### Bước 3: Tạo Task Definition (Định Nghĩa Task)

```bash
# Tạo CloudWatch Log Group
aws logs create-log-group --log-group-name "/ecs/sample-api"

cat > task-definition.json << EOF
{
  "family": "sample-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "$EXEC_ROLE_ARN",
  "containerDefinitions": [{
    "name": "sample-api",
    "image": "$REPO_URI:latest",
    "portMappings": [{
      "containerPort": 8080,
      "protocol": "tcp"
    }],
    "environment": [
      {"name": "ENVIRONMENT", "value": "lab"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/sample-api",
        "awslogs-region": "$AWS_REGION",
        "awslogs-stream-prefix": "ecs"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3
    }
  }]
}
EOF

TASK_DEF_ARN=$(aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "Task Definition: $TASK_DEF_ARN"
```

### Bước 4: Tạo ECS Service

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)
SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" --query 'Subnets[*].SubnetId' --output json | jq -r 'join(",")')

# Tạo Security Groups
ECS_SG=$(aws ec2 create-security-group --group-name "ecs-sg" --description "ECS tasks SG" --vpc-id $VPC_ID --query 'GroupId' --output text)
ALB_SG=$(aws ec2 create-security-group --group-name "alb-sg-ecs" --description "ALB SG" --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $ECS_SG --protocol tcp --port 8080 --source-group $ALB_SG

# Tạo ALB và Target Group
SUBNET_LIST=$(echo $SUBNET_IDS | tr ',' ' ')
ALB_ARN=$(aws elbv2 create-load-balancer --name "ecs-alb" --subnets $SUBNET_LIST --security-groups $ALB_SG --query 'LoadBalancers[0].LoadBalancerArn' --output text)

TG_ARN=$(aws elbv2 create-target-group \
  --name "ecs-tg" --protocol HTTP --port 8080 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-path "/health" \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Tạo ECS Service
aws ecs create-service \
  --cluster "lab-cluster" \
  --service-name "sample-api-service" \
  --task-definition "sample-api" \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$ECS_SG],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=sample-api,containerPort=8080" \
  --deployment-configuration "minimumHealthyPercent=100,maximumPercent=200,deploymentCircuitBreaker={enable=true,rollback=true}"
```

### Bước 5: Kiểm Tra và Deploy Update

```bash
# Chờ service stable (2-3 phút)
aws ecs wait services-stable --cluster "lab-cluster" --services "sample-api-service"

# Test API
ALB_DNS=$(aws elbv2 describe-load-balancers --names "ecs-alb" --query 'LoadBalancers[0].DNSName' --output text)
curl http://$ALB_DNS/
curl http://$ALB_DNS/health

# Xem logs trong CloudWatch
aws logs tail /ecs/sample-api --follow

# Simulate rolling update: thay đổi environment variable
aws ecs update-service \
  --cluster "lab-cluster" \
  --service "sample-api-service" \
  --force-new-deployment

# Theo dõi quá trình deploy
aws ecs describe-services --cluster "lab-cluster" --services "sample-api-service" \
  --query 'services[0].deployments'
```

### ✅ Checklist Kiểm Tra

- [ ] ECS Service có 2 tasks Running
- [ ] `curl http://<ALB_DNS>/health` trả về `{"status": "healthy"}`
- [ ] CloudWatch Logs hiển thị request logs
- [ ] Rolling update hoàn thành không có downtime

---

## Lab 4: CloudWatch Monitoring và Alerting

**Mục tiêu:** Thiết lập monitoring hoàn chỉnh với custom metrics, alarms, và auto-remediation.
**Thời gian:** 45-60 phút
**Chi phí:** Miễn phí (trong Free Tier limits)

### Bước 1: Publish Custom Metrics từ Lambda

```bash
# Lambda function để publish custom metrics
cat > metrics_publisher.py << 'EOF'
import boto3
import json
import datetime

cloudwatch = boto3.client('cloudwatch')

def handler(event, context):
    """Publish business metrics lên CloudWatch"""
    
    # Giả lập metrics từ business logic
    metrics = {
        'OrdersProcessed': 150,
        'OrdersFailedValidation': 5,
        'AverageOrderValue': 85.50,
        'PaymentSuccessRate': 98.5
    }
    
    # Batch publish metrics
    metric_data = []
    for metric_name, value in metrics.items():
        metric_data.append({
            'MetricName': metric_name,
            'Value': value,
            'Unit': 'Count' if 'Rate' not in metric_name else 'Percent',
            'Dimensions': [
                {'Name': 'Environment', 'Value': 'production'},
                {'Name': 'Service', 'Value': 'order-service'}
            ]
        })
    
    cloudwatch.put_metric_data(
        Namespace='BusinessMetrics/OrderService',
        MetricData=metric_data
    )
    
    print(f"Published {len(metric_data)} metrics to CloudWatch")
    return {'statusCode': 200}
EOF

zip metrics_publisher.zip metrics_publisher.py

# Tạo Lambda function (dùng role từ Lab 2)
aws lambda create-function \
  --function-name "metrics-publisher" \
  --runtime python3.12 \
  --handler metrics_publisher.handler \
  --role $ROLE_ARN \
  --zip-file fileb://metrics_publisher.zip \
  --timeout 30

# Tạo CloudWatch Event để chạy mỗi phút
aws events put-rule \
  --name "metrics-publisher-schedule" \
  --schedule-expression "rate(1 minute)" \
  --state ENABLED

LAMBDA_ARN=$(aws lambda get-function --function-name metrics-publisher --query 'Configuration.FunctionArn' --output text)

aws events put-targets \
  --rule "metrics-publisher-schedule" \
  --targets "Id=metrics-publisher,Arn=$LAMBDA_ARN"

aws lambda add-permission \
  --function-name "metrics-publisher" \
  --statement-id "allow-eventbridge" \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn $(aws events describe-rule --name "metrics-publisher-schedule" --query 'Arn' --output text)
```

### Bước 2: Tạo CloudWatch Alarms

```bash
# Alarm cho Lambda error rate (dùng Lambda từ Lab 2)
aws cloudwatch put-metric-alarm \
  --alarm-name "order-processor-error-rate" \
  --alarm-description "Alert khi Lambda error rate > 5%" \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --dimensions Name=FunctionName,Value=order-processor \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 5 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching

# Alarm cho business metric — PaymentSuccessRate
aws cloudwatch put-metric-alarm \
  --alarm-name "payment-success-rate-low" \
  --alarm-description "Alert khi payment success rate < 95%" \
  --metric-name PaymentSuccessRate \
  --namespace BusinessMetrics/OrderService \
  --dimensions "Name=Environment,Value=production" "Name=Service,Value=order-service" \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 95 \
  --comparison-operator LessThanThreshold

# Composite Alarm (Cảnh Báo Tổng Hợp) — trigger khi CẢ HAI alarm đang ALARM
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=$(aws configure get region)

aws cloudwatch put-composite-alarm \
  --alarm-name "critical-payment-issue" \
  --alarm-description "CRITICAL: Cả Lambda errors VÀ payment rate đều bất thường" \
  --alarm-rule "ALARM(\"order-processor-error-rate\") AND ALARM(\"payment-success-rate-low\")"
```

### Bước 3: CloudWatch Log Insights Queries

```bash
# Query: Top 10 lỗi trong 1 giờ qua
aws logs start-query \
  --log-group-name "/aws/lambda/order-processor" \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    filter @type = "REPORT"
    | stats 
        count(*) as invocations,
        avg(@duration) as avg_duration,
        max(@duration) as max_duration,
        sum(@billedDuration) as total_billed_ms
        by bin(5m)
    | sort by bin
  '

# Query: Tìm timeouts
aws logs start-query \
  --log-group-name "/aws/lambda/order-processor" \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    filter @message like /Task timed out/
    | fields @timestamp, @requestId, @message
    | sort @timestamp desc
    | limit 20
  '
```

### ✅ Checklist Kiểm Tra

- [ ] Custom metrics xuất hiện trong CloudWatch Metrics sau 2-3 phút
- [ ] Alarms được tạo ở trạng thái OK
- [ ] CloudWatch Log Insights query trả về kết quả
- [ ] Composite alarm hiển thị trong CloudWatch Console

---

## Lab 5: EKS Cluster với HPA và Cluster Autoscaler

**Mục tiêu:** Deploy EKS cluster, cấu hình HPA (Horizontal Pod Autoscaler — Tự Động Co Giãn Pod Theo Chiều Ngang) và Cluster Autoscaler (Tự Động Co Giãn Cluster), test auto-scaling end-to-end.
**Thời gian:** 90-120 phút
**Chi phí:** ~0.20 USD/giờ (EKS control plane) + EC2 instances

### Bước 1: Tạo EKS Cluster với eksctl

```bash
# Cài đặt eksctl nếu chưa có
curl --silent --location "https://github.com/eksctl/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Tạo EKS cluster (mất 15-20 phút)
eksctl create cluster \
  --name "lab-eks" \
  --region ap-southeast-1 \
  --nodegroup-name "managed-nodes" \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 5 \
  --managed \
  --asg-access  # Thêm permission cho Cluster Autoscaler

# Cập nhật kubeconfig (cấu hình kết nối kubectl)
aws eks update-kubeconfig --name lab-eks --region ap-southeast-1

# Kiểm tra cluster
kubectl get nodes
kubectl get pods --all-namespaces
```

### Bước 2: Deploy Sample Application với HPA

```bash
# Deploy nginx với resource requests/limits
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "100m"      # 0.1 CPU cores
            memory: "128Mi"  # 128 MB
          limits:
            cpu: "500m"      # 0.5 CPU cores
            memory: "256Mi"  # 256 MB
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Cài đặt Metrics Server (cần thiết cho HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Chờ Metrics Server ready
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=60s

# Kiểm tra metrics hoạt động
kubectl top nodes
kubectl top pods
```

### Bước 3: Tạo HPA

```bash
# Tạo HPA — scale khi CPU > 50%
kubectl apply -f - << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60  # Chờ 1 phút trước khi scale-down
    scaleUp:
      stabilizationWindowSeconds: 0   # Scale-up ngay khi cần
EOF

# Theo dõi HPA
kubectl describe hpa nginx-hpa
kubectl get hpa nginx-hpa --watch
```

### Bước 4: Cài Đặt Cluster Autoscaler

```bash
# Tạo IAM Policy cho Cluster Autoscaler
cat > cluster-autoscaler-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "autoscaling:DescribeAutoScalingGroups",
      "autoscaling:DescribeAutoScalingInstances",
      "autoscaling:DescribeLaunchConfigurations",
      "autoscaling:DescribeScalingActivities",
      "autoscaling:SetDesiredCapacity",
      "autoscaling:TerminateInstanceInAutoScalingGroup",
      "ec2:DescribeInstanceTypes",
      "ec2:DescribeLaunchTemplateVersions"
    ],
    "Resource": "*"
  }]
}
EOF

aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json

# Kết hợp với node IAM role
NODE_ROLE=$(aws eks describe-nodegroup \
  --cluster-name lab-eks \
  --nodegroup-name managed-nodes \
  --query 'nodegroup.nodeRole' --output text | awk -F'/' '{print $NF}')

POLICY_ARN=$(aws iam list-policies --query "Policies[?PolicyName=='ClusterAutoscalerPolicy'].Arn" --output text)

aws iam attach-role-policy --role-name $NODE_ROLE --policy-arn $POLICY_ARN

# Deploy Cluster Autoscaler
CLUSTER_NAME="lab-eks"
AWS_REGION="ap-southeast-1"

kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --cloud-provider=aws
        - --namespace=kube-system
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/$CLUSTER_NAME
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        env:
        - name: AWS_REGION
          value: $AWS_REGION
EOF
```

### Bước 5: Load Test và Quan Sát Auto-scaling

```bash
# Deploy load generator
kubectl run load-generator \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://nginx-service; done"

# Terminal 1: Theo dõi HPA
kubectl get hpa nginx-hpa --watch

# Terminal 2: Theo dõi pods
kubectl get pods -l app=nginx --watch

# Terminal 3: Theo dõi nodes
kubectl get nodes --watch

# Sau 5-10 phút, HPA sẽ scale pods
# Nếu nodes không đủ, Cluster Autoscaler sẽ thêm node

# Dừng load test
kubectl delete pod load-generator

# Quan sát scale-down (mất 5-10 phút vì cooldown period)
kubectl get hpa nginx-hpa --watch
```

### ✅ Checklist Kiểm Tra

- [ ] `kubectl top pods` hiển thị CPU/memory usage
- [ ] HPA tăng replicas khi load test đang chạy (từ 2 → 5+ pods)
- [ ] Cluster Autoscaler thêm node mới khi pods Pending (nếu 2 nodes đủ)
- [ ] Sau khi dừng load test, pods giảm dần về minimum (2 replicas)

### 🧹 Dọn Dẹp EKS (Quan Trọng — Tránh Chi Phí Lớn)

```bash
# Xóa tất cả resources trong cluster
kubectl delete all --all

# Xóa EKS cluster (mất 10-15 phút)
eksctl delete cluster --name lab-eks --region ap-southeast-1

# Xóa ECR repository (từ Lab 3)
aws ecr delete-repository --repository-name sample-api --force
```

---

## 📊 Tóm Tắt Kỹ Năng Thực Hành

| Lab                    | Kỹ Năng Chính                                | Thời Gian | Chi Phí   |
| ---------------------- | -------------------------------------------- | --------- | --------- |
| Lab 1: EC2 + ASG       | Security Groups, Launch Template, ALB, ASG   | 60-90 phút | < 1 USD  |
| Lab 2: Lambda + SQS    | Event Source Mapping, DLQ, partial failures  | 45-60 phút | Miễn phí |
| Lab 3: ECS Fargate     | Task Definition, Service, rolling deploy     | 60-90 phút | ~0.5 USD/hr |
| Lab 4: CloudWatch      | Custom metrics, Alarms, Log Insights         | 45-60 phút | Miễn phí |
| Lab 5: EKS + HPA       | Kubernetes scaling, Cluster Autoscaler       | 90-120 phút | ~0.20 USD/hr |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành — 5 bài tập thực hành chi tiết
