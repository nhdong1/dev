# ServiceAccount — Danh Tính Cho Pod Trong Kubernetes

> Hướng dẫn chi tiết về ServiceAccount (Tài Khoản Dịch Vụ): tạo và quản lý, token projection, kiểm soát automount, và IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho ServiceAccount) trên Amazon EKS.

## Mục Lục

1. [ServiceAccount Là Gì?](#serviceaccount-là-gì)
2. [Token ServiceAccount](#token-serviceaccount)
3. [Tạo và Cấu Hình ServiceAccount](#tạo-và-cấu-hình-serviceaccount)
4. [Gán ServiceAccount Cho Pod](#gán-serviceaccount-cho-pod)
5. [Kiểm Soát Token Mounting](#kiểm-soát-token-mounting)
6. [IRSA Trên EKS](#irsa-trên-eks)
7. [Workload Identity Trên GKE](#workload-identity-trên-gke)
8. [Best Practice](#best-practice)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ServiceAccount Là Gì?

**ServiceAccount** là danh tính (identity) Kubernetes cấp cho **tiến trình chạy trong Pod** — khác với user account dành cho con người.

```
Con người                    Pod
    │                          │
    │ dùng                     │ dùng
    ▼                          ▼
User Account             ServiceAccount
(certificate / OIDC)     (token JWT tự động mount)
    │                          │
    └──────────┬───────────────┘
               │ cả hai đều là subject của
               ▼
           RBAC Role / ClusterRole
```

**Tại sao cần ServiceAccount?**

- Pod cần gọi Kubernetes API (operator, controller, ArgoCD agent)
- Pod cần truy cập dịch vụ cloud (S3, RDS) — qua IRSA trên EKS
- Pod cần xác thực với dịch vụ nội bộ khác

**Mặc định:** Nếu không khai báo, Pod dùng ServiceAccount `default` trong namespace. SA `default` không có RBAC gì nhưng token vẫn được mount vào Pod — đây là rủi ro bảo mật nếu ứng dụng không cần gọi K8s API.

---

## Token ServiceAccount

### Lịch Sử Thay Đổi

```
K8s < 1.20:  Token không expire, lưu trong Secret, mount tự động vào Pod
K8s 1.20+:   Token-based ServiceAccount Token (bound service account token) — có expire
K8s 1.24+:   Không tự tạo Secret token — dùng projected volume với TTL
```

### Bound Service Account Token (Từ K8s 1.20+)

Token mới có 4 thuộc tính ràng buộc:
- **Audience (Đối Tượng):** Token chỉ hợp lệ cho một audience cụ thể (mặc định: API Server)
- **Expiry (Hết Hạn):** Token expire sau 1 giờ mặc định, kubelet tự refresh
- **Pod binding:** Token ràng buộc với Pod cụ thể — Pod khác không dùng được
- **Namespace binding:** Token chỉ hợp lệ trong namespace của SA

```bash
# Xem token được mount trong Pod
kubectl exec -it myapp-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Decode JWT để xem nội dung (không cần secret)
kubectl exec -it myapp-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d '.' -f 2 | base64 -d 2>/dev/null | python3 -m json.tool
```

Token payload ví dụ:
```json
{
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1747612800,
  "iat": 1747609200,
  "iss": "https://kubernetes.default.svc",
  "kubernetes.io": {
    "namespace": "production",
    "pod": {
      "name": "myapp-pod-abc123",
      "uid": "12345678-1234-1234-1234-123456789012"
    },
    "serviceaccount": {
      "name": "myapp-sa",
      "uid": "abcdef12-abcd-abcd-abcd-abcdef123456"
    }
  },
  "sub": "system:serviceaccount:production:myapp-sa"
}
```

### Tạo Token Thủ Công (Cho Non-Pod Usage)

```bash
# Tạo token có TTL (time to live — thời gian sống) 1 giờ
kubectl create token myapp-sa -n production --duration=1h

# Tạo token cho CI/CD (24 giờ)
kubectl create token github-actions-sa -n cicd --duration=24h

# Tạo Secret token không expire (legacy — không khuyến nghị)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: myapp-sa-token
  namespace: production
  annotations:
    kubernetes.io/service-account.name: myapp-sa
type: kubernetes.io/service-account-token
EOF
```

---

## Tạo và Cấu Hình ServiceAccount

### Tạo ServiceAccount Cơ Bản

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    # Annotation cho IRSA (EKS) — xem phần sau
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/myapp-role
  labels:
    app: myapp
    team: backend
```

```bash
# Tạo bằng kubectl
kubectl create serviceaccount myapp-sa -n production

# Xem ServiceAccount
kubectl get serviceaccount myapp-sa -n production -o yaml

# Xem token được tạo tự động (K8s 1.24+ không còn Secret token)
kubectl get secrets -n production | grep myapp-sa
```

### ServiceAccount + RBAC

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["myapp-secrets"]     # chỉ secret cụ thể
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
```

---

## Gán ServiceAccount Cho Pod

### Khai Báo Trong Pod/Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: myapp-sa        # gán ServiceAccount
      automountServiceAccountToken: true  # mount token vào Pod (mặc định true)
      containers:
        - name: app
          image: myapp:v1
```

### Sử Dụng Token Trong Ứng Dụng

```python
# Python — gọi Kubernetes API từ trong Pod
import requests

# Token được mount tự động
with open('/var/run/secrets/kubernetes.io/serviceaccount/token') as f:
    token = f.read().strip()

# CA certificate của cluster
ca_cert = '/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'

# Namespace hiện tại
with open('/var/run/secrets/kubernetes.io/serviceaccount/namespace') as f:
    namespace = f.read().strip()

# Gọi K8s API
headers = {'Authorization': f'Bearer {token}'}
response = requests.get(
    f'https://kubernetes.default.svc/api/v1/namespaces/{namespace}/configmaps',
    headers=headers,
    verify=ca_cert
)
```

```go
// Go — dùng client-go với in-cluster config
import (
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
)

func main() {
    // Tự động dùng token ServiceAccount được mount
    config, err := rest.InClusterConfig()
    clientset, err := kubernetes.NewForConfig(config)
    
    // Gọi API
    pods, err := clientset.CoreV1().Pods("production").List(context.TODO(), metav1.ListOptions{})
}
```

---

## Kiểm Soát Token Mounting

### Tắt Auto-mount Cho Pod Không Cần Gọi K8s API

```yaml
# Tắt ở cấp ServiceAccount — áp dụng cho mọi Pod dùng SA này
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-k8s-api-sa
  namespace: production
automountServiceAccountToken: false    # tắt mặc định cho SA này

---
# Tắt ở cấp Pod — override SA setting
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: myapp-sa
      automountServiceAccountToken: false   # Pod không mount token dù SA cho phép
```

### Projected Volume — Token Với Audience Tuỳ Chỉnh

```yaml
spec:
  volumes:
    - name: aws-token
      projected:
        sources:
          - serviceAccountToken:
              audience: sts.amazonaws.com    # audience cho AWS STS
              expirationSeconds: 86400       # 24 giờ
              path: token

    - name: k8s-token
      projected:
        sources:
          - serviceAccountToken:
              audience: https://kubernetes.default.svc
              expirationSeconds: 3600        # 1 giờ
              path: token

  containers:
    - name: app
      volumeMounts:
        - name: aws-token
          mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
          readOnly: true
```

---

## IRSA Trên EKS

**IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho ServiceAccount)** cho phép Pod trên EKS assume IAM role mà **không cần lưu AWS access key**. Đây là cách bảo mật nhất để Pod truy cập AWS services (S3, RDS, DynamoDB, SQS...).

### Cơ Chế Hoạt Động

```
Pod
 │  có ServiceAccount với annotation IAM role ARN
 │  token ServiceAccount (audience: sts.amazonaws.com) được mount
 │
 ▼
AWS SDK trong Pod gọi AssumeRoleWithWebIdentity
 │  gửi: token + IAM role ARN
 │
 ▼
AWS STS (Security Token Service — Dịch Vụ Token Bảo Mật)
 │  verify token qua OIDC endpoint của EKS cluster
 │  kiểm tra trust policy: namespace + SA name có khớp không?
 │
 ▼
Trả về temporary credentials (AccessKeyId, SecretAccessKey, SessionToken)
 │  hết hạn sau 1 giờ — SDK tự refresh
 │
 ▼
Pod dùng temporary credentials để gọi S3, RDS, v.v.
```

### Thiết Lập IRSA Step-by-Step

#### Bước 1: Tạo OIDC Provider Cho EKS Cluster

```bash
# Lấy OIDC endpoint của cluster
aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" --output text
# Ví dụ: https://oidc.eks.us-east-1.amazonaws.com/id/ABCDEF1234567890

# Tạo OIDC provider (thực hiện một lần cho mỗi cluster)
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve
```

#### Bước 2: Tạo IAM Role Với Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/ABCDEF1234567890"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/ABCDEF1234567890:sub": 
            "system:serviceaccount:production:myapp-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/ABCDEF1234567890:aud": 
            "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

```bash
# Tạo role với eksctl (dễ hơn)
eksctl create iamserviceaccount \
  --name myapp-sa \
  --namespace production \
  --cluster my-cluster \
  --role-name myapp-s3-role \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

#### Bước 3: Annotate ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/myapp-s3-role
    eks.amazonaws.com/token-expiration: "86400"    # token TTL (tuỳ chọn)
```

#### Bước 4: Sử Dụng Trong Pod

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: myapp-sa    # SA đã có annotation IRSA
      containers:
        - name: app
          image: myapp:v1
          env:
            - name: AWS_REGION
              value: us-east-1
          # Không cần AWS_ACCESS_KEY_ID hay AWS_SECRET_ACCESS_KEY
          # AWS SDK tự detect IRSA qua environment variables được inject bởi EKS:
          # AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
          # AWS_ROLE_ARN=arn:aws:iam::123456789012:role/myapp-s3-role
```

```python
# Python AWS SDK (boto3) — không cần credential
import boto3

# boto3 tự detect IRSA từ environment variables
s3 = boto3.client('s3', region_name='us-east-1')
objects = s3.list_objects_v2(Bucket='my-bucket')
```

### Verify IRSA Hoạt Động

```bash
# Exec vào Pod và kiểm tra identity
kubectl exec -it myapp-pod -n production -- \
  aws sts get-caller-identity

# Kết quả mong đợi:
# {
#   "UserId": "AROAEXAMPLEID:botocore-session-xxxxx",
#   "Account": "123456789012",
#   "Arn": "arn:aws:sts::123456789012:assumed-role/myapp-s3-role/botocore-session-xxxxx"
# }
```

---

## Workload Identity Trên GKE

**Workload Identity** là tương đương IRSA trên Google Kubernetes Engine — cho phép Pod assume Google Service Account (GSA) mà không cần key file.

```bash
# Tạo Google Service Account
gcloud iam service-accounts create myapp-gsa \
  --project=my-project

# Gán quyền cho GSA
gcloud projects add-iam-policy-binding my-project \
  --member "serviceAccount:myapp-gsa@my-project.iam.gserviceaccount.com" \
  --role "roles/storage.objectViewer"

# Liên kết Kubernetes SA với Google SA
gcloud iam service-accounts add-iam-policy-binding myapp-gsa@my-project.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:my-project.svc.id.goog[production/myapp-sa]"

# Annotate Kubernetes SA
kubectl annotate serviceaccount myapp-sa \
  --namespace production \
  iam.gke.io/gcp-service-account=myapp-gsa@my-project.iam.gserviceaccount.com
```

---

## Best Practice

### Tối Thiểu Hoá Quyền ServiceAccount

```yaml
# Tắt auto-mount cho SA không cần gọi K8s API
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: production
automountServiceAccountToken: false   # ứng dụng web thông thường không cần K8s API

---
# Chỉ tạo SA với token khi thực sự cần
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-operator-sa
  namespace: production
# automountServiceAccountToken: true (default) — operator cần gọi K8s API
```

### Không Dùng SA `default`

```bash
# Kiểm tra Pod nào đang dùng SA default
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.serviceAccountName}{"\n"}{end}' | grep "default"

# Tạo SA riêng cho từng application
kubectl create serviceaccount app1-sa -n production
kubectl create serviceaccount app2-sa -n production
```

### Rotate Token Định Kỳ

```bash
# Xoá Secret token cũ (K8s sẽ tạo lại)
kubectl delete secret myapp-sa-token -n production

# Với bound token (K8s 1.20+), kubelet tự động rotate — không cần thủ công
```

### Giám Sát ServiceAccount Activity

```bash
# Xem events liên quan đến ServiceAccount
kubectl get events -n production --field-selector reason=TokenReview

# Audit log: tìm SA activity bất thường
# Log format: user.username = "system:serviceaccount:production:myapp-sa"
```

---

## Câu Hỏi Phỏng Vấn

**ServiceAccount khác User Account thế nào?**

> `User Account` dành cho con người — được quản lý bên ngoài K8s (certificate, OIDC provider, LDAP) và có hiệu lực toàn cluster. `ServiceAccount` dành cho tiến trình trong Pod — được K8s quản lý, lưu trong namespace, có token JWT được mount tự động vào Pod. Cả hai đều là subject trong RBAC và có thể được gán Role/ClusterRole như nhau.

**IRSA hoạt động thế nào? Tại sao tốt hơn lưu AWS credential trong Secret?**

> IRSA cho phép Pod assume IAM role qua OIDC federation: (1) Pod có ServiceAccount với annotation IAM role ARN; (2) EKS mount token với audience `sts.amazonaws.com`; (3) AWS SDK gọi `AssumeRoleWithWebIdentity` gửi token + role ARN; (4) AWS STS verify token qua OIDC endpoint của cluster và kiểm tra trust policy (namespace + SA name phải khớp); (5) STS trả về temporary credential hết hạn sau 1 giờ, SDK tự refresh. Tốt hơn secret vì: không có static credential nào để lộ, credential tự rotate mỗi giờ, phạm vi giới hạn theo IAM policy, audit đầy đủ qua CloudTrail.

**Điều gì xảy ra nếu không khai báo serviceAccountName trong Pod?**

> Pod sẽ dùng ServiceAccount `default` của namespace. SA `default` không có RBAC gì nhưng token vẫn được mount tại `/var/run/secrets/kubernetes.io/serviceaccount/token`. Rủi ro: nếu ứng dụng bị compromise, attacker có token để gọi K8s API (dù quyền hạn chế). Best practice: tắt `automountServiceAccountToken: false` ở SA `default` của namespace production, và luôn tạo SA riêng với quyền tối thiểu.

**Làm thế nào để Pod trong namespace A truy cập Secret trong namespace B?**

> Không thể trực tiếp — đây là thiết kế bảo mật cố ý. Namespace là ranh giới phân vùng tài nguyên. Các cách xử lý: (1) Dùng External Secrets Operator để đồng bộ secret từ Vault/AWS Secrets Manager vào cả hai namespace; (2) Tạo controller/operator có ClusterRole đọc secret từ nhiều namespace rồi làm trung gian; (3) Mount secret qua NFS hoặc shared volume (không khuyến nghị vì phức tạp và kém bảo mật); (4) Nếu Pod thực sự cần, xem xét lại kiến trúc — thường có vấn đề thiết kế service boundary.
