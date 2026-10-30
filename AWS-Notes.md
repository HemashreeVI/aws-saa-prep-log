# AWS Notes: EC2, S3 and Core Services

Quick-reference notes covering the most important AWS services, with CLI examples and exam/interview tips.

## Table of Contents

1. [Cloud Basics](#1-cloud-basics)
2. [IAM](#2-iam-identity-and-access-management)
3. [EC2](#3-ec2-elastic-compute-cloud)
4. [EBS and EFS](#4-ebs-and-efs-storage-for-ec2)
5. [S3](#5-s3-simple-storage-service)
6. [VPC](#6-vpc-virtual-private-cloud)
7. [Load Balancing and Auto Scaling](#7-load-balancing-and-auto-scaling)
8. [RDS and DynamoDB](#8-databases-rds-and-dynamodb)
9. [Lambda and Serverless](#9-lambda-and-serverless)
10. [Route 53 and CloudFront](#10-route-53-and-cloudfront)
11. [CloudWatch and CloudTrail](#11-monitoring-cloudwatch-and-cloudtrail)
12. [SQS, SNS and Messaging](#12-messaging-sqs-sns-eventbridge)
13. [Containers: ECS, EKS, ECR](#13-containers-ecs-eks-ecr)
14. [Infrastructure as Code](#14-infrastructure-as-code-cloudformation-and-terraform)
15. [Cheat Sheet and Best Practices](#15-cheat-sheet-and-best-practices)

---

## 1. Cloud Basics

**Cloud computing** means renting computing resources (servers, storage, databases) over the internet and paying only for what you use.

| Model | You manage | AWS manages | Example |
|-------|-----------|-------------|---------|
| **IaaS** | OS, apps, data | Hardware, virtualization | EC2, EBS |
| **PaaS** | Apps, data | OS, runtime, hardware | Elastic Beanstalk, RDS |
| **SaaS** | Just use it | Everything | WorkMail, Chime |

**Global infrastructure**

- **Region**: a geographic area (e.g. `ap-south-1` Mumbai, `us-east-1` N. Virginia). Data stays in the region unless you move it.
- **Availability Zone (AZ)**: one or more isolated data centers inside a region (e.g. `ap-south-1a`). Spread resources across AZs for high availability.
- **Edge Location**: used by CloudFront and Route 53 to serve content close to users.

**Shared Responsibility Model**

- AWS is responsible for security **of** the cloud (hardware, facilities, hypervisor).
- You are responsible for security **in** the cloud (IAM, data, OS patches, security groups, encryption choices).

---

## 2. IAM (Identity and Access Management)

IAM controls **who** can do **what** on **which** resources. It is global (not region-specific) and free.

**Core concepts**

- **User**: a person or app with long-term credentials.
- **Group**: a collection of users sharing permissions.
- **Role**: a set of permissions assumed temporarily (by EC2, Lambda, other accounts). Preferred over access keys.
- **Policy**: a JSON document that allows or denies actions.

**Example policy (read-only access to one bucket)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

**Best practices**

- Lock down the root account: enable MFA and do not use it day to day.
- Follow **least privilege**: grant only what is needed.
- Use **roles** for EC2/Lambda instead of hard-coding access keys.
- Rotate credentials and enable MFA for all users.
- An explicit **Deny** always overrides an Allow.

---

## 3. EC2 (Elastic Compute Cloud)

EC2 provides resizable virtual servers (instances) in the cloud.

### 3.1 Key concepts

- **AMI (Amazon Machine Image)**: template with OS and software used to launch an instance.
- **Instance type**: CPU/RAM/network combination, e.g. `t3.micro` (family `t3`, size `micro`).
- **Key pair**: public/private key used for SSH access (Linux) or password decryption (Windows).
- **Security group**: a stateful virtual firewall attached to the instance.
- **Elastic IP**: a static public IPv4 address you can attach to an instance.
- **User data**: a script that runs on first boot.
- **Instance profile**: attaches an IAM role to an instance.

### 3.2 Instance type families

| Family | Optimized for | Examples |
|--------|---------------|----------|
| **T / M** | General purpose | `t3.micro`, `m5.large` |
| **C** | Compute | `c5.xlarge` |
| **R / X** | Memory | `r5.large` |
| **I / D** | Storage (high IOPS) | `i3.large` |
| **P / G** | GPU (ML, graphics) | `p3.2xlarge`, `g4dn.xlarge` |

`t` instances are **burstable**: they earn CPU credits when idle and spend them under load. Good for dev/test and low-traffic apps.

### 3.3 Purchasing options

| Option | Best for | Savings |
|--------|----------|---------|
| **On-Demand** | Short, unpredictable workloads | None (pay per second) |
| **Reserved / Savings Plans** | Steady usage, 1 or 3 year commitment | Up to ~70% |
| **Spot** | Fault-tolerant, flexible jobs (batch, CI) | Up to ~90%, can be interrupted |
| **Dedicated Host** | Compliance, BYO licenses | Higher cost |

### 3.4 Instance lifecycle

`pending` → `running` → `stopping` → `stopped` → `terminated`

- **Stop**: instance is shut down, EBS volumes persist, no compute charge (storage still charged).
- **Terminate**: instance is deleted; root volume is deleted by default.
- **Hibernate**: saves RAM to disk so the instance resumes where it left off.
- A **public IP changes** after stop/start unless you use an Elastic IP.

### 3.5 Security groups vs NACLs

| | Security Group | Network ACL |
|---|---|---|
| Level | Instance | Subnet |
| State | **Stateful** | **Stateless** |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules together | In rule-number order |

### 3.6 Launching and connecting (CLI)

```bash
# Launch an instance
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0 \
  --count 1

# List instances
aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]" --output table

# Stop / start / terminate
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws ec2 start-instances --instance-ids i-0123456789abcdef0
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0
```

```bash
# SSH into a Linux instance
chmod 400 my-key.pem
ssh -i my-key.pem ec2-user@<public-ip>     # Amazon Linux
ssh -i my-key.pem ubuntu@<public-ip>       # Ubuntu
```

### 3.7 User data example (install a web server on boot)

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl enable --now httpd
echo "<h1>Hello from $(hostname -f)</h1>" > /var/www/html/index.html
```

### 3.8 Common issues

- **SSH timeout**: security group missing inbound port 22, no public IP, or route table without an internet gateway.
- **Permission denied (publickey)**: wrong username or key permissions (must be `chmod 400`).
- **Instance cannot reach internet**: subnet route table missing `0.0.0.0/0` to IGW (public) or NAT Gateway (private).

---

## 4. EBS and EFS (Storage for EC2)

### EBS (Elastic Block Store)

Network-attached block storage, like a virtual hard disk. Lives in **one AZ** and attaches to one instance at a time (Multi-Attach exists for io1/io2).

| Type | Description | Use case |
|------|-------------|----------|
| **gp3 / gp2** | General purpose SSD | Boot volumes, most workloads |
| **io1 / io2** | Provisioned IOPS SSD | Databases needing high IOPS |
| **st1** | Throughput HDD | Big data, logs |
| **sc1** | Cold HDD | Infrequently accessed data |

- **Snapshots** are incremental backups stored in S3 and can be copied across regions.
- Encrypt volumes with KMS; snapshots of encrypted volumes stay encrypted.

### EFS (Elastic File System)

Managed NFS file system that many Linux instances can mount at once, across AZs. Scales automatically; you pay for what you store.

### Instance Store

Physically attached disk with very high speed, but data is **lost when the instance stops or terminates**. Use for temporary data and caches.

---

## 5. S3 (Simple Storage Service)

S3 is object storage: unlimited capacity, 99.999999999% (11 nines) durability. Objects are stored in **buckets**.

### 5.1 Key concepts

- **Bucket**: container with a **globally unique** name, created in a specific region.
- **Object**: file plus metadata. Maximum size **5 TB**; use multipart upload for anything over 100 MB (required above 5 GB).
- **Key**: the object's full path/name, e.g. `images/2026/photo.jpg`. S3 is flat; "folders" are just key prefixes.
- Strong **read-after-write consistency** for all operations.

### 5.2 Storage classes

| Class | Use case | Retrieval |
|-------|----------|-----------|
| **Standard** | Frequently accessed data | Instant |
| **Intelligent-Tiering** | Unknown or changing access patterns | Instant, auto-moves objects |
| **Standard-IA** | Infrequent access, multi-AZ | Instant, retrieval fee |
| **One Zone-IA** | Infrequent, re-creatable data, single AZ | Instant |
| **Glacier Instant Retrieval** | Archive, millisecond access | Instant |
| **Glacier Flexible Retrieval** | Archive | Minutes to hours |
| **Glacier Deep Archive** | Long-term archive, cheapest | Up to 12 hours |

### 5.3 Security

- Buckets are **private by default**. Keep **Block Public Access** enabled unless you really need public content.
- Access control layers: **IAM policies**, **bucket policies**, ACLs (legacy), **pre-signed URLs** for temporary access.
- **Encryption at rest**: SSE-S3 (default), SSE-KMS, SSE-C. **In transit**: HTTPS.
- **Versioning** protects against accidental deletes/overwrites; **MFA Delete** adds extra protection.
- **Object Lock** provides WORM (write once, read many) for compliance.

**Example bucket policy (public read for a static site)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-site-bucket/*"
    }
  ]
}
```

### 5.4 Useful features

- **Lifecycle rules**: automatically move objects between classes or delete them after N days.
- **Static website hosting**: serve HTML/CSS/JS directly from a bucket (pair with CloudFront for HTTPS).
- **Cross-Region Replication (CRR)** and **Same-Region Replication (SRR)**: need versioning enabled on both buckets.
- **Event notifications**: trigger Lambda, SQS or SNS on object create/delete.
- **Transfer Acceleration**: faster long-distance uploads via edge locations.
- **S3 Select / Athena**: query data in place without loading it elsewhere.

### 5.5 CLI examples

```bash
# Create a bucket
aws s3 mb s3://my-unique-bucket-name --region ap-south-1

# Upload / download
aws s3 cp file.txt s3://my-unique-bucket-name/
aws s3 cp s3://my-unique-bucket-name/file.txt ./downloaded.txt

# Sync a folder (upload only changed files)
aws s3 sync ./site s3://my-unique-bucket-name --delete

# List contents
aws s3 ls s3://my-unique-bucket-name --recursive --human-readable

# Pre-signed URL valid for 1 hour
aws s3 presign s3://my-unique-bucket-name/file.txt --expires-in 3600

# Delete an object, then the bucket
aws s3 rm s3://my-unique-bucket-name/file.txt
aws s3 rb s3://my-unique-bucket-name --force
```

### 5.6 Python (boto3) example

```python
import boto3

s3 = boto3.client("s3")

s3.upload_file("report.pdf", "my-unique-bucket-name", "reports/report.pdf")
s3.download_file("my-unique-bucket-name", "reports/report.pdf", "local.pdf")

url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-unique-bucket-name", "Key": "reports/report.pdf"},
    ExpiresIn=3600,
)
print(url)
```

---

## 6. VPC (Virtual Private Cloud)

A VPC is your own isolated virtual network inside AWS.

**Components**

- **CIDR block**: IP range of the VPC, e.g. `10.0.0.0/16`.
- **Subnet**: a slice of the VPC in **one AZ**. *Public* subnets route to an Internet Gateway; *private* subnets do not.
- **Internet Gateway (IGW)**: lets the VPC talk to the internet.
- **NAT Gateway**: lets private-subnet instances reach the internet outbound only. Lives in a public subnet.
- **Route table**: rules that decide where traffic goes.
- **Security groups and NACLs**: firewalls (see EC2 section).
- **VPC Peering / Transit Gateway**: connect VPCs together.
- **VPC Endpoints**: private access to AWS services (S3, DynamoDB) without going over the internet.

**Typical 2-tier layout**

```
VPC 10.0.0.0/16
├── Public subnet  10.0.1.0/24  (AZ-a)  → Load balancer, NAT Gateway, bastion host
├── Public subnet  10.0.2.0/24  (AZ-b)
├── Private subnet 10.0.3.0/24  (AZ-a)  → App servers
├── Private subnet 10.0.4.0/24  (AZ-b)
└── Private subnet 10.0.5.0/24  (AZ-a/b) → Databases
```

---

## 7. Load Balancing and Auto Scaling

### Elastic Load Balancing (ELB)

| Type | Layer | Use case |
|------|-------|----------|
| **ALB** (Application) | 7 (HTTP/HTTPS) | Path/host-based routing, microservices, containers |
| **NLB** (Network) | 4 (TCP/UDP) | Extreme performance, static IP |
| **GLB** (Gateway) | 3 | Third-party virtual appliances |

- Use **health checks** so traffic only goes to healthy targets.
- Register instances in a **target group**.

### Auto Scaling Group (ASG)

Automatically adds or removes EC2 instances based on demand.

- **Launch template**: defines AMI, instance type, security groups, user data.
- **Min / Desired / Max** capacity.
- **Scaling policies**: target tracking (e.g. keep CPU at 50%), step scaling, scheduled scaling.
- Spread instances across multiple AZs for fault tolerance.

---

## 8. Databases: RDS and DynamoDB

### RDS (Relational Database Service)

Managed relational databases: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server and Aurora.

- AWS handles patching, backups and failover; you cannot SSH into the host.
- **Multi-AZ**: standby copy in another AZ for **high availability** (automatic failover).
- **Read Replicas**: extra copies for **read scaling** (asynchronous).
- **Aurora**: AWS-built engine compatible with MySQL/PostgreSQL, up to 5x faster than standard MySQL; storage auto-scales.
- Automated backups with point-in-time restore, plus manual snapshots.

### DynamoDB

Fully managed **NoSQL** key-value/document database with single-digit millisecond latency at any scale.

- **Primary key**: partition key, or partition key + sort key.
- **Capacity modes**: on-demand or provisioned (RCU/WCU).
- **GSI / LSI**: secondary indexes for alternative query patterns.
- **DAX**: in-memory cache for microsecond reads.
- **Streams**: capture item changes to trigger Lambda.
- **TTL**: auto-expire items.

### Other data services

| Service | Purpose |
|---------|---------|
| **ElastiCache** | In-memory cache (Redis, Memcached) |
| **Redshift** | Data warehouse / analytics |
| **Neptune** | Graph database |
| **DocumentDB** | MongoDB-compatible document DB |

---

## 9. Lambda and Serverless

**AWS Lambda** runs your code without managing servers. You pay per request and per millisecond of execution.

- Triggers: API Gateway, S3, DynamoDB Streams, SQS, EventBridge schedules, and more.
- Limits: up to **15 minutes** per invocation, up to 10 GB memory, ephemeral `/tmp` storage.
- **Cold start**: the first invocation after idle time is slower while the environment spins up.
- Attach an **execution role** for permissions.

**Example Python handler**

```python
import json

def lambda_handler(event, context):
    name = event.get("name", "world")
    return {
        "statusCode": 200,
        "body": json.dumps({"message": f"Hello, {name}!"}),
    }
```

**Related serverless services**

- **API Gateway**: create REST/HTTP/WebSocket APIs in front of Lambda.
- **Step Functions**: orchestrate multi-step workflows.
- **Cognito**: user sign-up, sign-in and identity federation.
- **DynamoDB + S3**: the usual serverless data layer.

---

## 10. Route 53 and CloudFront

### Route 53 (DNS)

- Hosted zones hold DNS records (A, AAAA, CNAME, MX, TXT, **Alias**).
- **Alias** records point to AWS resources (ELB, CloudFront, S3) and work at the zone apex (root domain).
- **Routing policies**: Simple, Weighted, Latency, Failover, Geolocation, Multi-value.
- Health checks enable DNS failover.

### CloudFront (CDN)

- Caches content at edge locations worldwide for lower latency.
- Origins: S3, ALB, EC2, or any HTTP server.
- Use **Origin Access Control (OAC)** so users can reach S3 only through CloudFront.
- Supports free HTTPS certificates through **ACM**.
- Invalidate cached objects with `aws cloudfront create-invalidation`.

---

## 11. Monitoring: CloudWatch and CloudTrail

| Service | What it does |
|---------|--------------|
| **CloudWatch Metrics** | CPU, network, disk and custom metrics (EC2 basic monitoring = 5 min, detailed = 1 min) |
| **CloudWatch Logs** | Collect and search application and system logs |
| **CloudWatch Alarms** | Trigger notifications or Auto Scaling actions when a metric crosses a threshold |
| **CloudTrail** | Records **API calls** (who did what, when). Used for auditing and security |
| **AWS Config** | Tracks resource configuration changes and compliance |
| **X-Ray** | Distributed tracing for applications |

Note: memory and disk-usage metrics for EC2 are **not** collected by default; install the CloudWatch Agent.

---

## 12. Messaging: SQS, SNS, EventBridge

| Service | Model | Use case |
|---------|-------|----------|
| **SQS** | Queue (pull) | Decouple components; each message processed by one consumer |
| **SNS** | Pub/Sub (push) | Fan-out one message to many subscribers (email, SMS, Lambda, SQS) |
| **EventBridge** | Event bus | Route events from AWS services/SaaS apps by rules; scheduled jobs |
| **Kinesis** | Streaming | Real-time data streams (logs, clicks, IoT) |

- SQS **Standard** = at-least-once delivery, best-effort ordering. **FIFO** = exactly-once, strict ordering.
- Use a **Dead-Letter Queue (DLQ)** to capture messages that repeatedly fail.
- Common pattern: **SNS → multiple SQS queues** (fan-out).

---

## 13. Containers: ECS, EKS, ECR

- **ECR**: private Docker image registry.
- **ECS**: AWS-native container orchestration. Run on **EC2** (you manage hosts) or **Fargate** (serverless, no hosts to manage).
- **EKS**: managed Kubernetes control plane.
- **Fargate**: pay per vCPU/memory of running tasks; ideal when you do not want to manage servers.

```bash
# Push an image to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com

docker build -t myapp .
docker tag myapp:latest <account-id>.dkr.ecr.ap-south-1.amazonaws.com/myapp:latest
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/myapp:latest
```

---

## 14. Infrastructure as Code: CloudFormation and Terraform

Define infrastructure in files so it is repeatable and version-controlled.

**CloudFormation (YAML) example: an S3 bucket and an EC2 instance**

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration:
        Status: Enabled

  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: ami-0abcdef1234567890
```

**Terraform equivalent**

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name"
}

resource "aws_instance" "my_instance" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
}
```

```bash
terraform init
terraform plan
terraform apply
terraform destroy
```

---

## 15. Cheat Sheet and Best Practices

### Which service for what?

| Need | Service |
|------|---------|
| Virtual server | EC2 |
| Store files / backups / static site | S3 |
| Disk for an EC2 instance | EBS |
| Shared file system | EFS |
| Relational database | RDS / Aurora |
| NoSQL database | DynamoDB |
| Run code without servers | Lambda |
| Distribute traffic | ELB |
| Scale instances automatically | Auto Scaling |
| DNS | Route 53 |
| CDN | CloudFront |
| Queue / pub-sub | SQS / SNS |
| Logs and metrics | CloudWatch |
| Audit API calls | CloudTrail |
| Containers | ECS / EKS / Fargate |
| Infrastructure as code | CloudFormation / Terraform |

### Best practices (Well-Architected pillars)

1. **Operational excellence**: automate with IaC, monitor everything.
2. **Security**: least privilege, MFA, encrypt data, keep resources private.
3. **Reliability**: multi-AZ, backups, health checks, auto scaling.
4. **Performance efficiency**: pick the right instance/storage type, use caching and CDN.
5. **Cost optimization**: right-size, use Savings Plans/Spot, delete unused resources, set budgets and billing alarms.
6. **Sustainability**: use efficient resources and scale down when idle.

### Cost tips for learners

- Use the **Free Tier** and set a **billing alarm / AWS Budget** on day one.
- **Stop or terminate** EC2 instances when finished; release unused Elastic IPs.
- NAT Gateways and idle load balancers are common surprise charges.
- Delete test S3 buckets, EBS snapshots and RDS instances after practice.

---

## Useful Links

- [AWS Documentation](https://docs.aws.amazon.com/)
- [AWS CLI Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/index.html)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Free Tier](https://aws.amazon.com/free/)

---

*Notes compiled for learning and quick revision. Verify limits and pricing in the official AWS docs, as they change over time.*
