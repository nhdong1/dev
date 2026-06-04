# CloudFormation Template Anatomy — Giải Phẫu Cấu Trúc Template

> **Template** là file YAML hoặc JSON mô tả toàn bộ tài nguyên AWS bạn muốn tạo. Hiểu rõ cấu trúc template là nền tảng để viết, đọc và debug CloudFormation hiệu quả.

---

## 📚 Mục Lục

1. [Cấu Trúc Tổng Quan](#cấu-trúc-tổng-quan)
2. [AWSTemplateFormatVersion & Description](#awstemplateformatversion--description)
3. [Parameters — Tham Số Đầu Vào](#parameters--tham-số-đầu-vào)
4. [Mappings — Bảng Tra Cứu](#mappings--bảng-tra-cứu)
5. [Conditions — Điều Kiện](#conditions--điều-kiện)
6. [Resources — Tài Nguyên (Bắt Buộc)](#resources--tài-nguyên-bắt-buộc)
7. [Outputs — Giá Trị Xuất Ra](#outputs--giá-trị-xuất-ra)
8. [Metadata — Cấu Hình Giao Diện](#metadata--cấu-hình-giao-diện)
9. [Intrinsic Functions — Hàm Tích Hợp](#intrinsic-functions--hàm-tích-hợp)
10. [Pseudo Parameters — Tham Số Giả](#pseudo-parameters--tham-số-giả)
11. [Ví Dụ Template Hoàn Chỉnh](#ví-dụ-template-hoàn-chỉnh)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cấu Trúc Tổng Quan

```yaml
AWSTemplateFormatVersion: "2010-09-09"    # (Tùy chọn) Chỉ có 1 giá trị hợp lệ
Description: "Mô tả ngắn về stack này"    # (Tùy chọn)

Metadata:                                  # (Tùy chọn) Cấu hình giao diện Console
  AWS::CloudFormation::Interface:
    ParameterGroups: [...]

Parameters:                                # (Tùy chọn) Tham số đầu vào
  Environment:
    Type: String

Mappings:                                  # (Tùy chọn) Bảng tra cứu tĩnh
  RegionToAMI:
    ap-southeast-1:
      Amazon2: ami-0abcdef1234567890

Conditions:                                # (Tùy chọn) Logic điều kiện
  IsProd: !Equals [!Ref Environment, prod]

Resources:                                 # [BẮT BUỘC] Danh sách tài nguyên
  MyBucket:
    Type: AWS::S3::Bucket

Outputs:                                   # (Tùy chọn) Giá trị xuất ra
  BucketName:
    Value: !Ref MyBucket
    Export:
      Name: SharedBucketName
```

> **Lưu ý:** Chỉ có `Resources` là bắt buộc. Mọi section khác đều tùy chọn.

---

## AWSTemplateFormatVersion & Description

```yaml
AWSTemplateFormatVersion: "2010-09-09"
# Đây là giá trị DUY NHẤT hợp lệ tính đến nay.
# Bỏ qua section này cũng được — CloudFormation tự nhận diện.

Description: |
  Production VPC Stack — tạo VPC 3 tầng với public, private và database subnets.
  Sử dụng cho môi trường production ở region ap-southeast-1.
# Tối đa 1024 ký tự. Hiển thị trên CloudFormation Console.
```

---

## Parameters — Tham Số Đầu Vào

Parameters cho phép người dùng truyền giá trị vào template khi deploy, tránh hard-code.

### Các Kiểu Dữ Liệu

```yaml
Parameters:
  # === String ===
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]    # Giới hạn giá trị được phép
    Description: "Môi trường triển khai"

  # === Number ===
  DesiredCapacity:
    Type: Number
    Default: 2
    MinValue: 1
    MaxValue: 10

  # === List<Number> ===
  AZList:
    Type: List<Number>

  # === CommaDelimitedList ===
  SubnetIds:
    Type: CommaDelimitedList
    Description: "Danh sách Subnet ID, cách nhau bằng dấu phẩy"

  # === AWS-Specific Parameter Types (Tự Validate) ===
  KeyPairName:
    Type: AWS::EC2::KeyPair::KeyName        # Chỉ cho phép key pair tồn tại
  
  VpcId:
    Type: AWS::EC2::VPC::Id                 # Chỉ cho phép VPC ID hợp lệ
  
  SubnetList:
    Type: List<AWS::EC2::Subnet::Id>        # Danh sách Subnet ID hợp lệ

  # === SSM Parameter Store (Lấy giá trị động) ===
  LatestAMI:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
    # Tự động lấy AMI ID mới nhất từ SSM — không cần cập nhật template thủ công

  # === NoEcho — Ẩn giá trị nhạy cảm ===
  DBPassword:
    Type: String
    NoEcho: true                            # Không hiển thị trong Console/Events
    MinLength: 8
    MaxLength: 41
    ConstraintDescription: "Mật khẩu phải từ 8–41 ký tự"
```

### Tham Chiếu Parameter Trong Template

```yaml
Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !If
        - IsProd
        - m5.large
        - t3.micro
      KeyName: !Ref KeyPairName          # Tham chiếu parameter
      ImageId: !Ref LatestAMI
      Tags:
        - Key: Environment
          Value: !Ref Environment
```

---

## Mappings — Bảng Tra Cứu

Mappings là bảng key-value tĩnh — dùng để ánh xạ giá trị không thay đổi theo môi trường, region hoặc account.

```yaml
Mappings:
  # Ánh xạ Region → AMI ID
  RegionAMIMap:
    ap-southeast-1:                        # Singapore
      Amazon2: ami-0abcdef1234567890
      Ubuntu: ami-0fedcba9876543210
    us-east-1:                             # N. Virginia
      Amazon2: ami-0123456789abcdef0
      Ubuntu: ami-0987654321fedcba0

  # Ánh xạ Environment → Instance Type
  EnvironmentInstanceType:
    prod:
      InstanceType: m5.xlarge
      MultiAZ: "true"
    staging:
      InstanceType: m5.large
      MultiAZ: "false"
    dev:
      InstanceType: t3.micro
      MultiAZ: "false"

  # Ánh xạ Region → Availability Zones
  AZMap:
    ap-southeast-1:
      AZ1: ap-southeast-1a
      AZ2: ap-southeast-1b
      AZ3: ap-southeast-1c
```

### Sử Dụng FindInMap

```yaml
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      # Cú pháp: !FindInMap [MapName, TopLevelKey, SecondLevelKey]
      ImageId: !FindInMap
        - RegionAMIMap
        - !Ref AWS::Region                 # Key Level 1: Region hiện tại
        - Amazon2                          # Key Level 2: OS type

      InstanceType: !FindInMap
        - EnvironmentInstanceType
        - !Ref Environment
        - InstanceType
```

> **Lưu ý:** FindInMap hỗ trợ tối đa 3 cấp key. Giá trị trong Mappings là static — không thể dùng Ref hay hàm động bên trong.

---

## Conditions — Điều Kiện

Conditions cho phép tạo hoặc không tạo resource tùy thuộc vào giá trị parameter.

### Khai Báo Conditions

```yaml
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]

  EnableMultiAZ:
    Type: String
    AllowedValues: ["true", "false"]
    Default: "false"

Conditions:
  # Điều kiện đơn giản
  IsProd: !Equals [!Ref Environment, prod]
  IsNotProd: !Not [!Equals [!Ref Environment, prod]]
  
  # Điều kiện phức hợp
  IsProdOrStaging: !Or
    - !Equals [!Ref Environment, prod]
    - !Equals [!Ref Environment, staging]

  IsProdAndMultiAZ: !And
    - !Equals [!Ref Environment, prod]
    - !Equals [!Ref EnableMultiAZ, "true"]

  IsUsEast1: !Equals [!Ref AWS::Region, us-east-1]
```

### Áp Dụng Conditions

```yaml
Resources:
  # Tạo resource chỉ khi IsProd = true
  ProdAlarmTopic:
    Type: AWS::SNS::Topic
    Condition: IsProd              # Resource này chỉ được tạo ở môi trường prod

  # Tạo với cấu hình khác nhau dựa theo condition
  MyRDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: !If
        - IsProd
        - db.r5.large              # Prod: instance lớn hơn
        - db.t3.micro              # Non-prod: instance nhỏ
      MultiAZ: !If
        - IsProdAndMultiAZ
        - true
        - false
      BackupRetentionPeriod: !If
        - IsProd
        - 7                        # Prod: giữ backup 7 ngày
        - 1                        # Non-prod: giữ 1 ngày

  # Tạo Elastic IP chỉ cho prod
  ProdEIP:
    Type: AWS::EC2::EIP
    Condition: IsProd
    Properties:
      InstanceId: !Ref MyEC2

Outputs:
  # Output có điều kiện
  AlarmTopicARN:
    Condition: IsProd
    Value: !Ref ProdAlarmTopic
```

---

## Resources — Tài Nguyên (Bắt Buộc)

Resources là section quan trọng nhất — mô tả tất cả tài nguyên AWS cần tạo.

### Cấu Trúc Resource

```yaml
Resources:
  ResourceLogicalId:              # Tên logic (dùng nội bộ trong template)
    Type: AWS::Service::Resource  # Resource type (xác định loại tài nguyên)
    DependsOn: OtherResource      # (Tùy chọn) Ép thứ tự tạo
    DeletionPolicy: Retain        # (Tùy chọn) Hành vi khi xóa stack
    UpdateReplacePolicy: Retain   # (Tùy chọn) Hành vi khi resource bị replace
    Condition: IsProd             # (Tùy chọn) Chỉ tạo nếu condition đúng
    Properties:                   # Cấu hình cụ thể của resource
      PropertyKey: PropertyValue
    Metadata:                     # (Tùy chọn) Metadata cho resource
      AWS::CloudFormation::Init:
        ...
```

### DeletionPolicy — Chính Sách Xóa

```yaml
Resources:
  ProductionDatabase:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot      # Tạo snapshot trước khi xóa
    # Các giá trị:
    # - Delete    (mặc định): Xóa resource khi xóa stack
    # - Retain    : Giữ lại resource, không xóa khi xóa stack
    # - Snapshot  : Tạo snapshot trước khi xóa (chỉ áp dụng cho RDS, ElastiCache, Redshift)

  ImportantS3Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain        # Bucket vẫn tồn tại sau khi xóa stack

  LogGroup:
    Type: AWS::Logs::LogGroup
    DeletionPolicy: Delete        # Xóa cùng với stack (mặc định)
```

### UpdateReplacePolicy — Chính Sách Khi Thay Thế

```yaml
Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    UpdateReplacePolicy: Retain   # Nếu update buộc phải tạo mới instance, giữ lại instance cũ
    # Có giá trị tương tự DeletionPolicy: Delete | Retain | Snapshot
```

### DependsOn — Thứ Tự Tạo Tài Nguyên

```yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    DependsOn: InternetGateway    # Ép tạo IGW trước khi attach
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment  # Cần attachment trước
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
```

> **Lưu ý:** CloudFormation tự động xử lý dependency qua Ref và GetAtt. DependsOn chỉ cần khi dependency không thể hiện qua Ref/GetAtt.

---

## Outputs — Giá Trị Xuất Ra

Outputs cho phép export giá trị ra ngoài stack để:
- Xem trên Console
- Dùng trong CLI/API
- Import vào stack khác (Cross-Stack Reference)

```yaml
Outputs:
  # Output đơn giản
  VPCId:
    Description: "ID của VPC vừa tạo"
    Value: !Ref MyVPC

  # Output với Export — để stack khác import
  PublicSubnetId:
    Description: "Public Subnet ID cho Load Balancer"
    Value: !Ref PublicSubnet
    Export:
      Name: !Sub "${AWS::StackName}-PublicSubnetId"
      # Tên Export phải duy nhất trong toàn account + region

  # Output dùng GetAtt
  LoadBalancerDNS:
    Description: "DNS name của Application Load Balancer"
    Value: !GetAtt ApplicationLoadBalancer.DNSName

  # Output có điều kiện
  BastionEIP:
    Condition: IsProd
    Description: "Elastic IP của Bastion Host (chỉ prod)"
    Value: !Ref BastionHostEIP
```

### Cross-Stack Reference (Tham Chiếu Chéo Giữa Stack)

```yaml
# Stack A — xuất VPC ID
Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: NetworkStack-VPCId

# Stack B — import VPC ID từ Stack A
Resources:
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !ImportValue NetworkStack-VPCId   # Import từ Stack A
```

> **Ràng buộc quan trọng:** Không thể xóa Stack A khi Stack B đang import output của nó. Phải xóa Stack B trước.

---

## Metadata — Cấu Hình Giao Diện

```yaml
Metadata:
  # Nhóm và sắp xếp Parameters trên Console
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: "Cài Đặt Mạng"
        Parameters:
          - VpcCidr
          - PublicSubnetCidr
          - PrivateSubnetCidr
      - Label:
          default: "Cài Đặt Ứng Dụng"
        Parameters:
          - Environment
          - InstanceType
          - KeyPairName
    ParameterLabels:
      VpcCidr:
        default: "CIDR Block cho VPC"
      Environment:
        default: "Môi Trường Triển Khai"
```

---

## Intrinsic Functions — Hàm Tích Hợp

### Bảng Tổng Hợp

| Hàm | Cú Pháp YAML | Mô Tả |
|-----|-------------|-------|
| `Ref` | `!Ref LogicalId` | Tham chiếu resource hoặc parameter |
| `Fn::GetAtt` | `!GetAtt Resource.Attribute` | Lấy attribute của resource |
| `Fn::Sub` | `!Sub "chuỗi ${Var}"` | Nội suy biến vào chuỗi |
| `Fn::Join` | `!Join [",", [a, b, c]]` | Nối chuỗi với separator |
| `Fn::Select` | `!Select [0, !Ref ListParam]` | Chọn phần tử từ list |
| `Fn::Split` | `!Split [",", "a,b,c"]` | Tách chuỗi thành list |
| `Fn::If` | `!If [CondName, A, B]` | Trả về A nếu condition đúng, B nếu sai |
| `Fn::Not` | `!Not [Condition]` | Phủ định condition |
| `Fn::And` | `!And [C1, C2]` | AND của nhiều condition |
| `Fn::Or` | `!Or [C1, C2]` | OR của nhiều condition |
| `Fn::Equals` | `!Equals [A, B]` | So sánh bằng nhau |
| `Fn::FindInMap` | `!FindInMap [Map, K1, K2]` | Tra cứu Mappings |
| `Fn::ImportValue` | `!ImportValue ExportName` | Import Output từ stack khác |
| `Fn::Base64` | `!Base64 "string"` | Encode Base64 (UserData) |
| `Fn::Cidr` | `!Cidr [CIDR, Count, Mask]` | Tạo danh sách CIDR từ CIDR cha |
| `Fn::GetAZs` | `!GetAZs !Ref AWS::Region` | Lấy danh sách AZ của region |
| `Fn::Length` | `!Length List` | Đếm phần tử trong list |

### Ví Dụ Thực Tế

```yaml
Resources:
  MyEC2:
    Type: AWS::EC2::Instance
    Properties:
      # Ref — lấy giá trị parameter
      KeyName: !Ref KeyPairName

      # GetAtt — lấy attribute của resource khác
      SubnetId: !GetAtt PublicSubnet.SubnetId

      # Sub — nội suy chuỗi
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-WebServer-${AWS::Region}"
        - Key: Project
          Value: !Sub "arn:aws:iam::${AWS::AccountId}:root"

      # UserData — cần encode Base64
      UserData:
        !Base64 |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          echo "Hello from $(hostname)" > /var/www/html/index.html

  # Cidr — tự động chia subnet
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select
        - 0
        - !Cidr [!GetAtt VPC.CidrBlock, 6, 8]
        # Chia VPC CIDR thành 6 subnet /24, lấy subnet thứ 0

  # GetAZs + Select — lấy AZ cụ thể
  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Select
        - 1
        - !GetAZs !Ref AWS::Region          # ["ap-southeast-1a", "1b", "1c", ...]
      CidrBlock: !Select
        - 1
        - !Cidr [!GetAtt VPC.CidrBlock, 6, 8]
```

---

## Pseudo Parameters — Tham Số Giả

AWS tự động cung cấp các pseudo parameters — không cần khai báo trong `Parameters`.

```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "logs-${AWS::AccountId}-${AWS::Region}"
      # Kết quả: logs-123456789012-ap-southeast-1

  MyRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "${AWS::StackName}-LambdaRole"
      # Kết quả: my-prod-stack-LambdaRole

Outputs:
  StackInfo:
    Value: !Sub |
      Stack: ${AWS::StackName}
      Account: ${AWS::AccountId}
      Region: ${AWS::Region}
      Partition: ${AWS::Partition}
      Stack ID: ${AWS::StackId}
```

| Pseudo Parameter | Ví Dụ Giá Trị | Mô Tả |
|-----------------|--------------|-------|
| `AWS::AccountId` | `123456789012` | ID tài khoản AWS |
| `AWS::Region` | `ap-southeast-1` | Region đang deploy |
| `AWS::StackName` | `prod-vpc-stack` | Tên stack |
| `AWS::StackId` | `arn:aws:cloudformation:...` | ARN đầy đủ của stack |
| `AWS::Partition` | `aws` | Partition (aws/aws-cn/aws-us-gov) |
| `AWS::URLSuffix` | `amazonaws.com` | Domain suffix |
| `AWS::NoValue` | _(loại bỏ property)_ | Dùng trong `!If` để bỏ property |

### Dùng AWS::NoValue

```yaml
Resources:
  MyRDS:
    Type: AWS::RDS::DBInstance
    Properties:
      DBSnapshotIdentifier: !If
        - IsRestore
        - !Ref SnapshotId    # Nếu restore từ snapshot
        - !Ref AWS::NoValue  # Nếu không restore — bỏ property này hoàn toàn
```

---

## Ví Dụ Template Hoàn Chỉnh

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "VPC với Public/Private Subnets — Multi-AZ"

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
  VpcCidr:
    Type: String
    Default: "10.0.0.0/16"

Mappings:
  SubnetConfig:
    VPC:
      CIDR: "10.0.0.0/16"
    PublicSubnet1:
      CIDR: "10.0.1.0/24"
    PrivateSubnet1:
      CIDR: "10.0.10.0/24"

Conditions:
  IsProd: !Equals [!Ref Environment, prod]

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-VPC"
        - Key: Environment
          Value: !Ref Environment

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  FlowLog:
    Type: AWS::EC2::FlowLog
    Condition: IsProd              # Chỉ bật VPC Flow Logs ở prod
    Properties:
      ResourceId: !Ref VPC
      ResourceType: VPC
      TrafficType: ALL
      LogDestinationType: cloud-watch-logs
      LogGroupName: !Sub "/aws/vpc/flowlogs/${AWS::StackName}"
      DeliverLogsPermissionArn: !GetAtt FlowLogRole.Arn

  FlowLogRole:
    Type: AWS::IAM::Role
    Condition: IsProd
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: vpc-flow-logs.amazonaws.com
            Action: sts:AssumeRole

Outputs:
  VPCId:
    Description: "ID của VPC"
    Value: !Ref VPC
    Export:
      Name: !Sub "${AWS::StackName}-VPCId"

  Environment:
    Description: "Môi trường triển khai"
    Value: !Ref Environment
```

---

## Câu Hỏi Phỏng Vấn

**Q: Parameters khác Mappings ở điểm nào?**
> **Parameters** là giá trị động — người dùng truyền vào lúc deploy. **Mappings** là dữ liệu tĩnh được hard-code trong template, dùng cho các ánh xạ cố định như region → AMI ID. Parameters có thể thay đổi mỗi lần deploy; Mappings luôn giống nhau cho mọi lần deploy cùng một template.

**Q: Khi nào dùng DependsOn thay vì Ref/GetAtt?**
> CloudFormation tự tạo dependency khi bạn dùng `!Ref` hoặc `!GetAtt` để tham chiếu resource khác. Chỉ dùng `DependsOn` tường minh khi dependency tồn tại nhưng không được thể hiện qua Ref/GetAtt — ví dụ: EC2 instance cần Internet Gateway đã được attach vào VPC trước khi có thể reach internet, nhưng EC2 không directly reference IGW.

**Q: NoEcho trong Parameters có thực sự bảo mật không?**
> `NoEcho: true` ngăn CloudFormation hiển thị giá trị trên Console và trong Stack Events. Tuy nhiên, giá trị vẫn có thể lộ nếu bạn pass nó vào UserData, Outputs, hoặc ghi vào file plain text. Với secrets thực sự nhạy cảm, nên dùng Parameter Store SecureString hoặc Secrets Manager rồi reference bằng dynamic reference `{{resolve:secretsmanager:...}}`.

**Q: Cross-Stack Reference có hạn chế gì?**
> Không thể xóa stack đang export Output khi vẫn còn stack khác đang import. Phải xóa stack phụ thuộc trước. Ngoài ra, Export Name phải duy nhất trong toàn account + region — không thể có 2 stack export cùng một tên.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
