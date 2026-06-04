# Tích Hợp AWS Trong CircleCI

> **AWS — Amazon Web Services**: pipeline thường dùng **aws-cli orb**, **OIDC** (xem module Security), và các dịch vụ **S3**, **ECR**, **ECS**, **Lambda**, **CodeDeploy**.

## 📚 Mục Lục

1. [Thiết Lập aws-cli Orb](#thiết-lập-aws-cli-orb)
2. [S3 — Static Hosting & Artifacts](#s3--static-hosting--artifacts)
3. [ECR — Container Registry](#ecr--container-registry)
4. [ECS — Elastic Container Service](#ecs--elastic-container-service)
5. [Lambda — Serverless](#lambda--serverless)
6. [CodeDeploy](#codedeploy)
7. [Workflow & Context](#workflow--context)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Thiết Lập aws-cli Orb

```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@4.0.0

jobs:
  aws-job:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - aws-cli/setup:
          role-arn: arn:aws:iam::123456789012:role/circleci-oidc-role
          region: ap-southeast-1
          session-duration: "3600"
      - run:
          command: aws sts get-caller-identity
```

**OIDC — OpenID Connect** (khuyến nghị): không cần `AWS_ACCESS_KEY_ID` trong Context. Chỉ cần `role-arn` và trust policy IAM cho CircleCI org/project.

Fallback access key: đặt trong Context `production-aws`, job khai báo `context: production-aws`.

---

## S3 — Static Hosting & Artifacts

### Deploy frontend / static site

```yaml
      - run:
          name: Sync lên S3
          command: |
            aws s3 sync ./dist s3://my-bucket-prod \
              --delete \
              --cache-control "public,max-age=31536000,immutable"
      - run:
          name: Invalidate CloudFront — CDN
          command: |
            aws cloudfront create-invalidation \
              --distribution-id $CLOUDFRONT_DIST_ID \
              --paths "/*"
```

### Lưu artifact build

```yaml
      - run:
          command: |
            aws s3 cp ./build.zip s3://ci-artifacts/${CIRCLE_PROJECT_REPONAME}/${CIRCLE_BUILD_NUM}/build.zip
```

Hoặc dùng built-in `store_artifacts` cho artifact nhỏ, tạm thời.

---

## ECR — Container Registry

```yaml
      - setup_remote_docker
      - aws-cli/setup:
          role-arn: $AWS_ECR_ROLE_ARN
      - run:
          command: |
            ACCOUNT=123456789012
            REGION=ap-southeast-1
            REPO=$ACCOUNT.dkr.ecr.$REGION.amazonaws.com/myapp
            aws ecr get-login-password --region $REGION | \
              docker login --username AWS --password-stdin $ACCOUNT.dkr.ecr.$REGION.amazonaws.com
            docker build -t $REPO:${CIRCLE_SHA1:0:7} .
            docker push $REPO:${CIRCLE_SHA1:0:7}
```

Tạo repository trước: `aws ecr create-repository --repository-name myapp`.

---

## ECS — Elastic Container Service

Luồng: đăng ký **task definition** mới với image tag mới → `update-service` → chờ deployment ổn định.

```yaml
      - run:
          name: Deploy ECS Fargate
          command: |
            IMAGE=123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/myapp:${CIRCLE_SHA1:0:7}

            TASK_JSON=$(aws ecs describe-task-definition \
              --task-definition myapp --query taskDefinition)

            NEW_TASK=$(echo "$TASK_JSON" | jq \
              --arg IMG "$IMAGE" \
              '.containerDefinitions[0].image = $IMG | del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)')

            aws ecs register-task-definition --cli-input-json "$NEW_TASK"

            aws ecs update-service \
              --cluster production \
              --service myapp-svc \
              --task-definition myapp \
              --force-new-deployment

            aws ecs wait services-stable \
              --cluster production \
              --services myapp-svc
```

---

## Lambda — Serverless

### Deploy bằng ZIP

```yaml
      - run:
          command: |
            cd lambda && zip -r ../function.zip .
            aws lambda update-function-code \
              --function-name my-api \
              --zip-file fileb://../function.zip
```

### Container image Lambda

Push image lên ECR, rồi:

```yaml
            aws lambda update-function-code \
              --function-name my-api \
              --image-uri $ECR_URI:${CIRCLE_SHA1:0:7}
```

Công cụ **SAM — Serverless Application Model** hoặc **Serverless Framework** có thể gói trong một job `sam deploy`.

---

## CodeDeploy

Phù hợp EC2 hoặc on-prem với **CodeDeploy agent**:

```yaml
      - run:
          command: |
            aws deploy create-deployment \
              --application-name MyApp \
              --deployment-group-name Production \
              --s3-location bucket=my-codedeploy-bucket,key=appspec.zip,bundleType=zip \
              --file-exists-behavior OVERWRITE
```

CircleCI build artifact → upload S3 → trigger deployment.

---

## Workflow & Context

```yaml
workflows:
  aws-release:
    jobs:
      - test
      - build-and-push-ecr:
          requires: [test]
          context: aws-staging
          filters:
            branches:
              only: develop
      - hold-prod:
          type: approval
          requires: [build-and-push-ecr]
          filters:
            branches:
              only: main
      - deploy-ecs-prod:
          requires: [hold-prod]
          context: aws-production
```

Tách Context theo môi trường: `aws-staging` vs `aws-production` (role ARN, cluster name khác nhau).

---

## Câu Hỏi Phỏng Vấn

**Khi nào dùng ECS thay vì EKS?**  
ECS đơn giản hơn nếu chỉ cần chạy container trên AWS native; EKS khi cần ecosystem Kubernetes, multi-cloud skillset.

**IAM role tối thiểu cho pipeline deploy?**  
Principle of least privilege: chỉ quyền ECR push, ECS update trên resource ARN cụ thể, không `*`.

**CloudFormation/Terraform vs script trong job?**  
Script phù hợp deploy app; hạ tầng nên IaC — xem `5-terraform-pipeline.md`.
