# 2 — Deployment Strategies (Chiến Lược Triển Khai)

> Deployment Strategy — Chiến Lược Triển Khai — là cách bạn đưa phiên bản mới của ứng dụng lên production trong khi giảm thiểu downtime (thời gian ngừng hoạt động) và rủi ro. Mỗi chiến lược có trade-offs khác nhau giữa tốc độ, rủi ro và tài nguyên cần thiết.

## 🗺️ So Sánh Tổng Quan

| Chiến Lược | Downtime | Rủi Ro | Tài Nguyên | Rollback | Độ Phức Tạp |
|---|---|---|---|---|---|
| Recreate (Tạo Lại) | Có | Cao | 1x | Chậm | Thấp |
| Rolling (Cuộn) | Không | Trung Bình | 1x | Chậm | Thấp |
| Blue/Green | Không | Thấp | 2x | Tức Thì | Trung Bình |
| Canary (Thử Nghiệm) | Không | Rất Thấp | ~1.1x | Tức Thì | Cao |
| A/B Testing | Không | Rất Thấp | ~1.2x | Tức Thì | Rất Cao |

---

## 1️⃣ Recreate Deployment (Triển Khai Tạo Lại)

### Cơ chế hoạt động

```
Step 1: Dừng TẤT CẢ instances cũ
        [v1] [v1] [v1] → STOP → [  ] [  ] [  ]

Step 2: Khởi động instances mới
        [  ] [  ] [  ] → START → [v2] [v2] [v2]
```

**Có downtime trong khoảng giữa Step 1 và Step 2.**

### Khi nào dùng

- Database migrations (di chuyển dữ liệu) không tương thích ngược
- Thay đổi lớn về kiến trúc không thể chạy song song
- Môi trường development/staging (không cần zero-downtime)

### GitHub Actions Implementation

```yaml
name: Recreate Deployment

jobs:
  deploy:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Stop old service
        run: |
          aws ecs update-service \
            --cluster my-cluster \
            --service my-service \
            --desired-count 0
          aws ecs wait services-stable \
            --cluster my-cluster \
            --services my-service

      - name: Run database migrations
        run: |
          aws ecs run-task \
            --cluster my-cluster \
            --task-definition migration-task \
            --launch-type FARGATE \
            --network-configuration "..."
          # Đợi migration hoàn thành...

      - name: Deploy new version
        run: |
          aws ecs update-service \
            --cluster my-cluster \
            --service my-service \
            --task-definition my-task:${{ env.NEW_REVISION }} \
            --desired-count 3
```

---

## 2️⃣ Rolling Deployment (Triển Khai Cuộn)

### Cơ chế hoạt động

```
Trạng Thái Ban Đầu: [v1][v1][v1][v1]

Bước 1: Thay 1 instance   [v2][v1][v1][v1]
Bước 2: Thay tiếp         [v2][v2][v1][v1]
Bước 3: Tiếp tục          [v2][v2][v2][v1]
Bước 4: Hoàn thành        [v2][v2][v2][v2]

Trong mỗi bước:
- v1 và v2 chạy song song một thời gian ngắn
- Traffic được route (định tuyến) đến cả hai
```

### Khi nào dùng

- API changes tương thích ngược (backward-compatible)
- Phần lớn các deployment thông thường
- Kubernetes default deployment strategy

### Tham số quan trọng

```yaml
# Kubernetes Rolling Update
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # Tối đa bao nhiêu pods có thể unavailable cùng lúc
      maxSurge: 1          # Tối đa bao nhiêu pods có thể tạo thêm trên desired count
```

```
maxUnavailable: 1, maxSurge: 1
Desired: 4 pods

Timeline:
  Start:          [v1][v1][v1][v1]     = 4 running, 0 surge
  Create surge:   [v1][v1][v1][v1][v2] = 5 running (1 surge)
  Terminate old:  [v1][v1][v1][v2]     = 4 running (1 terminated)
  Create surge:   [v1][v1][v1][v2][v2] = 5 running
  Terminate old:  [v1][v1][v2][v2]     = 4 running
  ...và tiếp tục
```

### GitHub Actions Implementation

```yaml
name: Rolling Deployment

jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name prod-cluster \
            --region us-east-1

      - name: Update image (triggers rolling update)
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/myorg/myapp:${{ github.sha }}

      - name: Wait for rollout (đợi quá trình cuộn hoàn thành)
        run: |
          kubectl rollout status deployment/myapp \
            --timeout=5m

      - name: Verify deployment
        run: |
          kubectl get pods -l app=myapp
          # Kiểm tra không có pod ở trạng thái Error hoặc CrashLoopBackOff
          kubectl get pods -l app=myapp \
            --field-selector=status.phase!=Running \
            -o name | wc -l | grep -q "^0$"

      - name: Rollback on failure
        if: failure()
        run: |
          kubectl rollout undo deployment/myapp
          kubectl rollout status deployment/myapp --timeout=3m
```

---

## 3️⃣ Blue/Green Deployment

### Cơ chế hoạt động

```
Initial State:
  Blue (LIVE):  [v1][v1][v1]  ← Load Balancer → 100% traffic
  Green (IDLE): [  ][  ][  ]

Step 1: Deploy v2 lên Green
  Blue:  [v1][v1][v1]  ← traffic
  Green: [v2][v2][v2]  (chưa nhận traffic)

Step 2: Test Green kỹ lưỡng
  Blue:  [v1][v1][v1]  ← traffic
  Green: [v2][v2][v2]  ← smoke tests, health checks

Step 3: Switch traffic sang Green
  Blue:  [v1][v1][v1]  (standby — chờ, có thể rollback)
  Green: [v2][v2][v2]  ← traffic 100%

Step 4: Sau khi ổn định, terminate Blue (hoặc giữ để rollback)
  Blue:  (terminated hoặc kept for rollback)
  Green: [v2][v2][v2]  ← traffic 100%
```

### Rollback tức thì

```
Nếu có lỗi sau khi switch:
  Green: [v2][v2][v2]  (problematic)
  Blue:  [v1][v1][v1]  ← chuyển traffic trở lại — trong vài giây!

So với Rolling:
  Rolling rollback = chờ từng pod rollback = vài phút
  Blue/Green rollback = 1 DNS/LB switch = vài giây
```

### GitHub Actions Implementation

```yaml
name: Blue/Green Deployment

env:
  AWS_REGION: us-east-1
  ECS_CLUSTER: prod-cluster
  ECS_SERVICE: myapp

jobs:
  deploy-bluegreen:
    environment: production
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Identify current color (blue hay green?)
        id: current
        run: |
          CURRENT=$(aws ecs describe-services \
            --cluster $ECS_CLUSTER \
            --services $ECS_SERVICE \
            --query 'services[0].tags[?key==`color`].value' \
            --output text)

          if [ "$CURRENT" = "blue" ]; then
            echo "current=blue" >> $GITHUB_OUTPUT
            echo "next=green" >> $GITHUB_OUTPUT
          else
            echo "current=green" >> $GITHUB_OUTPUT
            echo "next=blue" >> $GITHUB_OUTPUT
          fi

      - name: Deploy to inactive environment (${{ steps.current.outputs.next }})
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service myapp-${{ steps.current.outputs.next }} \
            --task-definition myapp:${{ github.run_number }} \
            --desired-count 3
          aws ecs wait services-stable \
            --cluster $ECS_CLUSTER \
            --services myapp-${{ steps.current.outputs.next }}

      - name: Run smoke tests against inactive environment
        run: |
          INACTIVE_URL=$(aws elbv2 describe-target-groups \
            --names myapp-${{ steps.current.outputs.next }} \
            --query 'TargetGroups[0].LoadBalancerArns[0]' \
            --output text)
          curl -f "$INACTIVE_URL/health"
          curl -f "$INACTIVE_URL/api/version"

      - name: Switch traffic to new environment
        run: |
          # Điểm không thể quay lại — cần xác nhận kỹ trước bước này
          aws elbv2 modify-listener \
            --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
            --default-actions Type=forward,TargetGroupArn=$(
              aws elbv2 describe-target-groups \
                --names myapp-${{ steps.current.outputs.next }} \
                --query 'TargetGroups[0].TargetGroupArn' \
                --output text
            )

      - name: Verify new environment is serving traffic
        run: |
          sleep 10
          curl -f https://myapp.com/health
          # Kiểm tra version header để đảm bảo đúng version
          curl -s https://myapp.com/api/version | grep "${{ github.sha }}"

      - name: Tag new active color
        run: |
          aws ecs tag-resource \
            --resource-arn $(aws ecs describe-services \
              --cluster $ECS_CLUSTER \
              --services $ECS_SERVICE \
              --query 'services[0].serviceArn' \
              --output text) \
            --tags key=color,value=${{ steps.current.outputs.next }}

      # Emergency rollback — chạy thủ công nếu cần
      - name: Rollback (chỉ chạy khi verify thất bại)
        if: failure()
        run: |
          aws elbv2 modify-listener \
            --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
            --default-actions Type=forward,TargetGroupArn=$(
              aws elbv2 describe-target-groups \
                --names myapp-${{ steps.current.outputs.current }} \
                --query 'TargetGroups[0].TargetGroupArn' \
                --output text
            )
```

---

## 4️⃣ Canary Deployment (Triển Khai Thử Nghiệm)

### Cơ chế hoạt động

```
Giai đoạn 1: 5% canary
  [v1][v1][v1][v1][v1][v1][v1][v1][v1][v2]
   ↑                                    ↑
  90% traffic                         10% traffic
  Monitor: error rate, latency, business metrics

Giai đoạn 2: Nếu ổn, tăng 25%
  [v1][v1][v1][v1][v1][v1][v1][v2][v2][v2]
  75% traffic                 25%

Giai đoạn 3: 50%
  [v1][v1][v1][v1][v1][v2][v2][v2][v2][v2]

Giai đoạn 4: 100%
  [v2][v2][v2][v2][v2][v2][v2][v2][v2][v2]
```

Tên "Canary" — Chim Hoàng Yến — đến từ ngành khai thác mỏ: thợ mỏ mang theo chim hoàng yến để phát hiện khí độc sớm. Tương tự, canary deployment đưa một phần nhỏ traffic vào phiên bản mới để "phát hiện sự cố sớm" trước khi ảnh hưởng đến toàn bộ user.

### Khi nào dùng Canary

- Thay đổi lớn về tính năng, không chắc chắn về hiệu năng
- Release quan trọng cần theo dõi kỹ
- Khi cần data thực từ production để quyết định có nên roll out không
- Có monitoring tốt để phát hiện vấn đề sớm

### GitHub Actions với Kubernetes + Argo Rollouts

```yaml
name: Canary Deployment

jobs:
  canary-deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup kubectl và Argo Rollouts
        run: |
          curl -LO "https://dl.k8s.io/release/v1.28.0/bin/linux/amd64/kubectl"
          chmod +x kubectl && mv kubectl /usr/local/bin/

          curl -LO "https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64"
          chmod +x kubectl-argo-rollouts-linux-amd64
          mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

      - name: Update image for canary rollout
        run: |
          kubectl argo rollouts set image myapp \
            myapp=ghcr.io/myorg/myapp:${{ github.sha }}

      - name: Promote to 20% canary
        run: kubectl argo rollouts promote myapp --full=false

      - name: Wait and monitor canary (theo dõi 5 phút)
        run: |
          echo "Monitoring canary for 5 minutes..."
          sleep 300

          # Kiểm tra error rate từ Prometheus/CloudWatch
          ERROR_RATE=$(curl -s "${{ vars.PROMETHEUS_URL }}/api/v1/query" \
            --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[5m])' \
            | jq '.data.result[0].value[1]' | tr -d '"')

          echo "Current error rate: $ERROR_RATE"
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high! Aborting canary..."
            exit 1
          fi

      - name: Full rollout (promote 100%)
        run: kubectl argo rollouts promote myapp --full

      - name: Rollback canary on failure
        if: failure()
        run: kubectl argo rollouts abort myapp
```

### Canary với AWS CodeDeploy

```yaml
  - name: Create CodeDeploy deployment (Canary 10% → 90%)
    run: |
      aws deploy create-deployment \
        --application-name MyApp \
        --deployment-group-name production \
        --deployment-config-name CodeDeployDefault.ECSCanary10Percent5Minutes \
        --description "Canary deployment ${{ github.sha }}"
```

**CodeDeploy Canary Configs sẵn có:**
```
CodeDeployDefault.ECSCanary10Percent5Minutes   — 10% → 5 phút → 90%
CodeDeployDefault.ECSCanary10Percent15Minutes  — 10% → 15 phút → 90%
CodeDeployDefault.ECSLinear10PercentEvery1Minutes  — tăng 10% mỗi phút
CodeDeployDefault.ECSLinear10PercentEvery3Minutes  — tăng 10% mỗi 3 phút
CodeDeployDefault.ECSAllAtOnce                 — rolling thông thường
```

---

## 5️⃣ Feature Flags (Cờ Tính Năng) — Deployment vs Release

Feature flags cho phép bạn deploy code nhưng chưa "release" (mở) tính năng cho users:

```
Deploy code:  Luôn tự động, không rủi ro (code tồn tại nhưng bị tắt)
Release feature: Được kiểm soát qua feature flag, không cần deploy mới
```

```yaml
# Workflow deploy code với feature flags
- name: Deploy (feature flags tắt)
  run: ./deploy.sh

# Sau khi verify, bật flag cho 5% users
- name: Enable feature flag cho 5% users
  run: |
    curl -X PATCH https://api.launchdarkly.com/api/v2/flags/default/new-checkout \
      -H "Authorization: ${{ secrets.LAUNCHDARKLY_API_KEY }}" \
      -H "Content-Type: application/json" \
      -d '{"patch": [{"op": "replace", "path": "/environments/production/on", "value": true}]}'
```

**Tools phổ biến cho Feature Flags:**
- LaunchDarkly
- Unleash (self-hosted, open source)
- Split.io
- AWS AppConfig

---

## 🔄 Automated Rollback (Hoàn Nguyên Tự Động)

### Health Check + Automatic Rollback

```yaml
jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - name: Deploy new version
        id: deploy
        run: |
          kubectl set image deployment/myapp myapp=ghcr.io/myorg/myapp:${{ github.sha }}
          kubectl rollout status deployment/myapp --timeout=5m

      - name: Progressive health check (kiểm tra sức khỏe từng bước)
        id: health
        run: |
          # Kiểm tra 5 lần trong 2 phút
          for i in {1..5}; do
            sleep 30
            HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.com/health)
            if [ "$HTTP_CODE" != "200" ]; then
              echo "Health check failed with HTTP $HTTP_CODE"
              exit 1
            fi
            echo "Health check $i/5 passed"
          done

      - name: Rollback if health check fails
        if: failure() && steps.deploy.outcome == 'success'
        run: |
          echo "🔄 Rolling back deployment..."
          kubectl rollout undo deployment/myapp
          kubectl rollout status deployment/myapp --timeout=3m
          echo "✅ Rollback completed"

      - name: Alert team on rollback
        if: failure() && steps.deploy.outcome == 'success'
        uses: slackapi/slack-github-action@v1
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "⚠️ ROLLBACK: Production deployment of ${{ github.sha }} was rolled back due to failed health checks. @oncall please investigate."
            }
```

### Metrics-Based Rollback (Dựa Trên Chỉ Số)

```yaml
      - name: Monitor error rate sau deploy
        run: |
          # Đợi 2 phút để metrics ổn định
          sleep 120

          # Query Datadog/Prometheus/CloudWatch
          ERROR_RATE=$(aws cloudwatch get-metric-statistics \
            --namespace MyApp \
            --metric-name ErrorRate \
            --start-time $(date -u -d '2 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 120 \
            --statistics Average \
            --query 'Datapoints[0].Average' \
            --output text)

          P99_LATENCY=$(aws cloudwatch get-metric-statistics \
            --namespace MyApp \
            --metric-name p99Latency \
            --start-time $(date -u -d '2 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 120 \
            --statistics p99 \
            --query 'Datapoints[0].p99' \
            --output text)

          # Thresholds (ngưỡng) — tự điều chỉnh theo app
          if (( $(echo "$ERROR_RATE > 0.005" | bc -l) )); then
            echo "Error rate ($ERROR_RATE) exceeds threshold (0.5%)! Initiating rollback..."
            exit 1
          fi

          if (( $(echo "$P99_LATENCY > 2000" | bc -l) )); then
            echo "p99 latency ($P99_LATENCY ms) exceeds threshold (2000ms)! Initiating rollback..."
            exit 1
          fi

          echo "✅ Metrics within acceptable range"
```

---

## 📊 Chọn Deployment Strategy Cho Dự Án

### Decision Tree (Cây Quyết Định)

```
Có thể có downtime không?
  ├── Có → Recreate (đơn giản nhất, chấp nhận downtime)
  └── Không → Zero-downtime strategy

      Cần rollback nhanh không?
        ├── Không cần nhanh → Rolling (đơn giản, tiết kiệm)
        └── Cần < 1 phút → Blue/Green hoặc Canary

            Muốn test với % users thực không?
              ├── Có → Canary (phức tạp nhưng an toàn nhất)
              └── Không → Blue/Green (đơn giản, rollback tức thì)
```

### Recommendations (Khuyến Nghị)

| Trường Hợp | Strategy Khuyến Nghị |
|---|---|
| Startup, move fast | Rolling (đủ tốt, đơn giản) |
| E-commerce high traffic | Blue/Green (rollback nhanh khi sự cố) |
| Big feature release | Canary (test với % users thực) |
| Database migration | Recreate (sau giờ thấp điểm) |
| Microservices | Rolling per service + Canary cho services quan trọng |
| Regulated industry | Blue/Green + manual approval |

---

## 🔗 Điều Hướng

| ← Trước | Hiện Tại | Tiếp Theo → |
|---|---|---|
| [1-environments.md](1-environments.md) | 2-deployment-strategies.md | [3-docker-deployments.md](3-docker-deployments.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-11
