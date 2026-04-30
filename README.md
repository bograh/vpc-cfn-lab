# Securely Deploying Resources in a VPC using CloudFormation

**Author:** Bernard Ograh

**Lab:** Securely deploying resources in a VPC using cfn LAB

**Region:** us-east-1

## Architecture

A highly available, secure, multi-AZ VPC across `us-east-1a` and `us-east-1b`:

| Component | Detail |
|---|---|
| VPC CIDR | `10.0.0.0/16` |
| Public Subnets | `10.0.1.0/24` (1a), `10.0.2.0/24` (1b) — Apache web tier |
| Private Subnets | `10.0.3.0/24` (1a), `10.0.4.0/24` (1b) — app tier, no public IPs |
| Internet Gateway | Direct internet access for public subnets |
| NAT Gateways | Two — one per AZ, no cross-AZ dependency |
| EC2 Instances | 4 × t3.micro Amazon Linux 2, managed via SSM Session Manager |
| SSH Access | Disabled — no key pairs anywhere in the stack |

## Repository Structure

```
.
├── template.yaml   # Single CloudFormation template (5 commented sections)
└── README.md
```

## Prerequisites

- AWS CLI configured with credentials that have permissions for: CloudFormation, EC2, VPC, IAM, SSM
- AWS CLI v2 installed

## Deploy

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name vpc-cfn-lab \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

## View Outputs (public IPs, instance IDs)

```bash
aws cloudformation describe-stacks \
  --stack-name vpc-cfn-lab \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[*].{Key:OutputKey,Value:OutputValue}' \
  --output table
```

## Validate

1. Open `http://<WebServerAPublicIP>` and `http://<WebServerBPublicIP>` in a browser
2. Connect to any instance: AWS Console → EC2 → select instance → Connect → Session Manager
3. From a private instance, run `ping 8.8.8.8` to verify NAT Gateway internet routing

## Teardown

```bash
aws cloudformation delete-stack \
  --stack-name vpc-cfn-lab \
  --region us-east-1
```

> **Cost note:** NAT Gateways and Elastic IPs incur hourly charges. Delete the stack when the lab is complete.
