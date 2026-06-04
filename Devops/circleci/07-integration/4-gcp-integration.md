# Tích Hợp GCP Trong CircleCI

> **GCP — Google Cloud Platform**: **GCR/Artifact Registry**, **GKE — Google Kubernetes Engine**, **Cloud Run** — chạy container serverless, và **gcloud CLI** trong pipeline.

## 📚 Mục Lục

1. [Xác Thực GCP](#xác-thực-gcp)
2. [Artifact Registry / GCR](#artifact-registry--gcr)
3. [GKE Deploy](#gke-deploy)
4. [Cloud Run](#cloud-run)
5. [Lưu Trữ & CDN](#lưu-trữ--cdn)
6. [Best Practices](#best-practices)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Xác Thực GCP

### Service Account Key (legacy — tránh nếu có thể)

```yaml
      - run:
          name: Auth gcloud
          command: |
            echo "$GCP_SERVICE_KEY" | base64 -d > ${HOME}/gcp-key.json
            gcloud auth activate-service-account --key-file=${HOME}/gcp-key.json
            gcloud config set project $GCP_PROJECT_ID
```

Key lưu trong **Context** — base64 một dòng.

### Workload Identity Federation (khuyến nghị)

Tương tự **OIDC** với AWS: CircleCI phát JWT, GCP trust pool cấp credentials ngắn hạn. Cấu hình trên GCP Console (Workload Identity Pool + Provider cho CircleCI) và orb/partner docs mới nhất.

Tham chiếu chi tiết OIDC pattern: `06-security/2-oidc-integration.md`.

---

## Artifact Registry / GCR

**Artifact Registry** thay thế GCR legacy; hostname dạng `REGION-docker.pkg.dev/PROJECT/REPO`.

```yaml
jobs:
  push-gar:
    docker:
      - image: google/cloud-sdk:slim
    steps:
      - checkout
      - setup_remote_docker
      - run:
          name: Configure docker cho Artifact Registry
          command: |
            gcloud auth configure-docker asia-southeast1-docker.pkg.dev --quiet
      - run:
          name: Build và push
          command: |
            IMAGE=asia-southeast1-docker.pkg.dev/my-project/apps/myapp
            docker build -t $IMAGE:${CIRCLE_SHA1:0:7} .
            docker push $IMAGE:${CIRCLE_SHA1:0:7}
```

---

## GKE Deploy

```yaml
      - run:
          name: Lấy credentials cluster
          command: |
            gcloud container clusters get-credentials my-gke-cluster \
              --region asia-southeast1 \
              --project my-project
      - run:
          name: Deploy
          command: |
            kubectl set image deployment/myapp \
              myapp=asia-southeast1-docker.pkg.dev/my-project/apps/myapp:${CIRCLE_SHA1:0:7} \
              -n production
            kubectl rollout status deployment/myapp -n production
```

Service account cần roles: `roles/container.developer` (hoặc tối thiểu hơn với custom).

---

## Cloud Run

**Cloud Run** — chạy container theo request, scale tự động:

```yaml
      - run:
          name: Deploy Cloud Run
          command: |
            gcloud run deploy my-service \
              --image asia-southeast1-docker.pkg.dev/my-project/apps/myapp:${CIRCLE_SHA1:0:7} \
              --region asia-southeast1 \
              --platform managed \
              --allow-unauthenticated   # Chỉ dev; prod dùng IAM
```

Deploy từ source (không Docker) cũng được với `gcloud run deploy --source .` nhưng CI thường build image trước để kiểm soát artifact.

---

## Lưu Trữ & CDN

```yaml
      - run:
          command: |
            gsutil -m rsync -r -d ./dist gs://my-static-bucket
            gcloud compute url-maps invalidate-cdn-cache my-lb --path "/*"
```

`gsutil` song song (`-m`) tăng tốc upload lớn.

---

## Best Practices

1. Một **GCP project** cho staging, một cho production — Context tách biệt
2. Dùng **Artifact Registry** thống nhất thay GCR `gcr.io` cũ
3. **Cloud Run** production: bỏ `--allow-unauthenticated`, dùng IAM + load balancer
4. Pin `google/cloud-sdk` image tag hoặc cài gcloud version cố định
5. Kết hợp **VPC connector** cho Cloud Run truy cập private DB (cấu hình ngoài CircleCI)

---

## Câu Hỏi Phỏng Vấn

**Cloud Run vs GKE?**  
Cloud Run: ít vận hành cluster, phù hợp API/stateless; GKE: cần Kubernetes đầy đủ, DaemonSet, network policy phức tạp.

**Làm sao deploy đa region trên GCP?**  
Job matrix hoặc nhiều step `gcloud run deploy --region` với cùng image tag SHA.

**So sánh AWS ECR và GCP Artifact Registry?**  
Cùng vai trò registry; CI pattern build → auth configure-docker → push giống nhau.
