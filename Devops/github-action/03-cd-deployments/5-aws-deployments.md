# 5 — AWS Deployments (Triển Khai trên Amazon Web Services)

> AWS là cloud provider (nhà cung cấp đám mây) lớn nhất thế giới. GitHub Actions có bộ actions chính thức của AWS (`aws-actions/*`) hỗ trợ deploy lên ECS, EKS, Lambda, S3, và nhiều dịch vụ khác.

## 🔐 Xác Thực với AWS

### OIDC — Cách Khuyến Nghị (Không Cần Long-lived Keys)

```yaml
# Cấu hình OIDC Provider trên AWS IAM một lần:
# IAM → Identity Providers → Add provider
# Provider URL: https://token.actions.githubusercontent.com
# Audience: sts.amazonaws.com

jobs:
  deploy:
    permissions:
      id-token: write    # Bắt buộc để lấy OIDC token
      contents: read

    steps:
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: us-east-1
          # Không cần AWS_ACCESS_KEY_ID hay AWS_SECRET_ACCESS_KEY!
```

### IAM Role Trust Policy (Chính Sách Tin Cậy)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
        }
      }
    }
  ]
}
```

**Giới hạn trust condition theo environment để tăng bảo mật:**
```json
"StringEquals": {
  "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:environment:production"
}
```

---

## 🐳 ECR — Elastic Container Registry (Kho Container AWS)

```yaml
jobs:
  build-push-ecr:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    outputs:
      image-uri: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

    env:
      ECR_REPOSITORY: myapp

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push to ECR
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 🚀 ECS — Elastic Container Service

ECS — Elastic Container Service — Dịch Vụ Container Linh Hoạt — là managed container orchestration service của AWS. Có hai launch types:
- **Fargate** — serverless, AWS quản lý server (không cần quản lý EC2)
- **EC2** — bạn quản lý EC2 instances bên dưới

### Deploy lên ECS Fargate

```yaml
jobs:
  deploy-ecs:
    needs: build-push-ecr
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1

      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition myapp-production \
            --query taskDefinition \
            > task-definition.json

      - name: Update image trong task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: myapp
          image: ${{ needs.build-push-ecr.outputs.image-uri }}

      - name: Deploy lên ECS service
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: myapp-production
          cluster: production-cluster
          wait-for-service-stability: true     # Đợi deployment ổn định
          wait-for-minutes: 10
```

### ECS Blue/Green với CodeDeploy

```yaml
      - name: Deploy ECS Blue/Green via CodeDeploy
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: myapp-production
          cluster: production-cluster
          wait-for-service-stability: true
          codedeploy-appspec: appspec.yaml
          codedeploy-application: myapp-production
          codedeploy-deployment-group: myapp-production-dg
```

```yaml
# appspec.yaml — CodeDeploy AppSpec cho ECS
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: myapp
          ContainerPort: 3000
        PlatformVersion: LATEST
Hooks:
  - BeforeAllowTraffic: BeforeAllowTrafficHook
  - AfterAllowTraffic: AfterAllowTrafficHook
```

---

## ☸️ EKS — Elastic Kubernetes Service

EKS — Elastic Kubernetes Service — Dịch Vụ Kubernetes Linh Hoạt — là managed Kubernetes service của AWS.

```yaml
jobs:
  deploy-eks:
    needs: build-push-ecr
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v4
      - uses: azure/setup-helm@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_EKS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name production-eks-cluster \
            --region us-east-1

      - name: Deploy với Helm
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --set image.repository=${{ steps.login-ecr.outputs.registry }}/myapp \
            --set image.tag=${{ github.sha }} \
            --values ./charts/myapp/values-production.yaml \
            --wait --timeout 10m --atomic

      - name: Verify deployment
        run: |
          kubectl get pods -n production -l app=myapp
          kubectl rollout status deployment/myapp -n production
```

### EKS với aws-auth ConfigMap (Cho phép GitHub Actions truy cập cluster)

```yaml
# aws-auth ConfigMap cần được cập nhật một lần bởi cluster admin:
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
      username: github-actions
      groups:
        - system:masters    # Hoặc tạo custom RBAC role ít quyền hơn
```

---

## ⚡ Lambda — Serverless Functions

Lambda — Dịch Vụ Hàm Không Máy Chủ — cho phép chạy code mà không cần quản lý servers.

### Deploy Lambda Function trực tiếp

```yaml
jobs:
  deploy-lambda:
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt -t ./package/
          cp -r src/ ./package/

      - name: Package Lambda
        run: |
          cd package
          zip -r ../lambda.zip .

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_LAMBDA_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy Lambda function
        run: |
          aws lambda update-function-code \
            --function-name my-function-production \
            --zip-file fileb://lambda.zip

      - name: Wait for update to complete
        run: |
          aws lambda wait function-updated \
            --function-name my-function-production

      - name: Publish new version và update alias
        run: |
          VERSION=$(aws lambda publish-version \
            --function-name my-function-production \
            --query Version --output text)

          aws lambda update-alias \
            --function-name my-function-production \
            --name production \
            --function-version $VERSION

      - name: Test Lambda invocation
        run: |
          RESULT=$(aws lambda invoke \
            --function-name my-function-production:production \
            --payload '{"test": true}' \
            --cli-binary-format raw-in-base64-out \
            response.json)
          cat response.json
          # Kiểm tra không có lỗi
          grep -v "FunctionError" response.json || exit 1
```

### Deploy Lambda Container Image

```yaml
      - name: Deploy Lambda từ ECR image
        run: |
          # Lambda hỗ trợ deploy từ container image ECR
          aws lambda update-function-code \
            --function-name my-function-production \
            --image-uri ${{ needs.build-push-ecr.outputs.image-uri }}

          aws lambda wait function-updated \
            --function-name my-function-production
```

### Canary Deployment cho Lambda

```yaml
      - name: Lambda Canary deploy (10% traffic trước)
        run: |
          VERSION=$(aws lambda publish-version \
            --function-name my-function-production \
            --query Version --output text)

          # Routing config: 10% sang version mới, 90% sang production alias
          aws lambda update-alias \
            --function-name my-function-production \
            --name production \
            --routing-config AdditionalVersionWeights="{\"$VERSION\": 0.1}"

      - name: Monitor error rate (10 phút)
        run: |
          sleep 600
          ERROR_RATE=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/Lambda \
            --metric-name Errors \
            --dimensions Name=FunctionName,Value=my-function-production \
            --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 600 \
            --statistics Sum \
            --query 'Datapoints[0].Sum' --output text)

          # Nếu error rate quá cao, rollback
          if (( $(echo "$ERROR_RATE > 10" | bc -l) )); then
            aws lambda update-alias \
              --function-name my-function-production \
              --name production \
              --routing-config AdditionalVersionWeights='{}'
            echo "❌ Canary rollback due to high error rate"
            exit 1
          fi

      - name: Promote Lambda canary to 100%
        if: success()
        run: |
          aws lambda update-alias \
            --function-name my-function-production \
            --name production \
            --function-version ${{ env.NEW_VERSION }} \
            --routing-config AdditionalVersionWeights='{}'
```

---

## 🌐 S3 + CloudFront — Static Site Deployment

### Deploy Frontend lên S3

```yaml
jobs:
  deploy-frontend:
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install và Build
        run: |
          npm ci
          npm run build
        env:
          VITE_API_URL: ${{ vars.API_URL }}
          VITE_ENV: production

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_S3_ROLE_ARN }}
          aws-region: us-east-1

      - name: Sync build output lên S3
        run: |
          # Upload static assets với cache dài (1 năm) — vì có content hash trong filename
          aws s3 sync dist/ s3://${{ secrets.S3_BUCKET }}/ \
            --exclude "index.html" \
            --cache-control "max-age=31536000,public,immutable"

          # Upload HTML với cache ngắn — cần reload ngay khi deploy mới
          aws s3 cp dist/index.html s3://${{ secrets.S3_BUCKET }}/index.html \
            --cache-control "max-age=0,no-cache,no-store,must-revalidate"

      - name: Invalidate CloudFront cache (xóa cache CDN)
        run: |
          INVALIDATION_ID=$(aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*" \
            --query 'Invalidation.Id' --output text)

          echo "Created invalidation: $INVALIDATION_ID"

          # Đợi invalidation hoàn thành
          aws cloudfront wait invalidation-completed \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --id $INVALIDATION_ID

          echo "✅ CloudFront cache invalidated"

      - name: Verify deployment
        run: |
          sleep 10
          curl -f https://myapp.com
          curl -f -I https://myapp.com | grep "x-cache"    # Kiểm tra CDN hit
```

---

## 📊 AWS Deployment Summary

```yaml
name: AWS Full Deployment Pipeline

on:
  push:
    branches: [main]

jobs:
  # ── 1. Build ───────────────────────────────────────────
  build:
    name: Build & Push to ECR
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    outputs:
      image-uri: ${{ steps.ecr.outputs.registry }}/myapp:${{ github.sha }}

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_BUILD_ROLE_ARN }}
          aws-region: us-east-1
      - name: Login ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - name: Build & push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.ecr.outputs.registry }}/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── 2. Deploy Staging (ECS) ────────────────────────────
  deploy-staging:
    needs: build
    environment: staging
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - name: AWS credentials (staging)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_STAGING_ROLE_ARN }}
          aws-region: us-east-1
      - run: |
          aws ecs describe-task-definition --task-definition myapp-staging \
            --query taskDefinition > task-def.json
      - id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-def.json
          container-name: myapp
          image: ${{ needs.build.outputs.image-uri }}
      - uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: myapp-staging
          cluster: staging
          wait-for-service-stability: true

  # ── 3. Deploy Production (ECS + approval) ─────────────
  deploy-production:
    needs: deploy-staging
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - name: AWS credentials (production)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1
      - run: |
          aws ecs describe-task-definition --task-definition myapp-production \
            --query taskDefinition > task-def.json
      - id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-def.json
          container-name: myapp
          image: ${{ needs.build.outputs.image-uri }}
      - uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: myapp-production
          cluster: production
          wait-for-service-stability: true
      - name: Health check
        run: curl -f https://myapp.com/health
```

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---|---|---|
| [4-kubernetes-deployments.md](4-kubernetes-deployments.md) | 5-aws-deployments.md | [6-gcp-deployments.md](6-gcp-deployments.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-11
