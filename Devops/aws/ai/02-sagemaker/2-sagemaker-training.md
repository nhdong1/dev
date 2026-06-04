# SageMaker Training — Huấn Luyện Mô Hình ML Trên Managed Infrastructure

> SageMaker Training cho phép chạy training jobs (công việc huấn luyện) trên managed infrastructure (hạ tầng được quản lý) — AWS tự lo provisioning, scaling và teardown máy chủ. Bạn chỉ cần cung cấp code, data và chọn instance.

---

## 📚 Mục Lục

1. [Training Job Là Gì?](#training-job-là-gì)
2. [Built-in Algorithms](#built-in-algorithms)
3. [Custom Training Scripts](#custom-training-scripts)
4. [Bring Your Own Container — BYOC](#bring-your-own-container)
5. [Hyperparameter Tuning — AMT](#hyperparameter-tuning)
6. [Distributed Training](#distributed-training)
7. [Managed Spot Training](#managed-spot-training)
8. [Debugging và Monitoring](#debugging-và-monitoring)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Training Job Là Gì?

**Training Job** (Công Việc Huấn Luyện) là một đơn vị compute được SageMaker quản lý hoàn toàn:

```
Training Job Lifecycle (Vòng Đời Training Job):

[STARTING]          [DOWNLOADING]       [TRAINING]        [UPLOADING]        [COMPLETED]
Provision    ──►    Pull Docker   ──►   Run your    ──►   Save model   ──►   Terminate
instance            image + data        training          to S3              instance
(~2-5 phút)        from S3             script
```

### Các Thành Phần Bắt Buộc

```python
estimator = sagemaker.estimator.Estimator(
    image_uri="...",           # Docker image chứa training framework
    role="...",                # IAM Role để access S3, ECR
    instance_type="ml.m5.xlarge",  # Loại instance để train
    instance_count=1,          # Số instances (>1 cho distributed training)
    output_path="s3://bucket/output/",  # Lưu model artifacts
    hyperparameters={...}      # Siêu tham số truyền vào script
)

estimator.fit({
    "train": "s3://bucket/train/",   # Channel đầu vào dữ liệu
    "test": "s3://bucket/test/"
})
```

### Input Modes (Chế Độ Đầu Vào Dữ Liệu)

| Mode | Cơ Chế | Phù Hợp |
|---|---|---|
| **File** | Copy toàn bộ S3 data xuống instance trước khi train | Dataset vừa, đọc nhiều lần |
| **Pipe** | Stream data trực tiếp từ S3 trong khi train | Dataset lớn, đọc 1 lần |
| **FastFile** | Mount S3 như local filesystem (không copy) | Dataset rất lớn |

---

## Built-in Algorithms

SageMaker cung cấp **built-in algorithms** (thuật toán tích hợp sẵn) — được AWS optimize cho performance và scale trên SageMaker infrastructure.

### Nhóm Supervised Learning (Học Có Giám Sát)

#### XGBoost (eXtreme Gradient Boosting — Tăng Cường Gradient Cực Đoan)

Thuật toán gradient boosting mạnh nhất cho tabular data (dữ liệu dạng bảng):

```python
from sagemaker.xgboost import XGBoost

estimator = XGBoost(
    entry_point='train.py',
    framework_version='1.7-1',
    instance_type='ml.m5.xlarge',
    instance_count=1,
    role=role,
    hyperparameters={
        'max_depth': 6,          # Độ sâu tối đa của cây quyết định
        'eta': 0.3,              # Learning rate (tốc độ học)
        'objective': 'reg:squarederror',  # Hàm mục tiêu
        'num_round': 100,        # Số vòng boosting
        'subsample': 0.8,        # Tỷ lệ mẫu dùng mỗi round
        'eval_metric': 'rmse'    # Metric đánh giá
    }
)
```

**Use cases tốt:** House price prediction, fraud detection, churn prediction, credit scoring.

#### Linear Learner (Người Học Tuyến Tính)

Cho regression (hồi quy) và classification (phân loại) tuyến tính:

```python
from sagemaker import LinearLearner

estimator = LinearLearner(
    role=role,
    instance_count=1,
    instance_type='ml.m4.xlarge',
    predictor_type='binary_classifier',  # 'regressor' hoặc 'multiclass_classifier'
    num_classes=2,
    positive_example_weight_mult='balanced'  # Tự động xử lý class imbalance
)
```

**Đặc biệt:** Tự động normalize features, xử lý class imbalance, tìm optimal threshold.

### Nhóm Unsupervised Learning (Học Không Giám Sát)

#### K-Means (Phân Cụm K Phần Tử)

Phân cụm (clustering) dữ liệu:

```python
from sagemaker import KMeans

estimator = KMeans(
    role=role,
    instance_count=1,
    instance_type='ml.c4.xlarge',
    k=10,           # Số cụm (clusters)
    feature_dim=784 # Số chiều features
)
```

#### PCA — Principal Component Analysis (Phân Tích Thành Phần Chính)

Giảm chiều dữ liệu (dimensionality reduction):

```python
from sagemaker import PCA

estimator = PCA(
    role=role,
    instance_count=1,
    instance_type='ml.c4.xlarge',
    num_components=50,  # Số chiều muốn giữ lại
    algorithm_mode='randomized'  # 'regular' hoặc 'randomized'
)
```

### Nhóm Deep Learning (Học Sâu)

#### Image Classification (Phân Loại Ảnh)

ResNet-based model cho phân loại ảnh:

```python
from sagemaker import image_uris

# Lấy URI của built-in Image Classification algorithm
training_image = image_uris.retrieve(
    framework='image-classification',
    region='us-east-1'
)

estimator = sagemaker.estimator.Estimator(
    image_uri=training_image,
    role=role,
    instance_count=1,
    instance_type='ml.p3.2xlarge',  # Cần GPU
    hyperparameters={
        'num_classes': 10,
        'num_training_samples': 50000,
        'epochs': 30,
        'learning_rate': 0.01,
        'mini_batch_size': 32
    }
)
```

#### Object Detection (Phát Hiện Đối Tượng)

SSD (Single Shot Detector) hoặc Faster R-CNN framework:

```python
hyperparameters = {
    'num_classes': 20,           # Số loại đối tượng cần phát hiện
    'num_training_samples': 16551,
    'base_network': 'resnet-50', # Backbone network
    'epochs': 30,
    'lr_scheduler_step': '10,20',
    'lr_scheduler_factor': 0.1
}
```

### Bảng Tổng Hợp Built-in Algorithms

| Algorithm | Loại | Use Case Điển Hình | Instance |
|---|---|---|---|
| XGBoost | Supervised | Tabular classification/regression | CPU |
| Linear Learner | Supervised | Linear classification/regression | CPU |
| Factorization Machines | Supervised | Recommendation, click prediction | CPU |
| K-Nearest Neighbors | Supervised | Classification, regression | CPU |
| Image Classification | Supervised | Ảnh → class label | GPU |
| Object Detection | Supervised | Ảnh → bounding boxes | GPU |
| Semantic Segmentation | Supervised | Ảnh → pixel-level labels | GPU |
| K-Means | Unsupervised | Clustering | CPU |
| PCA | Unsupervised | Dimensionality reduction | CPU |
| Neural Topic Model | Unsupervised | Topic modeling văn bản | CPU/GPU |
| Random Cut Forest | Unsupervised | Anomaly detection | CPU |
| BlazingText | NLP | Word2Vec, text classification | CPU/GPU |
| DeepAR | Time Series | Forecasting | CPU/GPU |

---

## Custom Training Scripts

Khi built-in algorithms không đủ, bạn mang script Python của mình.

### Script Mode (Chế Độ Script)

AWS cung cấp managed containers cho các frameworks phổ biến (TensorFlow, PyTorch, Scikit-learn, MXNet), bạn chỉ cần viết training script:

```python
# train.py — Script training của bạn
import argparse
import os
import pandas as pd
from sklearn.ensemble import GradientBoostingClassifier
import joblib

def model_fn(model_dir):
    """Hàm load model — SageMaker gọi khi serving"""
    model = joblib.load(os.path.join(model_dir, 'model.joblib'))
    return model

if __name__ == '__main__':
    # SageMaker truyền hyperparameters và data paths qua arguments
    parser = argparse.ArgumentParser()
    
    # Hyperparameters (siêu tham số)
    parser.add_argument('--n-estimators', type=int, default=100)
    parser.add_argument('--max-depth', type=int, default=5)
    
    # SageMaker environment variables (biến môi trường)
    parser.add_argument('--model-dir', type=str,
                        default=os.environ.get('SM_MODEL_DIR'))      # Nơi lưu model
    parser.add_argument('--train', type=str,
                        default=os.environ.get('SM_CHANNEL_TRAIN'))  # Thư mục data train
    parser.add_argument('--test', type=str,
                        default=os.environ.get('SM_CHANNEL_TEST'))   # Thư mục data test
    
    args = parser.parse_args()
    
    # Load data
    train_df = pd.read_csv(os.path.join(args.train, 'train.csv'))
    X_train = train_df.drop('label', axis=1)
    y_train = train_df['label']
    
    # Train model
    clf = GradientBoostingClassifier(
        n_estimators=args.n_estimators,
        max_depth=args.max_depth
    )
    clf.fit(X_train, y_train)
    
    # Lưu model — SageMaker sẽ upload thư mục SM_MODEL_DIR lên S3
    joblib.dump(clf, os.path.join(args.model_dir, 'model.joblib'))
    print("Model saved!")
```

```python
# launcher.py — Code launch training job
from sagemaker.sklearn import SKLearn

estimator = SKLearn(
    entry_point='train.py',      # Script vừa viết ở trên
    framework_version='1.2-1',
    instance_type='ml.m5.xlarge',
    role=role,
    hyperparameters={
        'n-estimators': 200,
        'max-depth': 6
    }
)

estimator.fit({
    'train': 's3://bucket/train/',
    'test': 's3://bucket/test/'
})
```

### SageMaker Environment Variables Quan Trọng

| Variable | Giá Trị | Mô Tả |
|---|---|---|
| `SM_MODEL_DIR` | `/opt/ml/model` | Lưu model artifacts tại đây |
| `SM_CHANNEL_TRAIN` | `/opt/ml/input/data/train` | Thư mục data channel "train" |
| `SM_CHANNEL_TEST` | `/opt/ml/input/data/test` | Thư mục data channel "test" |
| `SM_NUM_GPUS` | `0`, `1`, `4`, ... | Số GPU có sẵn |
| `SM_HOSTS` | `["algo-1","algo-2"]` | List các host trong distributed training |
| `SM_CURRENT_HOST` | `algo-1` | Host hiện tại |

---

## Bring Your Own Container — BYOC (Mang Container Của Bạn)

Khi cần hoàn toàn kiểm soát training environment (môi trường huấn luyện):

### Cấu Trúc Docker Image

```dockerfile
FROM python:3.10-slim

# Cài dependencies
RUN pip install --no-cache-dir \
    sagemaker-training \
    numpy pandas scikit-learn

# Copy training code
COPY train.py /opt/ml/code/train.py

# SageMaker mặc định chạy lệnh này
ENV SAGEMAKER_PROGRAM train.py
```

### Push Lên Amazon ECR (Elastic Container Registry)

```bash
# Authenticate với ECR (Elastic Container Registry — Kho Chứa Container)
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS \
    --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build và push image
docker build -t my-training-image .
docker tag my-training-image:latest \
    123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training-image:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training-image:latest
```

```python
# Sử dụng custom container
estimator = sagemaker.estimator.Estimator(
    image_uri="123456789012.dkr.ecr.us-east-1.amazonaws.com/my-training-image:latest",
    role=role,
    instance_type='ml.m5.xlarge',
    instance_count=1
)
```

---

## Hyperparameter Tuning — AMT (Automatic Model Tuning — Điều Chỉnh Mô Hình Tự Động) {#hyperparameter-tuning}

**AMT** tự động tìm bộ hyperparameters (siêu tham số) tốt nhất bằng cách chạy nhiều training jobs song song.

### Các Chiến Lược Tìm Kiếm

| Chiến Lược | Mô Tả | Phù Hợp |
|---|---|---|
| **Random Search** (Tìm Ngẫu Nhiên) | Thử tổ hợp ngẫu nhiên | Nhanh, ít tuning budget |
| **Bayesian Optimization** (Tối Ưu Bayes) | Học từ kết quả trước để đề xuất tiếp | Chất lượng cao, chậm hơn |
| **Hyperband** | Early-stopping kết hợp random | Cân bằng speed/quality |
| **Grid Search** (Tìm Lưới) | Thử tất cả tổ hợp trong grid | Nhỏ, ít hyperparams |

### Ví Dụ AMT

```python
from sagemaker.tuner import (
    HyperparameterTuner,
    IntegerParameter,
    ContinuousParameter,
    CategoricalParameter
)

# Định nghĩa không gian tìm kiếm hyperparameter
hyperparameter_ranges = {
    'max_depth': IntegerParameter(3, 10),           # Số nguyên từ 3 đến 10
    'eta': ContinuousParameter(0.01, 0.3),          # Số thực từ 0.01 đến 0.3
    'subsample': ContinuousParameter(0.5, 1.0),     # Số thực từ 0.5 đến 1.0
    'colsample_bytree': ContinuousParameter(0.5, 1.0),
    'min_child_weight': IntegerParameter(1, 10)
}

# Metric mục tiêu muốn tối ưu
objective_metric_name = 'validation:rmse'

# Tạo HyperparameterTuner
tuner = HyperparameterTuner(
    estimator=estimator,
    objective_metric_name=objective_metric_name,
    hyperparameter_ranges=hyperparameter_ranges,
    max_jobs=20,            # Tổng số training jobs chạy
    max_parallel_jobs=4,    # Số jobs chạy song song cùng lúc
    objective_type='Minimize',  # Muốn minimize RMSE
    strategy='Bayesian'     # Chiến lược Bayesian Optimization
)

# Chạy tuning job
tuner.fit({'train': s3_train_path, 'validation': s3_val_path})

# Lấy kết quả tốt nhất
best_training_job = tuner.best_training_job()
print(f"Best job: {best_training_job}")
```

### Warm Start (Khởi Động Ấm)

Tiếp tục tìm kiếm từ kết quả tuning job trước — tiết kiệm thời gian và budget:

```python
from sagemaker.tuner import WarmStartConfig, WarmStartTypes

warm_start_config = WarmStartConfig(
    warm_start_type=WarmStartTypes.IDENTICAL_DATA_AND_ALGORITHM,
    parents={"previous-tuning-job-name"}  # Kế thừa từ job trước
)

tuner = HyperparameterTuner(
    ...,
    warm_start_config=warm_start_config
)
```

---

## Distributed Training (Huấn Luyện Phân Tán)

Khi model hoặc data quá lớn cho một instance đơn lẻ.

### Data Parallelism (Song Song Hóa Dữ Liệu)

Chia nhỏ dataset, mỗi GPU/instance xử lý một phần, sau đó tổng hợp gradients:

```
Batch 1000 samples
    │
    ├── Instance 1 (GPU 1): Samples 0-249   → Gradients 1
    ├── Instance 2 (GPU 2): Samples 250-499  → Gradients 2
    ├── Instance 3 (GPU 3): Samples 500-749  → Gradients 3
    └── Instance 4 (GPU 4): Samples 750-999  → Gradients 4
                                              │
                                              ▼ AllReduce
                                         Combined Gradients
                                              │
                                              ▼
                                         Update Weights
```

**SageMaker Data Parallel Library (SMDDP — Thư Viện Song Song Dữ Liệu SageMaker):**

```python
from sagemaker.pytorch import PyTorch

estimator = PyTorch(
    entry_point='train_ddp.py',
    framework_version='2.0.0',
    instance_type='ml.p3.16xlarge',  # 8 GPU per instance
    instance_count=2,                 # 2 instances = 16 GPUs total
    distribution={
        'smdistributed': {
            'dataparallel': {
                'enabled': True,
                'custom_mpi_options': '--NCCL_DEBUG INFO'
            }
        }
    }
)
```

### Model Parallelism (Song Song Hóa Mô Hình)

Khi model quá lớn để fit trên 1 GPU — chia model thành các partitions:

```python
estimator = PyTorch(
    ...,
    distribution={
        'smdistributed': {
            'modelparallel': {
                'enabled': True,
                'parameters': {
                    'microbatches': 4,         # Micro-batches cho pipeline
                    'placement_strategy': 'spread',  # Cách đặt layers
                    'pipeline': 'interleaved',  # Pipeline execution
                    'optimize': 'speed'         # Tối ưu theo speed hoặc memory
                }
            }
        }
    }
)
```

---

## Managed Spot Training (Huấn Luyện Sử Dụng Spot Instances)

**EC2 Spot Instances** (Phiên Bản Spot EC2) là capacity dư của AWS, giá rẻ hơn On-Demand 60-90% nhưng có thể bị AWS thu hồi (interrupt) bất kỳ lúc nào.

### Cơ Chế Hoạt Động

```
Spot Training Timeline (Dòng Thời Gian):

t=0:00  [START]     Training bắt đầu trên Spot instance
t=1:30  [CHECKPOINT] Lưu checkpoint (điểm kiểm tra) lên S3
t=2:00  [INTERRUPT]  AWS thu hồi Spot instance (tiết kiệm chi phí)
          ↓
t=2:15  [RESUME]    SageMaker tự động tìm Spot instance mới
t=2:20  [RESTORE]   Load checkpoint từ S3, tiếp tục từ epoch đã lưu
t=4:00  [COMPLETE]  Training hoàn thành
```

### Bật Spot Training

```python
estimator = XGBoost(
    ...,
    use_spot_instances=True,              # Bật Spot Training
    max_run=3600,                         # Tối đa 1 giờ tổng thời gian
    max_wait=7200,                        # Tối đa 2 giờ chờ Spot capacity
    checkpoint_s3_uri='s3://bucket/checkpoints/'  # S3 path lưu checkpoints
)
```

### Viết Code Hỗ Trợ Checkpoint

```python
# train.py — Cần xử lý checkpoint để Spot Training hoạt động đúng
import os
import torch

CHECKPOINT_DIR = os.environ.get('SM_CHECKPOINTS', '/opt/ml/checkpoints')

def save_checkpoint(model, optimizer, epoch, loss):
    """Lưu checkpoint sau mỗi epoch"""
    checkpoint = {
        'epoch': epoch,
        'model_state_dict': model.state_dict(),
        'optimizer_state_dict': optimizer.state_dict(),
        'loss': loss
    }
    path = os.path.join(CHECKPOINT_DIR, f'checkpoint-epoch-{epoch}.pt')
    torch.save(checkpoint, path)
    print(f"Checkpoint saved: {path}")

def load_latest_checkpoint(model, optimizer):
    """Load checkpoint mới nhất nếu có (sau khi Spot interrupt)"""
    checkpoints = sorted([
        f for f in os.listdir(CHECKPOINT_DIR) if f.endswith('.pt')
    ])
    
    if not checkpoints:
        return 0  # Bắt đầu từ epoch 0
    
    latest = os.path.join(CHECKPOINT_DIR, checkpoints[-1])
    checkpoint = torch.load(latest)
    model.load_state_dict(checkpoint['model_state_dict'])
    optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
    start_epoch = checkpoint['epoch'] + 1
    print(f"Resuming from epoch {start_epoch}")
    return start_epoch

# Training loop
start_epoch = load_latest_checkpoint(model, optimizer)
for epoch in range(start_epoch, total_epochs):
    train_one_epoch(...)
    save_checkpoint(model, optimizer, epoch, loss)
```

### Tiết Kiệm Chi Phí Thực Tế

```
Ví Dụ: Training 10 giờ trên ml.p3.2xlarge

On-Demand:    $3.06/giờ × 10 giờ = $30.60
Spot (~70% off): $0.92/giờ × 10 giờ = $9.20 (tiết kiệm $21.40 = 70%)

Lưu ý: Với Spot, job có thể kéo dài hơn do interruptions,
nhưng chi phí vẫn thấp hơn nhiều.
```

---

## Debugging và Monitoring Training Jobs

### CloudWatch Logs (Nhật Ký CloudWatch)

Mọi training job tự động stream logs (nhật ký) lên CloudWatch:

```bash
# Xem logs training job
aws logs get-log-events \
    --log-group-name /aws/sagemaker/TrainingJobs \
    --log-stream-name my-training-job/algo-1-xxxx
```

### Training Job Status

```python
# Kiểm tra trạng thái job
import boto3
sm = boto3.client('sagemaker')

response = sm.describe_training_job(TrainingJobName='my-training-job')
print(f"Status: {response['TrainingJobStatus']}")
print(f"Secondary Status: {response['SecondaryStatus']}")
# Possible: InProgress, Completed, Failed, Stopping, Stopped

# Xem training metrics trong real-time
for metric in response.get('FinalMetricDataList', []):
    print(f"{metric['MetricName']}: {metric['Value']}")
```

### SageMaker Debugger Integration

```python
from sagemaker.debugger import DebuggerHookConfig, CollectionConfig

hook_config = DebuggerHookConfig(
    s3_output_path='s3://bucket/debug/',
    collection_configs=[
        CollectionConfig(name='weights'),      # Thu thập weights
        CollectionConfig(name='gradients'),    # Thu thập gradients
        CollectionConfig(name='losses')        # Thu thập loss values
    ]
)

estimator = PyTorch(
    ...,
    debugger_hook_config=hook_config
)
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Khi nào dùng Built-in Algorithms vs Custom Script vs BYOC?

**Trả lời tốt:**
> Tôi chọn theo mức độ từ đơn giản đến phức tạp. Built-in algorithms — khi bài toán phù hợp với XGBoost, Linear Learner, hay Image Classification và không cần tùy chỉnh sâu: lợi điểm là AWS đã optimize cho performance và scale, tôi không cần quản lý dependencies. Custom Script (Script Mode) — khi cần viết training logic riêng nhưng dùng framework chuẩn như PyTorch, TensorFlow, Scikit-learn: AWS quản lý container và runtime, tôi chỉ lo code. BYOC (Bring Your Own Container) — khi dùng thư viện đặc thù chưa có trong managed containers, hoặc cần kiểm soát hoàn toàn environment. Trên thực tế, 80% trường hợp Script Mode là đủ.

### Q2: Managed Spot Training hoạt động thế nào và rủi ro là gì?

**Trả lời tốt:**
> Spot Training dùng EC2 Spot Instances — capacity dư của AWS giá rẻ hơn 60-90% so với On-Demand. Rủi ro là AWS có thể thu hồi (interrupt) bất kỳ lúc nào khi cần capacity. SageMaker xử lý interruption bằng cách: tự động save checkpoint lên S3, sau đó tìm Spot instance mới và resume từ checkpoint. Để dùng hiệu quả, code training phải hỗ trợ checkpoint — save model state sau mỗi epoch và load checkpoint khi khởi động. Spot Training phù hợp nhất cho training jobs không time-critical, có thể chấp nhận thời gian chạy dài hơn để đổi lấy tiết kiệm chi phí lớn.

### Q3: Hyperparameter Tuning với AMT khác gì tự mình chạy Grid Search?

**Trả lời tốt:**
> AMT (Automatic Model Tuning) có hai lợi thế lớn. Thứ nhất, Bayesian Optimization strategy — thay vì thử random hoặc exhaustive grid search, AMT học từ kết quả các jobs trước để đề xuất hyperparameter values có khả năng cao cho kết quả tốt hơn, hiệu quả hơn cả Random và Grid Search. Thứ hai, parallelism — AMT chạy nhiều jobs song song (VD: 4 jobs cùng lúc), tiết kiệm thời gian tổng thể. Nếu tự chạy Grid Search trong notebook, tôi phải chờ từng job xong mới chạy tiếp. AMT còn có Warm Start để kế thừa kết quả từ tuning jobs trước — rất hữu ích khi muốn tinh chỉnh thêm mà không muốn bắt đầu lại từ đầu.

---

## 📊 Tóm Tắt

```
SageMaker Training
├── Training Job (Công Việc Huấn Luyện)
│   ├── Provision → Download → Train → Upload → Terminate
│   └── Input Modes: File | Pipe | FastFile
│
├── Algorithm Options (Lựa Chọn Thuật Toán)
│   ├── Built-in: XGBoost, Linear Learner, KMeans, Image Classification...
│   ├── Script Mode: Mang script Python + dùng managed framework container
│   └── BYOC: Mang Docker image hoàn toàn tùy chỉnh
│
├── Hyperparameter Tuning — AMT
│   ├── Strategies: Random | Bayesian | Hyperband | Grid
│   ├── Max Jobs + Max Parallel Jobs
│   └── Warm Start (kế thừa từ job trước)
│
├── Distributed Training (Huấn Luyện Phân Tán)
│   ├── Data Parallelism: Chia dataset, AllReduce gradients
│   └── Model Parallelism: Chia model layers
│
└── Managed Spot Training (Tiết Kiệm 60-90%)
    ├── Automatic checkpoint + resume sau interruption
    └── Cần: use_spot_instances=True + checkpoint_s3_uri
```

---

**File tiếp theo:** [3-sagemaker-inference.md](./3-sagemaker-inference.md) — Inference & Endpoint Deployment

**Cập Nhật Lần Cuối:** 2026-06-03
