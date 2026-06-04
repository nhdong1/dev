# IP Ranges — Dải IP: Kiểm Soát Truy Cập Mạng

> IP Ranges — Dải IP là tính năng CircleCI cho phép pipeline của bạn sử dụng một **tập IP cố định, đã biết trước** khi gọi ra các dịch vụ bên ngoài. Điều này giúp các tổ chức có yêu cầu bảo mật mạng (firewall whitelist — danh sách trắng tường lửa) cho phép chỉ CircleCI mới có thể kết nối vào hệ thống của họ.

## 📚 Mục Lục

1. [IP Ranges là gì?](#ip-ranges-là-gì)
2. [Tại Sao Cần IP Ranges?](#tại-sao-cần-ip-ranges)
3. [Cách Hoạt Động](#cách-hoạt-động)
4. [Kích Hoạt IP Ranges](#kích-hoạt-ip-ranges)
5. [Lấy Danh Sách IP CircleCI](#lấy-danh-sách-ip-cơ-sở-hạ-tầng-circleci)
6. [Cấu Hình Firewall/Security Groups](#cấu-hình-firewallsecurity-groups)
7. [Chi Phí và Giới Hạn](#chi-phí-và-giới-hạn)
8. [Các Trường Hợp Sử Dụng Thực Tế](#các-trường-hợp-sử-dụng-thực-tế)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## IP Ranges là gì?

Trong điều kiện bình thường, khi CircleCI chạy pipeline, các job có thể kết nối ra internet từ **bất kỳ địa chỉ IP nào** trong dải IP rộng lớn của cloud provider (AWS, GCP). Điều này gây khó khăn khi bạn muốn hạn chế ai được kết nối vào database hay API nội bộ.

```
Không có IP Ranges:
  Job CircleCI → kết nối từ IP ngẫu nhiên trong 52.x.x.x/8 (AWS)
  Firewall không biết đây là CircleCI hay attacker
  → Không thể whitelist — danh sách trắng chính xác

Có IP Ranges (Feature bật):
  Job CircleCI → kết nối từ một trong các IP cố định đã biết
  → Whitelist đúng IP CircleCI trong firewall
  → Chỉ CircleCI mới vào được, reject mọi IP khác
```

### Danh Sách IP CircleCI (Tháng 5/2026 — Ví Dụ)

CircleCI công bố danh sách IP chính thức tại:
`https://circleci.com/docs/ip-ranges/`

Dải IP ví dụ (có thể thay đổi, luôn kiểm tra tài liệu chính thức):

```
# Khu vực us-east-1 (Virginia, Mỹ)
54.164.xxx.xxx/32
34.194.xxx.xxx/32
...

# Khu vực eu-west-1 (Ireland)  
52.208.xxx.xxx/32
52.210.xxx.xxx/32
...
```

---

## Tại Sao Cần IP Ranges?

### Các Tình Huống Yêu Cầu IP Cố Định

| Tình Huống | Vấn Đề Nếu Không Có IP Ranges |
|------------|-------------------------------|
| Database trong VPC có Security Group | Không thể whitelist IP CircleCI → phải mở public |
| API nội bộ có IP whitelist | Pipeline không thể gọi được API |
| Deployment target có firewall nghiêm ngặt | `kubectl apply` bị từ chối |
| License server kiểm tra IP | License bị từ chối |
| Third-party service với IP allowlist | Tích hợp không hoạt động |
| Compliance requirement (PCI DSS, HIPAA) | Kiểm toán yêu cầu nguồn gốc kết nối rõ ràng |

### Ví Dụ Thực Tế

```
Bài toán:
  Công ty có PostgreSQL database trong AWS RDS
  Security Group chỉ cho phép các IP nội bộ
  CircleCI cần chạy migration khi deploy
  
Không có IP Ranges:
  → Phải mở database ra public internet (rủi ro bảo mật cao)
  → Hoặc phải dùng VPN (phức tạp, khó tự động hóa)

Có IP Ranges:
  → Whitelist chính xác các IP CircleCI trong RDS Security Group
  → Pipeline chạy migration thành công
  → Database vẫn không public
```

---

## Cách Hoạt Động

### Kiến Trúc Kỹ Thuật

```
Bình thường (không có IP Ranges):
  CircleCI Job → Docker container → NAT Gateway ngẫu nhiên → Internet

Với IP Ranges bật:
  CircleCI Job → Docker container → IP Ranges proxy → Internet
                                    (IP cố định đã biết)
```

### Luồng Kết Nối

```
1. Job CircleCI chạy với ip_ranges: true
2. Tất cả outbound traffic — lưu lượng ra ngoài được định tuyến qua
   CircleCI IP Ranges infrastructure
3. External service nhận request từ IP đã được whitelist
4. Service cho phép kết nối
5. Job hoàn thành task
```

---

## Kích Hoạt IP Ranges

IP Ranges được kích hoạt ở **cấp job**, không phải cấp workflow hay pipeline:

```yaml
version: 2.1

jobs:
  database-migration:
    docker:
      - image: cimg/postgres:15.0
    
    # Bật IP Ranges cho job này
    circleci_ip_ranges: true    # ← Key kích hoạt tính năng
    
    steps:
      - checkout
      - run:
          name: Chạy migration với IP cố định
          command: |
            # Kết nối đến RDS đã whitelist IP CircleCI
            psql $DATABASE_URL -f migrations/latest.sql

  deploy-kubernetes:
    docker:
      - image: cimg/base:stable
    
    circleci_ip_ranges: true    # ← Bật cho job deploy đến cluster private
    
    steps:
      - run:
          name: Deploy lên private Kubernetes cluster
          command: |
            kubectl apply -f k8s/deployment.yaml

  # Job này KHÔNG cần IP cố định → không bật để tiết kiệm chi phí
  run-tests:
    docker:
      - image: cimg/node:20.11
    steps:
      - checkout
      - run: npm test

workflows:
  ci-cd:
    jobs:
      - run-tests                # Không cần IP Ranges
      - database-migration:
          requires: [run-tests]  # IP Ranges bật
          context: database-creds
      - deploy-kubernetes:
          requires: [database-migration]  # IP Ranges bật
```

---

## Lấy Danh Sách IP Cơ Sở Hạ Tầng CircleCI

### Qua API CircleCI

```bash
# Lấy danh sách IP hiện tại qua API
curl -s https://circleci.com/api/v1.1/ips \
  -H "Circle-Token: $CIRCLECI_TOKEN" | jq .

# Kết quả ví dụ:
# {
#   "ip_ranges": {
#     "jobs": [
#       "3.228.37.151/32",
#       "3.228.121.139/32",
#       ...
#     ]
#   }
# }
```

### Tự Động Cập Nhật Firewall

```bash
#!/bin/bash
# Script tự động cập nhật AWS Security Group với IP CircleCI mới nhất

SECURITY_GROUP_ID="sg-0123456789abcdef0"
CIRCLECI_TOKEN="your-token"

# Lấy IP mới nhất
CIRCLECI_IPS=$(curl -s "https://circleci.com/api/v1.1/ips" \
  -H "Circle-Token: $CIRCLECI_TOKEN" \
  | jq -r '.ip_ranges.jobs[]')

# Xóa rules cũ
aws ec2 revoke-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --ip-permissions file://old-circleci-rules.json

# Thêm rules mới
for IP in $CIRCLECI_IPS; do
  aws ec2 authorize-security-group-ingress \
    --group-id $SECURITY_GROUP_ID \
    --protocol tcp \
    --port 5432 \
    --cidr $IP
done

echo "Đã cập nhật Security Group với IP CircleCI mới nhất"
```

---

## Cấu Hình Firewall/Security Groups

### AWS Security Groups — Nhóm Bảo Mật AWS

```bash
# Thêm CircleCI IPs vào Security Group của RDS
SECURITY_GROUP_ID="sg-rds-database-production"

# Mảng IP từ CircleCI docs (cập nhật định kỳ)
CIRCLECI_IPS=(
  "3.228.37.151/32"
  "3.228.121.139/32"
  "3.228.150.235/32"
  # ... thêm các IP từ danh sách chính thức
)

for IP in "${CIRCLECI_IPS[@]}"; do
  aws ec2 authorize-security-group-ingress \
    --group-id $SECURITY_GROUP_ID \
    --protocol tcp \
    --port 5432 \
    --cidr "$IP" \
    --description "CircleCI IP Range for CI/CD pipeline"
done
```

### GCP Firewall Rules — Quy Tắc Tường Lửa GCP

```bash
# Tạo firewall rule cho CircleCI
gcloud compute firewall-rules create allow-circleci-db \
  --network my-vpc \
  --allow tcp:5432 \
  --source-ranges "3.228.37.151/32,3.228.121.139/32" \
  --description "Allow CircleCI IP Ranges to access PostgreSQL"
  --target-tags database-servers
```

### Azure Network Security Groups — Nhóm Bảo Mật Mạng Azure

```bash
# Thêm inbound rule cho CircleCI
az network nsg rule create \
  --resource-group my-rg \
  --nsg-name my-database-nsg \
  --name allow-circleci \
  --priority 200 \
  --source-address-prefixes "3.228.37.151/32" "3.228.121.139/32" \
  --destination-port-ranges 5432 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound
```

### Nginx / HAProxy Allowlist

```nginx
# Nginx — Chỉ cho phép CircleCI IP truy cập /deploy endpoint
location /deploy {
    allow 3.228.37.151;
    allow 3.228.121.139;
    # Thêm các IP CircleCI khác...
    deny all;
    
    proxy_pass http://backend;
}
```

---

## Chi Phí và Giới Hạn

> **Lưu ý quan trọng:** IP Ranges là tính năng **trả phí** (Scale Plan trở lên).

### Mô Hình Tính Phí

```
Tính phí dựa trên data transfer — Lượng dữ liệu truyền qua IP Ranges proxy:

Khoảng 450 credits / GB outbound traffic

Ví dụ:
  Pipeline push 1GB Docker image → ~450 credits
  Pipeline download 100MB artifacts → ~45 credits

→ Chỉ bật circleci_ip_ranges: true khi thực sự cần!
```

### Tối Ưu Chi Phí

```yaml
workflows:
  deploy:
    jobs:
      # Job 1: Download dependencies, chạy tests → không cần IP cố định
      - build-and-test:
          # Không có circleci_ip_ranges
          
      # Job 2: Migration database → CẦN IP cố định
      - database-migration:
          circleci_ip_ranges: true    # Chỉ job này dùng tính năng
          requires: [build-and-test]
          
      # Job 3: Deploy Kubernetes → CẦN IP cố định  
      - deploy:
          circleci_ip_ranges: true    # Chỉ job này dùng tính năng
          requires: [database-migration]
```

### Giới Hạn Kỹ Thuật

| Giới Hạn | Giá Trị |
|----------|---------|
| Chỉ hỗ trợ executor | Docker executor |
| Machine executor | Không hỗ trợ |
| Self-hosted runner | Không áp dụng (có IP riêng) |
| Availability | Chỉ Scale Plan |

---

## Các Trường Hợp Sử Dụng Thực Tế

### Use Case 1: Database Migration Pipeline

```yaml
version: 2.1

jobs:
  run-migrations:
    docker:
      - image: flyway/flyway:9.22
    circleci_ip_ranges: true
    environment:
      FLYWAY_URL: jdbc:postgresql://$DB_HOST:5432/myapp
    steps:
      - checkout
      - run:
          name: Chạy database migration
          command: flyway migrate
      - run:
          name: Xác nhận migration thành công
          command: flyway info

workflows:
  deploy:
    jobs:
      - run-migrations:
          context: database-production
          filters:
            branches:
              only: main
```

### Use Case 2: Deploy lên Private Kubernetes Cluster

```yaml
jobs:
  deploy-k8s:
    docker:
      - image: bitnami/kubectl:1.28
    circleci_ip_ranges: true    # K8s API server chỉ cho phép IP CircleCI
    steps:
      - checkout
      - run:
          name: Cấu hình kubectl
          command: |
            echo "$KUBE_CONFIG" | base64 -d > ~/.kube/config
      - run:
          name: Deploy lên cluster
          command: |
            kubectl set image deployment/my-app \
              my-app=registry.example.com/my-app:${CIRCLE_SHA1}
            kubectl rollout status deployment/my-app --timeout=5m
```

### Use Case 3: Gọi Internal API Có IP Whitelist

```yaml
jobs:
  trigger-deployment:
    docker:
      - image: cimg/base:stable
    circleci_ip_ranges: true    # Internal API chỉ nhận từ IP đã biết
    steps:
      - run:
          name: Trigger deployment qua internal API
          command: |
            curl -X POST https://internal-deploy-api.company.com/deploy \
              -H "Authorization: Bearer $DEPLOY_TOKEN" \
              -H "Content-Type: application/json" \
              -d "{\"version\": \"$CIRCLE_SHA1\", \"env\": \"production\"}"
```

### Use Case 4: License Server Validation

```yaml
jobs:
  build-licensed-software:
    docker:
      - image: cimg/base:stable
    circleci_ip_ranges: true    # License server kiểm tra IP
    steps:
      - run:
          name: Activate license
          command: |
            license-tool activate --key $LICENSE_KEY
            # License server verify IP → phải là IP CircleCI đã đăng ký
      - run:
          name: Build với licensed features
          command: make build-premium
```

---

## Best Practices

### 1. Chỉ Bật Khi Cần Thiết

```yaml
# ✅ Đúng: Chỉ bật cho job thực sự cần IP cố định
jobs:
  unit-tests:
    # Không cần IP cố định → không bật
    steps: [...]
    
  integration-tests:
    # Gọi internal test DB → CẦN
    circleci_ip_ranges: true
    steps: [...]
    
  build-image:
    # Chỉ build, không cần kết nối nội bộ → không bật
    steps: [...]
    
  deploy:
    # Kết nối K8s cluster nội bộ → CẦN
    circleci_ip_ranges: true
    steps: [...]
```

### 2. Tách Biệt Concern — Separation of Concerns

```yaml
# Tốt hơn: Tách job cần IP cố định ra riêng
# Thay vì 1 job lớn với IP Ranges, tách thành:
workflows:
  deploy:
    jobs:
      - build           # Không cần IP Ranges
      - test            # Không cần IP Ranges
      - push-image:     # Không cần IP Ranges (ECR public endpoint)
          requires: [build, test]
      - migrate-db:     # CẦN IP Ranges — tách riêng
          circleci_ip_ranges: true
          requires: [push-image]
      - deploy:         # CẦN IP Ranges — tách riêng
          circleci_ip_ranges: true
          requires: [migrate-db]
```

### 3. Kết Hợp với OIDC

```yaml
# Tốt nhất: Dùng IP Ranges + OIDC cùng nhau
jobs:
  deploy:
    docker:
      - image: cimg/aws:2023.09
    circleci_ip_ranges: true    # Dùng IP cố định
    steps:
      - aws-cli/setup:
          role_arn: $AWS_ROLE_ARN    # Dùng OIDC, không cần static key
      - run:
          name: Deploy với vừa IP cố định vừa temp credentials
          command: aws ecs update-service ...
```

### 4. Cập Nhật Firewall Khi IP Thay Đổi

CircleCI thỉnh thoảng thêm IP mới. Thiết lập alert để không bị gián đoạn:

```bash
# Cron job kiểm tra IP mới hàng tuần
#!/bin/bash
CURRENT_IPS=$(curl -s https://circleci.com/api/v1.1/ips | jq -r '.ip_ranges.jobs[]' | sort)
CACHED_IPS=$(cat /tmp/circleci_ips_cache.txt)

if [ "$CURRENT_IPS" != "$CACHED_IPS" ]; then
    echo "CircleCI IP list đã thay đổi! Cần cập nhật firewall rules"
    # Gửi alert Slack
    curl -X POST $SLACK_WEBHOOK -d '{"text":"⚠️ CircleCI IP list changed. Update firewall rules!"}'
    echo "$CURRENT_IPS" > /tmp/circleci_ips_cache.txt
fi
```

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: IP Ranges trong CircleCI giải quyết vấn đề gì?**

> **A:** Mặc định, CircleCI pipeline kết nối ra internet từ IP ngẫu nhiên trong dải lớn của AWS/GCP, khiến không thể whitelist chính xác trong firewall. IP Ranges cho phép job sử dụng một tập IP cố định đã biết, giúp các tổ chức có thể whitelist IP CircleCI trong Security Groups/firewall để cho phép pipeline truy cập database, API nội bộ, hoặc K8s cluster mà không cần mở public.

**Q: Khi nào nên bật và khi nào không nên bật `circleci_ip_ranges: true`?**

> **A:** Chỉ bật khi job thực sự cần kết nối đến resource có IP whitelist. Không bật cho jobs chỉ chạy tests không cần external connections, hay build Docker image. Lý do: IP Ranges là tính năng trả phí tính theo data transfer (~450 credits/GB). Bật cho mọi job sẽ tăng chi phí đáng kể mà không có lợi ích.

### Câu Hỏi Nâng Cao

**Q: IP Ranges có hoạt động với Machine executor không?**

> **A:** Không — IP Ranges chỉ hỗ trợ Docker executor. Nếu cần IP cố định với Machine executor, giải pháp thay thế là dùng Self-Hosted Runner — chạy trên server có IP cố định trong network nội bộ của công ty.

**Q: Nếu CircleCI cập nhật danh sách IP mới, điều gì xảy ra với pipeline đang chạy?**

> **A:** Pipeline đang chạy không bị ảnh hưởng ngay. Nhưng sau khi CircleCI thêm IP mới, nếu pipeline chạy từ IP mới đó mà firewall chưa được cập nhật, kết nối sẽ bị từ chối. Cần thiết lập quy trình giám sát sự thay đổi danh sách IP và cập nhật firewall kịp thời. CircleCI thường thông báo trước khi thêm IP mới.

---

## 🔗 Điều Hướng

| ← Trước | Vị Trí | Tiếp → |
|---------|--------|--------|
| [2-oidc-integration.md](./2-oidc-integration.md) | **3-ip-ranges.md** | [4-audit-log.md](./4-audit-log.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Độ Khó:** ⭐⭐ Trung Bình
**Thời Gian Đọc:** ~30 phút
