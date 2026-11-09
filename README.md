
<div align="center">

# AWS VPC, CloudWatch & Billing

**A hands-on guide to building custom VPCs, peering them, monitoring costs with CloudWatch billing alarms, and shipping EC2 logs to CloudWatch, all with the AWS CLI.**

![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?logo=amazonaws&logoColor=white)
![CLI](https://img.shields.io/badge/AWS%20CLI-v2-232F3E?logo=amazonaws&logoColor=white)
![VPC](https://img.shields.io/badge/VPC-Networking-8C4FFF)
![CloudWatch](https://img.shields.io/badge/CloudWatch-Monitoring-E7157B)
![License](https://img.shields.io/badge/License-MIT-green)

</div>

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [1. Create a Custom VPC](#1-create-a-custom-vpc)
- [2. VPC Peering](#2-vpc-peering)
- [3. Billing Alerts with CloudWatch and SNS](#3-billing-alerts-with-cloudwatch-and-sns)
- [4. Launch EC2 in the Custom VPC](#4-launch-ec2-in-the-custom-vpc)
- [5. Send Logs to CloudWatch](#5-send-logs-to-cloudwatch)
- [Cleanup](#cleanup)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Quick Reference](#quick-reference)
- [Resources](#resources)
- [License](#license)

---

## Overview

This repository documents a practical, client-style workflow on AWS:

| Topic | What you will do |
|---|---|
| **VPC** | Create an isolated network with public and private subnets, route tables and an Internet Gateway |
| **VPC Peering** | Connect two VPCs privately using private IPs |
| **Billing** | Understand pay-as-you-go pricing and set up cost alerts |
| **CloudWatch + SNS** | Trigger an automatic email when estimated charges cross a threshold |
| **EC2 + Logging** | Run instances in a custom VPC and send logs and metrics to CloudWatch |

---

## Architecture

```
                         AWS Region
 ┌──────────────────────────────────────────────────────────┐
 │  VPC A  10.0.0.0/16                 VPC B  10.1.0.0/16   │
 │ ┌──────────────────────────┐       ┌───────────────────┐ │
 │ │ Public Subnet 10.0.1.0/24│       │ Subnet 10.1.1.0/24│ │
 │ │   EC2 (web)  ──► IGW ──► Internet│   EC2 (app)       │ │
 │ ├──────────────────────────┤       └─────────▲─────────┘ │
 │ │ Private Subnet 10.0.2.0/24│                 │           │
 │ │   EC2 (db)               │◄────── VPC Peering ─────────┘ │
 │ └──────────────────────────┘                               │
 │                                                            │
 │   EC2 ──► CloudWatch Agent ──► Log Groups / Metrics        │
 │   Billing ──► CloudWatch Alarm ──► SNS Topic ──► Email     │
 └──────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- An AWS account with an IAM user or role (avoid using root credentials)
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed
- Basic knowledge of networking (CIDR, subnets, routing)

**Install AWS CLI (Linux):**

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

**Configure credentials:**

```bash
aws configure
# AWS Access Key ID     : <your-access-key>
# AWS Secret Access Key : <your-secret-key>
# Default region name   : us-east-1
# Default output format : json
```

**Verify identity:**

```bash
aws sts get-caller-identity
```

---

## 1. Create a Custom VPC

### 1.1 VPC and subnets

```bash
# VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=custom-vpc}]' \
  --query 'Vpc.VpcId' --output text)

# Public subnet
PUB_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet}]' \
  --query 'Subnet.SubnetId' --output text)

# Private subnet
PRIV_SUBNET=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet}]' \
  --query 'Subnet.SubnetId' --output text)

# Auto-assign public IPs in the public subnet
aws ec2 modify-subnet-attribute --subnet-id $PUB_SUBNET --map-public-ip-on-launch
```

### 1.2 Internet Gateway and routing

```bash
# Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Public route table with a default route to the IGW
RTB_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_ID \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $PUB_SUBNET
```

### 1.3 Security group

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name web-sg --description "SSH and HTTP" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Restrict SSH to your own IP in real environments
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 22 --cidr <YOUR_IP>/32
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

> **Security Group vs NACL**
>
> | | Security Group | Network ACL |
> |---|---|---|
> | Level | Instance | Subnet |
> | State | Stateful | Stateless |
> | Rules | Allow only | Allow and deny |

---

## 2. VPC Peering

Peering creates a private connection between two VPCs. Traffic uses private IPs and stays on the AWS network.

```bash
# Create a second VPC (CIDR must NOT overlap with the first)
VPC_B=$(aws ec2 create-vpc --cidr-block 10.1.0.0/16 \
  --query 'Vpc.VpcId' --output text)

# Request peering
PCX_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_ID --peer-vpc-id $VPC_B \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# Accept the request
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PCX_ID

# Update route tables on BOTH sides
aws ec2 create-route --route-table-id $RTB_ID \
  --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PCX_ID

aws ec2 create-route --route-table-id <RTB_OF_VPC_B> \
  --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id $PCX_ID
```

**Rules to remember**

- CIDR blocks must **not overlap**
- Peering is **not transitive** (A-B and B-C does not give A-C)
- Route tables **and** security groups must allow the traffic

---

## 3. Billing Alerts with CloudWatch and SNS

AWS is **pay-as-you-go**. Useful tools: *Billing Dashboard*, *Cost Explorer*, *Budgets* and *Free Tier tracking*.

> Billing metrics are only available in **us-east-1 (N. Virginia)**.

### 3.1 Enable billing alerts

In the console: **Billing and Cost Management → Billing preferences → Receive CloudWatch Billing Alerts**.

### 3.2 Create the SNS topic and email subscription

```bash
TOPIC_ARN=$(aws sns create-topic --name billing-alerts \
  --region us-east-1 --query 'TopicArn' --output text)

aws sns subscribe --topic-arn $TOPIC_ARN \
  --protocol email --notification-endpoint you@example.com \
  --region us-east-1
```

Open your inbox and **confirm the subscription**, otherwise no emails will arrive.

### 3.3 Create the billing alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name billing-over-10-usd \
  --alarm-description "Alert when estimated charges exceed 10 USD" \
  --namespace AWS/Billing \
  --metric-name EstimatedCharges \
  --dimensions Name=Currency,Value=USD \
  --statistic Maximum \
  --period 21600 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions $TOPIC_ARN \
  --region us-east-1
```

Check the alarm:

```bash
aws cloudwatch describe-alarms --alarm-names billing-over-10-usd --region us-east-1
```

---

## 4. Launch EC2 in the Custom VPC

```bash
aws ec2 run-instances \
  --image-id <AMI_ID> \
  --instance-type t2.micro \
  --subnet-id $PUB_SUBNET \
  --security-group-ids $SG_ID \
  --key-name <KEY_PAIR_NAME> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=custom-vpc-ec2}]'
```

- **Public subnet:** reachable directly (public IP + IGW route)
- **Private subnet:** reach it through a bastion host, or give it outbound access via a NAT Gateway

Useful commands:

```bash
aws ec2 describe-instances --output table
aws ec2 stop-instances --instance-ids <INSTANCE_ID>
aws ec2 start-instances --instance-ids <INSTANCE_ID>
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
```

---

## 5. Send Logs to CloudWatch

### 5.1 IAM role for the instance

```bash
cat > trust.json << 'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
JSON

aws iam create-role --role-name EC2-CloudWatch-Role \
  --assume-role-policy-document file://trust.json

aws iam attach-role-policy --role-name EC2-CloudWatch-Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam create-instance-profile --instance-profile-name EC2-CloudWatch-Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-CloudWatch-Profile --role-name EC2-CloudWatch-Role

aws ec2 associate-iam-instance-profile --instance-id <INSTANCE_ID> \
  --iam-instance-profile Name=EC2-CloudWatch-Profile
```

### 5.2 Install and start the CloudWatch Agent (on the instance)

```bash
# Amazon Linux 2023
sudo dnf install -y amazon-cloudwatch-agent

# Generate a config interactively
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start the agent with that config
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

### 5.3 Manage log groups

```bash
aws logs create-log-group --log-group-name /my-app
aws logs put-retention-policy --log-group-name /my-app --retention-in-days 7
aws logs describe-log-groups
aws logs tail /my-app --follow
```

**Key concepts**

- **Log Group** → collection of log streams
- **Log Stream** → sequence of events from one source
- **Retention** → set it to control storage cost
- **Monitoring** → basic = 5-minute intervals, detailed = 1-minute

---

## Cleanup

Delete resources to avoid unexpected charges:

```bash
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PCX_ID
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-subnet --subnet-id $PUB_SUBNET
aws ec2 delete-subnet --subnet-id $PRIV_SUBNET
aws ec2 delete-security-group --group-id $SG_ID
aws ec2 delete-route-table --route-table-id $RTB_ID
aws ec2 delete-vpc --vpc-id $VPC_ID
aws logs delete-log-group --log-group-name /my-app
aws cloudwatch delete-alarms --alarm-names billing-over-10-usd --region us-east-1
aws sns delete-topic --topic-arn $TOPIC_ARN --region us-east-1
```

---

## Best Practices

- Use **separate VPCs** per environment (dev / test / prod)
- Keep databases in **private subnets**
- Apply **least privilege** to IAM and security groups
- Never use root access keys; enable **MFA**
- Create **billing alarms on day one**
- Set **log retention** policies
- Plan **non-overlapping CIDR ranges** early
- Enable **CloudTrail** for audit logs

---

## Troubleshooting

| Problem | Check |
|---|---|
| No internet from instance | Internet Gateway attached, route `0.0.0.0/0` to IGW, public IP assigned |
| Cannot SSH | Security group allows port 22 from your IP, correct key pair, public subnet |
| Peering not working | Connection accepted, routes on both sides, no overlapping CIDRs, security groups |
| No billing data | Region must be **us-east-1**, billing alerts enabled |
| No alert email | SNS subscription confirmed, alarm action set to the topic |
| No logs in CloudWatch | IAM role attached, agent running, correct log group name |
| `AccessDenied` | Check IAM permissions |

Debug command:

```bash
aws sts get-caller-identity --debug
```

---

## Quick Reference

| Task | Command |
|---|---|
| Create VPC | `aws ec2 create-vpc --cidr-block 10.0.0.0/16` |
| Create subnet | `aws ec2 create-subnet --vpc-id <id> --cidr-block 10.0.1.0/24` |
| Create IGW | `aws ec2 create-internet-gateway` |
| Peer VPCs | `aws ec2 create-vpc-peering-connection --vpc-id <a> --peer-vpc-id <b>` |
| Create SNS topic | `aws sns create-topic --name billing-alerts` |
| Billing alarm | `aws cloudwatch put-metric-alarm ...` |
| Create log group | `aws logs create-log-group --log-group-name /my-app` |
| List alarms | `aws cloudwatch describe-alarms` |

**Useful global options:** `--region`, `--profile`, `--output json|text|table`, `--query`, `--debug`, `--no-cli-pager`

---

## Resources

- [Amazon VPC Documentation](https://docs.aws.amazon.com/vpc/)
- [VPC Peering Guide](https://docs.aws.amazon.com/vpc/latest/peering/)
- [Amazon CloudWatch Documentation](https://docs.aws.amazon.com/cloudwatch/)
- [Amazon SNS Documentation](https://docs.aws.amazon.com/sns/)
- [AWS CLI Reference](https://docs.aws.amazon.com/cli/)
- [AWS Free Tier](https://aws.amazon.com/free/)

---

## License

This project is licensed under the [MIT License](LICENSE).

<div align="center">

**If this helped you, give the repo a star.**

</div>
