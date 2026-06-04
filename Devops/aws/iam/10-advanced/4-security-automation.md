# ⚡ Security Automation — Tự Động Hóa Bảo Mật Với EventBridge + Lambda

> **Security Automation** là khả năng phát hiện, phân loại và phản hồi các sự kiện bảo mật **tự động, không cần con người can thiệp** — rút ngắn MTTR (Mean Time to Remediate — Thời Gian Trung Bình Khắc Phục) từ hàng giờ xuống vài phút. Kiến trúc cốt lõi: GuardDuty → Security Hub → EventBridge → Lambda → Auto-remediation.

---

## 📚 Mục Lục

1. [Tại Sao Cần Security Automation](#1-tại-sao-cần-security-automation)
2. [Auto-Remediation Architecture](#2-auto-remediation-architecture)
3. [EventBridge Rules Cho Security Events](#3-eventbridge-rules-cho-security-events)
4. [Lambda Auto-Remediation Examples](#4-lambda-auto-remediation-examples)
5. [AWS Systems Manager Automation](#5-aws-systems-manager-automation)
6. [Step Functions Cho Multi-Step Remediation](#6-step-functions-cho-multi-step-remediation)
7. [Security Hub Custom Actions](#7-security-hub-custom-actions)
8. [AWS Config Auto Remediation](#8-aws-config-auto-remediation)
9. [Runbook Tự Động vs Manual Approval](#9-runbook-tự-động-vs-manual-approval)
10. [Testing Security Automation](#10-testing-security-automation)
11. [Câu Hỏi Phỏng Vấn](#11-câu-hỏi-phỏng-vấn)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. Tại Sao Cần Security Automation

### Phản Ứng Thủ Công vs Tự Động

```
PHẢN ỨNG THỦ CÔNG:
Time 0:00  → GuardDuty phát hiện EC2 bị compromise
Time 0:05  → Email alert đến mailbox security team
Time 0:20  → Security engineer đọc email, triage finding
Time 0:45  → Engineer login AWS console, investigate
Time 1:15  → Engineer quyết định isolate instance
Time 1:20  → Instance được isolate
Time 1:20  → Lateral movement đã xảy ra trong 80 phút!

PHẢN ỨNG TỰ ĐỘNG:
Time 0:00  → GuardDuty phát hiện EC2 bị compromise
Time 0:01  → EventBridge trigger Lambda
Time 0:02  → Lambda attach restrictive Security Group, snapshot disk
Time 0:03  → Notification gửi đến security team
Total: Instance bị isolate trong 3 phút — không có lateral movement
```

### 3 Lợi Ích Chính

| Lợi Ích | Mô Tả | Metric |
|---|---|---|
| **Tốc độ** | Phản ứng trong giây, không phải phút hay giờ | MTTR: từ giờ → phút |
| **Nhất quán** | Cùng procedure áp dụng mọi lần, không có sai sót con người | Error rate ≈ 0% |
| **Scale** | Xử lý 1,000 events/giây không cần thêm người | Linear cost scaling |

### Tại Sao Không Automation Tất Cả?

```
Một số actions cần human approval:

✅ Tự động hóa an toàn:
   - Block IP trong WAF (dễ revert)
   - Isolate EC2 với restrictive SG (không xóa instance)
   - Rotate/disable access key (nếu confirmed compromise)
   - Enable S3 block public access (không ảnh hưởng private access)
   - Create snapshot trước khi investigate

⚠️ Cần manual approval:
   - Terminate production instance
   - Revoke tất cả access của một user
   - Xóa S3 bucket (dù public)
   - Thay đổi production database
   - Modify routing/firewall rules ảnh hưởng production traffic
```

---

## 2. Auto-Remediation Architecture

### Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DETECT LAYER (Phát Hiện)                          │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐   │
│  │  GuardDuty   │  │  AWS Config  │  │  CloudTrail + Athena    │   │
│  │  (threats)   │  │  (compliance)│  │  (custom detection)     │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬──────────────┘   │
│         │                 │                       │                   │
└─────────────────────────────────────────────────────────────────────┘
          │                 │                       │
          ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   AGGREGATE LAYER (Tổng Hợp)                         │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    AWS Security Hub                            │  │
│  │  Normalize findings → Enrich → Prioritize → Route            │  │
│  └────────────────────────────┬──────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────-┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     ROUTE LAYER (Định Tuyến)                         │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                   Amazon EventBridge                           │  │
│  │  Rule 1: GuardDuty HIGH → auto-isolate Lambda                 │  │
│  │  Rule 2: Config NON_COMPLIANT → SSM Automation               │  │
│  │  Rule 3: S3 public → block-public-access Lambda              │  │
│  │  Rule 4: IAM key created → notify + monitor                   │  │
│  └────────────────────────────┬──────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────-┘
                                │
          ┌─────────────────────┼──────────────────────┐
          ▼                     ▼                       ▼
┌──────────────────┐ ┌───────────────────┐ ┌────────────────────────┐
│  Lambda Function │ │  Step Functions   │ │  SSM Automation        │
│  (simple, fast)  │ │  (complex, multi  │ │  (pre-built runbooks)  │
│                  │ │   step)           │ │                        │
└──────────────────┘ └───────────────────┘ └────────────────────────┘
          │                     │                       │
          └─────────────────────┴──────────────────────┘
                                │
          ┌─────────────────────┼──────────────────────┐
          ▼                     ▼                       ▼
┌──────────────────┐ ┌───────────────────┐ ┌────────────────────────┐
│  Isolate EC2     │ │  SNS Notification │ │  JIRA/ServiceNow ticket│
│  Block IP in WAF │ │  (Slack/PagerDuty)│ │  (for manual review)   │
│  Rotate keys     │ │                   │ │                        │
└──────────────────┘ └───────────────────┘ └────────────────────────┘
```

---

## 3. EventBridge Rules Cho Security Events

### Rule Cho GuardDuty Findings

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{"numeric": [">=", 7]}],
    "type": [
      "UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS",
      "Trojan:EC2/BlackholeTraffic",
      "Backdoor:EC2/C&CActivity.B"
    ]
  }
}
```

```bash
# Tạo EventBridge rule với AWS CLI
aws events put-rule \
  --name "HighSeverityGuardDutyFinding" \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [{"numeric": [">=", 7]}]
    }
  }' \
  --state ENABLED \
  --description "Route high severity GuardDuty findings to auto-remediation Lambda"

# Thêm Lambda target
aws events put-targets \
  --rule "HighSeverityGuardDutyFinding" \
  --targets '[{
    "Id": "AutoRemediateLambda",
    "Arn": "arn:aws:lambda:us-east-1:123456789012:function:SecurityAutoRemediate",
    "RoleArn": "arn:aws:iam::123456789012:role/EventBridgeLambdaRole"
  }]'
```

### Rule Cho S3 Public Access Events

```json
{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["s3.amazonaws.com"],
    "eventName": [
      "PutBucketAcl",
      "PutBucketPolicy",
      "DeletePublicAccessBlock"
    ]
  }
}
```

### Rule Cho Config Non-Compliance

```json
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"],
  "detail": {
    "messageType": ["ComplianceChangeNotification"],
    "newEvaluationResult": {
      "complianceType": ["NON_COMPLIANT"]
    },
    "configRuleName": [
      "s3-bucket-public-read-prohibited",
      "ec2-security-group-attached-to-eni-periodic",
      "iam-root-access-key-check"
    ]
  }
}
```

---

## 4. Lambda Auto-Remediation Examples

### 4.1 Revoke Compromised IAM Credentials

```python
"""
Lambda function: revoke-compromised-credentials.py
Trigger: GuardDuty finding về credential exfiltration hoặc compromise
"""
import boto3
import json
import logging
from datetime import datetime, timezone

logger = logging.getLogger()
logger.setLevel(logging.INFO)

iam = boto3.client('iam')
sns = boto3.client('sns')
NOTIFICATION_TOPIC = "arn:aws:sns:us-east-1:123456789012:security-alerts"


def lambda_handler(event, context):
    """
    Xử lý GuardDuty finding liên quan đến credential compromise.
    Actions: Disable access keys, attach DenyAll policy, notify team.
    """
    logger.info(f"Received event: {json.dumps(event)}")

    finding = event.get('detail', {})
    finding_type = finding.get('type', '')
    severity = finding.get('severity', 0)

    # Lấy thông tin user/role bị compromise
    principal = extract_principal(finding)
    if not principal:
        logger.warning("Could not extract principal from finding")
        return {"status": "skipped", "reason": "no_principal"}

    actions_taken = []

    # HIGH severity (>= 7): disable keys + deny all
    if severity >= 7:
        if principal['type'] == 'IAMUser':
            # Disable tất cả access keys của user
            keys_disabled = disable_user_access_keys(principal['name'])
            actions_taken.extend(keys_disabled)

            # Attach emergency DenyAll policy
            policy_attached = attach_deny_all_policy(principal['name'])
            actions_taken.append(policy_attached)

    # MEDIUM severity (4-7): disable keys only
    elif severity >= 4:
        if principal['type'] == 'IAMUser':
            keys_disabled = disable_user_access_keys(principal['name'])
            actions_taken.extend(keys_disabled)

    # Notify security team
    send_notification(finding, principal, actions_taken)

    return {
        "status": "completed",
        "principal": principal,
        "actions_taken": actions_taken,
        "finding_id": finding.get('id', '')
    }


def extract_principal(finding):
    """Trích xuất thông tin principal từ GuardDuty finding."""
    try:
        service = finding.get('service', {})
        action = service.get('action', {})

        # Thử lấy từ userIdentity
        resources = finding.get('resource', {})
        access_key_details = resources.get('accessKeyDetails', {})

        if access_key_details:
            return {
                'type': access_key_details.get('userType', 'IAMUser'),
                'name': access_key_details.get('userName', ''),
                'arn': access_key_details.get('arn', ''),
                'access_key_id': access_key_details.get('accessKeyId', '')
            }
    except Exception as e:
        logger.error(f"Error extracting principal: {e}")

    return None


def disable_user_access_keys(username):
    """Disable tất cả access keys của IAM user."""
    disabled_keys = []
    try:
        paginator = iam.get_paginator('list_access_keys')
        for page in paginator.paginate(UserName=username):
            for key in page['AccessKeyMetadata']:
                if key['Status'] == 'Active':
                    iam.update_access_key(
                        UserName=username,
                        AccessKeyId=key['AccessKeyId'],
                        Status='Inactive'
                    )
                    logger.info(f"Disabled key {key['AccessKeyId']} for {username}")
                    disabled_keys.append(f"disabled_key:{key['AccessKeyId']}")
    except Exception as e:
        logger.error(f"Error disabling keys for {username}: {e}")
        disabled_keys.append(f"error_disabling_keys:{str(e)}")

    return disabled_keys


def attach_deny_all_policy(username):
    """Attach inline DenyAll policy để block mọi actions."""
    deny_all_policy = {
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Deny",
            "Action": "*",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:RequestedRegion": ["*"]  # Block everywhere
                }
            }
        }]
    }

    try:
        iam.put_user_policy(
            UserName=username,
            PolicyName="EmergencyDenyAll",
            PolicyDocument=json.dumps(deny_all_policy)
        )
        logger.info(f"Attached DenyAll policy to {username}")
        return "attached_deny_all_policy"
    except Exception as e:
        logger.error(f"Error attaching DenyAll policy: {e}")
        return f"error_attaching_policy:{str(e)}"


def send_notification(finding, principal, actions_taken):
    """Gửi notification đến security team qua SNS."""
    message = {
        "alert_type": "CREDENTIAL_COMPROMISE_AUTO_REMEDIATION",
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "finding_id": finding.get('id', ''),
        "finding_type": finding.get('type', ''),
        "severity": finding.get('severity', 0),
        "affected_principal": principal,
        "actions_taken": actions_taken,
        "next_steps": [
            "1. Review GuardDuty finding for details",
            "2. Check CloudTrail for actions taken with compromised credentials",
            "3. Rotate all secrets the user had access to",
            "4. Review if attacker created new users/keys",
            "5. Re-enable user access after confirming security"
        ]
    }

    sns.publish(
        TopicArn=NOTIFICATION_TOPIC,
        Subject=f"[HIGH] Credential Compromise Auto-Remediated: {principal.get('name', 'Unknown')}",
        Message=json.dumps(message, indent=2)
    )
```

### 4.2 Isolate Compromised EC2 Instance

```python
"""
Lambda function: isolate-compromised-ec2.py
Trigger: GuardDuty EC2 findings (malware, bitcoin mining, C2 activity)
"""
import boto3
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

ec2 = boto3.client('ec2')
sns = boto3.client('sns')
NOTIFICATION_TOPIC = "arn:aws:sns:us-east-1:123456789012:security-alerts"
FORENSICS_BUCKET = "company-forensics-bucket"


def lambda_handler(event, context):
    finding = event.get('detail', {})
    finding_type = finding.get('type', '')

    # Chỉ xử lý EC2 threats
    if 'EC2' not in finding_type and 'EKS' not in finding_type:
        return {"status": "skipped", "reason": "not_ec2_finding"}

    # Lấy instance ID từ finding
    instance_id = extract_instance_id(finding)
    if not instance_id:
        logger.warning("Could not find instance ID in finding")
        return {"status": "error", "reason": "no_instance_id"}

    region = finding.get('region', 'us-east-1')
    actions = []

    # BƯỚC 1: Snapshot disk cho forensics trước khi isolate
    snapshot_id = create_forensic_snapshot(instance_id, finding)
    if snapshot_id:
        actions.append(f"snapshot_created:{snapshot_id}")

    # BƯỚC 2: Isolate bằng cách thay Security Group
    isolation_sg_id = get_or_create_isolation_sg(region)
    if isolate_instance(instance_id, isolation_sg_id):
        actions.append(f"isolated_with_sg:{isolation_sg_id}")

    # BƯỚC 3: Disable termination protection (giữ để forensics)
    try:
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            DisableApiTermination={'Value': True}
        )
        actions.append("termination_protection_enabled")
    except Exception as e:
        logger.error(f"Error enabling termination protection: {e}")

    # BƯỚC 4: Tag instance
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[
            {'Key': 'SecurityStatus', 'Value': 'QUARANTINED'},
            {'Key': 'QuarantineReason', 'Value': finding_type},
            {'Key': 'QuarantineTime', 'Value': finding.get('updatedAt', '')}
        ]
    )
    actions.append("instance_tagged_quarantined")

    # BƯỚC 5: Notify security team
    send_notification(finding, instance_id, actions)

    return {
        "status": "completed",
        "instance_id": instance_id,
        "actions_taken": actions
    }


def extract_instance_id(finding):
    """Trích xuất instance ID từ GuardDuty EC2 finding."""
    try:
        resources = finding.get('resource', {})
        instance_details = resources.get('instanceDetails', {})
        return instance_details.get('instanceId')
    except Exception:
        return None


def get_or_create_isolation_sg(region):
    """
    Lấy hoặc tạo Security Group cho forensic isolation.
    SG này chặn ALL ingress và egress — giữ instance trên mạng
    nhưng không thể communicate với gì cả.
    """
    sg_name = "ForensicIsolationSG"

    try:
        # Tìm SG hiện có
        response = ec2.describe_security_groups(
            Filters=[{'Name': 'group-name', 'Values': [sg_name]}]
        )
        if response['SecurityGroups']:
            return response['SecurityGroups'][0]['GroupId']
    except Exception:
        pass

    # Lấy default VPC
    vpcs = ec2.describe_vpcs(Filters=[{'Name': 'isDefault', 'Values': ['true']}])
    vpc_id = vpcs['Vpcs'][0]['VpcId'] if vpcs['Vpcs'] else None

    # Tạo isolation SG
    sg = ec2.create_security_group(
        GroupName=sg_name,
        Description="Emergency isolation SG - blocks all traffic",
        VpcId=vpc_id
    )
    sg_id = sg['GroupId']

    # Xóa default egress rule (allow all)
    try:
        ec2.revoke_security_group_egress(
            GroupId=sg_id,
            IpPermissions=[{
                'IpProtocol': '-1',
                'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
            }]
        )
    except Exception:
        pass

    # Thêm rule cho phép SSM (để forensics) từ VPC Endpoint
    ec2.authorize_security_group_ingress(
        GroupId=sg_id,
        IpPermissions=[{
            'IpProtocol': 'tcp',
            'FromPort': 443,
            'ToPort': 443,
            'IpRanges': [{'CidrIp': '10.0.0.0/8',
                          'Description': 'Allow SSM from internal only'}]
        }]
    )

    logger.info(f"Created isolation SG: {sg_id}")
    return sg_id


def isolate_instance(instance_id, isolation_sg_id):
    """Thay tất cả SGs của instance bằng isolation SG."""
    try:
        # Lấy network interfaces của instance
        instance = ec2.describe_instances(InstanceIds=[instance_id])
        interfaces = instance['Reservations'][0]['Instances'][0].get(
            'NetworkInterfaces', [])

        for interface in interfaces:
            ec2.modify_network_interface_attribute(
                NetworkInterfaceId=interface['NetworkInterfaceId'],
                Groups=[isolation_sg_id]
            )
            logger.info(f"Replaced SGs on {interface['NetworkInterfaceId']}")

        return True
    except Exception as e:
        logger.error(f"Error isolating instance {instance_id}: {e}")
        return False


def create_forensic_snapshot(instance_id, finding):
    """Tạo snapshot của tất cả volumes để forensics."""
    try:
        instance = ec2.describe_instances(InstanceIds=[instance_id])
        volumes = []
        for reservation in instance['Reservations']:
            for inst in reservation['Instances']:
                for bdm in inst.get('BlockDeviceMappings', []):
                    volumes.append(bdm['Ebs']['VolumeId'])

        snapshot_ids = []
        for volume_id in volumes:
            snapshot = ec2.create_snapshot(
                VolumeId=volume_id,
                Description=f"Forensic snapshot - GuardDuty finding {finding.get('id', '')}",
                TagSpecifications=[{
                    'ResourceType': 'snapshot',
                    'Tags': [
                        {'Key': 'Purpose', 'Value': 'ForensicEvidence'},
                        {'Key': 'SourceInstance', 'Value': instance_id},
                        {'Key': 'FindingType', 'Value': finding.get('type', '')}
                    ]
                }]
            )
            snapshot_ids.append(snapshot['SnapshotId'])

        return ','.join(snapshot_ids)
    except Exception as e:
        logger.error(f"Error creating forensic snapshot: {e}")
        return None


def send_notification(finding, instance_id, actions):
    sns.publish(
        TopicArn=NOTIFICATION_TOPIC,
        Subject=f"[CRITICAL] EC2 Instance Quarantined: {instance_id}",
        Message=json.dumps({
            "alert_type": "EC2_QUARANTINE",
            "instance_id": instance_id,
            "finding_type": finding.get('type', ''),
            "severity": finding.get('severity', 0),
            "actions_taken": actions,
            "next_steps": [
                "Review GuardDuty finding details",
                "Analyze forensic snapshots",
                "Check VPC Flow Logs for lateral movement",
                "Check if attacker pivoted to other instances",
                "Review CloudTrail for instance actions",
                "After investigation: terminate or restore instance"
            ]
        }, indent=2)
    )
```

### 4.3 Block IP Address Trong WAF

```python
"""
Lambda function: block-ip-in-waf.py
Trigger: GuardDuty finding với malicious IP hoặc manual trigger
"""
import boto3
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

wafv2 = boto3.client('wafv2')
IP_SET_ID = "ip-set-id-xxx"   # Pre-created WAF IP Set
IP_SET_NAME = "BlockedMaliciousIPs"
IP_SET_SCOPE = "REGIONAL"  # hoặc "CLOUDFRONT"


def lambda_handler(event, context):
    finding = event.get('detail', {})

    # Lấy IP từ GuardDuty finding
    malicious_ips = extract_malicious_ips(finding)
    if not malicious_ips:
        return {"status": "skipped", "reason": "no_malicious_ips"}

    # Lấy IP set hiện có
    ip_set = wafv2.get_ip_set(
        Name=IP_SET_NAME,
        Scope=IP_SET_SCOPE,
        Id=IP_SET_ID
    )

    current_addresses = set(ip_set['IPSet']['Addresses'])
    lock_token = ip_set['LockToken']

    # Thêm IPs mới (convert sang CIDR notation)
    new_addresses = set(f"{ip}/32" for ip in malicious_ips)
    updated_addresses = list(current_addresses | new_addresses)

    # Cập nhật IP Set
    wafv2.update_ip_set(
        Name=IP_SET_NAME,
        Scope=IP_SET_SCOPE,
        Id=IP_SET_ID,
        Addresses=updated_addresses,
        LockToken=lock_token
    )

    logger.info(f"Blocked IPs: {malicious_ips}")
    return {
        "status": "completed",
        "blocked_ips": list(malicious_ips),
        "total_blocked": len(updated_addresses)
    }


def extract_malicious_ips(finding):
    """Trích xuất IP độc hại từ GuardDuty finding."""
    ips = set()
    try:
        service = finding.get('service', {})
        action = service.get('action', {})

        # Từ network connection
        network_action = action.get('networkConnectionAction', {})
        remote_ip = network_action.get('remoteIpDetails', {}).get('ipAddressV4')
        if remote_ip:
            ips.add(remote_ip)

        # Từ port probe
        port_probe = action.get('portProbeAction', {})
        for detail in port_probe.get('portProbeDetails', []):
            ip = detail.get('remoteIpDetails', {}).get('ipAddressV4')
            if ip:
                ips.add(ip)

    except Exception as e:
        logger.error(f"Error extracting IPs: {e}")

    return ips
```

### 4.4 Restrict S3 Bucket Public Access Tự Động

```python
"""
Lambda function: enforce-s3-block-public-access.py
Trigger: EventBridge rule khi S3 bucket ACL/policy thay đổi
         hoặc Config rule NON_COMPLIANT cho s3-bucket-public-read-prohibited
"""
import boto3
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

s3 = boto3.client('s3')
s3control = boto3.client('s3control')
sts = boto3.client('sts')
sns = boto3.client('sns')
NOTIFICATION_TOPIC = "arn:aws:sns:us-east-1:123456789012:security-alerts"


def lambda_handler(event, context):
    # Xác định nguồn event
    source = event.get('source', '')

    if source == 'aws.config':
        bucket_name = extract_bucket_from_config(event)
        action_reason = "Config_NonCompliant"
    elif source == 'aws.s3':
        bucket_name = extract_bucket_from_cloudtrail(event)
        action_reason = "S3_ACL_Change"
    else:
        # Manual trigger hoặc test
        bucket_name = event.get('bucket_name')
        action_reason = "Manual"

    if not bucket_name:
        return {"status": "error", "reason": "no_bucket_name"}

    actions_taken = []

    # BƯỚC 1: Enable Block Public Access
    try:
        s3.put_public_access_block(
            Bucket=bucket_name,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        actions_taken.append("block_public_access_enabled")
        logger.info(f"Blocked public access for bucket: {bucket_name}")
    except Exception as e:
        logger.error(f"Error blocking public access: {e}")
        actions_taken.append(f"error: {str(e)}")

    # BƯỚC 2: Check và xóa public ACLs nếu có
    try:
        acl = s3.get_bucket_acl(Bucket=bucket_name)
        public_grants = [
            g for g in acl['Grants']
            if 'AllUsers' in g.get('Grantee', {}).get('URI', '')
            or 'AuthenticatedUsers' in g.get('Grantee', {}).get('URI', '')
        ]

        if public_grants:
            # Reset về private ACL
            s3.put_bucket_acl(Bucket=bucket_name, ACL='private')
            actions_taken.append(f"removed_public_acls:{len(public_grants)}_grants")
    except Exception as e:
        logger.error(f"Error checking/removing ACLs: {e}")

    # BƯỚC 3: Notify
    account_id = sts.get_caller_identity()['Account']
    sns.publish(
        TopicArn=NOTIFICATION_TOPIC,
        Subject=f"[AUTO-REMEDIATED] S3 Public Access Blocked: {bucket_name}",
        Message=json.dumps({
            "alert_type": "S3_PUBLIC_ACCESS_AUTO_REMEDIATED",
            "bucket": bucket_name,
            "account": account_id,
            "reason": action_reason,
            "actions_taken": actions_taken,
            "recommendation": "Review if this bucket should be public. "
                              "If legitimate, update bucket policy to use "
                              "presigned URLs or CloudFront instead."
        }, indent=2)
    )

    return {
        "status": "completed",
        "bucket": bucket_name,
        "actions_taken": actions_taken
    }


def extract_bucket_from_config(event):
    """Lấy bucket name từ Config compliance event."""
    try:
        return event['detail']['resourceId']
    except Exception:
        return None


def extract_bucket_from_cloudtrail(event):
    """Lấy bucket name từ CloudTrail S3 event."""
    try:
        return event['detail']['requestParameters']['bucketName']
    except Exception:
        return None
```

---

## 5. AWS Systems Manager Automation

SSM Automation cung cấp pre-built runbooks (sổ tay quy trình) cho các tác vụ remediation phổ biến.

### Sử Dụng AWS-Managed Runbooks

```bash
# Xem danh sách automation documents có sẵn cho security
aws ssm list-documents \
  --filter '{"Key":"Owner","Values":["Amazon"]}' \
  --query 'DocumentIdentifiers[?contains(Name,`Security`) || contains(Name, `EC2`)].[Name,DocumentVersion]' \
  --output table

# Chạy runbook tắt EC2 instance
aws ssm start-automation-execution \
  --document-name "AWS-StopEC2Instance" \
  --parameters "InstanceId=i-1234567890abcdef0"

# Kiểm tra execution status
aws ssm get-automation-execution \
  --automation-execution-id "xxx" \
  --query 'AutomationExecution.[Status,StepExecutions[*].[StepName,StepStatus]]'
```

### Custom SSM Automation Document

```yaml
# custom-isolate-ec2.yaml
schemaVersion: "0.3"
description: "Isolate compromised EC2 instance for forensics"
parameters:
  InstanceId:
    type: String
    description: "EC2 Instance ID to isolate"
  GuardDutyFindingId:
    type: String
    description: "Associated GuardDuty finding ID"
    default: "manual"

mainSteps:
  - name: CreateForensicSnapshot
    action: aws:executeScript
    inputs:
      Runtime: python3.8
      Handler: script_handler
      Script: |
        import boto3

        def script_handler(events, context):
            ec2 = boto3.client('ec2')
            instance_id = events['InstanceId']

            instance = ec2.describe_instances(InstanceIds=[instance_id])
            volumes = [
                bdm['Ebs']['VolumeId']
                for r in instance['Reservations']
                for i in r['Instances']
                for bdm in i.get('BlockDeviceMappings', [])
            ]

            snapshot_ids = []
            for vol_id in volumes:
                snap = ec2.create_snapshot(
                    VolumeId=vol_id,
                    Description=f"Forensic - {instance_id}"
                )
                snapshot_ids.append(snap['SnapshotId'])

            return {"SnapshotIds": snapshot_ids}
      InputPayload:
        InstanceId: "{{ InstanceId }}"

  - name: ApplyIsolationTag
    action: aws:createTags
    inputs:
      ResourceType: EC2
      ResourceIds:
        - "{{ InstanceId }}"
      Tags:
        - Key: SecurityStatus
          Value: QUARANTINED
        - Key: FindingId
          Value: "{{ GuardDutyFindingId }}"

  - name: NotifySecurityTeam
    action: aws:executeAwsApi
    inputs:
      Service: sns
      Api: Publish
      TopicArn: "arn:aws:sns:us-east-1:123456789012:security-alerts"
      Subject: "EC2 Instance Quarantined via SSM Automation"
      Message: "Instance {{ InstanceId }} has been isolated. Snapshots: {{ CreateForensicSnapshot.SnapshotIds }}"
```

---

## 6. Step Functions Cho Multi-Step Remediation

Khi remediation phức tạp (cần parallel steps, retry logic, approval gate), dùng Step Functions.

### State Machine Định Nghĩa (JSON)

```json
{
  "Comment": "Multi-step security incident response workflow",
  "StartAt": "TriageFinding",
  "States": {
    "TriageFinding": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:TriageSecurityFinding",
      "Next": "SeverityCheck",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "NotifyOnError"
      }]
    },

    "SeverityCheck": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.severity",
          "NumericGreaterThanEquals": 8,
          "Next": "ParallelCriticalResponse"
        },
        {
          "Variable": "$.severity",
          "NumericGreaterThanEquals": 4,
          "Next": "AutoRemediate"
        }
      ],
      "Default": "LogAndMonitor"
    },

    "ParallelCriticalResponse": {
      "Type": "Parallel",
      "Next": "WaitForApproval",
      "Branches": [
        {
          "StartAt": "IsolateInstance",
          "States": {
            "IsolateInstance": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:IsolateEC2",
              "End": true
            }
          }
        },
        {
          "StartAt": "RevokeCredentials",
          "States": {
            "RevokeCredentials": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:RevokeCredentials",
              "End": true
            }
          }
        },
        {
          "StartAt": "CreateForensicSnapshot",
          "States": {
            "CreateForensicSnapshot": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ForensicSnapshot",
              "End": true
            }
          }
        }
      ]
    },

    "WaitForApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "SendApprovalRequest",
        "Payload": {
          "taskToken.$": "$$.Task.Token",
          "incident.$": "$"
        }
      },
      "TimeoutSeconds": 3600,
      "HeartbeatSeconds": 300,
      "Next": "FinalRemediation",
      "Catch": [{
        "ErrorEquals": ["States.TaskTimedOut"],
        "Next": "EscalateToOnCall"
      }]
    },

    "AutoRemediate": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:AutoRemediate",
      "Retry": [{
        "ErrorEquals": ["Lambda.ServiceException"],
        "IntervalSeconds": 5,
        "MaxAttempts": 3,
        "BackoffRate": 2
      }],
      "Next": "LogAndMonitor"
    },

    "FinalRemediation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:FinalRemediation",
      "Next": "CloseIncident"
    },

    "CloseIncident": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:CloseIncident",
      "End": true
    },

    "EscalateToOnCall": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:EscalateIncident",
      "End": true
    },

    "LogAndMonitor": {
      "Type": "Pass",
      "End": true
    },

    "NotifyOnError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:NotifyError",
      "End": true
    }
  }
}
```

---

## 7. Security Hub Custom Actions

Security Hub Custom Actions cho phép analyst trigger remediation thủ công từ Security Hub console.

```bash
# Tạo Custom Action trong Security Hub
aws securityhub create-action-target \
  --name "IsolateEC2" \
  --description "Trigger EC2 isolation workflow" \
  --id "IsolateEC2"

# Custom Action tạo ra EventBridge event khi analyst click
# Event pattern:
# {
#   "source": ["aws.securityhub"],
#   "detail-type": ["Security Hub Findings - Custom Action"],
#   "detail": {
#     "actionName": ["IsolateEC2"]
#   }
# }

# Lambda xử lý Custom Action event
```

```python
def handle_custom_action(event, context):
    """Xử lý Security Hub Custom Action."""
    findings = event['detail']['findings']

    for finding in findings:
        # Lấy resource từ finding
        for resource in finding.get('Resources', []):
            if resource['Type'] == 'AwsEc2Instance':
                instance_id = resource['Id'].split('/')[-1]
                # Trigger isolation workflow
                sfn = boto3.client('stepfunctions')
                sfn.start_execution(
                    stateMachineArn="arn:aws:states:...:IsolationWorkflow",
                    input=json.dumps({
                        "instance_id": instance_id,
                        "finding_id": finding['Id'],
                        "severity": finding['Severity']['Label']
                    })
                )
```

---

## 8. AWS Config Auto Remediation

```bash
# Tạo Config rule với auto-remediation
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
  }'

# Tạo remediation configuration
aws configservice put-remediation-configurations \
  --remediation-configurations '[{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "TargetType": "SSM_DOCUMENT",
    "TargetId": "AWS-DisableS3BucketPublicReadWrite",
    "Parameters": {
      "AutomationAssumeRole": {
        "StaticValue": {
          "Values": ["arn:aws:iam::123456789012:role/ConfigRemediationRole"]
        }
      },
      "S3BucketName": {
        "ResourceValue": {
          "Value": "RESOURCE_ID"
        }
      }
    },
    "Automatic": true,
    "MaximumAutomaticAttempts": 3,
    "RetryAttemptSeconds": 60
  }]'
```

---

## 9. Runbook Tự Động vs Manual Approval

### Decision Framework

```
Khi nào tự động hoàn toàn?
  ├── Tác động thấp (dễ revert)
  ├── False positive rate thấp
  ├── Không ảnh hưởng production availability
  └── Examples: Block IP, Enable S3 Block Public Access, Tag resource

Khi nào cần manual approval (human-in-the-loop)?
  ├── Tác động cao (khó revert, ảnh hưởng users)
  ├── False positive rate cao
  ├── Terminate/delete resources
  └── Examples: Terminate production EC2, Revoke all user access, Delete data

Hybrid Approach:
  Step 1: Tự động (ngay lập tức) → Mitigate immediate risk
    └── Isolate instance (không terminate)
    └── Disable key (không xóa)
    └── Block IP (dễ unblock)

  Step 2: Notify humans + Wait
    └── Send alert với context và recommended actions
    └── Wait for approval (timeout → escalate)

  Step 3: Human approves → Final remediation
    └── Terminate instance
    └── Revoke all user sessions
    └── Update permanent firewall rules
```

### Approval Workflow Với SNS + Lambda

```python
def send_approval_request(task_token, incident):
    """Gửi approval request với actionable buttons."""
    approve_url = f"https://api-gateway-url/approve?token={task_token}&action=approve"
    deny_url = f"https://api-gateway-url/approve?token={task_token}&action=deny"

    message = f"""
    🚨 SECURITY INCIDENT REQUIRES APPROVAL

    Incident: {incident['finding_type']}
    Severity: {incident['severity']}
    Resource: {incident['resource_id']}
    Time: {incident['timestamp']}

    Automated actions already taken:
    {chr(10).join(incident['actions_taken'])}

    Recommended next action: {incident['recommended_action']}

    Please review and approve/deny within 1 hour:
    ✅ APPROVE: {approve_url}
    ❌ DENY: {deny_url}

    If no action taken in 1 hour, escalation to on-call team.
    """

    sns.publish(
        TopicArn=NOTIFICATION_TOPIC,
        Subject=f"[ACTION REQUIRED] Security Approval: {incident['resource_id']}",
        Message=message
    )
```

---

## 10. Testing Security Automation

### Unit Testing Lambda Functions

```python
# test_isolate_ec2.py
import pytest
import json
from unittest.mock import patch, MagicMock
from lambda_function import lambda_handler, isolate_instance


@pytest.fixture
def guardduty_event():
    return {
        "detail": {
            "type": "Backdoor:EC2/C&CActivity.B",
            "severity": 8,
            "region": "us-east-1",
            "resource": {
                "instanceDetails": {
                    "instanceId": "i-1234567890abcdef0"
                }
            }
        }
    }


@patch('lambda_function.ec2')
@patch('lambda_function.sns')
def test_isolate_compromised_instance(mock_sns, mock_ec2, guardduty_event):
    """Test rằng instance bị isolate khi severity cao."""
    # Mock EC2 responses
    mock_ec2.describe_instances.return_value = {
        'Reservations': [{
            'Instances': [{
                'NetworkInterfaces': [{
                    'NetworkInterfaceId': 'eni-xxx'
                }],
                'BlockDeviceMappings': [{
                    'Ebs': {'VolumeId': 'vol-xxx'}
                }]
            }]
        }]
    }
    mock_ec2.create_snapshot.return_value = {'SnapshotId': 'snap-xxx'}
    mock_ec2.describe_security_groups.return_value = {
        'SecurityGroups': [{'GroupId': 'sg-isolation'}]
    }

    result = lambda_handler(guardduty_event, None)

    assert result['status'] == 'completed'
    assert 'isolated_with_sg' in str(result['actions_taken'])
    mock_ec2.modify_network_interface_attribute.assert_called_once()


@patch('lambda_function.ec2')
def test_skips_non_ec2_findings(mock_ec2):
    """Test rằng non-EC2 findings bị bỏ qua."""
    event = {
        "detail": {
            "type": "UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B",
            "severity": 5
        }
    }
    result = lambda_handler(event, None)
    assert result['status'] == 'skipped'
    mock_ec2.describe_instances.assert_not_called()
```

### Integration Testing Với Real Events

```bash
# Tạo fake GuardDuty finding để test automation pipeline
aws guardduty create-sample-findings \
  --detector-id $(aws guardduty list-detectors --query 'DetectorIds[0]' --output text) \
  --finding-types "Backdoor:EC2/C&CActivity.B"

# Kiểm tra EventBridge rule đã triggered
aws events list-event-buses

# Kiểm tra Lambda invocation
aws logs filter-log-events \
  --log-group-name "/aws/lambda/SecurityAutoRemediate" \
  --start-time $(date -d "5 minutes ago" +%s000) \
  --filter-pattern "REPORT"

# Kiểm tra Step Functions execution
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:... \
  --status-filter RUNNING
```

---

## 11. Câu Hỏi Phỏng Vấn

**Q1: Kiến trúc security automation điển hình trên AWS là gì?**

> GuardDuty phát hiện threat → findings gửi đến Security Hub để normalize và enrich → EventBridge rules định tuyến theo loại/severity → Lambda thực hiện automated remediation (isolate EC2, disable IAM key, block IP in WAF) → SNS notification đến team → Step Functions cho multi-step workflows cần approval. AWS Config Auto Remediation cho compliance issues.

**Q2: Làm thế nào tránh "automation runaway" — automation gây ra outage?**

> (1) Dry-run mode trước khi enable tự động; (2) Chỉ tự động hóa low-risk actions (isolate, không terminate; disable key, không xóa); (3) Human-in-the-loop cho high-impact actions; (4) Rate limiting và circuit breaker trong Lambda; (5) Canary testing với sample findings; (6) Clear rollback procedure cho mọi automated action; (7) Extensive logging và monitoring cho automation itself.

**Q3: Sự khác biệt khi dùng Lambda vs Step Functions vs SSM Automation?**

> Lambda: đơn giản, nhanh, cho single-action remediation (< 15 phút). Step Functions: phức tạp, multi-step với conditional logic, parallel execution, human approval gates — phù hợp incident response workflow. SSM Automation: pre-built runbooks cho AWS resources, có audit trail tốt, phù hợp operations tasks như patch, backup.

**Q4: Làm thế nào test security automation mà không affect production?**

> (1) Unit test với mocked AWS SDK; (2) Test trong separate sandbox account; (3) Dùng GuardDuty sample findings để trigger automation pipeline; (4) EventBridge Sandbox mode để test rules; (5) Tag test resources để automation có logic "if test_resource: log only, don't remediate"; (6) Chaos engineering: chủ động inject security events để test MTTR.

**Q5: Làm thế nào xử lý false positives trong security automation?**

> (1) Whitelist/suppress rules: tag resources với `SecurityException=true` để skip; (2) Severity threshold: chỉ auto-remediate HIGH/CRITICAL, MEDIUM cần manual; (3) Context enrichment trước khi remediate: kiểm tra finding trong lịch sử 7 ngày — nếu pattern bình thường thì skip; (4) Human-in-the-loop cho mọi terminate/delete actions; (5) Feedback loop: track false positives, tune rules định kỳ.

---

## 12. Key Takeaways

> **MTTR là metric quan trọng nhất** — mỗi phút delay = thêm cơ hội lateral movement. Security automation giúp đi từ "30-60 phút" xuống "dưới 3 phút" cho common incident types.

> **"Automate to contain, not to eliminate"** — automation nên isolate/contain threat ngay lập tức, nhưng quyết định terminate/xóa cần human review để tránh false positive gây outage.

> **EventBridge là routing engine** — tách biệt detection (GuardDuty, Config) với remediation (Lambda, Step Functions). Dễ thêm/sửa rules mà không cần sửa detection hay remediation code.

> **Step Functions cho complex workflows** — khi remediation cần parallel execution, retry logic, conditional branching, human approval, và audit trail, Step Functions là lựa chọn đúng hơn Lambda đơn thuần.

> **Testing là bắt buộc** — automation sai có thể gây outage production. Unit test + integration test với sample findings + regular chaos engineering drills là cần thiết trước khi enable auto-remediation.
